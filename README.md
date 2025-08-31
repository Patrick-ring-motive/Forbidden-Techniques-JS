# Forbidden-Techniques-JS
Forbidden Techniques is a collection of my techniques developed frok mybexploration of the "forbidden" techniques in JS. Includes some weird JS quirks inspired by wtfjs. Largely this is how to effectively apply monkey patches and modify prototype chains.

TODO: toc

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

## 3. Basic monkey-patch on fetch
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
     try{
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
          for(const arg of openArgs){
            const sarg = stringify(arg);
            for(const block of blocks){
              if(sarg.includes(block))return;
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
     }catch{}
    }
  })();
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
const insert = {"3":document.querySelector('wtf')};
Object.defineProperty(arr,'length',{
  value: arr.length+1,
  configurable:true,
  writable:true
});
[insert.__proto__,arr.__proto__] = [arr.__proto__,insert];
console.log(arr); // > [<div/>,<div/>,<div/>,<wtf/>]
</script>
```
We can extrapolate this out into a push method.
```html
<script>
(()=>{
NodeList.prototype.push = function push(x){
  // try the proper way first
if(x instanceof Node){
  if(this[0]?.parentNode?.childNodes === this){
    this[0].parentNode.appendChild(x);
    return this.length;
  }
  if(this[0]?.parentElement?.childNodes === this){
    this[0].parentElement.appendChild(x);
    return this.length;
  }
}
const insert = {};
insert[this.length] = x;
Object.defineProperty(this,'length',{
  value: this.length+1,
  configurable:true,
  writable:true
});
[insert.__proto__,this.__proto__] = [this.__proto__,insert];
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
        const hasProp = (obj,prop)=>{
          try{
            return !!Object.getOwnPropertyDescriptor(obj,prop);
          }catch{}
        };
        // create a map to store additional object properties
        const $Map = self.WeakMap ?? Map;
        const keyMap = new $Map();
        return (obj, key, val) => {
          const proto = obj.__proto__;
          // if the object already has this property then this trick wont work
          // if the prototype already has this property then this trick would corrupt the prototype
          if(hasProp(obj,key)||hasProp(proto,key))return;
          const objMap = keyMap.get(obj) ?? Object.create(null);
          objMap[key] = val;
          keyMap.set(obj,objMap);
          Object.defineProperty(proto, key, {
            get() {
              const objMap = keyMap.get(this) ?? Object.create(null);
              keyMap.set(this,objMap);
              return objMap[key];
            },
            set(x) {
              const objMap = keyMap.get(this) ?? Object.create(null);
              keyMap.set(this,objMap);
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
We can modify non-primitive properties of frozen objects by essentially redirecting everything on property object to a new value. This can also be used to redefine a `const` in place. Keep in mind that this in place modification effects every reference to this propert object. 
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
const objWriteProp = (obj,prop,def)=>{
  try{
    const old = Object.getOwnPropertyDescriptor(obj,prop);
    if(old?.writable && !old?.configurable){
      obj[prop] = def;
    }else{
      objDefProp(obj,prop,def);
    }
  }catch{}
};

const objWriteEnum = (obj,prop,def)=>{
  try{
    const old = Object.getOwnPropertyDescriptor(obj,prop);
    if(old?.writable && !old?.configurable){
      obj[prop] = def;
    }else{
      objDefEnum(obj,prop,def);
    }
  }catch{}
};

const getKeys = x =>{
  try{
    return Reflect.ownKeys(x);
  }catch{
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
  for(const identity of ["valueOf","toString","toLocaleString",Symbol.toPrimitive]){
    try{
      objWriteProp(target,identity,()=>src);
    }catch{}
  }
  try{
    Object.defineProperty(target,Symbol.toStringTag,{
      configurable:true,
      enumerable:true,
      get:()=>src
    });
  }catch{}
  // finally assign the prototype of src to target
  try{target.__proto__ = src.__proto__;}catch{}
  return target;
}

const obj = {};

console.log(assignAll(obj,new Response("cheese"))); // > [object Response]

(async()=>console.log(await obj.text()))(); // > "cheese"

</script>
```

## 8. Frozen Objects - Modify Anything

The best way to modify frozen objects is to never let them freeze in the first place. You can do this by mobkey patching all the ways that things get frozen.

```js

(()=>{
  const _freeze = Object.freeze;
  Object.freeze = Object.setPrototypeOf(function freeze(obj){
    return obj;
  },_freeze);
})();

(()=>{
  const _seal = Object.seal;
  Object.seal = Object.setPrototypeOf(function seal(obj){
    return obj;
  },_seal);
})();

(()=>{
  const _preventExtensions = Object.preventExtensions;
  Object.preventExtensions = Object.setPrototypeOf(function preventExtensions(obj){
    return obj;
  },_preventExtensions);
})();

(()=>{
  const _preventExtensions = Reflect.preventExtensions;
    Reflect.preventExtensions = Object.setPrototypeOf(function preventExtensions(obj){
    return true;
  },_preventExtensions);
})();

(()=>{
  const _defineProperty = Object.defineProperty;
  Object.defineProperty = Object.setPrototypeOf(function defineProperty(obj,prop,desc){
    return _defineProperty(obj,prop,{
      ...desc,
      configurable:true
    })
  },_defineProperty);
})();

(()=>{
  const _defineProperties = Object.defineProperties;
  Object.defineProperties = Object.setPrototypeOf(function defineProperties(obj,desc){
    for(const key in desc){
      desc[key].configurable = true;
    }
    for(const key of Reflect.ownKeys(desc)){
      desc[key].configurable = true;
    }
    return _defineProperties(obj,desc)
  },_defineProperties);
})();

(()=>{
  const _defineProperty = Reflect.defineProperty;
  Reflect.defineProperty = Object.setPrototypeOf(function defineProperty(obj,prop,desc){
    return _defineProperty(obj,prop,{
      ...desc,
      configurable:true
    })
  },_defineProperty);
})();

```

## 9. Sync Blob Parse

```js
  // synchronously turn a blob into text
  function blobText(blob){
    // create blob url
    const url = URL.createObjectURL(helloWorlBlob);
    // create an ajax request targeted ar rge blob url
    // set async to false
    const xhr = new XMLHttpRequest();
    xhr.open('GET',url,false);
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

TODO:

promise short curcuit 1

promise short curcuit 2 (util.inspect)

sync blob

idempotent http

google colab js

sync import


