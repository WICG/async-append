# Make rendering schedulable

Today's DOM operations are synchronous. This pauses scripts and events such as link navigation or touch scrolls.

Scripts that mutate *large* subtrees can create noticeable jank. It's not hard to achieve a 150ms+ delay while trying to mutate the DOM in reasonable ways. If web developers attempt to do pieces of this mutation work across multiple timeouts, they will show partially-constructed DOM, which can be even worse for the user experience than jank.

![synchronous rendering lifecycle](sync-lifecycle.png)

This spec proposes a way to explicitly declare DOM mutations to be "async", returning immediately with a promise that is fulfilled when the work is done or rejected if the work is invalidated. This API would be a parallel option to the current, synchronous way to mutate DOM.

A developer could create a subtree to append or build up a set of mutations to be applied to the existing DOM. Once all the DOM mutations were calculated, the developer would pass them to the async API. In the background, a user agent would, at minimum, chunk the resulting work into broad phases yielding back to the event loop between each phase. Based on early investigations, chunking rendering work into broad phases could reduce jank by 30-50% depending on browser.

![asynchronous rendering lifecycle](async-lifecycle.png)

Later implementations of the API could further chunk each phase until they fit within a frame's spare time. This would essentially make DOM work jank-free. This level of chunking is expected to be difficult, and shouldn't be required for a V0 of the API.

# Constraints
We have heard from several developers that DOM mutations can only be async up to a point. After a certain number of frames, it is better to jank while focusing on render work rather than delay the UI update any further. This API should enable developers to specify how long the async operation is allowed to take before it becomes blocking.

This API should closely match the way that sites are manipulating DOM today. Large sites and frameworks are much more likely to use the feature if adoption means "feature detect and swap sync mutations for async append if available". If the API requires an entire rewrite of DOM mutation logic, adoption will be low.

