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
): (true | Iterator<Deviation>) | undefined;
```

### CompareOptions

```ts
type CompareOptions = {
  mode?:
    | 'fast' // (default) return => boolean
    | 'full' // return => Iterator<Deviation>
  ,
  reasons?: Partial<{
    constructor: boolean,     // default: `false`
    descriptors: boolean,     // default: `false`
    promise: 'ref' | 'value', // default: `'value'`
    prototype: boolean,       // default: `false`
    weak: 'ref' | 'value',    // default: `'value'`
  }>,
};
```

<dl>
  <dt><em>mode</em></dt>
  <dd>How the comparison reports the result</dd>

  <dt><em>mode</em> <strong title="default value"><code>fast</code></strong><dt>
  <dd>Return <code>true</code> when deviation(s) exist or <code>undefined</code> when no deviation exist.</dd>

  <dt><em>mode</em> <code>full</code><dt>
  <dd>Return an <code>Iterator</code> of <code>Deviations</code> with all deviations, or <code>undefined</code> when no deviation exist.</dd>

  <dt><em>reasons</em></dt>
  <dd>Whether/how to handle more esoteric cases when determining differences.</dd>

  <dt><em>reasons.constructor</em> <strong title="default value"><code>false</code></strong> | <code>true</code></dt>
  <dd>Whether to consider constructor. This affects, amongst others, Box Primitives (<code>new Boolean(true)</code> vs <code>true</code>) and <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray">TypedArrays</a> (<code>new Int8Array([1,2])</code> vs <code>new Uint8Array([1,2])</code>) where differenes are pedantic.</dd>

  <dt><em>reasons.descriptors</em> <strong title="default value"><code>false</code></strong> | <code>true</code></dt>
  <dd>Whether to consider property non-enumerability descriptors (configurable, getter vs value, writeable).</dd>

  <dt><em>reasons.promise</em> <strong title="default value"><code>'ref'</code></strong> | <code>'value'</code></dt>
  <dd>How to determine equality of promises.</dd>

  <dt><em>reasons.prototype</em> <strong title="default value"><code>false</code></strong> | <code>true</code></dt>
  <dd>Whether to consider prototype.</dd>

  <dt><em>reasons.weak</em> <strong title="default value"><code>'ref'</code></strong> | <code>'value'</code></dt>
  <dd>How to determine equality of Weak objects (<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap"><code>WeakMap</code></a>, <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef"><code>WeakRef</code></a>, <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakSet"><code>WeakSet</code></a>).</dd>
</dl>

### Deviations

```ts
type Deviations = Iterator<
  string, // "foo['bar-qux']['zed']"
  {
    actual:
      | bigint
      | boolean
      | null
      | number
      | string
      | symbol
      | undefined
    ,
    expected:
      | bigint
      | boolean
      | null
      | number
      | string
      | symbol
      | undefined
    ,
    reason: {
      constructor?: boolean,
      descriptor?: boolean,
      enumerability: boolean,
      equality: boolean,
      missing: boolean,
      prototype?: boolean,
      reference: boolean,
      type: boolean,
    },
  },
>;
```

<dl>
  <dt><em>key</em></dt>
  <dd>A <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Working_with_objects#accessing_properties">bracket-notation</a> path like <code>"foo['bar-qux']['zed']"</code>. When comparing non-objects, (eg strings), the path is an empty string <code>""</code>.</dd>

  <dt><em>actual</em></dt>
  <dd>The leaf value from the <strong>second</strong> argument.</dd>

  <dt><em>expected</em></dt>
  <dd>The leaf value from the <strong>first</strong> argument.</dd>

  <dt><em>reason</em></dt>
  <dd>
  The reason(s) comparison failed to match.

    {
      expected: undefined,
      actual: undefined,
      reason: { missing: true, … },
    }

  Reason(s) are general to specific, outter-most to inner-most: `compare(true, new Date())` → "type" is the reason for the deviation. Specific order is engine-defined.
  </dd>
</dl>

### Equality

To avoid suppressing potentially relevant differences, primitive values are compared with [SameValue](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-samevalue) (which does not differentiate any NaN but does differentiate `-0` from `0`/`+0`). This might become configurable via CompareOptions.

* TypedArrays containing the _same values in the same sequence_ are equal, except when `CompareOptions.reasons.constructor` is enabled.
* A box primitive (eg `new Boolean(true)`) equals its primitive (eg `true`), except when `CompareOptions.reasons.constructor` is enabled.

Custom types are handled by HostTypes (to avoid custom comparison).

### Examples

#### Fast mode: equal

```js
compare('a', 'a');

