# Rational Numbers

An exact fraction, numerator over denominator, with no floating-point error. Python has `fractions.Fraction`, Rust `num::Rational`, Haskell `Data.Ratio`, Scheme and Common Lisp exact rationals, and Raku rationals by default. Where a float drifts — `0.1 + 0.2` is not `0.3`, and `1/3 + 1/3 + 1/3` is not quite `1` — a rational is exact by construction: the three thirds sum to `1` on the nose. The uses are exact probability, rational coordinates in geometry, accumulation without rounding error, and any ratio a decimal can't hold because it isn't a multiple of a power of ten, like a third or a seventh.

This proposal already lists `rational` among its primitive types. This document defines it: a value type with language-level operators, the same way `int32` and `float64` are.

## Representation

`rational.<N>` is a value type holding two `int.<N>` fields, a numerator and a denominator, always kept in **canonical form**: reduced to lowest terms, denominator strictly positive, and zero represented as `0/1`. It occupies `2N` bits with the alignment of `int.<N>`. The bare name `rational` is `rational.<64>` — two `int64`, sixteen bytes — which is the default the primitive-types list refers to.

```js
type rational = rational.<64>;
```

Canonical form is the whole trick. Because every rational is stored reduced with a positive denominator, two rationals are equal exactly when their bytes are equal, so structural `==` *is* mathematical equality, and hashing and `Map` keys follow for free with no cross-multiplication and no separate equality method:

```js
rational(1, 2) == rational(2, 4);   // true: both are the bytes 1/2
new Set.<rational>([rational(1, 2), rational(50, 100)]).size;  // 1
```

Being a value type, a `rational` copies on assignment, lives inline in a `[].<rational>` as interleaved numerator/denominator pairs, and splits into numerator and denominator columns under [structure of arrays](soa.md). `typeof` reports `"object"`, as it does for the other composite numeric types.

## Literals and construction

No new syntax is needed. A numeric literal takes the type of its context, and `/` between rational-typed operands is rational division, so the natural spelling of a fraction just works:

```js
let third: rational = 1 / 3;    // exactly 1/3
let half: rational = 6 / 12;    // reduced to 1/2
let whole: rational = 5;        // 5/1
```

This is worth reading twice, because the same tokens mean different things in different contexts, exactly as the conversions section already requires. In `let third: rational = 1 / 3` the context is `rational`, so `1` and `3` are rational literals and `/` is rational division, giving `1/3`. With an `int32` context the same `1 / 3` is integer division and gives `0`; in an untyped context it is `Number` division and gives `0.333…`. The literal never converts a typed value — it just adopts the type the context asks for.

Construction from parts reduces and normalizes the sign, moving it to the numerator:

```js
rational(2, 4);     // 1/2
rational(1, -3);    // -1/3
rational(0, 5);     // 0/1
rational(5);        // 5/1
// rational(1, 0);  // RangeError: zero denominator
rational.parse('3/4');   // 3/4, per the parse convention for the numeric types
rational.parse('0.1');   // 1/10 - any literal of the type, as 0.1 in a rational position is
// rational.parse('1/0'); // RangeError: zero denominator, as the constructor's
```

The two-argument constructor takes `int.<N>` numerator and denominator; literals propagate into them, and a value of another integer type is converted explicitly, as everywhere else.

## Operators

`++` and `--` step a rational by one in its own type, as they step an integer, so an exact counter can be a rational.

The language defines the operators directly, the way it does for `int32`, not through operator overloading. Every result is exact and returned in canonical form:

- `+`, `-`, `*`, `/` are exact rational arithmetic.
- unary `-` negates the numerator.
- `<`, `<=`, `>`, `>=`, `==`, `!=` are the exact total order on rationals — the cross-multiplication is the implementation's, never the caller's.
- `**` with an integer exponent is the exact power `(a/b)**n`, with a negative exponent inverting.

```js
let a: rational = 1 / 6;
let b: rational = 1 / 3;
a + b;        // 1/2
a * b;        // 1/18
b / a;        // 2/1
b < a;        // false
rational(2, 3) ** 3; // 8/27, exact
```

Overflow — a numerator or denominator that no longer fits `int.<N>` after reduction — raises a `RangeError`, the same choice the arithmetic section makes for decimals, because a rational's range is a property of its type rather than of a bit pattern to wrap. Dividing by a zero rational raises a `RangeError`. A wider width postpones the first case: `rational.<128>` reduces overflow to a practical non-issue for most accumulation, and an arbitrary-precision rational is a `bigint`-backed reference type outside this value-type primitive.

Mixing follows the no-implicit-widening rule. A literal propagates, but a value of another type does not convert on its own: `r + n` for a `rational` `r` and an `int32` `n` is a `TypeError`, written `r + rational(n)`; and `rational.<32>` and `rational.<64>` do not mix without an explicit cast, just as `int32` and `int64` don't.

## Components, methods, and conversions

