- Feature Name: volta_use
- Start Date: 2024-02-22
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary

[summary]: #summary

Updating `volta pin` by adding a `--default` flag to support pinning the _default_ version of a tool (instead of the local project version). Combining the behavior of `volta pin` and `volta install` into a single `pin` command that is more intuitive and better matches the mental model of Volta.

# Motivation

[motivation]: #motivation

Early on in Volta's history, we picked the verb `install` to represent choosing the default version of tools (i.e. the version used when outside of a pinned project). This was chosen, in part, to align with users' expectations from other similar tools. However, due to Volta's auto-download behavior when it detects a version that isn't currently cached, `install` centers the wrong action. The more important part of `volta install` is the selecting a default version.

This has led to some confusion for users. We get questions fairly regularly asking how users should select an "active" version of Node, once they have installed them using `volta install`. Additionally, by centering the install action, rather than the selection, `volta install` doesn't help users internalize Volta's mental model of "select a version and Volta will automatically download and make sure it's available."

Using the existing `pin` verb for this action should make it more clear how Volta is intended to work.

# Pedagogy

[pedagogy]: #pedagogy

This will be a fairly large change to our recommended workflows. As `volta pin` will work for selecting both local _and_ default versions, we will need to update the documentation throughout to take advantage of the new flag. We will also likely want to write a blog post for the release which includes this change, explaining the reasoning and the new workflows.

Notably, we will _not_ be removing the existing `volta install` command right away. They will continue to work as they already do. In the future, we might consider adding notifications to them to point users to the updated command, but for the initial implementation we don't want to disrupt current users' workflows.

All that said, the new behavior should simplify things a lot, especially for new users. It will allow us to explain Volta by saying that you pick a version with `volta pin`, and don't worry about installing it: Volta will handle making it available. It also should do a better job of living up to the claim on our website by truly having a _single_ command that users can run to get set up.

# Details

[details]: #details

## New flag

A new flag, `-d` or `--default`, added to the `volta pin` command that will cause Volta to pin the _default_ version of a tool, rather than the _project_ version.

## Behavior

### Without flag

The behavior of `volta pin` without the flag will remain unchanged: we will resolve the version and attempt to pin it to the local project. If there is no project (i.e. we can't locate a `package.json` file), then we will continue to show an error rather than contextually switching to the `--default` behavior. The error message should be updated to include a suggestion of using the `--default` flag, however.

### With flag

When called with `--default`, `volta pin` should work identically to the current behavior of `volta install`: resolve the version and write the default version into the `VOLTA_HOME` directory. We should also keep the contextual notifications that we show the user, letting them know if a local project has a different version.

### 3rd-party packages

One notable difference between the current `volta install` behavior and the new `volta pin` is that `pin` will not support installing 3rd-party packages (e.g. `ember-cli`). The current behavior of `volta pin` does not support those, and keeping the consistency with that behavior should more clearly separate pinning tool versions and installing global shims.

# Critique

[critique]: #critique

## Using `--global` rather than `--default`

A different possibility for the flag itself would be to use `--global`, which aligns well with existing tools (e.g. `npm install --global`). This may be more intuitive for users who already know what `--global` typically means. However, we have intentionally used `default` rather than `global` in all of our messaging within Volta, to avoid the negative connotations associated with "global" installs in the existing ecosystem.

## Changing both `pin` and `install` to a different combined verb

An earlier version of this RFC proposed combining `volta pin` and `volta install` into a single new command: `volta use`. That proposal had the behavior of the new command contextual based on whether the user was in a project or not. In the discussion, we felt that behavior was too surprising and resulted in a bad user experience around edge cases.

Combining the commands into `volta pin` with an explicit flag means that the behavior is clear and expected, while still taking advantage of the fact that `pin` is a verb that fits better into Volta's model.

# Unresolved questions

[unresolved]: #unresolved-questions
