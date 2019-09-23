- Feature Name: volta_run
- Start Date: 2019-09-14
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Add a `volta run` command, allowing users to execute tools with an override on the detected platform.

# Motivation
[motivation]: #motivation

Volta currently will automatically detect the platform (Node runtime and yarn version) based on the context of where you are in the file system. However, there are occasionally times when you want to run a command using a different version of Node or Yarn than what Volta detects, without changing the pinned or default versions. To support those use-cases, we should provide a command that allows users to execute a command with a platform specified on the command-line.

# Pedagogy
[pedagogy]: #pedagogy

We will need to document the existence of `volta run` and how the users can specify the versions of Node and Yarn when running a command. This will be provided on the command line help output, however it should also be explained in detail in the documentation. We should also document on the website how the overall platform is resolved if the user specifies only one of Node or Yarn, so that the behavior is clear to users.

# Details
[details]: #details

The new command will have the structure:

```
volta run [--node <version>] [--yarn <version> | --no-yarn] [--env <variable>=<value>] <command>
```

Some examples:

```
volta run --node 8 --yarn 1.10 ember new my-app --yarn
```

Would run the command `ember new my-app --yarn` with Node 8 and Yarn 1.10.

```
volta run --node latest --env EMBER_ENV=production ember test
```

Would run the command `ember test` with the latest Node and with the environment variable `EMBER_ENV` set to `production`.

Of note, all of the flags will need to be specified _before_ the command to be executed, as everything after the command will be treated as arguments to the command itself. This will allow users to specify a full command, with any command-line flags they need, without having to append `--` to separate the `volta run` command from the others.

## Platform Resolution

If the user specifies only one (or none) of the platform settings when calling `volta run`, the remaining settings will fall back to the existing context-aware logic. So if the user is in a project with `node@10.16.0` and `yarn@1.17.3` pinned, and runs:

```
volta run --node 8 ember serve
```

We will resolve the a platform that has `node@8` and `yarn@1.17.3`, with the latter pulled from the existing pinned project and the former overridden by the command-line flag. This allows users to specify the parts of the platform that they want to change, while avoiding the necessity of specifying everything, even those things that don't change.

## The `--no-yarn` flag

Instead of specifying the version of `yarn` to use, the user can also pass `--no-yarn` to `volta run`, which will tell the system to not use any Yarn and _not_ allow the Yarn version to be inherited from the project or system. This will likely be a rarer use-case and so can potentially be added later in the implementation, but it should be considered as part of the design of the command.

## 3rd-Party Binaries

After resolving the platform, if the command the user is executing is a 3rd party binary (not `node`, `npm`, `npx`, or `yarn`), then we will need to maintain the existing resolution of that binary relative to the current project (if any). That means if we are inside a project with that binary as a dependency, we will still use the project-local version of that tool, however we will use the Node / Yarn versions specified on the command-line to execute the tool.

## Specifying Environment Variables

The `volta run` command should have a `--env` flag that can be set multiple times to specify different environment variables. These flags will allow the user to add extra environment variables to the command they are executing, in a cross-platform way. On Unix systems, this behavior can be simulated by setting the environment variables while calling `volta run` (e.g. `EMBER_ENV=production volta run ...`), however on Windows, there isn't a simple shorthand for adding environment variables to a one-off command. The `--env` option on `volta run` will allow users to set those environment variables at the command line on any system, without having to worry about how that system manages its environment.

## Verbose Output

In order to provide meaningful verbose output, we will need to update the `SourcedPlatformSpec` model to support platforms that come from the command line as well. This will likely require a refactor to support specifying the version for each tool explicitly, as opposed to having a single `Source` for the Platform as a whole.

## Additional Package Managers

Currently, the only package manager that we support specifying on its own is Yarn. However, there are plans to support `npm` in the near future and farther out, we may want to support others (e.g. `pnpm`, `entropic`, etc). For each aspect of the platform that we support, we'll need to accompanying flags to the `run` command that support specifying the version of that tool. (i.e. `--pnpm <version>` and `--no-pnpm`).

