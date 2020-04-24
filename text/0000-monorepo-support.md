- Feature Name: monorepo_support
- Start Date: 2020-04-23
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Improve the ergonomics of working with Volta in a monorepo environment.

# Motivation
[motivation]: #motivation

Currently, Volta treats the first `package.json` file it finds in the directory tree as the root of the current project and doesn't look any further for more context. This means that users in a monorepo environment are forced to duplicate their Volta settings in each subproject. Additionally, if they ever want to update their pinned `node` or `yarn` versions, they need to update each subproject individually, negating many of the reasons for using a monorepo in the first place. Volta should provide more ergonomic solutions so users in a monorepo can manage their tool versions in a single place without having to duplicate the settings in each subproject.

# Pedagogy
[pedagogy]: #pedagogy

The changes proposed by this RFC are primarily advanced features required by users who _manage_ monorepo projects. Users who are purely consumers should be able to continue using Volta without needing to consider the details of how Volta determines the appropriate context. For the advanced users, we will need to document:

- Volta's model for having multiple project roots, using the `"extends"` key in `package.json`
- The logic Volta uses to combine platform settings from multiple locations into a single context (including the ability to disable inheriting with the `false` value).
- The additional impact of having multiple project roots, i.e. `hooks` and local binary resolution.
- The behavior of `volta pin` in subprojects vs the root project.

Ideally, we should document all of these with specific examples to show how the different pieces interact. We could potentially even create an `examples` repo (or directory within the main repo) for users to reference.

# Details
[details]: #details

## Multiple Roots

The core change to Volta will be expanding our `Project` model to support a _sequence_ of project roots, instead of a single project root. We will support a sequence by adding the ability to put an `"extends"` entry inside the `"volta"` section of `package.json`. The value of that entry will be the path to another JSON file, which we will parse to determine the next level in the project hierarchy. Note: The file pointed to by `"extends"` does _not_ need to be named `package.json`, but it does need to be a JSON file with a top-level object that has a `"volta"` property. The directory that new file is in will be the next value in the sequence of project roots. Then, if that new file also has an `"extends"` entry, we will continue the same process, building up effectively a linked list of project roots to use for all of our project information.

Importantly, the order of the sequence of project roots will start with the closest `package.json` (what we previously called the sole project root), and then follow the `"extends"` chain out from there.

### Example

Given this folder structure for a monorepo project:

```
.
├── package.json
└── packages
    ├── bar
    │   └── package.json
    └── foo
        └── package.json
```

A common setup would have the following values in the various `package.json` files:

* `package.json`
        {
          "volta": {
            "node": "12.16.1",
            "yarn": "1.22.0"
          }
        }

* `packages/bar/package.json`
        {
          "volta": {
            "extends": "../../package.json"
          }
        }

* `packages/foo/package.json`
        {
          "volta": {
            "extends": "../../package.json"
          }
        }

With this setup (as described in the next section), both `packages/bar` and `packages/foo` would use the `node` and `yarn` versions defined in the top-level `package.json` file.

## Platform Resolution

To resolve the Project platform, taking into account all of the settings in the different project files, we will merge the platform settings from each project file, giving precedence at all times to the files earlier in the sequence. So the highest precedence will be at the existing project root, and then each file that we resolve after that will only update settings that haven't been specified by an earlier entry.

Additionally, while we are processing this chain of project files, we will need to take care to detect accidental loops in the files pointed to by `"extends"` and exit early with a helpful error message (as opposed to entering an infinite loop of parsing and never finishing).

Finally, in lieu of the current parse error for not setting `"node"` in the `"volta"` section, we should merge all of the project settings into a single project Platform, and then only show an error if none of the files specified a Node version.

### Example

Given the following folder structure and `package.json` contents:

```
.
├── package.json
├── bar
│   └── package.json
└── foo
    ├── package.json
    └── inner
        └── package.json
```

* `package.json`
        {
          "volta": {
            "node": "12.16.1",
            "yarn": "1.22.0"
          }
        }

* `bar/package.json`
        {
          "volta": {
            "extends": "../package.json",
            "node": "10.15.0"
          }
        }

* `foo/package.json`
        {
          "volta": {
            "extends": "../package.json",
            "yarn": "1.17.0"
          }
        }

* `foo/inner/package.json`
        {
          "volta": {
            "extends": "../package.json",
            "node": "14.0.0"
          }
        }

The following versions will be used when running commands inside subdirectories:

