- Feature Name: global_packages
- Start Date: 2020-07-15
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Update our handling of global package installs to use existing package manager idioms (e.g. `npm i -g`) with minimal mental overhead.

# Motivation
[motivation]: #motivation

Our current approach to global package installs with `volta install` is to sandbox each package separately. This allows us to ensure consistency and reliability of the global binaries, since each has an associated Node version independent of the default version. We can be sure that calls to a given package's binaries will continue to work, because we always run within that sandboxed environment. However, this behavior leads to a number of issues that hurt the user experience and break Volta's stated goal of transparency:

- **The selection of an associated Node version, while documented, is opaque** We surface the version that we select in `volta list`, but we don't currently provide any feedback immediately about what version was picked nor do we allow users to modify that version. This makes it tricky for users to fully grasp which Node versions will run when they execute a command (c.f. [#605](https://github.com/volta-cli/volta/issues/605) and [#631](https://github.com/volta-cli/volta/issues/631)).
- **Custom behavior breaks standard workflows** Since we do a lot of customization of the install process, we use a separate command (`volta install`) instead of the industry-standard approach of `npm i -g` or `yarn global add`. To ensure that everything works, we also intercept those standard commands and redirect the user to use `volta install`. This is a major point of confusion for users of Volta, especially when they are following the instructions all over the internet to install a tool with `npm i -g` and are met with a weird error message that says to use `volta install` instead.
- **Custom behavior can be incompatible with package manager installs** Our workflow currently is subtly different from a standard global install, which causes difficult to diagnose and fix bugs around the install process (c.f. [#622](https://github.com/volta-cli/volta/issues/622) and [#681](https://github.com/volta-cli/volta/issues/681)).
- **Alternative package sources aren't supported.** Since we have a custom workflow for installing a package, we don't currently support installing packages from tarballs or from GitHub coordinates. The packages managers can already support this behavior, so we impose an arbitrary restriction on that functionality in Volta (c.f. [#427](https://github.com/volta-cli/volta/issues/427) and [#472](https://github.com/volta-cli/volta/issues/472)).
- **Interoperability between packages** As our packages are separated into their own silos, we have a number of issues all related to global packages interacting with other globals:
    - https://github.com/volta-cli/volta/issues/762 - Dependency can't find parent project because of directory structure
    - https://github.com/volta-cli/volta/issues/652 - One global tool can't find another even when both are installed
    - https://github.com/volta-cli/volta/issues/555 - Access to global libraries through NODE_PATH
    - https://github.com/volta-cli/volta/issues/492 - Global tools calling other tools

Ultimately, the goal of Volta is to improve the workflow of developers while staying out of their way. Our current package handling doesn't meet that goal and should be improved to better fit user expectations.

# Pedagogy
[pedagogy]: #pedagogy

The goal of this change is that there should be _less_ we need to teach users, instead of _more_. By shifting to use the existing workflows, package installs should "just work" the way users expect them to, with some small improvements. We won't have to teach users about an entirely new command to manage their tools, only about the steps we take to smooth out their workflows. Those changes boil down to:

- Delegation to project-local binaries where appropriate.
- Automatic maintenance of globals when switching Node versions.

This will greatly simplify the mental model needed to use Volta and reduce confusion around the package install process.

## Current Users

While the overall result will be a significant simplification in the user experience, this change could potentially be a big one for our current users. Ideally the solution will be implemented in a way that it's roughly compatible and existing users won't notice a major shift—apart from new possibilities and fewer issues—but it's a significant shift in the mental model.

To ease the transition and for consistency with the existing workflows, we should continue to support `volta install <package>` as we do now, fitting it into the new model by performing the same actions we would if a user runs `npm i -g <package>`.

# Details
[details]: #details

## Constraints

* **Global installs with package managers should work** We need to move away from our hard error interception of `npm i -g` and `yarn global add` and instead move to use those commands as the _primary_ way of installing global packages. This will allow users to continue with the familiar workflows they already use.
* **Global packages should continue to work when switching Node versions** While we currently handle this with siloed versions of Node, this is one of the key benefits that Volta provides over other Node managers and it should be preserved.
* **Global binaries should be able to call other global binaries** With a "standard" npm or Yarn setup, all of the tools that are globally installed are put onto the PATH directly, so they can all call each other without issue. We add the associated Node version to global tools, so we likely need to go through the shim logic, but that shouldn't stop tools from being able to call other tools (or themselves recursively).
* **Global libraries should be allowed** While not extremely useful, since they aren't generally accessible from arbitrary scripts, we should nevertheless allow users to install global packages that don't have any binaries. This is important for things like peer dependencies and `yeoman` generators.
* **Global packages should be able to `require` globally installed libraries** npm installs all global packages under a single `node_modules` directory, so when calling `require` from one package, it automatically can see the other packages that were installed globally. We should preserve this behavior as much as we can; the main difficulty I see here is native modules and differing associated Node versions.
* **Global packages should leverage Node module resolution** Our directory structure currently doesn't have any folders named `node_modules`, so the Node module resolution algorithm is completely unable to detect our packages, even from within a package's dependencies. There shouldn't be a need to completely reinvent the wheel here, we should do our best to leverage the existing resolution algorithm.

## Overview

The core of this approach is to take advantage of the existing behavior of package manager global installs like `npm i -g` and `yarn global add`. Those tools already provide the expected interoperability between global packages that we want and users have intuition around how they work. We can insert some customization before the global install runs to point it at a custom directory that Volta controls, so we can keep track of everything. Additionally, we can do record-keeping after the global install finishes to ensure that we keep track of which tools and binaries are installed.

Specifically, we would use the in-built behavior of `npm i -g` and `yarn global add`—along with the user's current default platform—to actually perform global installs. We can take advantage of the methods provided by the package managers to redirect the global directory to one that we control and then make sure to do our record-keeping after the global install finishes.

Finally, to avoid forcing the user to re-install every global package whenever they switch Node versions, we can use our manifest of installed packages to automatically re-install them whenever the user changes their default Node version. That way they get the benefits of global installs without the added hassles.

## Sandbox Per Node Major Version

The major issue preventing us from having a single persistent store of global packages is that Node major versions can represent breaking changes that cause a package to stop working. This is most notable with native modules, which are compiled against a specific Node Modules API version and need to be recompiled to work against a different Node. Since these types of changes are primarily limited to major version bumps in Node, we can keep a separate cache of global packages for each major version. This will be stored within the Volta directory, most likely under `tools/image/packages/<node version>`.

Within a given Node Major bucket, we expect packages to continue to work without needing to be re-installed, so we can avoid needing to have a separate cache for every individual Node version. This also allows us to reduce the amount of time we spend re-installing packages on Node version switches.

## Interception of Global Installs

When a user runs `npm i -g <package>` or `yarn global add <package>`, instead of throwing a hard error as we currently do, we can make 2 small changes to the workflow but otherwise allow the command to run as expected:

1. Use the `prefix` and `global-folder` options to redirect the install directory to the Volta-managed location. Package managers already provide support for changing the global install directory, so we can leverage that to make sure we always install into the appropriate Node major version cache. We should set these (if possible) using environment variables, so that they can still be overridden by command-line switches from the user in cases where they want to have control over the directories.

2. After the install completes, take inventory of the packages that changed (whether installed, upgraded, or uninstalled) to update an internal manifest of all packages the user has globally installed.

Additionally, if a user calls `volta install <package>`, we can perform the same steps and shell out to `npm i -g <package>` as if the user had typed that directly.

## Automatic Migration When Switching Node Versions

In order to avoid requiring users to keep track of their global packages and manually re-install them whenever they switch Node versions, we can instead manage that for them. Using the manifest generated above whenever a package is updated, we will always know which tools the user has available to them. Then, whenever they switch to a new major version of Node (with `volta install node@<version>`), we can ensure that the full list of packages is made available by running the appropriate `install` commands.

In order to be efficient at this change, we should keep a separate manifest for each major version of Node, so that we can perform a diff with the "primary" manifest and deteremine exactly which changes are needed to get the system in a working state.

This process may take some time, so we should provide clear visual feedback to the user to indicate that we're refreshing their packages (and if possible, progress on how much is left to do). Additionally, while we generally want this behavior to maintain Volta's guarantees about binary availability, there are special cases so we should have a command-line switch to allow users to skip the consolidation if they don't need it.

### Errors While Migrating

Once switching Node versions completes, we should consider that a success, regardless of whether any of the global package installs fails. This means that packages failing to install shouldn't ever result in the `volta install node@<version>` command returning a non-zero exit code. However, for any packages that do fail to install, we should show a warning letting the user know that we were unable to make that package available and so it likely won't work correctly. We may also want to include a call-to-action prompting them to change to a different version of the tool that is compatible with their new Node version.

## Handling Global Libraries

Since libraries without binaries are already supported by `npm i -g`, this approach also includes that support without needing any modifications.

## `volta list`

Since this change represents a significant update of the internal model for packages, it will require the output of `volta list` around packages to be redesigned. Specifically, packages will no longer be associated with a specific platform any more, so we should be able to simplify the output to only show the package version (and whether it is the default version or a project-local version). For advanced users / more verbose output, we can also show the specific version of Node used to install a specific package, in case there are issues with different Nodes within the same major bucket (though we also want to be very explicit about what that annotation means, to avoid confusion).

## Workflow Examples

On a fresh install, a user selects the latest 12.* version of Node as their default:

```bash
> volta install node@12
```

Once this is complete, they are using `node@12` and don't have any global packages installed yet. Next, the user installs TypeScript 3.9 as a global:

```bash
> npm i -g typescript@3.9
```

Volta detects this installation and ensures that `npm` installs `typescript` inside of Volta's internal Node 12 bucket. It also notes the specific version (3.9.7 as of this writing) of `typescript` that is installed in the manifest of installed packages.

Now, the user decides they actually want to work on the bleeding edge of Node, so they install Node 14:

```bash
> volta install node@14
```

After making Node available, Volta will check the manifest of global packages and see that we need to make sure `typescript@3.9.7` is available, so it will internally run `npm i -g typescript@3.9.7` under Node 14, making sure that it is installed into the Node 14 bucket. At the end of this process, the user can run `tsc` as a global just like they could under Node 12.

While working in Node 14, the user installs `ember-cli@3.18.0` and also decides that they want to downgrade to `typescript@3.8.2`. Volta ensures that those changes are reflected in the Node 14 bucket, as well as keeping track of the active versions in the global manifest:

```bash
> npm i -g ember-cli@3.18.0
> npm i -g typescript@3.8.2
```

Finally, the user decides that working on Node 14 isn't working out, and they switch back to Node 12:

```bash
> volta install node@12
```

After ensuring Node 12 is available, Volta will once again check the manifest of global packages against the packages that are currently set up in the Node 12 bucket. In this case, Volta detects that we need to install `ember-cli@3.18.0` and downgrade `typescript` to `3.8.2`, so it runs `npm` commands internally to ensure those changes are made. When the process is complete, the user has the exact same tools that they did on Node 14.

The end result is that the user doesn't have to think about a matrix of which tools they installed for which Node versions. They only have to consider which tools they have installed and which Node version they want to use. This greatly streamlines their workflow and makes it easier to work with globals, while still getting the expected behaviors of a "global" install.

# Critique
[critique]: #critique

## Reinstallation Time on Node Version Switch

As the number of global packages installed on a user's machine grows, the process of re-installing them for new Node versions will take longer and longer. This is a concern we need to keep an eye on, however it is mitigated by a couple of factors:

1. We only need to do a re-install when the user switches _major_ Node versions, which are only released on a 6-month cadence.
2. By doing a diff of the global manifest with the major version manifest, we can limit the changes to only the necessary subset. This will reduce the amount of work we need to do on any given Node switch, as well as allow us to completely skip re-installs if no changes have been made.

Finally, we plan to offer a command-line switch of some kind to avoid the re-installation altogether, for times when the user needs to make a quick Node version switch and doesn't care about the specifics of which packages are available.

## Just-In-Time Binary Installation

Related to the other concern, instead of re-installing every package when switching Node versions, we could instead use the shims to do a just-in-time installation of a binary. Under this model, if a user switches Node versions, no changes are made. Then, when they go to execute a binary (e.g. `tsc`), we detect that the correct version isn't available right before executing and run the install at that point. This has the benefit of only requiring re-installation when a tool is _used_, as opposed to when the user switches Node versions.

However, this approach falls down when it comes to the interop of tools. Since we don't know ahead of time which other binaries and libraries will be called from a given tool, there's no way to know the full set of packages we need to install to ensure a given command works as expected. This breaks users expectations and would cause significant confusion around why some packages are available and some aren't.
