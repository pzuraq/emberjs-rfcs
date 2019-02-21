- Start Date: 2019-02-20
- Relevant Team(s): Ember.js, Ember Data, Learning
- RFC PR:
- Ember Issue: (leave this empty)

# Array Functions

## Summary

This RFC proposes a way to work with arrays in ember without requiring that we
extend the global `Array` prototype or manually wrap arrays with
`Ember.A(array)`. In addition, it proposes a way to provide integration for
arrays with _tracked properties_.

Specifically, for each method on both the global `Array` class and the
`MutableArray` mixin we will provide a similarly named function that takes an
array as its first argument followed by the arguments to the method.

For example, instead of calling `array.pushObject(item)` you can call
`pushObject(array, item)`. For a full list see the expandible list in [Detailed
design](#detailed-design).

These functions will be added as an official Ember addon, rather than as part of
Ember's core codebase.

## Motivation

Currently it is not a great experience to write an app with `Array` prototype
extensions disabled. You end up needing to wrap arrays everywhere with
`Ember.A(array)` which loops through each method on the `NativeArray` mixin and
assigns it as an enumerable own property on the array.

The are several problems here:

- It's annoying to defensively write `Ember.A()` around every array
- It's wasteful to looping through all the array methods and assign them all.
  Especially if you're only going to use one of them
- Iterating over an array with `for..in` or `Object.keys` will not behave as
  generally expected because the methods are assigned enumerably (for
  performance reasons)

All of these problems go away when using array functions instead of array
methods.

It's also worth noting that when you using fastboot you are required to have
`Array` prototype extensions disabled currently and thus forced to deal with
these issues.

Additionally, native arrays currently do not integrate well with tracked
properties, even when wrapped with `Ember.A()`. This is because every single
method that can be called on the array must be wrapped in order for autotracking
to pick up that the array has been used. `Ember.A()` does _not_ wrap any
existing native functions, and even if it did, there are functions which are
implemented on native arrays but _not_ on `MutableArray`, such as
`Array.prototype.join()`.

Tracked properties can integrate in other ways with arrays (e.g. by using
immutable patterns, unidirectional data flow, or native `Proxy`) but most of
these patterns will be new to existing Ember applications, and will take time to
adopt (or in the case of `Proxy`, dropping support for IE11). Providing these
functions is essentially a compatibility layer, similar to the way we support
usage of `get` and `set` within tracked getters/contexts. In time, the community
will be able to migrate off of them, but for now they will allow us to move
forward.

These functions will be added as an official addon, allowing users to opt out of
adding them to their codebase if they _are_ using other patterns for arrays. It
will be installed with the default blueprint, and referenced in the guides, but
it won't be part of the core Ember codebase.

## Detailed design

For each method in the `Array` class, and each method in the `MutableArray`
mixin we will export a similarly named function from `@ember/array`.

For example, the method `array.objectAt(index)` will map to the function
`objectAt(array, index)` with implementation:

```js
function objectAt(array, index) {
  get(array, '[]');

  if (Array.isArray(array)) {
    return array[index];
  } else {
    return array.objectAt(index);
  }
}
```

As you can see, the native array case is handled explicitly in order to avoid
assuming that the global `Array` prototype has been extended, but otherwise we
just delegate to the `objectAt` method. The call to `get('[]')` entangles the
array's `[]` property with the tracking stack. That property is what will be
updated whenever the array is mutated with one of the mutation methods such as
`pushObject`, matching the current behavior.

For reference, below is an expandable list of all array functions we will need.

<details>
  <summary>List of Array functions (32)</summary>
  <code>
    <ul>
      <li>concat</li>
      <li>copyWithin</li>
      <li>entries</li>
      <li>every</li>
      <li>fill</li>
      <li>filter</li>
      <li>find</li>
      <li>findIndex</li>
      <li>flat</li>
      <li>flatMap</li>
      <li>forEach</li>
      <li>includes</li>
      <li>indexOf</li>
      <li>join</li>
      <li>keys</li>
      <li>lastIndexOf</li>
      <li>map</li>
      <li>pop</li>
      <li>push</li>
      <li>reduce</li>
      <li>reduceRight</li>
      <li>reverse</li>
      <li>shift</li>
      <li>slice</li>
      <li>some</li>
      <li>sort</li>
      <li>splice</li>
      <li>toLocaleString</li>
      <li>toSource</li>
      <li>toString</li>
      <li>unshift</li>
      <li>values</li>
    </ul>
  </code>
  <summary>List of MutableArray functions (43)</summary>
  <code>
    <ul>
      <li>addArrayObserver</li>
      <li>addObject</li>
      <li>addObjects</li>
      <li>any</li>
      <li>arrayContentDidChange</li>
      <li>arrayContentWillChange</li>
      <li>clear</li>
      <li>compact</li>
      <li>filterBy</li>
      <li>findBy</li>
      <li>getEach</li>
      <li>includes</li>
      <li>indexOf</li>
      <li>insertAt</li>
      <li>invoke</li>
      <li>isAny</li>
      <li>isEvery</li>
      <li>lastIndexOf</li>
      <li>mapBy</li>
      <li>objectAt</li>
      <li>objectsAt</li>
      <li>popObject</li>
      <li>pushObject</li>
      <li>pushObjects</li>
      <li>reject</li>
      <li>rejectBy</li>
      <li>removeArrayObserver</li>
      <li>removeAt</li>
      <li>removeObject</li>
      <li>removeObjects</li>
      <li>replace</li>
      <li>reverseObjects</li>
      <li>setEach</li>
      <li>setObjects</li>
      <li>shiftObject</li>
      <li>slice</li>
      <li>sortBy</li>
      <li>toArray</li>
      <li>uniq</li>
      <li>uniqBy</li>
      <li>unshiftObject</li>
      <li>unshiftObjects</li>
      <li>without</li>
    </ul>
  </code>
</details>

## How we teach this

This functions must obviously be documented under the `@ember/array` package in
the API docs.

We should encourage people to use the phrase "array functions" to describe these
functions and "array methods" to describe the method versions.

We should encourage their usage in addons and fastboot apps instead of using
`Ember.A`.

In the future, we may want to encourage their usage in all apps so that we can
deprecate extending the global `Array` prototype, but that is out of scope for
this RFC.

## Drawbacks

This RFC proposes a considerable increase in API surface area however it's
mostly duplicating an existing, well known API so it does not increase the
mental burden very much.

I also expect the total number of bytes added to the framework be fairly small
since it is mostly just shuffling around existing code.

## Alternatives

- This could be made as an official addon, `ember-array-functions`. This would
