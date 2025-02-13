---
layout: post
title: Lazy-loading images
authors:
  - jlwagner
  - rachelandrew
date: 2019-08-16
updated: 2022-08-11
description: |
  This post explains lazy-loading and the options available to you when lazy-loading images.
tags:
  - performance
  - images
feedback:
  - api
---

Images can appear on a webpage due to being inline in the HTML as `<img>` elements
or as CSS background images. In this post you will find out how to lazy-load both types of image.

## Inline images {: #images-inline }

The most common lazy-loading candidates are images as used in `<img>` elements.
With inline images we have three options for lazy-loading,
which may be used in combination for the best compatibility across browsers:

- [Using browser-level lazy-loading](#images-inline-browser-level)
- [Using Intersection Observer](#images-inline-intersection-observer)

### Using browser-level lazy-loading {: #images-inline-browser-level }

Chrome and Firefox both support lazy-loading with the `loading` attribute.
This attribute can be added to `<img>` elements, and also to `<iframe>` elements.
A value of `lazy` tells the browser to load the image immediately if it is in the viewport,
and to fetch other images when the user scrolls near them.

{% Aside %}
  Note `<iframe loading="lazy">` is currently non-standard.
  While implemented in Chromium, it does not yet have a specification and is subject to future change when this does happen.
  We suggest not to lazy-load iframes using the `loading` attribute until it becomes part of the specification.
{% endAside %}

See the `loading` field of MDN's
[browser compatibility](https://developer.mozilla.org/docs/Web/HTML/Element/img#Browser_compatibility)
table for details of browser support.
If the browser does not support lazy-loading then the attribute will be ignored
and images will load immediately, as normal.

For most websites, adding this attribute to inline images will be a performance boost
and save users loading images that they may not ever scroll to.
If you have large numbers of images and want to be sure that users of browsers without support for lazy-loading benefit
you will need to combine this with one of the methods explained next.

To learn more, check out [Browser-level lazy-loading for the web](/browser-level-image-lazy-loading/).

### Using Intersection Observer {: #images-inline-intersection-observer }

To polyfill lazy-loading of `<img>` elements, we use JavaScript to check if they're in the
viewport. If they are, their `src` (and sometimes `srcset`) attributes are
populated with URLs to the desired image content.

If you've written lazy-loading code before, you may have accomplished your task
by using event handlers such as `scroll` or `resize`. While this approach is the
most compatible across browsers, modern browsers offer a more performant and
efficient way to do the work of checking element visibility via [the
Intersection Observer API](https://developer.chrome.com/blog/intersectionobserver/).

Intersection Observer is easier to use and read than code relying on various
event handlers, because you only need to register an observer to watch
elements rather than writing tedious element visibility detection code. All
that's left to do is to decide what to do when an element is visible.
Let's assume this basic markup pattern for your lazily loaded `<img>` elements:

```html
<img class="lazy" src="placeholder-image.jpg" data-src="image-to-lazy-load-1x.jpg" data-srcset="image-to-lazy-load-2x.jpg 2x, image-to-lazy-load-1x.jpg 1x" alt="I'm an image!">
```

There are three relevant pieces of this markup that you should focus on:

1. The `class` attribute, which is what you'll select the element with in
JavaScript.
2. The `src` attribute, which references a placeholder image that will appear when
the page first loads.
3. The `data-src` and `data-srcset` attributes, which are placeholder attributes
containing the URL for the image you'll load once the element is in the viewport.

Now let's see how to use Intersection Observer in JavaScript to lazy-load
images using this markup pattern:

```javascript
document.addEventListener("DOMContentLoaded", function() {
  var lazyImages = [].slice.call(document.querySelectorAll("img.lazy"));

  if ("IntersectionObserver" in window) {
    let lazyImageObserver = new IntersectionObserver(function(entries, observer) {
      entries.forEach(function(entry) {
        if (entry.isIntersecting) {
          let lazyImage = entry.target;
          lazyImage.src = lazyImage.dataset.src;
          lazyImage.srcset = lazyImage.dataset.srcset;
          lazyImage.classList.remove("lazy");
          lazyImageObserver.unobserve(lazyImage);
        }
      });
    });

    lazyImages.forEach(function(lazyImage) {
      lazyImageObserver.observe(lazyImage);
    });
  } else {
    // Possibly fall back to event handlers here
  }
});
```

On the document's `DOMContentLoaded` event, this script queries the DOM for all
`<img>` elements with a class of `lazy`. If Intersection Observer is available,
create a new observer that runs a callback when `img.lazy` elements enter the
viewport.

{% Glitch {
  id: 'lazy-intersection-observer',
  path: 'index.html',
  previewSize: 0
} %}

Intersection Observer is available in all modern browsers.
Therefore using it as a polyfill for `loading="lazy"` will ensure that lazy-loading is available for most visitors.

## Images in CSS {: #images-css }

While `<img>` tags are the most common way of using images on web pages, images
can also be invoked via the CSS
[`background-image`](https://developer.mozilla.org/docs/Web/CSS/background-image)
property (and other properties). Browser-level lazy-loading does not apply to CSS background images,
so you need to consider other methods if you have background images to lazy-load.

Unlike `<img>` elements which load regardless
of their visibility, image loading behavior in CSS is done with more
speculation. When [the document and CSS object
models](/critical-rendering-path-constructing-the-object-model/)
and [render
tree](/critical-rendering-path-render-tree-construction/)
are built, the browser examines how CSS is applied to a document before
requesting external resources. If the browser has determined a CSS rule
involving an external resource doesn't apply to the document as it's currently
constructed, the browser doesn't request it.

This speculative behavior can be used to defer the loading of images in CSS by
using JavaScript to determine when an element is within the viewport, and
subsequently applying a class to that element that applies styling invoking a
background image. This causes the image to be downloaded at the time of need
instead of at initial load. For example, let's take an element that contains a
large hero background image:

```html
<div class="lazy-background">
  <h1>Here's a hero heading to get your attention!</h1>
  <p>Here's hero copy to convince you to buy a thing!</p>
  <a href="/buy-a-thing">Buy a thing!</a>
</div>
```

The `div.lazy-background` element would normally contain the hero background
image invoked by some CSS. In this lazy-loading example, however, you can isolate
the `div.lazy-background` element's `background-image` property via a `visible`
class added to the element when it's in the viewport:

```css
.lazy-background {
  background-image: url("hero-placeholder.jpg"); /* Placeholder image */
}

.lazy-background.visible {
  background-image: url("hero.jpg"); /* The final image */
}
```

From here, use JavaScript to check if the element is in the viewport (with
Intersection Observer!), and add the `visible` class to the
`div.lazy-background` element at that time, which loads the image:

```javascript
document.addEventListener("DOMContentLoaded", function() {
  var lazyBackgrounds = [].slice.call(document.querySelectorAll(".lazy-background"));

  if ("IntersectionObserver" in window) {
    let lazyBackgroundObserver = new IntersectionObserver(function(entries, observer) {
      entries.forEach(function(entry) {
        if (entry.isIntersecting) {
          entry.target.classList.add("visible");
          lazyBackgroundObserver.unobserve(entry.target);
        }
      });
    });

    lazyBackgrounds.forEach(function(lazyBackground) {
      lazyBackgroundObserver.observe(lazyBackground);
    });
  }
});
```

{% Glitch {
  id: 'lazy-background',
  path: 'index.html',
  previewSize: 0
} %}

### Effects on Largest Contentful Paint (LCP)

Lazy loading is a great optimization that reduces both overall data usage and network contention during startup by deferring the loading of images to when they're actually needed. This can improve startup time, and reduce processing on the main thread by reducing time needed for image decodes.

However, lazy loading is a technique that can affect your website's [Largest Contentful Paint LCP](/lcp/) in a negative way if you're too eager with the technique. One thing you will want to avoid is lazy loading images that are in the viewport during startup.

When using JavaScript-based lazy loaders, you will want to avoid lazy loading in-viewport images because these solutions often use a `data-src` or `data-srcset` attribute as a placeholder for `src` and `srcset` attributes. The problem here is that the loading of these images will be delayed because [the browser preload scanner can't find them during startup](/preload-scanner/#lazy-loading-with-javascript).

Even using browser-level lazy loading to lazy load an in-viewport image can backfire. When `loading="lazy"` is applied to an in-viewport image, [that image will be delayed until the browser knows for sure it's in the viewport](/optimize-lcp/#optimize-the-priority-the-resource-is-given), which can affect a page's LCP.

_Never_ lazy load images that are visible in the viewport during startup. It's a pattern that will affect your site's LCP negatively, and therefore the user experience. If you need an image at startup, load it at startup as quickly as possible by not lazy loading it!

## Lazy-loading libraries {: #libraries }

You should use browser-level lazy loading whenever possible, but if you find yourself in a situation where that isn't an option&mdash;such as a significant group of users still reliant on older browsers&mdash;the following libraries can be used to lazy-load images:

- [lazysizes](https://github.com/aFarkas/lazysizes) is a full-featured lazy
loading library that lazy-loads images and iframes. The pattern it uses is quite
similar to the code examples shown here in that it automatically binds to a
`lazyload` class on `<img>` elements, and requires you to specify image URLs in
`data-src` and/or `data-srcset` attributes, the contents of which are swapped
into `src` and/or `srcset` attributes, respectively. It uses Intersection
Observer (which you can polyfill), and can be extended with [a number of
plugins](https://github.com/aFarkas/lazysizes#available-plugins-in-this-repo) to
do things like lazy-load video. [Find out more about using lazysizes](/use-lazysizes-to-lazyload-images/).
- [vanilla-lazyload](https://github.com/verlok/vanilla-lazyload) is a
  lightweight option for lazy-loading images, background images, videos, iframes,
  and scripts. It leverages Intersection Observer, supports responsive images, and
  enables browser-level lazy loading.
- [lozad.js](https://github.com/ApoorvSaxena/lozad.js) is a another lightweight
option that uses Intersection Observer only. As such, it's highly performant,
but will need to be polyfilled before you can use it on older browsers.
- If you need a React-specific lazy-loading library, consider
[react-lazyload](https://github.com/jasonslyvia/react-lazyload). While it
doesn't use Intersection Observer, it _does_ provide a familiar method of lazy
loading images for those accustomed to developing applications with React.
