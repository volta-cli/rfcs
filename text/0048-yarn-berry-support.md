- Feature Name: yarn_berry_support
- Start Date: 2022-02-16
- RFC PR: https://github.com/volta-cli/rfcs/pull/48
- Volta Issue: https://github.com/volta-cli/volta/issues/651

# Summary
[summary]: #summary

Roadmap of the necessary changes to support Yarn Berry (Yarn 2.0+)

# Motivation
[motivation]: #motivation

We currently only support legacy Yarn 1.x installation and pinning. While Yarn 2.x had a lot of infrastructure churn that made it difficult to support, Yarn 3.0+ is more stable and provides all of the pieces that we need to add support. To maintain compatibility as Yarn continues to evolve, we should support Yarn Berry.

# Pedagogy
[pedagogy]: #pedagogy

The ideal outcome with this change is that there will be very little we need to teach users. Newer versions of Yarn will be supported and users will be able to continue to use Volta the way they always have, without needing to worry about specific details of their setup.

The main detail we will need to communicate is the lack of support for the 2.x major version of Yarn (only supporting Yarn 1.x and Yarn 3.0+).

# Details
[details]: #details

## Skip Yarn 2.x Support

As mentioned above, the initial release of Yarn Berry involved a lot of changes that were difficult to track and didn't fit with Volta's model for handling package installation. The biggest blocker is that the distribution package on npm wasn't created until the last version of Yarn 2. Before that, there wasn't any reliable way to fetch the Yarn binary and make it available within Volta.

Since the only viable distribution is the _final_ version of Yarn 2 and Yarn has since released Yarn 3, it makes the most sense to skip support for Yarn 2 alltogether and jump straight from Yarn 1 to Yarn 3. That will allow us to detect the supported versions very easily, using only the major version number rather than a bespoke list of "allowed" versions. We should provide a useful error message if a user attempts to install or pin Yarn 2, directing them instead to use Yarn 3 for the most up-to-date experience.

## Installation

Currently, for Yarn 1, we fetch the tarball from the [`yarn`](https://www.npmjs.com/package/yarn) package on the npm registry. However, Yarn Berry isn't included in that package, so we instead need to fetch the package at [`@yarnpkg/cli-dist`](https://www.npmjs.com/package/@yarnpkg/cli-dist). That package includes everything we need for Volta's drop-in install: The JS script and the executables to launch them. When fetching a version of Yarn, we will need to detect the version and choose the appropriate package, then download and install it using the existing logic.

### Hooks

One outstanding question is how we want to handle the hooks for installs. Since the Hooks format was created before we switched Yarn to use the npm registry, the logic is rather complicated:

* If we aren't using hooks, then we fetch from npm and expect the data to match the npm package format
* If we _are_ using hooks, then we fetch from the location specified by the hook and we expect the data to be in the old GitHub API format

With the addition of Yarn 3 pulling from a different package (though still using the npm registry), this could get more complicated. My current suggestion is that we keep the hooks format the same even for Yarn Berry releases. This will mean that teams using hooks will have to make sure the correct package is available at the appropriate URLs themselves. Then, in a future change where we update the hooks format, we can migrate all of Yarn to use the npm format and normalize the behavior.

## Plug'n'Play

Beyond installation, the other blocker for fully supporting Yarn Berry is that our global package delegation does not work with Yarn Plug'n'Play. The root issue is that we expect any project-local binaries to be inside of `node_modules/.bin`, which isn't the case for Plug'n'Play installations.

To work around that, I believe we can detect the presence of the Plug'n'Play include file (named `.pnp.js` or `.pnp.cjs` depending on the Yarn version) and instead of calling the binary ourselves, delegate to it using `yarn run <binary>`. This will make launching a tool slightly slower in those scenarios, however there is no way (without tightly coupling to internal Yarn implementation details) to resolve the path to a binary in Plug'n'Play without invoking `yarn`.

# Critique
[critique]: #critique

## Broader Version Support

One alternative approach would be for us to attempt to support as many versions of Yarn Berry as we can, including in the 2.x major version. This would give us the greatest coverage of possible use-cases, however given the churn in distribution and API, it will be very labor-intensive to implement. It also would likely still not support every possible version of Yarn, still leaving some gaps in our coverage. Since Yarn 2.x is not recommended by Yarn themselves either at this point, the additional work to support various 2.x releases feels like it would have a low impact while significantly increasing the complexity of the change.

## Hooks Format

The current suggestion for the hooks is the easiest from an implementation standpoint, however it offloads that work onto users that are taking advantage of hooks. However, our Hooks behavior is already not ideal and adding more complexity to it feels like it would make things even more difficult to understand. Separate from this issue, we should investigate an improved Hooks V2, with a new file format and better ergonomics, taking into account everything we learned from the original hooks implementation. At that time, we can define a more appropriate API for the Yarn hooks. That said, I don't believe our support for Yarn Berry should be blocked on redesigning Hooks, which is likely a much bigger issue overall.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should we consider Corepack support as part of support for Yarn Berry? ([Corepack](https://nodejs.org/api/corepack.html) is Yarn's currently-recommended way of installing / managing Yarn versions)
