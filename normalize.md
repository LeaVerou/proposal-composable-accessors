# `@normalize` built-in decorator

The `@normalize` decorator transforms to a canonical format before it is persisted.

```js

const toArray = value =>
	Array.isArray(value) ? value :
	!value ? [] :
	value[Symbol.iterator] ? Array.from(value) :
	[value];

class C {
	@normalize(toArray) accessor options;
}
```

It applies to the following kinds of decorators:
- `accessor`
- `setter`
- `field`

and these decorator extensions:
- `parameter`
- `let`
- `const`

## Implementation sketch

```js
function normalize (normalizer) {
	if (typeof normalizer !== "function") {
		throw new TypeError(`@normalize parameter must be a function`);
	}

	return function(value, { kind, name }) {
		if (kind === "accessor") {
			let { set } = value;
			value.set = function (newValue) {
				return set.call(this, normalizer.call(this, newValue));
			};
			value.init = function (initialValue) {
				return normalizer.call(this, initialValue);
			};
			return value;
		}

		if (kind === "setter") {
			return function(newValue) {
				return value.call(this, normalizer.call(this, newValue));
			};
		}

		if (kind === "field") {
			return function init (initialValue) {
				return normalizer.call(this, initialValue);
			};
		}

		throw new TypeError(`@normalize cannot be used on ${kind} constructs`);
	};
}
```
