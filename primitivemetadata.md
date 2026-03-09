# Primitive Metadata 

## Introduction

Primitive/value types lack information about what their value means and where it's relevant. Units of measure are a common example where the value ```1``` can be used anywhere as it's dimensionless; however, the value and unit ```1m```, 1 meter, is more specific and might only work in certain situations. The ```m``` can be thought of as metadata attached to the type that the language can use to further restrict where it's accepted. Operations performed on such a value and unit can modify the unit portion. This is just applying custom operators on the metadata, like when multiplying two lengths the metadata changes to represent ```m**2```.

Rather than hardcoding specific metadata features into the language, like units of measure, a system should be defined that allows any metadata to coexist with a primitive value. This document specifies a system for attaching structured metadata to primitive types, enabling compile-time and runtime dimensional analysis (units of measure), value constraints (JSON Schema constraints), and user-defined operator semantics.

This should be viewed as a rough draft to list requirements and a minimal syntax proposal to work from. (Which is how the main proposal is written. Put something out there and update it iteratively).

This analysis covers some features or overlaps with features in the following proposals:

https://github.com/tc39/proposal-amount  
https://github.com/sirisian/ecmascript-types/blob/master/decorators.md

## Proposed Solution

The proposed solution has three components:

1. Metadata types - plain type declarations whose fields attach to primitives via <{...}> syntax.
2. Meta protocols - meta blocks that teach the compiler the semantics of a metadata type (subtyping, validation, narrowing).
3. Primitive operator blocks - primitive T<M: MetaType> blocks that define how operators transform metadata.

### Metadata Types and Protocols

The metadata type is just a typed object defining the metadata parts and their individual types.

```js
type Metadata = { part: int32 };
const a: float32<{ part: 1 }> = 1;
```

The metadata protocol defines how a primitive with a metadata type propagates through the language at compile time and runtime. This protocol is defined in meta blocks that define the semantic hooks for a metadata type. All hooks are pure functions that can be evaluated at both compile time and runtime.

```js
meta T {
	// Required: the "unconstrained" / "not specified" value.
	// Used when a value has no fields belonging to this meta type.
	default: T;

	// Required: is sub's constraint set a subset of sup's constraint set?
	// Used for assignment compatibility checks.
	// The compiler calls this to determine if a value of type T<sub> is assignable to T<sup>.
	subtype(sub: T, sup: T): boolean;

	// Optional: does a concrete value satisfy the constraint?
	// Used for runtime validation when subtype() can't prove compatibility at compile time. Also used by type constructors.
	validate?(value: primitive, constraint: T): boolean;

	// Optional: single-branch control flow narrowing.
	// Called by the compiler when it encounters a comparison
	// in an if/while/ternary condition.
	// The compiler handles operator negation for the else branch:
	// >=  to  <
	// >   to  <=
	// ==  to  !=
	// Returns only meaningful fields, absence means unconstrained.
	narrow?(current: T, op: string, value: primitive): T;

	// Optional: human-readable description for error messages.
	describe?(constraint: T): string;
}
```

It's possible to hold a reference to a meta protocol:

```js
interface MetaProtocol<T> {
    default: T;
    subtype(sub: T, sup: T): boolean;
    validate?(value: any, constraint: T): boolean;
    narrow?(current: T, op: string, value: any): T;
    describe?(constraint: T): string;
}
```

Two examples will be used, a dimensions (units of measure) and bounds (minimum and maximum constraints from JSON Schema).

**Metadata Type**: Dimensions  
Tracks SI base dimensions as integer exponents plus a rational scale factor for unit prefixes. The ratio field encodes the relationship between a prefixed unit and its base SI unit (e.g., ratio: 1000.0 for kilometers, ratio: 0.001 for millimeters).

```js
type Dimensions = {
	m: int32, // length exponent
	kg: int32, // mass exponent
	s: int32, // time exponent
	ratio: float32, // scale factor relative to base SI (1.0 = base)
};

meta Dimensions {
	default = { m: 0, kg: 0, s: 0, ratio: 1.0 };

	// Dimensional subtyping: exact match on exponents.
	// Ratio can differ - a Kilometer is assignable to a Meter-typed slot; the value is scaled at assignment time.
	subtype(sub: Dimensions, sup: Dimensions): boolean {
		return sub.m == sup.m && sub.kg == sup.kg && sub.s == sup.s;
	}

	// No validate - dimensions constrain type compatibility, not value ranges.
	// Any float32 value is valid for any dimension.

	// No narrow - comparison operators don't affect dimensions.

	describe(constraint: Dimensions): string {
		const parts: string[] = [];
		if (constraint.m != 0)
			parts.push(constraint.m == 1 ? "m" : `m^${constraint.m}`);
		if (constraint.kg != 0)
			parts.push(constraint.kg == 1 ? "kg" : `kg^${constraint.kg}`);
		if (constraint.s != 0)
			parts.push(constraint.s == 1 ? "s" : `s^${constraint.s}`);
		const dim = parts.join("·") || "dimensionless";
		return !feq(constraint.ratio, 1.0) ? `${constraint.ratio}* ${dim}` : dim;
	}
}
```

**Metadata Type**: Bounds  
JSON Schema numeric constraints. Absent fields mean unconstrained in that direction. Both inclusive (minimum, maximum) and exclusive (exclusiveMinimum, exclusiveMaximum) bounds are supported. When both exist on the same side, the tighter (more restrictive) one takes effect.

<details>
	<summary>Expand for float32 epsilon helper functions.</summary>
	
```js
const FLOAT32_EPSILON: float32 = 1.1920929e-7;
const REL_TOLERANCE: float32 = 4.0 * FLOAT32_EPSILON;
const ABS_TOLERANCE: float32 = 1e-12;

function feq(a: float32, b: float32): boolean {
	if (a == b) return true; // handles ±0, exact matches
	const diff = Math.abs(a - b);
	const magnitude = Math.max(Math.abs(a), Math.abs(b));
	return diff <= Math.max(ABS_TOLERANCE, magnitude * REL_TOLERANCE);
}

function fle(a: float32, b: float32): boolean {
	return a < b || feq(a, b);
}

function fge(a: float32, b: float32): boolean {
	return a > b || feq(a, b);
}

function flt(a: float32, b: float32): boolean {
	return a < b && !feq(a, b);
}

function fgt(a: float32, b: float32): boolean {
	return a > b && !feq(a, b);
}
```
</details>

```js
type Bounds = {
	minimum?: float32,
	maximum?: float32,
	exclusiveMinimum?: float32,
	exclusiveMaximum?: float32,
};

meta Bounds {
	default = {};

	subtype(sub: Bounds, sup: Bounds): boolean {
		const subLo = effectiveMin(sub);
		const supLo = effectiveMin(sup);
		const subHi = effectiveMax(sub);
		const supHi = effectiveMax(sup);
		const subLoX = isExclusiveMin(sub);
		const supLoX = isExclusiveMin(sup);
		const subHiX = isExclusiveMax(sub);
		const supHiX = isExclusiveMax(sup);

		// Sub's lower bound must be at least as tight as sup's
		if (supLo != null) {
			if (subLo == null) return false;
			if (flt(subLo, supLo)) return false;
			// Equal within epsilon: exclusive is tighter than inclusive
			if (feq(subLo, supLo) && supLoX && !subLoX) return false;
		}

		// Sub's upper bound must be at least as tight as sup's
		if (supHi != null) {
			if (subHi == null) return false;
			if (fgt(subHi, supHi)) return false;
			if (feq(subHi, supHi) && supHiX && !subHiX) return false;
		}

		return true;
	}

	validate(value: float32, constraint: Bounds): boolean {
		if (constraint.minimum != null && flt(value, constraint.minimum))
			return false;
		if (constraint.exclusiveMinimum != null && fle(value, constraint.exclusiveMinimum))
			return false;
		if (constraint.maximum != null && fgt(value, constraint.maximum))
			return false;
		if (constraint.exclusiveMaximum != null && fge(value, constraint.exclusiveMaximum))
			return false;
		return true;
	}

	narrow(current: Bounds, op: string, value: float32): Bounds {
		const result: Bounds = { ...current };

		switch (op) {
			case ">=": {
				const curMin = effectiveMin(current);
				if (curMin == null || fgt(value, curMin)) {
					result.minimum = value;
					delete result.exclusiveMinimum;
				}
				break;
			}
			case ">": {
				const curMin = effectiveMin(current);
				if (curMin == null || fge(value, curMin)) {
					result.exclusiveMinimum = value;
					delete result.minimum;
				}
				break;
			}
			case "<=": {
				const curMax = effectiveMax(current);
				if (curMax == null || flt(value, curMax)) {
					result.maximum = value;
					delete result.exclusiveMaximum;
				}
				break;
			}
			case "<": {
				const curMax = effectiveMax(current);
				if (curMax == null || fle(value, curMax)) {
					result.exclusiveMaximum = value;
					delete result.maximum;
				}
				break;
			}
			case "==": {
				result.minimum = value;
				result.maximum = value;
				delete result.exclusiveMinimum;
				delete result.exclusiveMaximum;
				break;
			}
			// "!=" cannot meaningfully narrow a single range
			// (would require union of two disjoint ranges)
		}

		return clean(result);
	}

	describe(constraint: Bounds): string {
		const parts: string[] = [];
		if (constraint.minimum != null)
			parts.push(`>= ${constraint.minimum}`);
		if (constraint.exclusiveMinimum != null)
			parts.push(`> ${constraint.exclusiveMinimum}`);
		if (constraint.maximum != null)
			parts.push(`<= ${constraint.maximum}`);
		if (constraint.exclusiveMaximum != null)
			parts.push(`< ${constraint.exclusiveMaximum}`);
		return parts.join(" and ") || "unconstrained";
	}
}
```

