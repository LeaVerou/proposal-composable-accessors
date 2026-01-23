# Composable Accessors via built-in decorators

## Status

Stage: **1** (as of the Jan 2026 plenary)

Author/champion: **Lea Verou** ([@leaverou](https://github.com/leaverou))

For history, see the [original proposal](https://github.com/LeaVerou/proposal-composable-value-accessors).

## Contents

1. [Status](#status)
2. [Motivation](#motivation)
	1. [Why built-in decorators instead of syntax?](#why-built-in-decorators-instead-of-syntax)
	2. [Why not just rely on userland decorators?](#why-not-just-rely-on-userland-decorators)
3. [Example](#example)
4. [Index of current ideas](#index-of-current-ideas)


## Motivation

There are large classes of accessor use cases with strong commonalities and they deserve better DX and tooling support.

Most of these accessors are **_additive_ or _composable_**:
Rather than the mental model of replacing a property with arbitrary code that regular accessors are designed around,
composable accessors are **value-backed**: as a baseline they proxy another property (stored in an internal slot by default, or provided by another property), and any validation logic, transformations, side effects, etc. are **layered over that baseline**.

The underlying value-backed plumbing is provided by [auto-accessors](https://github.com/tc39/proposal-grouped-and-auto-accessors) or [alias accessors](https://github.com/tc39/proposal-alias-accessors).
Then, the additional functionality is layered over that baseline through built-in decorators, which is the subject of this proposal.

Some examples include:
- **Lazy evaluation:** Defer expensive computations until a property is accessed and then cache them
- **Data validation:** Reject certain writes (loudly or silently)
- **Data normalization:** Accept multiple formats but only store a canonicalized version
- **Side effects:** Run code before or after writes

This proposal aims to explore which of these may have good Impact/Effort to expose as built-in [decorators](https://github.com/tc39/proposal-decorators).

### Why built-in decorators instead of syntax?

The [original proposal](https://github.com/LeaVerou/proposal-composable-value-accessors) for composable accessors defined descriptor keys like `validate`, `transform` etc.
However, decorators have several benefits over syntax:
- They are a much smaller delta, increasing impact/effort ratio
- Runtime semantics can be polyfilled, syntax needs to be transpiled
- Can extend to non-property values where this makes sense. For example, there are potential extensions to support decorators on [function parameters](https://github.com/tc39/proposal-blob/master/EXTENSIONS.md#parameter-decorators-and-annotations), [`let` variables](https://github.com/tc39/proposal-blob/master/EXTENSIONS.md#let-decorators), and [`const` variables](https://github.com/tc39/proposal-blob/master/EXTENSIONS.md#const-decorators). Most of these decorators could be immensely useful there too.

Using decorators does come with some disadvantages, but they can all be mitigated:
| Problem | Description | Mitigation |
|---------|-------------|------------|
| Readability | Poor readability when passing large arguments, putting potentially several lines of auxiliary information before the core ones (property name and initial value). | Use references for longer arguments. |
| Lossiness | Decorators modify accessor functions by wrapping them, which obliterates the original setter. | Decorator implementation can be designed to preserve the original setter on a property. There may even be space for a generic `Function` feature that does this. |
| Imperative API | Decorators cannot be applied imperatively. | Many use cases can be mitigated by stubbing the decorator function and applying the result to the property descriptor. |

### Why not just rely on userland decorators?

- Because the use cases are common enough they should be doable out of the box
- Because tooling cannot extract meaning from userland decorators
- Because built-in decorators are guaranteed to work for all relevant kinds

## Example

This proposal is meant to be combined with the [grouped and auto-accessors proposal](https://github.com/tc39/proposal-grouped-and-auto-accessors) and the [alias accessors proposal](https://github.com/tc39/proposal-alias-accessors) for improved ergonomics.

```ts
import isNumeric from "./validators.js";
const { validate, memoized } = Decorators;

class C {
	@validate(isNumeric)
	accessor min = 0;

	@validate(function(v) { return v >= this.min; })
	accessor max = 100;

	@memoized get foo () {
		return expensiveComputation();
	}
}
```

In addition to the details of the individual decorators, another question is what should the namespace for these decorators be?
The example above uses a new `Decorators` global object. Perhaps it could be a sub-object of that, e.g. `Decorators.known`? Or something entirely different?

Note that because decorators are just variables, they can also be aliased to different names.
For example:

```js
import { array } from "./transformers.js";
const { normalize: to } = Decorators;

class C {
	@to(array) accessor options;
}
```

## Index of current ideas

> [!IMPORTANT]
> This is an early exploration into what types of built-in decorators might be useful.
> As such, it errs on the side of breadth, but it is expected to be narrowed down and refined over time.
> Sample code is illustrative and not meant to be production-ready or exhaustively tested.

- [`@memoized`](memoized.md) - Cache the result of expensive computations. Also useful for implementing lazy evaluation (since decorators cannot convert to a value property).
- [`@validate`](validate.md) - Silently or loudly reject writes that do not fulfill certain criteria
- [`@normalize`](normalize.md) - Normalize values to a canonical format before writes
- [Side effects: `@willSet`, `@didSet`,  `@willChange`, `@changed`](side-effects.md) - Run code before or after writes

