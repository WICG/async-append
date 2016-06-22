# Make rendering schedulable

Today's DOM operations are synchronous: style resolution, layout, etc must be completed before control can be returned to the JS thread. This pauses not only scripts, but also events such as link navigation or touch scrolls (if there are non-passive event listeners subscribed).

If the script is mutating a *large* subtree, this can create noticeable jank. It's not too hard to achieve a 150ms+ delay, which is very noticeable to a user. The author can't reasonably break this up, either; if they do the standard "do pieces of the work across multiple timeouts" approach to de-janking, they'll instead get partially-constructed DOM showing, which can be even worse for user experience.

![synchronous rendering lifecycle](sync-lifecycle.png)

This spec proposes a way to explicitly declare DOM mutations to be "async", returning immediately with a promise that is fulfilled when the work is done (or rejected if the work is invalidated). This API would be a parallel option to the current, synchronous way to mutate DOM.  

A developer could create a subtree to append or build up a set of mutations to be applied to the existing DOM. Once all the DOM mutations were calculated, the developer would call pass them to the async API. In the background, a user agent would, at minimum, chunk the resulting work into broad phases -- informally, "parsing", "styling", "layout", and "painting" -- yielding back to the event loop between each phase. Based on early investigations, chunking rendering work into broad phases could reduce jank by 30-50% depending on browser.

![asynchronous rendering lifecycle](async-lifecycle.png)

While breaking work into broad phases is a simpler way to get a large win, later implementations of the API could chunk each phase too, chopping them into work-slices that fit within a frame's spare time. This would essentially  make DOM work jank-free. This level of chunking is expected to be difficult work, and shouldn't be required for a V0 of the API.

# Constraints
It is important that this API closely matches the way that sites are manipulating DOM today. Large sites and frameworks are much more likely to use the feature if it is simply a matter of "feature detect and use async append if available". If the API requires an entire rewrite of DOM mutation logic, adoption will be low.

Calculating DOM mutations in a worker is a natural thing to pair with async rendering. This API should be relatively unopinionated about input format, so that things like [virtual dom](https://github.com/Matt-Esch/virtual-dom), [`WorkerNode`](github.com/drufball/worker-node), and [`DOMChangeList`](https://github.com/whatwg/dom/issues/270) are all potentially supported.

# Possible API

The shape of the API is not nailed down yet.

A simple possibility is to add "async" versions of all the append methods:

```javascript
partial interface ParentNode {
  CancelablePromise<MutationResult> appendAsync((Node or DOMString)... nodes);
  CancelablePromise<MutationResult> prependAsync((Node or DOMString)... nodes);
  CancelablePromise<MutationResult> removeAsync();
  CancelablePromise<MutationResult> insertBeforeAsync(Node node, Node? child);
};
```

Another possibility is to explicitly batch the async operations into a transaction. This makes it possible for the site to make edits to multiple parts of the DOM, and have them all show up at once when they're done preparing, rather than showing up piecemeal:

```javascript
const mutator = new DOMMutator();
// Not married to the name
// Can add options, e.g. new DOMMutator({ priority }), in the future?

for (let i = 0; i < 500; ++i) {
  myTable.appendAsync(mutator, createTableRow());
}

// Call .commit() to signal that you're done batching operations.
// The operations have been running in the background before the commit,
// but they're not allowed to "finish" until the commit happens.
mutator.commit();

// Once .commit() is called, the mutator is "dead" - if you try to use it in any
// further async operations, the operation automatically fails. Throws error?

// Also .cancel() to throw away the whole transaction. Can only be done before
// calling .commit(); ignored/throws after .commit() happens.

// Probably a .ready promise that fulfills when all the operations are complete;
// replaced whenever you add a new operation. Lets you hold off on the .commit()
// until other async operations complete, or lets your provide a cancellation window --
// if you only commit when the appends are finished, you can let the user cancel
// until that last moment.
```

The preceding code adds `async*()` methods to all nodes, and has them take the transaction object as their first argument. An alternative shape is for the transaction object to "wrap" DOM nodes, and just expose the *normal* names for all the append operations:

```javascript
const app = new DOMAsyncAppender();

for (let i = 0; i < 500; ++i) {
  app(myTable).appendChild(createTableRow());
}

app.commit();
```

# What Is Visible Before It Finishes?

An open question is what, if anything, to show between the time the async append is called and the time the operation completes.

* One possibility is to add the nodes to the DOM immediately, but in an "inert" state,
	where they're forced to act like `display: none`,
	and any non-async mutations to the subtree throw
	or maybe cancel the append operation and yank the subtree from the DOM.
* Another is to not add anything to the DOM,
	but keep track of where the subtree would be added,
	and relevant mutations (defined appropriately) again cancel the operation.

# Script Elements

`<script>` elements appended asyncly never run, just like ones added via `innerHTML`.