# Critique
[critique]: #critique

## Command Name

There are alternative names we could use for the command, such as:

- `volta exec`
- `volta do`

However, `volta run` aligns with the "run script" command in both `npm` and `yarn`, so it seems to fit the best for this use-case.

## Version Flags

Instead of having `--node <version>` and `--yarn <version>`, we could support a more general syntax of `--with <tool>@<version>`. That would align with the syntax that `volta install` and `volta pin` use. However, it adds some potential ambiguity in the command:

- When a user runs `volta pin yarn`, that is interpreted as `volta pin yarn@latest`.
- If a user runs `volta run --with yarn`, should that _also_ be interpreted as `volta run --with yarn@latest`? It's not clear that's necessarily the best default, or that any default makes sense.
- Should we instead require the `@<version>` in `volta run`? If so, then the syntax is different from `volta install`, but in a subtle and somewhat hard-to-understand way.

Beyond the potential confusion, there is also the issue that for common use-cases, `--with node@10 --with yarn@1.17` is more verbose than `--node 10 --yarn 1.17`. Given that `volta run` will often be used for one-off commands, making the syntax shorter is a boon to usability.

## Platform Resolution

Instead of inheriting the platform from the current, we could treat it as purely defined on the command-line. However, this would mean that if a user doesn't specify `--node`, no commands will work, since all JS tools ultimately require `node` at some point. Additionally, users have to specify `yarn` if their command needs it, even if they only want to change the version of `node` being used.

## Overriding the shims directly

Instead of (or in addition to) the `volta run` command, we could provide the ability for users to override the shims using environment variables (e.g. `VOLTA_NODE` to change the version of Node). This would provide a different way of specifying the platform that overrides the normal resolution algorithm. However, ultimately this approach has a few issues that clash with the goals of Volta:

- It requires users to think about the existence of Volta when running the shims. When a user calls `yarn`, the goal of Volta is that they shouldn't have to think about the fact that Volta is figuring it out, it should just work as they expect. If we add environment variables that can change how those work, it could lead to difficult-to-debug situations where a user has an exported variable in their terminal that is breaking things and it's not immediately obvious why.

- It's not as ergonomic cross-platform. As mentioned above, Windows doesn't have as easy as shorthand for setting environment variables on a given command, so Windows users would have to jump through more hoops to get access to this behavior. Volta is committed to keeping Windows as a first-class citizen, so we don't want to break that by relying too much on Unix-isms.

- If added in conjunction with `volta run`, it would provide multiple ways to do the same thing, which comes with its own set of problems. It may not be clear to users that `volta run --node 8 <command>` and `VOLTA_NODE=8 <command>` behave the same, so it could lead to confusion around the difference between those two commands.

- If a user specifies `VOLTA_NODE=8` while running `volta run --node 10`, we will need to have a precedence order for which version wins. While it would be well-documented, there's still a potential for confusion if a user expects us to do something different from what we do.

Due to these issues, it seems better to stick to a single method for overriding the platform resolution logic: `volta run`.

## JSON Configuration File

As an alternative to specifying the versions on the command-line, we could provide the ability for the user to specify the versions of the various platform tools in a JSON file, and then pass that file to `volta run`. We could do this in a way that is mutually exclusive with the various tool flags, so that there isn't any ambiguity of config file vs command-line flag. 

However, this would introduce some confusion with the `"volta"` key in `package.json`. In the `package.json` case, we require that the version be a fully resolved version, for repeatability. However, in the case of `volta run --config <file>`, we want to support semver ranges and `latest`, so the two types of JSON configuration would be non-obviously different.

Additionally, a JSON configuration wouldn't be able to specify the `--no-yarn` case as easily. We could support a specific value, e.g. the literal `false`, as a way of telling Volta to leave off Yarn and not inherit it, but that's not as discoverable as `--no-yarn` would be.

# Unresolved questions
[unresolved]: #unresolved-questions

- Do we want to support changing a package version as well, letting the user run a different version of a tool than they currently have installed / available?
