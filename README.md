# Proposal: Comparisons

Champions:
* Jacob Smith ([@JakobJingleheimer](https://github.com/JakobJingleheimer))
* Richard Gibson ([@gibson042](https://github.com/gibson042))

Authors:
* Jacob Smith ([@JakobJingleheimer](https://github.com/JakobJingleheimer))
* Ruben Bridgewater ([@BridgeAR](https://github.com/BridgeAR))

Reviewers:
* Jordan Harband ([@ljharb](https://github.com/ljharb))
* Olivier Flückiger ([@o-](https://github.com/o-))

## [Stage](https://tc39.github.io/process-document/)

**Current**: 0

**Requesting**: 1

## The Problem

Determine whether and/or how A and B deviate from each other—a very common need that is currently solved only for very narrow cases (primitives, and to some extent `JSON.stringify`able data structures). This issue has 2 parts: (deep) equality and details.

### Equality (currently)

Primitives are mostly trivial: `Object.is` provides the strictest comparison (SameValue), and `===` (IsStrictlyEqual) is only slightly looser, failing to differentiate oppositely-signed zeros and failing to equate NaNs.

```js
 'foo' === 'bar'
    1  ===  2
  true === false
```

But what "similar" means is not straightforward for objects. A human considers these "equal" (but the language does not):

```js
const a = { a: 1 };
const b = { a: 1 };
```
```js
const a = [1];
const b = [1];
```
```js
const a = new String('foo');
const b = new String('foo');
```

There is some variation in the ecosystem regarding the nuances of comparing objects.

Even more important is the details: Merely knowing that A and B differ is almost useless without knowing specifically _how_ they differ.

Annoying:
```js
if (A !== B) throw new Error('A does not equal B');
// Error: A does not equal B
```

Better
```js
if (A !== B) throw new Error(`${A} does not equal ${B}`);
// Error: 1 does not equal 2
```

But brittle
```js
if (A !== B) throw new Error(`${A} does not equal ${B}`);
// Error: [object Object] does not equal [object Object]
```

## Use-cases

### Production: Delta for HTTP `PATCH`

Many client-side apps manipulate data, sometimes very large data. That could be via a `<form>`, a text editor, or something else. Since the before and after are known, only the delta is needed (sent via [http `patch`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods/PATCH)).

```jsx
<Form onSubmit={submitPatch}>

function submitPatch(prev, next) {
  const patch = composeDelta(prev, next);

  fetch(…, {
    body: JSON.stringify(patch),
    method: 'PATCH',
  });
}
```

### Production: Logging

```js
log('bad data', compare(initiallyGood, nowBad));
```

### Production: State management

In frameworks such as React, state is often based on derived data in which updates don't always include a material change:

```js
setState((prev) => ({
  ...prev,
  x: x / 2,
}));
```

This is currently left up to the user to guard against because it's too difficult and expensive to for the library to check.

React tried to get this before (see [Prior Art → Shallow Equal](#proposals)).

### Production: Validation

Input from an uncontrolled origin:

```js
try {
  assert.is(
    total += value,
    NaN,
  );
} catch (err) {
  toast(…);
}
```

### Production: Virtual DOM

```jsx
{items.map(({ id, label }) => (
  <button onClick={() => remove(id)}>
    {label}
  </button>
))}
```

### Testing

```js
assert.equal(
  { foo: 1         },
  { foo: 1, bar: 2 },
);
```

## Explicitly out of scope

* This is not a test runner (`describe`, `it`, etc).
* This is not a test utility suite (`mock`, `stub`, etc).

## Solution (sketches)

### Compare

A function to deeply compare values.

```ts
function compare(
  expected: any,
  actual: any,
  options: CompareOptions,
): (true | IterableIterator<Deviation>) | undefined;
```

### CompareOptions

```ts
type CompareOptions = {
  iterate?:
    | 'none' // (default) return => boolean
    | 'deviations' // return => IterableIterator<Deviation>
  ,
  mode?:
    | 'value'                // (default) limit to the values of enumerable properties
    | 'descriptor'           // descend from an object to all of its property
                             // descriptors rather than its enumerable property values
    | 'descriptor-and-value' // like 'descriptor', but also invoke each getter into a value
                             // with a path distinct from that of the descriptor
  ,
  prototypes?:
    | 'same-value' // (default) compare the [[Prototype]]s of non-primitive values by SameValue
    | 'ignore'     // ignore [[Prototype]]
    | 'recurse'    // structurally compare the [[Prototype]]s of non-primitive values
};
```

<dl>
  <dt><em>iterate</em></dt>
  <dd>What the function returns<dl>
    <dt>"none"</dt>
    <dd>Return <code>true</code> when there is at least one deviation; <code>undefined</code> otherwise.</dd>
    <dt>"deviations"</dt>
    <dd>Return an iterable iterator of Deviations when there is at least one deviation; <code>undefined</code> otherwise.</dd>
  </dl></dd>

  <dt><em>mode</em></dt>
  <dd>What the function returns<dl>
    <dt>"value"</dt>
    <dd>Define the children of each non-leaf value as the values of its enumerable own properties.</dd>
    <dt>"descriptor"</dt>
    <dd>Define the children of each non-leaf value as the its property descriptors, regardless of enumerability.</dd>
    <dt>"descriptor-and-value"</dt>
    <dd>Define the children of each non-leaf value as the its property descriptors, regardless of enumerability, and additionally populate a <code>value</code> property by invoking each <code>get</code> accessor function.</dd>
  </dl></dd>

  <dt><em>prototypes</em></dt>
  <dd>What to do with the [[Prototype]] of each non-leaf value<dl>
    <dt>"same-value"</dt>
    <dd>Treat [[Prototype]] as a leaf, comparing by SameValue.</dd>
    <dt>"ignore"</dt>
    <dd>Ignore [[Prototype]].</dd>
    <dt>"recurse"</dt>
    <dd>Treat [[Prototype]] as a child node and recurse as with any other.</dd>
  </dl></dd>
</dl>

CompareOptions might also be extended to support more esoteric or particularly common configuration:
* Treat `-0` as equal to `0` (SameValueZero).
* Require the enumerated order of keys to match.
* Provide a classifier to mark nodes for leaf comparison (i.e., SameValue), ignoring, or [surrogate] recursion.
  * This is expected to be relevant for nodes whose value is a function or otherwise contains hidden state (ArrayBuffer, Promise, Map, Set, WeakMap, WeakSet, FinalizationRegistry, WeakRef, etc.).
* Treat the `constructor` of a non-primitive value as a child for either leaf comparison or recursion, even when ignoring [[Prototype]]
  * This is expected to be relevant for cross-realm objects, and also for TypedArray instances whose elements have values in the intersection of differing types.

### Deviations

```ts
type Deviation = {
  path: Array<string | symbol | { special: "descriptor" | "prototype" | "value" }>,
  actual: unknown,
  expected: unknown,
  kind: "extra" | "missing" | "mismatch",
};
```

<dl>
  <dt><em>path</em></dt>
  <dd>
    An array of segments necessary to reach the <code>actual</code> and <code>expected</code> nodes from their respective root.
    The path for a root itself is an empty array.
    The path to a property descriptor extends the path of its property by a <code>{ special: "descriptor" }</code> object,
    the path to a [[Prototype]] extends the path of its node by a <code>{ special: "prototype" }</code> object,
    and the path to a value returned from a manual getter invocation extends the path of the containing property by a <code>{ special: "value" }</code> object (all distinct from any property key).
  </dd>

  <dt><em>actual</em></dt>
  <dd>The leaf value from the <strong>second</strong> argument.</dd>

  <dt><em>expected</em></dt>
  <dd>The leaf value from the <strong>first</strong> argument.</dd>

  <dt><em>kind</em></dt>
  <dd><dl>
    <dt>"extra"</dt>
    <dd>when the node is present under <code>actual</code> but not <code>expected</code></dd>
    <dt>"missing"</dt>
    <dd>when the node is present under <code>expected</code> but not <code>actual</code></dd>
    <dt>"mismatch"</dt>
    <dd>when the node is present under both <code>actual</code> and <code>expected</code></dd>
  </dl></dd>
</dl>

### Equality

Non-enumerable properties are ignored unless `mode` is "descriptor" or "descriptor-and-value".

Leafs are compared with [SameValueZero](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-samevaluezero).

* TypedArrays containing the _same values in the same sequence_ are equal, except when not ignoring prototypes.
* A boxed primitive (eg `new Boolean(true)`) never equals an actual primitive (eg `true`).
* `NaN` equals `NaN` (for performance and sanity).
* Zero (`0`, `-0`, `+0`) equals zero (for now? possibly an option in `CompareOptions` in future).

Custom types are handled by HostTypes (to avoid custom comparison).

### Examples

#### Non-iterated: equal

```js
compare('a', 'a');

undefined
```

#### Non-iterated: unequal

```js
compare('a', 'b');

true
```

#### Iterated: unequal
```js
compare('a', 'b', { iterate: 'deviations' });

Iterator => IterableIterator(1) {
  {
    path: [],
    expected: 'a',
    actual: 'b',
    kind: 'mismatch',
  },
}
```

#### Non-iterated: object descriptor vs literal

```js
compare(
  Object.create({}, { foo: { enumerable: true, value: 'a' } }),
  { foo: 'a' },
);

undefined
```

```js
compare(
  Object.create({}, { foo: { enumerable: true, value: 'a' } }),
  { foo: 'a' },
  { reasons: { descriptor: true } },
);

true
```

```js
compare(
  Object.create({}, { foo: { enumerable: true, get: () => 'a' } }),
  { foo: 'a' },
);

undefined
```

```js
compare(
  Object.create({}, { foo: { enumerable: true, get: () => 'a' } }),
  { foo: 'a' },
  { reasons: { descriptor: true } },
);

true
```

#### Iterated: object descriptor vs literal

```js
compare(
  Object.create({}, { foo: { enumerable: true, value: 'a' } }),
  { foo: 'a' },
  { iterate: 'deviations', mode: 'descriptor-and-value' },
);

Iterator => IterableIterator(2) {
  {
    path: ['a', { special: 'descriptor' }, 'writable'],
    expected: false,
    actual: true,
    kind: 'mismatch',
  },
  {
    path: ['a', { special: 'descriptor' }, 'configurable'],
    expected: false,
    actual: true,
    kind: 'mismatch',
  },
}
```

```js
compare(
  Object.create({}, { foo: { enumerable: true, get: () => 'a' } }),
  { foo: 'a' },
  { iterate: 'deviations', mode: 'descriptor-and-value' },
);

Iterator => IterableIterator(4) {
  {
    path: ['a', { special: 'descriptor' }, 'get'],
    expected: <function "get">,
    actual: undefined,
    kind: 'missing',
  },
  {
    path: ['a', { special: 'descriptor' }, 'configurable'],
    expected: false,
    actual: true,
    kind: 'mismatch',
  },
  {
    path: ['a', { special: 'descriptor' }, 'value'],
    expected: undefined,
    actual: 'a',
    kind: 'extra',
  },
  {
    path: ['a', { special: 'descriptor' }, 'writable'],
    expected: undefined,
    actual: true,
    kind: 'extra',
  },
}
```

```js
compare(
  Object.create({}, { foo: { enumerable: false, configurable: false, get: () => 'a' } }),
  Object.create({}, { foo: { enumerable: false, configurable: true, get: () => 'b' } }),
  { iterate: 'deviations', mode: 'descriptor-and-value' },
);

Iterator => IterableIterator(2) {
  {
    path: ['a', { special: 'descriptor' }, 'configurable'],
    expected: false,
    actual: true,
    kind: 'mismatch',
  },
  {
    path: ['a'],
    expected: 'a',
    actual: 'b',
    kind: 'mismatch',
  },
}
```

#### Iterated: type unequal

```js
compare('1', 1, { iterate: 'deviations' });

Iterator => IterableIterator(1) {
  {
    path: [],
    expected: '1',
    actual: 1,
    kind: 'mismatch',
  },
}
```

#### Iterated: non-enumerable value

```js
compare(
  Object.create({}, { foo: { enumerable: false, value: 'a' } }),
  { foo: 'a' },
  { iterate: 'deviations' },
);

Iterator => IterableIterator(1) {
  {
    path: ['foo'],
    expected: undefined,
    actual: 'a',
    kind: 'extra',
  },
}
```

#### Iterated: non-enumerable getter

```js
compare(
  Object.create({}, { foo: { get: () => 'a' } }),
  { foo: 'a' },
  { iterate: 'deviations' },
);

Iterator => IterableIterator(1) {
  {
    path: ['foo'],
    expected: undefined,
    actual: 'a',
    kind: 'extra',
  },
}
```

#### Iterated: multiple leafs unequal

```js
compare(
  { foo: 'a', bar: 'c' },
  { foo: 'b', bar:  2  },
  { iterate: 'deviations' },
);

Iterator => IterableIterator(2) {
  {
    path: ['foo'],
    expected: 'a',
    actual: 'c',
    kind: 'mismatch',
  },
  {
    path: ['bar'],
    expected: 'c',
    actual: 2,
    kind: 'mismatch',
  },
}
```

#### Iterated: multiple leafs unequal and missing

```js
compare(
  { foo: { bar: 'a'           } },
  { foo: { bar: 'b', qux: 'c' } },
  { iterate: 'deviations' },
);

Iterator => IterableIterator(2) {
  {
    path: ['foo', 'bar'],
    expected: 'a',
    actual: 'b',
    kind: 'mismatch',
  },
  {
    path: ['foo', 'qux'],
    expected: undefined,
    actual: 'c',
    kind: 'extra',
  },
}
```

#### Iterated: multiple leafs unequal and prototype
```js
compare(
  { foo: 'a', __proto__: null },
  { foo: 'b' },
  { iterate: 'deviations' },
);

Iterator => IterableIterator(2) {
  {
    path: [{ special: 'prototype' }],
    expected: null,
    actual: Object.prototype,
    kind: 'mismatch',
  },
  {
    path: ['foo'],
    expected: 'a',
    actual: 'b',
    kind: 'mismatch',
  },
}
```

#### Iterated: multiple array items unequal and missing
```js
compare(
  ['a', 'b', 'c'     ],
  ['a', 'b', 'd', 'e'],
  { iterate: 'deviations' },
);

Iterator => IterableIterator(2) {
  {
    path: ['2'],
    expected: 'c',
    actual: 'd',
    kind: 'mismatch',
  },
  {
    path: ['3'],
    expected: undefined,
    actual: 'e',
    kind: 'extra',
  },
}
```

## Sibling proposals

The current proposal is useful on its own and sets a foundation for the following to be addressed subsequently.

The current proposal does not include features likely to attract customisation, so punting these delays the need to determine how customisation will be facilitated.

* [Inspector](https://github.com/tc39/proposal-inspector)
* [Modes](https://github.com/JakobJingleheimer/proposal-modes)

## Other related proposals

* [Pattern Matching](https://github.com/tc39/proposal-pattern-matching)

## Prior art

### Assertions and expectations

The vast majority of ECMAScript engineers use one of 2 forms: `assert` and `expect`. These come from one of ~4 libraries: `chai` (`20M` weekly), `jasmine` (`1.4M` weekly), `jest` (`29M` weekly), `node:assert` (indeterminable). These are direct competitors, so we can assume there is no overlap and the numbers are summable: at least `~51M` weekly (probably significantly higher when `node:assert` numbers are added).

#### Assert

* `node:assert` and `chai`'s TDD set have large overlap.

#### Expect

* `jasmine` and `jest` are (nearly?) identical with dedicated methods: `expect(a).toEqual(b)`
* `chai`'s BDD set is a chain-style that builds upon itself: `expect(a).to.equal(b)`

### Neighbours

Many major languages natively include a form of assertion. To name a relevant few:

* [`c++`](https://en.cppreference.com/w/cpp/error/assert)
* [`go`](https://pkg.go.dev/github.com/stretchr/testify/assert)
* [`kotlin`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/assert.html)
* [`python`](https://docs.python.org/3/reference/simple_stmts.html#the-assert-statement)
* [`rust`](https://doc.rust-lang.org/std/macro.assert.html)

### Proposals

* [Array equality](https://github.com/tc39/proposal-array-equality)
* [Object deep equal](https://github.com/misha98857/proposal-object-deep-equal)
* [Object shallow equal](https://github.com/sebmarkbage/ecmascript-shallow-equal)
