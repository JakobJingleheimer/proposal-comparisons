# Proposal: Assertions

Champions:
* @JakobJingleheimer

Authors:
* @JakobJingleheimer
* @bridgeAR

## [Stage](https://tc39.github.io/process-document/)

**Current**: 0
<br />
**Requesting**: 1

## The Problem(s)

This proposal may eventually split into 2 different topics:

### A vs B

Determining whether B is sufficiently similar (or different) to A.

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

This may be addressed by the [Pattern Matching proposal](https://github.com/tc39/proposal-pattern-matching).

### Output

Even more important than A vs B is the output: Merely knowing A is unexpected is almost useless when you can't see what A and B are.

<details>
<summary>examples</summary>

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
</details>

There is potentially also an issue of representing multiple problems (such as is possible via Pattern Matching).

## Ecosystem today

The vast majority of ECMAScript engineers use one of 2 forms: `assert` and `expect`. These come from one of ~4 libraries: `chai` (`20M` weekly), `jasmine` (`1.4M` weekly), `jest` (`29M` weekly), `node:assert` (indeterminable). These are direct competitors, so we can assume there is no overlap and the numbers are summable: at least `~51M` weekly (probably significantly higher when `node:assert` numbers are added).

Expect:

* `jasmine` and `jest` are (nearly?) identical with dedicated methods: `expect(a).toEqual(b)`
* `chai`'s BDD set is a chain-style that builds upon itself: `expect(a).to.equal(b)`

Assert:

* `node:assert` and `chai`'s TDD set have large overlap.

### Usage outside of test suites

It is not uncommon to include assertions within code as a means of input checking (especially input coming from an end-user):

```js
function sum(limit, ...inputs) {
  let total = 0;
  for (const { valueAsNumber: val } of inputs) total += val;

  assert.ok(total <= limit);

  return total;
}

const expenses = sum(budget, ...form.elements.expenses);
```

These are then caught and surfaced to the user in a human-friendly message (such as via a "toast").

### TDLR

The functionality is **widely** used throughout the ecosystem with almost no variation. Users largely do not care about one verses the other—they care about:

* "behaves as expected"
* convenience
* how much they have to look up

The first depends on getting it right. We fortunately have decades of experience from the ecosystem to build upon.

The second two are addressed by nature of native inclusion:

* convenience: it's right there (can't get more convenient).
* how much to look up: when everyone is regularly using the same thing, it's top-of-mind so there's no lookup.

Will it be difficult: very.
<br />
Is it worth doing: yes.

### Neighbours

Many major languages natively include a form of assertion. To name a relevant few:

* [`c++`](https://en.cppreference.com/w/cpp/error/assert)
* [`go`](https://pkg.go.dev/github.com/stretchr/testify/assert)
* [`kotlin`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/assert.html)
* [`python`](https://docs.python.org/3/reference/simple_stmts.html#the-assert-statement)
* [`rust`](https://doc.rust-lang.org/std/macro.assert.html)

## Explicitly out of scope

* This is not a test runner (`describe`, `it`, etc).
* This is not a test utility suite (`mock`, `stub`, etc).

## Prior to stage 2

* Investigate (and potentially decide on) `assert` vs `expect` (preliminary investigation suggests `assert`).
  * Possibly an amalgamation of Pattern Matching.
* Consider extensibility (public symbols?)
  * Additional/replacement comparison algorithms
  * Additional/replacement output handlers
* Decide on a narrow initial scope (surface-area is enormous: assertions/expectations plus output).
