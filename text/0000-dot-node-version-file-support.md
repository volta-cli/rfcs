- Feature Name: dot_node_version_file_support
- Start Date: 2025-03-04
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Introduce support for a `.node-version` file, enabling it as a fallback source for determining project-specific Node versions. If Node version is not specified in `package.json`, fallback to `.node-version` before using global defaults.

# Motivation
[motivation]: #motivation

The aim is to align Volta with other popular Node version managers like [mise](https://github.com/jdx/mise), [fnm](https://github.com/Schniz/fnm), [n](https://github.com/tj/n) and [asdf](https://github.com/asdf-vm/asdf-nodejs), which support `.node-version`. This enhances flexibility and user experience by allowing developers to specify their preferred Node versions in a widely recognized format.  
A list of supporting products can be found here: <https://github.com/shadowspawn/node-version-usage>

# Pedagogy
[pedagogy]: #pedagogy

Existing Node developers are likely already familiar with different Node versioning methods. Introducing support for `.node-version` aligns Volta with other Node version managers.

# Details
[details]: #details

## Compatible file format

The contents of `.node-version` will be:

- optionally a leading `v`
- three part numeric version (e.g. 20.18.2)
- optionally a trailing newline (either Unix style or Windows style line endings are allowed)

Any content found in addition to the above will result in an error message explaining that the `.node-version` file is malformed.

A leading `v` is widely supported, so this will work with most implementations:

```sh
$ node --version
v20.18.2
$ node --version > .node-version
```

It is recommended to support optional leading `v` and any line ending [[reference](https://github.com/shadowspawn/node-version-usage#suggested-compatible-format)].  
Allowing a leading `v` is common and gives a nice symmetry with `node --version`.  
Allowing any line ending makes it easier for users and especially Windows users to create a compatible file.

### Invalid `.node-version`

When an invalid `.node-version` is found (see [Compatible file format](#compatible-file-format)), Volta will emit an error indicating, as best as we can tell, what is wrong with the file, along with it's file path. _(Implementation note: this could either be relative path or absolute path.)_

Primary error cases:

- Invalid semver range (e.g. `lts/*` or `20`)
- Unsupported content (leading content, trailing content, etc)

> The .node-version file at _\<path\>_ is malformed

## Inheritance from current solution

The new feature will act as a fallback when resolving Node version, if `package.json` does not have a `volta` section specifying the Node version.

## Standardized Approach

This implementation should mirror the behavior of other Node version managers, ensuring Volta remains competitive and user-friendly.

## Lookup Hierarchy

The Node version lookup process should follow the order described:

1. Resolve `package.json` and check its `volta.node` field.
1. Check for the presence of `.node-version`.
1. Fallback to the default active Node toolchain.

## Backward Compatibility

This feature does not disrupt current functionality and can be released immediately without breaking changes.

## Implementation Considerations

Ensure the implementation respects the same environment inheritance as today’s solution when reading versions from `package.json`.
Maintain consistent behavior across different scenarios, ensuring that if no specific version is found in either `package.json` or `.node-version`, Volta defaults to its standard global setting.

No new commands are required; this feature should integrate seamlessly with existing Volta commands.

## Testing Strategy

Develop comprehensive tests that cover various scenarios, including:

- Presence and absence of `.node-version`.
- Conflicting versions between `.node-version` and `package.json`.
- Interaction with global Node version settings.

# Critique
[critique]: #critique

By supporting `.node-version`, Volta will provide users with more flexibility in how they manage their Node environments, while maintaining its commitment to backward compatibility. This RFC invites feedback on the proposed implementation and any potential edge cases that might arise.

# Unresolved questions
[unresolved]: #unresolved-questions

Should we also support `.nvmrc`?
