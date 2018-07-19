---
title: Writing Templates
---

# Writing Templates

lit-html templates are written using JavaScript template literals tagged with the `html` tag. The contents of the literal are mostly plain, declarative, HTML:

```js
html`<h1>Hello World</h1>`
```

**Bindings** or expressions are denoted with the standard JavaScript syntax for template literals:

```js
html`<h1>Hello ${name}</h1>`
```

## Thinking Functionally

lit-html is ideal for use in a functional approach to describing UIs. If you think of UI as a function of data, commonly expressed as `UI = f(data)`, you can write lit-html templates that mirror this exactly:

```js
let ui = (data) => html`...${data}...`;
```

This kind of function can be called any time data changes, and is extremely cheap to call. The only thing that lit-html does in the `html` tag is forward the arguments to the templates.

When the result is rendered, lit only updates the expressions whose values have changed since the previous render.

This leads to an easy to write and reason about model: always try to describe your UI as a simple function of the data it depends on, an avoid caching intermediate state, or doing manual DOM manipulation. lit-html will almost always be fast enough with the simplest description of your UI.

## Template Structure

lit-html templates are parsed by the built-in HTML parser before any values are interpolated. This means that the required structure of templates is determined by the HTML specification, and expressions can only occur in certain places.

The HTML parser in browsers never throws exceptions for malformed HTML. Instead, the specification defines how all malformed HTML cases are handled and what the resulting DOM should be. Most cases of malformed templates are not detectable by lit-html - the only side-effect will be that templates do not behave as you expected - so take extra care to properly structure templates.


Follow these rules for well-formed templates:

 * Templates must be well-formed when all expressions are replaced by empty values.
 * Expressions can only occur in attribute-value and text-content positions
 * Expressions cannot appear where tag or attribute names would appear
 * Templates can have multiple top-level elements and text
 * Templates should not contain unclosed elements - they will be closed by the HTML parser

## Binding Types

Expressions can occur in text content or in attribute value positions.

There are a few types of bindings:
  * Text:
    ```js
    html`<h1>Hello ${name}</h1>`
    ```
  * Attribute:
    ```js
    html`<div id=${id}></div`
    ```
  * Boolean Attribute:
    ```js
    html`<input type="checkbox" ?checked=${checked}>`
    ```
  * Property:
    ```js
    html`<input .value=${value}>`
    ```
  * Event Handler:
    ```js
    html`<button @click=${(e) => console.log('clicked')}>Click Me</button>`
    ```

## Supported Types

Each binding type supports different types of values:

 * Text content bindings: Many supported types - see below.
 * Attribute bindings: All values are converted to strings.
 * Boolean attribute bindings: All values evaluated for truthiness.
 * Property bindings: Any type of value.
 * Event handler bindings: Event handler functions or objects only

Text content bindings accept a large range of value types:

### Primitive Values: String, Number, Boolean, null, undefined

Primitives values are converted to strings when interpolated. They are checked for equality to the previous value so that the DOM is not updated if the value hasn't changed.

### TemplateResult

Templates can be nested by passing a `TemplateResult` as a value of an expression:

```js
const header = html`<h1>Header</h1>`;

const page = html`
  ${header}
  <p>This is some text</p>
`;
```

### Node

Any DOM Node can be passed to a text position expression. The node is attached to the DOM tree at that point, and so removed from any current parent:

```js
const div = document.createElement('div');
const page = html`
  ${div}
  <p>This is some text</p>
`;
```

### Arrays / Iterables

Arrays and Iterables of supported types are supported as well. They can be mixed values of different supported types.

```javascript
const items = [1, 2, 3];
const render = () => html`items = ${items.map((i) => `item: ${i}`)}`;
```

```javascript
const items = {
  a: 1,
  b: 23,
  c: 456,
};
const render = () => html`items = ${Object.entries(items)}`;
```

### Promises

Promises are rendered when they resolve, leaving the previous value in place until they do. Races are handled, so that if an unresolved Promise is overwritten, it won't update the template when it finally resolves.

```javascript
const render = () => html`
  The response is ${fetch('sample.txt').then((r) => r.text())}.
`;
```

## Control Flow with JavaScript

