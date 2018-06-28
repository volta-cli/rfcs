- Feature Name: events_and_metrics
- Start Date: 2018-05-04
- RFC PR: (leave this empty)
- Notion Issue: (leave this empty)

# Summary
[summary]: #summary

This proposal describes the events that Notion generates and the API that it provides for plugins to consume these events for reporting metrics.

In brief:
For all commands and shims, Notion generates event objects for each major operation, and collects those events in an array.
Users can configure a plugin to process the array of events.
At the end of the execution of the command or shim, the plugin will be executed in a separate process, and will receive the array of events as JSON in stdin.
Notion will exit, and the plugin process will be allowed to run to completion independently in the background.
The plugin is responsible for generating metrics from the input events and reporting them appropriately.
If there is no plugin configured, the events will be discarded at the end of the command or shim.


# Motivation
[motivation]: #motivation

## Companies

Companies care about collecting metrics for their developers and tools, to discover issues and track productivity.
Specifically, executives and managers care about the metrics that can be generated from the events in Notion.
For example, how long it takes to build a project, or to update dependencies.

## OSS

For OSS, this is probably not a feature that will be regularly used.
By default, it should preserve privacy of the user.
The events will be generated internally, but nothing will be reported.


# Pedagogy
[pedagogy]: #pedagogy

## Normal users

For normal users, this should be transparent. There are no extra commands or options to learn.

## Plugin writers

For people who write plugins to report metrics, it should be easy to write a plugin.
The API should be well-documented, with simple example plugins.
The plugins should be language-agnostic.
Using a plugin in JS should be an interesting challenge (open question?).

# Details
[details]: #details


## speed

One of the overall constraints is that the Notion commands and shims should be as fast as possible, and not cause any unnecessary latency for the user.
Event reporting happens in a separate process in the background, so that any operations such as network calls and writing to disk do not block command execution.


## event types

There are three event types that can be generated for any operation:
- 'start' - beginning of the operation
- 'end' - end of the operation
- 'error' - an error was encountered when running the operation, containing the stack trace and other info

Events are generated for the overall execution of every command and shim, as well as internal operations of interest:
- downloading and installing node, yarn, etc.
- any operations that take time and could fail

The balance of how fine-grained these events should be will depend on implementation and feedback from users.


## format

Every event includes a type, timestamp, and name, and optional parameters specific to that event.

For example, when running the `node` shim, the array of events would look like:

```js
[
  {
    "event": "start"
    "timestamp": 1525301445485,
    "name": "node",
    "params": {
      "command_line": "node --version"
    }
  },
  {
    "event": "end"
    "timestamp": 1525301445578,
    "name": "node",
    "params": {
      "exit_code": 0
    }
  }
]
```

## implementation

Currently all commands and shims maintain an internal session.
This session will contain an array to keep track of all the events that happen during that command or shim.
Every event that is generated is added to the array of events.

At the end of the session, if an events plugin has been configured:
1. a child process is spawned with the command defined for the plugin
2. the events are serialized into an array of JSON objects
3. that JSON array is passed to stdin of the plugin process
4. Notion exits, letting the plugin process run to completion in the background

If a plugin has not been configured, then no process is spawned and the events are discarded.


## plugin configuration

The plugin command is specified in the project manifest:

```js
"notion": {
  // TODO better name
  "events_plugin": "python process_metrics.py"
}
```

One open issue is how to run a plugin written in JavaScript. This is discussed below in [unresolved questions]("#unresolved-questions).


# Critique
[critique]: #critique

## events API

One option that was considered was to start a separate process and send events to that process as they were generated.
That process would start the plugin in another process, and pass the events to it one at a time.
That design was overly complicated:
- if the process dies for some reason, have to handle restarting it
- since the connection will stay open, the events must be delineated, using something like JSON-ipc
- there would be three processes running for every command or shim

But, one advantage of that design is that if the Notion process dies and cannot be recovered (e.g. `kill -9`), there would at least be some events reported.
The current design waits until the end of the command to send all the events at once.
So if the Notion process cannot recover, then there will be no events for that.

## hooks into Notion

This design does not provide hooks into the Notion code, since plugins could introduce things like network calls and slow down execution.

## reporting outside of a project

This will not do any reporting outside of a project with a notion configuration in the manifest.
There could be a system-wide config for this, but I don't know if that is something we want to support.


# Unresolved questions
[unresolved]: #unresolved-questions

- How do we run plugins that are written in JavaScript? We need node to do that, which is a Notion shim, and we run into a chicken-and-egg problem.
- If the plugin process dies, how do we report that? Should this show something to the user?
- Should there be a system-wide Notion config file? Something in ~/.notion? If so, that is out of scope and should be a separate RFC.

