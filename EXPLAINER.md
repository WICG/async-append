# Make rendering schedulable
To prevent DOM creation from blocking the main thread, the rendering process needs to be broken up into chunks that the browser can schedule around other main thread tasks.

Initially, we can simply split rendering into the distinct lifecycle phases: style, layout, and paint. This should cut main thread jank significantly without requiring too much work - just the ability to pause between lifecycle phases.

Eventually, browsers could make each lifecycle phase chunkable as well, removing all jank from the main thread.

# The API
```javascript
partial interface ParentNode {
  CancelablePromise<MutationResult> appendAsync((Node or DOMString)... nodes);
  CancelablePromise<MutationResult> prependAsync((Node or DOMString)... nodes);
  CancelablePromise<MutationResult> removeAsync();
  CancelablePromise<MutationResult> insertBeforeAsync(Node node, Node? child);
};
```

The user agent will return a promise that completes when the subtree's style, layout, and paint are computed. When the promise returns, the site calls `commit()` to add the subtree to the document. This two stage model allows sites to coordinate the display of components, such as having a title bar and navigation display together.

__Conditions__
- Modifying the subtree structure or inline style will cause the promise to fail.
- All getters for style or layout related information return values as if the element was display: none.
- `<script>` elements in input are never run, just as in `innerHTML`.

# Example Code
```javascript
const mutator = new DOMMutator();
// Not married to the name
// Can add options, e.g. new DOMMutator({ priority }), in the future?

for (let i = 0; i < 500; ++i) {
  myTable.appendAsync(mutator, createTableRow());
  // promises are for void; we don't use them anyway.
}

// Can do this immediately, which means:
// wait for async mutation operations that currently have the mutator reference
// to complete. Once they do, commit them all at once.
mutator.commit();

// Or you could wait for some async signal to call mutator.commit().
// It will still wait for all operations to complete though.

// If you try to pass mutator anywhere after doing mutator.commit(),
// that operation rejects.

// There's also mutator.cancel() as a way of doing batch cancelation,
// instead of a single cancelation based on cancelable promises.
```
