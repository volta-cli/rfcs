- Feature Name: per-project-config
- Start Date: 2019-05-01
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

Support Notion configuration of both hooks and platform information on a per-project basis using a `.notionrc.json` file in the project root.

# Motivation
[motivation]: #motivation

We currently only support Notion configuration on a per-user basis in the `hooks.toml` file. This file allows users to change the URLs that tools are downloaded from. However, since a user could be working on both an open-source project (that uses the public registry) and an internal project (that uses a private registry), we instead want to allow the users to specify these settings on a per-project basis.

Additionally, we currently already support per-project settings for the platform (node, npm, and yarn versions), specified in the `toolchain` key inside of `package.json`. Multiple users have requested an alternate method in a separate file, for various reasons (see https://github.com/notion-cli/notion/issues/282 for some discussion of one use-case). As we are already looking at adding per-project hooks to a separate file, that file should also support specifying the platform information.

# Pedagogy
[pedagogy]: #pedagogy

We need to document the format of the `.notionrc.json` file, which will mostly match the features available in the existing `hooks.toml`, however it will also include the additional settings for `toolchain` similar to . Additionally, we need to document and explain the precedence order for when multiple configs are specified (e.g. user-level _and_ project-level).

Additionally, for consistency of configuration, we should update the user-wide config stored in `~/.notion` from `hooks.toml` to also be named `.notionrc.json`, so that it's immediately obvious that it contains the same settings as the project-level configuration. This will be a breaking change that requires users to translate their settings from the old format to the new.

This feature will primarily be used for more advanced use-cases, as the default public repositories and `package.json` settings should cover the majority of open-source projects.

# Details
[details]: #details

## User-level configuration file

We will need to update the `HookConfig` to parse the user-level `.notionrc.json`, if it exists. We should also allow the user to specify the user platform (`node`, `npm`, and `yarn`) in that file. To that end, we can likely merge the existing `~/.notion/tools/user/platform.json` into `~/.notion/.notionrc.json` as our source of truth for the user-default selected versions. This may also allow us to simplify some of the logic around determining the currently active platform, as we can merge all of the available configs into a single result once, and then use that to execute a tool.

## Configuration Precedence

When configurations are specified in multiple places, we need to deep merge them in a specified order so that we get a single configuration to work from. The order will be defined generally as more specific configurations having precedence over more general configurations. This means:

- Per-project configuration beats per-user configuration
- `.notionrc.json` platform spec beats `package.json` platform spec

## Merging Configurations

When multiple configurations are specified, we should merge them deeply using the above precedence such that the final configuration includes all hooks defined by any of the configurations. For example, if:

- User-level `.notionrc.json` includes the `[node.index]` and `[node.distro]` hooks
- Project-level `.notionrc.json` includes the `[node.distro]` hook and the `node` version setting for the platform
- `package.json` includes both `node` and `yarn` for the platform

The final configuration that Notion uses will have:

- `[node.index]` hook from user-level `.notionrc.json`
- `yarn` version from `package.json`
- `[node.distro]` hook and `node` version from project-level `.notionrc.json`

This will likely require that the `Project` and `Hooks` objects in `notion-core` are merged in some way into a single configuration object that pulls from all possible sources.

## `notion config` Reference

To help users understand which configurations are active and where they are coming from, we should implement a `notion config list` command that works similarly to `npm config list`. This will allow users to immediately see what Notion understands about the state of the world from a given place in the filesystem, while also providing us with a useful tool for debugging the implementation. This also opens the option going forward of providing CLI commands for editing the configuration (e.g. `notion config set`).

## Multi-stage Implementation

To facilitate the work and allow users to work with parts of these changes before we make an entire overhaul to the system, we can implement this RFC in multiple stages:

1. Add support for parsing a project-local `notion-hooks.toml` file, if it exists, as well as the `~/.notion/hooks.toml` that we currently check for.
2. Rename `hooks.toml` into `.notionrc.json`, changing the file format but supporting the same set of configurations (i.e. Only hooks, no platform specifications at this point).
3. Fully merge the hooks config with the platform config internally and add support for `toolchain` within `.notionrc.json`

# Critique
[critique]: #critique

## Configuration Precedence

One possible update is that a different precedence order could be used for the configurations. However, it is a common pattern in computing that more specific settings beat less specific settings, so we should still adhere to that. Additionally, the specific-to-general precedence order matches the order used by `npm` for `npmrc` files.

As discussed above, the project-level `.notionrc.json` setting is considered to be a more "advanced" way of specifying per-project settings, so it should have precedence over `package.json`.

## Merging Configurations

Another approach to determining the final configuration from the available files would be to use _only_ the highest-precedence `.notionrc.json` if it is present and fall back to the existing `package.json` if there isn't a `.notionrc.json` present. This would mean that if any `.notionrc.json` is present, it represents the single source of truth for all of the Notion settings. However, it would remove the ability to have settings cascade down from the user-level settings to the project-level settings when a project doesn't have any custom configuration beyond what the user has specified.

The merge approach suggested in the RFC matches how `npm` resolves configuration from multiple sources, pulling each value in precedence order, but allowing different files to specify different settings.

# Unresolved questions
[unresolved]: #unresolved-questions

- What additional settings, if any, should we include in `.notionrc.json`?
