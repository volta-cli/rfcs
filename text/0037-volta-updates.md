- Feature Name: volta_updates
- Start Date: 2019-08-01
- RFC PR: https://github.com/volta-cli/rfcs/pull/37
- Volta Issue: https://github.com/volta-cli/volta/issues/568

# Summary
[summary]: #summary

Provide a path for users to keep Volta up-to-date, ranging from notifications that updates are available to automatic background updates.

# Motivation
[motivation]: #motivation

We are constantly working to improve Volta, so we should make sure that those updates are made available to our users, especially those who don't keep a close eye on our releases. That way users are always able to take advantage of new functionality and improvements. We also want to support managed environments such as a corporate networks where updates can be automatically pushed onto users' machines.

# Pedagogy
[pedagogy]: #pedagogy

The main new concept for update functionality is the difference between Individual and Managed users:

- Individual users: The current model, where Volta is installed in the users' home directory and only available to that user.
- Managed users: For Enterprise environments, where Volta is installed by IT on users' machines, available to all users of the computer (Each user will still have their own data directory to manage their own tools). This will also likely be the case in many CI environments that are managed by IT, removed from the control of individual users.

For Individual users, the goal of this feature is to not require any education. The updates will be checked and made available through notifications or automatically applied, requiring nothing new from the user.

For Managed users, there will be a new command: `volta setup`. This command will make sure that a user's environment is set up to run Volta and the shims. This command should be idempotent, so that if a user accidentally runs it again, there are no adverse effects. Additionally, we will need to document what files an Enterprise needs to distribute in order to run Volta in a Managed environment (see [Updating the Binaries](#updating-the-binaries) for more information).

# Details
[details]: #details

For Volta updates, there are three distinct sets of changes that need to be considered: Updating the Binaries, Updating the Data Directory, and Updating the Profile.

## Updating the Binaries

The large majority of updates to Volta involve _only_ updating the binaries, without requiring updates to the Data Directory or Profile.

### Managed Users

For Managed users, updating the binaries will be handled by the team managing the distribution, making sure that they are always on the latest binaries. Specifically, we currently provide 2 binaries:

- `volta` - Provides the functionality of all `volta` commands
- `shim` - Provides the shimming behavior for all tools

Both of these binaries will need to be updated by the group managing the installs, made available on the PATH for all users, and be in the same directory as one another (for discoverability). Since the number of binaries that need to be provided could change in the future, we should also provide a `volta.manifest` file as part of the distribution. That file will list all of the binaries that need to be made available for Volta to work correctly.

### Individual Users

For Individual users, updating the binaries can be handled in 3 phases:

- First, we can provide a notification that the user's version is out-of-date, letting them know that it's time to upgrade. We can also include the platform-specific install instructions in that notification, making it easy for them to act on the notification. This will still require that the user manually update their system, but it will let them know when it is needed and provide guidance on how to affect that update.

- Next, we can implement a command `volta selfupdate` that checks for a new version, and if one is available, downloads and unpacks the binaries, replacing the existing ones. Then the binaries will use the new versions from that point on.

- Later, we can implement an automatic updater, which runs in the background and mirrors the effect of `volta selfupdate` automatically. This will result in Individual users having an experience that is effectively the same as Managed users: The binaries will be silently updated behind the scenes, without them having to take any action. This should be as atomic of a process as we can make it, so that the user is less likely to notice any strange behaviors because they are in the middle of an update.

For both of these cases, we want to make sure that the checking does not have an adverse effect on the performance of the tools, especially on the performance of shims. For that reason, we likely want to perform the checks and updates in a separate process or thread, so that the main thread can continue to run. This is especially important in offline or low-bandwidth situations, where the check for new versions may take a long time before ultimately failing, and we don't want the user to have to wait. Additionally, we want to make sure that any separate threads we spawn don't write output to the terminal, as we want to make sure that all output is handled in a deterministic way.

Additionally, for notifications, we don't want to show the notification every time the user runs a command, so we should limit the notifications to one per day so that we don't pester the user incessantly.

## Updating the Data Directory

Occasionally, there will be updates that require migrating the user's data directory (`VOLTA_HOME`) to a different layout as part of the update. Also, a new install can be considered a special-case of updating the data directory (from _nothing_ to _current_). Since both the Managed user case and the Individual automatic updater will result in the binaries being updated without any direct changes to the data directory, we will need to detect when changes are necessary and carry them out. The first of these changes will be the update that includes Managed environment support, since we will want to change how the files are laid out and remove some unnecessary files (`load.sh` and `load.fish`, for example).

We should have a way of determining what layout version the user is currently on, so it can be compared with the version that the running binary needs. We can likely write the layout version into a file and detect that on startup of either `volta` or a shim. We will then need to check that version on every invocation of `volta` or a shim, so we should make sure that the check for outdated layout is as fast as possible. If the user's layout is outdated, we should block while migrating the layout, since successful execution of the tool will depend on the data directory being in the expected layout. We should also provide feedback to the user about what is happening, probably with a progress spinner. Additionally, we will need to ensure that multiple calls with an outdated layout don't cause issues of different tools trying to update at the same time.

## Updating the Profile

