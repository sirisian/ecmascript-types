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

1. Metadata types - plain type declarations whose fields attach to primitives via ```.<{...}>``` syntax.
2. Meta protocols - meta blocks that teach the compiler the semantics of a metadata type (subtyping, validation, narrowing).
3. Primitive operator blocks - primitive T<M: MetaType> blocks that define how operators transform metadata.

### Metadata Types and Protocols

The metadata type is just a typed object defining the metadata parts and their individual types.

```js
type Metadata = { part: int32 };
const a: float32.<{ part: 1 }> = 1;
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
	// Called by the compiler when it encounters a comparison in an if/while/ternary condition.
	// The compiler handles operator negation for the else branch:
	// >=  to  <
	// >   to  <=
	// ==  to  !=
	// Returns only meaningful fields, absence means unconstrained.
	narrow?(current: T, op: string, value: primitive): T;

	// Optional: linear conversion between two subtype-compatible parameterizations of this meta type. Returns the factor the value is multiplied by when converting `from` to `to`. Applied at assignment boundaries, including operator parameters (see composition rules).
	conversionFactor?(from: T, to: T): float64;

	// Optional: adjust this meta type's constraint when the underlying value is scaled by another meta type's conversion, keeping value-space metadata like bounds consistent. When absent while a conversion occurs, this meta type's portion falls back to `default`.
	rescale?(constraint: T, factor: float64): T;

	// Optional: map a value onto the representation this constraint requires, such as rounding a decimal to a fixed scale. Applied at assignment, argument, and return boundaries after `subtype` passes and after any `conversionFactor` scaling, so intermediate results within an expression keep full precision.
	quantize?(value: primitive, constraint: T): primitive;

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
	conversionFactor?(from: T, to: T): float64;
	rescale?(constraint: T, factor: float64): T;
	quantize?(value: any, constraint: T): any;
	describe?(constraint: T): string;
}
```

**Naming the base.** The ```primitive``` in those signatures is the type being parameterized - the meta type's *base* - and it reaches the hooks but not the constraint shape, because a shape is a free-standing ```type``` alias that any code may reference and a keyword bound to nothing would be meaningless there. A meta type whose fields live in the base's own value space therefore **declares one type parameter, which is bound to the base at each parameterization**:

```js
type NumberBounds<T: Ordered.<T>> = { bounds?: RangeBounds.<T>, nonZero?: boolean };

meta NumberBounds<T: Ordered.<T>> {
	default: NumberBounds.<T>;
	subtype(sub: NumberBounds.<T>, sup: NumberBounds.<T>): boolean;
	validate?(value: T, constraint: NumberBounds.<T>): boolean;
	// ... and so on for each hook, with `T` where `primitive` stood
}
```

The parameter is not new power; it is a **name** for what ```primitive``` already means, usable where a keyword cannot reach. Three things follow. It is supplied by position and never written - ```uint8.<{ bounds: 1..=6 }>``` binds ```T``` to ```uint8``` because that is the type being parameterized - so nothing at a use site changes. It must be bounded, and the shape says how tightly: ```RangeBounds.<T>``` requires ```T: Ordered.<T>```, so a base that carries no ordering is rejected at the declaration rather than at some later use. And claiming is unchanged, being a property of the declaration rather than of an instantiation: ```NumberBounds``` claims ```bounds``` and ```nonZero``` once and globally, whatever ```T``` turns out to be.

A meta type that needs the base only inside a hook declares no parameter and keeps ```primitive```, which is every other meta type in this document: ```Dimensions``` carries exponents and a ratio, and ```DecimalContext``` a scale and a rounding mode, none of them in the base's value space. The generic form is what a meta type opts into by declaring a parameter, and the hooks of one that does are more precise for it, since ```validate(value: T, ...)``` says which primitive where ```validate(value: primitive, ...)``` does not.

Two examples will be used, a dimensions (units of measure) and bounds (minimum and maximum constraints from JSON Schema).

**Metadata Type**: Dimensions  
Tracks SI base dimensions as integer exponents plus a rational scale factor for unit prefixes. The ratio field encodes the relationship between a prefixed unit and its base SI unit (e.g., ```ratio: 1000``` for kilometers, ```ratio: 1/1000``` for millimeters).

```js
type Dimensions = {
	m: int32, // length exponent
	kg: int32, // mass exponent
	s: int32, // time exponent
	A: int32, // electric current exponent
	K: int32, // temperature exponent
	mol: int32, // amount of substance exponent
	cd: int32, // luminous intensity exponent
	ratio: rational, // exact scale factor relative to base SI (1 = base)
};

meta Dimensions {
	default = { m: 0, kg: 0, s: 0, A: 0, K: 0, mol: 0, cd: 0, ratio: 1 };

	// Dimensional subtyping: exact match on exponents. Exact comparisons keep the relation transitive; epsilon comparisons are reserved for runtime value checks.
	// Ratio can differ - a Kilometer is assignable to a Meter-typed slot; the value is scaled at the assignment via conversionFactor().
	subtype(sub: Dimensions, sup: Dimensions): boolean {
		return sameDimensions(sub, sup);
	}

	// Unit conversion within a dimension is linear: multiply the value by
	// the ratio of ratios. With rational ratios the factor is exact.
	conversionFactor(from: Dimensions, to: Dimensions): float64 {
		return from.ratio / to.ratio;
	}

	// No validate - dimensions constrain type compatibility, not value ranges.
	// Any float32 value is valid for any dimension.

	// No narrow - comparison operators don't affect dimensions.

	describe(constraint: Dimensions): string {
		const parts: [].<string> = [];
		const exponents = [
			['m', constraint.m],
			['kg', constraint.kg],
			['s', constraint.s],
			['A', constraint.A],
			['K', constraint.K],
			['mol', constraint.mol],
			['cd', constraint.cd]
		];
		for (const [name, exponent] of exponents) {
			if (exponent != 0)
				parts.push(exponent == 1 ? name : `${name}^${exponent}`);
		}
		const dim = parts.join('·') || 'dimensionless';
		return constraint.ratio != 1 ? `${constraint.ratio}* ${dim}` : dim;
	}
}
```

Helper functions shared by the ```meta``` block, ```where``` clauses, and return type annotations. With seven base dimensions these keep the operator blocks readable:

```js
// Same physical dimension: exact match on all seven exponents. Ratio is
// deliberately excluded; it's a unit scale within the dimension.
function sameDimensions(a: Dimensions, b: Dimensions): boolean {
	return a.m == b.m && a.kg == b.kg && a.s == b.s
		&& a.A == b.A && a.K == b.K && a.mol == b.mol && a.cd == b.cd;
}

// Dimension algebra for multiplication: exponents add, ratios multiply exactly.
function multiplyDimensions(a: Dimensions, b: Dimensions): Dimensions {
	return {
		m: a.m + b.m, kg: a.kg + b.kg, s: a.s + b.s, A: a.A + b.A,
		K: a.K + b.K, mol: a.mol + b.mol, cd: a.cd + b.cd,
		ratio: a.ratio * b.ratio,
	};
}

// Dimension algebra for division: exponents subtract, ratios divide exactly.
function divideDimensions(a: Dimensions, b: Dimensions): Dimensions {
	return {
		m: a.m - b.m, kg: a.kg - b.kg, s: a.s - b.s, A: a.A - b.A,
		K: a.K - b.K, mol: a.mol - b.mol, cd: a.cd - b.cd,
		ratio: a.ratio / b.ratio,
	};
}

// All exponents even, for square roots.
function evenDimensions(d: Dimensions): boolean {
	return d.m % 2 == 0 && d.kg % 2 == 0 && d.s % 2 == 0
		&& d.A % 2 == 0 && d.K % 2 == 0 && d.mol % 2 == 0 && d.cd % 2 == 0;
}

// Halved exponents, for square roots. The ratio must be a perfect square
// of a rational for the result to stay exact; convert to base units
// (ratio 1) first otherwise.
function halveDimensions(d: Dimensions): Dimensions {
	return {
		m: d.m / 2, kg: d.kg / 2, s: d.s / 2, A: d.A / 2,
		K: d.K / 2, mol: d.mol / 2, cd: d.cd / 2,
		ratio: Math.sqrt(d.ratio),
	};
}
```

