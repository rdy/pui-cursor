# PUI Cursor
[![Build Status](https://travis-ci.org/pivotal-cf/pui-cursor.svg)](https://travis-ci.org/pivotal-cf/pui-cursor)

Utility designed for immutable data in a React flux architecture.
Also contains pure render mixin for cursors.

## Table of Contents

* [Overview](#cursors)
* [API](#API)
    * [get()](#get)
    * [set()](#set)
    * [refine()](#refine)
    * [merge()](#merge)
    * [push()](#push)
    * [apply()](#apply)
    * [remove()](#remove)
    * [splice()](#splice)
    * [unshift()](#unshift)

##Cursors

PUI Cursors are simplified versions of [Om Cursors](https://github.com/omcljs/om/wiki/Cursors) designed for use with a
React Flux architecture. It enables targeted, immutable updates to data; these updates are particularly useful for updating a store in
React.

A cursor takes in data and a callback. The callback is used to propagate data into an app and create a new cursor with
the updated data.

A minimal example of cursor setup is below:

```js
const Cursor = require('pui-cursor');
const React = require('react');
const Zoo = require('./zoo');

class Application extends React.Component {
  constructor(props, context) {
    super(props, context);
    this.state.store = {animals: {lion: 'Larry', seal: 'Sebastian'}};
  }

  render() {
    const $store = new Cursor(this.state.store, updatedStore => this.setState({store: updatedStore}));

    return <Zoo animals={this.state.store.animals} $store={$store}/>;
  }
}
```

Our convention is to prefix Cursor instances with `$`, like `$store` in the above example. This convention
differentiates the cursor from the data it contains.

For example in this setup, if the `Zoo` component calls `this.props.$store.merge({visitors: ['Charles', 'Adam', 'Elena']})`,
the application store will now have `visitors` in addition to `animals`.

##API

PUI Cursor provides wrappers for the [React immutability helpers](https://facebook.github.io/react/docs/update.html).
These wrappers allow you to transform the data in your cursor; the transformation you specify is applied and the new result
is used to update the cursor value.

###`get()`

Returns your current node

```
var store = {animals: {lion: 'Larry', seal: 'Sebastian'}};
const $store = new Cursor(store, callback);
```

The cursor never updates its own data structure, so `get` is prone to returning stale data.

If you execute `$store.refine('animals', 'lion').set('Scar').get();`, it will return 'Larry' instead of 'Scar'

In general, we recommend that you not use `get` and instead access the store directly with props.
If you want to use `get`, ensure that you are using the newest version of your Cursor.

###`set()`

Sets the data for your current node. If you call `set at the top of the data tree, it sets the data for every node.

```
var store = {animals: {lion: 'Larry', seal: 'Sebastian'}};
const $store = new Cursor(store, callback);
```


If you execute `$store.refine('animals').set({lion: 'Simba', warthog: 'Pumba'})`,
the callback will be called with `{animals: {lion: 'Simba', warthog: 'Pumba'}}`.

###`refine()`

Changes where you are in the data tree. You can provide `refine` with multiple arguments to take you deeper into the tree.

If the data node that you're on is an **object**, refine expects a string that corresponds to a key in the object.

```
var store = {animals: {lion: 'Larry', seal: 'Sebastian'}};
const $store = new Cursor(store, callback);
```

For example, `$store.refine('animals', 'seal').get();`,  will return 'Sebastian'.

If the data node that you're on is an **array of objects**, refine expects an index or an element of the array.

```
var hey = {greeting: 'hey'}
var hi = {greeting: 'hi'}
var hello = {greeting: 'hello'}
var store = {greetings: [hey, hi, hello]}
const $store = new Cursor(store, callback);
```

then `$store.refine('greetings', 1, 'greeting').get()` will return 'hi'. If you have the element of an array but not the index,
`$store.refine('greetings', hi, 'greeting').get()` will also return 'hi'.

###`merge()`

Merges data onto the object at your current node

```
$store.refine('animals').merge({squirrel: 'Stumpy'})
```

The callback will be called with `{animals: {lion: 'Larry', seal: 'Sebastian', squirrel: 'Stumpy'}}`.

###`push()`

Pushes to the array at your current node

```
var hey = {greeting: 'hey'}
var hi = {greeting: 'hi'}
var hello = {greeting: 'hello'}
var yo = {grettings: 'yo'}
var store = {greetings: [hey, hi, hello]}
const $store = new Cursor(store, callback);
```

If you execute `$store.refine('greetings').push({greeting: 'yo'})`, the callback will be called with `{greetings: [hey, hi, hello, yo]}`.

###`apply()`

If the simpler functions like `set`, `merge`, or `push` cannot describe the update you need,
you can always call `apply` to specify an arbitrary transformation.

Example:

```js
var currentData = {foo: 'bar'}
var cursor = new Cursor(currentData, function(newData){ this.setState({data: newData})
cursor.apply(function(shallowCloneOfOldData) {
  shallowCloneOfOldData.foo += 'bar';
  return shallowCloneOfOldData;
});
```

__Warning:__ The callback for `apply` is given a shallow clone of your data
(this is the behavior of the apply function in the React immutability helpers).
This can cause unintended side effects, illustrated in the following example:

```js
var currentData = {animals: {mammals: {felines: 'tiger'}}}
var cursor = new Cursor(currentData, function(newData){ this.setState({data: newData})});

cursor.apply(function(shallowCloneOfOldData) {
  shallowCloneOfOldData.animals.mammals.felines = 'lion';
  return shallowCloneOfOldData;
});
```

Since the data passed into the callback is a shallow clone of the old data, values that are nested more than one level
deep are not copied, so `shallowCloneOfOldData.animals.mammals` will refer to the exact same object in memory as `currentData.animals.mammals`.

The above version of `apply` will mutate the previous data in the cursor (`currentData`) in addition to updating the cursor.
As a side effect, `PureRenderMixin` will not detect any changes in the data when it compares previous props and new props.
To safely use `apply` on nested data, you need to use the React immutability helpers directly:

```js
var reactUpdate = require('react/lib/update');

cursor.apply(function(shallowCloneOfOldData) {
  return reactUpdate.apply(shallowCloneOfOldData, {
    animals: {
      mammals: {
        felines: {$set: 'lion'}
      }
    });
  });
});
```

###`remove()`

Removes your current node

If the current node is an object and you call remove(key), remove deletes the key-value.

```
var store = {animals: {lion: 'Larry', seal: 'Sebastian'}};
const $store = new Cursor(store, callback);
```

If you execute `$store.refine('animals', 'seal').remove()`, the callback will be called with `{animals: {lion: 'Larry'}}`.

If the current node is an array:

```
var hey = {greeting: 'hey'}
var hi = {greeting: 'hi'}
var hello = {greeting: 'hello'}
var store = {greetings: [hey, hi, hello]}
const $store = new Cursor(store, callback);
```

If you execute `$store.refine('greetings').remove(hello)`, the callback will be called with `{greetings: [hey, hi]}`.

###`splice()`

Splices an array in a very similar way to `array.splice`. It expects an array of 3 elements as an argument.
The first element is the starting index, the second is how many elements from the start you want to replace, and the
third is what you will replace those elements with.

```
var hey = {greeting: 'hey'}
var hi = {greeting: 'hi'}
var hello = {greeting: 'hello'}
var yo = {greeting: 'yo'}
var store = {greetings: [hey, hi, hello]}
const $store = new Cursor(store, callback);
```

If you execute `$store.refine('greetings').splice([2, 1, yo])`, the callback will be called with `{greetings: [hey, hi, yo]}`.

###`unshift()`

Adds an element to the start of the array at the current node.

```
var hey = {greeting: 'hey'}
var hi = {greeting: 'hi'}
var hello = {greeting: 'hello'}
var yo = {greeting: 'yo'}
var store = {greetings: [hey, hi, hello]}
const $store = new Cursor(store, callback);
```

If you execute `$store.refine('greetings').unshift(yo)`, the callback will be called with `{greetings: [yo, hey, hi, hello]}`

---

(c) Copyright 2015 Pivotal Software, Inc. All Rights Reserved.