Last, since Volta relies on having shims in the `PATH` to work, we need to update the user's terminal profile to ensure that the shims are available (`~/.bashrc`, `~/.profile` or similar). We should also be able to adapt an old profile block into a new one, in case things change about how we want `volta` to be called. Since migrations like that should be exceedingly rare, I don't think we will need a versioning system for the profile block, but we should be aware that may be needed in the future.

Since the user's profile is an area outside of Volta's direct control, we should only modify it when the user explicitly requests a change be made. To that end, we won't try to modify the profile as part of updating the data directory, but only when the user explicitly calls `volta setup`. This command will ensure that the data directory is up-to-date (through the above section), then modify the user's profile as necessary to ensure that Volta is available.

## Miscellaneous

### Installer

Under this model, the Individual user installer itself can be greatly simplified: It will download the correct binaries, unpack them into the `VOLTA_HOME` directory, and then call `volta setup` using the new Volta binary that was just unpacked. `volta setup` will then be responsible for building / updating the user's data directory and making the necessary profile modifications.

### Removal of `volta activate` and `volta deactivate`

To support Managed users, we want to allow `volta` to be called directly, instead of using a shell function wrapper. This has the benefit of allowing `which volta` to work as expected, however it comes with the downside of not supporting `volta activate` or `volta deactivate`. Those commands were added as an escape hatch during beta testing, and as we have solidified the experience of Volta, they have become less and less necessary, so at this point it's reasonable to remove them.

### Rename of Shim executable

Since Managed environments will involve distributing the binary files (`volta` and `shim`) to the users' machines, in a location that is available on the `PATH`, we should rename `shim` to `volta-shim` so it is clearly related to `volta`, since it will be on the `PATH` and we don't want to have possible collisions with any other binaries named `shim`.

### Refactor `path` Module

For Managed environments, the `volta` and `volta-shim` binaries will be in a different location than `VOLTA_HOME`, so the `path` module will need to be updated to take that into account. This is already the case for Windows, so much of the groundwork has already been laid, however it will need to be adapted to support Unix as well.

### Detecting Managed vs Individual Environment

Since the Individual install will remain the same, with all the files located within `VOLTA_HOME`, on Unix we can detect whether we are in a Managed or Individual environment by checking if the executable path is a child of `VOLTA_HOME` or not. This will allow us to conditionally activate the update notifier / automatic updater based on what mode the user is working under. For Windows environments, we will need to come up with a different method of detecting whether we are in a Managed or Individual environment, since even Individual environments will have the binaries separated from the `VOLTA_HOME` directory.

### Kill Switch

Since we will inevitably wind up in a state where an update can't be handled automatically, we should support some way to indicate that an update cannot be performed automatically and needs to be handled with some user intervention. As discussed in [Updating the Data Directory](#updating-the-data-directory), we will need some way of detecting the current update level that we can compare to the current level in the running binary in order to perform the update. One option for this kill switch would be to have the update level follow a SemVer specification, where any patch or minor change can represent an automatic update, while a major version bump will indicate a significant migration that needs some level of human interaction. This "update version" would be separate from the overall program version, so we can iterate both independently as necessary.

It will be important to include this feature in the first release of updates, since once those are released, we can't be sure that users did or did not get any specific release, meaning we could never reliably use it if it came in a subsequent update.

# Critique
[critique]: #critique

Earlier iterations of this RFC treated the update process mostly the same between Managed and Individual installs, however when discussing the overall design of the system to support both situations, it became clear that the two are fundamentally disconnected and will need to be treated separately. From there, it grew into the current proposal, where we support both modes and then can incrementally work towards a situation where the Individual users are updated automatically, resulting in them being functionally similar to Managed users.

## Incremental or Monolithic

One question is whether we should tackle the Individual update story incrementally, with the notifier followed by the automatic updater later on. Tackling it all at once would mean that we can present users with a polished, delightful update process from the first time it available. However, the full process will likely take a lot of effort to implement and in the interim there wouldn't be anything. Doing incremental updates will probably take more effort overall, but will allow us to give users a working experience and then progressively enhance it until we are at the final state. It's also likely that we will discover edge cases and workflow issues along the way that will inform our final design for the update experience, making it that much better when we reach the goal.

## New Command Name

Another area of critique is in the name of the new command. `volta setup` Seems to fit the use-case cleanly, since it's something you will likely run once and then never need to run again, but there are other potential options:

- `volta init`
- `volta bootstrap`
- `volta start`
- `volta activate` (Has a clear collision with the existing command that is proposed to be removed)
- `volta selfinstall`
- `volta self install`

## Detecting Environment Type

The current proposal detects the user's Environment using the location of the binary relative to `VOLTA_HOME`, however that would preclude us from using an Individual installer (with automatic updates) that uses a directory other than `VOLTA_HOME` for the binary files. Is there a different way we can detect which environment the user is working under?

## Windows

The Windows automatic update story is complicated, since the binaries are currently installed into the `Program Files` directory, and the user may or may not have edit access to that directory. Should we update the installer to unpack the files into the `LocalAppData` directory and then run `volta setup`, in the same way that the Unix install works? That would guarantee that the user has permissions to the binaries for automatic updates, and would align the Windows and Unix stories more closely.

# Unresolved questions
[unresolved]: #unresolved-questions

- Are there other update models we should implement along the way?
- Do we want to provide separate installers? One for the "Managed environment" case and one for the "individual user" case?
