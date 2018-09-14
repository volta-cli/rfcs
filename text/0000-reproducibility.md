- Feature Name: reproducibility
- Start Date: 2018-09-14
- RFC PR: 
- Notion Issue: 

# Summary
[summary]: #summary

This RFC proposes a toolchain management model for Notion that’s built around a primary goal of *reproducibility*: that is, giving users end-to-end control around toolchains that, once set up, always exhibit identical behavior. The resulting user experience should be **”install and forget”** — after setting up a toolchain, a user or team never has to worry about the many forms of *tool bitrot*—hidden state changes (e.g. the addition or removal of global packages), subtle differences in tool versions—on an existing machine, between project collaborators’ machines, or when setting up a new machine. In short, reproducibility means Notion’s declarative configuration completely determines a Node environment’s behavior.

# Motivation
[motivation]: #motivation

One important motivation of this RFC is to reconcile two competing constraints in the Node ecosystem:

- Global package binaries (via `npm i -g` or `yarn global add`) are a convenient and popular deployment model for tools—not only project build tools like `babel` and `tsc`, for which `npx` is a popular solution, but also for use cases that aren’t associated with a pre-existing project, such as `surge`, `ember new`, or `svgo`.
- Global package binaries make projects sensitive to the global state of a user’s machine, making project builds brittle, unportable, and unreliable. This is such a pervasive problem that many developers recommend against the use of global installations entirely.

A separate but related motivation is the desire to be able to install user tools on a local machine one and not worry about “drift” over time—that is, a developer should be able to install, say, `surge` or `svgo` and not worry that those tools might stop working because of changes to the currently-installed version of Node.


## Scenarios

It’s helpful to think about a few different kinds of scenarios that should support, and how reproducibility fits in.

### Project tools: `tsc`

Using tools from a project’s `"dependencies"`  or `"devDependencies"` should be as simple as calling the tool directly by name, and Notion should run that tool with the version of Node selected by the project.

For example, in a project using TypeScript, the `package.json` might look like this:

```js
    ...
    "dependencies": {
      "typescript": "^3.0"
    },
    "toolchain": {
      "node": "10.10.0"
    }
    ...
```

If the lockfile pins `"typescript"` at version 3.0.3, then entering the project and running:

```
tsc
```

runs version 3.0.3 of the TypeScript compiler using Node 10.10.0.

*How this supports reproducibility:* The project collaborators have selected their toolchain via their project’s `package.json`, including the version of Node and the set of tools required to build, test, and run the project. The exact versions of each tool can be determined completely from `package.json` and the lockfile.

### User tools: `surge`

