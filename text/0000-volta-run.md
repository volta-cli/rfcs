- Feature Name: volta_run
- Start Date: 2019-09-14
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Add a `volta run` command, allowing users to execute tools with an override on the detected platform.

# Motivation
[motivation]: #motivation

Volta currently will automatically detect the platform (Node runtime and yarn version) based on the context of where you are in the file system. However, there are occasionally times when you want to run a command using a different version of Node or Yarn than what Volta detects, without changing the pinned or default versions. To support those use-cases, we should provide a command that allows users to execute a command with a platform specified on the command-line.

# Pedagogy
[pedagogy]: #pedagogy

We will need to document the existence of `volta run` and how the users can specify the versions of Node and Yarn when running a command. This will be provided on the command line help output, however it should also be explained in detail in the documentation. We should also document how the overall platform is resolved if the user specifies only one of Node or Yarn, so that the behavior is clear to users.

# Details
[details]: #details

The new command will have the structure:

```
volta run [--node <version>] [--yarn <version>] <command>
```

Specifically, the `--node` and `--yarn` settings will need to be specified _before_ the command to be executed, as everything after the command will be treated as arguments to the command itself. This will allow users to specify a full command, with any command-line flags they need, without having to append `--` to separate the `volta run` command from the others.

## Platform Resolution

If the user specifies only one (or none) of the platform settings when calling `volta run`, the remaining settings will fall back to the existing context-aware logic. So if the user is in a project with `node@10.16.0` and `yarn@1.17.3` pinned, and runs:

```
volta run --node 8 ember serve
```

We will resolve the a platform that has `node@8` and `yarn@1.17.3`, with the latter pulled from the existing pinned project and the former overridden by the command-line flag. This allows users to specify the parts of the platform that they want to change, while avoiding the necessity of specifying everything, even those things that don't change.

## 3rd-Party Binaries

After resolving the platform, if the command the user is executing is a 3rd party binary (not `node`, `npm`, `npx`, or `yarn`), then we will still do resolution of that binary relative to the current project (if any). That means if we are inside a project with that binary as a dependency, we will still use the project-local version of that tool.

## Verbose Output

In order to provide meaningful verbose output, we will need to update the `SourcedPlatformSpec` model to support platforms that come from the command line as well. This will likely require a refactor to support specifying the version for each tool explicitly, as opposed to having a single `Source` for the Platform as a whole.

# Critique
[critique]: #critique

## Command Name

There are alternative names we could use for the command, such as:

- `volta exec`
- `volta do`

However, `volta run` aligns with the "run script" command in both `npm` and `yarn`, so it seems to fit the best for this use-case.

## Platform Resolution

Instead of inheriting the platform from the current, we could treat it as purely defined on the command-line. This would give users total control over all of the pieces of the platform, including allowing them to remove `yarn`, even if it is specified in the project or default platforms.

However, it would also mean that if a user doesn't specify `--node`, that no commands will work, since all JS tools ultimately require `node` at some point. Additionally, users have to specify `yarn` if their command needs it, even if they only want to change the version of `node` being used.

# Unresolved questions
[unresolved]: #unresolved-questions

- Do we want to support removing `yarn` from the platform (e.g. `--yarn none`) in a way that won't be overridden by the platform inheritance?
- Do we want to support changing a package version as well, letting the user run a different version of a tool than they currently have installed / available?
