- Feature Name: helpful_error_messages
- Start Date: 2019-02-28
- RFC PR: [#31](https://github.com/volta-cli/rfcs/pull/31)
- Notion Issue: [#294](https://github.com/volta-cli/volta/issues/294)

# Summary
[summary]: #summary

Improve error messages so that users are better able to understand and troubleshoot what went wrong.

# Motivation
[motivation]: #motivation

Our current error messages are often cryptic and unhelpful. We have, at last count, nearly 100 calls to `.unknown()` for external errors, effectively swallowing the underlying error and presenting a generic `an internal error occurred` message. This doesn't help the user understand what went wrong nor what they could do to fix the issue, leading to frustration with Notion. Where we do have error messages, they are often utilitarian and not as helpful as they could be. To improve the user experience and make Notion a delightful tool to use, we should improve those error messages so that they are more helpful for the users.

Additionally, we often have extra information about an error that may be useful to help the user debug, however it would clutter the terminal if we always wrote it out. We should have a flag to allow the user to opt-in to more information on their errors, and write errors to a log so they can be easily looked up and submitted with bug reports.

Lastly, when users report an error to the maintainers, it can sometimes be difficult to track down exactly where that error originated. By including a unique identifying code in every error, we will be more able to pinpoint the exact source of an error when a user reports an issue.

# Pedagogy
[pedagogy]: #pedagogy

Ideally, there should be nothing new we need to teach the users about the errors. The error messages will be more descriptive and helpful, and in the event that the user still can't solve the problem, the unique identifier will make it more straightforward for us to assist.

We will, however, need to introduce the existence of the `--verbose` flag for showing more information on errors and the location of the log files so that users can use the error logs to troubleshoot as well as submit issues.

There may also be some additional education and documentation necessary to introduce concepts used by Notion that are now being exposed somewhat by the more descriptive error messages.


# Details
[details]: #details

## Style

After some short internal discussions, the general consensus is that the error messages should be conversational and provide the necessary context around what happened. However, we should try to avoid being too verbose as Notion is a CLI tool and we don't want long, informal error messages filling up logs when it isn't necessary. In order to be as useful as possible, without unneeded cluttering, the error messages should generally have 2 parts:

1. A short description of what went wrong. This should be kept to one or two lines, providing the context of what Notion was trying to do and what failed.
2. A call-to-action for likely next steps the user can take to resolve the issue.

```
Global package installs are not recommended.

Use `notion install` to add a package to your toolchain (see `notion help install` for more info).
```

Additionally, there are some style standards that we should follow with our error messages, to make them consistent and predictable:

### Line Lengths

Since the error messages often include dynamic segments (e.g. tool names), we shouldn't try to have a strict line-length requirement for the error messages. However, for usability, we should aim to not have any lines that are significantly more than 100 characters long. Also, the primary error description should fit into a single line, with any additional calls to action or troubleshooting tips on additional lines.

### Prefixes

When an error occurs in a tool shim, we should prefix any errors with `Notion error:`, to make it explicit that the error occurred as a result of Notion, and not the underlying tool. If the error occurred during the execution of the primary `notion` executable, adding `Notion error:` would be redundant, so we should instead use `error:` to introduce the error. This will provide a consistency with error reporting that will make it easy for users to understand what has happened.

### Color

We shouldn't go overboard with colors or additional styling, since having too many can distract from the main message we are trying to get across. However, we _do_ want errors to be visually distinctive, both on the command line and in log output, so we should make the prefix (described above) bold and red. Red is the common color for errors, so it will stand out as a visual indicator to the user that something went wrong.

### Additional Context

Most of our errors will be self-contained, issues with executing some part of the operation. However, when there is an error with parsing a configuration file (`hooks.toml`, `package.json`, etc.), we should provide additional context to the user, letting them know where the error is and highlighting the incorrect parts of the file. This should only occur for user-editable files, not for any of our internal files that are used for maintaining state.

## Verbose Mode

While we want to keep our messages concise by default, often there will be situations where we have additional information that would help the user troubleshoot. We want to provide a way for the user to access that additional information if they want it, while allowing other users to easily ignore it when they don't need more information.

### `--verbose` Flag

We should provide a general option to the `notion` command, `--verbose`, that will include additional error information in the event of a problem. This will allow the user to see more information directly in the terminal if they are running a `notion` command.

### Error Logs

We should also write any errors into a log file in the `NOTION_HOME` directory, so that the user can access the verbose error message even if they didn't execute the command with the `--verbose` flag. This will also be where verbose errors for shims can be accessed, as the shims should be seamless so we can't add a `--verbose` option to them.

## Examples

### Global package install is not recommended

<details>
  <summary>Dark background</summary>

  ![Global package error on dark background](/images/global_install.png)
</details>

<details>
  <summary>Light background</summary>

  ![Global package error on light background](/images/global_install_light.png)
</details>

### Pinning yarn without pinned node

<details>
  <summary>Dark background</summary>

  ![Pin yarn error on dark background](/images/pin_yarn_without_node.png)
</details>

<details>
  <summary>Light background</summary>

  ![Pin yarn error on light background](/images/pin_yarn_without_node_light.png)
</details>

### Version not found in registry

<details>
  <summary>Dark background</summary>

  ![Version not found on dark background](/images/version_not_in_registry.png)
</details>

<details>
  <summary>Light background</summary>

  ![Version not found on light background](/images/version_not_in_registry_light.png)
</details>

### Registry fetch error (default)

<details>
  <summary>Dark background</summary>

  ![Registry fetch error on dark background](/images/registry_fetch_short.png)
</details>

<details>
  <summary>Light background</summary>

  ![Registry fetch error on light background](/images/registry_fetch_short_light.png)
</details>

### Registry fetch error (verbose)

<details>
  <summary>Dark background</summary>

  ![Verbose registry fetch error on dark background](/images/registry_fetch_long.png)
</details>

<details>
  <summary>Light background</summary>

  ![Verbose registry fetch error on light background](/images/registry_fetch_long_light.png)
</details>

### Tool download error (default)

<details>
  <summary>Dark background</summary>

  ![Tool download error on dark background](/images/tool_download_short.png)
</details>

<details>
  <summary>Light background</summary>

  ![Tool download error on light background](/images/tool_download_short_light.png)
</details>

### Tool download error (verbose)

<details>
  <summary>Dark background</summary>

  ![Verbose tool download error on dark background](/images/tool_download_long.png)
</details>

<details>
  <summary>Light background</summary>

  ![Verbose tool download error on dark background](/images/tool_download_long_light.png)
</details>

## Implementation

Beyond the error message itself, it would be a good idea to include a "Troubleshooting" section of the documentation website, with sections for each error. There, we could provide more context and additional steps to resolve the issue. We could then write out a link with each error to the documentation for that error, affording users the opportunity to dig in deeper if necessary.

The first phase of technical groundwork for improving the error messages was completed in notion-cli/notion#249. The additional work has several phases:

### Add Verbose Flag and Logging

We will need to update our error reporting to handle the `--verbose` flag and output any additional information that we have available. We also will need to write those errors into a log file regardless of the existence of the flag.

### Audit Existing Errors

We have a number of existing error messages of varying levels of helpfulness. We should update those to all be at the same level (and potentially split them into multiple errors if necessary).

### Create Error Messages for Unknown Errors

The bulk of the work for this RFC will be in creating new error messages (and codes) and replacing every call to `.unknown()` with a call to `.with_context()` that returns the appropriate error. As of the start of this RFC, there are ~80 uses of `.unknown()`, so this will be a significant undertaking, and likely best handled with a quest issue. For each error, we will need to determine the appropriate context and then craft an error message to explain to the user what went wrong and how best to fix it.

### Unique Identifiers

Update the existing error implementation to support a unique error code assigned to each error type. Additionally, we'll need to update the `Display` implementation to output that error code and the link to the online documentation about that error with the error message.

### Online Documentation

We will need to create the online documentation for all of the error messages, and keep it updated as new errors are added. For a day 0 implementation, the documentation could simply be a restatement of the error message, but going forward we should try to add additional context where possible. For example, a relatively common error is `No such file or directory (os error 2)` when trying to execute a command. This very often means that the command being executed couldn't be found in the `PATH`, so we should be able to include that additional information in the online troubleshooting documentation.

# Critique
[critique]: #critique

## Voice

The primary point of critique is with the exact voice we want to use for the error messages. There are two extremes we could take, though this proposal is for doing a middle-ground that hopefully gets the benefits of each without significant drawbacks.

On the one hand, we could argue that since Notion is a CLI tool, and many of the commands (e.g. `npm run ...`) will be run by automated scripts, we should keep the error messages as concise as possible. This prevents us from filling up logs with human-readable text, while still providing feedback in the form of an error code and short description. The issue with this is that Notion will also be used very often in non-automated contexts. Putting an extra step between the user and useful feedback hurts the user experience and makes it less likely for the helpful information to actually be of use.

On the other hand, we could opt for extremely verbose error messages, in the vein of the compiler messages provided by [Elm](https://elm-lang.org). This has the benefit of being extremely friendly to the user in pointing them towards where their problem is, but is a lot of clutter. Additionally, while compile errors are a normal and expected part of development, errors in a tool like Notion should be rare and exceptional. While a compiler providing very verbose output to explain how to fix your code makes sense, it makes less sense for a tool that shouldn't be putting errors out anyway, to give that much detail all the time.

## Style

An additional point of critique is with the specific style used for the error messages. This includes:

### Line Length

We could have a fixed requirement that lines be a specific length and no longer, however this would require the error message author to know every possible value that a dynamic segment could take. Additionally, if the possible values of that segment change in the future, the error message would need to be updated to support the fixed line length. This could also lead to lines that appear too short, as the break needs to be done to accommodate a potentially longer dynamic segment.

### Color

We could make an argument for using more color, perhaps making the entire error description red (which is the approach used by [ember-cli](https://ember-cli.com/)). This would make the error even more explicit and visually distinctive. However, red on black (which is a common terminal background) can be difficult to read, and the goal is for the error messages to be readable and helpful. Additionally, having the message be red but then the calls to action in white provides a contrast that makes it somewhat ambiguous if the text is meant to supplement the error.

# Unresolved questions
[unresolved]: #unresolved-questions

- Once we have completed adding error messages to all the calls to `.unknown()`, should we remove the `.unknown()` helper method? It is a useful tool for rapid prototyping, but as Notion matures, it makes sense to require that external errors be handled in a way consistent with the rest of the project.
