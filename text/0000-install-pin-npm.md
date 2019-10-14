- Feature Name: install_pin_npm
- Start Date: 2019-10-14
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Support installation and pinning of arbitrary `npm` versions, decoupled from the selected Node version.

# Motivation
[motivation]: #motivation

For package managers, we currently only support using an arbitrary version of `yarn`, but not `npm`. For `npm`, users are forced to use the version that is automatically bundled with Node, meaning they can't upgrade or downgrade without changing the Node version. We should support `npm` to the same level that we support `yarn` and give users full control over the version of their package manager.

# Pedagogy
[pedagogy]: #pedagogy

There shouldn't be anything new we need to document / teach users, as the behavior already exists for Node and Yarn. The only change is that we will support the `volta install npm` and `volta pin npm` commands in the same way that we support `volta install yarn` and `volta pin yarn`.

# Details
[details]: #details

## Installation

Unlike Yarn, `npm` doesn't provide a pre-packaged tarball for download. Instead, the rely on using the existing `npm` installation that is provided with Node to do any upgrades. This is very similar to how we handle global 3rd-party packages, so we should be able to leverage a similar approach for installing `npm`. That does mean, however, that in the same way we require Node to be available to install packages, we will also require Node to be available to install a custom version of `npm`.

It appears we can also skip the `npm install` step, as the `npm` package tarball comes with all of the dependencies already included.

## Running

Currently, to execute `npm`, we add the Node bin directory to the PATH and then call `npm`, relying on the fact that `npm` is bundled with Node, so `npm` is available within the Node bin directory. To support arbitrary versions of `npm`, we will need to take the approach that is used for packages and call the `npm-cli.js` file directly. Since this isn't an arbitrary package, but will always be `npm`, we can likely shortcut the configuration files and detection of binaries that we need for packages and use the known path to the file.

For Windows compatibility, we will need to call `node /path/to/npm-cli.js` instead of executing the `.js` file directly, since Windows doesn't understand shebang loaders.

## Volta Directory Layout

We currently unpack Node in `tools/image/node/{node_version}/{bundled_npm_version}`. However, the final directory representing the bundled `npm` version isn't needed, since we will be keeping each version of `npm` separate from the `node` image directory anyway. For simplicity, we should remove the extra directory and install Node under `tools/image/node/{node_version}`. That will allow us to undo some of the coupling between Node and `npm` that currently exists in the code base, as well as more easily support arbitrary versions of `npm`.

The `npm` packages themselves should be unpacked under `tools/image/npm/{npm_version}`, in the same way that `yarn` is structured currently.

## Upgrading Node

One tricky point with supporting `npm` is that it is always bundled with Node. This means that there is some potential ambiguity when a user upgrades their version of Node: If the version of `npm` that comes with that Node is newer than the one they have pinned, do they want to upgrade to use the bundled version, or stay with the pinned version? There isn't a clear default choice that makes sense, so instead in that situation we can ask the user:

```bash
$ volta pin node@latest
...
node@12.12.0 includes a newer version of npm (6.11.3) than you currently have pinned in this project (6.9.0)!
Would you like to upgrade to npm@6.11.3 (Y/n)?
```

If the user chooses to upgrade to the bundled version, we can remove the custom setting and allow the project to use the default behavior of the bundled `npm`. If they choose not to upgrade, we leave the custom setting as-is.

We should also include a `-y` option for `volta pin` and `volta install`, so that if users want to always accept the default answer to the question, they can run the commands non-interactively.

# Critique
[critique]: #critique

## Upgrading Node

Other options for the conflict when upgrading Node would be to choose a default behavior (always upgrade or always keep the existing setting) and provide a CLI flag to override that behavior. The difficulty with that approach is that the setting of `npm` on a user's machine or within a project is very much an individual decision. There isn't a clear default that it would make sense for us to use, so we would be doing something unexpected in a large percentage of cases. Given the lack of a clear default, it makes sense to ask the user directly what their intentions are, so that we always do the right thing.

# Unresolved questions
[unresolved]: #unresolved-questions