Calculating DOM mutations in a worker is a natural thing to pair with async rendering. This API should be relatively unopinionated about input format, so that things like [virtual dom](https://github.com/Matt-Esch/virtual-dom), [`WorkerNode`](github.com/drufball/worker-node), and [`DOMChangeList`](https://github.com/whatwg/dom/issues/270) are all supported.

# Current (Rough) Planned API

```javascript
partial interface Element {
  void asyncAppend(DOMBatch, Element);
  void asyncAppend(Element);
  /* plus pairs for all the other insertion/removal methods,
     on Element and ChildNode */
};

interface DOMBatch {
  readonly attribute boolean started;
  readonly attribute Promise<void> ready;

  void finish();
  void cancel(); // or a cancel token to DOMBatch's constructor
}
```

All the mutation operations come in two forms:

* a "fire and forget" version with identical signature to the sync method,
  that just triggers an async append that finishes "sometime" in the future.
  This is ideal for simpler applications that aren't doing complicated time/state management,
  but still want the benefit of avoiding jank in the rest of the application
  when they're mutating the DOM.
* an explicit tracking/batching version that takes a DOMBatch object,
  followed by the normal signature of the method.
  This lets you track the process and know when it completes
  (by watching the `ready` promise),
  cancel the mutation if it turns out you don't need it,
  and allows you to explicitly batch several mutations together
  so they'll all show up in the DOM at the same time
  (by passing the same DOMBatch to multiple calls).

The DOMBatch object passes through several distinct phases:
* batching
* started
* ready
* finished

When initially constructed, it's "batching",
and can be passed to mutation methods.
While in this state, the UA does nothing but track the mutations that will be performed.

At end of microtask, it switches to "started",
and sets its "started" boolean to `true`.
At this point the UA begins the async work:
styling, layout, paint, etc.
The DOMBatch can no longer be passed to mutation methods;
doing so will cause the method to throw an XXXError.

When the async work is finished,
and the mutation operation is ready to be mapped into the DOM,
it switches to "ready",
and fulfills its `ready` promise.
Calling `finish()` at this point *should* be a very fast sync call.
(But see below for calling `finish()` early or late.)

After `finish()` is called,
it switches to "finished".
At this state the operation is fully complete,
and the DOMBatch will not change in the future.
In this stage the `cancel()` method has no effect
(and maybe should throw?).

## Calling `cancel()`

As long as the DOMBatch is not in the "finished" state,
the author can call `cancel()` to stop the mutation.
This immediately shifts the DOMBatch into its "finished" state,
and throws away any pending mutation work the UA might have been doing.

## Calling `finish()` early or late

Under normal circumstances,
the author is expected to call `finish()` in the fulfillment callback
of the DOMBatch's `ready` promise.
At that time the async work has just been completed,
and `finish()` should be a very fast bit of sync work.

Calling `finish()` before the DOMBatch is in the "ready" state is allowed,
and causes the UA to synchronously finish the mutation immediately.
This is useful to give the UA several frames of delay to do async work,
while still guaranteeing that the work is put on the screen ASAP after that deadline.

Calling `finish()` much later than the "started"=>"ready" transition
*should* be identical to calling it immediately after the transition, as intended
(a quick bit of sync work),
but the UA *may* discard pending async work after a period of time
or when it's under memory pressure.
If this occurs, calling `finish()` just synchronously redoes the work,
same as calling `finish()` before the DOMBatch is "ready".

The essential guarantee here is that calling `finish()` must always succeed,
and it must always block until the work is complete,
guaranteeing that the DOM is in the desired state when it returns.

# What Is Visible Before It Finishes?

Nothing is added to the DOM until `finish()` is called.

This enables an interesting trivial usage:
one can do a sequence of DOM reads and *async* DOM writes,
and then `finish()` them at the end for a sync insertion.
This duplicates the behavior of the "FastDOM" library,
which explicitly separates DOM reading and writing phases
to prevent accidental interweaving
triggering unwanted sync layouts.
(Allowing the insertion to be fully async is even better,
but not always possible or desired.)

# Mutating the DOM While An Async Is In-Flight

Doing the mutation work async assumes that
the UA can accurately determine what selectors will apply to the content
and thus how it will be laid out/painted/etc.
Further mutations between the time the UA starts a batch of async work
(DOMBatch state of "started")
and when it's finished
(DOMBatch state of "finished")
can invalidate this work,
requiring the UA to throw out the work-so-far and start over.

While the DOMBatch is in the "started" state,
the UA must silently restart its async work
whenever it's invalidated,
but otherwise continue as normal.

While the DOMBatch is in the "ready" state,
the UA *may* silently restart its async work
whenever its invalidated
(thus possibly causing a call to `finish()` to take extra synchronous time,
if it's called before the work is readied again).
It is not required to do so, however;
it may simply wait until `finish()` is called
and then synchronously redo the work.

The essential guarantee here is that
an author can naively call `finish()` when the UA tells them the work is done
and have the operation successfully complete,
even if further mutations have invalidated work in the meantime.

UAs are encouraged to alert the author when this invalidation happens,
such as thru logging a message to the console.

Authors are encouraged to explicitly batch mutation work with a DOMBatch,
and avoid mutating the DOM otherwise if at all possible.

# Script Elements

`<script>` elements appended asyncly never run, just like ones added via `innerHTML`.

# Possible Extensions/Changes

* adding a `priority` argument to the DOMBatch constructor,
  allowing the author to explicitly differentiate between "important" and "less important" mutations,
  and ensure that important updates don't get inadvertently starved and delayed by a flurry of trivial updates.
* adding an `explicitStart` argument to the DOMBatch constructor,
  that turns off the DOMBatch's "automatically start at end of microtask" behavior,
  so work can be batched across microtasks
  and then explicitly started
  (by some additional method added to DOMBatch).
* switching from a `started` boolean to a `state` enum?
* having the "fire and forget" mutation methods return a promise
  (fulfilled when the work is "finished"),
  so you can track their work
  without having to explicitly create and manage a DOMBatch.
