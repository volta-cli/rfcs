- Feature Name: notion which
- Start Date: 2019-01-23
- RFC PR: [#29](https://github.com/volta-cli/rfcs/pull/29)
- Notion Issue: [#293](https://github.com/volta-cli/volta/pull/293)

# Summary
[summary]: #summary

Add a command `notion which` takes one or more arguments. For each of its arguments it prints to stdout 
the full path of the executable that notion's shim will execute. This command will function identically to `which` but instead of returning
the path to the shim it will return the path to the underlining executable.


# Motivation
[motivation]: #motivation

With notion "shimming" executables in the users path this causes the existing `which` command to return the path to the shim instead
of the full path to the underlining executable. In most cases knowing the exact path is not needed however cases where coping from the 
location of say `which node` would be broken under notion. This command would offer an escape valve for users to regain the lost functionallity
of `which` while also providing debugging benefits.

# Pedagogy
[pedagogy]: #pedagogy

Since this command is intended to "mimic" the behavior of `which` teaching the command should not be much more than pointing to the
man page of `which`. The only thing that would need to be called out is that it will not return the path of the shim but the path of the
underlining notion controlled path.

# Details
[details]: #details

Command structure:

**notion which commandName [commandName2, ...]**

Command output:

**/full/path/to/commandName**

**/full/path/to/commandName2**

# Critique
[critique]: #critique

The biggest drawback would be the complexity of adding the command. It is another command that must have dev support, 
documentation, etc. 

# Unresolved questions
[unresolved]: #unresolved-questions

- Should `notion which` work exactly like `which` or should it only work for executables that notion "owns"?

Example would be what should `notion which ls` return? Should it return the path of `ls` or should it error with something about 
notion not knowing where `ls` is?

- Should notion shim `which` instead of adding a new command?

Another approach would be to shim `which` and add the functionality to the shim. This seems like a more complicated approach but the
tradeoffs should be evaluated vs adding the command.

- Should `notion which` support all of the options that `which` does?
