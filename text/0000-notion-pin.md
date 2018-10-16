- Feature Name: notion_pin
- Start Date: 2018-10-15
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

# Motivation
[motivation]: #motivation

Notion needs to have two separate and intuitively distinct commands for selecting tools for _user_ toolchains and _project_ toolchains, respectively. The command `notion install` is perfect for the former---it exactly conveys the right intuition using a universal concept across all computer systems: namely, software installation. The other command, however, is less universal and needs to communicate Notion's specific functionality of pinning the toolchain dependencies for a project.

The current command for this, `notion use`, is confusing both because the fact that it's tied to a project (as opposed to referring to the user toolchain) is not guessable, and because existing Node version managers have a `use` command whose usage pattern is closer to the user toolchain than a project toolchain.

This RFC proposes we rename that command to `notion pin`.

# Pedagogy
[pedagogy]: #pedagogy

Renaming this command should make it closer to the terminology used in even quick introductions to Notion. "Pinning" can be the shorthand name for the feature, and brings congruity to Notion's messaging, documentation, and workflow.

# Details
[details]: #details

The current command `notion use` would be changed to `notion pin`. All the documentation and CLI help pages would need to change to match.

Some examples:

```
$ notion pin node 10
$ notion pin yarn 1.10
$ notion pin npm latest
$ notion pin npm default
```

In turn, these mean:

- Pin the current project to the latest available version of Node matching major version 10.
- Pin the current project to the latest available version of Yarn matching version 1.10.x.
- Pin the current project to the latest available version.
- Pin the current project to the version that came by default with the pinned version of Node.

# Critique
[critique]: #critique

There are a number of command names we could choose from. Here's a selection of some:

- `notion use node 10`
- `notion set node 10`
- `notion save node 10`
- `notion add node 10`
- `notion choose node 10`
- `notion select node 10`
- `notion pick node 10`

While many options are defensible, the nice thing about "pin" is that it's a commonly-used term and it makes it more obvious that it's specific to the project toolchain, since you have to pin _to_ something.

# Unresolved questions
[unresolved]: #unresolved-questions

- Other alternative syntaxes we may have missed?
