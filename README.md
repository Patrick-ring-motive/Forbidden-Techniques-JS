# Forbidden-Techniques-JS
Forbidden Techniques is a collection of my techniques developed from my exploration of the "forbidden" techniques in JS. Includes some weird JS quirks inspired by wtfjs. Largely this is how to effectively apply monkey patches and modify prototype chains. It seems that this type of exploration often goes missed in the modern learning paths. These are things you should probably never do in a production application because they can corrupt your runtime, cause various memory leaks, or trigger infinite recursion if not handled correctly.

## Table of Contents
- [1. `await` keyword overloading](#1-await-keyword-overloading)
- [2. Character data interpreted in XHTML](#2-character-data-interpreted-in-xhtml)
- [3. Basic monkey-patch on fetch](#3-basic-monkey-patch-on-fetch)
- [4. Advanced Monkey Patch XHR](#4-advanced-monkey-patch-xhr)
- [5. Modifying read-only NodeList](#5-modifying-read-only-nodelist)
- [6. Frozen Objects - Adding Properties](#6-frozen-objects---adding-properties)
- [7. Frozen Objects - Modifying Existing Properties](#7-frozen-objects---modifying-existing-properties)
- [8. Frozen Objects - Modify Anything](#8-frozen-objects---modify-anything)
- [9. Sync Blob Parse](#9-sync-blob-parse)
- [10. Short Circuit Promises](#10-short-circuit-promises)
- [11. Node in Google Colab](#11-Node-in-Google-Colab)
- [12. Short Circuit Promises with `util.inspect()`](#12-short-circuit-promises-with-utilinspect)
- [13. Idempotent `fetch`](#13-Idempotent-fetch)

## 1. `await` keyword overloading

`await` is a keyword in js that works in async code blocks to wait for a promise to resolve. While it is a keyword, it is not reserved. This means it can be overloaded at the global level. When you do this, you can get some interesting behavior. Consider this [example](https://codepen.io/Patrick-Ring/pen/azvVNKq).

```html

<script>
  globalThis.await = _ => _;
  console.alog = async (x) => {
    await "wtf";
    console.log(x);
  };
</script>

<script>
  await(console.alog(1));
  console.log(2);
  // prints 2 then 1
</script>

<script type="module">
  await(console.alog(1));
  console.log(2);
  // prints 1 then 2
</script>

```

What's happening is we are assigning a property `await` to the global object that just takes a parameter and returns it. So now when you call `await(x);` in synchronous code, you get the new behavior but when you call `await(x);` in async code you get the original behavior. Regular script tags run synchronously at the top level so there is no waiting. In modules, the top level runs asynchronously so the result is awaited as you might expect.


## 2. Character data interpreted in XHTML

`script` tags are interpreted as [CDATA](https://en.m.wikipedia.org/wiki/CDATA) in typical html but in xhtml characters are interpreted using the standard syntax so for example, `<` and `&` are represented by `&lt;` and `&amp;`. You may think this is irrelevant in modern web because nobody uses xhtml but they do. Any time a `script` tag is nested inside an `svg` tag, it is interpreted as xhtml. See this [example](https://codepen.io/Patrick-Ring/pen/VYvrjJa):

```html

<script type="module">
  const gt = 5;
  const test = 5 &gt; + 7;
  console.log(+test); // > 5
</script>
<svg>
  <script type="module">
    const gt = 5;
    const test = 5 &gt; + 7;
    console.log(+test); // > 0
  </script>
</svg>

```

In the first script `test` is set to `5 & gt` which is `5 & 5` resulting in `5`. In the second script `&gt;` is converted to `>` so `test` is set to `5 > +7` which is `false`. Then `+test` is coerced into `0`.

## 3. Basic monkey-patch on fetch
Monkey patching is modifying in-built JS functions with custom behavior. you have to be very careful how you go about this to not break other people's code. `fetch` is the most common function that I generally monkey patch. This example is to one I use to catch any errors and convert them to http response errors. This helps keep error handling consistent amd not having to duplicate code. You want to either do this or throw on http errors.

```js
// start with an IIFE to contain everything in closures.
(() => {
  // _fetch stores the original fetch function as a closure variable
  const _fetch = self.fetch;
  // set the prototype of the new function to the old function so we inherit any other custom modifications done by others.
  self.fetch = Object.setPrototypeOf(async function fetch(...args) {
    try {
      // be sure to await or errors won't get caught
      return await _fetch.apply(this, args);
    } catch (e) {
      return new Response(e.stack, {
        status: 469,
        statusText: e.message
      });
    }
  }, _fetch);
})();
```

## 4. Advanced Monkey Patch XHR

Second to fetch is the older api for network calls XMLHttpRequest. Its a bit more oddly shaped. This patch blocks requests containing certain strings in the url.


```html
<script>
  // Wrap in IIFE to create non polluting closures
  (() => {
    // fallback stringifier
    const stringify = x => {
      try {
        return JSON.stringify(x);
      } catch {
        return String(x);
      }
    };
    // blocks is a list of strings we intend to filter out
    const blocks = ["example.com", "example.net"];
    // Create a closure map to store instance properties that are in accessible to external viewers
    // use WeakMap if available for better memory management but regular map also works
    const $Map = self?.WeakMap ?? Map;
    // storing the input arguments to the open method so we can access them later in the send method
    const _openArgs = new $Map();
    // List of objects in the xhr api, xhr event target is the parent class so we want to patch it last
    for (const xhr of [XMLHttpRequest, XMLHttpRequestUpload, XMLHttpRequestEventTarget]) {
      try {
        // extra IIFE layer for additional closures
        (() => {
          // store the original open method
          const _open = xhr.prototype.open;
          if (!_open) return;
          // set up inheritance between new method to old one to maintain other customizations from others
          xhr.prototype.open = Object.setPrototypeOf(function open(...args) {
            // store input args in closure map
            _openArgs.set(this, args);
            return _open.apply(this, args);
          }, _open);
        })();

        (() => {
          // store the original send method
          const _send = xhr.prototype.send;
          if (!_send) return;
          // set up inheritance between new method to old one to maintain other customizations from others
          xhr.prototype.send = Object.setPrototypeOf(function send(...args) {
            // store input args in closure map
            const openArgs = _openArgs.get(this) ?? [];
            for (const arg of openArgs) {
              const sarg = stringify(arg);
              for (const block of blocks) {
                if (sarg.includes(block)) return;
              }
            }
            return _send.apply(this, args);
          }, _send);
        })();

        // patching a property is similar to patching a method but only for the property getter
        // this example block the response if it contains one of our string representations
        for (const res of ['response', 'responseText', 'responseURL', 'responseXML']) {
          (() => {
            const _response = Object.getOwnPropertyDescriptor(xhr.prototype, res)?.get;
            if (!_response) return;
            Object.defineProperty(xhr.prototype, res, {
              configurable: true,
              enumerable: true,
              get: Object.setPrototypeOf(function response() {
                for (const block of blocks) {
                  // block request if it matches list
                  if (stringify(x).includes(block)) {
                    console.warn('blocking xhr response', stringify(x));
                    // return the expected object type but empty
                    return Object.create(_response.call(this)?.__proto__);
                  }
                }
                return _response.call(this);
              }, _response)
            });
          })()
        }
      } catch {}
    }
  })();
</script>
```

## 5. Modifying read-only NodeList
In this example we modify a NodeList which has a fixed set of nodes. We can change it by taking a new object with new properties and inserting it into the prototype chain between the NodeList and the Nodelist prototype.
```html
<div></div><div></div><div></div>
<wtf></wtf>
<script>
  const arr = document.querySelectorAll('div');
  arr[3] = document.querySelector('wtf');
  console.log(arr[3]); // > undefined
  const insert = {
    "3": document.querySelector('wtf')
  };
  Object.defineProperty(arr, 'length', {
    value: arr.length + 1,
    configurable: true,
    writable: true
  });
  [insert.__proto__, arr.__proto__] = [arr.__proto__, insert];
  console.log(arr); // > [<div/>,<div/>,<div/>,<wtf/>]
</script>
```
We can extrapolate this out into a push method.
```html
<script>
  (() => {
    NodeList.prototype.push = function push(x) {
      // try the proper way first by appending to the parent element
      if (x instanceof Node) {
        if (this[0]?.parentNode?.childNodes === this) {
          this[0].parentNode.appendChild(x);
          return this.length;
        }
        if (this[0]?.parentElement?.childNodes === this) {
          this[0].parentElement.appendChild(x);
          return this.length;
        }
      }
      // if the elements don't share a common parent then apply this hack
      const insert = {};
      insert[this.length] = x;
      Object.defineProperty(this, 'length', {
        value: this.length + 1,
        configurable: true,
        writable: true
      });
      [insert.__proto__, this.__proto__] = [this.__proto__, insert];
      return this.length;
    };
  })();

  const nodes = document.querySelectorAll('div');
  nodes.push(document.querySelector('wtf'));
  console.log(nodes); // > [<div/>,<div/>,<div/>,<wtf/>]
</script>
```

## 6. Frozen Objects - Adding Properties
We can modify frozen objects by appending properties on the prototype that are returned based on a map keyed by the original object.

```html
<script>
  // get our frozen object
  const froze = Object.freeze({});
  const unfreeze = (() => {
    const hasProp = (obj, prop) => {
      try {
        return !!Object.getOwnPropertyDescriptor(obj, prop);
      } catch {}
    };
    // create a map to store additional object properties
    const $Map = self.WeakMap ?? Map;
    const keyMap = new $Map();
    return (obj, key, val) => {
      const proto = obj.__proto__;
      // if the object already has this property then this trick wont work
      // if the prototype already has this property then this trick would corrupt the prototype
      if (hasProp(obj, key) || hasProp(proto, key)) return;
      const objMap = keyMap.get(obj) ?? Object.create(null);
      objMap[key] = val;
      keyMap.set(obj, objMap);
      Object.defineProperty(proto, key, {
        get() {
          const objMap = keyMap.get(this) ?? Object.create(null);
          keyMap.set(this, objMap);
          return objMap[key];
        },
        set(x) {
          const objMap = keyMap.get(this) ?? Object.create(null);
          keyMap.set(this, objMap);
          return objMap[key] = x;
        },
        enumerable: true,
        configurable: true
      });
    };
  })();
  unfreeze(froze, 'test', 7);
  console.log(froze.test); // > 7
  froze.test = 8;
  console.log(froze.test); // > 8
  let x = {};
  x.test = 'gg';
  console.log(x.test); // > 'gg'
  console.log(froze.test); // > 8  
</script>
```

## 7. Frozen Objects - Modifying Existing Properties
We can modify non-primitive properties of frozen objects by essentially redirecting everything on property object to a new value. This can also be used to redefine a `const` in place. Keep in mind that this in place modification effects every reference to this property object. 
```html
<script>
  //shorthand for defining properties on objects
  const objDoProp = function(obj, prop, def, enm, mut) {
    return Object.defineProperty(obj, prop, {
      value: def,
      writable: mut,
      enumerable: enm,
      configurable: mut,
    });
  };
  const objDefProp = (obj, prop, def) => objDoProp(obj, prop, def, false, true);
  const objDefEnum = (obj, prop, def) => objDoProp(obj, prop, def, true, true);

  // fallback to writeable if configurable is false
  const objWriteProp = (obj, prop, def) => {
    try {
      const old = Object.getOwnPropertyDescriptor(obj, prop);
      if (old?.writable && !old?.configurable) {
        obj[prop] = def;
      } else {
        objDefProp(obj, prop, def);
      }
    } catch {}
  };

  const objWriteEnum = (obj, prop, def) => {
    try {
      const old = Object.getOwnPropertyDescriptor(obj, prop);
      if (old?.writable && !old?.configurable) {
        obj[prop] = def;
      } else {
        objDefEnum(obj, prop, def);
      }
    } catch {}
  };

  const getKeys = x => {
    try {
      return Reflect.ownKeys(x);
    } catch {
      return [];
    }
  };

  //assign all properties from src to target
  //bind functions to src when assigning to target
  function assignAll(target, src) {
    const excepts = ["prototype", "constructor", "__proto__"];
    const enums = [];
    let source = src;
    while (source) {
      for (const key in source) {
        try {
          if (excepts.includes(key)) {
            continue;
          }
          objWriteEnum(target, key, source[key]?.bind?.(src?.valueOf?.() ?? src) ?? source[key]);
          enums.push(key);
        } catch {}
      }
      for (const key of getKeys(source)) {
        try {
          if (enums.includes(key) || excepts.includes(key)) {
            continue;
          }
          objWriteProp(target, key, source[key]?.bind?.(src?.valueOf?.() ?? src) ?? source[key]);

        } catch {}
      }
      // walk up the prototype chain for more properties
      source = Object.getPrototypeOf(source);
    }
    // make sure identifying properties point to src
    for (const identity of ["valueOf", "toString", "toLocaleString", Symbol.toPrimitive]) {
      try {
        objWriteProp(target, identity, () => src);
      } catch {}
    }
    try {
      Object.defineProperty(target, Symbol.toStringTag, {
        configurable: true,
        enumerable: true,
        get: () => src
      });
    } catch {}
    // finally assign the prototype of src to target
    try {
      target.__proto__ = src.__proto__;
    } catch {}
    return target;
  }

  const obj = {};

  console.log(assignAll(obj, new Response("cheese"))); // > [object Response]

  (async () => console.log(await obj.text()))(); // > "cheese"

  const froze = Object.freeze({ prop:{} });
  assignAll(froze.prop, "hello");
  console.log(`${froze.prop} world`); // > hello world
</script>
```

## 8. Frozen Objects - Modify Anything

The best way to modify frozen objects is to never let them freeze in the first place. You can do this by monkey patching all the ways that things get frozen.

```js

  (() => {
    const _freeze = Object.freeze;
    Object.freeze = Object.setPrototypeOf(function freeze(obj) {
      return obj;
    }, _freeze);
  })();

  (() => {
    const _seal = Object.seal;
    Object.seal = Object.setPrototypeOf(function seal(obj) {
      return obj;
    }, _seal);
  })();

  (() => {
    const _preventExtensions = Object.preventExtensions;
    Object.preventExtensions = Object.setPrototypeOf(function preventExtensions(obj) {
      return obj;
    }, _preventExtensions);
  })();

  (() => {
    const _preventExtensions = Reflect.preventExtensions;
    Reflect.preventExtensions = Object.setPrototypeOf(function preventExtensions(obj) {
      return true;
    }, _preventExtensions);
  })();

  (() => {
    const _defineProperty = Object.defineProperty;
    Object.defineProperty = Object.setPrototypeOf(function defineProperty(obj, prop, desc) {
      return _defineProperty(obj, prop, {
        ...desc,
        configurable: true
      })
    }, _defineProperty);
  })();

  (() => {
    const _defineProperties = Object.defineProperties;
    Object.defineProperties = Object.setPrototypeOf(function defineProperties(obj, desc) {
      for (const key in desc) {
        desc[key].configurable = true;
      }
      for (const key of Reflect.ownKeys(desc)) {
        desc[key].configurable = true;
      }
      return _defineProperties(obj, desc)
    }, _defineProperties);
  })();

  (() => {
    const _defineProperty = Reflect.defineProperty;
    Reflect.defineProperty = Object.setPrototypeOf(function defineProperty(obj, prop, desc) {
      return _defineProperty(obj, prop, {
        ...desc,
        configurable: true
      })
    }, _defineProperty);
  })();

```
After applying this patch, every attempt to freeze an object will leave it as mutable as before. This will break anything that depends on immutability.

## 9. Sync Blob Parse

On a `Blob`, calling `text()` [returns a promise](https://developer.mozilla.org/en-US/docs/Web/API/Blob/text). However there are some tricks you can do to synchronously unravel a blob. One way that only works in web workers is to use [`FileReaderSync`](https://developer.mozilla.org/en-US/docs/Web/API/FileReaderSync). Another way that works on the main thread is to exploit synchronous [`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Synchronous_and_Asynchronous_Requests#synchronous_request).

```js
  // synchronously turn a blob into text
  function blobText(blob) {
    if (typeof FileReaderSync) {
      return new FileReaderSync().readAsText(blob);
    }
    // create blob url
    const url = URL.createObjectURL(blob);
    // create an ajax request targeted ar rge blob url
    // set async to false
    const xhr = new XMLHttpRequest();
    xhr.open('GET', url, false);
    // execute the "network" request
    xhr.send();
    //return the response as text
    return xhr.responseText;
  };
  // test 
  const helloWorlBlob = new Blob(['Hello World']);
  const helloWorldText = blobText(helloWorlBlob);
  console.log(helloWorldText);
```

## 10. Short Circuit Promises

When you have a promise, you must call `await` in an async context in order to get the resolved value. When you call `await`, everything on the call stack and the micotask queue will execute before the async function continues even if the promise is already settled. The simplest way to shortcircuit this is to use a value assignment within async code. Then you can check if the promise is resolved before using `await`.

```html
<script type="module">
  let value = new Promise(resolve => resolve("hello"));
  (async () => value = await value)();
  console.log(value?.constructor?.name, value);
  if (value instanceof Promise) await "anything"
  console.log(value?.constructor?.name, value);
</script>
```

We can package this up in a simple wrapper class
```html
<script type="module">
  class PromiseWrapper {
    constructor(promise) {
      this.promise = promise;
      (async () => {
        try {
          this.value = await promise;
        } catch (e) {
          this.error = e;
        }
      })();
    }
  }
  const value = new Promise(resolve => resolve("hello"));
  const wrap = new PromiseWrapper(value);
  console.log(wrap);
  await value;
  console.log(wrap);
</script>
```

## 11. Node in Google Colab

This is not really a JavaScript hack but a Google Colab trick. While primarily used for Python, Google Colab had multiple ways to run JavaScript. Google Colab instances come preinstalled with NodeJS and this makes it useful to me to share JS tricks that are specific to NodeJS. [Here's how it is invoked](https://colab.research.google.com/drive/1x3Aeiz08wmWaatPar6MgCBbvMutl7FRW?usp=sharing).

```js
%%bash
node -e "$(cat <<-END

  console.log('hello world');

END
)"

```

`%%bash` turns the cell into a Bash script from which we can invoke `node -e` to run serverside code. `%%javascript` can be use but this only runs code on the frontend in a sandbox.


## 12. Short Circuit Promises with `util.inspect()`

Using the above colab trick I can share the NodeJS that makes use of [`util.inspect()` to synchronously unwrap a promise](https://colab.research.google.com/drive/1a1dQVNql-EQlBgNifh5A3vt3zL5fGWOj?usp=sharing).

```js
%%bash
node -e "$(cat <<-END

  //import util.inspect
  const { inspect } = require("util");
  //create a simple promise
  const promise = (async () => "hello world")();
  //inspect checks internals without needing to await anything
  const value = inspect(promise).slice(11, -3);
  console.log(value); //> hello world

END
)"
```

Notice how `"hello world"` is never awaited or assigned directly. `util.inspect()` uses Node internals to peek into the promise.

## 13. Idempotent `fetch`

You'll see `Request` and `Response` objects have consumable contents. So calling `response.text()` will give you the content as text the first time but will throw an error if called again. This optimization exists to prevent browser memory from filling up. While this makes sense geberally, it is not how most objects in JS work and can be hard to wrap ypur head around. If you use `response.clone().text()` instead, you can call it multiple times and on most files this will not cause any issues. Files would have to be very large to have any sort of negative impact. Using the monkey patch below, you can bake this cloning behavior in by default.

```html
<script type="module">
  (() => {
    // Non-leaking IIFE
    // Apply to both request and response
    for (const r of [Request.prototype, Response.prototype]) {
      // Apply to all functions that can consume the body
      for (const fn of ['arrayBuffer', 'blob', 'bytes', 'formData', 'json', 'text']) {
        // skip if doesn't exist
        if (typeof r[fn] !== 'function') continue;
        // store the native function
        const _fn = r[fn];
        // Shadow the native function with a wrapper that clones first
        r[fn] = Object.setPrototypeOf(function() {
          return _fn.call(this.clone());
        }, _fn);
      }
      // Apply to the getter of the body itself
      const _body = Object.getOwnPropertyDescriptor(r, 'body').get;
      if (_body) {
        Object.defineProperty(r, 'body', {
          get:Object.setPrototypeOf(function body(){
          return _body.call(this.clone());
        },ReadableStream),
        });
      }
    }
  })();

  // clone inputs to the constructors so they don't get consumed
  (()=>{
    const _Request = Request;
    const $Request = class Request extends _Request{
      constructor(...args){
         super(...args.map(x=>x?.clone?.() ?? x));
      }
    };
    globalThis.Request = $Request;
  })();

  (()=>{
    const _Response = Response;
    const $Response = class Response extends _Response{
      constructor(...args){
         super(...args.map(x=>x?.clone?.() ?? x));
      }
    };
    globalThis.Response = $Response;
  })();

  // patch fetch to not consume requests
  (()=>{
    const _fetch = fetch;
    globalThis.fetch = Object.setPrototypeOf(function fetch(...args){
      return _fetch.apply(this,args.map(x?.clone?.() ?? x)):
    },_fetch);
  })();

  const res = new Response('asdf');
  console.log(await res.text()); //> asdf
  console.log(await res.text()); //> asdf
</script>
```

This can be particularly useful when doing your own clientside caching and preventing async race conditions.

## 14. Prototype Pollution

Strings are iterable in JavaScript, but they don't normally inherit all
of `Array.prototype`. With some prototype pollution, we can inject every
array method into strings, making them behave like hybrid string/array
objects. This can lead to some very cursed behavior.

``` html
<script>
(()=>{
  const isType = (x,type) => 
    typeof x === String(type).toLowerCase() 
    || x instanceof globalThis[type] 
    || x?.constructor?.name === type
    || globalThis[type]?.[`is${type}`]?.(x);

  const to = {
    String : (x)=>[...x].every(s=>isType(s,'String'))?[...x].join(''):x,
    Set    : (x)=>new Set(x),
  };

  for(const type of ['String','Set','NodeList','HTMLCollection']){
    for(const prop of Reflect.ownKeys(Array.prototype)){
      if(typeof Array.prototype[prop] !== 'function') continue;
      (globalThis[type]?.prototype ?? {})[prop] ??= function(...args){
        const res = [...this][prop](...args);
        return isType(res,'Array') ? (to[type]?.(res) ?? res) : res;
      };
    }
  }
})();
</script>
```

Now every string in your runtime inherits array methods:

``` js
console.log("cheese".map(c => c.toUpperCase())); 
// CHEESE

console.log("abc".filter(c => c > "a")); 
// bc

console.log("cool".join("-")); 
// c-o-o-l

console.log("dank".reverse()); 
// knad
```

### What's happening

-   We loop through every property of `Array.prototype`.
-   If it's a function (like `.map`, `.filter`, `.reverse`), we copy it
    onto `String.prototype`.
-   When called on a string, we spread it into characters (`[...this]`),
    run the array method, and join back into a string if possible.

### Why this is cursed

-   **Identity confusion:** `"foo".map` suddenly exists, violating
    assumptions.
-   **Hidden allocations:** Every method call spreads the string into an
    array, creating extra memory churn.
-   **Polyfill collisions:** Any code that checks for missing string
    methods may break.
-   **Security pitfalls:** Enumerating `String.prototype` now reveals a
    bunch of extra methods, which may surprise libraries.

This technique is extremely powerful but will break *everything* that
assumes strings don't behave like arrays.
