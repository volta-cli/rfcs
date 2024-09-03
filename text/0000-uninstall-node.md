- Feature Name: uninstall_node
- Start Date: 2024-08-21
- RFC PR:
- Volta Issue:

# Summary

[summary]: #summary

Add support for uninstalling node to `volta uninstall`

# Motivation

[motivation]: #motivation

Currently, Volta doesn't support uninstalling node due to this main reasons(as discussed in [#327](!https://github.com/volta-cli/volta/issues/327)): Its main purpose is to save disk space, nowadays disk size is not a big problem in many cases. So this task is in a low priority.But community has this demand since 2019, they need to manually remove some files to achieve that.Add support for uninstalling node is more convinent in this case, btw since we have `install` here we should also have `uninstall` here.

# Pedagogy

[pedagogy]: #pedagogy

The user's intuition and way of treating it are opposite to `install`. Since `install` fetch node image, unpack it and set it as default, `unintall` should remove those things.

# Details

[details]: #details

The new command will have the structure:

```
volta uninstall node@version
```

totally the same with `install`.

Some examples:

```
volta uninstall node@20.16.0
```

```
volta uninstall node@lts
```

By add a `uninstall` to `Tool` trait and move uninstall logic from `Spec` to `Node, Package, Npm, Yarn, Pnpm`, it would resolve version with `node::resolve` first, and then remove tar.gz file and npm file in `home.node_inventory_dir()` and node in `home.node_image_root_dir()`

# Critique

[critique]: #critique

In fact we want `uninstall` to save disk space or just to remove those things we don't need , it seems like a command like `prune` or `clean` is more suitable, which tracks every tool's status, and when `prune` is runned, we can clean all those tools that is no longer used.

In this case, maybe we should use a file to track every tool. When `volta pin` or `volta install`
is runned, the toolchain must be added to that file (means this toolchain is used in a project or is set as default), and removed outdated toolchain from that file. And we can delete all that we don't use anymore.

But this way will make some big change in volta, and how to track a toolchain's status is complex.

# Unresolved questions

[unresolved]: #unresolved-questions

1. What should `uninstall` do when uninstall a default node?
   Since `install` actually fetch (in background),unpack(in background), and set as default(in user's view), `uninstall` should at least remove tar.gz file, npm file and node dir. but what `uninstall` should do when uninstall a node version that is set as default?
