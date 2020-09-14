---
layout: post
title: "Vanilla Haxe JS"
date: "2014-11-01"
---

I’ve seen it all in JavaScript, from Netscape 4’s years of pain, to jQuery to abstract browser differences, and finally to [Vanilla JS](http://vanilla-js.com/) and [node.js](http://nodejs.org/) – we now live in a world where coding in JavaScript is relatively sane and consistent.

Except we still have to use JavaScript. Not that I don’t like JavaScript – I’m quite found of the language and have sympathy for its quirks, but it’s just missing something to be productive once your code becomes more that half a dozen of classes.

Here comes the Haxe language and it’s almighty type inference!

![haxe-inference](/images/haxe-inference1.png "haxe-inference")  _Enjoy smart Haxe completion in [many popular editors](http://haxe.org/documentation/introduction/editors-and-ides.html)_

#### Vanilla JS?

Started as a kind of joke, Vanilla JS means working without jQuery and all those band-aid libraries because, quite frankly, they aren’t that useful nowadays if your project can target IE9+.

I enjoyed my time with jQuery but I’ve grown to prefer [manipulating my DOM elements myself](https://developer.mozilla.org/en-US/docs/Web/API/document) – it turns out the API is incredibly rich, not really complicated or more verbose, and it will run generally a lot faster than jQuery & co.

#### Vanilla Haxe JS?

Everything you can do in JavaScript you can do it in Haxe JS. Simple.

And you can do it better because of many little details and because Haxe is an amazing language that [improves on JavaScript](http://io.pellucid.com/blog/the-benefits-of-transpiling-to-javascript) while being syntactically very close to it.

#### Compiling to JavaScript?

There must be a reason why [compiling to JavaScript is incredibly popular](https://github.com/jashkenas/coffeescript/wiki/list-of-languages-that-compile-to-js) nowadays. And even if you’re coding in “pure” JavaScript you’re probably still using compile-like steps to browserify, minify, etc. so you’re just adding an additional step.

And you will barely feel like you are compiling because the Haxe compiler is so damn fast – even hundreds of classes take a pair of seconds to compile.

### A concrete example

Let’s say we want to show a list of items and a button to create new items. Simple enough but covering the different interesting bits.

#### Creating the view

Haxe is a strongly typed language, applying stricly type inference on all your code unless explicitly annotated as `Dynamic` or prefixed with `untyped`.

Notice how I added type annotations only where required (class members), while the rest is all inferred (and verified) by the compiler. Notice also that `this` is optional.

Haxe exposes the browser API through the `Browser` class. You can use `document.createElement` as usual, but Haxe adds aliases which return the right DIV/UL/LI/etc node type. Once compiled it’s all replaced by regular, inlined, `createElement(tagName)` calls.

```haxe
var view:DivElement;
var list:UListElement;
var button:ButtonElement;

function createChildren()
{
  var doc = Browser.document;

  view = doc.createDivElement();
  view.className = "listview";

  list = doc.createUListElement();

  button = doc.createButtonElement();
  button.textContent = "Add an item";

  view.appendChild(list);
  view.appendChild(button);
  doc.body.appendChild(view);
}
```

#### Loading JSON data

Haxe will use the built-in JSON API but you can force the inclusion of a fallback using the `-D haxeJSON` compiler directive (see [conditional compilation](http://haxe.org/manual/lf-condition-compilation.html)).

```json
[
	{ "label":"Item 1", "id":0, "date":"2014-10-27 12:00:00" },
	{ "label":"Item 2", "id":1, "date":"2014-10-27 14:00:00" },
	{ "label":"Item 3", "id":2, "date":"2014-10-27 16:00:00" },
	{ "label":"Item 4", "id":3, "date":"2014-10-27 18:00:00" },
	{ "label":"Item 5", "id":4, "date":"2014-10-27 20:00:00" }
]
```

I’m using Haxe’s crossplatform version of `XmlHttpRequest`. But you can obviously directly use `XmlHttpRequest` if you prefer, or `Browser.createXMLHttpRequest()` for a crossbrowser version.

Notice that you don’t need to use any trick to maintain the scope in the `onData` handler: you have access to the class’ `data` and `render` methods.

Notice `untyped alert(err)`: here I explicitly disable Haxe typing to write unverified JS code. Very handy to work with external APIs for which you don’t have compiler definitions.

```haxe
var data:Array<ItemInfo>;

function loadData()
{
  var loader = new Http("data.json");
  loader.onData = function(raw) {
    try {
      data = Json.parse(raw);
      if (!Std.is(data, Array))
        throw "ArgumentError: Json data is not an array";
      render();
    }
    catch (err:Dynamic) {
      untyped alert(err);
    }
  }
  loader.request();
}
```

Isn’t it curious that I wrote that the data is an `Array<ItemInfo>`? One of the features in Haxe I absolutely love is `typedef` types, aka "strongly typed duck-typing" (it's brilliant when working with JSON), so I defined the type of my JSON items as:

```haxe
typedef ItemInfo = {
	label:String,
	?date:String, // optional
	id:Int
}
```

#### Rendering

Again, let’s just use Vanilla JS to render the list. I'm leveraging the [document fragment API](https://developer.mozilla.org/en-US/docs/Web/API/Document.createDocumentFragment) to create the list items off screen and to append them all at once.

Notice the loop iterating on the array values – it’s generated as a `while` loop without the nasty side effects of JavaScript’s `for-in`.

```haxe
function render()
{
  var doc = Browser.document;
  var fragments = doc.createDocumentFragment();
  for (info in data)
  {
    var li = doc.createLIElement();
    var label = doc.createDivElement();
    label.textContent = info.label;
    li.appendChild(label);
    fragments.appendChild(li);
  }
  list.innerHTML = "";
  list.appendChild(fragments);
}
```

#### Adding new items

Easy stuff, especially thanks to Haxe’s scope conservation and type inference.

It is important to remind that the compiler will infer everything and will report errors if the typing is wrong:

- the `onclick` handler is required to have exactly one (Dynamic) parameter,
- I’m not allowed to push in `data` an object that doesn’t respect the `ItemInfo` typedef.

Bonus feature: [string interpolation](http://haxe.org/manual/lf-string-interpolation.html) to create the item’s label.

```haxe
button.onclick = function(\_) {
  data.push({
    label: 'New item ${data.length+1}',
    id: data.length
  });
  render();
}
```

### Conclusion

Honestly, doesn’t it look like better JavaScript? It’s brilliant I think.

You can get the [full sample project code is on Github](https://github.com/elsassph/vanilla-haxe-js).

#### Using external JavaScript libraries from Haxe

This isn’t ways easy so read my next article about [working with JavaScript libraries](http://philippe.elsass.me/2014/11/haxe-working-with-javascript-libraries/).

#### Interop from JavaScript

If you want to use your Haxe JS generated code from JavaScript, simply declare your class as `@:expose class MyClass` and you'll be able to write `new MyClass()` in your JavaScript. Easy and natural!

#### But, but, and jQuery?

Yes you can still use jQuery if you want – it’s in Haxe’s default JavaScript API. The only constraint is that you can’t use the `$()` shortcut but `new JQuery()`.

If you want to add externs for jQuery plugins you can use Haxe’s very handy [static extension methods](http://haxe.org/manual/lf-static-extension.html).
