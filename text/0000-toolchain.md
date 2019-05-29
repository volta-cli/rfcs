- Feature Name: toolchain
- Start Date: 2018-12-07
- RFC PR: 
- Volta Issue: 

# Summary
[summary]: #summary

This RFC describes a design for the Volta's central unifying design concept: the **toolchain**.

# Motivation
[motivation]: #motivation

The Volta toolchain is designed with several goals:

- **Reproducible project tooling.** With this design, Volta should make it convenient and reliable for all contributors on a project to get the same exact version of the Node runtime, the package manager, and any package binaries configured for use by `package.json`.
- **Reconciling the use of npm as a software distribution platform with best practices.** It's considered a best practice for project development to avoid ever installing global packages. But npm is a popular and convenient platform for distributing command-line tools. The toolchain design reconciles this tension by isolating _user tools_, which are installed for personal use, making them invisible to JS project scripts, and distinguishing them from _project tools_.
- **Install and forget.** This design solves the problem of _tool bitrot_, where a tool stops working because of Node upgrades. It also avoids the need to reinstall global tools every time a new Node version is provisioned. With Volta, once you install a user tool and get it working, it keeps working unless you deliberately decide to change or uninstall it.
- **Low cognitive overhead via statelessness.** This design allows Volta users to generally avoid thinking at all about the version of Node that's currently installed, instead relying on saved configuration to ensure that tools and projects have already declaratively specified their Node platform version.

## User stories

To demonstrate this, here are three representative user stories.

### Project tools: `tsc`

A project maintainer selects a version of TypeScript in `package.json`:

```js
  ...
  "dependencies": {
      "typescript": "^3.2"
  },
  "volta": {
      "node": "10.10.0"
  }
  ...
```

and the lockfile pins TypeScript to version 3.2.2.

Users can call the TypeScript compiler directly as long as they've installed it:
```
volta install typescript
```
Afterwards, running `tsc` from within the project directory runs version 3.2.2 of the TypeScript compiler, using Node 10.10.0 as the runtime.

### User tools: `surge`