**Metadata Type**: NumberBounds  
A range constraining the value, and a flag for the one constraint a range cannot express. `bounds` is the range a value must fall within; `nonZero` excludes zero from a range that otherwise contains it, which is what a divisor needs. An absent key means the meta type's default, an unconstrained bound.

The range under `bounds` is a **value**, one of the four shapes of the [ranges](ranges.md) extension, and not an arbitrary `RangeBounds` implementor: a class of a program's own has no structural comparison, so interning would have no answer for it and an expansion artifact no way to carry it. Its endpoints are compile-time constants, and each is taken at the type the metadata parameterizes rather than at the type its literal would otherwise have - `uint8.<{ bounds: 1..=6 }>` has `uint8` endpoints - which is what makes the same spelling in two modules the same type, since a `uint8` 1 and a Number 1 are never SameValue under this proposal.

There is no bare-range metadata argument: `uint8.<1..=6>` is not a type, and the range is written under the key that says what it means. Dispatch in the metadata system is by claimed key, and a bare range carries no key to route by; a grammar-level desugaring would need no dispatch but would give one type two spellings, which is the thing `bounds` was adopted to stop.

```js
type NumberBounds<T: Ordered.<T>> = {
	bounds?: RangeBounds.<T>,
	nonZero?: boolean,
};

// A constraint excludes zero when it says so or when its bounds already do.
// This is the rule that keeps `uint32.<{ bounds: 1.. }>` a Divisor without
// its having to repeat itself.
function excludesZero<T: Ordered.<T>>(c: NumberBounds.<T>): boolean {
	return c.nonZero || !c.bounds.contains(0);
}

meta NumberBounds<T: Ordered.<T>> {
	// The default is total: every key has a value, so no hook tests for absence.
	default = { bounds: .., nonZero: false };

	subtype(sub: NumberBounds.<T>, sup: NumberBounds.<T>): boolean {
		// A range that cannot contain zero is non-zero whether or not it says so.
		if (excludesZero(sup) && !excludesZero(sub)) return false;
		return sup.bounds.contains(sub.bounds);
	}

	validate(value: T, constraint: NumberBounds.<T>): boolean {
		// `value == 0` is true for -0 as well, which is also a zero divisor.
		if (constraint.nonZero && value == 0) return false;
		return constraint.bounds.contains(value);
	}

	// Bounds live in the value's own unit space. When another meta type's
	// conversion scales the value by `factor`, the bounds scale identically.
	// A non-zero factor maps non-zero values to non-zero values, so `nonZero`
	// carries through unchanged.
	rescale(constraint: NumberBounds.<T>, factor: float64): NumberBounds.<T> {
		return clean({ ...constraint, bounds: constraint.bounds.scale(factor) });
	}

	narrow(current: NumberBounds.<T>, op: string, value: T): NumberBounds.<T> {
		// Each operator is a one-sided range, and narrowing is intersection.
		// Intersection is monotone, so a comparison looser than what is already
		// known changes nothing and no tightness test is needed.
		switch (op) {
			case '>=': return clean({ ...current, bounds: current.bounds.intersect(value..) });
			case '>': return clean({ ...current, bounds: current.bounds.intersect(value<..) });
			case '<=': return clean({ ...current, bounds: current.bounds.intersect(..=value) });
			case '<': return clean({ ...current, bounds: current.bounds.intersect(..<value) });
			// An equality outside the bounds already known intersects to an
			// empty range, which is a dead branch stated precisely rather than
			// a fabricated bound.
			case '==': return clean({ ...current, bounds: current.bounds.intersect(value..=value) });
			case '!=': {
				// A single range cannot exclude an arbitrary value, since that
				// would need a union of two disjoint ranges. Zero is the one
				// exception, because `nonZero` names exactly that hole.
				return value == 0 ? clean({ ...current, nonZero: true }) : current;
			}
		}
		return current;
	}

	describe(constraint: NumberBounds.<T>): string {
		const parts: [].<string> = [];
		if (!constraint.bounds.isFull)
			parts.push(`in ${constraint.bounds}`);
		if (constraint.nonZero)
			parts.push('!= 0');
		return parts.join(' and ') || 'unconstrained';
	}
}

// Canonicalization: drop what the default already says, and drop a flag the
// bounds already imply, so one constraint has one spelling.
function clean<T: Ordered.<T>>(b: NumberBounds.<T>): NumberBounds.<T> {
	return { ...b, nonZero: b.nonZero && b.bounds.contains(0) };
}
```

`conversionFactor` and `quantize` are absent, and correctly: a bound admits or rejects a value, it never changes one.

**Narrowing is where the range earns its place.** Each comparison operator *is* a one-sided range in the syntax the [ranges](ranges.md) extension gives - `>` is `value<..`, which is the form that motivated giving an exclusive start a type at all - and narrowing is their intersection. That makes it monotone by construction: a comparison looser than what is already known intersects to the same range, where four separate fields needed a tightness test in every branch to avoid *widening* a constraint on a comparison against a looser bound.

Two properties follow that the four-field form could not have. Illegal states stop being representable: `{ minimum: 5, exclusiveMinimum: 7 }` was two lower bounds at once, which is why the old rule had to say that the tighter one takes effect on each side, and **that rule is now gone rather than restated**. And `subtype` becomes containment - one range inside another - rather than a four-way reconciliation of which bound on which side was tighter.

**What it costs.** The correspondence with JSON Schema's `minimum`, `maximum`, `exclusiveMinimum`, and `exclusiveMaximum` is gone, so a tool generating metadata from a schema translates rather than copies; the translation is total and mechanical in both directions, which is the whole of the loss.

The larger change is that **validation is exact**. The four-field form compared with a relative tolerance of four float32 epsilons while `subtype` compared exactly, so a value a hair outside a bound validated and a type a hair outside did not. `Range.contains` is exact, as it must be - a `case 200..<300` that matched `300.0000001`, or a slice bound that did, would be indefensible - so validation now agrees with subtyping and with narrowing rather than disagreeing with both. Two consequences are worth stating plainly: a value within the old tolerance of a bound now fails where it used to pass, and the epsilon helpers are deleted rather than relocated. It is also the only answer that generalizes, since `NumberBounds` is now generic in its host primitive and a float32 epsilon means nothing to a `bigint` or a `decimal128` bound.

**This is the meta type that made the base worth naming.** `uint8.<{ bounds: 1..=6 }>` wants a `Range.<uint8>`, so `NumberBounds` is the first constraint shape whose fields live in the base's own value space, and it takes the type parameter described under the protocol above. The fields it replaces were declared `float32` outright, so a bound on a `float64` truncated and a bound on a `decimal128` truncated further; the parameterized form fixes a live deficiency rather than a hypothetical one, and it is what makes the operators below well typed, since `B.bounds + B2.bounds` is range arithmetic only if `B.bounds` is a range over the base.

### nonZero and division

`nonZero` is the one constraint a range cannot express, since excluding a single interior value would need a union of two disjoint ranges. It earns its place because it is the constraint the hardware cares about.

Integer division throws a RangeError when its divisor is zero, so an engine emits a test and a branch before every `idiv`. A divisor whose type cannot be zero needs neither, and the check moves from every division to the one boundary where the value was built.

```js
type Divisor = int32.<{ nonZero: true }>;

function scale(x: int32, by: Divisor): int32 {
	return x / by; // No guard. `by` cannot be zero
}

scale(10, 3); // The literal is validated once, here
// scale(10, 0); // TypeError: literal 0 fails NumberBounds.validate
```

A range that already excludes zero is non-zero without saying so, which `subtype` accounts for, so `uint32.<{ bounds: 1.. }>` is a `Divisor` and an array length is one for free.

The `narrow` hook makes the constraint reachable from ordinary control flow rather than only from an annotation. Guarding a divisor is the way this is written already, and now the guard is what removes the guard:

```js
function ratio(a: int32, b: int32): int32 {
	if (b != 0) {
		return a / b; // b: int32.<{ nonZero: true }> here, and the division cannot throw
	}
	return 0;
}
```