### Primitive Operators

```js
primitive float32<D: Dimensions> {
	// Same-dimension addition
	// D2 captures the RHS's actual metadata, including its ratio.
	// The where clause enforces matching exponents (same physical dimension).
	// If ratios differ, RHS is scaled to LHS's unit system.
	operator+<D2: Dimensions>(rhs: float32<D2>): float32<D>
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (feq(D.ratio, D2.ratio)) {
			return this + rhs;
		}
		return this + rhs * (D2.ratio / D.ratio);
	}

	operator-<D2: Dimensions>(rhs: float32<D2>): float32<D>
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (feq(D.ratio, D2.ratio)) {
			return this - rhs;
		}
		return this - rhs * (D2.ratio / D.ratio);
	}

	// Dimension-combining multiplication
	// Exponents add, ratios multiply.
	operator*<D2: Dimensions>(rhs: float32<D2>):
		float32<{
			m: D.m + D2.m,
			kg: D.kg + D2.kg,
			s: D.s + D2.s,
			ratio: D.ratio * D2.ratio,
		}>
	{
		return this * rhs;
	}

	// Dimension-combining division
	// Exponents subtract, ratios divide.
	operator/<D2: Dimensions>(rhs: float32<D2>):
		float32<{
			m: D.m - D2.m,
			kg: D.kg - D2.kg,
			s: D.s - D2.s,
			ratio: D.ratio / D2.ratio,
		}>
	{
		return this / rhs;
	}

	// Unary operators

	operator-(): float32<D> {
		return -this;
	}

	operator+(): float32<D> {
		return +this;
	}

	// Scalar multiplication/division
	// A plain float32 (no metadata) is dimensionless.
	// Multiplying preserves the LHS dimension.

	operator*(rhs: float32): float32<D> {
		return this * rhs;
	}

	operator/(rhs: float32): float32<D> {
		return this / rhs;
	}

	// Compound assignment
	// `this` is assignable inside operator bodies, enabling
	// in-place mutation. Returns the LHS type so the expression
	// evaluates to the new value, allowing chaining.

	operator+=<D2: Dimensions>(rhs: float32<D2>): float32<D>
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (!feq(D.ratio, D2.ratio)) {
			this += rhs * (D2.ratio / D.ratio);
		} else {
			this += rhs;
		}
		return this;
	}

	operator-=<D2: Dimensions>(rhs: float32<D2>): float32<D>
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (!feq(D.ratio, D2.ratio)) {
			this -= rhs * (D2.ratio / D.ratio);
		} else {
			this -= rhs;
		}
		return this;
	}

	operator*=(rhs: float32): float32<D> {
		this *= rhs;
		return this;
	}

	operator/=(rhs: float32): float32<D> {
		this /= rhs;
		return this;
	}

	// Comparison operators
	// Same dimension required (enforced by where clause).
	// Values are normalized to base SI units before comparison so that e.g. Kilometer(1) == Meter(1000).

	operator==<D2: Dimensions>(rhs: float32<D2>): boolean
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		return feq(this * D.ratio, rhs * D2.ratio);
	}

	operator!=<D2: Dimensions>(rhs: float32<D2>): boolean
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		return !feq(this * D.ratio, rhs * D2.ratio);
	}

	operator< <D2: Dimensions>(rhs: float32<D2>): boolean
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		return flt(this * D.ratio, rhs * D2.ratio);
	}

	operator<=<D2: Dimensions>(rhs: float32<D2>): boolean
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		return fle(this * D.ratio, rhs * D2.ratio);
	}

	operator> <D2: Dimensions>(rhs: float32<D2>): boolean
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		return fgt(this * D.ratio, rhs * D2.ratio);
	}

	operator>=<D2: Dimensions>(rhs: float32<D2>): boolean
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		return fge(this * D.ratio, rhs * D2.ratio);
	}

	// Cast: strip dimension, apply scale to get base SI value

	operator float32(): float32 {
		return this * D.ratio;
	}
}
```

**Bounds Operators**  
Bounds operators only modify the return metadata with no value, so they have no function body.

<details>
	<summary>Expand for boundary helper functions.</summary>
	