Some tools have nothing specifically to do with JavaScript development and are simply deployed as npm packages because it’s a convenient deployment mechanism. For example, [surge.sh](https://surge.sh) deploys their workflow tool as an npm CLI app. Installing an app like that should be as simple as typing:

```
notion install surge
```

If the `surge` tool specifies a `"toolchain"` in its manifest, then the installed shim should be pinned to its specified version of Node. Whenever the user runs

```
surge
```

on their machine, it executes the tool with that same version of Node.

*How this supports reproducibility:* The developers of the `surge` tool are best positioned to know what versions of the Node platform it needs to run properly, so not only does specifying the `"toolchain"` section aid them for development, it also allows users of the tool to run `surge` in the right environment. This way, the tool never bitrots on a user’s machine or runs into subtle issues because of changing around Node versions.

### User tools in projects: `project-explorer`

Sometimes, a programmer may want to use a tool on a project that isn’t actually part of that project’s chosen set of tools. For example, the [project-explorer](https://sdras.github.io/project-explorer-site/) tool lets you print out visualizations of project trees. That tool requires Node 8 to run, but a developer might want to visualize a project that’s using Node 6. If that tool specifies `"node": "8.12.0"` in its `"toolchain"` manifest, then when the user installs it with

```
notion install project-explorer
```

the tool is pinned to Node 8.12.0. So then when the user goes into `myproject` and runs

```
pexx myproject
```

the `pexx` binary runs with Node 8.12.0 even though `myproject` normally uses Node 6 for its project tools.

*How this supports reproducibility:* Since project-explorer is not one of the project tools, its installation is managed by the end user. It also means the project hasn’t aligned its toolchain with project-explorer—in particular, `myproject` is built with Node 6 whereas project-explorer is built to depend on Node 8 (for example, if it depends on [syntax that wasn’t supported until Node 8](https://node.green/#ES2017-features-trailing-commas-in-function-syntax)). By ensuring that the tool runs with the version of Node it was built for, the tool continues to work predictably in any directory, even a project built with a different version of Node.

# Pedagogy
[pedagogy]: #pedagogy
The user’s mental model rests on two key concepts: the *tool* and the *toolchain*.

## Tools

For the purposes of this RFC, the only tools we’re discussing are:

- **Node.js**
- **npm**
- **Yarn**
- Any **package binary**, i.e., a CLI tool distributed through an npm package and exposed as a command via the `"bin"` section of that package’s manifest.

(In a later section we’ll briefly discuss the possibility of opening this up to other kinds of tools not distributed via npm through a lower-level extension API, but that’s mostly left to future RFCs.)

## Toolchains

This RFC distinguishes between two types of toolchains:

- The **user toolchain**, which represents the set of tools the user has installed on their system for personal use. These are installed via `notion install` instead of `npm i -g` or `yarn global add`—more on this below.
- A **project toolchain**, which represents the set of tools a JS project has pinned in the `"toolchain"` section of its manifest.

## Advanced: Node platform images

The key mechanism for reproducibility is an encapsulated state of the Node platform, called a **platform image**. A platform image encapsulates the following three pieces of information:

- A precise (major.minor.patch) version of Node.js.
- A precise (major.minor.patch) version of `npm`, or an indication that `npm` should be disabled.
- A precise (major.minor.patch) version of Yarn, or an indication that Yarn should be disabled.

As a matter of specification, these version specifiers are sufficient to fully describe the state of a clean installation of the Node platform. In operational terms, a user can think of a platform image as behaving as if someone had just downloaded a clean copy of the platform and installed it from scratch. In practice, this can be heavily optimized (discussed more below).

As an aside: while Yarn is of course not a part of the official Node platform, the goal of Notion is to make the real user experience of installing Node toolchains declarative and predictable. Yarn is a common part of that experience for many Node developers, so supporting that use case is important. In time, Yarn support may eventiually be able to be expressed via an extensibility mechanism, i.e., as a plug-in that ships with Notion by default. From a usability standpoint, that wouldn’t make much difference, but it would be evidence of a well-factored design.

Most users wouldn’t need to understand this concept, but advanced users and plugin authors would likely benefit from having this terminology.

# Details
[details]: #details

## Pinning

Lockfiles advanced the world of software engineering by introducing a mechanism for achieving *reproducible dependency graphs*. With this RFC, Notion adds the “last mile” of reproducibility by also providing *reproducible toolchains*.

So if the requirements of a Node package are not only its dependency graph but also its toolchain, we can talk about *pinning* those requirements, meaning to determine:

- The precise Node platform to use, and
- The precise versions of any package dependencies of the tool.

Pinning can be accomplished with two pieces of data:

- The `"toolchain"` section of the manifest, which uniquely determines the platform image, and
- The lockfile, which uniquely determines the package dependencies of the tool.

For user toolchains, the package associated with an installed tool provides defaults via its lockfile and `"toolchain"` section (if present), but the user should also be able to override these defaults and specify a preferred version of the requirements for a tool in their user toolchain. These selections should be stored in a user configuration file and consulted when executing tools in the user’s toolchain.

## Reproducibility

To actually accomplish the fully reproducible behavior, executing a tool with its pinned platform image should behave the same as if the particular Node platform image had just been installed from scratch.

Ignoring performance, the user could imagine this is being implemented by copying a freshly installed copy of the platform image into a directory to add to the PATH just in time every time a tool is invoked. Of course, this is not how it will actually be implemented; see below for real implementation strategies.

## Fetching

Invoking a tool may involve fetching the required Node or Yarn distribution over the network. Some use cases require the ability to ensure that fetching has already happened, so that subsequent invocations can be guaranteed not to hit the network. For situations like working on an airplane or running builds in a sandboxed CI system, it should be possible to ensure the pre-fetching of platform images via e.g.:

```
notion fetch node 10.10.0
```

Reproducibility and precise pinning mean that as users install different tools or work with different projects, they need many different versions of Node fetched onto their machine. A naïve strategy would cause a lot of downloading and unpacking. For this reason, this RFC proposes experimenting with fetching compressed bundles of many versions, in order to make both switching between versions of Node and fetching new versions over the nework efficient. The next section discusses implementation strategies to investigate, with some related performance hypotheses based on some initial experiments.

## Implementation strategies

Using git as a storage representation for multiple Node distributions can result in very good levels of compression, fast cloning of snapshots of a distribution directory, and subtly but importantly, delta-compressed incremental updates when new versions get released.

### Platform packs

In the [Node release process](https://github.com/nodejs/Release), there are generally two most recent even-numbered major versions of Node that are the most commonly used (corresponding to either one *Active LTS* and one *Current*, or two Active LTS, depending on the time of year), and one most recent odd-numbered major version. All of these versions are actively supported and may acquire new point-releases at any time.

Based on some initial experiments, after running `git gc`, a repository containing all versions (for one platform, e.g., darwin-x64) of a single major version of Node compresses to roughly 50MB; a repo consisting of all versions of 2 major versions of Node compresses to roughly 100 - 150MB; and a repo consisting of all versions of 3 major versions compresses to roughly 200MB.

This suggests that during the Notion installation process, we could fetch a current **platform pack** with all of the two most recent even-numbered and one most recent odd-numbered versions of Node in a compressed git directory. (Less commonly-needed versions of Node could be provided in separate repositories, perhaps one major-version per repository.) This would take a bit of up-front setup time, but as a result, users would get the following benefits:


- Switching between projects would **rarely require fetching** anything new, since despite differences in minor or patch version, most projects will use one of the most recent versions of Node. This means saving on 15 - 20MB of download for every Node version.
- The process of preparing an already-fetched Node distribution would mean **cloning a git snapshot instead of decompressing an archive**, which should provide noticeable improvements. This could save many seconds of setup time, perhaps as much as a minute depending on the compression format (Unixes use tarball; Windows distributions use Zip) and speed of the developer’s machine.
- New minor and patch releases of the distribution pack’s versions would be fetched via `git fetch`, which uses **delta compression for fast incremental updates**. This means that even when a Node version isn’t on the current machine, the update might be on the order of 1 - 5MB instead of 15 - 20MB.

### Rollovers

Because these optimizations only make sense relative to the Node release calendar, the current platform pack would need to roll over to a new repo on a regular schedule. Based on the Node release schedule, **a plausible cadence is twice a year**. For example, the schedule for the next couple years would look something like:

| Spring 2018 | 8.x, 9.x, 10.x   |
| ----------- | ---------------- |
| Autumn 2018 | 8.x, 10.x, 11.x  |
| Spring 2019 | 10.x, 11.x, 12.x |
| Autumn 2019 | 10.x, 12.x, 13.x |

There should be commands for managing the state of these local repositories, including a command for triggering a rollover. We may also want to trigger a background rollover automatically with some commands, to ensure that the user’s machine does not get stale. The details of how rollovers work will require more investigation, and will require careful handling of concurrency.

## Other considerations

This section discusses a few extra details.

### Offline mode
Some environments, especially in CI, want to be able to ensure that no operations managed by Notion will hit the network. It should be possible to offer `--offline` flags or environment variables that guarantee never to touch the network (and error out if they need to). This also means for every operation that might sometimes need to hit the network, there needs to be a paired command that can be run up front to pre-fetch everything that’s needed to perform the operation offline.

### Limited connectivity mode
We may eventually want to offer a mode that users can opt into to use smaller platform packs, for situations where limited connectivity makes it hard to download the default packs. This should not be the default, and the UX should not *encourage* this mode, because it’s too tempting to avoid up-front downloads and then end up with less compression, more aggregate downloads and disk usage, and generally slower behavior. Also, users might be confused into thinking that the initial download is wasteful and spread misinformation about disk usage. Generally, the bet is that a multi-version git repository will lead to lower disk usage, so we should not encourage the use of more minimal packs. But as long as the UX is framed in terms of the use case of *limited connectivity* (for example, `install.sh --limited-connectivity`), this should help avoid misconceptions.

Moreover, it should still be possible to upgrade a limited-connectivity install to a normal install, if the user finds themselves in better network conditions later on, without having to reinstall Notion from scratch.

This particular feature is lower priority than the core functionality of this RFC and can be explored later.

### Extensibility

We will need a model for overriding the public git repositories, so that companies and CI systems can host their own Node distributions.

# Critique
[critique]: #critique

It’s natural to question whether this implementation strategy is worth the effort: an alternative approach would be to allow projects to specify less precise version requirements (such as the ranges typically expressed in the `"engines"` field of `package.json`) and assume most differences will be benign. However, behavioral divergences between versions of Node do happen and are some of the trickiest bugs to nail down. Putting in extra work up front to ensure that these divergences *cannot happen, by construction* will eventually pay for itself when scaled across the Node ecosystem.

Another reasonable criticism is that pinning the Node version for tools in the user toolchain means that users will not automatically benefit from performance and security improvements in Node. There are a couple of reasons this is outweighed by the benefits of pinning. First, as users upgrade the tool version itself, they will automatically get re-pinned to the newer version of Node specified by newer versions of the tool’s `package.json`. Second, we should at least allow users the option to override the tool’s specified Node version. But by letting the tool choose its platform version by default, user’s have a stronger guarantee that the tool will work and continue to work consistently based on how it was tested.

# Unresolved questions
[unresolved]: #unresolved-questions

- There is a lot of experimentation to do with the git representation before we have proved the implementation strategy out entirely.
- What should the exact policies be around number of versions per platform pack and frequency of rollovers?
- How should we structure the user’s configuration files to match the terminology of this RFC?
- What about the (rarer) cases of user tools that want to work with the current project’s toolchain choice instead of a statically-bound choice? For example, tools that wrap the Node REPL.
- Do we need to tweak the performance model for CI environments? Presumably, a CI system should be able to fetch git repositories from the local intranet, but will this be fast enough? We’ll need to learn from experimentation.

# Appendix
[appendix]: #appendix

## The Node.js release schedule

|                |        |             | **Fall ’15** | 4  | Active      |
| -------------- | ------ | ----------- | ------------ | -- | ----------- |
|                |        |             |              | 5  |             |
|                |        |             |              |    |             |
| **Spring ’16** | 4      | Active      | **Fall ’16** | 4  | Active      |
|                | 5      |             |              | 6  | Active      |
|                | 6      |             |              | 7  |             |
|                |        |             |              |    |             |
| **Spring ’17** | 4      | Maintenance | **Fall ’17** | 4  | Maintenance |
|                | 6      | Active      |              | 6  | Active      |
|                | 7      |             |              | 8  | Active      |
|                | 8      |             |              | 9  |             |
|                |        |             |              |    |             |
| **Spring ’18** | 4      |             | **Fall ’18** | 6  | Maintenance |
|                | 6      | Maintenance |              | 8  | Active      |
|                | 8      | Active      |              | 10 | Active      |
|                | ~~9~~  |             |              | 11 |             |
|                | 10     |             |              |    |             |
|                |        |             |              |    |             |
| **Spring ’19** | ~~6~~  |             | **Fall ’19** | 8  | Maintenance |
|                | 8      | Maintenance |              | 10 | Active      |
|                | 10     | Active      |              | 12 | Active      |
|                | ~~11~~ |             |              | 13 |             |
|                | 12     |             |              |    |             |
|                |        |             |              |    |             |
| **Spring ’20** | ~~8~~  |             | **Fall ’20** | 10 | Maintenance |
|                | 10     | Maintenance |              | 12 | Active      |
|                | 12     | Active      |              | 14 | Active      |
|                | ~~13~~ |             |              | 15 |             |
|                | 14     |             |              |    |             |