Because the main proposal makes the most negative value divided by `-1` wrap rather than throw, a division by a non-zero divisor cannot fail at all. It is a pure expression, so it can be hoisted out of a loop, shared between `a / b` and `a % b`, and eliminated when its result is unused. That is the second half of what the constraint buys.

### Primitive Operators

Two rules govern operator blocks:

**Raw operations inside bodies.** Inside a ```primitive``` operator body, expressions evaluate on raw values; no operator blocks (value or metadata-only) are re-entered. Without this rule ```return this + rhs;``` inside ```operator+``` would recurse infinitely, and the cast body ```return this * D.ratio;``` would re-enter ```operator*```. The metadata of an operation's result comes solely from the return type annotations per the composition rules below.

**Operator generic parameter lists use ```.<...>```.** The operator token itself may end in ```<``` or ```>```, so ```operator<.<D2: Dimensions>``` lexes unambiguously where ```operator< <D2: Dimensions>``` would collide with the ```<<``` token.

Unit conversion between operands happens at the operator's argument boundary through the standard implicit conversion described under the composition rules; the bodies below can assume ```rhs``` is already in ```this```'s unit system.

The comparison operators below compare with a tolerance, and they are the only place in this document that does. A conversion between units is a multiplication, so `Kilometer(1)` and `Meter(1000)` are equal in principle and need not be equal bit for bit once one has been scaled into the other's space; comparing dimensioned quantities exactly would make that pair unequal for no reason a reader would accept. This is a fact about converted *values* and not about bounds: `NumberBounds` validates exactly, so that a value admitted by a constraint and a type admitted by a constraint agree.

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
primitive float32<D: Dimensions> {
	// Same-dimension addition
	// The parameter type reuses D, so passing a value with matching exponents but a different ratio goes through the standard implicit conversion at the call boundary: the value is scaled by conversionFactor() and every other meta type's metadata is passed through its rescale() hook.
	// Passing a different dimension fails subtype() and is a compile error.
	operator+(rhs: float32.<D>): float32.<D> {
		return this + rhs;
	}

	operator-(rhs: float32.<D>): float32.<D> {
		return this - rhs;
	}

	// Dimension-combining multiplication
	// Exponents add, ratios multiply. No boundary conversion happens here;
	// the parameter captures the RHS parameterization exactly, and raw values
	// combine with the ratio product carrying the scale.
	operator*.<D2: Dimensions>(rhs: float32.<D2>): float32.<multiplyDimensions(D, D2)> {
		return this * rhs;
	}

	// Dimension-combining division
	// Exponents subtract, ratios divide.
	operator/.<D2: Dimensions>(rhs: float32.<D2>): float32.<divideDimensions(D, D2)> {
		return this / rhs;
	}

	// Unary operators

	operator-(): float32.<D> {
		return -this;
	}

	operator+(): float32.<D> {
		return +this;
	}

	// Scalar multiplication/division
	// A plain float32 (no metadata) is dimensionless.
	// Multiplying preserves the LHS dimension.

	operator*(rhs: float32): float32.<D> {
		return this * rhs;
	}

	operator/(rhs: float32): float32.<D> {
		return this / rhs;
	}

	// Compound assignment
	// `this` is assignable inside operator bodies, enabling
	// in-place mutation. Returns the LHS type so the expression
	// evaluates to the new value, allowing chaining.

	operator+=(rhs: float32.<D>): float32.<D> {
		this += rhs;
		return this;
	}

	operator-=(rhs: float32.<D>): float32.<D> {
		this -= rhs;
		return this;
	}

	operator*=(rhs: float32): float32.<D> {
		this *= rhs;
		return this;
	}

	operator/=(rhs: float32): float32.<D> {
		this /= rhs;
		return this;
	}

	// Comparison operators
	// Same dimension required via the parameter type. Differing ratios are normalized by the boundary conversion, so e.g. Kilometer(1) == Meter(1000).

	operator==(rhs: float32.<D>): boolean {
		return feq(this, rhs);
	}

	operator!=(rhs: float32.<D>): boolean {
		return !feq(this, rhs);
	}

	operator<(rhs: float32.<D>): boolean {
		return flt(this, rhs);
	}

	operator<=(rhs: float32.<D>): boolean {
		return fle(this, rhs);
	}

	operator>(rhs: float32.<D>): boolean {
		return fgt(this, rhs);
	}

	operator>=(rhs: float32.<D>): boolean {
		return fge(this, rhs);
	}

	// Cast: strip dimension, apply scale to get base SI value

	operator float32(): float32 {
		return this * D.ratio;
	}
}
```

**NumberBounds Operators**  
NumberBounds operators only modify the return metadata with no value, so they have no function body.

```js
primitive float32<B: NumberBounds.<float32>> {
	operator+.<B2: NumberBounds.<float32>>(rhs: float32.<B2>): float32.<{ bounds: B.bounds + B2.bounds }>;
	operator-.<B2: NumberBounds.<float32>>(rhs: float32.<B2>): float32.<{ bounds: B.bounds - B2.bounds }>;
	operator*.<B2: NumberBounds.<float32>>(rhs: float32.<B2>): float32.<{ bounds: B.bounds * B2.bounds }>;
	operator/.<B2: NumberBounds.<float32>>(rhs: float32.<B2>): float32.<{ bounds: B.bounds / B2.bounds }>;

	operator-(): float32.<{ bounds: -B.bounds }>;

	operator*(rhs: float32): float32.<{ bounds: B.bounds.scale(rhs) }>;
	operator/(rhs: float32): float32.<{ bounds: rhs == 0 ? .. : B.bounds.scale(1 / rhs) }>;
}
```

The bound of a computed value is the interval arithmetic of the bounds it was computed from, so these declarations are the range operators of the [ranges](ranges.md) extension applied to the `bounds` of each operand and nothing else. The helper layer this replaces reimplemented that arithmetic over four separate fields, and every one of its subtleties - the crossed endpoints of a difference, the four products of a multiplication and the rule that a product bound is exclusive only where every combination attaining it is, the divisor that must be bounded away from zero rather than merely non-zero - is a fact about intervals rather than about metadata, and is stated once where ranges are defined.

`nonZero` does not propagate through these, which is sound and lossy in one direction only: the product of two non-zero values is non-zero, so an operator could carry the flag through `*` and does not. The bound is what the arithmetic computes, and a flag the result could have had but does not costs a check at a later boundary and nothing else.

### Composition Rules

For a given operator invocation:

1. **Conversion at the argument boundary.** Operator parameters convert like any other typed parameter. For each meta type on the parameter's type: `subtype()` must hold; if `conversionFactor()` is defined and yields a factor other than `1`, the argument's value is multiplied by it and every *other* meta type's metadata on that argument is passed through its `rescale()` hook (falling back to `default` when `rescale` is absent); finally, any meta type defining `quantize()` maps the converted value onto the representation its constraint requires. Value blocks therefore receive operands already in the LHS's unit system and representation.
2. At most one value block may match. Its body computes the result value. If two value blocks match the same operator, the compiler reports an ambiguity error.
3. Any number of metadata-only blocks may match. Each contributes its portion of the result metadata via its return type annotation, observing operand metadata *after* the conversion in step 1.
4. If no value block matches, the default primitive operation runs.
5. All return type annotations (from both value and metadata-only blocks) are evaluated independently, and their metadata fields are merged into the flat result object.

#### Example: `Kilometer(5) + Meter(300)` where both have `NumberBounds { bounds: 0.. }`

**Conversion at the boundary:** `operator+`'s parameter is typed `float32.<D>` with `D` being the LHS's Dimensions. The `Meter` argument passes `Dimensions.subtype` (same exponents). `Dimensions.conversionFactor(ratio: 1, ratio: 1000)` = `1/1000`, so the argument value becomes `300 * 1/1000 = 0.3` and the argument's NumberBounds pass through `NumberBounds.rescale({ bounds: 0.. }, 1/1000)` = `{ bounds: 0.. }`.

**Dimensions block** (value block):
- Body runs on converted operands: `5 + 0.3 = 5.3`
- Return metadata (Dimensions portion): `{ m: 1, ratio: 1000 }` (remaining exponents `0`)

**NumberBounds block** (metadata-only):
- Return metadata (NumberBounds portion): `{ bounds: (0..) + (0..) }` = `{ bounds: 0.. }`
- No body, so no conflicting value calculation.

**Final result:** float32 value `5.3` with merged metadata `{ m: 1, kg: 0, s: 0, A: 0, K: 0, mol: 0, cd: 0, ratio: 1000, bounds: 0.. }`

The boundary conversion is what keeps cross-meta-type interactions sound. Had `Meter`'s bounds been `{ bounds: 100.. }` (at least 100 meters), the rescale step yields `{ bounds: 0.1.. }` in kilometer space and the sum's bound becomes `{ bounds: 0.1.. }`, correctly expressed in the result's unit space. Without rescaling, the range addition would have mixed meter-space and kilometer-space numbers. The metadata-only block never touches the value.

## Unit Type Aliases

All metadata is specified as flat objects. The compiler decomposes automatically based on field ownership: `{m, kg, s, A, K, mol, cd, ratio}` fields are claimed by `Dimensions`, `{bounds, nonZero}` fields are claimed by `NumberBounds`.

### Base SI Units

```js
type Meter = float32.<{ m: 1 }>;
type Kilogram = float32.<{ kg: 1 }>;
type Second = float32.<{ s: 1 }>;
type Ampere = float32.<{ A: 1 }>;
type Kelvin = float32.<{ K: 1 }>;
type Mole = float32.<{ mol: 1 }>;
type Candela = float32.<{ cd: 1 }>;
```

Fields omitted from a metadata object take the owning meta type's default values, so the remaining exponents are `0` and the ratio is `1`.

### Prefixed Units

The `ratio` field encodes the prefix as a scale factor relative to the base SI unit.

```js
type Kilometer = float32.<{ m: 1, ratio: 1000 }>;
type Centimeter = float32.<{ m: 1, ratio: 1/100 }>;
type Millimeter = float32.<{ m: 1, ratio: 1/1000 }>;
type Micrometer = float32.<{ m: 1, ratio: 1/1000000 }>;
type Gram = float32.<{ kg: 1, ratio: 1/1000 }>;
type Milligram = float32.<{ kg: 1, ratio: 1/1000000 }>;
type Millisecond = float32.<{ s: 1, ratio: 1/1000 }>;
type Microsecond = float32.<{ s: 1, ratio: 1/1000000 }>;
```

### Derived SI Units

```js
type Velocity = float32.<{ m: 1, s: -1 }>; // m/s
type Acceleration = float32.<{ m: 1, s: -2 }>; // m/s**2
type Newton = float32.<{ m: 1, kg: 1, s: -2 }>; // kg*m/s**2
type Joule = float32.<{ m: 2, kg: 1, s: -2 }>; // kg*m**2/s**2
type Watt = float32.<{ m: 2, kg: 1, s: -3 }>; // kg*m**2/s**3
type Pascal = float32.<{ m: -1, kg: 1, s: -2 }>; // kg/(m*s**2)
type Hertz = float32.<{ s: -1 }>; // 1/s
type Momentum = float32.<{ m: 1, kg: 1, s: -1 }>; // kg*m/s
type Density = float32.<{ m: -3, kg: 1 }>; // kg/m**3
type Dimensionless = float32.<{}>;
```

### Compound Prefixed Derived Units

```js
type KilometersPerHour = float32.<{ m: 1, s: -1, ratio: 1000/3600 }>;
type GramsPerCubicCm = float32.<{ m: -3, kg: 1, ratio: (1/1000) / (1/1000000) }>;
```

### NumberBounds-Only Types (No Dimension)

```js
type Positive = float32.<{ bounds: 0<.. }>;
type NonNegative = float32.<{ bounds: 0.. }>;
type Normalized = float32.<{ bounds: 0..=1 }>;
type Probability = Normalized;
type Percentage = float32.<{ bounds: 0..=100 }>;
```

### Combined: Dimension + NumberBounds

Fields from both `Dimensions` and `NumberBounds` appear in a single flat object. The compiler decomposes them automatically.

```js
type PositiveMeter = float32.<{
	m: 1,
	bounds: 0<..
}>;