```js
// Effective lower bound: if both inclusive and exclusive exist,
// the tighter (larger) one wins.
function effectiveMin(b: Bounds): float32 | null {
	if (b.minimum != null && b.exclusiveMinimum != null) {
		return Math.max(b.minimum, b.exclusiveMinimum);
	}
	return b.minimum ?? b.exclusiveMinimum ?? null;
}

// Effective upper bound: if both inclusive and exclusive exist,
// the tighter (smaller) one wins.
function effectiveMax(b: Bounds): float32 | null {
	if (b.maximum != null && b.exclusiveMaximum != null) {
		return Math.min(b.maximum, b.exclusiveMaximum);
	}
	return b.maximum ?? b.exclusiveMaximum ?? null;
}

// Determine whether the effective minimum is exclusive
function isExclusiveMin(b: Bounds): boolean {
	if (b.minimum != null && b.exclusiveMinimum != null) {
		return b.exclusiveMinimum >= b.minimum;
	}
	return b.exclusiveMinimum != null;
}

// Determine whether the effective maximum is exclusive
function isExclusiveMax(b: Bounds): boolean {
	if (b.maximum != null && b.exclusiveMaximum != null) {
		return b.exclusiveMaximum <= b.maximum;
	}
	return b.exclusiveMaximum != null;
}

// Remove undefined fields, absence means unconstrained.
// Never store infinities in metadata.
function clean(b: Bounds): Bounds {
	const result: Bounds = {};
	if (b.minimum != null) result.minimum = b.minimum;
	if (b.maximum != null) result.maximum = b.maximum;
	if (b.exclusiveMinimum != null) result.exclusiveMinimum = b.exclusiveMinimum;
	if (b.exclusiveMaximum != null) result.exclusiveMaximum = b.exclusiveMaximum;
	return result;
}

// Construct a Bounds from effective values + exclusivity flags
function makeBounds(
	lo: float32 | undefined, loExclusive: boolean,
	hi: float32 | undefined, hiExclusive: boolean,
): Bounds {
	const result: Bounds = {};
	if (lo != null) {
		if (loExclusive) {
			result.exclusiveMinimum = lo;
		} else {
			result.minimum = lo;
		}
	}
	if (hi != null) {
		if (hiExclusive) {
			result.exclusiveMaximum = hi;
		} else {
			result.maximum = hi;
		}
	}
	return result;
}

// Interval arithmetic for multiplication.
// Computes the bounds of the product of two bounded values.
function boundsFromProducts(a: Bounds, b: Bounds): Bounds {
	const aLo = effectiveMin(a);
	const aHi = effectiveMax(a);
	const bLo = effectiveMin(b);
	const bHi = effectiveMax(b);

	// Can't propagate without complete bounds on both sides
	if (aLo == null || aHi == null || bLo == null || bHi == null) {
		// Partial propagation for semi-bounded cases:
		// If both are non-negative with a lower bound, result has a lower bound
		if (aLo != null && bLo != null && fge(aLo, 0) && fge(bLo, 0)) {
			const loExclusive = isExclusiveMin(a) || isExclusiveMin(b);
			return clean(makeBounds(aLo * bLo, loExclusive, undefined, false));
		}
		return {};
	}

	const products = [aLo * bLo, aLo * bHi, aHi * bLo, aHi * bHi];
	const lo = Math.min(...products);
	const hi = Math.max(...products);

	// Determine exclusivity: if the contributing bounds were exclusive,
	// the result bound is exclusive.
	// Find which product(s) achieved the min/max and check their source bounds.
	const loExclusive = products.some((p, i) => {
		if (!feq(p, lo)) return false;
		const usedALo = (i == 0 || i == 1);
		const usedBLo = (i == 0 || i == 2);
		return (usedALo && isExclusiveMin(a)) || (!usedALo && isExclusiveMax(a))
			|| (usedBLo && isExclusiveMin(b)) || (!usedBLo && isExclusiveMax(b));
	});

	const hiExclusive = products.some((p, i) => {
		if (!feq(p, hi)) return false;
		const usedALo = (i == 0 || i == 1);
		const usedBLo = (i == 0 || i == 2);
		return (usedALo && isExclusiveMin(a)) || (!usedALo && isExclusiveMax(a))
			|| (usedBLo && isExclusiveMin(b)) || (!usedBLo && isExclusiveMax(b));
	});

	return clean(makeBounds(lo, loExclusive, hi, hiExclusive));
}

// Sum: [a_lo, a_hi] + [b_lo, b_hi] = [a_lo + b_lo, a_hi + b_hi]
// Exclusivity: if EITHER contributing bound is exclusive, the result is exclusive.
// Example: (>= 3) + (> 5) = (> 8) because b can approach 5 but never reach it.
function boundsFromSum(a: Bounds, b: Bounds): Bounds {
	const aLo = effectiveMin(a);
	const aHi = effectiveMax(a);
	const bLo = effectiveMin(b);
	const bHi = effectiveMax(b);

	const lo = (aLo != null && bLo != null) ? aLo + bLo : undefined;
	const hi = (aHi != null && bHi != null) ? aHi + bHi : undefined;

	const loExclusive = lo != null && (isExclusiveMin(a) || isExclusiveMin(b));
	const hiExclusive = hi != null && (isExclusiveMax(a) || isExclusiveMax(b));

	return clean(makeBounds(lo, loExclusive, hi, hiExclusive));
}

// Difference: [a_lo, a_hi] - [b_lo, b_hi] = [a_lo - b_hi, a_hi - b_lo]
// Note the cross: result lower = a's lower - b's UPPER.
// Exclusivity: if either contributing bound is exclusive, result is exclusive.
// Example: (>= 5) - (< 3) = (> 2) because b can approach 3 from below,
// making the difference approach 2 from above but never reaching it.
function boundsFromDifference(a: Bounds, b: Bounds): Bounds {
	const aLo = effectiveMin(a);
	const aHi = effectiveMax(a);
	const bLo = effectiveMin(b);
	const bHi = effectiveMax(b);

	const lo = (aLo != null && bHi != null) ? aLo - bHi : undefined;
	const hi = (aHi != null && bLo != null) ? aHi - bLo : undefined;

	const loExclusive = lo != null && (isExclusiveMin(a) || isExclusiveMax(b));
	const hiExclusive = hi != null && (isExclusiveMax(a) || isExclusiveMin(b));

	return clean(makeBounds(lo, loExclusive, hi, hiExclusive));
}

// Negation: -[lo, hi] = [-hi, -lo]
// Exclusivity follows the original bound that was negated.
function negateBounds(b: Bounds): Bounds {
	const lo = effectiveMin(b);
	const hi = effectiveMax(b);
	const loExclusive = isExclusiveMin(b);
	const hiExclusive = isExclusiveMax(b);

	return clean(makeBounds(
		hi != null ? -hi : undefined, hiExclusive,
		lo != null ? -lo : undefined, loExclusive,
	));
}

// Scalar multiplication of bounds.
// Correctly handles negative scalars by swapping and re-pairing
// the inclusive/exclusive attributes.
function scalarMulBounds(b: Bounds, scalar: float32): Bounds {
	if (feq(scalar, 0)) {
		return { minimum: 0, maximum: 0 };
	}

	const lo = effectiveMin(b);
	const hi = effectiveMax(b);
	const loExclusive = isExclusiveMin(b);
	const hiExclusive = isExclusiveMax(b);

	if (scalar > 0) {
		// Positive scalar: bounds scale directly, exclusivity preserved
		return clean(makeBounds(
			lo != null ? lo * scalar : undefined, loExclusive,
			hi != null ? hi * scalar : undefined, hiExclusive,
		));
	}

	// Negative scalar: lo and hi swap roles
	//   old lower bound * negative -> new upper bound
	//   old upper bound * negative -> new lower bound
	//   exclusivity follows the original bound, not the position
	return clean(makeBounds(
		hi != null ? hi * scalar : undefined, hiExclusive, // old hi -> new lo
		lo != null ? lo * scalar : undefined, loExclusive, // old lo -> new hi
	));
}
```
</details>

```js
primitive float32<B: Bounds> {
	operator+<B2: Bounds>(rhs: float32<B2>): float32<boundsFromSum(B, B2)>;
	operator-<B2: Bounds>(rhs: float32<B2>): float32<boundsFromDifference(B, B2)>;
	operator*<B2: Bounds>(rhs: float32<B2>): float32<boundsFromProducts(B, B2)>;

	operator-(): float32<negateBounds(B)>;

	operator*(rhs: float32): float32<scalarMulBounds(B, rhs)>;
	operator/(rhs: float32): float32<scalarMulBounds(B, 1.0 / rhs)>;
}
```

### Composition Rules

For a given operator invocation:

1. At most one value block may match. Its body computes the result value. If two value blocks match the same operator, the compiler reports an ambiguity error.
2. Any number of metadata-only blocks may match. Each contributes its portion of the result metadata via its return type annotation.
3. If no value block matches, the default primitive operation runs.
4. All return type annotations (from both value and metadata-only blocks) are evaluated independently, and their metadata fields are merged into the flat result object.

#### Example: `Kilometer(5) + Meter(300)` where both have `Bounds { minimum: 0 }`

**Dimensions block** (value block):
- `where` clause: `m: 1 == m: 1 && kg: 0 == kg: 0 && s: 0 == s: 0`
- Body runs: `5 + 300 * (1 / 1000) = 5.3`
- Return metadata (Dimensions portion): `{ m: 1, kg: 0, s: 0, ratio: 1000 }`

**Bounds block** (metadata-only):
- Return metadata (Bounds portion): `boundsFromSum({ minimum: 0 }, { minimum: 0 })` = `{ minimum: 0 }`
- No body, so no conflicting value calculation.

**Final result:** float32 value `5.3` with merged metadata `{ m: 1, kg: 0, s: 0, ratio: 1000, minimum: 0 }`

This resolves the potential conflict where Dimensions would compute `5.3` (ratio-scaled) but a Bounds body would compute `305` (unscaled). The metadata-only block never touches the value.

## Unit Type Aliases

All metadata is specified as flat objects. The compiler decomposes automatically based on field ownership: `{m, kg, s, ratio}` fields are claimed by `Dimensions`, `{minimum, maximum, exclusiveMinimum, exclusiveMaximum}` fields are claimed by `Bounds`.

### Base SI Units

```js
type Meter = float32<{ m: 1, kg: 0, s: 0, ratio: 1.0 }>;
type Kilogram = float32<{ m: 0, kg: 1, s: 0, ratio: 1.0 }>;
type Second = float32<{ m: 0, kg: 0, s: 1, ratio: 1.0 }>;
```

### Prefixed Units

The `ratio` field encodes the prefix as a scale factor relative to the base SI unit.

