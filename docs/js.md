# Table of Contents

- [Inline Example](#inline-example)
- [Installation](#installation)
- [Examples](#examples)
- [Documentation](#documentation)
	- [init](#init)
	- [h](#h)
	- [patch](#patch)
	- [deleteVNode](#deletevnode)
	- [toVNode](#tovnode)
	- [toHTML](#tohtml)
- [Notes](#notes)
	- [boolean attributes](#boolean-attributes)
  - [ref](#ref)
  - [fragments](#fragments)
- [Helpers](#helpers)
  - [svg](#svg)
- [Server side rendering](#server-side-rendering)
- [WebComponents](#webcomponents)
  - [Using WebComponents in asm-dom](#using-webcomponents-in-asm-dom)
  - [Using asm-dom in WebComponents](#using-asm-dom-in-webcomponents)
- [Structuring applications](#structuring-applications)

## Inline Example

```js
import init from 'asm-dom';

const asmDom = await init();
// or init().then(asmDom => { ... });
const { h, patch } = asmDom;

const root = document.getElementById('root');

const vnode = h('div', {
  raw: { onclick: () => console.log('clicked') }
}, [
  h('span', { style: 'font-weight: bold' }, 'This is bold'),
  ' and this is just normal text',
  h('a', { href: '/foo' }, 'I\'ll take you places!')
]);
// Patch into empty DOM element – this modifies the DOM as a side effect
patch(root, vnode);

const newVnode = h('div', {
  raw: { onclick: () => console.log('another click') }
}, [
  h('span', { style: 'font-weight: normal; font-style: italic' }, 'This is now italic type'),
  ' and this is just normal text',
  h('a', { href: '/bar' }, 'I\'ll take you places!')
]);

// Second `patch` invocation
patch(vnode, newVnode); // asm-dom efficiently updates the old view to the new state
```

## Installation

You can install asm-dom using [npm](https://www.npmjs.com/package/asm-dom):

```bash
npm install --save asm-dom
```

if you are using webpack and you have some problems with fs, you can add this to your webpack config:

```js
node: {
  fs: 'empty',
},
```

## Examples

Examples are available in the [examples folder](https://github.com/mbasso/asm-dom/tree/master/examples):

- [TODO MVC](https://github.com/mbasso/asm-dom/tree/master/examples/todomvc%20-%20js)

## Documentation

### init

By default asm-dom returns an `init` function that takes an optional configuration object. This represents the [Module](https://kripken.github.io/emscripten-site/docs/api_reference/module.html) object passed to emscripten with 4 additional props:
- `useWasm`: `true` if you want to force the usage of WebAssembly
- `useAsmJS`: `true` if you want to force the usage of asm.js
- `clearMemory`: `true` by default, set it to `false` if you want to free memory manually, for more information see [deleteVNode](#deletevnode).
- `unsafePatch`: `false` by default, set it to `true` if you haven't a single `patch` in your application. This allows you to call patch with an `oldVnode` that hasn't been used previously. 

By default asm-dom uses WebAssembly if supported, otherwise asm.js

Please note that **this function creates the module only the first time that is called, the next times returns a Promise that resolve with the same, cached object**.

```js
import init from 'asm-dom';

// init returns a Promise
const asmDom = await init();
// const asmDom = await init({ useAsmJS: true });
// init().then(asmDom => { ... });
```

### h

You can create vnodes using `h` function. `h` accepts a tag/selector as a string, an optional data object and an optional **string** or array of children. The data object contains all attributes, callbacks and 4 special props:
- `ns`: the namespace URI to associate with the element
- `key`: this property is used to keep pointers to DOM nodes that existed previously to avoid recreating them if it is unnecessary. This is very useful for things like list reordering.
- `raw`: an object that contains values applied to the DOM element with the dot notation instead of `node.setAttribute`.
- `ref`: a callback that provides a way to access DOM nodes, you can learn more about that [here](#ref)

This returns the memory address of your virtual node.

```js
const { h } = asmDom;
const vnode = h('div', { style: 'color: #000' }, [
  h('h1', 'Headline'),
  h('p', 'A paragraph'),
]);

const vnode2 = h('div', {
  id: 'an-id', // node.setAttribute('id', 'an-id')
  key: 'foo', // key is a special field
  className: 'foo', // className is a special attribute evaluated as 'class'
  'data-foo': 'bar', // a dataset attribute
  onclick: (e) => console.log('clicked: ', e.target), // a callback
  ref: (node) => console.log('DOM node: ', node),
  raw: {
    foo: 'bar', // raw value applied with the dot notation: node.foo = 'bar'
  },
});
```

### patch

The `patch` takes two arguments, the first is a DOM element or a vnode representing the current view. The second is a vnode representing the new, updated view. If `patch` succedeed, the new vnode (the second parameter) is returned.

If a DOM element is passed, `newVnode` will be turned into a DOM node, and the passed element will be replaced by the created DOM node. If an `oldVnode` is passed, asm-dom will efficiently modify it to match the description in the new vnode.

**If `unsafePatch` in `init` is equal to false, any old vnode passed must be the resulting vnode from the previous call to patch. Otherwise, no operation is performed and undefined is returned.**

```js
const { h, patch } = asmDom;

const oldVnode = h('span', 'old node');
const newVnode = h('span', 'new node');

patch(document.getElementById('root'), oldVnode);
patch(oldVnode, newVnode);

// with unsafePatch = false
const vnode = h('div');
patch(oldVnode, vnode); // returns undefined, found oldVnode, expected newVnode
```

With `unsafePatch = true` you can implement some interesting mechanisms, for example you can do something like this:

```js
const oldText = h('span', 'old text');
const vnode = h('div', [
  h('span', 'this is a text'),
  oldText
]);

patch(document.getElementById('root'), vnode);

const newText = h('span', 'new text');
// patch only the child
patch(oldText, newText);
```

### deleteVNode

As we said before the `h` returns a memory address. This means that this memory have to be deleted manually, as we have to do in C++ for example. By default asm-dom automatically delete the old vnode from memory when `patch` (or `toHTML`) is called. However, if you want to create a vnode that is not patched, or if you want to manually manage this aspect (setting `clearMemory: false` in the `init` function), you have to delete it manually. For this reason we have developed a function that allows you to delete a given vnode and all its children recursively:

```js
const vnode1 = h('div');
const vnode2 = h('div', [
  h('span')
]);
patch(vnode1, vnode2); // vnode1 automatically deleted

const child1 = h('span', 'child 1');
const child2 = h('span', 'child 2');
const vnode = h('span', [
  child1,
  child2,
]);
deleteVNode(vnode); // manually delete vnode, child1 and child2 from memory
```

## toVNode

Converts a DOM node into a virtual node. This is especially good for patching over an pre-existing, server-side generated content. Using this function together with `toHTML` you can implement server-side rendering.

```js
// supposing that 'root' is a server-side generated div
const vnode = toVNode(document.getElementById('root'));

const newVnode = h('div', {
  id: 'root',
  style: 'color: #000',
}, [
  h('h1', 'Headline'),
  h('p', 'A paragraph'),
]);

patch(vnode, newVnode);
```

## toHTML

Renders a vnode to HTML string. This is particularly useful if you want to generate HTML on the server. Using this function together with `toVNode` you can implement server-side rendering.

```js
const vnode = h('div', {
  id: 'root',
  style: 'color: #000',
}, [
  h('h1', 'Headline'),
  h('p', 'A paragraph'),
]);

const html = toHTML(vnode);
// html = '<div id="root" style="color: #000"><h1>Headline</h1><p>A paragraph</p></div>';
```

## Notes

### boolean attributes

If you want to set a boolean attribute, like `readonly`, you can just pass `true` or `false`, asm-dom will handle it for you:

```js
const vnode = h('input', {
  readonly: true,
  // or readonly: false,
});
```

## Ref

If you want to access direclty DOM nodes created by asm-dom, for example to managing focus, text selection, or integrating with third-party DOM libraries, you can use `refs callbacks`.
`ref` is a special callback called after that the DOM node is mounted, if the ref callback changes or after that the DOM node is removed from the DOM tree, in this case the param is equal to `null`.
Here is an example of the first and the last case:

```js
const refCallback = (node) => {
  if (node === null) {
    // node unmounted
    // do nothing
  } else {
    // node mounted
    // focus input
    node.focus();
  }
};

const vnode1 = h('div',
  h('input', {
    ref: refCallback
  })
);

patch(
  document.getElementById('root'),
  vnode1
);

const vnode2 = h('div');
patch(vnode1, vnode2);

deleteVNode(vnode2);
```

As we said before `ref callback` is also invoked if it changes, in the following example asm-dom will call `refCallback` after that the DOM node is mounted and then `anotherRefCallback` after the update:

```c++
const vnode1 = h('div',
  h('input', {
    ref: refCallback
  })
);

patch(
  document.getElementById('root'),
  vnode1
);

const vnode1 = h('div',
  h('input', {
    ref: anotherRefCallback
  })
);

patch(vnode1, vnode2);
```

## Fragments

If you want to group a list of children without adding extra nodes to the DOM or you want to use [DocumentFragments](https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment) to improve the performance of your app, you can do that creating a `VNode` with an empty selector:

```js
// this cannot be done
/* const vnode = [
    h('div', 'Child 1'),
    h('div', 'Child 2'),
    h('div', 'Child 3')
]; */

// this is a valid alternative to the code above
const vnode = h('', [
  h('div', 'Child 1'),
  h('div', 'Child 2'),
  h('div', 'Child 3')
]);
```

## Helpers

### svg

SVG just works when using the `h` function for creating virtual nodes. SVG elements are automatically created with the appropriate namespaces.

```js
const vnode = h('div', [
  h('svg', { width: 100, height: 100 }, [
    h('circle', { cx: 50, cy: 50, r: 40, stroke: 'green', 'stroke-width': 4, fill: 'yellow'})
  ])
]);
```

## Server side rendering

If you are interested in server side rendering, you can do that with asm-dom in 2 simple steps:

- You can use `toHTML` to generate HTML on the server and send it to the client for faster page loads and to allow search engines to crawl your pages for SEO purposes.
- After that you can call `toVNode` on the node that you have server-rendered and patch it with a vnode created on the client. In this way asm-dom will preserve it and only attach event handlers, providing a fantastic first-load experience.

Here is an example:

```js
// a function that returns the view, used on client and server
const view = () => (
  h('div', {
    id: 'root',
  }, [
    h('h1', 'Title'),
    h('button', {
      className: 'btn',
      raw: {
        onclick: onButtonClick,
      },
    }, 'Click Me!'),
  ])
);

// on the server
const vnode = view();
const htmlString = toHTML(vnode);
response.send(`
  <!DOCTYPE html>
  <html>
    <head>
      <title>My Awesome App</title>
      <link rel="stylesheet" href="/index.css" />
    </head>
    
    <body>
      ${htmlString}
    </body>
    
    <script src="/bundle.js"></script>
  </html>
`);

// on the client
const oldVNode = toVNode(document.getElementById('root'));
const vnode = view();
patch(oldVNode, vnode); // attach event handlers
```

## WebComponents

Virtual DOM and WebComponents represent different technologies. Virtual DOM provides a declarative way to write the UI and keep it sync with the data, while WebComponents provide an encapsulation for reusable components. There are no limitation to use them together, you can use asm-dom with WebComponents or use asm-dom inside WebComponents.

### Using WebComponents in asm-dom

With asm-dom you can just use WebComponents as any other element:

```js
// customElements.define('my-tabs', MyTabs);

const vnode = h('my-tabs', {
  className: 'css-class',
  attr: 'an attribute',
  'tab-select': onTabSelect,
  raw: {
    prop: 'a prop',
  },
}, [
  h('p', 'I\'m a child!'),
]);
```

### Using asm-dom in WebComponents

If you want to use asm-dom to build a WebComponent, please make sure to enable the usage of [`patch`](#patch) in multiple points of your app with `unsafePatch = true` in the [`init`](#init) function. After that you can do something like this:

```js
class HelloComponent extends HTMLElement {
  static get observedAttributes() {
    return ['name'];
  }

  constructor() {
    super();
    // init the view
    this.update();
  }

  attributeChangedCallback() {
    // update the view
    this.update();
  }

  disconnectedCallback() {
    // clear memory
    window.asmDom.deleteVNode(this.currentView);
  }

  update() {
    const { patch } = window.asmDom;
    if (!this.currentView) {
      const root = document.createElement('div');
      this.attachShadow({ mode: 'open' }).appendChild(root);
      this.currentView = root;
    }
    this.currentView = patch(this.currentView, this.render());
  }

  render() {
    const { h } = window.asmDom;
    const name = this.props.name;

    return h('div', `Hello ${name}!`);
  }
}

customElements.define('hello-component', WebComponent);
```

## Structuring applications

asm-dom is a low-level virtual DOM library. It is unopinionated with regards to how you should structure your application.

You can take a look to [this](https://github.com/snabbdom/snabbdom#structuring-applications) list in snabbdom repository, a js virtual DOM that inspire this library. Snabbdom has some different APIs but you can still take inspiration from it.
