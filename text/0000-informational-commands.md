- Feature Name: informational_commands
- Start Date: 2019-05-15
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Add an informational command, `volta list`, replacing the `volta current` command and expanding on its capabilities. The `list` command will allow users to see their default and project-specific toolchains, and will support both human-friendly output and a tool-friendly output.

- [Summary](#summary)
- [Motivation](#motivation)
- [Pedagogy](#pedagogy)
    - [Why `list`?](#why-list)
    - [Prior art](#prior-art)
- [Details](#details)
    - [Information supplied by the command](#information-supplied-by-the-command)
    - [Output modes](#output-modes)
    - [Detailed command output](#detailed-command-output)
        - [Assumed Configuration](#assumed-configuration)
        - [`volta list` (no flags)](#volta-list-no-flags)
        - [`volta list all`](#volta-list-all)
        - [`volta list <package>`](#volta-list-package)
        - [`volta list <tool>`](#volta-list-tool)
    - [Deprecating `volta current`](#deprecating-volta-current)
- [Critique](#critique)
    - [Use another command name](#use-another-command-name)
    - [Use shorthands instead of `list`](#use-shorthands-instead-of-list)
    - [Keep `volta current` as an alias for `volta list`](#keep-volta-current-as-an-alias-for-volta-list)
    - [Do not add `list`](#do-not-add-list)
- [Unresolved questions](#unresolved-questions)

# Motivation
[motivation]: #motivation

Users of Volta currently have no way to get the answers to the following questions:

- What versions of a runtime, packager, or tool do I have available already on my machine?
- What versions of a runtime, packager, or tool are currently active? And why?
- What versions of a runtime, packager, or tool are the *defaults* for the system?
- What binaries are supplied by a given package which has been installed?
- What Node version and packager are used by a given tool binary?

The only tooling available to answer *any* of these questions are the `volta current` and `volta which` commands. `volta current` prints exactly and only the version of Node itself which will be executed in the user’s current directory. `volta which` answers the question for any single item in the user's toolchain, but only by parsing a path. Users must resort to poking through the `~/.volta` directory to answer these questions, and its internals are *not* public API. Thus, any tooling a user might build around the directory could break between versions.

Adding a `volta list` command addresses this need. Adding it with both human- and tool-friendly output formats makes for a better user-experience: the default output will make it easy for people to *read*, and the machine-readable formats will make it easy for people to build *tools* around.

# Pedagogy
[pedagogy]: #pedagogy

We introduce a `list` command, which lets users get a description of their Volta environment: the currently-active items in their toolchain, the reason those items are currently active, and the total set of available tools on their system.

- To get the currently-active runtime, packager, and available binaries, the user can run `volta list`.

- To get *all* runtimes, packagers, and binaries currently fetched to the user's system, the user can run `volta list all`.

- The user can also narrow the query:
    - to runtimes: `volta list node`
    - to packagers: `volta list yarn` (and, when custom installations of `npm` are supported, `volta list npm`)
    - to a specific package: `volta list <package name>`
    - to a specific tool: `volta list <tool name>`

These commands represent the Volta ideas of the user's *toolchain*, including *default* and *current* items within that chain.

These ideas already exist implicitly in Volta's vocabulary. Introducing `list` provides a specific place for users to see what the terms mean and how they are employed. Indeed, one of the primary benefits of the `list` command is that it provides a point where the Volta model—as distinct from e.g. the nvm or nodenv models—can be *made explicit* and thereby taught to users.

## Why `list`?
[why-list]: #why-list

Potential options include `list`, `ls`, `tools`, `toolchain`, `info`, `current`, or simply bare names like `node`, `yarn`, or `<package>` or `<tool>`. However, `list` is the most flexible, supplying variants to let the user describe *parts* of their toolchain as desired. It is also is a very common name for this operation in other tools (see the survey below). Additionally, choosing `list` as the primary name of the command does not preclude adding aliases for common operations later.

## Prior art
[prior-art]: #prior-art

A brief survey of the broader developer ecosystem indicates that `list` is by far the most common (sub)command used in CLI tools for listing installed versions of tools. The only major exception is `nodenv`, which (reasonably) seems to treat the "list" action as implicit in the user's intention. However, this is clearer for nodenv because it *only* manages specific versions of Node. `nvm` uses `ls` and `ls-remote`, which are standard Unix shortenings of "list."

Survey details:

- `nvm` (and the other `*vm` tools):
    - `ls` for installed versions
    - `ls-remote` for available versions

- `nodenv` (and the other `*env` tools) 
    - `versions`: list all installed versions
    - `version`: displays the currently-active version *and* how it was set
    - `local`: list/set the version specified for a given directory tree, if any
    - `global`: list/set the version specified for a global default, if any
    - `shell`: list/set the version specified for a given shell, if any

- [`dotnet`](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet?tabs=netcore21)
    - [`tool list`](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-tool-list), which requires either
        - `-g, --global` -> lists “user-wide Global tools”; mutually exclusive with `--tool-path` variant
        - `--tool-path <path>` -> a custom path to look for global tools; mutually exclusive with `--global` variant

- [Chocolatey](https://chocolatey.org):
    - `list`: searches for both local and remote packages, alias for `search`
        - `-l, --lo, --localonly, --local-only`: only items on the local machine
        - `-a, all, allversions, all-versions`: results from all versions
        - `--version=VALUE`: only this specific version match
        - ` -e, --exact`: only exact matches for the name
    - `info` for displaying details about a specific installed package

- [Homebrew](https://brew.sh):
    - `list`, `ls`
    - takes a variety of arguments to limit the output:
        - `--full-name`: give fully-qualified names
        - `--versions`: show version number, takes an optional list of packages
        - `--multiple`: only packages that have multiple versions installed
        - `--pinned`: only for pinned formulae (things brew won’t upgrade without forcing)
    - `info` for displaying 

- [Scoop](https://scoop.sh):
    - `list`

- apt:
    - `list`, lists `<source>/<package>,<package> <version> <arch>`
    - explicitly does not have a stable CLI

- yum:
    - `list`: List package names from repositories
        - `list available`: List all available packages
        - `list installed`: List all installed packages
        - `list all`: List installed and available packages
        - `list kernel`: List installed and available kernel packages
    - `info`: Display information about a package

# Details
[details]: #details

The `volta list` command supports four *variants*:

- **`volta list`:** comparable to the existing `volta current`, but expanded. Shows not only the user’s current runtime but also their current package manager and any available tool binaries, as well as explanation of why the current values are what they are.

- **`volta list all`:** shows every fetched runtime, package manager, and tool version along with the binaries available for each tool. Also indicates of whether each is a default or is set for a project in the user’s current working directory.

- **`volta list <package>`:** shows a subset of the output from `all`, scoped to the information for a specific package which has been fetched to the user’s inventory.

- **`volta list <tool>`:** shows a subset of the output from `all`, scoped to the information for a specific tool which has been fetched to the user’s inventory. `tool` is similar to `package`, but if a package has more than one tool associated with it, only the specified tool will be shown.

The command also supports (initially) two *output modes*: *human-friendly* (`human`) and *machine-friendly* (`plain`). It may later be expanded to support JSON or other machine-focused outputs.

## Information supplied by the command

The `volta list` command always prints the following information for a set of runtimes, package managers, and tools:

- name
- version
- whether it is the user's default toolchain or a project version
- for tools, the associated runtime and package manager versions

## Flags

Besides the subcommands, `volta list` accepts *context* flags and *output mode* flags.

### Context

The tool supports limiting the output to either the *current* or the *default* values for any given argument:

- `--current`: prints the *currently active* tool or set of tools (per the invoked command), along with their source
- `--default`: prints the user's *default* tool or set of tools (per the invoked command)

The two are *optional* and *mutually exclusive*.

### Output modes

The tool will support multiple modes (two initially), which include exactly the same information but presented in different human- or machine-friendly formats. All modes include the same information for runtimes, package managers, and tools: name, version, whether it is a user- or project-specified version, and (for tools) the Node version and package manager (i.e. platform).

- "human" mode, the default if the context is a user-facing terminal; also invokable with `--format=human` in any context. An indented listing of the user's current runtime, package manager (if specified), and any installed binaries. See the detailed sections below for examples of the format.

- "plain" mode, the default if the context is not a user-facing terminal (e.g. when piped into another command); also invokable with `--format=plain` in any context. A simple plain text format which prints a line per runtime, package manager, or tool, with space-separated output on each line.

    - runtimes:

        ```
        runtime node@<version>
        ```

    - package managers:

        ```
        packager-manager (yarn|npm)@<version>
        ```

    - tools: 

        ```
        tool <tool name> / <package name>@<package version> [node@<version>] [<yarn|npm>@<version>] [(default|current @ <project path>)]
        ```

This RFC does not propose, but allows for the possibility of, a JSON mode (`--format=json`) or similar at a later time if that proves desirable.

## Detailed command output

Here we supply a worked example of each *variant* in both *output styles*.

### Assumed Configuration

Throughout, we will assume the user has the following configuration for illustrative purposes:

- Node versions installed: v12.2.0, v11.9.0, v10.15.3 (default), v8.16.0
- Yarn versions installed: v1.16.0, v1.12.3 (default)
- Tools installed:
    - ember-cli, with binary `ember`:
	    - v3.10.0 on Node v12.2.0 with built-in npm (default)
	    - v3.8.2 on Node v12.2.0 with built-in npm
    - typescript, with binaries `tsc`, `tsserver`:
	    - v3.4.5 on Node v12.2.0 with built-in npm
	    - v3.0.3 on Node v12.2.0 with built-in npm (default)
    - create-react-app, with binary `create-react-app`: v3.0.1 on Node v12.2.0 with built-in npm (default)
    - yarn-deduplicate, with binary `yarn-deduplicate`: v1.1.1, fetched but not installed (and therefore with no associated platform)

They also have two projects with the following pins:

- `~/node-only/package.json`:

    ```json
    {
      "volta": {
        "node": "v8.16.0",
      }
    }
    ```

- `~/node-yarn-ember/package.json`:

    ```json
    {
      "devDependencies": {
        "ember-cli": "v3.8.2",
      },
      "volta": {
        "node": "v12.2.0",
        "yarn": "v1.16.0"
      }
    }
    ```

### `volta list` (no flags)

#### Human

The format is:

```sh
$ volta list --format=human
Currently active tools:

    Node: <version> (default|current @ <project path>)
    <Yarn|npm>: v1.12.3 (default|current @ <project path>)
    Package binaries available: [NONE]
        <tool name>[, <tool name>...] (default|current @ <project path>)

See options for more detailed reports by running `volta list --help`.
```

<details><summary>Outside a project</summary>

```sh
$ volta list --format=human
Currently active tools:

    Node: v8.16.0 (default)
    Yarn: v1.12.3 (default)
    Package binaries available:
        create-react-app, ember, tsc, tsserver (default)

See options for more detailed reports by running `volta list --help`.
```

</details>

<details><summary>In the `node-only` project</summary>

```sh
$ volta list --format=human
Currently active tools:

    Node: v8.16.0 (@ ~/node-only/package.json)
    Yarn: v1.12.3 (default)
    Package binaries available:
        create-react-app, ember, tsc, tsserver (default)

See options for more detailed reports by running `volta list --help`.
```

</details>

<details><summary>In the <code>node-yarn-ember</code> project</summary>

```sh
$ volta list --format=human
Currently active tools:

    Node runtime: v12.2.0 (@ ~/node-yarn-ember/package.json)
    Package manager: Yarn: v1.16.0 (@ ~/node-yarn-ember/package.json)
    Package binaries available:
        create-react-app, tsc, tsserver (default)
        ember (current @ ~/node-yarn-ember/package.json) 

See options for more detailed reports by running `volta list --help`.
```

</details>

#### Plain

The format is:

```sh
$ volta list --format=plain
runtime node@<version> (default|current @ <project path>)
package-manager <npm|yarn>@<version> (built-in|default|current @ <project path>)
```

<details><summary>Outside a project</summary>

```sh
$ volta list --format=plain
runtime node@v10.15.3 (default)
package-manager yarn@v1.12.3 (default)
```

</details>

<details><summary>In the `node-only` project</summary>

<b>Note:</b> this assumes the implementation of a fix for [volta-cli/volta#436](https://github.com/volta-cli/volta/issues/436).

```sh
$ volta list --format=plain
runtime node@v8.16.0 (~/node-only/package.json)
package-manager yarn@v1.12.3 (default)
```

</details>

<details><summary>In the <code>node-yarn-ember</code> project</summary>

```sh
$ volta list --format=plain
runtime node@v12.2.0 (~/node-yarn-ember/package.json)
package-manager yarn@v1.16.0 (~/node-yarn-ember/package.json)
```

</details>

### `volta list all`

#### Human

The basic format is:

```sh
$ volta list all --format=human
User toolchain:

    Node runtimes:
        <version> (default|current @ <project path>|fetched)

    Package managers:
        (Yarn|npm):
        <version> (default|current @ <project path>|fetched)

    Packages:
        <package name>
            [<version> (default | current @ <project path>)
                binaries: [<binary name>]...
                platform:
                    runtime: node@<version>
                    package manager: built-in npm|<npm|yarn>@<version>]
            [<version> (fetched)]
```

<details><summary>Outside a project directory</summary>

```sh
$ volta list all --format=human
User toolchain:

    Node runtimes:
        v10.15.3 (default)
        v8.16.0 (fetched)
        v11.9.0 (fetched)
        v12.2.0 (fetched)

    Package managers:
        Yarn:
            v1.16.0 (default)
            v1.12.3 (fetched)

    Packages:
        create-react-app:
            v3.0.1 (default)
                binaries: create-react-app
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
        ember-cli:
            v3.10.0 (default)
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
            v3.8.2 (fetched)
        typescript:
            v3.0.3 (default)
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
            v3.4.5 (fetched)
        yarn-deduplicate:
            v1.1.1 (fetched)
```

</details>

<details><summary>In the <code>node-only</code> project</summary>

```sh
$ volta list all --format=human
User toolchain:

    Node runtimes:
        v8.16.0 (current @ ~/node-only/package.json)
        v10.15.3 (default)
        v11.9.0 (fetched)
        v12.2.0 (fetched)

    Package managers:
        Yarn:
            v1.16.0 (default)
            v1.12.3 (fetched)

    Packages:
        create-react-app:
            v3.0.1 (default)
                binaries: create-react-app
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
        ember-cli:
            v3.10.0 (default)
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
            v3.8.2 (fetched)
        tsc:
            v3.0.3 (default)
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
            v3.4.5 (fetched)
        yarn-deduplicate:
            v1.1.1 (fetched)
```

</details>

<details><summary>In the <code>node-yarn-ember</code> project</summary>

```sh
$ volta list all --format=human
User toolchain:

    Node runtimes:
        v12.2.0 (current @ ~/node-yarn-ember/project.json)
        v10.15.3 (default)
        v8.16.0 (fetched)
        v11.9.0 (fetched)

    Package managers:
        Yarn:
            v1.16.0 (default)
            v1.12.3 (current @ ~/node-yarn-ember/project.json)

    Packages:
        create-react-app:
            v3.0.1 (default)
                binaries: create-react-app
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
        ember-cli:
            v3.10.0 (default)
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
            v3.8.2 (current @ ~/node-yarn-ember/project.json)
                binaries: ember
                platform:
                    runtime: node@v12.2.0 (@ ~/node-yarn-ember/project.json)
                    package manager: yarn@v1.12.3 (@ ~/node-yarn-ember/project.json)
        tsc:
            v3.0.3 (default)
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    package manager: built-in npm
            v3.4.5 (fetched)
        yarn-deduplicate:
            v1.1.1 (fetched)
```

</details>

#### Plain

The basic format is:

```sh
$ volta list all --format=plain
runtime node@<version> [(default|current @ <project path>)]
package-manager <packager>@<version> [(default|current @ <project path>)]
package <package>@<version> / [<tool>, ...] / <npm|yarn>@<built-in|version> (default|current @ <path>)
package <name>@<version> (fetched)
```

<details><summary>Outside a project directory</summary>

```sh
$ volta list all --format=plain
runtime node@v12.2.0
runtime node@v11.9.0
runtime node@v10.15.3 (default)
runtime node@v8.16.0
package-manager yarn@v1.16.0 (default)
package-manager yarn@v1.12.3
package create-react-app@v3.0.1 / create-react-app / node@v12.2.0 npm@built-in (default)
package ember-cli@v3.10.0 / ember / node@v12.2.0 npm@built-in (default)
package ember-cli@v3.8.2 (fetched)
package typescript@v3.4.5 (fetched)
package typescript@v3.0.3 / tsc, tsserver / node@v12.2.0 npm@built-in (default)
package yarn-deduplicate@v1.1.1 (fetched)
```

</details>

<details><summary>In the <code>node-only</code> project</summary>

```sh
$ volta list all --format=plain
runtime node@v12.2.0
runtime node@v11.9.0
runtime node@v10.15.3 (default)
runtime node@v8.16.0 (@ ~/node-only/package.json)
package-manager yarn@v1.16.0 (default)
package-manager yarn@v1.12.3
package create-react-app@v3.0.1 / create-react-app / node@v12.2.0 npm@built-in (default)
package ember-cli@v3.10.0 / ember / node@v12.2.0 npm@built-in (default)
package ember-cli@v3.8.2 (fetched)
package typescript@v3.4.5 (fetched)
package typescript@v3.0.3 / tsc, tsserver / node@v12.2.0 npm@built-in (default)
package yarn-deduplicate@v1.1.1 (fetched)
```

</details>

<details><summary>In the <code>node-yarn-ember</code> project</summary>

```sh
$ volta list all --format=plain
runtime node@v12.2.0 (@ ~/node-yarn-ember/project.json)
runtime node@v11.9.0
runtime node@v10.15.3 (default)
runtime node@v8.16.0
package-manager yarn@v1.16.0 (default)
package-manager yarn@v1.12.3 (@ ~/node-yarn-ember/project.json)
package create-react-app@v3.0.1 / create-react-app / node@v12.2.0 npm@built-in (default)
package ember-cli@v3.10.0 / ember / node@v12.2.0 npm@built-in (default)
package ember-cli@v3.8.2 / ember / node@v12.2.0 yarn@v1.12.3 (@ ~/node-yarn-ember/project.json)
package typescript@v3.4.5 (fetched)
package typescript@v3.0.3 / tsc, tsserver / node@v12.2.0 npm@built-in (default)
package yarn-deduplicate@v1.1.1 (fetched)
```

</details>

### `volta list <package>`

List all fetched versions of a specific package, along with its associated binaries.

#### Human

The basic format is:

```sh
volta list <package> --format=human

    [default: <version>|current: <version>(@ <project path>)
        binaries: [<binary name>]...
        platform:
            runtime: node@<version>
            package manager: built-in npm|<npm|yarn>@<version>]
    [fetched: [<version>, ...]]
```

For the TypeScript config specified in the canonical example:

```sh
volta list typescript --format=human
<package> versions in toolchain:

    default: v3.0.3
        binaries: tsc, tsserver
        platform:
            runtime: node@v12.2.0
            package manager: built-in npm

    fetched: v3.4.5
```

#### Plain

The basic format is:

```sh
volta list <package> --format=plain
package <package>@<version> / [<tool>, ...] / <npm|yarn>@<built-in|version> (default|current @ <path>)
package <package>@<version> (fetched)
```

For the TypeScript config specified in the canonical example:

```sh
volta list typescript --format=plain
package typescript@v3.0.3 / tsc, tsserver / node@12.2.0 npm@built-in (default)
package typescript@v3.4.5 (fetched)
```

### `volta list <tool>`

#### Human

The basic format is:

```sh
volta list <tool> --format=human
tool <tool> available from package:

    <package>@<version> (default|current @ <project path>)
        platform:
            runtime: node@<version>
            package manager: built-in npm|<npm|yarn>@<version>
```

For the TypeScript config specified in the canonical example:

```sh
volta list tsc --format=human
tool tsc available from:

    typescript@v3.0.3 (default)
        platform:
            runtime: node@v12.2.0
            package manager: built-in npm
```

#### Plain

The basic format is the same as the format for packages, but `fetched` can never appear (under the current implementation limitations):

```sh
volta list <tool> --format=plain
package <package>@<version> / [<tool>, ...] / <npm|yarn>@<built-in|version> (default|current @ <path>)
```

For the TypeScript config specified in the canonical example:

```sh
volta list tsc
package typescript@v3.0.3 / tsc, tsserver / node@12.2.0 npm@built-in (default)
```

## Deprecating `volta current`

Since `volta list` subsumes (and substantially extends upon) the functionality of `volta current`, `volta current` should be deprecated when `volta list` is implemented. The deprecation should include a warning that the command will be removed in a future version and information about how to use `volta list --current <tool>` as a replacement.

# Critique
[critique]: #critique

## Use another command name

As noted in [**Why `list`?**][why-list], options for this subcommand name include `list`, `ls`, `tools`, `toolchain`, `info`, and `current`. (For discussion of shorthands, see the next section.) Given the considerable [prior art][prior-art] using `list`, however, it seems the best option in terms of what users will expect.

If another option is preferred, e.g. `toolchain`, `list` could be a (hidden) alias for the actual subcommand.

## Use shorthands instead of `list`

One alternative direction suggested by nodenv's approach is to supply shorthands:

- `volta list <package>` → `volta package <package>`
- `volta list <tool>` → `volta tool <tool>`

However, in nodenv's case, those subcommands are also used for installation, which we are *not* doing under the current design of `install`. Moreover, the distinction drawn above is *not* presented in our installation process. That is, we do not actually allow users to install *tools* by name today, only *packages*. (This may suggest a simplification is in order; see [Unresolved Questions][unresolved] below.)

Additionally, we might consider here (and people have suggested in discussions in Discord) that we simply make the bare `volta` output the equivalent of `volta list`, with `volta --help` or `volta help` being required for the full help details. In this scenario, we would presumably also include a message to that effect for non-TTY, human-friendly contexts, so:

```sh
$ volta
Volta 0.5.2
Currently active tools:

    Node: v8.16.0 (default)
    Yarn: v1.12.3 (current @ ~/node-yarn-ember/package.json)
    Package binaries available:
        create-react-app, ember, tsc, tsserver

To install a tool in your toolchain, use `volta install`.
To pin your project's runtime or package manager, use `volta pin`.

See `volta help` for more options!
```

This would be a substantial change, and require considerably more work, but it might be a nice experience!

## Keep `volta current` as an alias for `volta list`

Rather than deprecating `volta current`, we could make it an alias for `volta list`. The primary downside to doing this is that we would want it to work for *only* the equivalent of *bare* `volta list`, and not support any subcommands. This might be mildly confusing to users, but that possibility of confusion should be balanced against the utility of meeting people's existing expectations.

## Do not add `list`

We could leave the current state of affairs as it is. However, users then have no insight into their toolchain: they must inspect the internals of `~/.volta`  to see what they actually have on their system, and in order to see what version of a tool is going to be used, they have must actually invoke it or run `volta which` and interpret the path.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should the bare command include:

    - all available package binaries?
    - a subset of package binaries once the number crosses a threshold, with instructions about how to see all?
    - no package binaries?

    And how should that be presented in `plain` vs. `human` mode? Should there be any differences between `plain` and `human` for the bare command (as in the current design)?

- How should the current version be identified? It is currently marked with `(current @ <path>)`. Should this be `(from <path>)` or some other design?

- Should the command accept flags to narrow the search instead of a positional argument, e.g. `volta list --node`, `volta list --yarn`, `volta list --package=typescript`, etc.?

- How should the built-in version of `npm` be displayed to the user? Options include:
    - associated with the runtime it ships with, e.g. `v12.2.0 (with npm v6.4.1)`
    - as a packager, but indicating its source as a built-in, e.g. `npm v6.4.1 (built-in from node@v12.2.0`