undefined
```

#### Fast mode: unequal

```js
compare('a', 'b');

true
```

#### Full mode: unequal
```js
compare('a', 'b', { mode: 'full' });

Iterator => Iterable(1) {
  "" => {
    expected: 'a',
    actual: 'b',
    reason: { equality: true, … },
  },
}
```

#### Fast mode: object descriptor vs literal

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

#### Full mode: type unequal

```js
compare('1', 1, { mode: 'full' });

Iterator => Iterable(1) {
  "" => {
    expected: '1',
    actual: 1,
    reason: { type: true, … },
  },
}
```

#### Full mode: non-enumerable value

```js
compare(
  Object.create({}, { foo: { enumerable: false, value: 'a' } }),
  { foo: 'a' },
  { mode: 'full' },
);

Iterator => Iterable(1) {
  "foo" => {
    expected: undefined,
    actual: 'a',
    reason: { enumerability: true, … },
  },
}
```

#### Full mode: non-enumerable getter

```js
compare(
  Object.create({}, { foo: { get: () => 'a' } }),
  { foo: 'a' },
  {
    mode: 'first',
  },
);

Iterator => Iterable(1) {
  "foo" => {
    expected: undefined,
    actual: 'a',
    reason: { enumerability: true, … },
  },
}
```

#### Full mode: multiple leafs unequal, plus red-herring from type-mismatch

```js
compare(
  { foo: 'a', bar: 'c' },
  { foo: 'b', bar:  2  },
  {
    mode: 'full',
  },
);

Iterator => Iterable(2) {
  "foo" => {
    expected: 'a',
    actual: 'c',
    reason: { equality: true, … },
  },
  "bar" => {
    expected: 'c',
    actual: 2,
    reason: { equality: true, … },
  },
}
```

#### Full mode: multiple leafs unequal and missing

```js
compare(
  { foo: { bar: 'a'           } },
  { foo: { bar: 'b', qux: 'c' } },
  { mode: 'full' },
);

Iterator => Iterable(2) {
  "foo['bar']" => {
    expected: 'a',
    actual: 'b',
    reason: { equality: true, … },
  },
  "foo['bar']['qux']" => {
    expected: undefined,
    actual: 'c',
    reason: { missing: true, … },
  },
}
```

#### Full mode: multiple leafs unequal and prototype
```js
compare(
  { foo: 'a', __proto__: null },
  { foo: 'b' },
  {
    mode: 'full',
    reasons: { prototype: true },
  },
);

Iterator => Iterable(2) {
  "[[Prototype]]" => {
    expected: null,
    actual: Object,
    reason: { prototype: true, … },
  },
  "foo" => {
    expected: 'a',
    actual: 'b',
    reason: { equality: true, … },
  },
}
```

#### Full mode: multiple array items unequal and missing
```js
compare(
  ['a', 'b', 'c'     ],
  ['a', 'b', 'd', 'e'],
  {
    mode: 'full',
  },
);

Iterator => Iterable(1) {
  "2" => {
    expected: 'c',
    actual: 'd',
    reason: { equality: true, … },
  },
  "3" => {
    expected: undefined,
    actual: 'e',
    reason: { missing: true, … },
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
