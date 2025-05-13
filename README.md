# Proposal: Assertions

Champions:
* @JakobJingleheimer

Authors:
* @JakobJingleheimer
* @bridgeAR

## Stage

**Current**: 0
<br />
**Requesting**: 1

## Rationale

The vast majority of ECMAScript engineers use one of 2 forms: `assert` and `expect`. These come from one of ~4 libraries: `chai` (`20M` weekly), `jasmine` (`1.4M` weekly), `jest` (`29M` weekly), `node:test` (indeterminable). These are direct competitors, so we can assume there is no overlap and the numbers are summable: at least `~51M` weekly (probably significantly higher when `node:test` numbers are added).

Expect:

* `jamsine` and `est` are (nearly?) identical with dedicated methods: `expect(a).toEqual(b)`
* `chai`'s BDD set is a chain-style that builds upon itself: `expect(a).to.equal(b)`

Assert:

* `node:test` and `chai`'s TDD set have large overlap.

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

Many other major languages natively include a form of assertion. To name a relevant few:

* [`c++`](https://en.cppreference.com/w/cpp/error/assert)
* [`go`](https://pkg.go.dev/github.com/stretchr/testify/assert)
* [`kotlin`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/assert.html)
* [`python`](https://docs.python.org/3/reference/simple_stmts.html#the-assert-statement)
* [`rust`](https://doc.rust-lang.org/std/macro.assert.html)

## Explicitly out of scope

* This is not a test runner (`describe`, `it`, etc).
* This is not a test utility suite (`mock`, `stub`, etc).

## Prior to stage 2

* Investigate and decide on `assert` vs `expect` (preliminary investigation suggests `assert`).
* Consider extensibility: public symbols?
* Decide on a narrow initial scope (surface-area is enormous: assertions/expectations plus output).
