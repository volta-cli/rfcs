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

# Pedagogy
[pedagogy]: #pedagogy

A list of exit codes must be maintained within the Notion source code for use by developers, which should contain descriptions of when each code should be used.  This same list can be exposed via consumer-facing documentation to ensure that Notion developers and consumers share an understanding of the meaning of each exit code.

# Details
[details]: #details

Exit codes shall be defined as an `enum` with appropriate documentation comments and an explicit integer value for each element.  Being explicit about values allows simple, at-a-glance correlation of exit codes to error types.

For example (this list is not meant to be exhaustive), the enum might be defined as follows:

```rust
/// Exit codes supported by the NotionFail trait.
pub enum ExitCode {
    /// No error occurred.
    Success = 0,

    /// An unknown error occurred.
    UnknownError = 1,

    /// An invalid combination of command-line arguments was supplied.
    InvalidArguments = 2,

    /// No match could be found for the requested version string.
    NoVersionMatch = 3,

    /// A network error occurred.
    NetworkError = 4,
}
```

Note that errors are grouped according to basic cause, with thought given to how external tooling might choose to react to the problem.  Doing so saves external tools from trying to parse Notion error messages in order to make these decisions.  In this example, an external tool receiving exit code 2 should absolutely not retry the operation (and may have a bug).  Exit code 3 indicates a potentially non-permanent error, but one which shouldn't be retried without first using another command to change Notion's current state.  Finally, exit code 4 indicates a kind of transient problem which might be desirable to retry after a short delay rather than immediately giving up entirely.

Customer-facing documentation will also need to be generated, either by hand or by using a purpose-made tool.  (Rustdoc alone is insufficient for this purpose, as it does not expose the enum's integer values.)  The documentation for the above example might look as follows.  (Note that the descriptions provided to the user is the same as the ones provided to developers.)

| Code | Description                                                    |
| ---: | :------------------------------------------------------------- |
|    0 | No error occurred.                                             |
|    1 | An unknown error occurred.                                     |
|    2 | An invalid combination of command-line arguments was supplied. |
|    3 | No match could be found for the requested version string.      |
|    4 | A network error occurred.                                      |

# Critique
[critique]: #critique

The primary downside introduced by this proposal is the loss of error granularity.  By grouping multiple errors into the same exit code, external tooling loses the ability to differentiate between those errors without capturing and parsing Notion's output.  However, it wasn't deemed realistic to provide a separate exit code for each error due to the limited nature of the former and the limitless potential of the latter.  🙂

Another issue is the communication of exit codes.  We want to publish the list as part of Notion's standard, user-facing documentation, but this requires either manually keeping the documentation in sync with the source or writing a tool to do so for us.  The alternative is to point interested parties directly to the Notion source, which is undesirable.

# Unresolved questions
[unresolved]: #unresolved-questions

- Is a process needed for 'vetting' new exit codes, to prevent duplication?  If so, is the RFC process appropriate?
- Are there specific, known error codes produced by e.g. Node, npm, or Yarn which should be avoided?  None of those projects themselves publish a list of exit codes…