type SafeSpeed = float32.<{
	m: 1, s: -1,
	bounds: 0..=343
}>;

type Latitude = float32.<{
	bounds: -90..=90
}>;

type Longitude = float32.<{
	bounds: -180..<180
}>;
```

## DecimalContext

The decimal types follow IEEE 754-2008 decimal arithmetic: `decimal128` carries 34 significant digits and rounds ties to even. Money, tax, and measurement code needs more than that. It needs values pinned to a scale - a number of digits after the decimal point - and it needs to say how a value that doesn't land on that grid is rounded. `DecimalContext` is a meta type carrying both, so scale and rounding live in the type instead of being threaded through every call as a context object.

```js
enum Rounding: uint8 { HalfEven, HalfUp, HalfDown, Down, Up, Ceiling, Floor };

type DecimalContext = {
	scale?: int32, // Digits kept after the decimal point
	rounding?: Rounding // Applied when quantizing to `scale`. Defaults to HalfEven
};

meta DecimalContext {
	default = {};

	// A value at a given scale is assignable where an unscaled value is
	// expected. Assigning between different scales is permitted and
	// quantizes; assigning an unscaled value into a scaled type quantizes
	// as well. Both go through rescale() below.
	subtype(sub: DecimalContext, sup: DecimalContext): boolean {
		return true;
	}

	// No conversionFactor: quantization changes a value's representation,
	// not its unit, so nothing is multiplied.

	// Round the value onto the constraint's grid. Called at boundaries, so an
	// expression's intermediates stay exact and only the stored value rounds.
	quantize(value: decimal128, constraint: DecimalContext): decimal128 {
		if (constraint.scale == null) {
			return value;
		}
		return value.round(constraint.scale, constraint.rounding ?? Rounding.HalfEven);
	}

	describe(constraint: DecimalContext): string {
		if (constraint.scale == null)
			return 'unscaled';
		return `scale ${constraint.scale}, ${Rounding.toString(constraint.rounding ?? Rounding.HalfEven)}`;
	}
}
```

Arithmetic is exact within an expression. Quantization happens where every other metadata rule is applied: at assignment, argument, and return boundaries. An intermediate result therefore keeps full precision and only the value that lands in a scaled type is rounded, so `a * b * c` rounds once rather than three times.

```js
primitive decimal128<C: DecimalContext> {
	// Arithmetic drops the scale: the result of an operation is exact and
	// unscaled until it reaches a boundary that fixes a scale.
	operator+(rhs: decimal128): decimal128;
	operator-(rhs: decimal128): decimal128;
	operator*(rhs: decimal128): decimal128;
	operator/(rhs: decimal128): decimal128;
}
```

```js
type Money = decimal128.<{ scale: 2 }>;

const price: Money = 19.999; // 20.00, quantized at the assignment boundary
const rate: decimal128 = 0.0825;
const total: Money = price * (1 + rate); // Exact product, quantized once on assignment

function charge(amount: Money) {}
charge(price * 3); // Quantized at the argument boundary
```

Division that has no exact representation quantizes to the destination's scale. When the destination fixes no scale, the result keeps 34 significant digits and rounds ties to even, as IEEE 754 specifies:

```js
const third: decimal128.<{ scale: 4, rounding: Rounding.Down }> = 1 / 3; // 0.3333
const exact: decimal128 = 1 / 3; // 0.3333333333333333333333333333333333
```

Comparisons compare values, not representations, so a scale difference never changes the answer:

```js
const a: decimal128.<{ scale: 2 }> = 1.5;
const b: decimal128.<{ scale: 4 }> = 1.5;
a == b; // true, 1.50 and 1.5000 are the same value
```

Because `DecimalContext` claims only `scale` and `rounding`, it composes with any other meta type on the same value by the flat merge rules below. A currency-tagged money type is the two together:

```js
type Currency = { currency: string };
meta Currency {
	default = { currency: '' };
	// No conversionFactor: currencies never convert implicitly.
	subtype(sub: Currency, sup: Currency): boolean {
		return sup.currency == '' || sub.currency == sup.currency;
	}
	describe(constraint: Currency): string {
		return constraint.currency || 'currency-less';
	}
}

