# Notion RFCs

A Notion RFC ("Request For Comments") is a proposal for a significant change to the Notion design or architecture.

Many changes, such as bug fixes and documentation improvements, can be implemented and reviewed via the normal GitHub pull request workflow.

Some changes are substantial, and we ask that these be put through a bit of design process in order to build consensus amongst the Notion community.

The RFC process is intended to provide a consistent, well-considered path for features and changes to make it into the project, so that all stakeholders can feel confident about the direction of the project.

# Table of contents

  - [When you need an RFC](#when-you-need-an-rfc)
  - [The process](#the-process)
  - [License](#license)

# When you need an RFC

You need an RFC if you intend to make substantial changes to Notion, or the RFC process itself. What constitutes a "substantial" change is evolving based on community norms, but may including the following:

  - Any change to the command-line interface (including command syntax and behavior).
  - Changes to the configuration formats.
  - Changes to the plugin API.
  - Removing features.

Some changes do not require an RFC:

  - Rephrasing, reorganizing, refactoring, or otherwise "changing shape does not change meaning."
  - Additions that strictly improve objective, numerical criteria (e.g., performance improvements, removing compiler warnings from the builds, better platform coverage, more parallelism, trapping more errors, etc).
  - Additions only likely to be _noticed_ by other developers of Notion, invisible to users of Notion.

If you submit a pull request to implement a new feature without going through the RFC process, we may politely ask you to submit an RFC first and close the pull request.

# The process

In short, getting a major feature added to Notion requires an RFC to be merged into the RFC repository. At that point the RFC is "active" and may be implemented with the goal of eventual inclusion into Notion.

The steps are:

- Fork the [RFC repository](https://github.com/notion-cli/rfcs).
- Copy `0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is descriptive; don't assign an RFC number yet).
- Fill in the RFC. Put care in the details: RFCs that do not present convincing motivation, demonstrate understanding of the impact of the design, or are disingenuous about the drawbacks or alternatives tend to be poorly received.
- Submit a pull request. As a pull request, the RFC will receive design feedback from the larger community, and the author should be prepared to revise it in response.
- Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments. Feel free to reach out to the project maintainers to get help identifying stakeholders and obstacles.
- The project maintainers will discuss the RFC pull request, as much as possible in the comment thread. Offline discussion will be summarized on the pull request comment thread.
- At some point, the project maintainers will announce a proposed conclusion and tag the RFC with a `final comment period` label, giving the community a last opportunity to discuss the proposed conclusion.
- Once an RFC enters its final comment period, if the discussion appears to have stabilized, the maintainers will finalize the decision and either merge or reject the RFC pull request.
- On the other hand, if there are significant new constraints or ideas being surfaced, the maintainers will decide to remove the `final comment period` label and allow discussion to continue.

# License

This repository is licensed under a [BSD 2-clause license](https://github.com/notion-cli/rfcs/blob/master/LICENSE).

# Contributions

Unless explicitly stated otherwise, any contribution intentionally submitted for inclusion in the work by any contributor to this repository shall be licensed by the [BSD 2-clause license](https://github.com/notion-cli/rfcs/blob/master/LICENSE) included in the repository, without any additional terms or conditions.
