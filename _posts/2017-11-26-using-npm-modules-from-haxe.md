---
layout: post
title: "Using NPM modules from Haxe"
date: "2017-11-26"
---

When targeting JavaScript, using NPM modules is a must, and although there is nothing you can’t do with Haxe, it requires to understand how these modules work in order to write (even minimal) externs.

Many Haxe developers are not really familiar with CommonJS and ES6 module system; consuming NPM modules becomes a source of confusion / frustration. The goal of this post is to provide enough technical understanding to consume most modules.

## What is CommonJS?

CommonJS is the module system introduced by NodeJS; it is a clever and scalable pattern to create encapsulated functionalities. It has become the de-facto standard for JavaScript modularisation.

CommonJS is natively supported by NodeJS, but it requires a (simple to automate) transformation in order to use it in the browser.

Here’s a sample module source code:

```javascript
// hello.js
function say(who) {
  console.log('Hello ' + who);
}

// EITHER attach to default exports
exports.say = say;

// OR replace default exports
// with any JS value
module.exports = {
  say: say
};
```

And a sample consumer:

```javascript
// world.js
const hello = require('./hello');
hello.say('World');
```

The consumer “requires” the module, which is **evaluated once, synchronously, and cached**; the function call then literally returns the value of `module.exports`.

## What about ES6 (ES2015) modules?

ES6 has defined modules following the principles of CommonJS; `import` keyword is syntaxic sugar for `require`. TypeScript is following the same syntax.

Access exports object:

```javascript
// hello.js
module.exports = function say() {}

// JS require
const say = require('hello');

// ES6 import
import * as say from 'hello';
```

Default export:

```javascript
// hello.js
exports.default = function say() {}
// or ES6
export default function say() {}

// JS require
const say = require('hello').default;

// ES6 import
import say from 'hello';
```

Named exports:

```javascript
// hello.js
exports.say = function say() {}
// or ES6
export function say() {}

// JS require
const say = require('hello').say;

// ES6 import
import { say } from 'hello';
```

“Namespaced” exports:

```javascript
// hello.js
exports.say = function say() {}
// or ES6
export function say() {}

// JS require
const hello = require('hello');

// ES6 import
import * as hello from 'hello';
hello.say();
```

Those are the 4 cases you’ll use – and you can even combine them, like default and named exports (`import defaultSay, { namedSay } from 'hello'`).

What matters is that you have to understand this mapping to comfortably use them from Haxe because code samples that you will see nowadays will be based on the ES6 import syntax.

## Consuming NPM modules from Haxe

Haxe essentially supports CommonJS natively, directly or through annotation. Both cases compile to CommonJS’ `require` and, if you want to run in a browser (nodejs supports it natively), you will have to post-process the Haxe JS output using for example [Browserify](http://browserify.org/) or [Webpack](https://github.com/jasononeil/webpack-haxe-loader).

Direct require:

```haxe
// hello.js
const hello = require('hello');
hello.say('World');

// Haxe
var hello = js.Lib.require('hello');
hello.say('World')

// Which can be shortened into
import js.Lib.require;
...
var hello = require('hello');
```

And extern classes:

```haxe
// hello.js
export default class DefaultClass {...}
export class NamedClass {...}

// JS usage
import DefaultClass from 'hello'; // require('hello').default
import { NamedClass } from 'hello'; // require('hello').NamedClass

// Haxe usage
@:jsRequire('hello', 'default')
extern class DefaultClass {
    /* declarations like in an interface */
}

@:jsRequire('hello', 'NamedClass')
extern class NamedClass {
    /* declarations like in an interface */
}
```

## About React components

React components are an interesting advanced case: for the compiler to accept these external component classes, they have to extend ReactComponent:

```javascript
// JS
import * as Bar from 'react-bar'; // require('react-bar')
import { Foo } from 'react-foo'; // require('react-foo').Foo
```

```haxe
// Haxe
@:jsRequire('react-bar')
extern class Bar extends ReactComponent {
}

@:jsRequire('react-foo', 'Foo')
extern class Foo extends ReactComponent {
}

// or if you want typed props
@:jsRequire('react-foo', 'Foo')
extern class Foo extends ReactComponentOfProps<FooProps> {
}
typedef FooProps = {
    param1: String,
    param2: Int
}
```

## What about CSS/Json/etc?

Oddly, it has become popular to require/import modules which are not code, like CSS.

UI components sometimes come with their styles and you may be confused by ES6 code samples:

```javascript
// JS
import { HLayout, HLayoutItem } from 'react-flexbox-layout';
import 'react-flexbox-layout/lib/styles.css';
```

You should now be able to figure how to write the classes externs, but what about the CSS import? The basics is that you need to use an explicit `require` somewhere in your code, for instance in your main entry point.

```haxe
// Haxe entry point
static function main() {
    js.Lib.require('react-flexbox-layout/lib/styles.css');
}
```

However, danger: by default Browserify and Webpack do not understand CSS; both require the set up of a “loader” for non-code modules ([browserify-css](https://www.npmjs.com/package/browserify-css), [Webpack style-loader](https://webpack.js.org/guides/asset-management/#loading-css)).

Moreover if you’re using [Haxe Modular](https://github.com/elsassph/haxe-modular), you should not require the CSS in your Haxe code but rather in the `libs.js` which is the only file really processed by Browserify.

## Final words

Hopefully you’re read so far, and now you know everything about consuming NPM modules from Haxe code!

If you’re into Haxe JavaScript, make sure to check my projects allowing the easy code splitting of Haxe applications, with or without Webpack, yet both supporting hot-reload of React components!

- [Haxe Modular](https://github.com/elsassph/haxe-modular); production-proven, standalone, solution for JS code splitting,
- [Haxe Webpack Loader](https://github.com/jasononeil/webpack-haxe-loader); state of the art integration of Haxe with Webpack.
