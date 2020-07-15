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

The main concept we will need to document is the concept of bundled versions, and how they differ from explicit versions within the Volta model. The hope is that this is intuitive, however we should document the potential ambiguities so that users can easily discover how to get Volta to do what they want.

Beyond that concept, all of the other behavior is already provided in our implementation of Node and Yarn. The only change is that we will now support `volta install npm` and `volta pin npm` in addition to the other tools.

# Details
[details]: #details

## Bundled vs. Explicit Versions

One point of ambiguity when dealing with `npm` is that Node always comes bundled with a version of `npm`, so whenever a user upgrades their version of Node, they are also upgrading the bundled version of `npm`. This can lead to a situation where the user has picked a version of `npm` but is now upgrading to a Node version that has a newer version of `npm` available. To resolve this potential ambiguity, this RFC proposes being clear about the difference between a bundled version and an explicitly chosen version.

If the user has not specified (via `volta pin` or `volta install`) a version of `npm`, then we write nothing into the respective `json` files, and when we run `npm`, we run the bundled version. This means if the user hasn't specified `npm` and upgrades their version of Node, their `npm` install will also be upgraded. Currently, this is always the behavior of `npm`, so we preserve that for those users who don't want or need to specify a custom version.

If, however, the user has explicitly chosen a version of `npm`, then we write that into the appropriate `json` file and we always respect the user's choice. If the user upgrades to a version of Node with a newer version of `npm`, until they explicitly also upgrade to a newer version of `npm`, we maintain their previous setting. We should, however, show an informational message to the user, letting them know that the bundled version includes a newer `npm` and providing them with a command they can use to switch to the bundled version.

## Installation

Similar to Yarn, we can download the `npm` tarball directly from `https://npmjs.org/npm/-/npm-{{version}}.tgz`, which comes pre-packaged with all of the necessary dependencies already included. This means that we can treat npm exactly the same way we treat Yarn, unpacking it into its own directory. The only issue is that the executable scripts provided by the tarball have extra path-finding behaviors that don't work with Volta's model, so those will need to be overwritten (see below for details).

To support downloading from custom repositories, we will also need to support hooks for `npm`, allowing the user to redirect the `index`, `latest`, and `distro` checks to a different URL. This mirrors the implementation of Yarn within Volta.

Additionally, we should provide a way for the user to switch from an explicit version of `npm` (specified in the `package.json` file) to the bundled version (with no entry in the `package.json`). To support this, we can allow a custom "version" that represents the bundled `npm`, for example "bundled", meaning the user can switch to use the bundled version with `volta pin npm@bundled`. To help make this custom version more discoverable, the error message a user sees when they run `volta uninstall npm` should include a CTA to run `volta install npm@bundled` or `volta pin npm@bundled` if they want to switch to use the bundled version and stop using an explicit version.

## Running

We want to enable `npm` to execute by adding it to the PATH, in the same way that we support `yarn`. This allows us to support tools that subsequently call `npm` (e.g. `ember install`) as well as direct invocations of `npm`. Unfortunately, the executable scripts that the `npm` package provides have additional logic that is incompatible with Volta's model. Specifically, they check for the Node prefix and then invoke `npm` from there, meaning that they will always get the bundled version of `npm` (this works in the non-Volta case because `npm i -g npm` will install itself in that location, so the check finds the correct version). Instead, we will need to write our own executable scripts that call `node npm-cli.js` and `node npx-cli.js` for the two npm commands. These scripts can actually be relatively simple, since we are in control of the overall environment.

## Volta Directory Layout

We currently unpack Node in `tools/image/node/{node_version}/{bundled_npm_version}`. However, the final directory representing the bundled `npm` version isn't needed, since we will be keeping each version of `npm` separate from the `node` image directory anyway. For simplicity, we should remove the extra directory and install Node under `tools/image/node/{node_version}`. That will allow us to undo some of the coupling between Node and `npm` that currently exists in the code base, as well as more easily support arbitrary versions of `npm`.

Explicitly installed `npm` packages themselves should be unpacked under `tools/image/npm/{npm_version}`, in the same way that `yarn` is structured currently.

## Volta List

Within the output of the `volta list` command, `npm` should be listed along-side `yarn` as a "Package Manager". We should also show if the version is available because it is bundled or was explicitly chosen.

# Critique
[critique]: #critique

## Upgrading Node

Instead of defaulting to always keeping the selected version of `npm`, we could prompt the user to ask if they want to upgrade whenever the situation occurs. This, however, makes the `volta install` command harder to script, as well requiring user input at odd times (it's not uncommon for a user to start a longer-running process and step away for a moment). We also don't currently have any interactive prompts within Volta, so this would be new functionality that would need extensive testing to ensure it works correctly in all environments. Providing an informational message, on the other hand, allows us to let the user know there's a newer version available and give them the opportunity to upgrade if they like.

Alternatively, we could make the default to always take the bundled node if it's newer. The problem there is that we are then ignoring the explicit choice the user made to pin their version of `npm`. One design goal of Volta is to be as invisible and unobtrusive as possible; we would be moving away from that goal if we arbitrarily ignored the decisions the user made.

## Volta Uninstall

We already have a `volta uninstall` command that is used for 3rd-party packages. Instead of a custom version (`volta pin npm@bundled`), we could use that command to allow the user to switch to the bundled `npm`. The problem with this approach, however, is that it means the command doesn't do what it says: If the user runs `volta uninstall npm`, we will uninstall their custom version, but `npm` will still be available to them. So it wasn't really uninstalled, they just switched to a different version. Given that, it seems better to align with how users need to pick any other version (`volta pin` / `volta install`), even if it is slightly less discoverable.

# Unresolved questions
[unresolved]: #unresolved-questions
