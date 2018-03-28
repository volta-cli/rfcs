- Feature Name: system_node
- Start Date: 2018-03-26
- RFC PR:
- Notion Issue:

# Summary
[summary]: #summary

This is a high-level proposal, establishing how Notion should position itself relative to system installations of Node. Briefly: **Notion should recommend against being installed alongside system installations of Node.** On Unix operating systems, it's possible for Notion to offer a (non-default, opt-in) mode to forcibly override the system Node (with some caveats), but this should not be the recommended setup. Finally, this proposal also suggests future proposal ideas for helping users detect likely issues with an existing Notion installation, including other Node installations that may indicate conflicts.

# Motivation
[motivation]: #motivation

## System installs of Node are problematic

System installs of Node in global locations like `/usr/bin` or `/usr/local/bin` are problematic. They have historically had [permissions issues](https://givan.se/do-not-sudo-npm/), and the advice on how to properly configure a system Node installation is [confusing at best](https://github.com/npm/npm/issues/3139#issuecomment-13358582).

Global installs of Node also pose a security risk. Paths like `/usr/bin` are typically high precedence in a user's `PATH` variable, allowing for the possibility of accidental or even malicious shadowing of standard tools by third-party npm package executables. (As a relatively benign but still frustrating example, I have personally experienced a multi-hour debugging session where we eventually discovered a colleague's `which` had been overridden by the `node-which` package.)

Maintaining additional Node installations alongside a system Node is especially problematic. For one, it's impossible for the second installation to override the system Node without placing third-party binaries _extremely_ early in the `PATH`, which carries the same security risks mentioned above. For two, even in this scenario, the user experience of global npm package binaries gets muddled: a global binary installed with the system Node may still get picked up if it hasn't been installed as a global in the current Node.

For all the above reasons, **the best approach to using a Node version manager is to avoid having any other Node installations on the system**. Rather than attempting to hide the risks associated with co-installing along with a system Node, we should clearly recommend to users of Notion that if they have an existing system installation of Node, they should uninstall it first.

## Unix and Windows

In Unix OSes, we may want to consider still allowing the ability to opt into overriding a system installation. But since this incurs greater security risk and is generally less reliable, it should not be the recommended path. The installation steps should ask for confirmation and should recommend aborting and choose this as the default action.

In Windows, the system `Path` variable takes precedence over the user `Path` variable---not dissimilar from how Unix environments place `/usr/bin` towards the beginning of `PATH`. But unlike Unix, Windows provides no general mechanism to circumvent this (whereas Unix users can simply prepend new directories to the front of `PATH`). This suggests that the Windows design has a stronger expectation that system installations take precedence over user installations.

Also unlike Unix, Windows developers do not expect environment variables to be configured by terminal bootup scripts. Windows users should be able to expect use cases like executing Node from "Start | Run" or in IDEs to work, which would not work with terminal configuration scripts.

Rather than attempting to circumvent standard practice or provide partial solutions that don't work for all expected contexts, the Windows Notion installer should simply fail if a system Node is detected.

## Detecting system changes

Detecting a system Node at installation time is insufficient since users or system administrators could install another Node installation on the system at any time. So we also need to provide users with the tools to detect these kinds of issues at any time after Notion has already been installed.

# Pedagogy
[pedagogy]: #pedagogy

The simplest story to explain to users is: **don't install Notion alongside other Node installations**.

With override, we can soften this to a recommendation, and optionally allow overriding.

The installation UI provides one opportunity to explain this; it's also worth recommending on the Notion web site.

The idea that Notion installs the toolchain in the user's own space should not be too difficult to learn, since it's already the dominant model for many toolchain managers.

# Details
[details]: #details

## Installation

At installation time, the installer should check for:

- Existing `node`, `npm`, `npx`, or `yarn` commands in the path.
- Existing symlinks to global npm package binaries (which are often accidentally left around when a user attempts to manually uninstall Node).

In Unix, if any of these checks detects pre-existing Node artifacts, the installer displays a warning and presents the user with two options: `"Abort (recommended)"` and `"Force override (advanced)"`. The first, default option exits the installer with a message suggesting the user should try uninstalling those artifacts first before trying again. The second option installs Notion in "force override" mode (see below).

In Windows, the installer aborts unconditionally if any of these checks detects pre-existing Node artifacts, displaying a similar warning to the one in Unix.

## Force override installations

A "force override" install places the Notion executable directories at the beginning of the `PATH` environment variable to ensure that whatever executables Notion installs always take precedence over other Node installations. This mode is inherently riskier than a standard installation of Notion, since it means third-party npm packages may shadow standard Unix executables, and if a system installation of Node has global packages with executables, they may still show through to the user (in particular if the same globals have never been installed by Notion).

The diagnostic tools (see "Notion Doctor" below) probably should also include additional information reminding a user that they have a force-override installation, reminding them of the risks.

## `notion install --force-override`

We need a separate RFC detailing the design and workflow for installing global npm package executables. Assuming the command looks something like `notion install node-which`, in Unix we will probably also want to offer the ability to optionally force binaries to be installed in a high-precedence part of the `PATH`. (Since this is a more dangerous operation, it should require opt-in, and probably deserves terminology like "`force`" in the name of the flag, to signal to developers a similar intuition as dangerous operations in other popular development tools like `git`.)

In a force-override installation, every installation implicitly uses `--force-override`; if a user explicitly types this option it may indicate a confusion, and should probably signal a one-line warning.

## Notion Doctor

Notion should offer a thorough set of health checks, with detailed diagnostic output, available to users in the form of a `notion doctor` command (analogous to Homebrew's `brew doctor`).

In some OSes, it might even be possible to launch background/startup service that periodically check for problems and store them somewhere that Notion can quickly detect on startup.

## Startup checks

Notion shims are intended to be low-latency, so they should not involve expensive bootup logic. The diagnostic checks of Notion Doctor may get expensive. However, there may be some small number of high-value, low-cost checks we can do whenever a Notion shim launches. This could save users from running into trouble with common pitfalls.

# Critique
[critique]: #critique

We could simply choose the "force override" option by default, like `nvm` does. This proposal doesn't choose that approach for a few reasons:
- It's riskier for users, as described above.
- It's hostile to Windows.

We could choose not to even allow "force override" installations. In fact, we can experiment with not implementing it at first and see how well it works for users. But it's good to work out a plan for it so that users (who, for example, want to use Notion on systems where they don't have the permissions or ability to convince system administrators to remove a Node installation) can see that there is a plan in place to serve their needs.

We could choose not to have a `notion install --force-override` option. I think that doesn't need to be resolved for the rest of the aspects of this RFC, and can be considered more thoroughly in the discussion of a `notion install` RFC. I just wanted to include it here to show the full mental model of "force override" and where in the overall Notion design it potentially applies.

We could attempt to offer a "force override" model in Windows too, for a more complete consistency across OSes. (We might attempt to accomplish this with, for example, a system installer that creates a world-writeable parent directory, and directory junctions that can be mutated to point to different Node versions.) Since Windows seems to discourage letting users override system installations, and since "force override" is not the recommended mode, I'm suggesting we leave this out of the plan for now.

# Unresolved questions
[unresolved]: #unresolved-questions

Some issues we can work on during implementation:
- What should the detailed logic of the installer be?
- What should the warnings look like?
- What should the initial set of `notion doctor` checks be?
- What should the exact directory layout be for the different kinds of executables (`notion`, the core Node tools, global npm binaries) and other data (npm packages, Node versions)?

Some issues we will discover over time:
- What common conflicts should `notion doctor` attempt to detect?
