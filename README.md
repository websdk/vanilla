_This is currently under draft and not complete. Feedback welcome on the issue tracker or via [twitter](https://twitter.com/pemrouz)_

# Vanilla
## Ergonomic and Widely Reusable Components

![image](https://img.shields.io/badge/component-vanilla-green.svg?style=flat-square)

This is a general pattern for authoring components that gives component developers the ability to create components and not worry that it can reliably reused in a wide range of contexts. This is a concern in an increasingly fragmented application framework landscape, and also with the advent of encapsulated Web Components which further tilts the balance in favour of writing reusable components rather than rewriting them for each framework. These components follow a simple unidirectional model and progressively enhanced by the Web Components specifications, but are not required for them to work. 

The primary design goals are to refine the boundaries/seams of components, and maximise the ergonomics of authoring components. This is to separate out the concerns of and enable further customisations and innovations to take place orthogonal to the implementation of components. For example, you may wish to take certain actions before a component renders such as pre-applying a related template or inlining styles or invoking lifecyle hooks or catch diffs after the component has run. The components are framework-agnostic, but not the lowest common denominator at the cost of being a long-term solution.

* [**Authoring**](#authoring)
<br>[i. Javascript](#1-javascript)
<br>[ii. Styling](#2-styling)
<br>[iii. Communication](#3-communication)

* [**Using**](#using)
<br>[i. Vanilla](#1-vanilla-component)
<br>[ii. Ripple](#2-ripple)
<br>[iii. D3](#3-d3)
<br>[iv. React](#4-react)
<br>[v. Angular](#5-angular)
* [Testing](#testing)
* [Performance](#performance)
* [Example Repo](#example-repo)
* [Footnotes](#footnotes)

<br><br>
## Authoring

### 1. JavaScript

```js
/* Define - component.js */
export default function component(node, data){ .. }
```

**Stateless:** Components are just plain old functions. `node` is the DOM node being operated on and `data` is an object with all the state and data <a name="fn1" href="#fn1-more">[1]</a> the component requires to render. This makes components agnostic as to how or where the node/data is injected, which simplifies testing and allows frameworks to co-ordinate linking state with elements in different ways <a name="fn2" href="#fn2-more">[2]</a>. 

**Idempotent**: For a given dataset, the component should always result in the same representation. This means components should be written declaratively. `node.innerHTML = 'Hi!'` is perhaps the simplest example of this, but use of the `innerHTML` is not the most efficient <a name="fn3" href="#fn3-more">[3]</a>. A component should not update anything above and beyond it's own scope (`node`).

**Serializable:** You should not hold any selections or state within the closure of the component (other than variables that will be used within that the cycle). These components are stamps. They will be applied to all instances of the same type. They may be invoked on the server and the client. They may be streamed over WebSockets. They may be cached in localStorage.

**Declarative:** Event handlers and component API should update the state object and then call `node.draw` which will redraw the component. This is in contrast to modifying the DOM directly and greatly simplifies components by disentagling rendering logic from update logic. The `node.draw` hook can then be extended by frameworks to form their own [rendering pipeline](#rendering-middleware) or simply stubbed in tests.

**Defaults:** Default values for state can simply be set using the native ES6 syntax:

```js
function component(node, { color = 'red', focused = false }){ ... }
```

If you need to default and set initial values on the state object only the first time you can use a [helper function](https://github.com/utilise/utilise#--defaults). The same technique can be used to idempotently expose your component API.

<br>
### 2. Styling

```css
/* Style - component.css */
:host { }
```

Component styles should be co-located with the component. Additional styles or skins can be provided as a separate file (`component-modifier.css` or `some-feature.css`). The styles should be written in the [Web Component syntax](http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-201/) and should work if they were used accordingly (i.e. within a Shadow DOM). There are various modules that interpret these in different ways as a separate concern: apply conventions, dedupe, compile, scope for non-shadow dom browsers, inline, etc. 

You will want a mix of styles to be inherited and not inherited. To make styling robust across different environments, the root element [should defensively set](https://github.com/vanillacomponents/ux-button/blob/4d98a9ae90a6d9632134cc2c0c86f8cbcb4aa311/src/ux-button.css#L1-L17) all the unwanted styles which may leak into the component. It should then [explicitly set those it does wish to inherit](https://github.com/vanillacomponents/ux-button/blob/4d98a9ae90a6d9632134cc2c0c86f8cbcb4aa311/src/ux-button.css#L17) by setting their value to `inherit`, such as `font-size: inherit` etc. Where possible, it's also recommended to [use `em`](https://github.com/vanillacomponents/ux-button/blob/4d98a9ae90a6d9632134cc2c0c86f8cbcb4aa311/src/ux-button.css#L15-L16) to make components responsive to their immediate context.

<br>
### 3. Communication

```js
// Communicate
node.addEventListener('event', d => { .. })
node.dispatchEvent(event)
```

The final aspect is that child components will need to communicate with parent components. This is achieved by emitting events on the host node, as this is the only visible part to the parent (the component implementation may indeed be entirely hidden away in a closed Shadow DOM). You can use the native `addEventListener` and `dispatchEvent` for this <a name="fn4" href="#fn4-more">[4]</a>. If something changes in a parent component that requires the child to update, it should redraw it with the new state (unidirectional architecture).

<br><br>
# Using

### 1. Vanilla Component

The simplest way to invoke a component is:

```js
import { component } from 'component'
component.call(node, data)
```

This is the pure, low-level, 100% dependency free API. You just `require`/`import` the component function and invoke it on an element with some data. Similarly, you can load the styles by just including the stylesheet via `<link rel="stylesheet">` or `<style>`. This API is super-useful for [single-pass shallow unit testing](https://github.com/pemrouz/vanilla#testing) or application frameworks to build upon.

<br>
### 2. [Ripple](https://github.com/rijs/minimal#minimal)

Ripple allows you to use these components as Web Components:

```js
ripple(require('component')) // load all the resources the component exports

component = document.body.appendChild(document.createElement('component'))
component.state = { ... }
component.draw()
```

By having a consistent contract across components (set `state`, then `.draw`), this makes it easy to compose independent components. For example, you could have the following application/component:

```html
<app-vanilla>
```

Which expands to:

```html
<app-vanilla>
  <nav-top>
  <nav-left>
  <grid-main>
```

And each custom element may in turn further expand itself. This leads to a simple unidirectional and fractal architecture <a name="fn5" href="#fn5-more">[5]</a>. 

Note: If you use [once](https://github.com/utilise/once#once), it will efficiently create or update elements to match your data and also then redraw them:

```js
once(document.body)
  ('component', [{ .. }, { .. }, { .. }])
```

<br>
### 3. D3

Works very closely with the D3 `.each` signature (`component.call(node, data)` vs `component(node, data)`). You can use a helper function to convert the signature:

```js
const th = fn => function(d){ fn(this, d) }

d3.select(node)
  .datum(data)
  .each(th(component))
```

<br>
### 4. React

You can use [React Faux DOM](https://github.com/Olical/react-faux-dom) to invoke these components within React:

```js
import { component } from 'component'

const reactComponent = reactify(component)

ReactDOM.render(React.createElement(reactComponent, { .. }), mountNode)

function reactify(fn) {
  return function(state) {
    var el = ReactFauxDOM.createElement('div')
    fn.call(el, state)
    return el.toReact()
  }
}
```

An alternative approach would be to precompile the functions, similar to how JSX desugars to normal React code.

<br>
### 5. Angular

You can create a generic directive to invoke the element with the data from the scope when they change and proxy events.

<br>
# Testing

Due to the simplified nature of these components (plain functions of data) it becomes (a) drastically simpler to test (b) free of any library/framework/boilerplate and (c) increases the scope of what we can test. Below is several levels of component testing maturity:

1. **Shallow Unit Tests**: [Invoke the component function on a node with some data](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L18) and then [check the output](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L20-L25). It may be a bit brittle to assert the entire output every time, so you can [just pinpoint certain features](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L72).

1. **Shallow Functional Tests**: [Each API](https://github.com/vanillacomponents/ux-button/blob/master/src/ux-button.js#L15-L18) or [event handler will update some state and then redraw itself](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/ux-input.js#L23-L38). By setting the [draw function](https://github.com/utilise/tdraw/blob/ef62cc080aedf14971f052a6327d995aa7deaa94/index.js#L4), and after [invoking an API](https://github.com/vanillacomponents/ux-button/blob/baccaffbb422db4425cd0ea0eb2a201ff708498b/src/test.js#L47) or [dispatching an event](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L106-L110), you can check the new state/output is as expected.

1. **Universal Tests**: The tests can be run on Node or real browsers. This is simply achieved by just [importing `browserenv`](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L2) which [makes `window` and `document` available on the server](https://github.com/pemrouz/browserenv/blob/master/index.js#L4-L5). 

1. **Cross-Browser Tests**: Whilst running in Node is useful for things like coverage, it is not an alternative to running the tests in the browsers you actually support. To automate this, and get realtime feedback from different browsers/platforms [use Popper](https://github.com/pemrouz/popper/#popper-realtime-cross-browser-automation) and add a [.popper.yml](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/.popper.yml). 

1. **Visual Tests**: In addition to verifying output [by checking the rendered HTML](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L96), you should also pin down [the calculated styles using `getComputedStyle`](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L96-L97)

1. **CI Tests**: You should [set up your CI to run your tests on the browsers you support](https://travis-ci.org/vanillacomponents/ux-input/builds/109224330#L1357). Popper will soon [generate badges from your latest CI logs](https://github.com/pemrouz/popper/issues/10). 

1. **System Testing**: Since the entire application at any point in time can be replicated from a given dataset, and all actions simply transition from one state to another, you can conduct high-level tests on your entire application by rendering all components and in the same manner as you test individual components (test rendering with specific data, then test transitions between states, whilst making assertions across components).

<br>
# Performance

There is no single way to benchmark performance across different libraries/frameworks for all use cases, but a popular test is the DBMonster Challenge (see [here for comparisons](http://mathieuancelin.github.io/js-repaint-perfs)). This should be taken with the usual caveats, but is at least useful as a proof-of-concept to confirm how fast this approach can be:

* [**DBMonster Once**](http://mathieuancelin.github.io/js-repaint-perfs/once)
* [**DBMonster Ripple**](http://mathieuancelin.github.io/js-repaint-perfs/ripple)

<br>
# Example Repo

There are a [few example repo's](https://github.com/search?q=component-vanilla-green.svg&type=Code&utf8=%E2%9C%93) that covers all the above points:

* [pemrouz/markdown-editor](https://github.com/pemrouz/markdown-editor#markdow-editorpreviewer)
* [pemrouz/sweet-alert](https://github.com/pemrouz/sweet-alert#sweet-alert)
* [pemrouz/d3-chosen](https://github.com/pemrouz/d3-chosen#d3-chosen)
* [vanillacomponents/ux-input](https://github.com/vanillacomponents/ux-input#ux-input)

<br>
# Footnotes

<a name="fn1-more" href="#fn1">**[1]**</a> "State" normally refers to things that are unique to this instance of the component, such as it's scroll position or whether it is focused or not. "Data" is normally used to refer to things that are not unique to this instance of the component. They may be shared with other components and may be injected in from different sources depending on the application it is used in.

<a name="fn2-more" href="#fn2">**[2]**</a> There are a few common strategies around persisting state between renders:

* The `state` object is held privately in a parent closure and is modified externally via specific accessors. This is the approach outlined in [_Towards Reusable Charts_](http://bost.ocks.org/mike/chart/). In that paradigm, the components in this spec are identical to the inner functions.

* The `state` object is managed with the help of some secondary structure that roughly matches the structure of the DOM. This is the virtual DOM approach used by React. In that paradigm, the components in this spec are equivalent to just the `render` function.

* The `state` object for an element is co-located with the element itself (i.e. `node.state == state`). This is the approach used by Ripple. In that paradigm, by eliminating the need for any closures or secondary structures, a component is just the pure transformation function of data. This also means you can inspect the state of element by checking `$0.state`.

<a name="fn3-more" href="#fn3">**[3]**</a> Instead of `.innerHTML`, you could use jQuery and control statements (e.g. `if`) to narrow down which areas to update. But this imperative approach spawns many code paths, quickly leading to spaghetti code. You could use some form of templating to improve on that like Handlebars or JSX, but this comes at the cost of a new syntax and a compilation step.

A better approach is to use [`once`](https://github.com/utilise/once#once), which combines the best of both (declarative and JS):

* Once will **efficiently generate or update DOM** to match what it should be, making as minimal modifications as necessary
* Once is a **small library** (not a framework)
* Once is **JavaScript** (no new language syntax or compilation step)
* Once is **declarative** (no spaghetti code)
* Once is **versatile** (arbitrary selectors or even functions)
* Once is **composable** (each invocation returns a new selection of parents)
* Once is one of the [**fastest approaches**](http://mathieuancelin.github.io/js-repaint-perfs/) to rendering
* The **data driven**, D3-inspired API (expressing views as a function of data) is slightly higher level than markup-based templates, React, JSX and Virtual DOM which maps 1:1 to HTML. 

You can read more about [the evolution of different approaches to rendering here](https://github.com/rijs/docs/blob/master/components.md#guide-to-writing-components).

<a name="fn4-more" href="#fn4">**[4]**</a> The native events API can be a little bit more verbose and restrictive than necessary. If you are using `once`, you can interchangeably use the [more ergonomic emitterify semantics](https://github.com/utilise/utilise#--emitterify), `.on` and `.emit` on any selection it returns. `.emit` will create and dispatch a `CustomEvent` with the specified `type` and `detail` (or default to the element's state). `.on` will allow you to register multiple listeners for the same event and to also use namespaces.

```js
once(node)
  .on('event' d => { .. })
  .emit('event', data)

once(node)
  ('li', [1, 2, 3])              // create some elements first
    .on('event.ns1' d => { .. }) // multiple, namespaced handlers
    .on('event.ns2' d => { .. })
    .emit('event', data)
```

<a name="fn5-more" href="#fn5">**[5]**</a> [Unidirectional User Interface Architectures](http://staltz.com/unidirectional-user-interface-architectures.html)

> A unidirectional architecture is said to be fractal if subcomponents are structured in the same way as the whole is. In fractal architectures, the whole can be naively packaged as a component to be used in some larger application.