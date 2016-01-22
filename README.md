# react-test-tree [![Build Status](https://travis-ci.org/QubitProducts/react-test-tree.svg)](https://travis-ci.org/QubitProducts/react-test-tree) [![js-standard-style](https://img.shields.io/badge/code%20style-standard-brightgreen.svg?style=flat)](http://standardjs.com/)


[![Sauce Test Status](https://saucelabs.com/browser-matrix/oliverwoodings.svg)](https://saucelabs.com/u/oliverwoodings)

## Install

With [npm](http://npmjs.org) do:

```npm install --save-dev react-test-tree```

## Overview

`react-test-tree` is a simple, scalable and concise way of testing React components. It is an evolution of the [react-page-objects](https://github.com/QubitProducts/react-page-objects) library.

A test tree is dev-friendly representation of your entire React component tree, built by recursing through special props applied to your components.

React gives us some great utilities for testing React components, however they lead to overly-verbose boilerplate that clutters your tests. `react-test-tree` tidies this clutter away, allowing you to manipulate your components with short, concise statements:

```jsx
var BozComponent = React.createClass({
  render: function () {
    return (
      <div>
        <button testRef="biz">Biz</button>
      </div>
    );
  }
});

var FuzComponent = React.createClass({
  render: function () {
    return <div />
  }
});

var FooComponent = React.createClass({
  render: function () {
    return (
      <div>
        <button testRef="bar">Bar</button>
        <select testRefCollection="baz">
          <option>blue</option>
          <option>gold</option>
        </select>
        <BozComponent testRef="boz" />
        <FuzComponent testRef="fuz" />
      </div>
    );
  }
});

var fooTree = testTree(<FooComponent />, {
  stub: {
    fuz: null
  }
});
fooTree.get("bar").click(); // simulates a click
fooTree.getIn(["boz", "biz"]).click(); // simulates a click on a deep node
fooTree.get("baz").length === 2; // collection of nodes
fooTree.get("fuz") === null; // null due to being stubbed out
```

In the above example `react-test-tree` has recursively built a tree out of the `testRef` and `testRefCollection` props, represented as nodes of the tree, which can be retrieved using `get()` or `getIn()`. Any names that appear in the `stub` tree config get replaced or removed.


## testRef and testRefCollection

You should be familiar with the `ref` prop in React. They are used when you need to reference an element in your render function. You can look at the `testRef` prop like a `ref`, but purely for testing. It is necessary to distinguish between the two because of their applications; the React team is making it increasingly clear that `ref`s should only be used for very specific purposes, which don't primarily include testing.

As well as the basic `testRef` prop, `react-test-tree` makes it possible to retrieve the children of an element by use of `refCollection`. Declaring `testRefCollection` on a component will make all it's direct children available on the corresonding tree node as an array:

```jsx
var BarComponent = React.createClass({
  render: function () {
    return (
      <select testRef="foo" testRefCollection="bar">
        <option value="blue">Blue</option>
        <option value="gold">Gold</option>
      </select>;
    );
  }
});

var barTree = testTree(<BarComponent />);
barTree.get("bar").length === 2;
barTree.get("bar")[0].getAttribute("value") === "blue";
```

__Notes__:
* You can still apply a `testRef` as well as a `testRefCollection` if you want to be able to manipulate the parent element too.
* `testRef`s and `testRefCollection`s may not have the same name.
* Updates to the render tree will not be reflected in the test tree until the next tick. You should defer accessing the tree after causing any updates:

```jsx
var tree = testTree(<MyComponent />);
tree.get('button').click();
defer(function () {
  expect(tree.state.clicked).to.be.true; // passes
});
```


## Stubs

It is inevitable that at some point when testing React components you will want to avoid rendering part of a component. Perhaps it might trigger some sideways data loading, or maybe you want to replace it with a mock. `react-test-tree` allows you to quickly and easily stub out any testRefs in the tree with either `null` or a replacement component:

```jsx
var MockComponent = React.createClass({
  render: function () {
    console.log(this.props.aProp);
    return <div>{this.props.children}</div>;
  }
});

var BizComponent = React.createClass({
  render: function () {
    return (
      <div>
        <button testRef="fuz">Fuz</button>
      </div>
    );
  }
});

var FooComponent = React.createClass({
  render: function () {
    return (
      <div>
        <div testRef="bar" />
        <div testRef="baz" aProp="hello">Baz</div>
        <div testRef="boz" aProp="hello">Boz</div>
      </div>
    );
  }
});

var fooTree = testTree(<FooComponent />, {
  stub: {
    bar: null,
    baz: <MockComponent />
    boz: <MockComponent aProp="foobar">Bazza</MockComponent>,
    biz: {
      fuz: null
    }
  }
});
fooTree.get("bar"); // -> null
fooTree.getIn(["biz", "fuz"]); // -> null
fooTree.get("baz"); // -> replaced with `MockComponent` and renders `Baz` string as child
fooTree.get("boz"); // -> replaced with `MockComponent` and renders `Bazza` string as child
```

__Notes__:
* You can use any falsy stub value other than `undefined` to completely remove a component (e.g. `null`, `false`).
* The stub object supports nesting; you can stub testRefs nested deep inside child composite components.
* Mock components are rendered with the new props (and children) of the mock component merged into the original props (and children) of the stubbed testRef. This behaviour is demonstrated in the example above; `baz` will log `hello` and have the child `Baz`, whilst `boz` will log `foobar` and have the child `Bazza`.


## API

### `testTree(<Component />, {options})`
Creates the tree and returns the root node.

*__Options__*
* `stub`: see section on [stubs](#stubs)
* `mount`: if true, the tree's container will be mounted into the body rather than being rendered entirely in memory. Useful if you need to test various styling aspects.
* `context`: use this option to pass through the context object required for your component. test-tree will automatically wrap your component and pass through the context.
* `wrap`: if true, the tree will be wrapped in an outer component. This is useful if you want to pass elements with testRefs directly into test-tree without them being contained in a component, e.g.:
```jsx
var tree = testTree(
  <ul testRefCollection="foo">
    <li testRef="bar" />
    <li />
  </ul>
, { wrap: true });
tree.foo; // exists
```

### `node.get(refName)`
Returns the node for the specified `testRef` or `testRefCollection` name.

### `node.getIn([refName])`
Same as `node.get(refName)` except it allows you to cleanly retrieve a node from deep down the testRef tree. For example, instead of:

```jsx
tree.get("foo").get("bar").get("baz").click();
```

you could write:

```jsx
tree.getIn(["foo", "bar", "baz"]).click();
```

### `rootNode.dispose()`
Safely unmount the tree. Will only unmount if component is already mounted. Can only be called on the root node of the tree.

### `node.state`
Returns the state of your component.

### `node.value`
Getter/setter for the element value. Should only be used if the component is a valid HTML element that accepts the value attribute.

### `node.simulate`
Instance of `React.addons.TestUtils.Simulate`, bound to the node. All its methods (beforeInput, blur, change, click, compositionEnd, compositionStart, compositionUpdate, contextMenu, copy, cut, doubleClick, drag, dragEnd, dragEnter, dragExit, dragLeave, dragOver, dragStart, drop, error, focus, input, keyDown, keyPress, keyUp, load, mouseDown, mouseEnter, mouseLeave, mouseMove, mouseOut, mouseOver, mouseUp, paste, reset, scroll, select, submit, touchCancel, touchEnd, touchMove, touchStart, wheel) can be called.

For example, to simulate double-clicking a node called `myButton`, use:

```javascript
myButton.simulate.doubleClick();
```

### `node.click()`
Shorthand method for simulating a click on the node's element.

### `node.getAttribute(attributeName)`
Returns the specified attribute from the node's element.

### `node.getClassName()`
Shorthand method for getting the class attribute of the node's element.

### `node.getProp()`
Returns the specified prop from the node's element.

### `node.isMounted()`
Returns true if the component/element is mounted, false if not.

### `node.getDOMNode()`
Returns the DOM node for the node.

### `node.element`
Reference to the original React element for the node.

### `node.innerText`
Returns the `innerText` of the element (or `textContent` if `innerText` not present).


## Updating to v1.0.0
React 0.14 introduced stateless function components. This new type of component cannot contain refs and also cannot have refs applied to them. The React team is trying to encourage refs to only be used for very specific purposes, which doesn't primarily include testing. We have taken the decision with `react-test-tree` to support the React team in their decision to separate the usage of refs from testing by switching to using the `testRef` and `testRefCollection` props instead of `ref` and `testRef`. Here is an example of a component pre-v1.0.0 and after:

```jsx
// Pre-v1.0.0
var MyComponent = React.createClass({
  render: function () {
    return (
      <div ref='foo' refCollection='bar' />
    );
  }
});

// v1.0.0
var MyComponent = React.createClass({
  render: function () {
    return (
      <div testRef='foo' testRefCollection='bar' />
    );
  }
});
```

Pre-v1.0.0, refs and refCollections were accessible as direct properties of the node. This led to issues with collisions between ref names and react-test-tree's node methods. In v1.0.0 the API has been changed to solve this problem. Node are now accessed using the `node.get()` and `node.getIn()` methods:

```jsx
// Pre-v1.0.0
tree.foo.bar.click();

// v1.0.0
tree.get("foo").get("bar").click();
// or
tree.getIn(["foo", "bar"]).click();
```

Post-v1.0.0 have a new option, `wrap` that you'll need to pass if you need a `testRef` or `testRefCollection` on the component that you're passing into `react-test-tree`.

```jsx
// Pre-v1.0.0
var tree = testTree(
  <ul refCollection="foo">
    <li ref="bar" />
    <li />
  </ul>);

tree.foo; // exists

// v1.0.0
var tree = testTree(
  <ul testRefCollection="foo">
    <li testRef="bar" />
    <li />
  </ul>
, { wrap: true });

tree.foo; // exists
```


## React versions
The master branch currently supports React 0.14 and is not backwards compatible. 

* __React 0.14:__ `react-test-tree@latest`
* __React 0.13/0.12:__ `react-test-tree@^0.3.1`


## Contributing

* `make bootstrap` - install dependencies
* `make test` - run unit tests
* `make build` - build into `dist` folder
* `make lint` - lint the project
* `make test-watch` - run karma with the watch option
* `make release` - increment and publish to npm


## Want to work on this for your day job?

This project was created by the Engineering team at [Qubit](http://www.qubit.com). As we use open source libraries, we make our projects public where possible.

We’re currently looking to grow our team, so if you’re a JavaScript engineer and keen on ES2016 React+Redux applications and Node micro services, why not get in touch? Work with like minded engineers in an environment that has fantastic perks, including an annual ski trip, yoga, a competitive foosball league, and copious amounts of yogurt.

Find more details on our [Engineering site](https://eng.qubit.com). Don’t have an up to date CV? Just link us your Github profile! Better yet, send us a pull request that improves this project.
