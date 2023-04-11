# 🐈
↑ That's a literal cat. ↓ These are __Literal Constraints as Types__.

> **Note**
> This is still an untested idea. I'm not sure if it's actually useful yet. Might be simply something that gets in the way, or is made irrelevant by more granular number types and first-class support for units of measure. I'll try this out in my own projects to see if it improves compatibility.

## Overview

TypeScript is based on [structural subtyping][type-compatibility], but [nominal types][nominal-typing] are useful for expressing distinct concepts. Composing interfaces is a proven strategy. *Therefore...* Make use of structural subtyping by **composing tags by constraint instead of defining a single nominal tag per concept**.

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

Generally, **concepts are context-specific** — **constraints are universal**.

## Example usage

> **Note**
> This repository isn't a library. I'm exploring the *idea* of literal constraints as types. Optimally, the approach introduces no dependencies and low overhead.

### Constructing a constrained literal

It's necessary to parse input at the system boundary (e.g. public API). The boundary is a sensible place for introducing constructors for your local concepts. The exact implementation is a separate concern, but for the purposes of constraints as types, what matters is ending up with *the proper type signature with minimal overhead*.

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
  - If you're using a parsing library like [Zod][zod], it should be possible to attach tags to a schema. In Zod, one approach is using `refine`. (No need to have a type guard with extraneous checks, though!)

### Accepting a constrained literal as input

Constraints as types aims to make it easier to define functions that are compatible with concepts from external contexts. (My `Temperature` may be different from your `Temperature`.) Making constraints explicit on the type-level *avoids* a situation where you have to choose to either...

- fall back to the base literal (e.g. `number`) via defensive programming, or to
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

> **Warning**
> Checking constraints after every single step is often unproductive. As a rule of thumb, aim to construct concepts once at the system boundary. This provides an unambiguous starting point for processing. There's diminishing returns for — say — ensuring that a `number` is `Positive` after every subtraction. However, **it's vital that the defined constraints are actually conformed to**. The whole approach falls apart if you can't trust the types. Focus on defining and enforcing only the constraints that are necessary.

[type-compatibility]: https://www.typescriptlang.org/docs/handbook/type-compatibility.html
[nominal-typing]: https://basarat.gitbook.io/typescript/main-1/nominaltyping
[zod]: https://zod.dev/
