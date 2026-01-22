# `@memoized` built-in decorator

The `@memoized` decorator memoizes (caches) the result of a function or getter and returns the cached result on subsequent accesses.
This can be useful both for lazy evaluation and memoization of expensive computations.

```js
class C {
	@memoized accessor foo {
		get () {
			return expensiveComputation();
		}
	}

	@memoized get bar () {
		return expensiveComputation2();
	}

	// Memoized results are keyed by the arguments
	@memoized baz (a, b, c) {
		return expensiveComputation3(a, b, c);
	}
}
```

It applies to the following kinds of decorators:
- `accessor`
- `get`
- `method`
- `class`

Later, it can also apply to these decorator [extensions](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md):
- `let`
- `property`
- `function`

Using it on any other kind of decorator will throw a `TypeError`.

## Why not `@lazy`?

Note that decorators cannot be used to implement the common pattern of replacing a getter with a value property when accessed, so this seems to be the only path forwards for lazy evaluation.

<!--
However, even if it *were* possible to convert an accessor to a value property with decorators, this approach has the advantage that it works properly in deeply nested hierarchies of objects.
To illustrate, suppose we could do:

```js
class A {
	@lazy get foo () {
		console.log("Expensive computation");
		return expensiveComputation();
	}
}
```

And have it be replaced with a regular value property when accessed:

```js
let a = new A();
// logs both "Expensive computation" and the cached value
console.log(a.foo);
// Just logs the cached value
console(a.foo);
```

Now suppose we have subclasses:
```js
class B extends A {

}

class C extends B {

}
``` -->

## Implementation sketch

This is meant to be informative, and not a complete polyfill.
In this implementation, memoized constructors or methods depend on the [composites proposal](https://github.com/tc39/proposal-composites) for convenience and simplicity.

> [!NOTE]
> Using a Symbol property to store the old value for readability.
> In practice, the decorators would not add any observable properties to the object.

```js
const fnCache = new Map();

function memoized (value, { kind, name }) {
	if (kind === "accessor" || kind === "get") {
		let getter = kind === "accessor" ? value.get : value;
		let key = new Symbol(name);
		let get = function() {
			// Or should it be Object.hasOwn(this, key) ?
			if (key in this) {
				// Already computed
				return this[key];
			}
			// Compute and cache
			return this[key] = getter.call(this);
		}
		return kind === "accessor" ? Object.assign(value, { get }) : get;
	}

	if (kind === "method") {
		// Memoize keyed by arguments
		let fn = value;
		return function(...args) {
			let key = Composite([fn, ...args]);
			if (fnCache.has(key)) {
				return fnCache.get(key);
			}
			let result = fn.apply(this, args);
			fnCache.set(key, result);
			return result;
		}
	}

	if (kind === "class") {
		// Memoize keyed by arguments
		let fn = value;
		return class extends value {
			constructor(...args) {
				let key = Composite([fn, ...args]);
				if (fnCache.has(key)) {
					return fnCache.get(key);
				}

				super(...args);

				fnCache.set(key, this);
			}
		}
	}

	throw new TypeError(`@memoized cannot be used on ${kind} constructs`);
}
```