```js
type Kilometer = float32<{ m: 1, kg: 0, s: 0, ratio: 1000.0 }>;
type Centimeter = float32<{ m: 1, kg: 0, s: 0, ratio: 0.01 }>;
type Millimeter = float32<{ m: 1, kg: 0, s: 0, ratio: 0.001 }>;
type Micrometer = float32<{ m: 1, kg: 0, s: 0, ratio: 0.000001 }>;
type Gram = float32<{ m: 0, kg: 1, s: 0, ratio: 0.001 }>;
type Milligram = float32<{ m: 0, kg: 1, s: 0, ratio: 0.000001 }>;
type Millisecond = float32<{ m: 0, kg: 0, s: 1, ratio: 0.001 }>;
type Microsecond = float32<{ m: 0, kg: 0, s: 1, ratio: 0.000001 }>;
```

### Derived SI Units

```js
type Velocity = float32<{ m: 1, kg: 0, s: -1, ratio: 1.0 }>; // m/s
type Acceleration = float32<{ m: 1, kg: 0, s: -2, ratio: 1.0 }>; // m/s**2
type Newton = float32<{ m: 1, kg: 1, s: -2, ratio: 1.0 }>; // kg*m/s**2
type Joule = float32<{ m: 2, kg: 1, s: -2, ratio: 1.0 }>; // kg*m**2/s**2
type Watt = float32<{ m: 2, kg: 1, s: -3, ratio: 1.0 }>; // kg*m**2/s**3
type Pascal = float32<{ m: -1, kg: 1, s: -2, ratio: 1.0 }>; // kg/(m*s**2)
type Hertz = float32<{ m: 0, kg: 0, s: -1, ratio: 1.0 }>; // 1/s
type Momentum = float32<{ m: 1, kg: 1, s: -1, ratio: 1.0 }>; // kg*m/s
type Density = float32<{ m: -3, kg: 1, s: 0, ratio: 1.0 }>; // kg/m**3
type Dimensionless = float32<{ m: 0, kg: 0, s: 0, ratio: 1.0 }>;
```

### Compound Prefixed Derived Units

```js
type KilometersPerHour = float32<{ m: 1, kg: 0, s: -1, ratio: 1000.0 / 3600.0 }>;
type GramsPerCubicCm = float32<{ m: -3, kg: 1, s: 0, ratio: 0.001 / 0.000001 }>;
```

### Bounds-Only Types (No Dimension)

```js
type Positive = float32<{ exclusiveMinimum: 0 }>;
type NonNegative = float32<{ minimum: 0 }>;
type Normalized = float32<{ minimum: 0, maximum: 1 }>;
type Probability = Normalized;
type Percentage = float32<{ minimum: 0, maximum: 100 }>;
```

### Combined: Dimension + Bounds

Fields from both `Dimensions` and `Bounds` appear in a single flat object. The compiler decomposes them automatically.

```js
type PositiveMeter = float32<{
	m: 1, kg: 0, s: 0, ratio: 1.0,
	exclusiveMinimum: 0
}>;

type SafeSpeed = float32<{
	m: 1, kg: 0, s: -1, ratio: 1.0,
	minimum: 0,
	maximum: 343
}>;

type Latitude = float32<{
	m: 0, kg: 0, s: 0, ratio: 1.0,
	minimum: -90,
	maximum: 90
}>;

type Longitude = float32<{
	m: 0, kg: 0, s: 0, ratio: 1.0,
	minimum: -180,
	exclusiveMaximum: 180
}>;
```

## Compiler Decomposition Rules

When the compiler encounters `float32<{ ... }>`:

**Step 1.** Collect all registered `meta` types and their field sets.

```
Dimensions -> { m, kg, s, ratio }
Bounds -> { minimum, maximum, exclusiveMinimum, exclusiveMaximum }
```

**Step 2.** For each field in the metadata object, find which `meta` type claims it. Each field must belong to exactly one `meta` type. Unclaimed fields produce a compile error.

**Step 3.** Group fields by their `meta` type.

```
Input: { m: 1, kg: 0, s: -1, ratio: 1.0, minimum: 0, maximum: 343 }
  -> Dimensions: { m: 1, kg: 0, s: -1, ratio: 1.0 }
  -> Bounds: { minimum: 0, maximum: 343 }
```

**Step 4.** When executing an operator, run each `meta` type's operator block on its portion independently.

**Step 5.** Merge results from all blocks back into a flat object.

**Conflict detection:** If two `meta` types claim the same field name, the compiler reports an error at the `meta` declaration site, not at usage. This is enforced when a `meta` block is registered. Symbol-keyed fields can be used to avoid conflicts between third-party libraries.

**Missing fields:** When a value's metadata doesn't include fields for a given `meta` type, that type's `default` value is used. For example, a plain `float32` has no metadata fields, all meta types use their defaults (`Dimensions.default = { m:0, kg:0, s:0, ratio:1.0 }`, `Bounds.default = {}`).

**Operator block absence:** If a `meta` type has no operator block defining a particular operator, the metadata for that type falls back to `default` on the result. This means metadata types only need to define operators where they have meaningful propagation logic.

## Implicit Cast Operators

A raw `number` has no metadata. To allow assignment from `number` to a metadata-bearing `float32`, an implicit cast operator must be explicitly defined on `number`. Without these operators, `const v: Velocity = 10` would be a type error.

The `operator T()` syntax defines an implicit cast that the compiler invokes at assignment boundaries, function call sites, and return statements when the source type doesn't match the target type.

### Defining Cast Operators

```js
// Cast from number -> float32 -> float32 with Dimensions metadata.
// The value passes through unchanged. The target's
// Dimensions metadata is attached based on the destination type.

type NotDimensions<T> = keyof T & keyof Dimensions extends never ? T : never;
primitive float32<T extends NotDimensions<T>> {
	operator float32<Dimensions>() {
		return this;
	}
}

// Cast from number -> float32 -> float32 with Bounds metadata.
// The value passes through, but Bounds.validate() is called at the cast boundary.

type NotBounds<T> = keyof T & keyof Bounds extends never ? T : never;
primitive float32<T extends NotBounds<T>> {
	operator float32<Bounds>() {
		return this;
	}
}
```

Alternative syntax that could be used using ```where```:

```js
type NotDimensions<T> = keyof T & keyof Dimensions extends never ? T : never;
type NotBounds<T> = keyof T & keyof Bounds extends never ? T : never;

primitive float32<T> {
	operator float32<Dimensions>() where NotDimensions<T> {
		return this;
	}

	operator float32<Bounds>() where NotBounds<T> {
		return this;
	}
}
```

Both operators compose for combined types. When the target is `float32<{ m:1, ..., exclusiveMinimum: 0 }>`, the compiler decomposes into Dimensions and Bounds slots and invokes both cast operators. `Bounds.validate()` runs at the cast boundary (elided for compile-time-provable constant literals).

### Cast Operator Invocation Points

The compiler invokes implicit cast operators at these boundaries:

- **Variable declaration:** `const v: Velocity = 10;`
- **Function argument:** `kineticEnergy(80, 10)` where parameters are typed
- **Return statement:** `return 9.80665;` where return type is typed
- **Array element:** `const forces: Newton[] = [10, 20, 30];`
- **Constructor:** `Meter(100)` invokes the same cast as `const m: Meter = 100;`

## Examples

```js
// number -> Velocity via cast operator:
const v: Velocity = 10; // operator float32<Dimensions>() invoked

// number -> Probability via cast operator + validation:
const p: Probability = 0.7;
// operator float32<Bounds>() invoked
// Bounds.validate(0.7, { minimum: 0, maximum: 1 }) -> true

const p2: Probability = 1.5; // Bounds.validate(1.5, { minimum: 0, maximum: 1 }) -> false, throws TypeError("Expected >= 0 and <= 1, got 1.5")

// number -> PositiveMeter via both cast operators:
const h: PositiveMeter = 1.75; // Dimensions cast, Bounds.validate(1.75, { exclusiveMinimum: 0 }) -> true

const h2: PositiveMeter = -3;
// Bounds.validate(-3, { exclusiveMinimum: 0 }) -> false, throws TypeError

// Already-typed value, cast operator doesn't apply:
const m: Meter = 100;
const v1: Velocity = m; // Dimensions.subtype({ m: 1, s: 0 }, { m: 1, s: -1 }), throws TypeError

// Same type assignment: no checks
const v2: Velocity = 10;
const v3: Velocity = v2; // direct, same metadata

// Wider -> narrower: runtime check
const v4: Velocity = 100;
const safe: SafeSpeed = v4;
// Dimensions.subtype: exponents match
// Bounds.subtype: v4 has no bounds (default {}), SafeSpeed has { minimum: 0, maximum: 343 }
//   sup.minimum = 0, sub.minimum = null -> false
// Insert runtime check: Bounds.validate(v4_value, { minimum: 0, maximum: 343 })

// Narrowed by control flow: zero cost
if (v4 >= 0 && v4 <= 343) {
	const safe2: SafeSpeed = v4; // no runtime check
}
```

