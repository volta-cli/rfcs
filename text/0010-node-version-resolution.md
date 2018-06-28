- Feature Name: node_version_resolution
- Start Date: 2018-01-28
- RFC PR: https://github.com/notion-cli/rfcs/pull/10
- Notion Issue: https://github.com/notion-cli/notion/issues/26

# Summary
[summary]: #summary

This proposal describes the way Notion resolves the Node version number specified in subcommands and/or configuration. The intuition is that Notion always picks the **most recent locally-installed version** that is **semver-compatible** with the specifier.

# Motivation
[motivation]: #motivation

This proposal is intended to address some central aspects Notion:

- What does the developer mean when they specify the version of Node their project is using?
- What files does Notion involve?
- How and when do Node version upgrades happen?

# Pedagogy
[pedagogy]: #pedagogy

Similar to other version managers, Notion has a concept of the **currently-installed versions** of Node on the local machine.

When a user requests the installation of a Node version, the specifier is always understood as a **semver version specifier**.

Node versions specified in the manifest can simply be explained as **recording and automating** the Node version request—but **scoped to the local project**—as if the programmer had written `notion use $spec` manually at the command-line.

For projects that need a very specific version of Node, they can specify a pinned version as a full `$major.minor.patch` specifier. Typically this would be more important for apps than libraries.

# Details
[details]: #details

## Resolution semantics

For an installation request for Node version spec `$spec`, e.g.:

```
notion install $spec
```

or a project manifest with Node version spec `$spec`, e.g.:

```js
"notion": {
  "node": $spec
}
```

the resolution semantics follows these steps:

1. If there are any installed semver-compatible Node versions, return the latest one.
2. If there are any cached semver-compatible Node versions, install and return the latest one.
3. Let *result* be the result of calling the *resolve* hook from configuration on `$spec`.
4. If *result* is `{ "version": $version, "url": $url }`:
   1. Fetch the installer from `$url`.
   2. Cache and install the installer.
   3. Return `$version`.
5. If *result* is `{ "version": $version, "stream": true }`:
   1. Fetch installer from the *resolve* addon’s stdout.
   2. Cache and install the installer.
   3. Return `$version`.
6. Otherwise produce an error.

## Semver syntax

For consistency and least surprise within the Node ecosystem, the semver flavor we use treats the default semver version spec operator as `=` (unlike Cargo, which treats the default operator as `^`).

Again for consistency with npm, a leading `=` or `v` is stripped from a version specifier.

# Critique
[critique]: #critique

## Auto-pinning

This design does *not* automatically pin Node versions. The biggest drawback is that one project can trigger an upgrade and then another unpinned project can suddenly start to change behavior, resulting in surprising bugs.

An alternative would be to use a lockfile, which developers would check into version control to standardize across their team. This would ensure stronger guarantees, but at real costs:

- More unnecessary Node installs (download, install, install global packages, force rebuilding of native modules).
- Increased pressure for a service like Greenkeeper to help ease (and encourage) updating. Needing a service makes adopting Notion more costly.
- High process overhead relative to Notion’s narrow purpose. Notion’s goal is to “humbly” stay out of the developer’s way.

Meanwhile, pinning a version manually via configuration works fine—unlike with package managers—since there are no transitivity issues.

It’s also worth noting that pinning versions is useless and maybe even harmful for libraries, which need to support a variety of Node versions.

We may want to offer a `notion pin` command to make manual pinning even easier.

## Chunky plugin hooks

This proposal coalesces plugin calls down to at most one. An alternative would be to split it up into finer-grained hooks (e.g. `resolve` vs `fetch`). This design optimizes the overhead of possibly-expensive calls to external plugins.

## Non-extensible versioning

Similarly, this design avoids plugin calls when there is already an installed version that matches. This makes the most common case extremely fast. A downside is that these common cases don’t get to upgrade. But the upside is that it’s more under the developer’s control whether/when they want to upgrade. This also matches the existing mental model of version managers, so it should be more familiar to developers.

## Default semver operator

We could interpret semver specifiers with `^` by default instead of `=`. However, this would be inconsistent both with npm's behavior for library specifiers as well as most version managers' semantics. That is, it would be surprising to many users to type:
```
notion install 8.8
```
and find version 8.9.4 installed as a result.

# Unresolved questions
[unresolved]: #unresolved-questions

How should we do Yarn version resolution? Probably similarly, since upgrades between Yarn minor versions will tend to work in practice and again they can be pinned manually. But we’ll need to work out the details as we implement.

How should we integrate this with testing matrices (both for manual testing and CI)? This should be addressed in a follow-up design.
