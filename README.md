# `AbortSignal.any()`

## Authors

- [Scott Haseley](https://github.com/shaseley)

## Participate

- [Issue Tracker](https://github.com/shaseley/abort-signal-any/issues) for
  this repo
- Associated [DOM issue](https://github.com/whatwg/dom/issues/920)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Use Cases](#use-cases)
- [Userland Solutions](#userland-solutions)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [`AbortSignal.any(signals)`](#abortsignalanysignals)
- [`AbortSignal.any([])`](#abortsignalany)
- [Key scenarios](#key-scenarios)
  - [Chaining Signals](#chaining-signals)
  - [Adding a Timeout](#adding-a-timeout)
- [`TaskSignal` APIs](#tasksignal-apis)
  - [`TaskSignal.any(signals)`](#tasksignalanysignals)
  - [`TaskSignal.any([])`](#tasksignalany)
  - [`TaskSignal.any(abortSignals, {priority: String})`](#tasksignalanyabortsignals-priority-string)
  - [`TaskSignal.any(abortSignals, {priority: taskSignal})`](#tasksignalanyabortsignals-priority-tasksignal)
  - [`TaskSignal.any([], {priority: String})`](#tasksignalany-priority-string)
  - [`TaskSignal.any([], {priority: taskSignal})`](#tasksignalany-priority-tasksignal)
- [Prior Art](#prior-art)
- [Detailed Design Discussion](#detailed-design-discussion)
  - [API Shape](#api-shape)
    - [Follow-mutability](#follow-mutability)
    - [Exposure through `AbortSignal` vs. `AbortController`](#exposure-through-abortsignal-vs-abortcontroller)
    - [Decision](#decision)
  - [Constructor vs. Factory Method](#constructor-vs-factory-method)
  - [Should this work with a single signal?](#should-this-work-with-a-single-signal)
  - [How does the API handle null signals or empty arrays?](#how-does-the-api-handle-null-signals-or-empty-arrays)
  - [Memory Management/Leaks](#memory-managementleaks)
- [Considered Alternatives](#considered-alternatives)
  - [Other Shapes](#other-shapes)
  - [Other Names](#other-names)
    - [`AbortSignal.race(signals)`](#abortsignalracesignals)
    - [`AbortSignal.<bikeshed>`](#abortsignalbikeshed)
  - [Alternative or Additional APIs](#alternative-or-additional-apis)
    - [`AbortController.prototype.close()`](#abortcontrollerprototypeclose)
    - [Weak Event Listeners](#weak-event-listeners)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [Acknowledgements](#acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Aborting ongoing async work in response to user actions has user-facing
benefits, e.g. decreasing resource consumption due to wasted cycles (battery)
and packets (battery, data). Newer async APIs are often
[designed](https://w3ctag.github.io/design-principles/#aborting) to accept an
`AbortSignal` for cancellation, but what happens when the abort could come from
*multiple* sources?

Some async operations may be aborted due to a variety of reasons, e.g. due to
timeout or user cancellation. And while nothing prevents an `AbortController`'s
`abort` method from being invoked for multiple reasons, combining signals
enables *separate layers of control*, where the scope of cancellation may vary.

Since `AbortSignal`-accepting APIs only accept a single signal, combining
signals currently needs to be done in userland using event listeners, but this
is complicated by [challenges around memory
leaks](https://github.com/whatwg/dom/issues/920).

To solve this problem, we propose **`AbortSignal.any(signals)`**. This API
creates an `AbortSignal` that is aborted when any of the signals passed to it
are aborted.  This makes it easy to combine signals, and by allowing the UA to
maintain these relationships, the UA can free related resources when safe to do
so.

## Use Cases

1. Combining a top-level signal (e.g. entire task) with a finer-grained-signal
   (e.g. subtask). For example, switching the view in a single page app (tab
   click, etc.) might cancel all ongoing async operations, but scrolling that
   same view might only cancel a subset of the work (e.g. scrolling an infinite
   list).

2. Combining an `AbortSignal` with a timeout. For example, an `AbortSignal`
   might be used to cancel all ongoing fetches in response to user input (e.g.
   user clicks cancel), but the same fetches also might be canceled by a
   timeout.

See also [the discussion](https://github.com/whatwg/dom/issues/920) that
motivated this proposal.

## Userland Solutions

It's possible to chain signals together using using event listeners:

```js
const topLevelController = new AbortController();
{
  const subtaskController = new AbortController();
  topLevelController.signal.addEventListener('abort', () => {
    subtaskController.abort();
  });
}
```

But care must be taken to avoid memory leaks. In this example,
`subtaskController` and its signal will remain alive as long as
`topLevelController.signal` remains alive. For long-lived top-level signals,
this can be problematic, especially if a lot of signals are linked. This can be
fixed by removing the event listener on the parent signal when
`subtaskController` either aborts or never will (e.g. when it goes out of
scope), but this is non-trivial to implement in userland.

## Goals

- Create an ergonomic API that enables combining `AbortSignal`s and avoids the
  memory leaks inherent to the `addEventListener` userland approach
- Replace the `AbortSignal` [*follow
  algorithm*](https://dom.spec.whatwg.org/#abortsignal-follow) with a new
  primitive to be used in existing specs (fetch) and in specifiying this API

## Non-goals

- It is a non-goal to provide [additional
  primitives](#alternative-or-additional-apis) for userland signal memory
  management, e.g.  `AbortController.prototype.close()`
  or *weak event listeners*

## `AbortSignal.any(signals)`

`AbortSignal.any()` returns a new `AbortSignal` that will be aborted if *any*
of the provided signals are aborted.

```js
const controllers = [new AbortController(), new AbortController()];
// Combine the signals of |controllers| into a new signal.
const signal = AbortSignal.any([controllers[0].signal, controllers[1].signal]);

// Randomly pick one of the two controllers to abort.
const index = Math.floor(Math.random() * 2);
controllers[index].abort();

// Always true since |signal| follows both of the controllers' signals.
console.log(signal.aborted);  // true.
```

## `AbortSignal.any([])`

If an empty array is passed to `AbortSignal.any()`, the signal it returns [will
never be aborted](#how-does-the-api-handle-null-signals-or-empty-arrays):

```js
const signal = AbortSignal.any([]);

...

console.log(signal.aborted);  // Always false.
```

## Key scenarios

### Chaining Signals

This example shows how to chain a top-level `AbortSignal` passed to a function
with a signal for a specific subtask. The scenario is a page that displays
information in tiles, with each tile being individually abortable (e.g. through
the user closing it) or aborted en masse (e.g. the user closing the whole
panel).

```js
// Load an individual tile, aborting the operation if the tile's signal or the
// parentSignal is aborted.
async function loadTile(parentSignal, tile) {
  // In this example, |tile| has a signal tied to its close UI.
  const signal = AbortSignal.any([parentSignal, tile.signal]);

  let contents = await getTileContents(tile, signal);

  await displayTileContents(tile, contents, signal);

  tile.setLoaded(true);
}
```

### Adding a Timeout

Timeouts created with
[`AbortSignal.timeout()`](https://dom.spec.whatwg.org/#dom-abortsignal-timeout)
can be passed to `AbortSignal.any()` to create a signal that is aborted either
through a timeout or user-initiated cancellation.

```js
// Picking up from the previous example, this is a skeleton for getting data
// for the tile.
async function getTileContents(tile, signal) {
  const signalWithTimeout = AbortSignal.any([signal, AbortSignal.timeout(10_000)]);
  const response = await fetch(tile.url, {signal: signalWithTimeout});
  return response.json();
}
```

## `TaskSignal` APIs

[`TaskSignal`](https://wicg.github.io/scheduling-apis/#sec-task-signal)
inherits `AbortSignal.any()` since `TaskSignal` subclasses `AbortSignal`.  It
would be natural for `TaskSignal.any()` to return a `TaskSignal`, which is the
approach we take here. Since `TaskSignal`s have a *priority component* in
addition to an *abort component*, we also need to consider **how to instantiate
priority**.

Unlike combining multiple abort sources, we don't know of any use cases for
combining multiple priority sources, and do not think this would be a common or
recommended practice. As such, the requirements for combining `TaskSignal`s
differ: we want to allow combining **multiple abort sources** with **at most one
priority source**.

To achieve this, `TaskSignal.any(signals)` will take an additional optional
options bag with a priority parameter: `TaskSignal.any(signals, {priority})`,
where *priority* can be either a [priority
string](https://wicg.github.io/scheduling-apis/#sec-task-priorities) or
`TaskSignal`. The sections that follow describe the various combinations of
parameters and illustrate the flexibility this provides.

### `TaskSignal.any(signals)`

`TaskSignal.any(signals)` returns a `TaskSignal` whose *abort component* is
composed of all the abort components of the *signals* and whose priority is the
*default priority*
(["user-visible"](https://wicg.github.io/scheduling-apis/#dom-taskpriority-user-visible)).
This signal is interchangeable with an `AbortSignal` returned by
`AbortSignal.any(signals)` in terms of abort and priority (as used by
[`scheduler.postTask()`](https://wicg.github.io/scheduling-apis/#sec-scheduler)).

```js
// |signal| is a TaskSignal.
const signal = TaskSignal.any([signal1, signal2]);
console.log(signal.constructor.name);    // 'TaskSignal'

// This task will be scheduled at default ('user-visible') priority.
scheduler.postTask(task, {signal});

// Assume |signal1| is tied to |controller1|. |signal| inherits abort from
// |signal1| and |signal2|, so the following will cause |signal| to be aborted.
console.log(signal.aborted);  // false;
controller1.abort();
console.log(signal.aborted);  // true;
```

### `TaskSignal.any([])`

Similar to `AbortSignal.any([])`, this returns a `TaskSignal` that will not be
aborted. Like `TaskSignal.any(signals)`, the priority is the *default priority*.

```js
const signal = TaskSignal.any([]);

// Scheduled at default ('user-visible') priority.
scheduler.postTask(task, {signal});

...

console.log(signal.aborted);  // Always false.
```

### `TaskSignal.any(abortSignals, {priority: String})`

This variant of `TaskSignal.any()` returns a `TaskSignal` with a fixed priority
specified by a [priority
string](https://wicg.github.io/scheduling-apis/#sec-task-priorities), which
will be aborted if any of *abortSignals* are aborted.

```js
// |signal| inherits abort from |signal1| and |signal2|, but has 'background'
// priority.
const signal = TaskSignal.any([signal1, signal2], {priority: 'background'});
scheduler.postTask(task, {signal});

...

// Assume |signal1| is tied to |controller1|.
console.log(signal.aborted);  // false.
controller1.abort();
console.log(signal.aborted);  // true.
```

### `TaskSignal.any(abortSignals, {priority: taskSignal})`

This variant of `TaskSignal.any()` returns a `TaskSignal` whose priority
follows the priority of *taskSignal*, i.e. its priority is initially set
`taskSignal.priority` and is changed if `taskSignal.priority` changes. The
signal returned will also be aborted if any of *abortSignals* are aborted, just
like `TaskSignal.any(abortSignals)`.

**Note**: the *priority* option must be a `TaskSignal`, not an `AbortSignal`.

```js
// Represents a top-level signal.
const parentController = new AbortController();
const parentSignal = parentController.signal;

...

const controller = new TaskController();

// |signal| follows |parentSignal| and |controller.signal| for abort and
// |controller.signal| for priority.
const signal = TaskSignal.any(
  [parentSignal, controller.signal], {priority: controller.signal});

console.log(signal.priority);  // "user-visible"
controller.setPriority("background");
console.log(signal.priority);  // "background"

// Some time later...
console.log(signal.aborted);  // false
parentController.abort();
console.log(signal.aborted);  // true
```

### `TaskSignal.any([], {priority: String})`

If given an empty array of signals for abort, this API returns a signal that
will never be aborted. When combined with a priority string, the API returns a
fixed priority `TaskSignal` that will never be aborted.

```js
const signal = TaskSignal.any([], {priority: 'background'});

...

console.log(signal.priority);    // Always 'background'.
console.log(signal.aborted);     // Always false.
```

### `TaskSignal.any([], {priority: taskSignal})`

Similarly, this variant of `TaskSignal.any()` returns a signal that will never
be aborted and whose priority follows *taskSignal*:

```js
const parentController = new TaskController();
const parentSignal = parentController.signal;

const signal = TaskSignal.any([], {priority: parentSignal});

console.log(signal.priority);      // 'user-visible'
parentController.setPriority('background');
console.log(signal.priority);      // 'background'

console.log(signal.aborted);     // false.
parentController.abort();
console.log(signal.aborted);     // Still false.
```

## Prior Art

.NET has
[CancellationTokenSource.CreateLinkedTokenSource](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource.createlinkedtokensource?view=net-6.0),
which enables combining `CancellationTokens` (their version of `AbortSignal`).

In Go, `Contexts` carry a cancellation signal. [Derived
contexts](https://go.dev/blog/context) are analogous to combining
`AbortSignal`s: "when a `Context` is canceled, all `Context`s derived from it
are also canceled."  [This
example](https://go.dev/doc/database/cancel-operations) shows deriving a
context to cancel a database operation due to either a timeout or HTTP
disconnect/error. There's also an [open
issue](https://github.com/golang/go/issues/36503) about adding a new explicit
API for merging contexts.

## Detailed Design Discussion

### API Shape

There are two primary design questions that led to this API's proposed shape:

1. Should new abort sources be allowed to be *added* to existing signals
   (*follow-mutability*)?
1. Should the API be exposed on `AbortSignal` or `AbortController`?

These questions lead to four general shapes:

| Exposed On | Mutable Signal? | API (Class) |
| ---- | ---- | ---- |
| `AbortSignal` | No | `AbortSignal.any(signals)` |
| `AbortController` | No |`new AbortController(signals)` |
| `AbortSignal` | Yes | `AbortSignal.prototype.follow(signals)` |
| `AbortController` |Yes | `AbortController.prototype.follow(signals)` |


#### Follow-mutability

This design choice impacts the API's **functionality**: if allowed, signals
passed to APIs like `fetch()` can be updated after invoking them, i.e. existing
signals can *gain* abort sources after creation.

***Is this needed?***

*Use Cases*: The key scenarios described above can be easily solved without
mutating signals, and we aren't aware of use cases that require this
functionality. As long as a signal does not need to be updated after creation,
this functionality is not required.

*Prior Art*: Follow-mutable signals are not supported by the prior art that
we're aware of. .NET exposes this functionality through a [factory
method](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource.createlinkedtokensource?view=net-6.0),
and sources must be specified at construction time. Go supports this
functionality through deriving new contexts from a parent and/or timeout at
construction time.

*Specs*: We [audited
usage](https://docs.google.com/document/d/1Waqli90SkEKBBu4hM5o6IpVKbLT8fy55WTGh7VhMEyI/edit?usp=sharing&resourcekey=0-noad-L-TnHpy1tIi2Hxr_w)
of the [*follow algorithm*](https://dom.spec.whatwg.org/#abortsignal-follow)
and [*signal abort*](https://dom.spec.whatwg.org/#abortsignal-signal-abort) in
specs and Chromium and found [one
instance](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/cache_storage/cache.cc;drc=0cfd786513f074a435316fd41866db6255b0587b;bpv=1;bpt=1;l=1068?q=AbortSignal::Follow&ss=chromium)
where modifying an existing signal simplifies things &mdash; although this is
not specced to use *follow* and could be rewritten to avoid this pattern. In
other cases, using `AbortSignal.any()` &mdash; or adding a *clone* algorithm
&mdash; would suffice.

***`AbortSignal` vs. `AbortController` for follow-mutability***

`AbortController` and `AbortSignal` were designed such that control of a signal
is separate from the signal itself. Exposing follow-mutability through
`AbortSignal` would **break the controller/signal separation** since any code
with access to the signal could abort it. This caused us to eliminate the
`AbortSignal.prototype.follow` class of APIs, but we could be expose
follow-mutability through `AbortController`:

```js
function task(parentSignal) {
  const controller = new AbortController();
  controller.follow(parentSignal);
  const childSignal = controller.signal;
}
```

***Added complexity***

Follow-mutability would add a non-trivial amount of complexity. Memory
management and signal relationship tracking is more complex since signals might
acquire abort sources after creation, potentially limiting memory
optimizations. For example, the *signal graph*[^1] can be flattened to a depth of 1
with follow-immutability, simplifying and optimizing memory management.

There is also additional complexity inherent to exposing the API through
`AbortController`, as discussed in the next section.

[^1]: A signal graph is a directed graph representing the relationships between
signals, such that there is an edge from A to B if B follows A. Optimizations
based on this graph will be explored in a subsequent design doc.

#### Exposure through `AbortSignal` vs. `AbortController`

Exposing a signal combinator API through `AbortController` requires a
controller to be created for every composite signal &mdash; whether or not it
is necessary. Consider for example combining a timeout with an existing
top-level *parent signal*:

```js
function task(parentSignal) {
  // Option 1: AbortController constructor.
  // |controller| is unused except to get its signal.
  const controller = new AbortController([parentSignal, AbortSignal.timeout(1000)]);
  const combinedSignal =  controller.signal;

  // Option 2: With AbortController.prototype.follow().
  // |controller| is unused except to get its signal.
  const controller = new AbortController();
  controller.follow([parentSignal, AbortSignal.timeout(1000)];
  const combinedSignal =  controller.signal;

  // Option 3: With AbortSignal.any().
  const combinedSignal = AbortSignal.any([parentSignal, AbortSignal.timeout(1000)]);
}
```

`AbortSignal.any()` is clearer for this use case, as the intent is obvious and
it does not create unnecessary objects. When an additional controller is
needed, it can be created and its signal can be combined with others, making
the intent clear:

```js
function task(parentSignal) {
  const controller = new AbortController();
  const combinedSignal = AbortSignal.any(
    [parentSignal, controller.signal, AbortSignal.timeout(1000)]);
}
```

***Prior Art***

.NET's approach is equivalent to exposing the API through `AbortController`,
specifically through a factory method that returns an `AbortController` whose
signal.

Like .NET, Go also creates an independent cancellation source
([`CancelFunc`](https://pkg.go.dev/context#CancelFunc)) for the derived
context. But the [documentation](https://pkg.go.dev/context#WithTimeout)
recommends invoking the cancel function after done with the context to free
resources, which is different from how `AbortController` is typically used. If
cancellation is not actually needed, the `CancelFunc` could be ignored using a
blank identifier.

#### Decision

**Create follow-immutable signals.** Based on known use cases and prior art,
it's not clear that follow-mutability is needed. Making the API
follow-immutable is conceptually simpler and simplifies memory management,
while still covering known use cases. Should use cases arise that require
mutability, developers can still achieve this in other ways, e.g. an internal
API that invokes `controller.abort()` for the associated controller.

**Expose the API through AbortSignal.** Exposing the API through `AbortSignal`
is cleaner since it does not conflate controllers and signals &mdash; it simply
combines signals. If another controller is needed as an abort source, it can be
created and its signal can be included in the array of signals. While this is a
departure from .NET's approach, it does not limit functionality.

### Constructor vs. Factory Method

`AbortSignal` currently has a private constructor and `AbortSingal` objects can
only be be created indirectly through an `AbortController` or `AbortSignal`
factory methods. We don't want to change how such existing`AbortSignal`s are
constructed, so exposing a combinator through an `AbortSignal` constructor is
not a great conceptual fit since would *only* create composite signals (and not
the *constituent* signals). It might make sense if we were subclassing
`AbortSignal`, but that is not the approach we take here.

Exposing a signal combinator through a factory method is also consistent with
how other `AbortSignal`s are created, e.g. `AbortSignal.timeout()`, and is
[analogous](#abortsignalracesignals) to `Promise` combinators (i.e.
`Promise.any()`).

### Should this work with a single signal?

```js
const controller = new AbortController();
const signal = controller.signal;
// Is this okay?
const copy = AbortSignal.any([signal]);
```

Sure! First of all, it's not harmful and matches the behavior of
`Promise.any()`.  Second, this can be useful to add local state without
modifying the original signal (`AbortSignal.any()` always returns a new
signal). Finally, this is exactly how the signal portion of [Request
cloning](https://fetch.spec.whatwg.org/#dom-request-clone) works in the fetch
spec.

### How does the API handle null signals or empty arrays?

```js
const controller = new AbortController();
const signal = controller.signal;

// Should this throw an exception?
AbortSignal.any([signal, null]);
// What about this?
AbortSignal.any([null]);
// What about this?
AbortSignal.any([]);
```

The current thinking is that null/undefined signals will result in an error and
that passing an empty array [will return a signal that won't
abort](#abortsignalany). Null/undefined signals could be skipped over, but
since there is no way to follow such signals, so throwing an error seems
reasonable. On the other hand, an empty array is not necessarily invalid, but
corresponds to the case when there is nothing to combine. Intuitively,
combining zero signals results in a signal that won't abort. While we could
require the array to be non-empty, allowing an empty array makes the API more
flexible.

### Memory Management/Leaks

Does anything about this API help or hurt with memory management? The signals
this API creates are *follow-immutable*, meaning they can’t be made to follow
more signals after they're created. This is a nice property for memory
management since we know a signal's children can unfollow it when all of its
parents are in a settled state. This would not be the case for exposing the
[follow algorithm](https://dom.spec.whatwg.org/#abortsignal-follow) directly.

## Considered Alternatives

### Other Shapes

`AbortSignal.any(...signals)` was also considered, but using an array/iterable
matches `Promise.any()`, and that consistency is desirable.

### Other Names

#### `AbortSignal.race(signals)`

Both
[`Promise.any()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/any)
and
[`Promise.race()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race)
return a promise that is dependent on the state of its component promises, so
either might be a good analogy for the API under discussion. Both of these
promise APIs will fulfill the resulting promise if the first dependent promise
that settles is fulfilled. The difference is how they handle rejected promises.

Promises can be in [three
states](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise):
pending, fulfilled, or rejected. We can also think of `AbortSignal`s as being in
these three states, mapping to promise states as follows:

1. *Pending*: the signal is not aborted but might become aborted
1. *Fulfilled*: the signal is aborted
1. *Rejected*: the signal will not be aborted (e.g. associated with an
   `AbortController` that has been GCed)

`Promise.race()` races fulfillment and rejection, meaning the state of the
promise the API returns will match the state of the first dependent promise
that settles. The promise returned by `Promise.any()` is only rejected if all of
its dependent promises are rejected. Given the definition of rejection above
for `AbortSignal`s, the behavior of the `AbortSignal` API under discussion matches
that of `Promise.any()`: **the resulting signal cannot be considered rejected until
all of its component signals will not abort**.

So given this mapping &mdash; and specifically the definition of signal
rejection &mdash; `Promise.any()` is a better analogy because of how it handles
rejection.

#### `AbortSignal.<bikeshed>`

There are a bunch of verbs that probably work fine (`combine()`, `link()`,
etc.), but the current name was chosen because of the strong `Promise.any()`
analogy.

### Alternative or Additional APIs

`AbortSignal.any()` presents a relatively high-level solution for combining
`AbortSignal`s. There are proposals for primitives which could allow userland
`AbortSignal` combining while avoiding the memory management issues that come
with the existing [Userland Solutions](#userland-solutions).

In this section we detail those proposals. Our overall conclusion is that using
them would require very careful and complex userland code to get the same
benefits as `AbortSignal.any()`. Although we could consider adding them in the
future &mdash; in particular, weak event listeners seem like they probably have
other non-AbortSignal-related use cases &mdash; for targeting the `AbortSignal`
chaining problem, we think they are not the right approach.

#### `AbortController.prototype.close()`

It has been [suggested](https://github.com/whatwg/dom/pull/1042) that an API
for explicitly *closing* an AbortController could be helpful for this problem
&mdash; potentially as a way to make this problem more easily solvable in
userland. Such an API would transition a signal from a pending state to a
rejected state, using the [promise analogy](#abortsignalracesignals), and allow
the UA to clear event listeners to help with the memory management.

Knowing when a signal is settled is important for memory management for this
problem, but it's possible to determine this through the lifetime of the
associated controller. When the controller goes out of scope (i.e. is GCed),
its signal can no longer be aborted. Browsers can use this to free resources
without needing an explicit close method, which is more in line with JavaScript
compared to explicit resource management.

Such explicit memory management might be useful in userland implementations,
but can also be approximated with a `FinalizationRegistry` callback for a
controller's signal if only weak references to that signal are being held by
the implementation.

#### Weak Event Listeners

Another proposal in this space is *weak event listeners*, which could help with
[userland solutions](#userland-solutions). The idea is to tie the lifetime of
parent signal event listeners the child signal's controller:

```js
AbortSignal.any = function(signals) {
  const controller = new AbortController();
  for (const signal of signals) {
    // UA stores a WeakRef to |controller|, but what should its lifetime be?
    signal.addWeakEventListener('abort', controller, () => { controller.abort(); }
  }
  return controller.signal;
}
```

Even with weak event listeners, getting the lifetime of the associated
controller is complicated and involves a lot of parent/child relationship
management. There are three cases where we want an event listener removed from
*all* parents of a child signal:

1. When *any* parent is aborted
1. When *all* parents decidedly will not abort
1. When the signal has no children and *no abort event listeners*

Weak event listeners would save userland implementations from needing to track
and remove event listeners when the above conditions are met, but would not
prevent tracking those conditions. And while such an API might be generally
useful, we don't feel it solves enough of this problem.

## Stakeholder Feedback / Opposition

TBD

## Acknowledgements

This work builds off of a previous
[discussion](https://github.com/whatwg/dom/issues/920) and work by
[@shicks](https://github.com/shicks) exploring this space. Thanks to all the
folks involved in that discussion:
[@annevk](https://github.com/annevk),
[@benjamingr](https://github.com/benjamingr),
[@benlesh](https://github.com/benlesh),
[@bterlson](https://github.com/bterlson),
[@domenic](https://github.com/domenic),
[@jakearchibald](https://github.com/jakearchibald),
[@MattiasBuelens](https://github.com/MattiasBuelens),
[@noseratio](https://github.com/noseratio),
[@rbuckton](https://github.com/rbuckton), and
[@wanderview](https://github.com/wanderview).

And thanks to
[@dlras2](https://github.com/dlras2),
[@domenic](https://github.com/domenic), and
[@shicks](https://github.com/shicks)
for providing feedback on this proposal.
