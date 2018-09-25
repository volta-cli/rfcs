- Feature Name: project_toolchain_shims
- Start Date: 2018-09-19
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

Automatic management of project toolchain shims will be handled by the `yarn` and `npm` shims, which will scan the `package.json` files of the project and its immediate dependencies to determine which shims to create.

# Motivation
[motivation]: #motivation

Automatic management of shims is a core competency of Notion, in that its abstraction of the Node ecosystem workflow is incomplete without it.  Automatic management of user toolchain shims is provided by the `notion install` command, but no process for automatically managing project toolchain shims has yet been described.

# Pedagogy
[pedagogy]: #pedagogy

If this feature works as intended, there is little to teach.  The only significant divergence from typical JS workflows is that `./node_modules/.bin/` can be omitted when calling tools in the project toolchain:

```
$ ./node_modules/.bin/mocha  # The non-Notion way
$ mocha  # The Notion way
```

# Details
[details]: #details

Just prior to a successful completion inside a project, the `yarn` and `npm` shims will "scan" for dependency binaries and attempt to create shims for them.  A "successful" completion, in this sense, means that the underlying tool was successfully run and returned a successful (0) status.  This covers all potential scenarios in which either of those tools create new dependency binaries.

This scan will be a two-level process.  First, the project's `package.json` will be inspected for all dependencies and devDependencies.  Second, each of those dependencies' `package.json` files will be inspected for binaries to shim.  For each binary, Notion will attempt to create a shim symlink, catching `EEXISTS` and considering it a success.  Any other error will result in an appropriate message to `stderr`, but not halt the scanning/shimming process.

This same scan will also occur when pinning a version of Node using `notion use`, to ensure that shims are created for packages installed prior to setting up a toolchain for the project.  Finally, the scan will be available via a new `notion autoshim .` command.

Once the scan has completed, Notion will exit with a successful status, even if shims were unable to be created, as per [RFC 0020](0020-exit-codes.md).

Due to potential future changes (see [Critique](#critique) below), it is important that scanning be implemented in a modular fashion, such that a full scan, single dependency scan, or single shim creation can be kicked off independently.

# Critique
[critique]: #critique

## "Full scan" vs. "smart scan"

This RFC proposes that all shims be checked every time `yarn` or `npm` is run, but alternatives exist.  In particular, the amount of scanning being done could be significantly reduced by parsing the Yarn/npm command line arguments and only checking the shims actually being installed during `yarn add <module(s)>` or `npm install <module(s)>`.

However, it will always be necessary to implement a "full scan" scenario for the initial install of modules — i.e. `yarn install` and `npm install`.  And while a "smart scan" would increase speed, it is also potentially brittle in multiple ways.  It would require Notion to both effectively duplicate those tools' command line parsers and maintain an understanding of their command structures.  It would also fail to fill "holes" in Notion's shim coverage — once a shim has been missed, by any means, it will not be created until a full scan is done.

## Sync vs. async

Another alternative would be to run the scanning and shim creation process in the background, allowing an earlier return from the `yarn` and `npm` shims.  Asynchrony here would make Notion feel more responsive, but introduce a race condition where the shims are not available immediately after the command completes.  The two deciding factors in favor of synchronous shim creation are:

1. If shim creation is fast enough to complete between when the `yarn`/`npm` shim completes and when the user immediately thereafter tries to use one of the created shims, it *should* also be fast enough to run synchronously without affecting the "speed feel".
2. A race condition introduces the possibility for command chaining to fail:

```
$ yarn install && mocha
-bash: mocha: command not found
$ mocha
  example
    ✓ runs a test


  1 passing (11ms)
```

# Unresolved questions
[unresolved]: #unresolved-questions

- When, if ever, should shims be deleted?  The ability of a user to simply delete an entire project from disk makes tracking shim use problematic, at best.  What, exactly, is the behavior of a shim when the associated tool is not part of the toolchain?  If it simply "forwards" to the shell, this is less of an issue.
