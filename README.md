# 🐈
> ↑ That's a literal cat. ↓ These are __Literal Constraints as Types__.

## Summary

TypeScript is based on [structural subtyping][type-compatibility], but [nominal types][nominal-typing] are useful for expressing distinct concepts. Composing interfaces is a proven strategy. *Therefore...* Compose tags by constraint instead of defining a single tag per nominal concept.

```ts
type Temperature = number & Unit<'celsius'>
type Age = number & Integer & Nonnegative & Unit<'year'>
```

Compatibility between contexts doesn't require negotiating the concepts (`Temperature` and `Age`), but only their constraints (`Integer`, `Nonnegative`, `Unit`). If we establish conventions for constraints, there is no need to import anything.

```ts
// Constraints on the `number` literal type
type Integer = { Integer: true }
type Nonnegative = { Nonnegative: true }
type Positive = { Positive: true } & Nonnegative // express equvivalence
type Natural = Integer & Nonnegative // define aliases for convenience
type Unit<U extends Units> = { Unit: U } // e.g. some subset of CLDR units
// ...
```

Generally, **concepts are context-specific** — **constraints are universal**. Interfaces have proven useful for objects. Why not try a similar approach for literals?

## Example usage

> **Note**
> This repository isn't a library. I'm advocating for the *idea* of literal constraints as types. Optimally, the approach introduces no dependencies and low overhead.

### Constructing a constrained literal

It's necessary to parse input at the system boundary (e.g. public API). The boundary is a great place for introducing constructors for your local *concepts*. While it is possible to create composable constructors for each *constraint*, that's likely unnecessarily complex. The goal is to **attach the constraints that your concept conforms to as type-level tags**.

- Given a concept...
    ```ts
    type Age = number & Natural & Unit<'year'>
    ```
- Known types (e.g. constants) can be asserted
    ```ts
    const someAge = 42 as Age
    ```
- Values can be narrowed from the base literal (i.e. `number`) or `unknown` with a type guard
    ```ts
    function narrow(input: unknown): input is Age {
      return typeof input === 'number' && Number.isInteger(input) && input >= 0
    }
    ```
- The narrowing function can be used to implement a constructor
    ```ts
    function from(input: unknown): Age { // signature dependes on error handling
      if (narrow(input)) return input
      // throw Error, return Error, return undefined, use Result type, ...
    }
    ```
  - If you're using a parsing library like [Zod][zod], it should be possible to attach tags to a schema. In Zod, one approach is using `refine`. (No need to have a type guard with extraneous checks, though!) Note that the built-in `brand` method uses a `Symbol`, which complicates compatibility. It aims to brand *concepts*, not *constraints*. (That's not necessarily a bad thing! Nominal types and literal constraints as types are different approaches.)

### Accepting a constrained literal as input

The advantage of constraints as types is evident when defining context-independent functions. **Making constraints explicit on the type-level** avoids a situation where you have to choose to either...

- support the base literal (e.g. `number`) via defensive programming, or to
- negotiate the nominal concept (e.g. `AgeInSeconds`) via shared dependencies.

For example, the following function has type-level guarantees that the three `number` literals passed as arguments represent compatible concepts for calculating linear expansion. (Note that `coefficient` uses non-CLDR units. A useful library would likely expose a union of materials with known coefficients instead.)

```ts
function calculateLinearExpansion<Length extends number>(
  initialLength: Length,
  coefficient: number & (Unit<'per-degree-celsius'> | Unit<'per-kelvin'>),
  temperatureChange: number & (Unit<'celsius'> | Unit<'kelvin'>)
): Length {
  return (initialLength * coefficient * temperatureChange) as Length
}
```

[type-compatibility]: https://www.typescriptlang.org/docs/handbook/type-compatibility.html
[nominal-typing]: https://basarat.gitbook.io/typescript/main-1/nominaltyping
[zod]: https://zod.dev/