type USD = decimal128.<{ currency: 'USD', scale: 2 }>;
type EUR = decimal128.<{ currency: 'EUR', scale: 2 }>;

const a: USD = 19.99;
// const b: EUR = a; // TypeError: Currency.subtype({ currency: 'USD' }, { currency: 'EUR' }) failed
```

## Compiler Decomposition Rules

When the compiler encounters `float32.<{ ... }>`:

**Step 1.** Collect all registered `meta` types and their field sets.

```
Dimensions -> { m, kg, s, A, K, mol, cd, ratio }
NumberBounds -> { bounds, nonZero }
```

**Step 2.** For each field in the metadata object, find which `meta` type claims it. Each field must belong to exactly one `meta` type. Unclaimed fields produce a compile error.

**Step 3.** Group fields by their `meta` type.

```
Input: { m: 1, kg: 0, s: -1, ratio: 1, bounds: 0..=343 }
  -> Dimensions: { m: 1, kg: 0, s: -1, ratio: 1 }
  -> NumberBounds: { bounds: 0..=343 }
```

**Step 4.** When executing an operator, run each `meta` type's operator block on its portion independently.

**Step 5.** Merge results from all blocks back into a flat object.

**Conflict detection:** If two `meta` types claim the same field name, the compiler reports an error at the `meta` declaration site, not at usage. This is enforced when a `meta` block is registered. Symbol-keyed fields can be used to avoid conflicts between third-party libraries. Because field claiming is global and flat, a meta type cannot be extended with new fields by a third party; extensions ship as separate meta types with their own (ideally symbol-keyed) fields, as shown in the metadata key scoping section.

**Missing fields:** When a value's metadata doesn't include fields for a given `meta` type, that type's `default` value is used. For example, a plain `float32` has no metadata fields, all meta types use their defaults (`Dimensions.default = { m:0, kg:0, s:0, A:0, K:0, mol:0, cd:0, ratio:1 }`, `NumberBounds.default = {}`). This also applies per field: fields omitted from a metadata object take their values from the owning meta type's `default`, so `float32.<{ m: 1 }>` is a Meter and `float32.<{ m: 1, s: -1 }>` is a Velocity.

**Operator block absence:** If a `meta` type has no operator block defining a particular operator, the metadata for that type falls back to `default` on the result. This means metadata types only need to define operators where they have meaningful propagation logic.

## Implicit Cast Operators

A raw `number` has no metadata. To allow assignment from `number` to a metadata-bearing `float32`, an implicit cast operator must be explicitly defined on `number`. Without these operators, `const v: Velocity = 10` would be a type error.

The `operator T()` syntax defines an implicit cast that the compiler invokes at assignment boundaries, function call sites, and return statements when the source type doesn't match the target type.

### Defining Cast Operators

```js
// Cast from number -> float32 with metadata. The value passes through unchanged and the destination type's metadata is attached.
// One cast operator is defined per meta type; during decomposition the compiler invokes the cast for each meta type present on the destination but absent on the source (the source carries that meta type's default, which is why no extra "does not already have Dimensions" constraint is needed here).

primitive float32 {
	operator float32.<Dimensions>() {
		return this;
	}

	// NumberBounds.validate() is called at the cast boundary.
	operator float32.<NumberBounds>() {
		return this;
	}
}
```

Both operators compose for combined types. When the target is `float32.<{ m:1, ..., bounds: 0<.. }>`, the compiler decomposes into Dimensions and NumberBounds slots and invokes both cast operators. `NumberBounds.validate()` runs at the cast boundary (elided for compile-time-provable constant literals).

### Cast Operator Invocation Points

The compiler invokes implicit cast operators at these boundaries:

- **Variable declaration:** `const v: Velocity = 10;`
- **Function argument:** `kineticEnergy(80, 10)` where parameters are typed
- **Return statement:** `return 9.80665;` where return type is typed
- **Array element:** `const forces: [].<Newton> = [10, 20, 30];`
- **Constructor:** `Meter(100)` invokes the same cast as `const m: Meter = 100;`
- **Uninitialized declaration:** `let d: Meter;` invokes it on the base's zero

### Defaults of Metadata-Bearing Types

A declaration without an initializer holds its type's default, and a
metadata-bearing type's default is its base's zero **having crossed into it** —
the same crossing an initializer of the base's type makes, through the same
cast. `let d: Meter;` and `let d: Meter = 0;` therefore succeed together and
fail together, and which it is follows from what the program has declared:

```js
type Dimensions = { m: number, kg: number, s: number, /* ... */ };
meta Dimensions {
  default = { m: 0, kg: 0, s: 0 };
  subtype(a, b) { return a.m === b.m && a.kg === b.kg && a.s === b.s; }
  // No validate — dimensions constrain type compatibility, not value ranges.
}

// With the cast declared, a unit type has a zero.
primitive float32 { operator float32.<Dimensions>() { return this; } }
let d: Meter;                  // 0 m
class Vector3 { x: Meter; y: Meter; z: Meter; }
let field: [10].<Vector3>;     // ten zero-filled vectors

// Without it, nothing crosses from an unconstrained value, so there is no zero.
let e: Meter;                  // TypeError: needs an initializer
```

Two consequences worth stating outright, because both are load-bearing
elsewhere in the design.

**Declaring the cast is what makes unit types zero-fillable.** The memory
layout extension needs `let d: [10].<Vector3>;` to hold ten zero-filled
instances, and an aggregate has a default exactly where each of its parts does.
A `Vector3` of `Meter` fields therefore has a zero exactly where `float32`
declares the cast into `Dimensions` — which the section above declares anyway,
for `const v: Velocity = 10;` to compile. A design that ships unit types ships
that cast, and gets zero-filling with it.

**A meta type that declares no cast is how a type says it has no meaningful
zero.** `NumberBounds { bounds: 1.. }` has no zero because `validate` refuses
one — the crossing runs `validate`, so a cast is a way *in*, not a way past a
bound. A brand into which no cast is declared has none either, its values
arriving only through the operators that produce them. A value type class
holding such a field has no zero-filled form, which is the design's way of
saying that an instance must be constructed rather than defaulted.

## Examples

```js
// number -> Velocity via cast operator:
const v: Velocity = 10; // operator float32.<Dimensions>() invoked

// number -> Probability via cast operator + validation:
const p: Probability = 0.7;
// operator float32.<NumberBounds>() invoked
// NumberBounds.validate(0.7, { bounds: 0..=1 }) -> true

const p2: Probability = 1.5; // NumberBounds.validate(1.5, { bounds: 0..=1 }) -> false, throws TypeError('Expected in 0..=1, got 1.5')

// number -> PositiveMeter via both cast operators:
const h: PositiveMeter = 1.75; // Dimensions cast, NumberBounds.validate(1.75, { bounds: 0<.. }) -> true

const h2: PositiveMeter = -3;
// NumberBounds.validate(-3, { bounds: 0<.. }) -> false, throws TypeError

// Already-typed value, cast operator doesn't apply:
const m: Meter = 100;
const v1: Velocity = m; // Dimensions.subtype({ m: 1, s: 0 }, { m: 1, s: -1 }), throws TypeError

// Unit conversion on assignment via conversionFactor:
const km: Kilometer = 5;
const m2: Meter = km;
// subtype passes (same exponents), value * conversionFactor({ ratio: 1000 }, { ratio: 1 })
// = 5 * 1000 = 5000

// Same type assignment: no checks
const v2: Velocity = 10;
const v3: Velocity = v2; // direct, same metadata

// Wider -> narrower: runtime check
const v4: Velocity = 100;
const safe: SafeSpeed = v4;
// Dimensions.subtype: exponents match
// NumberBounds.subtype: v4 is unbounded (default `..`), SafeSpeed has { bounds: 0..=343 }
//   sup.minimum = 0, sub.minimum = null -> false
// Insert runtime check: NumberBounds.validate(v4_value, { bounds: 0..=343 })

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
// NumberBounds: no bounds on either -> default {} -> not present in result
// Result: float32.<{ m: 1, kg: 0, s: -1, ratio: 1 }> matches Velocity

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