- `.numerator` and `.denominator` return the `int.<N>` fields.
- `.reciprocal()` returns `b/a`, a `RangeError` on zero.
- `Math.abs`, `Math.sign`, `Math.min`, and `Math.max` are overloaded for `rational`; `Math.floor`, `Math.ceil`, `Math.round`, and `Math.trunc` return the `int.<N>` nearest in their direction.

Conversions are all explicit:

- To a float: `float64(r)` is `numerator / denominator` rounded to the nearest `float64`. This is the lossy step, and it is visible.
- To an integer: `int64(r)` truncates toward zero.
- From an integer: `rational(n)` is `n/1`, exact.
- From a `bigint`: `rational(5n)` is `5/1`, a `RangeError` where the width cannot hold it.
- Between widths: `r := rational.<32>` is the same value at the new width, a `RangeError` where it does not fit. The widths are distinct types - `rational.<32>` and `rational.<64>` do not mix without this cast, and `===` sees the width as it does for `int32` and `int64` while `==` compares the value.
- From a float: `rational(f)` is the float's *exact* dyadic value — a `float64` is itself a rational whose denominator is a power of two — which is exact but can overflow a fixed width, in which case it raises a `RangeError`. For a bounded approximation, `rational.approximate(f, maxDenominator)` returns the closest rational whose denominator does not exceed the bound, by the continued-fraction expansion. At a width it returns the closest value OF THAT TYPE - both parts in `int.<N>` - so `rational.<8>.approximate(Math.PI, 1000)` is `22/7`, where `355/113`'s numerator is not an `int.<8>`. Of two values equally near, it returns the one with the smaller denominator, then the one nearer zero; an argument outside the type's range is a `RangeError`, since that is overflow rather than approximation. A literal is not a float: `rational(0.1)` is `1/10`, as `let r: rational = 0.1` is, because a literal denotes its digits. The dyadic value of the double nearest one tenth is had by making it a float first, `rational(float64(0.1))`.

```js
rational(0.5);                          // 1/2, exact
float64(rational(1, 3));                // 0.3333333333333333, now rounded
rational.approximate(Math.PI, 1000);    // 355/113
```

## Example

An exact harmonic partial sum, next to the float version that drifts:

```js
function harmonic(n: int64): rational {
  let sum: rational = 0;          // 0/1
  for (let k: int64 = 1; k <= n; ++k) {
    sum += rational(1, k);        // 1/k, exact
  }
  return sum;
}

const h = harmonic(4);            // 25/12, exactly
h.numerator;                      // 25
h.denominator;                    // 12
float64(h);                       // 2.0833333333333335, only now rounded

// The float accumulation reaches a different, rounded value:
let f: float64 = 0;
for (let k: int64 = 1; k <= 4; ++k) {
  f += 1 / float64(k);
}
f == float64(h);                  // false — the exact sum and the drifted sum differ
```

Because the accumulator stays canonical, `harmonic(n) == harmonic(n)` is true structurally, the partial sums can be compared and ordered exactly, and the single rounding happens where the program asks for it, at the `float64` cast, not silently on every `+`.

## The unbounded rational

`rational.<N>` is bounded by design: two `int.<N>` fields, a fixed layout, a `RangeError` on overflow. For arithmetic that must never overflow there is `rational.<bigint>` - the quotient of two `bigint` values, every rational there is. It is the shape Julia (`Rational{BigInt}`), Rust (`Ratio<BigInt>`) and Haskell (`Ratio Integer`) give the same need: one rational family over its integer, the unbounded rational being the one over the unbounded integer.

```js
let x = rational.<bigint>(1, 3);
x * x * x;                              // 1/27
rational.<bigint>(2n ** 200n) / rational.<bigint>(3n);   // exact, however large
x.numerator;                            // 1n - a bigint
Math.floor(rational.<bigint>(7, 2));    // 3n
x := rational.<8>;                      // 1/3 - a RangeError where the width cannot hold it
```

Every rule of `rational.<N>` holds with the bound removed. Conversions into it are exact - from any integer, a `bigint`, a float's exact value or a decimal's - and refused only for NaN and the infinities; conversions out to a width are exact or a `RangeError`. It is a type of its own: `===` sees the type, `==` the value, and mixing it with a width needs a conversion. `approximate` bounds only the denominator.

Like `bigint`, it has no layout: its size belongs to the value. A class holding one is still a value type but has no layout, so it neither lives inline in an array nor splits into columns - the two things a fixed width buys.

The cost of never overflowing is real, and it is why this is a type a program chooses rather than the default. Squaring a rational roughly doubles its size: iterating `x = x * x + 1/7` from `1/3`, the denominator is 96 bits at the fifth step - where a `rational.<64>` has already thrown - and 3.1 million bits at the twentieth, whose single step took about 13 seconds in Python's unbounded `Fraction`. A bounded rational fails fast and says so; an unbounded one keeps going, and the only symptom is time.