lit-html has no built-in control-flow constructs. Instead you use normal JavaScript expressions and statements:

### Ifs with ternary operators

Ternary expressions are a great way to add inline-conditionals:

```js
html`
  ${user.isloggedIn
      ? html`Welcome ${user.name}`
      : html`Please log in`
  }
`;
```

### Ifs with if-statements

You can express conditional logic with if statements outside of a template to compute values to use inside of the template:

```js
let userMessage;
if (user.isloggedIn) {
  userMessage = html`Welcome ${user.name}`;
} else {
  userMessage = html`Please log in`;
}


html`
  ${userMessage}
`
```

### Loops with Array.map

To render lists, `Array.map` can be used to transform a list of data into a list of templates:

```js
html`
  <ul>
    ${items.map((i) => html`<li>${i}</li>`)}
  </ul>
`;
```

### Looping statements

```js
const itemTemplates = [];
for (const i of items) {
  itemTemplates.push(html`<li>${i}</li>`);
}

html`
  <ul>
    ${itemTemplates}
  </ul>
`;
```

## Directives

Directives are functions that can extend lit-html by directly interacting with the Part API.

Directives will usually be created from factory functions that accept some arguments for values and configuration. Directives are created by passing a function to lit-html's `directive()` function:

```javascript
html`<div>${directive((part) => { part.setValue('Hello')})}</div>`
```

The `part` argument is a `Part` object with an API for directly managing the dynamic DOM associated with expressions. See the `Part` API in api.md.

Here's an example of a directive that takes a function, and evaluates it in a try/catch to implement exception safe expressions:

```javascript
const safe = (f) => directive((part) => {
  try {
    return f();
  } catch (e) {
    console.error(e);
  }
});
```

Now `safe()` can be used to wrap a function:

```javascript
let data;
const render = () => html`foo = ${safe(_=>data.foo)}`;
```

This example increments a counter on every render:

```javascript
const render = () => html`
  <div>
    ${directive((part) => part.setValue((part.previousValue + 1) || 0))}
  </div>`;
```

lit-html includes a few directives:

#### `repeat(items, keyfn, template)`

A loop that supports efficient re-ordering by using item keys.

Example:

```javascript
const render = () => html`
  <ul>
    ${repeat(items, (i) => i.id, (i, index) => html`
      <li>${index}: ${i.name}</li>`)}
  </ul>
`;
```

#### `until(promise, defaultContent)`

Renders `defaultContent` until `promise` resolves, then it renders the resolved value of `promise`.

Example:

```javascript
const render = () => html`
  <p>
    ${until(
        fetch('content.txt').then((r) => r.text()),
        html`<span>Loading...</span>`)}
  </p>
`;
```

#### `asyncAppend(asyncIterable)` and `asyncReplace(asyncIterable)`

JavaScript asynchronous iterators provide a generic interface for asynchronous sequential access to data. Much like an iterator, a consumer requests the next data item with a a call to `next()`, but with asynchronous iterators `next()` returns a `Promise`, allowing the iterator to provide the item when it's ready.

lit-html offers two directives to consume asynchronous iterators:

 * `asyncAppend` renders the values of an [async iterable](https://github.com/tc39/proposal-async-iteration),
appending each new value after the previous.
 * `asyncReplace` renders the values of an [async iterable](https://github.com/tc39/proposal-async-iteration),
replacing the previous value with the new value.

Example:

```javascript

const wait = (t) => new Promise((resolve) => setTimeout(resolve, t));

/**
 * Returns an async iterable that yields increasing integers.
 */
async function* countUp() {
  let i = 0;
  while (true) {
    yield i++;
    await wait(1000);
  }
}

render(html`
  Count: <span>${asyncReplace(countUp())}</span>.
`, document.body);
```

In the near future, `ReadableStream`s will be async iterables, enabling streaming `fetch()` directly into a template:

```javascript
// Endpoint that returns a billion digits of PI, streamed.
const url =
    'https://cors-anywhere.herokuapp.com/http://stuff.mit.edu/afs/sipb/contrib/pi/pi-billion.txt';

const streamingResponse = (async () => {
  const response = await fetch(url);
  return response.body.getReader();
})();
render(html`π is: ${asyncAppend(streamingResponse)}`, document.body);
```
