- Feature Name: informational_commands
- Start Date: 2019-05-15
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

Add an informational command, `volta list`, replacing the `volta current` command and expanding on its capabilities. The `list` command will allow users to see their default and project-specific toolchains, and will support both pretty-printed user-facing output and at least one mode of tool-friendly output.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Pedagogy
[pedagogy]: #pedagogy

How should we explain and teach this feature to users? How should users understand this feature? This generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how users should _think_ about the feature, and how it should impact the way they use Notion. It should explain the impact as concretely as possible.
- If applicable, describe the differences between teaching this to existing Node programmers and new Node programmers.

It is not necessary to write the actual feature documentation in this section, but it should establish the concepts on which the documentation would be built.

# Details
[details]: #details

## Why `list`?

### Prior art

A brief survey of the broader developer ecosystem indicates that `list` is by far the most common (sub)command used in CLI tools for listing installed versions of tools. The only major exception is `nodenv`, which (reasonably) seems to treat the "list" action as implicit in the user's intention, given that nodenv serves *only* to manage specific versions of Node. `nvm` uses `ls` and `ls-remote`, which are standard Unix shortenings of "list."

The survey details:

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
        - `-a, --all, --allversions, --all-versions`: results from all versions
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

## Deprecating `volta current`

Since `volta list` subsumes (and substantially extends upon) the functionality of `volta current`, `volta current` should be deprecated when `volta list` is implemented. The deprecation should include a warning that the command will be removed in a future version and information about how to use `volta list --current <tool>` as a replacement.

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

This section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
-->

# Critique
[critique]: #critique

This section discusses the tradeoffs considered in this proposal. This must include an honest accounting of the drawbacks of the proposal, as well as list out all the alternatives that were considered and rationale for why this proposal was chosen over the alternatives. This section should address questions such as:

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
