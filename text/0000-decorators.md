- Start Date: 2018-10-20
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Bring Decorators into Ember

## Summary

Decorators provide a way to abstract functionality and improve the developer experience of working with native classes.  This RFC outlines the implementation and rollout plan for bringing decorators to ember's computed properties (and other behavior) for use in native classes.

Additional Reading:

 - [Why go native (from ember-decorators)](https://ember-decorators.github.io/ember-decorators/docs/why-go-native)
 - [Native Class Roadmap RFC](https://github.com/pzuraq/emberjs-rfcs/blob/b47e7f9ec4f02c7d27d50de64691130e7d22747d/text/0000-native-class-roadmap.md)

_NOTE: This RFC requires that decorators hit Stage 3 and should not be accepted until then._

## Motivation

Decorators bring a more natural way of declaring computed properties to native classes than the current computed macros.

For example, using the old `computed`:

```ts
import Component from '@ember/component';
import { computed } from '@ember/object';

export default class Profile extends Component {
  firstName = 'Diana';
  lastName = 'Prince';

  fullName = computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  });
}
```

However, with decorators, currently provided by ember-decorators:

```ts
import Component from '@ember/component';
import { computed } from '@ember-decorators/object';

export default class Profile extends Component {
  firstName = 'Diana';
  lastName = 'Prince';

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

Ultimately, at the end of the RFC, we want to be at a place where we can use method techniques with the same import, and then eventually remove the old functionality.

```ts
import Component from '@ember/component';
import { computed } from '@ember/object';

export default class Profile extends Component {
  firstName = 'Diana';
  lastName = 'Prince';

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```



## Detailed design

Most of the complexity in implementing this RFC will be around wrapping all of the computed property macros with an optional decorator wrapper around the current computed property macro.

Here is the current implementation of `computed` from [@ember/-internals/metal/lib/computed.ts](https://github.com/emberjs/ember.js/blob/be9552f29a329f3b98f1651a075a49f3ed2c92c6/packages/%40ember/-internals/metal/lib/computed.ts#L612)
```ts
export default function computed(...args: (string | ComputedPropertyConfig)[]): ComputedProperty {
  // details omitted, not relevant
}
```

And the current implementation of the `@computed` from [@ember-decorators/object/addon/index.js](https://github.com/ember-decorators/ember-decorators/blob/master/packages/object/addon/index.js#L101) (with assertions and other validations removed)

```ts
import { computed as emberComputed } from '@ember/object';
// ...
export const computed = computedDecoratorWithParams(({ key, descriptor }, params) => {
  let { get, set } = descriptor;
  // ...
  return emberComputed(...params, { get, set: setter });
});
```

The gist of the strategy from `@ember-decorators` is that all of the original computed property  macros are used, but wrapped in decorator-specific creation logic. The `computedDecoratorWithParams` abstracts a lot of this decorator-specific functionality, but it would be out of scope for the written portion of this RFC -- an accompanying pull request will be submitted along side this RFC to demonstrate how computed properties will interoperate with decorators and existing code using `computed` from `@ember/object`.

Both the existing `@ember` `computed` and the `@ember-decorators` `@computed` do some validation against the passed arguments to determine how to build the property. For example, the `computedDecoratorWithParams` can handle the scenarios where `@computed` has and does not have arguments passed by checking both the existence of certain parameters, the number of parameters, and the types of the parameters. This technique would be extended to support both `computed` and `@computed` usage from the same import.

This process would be repeated for each macro. The following tables represent the current mapping of `@ember-decorators` functionality with that found in `@ember`

| @ember-decorators/service | @ember/service |
| --- | --- |
| service | |

| @ember-decorators/controller | @ember/service |
| --- | --- |
| controller | |

| @ember-decorators/object | @ember/object |
| --- | --- |
| computed | computed |
| action |  |
| observes | observer |
| unobserves |  |
| on | in `@ember/object/evented` |
| off |  |
| readOnly |  |
| volatile |  |

| @ember-decorators/object/computed | @ember/object/computed |
| --- | --- |
| macro | |
| alias | alias |
| and | and |
| bool | bool |
| collect | collect |
| deprecatingAlias | deprecatingAlias |
| empty | empty |
| equal | equal |
|  | expandProperties |
| filter | filter |
| filterBy | filterBy |
| gt | gt |
| gte | gte |
| intersect | intersect |
| lt | lt |
| lte | lte |
| map | map |
| mapBy | mapBy |
| match | match |
| max | max |
| min | min |
| none | none |
| not | not |
| notEmpty | notEmpty |
| | oneWay |
| or | or |
| overridableReads | |
| | readOnly |
| reads | reads |
| setDiff | setDiff |
| sort | sort |
| sum | sum |
| union | union|
| uniq | uniq |
| uniqBy | uniqBy |


Out of scope: `@ember-decorators/data`, `@ember-decorators/component`.

ember data decorators should be handled in the ember-data repository, and should go through its own RFC process (which would hopefully be easy after this one).

the component decorators are all being omitted because of the direction components are heading. With the eventual glimmer components, `attribute`, `className`, `classNames`, `tagNames`, and `layout` will not be needed.


## How we teach this

Because the goal of this change is to only add capabilities and make absolutely no breaking changes with the already built in computed property macros, there should be 0 friction for non `@ember-decorators` users to migrate.

There are tree starting points for people in existing ember apps:
 1. old pre-ES class syntax
    - requires ES class codemod, then scenario 2
 2. ES class syntax without decorators
    - codemod to convert legacy computed properties to `@ember` decorators
 3. ES class Syntax with `@ember-decorators`;
    - codemod to convert `@ember-decorators` imports to `@ember` imports and also replace the appropriate incompatible / mismatched decorators with ones of equivalent functionality.


Once there is are codemods for all the scenarios, the legacy syntax can be deprecated, and eventually removed in favor of decorator-only usage.

## Drawbacks

`@readOnly` in `@ember-decorators` is totally different from `readOnly` in `@ember`.

The mapping:

| `@ember-decorators` | `@ember` |
| --- | --- |
| `@readOnly` | `computed().readOnly` |
| `@reads` | `computed.readOnly()` |
| `@overridableReads` | `computed.reads()` |

So, if these 3 decorators are to exist all at `@ember/object/computed`, there will be conflicts with existing `reads` and `readOnly` macros.


## Unresolved questions

- Should the `@ember-decorators/utils` package get pulled in as well -- it would greatly help with the conversion of all the computed properties.
TBD?
- Decide on a migration path for `readOnly` and `reads`