* Within `bar/`: `node@10.15.0` (from `bar/package.json`), `yarn@1.22.0` (from the root)
* Within `foo/`: `node@12.16.1` (from the root), `yarn@1.17.0` (from `foo/package.json`)
* Within `foo/inner/`: `node@14.0.0` (from `foo/inner/package.json`), `yarn@1.17.0` (from `foo/package.json`)

Note: In the last case, even though there is an `"extends"` chain that reaches the root, no values are used from the root settings, because all of the settings were overridden earlier in the process.

## Local Binary Resolution

Currently, in order to run a project-local binary, we look for the executable in `<project root>/node_modules/.bin/`. With multiple project roots, this process will be almost the same: For each root in the sequence, we check for the binary in the expected place, and if it's found, then we stop and use that version. If we don't find it, we move on to the next root in the sequence.

### Example

Given the following folder structure and `package.json` contents:

```
.
├── package.json
└── packages
    └── foo
        ├── inner
        │   └── package.json
        └── package.json
```

* `packages/foo/package.json`
        {
          "volta": {
            "extends": "../../package.json"
          }
        }

* `packages/foo/inner/package.json`
        {
          "volta": {
            "extends": "../package.json"
          }
        }

If you run a binary that you have previously installed with `volta install` within the `packages/foo/inner` directory, we will look in these locations to find the project-local version of that binary:

* `packages/foo/inner/node_modules/.bin`
* `packages/foo/node_modules/.bin`
* `node_modules/.bin`

Note that since there is no `"extends"` key that points to the `packages` directory, we _don't_ look inside of `packages/node_modules/.bin`.

## Project-Local Hooks

Similar to how we resolve the Platform, we will resolve project-local hooks by looking for `<project root>/.volta/hooks.json` for each project root. We will merge them together, with earlier entries having precedence over later ones, and then finally merge the result with the user-level hooks in the same way that we already do to merge project and user-level hooks.

### Example

Using the same folder structure and `package.json` contents from Local Binary Resolution example:

```
.
├── package.json
└── packages
    └── foo
        ├── inner
        │   └── package.json
        └── package.json
```

* `packages/foo/package.json`
        {
          "volta": {
            "extends": "../../package.json"
          }
        }

* `packages/foo/inner/package.json`
        {
          "volta": {
            "extends": "../package.json"
          }
        }

When you run a command within 

## Pinning Tools

When a user runs `volta pin`, we will always make the changes to the _first_ `package.json` that we find. This means that if a user wants to `pin` a tool for the root project, they will need to first navigate to the root directory (out of any subprojects) and _then_ run `volta pin`. This gives users the most flexibility to control where they want to `pin` tools, without trying to guess which location they were intending or requiring interactivity.

# Critique
[critique]: #critique

## Multiple Roots

Instead of allowing an entire sequence of project roots, we could instead only allow a single level. This would align with techniques like Lerna and Yarn Workspaces, which don't allow nested workspaces. However, apart from the detection of possible loops, there isn't any technical reason that 3 or 4 roots is less complicated than 2, so going with a general approach will allow us to be the most flexible.

## Requiring Changes in each Subproject

This proposal requires that every subproject in a monorepo have (at the very least) an `"extends"` entry pointing to the root, if the want to take advantage of these features. We could take the approach used by e.g. Yarn workspaces and instead have some way of marking the root project and indicating which subprojects should use those settings. This has a couple of major issues, however, that make it a less than ideal solution:

- It requires more speculative IO, as even once we find a `package.json`, we still need to continue looking up the directory tree for more, parsing them, and then trying to figure out if our project is correctly a subproject of the root.
- It requires duplicating the workspace configuration, since Volta will need to have the settings in addition to the existing Yarn workspace information in the root `package.json`.
- It obfuscates the source of the platform - A user trying to figure out why we are using a specific Node version will have to do the same digging to figure out where the settings are. In our model, the user can look at the subproject and it will point them to the root project (or next project in the chain), so they always know where to start when investigating.

## Pinning

Rather than always pin in the closest root to the current directory, we could instead give users more fine-grained control over where to pin tools with the `volta pin` command. However, since we are proposing to support a sequence of project roots, those controls will need to be fairly involved and would likely result in a difficult-to-understand experience. By always pinning the closest, we can easily explain that if you want to make changes to the root platform, first move to the root directory, then run the `pin` commands, and they will automatically apply to all of the subprojects.

# Unresolved questions
[unresolved]: #unresolved-questions

- In our output for `volta list`, we currently show the Project root for any tool that is determined by the project (e.g. `current @ /path/to/project`). Should we update our sources to include the specific file or continue to use the first project root as the "root" for that output?