// LHS ratio wins. The RHS argument is converted into the LHS unit
// system at the operator's argument boundary.
const totalKm: Kilometer = d1 + d2;
// Value: 5 + 300 * (1/1000) = 5 + 0.3 = 5.3
// totalKm == Kilometer(5.3)

// To get result in meters, put Meter on the LHS:
const totalM: Meter = d2 + d1;
// Value: 300 + 5 * (1000/1) = 300 + 5000 = 5300
// totalM == Meter(5300)

const d3: Centimeter = 50;
const d4: Millimeter = 200;
const sum: Centimeter = d3 + d4;
// Value: 50 + 200 * ((1/1000) / (1/100)) = 50 + 20 = 70
// sum == Centimeter(70)

// Cross-prefix comparison works via the boundary conversion:
const a: Kilometer = 1;
const b: Meter = 1000;
console.log(a == b); // true (rhs converted: 1000 * (1/1000) = 1, then 1 == 1)
```

### Combined Dimension + NumberBounds

```js
const width: PositiveMeter = 0.5;
const height: PositiveMeter = 1.75;
const area = width * height;
// Dimensions: { m: 1 } * { m: 1 } = { m: 2, kg: 0, s: 0, ratio: 1 }
// NumberBounds: (0<..) * (0<..)
//   two positive lower bounds, partial propagation -> 0<..
// area: float32.<{ m: 2, kg: 0, s: 0, ratio: 1, bounds: 0<.. }>
// That's a positive square-meter.
```

### Control Flow Narrowing

```js
function clampToSafe(v: Velocity): SafeSpeed {
	if (v >= 0) {
		// NumberBounds.narrow({ bounds: .. }, '>=', 0) -> { bounds: 0.. }
		// v: float32.<{ m: 1, kg: 0, s: -1, ratio: 1, bounds: 0.. }>

		if (v <= 343) {
			// NumberBounds.narrow({ bounds: 0.. }, '<=', 343) -> { bounds: 0..=343 }
			// v: float32.<{ m: 1, kg: 0, s: -1, ratio: 1, bounds: 0..=343 }>
			//
			// SafeSpeed.subtype check:
			//   Dimensions: exponents match
			//   NumberBounds: NumberBounds.subtype({ bounds: 0..=343 }, { bounds: 0..=343 }) -> true
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
	// NumberBounds: .. * .. * (0<..) -> .., since an unbounded factor bounds nothing
	// Joule states no bound of its own, so the result needs none.
	// Joule has no NumberBounds requirement -> extra bounds are fine (subtype).
}
```

### Pressure

```js
type SquareMeter = float32.<{ m: 2 }>;

function pressure(force: Newton, area: SquareMeter): Pascal {
	return force / area;
	// Dimensions: { m: 1, kg: 1, s: -2 } / { m: 2 } = { m: -1, kg: 1, s: -2 } Pascal
}
```

### NumberBounds Arithmetic

```js
const prob: Probability = 0.7;
const prob2: Probability = 0.2;
const sum = prob + prob2;
// NumberBounds: (0..=1) + (0..=1) = 0..=2
// sum: float32.<{ bounds: 0..=2 }>

// const bad: Probability = sum;
// NumberBounds.subtype({ bounds: 0..=2 }, { bounds: 0..=1 }) -> false (0..=1 does not contain 0..=2)

if (sum <= 1) {
	// NumberBounds.narrow({ bounds: 0..=2 }, '<=', 1) -> { bounds: 0..=1 }
	const safe: Probability = sum; // proven by narrowing
}
```

### Dimensional Errors

```js
// distance + time;
// operator+ requires the same dimension via its parameter type;
// Dimensions.subtype fails:
//    { m: 1, kg: 0, s: 0 } != { m: 0, kg: 0, s: 1 }
//    Error: cannot add Meter (m) to Second (s)

// speed + force;
// Dimensions.subtype: { m: 1, kg: 0, s: -1 } != { m: 1, kg: 1, s: -2 }
//    Error: cannot add Velocity (m/s) to Newton (kg*m/s**2)

// const bad: Velocity = distance;
// Dimensions.subtype: { m: 1, s: 0 } != { m: 1, s: -1 }
//    Error: Meter is not assignable to Velocity
```

## Metadata key scoping with Symbols

For library-quality code where name collisions are a concern, metadata keys can be symbols:

```js
const si = Object.freeze({
	m: Symbol('SI.length'),
	kg: Symbol('SI.mass'),
	s: Symbol('SI.time'),
});

type Dimensions = { [si.m]: int32, [si.kg]: int32, [si.s]: int32, ratio: rational };
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
	age: number.<{ bounds: 0..=150 }>;
	email: string.<{ pattern: /^[^@]+@[^@]+$/ }>;
}
```

The latter allows compile time validation as well as runtime validation. The following examples separate validation from decorators.

### Validate example

The following ```@validate``` decorator could still be created, but it would just be doing redundant work.

```js
// NumberBounds is the meta type defined earlier. StringBounds is its
// string counterpart, registered the same way:

type StringBounds = {
	pattern?: RegExp,
	minLength?: uint32,
	maxLength?: uint32,
};

meta StringBounds {
	default = {};

	subtype(sub: StringBounds, sup: StringBounds): boolean {
		if (sup.pattern != null && sub.pattern != sup.pattern) return false;
		if (sup.minLength != null && (sub.minLength == null || sub.minLength < sup.minLength)) return false;
		if (sup.maxLength != null && (sub.maxLength == null || sub.maxLength > sup.maxLength)) return false;
		return true;
	}

	validate(value: string, constraint: StringBounds): boolean {
		if (constraint.pattern != null && !constraint.pattern.test(value)) return false;
		if (constraint.minLength != null && value.length < constraint.minLength) return false;
		if (constraint.maxLength != null && value.length > constraint.maxLength) return false;
		return true;
	}
}

const validatorsKey = Symbol('validators');

type ValidateField<T> = {
	name: string,
	constraint: T,
	meta: MetaProtocol.<T>
};

partial class ClassMetadata {
	[validatorsKey]: [].<ValidateField.<NumberBounds> | ValidateField.<StringBounds>> = [];
}

function validate<B: NumberBounds, TClass>(
	{ name, metadata }: Reflect.ClassField.<number.<B>, TClass>
) where typeof name == 'string' {
	metadata[validatorsKey].push({ name, constraint: B, meta: NumberBounds });
}

function validate<S: StringBounds, TClass>(
	{ name, metadata }: Reflect.ClassField.<string.<S>, TClass>
) where typeof name == 'string' {
	metadata[validatorsKey].push({ name, constraint: S, meta: StringBounds });
}

function validateInstance<T>(instance: T): boolean {
	const entries = Reflect.getMetadata.<Reflect.Class, T>()[validatorsKey];
	if (!entries) return true;
	for (const { name, constraint, meta } of entries) {
		if (!meta.validate(instance[name], constraint)) {
			return false;
		}
	}
	return true;
}

class User {
	@validate
	age: number.<{ bounds: 0..=150 }>;
	@validate
	email: string.<{ pattern: /^[^@]+@[^@]+$/ }>;
}

const user = new User();
user.age = 25;
user.email = 'alice@example.com';
validateInstance(user); // true

user.age = -5; // Note: This would throw at compile time. If the value was dynamic then it would throw at runtime assuming the meta block defines a validate
validateInstance(user); // false, the 0..=150 bound is violated

user.age = 25;
user.email = 'not-an-email'; // Note: This would throw at compile time. If the value was dynamic then it would throw at runtime assuming the meta block defines a validate
validateInstance(user); // false, pattern violated
```

### JSON Serialization with field name overrides

Native typed JSON parsing that runs these constraints during the parse itself is specified in [Serialization](serialization.md). The decorator pattern here is the userland form and the way to express field name overrides.

```js
const schemaKey = Symbol('schema');

type SerializeData = {
	name: string,
	wireName: string
};

partial class ClassMetadata {
	[schemaKey]: [].<SerializeData> = [];
}

// @field() - registers a field for serialization with an optional wire name
function field<T, TClass>(
	{ name, metadata }: Reflect.ClassField.<T, TClass>
) where typeof name == 'string' {
	metadata[schemaKey].push({ name, wireName: name });
}
function field<T, TClass>(
	wireName: string,
	{ name, metadata }: Reflect.ClassField.<T, TClass>
) where typeof name == 'string' {
	metadata[schemaKey].push({ name, wireName });
}

function serialize<T>(instance: T): { [key: string]: any } {
	const result: { [key: string]: any } = {};
	for (const { name, wireName } of Reflect.getMetadata.<Reflect.Class, T>()[schemaKey]) {
		result[wireName] = instance[name];
	}
	return result;
}

function deserialize<T>(cls: { new(): T }, data: { [key: string]: any }): T {
	const instance = new cls();
	for (const { name, wireName } of Reflect.getMetadata.<Reflect.Class, T>()[schemaKey]) {
		instance[name] = data[wireName]; // implicit cast -> triggers meta validate
	}
	return instance;
}

class UserResponse {
	@field
	id: uint64;
	@field('user_name')
	userName: string.<{ minLength: 1, maxLength: 100 }>;
	@field('email_address')
	email: string.<{ pattern: /^[^@]+@[^@]+$/ }>;
	@field
	age: number.<{ bounds: 0..=150 }>;
}

// Incoming JSON:
// { "id": 42, "user_name": "alice", "email_address": "a@b.com", "age": 30 }
const user = deserialize(UserResponse, json);
// Assignment to `email` triggers: string.<StringBounds>.validate('a@b.com', { pattern: ... })
// Assignment to `age` triggers: number.<NumberBounds>.validate(30, { bounds: 0..=150 })

serialize(user);
// { "id": 42, "user_name": "alice", "email_address": "a@b.com", "age": 30 }
```

### Typed API routing

```js
const routeKey = Symbol('route');

type Route = {
	method: 'POST' | 'GET',
	path: string,
	handler: string
};

partial class ClassMetadata {
	[routeKey]: [].<Route> = [];
}

function get<T extends (...args: [].<any>) => any, TClass>(
	path: string,
	{ name, metadata }: Reflect.ClassMethod.<T, TClass>
) where typeof name == 'string' {
	metadata[routeKey].push({ method: 'GET', path, handler: name });
}

function post<T extends (...args: [].<any>) => any, TClass>(
	path: string,
	{ name, metadata }: Reflect.ClassMethod.<T, TClass>
) where typeof name == 'string' {
	metadata[routeKey].push({ method: 'POST', path, handler: name });
}

class EventController {
	@get('/events')
	list(
		limit: uint32.<{ bounds: 1..=100 }> = 20,
		offset: uint32.<{ bounds: 0.. }> = 0
	): [].<Event> {
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
	title: string.<{ minLength: 1, maxLength: 200 }>;
	date: string.<{ pattern: /^\\d{4}-\\d{2}-\\d{2}$/ }>;
	capacity: uint32.<{ bounds: 1..=10000 }>;
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

partial class ClassMetadata {
	[tableKey]: string = '';
	[columnKey]: [].<Column> = [];
}

function table<T>(
	name: string,
	{ metadata }: Reflect.Class.<T>
) {
	metadata[tableKey] = name;
}

function column<T, TClass>(
	{ name, metadata }: Reflect.ClassField.<T, TClass>
) where typeof name == 'string' {
	metadata[columnKey].push({ field: name, column: name });
}
function column<T, TClass>(
	column: string,
	{ name, metadata }: Reflect.ClassField.<T, TClass>
) where typeof name == 'string' {
	metadata[columnKey].push({ field: name, column });
}

@table('sensors')
class SensorReading {
	@column
	id: uint64;
	@column('sensor_id')
	sensorId: uint32;
	@column
	temperature: float32.<{ bounds: -273.15.. }>; // can't go below absolute zero
	@column
	humidity: float32.<{ bounds: 0..=100 }>; // percentage
	@column('recorded_at')
	recordedAt: string.<{ pattern: /^\\d{4}-\\d{2}-\\d{2}T/ }>;
}

// ORM builds:
//   SELECT id, sensor_id, temperature, humidity, recorded_at
//   FROM sensors
//   WHERE ...
//
// Row hydration assigns each column value to the typed field.
// temperature = row['temperature'] triggers:
//   float32.<{ bounds: -273.15.. }>.validate(value, { bounds: -273.15.. })
// A corrupted row with temperature = -300 fails validation at the ORM boundary, not deep in business logic.
```

## vec3 Example

This is more in-depth and covers function metadata propagation. This is the ```Dimensions``` setup for 3D which is a more practical example.

```js
// Generic 2- and 3-vectors parameterized by Dimensions.
// All components share the same dimensional metadata; the default {} is dimensionless,
// so bare vec2 and vec3 are ordinary float vectors.
type vec3<D: Dimensions = {}> = vector.<float32.<D>, 3>;
type vec2<D: Dimensions = {}> = vector.<float32.<D>, 2>;

// Concrete physics vector types
type Position = vec3.<{ m: 1 }>; // meters
type Velocity3 = vec3.<{ m: 1, s: -1 }>; // m/s
type Acceleration3 = vec3.<{ m: 1, s: -2 }>; // m/s**2
type Force3 = vec3.<{ m: 1, kg: 1, s: -2 }>; // N
type Momentum3 = vec3.<{ m: 1, kg: 1, s: -1 }>; // kg*m/s
type Torque3 = vec3.<{ m: 2, kg: 1, s: -2 }>; // N*m
type AngularVel3 = vec3.<{ s: -1 }>; // rad/s
type Unitless3 = vec3.<{}>; // direction, etc.

// Scalar types (from scalar spec, repeated for context)
type Meter = float32.<{ m: 1 }>;
type Kilogram = float32.<{ kg: 1 }>;
type Second = float32.<{ s: 1 }>;
type Velocity = float32.<{ m: 1, s: -1 }>;
type Newton = float32.<{ m: 1, kg: 1, s: -2 }>;
type Joule = float32.<{ m: 2, kg: 1, s: -2 }>;
type SquareMeter = float32.<{ m: 2 }>;

// Note: vector.<T, N> has built-in element-wise operators on raw values.
// So these operators skip redeclaring operator bodies.

primitive vector<float32<D: Dimensions>, N: uint32> {

	// Same-dimension add/subtract. The parameter type reuses D, so unit
	// conversion happens through the standard implicit conversion at the
	// call boundary, exactly as with the scalar operators. Each component
	// is scaled by the same conversionFactor().

	operator+(rhs: vector.<float32.<D>, N>): vector.<float32.<D>, N>;

	operator-(rhs: vector.<float32.<D>, N>): vector.<float32.<D>, N>;

	// Scalar multiply/divide
	// vec3.<D> * float32 = vec3.<D> (scale a vector)

	operator*(rhs: float32): vector.<float32.<D>, N>;

	operator/(rhs: float32): vector.<float32.<D>, N>;

	// Dimensioned scalar multiply/divide
	// vec3.<D> * float32.<D2> = vec3.<D*D2>
	// e.g. Velocity3 * Second = Position

	operator*.<D2: Dimensions>(rhs: float32.<D2>): vector.<float32.<multiplyDimensions(D, D2)>, N>;

	operator/.<D2: Dimensions>(rhs: float32.<D2>): vector.<float32.<divideDimensions(D, D2)>, N>;

	// Unary

	operator-(): vector.<float32.<D>, N>;

	// Compound assignment

	operator+=(rhs: vector.<float32.<D>, N>): vector.<float32.<D>, N>;

	operator-=(rhs: vector.<float32.<D>, N>): vector.<float32.<D>, N>;

	operator*=(rhs: float32): vector.<float32.<D>, N>;

	operator/=(rhs: float32): vector.<float32.<D>, N>;

	// Comparison
	// Per-component equality; differing ratios are normalized by the
	// boundary conversion. Just an example as float equality usually
	// requires special handling.

	operator==(rhs: vector.<float32.<D>, N>): boolean;

	operator!=(rhs: vector.<float32.<D>, N>): boolean;
}



// Dot product
// e.g. dot(Force3, Position) -> Joule (N*m = J)
//      dot(Velocity3, Velocity3) -> m**2/s**2 (speed**2)
function dot<D: Dimensions, D2: Dimensions>(
	a: vec3.<D>,
	b: vec3.<D2>
): float32.<multiplyDimensions(D, D2)> {
	return a[0] * b[0] + a[1] * b[1] + a[2] * b[2];
	// Each a[i]*b[i] has Dimensions { D.m+D2.m, D.kg+D2.kg, D.s+D2.s }.
	// All three products share the same dimension, so + is valid.
}

// Cross product
// e.g. cross(Position, Force3) -> Torque3 (m*N = N*m)
//      cross(Velocity3, Velocity3) -> vec3.<m**2/s**2>
function cross<D: Dimensions, D2: Dimensions>(
	a: vec3.<D>,
	b: vec3.<D2>,
): vec3.<multiplyDimensions(D, D2)> {
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
function Math.sqrt<D: Dimensions>(x: float32.<D>): float32.<halveDimensions(D)>
	where evenDimensions(D);

// Propagate Dimensions metadata
function Math.hypot<D: Dimensions>(...args: [].<float32.<D>>): float32.<D>;

// The remaining Math functions that take dimensioned arguments preserve the
// dimension, since selecting, negating, or rounding a quantity cannot change
// what it measures. Comparison-based selection requires the same
// parameterization on both arguments, so a Kilometer and a Meter are
// converted at the argument boundary before being compared.

function Math.min<D: Dimensions>(...args: [].<float32.<D>>): float32.<D>;
function Math.max<D: Dimensions>(...args: [].<float32.<D>>): float32.<D>;
function Math.abs<D: Dimensions>(x: float32.<D>): float32.<D>;
function Math.clamp<D: Dimensions>(x: float32.<D>, low: float32.<D>, high: float32.<D>): float32.<D>;
function Math.floor<D: Dimensions>(x: float32.<D>): float32.<D>;
function Math.ceil<D: Dimensions>(x: float32.<D>): float32.<D>;
function Math.round<D: Dimensions>(x: float32.<D>): float32.<D>;
function Math.trunc<D: Dimensions>(x: float32.<D>): float32.<D>;
function Math.fround<D: Dimensions>(x: float32.<D>): float32.<D>;

// sign returns a dimensionless -1, 0, or 1.
function Math.sign<D: Dimensions>(x: float32.<D>): float32;

// The transcendental functions take dimensionless arguments. Passing a
// dimensioned value is a TypeError, since exp, log, and the trigonometric
// functions are only defined on pure numbers.
// Math.log(Meter(5)); // TypeError: expected a dimensionless float32

// Magnitude (vector length)
// Uses Math.hypot, which preserves Dimensions.
// magnitude(Position) -> Meter
// magnitude(Force3) -> Newton
function magnitude<D: Dimensions>(v: vec3.<D>): float32.<D> {
	return Math.hypot(v[0], v[1], v[2]);
}

// Alternative via dot + sqrt (equivalent):
//function magnitude<D: Dimensions>(v: vec3.<D>): float32.<D> {
//	return Math.sqrt(dot(v, v));
//	// dot(v, v): float32.<{ m: 2*D.m, kg: 2*D.kg, s: 2*D.s, ... }>
//	// Math.sqrt:  halves exponents -> float32.<D>
//}

// Squared magnitude
// Returns D*D exponents.
function magnitudeSq<D: Dimensions>(v: vec3.<D>): float32.<multiplyDimensions(D, D)> {
	return dot(v, v);
}

// Normalize (unit vector)
// Divides out the dimension, returning a Unitless3.
// magnitude is float32.<D>, dividing vec3.<D> / float32.<D>
// yields vec3.<D-D> = vec3.<dimensionless>.

function normalize<D: Dimensions>(v: vec3.<D>): Unitless3 {
	return v / magnitude(v);
	// vec3.<D> / float32.<D>
	// Dimensions: { D.m - D.m, D.kg - D.kg, D.s - D.s } = { 0, 0, 0 }
	//   Unitless3
}

// Distance between two positions

function distance<D: Dimensions>(a: vec3.<D>, b: vec3.<D>): float32.<D> {
	return magnitude(a - b);
	// a - b: vec3.<D> (same dimension)
	// magnitude: float32.<D>
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
// accel * dt -> vec3.<{ m: 1, s: -2 }> * float32.<{ s: 1 }> -> vec3.<{ m: 1, s: -1 }> = Velocity3
// velocity += Velocity3 -> same dimension

position += velocity * dt;
// velocity * dt: Velocity3 * Second
//   = vec3.<{ m: 1, s: -1 }> * float32.<{ s: 1 }>
//   = vec3.<{ m: 1, s: 0 }> = Position
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
	// dot(v, v): float32.<{ m: 2, s: -2 }>, speed squared
	// * Kilogram: float32.<{ m: 2, kg: 1, s: -2 }>, Joule
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
// magnitude(Position) -> Meter
// hypot(3, 4, 0) = 5

const dir: Unitless3 = normalize(pos);
// Position / Meter -> Unitless3
// vec3(3/5, 4/5, 0) = vec3(0.6, 0.8, 0)

const speed: Velocity = magnitude(vel);
// magnitude(Velocity3) -> Velocity
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
	//   = float32.<Velocity> (scalar speed along normal)
	//
	// normal * Velocity: Unitless3 * float32.<Velocity>
	//   = vec3.<{ m: 1, s: -1 }> = Velocity3
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
type GravConst = float32.<{ m: 3, kg: -1, s: -2 }>;
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
//   operator+ parameter is typed with the LHS Dimensions;
//   Dimensions.subtype({ m: 1, s: -1 }, { m: 1, s: 0 }) fails, Invalid
// Error: cannot add Position (m) to Velocity3 (m/s)

// const bad: Force3 = accel;
//   vec3.<Acceleration> -> vec3.<Force>
// Dimensions.subtype: kg: 0 != kg: 1, Invalid

// cross(position, vel);
// This is allowed, cross multiplies dimensions:
// Dimensions: { m: 1, s: 0 } + { m: 1, s: -1 } = { m: 2, s: -1 }
// Result: vec3.<{ m: 2, s: -1 }>, Valid, not a named type

// dot(position, vel);
// Also allowed, dot multiplies dimensions:
// float32.<{ m: 2, s: -1 }>, Valid, unnamed
```


Math.sqrt compile errors

```js
const speedSq: SquareMeter = 25; // m**2
const spd: Meter = Math.sqrt(speedSq);
// D = { m: 2, s: 0 }. m%2==0, kg%2==0, s%2==0
// Result: float32.<{ m: 1, s: 0 }> = Meter, Valid

// const bad = Math.sqrt(Meter(9));
// D = { m: 1, s: 0 }. m%2 == 1 != 0, Invalid
// where clause fails: cannot sqrt an odd-exponent dimension.
// Compile error: sqrt requires even dimensional exponents.
```

Note that function blocks that define metadata operations follow the same merge rules as operators.

## TODO Items

### Metadata on array types

The invoicing example writes ```[].<LineItem>.<{ minLength: 1 }>``` for a non-empty array, which the ```minLength```/```maxLength``` pair on ```StringBounds``` already models for strings. The decision is to define an ```ArrayBounds``` meta type carrying ```minLength``` and ```maxLength```, registered the same way, and to let ```.<>``` apply to an array type as it does to a primitive - so ```[].<T>.<{ minLength: 1 }>``` is an array of at least one element, checked at the same assignment, argument, and return boundaries the other metadata is. Rejecting it and pushing every length constraint into a ```where``` clause on the containing type was considered and set aside, because it makes the common non-empty-list case verbose and asymmetric with strings.

### Metadata on reference types

I haven't put any thought into generalizing this to classes.

### Wouldn't a compile-time SMT-lite solver be potentially very expensive to run?

For practical cases a simple memoization for each type or pair of types negates most of the cost. It's possible to engineer situations where a timeout is required for compile-time/editor calculations.
