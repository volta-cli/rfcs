- Feature Name: volta_use
- Start Date: 2024-02-22
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary

[summary]: #summary

Adding a new command, `volta use`, which combines the effects of `volta install` and `volta pin` into a single command that is more intuitive and better matches the mental model of Volta.

# Motivation

[motivation]: #motivation

Early on in Volta's history, we picked the verb `install` to represent choosing the default version of tools (i.e. the version used when outside of a pinned project). This was chosen, in part, to align with users' expectations from other similar tools. However, due to Volta's auto-download behavior when it detects a version that isn't currently cached, `install` centers the wrong action. The more important part of `volta install` is the selecting a default version.

This has led to some confusion for users. We get questions fairly regularly asking how users should select an "active" version of Node, once they have installed them using `volta install`. Additionally, by centering the install action, rather than the selection, `volta install` doesn't help users internalize Volta's mental model of "select a version and Volta will automatically download and make sure it's available."

Selecting a different verb for the action of choosing a tool version should reduce confusion and make it more clear how Volta is intended to work.

# Pedagogy

[pedagogy]: #pedagogy

This will be a fairly large change to our recommended workflows. As `volta use` will work for both pinning _and_ selecting a default version, we will need to update the documentation throughout to take advantage of the new command. We will also likely want to write a blog post for the release which includes this change, explaining the reasoning and the new workflows.

Notably, we will _not_ be removing the existing `volta install` and `volta pin` commands right away. They will continue to work as they already do. In the future, we might consider adding notifications to them to point users to the updated command, but for the initial implementation we don't want to disrupt current users' workflows.

All that said, the new command should simplify things a lot, especially for new users. It also will do a better job of living up to our website by truly having a _single_ command that users can use to get set up.

# Details

[details]: #details

## Command definition

```
USAGE:
    volta use [FLAGS] <tool[@version]>...

FLAGS:
    -g, --global    Select the global version of a tool, which is used everywhere outside of pinned projects
    -p, --pin       Pin a version of a tool to the local project, creating a Volta entry in package.json

ARGS:
    <tool[@version]>... Tools to make available, like `node@lts` or `yarn@^1.14`
```

## Behavior

The behavior of `volta use` will ultimately be the same as the current `volta install` or `volta pin`. However, when run without flags, it will be contextual, based on where the command is being run. So if a user runs `volta use node@18` in a pinned project, the project will be updated to reflect the newly selected version. If they instead run `volta use node@18` _outside_ of a pinned project, then the default Node version will be updated. This ensures that running `volta use node@version` will always update the version that is currently active, meaning the next run of `node` will use that updated version.

Notably, if a user runs `volta use node` within a project that doesn't currently have a pinned version (i.e. has no `"volta"` entry in `package.json`), then the command will update the global / default version of Node, it will _not_ create a pin if one doesn't exist.

If a user wants to change a version that isn't currently active, they can use the flags to change the behavior.

### `--global`

Running `volta use --global` will work identically to the existing `volta install`, updating the default version regardless of whether the current directory is part of a pinned project.

### `--pin`

Similarly, `volta use --pin` will work identically to the current `volta pin`, always updating the current project. This includes creating a `volta` key in `package.json` if it isn't there already.

## Implementation

The technical implementation of `volta use` should be fairly straightforward. The new command will use the flags and context (i.e. if we are currently in a pinned project or did the user pass `--global`) to determine whether the behavior should be equivalent to `volta install` or `volta pin`. It will then delegate to the appropriate existing command behavior based on that determination. The ultimate flow should be:

- If `--global` was set -> use `volta install` behavior
- If `--pin` was set -> use `volta pin` behavior
- If _both_ were set, error (this can be handled by Clap I believe by making the two flags exclusive)
- If neither flag was set:
  - If we are in a pinned project (one that already has a `"volta"` key in `package.json`) -> use `volta pin` behavior.
  - Otherwise, use `volta install` behavior.

# Critique

[critique]: #critique

## Confusion around omitting version

One nice part about the existing `volta install` and `volta pin` commands is that they naturally work well with choosing a tool without selecting a version. For example, running `volta install node` makes intuitive sense as "pick whatever version and make it available." By contrast, `volta use node` doesn't immediately (to me, at least!) jump out as _picking_ a version of Node. `volta use node@lts` or similar, _do_ make sense, but omitting the version feels a little more awkward with `use` as the verb.

## Changing only the verb for `install`

One alternative would be to only change the verb for `install`, leaving `pin` completely untouched. This would be the least disruptive change, however it's very dependent on the verb / command that we choose. With the proposal of `volta use`, it would feel very unintuitive to run `volta use node@21` in a project, and then immediately get a message that it's set as the global but your local settings override. As a user, I would expect `volta use node@21` to mean the next call to `node --version` to display Node 21 is running. In order to make that work as users expect, we need to unify the two so that `volta use` _does_ actually change the Node for the current context.

Another difficulty of the unified behavior is that it's more difficult to explain when users run into edge cases. Right now there's a clear separation between "local" and "global" with two different commands. Unifying them means we'll need to clearly document how `volta use` behaves in each situation, and how to override that (with the `--global` or `--pin` flags).

## `volta default` or `volta global`

In the vein of replacing the verb for `install`, I also considered `default` or `global` for the command. The major issue with those is that they're adjectives, rather than verbs, so it's not immediately clear what the command means. Does `volta default node` mean "choose a default node" or "list the current default node"? That confusion is ultimately why I decided against using either of these for the command.

## `volta set`

Another possibility for the command would be to make it more verbose, e.g. `volta set default node`. That solves the problems around `volta default` or `volta global` by adding a verb to make it very clear what you are doing. However, it makes the commands a fair bit more cumbersome to use, which I feel is ultimately a net negative compared to a single command.

# Unresolved questions

[unresolved]: #unresolved-questions

- How do we handle the fact that you can install a 3rd-party package globally, but for a pinned project, we require users to install them normally using their package manager of choice? That is, what should we do when a user runs `volta use typescript` while they are in a package that is pinned?
