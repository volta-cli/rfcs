- Feature Name: helpful_error_messages
- Start Date: 2019-02-28
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

Improve error messages so that users are better able to understand and troubleshoot what went wrong.

# Motivation
[motivation]: #motivation

Our current error messages are often cryptic and unhelpful. We have, at last count, nearly 100 calls to `.unknown()` for external errors, effectively swallowing the underlying error and presenting a generic `an internal error occurred` message. This doesn't help the user understand what went wrong nor what they could do to fix the issue, leading to frustration with Notion. Where we do have error messages, they are often utilitarian and not as helpful as they could be. To improve the user experience and make Notion a delightful tool to use, we should improve those error messages so that they are more helpful for the users.

Additionally, when users report an error to the maintainers, it can sometimes be difficult to track down exactly where that error originated. By including a unique identifying code in every error, we will be more able to pinpoint the exact source of an error when a user reports an issue.

# Pedagogy
[pedagogy]: #pedagogy

Ideally, there should be nothing new we need to teach the users. The error messages will be more descriptive and helpful, and in the event that the user still can't solve the problem, the unique identifier will make it more straightforward for us to assist.

There may, however, be some additional education and documentation necessary to introduce concepts used by Notion that are now being exposed somewhat by the more descriptive error messages.

# Details
[details]: #details

After some short internal discussions, the general consensus is that the error messages should be conversational and provide the necessary context around what happened. However, we should try to avoid being too verbose as Notion is a CLI tool and we don't want long, informal error messages filling up logs when it isn't necessary. An example of an existing error message that provides the needed guidance without being too long is:

```
Global package installs are not recommended.

Consider using `notion install` to add a package to your toolchain (see `notion help install` for more info).
```

Beyond the error message itself, it would be a good idea to include a "Troubleshooting" section of the documentation website, with sections for each error. There, we could provide more context and additional steps to resolve the issue. We could then write out a link with each error to the documentation for that error, affording users the opportunity to dig in deeper if necessary.

The first phase of technical groundwork for improving the error messages was completed in notion-cli/notion#249. The additional work has several phases:

## Unique Identifiers

Update the existing error implementation to support a unique error code assigned to each error type. Additionally, we'll need to update the `Display` implementation to output that error code and the link to the online documentation about that error with the error message.

## Audit Existing Errors

We have a number of existing error messages of varying levels of helpfulness. We should update those to all be at the same level (and potentially split them into multiple errors if necessary).

## Create Error Messages for Unknown Errors

The bulk of the work for this RFC will be in creating new error messages (and codes) and replacing every call to `.unknown()` with a call to `.with_context()` that returns the appropriate error. As of the start of this RFC, there are ~80 uses of `.unknown()`, so this will be a significant undertaking, and likely best handled with a quest issue. For each error, we will need to determine the appropriate context and then craft an error message to explain to the user what went wrong and how best to fix it.

## Online Documentation

We will need to create the online documentation for all of the error messages, and keep it updated as new errors are added. For a day 0 implementation, the documentation could simply be a restatement of the error message, but going forward we should try to add additional context where possible. For example, a relatively common error is `No such file or directory (os error 2)` when trying to execute a command. This very often means that the command being executed couldn't be found in the `PATH`, so we should be able to include that additional information in the online troubleshooting documentation.

# Critique
[critique]: #critique

The primary point of critique is with the exact voice and style we want to use for the error messages. There are two extremes we could take, though this proposal is for doing a middle-ground that hopefully gets the benefits of each without significant drawbacks.

On the one hand, we could argue that since Notion is a CLI tool, and many of the commands (e.g. `npm run ...`) will be run by automated scripts, we should keep the error messages as concise as possible. This prevents up from filling up logs with human-readable text, while still providing feedback in the form of an error code and short description. The issue with this is that Notion will also be used very often in non-automated contexts. Putting an extra step between the user and useful feedback hurts the user experience and makes it less likely for the helpful information to actually be of use.

On the other hand, we could opt for extremely verbose error messages, in the vein of the compiler messages provided by [Elm](https://elm-lang.org). This has the benefit of being extremely friendly to the user in pointing them towards where their problem is, but is a lot of clutter. Additionally, while compile errors are a normal and expected part of development, errors in a tool like Notion should be rare and exceptional. While a compiler providing very verbose output to explain how to fix your code makes sense, it makes less sense for a tool that shouldn't be putting errors out anyway, to give that much detail all the time.

# Unresolved questions
[unresolved]: #unresolved-questions

- Once we have completed adding error messages to all the calls to `.unknown()`, should we remove the `.unknown()` helper method? It is a useful tool for rapid prototyping, but as Notion matures, it makes sense to require that external errors be handled in a way consistent with the rest of the project.
