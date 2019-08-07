- Feature Name: volta_updates
- Start Date: 2019-08-01
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Provide a path for users to keep Volta up-to-date, ranging from notifications that updates are available to automatic background updates.

# Motivation
[motivation]: #motivation

We are constantly working to improve Volta, so we should make sure that those updates are made available to our users, especially those who don't keep a close eye on our releases. That way users are always able to take advantage of new functionality and improvements. We also want to support managed environments such as a corporate networks where updates can be automatically pushed onto users' machines.

# Pedagogy
[pedagogy]: #pedagogy

The goal for this feature is to not require any education for typical users. The updates will be checked and made available through notifications or automatically applied, requiring nothing new from the user.

For Enterprise / Advanced users that want to directly manage their updates, we will need to provide documentation of which Volta files need to be provided with a given update. We also need to provide the ability to disable any update notification / automatic updates, so that the updates can be managed. We also will need to document the new command that will be available to allow users to bootstrap their local environment from a globally managed Volta install.

# Details
[details]: #details

There are two distinct use-cases that need to be considered: Individual users and Enterprise users. For individual users, there are multiple approaches to updates, each building incrementally on the previous ones. This will allow us to implement straightforward approaches first, and then roll out more advanced update techniques later.

## Notifications

The most straightforward technique to implement would be a check of the current version against the latest available version accompanied by a notification to the user that their version is out-of-date. We can then provide a CTA that tells the user how to update (on Unix, run the install script `curl https://get.volta.sh | bash` again, or on Windows run the new installer). This still requires that the user manually update their system, but it lets the users know when that is necessary and provides guidance on how to perform that update.

For Enterprise situations where Volta is installed from a different source than the default public repository, we will need to provide a way to redirect the "latest version" check to a different URL _and_ a way to alter the CTA (as these users will likely have a different install flow than the default).

## Update Command

After we have notifications, we should provide a command such as `volta selfupdate` that provides a single entry point to updating the user's current installation with a newer one. Implementing this will require us to have a programmatic update process that can handle any migrations from version to version that are necessary. It may be that we provide a separate "updater" application and `volta selfupdate` downloads the newest version of that and runs it. We can then change the CTA in the out-of-date notifications to simply tell the user they need to run `volta selfupdate`, instead of directing them to the specific install steps.

For Enterprise users in this model, we will need to maintain the "latest version" redirect for the notification checker, but we will also need to provide a way to tell Volta where to find the updater and the associated update files, so that `volta selfupdate` works as expected.

## Automatic Updates

Once there is a programmatic updater that can handle the updates, the final step would be to provide an automatic update process that will keep Volta up-to-date in the background, without requiring user intervention. We will likely need a daemon to run separately and periodically check for updates, prepare the update, and implement it. Ideally we would keep the update process as atomic as possible, so that users won't be in a situation where the file structure changes but the binary hasn't yet been updated, or vice versa, which would cause transient, hard-to-diagnose bugs.

We should also provide the ability for users to opt-out of automatic updates and fall-back on the notifications. Again for Enterprise users we will need to have hooks to control the "latest version" check and the updater / update files.

## Constraints

Though these approaches are different, they all have a few important constraints that should be kept in mind.

### Performance

Checking for updates should not have a noticeable impact on runtime performance, especially on the performance of shims. One of the goals of Volta is to be as invisible as possible, so we shouldn't affect the user's workflow just to pause and check if updates are available. This is especially important in offline or low-internet situations. The result is that any checks we do will likely have to be asynchronous in a separate process or thread, so that the main thread can continue its work unimpeded.

### Obtrusiveness

If the user's version is out-of-date, we want to inform them of such, but not pester them. We should do our best to limit notifications to one per day at most, so that users aren't constantly seeing "Your version is out-of-date!" and getting frustrated that we're interrupting their workflow.

For the automatic updater, as discussed above, we should try to make the process as atomic as we can, so that there is minimal disruption to the user's workflow and they don't even notice that an update has occurred.

### Customizability

As discussed above, each update process should provide a means for users to redirect it to a different source, which will accommodate users who need to install from a private repository or are otherwise limited in their public internet access. We should also give users the ability to opt-out of updating / checking for updates, as we want to make sure the user has control of their own machine.

## Enterprise Users

For Enterprise users where updates will be managed by an external IT department, we want to provide the ability for updates to be pushed onto users machines. This will mean that the "install" process could happen completely separate from the user who is actually using Volta, so we need to provide the ability to manage the `VOLTA_HOME` directory disconnected from the installer.

### `volta init` Command

For the initial bootstrap / opt-in to Volta, we should provide a new command `volta init` that, when called, will bootstrap a users' `VOLTA_HOME` directory and make any necessary changes to their startup scripts to allow Volta to work for their environment. This command can be made robust enough to be a no-op if the environment is already created and potentially to write a configuration file that disables the single-user update mechanisms.

### Detecting Layout Updates

Since in managed environments, the update process will only involve updating the binaries, we will need a method to detect if the user is on an older layout schema and needs to be migrated. This check will need to be made fast, since it will have to run on every execution of Volta or a Shim. If we detect a migration is needed, we want to automatically handle that migration and report that to the user before continuing with the rest of the command that they ran.

# Critique
[critique]: #critique

The main question is whether we should tackle the entire update story at once, or incrementally. Tackling it all at once would mean that we can present users with a polished, delightful update process from the first time it available. However, the full process will likely take a lot of effort to implement and in the interim there wouldn't be anything. Doing incremental updates will probably take more effort overall, but will allow us to give users a working experience and then progressively enhance it until we are at the final state. It's also likely that we will discover edge cases and workflow issues along the way that will inform our final design for the update experience, making it that much better when we reach the goal.

Another alternative approach would be to merge the Single-user and Enterprise-user cases so that the upgrade process is the same, regardless of how you install Volta. This, however, would require us to provide a number of hooks so that Enterprise users can fundamentally alter the flow of updates. The two approaches are different enough that shoe-horning them together would be difficult and would require a lot more complexity in the configuration than having separate behaviors.

# Unresolved questions
[unresolved]: #unresolved-questions

- Are there other update models we should implement along the way?
- Do we want to provide separate installers? One for the "Managed environment" case and one for the "individual user" case?
- Do we need to include the layout update detection in the initial rollout, or can we delay that until we get close to having an update that requires a layout change?
