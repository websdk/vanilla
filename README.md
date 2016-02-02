_This is currently under draft and not complete. Feedback welcome on the issue tracker or via [twitter](https://twitter.com/pemrouz)_

# Vanilla
## Ergonomic and Widely Reusable Components

![image](https://img.shields.io/badge/component-vanilla-green.svg?style=flat-square)

This is a general pattern for authoring components that gives component developers the ability to create components and not worry that it can reliably reused in a wide range of contexts. This is a concern in an increasingly fragmented application framework landscape, and also with the advent of encapsulated Web Components which further tilts the balance in favour of writing reusable components rather than rewriting them for each framework.

The primary design goals are to refine the boundaries/seams of components, and maximise the ergonomics of authoring components. This is to separate out the concerns of and enable further customisations and innovations to take place orthogonal to the implementation of components. For example, you may wish to take certain actions before a component renders such as pre-applying a related template or [inlining styles](http://blog.vjeux.com/2014/javascript/react-css-in-js-nationjs.html) or invoking lifecyle hooks or catch diffs after the component has run. The components are framework-agnostic, but not the lowest common denominator at the cost of being a long-term solution.

* [Spec]()
<br>[i. Javascript]()
<br>[ii. Styling]()
<br>[iii. Communication]()

* [Usage]()
<br>[i. Vanilla]()
<br>[ii. D3]()
<br>[iii. React]()
<br>[iv. Angular et al]()
<br>[v. Ripple]()
* [Composition and Fractals]()
* [Example Repo]()
* [Rendering Pipeline]()
* [Testing]()

<br><br>
## Spec

### 1. JavaScript

```js
/* Define - component.js */
export function component(data){ .. }
```

**Stateless:** Components can always expect that `this` is the DOM node being operated on and the first parameter will be an object with all the state and data <a name="fn1" href="#fn1-more">[1]</a> the component requires to render. This makes components agnostic as to how or where the first argument is injected, which simplifies testing and allows frameworks to co-ordinate linking state with elements in different ways <a name="fn2" href="#fn2-more">[2]</a>. 

**Idempotent**: For a given dataset, the component should always result in the same representation. This means components should be written declaratively. `this.innerHTML = 'Hi!'` is perhaps the simplest example of this, but use of the `innerHTML` is not the most efficient <a name="fn3" href="#fn3-more">[3]</a>. A component should not update anything above and beyond it's own scope (`this`).

**Serializable:** You should not hold any selections or state within the closure of the component (other than variables that will be used within that the cycle). These components are stamps. They will be applied to all instances of the same type. They may be invoked on the server and the client. They may be streamed over WebSockets. They may be cached in localStorage.

**Declarative:** Event handlers and component API should update the state object and then call `this.draw` which will redraw the component. This is in contrast to modifying the DOM directly and greatly simplifies components by disentagling rendering logic from update logic. The `this.draw` hook can then be extended by frameworks to form their own [rendering pipeline](https://github.com/pemrouz/vanilla#rendering-pipeline) or simply stubbed in tests.


<br>
### 2. Styling

```css
/* Style - component.css */
:host { }
```

Component styles should be co-located with the component. Additional styles or skins can be provided as a separate file (`component-modifier.css` or `some-feature.css`). The styles should be written in the [Web Component syntax](http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-201/) and should work if they were used accordingly (i.e. within a Shadow DOM). There are various modules that interpret these in different ways as a separate concern: apply conventions, dedupe, compile, scope for non-shadow dom browsers, inline, etc. 

<br>
### 3. Communication

```js
// Communicate
this.addEventListener('event', d => { .. })
this.dispatchEvent(event)
```

The final aspect is that child components will need to communicate with parent components. This is achieved by emitting events on the host node, as this is the only visible part to the parent (the component implementation may indeed be entirely hidden away in a closed Shadow DOM). You can use the native `addEventListener` and `dispatchEvent` for this <a name="fn4" href="#fn4-more">[4]</a>. If something changes in a parent component that requires the child to update, it should redraw it with the new state (unidirectional architecture).

<br><br>
## Usage

### Vanilla

The simplest way to invoke a component is:

```js
// Invoke
fn.call(node, data)
```

This is the pure, low-level API which you probably will not use regularly, but [application frameworks can use](https://github.com/rijs/components/blob/master/src/index.js#L94) to build their own conventions on top of. This API is also super-useful for [single-pass shallow unit testing](https://github.com/pemrouz/vanilla#testing).

<br>
### D3

Works out of the box with the D3 `.each` signature:

```js
d3.select(node)
  .datum(data)
  .each(component)
```

<br>
### React

You can use [React Faux DOM](https://github.com/Olical/react-faux-dom) to invoke these components within React (similar plumbing for the events will also be required):

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

<br>
### Angular 2 et al

Angular 2, and now many other architectures, supports the usage of native Web Components and so you are encouraged to use them in that manner. A [minimal build of Ripple](https://github.com/rijs/minimal) (core + components) which you can use alongside your existing application architecture is as low as [~3 kB](https://github.com/rijs/minimal/blob/master/dist/ripple.pure.min.js.gz). Alternatively, if the Angular 2 team were to allow the `template` hook to take a function rather than a string or file, you could write a similar wrapper to the React approach above.

<br>
### Ripple

In Ripple, the component definition is simply registered against the Custom Element name:

```js
ripple('simple-component', simpleComponent)
```

Then all instances of the element will be upgraded with this definition - leveraging the native Web Component callbacks where possible.

You can then use the component via markup:

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

```js
<simple-component css="simple-component.css simple-component-extension.css">
```

In addition to manually invoking a component with data, you can also declare the required data resources to inject into the element. This means that if that resource is updated, elements and their children that depend on it will also be redrawn (i.e. `document.querySelectorAll('[data=items]').map(d => d.draw())`). This is another simple technique that avoids the need to have a in-memory tree of the entire application.

```js
<simple-component data="items">
```

Ripple modules abstract a tonne of other convenient defaults too such as async rendering via `requestAnimationFrame` to keep the UI responsive, batching duplicate renders, server side rendering, async loading and deduping of CSS modules, rendering in the Shadow DOM rather than the light DOM, automatic backpressure on clients to only load the fine-grained resources used, caching resources for instant startup times (app renders last-known-good-state before any network requests made), client-server-clients synchronisation, offline caching and background sync via Service Workers, hot reloading by redrawing elements if a new component definition is registered...

## Fractal Composition

This spec aims to make it convenient to package and reuse individual components (e.g. `<google-maps>`), but it also enables developing applications with a more robust state management paradigm: Each component is really a sub-application with it's own model (state), view (rendering sequence), controllers (event handlers) and styles. This is the reason they can be used in very different contexts. An application is also then a simple component which aggregates other components:

> A unidirectional architecture is said to be fractal if subcomponents are structured in the same way as the whole is. In fractal architectures, the whole can be naively packaged as a component to be used in some larger application. In non-fractal architectures, the non-repeatable parts are said to be orchestrators over the parts that have hierarchical composition.
<br> — [Unidirectional User Interface Architectures](http://staltz.com/unidirectional-user-interface-architectures.html)

## Testing

Since each component is expected to convert any dataset into a particular representation, it becomes trivial to test Vanilla components in a more declarative and entirely framework/boilerplate-less manner. [See here for an example](https://github.com/vanillacomponents/ux-button/blob/master/test.js#L22-L34) of shallow unit tests and functional tests that invokes a component on a DOM node and then checks the output:

```js
test('unit test - simple', t => {
  t.plan(1)

  const host = el('ux-button')
  button.call(host, { label: 'foo' })

  t.equal(host.outerHTML, strip`
    <ux-button tabindex="-1" class="">
      <span>foo</span>
      <div class="spinner"></div>
    </ux-button>
  `)
})
```

## Rendering Middleware

During testing, you can set `el.draw = () => fn.call(el, el.state)` to test API's and event handlers which will attempt to redraw a component after changing some state. 

In your application, you can extend this in a similar style to Koa middleware:

> When a middleware invokes yield next the function suspends and passes control to the next middleware defined. After there are no more middleware to execute downstream, the stack will unwind and each middleware is resumed to perform its upstream behaviour.
<br> — [Koa](http://koajs.com/)

The general pattern is that you can replace the draw function with a function that takes the previous draw function and returns a new function that will be invoked with the element. Then you can perform some action in your middleware before (and after) calling the `next` (the previous draw) function. You can do this repeatedly to add different middleware. When a component's draw function is then invoked, it will then run through each middleware and step down to the next one via all the `next(el)` calls and once the component is rendered, the stack will unwind and each middleware will take perform their post-render operations. Each middleware should also return the result of the next, or nothing if it decides to stop the render. By composing the return values, it's possible to tell if the element rendered (element returned) or at least one middleware bailed (nothing returned):

```js
el.draw = middleware(el.draw)

const middleware = next => el => {
  // do something with el before
  return next(el)
  // or also do something with el after
}
```

In your application, you will likely want to point the draw function a single and more dynamic `draw` function (e.g. `ripple.draw`) rather than individually extending each component's draw function. Middleware should progressively enhance and optimise and components should always work without them. Few examples of rendering middleware:

* [PreCSS](https://github.com/rijs/precss/blob/master/src/index.js#L17-L45) - Transforms and applies component CSS 
* [Shadow](https://github.com/rijs/shadow#ripple--shadow-dom) - Creates and renders into the Shadow DOM rather than the Light DOM 
* [Features](https://github.com/rijs/features) - Extends a component with other components/features (mixins)
* [Memoize](https://github.com/rijs/memoize) - Checks if (immutable) data hasn't changed and skips the render if so (essentially `shouldComponentUpdate`)
* [Backpressure](https://github.com/rijs/backpressure/blob/master/src/index.js) - Fetches component implementations from the server as they are attempted to be drawn.

You can also use this provide hooks for extra life cycle functions. For example, for React:

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

## Example Repo

..

## Footnotes

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