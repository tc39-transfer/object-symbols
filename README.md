# ECMAScript Proposal: `Object.getOwnPropertySymbols( value, options? )`

## Status

Champion: Ruben Bridgewater, Jordan Harband

Author: Ruben Bridgewater <ruben@bridgewater.de>

Stage: 1

## Overview

This proposal extends `Object.getOwnPropertySymbols` with an *options object* that enables callers to filter the result list by **enumerability**.

```js
const allSymbols           = Object.getOwnPropertySymbols(obj); // Current behavior (all)
const allSymbolsTwo        = Object.getOwnPropertySymbols(obj, { enumerable: 'all' }); // Current behavior (all)
const enumerableSymbols    = Object.getOwnPropertySymbols(obj, { enumerable: true });
const nonEnumerableSymbols = Object.getOwnPropertySymbols(obj, { enumerable: false });
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

### Add an optional second parameter

```
Object.getOwnPropertySymbols ( O [ , options ] )
```

* **`O`** – the object whose own symbol properties are to be returned.
* **`options`** – optional object with a single property:
  * **`enumerable`**: `boolean | "all"` (default `"all"`)
    * `true`  → return only **enumerable** own symbol properties
    * `false` → return only **non‑enumerable** own symbol properties
    * `"all"` → return **all** own symbol properties (current behavior)
If `options` is `undefined` or `null`, behavior is equivalent to omitting the parameter (`"all"`).

### Abstract Algorithm Changes (Wording Sketch)

1. Let *obj* be ? `ToObject(O)`.
1. Let *keys* be **?** `obj.[[OwnPropertyKeys]]()`.
1. Let *symbols* be a new empty List.
1. For each *key* of *keys*, do
   1. If *key* is a Symbol, then
      1. If *filter* passes for *key*, append **key** to *symbols*.
1. Return CreateArrayFromList(*symbols*).

*Filter Determination*:

* Let *mode* be the `enumerable` option value if provided, else `"all"`.
* If *mode* is `"all"`, *filter* always passes.
* Else retrieve *desc* ← ? `obj.[[GetOwnProperty]](key)`.
  * If *mode* is `true`, *filter* passes iff *desc.\[\[Enumerable]]* is **true**.
  * If *mode* is `false`, *filter* passes iff *desc.\[\[Enumerable]]* is **false**.

### Invariants

* Order of returned symbols **must remain** the same relative order as in `[[OwnPropertyKeys]]`.
* No new observable operations other than the necessary `[[GetOwnProperty]]` look‑ups when `mode ≠ "all"`.
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

Object.getOwnPropertySymbols(obj, { enumerable: true });
// [visible]

Object.getOwnPropertySymbols(obj, { enumerable: false });
// [hidden]
```

## Polyfill

```js
const original = Object.getOwnPropertySymbols;
Object.getOwnPropertySymbols ??= function getOwnPropertySymbols(O, options) {
  const mode = options && Object.hasOwn(options, "enumerable")
    ? options.enumerable
    : "all";

  const symbols = original(O);
  if (mode === "all") { return symbols; }

  return symbols.filter(symbol => {
    const { enumerable } = Object.getOwnPropertyDescriptor(object, symbol);
    return mode ? enumerable : !enumerable;
  });
};
```

## Backwards Compatibility and Risk Assessment

* **No breaking changes**: existing two‑argument usages are non‑existent (today, the function has arity 1).
* Behavior when `options.enumerable === "all"` exactly matches current semantics.
* Engines may optimize the enumeration filter internally, avoiding the extra descriptor allocations present in userland polyfills.

## Security & Privacy Considerations

This proposal does **not** expose new data; it merely refines selection of already‑accessible property keys. Side‑effects via accessors are unchanged from the status quo.

## Implementation Experience

A canonical polyfill (above) demonstrates feasibility; prototypes in V8 & SpiderMonkey are straightforward, as both engines filter keys during `[[OwnPropertyKeys]]` enumeration.

## Ecosystem Impact

Libraries such as **lodash**, **Angular**, **React**, **Node.js**, **Next.js**, **TypeScript**, and many more implement filtering for symbol enumerability today. Native support will simplify their code and yield performance gains.

### React (filters for only enumerable symbols)

https://github.com/facebook/react/blob/336614679600af371b06371c0fbdd31fd9838231/packages/react-devtools-shared/src/utils.js#L109-L119

### Node.js (filters for only enumerable symbols)

- console.log() / util.inspect()
- assert.deepStrictEqual()

## Remaining Open Questions

1. Should `enumerable: "all"` be permitted explicitly, or treated as default only?
2. Should we expose additional filters (e.g., configurable, writable) in the same options object? – Out of scope for MVP.

## References & Prior Work

`Object.propertyCount()`. It is related in a way that it applies similar performance optimizations.
