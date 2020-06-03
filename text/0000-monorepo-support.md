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
- The logic Volta uses to combine platform settings from multiple locations into a single context.
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
    ```
    {
      "volta": {
        "node": "12.16.1",
        "yarn": "1.22.0"
      }
    }
    ```

* `packages/bar/package.json`
    ```
    {
      "volta": {
        "extends": "../../package.json"
      }
    }
    ```

* `packages/foo/package.json`
    ```
    {
      "volta": {
        "extends": "../../package.json"
      }
    }
    ```

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
    ```
    {
      "volta": {
        "node": "12.16.1",
        "yarn": "1.22.0"
      }
    }
    ```

* `bar/package.json`
    ```
    {
      "volta": {
        "extends": "../package.json",
        "node": "10.15.0"
      }
    }
    ```

* `foo/package.json`
    ```
    {
      "volta": {
        "extends": "../package.json",
        "yarn": "1.17.0"
      }
    }
    ```

* `foo/inner/package.json`
    ```
    {
      "volta": {
        "extends": "../package.json",
        "node": "14.0.0"
      }
    }
    ```

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
    ```
    {
      "volta": {
        "extends": "../../package.json"
      }
    }
    ```

* `packages/foo/inner/package.json`
    ```
    {
      "volta": {
        "extends": "../package.json"
      }
    }
    ```

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
    ```
    {
      "volta": {
        "extends": "../../package.json"
      }
    }
    ```

* `packages/foo/inner/package.json`
    ```
    {
      "volta": {
        "extends": "../package.json"
      }
    }
    ```

When you run a command within the `packages/foo/inner` directory, we will look in the following locations for Volta hooks:

* `packages/foo/inner/.volta/hooks.json`
* `packages/foo/.volta/hooks.json`
* `.volta/hooks.json`

Again note that we don't look in `packages/.volta/hooks.json`, because the `packages` directory is not in our chain of project roots.

## Pinning Tools

When a user runs `volta pin`, we will always make the changes to the _first_ `package.json` that we find. This means that if a user wants to `pin` a tool for the root project, they will need to first navigate to the root directory (out of any subprojects) and _then_ run `volta pin`. This gives users the most flexibility to control where they want to `pin` tools, without trying to guess which location they were intending or requiring interactivity.

## Support for common workflows

Beyond the basic support for nested projects provided by the above details, we can also provide enhancements that improve the experience when using common monorepo tools, such as [Yarn Workspaces](https://classic.yarnpkg.com/en/docs/workspaces/) or [Lerna](https://github.com/lerna/lerna). For example, both of those tools have a top-level configuration that lists all the packages that are contained in the workspace, so we can automate writing the `"extends"` keys into each of the subprojects when a user runs `volta pin` in the top-level directory.

# Critique
[critique]: #critique

## Requiring Changes in each Subproject

A significant criticism of this proposal is that it requires that every subproject in a monorepo explicitly opt-in to the behavior with an `"extends"` entry pointing to the root. This is less of a concern in small, single-project Monorepos where the same team is working on all of the subprojects, so maintaining that cohesion is straightforward. However, it becomes a bigger issue in monorepos for large organizations, where there may be a different team (or teams) managing the overall structure of the monorepo, separate from the team(s) managing the individual subprojects. This means that a team adding a new subproject will need to know how to opt-in to using the monorepo's configuration, reducing the organization's ability to modularize the responsibility.

However, when declaring dependencies within a monorepo, a subproject still needs to be aware of the overall structure of the monorepo, in order to correctly reference other in-repo dependencies. The Volta settings in `package.json` are conceptually very similar to dependencies, declaring platform dependencies as opposed to code dependencies. While our approach does result in subprojects having a responsibility to be aware of the overall monorepo configuration, that responsibility isn't an increase over the responsibility they already have to be good citizens of the monorepo with their declared dependencies.

Additionally, there are a number of technical and conceptual issues with using a top-down approach similar to that used for Yarn workspaces:

- It obfuscates the source of the platform - A user trying to figure out why we are using a specific Node version will have to do the same digging to figure out where the settings are. In our model, the user can look at the subproject and it will point them to the root project (or next project in the chain), so they always know where to start when investigating.
- It requires either duplicating the workspace configuration or tying our implementation tightly to how the package managers define workspaces:
    - If we have our own setting for indicating which projects are part of the workspace, then users will have to duplicate that setting between Yarn / npm / pnpm / lerna and Volta, which comes with additional maintenance costs as they will also need to keep them in sync.
    - If we instead chose to re-use the existing `"workspace"` key used by Yarn et al., we will need to ensure that our parsing aligns with those other tools exactly, so that we don't accidentally get out of sync ourselves. This also raises the question of how do we handle it when two tools diverge on their implementation?
- It requires more speculative IO. This is likely less of a significant concern, but is definitely something we would need to plan for and monitor to ensure we don't have significant performance regressions.


## Multiple Roots

Instead of allowing an entire sequence of project roots, we could instead only allow a single level. This would align with techniques like Lerna and Yarn Workspaces, which don't allow nested workspaces. However, apart from the detection of possible loops, there isn't any technical reason that 3 or 4 roots is less complicated than 2, so going with a general approach will allow us to be the most flexible.

## Pinning

Rather than always pin in the closest root to the current directory, we could instead give users more fine-grained control over where to pin tools with the `volta pin` command. However, since we are proposing to support a sequence of project roots, those controls will need to be fairly involved and would likely result in a difficult-to-understand experience. By always pinning the closest, we can easily explain that if you want to make changes to the root platform, first move to the root directory, then run the `pin` commands, and they will automatically apply to all of the subprojects.

# Unresolved questions
[unresolved]: #unresolved-questions

- In our output for `volta list`, we currently show the Project root for any tool that is determined by the project (e.g. `current @ /path/to/project`). Should we update our sources to include the specific file or continue to use the first project root as the "root" for that output?
