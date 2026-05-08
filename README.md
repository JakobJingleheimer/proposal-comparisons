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

Determine how A and B deviate—a very common need that is currently ~half-solved. This issue has 2 parts: (deep) equality and details.

### Equality (currently)

Non-objects are straightforward and trivial:

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

Even more important than A vs B is the details: Merely knowing B is unexpected is almost useless without knowing specifically _how_ they differ.

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

Such as in React, is often based on derived data that could actually result in no change:

```js
setState((prev) => ({
  ...prev,
  x: x / 2,
}));
```

This is currently left up to the user to guard against because it's too difficult and expensive to for the library to check.

React tried to get this before (see [Prior Art → Shallow Equal]).

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

## Solution

### Compare

A function to deeply compare values.

```ts
function compare(
  expected: any,
  actual: any,
  options: CompareOptions,
): true | undefined | Deviations;
```

### CompareOptions

```ts
type CompareOptions = {
  mode:
    | 'all'
    | 'fast' // default
    | 'first'
  ,
  prototypes: Boolean, // default: `false`
};
```

<dl>
  <dt><em>mode</em></dt>
  <dd>How the comparison reports the result</dd>

  <dt><em>mode</em> <strong aria-label="default value"><code>fast</code></strong><dt>
  <dd>Return <code>true</code> when deviation(s) exist or <code>undefined</code> when no deviation(s) exist.</dd>

  <dt><em>mode</em> <code>first</code><dt>
  <dd>Return <code>Deviations</code> with only the first divation.</dd>

  <dt><em>mode</em> <code>full</code><dt>
  <dd>Return an <code>Iterator</code> of <code>Deviations</code> with all deviations (or an empty <code>Iterator</code> when no deviations exist).</dd>

  <dt><em>prototypes</em></dt>
  <dd>Whether to consider prototype when determining differences.</dd>

  <dt><em>prototypes</em> <strong aria-label="default value"><code>false</code></strong><dt>
  <dd>Do not compare prototypes.</dd>

  <dt><em>prototypes</em> <code>true</code><dt>
  <dd>Do compare prototypes.</dd>
</dl>

### Deviations

```ts
type Deviations = Iterable<
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
      enumerability: boolean,
      equality: boolean,
      missing: boolean,
      prototype: boolean,
      type: boolean,
    },
  },
>;
```

An `Interator` of deviation information:

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
  </dd>
</dl>

### Equality

Leafs are compared with [SameValueZero](https://tc39.es/ecma262/multipage/abstract-operations.html#sec-samevaluezero).

* A box primitive (eg `new Boolean(true)`) equals its primitive (eg `true`), except when `CompareOptions.reasons.prototype` is enable.
* `NaN` equals `NaN` (for performance and sanity).
* Zero (`0`, `-0`, `+0`) equals zero (for now? possibly an option in `CompareOptions` in future).

Custom types are handled by HostTypes (to avoid custom comparison).

### Examples

#### Fast equal
```js
compare('a', 'a');

undefined
```

#### Fast unequal
```js
compare('a', 'b');

true
```

#### First unequal with reason
```js
compare('a', 'b', {
  mode: 'first',
  reason: true,
});

Iterator => Iterable(1) {
  "" => {
    expected: 'a',
    actual: 'b',
    reason: { equality: true, … },
  },
}
```

#### First loosely unequal
```js
compare('1', 1, {
  mode: 'first',
});

Iterator => Iterable(1) {
  "" => {
    expected: '1',
    actual: 1,
    reason: { type: true, … },
  },
}
```

#### First unequal (nested)
```js
compare(
  { foo: 'a', bar: 'c' },
  { foo: 'b', bar: 'd' },
  {
    mode: 'first',
  },
);

Iterator => Iterable(1) { // mode: first
  "foo" => {
    expected: 'a',
    actual: 'b',
    reason: { equality: true, … },
  },
}
```

#### Object descriptor vs literal
```js
compare(
  Object.create({}, { foo: { enumerable: true, value: 'a' } }),
  { foo: 'a' },
);

false
```

#### Getter vs literal
```js
compare(
  Object.create({}, { foo: { enumerable: true, get: () => 'a' } }),
  { foo: 'a' },
);

false
```

#### Non-enumerable value
```js
compare(
  Object.create({}, { foo: { enumerable: false, value: 'a' } }),
  { foo: 'a' },
  {
    mode: 'first',
  },
);

Iterator => Iterable(1) {
  "foo" => {
    expected: undefined,
    actual: 'a',
    reason: { enumerable: true, … },
  },
}
```

#### Non-enumerable getter
```js
compare(
  Object.create({}, { foo: { get() { return 'a' } } }),
  { foo: 'a' },
  {
    mode: 'first',
  },
);

Iterator => Iterable(1) {
  "foo" => {
    expected: undefined,
    actual: 'a',
    reason: { enumerable: true, … },
  },
}
```

#### Multiple (unequal and type-mismatch red-herring)
```js
compare(
  { foo: 'a', bar: 'c' },
  { foo: 'b', bar:  2  },
  {
    mode: 'all',
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

#### Multiple (unequal and missing)
```js
compare(
  { foo: { bar: 'a'           } },
  { foo: { bar: 'b', qux: 'c' } },
  {
    mode: 'all',
  },
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

#### Multiple (unequal and prototype)
```js
compare(
  { foo: 'a', __proto__: null },
  { foo: 'b' },
  {
    mode: 'all',
    prototypes: true,
  },
);

Iterator => Iterable(2) {
  "[[Prototype]]" => {
    expected: null,
    actual: Object,
    reason: { instance: true, … },
  },
  "foo" => {
    expected: 'a',
    actual: 'b',
    reason: { equality: true, … },
  },
}
```

#### Multiple array items (unequal and missing)
```js
compare(
  ['a', 'b', 'c'     ],
  ['a', 'b', 'd', 'e'],
  {
    mode: 'all',
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

## Terminology

Reference sample:
```js
{
  foo: {
    bar: 'a',
    qux: {
      zed: 1,
    },
  },
}
```

<dl>
  <dt><em>Branch</em></td>
  <dd>A path from root to tip (inclusive of leaf). <code>foo.bar</code> and <code>foo.qux.zed</code> in the reference sample are branches.</dd>

  <dt><em>Leaf(s)</em></dt>
  <dd>The end of a branch. <code>bar</code> and <code>zed</code> in the reference sample are leafs because their values do not continue the branch.</dd>

  <dt><em>Value</em></td>
  <dd>The value of a leaf. <code>'a'</code> and <code>1</code> in the reference sample are values.</dd>
</dl>

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
