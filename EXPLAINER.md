# Make rendering schedulable

Today's DOM operations are defined to be synchronous:
the operation, and all the side effects it entails
(style resolution, layout, etc)
must be completed before control can be returned back to the JS thread.
This pauses not only the script,
but also any events that might be affected by script,
such as link navigation or touch scrolls
(if there are non-passive listeners subscribed).

If the script is appending a *large* subtree to the page,
this can create noticeable jank;
it's not too hard to achieve a 150ms+ delay,
which is very noticeable to a user.
The author can't reasonably break this up, either;
if they do the standard "do pieces of the work across multiple timeouts" approach to de-janking,
they'll instead get partially-constructed DOM showing,
which can be even worse for user experience.

This spec proposes to define a way to explicitly declare DOM appends to be "async",
returning immediately with a promise that is fulfilled when the work is done
(or rejected when the work is invalidated).
In the background, the user agent must,
at minimum,
chunk the work into broad phases --
informally, "parsing", "styling", "layout", and "painting" --
yielding back to the event loop between each phase
(or doing it entirely in parallel, in UAs that support such).
This chunking has a significant effect on perceived page latency,
based on some early Chrome experiments.

(Ideally, the UA would be able to chunk each phase too,
chopping them into work-slices that fit within a frame's spare time,
and making the work essentially jank-free.
This is expected to be difficult work, though;
initially requirements will just be to split up the main phases as described above.)

To prevent DOM creation from blocking the main thread, the rendering process needs to be broken up into chunks that the browser can schedule around other main thread tasks.


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

Another possibility is to explicitly batch the async operations into a transaction.
This makes it possible for the site to make edits to multiple parts of the DOM,
and have them all show up at once when they're done preparing,
rather than showing up piecemeal:

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

An open question is what, if anything, to show between the time the async append is called and the time the operation completes.

* One possibility is to add the nodes to the DOM immediately, but in an "inert" state,
	where they're forced to act like `display: none`,
	and any non-async mutations to the subtree throw
	or maybe cancel the append operation and yank the subtree from the DOM.
* Another is to not add anything to the DOM,
	but keep track of where the subtree would be added,
	and relevant mutations (defined appropriately) again cancel the operation.

`<script>` elements appended asyncly never run,
just like ones added via `innerHTML`.
