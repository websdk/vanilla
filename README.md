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
<br>[i. Vanilla Component](#1-vanilla-component)
<br>[ii. Composing an Application](#2-composing-an-application)
<br>[iii. Testing](#3--testing)
<br>[iv. Performance](#4-performance)
* [Example Repo](#example-repo)
* [Footnotes](#footnotes)

<br><br>
## Authoring

### 1. JavaScript

```js
/* Define - component.js */
export default function component(data){ .. }
```

**Stateless:** Components are just plain old functions. It can always expect that `this` is the DOM node being operated on and the first parameter will be an object with all the state and data <a name="fn1" href="#fn1-more">[1]</a> the component requires to render. This makes components agnostic as to how or where the first argument is injected, which simplifies testing and allows frameworks to co-ordinate linking state with elements in different ways <a name="fn2" href="#fn2-more">[2]</a>. 

**Idempotent**: For a given dataset, the component should always result in the same representation. This means components should be written declaratively. `this.innerHTML = 'Hi!'` is perhaps the simplest example of this, but use of the `innerHTML` is not the most efficient <a name="fn3" href="#fn3-more">[3]</a>. A component should not update anything above and beyond it's own scope (`this`).

**Serializable:** You should not hold any selections or state within the closure of the component (other than variables that will be used within that the cycle). These components are stamps. They will be applied to all instances of the same type. They may be invoked on the server and the client. They may be streamed over WebSockets. They may be cached in localStorage.

**Declarative:** Event handlers and component API should update the state object and then call `this.draw` which will redraw the component. This is in contrast to modifying the DOM directly and greatly simplifies components by disentagling rendering logic from update logic. The `this.draw` hook can then be extended by frameworks to form their own [rendering pipeline](#rendering-middleware) or simply stubbed in tests.

**Defaults:** Default values for state can simply be set using the native ES6 syntax:

```js
function component({ color = 'red', focused = false }){ ... }
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
this.addEventListener('event', d => { .. })
this.dispatchEvent(event)
```

The final aspect is that child components will need to communicate with parent components. This is achieved by emitting events on the host node, as this is the only visible part to the parent (the component implementation may indeed be entirely hidden away in a closed Shadow DOM). You can use the native `addEventListener` and `dispatchEvent` for this <a name="fn4" href="#fn4-more">[4]</a>. If something changes in a parent component that requires the child to update, it should redraw it with the new state (unidirectional architecture).

<br><br>
# Using

### 1. Vanilla Component

The simplest way to invoke a component is:

```js
// Invoke
fn.call(node, data)
```

This is the pure, low-level, 100%-dependency free API which you probably will not use regularly, but [application frameworks can use](https://github.com/rijs/components/blob/master/src/index.js#L94) to build their own conventions on top of. This API is super-useful for [single-pass shallow unit testing](https://github.com/pemrouz/vanilla#testing), and makes it possible to use these components in existing libraries/frameworks, such as D3 <a name="fn5" href="#fn5-more">[5]</a>, React <a name="fn6" href="#fn6-more">[6]</a>, Angular 2 <a name="fn7" href="#fn7-more">[7]</a>, etc. 

<br>
### 2. Composing an Application

The previous section describes rendering a single component. With components as simple functions that transform some data to a HTML representation, we can write an application as another component that composes other components (which in turn may compose other components). For example, we may have the following application/component:

```html
<app-vanilla>
```

Which may expand to:

```html
<app-vanilla>
  <nav-top>
  <nav-left>
  <grid-main>
```

And each custom element may in turn further expand itself. This leads to a simple unidirectional and fractal architecture <a name="fn8" href="#fn8-more">[8]</a>. 

To facilitate this recursive expansion across components, [Ripple Minimal](https://github.com/rijs/minimal/) can be used as a small utility library <a name="fn9" href="#fn9-more">[9]</a>.

<br>
### 3.  Testing

Due to the simplified nature of these components (plain functions of data) it becomes (a) drastically simpler to test (b) free of any library/framework/boilerplate and (c) increases the scope of what we can test. Below is several levels of component testing maturity:

1. **Shallow Unit Tests**: [Invoke the component function on a node with some data](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L18) and then [check the output](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L20-L25). It may be a bit brittle to assert the entire output every time, so you can [just pinpoint certain features](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L72).

1. **Shallow Functional Tests**: [Each API](https://github.com/vanillacomponents/ux-button/blob/master/src/ux-button.js#L15-L18) or [event handler will update some state and then redraw itself](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/ux-input.js#L23-L38). By setting the [draw function](https://github.com/utilise/tdraw/blob/ef62cc080aedf14971f052a6327d995aa7deaa94/index.js#L4), and after [invoking an API](https://github.com/vanillacomponents/ux-button/blob/baccaffbb422db4425cd0ea0eb2a201ff708498b/src/test.js#L47) or [dispatching an event](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L106-L110), you can check the new state/output is as expected.

1. **Universal Tests**: The tests can be run on Node or real browsers. This is simply achieved by just [importing `browserenv`](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L2) which [makes `window` and `document` available on the server](https://github.com/pemrouz/browserenv/blob/master/index.js#L4-L5). 

1. **Cross-Browser Tests**: Whilst running in Node is useful for things like coverage, it is not an alternative to running the tests in the browsers you actually support. To automate this, and get realtime feedback from different browsers/platforms [use Popper](https://github.com/pemrouz/popper/#popper-realtime-cross-browser-automation) and add a [.popper.yml](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/.popper.yml). 

1. **Visual Tests**: In addition to verifying output [by checking the rendered HTML](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L96), you should also pin down [the calculated styles using `getComputedStyle`](https://github.com/vanillacomponents/ux-input/blob/b056b09c02b5b3d42852df63433332b71e529f23/src/test.js#L96-L97)

1. **CI Tests**: You should [set up your CI to run your tests on the browsers you support](https://travis-ci.org/vanillacomponents/ux-input/builds/109224330#L1357). Popper will soon [generate badges from your latest CI logs](https://github.com/pemrouz/popper/issues/10). 

1. **System Testing**: Since the entire application at any point in time can be replicated from a given dataset, and all actions simply transition from one state to another, you can conduct high-level tests on your entire application by rendering all components and in the same manner as you test individual components (test rendering with specific data, then test transitions between states).

<br>
### 4. Performance

There is no single way to benchmark performance across different libraries/frameworks for all use cases, but a popular test is the DBMonster Challenge (see [here for comparisons](http://mathieuancelin.github.io/js-repaint-perfs)). This should be taken with the usual caveats, but is at least useful as a proof-of-concept to confirm how fast this approach can be:

* [**DBMonster Once**](http://mathieuancelin.github.io/js-repaint-perfs/once)
* [**DBMonster Ripple**](http://mathieuancelin.github.io/js-repaint-perfs/ripple)

<br>
# Example Repo

An example component that covers all the above points is [available here](https://github.com/vanillacomponents/ux-input#ux-input).

<br>
# Footnotes

<a name="fn1-more" href="#fn1">**[1]**</a> "State" normally refers to things that are unique to this instance of the component, such as it's scroll position or whether it is focused or not. "Data" is normally used to refer to things that are not unique to this instance of the component. They may be shared with other components and may be injected in from different sources depending on the application it is used in.

<a name="fn2-more" href="#fn2">**[2]**</a> There are a few common strategies around persisting state between renders:

* The `state` object is held privately in a parent closure and is modified externally via specific accessors. This is the approach outlined in [_Towards Reusable Charts_](http://bost.ocks.org/mike/chart/). In that paradigm, the components in this spec are identical to the inner functions.

* The `state` object is managed with the help of some secondary structure that roughly matches the structure of the DOM. This is the virtual DOM approach used by React. In that paradigm, the components in this spec are equivalent to just the `render` function.

* The `state` object for an element is co-located with the element itself (i.e. `this.state == state`). This is the approach used by Ripple. In that paradigm, by eliminating the need for any closures or secondary structures, a component is just the pure transformation function of data. This also means you can inspect the state of element by checking `$0.state`.

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
once(this)
  .on('event' d => { .. })
  .emit('event', data)

once(this)
  ('li', [1, 2, 3])              // create some elements first
    .on('event.ns1' d => { .. }) // multiple, namespaced handlers
    .on('event.ns2' d => { .. })
    .emit('event', data)
```

<a name="fn5-more" href="#fn5">**[5]**</a> **D3:** Works out of the box with the D3 `.each` signature:

```js
d3.select(node)
  .datum(data)
  .each(component)
```

<a name="fn6-more" href="#fn6">**[6]**</a> **React:** You can use [React Faux DOM](https://github.com/Olical/react-faux-dom) to invoke these components within React (similar plumbing for the events will also be required):

```js
// Vanilla Component
function simpleComponent(state){
  once(this)
    ('li', state.items)
      .text(String)
      .on('click.sth', d => this.emit('wat', 'foo') )
}
    
const simpleReactComponent = reactify(simpleComponent)

ReactDOM.render(React.createElement(simpleReactComponent, { items: ['1', '2', '3'] }), mountNode)

function reactify(fn) {
  return function(state) {
    var el = ReactFauxDOM.createElement('div')
    fn.call(el, state)
    return el.toReact()
  }
}
```

An alternative approach would be to precompile the functions, similar to how JSX desugars to normal React code.

<a name="fn7-more" href="#fn7">**[7]**</a> **Angular 2:** Angular 2, and now many other architectures, supports the usage of native Web Components and so you are encouraged to use them in that manner. A [minimal build of Ripple](https://github.com/rijs/minimal) (core + components) which you can use alongside your existing application architecture is as low as [~3 kB](https://github.com/rijs/minimal/blob/master/dist/ripple.pure.min.js.gz). Alternatively, if the Angular 2 team were to allow the `template` hook to take a function rather than a string or file, you could write a similar wrapper to the React approach above.

<a name="fn8-more" href="#fn8">**[8]**</a> [Unidirectional User Interface Architectures](http://staltz.com/unidirectional-user-interface-architectures.html)

> A unidirectional architecture is said to be fractal if subcomponents are structured in the same way as the whole is. In fractal architectures, the whole can be naively packaged as a component to be used in some larger application.

<a name="fn9-more" href="#fn9">**[9]**</a> **Ripple (Minimal):** Ripple is made up of a simple core module to hold the global state of the application, that can also be extended by other modules:

```js
ripple(name, body)                     // setter
ripple(name)                           // getter
ripple(name).on('change', res => ...)  // change
```

By listening to changes, [other modules](https://github.com/rijs/docs/blob/master/architecture.md#index-of-modules) build up other behaviour. This is how the [components module](https://github.com/rijs/components#ripple--components) reactively updates the page, when a component, it's data or it's styles changes. Components in turn can update resources in the Ripple core, which will in turn update anything that depends on it:

![image](https://cloud.githubusercontent.com/assets/2184177/13036934/3d1b7b14-d36c-11e5-8ce7-1c5bb2403ea6.png)

Whenever you register a component definition against a Custom Element name, all instances of the element will be upgraded (drawn) with this definition.

```js
ripple('simple-component', simpleComponent)
```

You can instantiate components via markup:

```html
<simple-component>
```

Or you can append a `simple-component` to the page with the specified data:

```js
once(document.body)
  ('simple-component', { items: ['1', '2', '3']})
```

The (D3) data here (second parameter) will be passed down to the component. 

If there is a matching CSS resource, it will also load and apply that (in either the Shadow Root or the `head`). You can also explicitly specify the CSS resources a view requires via the `[css]` attribute (these will be each converted to a style tag): 

```html
<simple-component css="simple-component.css simple-component-extension.css">
```

In addition to manually invoking a component with data, you can also declare the required data resources to inject into the element. This means that if that resource is updated, elements and their children that depend on it will also be redrawn (i.e. `document.querySelectorAll('[data=items]').map(d => d.draw())`). 

```html
<simple-component data="items">
```

These named data resources are useful for modelling your domain and synchronising them with the server. However, most of the time components will simply transform data they receive for their children, interpret gestures and bubble up events, dirty state and redraw themselves, which will result in themselves and all their children being rerendered:

![image](https://cloud.githubusercontent.com/assets/2184177/13038726/7e675cd8-d38e-11e5-8215-92428676c935.png)

##### Debugging

You can inspect the state/data any component was last rendered with by just checking `$0.state` in the dev tools.

##### Rendering Middleware

Ripple points all the element `.draw` hooks to a generic `ripple.draw`, which rerenders an element. However, this can be extended in a similar style to Koa middleware:

> When a middleware invokes yield next the function suspends and passes control to the next middleware defined. After there are no more middleware to execute downstream, the stack will unwind and each middleware is resumed to perform its upstream behaviour.
<br> — [Koa](http://koajs.com/)

The general pattern is that you can replace the draw function with a function that takes the previous draw function and returns a new function that will be invoked with the element. Then you can perform some action in your middleware before (and after) calling the `next` (the previous draw) function. You can do this repeatedly to add different middleware. When a component's draw function is then invoked, it will then run through each middleware and step down to the next one via all the `next(el)` calls and once the component is rendered, the stack will unwind and each middleware will take perform their post-render operations. Each middleware should also return the result of `next(el)`, or nothing if it decides to stop the render. By composing the return values, it's possible to tell if the element rendered (element returned) or at least one middleware bailed (nothing returned):

```js
ripple.draw = middleware(ripple.draw)

const middleware = next => el => {
  // do something with el before
  return next(el)
  // or also do something with el after
}
```

Middleware should progressively enhance and optimise and components should always work without them. Few examples of rendering middleware:

* [PreCSS](https://github.com/rijs/precss/blob/master/src/index.js#L17-L45) - Transforms and applies component CSS 
* [Shadow](https://github.com/rijs/shadow#ripple--shadow-dom) - Creates and renders into the Shadow DOM rather than the Light DOM 
* [Features](https://github.com/rijs/features) - Extends a component with other components/features (mixins)
* [Memoize](https://github.com/rijs/memoize) - Checks if (immutable) data hasn't changed and skips the render if so (essentially `shouldComponentUpdate`)
* [Backpressure](https://github.com/rijs/backpressure/blob/master/src/index.js) - Fetches component implementations from the server as they are attempted to be drawn.

You can also use this to provide hooks for extra life cycle functions. For example, for React:

```js
draw = next => el => {
  
  if (hasChanged(el.state)) componentWillReceiveProps()

  if (!shouldComponentUpdate(el)) return

  if (isDetached(el)) componentWillMount()
  
  componentWillUpdate()
  
  var rendered = next(el)

  if (rendered && wasDetached(el)) componentDidMount()

  if (rendered) componentDidUpdate()

  if (isDetached(el)) componentWillUnmount()

}
```
