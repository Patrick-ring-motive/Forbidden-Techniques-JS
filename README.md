# Forbidden-Techniques-JS
Forbidden Techniques is a collection of my techniques developed frok mybexploration of the "forbidden" techniques in JS. Includes some weird JS quirks inspired by wtfjs. Largely this is how to effectively apply monkey patches and modify prototype chains.

## 1. `await` keyword overloading

`await` is a keyword in js that works in async code blocks to wait for a promise to resolve. While it is a keyword, it is not reserved. This means it can be overloaded at the global level. When you do this, you can get some interesting behavior. Consider this [example](https://codepen.io/Patrick-Ring/pen/azvVNKq).

```html

<script>
globalThis.await=_=>_;
console.alog = async(x)=>{
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

## 3. Basics monkey-patch on fetch
Monkey patching is modifying in-built JS functions with custom behavior. you have to be very careful how you go about this to not break other people's code. `fetch` is the most common function that I generally monkey patch. This example is to one I use to catch sny errors and convert them to http response errors. this helps keep error handling consistent amd notvhaving to duplicate code.

```js
// start with an IIFE to contain everything in closures.
(() => {
    // _fetch stores the original fetch function as a closure variable
    const _fetch = self.fetch;
    // set the prototype of the new function to the old function so we inherit any other custom modifications done by others.
    self.fetch = Object.setPrototypeOf(async function fetch(...args) {
    try{
      // be sure to await or errors won't get caught
      return await _fetch.apply(this, args);
    }catch(e){
      return new Response(e.stack,{
        status:469,
        statusText:e.message
      });
    }
  }, _fetch);
})();
```

## 4. Advanced Monkey Patch XHR

Second to fetch is the older api for network calls XMLHttpRequest. Its a bit more oddly shaped. This patch blocks requests containing certain strings in the url.


```js
  (() => {
    const blocks = ["example.com", "example.net"];
    const $Map = self?.WeakMap ?? Map;
    const _openArgs = new $Map();
    const xhrs = [XMLHttpRequest, XMLHttpRequestUpload, XMLHttpRequestEventTarget];
    for (const xhr of xhrs) {
      (() => {
        const _open = xhr.prototype.open;
        if (!_open) return;
        xhr.prototype.open = Object.setPrototypeOf(function open(...args) {
          _openArgs.set(this, args);
          return _open.apply(this, args);
        }, _open);
      })();
      (() => {
        const _send = xhr.prototype.send;
        if (!_send) return;
        xhr.prototype.send = Object.setPrototypeOf(function send(...args) {
          const openArgs = JSON.stringify(_openArgs.get(this) ?? []);
          _openArgs.delete(this);
          for (const block of blocks) {
            if (openArgs.includes(block)) {
              console.warn('blocking xhr', ...args);
              return;
            }
          }
          return _send.apply(this, args);
        }, _send);
      })();
    }
  })();
```

