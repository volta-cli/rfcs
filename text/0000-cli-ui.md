- Feature Name: cli-ui-improvements
- Start Date: 2019-02-25
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

Transition from [Docopt] to [StructOpt], for an easier path to nice user experience (and better developer experience when working on) the command line interface.

[Docopt]: https://docs.rs/docopt/1.0.2/docopt/
[StructOpt]: https://docs.rs/structopt/0.2.14/structopt/

# Motivation
[motivation]: #motivation

Our current command line parsing tool, [Docopt], works well but has several important limitations:

- It does not readily support generating completions for any shell other than bash (and even these are a bit hairy).

- It is, per its own description, a “black box”: if it works, it works well; if it doesn’t, developers are on their own for sorting out what is not parsing as expected and why. Those failures simply specify that the arguments failed to parse, not why or how, and they happen at runtime rather than compile time.

- It requires custom wiring-up internally for every distinct subcommand to generate consistent help output, version output, etc., and custom parsing for each of those inputs.

Switching to [StructOpt] improves our story in each of these areas:

- It supports automatically generating completions for all major Unix shells and Powershell during the build step.

- The configuration is validated by the compiler rather than at runtime, and the compiler’s error messages are usually much clearer about what is wrong than are the runtime errors from Doctopt.

- It properly passes through global options to subcommands, and it understands all variants of “help” invocations (position and both flag styles) out of the box, and can be trivially configured to display help/usage information when no arguments are passed.

Though StructOpt is not *universally* better than Docopt, it can generate similar output, but with a lower maintenance burden and a number of niceties *not* supported by Docopt.

# Pedagogy
[pedagogy]: #pedagogy

This is purely an internal, developer-facing change, with *benefits* to the user but no changes in actual use of Notion. The documentation for StructOpt (and its core dependency [`clap`]) are sufficient for developers working on the project.

[`clap`]: https://docs.rs/clap/2.32.0/clap/

# Details
[details]: #details

Migrating to StructOpt is straightforward, if somewhat time-consuming. 

1. Map the existing Docopt commands from the usage strings which specify them currently into a StructOpt-annotated `struct` or `enum` definition as appropriate.
2. Parse arguments in `Command::go` with StructOpt’s `from_args` method instead of the `Command::parse` method.
3. Update the `Command` trait to remove the elements provided automatically by StructOpt and make the corresponding changes to implementors:
	- the `help` method: superceded by StructOpt’s built-in help handling
	- the `parse` method: superceded by StructOpt’s `from_args` method (see step 2)
	- the `USAGE` constant: migrated to the `struct` or `enum` definition in step 1.
	- the `Args` associated type: superceded by the struct or enum used to define the command
4. Rework the existing `Command::run` implementations for each command, since they will no longer need to match against a `Help` variant. This should largely be a matter of *simplifying* each, in many cases simple to tag the `session` with `ActivityKind` and execute a specific operation.

# Critique
[critique]: #critique

The alternative paths we could take in this space (besides shaving the yak of writing our own custom option parser!) are:

- <b>Continue with Docopt</b>, improving on the existing implementation to provide a nice experience for users.
- <b>Migrate to [`clap`]</b>, the tool on top of which StructOpt is built.

## Continue with `docopt`

The biggest reason to stick with Docopt is simply that we already have it and it already works. We can improve the UI within the existing `docopt` tooling: it is possible to generate nicely-styled outputs in a cross-platform fashion using (for example) [`ansi_term`][ansi_term] and [`textwrap`][textwrap] for colored outputs.

[ansi_term]: https://docs.rs/ansi_term/0.11.0/ansi_term/
[textwrap]: https://docs.rs/textwrap/0.11.0/textwrap/index.html

Doing this requires *some* rework. At present the usage strings are in const contexts and none of the terminal color crates support `const fn` yet, and tools like `textwrap` *cannot*. We would therefore need to move them into a runtime context and write the formatting strings ourselves. This is not difficult, but it is a small increase in maintenance burden.

We would also eventually need to work with Docopt to support generating completions for all shells for the best user experience.

However, as noted in [<b>Motivation</b>](#motivation), Docopt is somewhat brittle and its errors quite abstruse, and it is therefore likely to be a headache for future developers (as it has for the author of this RFC already during research!). Much more simply works out of the box with StructOpt than with Docopt, and it requires a great deal less custom implementation when adding new flags or commands.

Not switching increases the cost of adding new features to Notion and entails doing a good deal of further development to support nice UX features like completions.

## Migrate to `clap`

As noted in [<b>Details</b>](#details), StructOpt is built on top of `clap`, which is a programmatic API for building UIs. `clap` is strictly a superset of StructOpt functionality-wise, but is much more verbose *and* does not yet include a way to deserialize command-line arguments into a data structure—as do both Docopt and StructOpt. This makes it much less pleasant to work with: every argument or subcommand must be matched via explicit calls like this:

```rust
let cli = /* configure the app */
let args = cli.get_matches();
if let Some(arg) = args.value_of("some_arg")
```

The only *interesting* capability it affords that is not offered in StructOpt is the ability to perform certain transformations at runtime, for example, setting custom colors or text wrapping on the `template` argument—which StructOpt cannot do since it works via compile-time code generation.

The user experience is much the same with StructOpt as with `clap`, and the developer experience with StructOpt is *much* better. Notably, the in-progress v3 branch of `clap` includes StructOpt’s functionality, just with the `StructOpt` custom derive renamed to `Clap` and the `structopt` procedural macro to `clap`. Though the timeline for the v3 release is unclear—it has been in progress for a year as of the time this RFC is opened—this provides good evidence that StructOpt is not a dead end.

# Unresolved questions
[unresolved]: #unresolved-questions

- How should we go about using the generated completions, however we generate them – via a command like `notion completions`, or during installation? This question we can simply defer to later. A `build.rs` script can make them available during installation; or they can be generated by the user on command at runtime; or possibly we can make *both* of these options available.