# ECMAScript Proposal: `Object.symbols( value )`

## Status

Champion: Ruben Bridgewater, Jordan Harband

Author: Ruben Bridgewater <ruben@bridgewater.de>

Stage: 1

## Overview

This proposal aligns with `Object.keys` to enable callers only get **enumerable** symbols, similar to Object.keys only returning enumerable string properties.

Most symbol retrieval is in fact filtering out non-enumerable symbols.

```js
const enumerableSymbols    = Object.symbols(obj);
const allSymbols           = Object.getOwnPropertySymbols(obj);
```

### Motivation

`Object.getOwnPropertySymbols` is invaluable for libraries that rely on symbol-keyed metadata. Frequently, however, a library needs **only the enumerable** (or **only the non‑enumerable**) symbols:

* **Serializers / logger utilities** wish to include only enumerable data.
* **Meta‑programming tools** often hide internals behind non‑enumerable symbols and need to avoid exposing them.

Today, developers must fetch **all** symbols and then filter manually:

```js
function enumerableSymbols(object) {
  return Object.getOwnPropertySymbols(object)
    .filter(symbol => Object.getOwnPropertyDescriptor(object, symbol)?.enumerable);
}
```

This pattern is error‑prone, incurs extra per‑property descriptor lookups, and cannot be optimized by engines. The proposed option provides an ergonomic, reliable, and performant solution.

## Detailed Design

### Add a new top level method to Object

```
Object.symbols ( O )
```

* **`O`** – the object whose own symbol properties are to be returned.

### Abstract Algorithm Changes (Wording Sketch)

1. Let *obj* be ? `ToObject(O)`.
2. Let *keys* be **?** `OrdinaryOwnPropertyKeys(obj)`.
3. Let *symbols* be a new empty List.
4. For each *key* of *keys*, do
   1. If *key* is a Symbol, then
      1. Let *desc* be ? `OrdinaryGetOwnProperty(obj, key)`.
      1. If *desc.\[\[Enumerable]]* is **true**, append *key* to *symbols*.
5. Return CreateArrayFromList(*symbols*).

### Invariants

* Order of returned symbols **must remain** the same relative order as in `[[OwnPropertyKeys]]`.
* No new observable operations other than the necessary `[[GetOwnProperty]]` look‑ups.
* Accessor side effects remain as in current spec (a property descriptor retrieval can invoke user code).

## Examples

```js
const hidden = Symbol("hidden");
const visible = Symbol("visible");

const obj = {};
Object.defineProperty(obj, hidden,  { enumerable: false, value: 1 });
Object.defineProperty(obj, visible, { enumerable: true,  value: 2 });

Object.getOwnPropertySymbols(obj);
Reflect.ownKeys(obj);
// [hidden, visible]

Object.symbols(obj);
// [visible]
```

## Polyfill

```js
const original = Object.getOwnPropertySymbols;
Object.getOwnPropertySymbols ??= function getOwnPropertySymbols(O) {
  const symbols = original(O);

  return symbols.filter(symbol => {
    return Object.getOwnPropertyDescriptor(object, symbol).enumerable;
  });
};
```

## Backwards Compatibility and Risk Assessment

* **No breaking changes**: existing two‑argument usages are non‑existent (today, the function has arity 1).
* Behavior when `options.enumerable === "all"` exactly matches current semantics.
* Engines may optimize the enumeration filter internally, avoiding the extra descriptor allocations present in userland polyfills.

## Implementation Experience

A canonical polyfill (above) demonstrates feasibility; prototypes in V8 & SpiderMonkey are straightforward, as both engines filter keys during `[[OwnPropertyKeys]]` enumeration.

## Ecosystem Impact

Libraries such as **lodash**, **Angular**, **React**, **Node.js**, **Next.js**, **TypeScript**, and many more implement filtering for symbol enumerability today. Native support will simplify their code and yield performance gains.

### React (filters for only enumerable symbols)

https://github.com/facebook/react/blob/336614679600af371b06371c0fbdd31fd9838231/packages/react-devtools-shared/src/utils.js#L109-L119

### Node.js (filters for only enumerable symbols)

- console.log() / util.inspect()
- assert.deepStrictEqual()
