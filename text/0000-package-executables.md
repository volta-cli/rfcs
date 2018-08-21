- Feature Name: package_executables
- Start Date: 2018-08-20
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

Many Node packages contain one or more executable files that they would like to install into the PATH. Notion should make these executables available to the user:

- locally on a per-project basis (project tools)
- globally, installed as if they are static binaries (user tools)

# Motivation
[motivation]: #motivation

## Project Tools

Notion will provide some killer features for working in Node projects (that no other version managers have):

### Isolating the project scripts:

As it currently works, any package executables referenced in the `"scripts"` resolve to executables installed locally for the project. But, these scripts can also make use of global executables, which may not be consistent for different collaborators. Notion removes this loophole by stripping the global package binaries from the PATH so the scripts run as if there are no global executables installed. This provides consistent environments for project collaborators as well as CI.

### Command-line access to package executables:

When the user is in the Node project, Notion gives command-line access to the package executables for the project. This is normally only possible through the project scripts, but that requires 1) setting up the script, and 2) running `npm run <script>`. With Notion, the user can just run the package executable directly, like `ember build`, instead of having to setup a script like `"build": { "ember build" }` and then `npm run build`.


## User Tools

Once a user installs a package executable globally, it should just work, and continue to work. The availability of the executable should not depend on the active or installed versions of Node.

Most version managers provide the ability to install package executables on a per-version basis. Once a user switches Node versions, they must re-install the executable for it to be available. Notion will eliminate this hassle - once the executable is installed, it will be statically linked to the Node version used to install it, and it will always run with a pristine version of Node. It will not break because of version switches, and its environment will not be polluted by other globally-installed package executables.

# Pedagogy
[pedagogy]: #pedagogy

Because this model is new and different from how other version managers work today, we will need to teach this to users, through documentation and good `--help` explanations.

## Concepts

### Shims

Users may already be familiar with shims, since these are commonly used by other version managers. Notion shims intercept calls to package executables that have been installed with Notion, and then redirect execution to the right executable based on the environment and current directory.

### Project tools

These are package executables installed locally for a Node project.

### User tools

These are package executables installed globally, as if they are static binaries available on the PATH.

## Examples

### Project tools:

A user is in a Node project which uses TypeScript, and has a dependency on `tsc`.
The user runs these commands for their workflow:

- `yarn install` - to install that project’s dependencies (and the `tsc` shim is automatically created)
- `tsc src/file.ts` - to compile some file

The last command will be redirected by Notion to actually run `/full/path/to/node_modules/.bin/tsc src/file.ts`.

The user is getting the version of `tsc` that is required by that project (along with the version of `node` used by that project). This is all transparent to the user.

### User tools:

A user is not in a Node project. They are using a Node version of 10.1.2 (installed with Notion), and they run these commands:

- `notion install cowsay` - this installs `cowsay` as a user tool
- `cowsay 'hi'` - this runs `cowsay` as expected
- `notion install node 8` - this sets the user version of Node to 8.x.x
- `cowsay 'hello'` - this still works as expected, and uses Node 10.1.2 to run it

Again this is transparent to the user. When they install a tool, it continues to work using the version of Node that it was installed with.

### Combination of project and user tools:

The user toolchain includes the `madge` tool (which is useful for analyzing the module graph of Node projects). The `madge` tool requires at least Node 6.x to run.

The user is using this tool to analyze a project (which is the point of the tool), but it’s not part of the scripts for that project. So it’s not a project tool.

The project happens to be using Node 4.x (older than what `madge` requires), but it works just fine. `madge` is pinned to the version of Node it was installed with in the user toolchain, not the version of Node the user’s project is pinned to.

Note also that the project scripts can’t accidentally depend on `madge` because it is a user tool and the scripts don’t see it in their PATH.


## How to think about these features

### Project tools

Any package executables installed by direct `dependencies` and `devDependencies` of a Node project are available for the user to use when they are in that project. These executables will run using the Node version that is configured for the project.

### User tools

When a user installs a package executable globally, it’s like they are installing a static binary. It will work in the same way that it was installed, despite changing directories or changing Node versions.

# Details
[details]: #details

To make this work, Notion will put shims on the PATH in `~/.notion/shims/` to intercept execution of package executables and delegate appropriately. These shims are implemented as symlinks to the `launchbin` executable, where the name of the symlink is the name of the shimmed executable. Since the shims directory will come first (or very early) in the PATH, when any shimmed executable is run, it will execute `launchbin` with the name of the executable as the first argument.