### Basic Dimensional Algebra

```js
const distance: Meter = 100;
const time: Second = 9.58;
const speed: Velocity = distance / time;
// Dimensions: { m: 1 - 0, kg: 0 - 0, s: 0 - 1, ratio: 1 / 1 } = { m: 1, kg: 0, s: -1, ratio: 1 }
// Bounds: no bounds on either -> default {} -> not present in result
// Result: float32<{ m: 1, kg: 0, s: -1, ratio: 1.0 }> matches Velocity

const mass: Kilogram = 80;
const accel: Acceleration = speed / time;
const force: Newton = mass * accel;
const energy: Joule = force * distance;
const power: Watt = energy / time;
```

### Prefix Scaling

```js
const d1: Kilometer = 5;
const d2: Meter = 300;

// LHS ratio wins. RHS is scaled to LHS unit system.
const totalKm: Kilometer = d1 + d2;
// Value: 5 + 300 * (1.0 / 1000.0) = 5 + 0.3 = 5.3
// totalKm == Kilometer(5.3)

// To get result in meters, put Meter on the LHS:
const totalM: Meter = d2 + d1;
// Value: 300 + 5 * (1000.0 / 1.0) = 300 + 5000 = 5300
// totalM == Meter(5300)

const d3: Centimeter = 50;
const d4: Millimeter = 200;
const sum: Centimeter = d3 + d4;
// Value: 50 + 200 * (0.001 / 0.01) = 50 + 20 = 70
// sum == Centimeter(70)

// Cross-prefix comparison works via normalization:
const a: Kilometer = 1;
const b: Meter = 1000;
console.log(a == b); // true (1 * 1000 == 1000 * 1)
```

### Combined Dimension + Bounds

```js
const width: PositiveMeter = 0.5;
const height: PositiveMeter = 1.75;
const area = width * height;
// Dimensions: { m: 1 } * { m: 1 } = { m: 2, kg: 0, s: 0, ratio: 1 }
// Bounds: boundsFromProducts({ exclusiveMinimum: 0 }, { exclusiveMinimum: 0 })
//   both non-negative, partial propagation -> { exclusiveMinimum: 0 }
// area: float32<{ m: 2, kg: 0, s: 0, ratio: 1.0, exclusiveMinimum: 0 }>
// That's a positive square-meter.
```

### Control Flow Narrowing

```js
function clampToSafe(v: Velocity): SafeSpeed {
	if (v >= 0) {
		// Bounds.narrow({}, ">=", 0) -> { minimum: 0 }
		// v: float32<{ m: 1, kg: 0, s: -1, ratio: 1, minimum: 0 }>

		if (v <= 343) {
			// Bounds.narrow({ minimum: 0 }, "<=", 343) -> { minimum: 0, maximum: 343 }
			// v: float32<{ m: 1, kg: 0, s: -1, ratio: 1, minimum: 0, maximum: 343 }>
			//
			// SafeSpeed.subtype check:
			//   Dimensions: exponents match
			//   Bounds: Bounds.subtype({ minimum:0, maximum:343 }, { minimum:0, maximum:343 }) -> true
			return v; // no cast, no runtime check
		}

		return SafeSpeed(343);
	}

	return SafeSpeed(0);
}
```

### Kinetic Energy

```js
function kineticEnergy(m: Kilogram, v: Velocity): Joule {
	return m * v * v * 0.5;
	// Step 1: m * v
	//   Dimensions: { kg: 1 } * { m: 1, s: -1 } = { m: 1, kg: 1, s: -1 } (Momentum)
	// Step 2: momentum * v
	//   Dimensions: { m: 1, kg: 1, s: -1 } * { m: 1, s: -1 } = { m: 2, kg: 1, s: -2 } (Joule)
	// Step 3: joule * 0.5
	//   Dimensions: scalar multiply -> { m: 2, kg: 1, s: -2 } preserved (Joule)
	// return type matches Joule
}

const ke: Joule = kineticEnergy(Kilogram(80), Velocity(10));
// ke == Joule(4000)
```

### Gravitational Potential Energy

```js
function potentialEnergy(m: Kilogram, h: PositiveMeter): Joule {
	const g: Acceleration = 9.80665;
	return m * g * h;
	// Dimensions: { kg: 1 } * { m: 1, s: -2 } * { m: 1 } = { m: 2, kg: 1, s: -2 } Joule
	// Bounds: {} * {} * { exclusiveMinimum: 0 } -> { exclusiveMinimum: 0 } propagated
	// Result has Bounds{ exclusiveMinimum: 0 }, energy is positive.
	// Joule has no Bounds requirement -> extra bounds are fine (subtype).
}
```

### Pressure

```js
type SquareMeter = float32<{ m: 2, kg: 0, s: 0, ratio: 1.0 }>;

function pressure(force: Newton, area: SquareMeter): Pascal {
	return force / area;
	// Dimensions: { m: 1, kg: 1, s: -2 } / { m: 2 } = { m: -1, kg: 1, s: -2 } Pascal
}
```

### Bounds Arithmetic

```js
const prob: Probability = 0.7;
const prob2: Probability = 0.2;
const sum = prob + prob2;
// Bounds: { minimum: 0, maximum: 1 } + { minimum: 0, maximum: 1 } = { minimum: 0, maximum: 2 }
// sum: float32<{ minimum: 0, maximum: 2 }>

// const bad: Probability = sum;
// Bounds.subtype({ minimum: 0, maximum: 2 }, { minimum: 0, maximum: 1 }) -> false (max 2 > max 1)

if (sum <= 1) {
	// Bounds.narrow({ minimum: 0, maximum: 2 }, "<=", 1) -> { minimum: 0, maximum: 1 }
	const safe: Probability = sum; // proven by narrowing
}
```

### Dimensional Errors

```js
// distance + time;
// Dimensions block: addition requires same dimension
//    { m: 1, kg: 0, s: 0 } != { m: 0, kg: 0, s: 1 }
//    Error: cannot add Meter (m) to Second (s)

// speed + force;
// Dimensions block: { m: 1, kg: 0, s: -1 } != { m: 1, kg: 1, s: -2 }
//    Error: cannot add Velocity (m/s) to Newton (kg*m/s**2)

// const bad: Velocity = distance;
// Dimensions.subtype: { m: 1, s: 0 } != { m: 1, s: -1 }
//    Error: Meter is not assignable to Velocity
```

## Metadata key scoping with Symbols

For library-quality code where name collisions are a concern, metadata keys can be symbols:

```js
const si = Object.freeze({
	m: Symbol("SI.length"),
	kg: Symbol("SI.mass"),
	s: Symbol("SI.time"),
});

type Dimensions = { [si.m]: int32, [si.kg]: int32, [si.s]: int32, ratio: float32 };
```

## Decorators

A common use case for decorators is validation. Consider the example below where validation is moved from decorators to metadata.

```js
// Decorator approach
class User {
	@Min(0) @Max(150)
	age: number;

	@Pattern(/^[^@]+@[^@]+$/)
	email: string;
}
```

```js
// Metadata approach
class User {
	age: number<{ minimum: 0, maximum: 150 }>;
	email: string<{ pattern: /^[^@]+@[^@]+$/ }>;
}
```

The latter allows compile time validation as well as runtime validation. The following examples separate validation from decorators.

### Validate example

The following ```@validate``` decorator could still be created, but it would just be doing redundant work.