An end user of the [surge.sh](https://surge.sh) service installs their CLI tool that is deployed as an npm package:
```
volta install surge
```
In this user story, the `surge` tool is published with `"engines": "11"` in its manifest, and at the time the command is run, the latest 11.x version of Node is 11.4.0. The CLI tool is installed in the user toolchain with Node 11.4.0 set as its default engine.

From this point on, unless the user changes their toolchain, running
```
surge
```
from the command-line always runs this tool using Node 11.4.0.

### User tools in projects: `pexx`

The [project-explorer](https://sdras.github.io/project-explorer-site/) tool is a useful CLI tool that lets you inspect JS projects, but is not itself a tool that a project would use in its scripts.

A user can install this tool to their toolchain:
```
volta install project-explorer
```
In this user story, the `project-explorer` manifest is published with `"engines": "8"` in its manifest. At the time the tool is installed, the latest 8.x version of Node is 8.15.0. So the `pexx` tool is installed with Node 8.15.0 as its default engine.

When the user runs
```
pexx myproject
```
the `pexx` binary runs with Node 8.15.0—even if `myproject` specifies a different version of Node in its `"volta"` spec.

# Pedagogy
[pedagogy]: #pedagogy

This section lists the set of concepts that users may encounter using Volta. The first two, **tools** and the **toolchain**, are central to using Volta. The latter two, **shims** and **engines**, are lower-level primitives that may be helpful for more implementation-oriented users.

## Tools

There are three types of tools:

- **Node runtime:** The version of Node itself, which in particular dictates which `node` binary gets invoked.
- **Package manager:** A version of npm or a version of Yarn. (In the future we may want to add support for other package managers such as pnpm or tink.)
- **Package binary:** An executable published and distributed as part of an npm package.

## Toolchain

The toolchain is a set of tools the user has installed for use at the command-line console. The user adds to their toolchain with `volta install` and removes with `volta uninstall`.

Each tool installed in the toolchain has a default version and engine, which can be overridden when the tool is invoked in a project that has a dependency on that tool.

## Primitive: Shims

Volta shims intercept calls to tools and redirect execution to the right executable based on the environment and current directory.

Users may already be familiar with shims, since these are commonly used by other version managers.

## Primitive: Engines

An **engine spec** is a complete description of a version of the Node platform:

- An exact version of the Node runtime.
- An optional exact version of npm.
- An optional exact version of Yarn.

An **engine image**, or **engine**, is an immutable instantiation of an engine spec on disk. (The name "image" is meant as an analogy to a container image, but for a snapshot of the Node platform as opposed to a snapshot of an operating system.)

Users may already be familiar with the "engine" concept because of the `"engines"` key of `package.json`.

# Details
[details]: #details

## Engine specs

In the manifest, an omitted npm version defaults to the version bundled with the specified Node runtime. An omitted Yarn version defaults to Yarn being unavailable, meaning that invoking `yarn` will produce an error. Both of these can be explicitly set to `null`, which means the specified tool is unavailable, i.e., the shim fails with an error when executed.

## Engine images

An engine image is an installation on disk of a specific version of the Node runtime, a specific version of npm (or none), and a specific version of Yarn (or none).

The _intention_ of Volta is for an image never to be modified. In particular, all commands that modify the state of an engine—e.g. `npm install --global` or `yarn global add`—should fail with an error when invoked through Volta shims. This behavior can be disabled with the environment variable `VOLTA_UNSAFE_GLOBAL`. The name is meant to indicate to the user that they are "voiding their warranty" and responsible for any violations of the expectation of immutability.

## Pinning a project

The `"volta"` section of `package.json` selects the engine image associated with a package.

Users can pin the engine by manually editing the `package.json` `"volta"` section or via [`volta pin`](https://github.com/volta-cli/rfcs/pull/24).

When the `node` shim or a package manager shim is executed from within a pinned project, the shim delegates to the version of that tool from the project's pinned engine.

## Default engine

Every package binary in the toolchain has a **default engine** associated with it.

When a package binary is executed outside of a Node project, or from a Node project that _does not_ have that package as a direct dependency, the package binary is run using its default engine.

When a package binary is executed from a Node project that _does_ have the package as a direct dependency but _does not_ have a pinned engine, the user's installed engine is used. If the user does not have an installed engine, the shim fails with an error indicating that no engine was selected.

## Installation

The `volta install` command installs a tool to the user's toolchain.

### Installing an engine

The `volta install node` subcommand installs the user's engine. When installing a version of the Node runtime, Volta also installs the default version of npm bundled with that verison of Node. This can be overridden with a subsequent `volta install npm` command.

### Installing a package binary

When installing a package binary, Volta checks the package for the standard [`"engines"`](https://docs.npmjs.com/files/package.json#engines) key to select the latest engine version compatible with the package and pins the tool's default engine to that engine version. If there is no `"engines"` key in the package manifest, it defaults to the user's current engine. If the user has no current engine, `volta install` fails with an error message suggesting the user choose at least a Node runtime version.

### Overriding the associated engine

When used for a package tool, `volta install` accepts an optional `--node` parameter for overriding the package's specified platform version:

```
volta install surge --node=latest
```

## Uninstallation

The `volta uninstall` command uninstalls a tool from the user's toolchain.

## Updating

Taken together, `volta uninstall` and `volta install` can be used to update a tool. Note that as the tool evolves over time, its authors may update its platform to newer Node versions. So as users upgrade their tools they will automatically get platform upgrades for the tool. But they also continue to be assured they are getting a version of the platform that the tool was tested with.

The `volta update` command is a shorthand command for doing the same thing:
```
volta update surge
```

# Critique
[critique]: #critique

It's natural to question whether pinning for reproducibility is worth the cost of extra fetching. An alternative approach would be to allow projects to specify less precise version requirements (such as the ranges expressed under the `"engines"` field of `package.json`) and assume most differences will be benign. However, behavioral divergences between versions of Node do happen and are tricky bugs to nail down. Putting in extra work up front to ensure that these divergences cannot happen, by construction should pay dividends when scaled across the Node ecosystem. And over time, we can investigate optimization techniques to save time and disk space for fetching multiple similar versions.

Theoretically, it might make more sense to put the `"volta"` section in a lockfile. But since there isn't a standardized single lockfile format for JS, and those formats aren't extensible, and we don't want to impose a whole new file to add to JS projects, using the package manifest seemed like the least imposition on users.

Another reasonable criticism is that pinning the Node version for tools in the user toolchain means that users will not automatically benefit from performance and security improvements in Node. There are a couple of reasons this is outweighed by the benefits of pinning. First, as described above, updating tools will typically get platform updates. Second, users can still override the default with the `--node` parameter.

# Unresolved questions
[unresolved]: #unresolved-questions

- What about the (rarer) cases of user tools that want to work with the current project’s toolchain choice instead of a statically-bound choice? For example, tools that wrap the Node REPL. Maybe a special keyword for the platform like `"node": "user"`? We'd need to think through the semantics.
- Should we consider some kind of update notification mechanism to inform you when your user tools are out of date? Probably at least something similar to `brew outdated` so users can explicitly ask for the information.
- What kinds of performance optimizations can we offer to make stateless platform images super fast?
- What about scenarios like CI and testing matrices, where users want to be able to specify different platform configurations without having to change the configuration file? Perhaps workflows similar to `ember try`?
- `update` vs `upgrade` syntax 😬
