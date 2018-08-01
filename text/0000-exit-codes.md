- Feature Name: exit_codes
- Start Date: 2018-06-11
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

Exit codes are an important part of Notion's communication with external tools and a first-class part of the design of its user-friendly error messages.  We need a strategy for ensuring they are used and communicated consistently.

# Motivation
[motivation]: #motivation

Each exit code is intended to signal to an external process some information about Notion's state at the end of its execution.  Ideally, we might wish to have a separate exit code for each error Notion is capable of producing, but many modern Unixes (including macOS) only support eight bits of exit status.  If the number of unique exit conditions in Notion exceeds that, we'll need to start "doubling up".

Because redefining exit codes later in Notion's life would be a major hassle for third-party tools reliant on them, we should consider at the outset grouping errors together by exit code.

Further, Notion must avoid conflicts between the exit codes produced by shims and those produced by the shimmed executables, in order to best preserve their behavior.

# Pedagogy
[pedagogy]: #pedagogy

A list of exit codes must be maintained within the Notion source code for use by developers, which should contain descriptions of when each code should be used.  This same list can be exposed via consumer-facing documentation to ensure that Notion developers and consumers share an understanding of the meaning of each exit code.

# Details
[details]: #details

## Windows exit codes

This RFC focuses largely on compatibility with Unix exit codes because there appears to be little consensus as to exit code meaning on Windows, aside from `0` indicating success and other values indicating failure.  Certain system applications (such as the Windows installer) use [stardard Windows error codes](https://docs.microsoft.com/en-us/windows/desktop/Debug/system-error-codes) as exit codes, but that list is nearly fully packed in the 8-bit range and is excessively granular (as it extends well beyond the 8-bit range).  For these reasons, compatibility with standard error codes is not a goal of this document.

## Shims

Of the three "first-class" shims (`node`, `npm`, and `yarn`), only Node currently publishes a [list of exit codes](https://nodejs.org/api/process.html#process_exit_codes); the other two only emit codes 0/1 for success/failure.  The exit codes for other shimmed executables is too large a space to examine, but Node follows what appears to be a common practice: simply numbering errors starting with 1, preserving the [special meanings commonly applied by POSIX shells](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_08_02).  This gives us incentive to avoid low-numbered exit codes (as likely to conflict) and exit codes with the high bit set (as typically representing termination due to signals).

Additionally, shims must do their utmost to preserve the exit code supplied by the shimmed executable, only introducing a shim-specific exit code if the executable cannot be run at all.  To this end, shims must return 0 when the shimmed executable does, even if the shim itself is not able to perform its work (for example, if a new shim cannot be created when installing a new executable via `npm` or `yarn`).

|    Code | Use
| ------: | :--
|   0-125 | Reserved for use by shimmed executables
|     126 | Shimmed executable could not be run
|     127 | Shimmed executable could not be found
| 128-255 | Signal `code - 128` was received **prior** to launching shimmed executable

## `notion`

As `notion` is not subject to conflicts, we are free to define its exit codes in any manner we choose.  However, it it still valuable to follow such conventions as exist, such as not conflicting with common shell codes (2, 126, and 127) and responding to signals in the usual way.

|    Code | Use
| ------: | :--
|       0 | Success
|       1 | Unknown/"unfriendly" error
|       2 | Unused (indicates builtin misuse in bash)
|   3-125 | Errors
| 126-127 | Unused (indicates failure to execute in POSIX shells)
| 128-255 | Terminated due to signal `code - 128`

## Implementation

Exit codes shall be defined as an `enum` with appropriate documentation comments and an explicit integer value for each element.  Being explicit about values allows simple, at-a-glance correlation of exit codes to error types (as well as enabling sparse values).

For example (this list is not meant to be exhaustive), the enum might be defined as follows:

```rust
/// Exit codes supported by the NotionFail trait.
pub enum ExitCode {
    /// No error occurred.
    Success = 0,

    /// An unknown error occurred.
    UnknownError = 1,

    // Exit code 2 is reserved.

    /// An invalid combination of command-line arguments was supplied.
    InvalidArguments = 3,

    /// No match could be found for the requested version string.
    NoVersionMatch = 4,

    /// A network error occurred.
    NetworkError = 5,

    /// The requested executable could not be run.
    ExecutionFailure = 126,

    /// The requested executable is not available.
    ExecutableNotFound = 127,
}
```

### Grouping

Errors will be grouped according to basic cause, with thought given to how external tooling might choose to react to the problem.  Doing so saves external tools from trying to parse Notion error messages in order to make these decisions.  For example, an external tool receiving an exit code representing an error parsing the command line should absolutely not retry the operation (and may have a bug).  An exit code indicating that no matching version is available indicates a potentially non-permanent error, but one which shouldn't be retried without first using another command to change Notion's current state.  An exit code indicating a network error signals a kind of transient problem which might be desirable to retry after a short delay rather than immediately giving up entirely.

### Documentation

Customer-facing documentation will also need to be generated, either by hand or by using a purpose-made tool.  (Rustdoc alone is insufficient for this purpose, as it does not expose the enum's integer values.)  The documentation for the above example might look as follows.  (Note that the descriptions provided to the user is the same as the ones provided to developers.)

| Code | Description
| ---: | :--
|    0 | No error occurred.
|    1 | An unknown error occurred.
|    3 | An invalid combination of command-line arguments was supplied.
|    4 | No match could be found for the requested version string.
|    5 | A network error occurred.
|  126 | The requested executable could not be run.
|  127 | The requested executable is not available.

# Critique
[critique]: #critique

The primary downside introduced by this proposal is the loss of error granularity.  By grouping multiple errors into the same exit code, external tooling loses the ability to differentiate between those errors without capturing and parsing Notion's output.  However, it doesn't seem realistic to provide a separate exit code for each error due to the limited nature of the former and the limitless potential of the latter.  🙂

Another issue is the communication of exit codes.  We want to publish the list as part of Notion's standard, user-facing documentation, but this requires either manually keeping the documentation in sync with the source or writing a tool to do so for us.  The alternative is to point interested parties directly to the Notion source, which isn't ideal.

# Unresolved questions
[unresolved]: #unresolved-questions

- Is a process needed for 'vetting' new exit codes, to prevent duplication?  If so, is the RFC process appropriate?  Alternatively, is code review sufficient?
- Should we engage npm/Yarn to improve error categorization (either through parsable output or diversifying exit codes)?
