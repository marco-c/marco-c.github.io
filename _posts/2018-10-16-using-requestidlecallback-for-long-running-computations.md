---
title: Using requestIdleCallback for long running computations
tags: [javascript]
---

One of the ways developers have tipically tried to keep a smooth web application, without interfering with the browser's animation and response to input, is to use a Web Worker for long running computations. For example, in the Prism.js (a library for syntax highlighting) API there's an `async` parameter to choose <cite>&ldquo;Whether to use Web Workers to improve performance and avoid blocking the UI when highlighting very large chunks of code&rdquo;</cite>.

This is perfectly fine, but web workers are not so easy to use or debug. To take Prism.js again as an example, the option I mentioned earlier is false by default. Why?
> In most cases, you will want to highlight reasonably sized chunks of code, and this will not be needed. Furthermore, using Web Workers is actually slower than synchronously highlighting, due to the overhead of creating and terminating the Worker. It just appears faster in these cases because it doesn’t block the main thread. In addition, since Web Workers operate on files instead of objects, plugins that hook on core parts of Prism (e.g. modify language definitions) will not work unless included in the same file (using the builder in the Download page will protect you from this pitfall). Lastly, Web Workers cannot interact with the DOM and most other APIs (e.g. the console), so they are notoriously hard to debug."

Another alternative to achieve the same result, without using Web Workers and making things more difficult, is to use [requestIdleCallback](https://developer.mozilla.org/docs/Web/API/Window/requestIdleCallback). This function allows a callback to be scheduled when the browser is idle, enabling us to perform background work / low priority work on the main thread without impacting animations / input response. N.B.: This will still be slower than synchronous, but might be cheaper than a Web Worker since you don't have to pay the price of the Worker initialization.

Here's an example, using [promises](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Promise) and [asynchronous functions](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Statements/async_function) we can also avoid callback hell and keep using normal loops.

```javascript
function idle() {
  return new Promise(resolve => requestIdleCallback(resolve));
}

async function work() {
  let deadline = await idle();

  for (let job of jobs) {
    if (deadline.timeRemaining() <= 1) {
      deadline = await idle();
    }

    // Do something with `job`...
  }
}
```

I'm doing something similar in my [Searchfox in Phabricator extension](https://addons.mozilla.org/it/firefox/addon/searchfox-phabricator/), to operate on one source line at a time and avoid slowing down the normal Phabricator operation. [Here's where I'm doing it](https://github.com/marco-c/mozsearch-phabricator-addon/blob/9cc24d3d104940863aed1adf417ad41b8aaffdfe/phabricator.js#L161).
