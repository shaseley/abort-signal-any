# AbortSignal Memory Management

This document summarizes `AbortSignal` lifetime requirements, memory management
challenges that arise `AbortSignal.any()` and `AbortSignal.timeout()`, as well
as strategies for managing signal lifetimes.

## Background

### Signaling Abort

An `AbortSignal` *can signal abort* when it is not already aborted and one of the
following is true:

- The signal was created by `AbortSignal.timeout()`. These signals will
  inevitably signal abort when the timer expires.

- The signal is the [`signal`](https://dom.spec.whatwg.org/#dom-abortcontroller-signal)
  property for an `AbortController` that has not been GCed. Once the controller
  is out of scope, it can no longer abort.

- The signal was created by `AbortSignal.any(signals)` and one of `signals` can
  signal abort. These *derived* or *composite* signals depend on the source 
  signals for abort, and the source signals are fixed at creation time. If any
  one of the sources can signal abort, so can the derived signal. But if none of
  the sources can signal abort, then neither can the derived signal since it
  cannot be aborted in any other way.

### Observing Abort

When an `AbortSignal` [signals abort](https://dom.spec.whatwg.org/#abortsignal-signal-abort),
it *can be observed* in two ways:

1. By 'abort' event listeners attached to the signal
1. By algorithms [added](https://dom.spec.whatwg.org/#abortsignal-add) by other
   specs. Often these algorithms have no effect after the underlying
   operation completes successfully, e.g. `fetch()`, in which case removing the
   algorithm would have no effect on correctness.

### Signal Lifetime Requirement

An `AbortSignal` needs to be kept alive as long it can signal abort and
[signaling abort](https://dom.spec.whatwg.org/#abortsignal-signal-abort) can be
observed.

## The Problem

Timeout and derived signals need to be kept alive by the browser since they can
still abort and be observed even when no references to them exist in userland.
For timeout signals, this can be solved by having the timer (or timer task)
hold a strong reference to the signal. For derived signals, this can be solved
by having each source signal hold a strong reference.

But this approach holds on to resources for potentially much longer than is
necessary, e.g. when using `AbortSignal.any()` with long-lived top-level
controllers or long timeouts associated with an already successful operation.
Can we free up resources when they are no longer needed?

## Signal Management

This summarizes the approach we're taking in the Chromium prototype for
`AbortSignal.any()` and cleanup of `AbortSignal.timeout()`. The strategy is to
detect when a signal can abort and has observers, and to only hold a strong
reference to it when both are true. This only applies to timeout and derived
signals (signals associated with a controller are kept alive by their
controllers, and signals returned by `AbortSignal.abort()` are already
aborted).

1. Ensure objects that add abort algorithms keep the associated signal alive
   while the abort algorithm can still have an effect. Also remove algorithms
   when signaling abort will no longer have an effect (this is somewhat
   orthogonal, but is another opportunity to free resources).

1. Simplify source/dependent relationships in `AbortSignal.any()`. Described in
   detail [here](https://docs.google.com/document/d/1LvmsBLV85p-PhSGvTH-YwgD6onuhh1VXLg8jPlH32H4/edit?usp=sharing),
   we cut out *intermediate* signals by linking derived signals directly to
   their "abort sources", which are signals associated with a timer or
   controller. This is allowed since new sources cannot be added after
   creation, and will be specced to provide an expected event dispatch order.

1. Keep a strong reference to the signal from its global if the signal has an
   'abort' event listener, the signal's relevant global is fully active (if a
   window), and the signal can still abort (either a timeout signal or it has
   a source that can still abort).

1. Add a pre-finalizer for `AbortController` that indicates its signal can no
   longer abort. This removes the source from any dependent (derived) signals.
   If the signal has no more sources that can abort, remove strong references
   held because of observers.

1. Store references to source signals as strong pointers in derived signals.
   This ensures timeouts remain alive. Sources associated with controllers are
   removed in the previous step. Timeout sources will always abort, so they
   aren't removed, but a derived signal can be GCed when nothing is observing
   it (if no userland references), which will release the strong reference to
   the timeout signal.

1. Store references to dependent/derived signals as weak pointers. Source
   signals don't need to keep dependent signals alive since they are already
   kept alive if there are observers.

1. Clear all references when a signal is aborted.

### Additional Resource Management

As discussed in [this issue](https://github.com/shaseley/abort-signal-any/issues/3)
pending timers associated with timeout signals where the abort cannot be
observed can be problematic on some platforms and can waste resources, e.g.
wasted CPU cycles and excessive wake-ups. This can be mitigated by:

1. Implementing the behavior described above, which allows a timeout signal to
   be GCed as long as it doesn't have observers, and
1. Adding a pre-finalizer on timeout signals that removes the timer

## References

- [`AbortSignal.any()` Memory Management](https://docs.google.com/document/d/1LvmsBLV85p-PhSGvTH-YwgD6onuhh1VXLg8jPlH32H4/edit?usp=sharing)
- [Abort Signals & Abort Algorithm Lifetime](https://docs.google.com/document/d/1UMZrd5-RztLLUZ2g4vf4Nk7MBTnpfdihNi1Xh5qrND8/edit?usp=sharing&resourcekey=0-Xd_3IULrml81fZ7CGrltYw)
