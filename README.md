_This is currently under draft and not complete. Feedback welcome on the issue tracker or via [twitter](https://twitter.com/pemrouz)_

# Vanilla
## Ergonomic and Widely Reusable Components

![image](https://img.shields.io/badge/component-vanilla-green.svg?style=flat-square)

This is a general pattern for authoring components that gives component developers the ability to create components and not worry that it can reliably reused in a wide range of contexts. This is a concern in an increasingly fragmented application framework landscape, and also with the advent of encapsulated Web Components which further tilts the balance in favour of writing reusable components rather than rewriting them for each framework.

The primary design goals are to refine the boundaries/seams of components, and maximise the ergonomics of authoring components. This is to separate out the concerns of and enable further customisations and innovations to take place orthogonal to the implementation of components. For example, you may wish to take certain actions before a component renders such as pre-applying a related template or [inlining styles](http://blog.vjeux.com/2014/javascript/react-css-in-js-nationjs.html) or catch diffs after the component has run (more in the [rendering pipeline]()). The components are framework-agnostic, but not the lowest common denominator at the cost of being a long-term solution.

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
* [Fractal Composition]()
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

**Zero Dependency Function:** This is the only necessary file. You should export just one function. Liberal usage of dependencies is a barrier to making widely reusable components, since an application consuming many public components _will_ end up with 27 versions of lodash, underscore and ramda. If you do need other dependencies, you should either use micro-libraries like modularised lodash or [utilise](https://github.com/utilise/utilise) that you can compile in, or require that they will be available at runtime (e.g. moment).

**Idempotent**: For a given dataset, the component should always result in the same representation. This means components should be written declaratively. `this.innerHTML = 'Hi!'` is perhaps the simplest example of this, but use of the `innerHTML` is not the most efficient. A better approach is to use `once` which will efficiently generate or update DOM to match what it should be:

```js
once(this)
 ('tr', rows)
   ('td', row => row.cells)
     .text(String)
```

If you call this sequence twice, it will amount to a no-op the second time. If there are more `rows` it will only update those, etc. You can think of `once` as React in the form of a micro-library, with the difference that it operates on an individual element level, rather than on a view level. As a result, there is no need for [calculating/maintaining an in-memory representation of everything](https://github.com/reddit/reddit-mobile/issues/247#issuecomment-118202269). The D3-inspired API (expressing views as a function of data) is slightly higher level than React, JSX and Virtual DOM which maps 1:1 to HTML. You can read more about [the evolution of different approaches to rendering here](https://github.com/rijs/docs/blob/master/components.md#guide-to-writing-components).

**Serializable:** You should not hold any selections or state within the closure of the component (other than variables that will be used within that the call). These components are stamps. They will be applied to all instances of the same type. They may be invoked on the server and the client. They may be streamed over WebSockets. They may be cached in localStorage.

**Stateless:** You can use `this.state` for persisting local state between renders. The first argument will be an object with all the data the component requires to render. This is normally data that is not unique to this instance of the component and may be injected in from different sources depending on the application it is used in.

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
this.on('event' d => { .. })
this.emit('event', data)
```

The final aspect is that child components will need to communicate with parent components. This is achieved by emitting events on the host node, as this is the only visible part to the parent (the component implementation may indeed be entirely hidden away in a closed Shadow DOM). You can use the native `addEventListener` and `dispatchEvent` for this. However, if you are using `once`, you can interchangeably use the [more ergonomic emitterify semantics](https://github.com/utilise/utilise#--emitterify), `.on` and `.emit`.

Parents should not communicate with children in the same way. If something changes in a parent component that requires the child to update, it should redraw it with the new state (unidirectional architecture).

<br><br>
## Usage

### Vanilla

The simplest way to invoke a component is:

```js
// Invoke
fn.call(node, data)
```

This is the pure, low-level API which you probably will not use regularly, but [application frameworks can use](https://github.com/rijs/components/blob/master/src/index.js#L94) to build their own conventions on top of.

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

Angular 2 supports the usage of native Web Components and so you are encouraged to use them in that manner. A minimal build of Ripple (core + components) is only ~500 bytes. Alternatively, if the Angular 2 team were to allow the `template` hook to take a function rather than a string, you could write a similar wrapper to the React approach above.

<br>
### Ripple

In Ripple, we simply register the component definition against the Custom Element name and then all instances of the element will be upgraded with this definition - leveraging the native Web Component callbacks where possible:

```js
ripple('simple-component', simpleComponent)
```

You can then use the component via markup:

```html
<simple-component>
```

Or you can append a `simple-component` to the page with the specified data:

```js
once(document.body)
  ('simple-component', { items: ['1', '2', '3']})
```

If there is a matching CSS resource, it will also load and apply that (in either the Shadow Root or the `head`). You can also explicitly specify the CSS resources a view requires via the `[css]` attribute (these will be each converted to a style tag): 

```js
<simple-component css="simple-component.css simple-component-extension.css">
```

In addition to manually invoking a component with data, you can also declare the required data resources to inject into the element. This means that if that resource is updated, elements and their children that depend on it will also be redrawn (i.e. `document.querySelectorAll('[data=items]').map(ripple.draw)`). This is another simple technique that avoids the need to have a in-memory tree of the entire application.

```js
<simple-component data="items">
```

Ripple abstracts a tonne of convenient defaults such as async rendering via `requestAnimationFrame` to keep the UI responsive, batching duplicate renders, server side rendering, async loading and deduping of CSS modules, rendering in the Shadow DOM rather than the light DOM, automatic backpressure on clients to only load the fine-grained resources used, caching resources for instant startup times (before any network requests made), client-server-clients synchronisation, offline caching and background sync via Service Workers, hot reloading by redrawing elements if a new component definition is registered...

## Fractal Composition

This spec aims to make it convenient to package and reuse individual components (e.g. `<google-maps>`), but it also enables developing applications with a more robust state management paradigm: Each component is really a sub-application with it's own model (state), view (rendering sequence), controllers (event handlers) and styles. This is the reason they can be used in very different contexts. An application is also then a simple component which aggregates other components:

> A unidirectional architecture is said to be fractal if subcomponents are structured in the same way as the whole is. In fractal architectures, the whole can be naively packaged as a component to be used in some larger application. In non-fractal architectures, the non-repeatable parts are said to be orchestrators over the parts that have hierarchical composition.
<br> — [Unidirectional User Interface Architectures](http://staltz.com/unidirectional-user-interface-architectures.html)

## Testing

Since each component is expected to convert any dataset into a particular representation, it becomes trivial to test Vanilla components in a more declarative manner. 

..

## Example Repo

..

## Rendering Pipeline

.. 
