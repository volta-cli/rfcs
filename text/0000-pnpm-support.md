- Feature Name: pnpm-support
- Start Date: 2021-07-26
- RFC PR:
- Volta Issue:

# Summary
[summary]: #summary

Add first-class support for the [`pnpm`](https://pnpm.io) package manager.

This will support everything we currently support for `npm` and `yarn`: pinning in `package.json`, auto-download, seamless version switching, global install interception, etc.
This should be a feature-add, not a breaking change (we will not change `package.json` in an incompatible way).


# Motivation
[motivation]: #motivation

pnpm is a popular Node package manager. At the time of this RFC, it has [11.9k stars on github](https://github.com/pnpm/pnpm), compared to [4.7k for npm](https://github.com/npm/cli) and [39.9k for yarn](https://github.com/yarnpkg/yarn).

There are outstanding issues on the [pnpm project](https://github.com/pnpm/pnpm/issues/3146) and in [Volta](https://github.com/volta-cli/volta/issues/737) requesting that Volta support pnpm, and at least a few people have inquired about pnpm support on our Discord channel.

Given that it's popular, and users want support for it, adding support for pnpm should further increase adoption of Volta, and additionally provide an example of how to onboard any new package managers we wish to support in the future.


# Pedagogy
[pedagogy]: #pedagogy

We already support `yarn` as a 3rd-party package manager, so things should work the same way for `pnpm` (we shouldn't require any new terminology or concepts):

- global installs (capturing commands and reworking them to install to our location)
- cache expiry
- hooks
- autodownload
- fetch / install / pin

Using `volta run` should work the same way, with the addition of `--pnpm` and `--no-pnpm` options.

## Documentation and Examples

We will need to add examples and documentation on the website - this feature is not done until that happens.


# Details
[details]: #details

## Fetch, Install, Pin

We will need to duplicate the existing Yarn functionality (in `crates/volta-core/src/tool/yarn/`) for pnpm (into `crates/volta-core/src/tool/pnpm/`). This will likely be copy-paste with some modifications, and may be a chance for us to refactor some of this duplication that also exists for Npm.

## Settings

We will need to add support for reading and writing `pnpm` settings, mainly in the `platform`, `project`, and `toolchain` modules in `crates/volta-core/src/`. This should mostly involve adding the `pnpm` key alongside `npm` and `yarn`.

This will modify the `package.json` format to add an additional optional `pnpm` field:

```
{
  "volta": {
    "node": 1.2.3,
    "npm": 2.3.4,
    "yarn": 3.4.5,
    "pnpm": 4.5.6
  }
}
```

And we will need to add `--pnpm` and `--no-pnpm` to the options for the `volta run` command (in `src/command/run`).

## Shims

pnpm includes two executables, `pnpm` and `pnpx`. These will need to be supported in `crates/volta-core/src/run/`, the same way that we support `yarn` and `npx`. There is probably some opportunity for refactoring duplicated code in this section as well.

We will also need to intercept these global install commands:

* `pnpm add --global <package>`
* `pnpm add -g <package>`

The same way we intercept `yarn` and `npm` global commands, in `crates/volta-core/src/run/parser`.

## Installing Local Packages

Package installation with pnpm works differently than Npm and Yarn. Instead of storing all files local to the project, it hard-links the files in `node_modules` to a shared content-addressable store. This is typically at `$HOME/.pnpm-store/`. Since this is a versioned store that links to individual files, based on their contents, this should be safe to leave as-is, even if there are multiple different versions of pnpm used on the same machine at the same time.

## Installing Global Packages

Currently, pnpm installs global packages based on the `PATH` (see [the discussion in this issue](https://github.com/pnpm/pnpm/issues/1360)). The first directory that has `node`, `npm`, or some other node-related binaries is chosen, or failing that, the first directory that is writeable.

```
$ node --version
v14.16.1

$ pnpm add --global cowsay
Packages: +33
+++++++++++++++++++++++++++++++++
Progress: resolved 33, reused 33, downloaded 0, added 0, done

/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5:
+ cowsay 1.5.0
```

In that case, the `cowsay` package has been installed under `pnpm-global/`, which is a peer directory to the `bin/` directory that appears in the `PATH`:

```
$ ll ~/.volta/tools/image/node/14.16.1/bin/
total 161328
drwxr-xr-x   7 mikrostew  staff       224 Jul 26 15:52 ./
drwxr-xr-x  10 mikrostew  staff       320 Jul 26 15:52 ../
-rwxr-xr-x   1 mikrostew  staff      2194 Jul 26 15:52 cowsay*
-rwxr-xr-x   1 mikrostew  staff      2194 Jul 26 15:52 cowthink*
-rwxr-xr-x   1 mikrostew  staff  73883024 Apr  6 11:22 node*
lrwxr-xr-x   1 mikrostew  staff        38 May  5 23:01 npm@ -> ../lib/node_modules/npm/bin/npm-cli.js
lrwxr-xr-x   1 mikrostew  staff        38 May  5 23:01 npx@ -> ../lib/node_modules/npm/bin/npx-cli.js
```

The executable `cowsay` in the `bin/` directory is a shell script that sets up the `NODE_PATH`, and tries to execute the `cowsay` file using the `node` that it was installed with.

```
$ cat cowsay
#!/bin/sh
basedir=$(dirname "$(echo "$0" | sed -e 's,\\,/,g')")

case `uname` in
    *CYGWIN*) basedir=`cygpath -w "$basedir"`;;
esac

if [ -z "$NODE_PATH" ]; then
  export NODE_PATH="/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules/.pnpm/cowsay@1.5.0/node_modules/cowsay/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules/.pnpm/cowsay@1.5.0/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules/.pnpm/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/node_modules:/Users/mikrostew/.volta/tools/image/node/node_modules:/Users/mikrostew/.volta/tools/image/node_modules:/Users/mikrostew/.volta/tools/node_modules:/Users/mikrostew/.volta/node_modules:/Users/mikrostew/node_modules:/Users/node_modules:/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules/cowsay/node_modules"
else
  export NODE_PATH="$NODE_PATH:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules/.pnpm/cowsay@1.5.0/node_modules/cowsay/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules/.pnpm/cowsay@1.5.0/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules/.pnpm/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/node_modules:/Users/mikrostew/.volta/tools/image/node/node_modules:/Users/mikrostew/.volta/tools/image/node_modules:/Users/mikrostew/.volta/tools/node_modules:/Users/mikrostew/.volta/node_modules:/Users/mikrostew/node_modules:/Users/node_modules:/node_modules:/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5/node_modules/cowsay/node_modules"
fi
if [ -x "$basedir/node" ]; then
  exec "$basedir/node"  "$basedir/../pnpm-global/5/node_modules/cowsay/cli.js" "$@"
else
  exec node  "$basedir/../pnpm-global/5/node_modules/cowsay/cli.js" "$@"
fi
```

We need to change where these global packages are installed (to `$HOME/.volta/tools/image/packages/` in our current layout), to provide the same benefits that we give to global installs using Npm and Yarn.

There are a couple options that we can use for that:

- the `NPM_CONFIG_GLOBAL_DIR` environment variable
- the `--global-dir` command-line option

Both of those were introduced in pnpm 4.2, and function the same way.

Using `NPM_CONFIG_GLOBAL_DIR`:

```
$ NPM_CONFIG_GLOBAL_DIR=/Users/mikrostew/src/test-pnpm pnpm add --global cowsay
Packages: +33
+++++++++++++++++++++++++++++++++
Packages are hard linked from the content-addressable store to the virtual store.
  Content-addressable store is at: /Users/mikrostew/.pnpm-store/v3
  Virtual store is at:             node_modules/.pnpm

/Users/mikrostew/src/test-pnpm/5:
+ cowsay 1.5.0

Progress: resolved 33, reused 33, downloaded 0, added 33, done
```

It installs to the provided directory, but still adds shell scripts to the bin dir where `node` is located:

```
$ ll /Users/mikrostew/.volta/tools/image/node/14.16.1/bin/
total 161328
drwxr-xr-x   7 mikrostew  staff       224 Jul 27 10:58 ./
drwxr-xr-x  10 mikrostew  staff       320 Jul 27 10:57 ../
-rwxr-xr-x   1 mikrostew  staff      1812 Jul 27 10:58 cowsay*
-rwxr-xr-x   1 mikrostew  staff      1812 Jul 27 10:58 cowthink*
-rwxr-xr-x   1 mikrostew  staff  73883024 Apr  6 11:22 node*
lrwxr-xr-x   1 mikrostew  staff        38 May  5 23:01 npm@ -> ../lib/node_modules/npm/bin/npm-cli.js
lrwxr-xr-x   1 mikrostew  staff        38 May  5 23:01 npx@ -> ../lib/node_modules/npm/bin/npx-cli.js
```

Using the `--global-dir` command-line option does the same thing:

```
$ pnpm add --global-dir /Users/mikrostew/src/test-pnpm --global cowsay
Packages: +33
+++++++++++++++++++++++++++++++++

/Users/mikrostew/src/test-pnpm/5:
+ cowsay 1.5.0

Progress: resolved 33, reused 33, downloaded 0, added 0, done
```

Again, this adds the binaries to the same bin directory as `node`:

```
$ ll /Users/mikrostew/.volta/tools/image/node/14.16.1/bin/
total 161328
drwxr-xr-x   7 mikrostew  staff       224 Jul 27 11:22 ./
drwxr-xr-x  10 mikrostew  staff       320 Jul 27 11:21 ../
-rwxr-xr-x   1 mikrostew  staff      1812 Jul 27 11:22 cowsay*
-rwxr-xr-x   1 mikrostew  staff      1812 Jul 27 11:22 cowthink*
-rwxr-xr-x   1 mikrostew  staff  73883024 Apr  6 11:22 node*
lrwxr-xr-x   1 mikrostew  staff        38 May  5 23:01 npm@ -> ../lib/node_modules/npm/bin/npm-cli.js
lrwxr-xr-x   1 mikrostew  staff        38 May  5 23:01 npx@ -> ../lib/node_modules/npm/bin/npx-cli.js
```

I don't see an easy way to change where those schell scripts are written. We need to have `node` on the `PATH` to run pnpm.

Having those shell scripts written to the Node image directory interferes with the isolation that we provide for global packages. We will probably have to clean those up, or move them somehow, after the install completes.


## Linking

Using `pnpm link` creates a symlink directly from one package to another, which should work without a problem.

But, `pnpm link --global` stores the links globally, in the same directory as the `node` version that was used to install it:

```
$ pnpm link --global

/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5:
+ yaml-fromat 0.2.0 <- ../../../../../../../src/yaml-fromat
```

That means that if two projects are using different Node versions, they will not be able to link to each other globally using this workflow.
This is basically the problem we had with `npm link`, so this will need functionality similar to that: <https://github.com/volta-cli/volta/pull/888>


# Critique
[critique]: #critique


## Global package install using PATH

For a global package installation alternative, I tried to setup the `PATH` to force it to use a specific directory for everything.

If there is `node` somewhere on the `PATH`, it will install where it finds `node`:

```
$ PATH=/Users/mikrostew/.volta/bin:/Users/mikrostew/src/test-pnpm pnpm add --global cowsay
Packages: +33
+++++++++++++++++++++++++++++++++
Progress: resolved 33, reused 33, downloaded 0, added 0, done

/Users/mikrostew/.volta/tools/image/node/14.16.1/pnpm-global/5:
+ cowsay 1.5.0
```

If there is no directory containing `node` (or similar command), it will fail to install:

```
$ PATH=/Users/mikrostew/src/test-pnpm /Users/mikrostew/.volta/tools/image/node/14.16.1/bin/node /Users/mikrostew/.volta/tools/image/packages/pnpm/bin/pnpm add --global cowsay
 ERROR  Couldn't find a suitable global executables directory.
There should be a node, nodejs, npm, or pnpm directory in the "PATH" environment variable
```

This approach seems brittle, because manipulating the `PATH` relies on internal behavior that may change in the future.

For those reasons I did not include this as a workable option.


# Unresolved questions
[unresolved]: #unresolved-questions

## Global package shell scripts

How do we handle the shell scripts that are auto-installed for global packages? This is probably something that will need to be worked out during the implementation of this feature.

## pnpm Version Support

What versions of pnpm should we support? Should we only support certain versions?
- versions of pnpm prior to v3 don't support the current Node LTS versions (see <https://pnpm.io/installation#compatibility>)
- versions of pnpm prior to v4.2 don't support `NPM_CONFIG_GLOBAL_DIR` and `--global-dir`, for global package installation
