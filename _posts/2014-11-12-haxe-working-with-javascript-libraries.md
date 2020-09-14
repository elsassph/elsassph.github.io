---
layout: post
title: "Haxe: working with JavaScript libraries"
date: "2014-11-12"
---

Let’s say you’re interested in using [Haxe for JavaScript](http://philippe.elsass.me/2014/11/vanilla-haxe-js/) development, but you are wondering how you are going to use the libraries you are used to.

Clearly this will be a little bit more complicated than when using languages like TypeScript, ES6/Traceur or CoffeeScript which essentially allow you to include any regular JavaScript without the compiler complaining.

Haxe is a different beast: you can’t just drop JavaScript code and expect it to be accepted. Let’s see the different strategies to use external JavaScript from Haxe.

## 1\. Don’t fight JavaScript

Honestly, sometimes the simplest way to use a JS library from Haxe is to… write JS.

### Why?

It’s easy to call global JS functions from Haxe, it’s hard to instantiate JS objects.

It works really well for anything that can be encapsulated behind a (few) global function(s) and for which you just want to copy&paste some sample code.

### How?

Write a side JS calling what has to be called from the JS library – just copy/paste/tweak what has to be written and put it inside a global JS function or “singleton” object.

From Haxe, just call your exposed functions thanks to the `untyped` keyword:

```haxe
// fire and forget
untyped window.alert("Hello");
untyped window.trackEvent("page1");

// exchanging data synchronously
var map:Element = untyped window.createGoogleMap(300, 300, pointsData);
view.appendChild(map);

// using callbacks
function onConnected() {
 trace("ready to SPAM!");
}
untyped window.facebookConnect(onConnected);
```

## 2\. Hack with JS injection

Although I’d generally recommend method 1., it’s fun to know the funky things you can do in Haxe so I’ll show you anyway ;)

### Why?

Maybe you don’t want a side JS, or maybe you really prefer the code to happen on the Haxe side but you don’t have good quality “externs” (see 3.) and don’t want to write them.

### How?

Thanks to untyped and the `__js__` magic function (note: `js.Syntax` in Haxe 4) you can inject code fragments into the generated JS code – it’s up to you to generate something that makes sense.

You can be “brutal” and insert a whole block of JS code:

```haxe
untyped __js__('
var params = { allowScriptAccess: "always" };
var atts = { id: "myytplayer" };
swfobject.embedSWF("http ://www.youtube.com/v/VIDEO_ID?enablejsapi=1&…",
 "ytapiplayer", "425", "356", "8", null, null, params, atts);
');
// multipline strings \o/
```

Or be more creative and insert just what is needed (Haxe doesn’t let you instantiate JS classes):

```haxe
function intialize() {
 var mapOptions = {
 center: { lat: -34.397, lng: 150.644},
 zoom: 8
 };
 var mapCanvas = Browser.document.getElementById("map-canvas");
 var map = untyped __new__("google.maps.Map", mapCanvas, mapOptions);
 // compiles as: var map = new google.maps.Map(mapCanvas, mapOptions);
 // ‘map’ is infered as Dynamic and you can call any function/property on it
}
untyped window.google.maps.event.addDomListener(window, "load", initialize);
```

If you’re “creative” you should be able to use simple JS APIs directly from Haxe, but without type-safety net.

## 3\. Use types definitions, or Haxe “externs”

If you’ve followed a tiny bit TypeScript, you’ll have heard of the their use of [types definitions](https://github.com/borisyankov/DefinitelyTyped) to hint the compiler when using JS libraries.

This isn’t a new concept if you think about it: C++ header files, old ActionScript 2 “intrinsics”, and Haxe “externs” serve the same role: telling the compiler that a number of types you want to use will be “available” when your code will run.

### What does it looks like?

```javascript
// JavaScript
function MyJSClass() {
}
MyJSClass.SOME_PROP = 42;
MyJClass.someFunc = function() {
 return "hello";
}
MyJSClass.prototype.myProp = null;
MyJSClass.prototype.myFunc = function(prop) {
 this.myProp = prop;
}
```

```haxe
// Haxe extern
extern class MyJSClass {
 static var SOME_PROP:Int;
 static function someFunc():Void;

 var myProp:String;
 function new();
 function myFunc(prop:String):Void;
}
```

Once you have such extern (it looks like an interface) in your classpath, the Haxe compiler will let you naturally manipulate external JS code, providing excellent inference and compile-time errors.

But how do you write externs for complex libraries if they doesn’t exist already? Sadly the answer is: with great courage, patience and a lot of free time… Although [it is possible to fully automate the process](https://github.com/Meychi/CreateJS-Haxe), it is more often entirely manual.

I won’t recommend you to jump right away on creating externs for a big JS library out there, especially if you aren’t an experienced and motivated Haxe developer: creating and keeping up to date such definitions require dedication; there is nothing more frustrating that discovering out of date (or incorrect) ones.

### Is there one already?

Generally, googling “haxelib nameOfTheLib” or “haxe nameOfTheLib” should tell you. At this stage you really want to find an exiting library. Spend the right time evaluating the completeness of what you find, but generally the best versions are already on Haxelib.

For instance for Pixi.js you’ll find [http://lib.haxe.org/p/pixijs](http://lib.haxe.org/p/pixijs "http://lib.haxe.org/p/pixijs") which means it’s available in Haxe’s packages.

Haxelib is Haxe’s package manager (think npm or bower) and using it is very simple:

installation
$ haxelib install pixijs

```haxe
// usage
import pixi.display.Stage;
import pixi.utils.Detector;

class Main {
 static public function main() {
 var renderer = Detector.autoDetectRenderer(600, 600);
 js.Browser.document.body.appendChild(renderer.view);
 var stage = new Stage(0x000000);
 renderer.render(stage);
 }
}
```

compilation
```bash
$ haxe -lib pixijs -main Main -js index.js
```

path to externs (all the class paths and extra compiler arguments)
```bash
$ haxelib path pixijs
C:/HaxeToolkit/haxe/lib/pixijs/0,2,7/
-D pixijs
```

Isn’t that simple?

### Patching externs

However it happens that the externs aren’t super up to date. First thing is to locate the project source, generally indicated in the Haxelib information page.

For [Pixi.js on Haxelib](http://lib.haxe.org/p/pixijs) it links to the [github haxe-pixi project](https://github.com/adireddy/haxe-pixi). This project is well maintained and the externs should be accurate, but that’s not always the case.

Don’t be afraid by outdated externs: updating them is tedious but easy, going through the doc and verifying the different classes signatures. Helping to keep externs up to date is extremely important for the community!

## Where to go now?

There are a number of well covered JS libraries: [CreateJS](http://lib.haxe.org/p/createjs-full), [Phaser](http://lib.haxe.org/p/phaser), [Angular](http://lib.haxe.org/p/angular.haxe), [Google Analytics](https://github.com/fbricker/haxe-ga), some libraries that would need to be updated, like [three.js](https://github.com/labe-me/haxe-three.js), and promising new libraries like [React.js](https://github.com/fponticelli/react.hx), [Mithril MVC](https://github.com/ciscoheat/mithril-hx).

Obviously don’t forget “native” Haxe libraries for JS development: [Haxe DOM](https://github.com/Blank101/haxe-dom) (client/server DOM manipulation), and standard crossplatform libraries like msignal, mloader, munit/mockatoo…

And finally: join the mailing list or the `#haxe` IRC channel to get help!
