+++
title = "Things I Discovered Building a Lib"

date = "2025-10-01"
tags = [
    "js"
]
+++

![lizard-sherlock.png](/lizard-sherlock.png)

One of my recent [side projects](https://github.com/Lemick/open-iframe-resizer) is a browser library to help with iframe integrations, as the other iframe-resizer changed its licensing. Here are some things I noticed while building and maintaining it.

## 1. Everyone Hates iframes

But everyone uses them. 😅

The success of the library (half a million hits on the CDN each month and thousands of weekly downloads via npm) tells me something important. Despite often being one of the web's most hated features, iframes are clearly still everywhere. Honestly, I can't recall a single company I've worked for that didn't use an iframe integration at some point.

While the overall user experience is sometimes debatable and often worse than a true integration, their simplicity for embedding external content keeps them popular.

The problem with people hating iframes is often that they’re using the wrong tool for the job.

## 2. API Design Is Easy at First

But hard to change later 🙅‍♂️

I started this project by essentially copying the API of the original. My thinking was that this would make it easier for people to switch. Now? I'm starting to second-guess that decision.

After some reflection, I realize I'd prefer to rely more on customizable, generic callbacks rather than a plethora of high-level properties. This approach is less "plug-and-play" but can often cover more use cases since you let the consumer define the behaviour.

For example, due to a specific feature request from a user who wanted to have an automatic scroll on resize to keep the iframe in the viewport, I implemented an event handler that gets called whenever the iframe resizes, passing the new size as an argument. That can be used for different use cases, like updating the scroll position of the parent window if the iframe is focused, or updating some other UI element based on the new size.

```js
onIframeResize: (context: ResizeContext) => void
```

A more straightforward but less configurable approach would have been to add a boolean property like this:

```js
scrollOnResize: boolean
```

To have the best of both worlds, I added to the lib a ready-to-use function that users can apply to cover their use case:

```js
initialize({ onIframeResize: updateParentScrollOnResize })
```

This might make it slightly more complex for some users initially, but it keeps the API surface lean and prevents a huge number of parameters. It's a tough balance to find, and I'm still figuring it out.

Even though I don’t regret replicating the API of the other lib to gain popularity at first, having a clear vision from the beginning when designing your API is crucial to keeping it consistent.

## 3. Playwright Is Fantastic

But Screenshots are time-consuming 👀

[Playwright](https://playwright.dev/) has been the best testing library I've discovered in years. It's easy to set up, incredibly fast, and just a joy to work with.

One of Playwright's cool features is screenshot comparison. It allows you to verify the actual visual rendering of a page, not just its textual or interactive content. However, this comes with significant maintenance overhead.

Nowadays, projects are rarely built and tested on a single machine architecture. You've got your local developer machine and at least one CI pipeline (often a Linux distribution). This complicates test data because page rendering can differ based on:

- The browser (Chrome, Firefox, Safari, etc.)
- The operating system (Windows, macOS, Linux)
- Sometimes even the CPU architecture!

So, if you want to run screenshot comparisons on Chrome, Firefox, and Safari, you'll need at least six baseline screenshots (2 OS × 3 browsers) for every single visual test. The more browsers and OS you want to cover, the more you’re hit by this Cartesian product of variables.

This is why I try to use screenshot comparisons sparingly, reserving them for very specific, critical use cases.

## 4. I Love Biome

But maybe I just dislike ESLint + Prettier 👿

One of the biggest headaches in the JavaScript ecosystem for me has always been the sheer number of tools and configurations needed for simple tasks. I never understood why we need multiple dependencies and several configuration files just to lint and format text files. It always baffled me.

Then I tried [Biome](https://biomejs.dev/), and I haven't looked back. It's fast, has excellent "convention over configuration" defaults, and even boasts an IntelliJ plugin! It just works, and that's a beautiful thing.

## 5. Async Comes at a Cost

This one is more technical, but I found it interesting.

For `v2`, I had to change the `initialize` API and its return value from an `Array` to a `Promise<Array>`, due to the asynchronous nature of initialization.

One interesting thing I discovered is how much it can impact your consumer when your API returns a promise compared to a normal result.

For example, in a React component’s `useEffect`, if you call a function not returning a promise, you can do this:

```js
useEffect(() => {
  const results = initialize(settings, iframeRef.current);
  return () =>
    results.forEach((value) => {
      value.unsubscribe();
    });
}, []);
```

If now, the `initialize` method returns a promise, you have to do this:
```js
const unsubscribeIframeRef = useRef([]);

useEffect(() => {
  let isUnmounted = false;

  initialize(settings, iframeRef.current).then((results) => {
    if (isUnmounted) {
      results.forEach((c) => {
        c.unsubscribe();
      });
    } else {
      unsubscribeIframeRef.current = results;
    }
  });

  return () => {
    isUnmounted = true;
    unsubscribeIframeRef.current?.forEach((c) => {
      c.unsubscribe();
    });
    unsubscribeIframeRef.current = [];
  };
}, []);
```

This complex pattern is well known for side effects returning a promise (e.g., HTTP calls). This code tracks if the component has been unmounted before the promise is resolved. We often use a library to avoid this ugly pattern in React.

It's an interesting downside of using async calls in code with a different lifecycle (like React components), as once you create async tasks, **you exit the main task loop and decorrelate your component lifecycle from your async side effects** (your task goes into the global microtask queue and you can’t control it anymore).

I believe asynchronous methods should not have such a huge impact on the consumer code, that’s definitely a weakness of the async JS paradigm. In a better world, asynchronous tasks spawned in a component should be created within the scope of that component and automatically cancelled when it’s unmounted. This would make async tasks much easier to use in UI components.

Here’s a virtual example where:
- JS supports cancelable scoped async tasks (no need to pass an `AbortController` everywhere).
- Cancellation is cooperative, so the called code must handle its own cancellation.
- `useScopedEffect` is a custom hook that supports automatic cancellation and automatically cleans up any async task created in its scope when unmounted.

```js
useScopedEffect(async () => {
  // Initialization that supports automatic cancellation
  const results = await initialize(settings, iframeRef.current);
  
  // Register cleanup for resources that exist after initialization
  results.forEach((c) => scope.onUnmount(c.unsubscribe));
}, [settings]);
```

After a bit of research, I found that this problem is commonly called the [await event horizon](https://frontside.com/blog/2023-12-11-await-event-horizon/). Some libraries like [Effection](https://effection-www.deno.dev/) try to tackle this problem using [generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*), because they can be "cancelled" by design using the `return()` method.

Sadly for now, all evolution of the language to support this natively [seems stalled](https://github.com/tc39/proposal-cancelable-promises/issues/70). A lot of languages already have structured concurrency support, so we’ll see if JS ever gets there 🤞.
