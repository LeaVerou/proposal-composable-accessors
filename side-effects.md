# Built-in decorators for side effects

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

Another option could be `@willChange` and `@changed` decorators that only run when the value has actually changed or is about to change.

```js
class C {
	@changed(function() {
		this.dispatchEvent(new CustomEvent('change'))
	})
	accessor value;
}
```

Applies to the following kinds of decorators:
- `accessor`
- `setter`

Later, it can also apply to these decorator [extensions](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md):
- `let`
- `property`

Using it on any other kind of decorator will throw a `TypeError`.

## Practical example


Here is an example from [tiny-signals](https://github.com/jsebrech/tiny-signals/blob/main/signals.js):

Original syntax:
```js
export class Signal extends EventTarget {
    #value;
    get value () {
		return this.#value;
	}
	set value (value) {
		if (this.#value === value) {
			return;
		}

        this.#value = value;

        this.dispatchEvent(new CustomEvent('change'));
    }

    // ...
}
```

With a [`@changed` decorators](decorators/side-effects.md#changed) and [alias accessors](https://github.com/tc39/proposal-alias-accessors) this could be written as:
```js
export class Signal extends EventTarget {
	@changed(function(value, oldValue) {
		this.dispatchEvent(new CustomEvent('change', { detail: { value, oldValue } }));
	})
    alias value to #value;
}
```


## Implementation sketch

These are meant to be minimal and illustrative, not production-ready.

### `@willSet`

```js
function willSet (fn) {
	if (typeof fn !== "function") {
		throw new TypeError(`@willSet parameter must be a function`);
	}

	return function(value, { kind, name }) {
		if (kind === "accessor" || kind === "setter") {
			let setter = kind === "accessor" ? value.set : value;
			let set = function(newValue) {
				fn.call(this, newValue);
				return setter.call(this, newValue);
			};
			return kind === "accessor" ? Object.assign(value, { set }) : set;
		}

		throw new TypeError(`@willSet cannot be used on ${kind} constructs`);
	};
}
```

### `@didSet`

> [!NOTE]
> Using a Symbol property to store the old value for readability.
> In practice, the decorators would not add any observable properties to the object.

```js
function didSet (fn) {
	if (typeof fn !== "function") {
		throw new TypeError(`@didSet parameter must be a function`);
	}

	return function(value, { kind, name }) {
		if (kind === "accessor" || kind === "setter") {
			let setter = kind === "accessor" ? value.set : value;
			let set = function(v) {
				set.call(this, v);
				fn.call(this, v);
				return result;
			};
			return kind === "accessor" ? Object.assign(value, { set }) : set;
		}

		throw new TypeError(`@didSet cannot be used on ${kind} constructs`);
	}
};
```

### `@willChange`

> [!NOTE]
> Using a Symbol property to store the old value for readability.
> In practice, the decorators would not add any observable properties to the object.

```js
function willChange (fn, equals = (a, b) => a === b) {
	if (typeof fn !== "function" || typeof equals !== "function") {
		throw new TypeError(`@changed parameters must be functions`);
	}

	return function(value, { kind, name }) {
		if (kind === "accessor" || kind === "setter") {
			let setter = kind === "accessor" ? value.set : value;
			let oldValue = new Symbol(name);
			let set = function(v) {
				if (equals.call(this, this[oldValue], v)) {
					return;
				}

				fn.call(this, v, this[oldValue]);
				set.call(this, v);
				this[oldValue] = v;
				return;
			};

			if (kind === "accessor") {
				const init = function (v) { return this[oldValue] = v };
				return Object.assign(value, { set, init });
			}

			return set;
		}

		throw new TypeError(`@changed cannot be used on ${kind} constructs`);
	}
};
```

### `@changed`

```js
function changed (fn, equals = (a, b) => a === b) {
	if (typeof fn !== "function" || typeof equals !== "function") {
		throw new TypeError(`@changed parameters must be functions`);
	}

	return function(value, { kind, name }) {
		if (kind === "accessor" || kind === "setter") {
			let setter = kind === "accessor" ? value.set : value;
			let oldValue = new Symbol(name);
			let set = function(v) {
				if (equals.call(this, this[oldValue], v)) {
					return;
				}

				set.call(this, v);
				fn.call(this, v, this[oldValue]);
				this[oldValue] = v;
				return;
			};

			if (kind === "accessor") {
				const init = function (v) { return this[oldValue] = v };
				return Object.assign(value, { set, init });
			}

			return set;
		}

		throw new TypeError(`@changed cannot be used on ${kind} constructs`);
	}
};
```
