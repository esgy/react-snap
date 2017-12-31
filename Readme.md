# react-snap [![npm](https://img.shields.io/npm/v/react-snap.svg)](https://www.npmjs.com/package/react-snap) ![npm](https://img.shields.io/npm/dt/react-snap.svg) [![Twitter Follow](https://img.shields.io/twitter/url/http/shields.io.svg?style=social&label=Follow)](https://twitter.com/stereobooster)

Pre-renders web app into static HTML. Uses headless chrome to crawl all available links starting from the root. Heavily inspired by [prep](https://github.com/graphcool/prep) and [react-snapshot](https://github.com/geelen/react-snapshot), but written from scratch. Uses best practices to get best loading performance.

## Features

- Enables **SEO** (google, duckduckgo...) and **SMO** (twitter, facebook...) for SPA.
- **Works out-of-the-box** with [create-react-app](https://github.com/facebookincubator/create-react-app) - no code-changes required.
- Uses **real browser** behind the scene, so no issue with unsupported HTML5 features, like WebGL or Blobs.
- Does a lot of load **performance optimization**. [Here are details](doc/load-performance-optimizations.md), if you are curious.
- **Does not depend on React**. The name is inspired by `react-snapshot`. Works with any technology.
- npm package does not have compilation step, so **you can fork** it, change what you need and install it with GitHub URL.

**Zero configuration** is the main feature. You do not need to worry how does it work or how to configure it. But if you are curious [here are details](doc/behind-the-scenes.md).

## Basic usage with create-react-app

Install:

```sh
yarn add --dev react-snap
```

Change `package.json`:

```json
"scripts": {
  "postbuild": "react-snap"
}
```

Change `src/index.js` (for React 16+):

```js
import { hydrate, render } from 'react-dom';

const rootElement = document.getElementById('root');
if (rootElement.hasChildNodes()) {
  hydrate(<App />, rootElement);
} else {
  render(<App />, rootElement);
}
```

That's it!

## Basic usage with Vue.js


Install:

```sh
yarn add --dev react-snap
```

Change `package.json`:

```json
"scripts": {
  "postbuild": "react-snap"
},
"reactSnap": {
  "source": "dist"
}
```

`source` - output folder of webpack or any other bundler of your choice

Example: [Switch from prerender-spa-plugin to react-snap](https://github.com/stereobooster/prerender-spa-plugin/commit/778594f55b5859cd3ca57dfd6e08b1d9008d2823)

## ✨ Examples

- [Emotion website load performance optimization](doc/emotion-site-optimization.md)
- [Load performance optimization](doc/an-almost-static-stack-optimization.md)
- [recipes](doc/recipes.md)
- [stereobooster/an-almost-static-stack](https://github.com/stereobooster/an-almost-static-stack)

## ⚙️ Customization

If you need to pass some options for `react-snap`, you can do this in the `package.json`, like this:

```json
"reactSnap": {
  "inlineCss": true
}
```

All options are not documented yet, but you can check `defaultOptions` in `index.js`.

### inlineCss

Experimental feature - requires improvements.

`react-snap` can inline critical CSS with the help of [minimalcss](https://github.com/peterbe/minimalcss) and full CSS will be loaded in a nonblocking manner with the help of [loadCss](https://www.npmjs.com/package/fg-loadcss).

Use `inlineCss: true` to enable this feature.

TODO: as soon as the feature will be stable it should be enabled by default.

## ⚠️ Caveats

### Async components

Also known as [code splitting](https://webpack.github.io/docs/code-splitting.html), [dynamic import](https://github.com/tc39/proposal-dynamic-import) (TC39 proposal), "chunks" (which are loaded on demand), "layers", "rollups", or "fragments".

Async component (in React) is a technique (typically implemented as a Higher Order Component) for loading components with dynamic `import`. There are a lot of solutions in this field. Here are some examples:

- [`loadable-components`](https://github.com/smooth-code/loadable-components)
- [`react-loadable`](https://github.com/thejameskyle/react-loadable)
- [`react-async-component`](https://github.com/ctrlplusb/react-async-component)
- [`react-code-splitting`](https://github.com/didierfranc/react-code-splitting)

It is not a problem to render async component with react-snap, tricky part happens when prerendered React application boots and async components are not loaded yet, so React draws "loading" state of a component, later when component loaded react draws actual component. As the result - user sees a flash.

```
100%                    /----|    |----
                       /     |    |
                      /      |    |
                     /       |    |
                    /        |____|
  visual progress  /
                  /
0%  -------------/
```

`react-loadable` and `loadable-components` solve this issue for SSR. But only `loadable-components` can solve this issue for "snapshot" setup:

```js
import { loadComponents } from "loadable-components";
import { getState } from "loadable-components/snap";
window.snapSaveState = () => getState();

loadComponents().then(() => {
  hydrate(AppWithRouter, rootElement);
});
```

**Caution**: there seems to be [an issue in `loadable-components`](https://github.com/smooth-code/loadable-components/issues/25). Be carefull.

### Redux

See: [Redux Srever Rendering Section](https://redux.js.org/docs/recipes/ServerRendering.html#the-client-side)

```js
// Grab the state from a global variable injected into the server-generated HTML
const preloadedState = window.__PRELOADED_STATE__

// Allow the passed state to be garbage-collected
delete window.__PRELOADED_STATE__

// Create Redux store with initial state
const store = createStore(counterApp, preloadedState || initialState)

// Tell react-snap how to save Redux state
window.snapSaveState = () => ({
  "__PRELOADED_STATE__": store.getState()
});
```

**Caution**: as of now only basic "JSON" data types are supported e.g. Date, Set, Map, NaN **won't** be handled right ([#54](https://github.com/stereobooster/react-snap/issues/54)).

### Third-party requests: Google Analytics, Mapbox etc.

You can block all third-party requests with the following config

```
"skipThirdPartyRequests": true
```

### AJAX

`react-snap` can capture all AJAX requests. It will store `json` request to the same domain in `window.snapStore[<path>]`, where `<path>` is the path of json request.

Use `cacheAjaxRequests: true` to enable this feature.

### Service Workers

By default `create-react-app` uses `index.html` as fallback:

```js
navigateFallback: publicUrl + '/index.html',
```

you need to change this to an unprerendered version of `index.html` - `200.html`, otherwise you will see a flash of `index.html` on other pages (if you have any). See [Configure sw-precache without ejecting](https://github.com/stereobooster/react-snap/blob/master/Recipes.md#configure-sw-precache-without-ejecting) for more information.

### WebGL

Headless chrome does not fully support WebGL, if you need render it you can use

```
"headless": false
```

### Containers and other restricted environments

Puppeteer (headless chrome) may fail due to sandboxing issues. To get around this,
you may use

```
"puppeteerArgs": ["--no-sandbox", "--disable-setuid-sandbox"]
```

Read more about [puppeteer troubleshooting.](https://github.com/GoogleChrome/puppeteer/blob/master/docs/troubleshooting.md)

### Semantic UI

[Semantic UI](https://semantic-ui.com/) is defined over class substrings that contain spaces
(eg. "three column"). Sorting the class names therefore breaks the styling. To get around this,
use the following configuration:

```
"minifyHtml": { "sortClassName": false }
```

## Possible improvements

- Improve [preconnect](http://caniuse.com/#feat=link-rel-preconnect), [dns-prefetch](http://caniuse.com/#feat=link-rel-dns-prefetch) functionality, maybe use [media queries](https://developer.mozilla.org/en-US/docs/Web/HTML/Preloading_content). Example: load in small screen - capture all assets, add with a media query for the small screen, load in big screen add the rest of the assets with a media query for the big screen.
- Do not load assets, the same way as minimalcss does
- Evaluate [penthouse](https://github.com/pocketjoso/penthouse) as alternative to [minimalcss](https://github.com/peterbe/minimalcss)
- Check if there is a way to improve font loading. See: [1](https://www.zachleat.com/web/comprehensive-webfonts/),
[2](https://github.com/malchata/unicode-ranger/tree/v2)

## Alternatives

See [alternatives](doc/alternatives.md)

## Contributing

### Report a bug

Please provide a reproducible demo of a bug and steps to reproduce it. Thanks

### Share on the web

Tweet it, like it, share it, star it. Thank you

### Code

It is hard to accept big PRs as of now because there are no tests yet. PRs for [`help wanted`](https://github.com/stereobooster/react-snap/issues?q=is:issue+is:open+label:%22help+wanted%22) tickets are very welcome.

You also can contribute to [minimalcss](https://github.com/peterbe/minimalcss), which is a big part of `react-snap`. Also, give some stars.