```js
type NumberBounds = {
	minimum?: float64,
	maximum?: float64,
	exclusiveMinimum?: float64,
	exclusiveMaximum?: float64
};

type StringBounds = {
	pattern?: RegExp,
	minLength?: uint32,
	maxLength?: uint32
};

const validatorsKey = Symbol('validators');

type SerializeData<T> = {
	name: string,
	constraint: T,
	meta: MetaProtocol<T>
};

partial class Metadata {
	[validatorsKey]: [].<SerializeData<NumberBounds> | SerializeData<StringBounds>> = [];
}

function validate<B: NumberBounds, TClass>({ name, metadata }: ClassFieldDecorator<number<B>, TClass>) {
	metadata[validatorsKey].push({ name, constraint: B, meta: Bounds });
}

function validate<S: StringBounds, TClass>({ name, metadata }: ClassFieldDecorator<string<S>, TClass>) {
	metadata[validatorsKey].push({ name, constraint: S, meta: StringBounds });
}

function validateInstance<T>(instance: T): boolean {
	const entries = Reflect.getMetadata<T>()[validatorsKey];
	if (!entries) return true;
	for (const { name, constraint, meta } of entries) {
		if (!meta.validate(instance[name], constraint)) return false;
	}
	return true;
}

class User {
	@validate
	age: number<{ minimum: 0, maximum: 150 }>;
	@validate
	email: string<{ pattern: /^[^@]+@[^@]+$/ }>;
}

const user = new User();
user.age = 25;
user.email = "alice@example.com";
validateInstance(user); // true

user.age = -5; // Note: This would throw at compile time. If the value was dynamic then it would throw at runtime assuming the meta block defines a validate
validateInstance(user); // false, minimum: 0 violated

user.age = 25;
user.email = "not-an-email"; // Note: This would throw at compile time. If the value was dyanmic then it would throw at runtime assuming the meta block defines a validate
validateInstance(user); // false, pattern violated
```

### JSON Serialization with field name overrides

```js
type StringBounds = {
	pattern?: RegExp,
	minLength?: uint32,
	maxLength?: uint32
};

const schemaKey = Symbol('schema');

type SerializeData = {
	name: string,
	wireName: string
};

partial class Metadata {
	[schemaKey]: [].<SerializeData> = [];
}

// @field() — registers a field for serialization with an optional wire name
function field<T, TClass>(
	{ name, metadata }: ClassFieldDecorator<T, TClass>,
) {
	metadata[schemaKey].push({ name, wireName: name });
}
function field<T, TClass>(
	wireName: string,
	{ name, metadata }: ClassFieldDecorator<T, TClass>,
) {
	metadata[schemaKey].push({ name, wireName });
}

function serialize<T>(instance: T): Record<string, any> {
	const result: Record<string, any> = {};
	for (const { name, wireName } of Reflect.getMetadata<T>()[schemaKey]) {
		result[wireName] = instance[name];
	}
	return result;
}

function deserialize<T>(cls: { new(): T }, data: Record<string, any>): T {
	const instance = new cls();
	for (const { name, wireName } of Reflect.getMetadata<T>()[schemaKey]) {
		instance[name] = data[wireName]; // implicit cast → triggers meta validate
	}
	return instance;
}

class UserResponse {
	@field
	id: uint64;
	@field('user_name')
	userName: string<{ minLength: 1, maxLength: 100 }>;
	@field('email_address')
	email: string<{ pattern: /^[^@]+@[^@]+$/ }>;
	@field
	age: number<{ minimum: 0, maximum: 150 }>;
}

// Incoming JSON:
// { "id": 42, "user_name": "alice", "email_address": "a@b.com", "age": 30 }
const user = deserialize(UserResponse, json);
// Assignment to `email` triggers: string<StringBounds>.validate("a@b.com", { pattern: ... })
// Assignment to `age` triggers: number<NumberBounds>.validate(30, { minimum: 0, maximum: 150 })

serialize(user);
// { "id": 42, "user_name": "alice", "email_address": "a@b.com", "age": 30 }
```

### Typed API routing

```js
type PathParam = { param: true };
type QueryParam = {
	param: true,
	query: true
};

const routeKey = Symbol('route');

type Route = {
	method: 'POST' | 'GET',
	path: string,
	handler: string
};

partial class Metadata {
	[routeKey]: [].<Route> = [];
}

function get<T extends (...args: any) => any, TClass>(
	path: string,
	{ name, metadata }: ClassMethodDecorator<T, TClass>
) {
	metadata[routeKey].push({ method: 'GET', path, handler: name });
}

function post<T extends (...args: any) => any, TClass>(
	path: string,
	{ name, metadata }: ClassMethodDecorator<T, TClass>
) {
	metadata[routeKey].push({ method: 'POST', path, handler: name });
}

class EventController {
	@get('/events')
	list(
		limit: uint32<{ minimum: 1, maximum: 100 }> = 20,
		offset: uint32<{ minimum: 0 }> = 0
	): Event[] {
		return db.events.slice(offset, offset + limit);
	}

	@get('/events/:id')
	getById(
		id: uint64
	): Event {
		return db.events.find(e => e.id === id) ?? throw new HttpError(404);
	}

	@post('/events')
	create(
		body: EventCreate
	): Event {
		return db.events.create(body);
	}
}

class EventCreate {
	title: string<{ minLength: 1, maxLength: 200 }>;
	date: string<{ pattern: /^\\d{4}-\\d{2}-\\d{2}$/ }>;
	capacity: uint32<{ minimum: 1, maximum: 10000 }>;
}
```

### Database model with column mapping and generated fields

```js
const tableKey = Symbol('table');
const columnKey = Symbol('columns');

type Column = {
	field: string,
	column: string
};

partial class Metadata {
	[tableKey]: string = '';
	[columnKey]: [].<Column> = [];
}

function table<T>(
	name: string,
	{ metadata }: ClassDecorator<T>
) {
	metadata[tableKey] = name;
}

function column<T, TClass>(
	{ name, metadata }: ClassFieldDecorator<T, TClass>
) {
	metadata[columnKey].push({ field: name, column: name });
}
function column<T, TClass>(
	column: string,
	{ name, metadata }: ClassFieldDecorator<T, TClass>
) {
	metadata[columnKey].push({ field: name, column });
}

@table('sensors')
class SensorReading {
	@column
	id: uint64;
	@column('sensor_id')
	sensorId: uint32;
	@column
	temperature: float32<{ minimum: -273.15 }>; // can't go below absolute zero
	@column
	humidity: float32<{ minimum: 0, maximum: 100 }>; // percentage
	@column('recorded_at')
	recordedAt: string<{ pattern: /^\\d{4}-\\d{2}-\\d{2}T/ }>;
}

// ORM builds:
//   SELECT id, sensor_id, temperature, humidity, recorded_at
//   FROM sensors
//   WHERE ...
//
// Row hydration assigns each column value to the typed field.
// temperature = row['temperature'] triggers:
//   float32<{ minimum: -273.15 }>.validate(value, { minimum: -273.15 })
// A corrupted row with temperature = -300 fails validation at the ORM boundary, not deep in business logic.
```

## vec3 Example

This is more in-depth and covers function metadata propagation. This is the ```Dimensions``` setup for 3D which is a more practical example.

