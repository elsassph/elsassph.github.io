---
layout: post
title: "Code-splitting Haxe's JavaScript"
date: "2018-03-12"
---

Our goal, as web application developers, is to create applications that perform well for the user. Unfortunately, complexity is often unavoidable, and the size of the JavaScript payload degrades the user experience.

Essentially, the solution is to fraction this payload, so it is loaded on-demand as the user navigates around in your application and functionality is required. A smaller initial bundle size means faster load times - especially on mobile.

![splitting](/images/react-loadable-split-bundles.png)

This is called **code splitting**.

## Coming up with a plan

To improve the JavaScript payload, and figure where to cut it, you must understand where the weight comes from:

1. consider all the features of your application: pages (sitemap routes), dialogs/flows (authentication, registration, payment,...), complex isolated features (games, video player,...),
2. investigate the size impact of these features; counting source files / LOCs can give an idea, but it should be possible to know precisely the output size of each feature,
3. consider what is needed for your app to start up: many features could wait until the user interacts with the application,
4. use an explicit asynchronous syntax to request these features on demand, and let tools do the work.

## Solutions in the JavaScript world

[Webpack](https://webpack.js.org) has initiated the code-splitting trend a few years ago, by providing a practical, working, solution for JavaScript following the CommonJS or AMD module patterns. While it's use it getting more widespread, it is still a fairly advanced topic in the JavaScript community.

It is interesting to read how two popular JavaScript bundlers present this technique:

- [Webpack guide: code splitting](https://webpack.js.org/guides/code-splitting/)
- [ParcelJS: code splitting](https://parceljs.org/code_splitting.html)

### Size report

Webpack has a rich environment of [tools and plugins to analyse the size of a JavaScript project](https://survivejs.com/webpack/optimizing/build-analysis/). One tools I like in particular is [Webpack Bundle Analyser](https://www.npmjs.com/package/webpack-bundle-analyzer), which provides an interactive visual map of Webpack bundles, down to the individual size of each included file.

![Webpack Bundle Analyser](/images/bundleanalyzer.jpg)

### Asynchronous API

Bundlers typically use the dynamic import syntax:

```javascript
import('./pages/about').then(function (page) {
  // Render page
  page.render();
});
```

Simple code analysis allows to understand that `about.js` and its direct dependencies can be split into a separate bundle, and this Promise will return after the bundle is loaded.

### Avoiding duplication

Consider a dependency shared by several bundles (A requires C, B requires C, C is common): it makes sense to avoid duplicating common code in both bundles.

- Webpack < 4 recommends adding the (built-in) CommonsChunkPlugin plugin for that, which moves duplicated modules into a "common" bundle; [Webpack 4 introduces a new built-in system](https://gist.github.com/sokra/1522d586b8e5c0f5072d7565c2bee693) which is much more granular and automatic: several common chunks bundles will be created, optimised for the real bundles hierachy, and several vendor bundles may be created likewise instead of blindly grouping all the npm modules in a big common bundle loaded at start up,
- ParcelJS automatically move common modules in a "shared" bundle, as far as I can tell.

In both cases, there is a risk that if you have a lot of shared code, this additional JavaScript's file size will become large and will tend to be loaded immediately, adding to the initial payload.

## Solutions in the Haxe world

Haxe's JavaScript output is very compact and efficient, with features that JavaScript bundlers like [RollupJS](https://rollupjs.org) are only scratching the surface of (fast dead-code elimination, constant propagation, inlining). And the compiler can effortlessly scale to large codebases and stay 10x faster than typical JavaScript compilers.

However the compiler output is always a single JavaScript file - we could use some code splitting magic as well.

### Haxe Modular

[Haxe Modular](https://github.com/elsassph/haxe-modular) is a library I have been building and maturing over the last 2 years (!). It has been tested in several significant production projects now, and has been optimised to scale to unimaginable project sizes: _imagine tens of MBs of JavaScript split into thousands of bundles._

### Size report

Modular includes a size reporting feature inspired by the aforementioned Webpack Bundle Analyser.

Use it simply in your Haxe-JS project by adding these lines in your compiler options:

```
-lib modular
-D modular\_dump
```

After compilation, next to your JavaScript output, 2 additional files will appear:

- `{outputname}.stats.json`: detailed, per package and per class, byte size of the generated output,
- `{outputname}.stats.html`: interactive visualisation of your code size.

![Modular size report](https://pbs.twimg.com/media/DX1YW_7W4AEqMSm.jpg:large)

### Asynchronous API

Modular proposes a very natural asynchronous API:

```haxe
import MyClass;
...
Bundle.load(MyClass).then(function(_) {
 var c = new MyClass();
});
```

Simply by using this API, Modular will be able to emit a second `MyClass.js` file, which will be loaded (once) when this code runs, and when it completes, this class will be naturally available.

### Avoiding duplication, and advanced control

Modular has an advanced deduplication algorithm, "hoisting" redundant modules in the common parent of modules where duplication happens. For instance:

- A loads module B,
- B loads C and D,
- If C and D share code, it will move in their parent, B.

And that's only the most basic functionality of Modular; it is also possible to explicitly [split entire libraries](https://github.com/elsassph/haxe-modular/blob/master/doc/libraries.md), and if you have really unique needs, [advanced hooks](https://github.com/elsassph/haxe-modular/blob/master/doc/advanced.md) can give you even more control on the process.

### How does it compare with Webpack?

Modular comes in 2 flavours:

1. [Standalone Modular](https://github.com/elsassph/haxe-modular/blob/master/doc/start.md) is an easy, drop-in, addition to any regular Haxe-JS build process - it is very lightweight and unobstrusive, and gives you the essential splitting feature,
2. or you can embrace Webpack and use [Webpack Haxe Loader](https://github.com/jasononeil/webpack-haxe-loader) to use Haxe and benefit from this huge bundler ecosystem.

Webpack does more than JavaScript bundling: it is a **universal bundler**, so it will take care of your CSS and static assets as well. It is a significant investment to become a Webpack power user, but it is worth it.
