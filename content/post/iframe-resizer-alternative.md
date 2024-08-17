+++
title = "An Iframe Resizer Alternative"

date = "2024-08-16"
tags = [
    "js",
    "library",
    "iframe"
]
+++

![lizard-iframe.jpeg](/lizard-iframe.jpeg)


If you’ve ever embedded content using `<iframe>`, you know how tricky it can be to get the sizing just right.
For a long time, the go-to solution was the [Iframe resizer](https://github.com/davidjbradshaw/iframe-resizer) library, which works very well. However, with its recent switch to a commercial license, 
developers like myself found themselves in need of an alternative, so I created [Open Iframe Resizer](https://github.com/Lemick/open-iframe-resizer), a lightweight, open-source solution under the MIT license.

## Features

The first version is quite simple, you inject the script, and your browser resizes the registered iframes, this version also supports cross-origin iframes, which works with [postMessage](https://developer.mozilla.org/fr/docs/Web/API/Window/postMessage) API as the
parent window cannot alter cross-origin iframes child document for security reasons.

<iframe src="https://codesandbox.io/embed/m655zt?view=preview"
style="width:100%; height: 700px; border:0; border-radius: 4px; overflow:hidden;"
title="open-iframe-resize-browser"
allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
></iframe>

This library provides an alternative usable in commercial closed-source projects. Moreover, the library is fully typed, and I'm trying to keep the original lib API to ease the migration.

The core package is available [here](https://www.npmjs.com/package/@open-iframe-resizer/core) and a React component is also available [here](https://www.npmjs.com/package/@open-iframe-resizer/react).

## Limitations

The library supports Chrome 64+ (2018). This limitation comes from the `ResizeObserver` API, which could be polyfilled, although it is not a priority in the current version. 

Additionally, it currently lacks some features from the original library, such as injecting styles into an iframe.

## Try it out
Feel free to check out the [full source code](https://github.com/Lemick/open-iframe-resizer) and propose a PR or a feature you want to have. If you find it helpful, don’t forget to star the repo and spread the word!
