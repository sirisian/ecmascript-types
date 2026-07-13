# Decimal Numbers

Base-ten arithmetic, where `0.1` is exactly one tenth and `0.1 + 0.2` is exactly `0.3`. Binary floating point cannot hold a tenth — `0.1` in a `float64` is `0.1000000000000000055…` — which is why money, tax, billing, and every human-facing decimal quantity reach for a decimal type: C#'s `decimal`, Java's `BigDecimal`, Python's `decimal.Decimal`, SQL's `NUMERIC`, COBOL, Swift's `Decimal`. This proposal's decimal types are IEEE 754-2008 decimal floating point, listed among its primitives. This document defines them as value types with language-level operators, the way `int32` and `float64` are.

A decimal is the tool for values that *are* decimal — prices, rates, quantities to a fixed number of places. It is not a [rational](rational.md): a rational is exact for every fraction, including a third, and can overflow; a decimal is exact for every *terminating* decimal and rounds a third to its digit budget. Where a rational stores `1/3` exactly, a decimal stores `0.3333…3`; where a decimal stores `0.1` exactly, so does a rational, as `1/10`.

## Representation

The three decimal types are IEEE 754-2008 decimal floating point — a base-ten coefficient and a decimal exponent — differing only in how many significant digits they carry:

| Type | Significant digits | Bytes |
|---|---|---|
| `decimal32` | 7 | 4 |
| `decimal64` | 16 | 8 |
| `decimal128` | 34 | 16 |

There is no unwidthed `decimal`; a width is always chosen, and `decimal128` is the default for money and for most work, its digit count and exponent range large enough that overflow is a non-issue in practice. Because these are *floating* decimals rather than fixed, a `decimal128` represents any decimal up to thirty-four digits at any decimal position; pinning one to a fixed number of places is the job of `DecimalContext`, below. Each is a value type, so it copies on assignment and lies inline in a `[].<decimal128>`, and `typeof` reports `"number"` — a decimal is a numeric type, not a composite one like `rational` or `complex`.

## Literals and construction

A decimal literal is read from its source digits directly, not routed through a binary `float64`, so the value is exactly what was written:

```js
let tenth: decimal128 = 0.1;      // exactly 1 × 10⁻¹
let price: decimal128 = 19.99;    // exactly 19.99
let big: decimal128 = 9.999999999999999999999999999999999;  // 34 nines, exact
```

This is the whole point, and it is the same context propagation the rest of the proposal uses: in a decimal context the literal `0.1` is the decimal one tenth, where in a `float64` context the same `0.1` is the nearest binary float. The distinction bites only when a `float64` *value* is involved — `decimal128(f)` carries whatever `f` already holds, so a binary `0.1` stays slightly off — which is why an exact decimal comes from a literal or a string, never from a round trip through binary:

```js
decimal128.parse('19.99');   // exact, per the parse convention
0.1 := decimal128;           // exact, forcing decimal on an otherwise-Number literal
// decimal128(someFloat64);  // carries the float's binary value, rounded to 34 digits
```

## Operators

The language defines `+`, `-`, `*`, `/`, `**`, and the comparisons directly, in base ten. A result is rounded to the type's significant digits, ties to even by default, so operations that stay within the digit budget on terminating decimals are exact:

```js
let a: decimal128 = 0.1;
let b: decimal128 = 0.2;
a + b == 0.3;             // true — exactly 0.3
let price: decimal128 = 19.99;
price * 3;                // 59.97, exact — the literal 3 takes the decimal type
```

Two kinds of result are not exact, and they are handled differently. A non-terminating quotient rounds to the precision silently, as IEEE requires — `decimal128(1) / decimal128(3)` is `0.333…3` to thirty-four digits — which is the defining difference from a rational, whose division is exact. And a result outside the type's exponent range raises a `RangeError` rather than saturating, as does division by zero, because a decimal's range is a property of its type; this is the deliberate departure from `float64`, whose overflow is a quiet `Infinity`.

Mixing follows the no-implicit-widening rule: a literal propagates and takes the other operand's type, but a value of another type does not convert on its own — `d + n` for a `decimal128` `d` and an `int32` `n` is a `TypeError`, written `d + decimal128(n)` — and the three decimal widths do not mix with each other without an explicit cast, just as the integer and float widths don't.

## Significance and equality

IEEE decimal preserves trailing zeros: `1.0`, `1.00`, and `1.000` are distinct representations of one value, and they display with the places they were given, which is exactly what a printed price wants. `==` compares numerical value, so `1.0 == 1.00` is `true`, and as a `Map` or `Set` key a decimal compares by value under SameValueZero, so `1.0` and `1.00` are one key rather than two. Operations carry significance forward by the IEEE rules, and `.round(scale, rounding)` sets it explicitly.

## Scale and rounding: money

A bare `decimal128` floats, but money needs a *fixed* number of places and a stated rounding mode. Both live in the type through the `DecimalContext` meta type of [primitive metadata](primitivemetadata.md):

```js
type Cents = decimal128.<{ scale: 2 }>;                        // two places, half-even by default
type Bankers = decimal128.<{ scale: 2, rounding: Rounding.HalfEven }>;
```

`scale` is the number of digits kept after the point, and the rounding mode is applied when a value is quantized onto that grid. Quantization happens at assignment, argument, and return boundaries, so an expression's *intermediates* keep full precision and only the stored result rounds — `price * quantity * (1 + taxRate)` computes exactly and rounds to the cent only when it lands in a `Cents` slot. Composed with a currency meta type, `decimal128.<{ currency: 'USD', scale: 2 }>` is a money type that both rounds to the minor unit and makes mixing currencies a compile error, which the [invoicing](examples/invoicing.md) example builds end to end.

## Conversions

Explicit in every direction, and each names its loss:

- To a binary float: `float64(d)` rounds to the nearest `float64`, reintroducing the binary error a decimal existed to avoid.
- To an integer: `int64(d)` truncates toward zero.
- Between widths: `decimal32` to `decimal128` is exact; the reverse rounds.
- To and from a [rational](rational.md): a terminating decimal is exactly a rational with a power-of-ten denominator, so `rational(d)` is exact — `0.1` becomes `1/10` — while `decimal128(r)` rounds a non-terminating rational to the digit budget.

## Example

An order total, exact to the cent, next to the binary-float version that drifts:

```js
function orderTotal(unitPrice: decimal128, quantity: int32, taxRate: decimal128): decimal128.<{ scale: 2 }> {
  const beforeTax = unitPrice * decimal128(quantity);   // full precision
  return beforeTax * (1 + taxRate);                     // rounded to two places at the return
}

const t = orderTotal(19.99, 3, 0.0875);   // 65.22, rounded once, at the boundary
t.toString();                             // "65.22" — two places preserved

// The same arithmetic in binary float drifts, and 19.99 was never exact to begin with:
let f: float64 = 19.99 * 3 * (1 + 0.0875);   // 65.21737499999999, needing a manual round to the cent
```

Because the multiplications run at full precision and round only when the result lands in the scaled return type, there is no per-operation error to accumulate, the stored value keeps exactly two places for display, and the single rounding is the one an accountant would expect.
