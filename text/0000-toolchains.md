- Feature Name: toolchains
- Start Date: 2018-12-07
- RFC PR: 
- Notion Issue: 

# Summary
[summary]: #summary

This RFC describes a design for **toolchains**, the central unifying concept of Notion.

# Motivation
[motivation]: #motivation

Notion toolchains attempt to achieve several goals:

- **Reproducible project tooling.** With this design, Notion should make it convenient and reliable for all contributors on a project to get the same exact version of the Node runtime, the package manager, and any package binaries configured for use by `package.json`.
- **Reconciling the use of npm as a software distribution platform with best practices.** It's considered a best practice for project development to avoid ever installing global packages. But npm is a popular and convenient platform for distributing command-line tools. This design reconciles the tension by isolating _user tools_, which are installed for personal use, making them invisible to JS project scripts, and distinguishing them from project tools.
- **Install and forget.** This design solves the problem of _tool bitrot_, where a tool stops working because of upgrades to the system Node version. It also avoids the need to reinstall global tools every time a new Node version is provisioned. With Notion, once you install a user tool and get it working, it keeps working unless you deliberately decide to change or uninstall it.
- **Lightweight cognitive model via statelessness.** This design allows Notion users to generally avoid thinking at all about the version of Node that's currently installed, instead relying on saved configuration to ensure that tools and projects have already declaratively specified their Node platform version.

## User stories

To demonstrate this, here are three representative user stories.

### Project tools: `tsc`

A project maintainer selects a version of TypeScript in `package.json`:

```js
  ...
  "dependencies": {
      "typescript": "^3.2"
  },
  "platform": {
      "node": "10.10.0"
  }
  ...
```

and the lockfile pins TypeScript to version 3.2.2.

Any Notion user who checks out the project gets an identical development experience when working within the project directory tree. Namely, typing
```
tsc
```
runs version 3.2.2 of the TypeScript compiler, using Node 10.10.0 as the runtime.

### User tools: `surge`

An end user of the [surge.sh](https://surge.sh) service installs their CLI tool that is deployed as an npm package:
```
notion install surge
```
Assuming the `surge` tool selects Node 11.4.0 in its `"platform"` manifest (it's OK if not; more details below), the CLI tool is installed in the user toolchain and pinned to Node 11.4.0.

From this point on, unless the user changes their toolchain, running
```
surge
```
from the command-line always runs this tool using Node 11.4.0.

### User tools in projects: `pexx`

The [project-explorer](https://sdras.github.io/project-explorer-site/) tool is a useful CLI tool that lets you inspect JS projects, but is not itself a tool that a project would use in its scripts.

A user can install this tool to their toolchain:
```
notion install project-explorer
```
Assuming the `project-explorer` manifest selects Node 8.12.0 in its manifest, the tool is installed pinned to that version of the Node runtime.

When the user runs
```
pexx myproject
```
the `pexx` binary runs with Node 8.12.0---even if `myproject` specifies a different version of Node in its `"platform"` spec.

# Pedagogy
[pedagogy]: #pedagogy

This section lists the set of concepts that users may encounter using Notion. The first two, **tools** and **toolchains**, are central to using Notion. The latter two, **shims** and **platforms**, are lower-level primitives that may be helpful for more implementation-oriented users.

## Tools

There are three types of tools:

- **Node runtime:** The version of Node itself, which in particular dictates which `node` binary gets invoked.
- **Package manager:** A version of npm or a version of Yarn. (In the future we may want to add support for other package managers such as pnpm or tink.)
- **Package binary:** An executable published and distributed as part of an npm package.

## Toolchains

There are two distinct toolchains managed by Notion:

- **User toolchain:** This is the set of tools installed for general-purpose, personal CLI use at the console. They are not associated with any JS project. They are available for personal use when running commands manually or in shell scripts, but not when running package scripts in a JavaScript project.
- **Project toolchain:** This is the set of tools a project has pinned via its manifest and lockfile. These always take precedence over user tools, and are the only tools available in the `PATH` when running package scripts in the project.

## Primitive: Shims

Notion shims intercept calls to tools and redirect execution to the right executable based on the environment and current directory.

Users may already be familiar with shims, since these are commonly used by other version managers.

## Primitive: Platforms

A **platform spec** is a complete description of a version of the Node platform:

- An exact version of the Node runtime.
- An optional exact version of npm.
- An optional exact version of Yarn.

A **platform image** is an immutable instantiation of a platform spec on disk. It can be thought of analogously to a container image, but for the Node platform as opposed to an operating system.

# Details
[details]: #details

## Toolchains

A toolchain is a pair of a platform image and a set of pinned package binaries.

A pinned package binary is a pair of an exact version of a package binary and a platform image.

## Platform specs

In the manifest, an omitted npm version defaults to the version bundled with the specified Node runtime. An omitted Yarn version defaults to Yarn being unavailable, meaning that invoking `yarn` will produce an error. Both of these can be explicitly set to `null`, which means the specified tool is unavailable.

## Platform images

The definition of a platform image is a pair consisting of a Node version and a Yarn version.

A Node version is a pair consisting of an exact Node runtime version, and either an exact npm version or `null`.

A Yarn version is either an exact Yarn version or `null`.

## Pinning

The `"platform"` section of `package.json` selects the platform image associated with a package. This is a change from earlier versions of Notion, which used the key `"toolchain"`, since the toolchain consists of _both_ the platform image _and_ the pinned package binaries (see above).

Users can pin the platform by manually editing the `package.json` `"platform"` section or via [`notion pin`](https://github.com/notion-cli/rfcs/pull/24).

# Critique
[critique]: #critique

It's natural to question whether pinning for reproducibility is worth the cost of extra fetching. An alternative approach would be to allow projects to specify less precise version requirements (such as the ranges typically expressed in the `"engines"` field of `package.json`) and assume most differences will be benign. However, behavioral divergences between versions of Node do happen and are tricky bugs to nail down. Putting in extra work up front to ensure that these divergences cannot happen, by construction should pay dividends when scaled across the Node ecosystem. And over time, we can investigate optimization techniques to save time and disk space for fetching multiple similar versions.

Theoretically, it might make more sense to put the `"platform"` section in a lockfile. But since there isn't a standardized single lockfile format for JS, and those formats aren't extensible, and we don't want to impose a whole new file to add to JS projects, using the package manifest seemed like the least imposition on users.

Another reasonable criticism is that pinning the Node version for tools in the user toolchain means that users will not automatically benefit from performance and security improvements in Node. There are a couple of reasons this is outweighed by the benefits of pinning. First, as users upgrade the tool version itself, they will automatically get re-pinned to the newer version of Node specified by newer versions of the tool's `package.json`. Second, we should at least allow users the option to override the tool's specified Node version. But by letting the tool choose its platform version by default, users have a stronger guarantee that the tool will work and continue to work consistently based on how it was tested.

# Unresolved questions
[unresolved]: #unresolved-questions

- What about the (rarer) cases of user tools that want to work with the current project’s toolchain choice instead of a statically-bound choice? For example, tools that wrap the Node REPL.
- Should we consider some kind of update notification mechanism to inform you when your user tools are out of date?
- What kinds of performance optimizations can we offer to make stateless platform images super fast?