```js
// Generic 3-vector parameterized by Dimensions
// All three components share the same dimensional metadata.
type vec3<D: Dimensions> = vector<float32<D>, 3>;

// Concrete physics vector types
type Position = vec3<{ m: 1, kg: 0, s: 0, ratio: 1.0 }>; // meters
type Velocity3 = vec3<{ m: 1, kg: 0, s: -1, ratio: 1.0 }>; // m/s
type Acceleration3 = vec3<{ m: 1, kg: 0, s: -2, ratio: 1.0 }>; // m/s**2
type Force3 = vec3<{ m: 1, kg: 1, s: -2, ratio: 1.0 }>; // N
type Momentum3 = vec3<{ m: 1, kg: 1, s: -1, ratio: 1.0 }>; // kg*m/s
type Torque3 = vec3<{ m: 2, kg: 1, s: -2, ratio: 1.0 }>; // N*m
type AngularVel3 = vec3<{ m: 0, kg: 0, s: -1, ratio: 1.0 }>; // rad/s
type Unitless3 = vec3<{ m: 0, kg: 0, s: 0, ratio: 1.0 }>; // direction, etc.

// Scalar types (from scalar spec, repeated for context)
type Meter = float32<{ m: 1, kg: 0, s: 0, ratio: 1.0 }>;
type Kilogram = float32<{ m: 0, kg: 1, s: 0, ratio: 1.0 }>;
type Second = float32<{ m: 0, kg: 0, s: 1, ratio: 1.0 }>;
type Velocity = float32<{ m: 1, kg: 0, s: -1, ratio: 1.0 }>;
type Newton = float32<{ m: 1, kg: 1, s: -2, ratio: 1.0 }>;
type Joule = float32<{ m: 2, kg: 1, s: -2, ratio: 1.0 }>;
type SquareMeter = float32<{ m: 2, kg: 0, s: 0, ratio: 1.0 }>;

// Note: vector.<T, N> has built-in element-wise operators on raw values.
// So these operators skip redeclaring operator bodies.

primitive vector<float32<D: Dimensions>, 3> {

	// Same-dimension add/subtract

	operator+<D2: Dimensions>(rhs: vec3<D2>): vec3<D>
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (feq(D.ratio, D2.ratio)) {
			return this + rhs;
		}
		return this + rhs * (D2.ratio / D.ratio);
	}

	operator-<D2: Dimensions>(rhs: vec3<D2>): vec3<D>
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (feq(D.ratio, D2.ratio)) {
			return this - rhs;
		}
		return this - rhs * (D2.ratio / D.ratio);
	}

	// Scalar multiply/divide
	// vec3<D> * float32 = vec3<D> (scale a vector)

	operator*(rhs: float32): vec3<D>;

	operator/(rhs: float32): vec3<D>;

	// Dimensioned scalar multiply/divide
	// vec3<D> * float32<D2> = vec3<D*D2>
	// e.g. Velocity3 * Second = Position

	operator*<D2: Dimensions>(rhs: float32<D2>): vec3<{
		m: D.m + D2.m,
		kg: D.kg + D2.kg,
		s: D.s + D2.s,
		ratio: D.ratio * D2.ratio
	}>;

	operator/<D2: Dimensions>(rhs: float32<D2>): vec3<{
		m: D.m - D2.m,
		kg: D.kg - D2.kg,
		s: D.s - D2.s,
		ratio: D.ratio / D2.ratio
	}>;

	// Unary

	operator-(): vec3<D>;

	// Compound assignment

	operator+=<D2: Dimensions>(rhs: vec3<D2>): vec3<D>
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (!feq(D.ratio, D2.ratio)) {
			this += rhs * (D2.ratio / D.ratio);
		} else {
			this += rhs;
		}
		return this;
	}

	operator-=<D2: Dimensions>(rhs: vec3<D2>): vec3<D>
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (!feq(D.ratio, D2.ratio)) {
			this -= rhs * (D2.ratio / D.ratio);
		} else {
			this -= rhs;
		}
		return this;
	}

	operator*=(rhs: float32): vec3<D>;
	//{
	//	this *= rhs;
	//	return this;
	//}

	operator/=(rhs: float32): vec3<D>;
	//{
	//	this /= rhs;
	//	return this;
	//}

	// Comparison
	// Per-component equality after normalization.
	// Just an example as float equality requires special handling usually

	operator==<D2: Dimensions>(rhs: vec3<D2>): boolean
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		if (feq(D.ratio, D2.ratio)) {
			return this == rhs;
		}
		return this * D.ratio == rhs * D2.ratio;
	}

	operator!=<D2: Dimensions>(rhs: vec3<D2>): boolean
		where D.m == D2.m && D.kg == D2.kg && D.s == D2.s
	{
		return !(this == rhs);
	}
}



// Dot product
// e.g. dot(Force3, Position) -> Joule (N*m = J)
//      dot(Velocity3, Velocity3) -> m**2/s**2 (speed**2)
function dot<D: Dimensions, D2: Dimensions>(
	a: vec3<D>,
	b: vec3<D2>
): float32<{
	m: D.m + D2.m,
	kg: D.kg + D2.kg,
	s: D.s + D2.s,
	ratio: D.ratio * D2.ratio
}> {
	return a[0] * b[0] + a[1] * b[1] + a[2] * b[2];
	// Each a[i]*b[i] has Dimensions { D.m+D2.m, D.kg+D2.kg, D.s+D2.s }.
	// All three products share the same dimension, so + is valid.
}

// Cross product
// e.g. cross(Position, Force3) -> Torque3 (m*N = N*m)
//      cross(Velocity3, Velocity3) -> vec3<m**2/s**2>
function cross<D: Dimensions, D2: Dimensions>(
	a: vec3<D>,
	b: vec3<D2>,
): vec3<{
	m: D.m + D2.m,
	kg: D.kg + D2.kg,
	s: D.s + D2.s,
	ratio: D.ratio * D2.ratio
}> {
	return vec3(
		a[1] * b[2] - a[2] * b[1],
		a[2] * b[0] - a[0] * b[2],
		a[0] * b[1] - a[1] * b[0]
	);
}

// Metadata aware math functions

// Math.sqrt need metadata overloads so that dimensional information propagates through magnitude computations.
// sqrt(m**2) = m, Valid (2/2 = 1)
// sqrt(m**2/s**2) = m/s, Valid (2/2 = 1, -2/2 = -1)
// sqrt(m) = ???, Invalid (1/2 not int32)
function Math.sqrt<D: Dimensions>(x: float32<D>): float32<{
	m: D.m / 2,
	kg: D.kg / 2,
	s: D.s / 2,
	ratio: Math.sqrt(D.ratio)
}>
where D.m % 2 == 0 && D.kg % 2 == 0 && D.s % 2 == 0;

// Propagate Dimensions metadata
function Math.hypot<D: Dimensions>(...args: float32<D>[]): float32<D>;

// Magnitude (vector length)
// Uses Math.hypot, which preserves Dimensions.
// magnitude(Position) -> Meter
// magnitude(Force3) -> Newton
function magnitude<D: Dimensions>(v: vec3<D>): float32<D> {
	return Math.hypot(v[0], v[1], v[2]);
}

// Alternative via dot + sqrt (equivalent):
//function magnitude<D: Dimensions>(v: vec3<D>): float32<D> {
//	return Math.sqrt(dot(v, v));
//	// dot(v, v): float32<{ m: 2*D.m, kg: 2*D.kg, s: 2*D.s, ... }>
//	// Math.sqrt:  halves exponents -> float32<D>
//}

// Squared magnitude
// Returns D*D exponents.
function magnitudeSq<D: Dimensions>(v: vec3<D>): float32<{
	m: D.m + D.m,
	kg: D.kg + D.kg,
	s: D.s + D.s,
	ratio: D.ratio * D.ratio
}> {
	return dot(v, v);
}

// Normalize (unit vector)
// Divides out the dimension, returning a Unitless3.
// magnitude is float32<D>, dividing vec3<D> / float32<D>
// yields vec3<D-D> = vec3<dimensionless>.

function normalize<D: Dimensions>(v: vec3<D>): Unitless3 {
	return v / magnitude(v);
	// vec3<D> / float32<D>
	// Dimensions: { D.m - D.m, D.kg - D.kg, D.s - D.s } = { 0, 0, 0 }
	//   Unitless3
}

// Distance between two positions

function distance<D: Dimensions>(a: vec3<D>, b: vec3<D>): float32<D> {
	return magnitude(a - b);
	// a - b: vec3<D> (same dimension)
	// magnitude: float32<D>
}
```
Usage

```js
// Basic vector arithmetic
const origin: Position = vec3(0, 0, 0);
const pos: Position = vec3(3, 4, 0);
const offset: Position = vec3(1, 0, 5);

const moved = pos + offset; // Position(4, 4, 5)

const displaced = pos - origin; // Position(3, 4, 0)

const scaled = pos * 2; // Position(6, 8, 0)
```

Newton's second law: F = m*a

```js
const mass: Kilogram = 10;
const acceleration: Acceleration3 = vec3(0, -9.80665, 0);
const gravity = acceleration * mass; // Force3(0, -98.0665, 0)
// Dimensions: { m: 1, s: -2 } + { kg: 1 } = { m: 1, kg: 1, s: -2 }
// Note: Could overload so mass * acceleration works
```

Euler integration

