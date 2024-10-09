- Feature Name: node_version_file_support
- Start Date: 2024-10-09
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Introduce support for specifying Node versions using `.node_version` files in addition to the `volta` key in `package.json` files. 

# Motivation
[motivation]: #motivation

This is a long-requested feature which enables many more potential users to use Volta. Today, adopting Volta on a project requires buy-in from the maintainers of the project, since it requires changes to the `package.json` file. Many projects already use the `.node_version` file, so if Volta supports it, users can rely on Volta for the Node runtime for those packages without any changes from the project itself. Moreover, since `.node_version` files exist outside the `package.json`, if they were supported by Volta, it would be possible for users to add them *without* committing them to a project, for example by adding `.node_version` to `.git/info/exclude` in a project where it is not welcome.[^voltarc]

Net, supporting `.node_version` files would be a relatively low-overhead way of making Volta “play nicely” with more of the ecosystem and enable more people to use it successfully in existing projects.

[^voltarc]: This second use case would also apply to a hypothetical `.volta.json` or `.volta/toolchain.json` file or similar. This RFC does not address that use case, but many of the considerations here would apply there as well.

# Pedagogy
[pedagogy]: #pedagogy

Volta will now support a second possible location for the “source of truth” for the Node version, via the `.node_version` file. This integrates cleanly with our existing idea of *toolchain management*.

Volta will not be *defining* but rather *adopting* [the existing loose agreement][node-version-non-spec] about what the file is allowed to contain. Quoting from that document’s [****][suggestion]:

[node-version-non-spec]: https://github.com/shadowspawn/node-version-usage
[suggestion]: https://github.com/shadowspawn/node-version-usage?tab=readme-ov-file#suggested-compatible-format

> ## Suggested Compatible Format
>
> If you are creating the file, a format with full compatibility across current tests is:
>
> - single line with unix line ending
> - three part numeric version e.g. 14.5.0
> 
> A leading 'v' is widely supported, so this will work with most implementations:
>
> ```bash
> $ node --version
> v20.12.0
> $ node --version > .node-version
> ```
>
> If you are reading the file in a new implementation, I suggest you also support optional leading `v` and any line ending. Allowing a leading `v` is common and gives nice symmetry with `node --version`. Allowing any line ending makes it easier for users and especially Windows users to create a file compatible with your product.

The current (as of authoring) Node LTS might then be stored in a file named `.node_version` with these contents:

```
v20.18.0
```

In [Managing your project][managing], we currently describe everything in terms of `package.json`. We will need to extend that to account for the new capability and the additional complexity it bundles:

[managing]: https://docs.volta.sh/guide/understanding#managing-your-project

- The `.node_version` file can *only* be used to specify a Node version; it cannot also be used to specify a package manager.

- If the version is specified in *both* a `.node_version` file and via the `volta` key in `package.json` in the same directory, they must agree.

- Much as we support different versions appearing at different parts of a folder hierarchy in a project for the sake of workspaces, we need to support `.node_version` files appearing at different places. This means that we need to teach the relationship between them, and specifically:
    - that the *closest* specifier wins, regardless of whether that is a `.node_version` file or a `volta` key in a `package.json`
    - that the version supplied by `.node_version` will *override* a version from higher in the tree, but that

### Examples

-   A project where `package.json` has no `volta` key, with this structure:
        
    ```
    project-root/
       package.json
       .node_version
    ```
    
    Volta will use the version specified by `.node_version`.

-   A project where `package.json` has a `volta` key which specifies the Node version (with or without other tools specified), with this structure:

    ```
    project-root/
       package.json
       .node_version
    ```
    
    Volta will:
    
    - Use the version specified by *both*, if they match.
    - Report an error if they do not match.

-   A project where `package.json` does not have a `volta` key, with this structure (notice that `.node_version` is in the *parent* directory):

    ```
    .node_version
    project-root/
       package.json
    ```

    Volta will use the version from the `.node_version` file.

<!--
How should we explain and teach this feature to users? How should users understand this feature? This generally means:
r
- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how users should _think_ about the feature, and how it should impact the way they use Volta. It should explain the impact as concretely as possible.
- If applicable, describe the differences between teaching this to existing Node programmers and new Node programmers.

It is not necessary to write the actual feature documentation in this section, but it should establish the concepts on which the documentation would be built.
-->

# Details
[details]: #details

<!-- 
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

This section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
 -->

# Critique
[critique]: #critique

<!-- 
This section discusses the tradeoffs considered in this proposal. This must include an honest accounting of the drawbacks of the proposal, as well as list out all the alternatives that were considered and rationale for why this proposal was chosen over the alternatives. This section should address questions such as:

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
 -->

Volta has gotten along without support for `.node_version` files for the past five years of general stability. It could presumably continue to do so. Given that there is additional complexity—including additional performance overhead for additional file system crawling in some scenarios—we could choose to leave things as they are.

The major design tradeoffs if we *do* add support are:

**How do we handle the file hierarchy?**

- We could choose *not* to support `.node_version` files in directories which do not also have `package.json` files. This would dramatically reduce the amount we need to crawl the file system. However, it would also mean Volta does not work the way users expect (and the way other tools do).

- We could also choose not to support resolving `.node_version` files once we have found a `package.json`, regardless of whether the `package.json` file has a `volta` key.

- Similarly, we could choose to recurse further if we find a `package.json` file with a `volta` key but missing a `volta.node` specifier *and* not having a `volta.extends` value.

    In the design as proposed scenario, Volta would currently *error*, since it would find an invalid `ProjectSpec` when resolving the package. The alternative would be to error in that case *only* if we also do not find a `.node_version` somewhere further up the file system hierarchy.
    
    This would substantially degrade our ability to give useful feedback for when the user might have unintentionally done something different than they intended. Consider the following layout:
    
    ```
    .node_version
    project-root/
        package.json
    ```
    
    If the user is in the midst of transitioning a whole set of projects in some directory to Volta one by one, and *meant* to pin Volta in the project, but did not, continuing to crawl the hierarchy would cause confusion.
    
    We do not have a good signal from the user about their intent in a case like this (outside of things we *really* do not want to be poking, such as: is that `.node_version` file part of the same repository?).
    
    The downside of not continuing to crawl upward is that a user cannot take advantage of the hierarchy to get `.node_version` support in that case, but since we *only* stop when we hit a `package.json` file which has a `volta` key but not a `volta.extends` value, most of the “ordinary” use cases for the crawling-based approach should work just fine.
    
    Of all options offered as alternatives here, this one seems most reasonable and might be worth picking.

**How do we handle having both `.node_version` and `package.json` with a `volta` key?**[^volta-config-difference] There are two main alternatives here:

    - Simply to forbid this situation entirely. If we do that, we

    -  or to have one always “win”

- **Do we treat `.node_version` as read-only or read-and-write?**

[^volta-config-difference]: Note that this *consideration* extends to a hypothetical standalone Volta configuration file, but the decision there could be substantially different.

# Unresolved questions
[unresolved]: #unresolved-questions

<!-- 
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
 -->
