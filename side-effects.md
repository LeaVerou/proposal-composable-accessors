# Built-in decorators for side effects (`@willSet`, `@didSet`)

The `@willSet` and `@didSet` decorators run code before and after the value is set, respectively.
The names are inspired by [Swift property observers](https://docs.swift.org/swift-book/documentation/swift/propertyobserver).

```js
class C {
	@willSet(console.log.bind(console, 'before'))
	accessor foo;
	@didSet(console.log.bind(console, 'after'))
	accessor bar;
}
```

Applies to the following kinds of decorators:
- `accessor`
- `setter`

and these decorator extensions:
- `let`
- `property`

Using it on any other kind of decorator will throw a `TypeError`.

## Practical example

## Implementation sketch

```js
function willSet (fn) {
	return function(value, { kind, name }) {
		if (kind === "accessor") {
			let { set } = value;
			value.set = function(newValue) {
				fn.call(this, newValue);
				return set.call(this, newValue);
			};
			return value;
		}

		if (kind === "setter") {
			return function(newValue) {
				fn.call(this, newValue);
				return value.call(this, newValue);
			};
		}

		throw new TypeError(`@willSet cannot be used on ${kind} constructs`);
	};
}

function didSet (fn) {
	return function(value, { kind, name }) {
		if (kind === "accessor") {
			let { set } = value;
			value.set = function(newValue) {
				return set.call(this, newValue);
			};
		}
	}
};
```
