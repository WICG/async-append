# async-append
A way to create DOM and add it to the document without blocking the main thread. For proposed solutions, see the [EXPLAINER](EXPLAINER.md).

# Problem
In order to minimize the number of DOM elements and improve load times, sites often construct elements as needed and add them to the page. _Hopefully_, this happens just in time for the user to see the element with no delay. In reality, the element trees constructed are quite large/complex. [YouTube Gaming](gaming.youtube.com) constructs its entire comment panel for a video when the user clicks on 'comments', janking the main thread for almost __500ms__.

In general, there are many UX patterns that lazily instantiate DOM:

- Infinite scrollings lists (news feeds)
- Side drawers
- Chat views
- Image slideshows

All of these would benefit from an ability to create and add DOM without blocking the main thread.


