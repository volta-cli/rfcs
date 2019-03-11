- Feature Name: config_hooks
- Start Date: 10/31/2018
- RFC PR: https://github.com/notion-cli/rfcs/pull/25
- Notion Issue: https://github.com/notion-cli/notion/pull/241

# Summary
[summary]: #summary

Rename `Config` to `Hooks` and allow additional methods for specifying tool resolution.

# Motivation
[motivation]: #motivation

The settings that are currently called `Configs` within Notion could more accurately be described as Infrastructure Hooks. They define settings that, as opposed to being individual user configurations, will most likely be defined across an entire organization:

- Alternate locations for tool indexes
- Alternate URLs for resolving specific tool versions
- Event handling for tracking performance metrics

To prevent confusion and naming conflicts with eventual user configurations, these settings should be renamed. Also, currently the only implemented settings are resolving tool versions using a shell command. It would be more useful if there were additional methods of customizing both the tool resolution and tool index, including but not limited to:

- Static URL Prefix
- URL Template with Predefined Variables
- (Existing) Shell Command

# Pedagogy
[pedagogy]: #pedagogy

We will need to define the allowed resolution methods and the structure of the relevant `.toml` file. We will also need to define the template format and the variables that are available for use in the templates. We should also have documentation of the API for the Shell Command, both the arguments that will be passed to it as well as the expected output.

# Details
[details]: #details

## Rename

This change is relatively straightforward, renaming from `Config` to `Hooks` internally, as well as in any documentation. It should also include switching the file from which these settings are loaded from `config.toml` to `hooks.toml`.

## URL Prefix Resolver

This option should store the user-defined URL prefix and then use the default behavior for determining the variable part of the URL in any resolution.

## URL Template

This option will store the user-defined template and then render it using the necessary variables to determine the resolved URL. Possible variables include:

- OS
- Architecture
- Tool Version
- Archive File Name

## Shell Command

This implementation already exists for `ResolverPlugin`. It will need to be implemented for the `LsRemote` plugin as well, in a similar manner.

# Critique
[critique]: #critique

## Rename

Even though there currently aren't any user-specific configuration options, it is better to make the change now when fleshing out the implementation. If we instead wait until we want to add configs to change the name, there will be more effort involved in the refactor.

## Resolver Options

There are other potential options to allow users to customize the resolution path (e.g. Dynamic Link Libraries), however these options should cover a majority of the use cases with a reasonable amount of implementation effort.

## Shell Command

There are some performance concerns with calling an external process to resolve a tool version or find the index, however those activities won't ever be on the performance-critical path. Whenever we are getting the tool index or resolving a specific version, we will then need to download, so the operation will always be I/O-bound and the small slowdown from using an external command won't be a bottleneck.

# Unresolved questions
[unresolved]: #unresolved-questions

- Will an organization need or want to manage the individual tool settings in separate files?
- Should we change the format of the Hooks file(s) to JSON, since the end-users are JS devs who will be more familiar with JSON than TOML?
