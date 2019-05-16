- Feature Name: per-project-config
- Start Date: 2019-05-01
- RFC PR: [#33](https://github.com/volta-cli/rfcs/pull/33)
- Volta Issue: [#411](https://github.com/volta-cli/volta/pull/411)

# Summary
[summary]: #summary

Support Volta configuration of hooks on a per-project basis using a `.volta/hooks.json` file in the project.

# Motivation
[motivation]: #motivation

We currently only support Volta configuration on a per-user basis in the `hooks.toml` file. This file allows users to change the URLs that tools are downloaded from. However, since a user could be working on both an open-source project (that uses the public registry) and an internal project (that uses a private registry), we instead want to allow the users to specify these settings on a per-project basis.

To allow for the potential of additional per-project Volta configurations in the future, we want to store this file in a `.volta` directory at the project root.

# Pedagogy
[pedagogy]: #pedagogy

We need to document the format of the `hooks.json` file, which will match the features available in the existing `hooks.toml`. We also need to document the existence of a `.volta` directory within a project for per-project configurations. Additionally, we need to document and explain the precedence order for when multiple configs are specified (e.g. user-level _and_ project-level).

Finally, for consistency of configuration, we should update the user-wide config stored in `~/.volta` from `hooks.toml` to also be named `hooks.json`, so that it's immediately obvious that it contains the same settings as the project-level configuration. This will not be a breaking change since the existing `hooks.toml` format wasn't documented and wasn't part of our public API.

This feature will primarily be used for more advanced use-cases, as the default public repositories and `package.json` settings should cover the majority of open-source projects.

# Details
[details]: #details

## User-level configuration file

We will need to update the `HookConfig` to parse the user-level `hooks.json`, if it exists, instead of the `hooks.toml` file.

## Configuration Precedence

When configurations are specified in multiple places, we need to deep merge them in a specified order so that we get a single configuration to work from. The order will be defined generally as more specific configurations having precedence over more general configurations. This means that per-project settings will beat per-user settings.

## Merging Configurations

When multiple configurations are specified, we should merge them deeply using the above precedence such that the final configuration includes all hooks defined by any of the configurations. For example, if:

- User-level `hooks.json` includes the `[node.index]` and `[node.distro]` hooks
- Project-level `.volta/hooks.json` includes the `[node.distro]` hook

The final configuration that Volta uses will have:

- `[node.index]` hook from user-level `hooks.json`
- `[node.distro]` hook from project-level `hooks.json`

## `volta config` Reference

To help users understand which configurations are active and where they are coming from, we should implement a `volta config list` command that works similarly to `npm config list`. This will allow users to immediately see what Volta understands about the state of the world from a given place in the filesystem, while also providing us with a useful tool for debugging the implementation. This also opens the option going forward of providing CLI commands for editing the configuration (e.g. `volta config set`).

# Critique
[critique]: #critique

## Configuration Precedence

One possible update is that a different precedence order could be used for the configurations. However, it is a common pattern in computing that more specific settings beat less specific settings, so we should still adhere to that. Additionally, the specific-to-general precedence order matches the order used by `npm` for `npmrc` files.

## Merging Configurations

Another approach to determining the final configuration from the available files would be to use _only_ the highest-precedence `hooks.json` if it is present, This would mean that if any `hooks.json` is present, it represents the single source of truth for all of the Volta hooks. However, it would remove the ability to have settings cascade down from the user-level settings to the project-level settings when a project doesn't have any custom configuration beyond what the user has specified.

The merge approach suggested in the RFC matches how `npm` resolves configuration from multiple sources, pulling each value in precedence order, but allowing different files to specify different settings.

## File Layout

An alternative to using a `.volta` directory with independent files would be to have a single `.voltarc` or `.voltarc.json` file in the root of the project. The trouble with this approach is that it would require any future project-level configuration we decide to add to also be in the same file. In a corporate setting, it's reasonable to expect that different groups will be responsible for managing different configurations (i.e. one group is in charge of changes to the hooks while another group manages the `node` platform settings). In this case, it would be preferable to have the files be separate, so to future-proof the design, we decided on having a configuration directory that can hold any number of independent files.

## File Format

Instead of using `.json`, we could use a format that supports comments for our configuration files, such as `.toml`. This would have the advantage of allowing the configuration to be more expressive, but would require a learning curve for a majority of our users, who don't necessarily have experience with `.toml`. By using `.json`, we're using a format that is a standard in the JS ecosystem, so the learning curve will be lowered. Additionally, there are proposals for extensions of JSON that support comments, so in the future we could adopt those in a backwards-compatible way, which would allow us to support comments without breaking existing users.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should we allow users to specify platform information (`node`, `npm`, and `yarn` versions) in the `.volta` directory? If so, what format should it take?
