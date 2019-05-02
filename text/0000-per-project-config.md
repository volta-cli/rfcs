- Feature Name: per-project-config
- Start Date: 2019-05-01
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

Support Notion configuration of both hooks and platform information on a per-project basis using a `.notionrc` file in the project root.

# Motivation
[motivation]: #motivation

We currently only support Notion configuration on a per-user basis in the `hooks.toml` file. This file allows users to change the URLs that tools are downloaded from. However, since a user could be working on both an open-source project (that uses the public registry) and an internal project (that uses a private registry), we instead want to allow the users to specify these settings on a per-project basis.

Additionally, we currently already support per-project settings for the platform (node and yarn versions), specified in the `toolchain` key inside of `package.json`. Multiple users have requested an alternate method in a separate file, for various reasons (see https://github.com/notion-cli/notion/issues/282 for some discussion of one use-case). As we are already looking at adding per-project hooks to a separate file, that file should also support specifying the platform information.

# Pedagogy
[pedagogy]: #pedagogy

We need to document the format of the `.notionrc` file, which will mostly match the format of `hooks.toml`, however it will also include the additional settings for `node` and `yarn` versions. Additionally, we need to document and explain the precedence order for when multiple configs are specified (e.g. user-level `hooks.toml` _and_ project-level `.notionrc`).

This feature will primarily be used for more advanced use-cases, as the default public repositories and `package.json` settings should cover the majority of open-source projects.

# Details
[details]: #details

## Configuration Precedence

When configurations are specified in multiple places, we need to merge them in a specified order so that we get a single configuration to work from. The order will be defined generally as more specific configurations having precedence over more general configurations. This means:

- Per-project configuration beats per-user configuration
- `.notionrc` platform spec beats `package.json` platform spec

## Merging Configurations

When multiple configurations are specified, we should merge them using the above precedence such that the final configuration includes all hooks defined by any of the configurations. For example, if:

- `hooks.toml` includes the `[node.index]` and `[node.distro]` hooks
- `.notionrc` includes the `[node.distro]` hook and the `node` version setting for the platform
- `package.json` includes both `node` and `yarn` for the platform

The final configuration that Notion uses will have:

- `[node.index]` hook from `hooks.toml`
- `yarn` version from `package.json`
- `[node.distro]` hook and `node` version from `.notionrc`

This will likely require that the `Project` and `Hooks` objects in `notion-core` are merged in some way into a single configuration object that pulls from all possible sources.

# Critique
[critique]: #critique

## Configuration Precedence

One possible update is that a different precedence order could be used for the configurations. However, it is a common pattern in computing that more specific settings beat less specific settings, so we should still adhere to that. As discussed above, the `.notionrc` setting is considered to be a more "advanced" way of specifying per-project settings, so it should have precedence over `package.json`.

## Merging Configurations

Another approach to determining the final configuration from the available files would be to use _only_ `.notionrc` if it is present and fall back to the existing combination of `package.json` and `hooks.toml` if there isn't a `.notionrc` present. This would mean that if `.notionrc` is present, it represents the single source of truth for all of the Notion settings. However, it would remove the ability to have settings cascade down from the user-level settings to the project-level settings when a project doesn't have any custom configuration beyond what the user has specified.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should we also include additional settings in `.notionrc`?
- Is there a better file name to use, other than `.notionrc`? `notion.toml` Perhaps?
- Is TOML the best format, or should we use something else for this per-project configuration?
