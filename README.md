# Forbidden-Techniques-JS
Forbidden Techniques is a collection of my techniques developed frok mybexploration of the "forbidden" techniques in JS. Includes some weird JS quirks inspired by wtfjs. Largely this is how to effectively apply monkey patches and modify prototype chains.

## `await` keyword overloading

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
