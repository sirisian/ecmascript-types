# Complex Numbers

A number with a real and an imaginary part, `a + bi`. Python has `complex` with a `j` suffix, Go `complex128` with an `i` suffix, C99 `_Complex`, Julia `im`, Fortran, MATLAB, and NumPy's `complex64`/`complex128`. Complex numbers are the working type of signal processing and the FFT, quantum computing, control theory, AC circuit analysis, polynomial roots, fractals, and 2D rotation. They are not a convenience wrapper over two floats: multiplication mixes the components, and that mixing is the point.

This proposal already lists `complex` among its primitive types. This document defines it: a value type with language-level operators, including a literal for the imaginary axis.

## Representation

`complex.<T>` is a value type holding two components of a real numeric type `T`, the real part and the imaginary part. The width-named shorthands count *total* bits, following the NumPy and Go convention, and the bare name uses `number`:

```js
type complex64 = complex.<float32>;   // two float32, eight bytes
type complex128 = complex.<float64>;  // two float64, sixteen bytes
type complex = complex.<number>;      // two number, sixteen bytes: the ergonomic default
```

Bare `complex` uses `number` components for the same reason the language keeps `number` distinct from `float64`: `number` is what an untyped literal and every `Math` function produce, so a `complex` built from `Math.PI` or a plain `3.5` composes without a cast. `complex64` and `complex128` fix the component width for an interleaved buffer or for interop, and convert to and from bare `complex` explicitly, exactly as `float32` and `float64` convert to and from `number`.

Being a value type, a `complex` copies on assignment and lies inline in an array, so `[].<complex128>` is the interleaved `(re, im, re, im, …)` buffer an FFT or a BLAS routine expects, while [structure of arrays](soa.md) splits it into separate real and imaginary columns for a lane-parallel pass. `typeof` reports `"object"`. `complex(0, 0)` is falsy on the same zero-is-falsy rule the other numeric types follow; every other complex value is truthy.

## Literals — the imaginary suffix

JavaScript reserves a numeric literal immediately followed by an identifier start as a `SyntaxError`, which is exactly the slot the `n` bigint suffix uses. The `i` suffix takes the same slot: `<number>i` is an imaginary literal.

```js
4i;      // complex(0, 4)
1i;      // the imaginary unit
2.5i;    // complex(0, 2.5)
```

Only `<number>i` is the suffix, so the bare identifier `i` is untouched and `for (let i = 0; i < n; ++i)` still means the loop variable. An imaginary literal is inherently complex, so it forces a real literal beside it onto the real axis, and the sum reads like the mathematics:

```js
let z: complex = 3 + 4i;    // complex(3, 4)
let w: complex = -2i;       // complex(0, -2)
let r: complex = 5;         // complex(5, 0)
```

Construction is also available by parts, and a single argument fills the real part:

```js
complex(3, 4);          // 3 + 4i
complex(5);             // 5 + 0i
complex.parse('3-2i');  // 3 - 2i, per the parse convention
```

In a bare `complex` context the components are `number`; in `complex64` they are `float32` and in `complex128` `float64`, the literal adopting the component type the same way any numeric literal adopts its context.

## Operators

The language defines complex arithmetic directly, not through operator overloading:

- `+` and `-` are componentwise.
- `*` is `(a + bi)(c + di) = (ac − bd) + (ad + bc)i`.
- `/` is `(a + bi)/(c + di) = [(ac + bd) + (bc − ad)i] / (c² + d²)`.
- unary `-` negates both components.
- `==` and `!=` compare both components with float semantics, so a complex with a `NaN` component is not equal to itself, exactly as a `NaN` float isn't.
- `**` is the complex power, by way of `exp` and `log`, and returns a complex.

```js
(1 + 2i) * (3 - 1i);   // 5 + 5i
(1 + 1i) / (1 - 1i);   // 0 + 1i
-(3 + 4i);             // -3 - 4i
(0 + 1i) ** 2;         // -1 + 0i
```

**Complex numbers are not ordered**, so `<`, `<=`, `>`, and `>=` on a complex are a `TypeError`. There is no order on the complex plane that respects arithmetic, and silently comparing real parts or magnitudes would hide the mistake; compare `Math.abs(z)` explicitly when a magnitude is what was meant. This is the one arithmetic operator a complex deliberately lacks.

Because a complex is float-backed, it inherits float behavior at the edges rather than throwing: dividing by `0 + 0i` yields components that are `NaN` or `Infinity`, the same result the underlying float division gives, with no `RangeError`.

Mixing follows the no-implicit-widening rule. A real literal propagates onto the real axis, so `z + 3` and `z * 2` read naturally — `2` becomes `complex(2, 0)` and the multiply scales both parts — but a real *value* does not convert on its own, so `z + x` for a `complex` `z` and a `number` `x` is a `TypeError`, written `z + complex(x)`. And `complex64` and `complex128` do not mix with bare `complex` or each other without an explicit cast, just as `float32`, `float64`, and `number` don't.

## Components, functions, and conversions

- `.re` and `.im` return the `T` real and imaginary parts.
- `Math.abs(z)` is the magnitude `√(re² + im²)`, a real `T`.
- `Math.conj(z)` is the conjugate `re − im·i`.
- `Math.arg(z)` is the phase `atan2(im, re)`, a real `T`.

The transcendental `Math` functions are overloaded for `complex` and return a `complex`, so the same name does the real thing on a real and the complex thing on a complex — `Math.sqrt` of a `complex` reaches the negative-argument answer a real `Math.sqrt` cannot:

```js
Math.sqrt(complex(-1));   // 0 + 1i
Math.sqrt(float64(-1));   // NaN, the real overload
Math.exp(complex(0, Math.PI));   // -1 + 0i, Euler's identity, within rounding
```

`Math.sin`, `Math.cos`, `Math.tan`, `Math.log`, and `Math.pow` extend the same way. Conversions are explicit in both directions: `complex(x)` lifts a real onto the plane, `.re` projects back off it, and `complex64` and `complex128` convert between each other with an explicit cast, like their component floats.

## Example

Mandelbrot escape time — iterate `z ← z² + c` and count the steps before the magnitude passes two:

```js
function escapeTime(c: complex, limit: uint32): uint32 {
  let z: complex = 0;               // 0 + 0i
  for (let n: uint32 = 0; n < limit; ++n) {
    if (Math.abs(z) > 2) {
      return n;
    }
    z = z * z + c;                  // native complex multiply and add
  }
  return 0;                         // stayed bounded
}

escapeTime(-0.5 + 0.5i, 100);       // escapes after a few steps
escapeTime(-0.1 + 0.75i, 100);      // 0: inside the set
```

The loop body is the definition of the set written verbatim, `z = z * z + c`, with the multiply mixing the components and the add offsetting them, both language operators. A tuned renderer would compare `Math.abs(z)` against a squared bound to skip the per-step square root, but the point stands: a complex is a first-class value here, so the arithmetic reads as arithmetic and a `[].<complex128>` of samples is the contiguous buffer the rest of a pipeline already wants.