Then `launchbin` uses this logic to determine how to delegate execution:

- If the user is in a Node project, and the executable is a direct dependency of the current project, then that is executed (using the absolute path to the executable). We determine that an executable is a direct dependency by reading the `"bin"` section of each direct dependency’s `package.json`. Only direct dependencies are executed because `npm` and `yarn` install executables for the complete tree of project dependencies, not just direct dependencies. Exposing all of those would be a security risk as well as confusing to debug.
- Otherwise this is installed as a user tool, so that is executed (using the absolute path to the executable).

### Project tools

Project executables will run using the version of Node that is configured for the project. This happens automatically, because any references to `node` in the script will resolve `node` to the version configured in the `"toolchain"` section of `package.json`. (And most executable scripts have `#!/usr/bin/env node` as their first line.) If the “toolchain” is not configured, then this is an error.

For the example above:

> A user is in a Node project which uses TypeScript, and has a dependency on `tsc`.
> The user runs these commands for their workflow:
> *  `yarn install` - to install that project’s dependencies (and the `tsc` shim is automatically created)
> *  `tsc src/file.ts` - to compile some file

When running `yarn install`, a symlink is automatically created in the PATH, named `tsc`, pointing to `launchbin`.
When the user runs `tsc src/file.ts`, that redirects to run `launchbin` with `tsc` as the first argument. Because the project is configured to use `node`, and `tsc` is a direct dependency of the project, `launchbin` passes execution to `/full/path/to/node_modules/.bin/tsc` with the rest of the arguments.


### User tools

User tools will run using the version of Node that was used to install them. Installing a user tool will automatically create a shim in the PATH.

Notion will keep a registry of installed user tools, along with the version of Node that they should use. When the executable is run, Notion will set the `NOTION_NODE_VERSION` environment variable for that executable, so that it will resolve that version of Node.

For the example above:

> A user is not in a Node project. They are using a Node version of 10.1.2 (installed with Notion), and they run these commands:
> *  `notion install cowsay` - this installs `cowsay` as a user tool
> *  `cowsay 'hi'` - this runs `cowsay` as expected
> *  `notion install node 8` - this sets the user version of Node to 8.x.x
> *  `cowsay 'hello'` - this still works as expected, and uses Node 10.1.2 to run it

Running `notion install cowsay` will install `cowsay` somewhere in the `.notion` directory, and automatically create a shim named `cowsay` that points to `launchbin`.

When the user runs `cowsay 'hi'`, this executes `launchbin` like in the local case, but it determines that `cowsay` is not a local project executable so it must be a user tool. It constructs the `/full/path/to/cowsay`, and sets `NOTION_NODE_VERSION=10.1.2`, then executes that. When `cowsay` runs, it uses Node version 10.1.2.

Even after the user switches Node versions, `launchbin` still sets `NOTION_NODE_VERSION=10.1.2`, so that `cowsay` will run that version of Node.

### Implementation

Some of this functionality has already been implemented:

- https://github.com/notion-cli/notion/pull/89 - basic logic for package executables
- https://github.com/notion-cli/notion/pull/100 - `notion shim` command (manually managing shims)

Some things remain to be implemented:

- User tools should be locked to a specific Node version.
- Shims should not fall back to running whatever is on the system (this is what currently happens, and it should throw an error in that case).


# Critique
[critique]: #critique

### Determining the executable name

Currently, this is done by looking at `arg0` of the incoming arguments to `launchbin`. This is normally the same as the executable name, but in some cases it may not be. Possible solutions, instead of using symlinks:
* Shims could be small shell scripts that explicitly set the executable name in the arguments that are passed to `launchbin`.
* Shims could be small binary launchers with different string constants for the executable name.


# Unresolved questions
[unresolved]: #unresolved-questions

These don’t necessarily need to be resolved with this RFC, but they should be resolved before Notion 1.0:

- For executables that don’t begin with `#!/usr/bin/env node`, what needs to be done to ensure that they are using the correct version of Node?
- How do executables work on Windows? Should they be wrapped with `cmd`?
- How do we detect that the user is installing a dependency (any variation of `yarn add`, `npm install`, etc.) and automatically create shims for those?
- Are there any issues in Windows shell scripts (Powershell or batch) with delegating execution and forwarding signals, etc.?
