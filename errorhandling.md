# Error Handling

This proposal handles failure the way JavaScript already does — with exceptions — and adds one thing: `catch` clauses can be typed, so a handler matches by error type instead of testing `instanceof` by hand. This document defines typed catch, describes the errors a typed program raises where the language inserts a check, and closes with the deliberate choice to keep exceptions rather than adopt a `Result` type.

## Typed catch clauses

A `catch` clause may name the error type it handles. Clauses are tried in order, and the first whose type the thrown value satisfies runs; an untyped clause at the end catches whatever remains. An untyped clause may only be the last: a typed clause after it could never run, so a `catch` that is not last must name a type.

```js
try {
  // Statement that throws
} catch (e: TypeError) {
  // Statements to handle TypeError exceptions
} catch (e: RangeError) {
  // Statements to handle RangeError exceptions
} catch (e: EvalError) {
  // Statements to handle EvalError exceptions
} catch (e) {
  // Statements to handle any unspecified exceptions
}
```

Within a typed clause the binding is narrowed to that type, so `e.message` and any properties specific to the error type are available without a cast. A value not matched by any typed clause propagates to the enclosing handler, exactly as an unhandled type would; typed clauses filter, they do not swallow. This is the same narrowing the `is` operator and `instanceof` provide, applied to the catch binding.

## The errors a typed program raises

Most of the errors a program catches are the ones it throws itself. But this proposal inserts checks in a few places, and each raises a specific built-in error, so a handler can catch them by type rather than by string-matching a message:

- A **checked numeric conversion** out of range is a `RangeError`. `uint8(300)`, a `decimal128` overflow, and a `rational.<64>` whose reduced form no longer fits are all `RangeError`, because a value's range is a property of its type.
- An **array access out of bounds** is a `RangeError`, from the bounds checks the array sections describe.
- A **failed conversion at a dynamic boundary** — assigning an `any` or an untyped value into a typed binding whose runtime value doesn't match — is a `TypeError`. This is the check that makes `any` safe: the mismatch surfaces at the boundary rather than corrupting typed code downstream.
- **Division by zero** in an integer, `decimal`, or `rational` type is a `RangeError`, since these types do not produce `Infinity` the way binary floats do.
- A **failed `parse`** — `uint8.parse('nope')` — throws rather than returning a hidden `NaN`.

These are the standard `RangeError` and `TypeError`, so existing handlers catch them, and a typed `catch (e: RangeError)` handles every range failure above in one clause.

## Custom errors

`Error` and its subclasses are reference types. A program defines its own error by extending `Error`, and a typed `catch` then matches it like any built-in:

```js
class ValidationError extends Error {
  field: string;
  constructor(field: string, message: string) {
    super(message);
    this.field = field;
  }
}

try {
  validate(input);
} catch (e: ValidationError) {
  report(e.field, e.message); // e is narrowed to ValidationError
} catch (e: RangeError) {
  // a different failure
}
```

Errors are *unchecked*, as in JavaScript: a function does not declare the exceptions it may throw, and there is no `throws` clause. The type system narrows a caught error but does not track which errors a call site might raise; that is the same contract JavaScript, Python, and C# offer, and it keeps typed code interoperable with untyped code that throws freely.

## Errors across async boundaries

A rejected `Promise.<T>` carries its rejection reason, and `await` propagates it as a throw, so a typed `catch` around an `await` handles asynchronous failure the same way as synchronous:

```js
async function load(url: string): Promise.<Config, Error> {
  try {
    return await fetchConfig(url);
  } catch (e: TypeError) {
    return defaultConfig; // e.g. a network TypeError
  }
}
```

The rejection reason is typed as broadly as any thrown value, since a promise may be rejected with anything; the typed clause narrows it just as in a synchronous `try`.

## Exceptions, not `Result`

Rust models recoverable failure with `Result<T, E>` and the `?` operator, making an error a value of a type the caller must handle. This proposal keeps exceptions, and the choice is deliberate. JavaScript's entire runtime and standard library throw — every existing API, every host call, every `JSON.parse` — so a `Result`-based channel would fork error handling in two, and typed code calling untyped code would have to translate at every boundary. Exceptions compose with what is already there.

The type system still narrows the failure space where it helps. A `T | null` union models `Option<T>` for the common absent-value case, with `?.` and narrowing to unwrap it; a thrown, typed error models the `Err` case with `catch` to match it. A program that genuinely wants error-as-value can build it — a `sealed abstract class Result` with `Ok` and `Err` subclasses is an exhaustively-matchable sum type — but the language's native failure channel is the exception, because that is the channel the platform already speaks.
