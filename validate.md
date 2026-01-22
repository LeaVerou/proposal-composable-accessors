# `@validate` built-in decorator

The `@validate` decorator validates a value before it is set and silently rejects the write if the value is invalid.
The function provided can throw to reject the write loudly.

```js
const isPositive = n => n > 0;
class C {
	@validate(isPositive)
	accessor foo;
}
```

## Decorator kinds

It applies to the following kinds of decorators:
- `accessor`
- `setter`

Later, it can also apply to these decorator [extensions](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md):
- `let`
- `property`

Using it on any other kind of decorator will throw a `TypeError`.

## Other potential names

- `assert`
- `check`
- `verify`
- `constraints`

## Implementation sketch

```js
function validate (...validators) {
	if (validators.some(v => typeof v !== "function")) {
		throw new TypeError(`@validate arguments must be functions`);
	}

	const validator = function(value) {
		return validators.every(v => v.call(this, value));
	};

	return function (value, { kind, name }) {
		let set;
		const setter = function(newValue) {
			if (validator.call(this, newValue)) {
				return set.call(this, newValue);
			}
		};

		if (kind === "accessor") {
			set = value.set;
			value.set = setter;
			return value;
		}

		if (kind === "setter") {
			set = value;
			return setter;
		}

		throw new TypeError(`@validate cannot be used on ${kind} constructs`);
	};
};
```

## Alternatives / extensions

- `@validateAny()` for the OR version?
- `@validate.loud()` version that throws when value is invalid.