```js
let position: Position = vec3(0, 100, 0);
let velocity: Velocity3 = vec3(10, 0, 0);
const dt: Second = 1 / 60;

// One integration step:
velocity += acceleration * dt;
// accel * dt -> vec3<{ m: 1, s: -2 }> * float32<{ s: 1 }> -> vec3<{ m: 1, s: -1 }> = Velocity3
// velocity += Velocity3 -> same dimension

position += velocity * dt;
// velocity * dt: Velocity3 * Second
//   = vec3<{ m: 1, s: -1 }> * float32<{ s: 1 }>
//   = vec3<{ m: 1, s: 0 }> = Position
// position += Position -> same dimension

// Full simulation loop:
function simulate(
	pos: Position,
	vel: Velocity3,
	acc: Acceleration3,
	dt: Second,
	steps: int32,
): Position {
	for (let i: int32 = 0; i < steps; ++i) {
		vel += acc * dt;
		pos += vel * dt;
	}
	return pos;
}
```

Kinetic energy via dot product

```js
function kineticEnergy(m: Kilogram, v: Velocity3): Joule {
	return dot(v, v) * m * 0.5;
	// dot(v, v): float32<{ m: 2, s: -2 }>, speed squared
	// * Kilogram: float32<{ m: 2, kg: 1, s: -2 }>, Joule
	// * 0.5: scalar multiply, still Joule
}

const ke: Joule = kineticEnergy(10, vec3(3, 4, 0));
// dot = 9 + 16 + 0 = 25  (m**2/s**2)
// * 10 kg * 0.5 = 125 J
```

Work: W = F * d

```js
function work(f: Force3, d: Position): Joule {
	return dot(f, d);
	// dot(Force3, Position):
	//   Dimensions: { m: 1, kg: 1, s: -2 } + { m: 1 } = { m: 2, kg: 1, s: -2 }
	//   = Joule
}

const w: Joule = work(vec3(10, 0, 0), vec3(5, 0, 0));
// dot = 50 + 0 + 0 = 50 J
```

Torque: τ = r * F

```js
const leverArm: Position = vec3(0, 0, 2);
const appliedForce: Force3 = vec3(0, 10, 0);
const torque: Torque3 = cross(leverArm, appliedForce);
// cross(Position, Force3):
//   Dimensions: { m: 1 } + { m: 1, kg: 1, s: -2 } = { m: 2, kg: 1, s: -2 }
//   = Torque3 (N*m)
// torque == vec3(0*0 - 2*10, 2*0 - 0*0, 0*10 - 0*0)
//        == vec3(-20, 0, 0) as Torque3
```

Magnitude and normalization

```js
const dist: Meter = magnitude(pos);
// magnitude(Position) -> float32<Meter>
// hypot(3, 4, 0) = 5

const dir: Unitless3 = normalize(pos);
// Position / Meter -> Unitless3
// vec3(3/5, 4/5, 0) = vec3(0.6, 0.8, 0)

const speed: Velocity = magnitude(vel);
// magnitude(Velocity3) -> float32<Velocity>
```

Distance and squared distance

```js
const a: Position = vec3(1, 2, 3);
const b: Position = vec3(4, 6, 3);

const d: Meter = distance(a, b); // magnitude(vec3(-3, -4, 0)) = 5 meters

const dSq: SquareMeter = magnitudeSq(a - b);
// dot(vec3(-3,-4,0), vec3(-3,-4,0)) = 9 + 16 + 0 = 25 m**2

// Squared distance avoids sqrt, useful for comparisons:
const threshold: SquareMeter = 100;  // 10**2 m**2
if (dSq < threshold) {
	// within 10 meters
}
```

Projectile motion

```js
function projectilePosition(
	v0: Velocity3,
	t: Second,
): Position {
	const g: Acceleration3 = vec3(0, -9.80665, 0);
	return v0 * t + g * t * t * 0.5; // TODO: Overload **
	// v0 * t: Velocity3 * Second = Position
	// g * t: Acceleration3 * Second = Velocity3
	// Velocity3 * t: Velocity3 * Second = Position
	// Position * 0.5: scalar -> Position
	// Position + Position -> Position
}

const landingPos: Position = projectilePosition(
	vec3(20, 30, 0), // launch velocity
	3, // time
);
// v0*t = vec3(60, 90, 0)
// g*t*t*0.5 = vec3(0, -9.80665*9*0.5, 0) = vec3(0, -44.13, 0)
// sum = vec3(60, 45.87, 0) as Position
```

Reflect a velocity off a surface

```js
function reflect(v: Velocity3, normal: Unitless3): Velocity3 {
	return v - normal * dot(v, normal) * 2;
	// dot(Velocity3, Unitless3):
	//   Dimensions: { m: 1, s: -1 } + { 0,0,0 } = { m: 1, s: -1 }
	//   = float32<Velocity> (scalar speed along normal)
	//
	// normal * Velocity: Unitless3 * float32<Velocity>
	//   = vec3<{ 0+m: 1, s: -1 }> = Velocity3
	//
	// Velocity3 * 2: scalar -> Velocity3
	// v - Velocity3: same dimension
}

const incoming: Velocity3 = vec3(1, -1, 0);
const wallNormal: Unitless3 = vec3(0, 1, 0);
const reflected: Velocity3 = reflect(incoming, wallNormal);
// dot(v, n) = 0 + (-1) + 0 = -1
// n * (-1) * 2 = vec3(0, -2, 0)
// v - vec3(0, -2, 0) = vec3(1, 1, 0) (reflected upward)
```

Angular velocity: v = ω * r

```js
const omega: AngularVel3 = vec3(0, 0, 5); // 5 rad/s around z-axis
const radius: Position = vec3(2, 0, 0); // 2m from axis

const tangentialVel: Velocity3 = cross(omega, radius);
// cross(AngularVel3, Position):
//   Dimensions: { s: -1 } + { m: 1 } = { m: 1, s: -1 }
//   = Velocity3
// cross = vec3(0*0-5*0, 5*2-0*0, 0*0-0*0) = vec3(0, 10, 0) m/s
```

Gravitational force between two masses

```js
// Gravitational constant G ≈ 6.674e-11 m³/(kg*s**2)
// G has Dimensions { m: 3, kg: -1, s: -2 }
type GravConst = float32<{ m: 3, kg: -1, s: -2, ratio: 1.0 }>;
const G: GravConst = 6.674e-11;

function gravitationalForce(
	m1: Kilogram,
	m2: Kilogram,
	p1: Position,
	p2: Position,
): Force3 {
	const r = p2 - p1; // Position
	const dSq = magnitudeSq(r); // SquareMeter
	const dir = normalize(r); // Unitless3
	const fMag = G * m1 * m2 / dSq;
	// G * kg * kg / m**2
	// Dimensions: { m: 3, kg: -1, s: -2 } + { kg: 1 } + { kg: 1 } - { m: 2 }
	//    = { m: 1, kg: 1, s: -2 } = Newton

	return dir * fMag;
	// Unitless3 * Newton = Force3
}
```

Dimensional errors with vectors

```js
// position + velocity;
//   where clause: { m: 1, s: 0 } != { m: 1, s: -1 }, Invalid
// Error: cannot add Position (m) to Velocity3 (m/s)

// const bad: Force3 = accel;
//   vec3<Acceleration> -> vec3<Force>
// Dimensions.subtype: kg: 0 != kg: 1, Invalid

// cross(position, vel);
// This is allowed, cross multiplies dimensions:
// Dimensions: { m: 1, s: 0 } + { m: 1, s: -1 } = { m: 2, s: -1 }
// Result: vec3<{ m: 2, s: -1 }>, Valid, not a named type

// dot(position, vel);
// Also allowed, dot multiplies dimensions:
// float32<{ m: 2, s: -1 }>, Valid, unnamed
```


Math.sqrt compile errors

```js
const speedSq: SquareMeter = 25; // m**2
const spd: Meter = Math.sqrt(speedSq);
// D = { m: 2, s: 0 }. m%2==0, kg%2==0, s%2==0
// Result: float32<{ m: 1, s: 0 }> = Meter, Valid

// const bad = Math.sqrt(Meter(9));
// D = { m: 1, s: 0 }. m%2 == 1 != 0, Invalid
// where clause fails: cannot sqrt an odd-exponent dimension.
// Compile error: sqrt requires even dimensional exponents.
```

Note: That function blocks that define metadata operations follow the same merge rules as operators.

## TODO Items

### Metadata on reference types

I haven't put any thought into generalizing this to classes.

### Wouldn't a compile-time SMT-lite solver be potentially very expensive to run?

For practical cases a simple memoization for each type or pair of types negates most of the cost. It's possible to engineer situations where a timeout is required for compile-time/editor calculations.
