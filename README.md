# async-append
A way to create DOM and add it to the document without blocking the main thread. For proposed solutions, see the [EXPLAINER](EXPLAINER.md).

# Problem
In order to minimize the number of DOM elements and improve load times, sites often construct elements as needed and add them to the page. _Hopefully_, this happens just in time for the user to see the element with no delay. In reality, the element trees constructed are quite large/complex.

In general, there are many UX patterns that lazily instantiate DOM:

- Infinite scrollings lists (news feeds)
- Side drawers
- Chat views
- Image slideshows

All of these would benefit from an ability to create and add DOM without blocking the main thread.

# Case Study: YouTube Gaming
[YouTube Gaming](gaming.youtube.com) constructs its comment panel for a video when the user clicks on 'comments', janking the main thread for anywhere from 55-180 ms for rendering (depending on browser). The number gets as high as __500ms__ when including custom element construction.

## [gaming.youtube.com](https://gaming.youtube.com/watch?v=i0purbwzs4U) 
The following numbers are ms measurements averaged over 5 times.
__Clicking on 'comments'__
- Chrome
  - Style: 65
  - Layout: 100
  - Paint: 15  
- Safari
  - Layout: 40
  - Paint: 15
- Firefox
  - Layout: 40
  - Paint: 15

__Scroll to bottom of 'comments'__
- Chrome
  - Style: 30
  - Layout: 15
  - Paint: 10  
- Safari
  - Layout (placeholder images): 25
  - Paint (placeholder images): 10
  - Layout (content): 25
  - Paint (content): 5
- Firefox
  - Layout: 15
  - Paint: 10


