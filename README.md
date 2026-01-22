# Composable Accessors via built-in decorators

## Status

Stage: **1** (as of the Jan 2026 plenary)

Author/champion: **Lea Verou** ([@leaverou](https://github.com/leaverou))

For history, see the [original proposal](https://github.com/LeaVerou/proposal-composable-value-accessors).


> [!IMPORTANT]
> This is in the process of being split from the [original proposal](https://github.com/LeaVerou/proposal-composable-value-accessors).
> It is currently a work in progress. Move along, nothing to see here.

## Motivation

There are large classes of accessor use cases with strong commonalities and they deserve better DX and tooling support.

Some examples include:
- Lazy evaluation: Defer expensive computations until a property is accessed and then cache them
- Data validation: Reject certain writes (loudly or silently)
- Data normalization: Accept multiple formats but only store a canonicalized version
- Side effects before or after writes

This proposal aims to explore which of these may have good Impact/Effort to expose as built-in [decorators](https://github.com/tc39/proposal-decorators).

### Why built-in decorators instead of syntax?

The [original proposal](https://github.com/LeaVerou/proposal-composable-value-accessors) for composable accessors defined descriptor keys like `validate`, `transform` etc.
However, decorators have several benefits over syntax:
- They are a much smaller delta, increasing impact/effort ratio
- Runtime semantics can be polyfilled, syntax needs to be transpiled
- Can extend to non-property values where this makes sense. For example, there are potential extensions to support decorators on [function parameters](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md#parameter-decorators-and-annotations), [`let` variables](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md#let-decorators), and [`const` variables](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md#const-decorators). Most of these decorators could be immensely useful there too.

Using decorators does come with some disadvantages, but they can all be mitigated:
| Problem | Description | Mitigation |
|---------|-------------|------------|
| Readability | Poor readability when passing large arguments, putting potentially several lines of auxiliary information before the core ones (property name and initial value). | Use references for longer arguments. |
| Lossiness | Decorators modify accessor functions by wrapping them, which obliterates the original setter. | Decorator implementation can be designed to preserve the original setter on a property. There may even be space for a generic `Function` feature that does this. |
| Imperative API | Decorators cannot be applied imperatively. | Some use cases can be mitigated by stubbing the decorator function or using [object literals](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md#object-literal-and-property-decorators-and-annotations) and copying out their descriptors. |

### Why not just rely on userland decorators?

- Because the use cases are common enough they should be doable out of the box
- Because tooling cannot extract meaning from userland decorators
- Because built-in decorators are guaranteed to work for all relevant kinds

## Example

This proposal is meant to be combined with the [grouped and auto-accessors proposal](https://github.com/tc39/proposal-grouped-and-auto-accessors) and the [alias accessors proposal](https://github.com/tc39/proposal-alias-accessors) for improved ergonomics.

```ts
import isNumeric from "./validators.js";
const { validate, lazy } = Decorators;

class C {
	@validate(isNumeric, function(v) { return v <= this.max; })
	accessor min = 0;

	@validate(isNumeric, function(v) { return v >= this.min; })
	accessor max = 100;

	@lazy accessor foo = function() {
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

- [`@validate`](validate.md) - Silently or loudly reject writes that do not fulfill certain criteria
- [`@lazy`](lazy.md) - Defer expensive computations until the property is accessed and then cache them
- [`@normalize`](normalize.md) - Normalize values to a canonical format before writes
- [`@willSet` and `@didSet`](side-effects.md) - Run code before or after writes

## Practical examples

For examples of how individual decorators can be used, see their pages.
Here, we include a few practical examples that pull multiple together.

### Tiny signals

Here is an example from [tiny-signals](https://github.com/jsebrech/tiny-signals/blob/main/signals.js):

Original syntax:
```js
export class Signal extends EventTarget {
    #value;
    get value () { return this.#value; }
    set value (value) {
        if (equals(this.#value, value)) return;
        this.#value = value;
        this.dispatchEvent(new CustomEvent('change'));
    }

    // elided
}
```

With decorators:
```js
export class Signal extends EventTarget {

	@validate(equals)
    @willSet(console.log.bind(console, 'before'))
    @didSet(console.log.bind(console, 'after'))
    alias value to #value;
}
```
