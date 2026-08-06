# Ranges

Every language a JavaScript programmer is likely to reach for has a range: Rust's ```1..=6```, Swift's ```1...6```, Ruby's ```1..6```, Kotlin's ```1..6```, Julia's ```1:6```, Python's ```range(1, 7)```. JavaScript has ```for (let i = 0; i < n; ++i)``` and, if the ```Iterator.range``` proposal lands, a function call. It has no way to name an interval as a value, which is why ```Math.random(min, max)``` must encode inclusivity in a convention nobody can see, why ```slice``` and ```subarray``` take loose pairs of numbers, and why "is x between a and b" is written out by hand every time.

This document proposes range literals, gives them types, and then checks the result against every feature of this proposal it could touch. The last part is the point: a range is only worth adding if it makes the rest of the language smaller, and the test of that is whether existing features absorb it without special cases.

## The Lexical Problem

Any range operator built from ```.``` collides with the numeric literal grammar, because ```1.``` is already a complete NumericLiteral. Precisely:

```js
1..<6          // SyntaxError today: NumericLiteral `1.` then `.` then `<`
0..<10         // SyntaxError today
a..<b          // SyntaxError today: `a.` is not a valid member access
1..=6          // SyntaxError today
1<..<6         // SyntaxError today
1...6          // SyntaxError today

1..toString()  // Parses today. `1.` then `.toString()`. Evaluates to "1"
1..x           // Parses today. `(1.).x`, which is undefined
```

So the ```..``` family is free. The one thing standing in the way is an integer literal followed by ```..``` followed by an identifier.

**The rule.** A ```.``` is not consumed as the decimal point of a NumericLiteral when the next character is also a ```.```. That single change makes ```1..<6```, ```1..=6```, and ```1..``` all lex correctly, and it is the whole of what the family costs the numeric grammar: ```1.```, ```1.5```, ```.5```, and the spread ```...``` are untouched. The forms that mark their start need nothing from it at all - ```1<..<6``` ends its literal at the ```<```, before any question of a decimal point arises.

**What breaks.** Exactly one production: ```IntegerLiteral .. IdentifierName```, the ```1..toString()``` idiom. It doesn't break silently, and under the syntax below it doesn't break quietly either. Because there is no bare two-endpoint form, ```1..``` is already a complete from-range and an identifier cannot continue it, so the idiom is a SyntaxError rather than a range whose end is a call result. Every other spelling of the same idiom keeps working:

```js
(1).toString(); // Fine
1 .toString(); // Fine
1.0.toString(); // Fine
1..toString(); // SyntaxError: `1..` is a range and `toString` cannot continue it
```

Rust made floats require ```1.0``` for precisely this reason, and got ```..``` in exchange. The trade here is smaller: JavaScript's ```1..toString()``` is a golfing curiosity, not a pattern in production code, and dropping the bare two-endpoint form turned the one breakage from a runtime failure into a parse error.

**The alternatives, and why not.** ```...``` is free lexically, since spread only ever appears in prefix position, but Ruby's ```..```/```...``` pairing, where one extra dot silently flips inclusivity, is a well-documented footgun and ```...``` reads as spread to every JavaScript programmer alive. Contextual keywords (```1 to 6```, Kotlin's ```until```) need no lexical change at all and are the fallback if the committee finds the breakage unacceptable, but they read poorly in index position (```arr[2 to 5]```) and interact badly with ASI.

**A correction on ```..<```.** An earlier draft ruled ```..<``` out on the grounds that this proposal already gives ```.<``` to generic application, so ```1..<6``` would begin parsing a type argument list. That was wrong, and the syntax below depends on its being wrong. The two are decided by their **second** character - ```.<``` requires a ```<``` there and ```..<``` requires a ```.``` - so no input ever reaches a state where both are live, and the third character is never consulted to tell them apart. The old objection was reaching for a real hazard about a different pair: with no dedicated token, ```a..<b``` would lex as ```(a..) < b```, a collision with the from-range rather than with generics. Making ```..<``` a token is what resolves it, and the whitespace note below is where the resolved reading still shows.

## Syntax

```
RangeExpression :
    ShortCircuitExpression? ..< ShortCircuitExpression
    ShortCircuitExpression? ..= ShortCircuitExpression
    ShortCircuitExpression <..< ShortCircuitExpression
    ShortCircuitExpression <..= ShortCircuitExpression
    ShortCircuitExpression ..
    ShortCircuitExpression <..
    ..
```

Nine forms. **Every range that has an end says whether it includes it, and a start is marked only when it is exclusive**, because every language that has ranges spells the start inclusive and the disagreement between them is entirely about the end:

```js
a..<b   // [a, b)   half-open
a..=b   // [a, b]   inclusive
a<..<b  // (a, b)   open
a<..=b  // (a, b]   open start, inclusive end
a..     // [a, ∞)   from
a<..    // (a, ∞)   from, exclusive
..<b    // (-∞, b)  to
..=b    // (-∞, b]  to, inclusive
..      // full
```

**There is no ```a..b``` and no ```..b```.** A two-dot form with an unmarked end does not exist, so nothing rests on remembering a default - and there is no default to remember. Rust reads ```a..b``` half-open; Kotlin, Ruby, Perl, Pascal, Ada, and Haskell read it inclusive. Among two-dot languages the majority reading is the opposite of the one this proposal wants, and two experienced readers disagree about the same four characters with equal confidence. Raku is the one language that resolves this the way this document does, with all four two-endpoint forms marked (```1..10```, ```1^..10```, ```1..^10```, ```1^..^10```), and it kept the bare inclusive form beside them; the corpus here is 64% half-open, so the bare form buys the minority reading a spelling it does not need. ```0..<array.length``` remains the loop everyone writes, and what makes it right is now that the end's exclusivity is written rather than assumed.

Bare ```..``` survives only where no endpoint's inclusivity is in question: ```..``` itself, and the tail of ```a..``` and ```a<..```, where a missing end has no inclusivity to state.

**Six tokens**, each a single token under longest match: ```..```, ```..<```, ```..=```, ```<..```, ```<..<```, ```<..=```. They are decided by a fixed lookahead, and never collide with each other or with anything the language already spells:

| at | second | third | token |
|---|---|---|---|
| ```.``` | ```.``` | ```.``` | ```...``` spread |
| ```.``` | ```.``` | ```<``` | ```..<``` |
| ```.``` | ```.``` | ```=``` | ```..=``` |
| ```.``` | ```.``` | anything else | ```..``` |
| ```.``` | ```<``` | | ```.<``` type argument list |
| ```.``` | anything else | | ```.``` member access, or a decimal point |
| ```<``` | ```.``` | ```.``` then ```<``` | ```<..<``` |
| ```<``` | ```.``` | ```.``` then ```=``` | ```<..=``` |
| ```<``` | ```.``` | ```.``` then anything else | ```<..``` |
| ```<``` | ```<``` | | ```<<```, ```<<=``` |
| ```<``` | ```=``` | | ```<=``` |
| ```<``` | anything else | | ```<``` relational |

The ```.``` rows are what make ```..<``` and ```.<``` independent. The ```<``` rows are affordable only because generic application is spelled ```.<```: a bare ```<``` in this language is always relational or a shift, so claiming ```<..``` costs exactly one reading - comparing something against a range - which is a TypeError in any case.

**Whitespace is significant at the family's edges**, as it already is at ```x+++y```:

```js
a..<b     // the half-open range
a.. < b   // (a..) < b, a comparison against a from-range
a<..      // the open-start from-range
a < ..    // a compared against the full range
a <.. < b // (a<..) < b, and not the open range: `<..<` is one token
```

None of the spaced readings is a program at all: a range binds **looser** than the relational operators, so a range can never be a relational operand and each of these is a SyntaxError. The last line is the one to read twice: spacing ```<..<``` apart does not give the open range, it gives a comparison against ```a<..``` that the grammar has no production for.

Parentheses put a range back under the relational operators, and there a range is **rejected** as a relational operand: ```(a..) < b``` and ```(0..<3) < 5``` are TypeErrors. A range does not implement ```Ordered```, so the comparison is meaningless, and the base language would otherwise answer it with ```false``` - which is worse than an error because it is invisible. The unspaced spellings are already SyntaxErrors, so this rule reaches only the parenthesized forms.

**One existing lookahead grows by a character.** ```?.``` is already not the optional chaining punctuator when a decimal digit follows, so that ```a?.5:b``` stays a ternary. It is now also not the punctuator when a ```.``` follows, so that ```cond?..<b:c``` is a ternary over a to-range rather than ```?.``` and a stray ```.<```. The extension is free: ```?.``` followed by ```.``` is a SyntaxError today in every case, because the punctuator must be followed by an identifier, ```(```, ```[```, a template, or a private name. It gives meaning to input that had none, which is the trade the digit half of the rule already makes.

The family binds tighter than assignment and looser than ```||``` and ```??```, so its operands are ShortCircuitExpressions, and it is **non-associative**: ```a..<b..<c``` is a SyntaxError rather than a puzzle. Member access binds tighter than the range, so ```0..<arr.length``` is ```0..<(arr.length)``` and the whole range is parenthesized to reach its own members:

```js
(0..<10).length; // 10
```

Two hazards worth writing into the grammar notes. A statement beginning with ```..```, ```..<```, ```..=```, or ```<..``` continues the previous expression, joining ```(```, ```[```, and ```` ` ```` on the list of ASI hazards; ```<..``` is the new entry. And a range is an expression, so it appears in ```case``` labels, object literals, and argument lists without further ceremony - and stands as a pattern in a ```when``` clause of [pattern matching](patternmatching.md), matching by the same containment a range ```case``` label uses.

## Types

A range is a value type class, so ```0..<10``` allocates nothing and copies by value. The endpoint kinds are distinct types, because an omitted endpoint is a different shape, not a missing value:

```js
enum Bound: uint8 { Closed, Open }; // Exposed as Range.Bound

interface Ordered<T> {
	operator<(other: T): boolean;
}

interface Scalable<T> {
	operator*(factor: float64): T; // Scalar multiplication, preserving T
}

interface Arithmetic<T> {
	operator+(other: T): T;
	operator-(other: T): T;
	operator-(): T; // Negation
	operator*(other: T): T;
	operator/(other: T): T;
}

interface RangeBounds<T: Ordered.<T>> {
	// The shape-independent view: an endpoint and its bound, or null where the
	// shape has neither. This is what code over an arbitrary range reads.
	get start(): T | null;
	get end(): T | null;
	get startBound(): Bound | null;
	get endBound(): Bound | null;

	get isEmpty(): boolean; // No value falls within it
	get isFull(): boolean;  // Every value of T does
	contains(value: T): boolean;
	contains(range: RangeBounds.<T>): boolean;
	intersect(other: RangeBounds.<T>): RangeBounds.<T>;
}

// Scaling needs arithmetic, and ordering does not imply it: Temporal.Instant is
// Ordered and cannot be multiplied. So scale exists on the instantiations whose
// element type scales, by the partial specialization rule in generics.md.
partial interface RangeBounds<T: Scalable.<T>> {
	scale(factor: float64): RangeBounds.<T>;
}

// Interval arithmetic, on the ranges whose element type has arithmetic. The
// operands are ranges and so is the result: this is what a bound on a computed
// value is, and the [metadata](primitivemetadata.md) extension's numeric
// operators are its first caller.
partial interface RangeBounds<T: Arithmetic.<T>> {
	operator+(other: RangeBounds.<T>): RangeBounds.<T>;
	operator-(other: RangeBounds.<T>): RangeBounds.<T>;
	operator-(): RangeBounds.<T>;
	operator*(other: RangeBounds.<T>): RangeBounds.<T>;
	operator/(other: RangeBounds.<T>): RangeBounds.<T>;
}

class Range<T: Ordered.<T>, S: Bound = Bound.Closed, E: Bound = Bound.Open> implements RangeBounds.<T> {
	readonly start: T;
	readonly end: T;
	get length(): uint32;
	get interval(): Interval; // Derived from S and E, never a parameter
	contains(value: T): boolean;
	step(by: T): Iterator.<T>;
	reverse(): Iterator.<T>;
	*operator...(): Iterator.<T>;

	// The bounds are value generics, not arguments. A runtime bound would
	// defeat the specialization the type exists to provide.
	//
	// Whether `of` should exist at all is OPEN. The case for it is narrow: a
	// literal fixes its bounds by its own marker, so constructing a range in
	// code generic over its bounds - `function widen<S: Bound, E: Bound>(a, b)
	// { return Range.of.<S, E>(a, b); }` - has no literal spelling, and without
	// it such a function returns a union of four types rather than
	// `Range.<T, S, E>`. The case against is that bound-generic code almost
	// always DERIVES a range rather than constructing one, and derivation is
	// already covered: `intersect`, `scale`, and the arithmetic are
	// bound-polymorphic by construction, so shifting a range is
	// `r + (by..=by)`, which preserves its bounds exactly. What is left is
	// construction from two loose endpoints at bounds named by type
	// parameters, which is rare enough that the member may not earn its
	// keep.
	static of<T: Ordered.<T>, S: Bound, E: Bound>(start: T, end: T): Range.<T, S, E>;
}

class RangeFrom<T: Ordered.<T>, S: Bound = Bound.Closed> implements RangeBounds.<T> {
	readonly start: T; // `end` and `endBound` are null
}

class RangeTo<T: Ordered.<T>, E: Bound = Bound.Open> implements RangeBounds.<T> {
	readonly end: T; // `start` and `startBound` are null
}

class RangeFull<T: Ordered.<T>> implements RangeBounds.<T> {}
```

```..``` has no endpoint to infer from, so ```RangeFull.<T>``` takes its ```T``` from the context that consumes it, and is ```RangeFull.<any>``` where there is none.

```T``` is bounded by ```Ordered.<T>```, an interface whose one member is ```operator<``` (the operator-bearing interface pattern from [operator overloading](operatoroverloading.md)), so ranges work over the integer types, the float types, ```bigint```, ```decimal128```, ```rational```, dimensioned quantities, and ```Temporal.Instant``` - anything with an ordering.

**Each endpoint's bound lives in the type**, one parameter per endpoint, so ```1..=6``` is a ```Range.<uint8, Bound.Closed, Bound.Closed>``` and a function taking a range specializes on its inclusivity with no branch surviving to runtime. Every marker in the syntax sets exactly one parameter - the start marker sets ```S```, the end marker sets ```E``` - so a literal desugars one endpoint at a time, and the defaults reproduce the unmarked spellings. All four two-endpoint intervals now have literals, so ```Range.of``` is no longer how any of them is written.

**Why two parameters rather than one four-valued interval kind.** An earlier draft carried a single ```I: Interval```, and the reasons for splitting it are worth recording, because the four states are the same either way. A one-ended shape has one endpoint, so a four-valued parameter gives it a half that means nothing: ```RangeFrom.<T, Interval.OpenClosed>``` would be a legal type that no value can inhabit and that an overload could name, which is the type-level twin of the contradictory metadata records this proposal exists to make unwritable. Every operation over a range decomposes the interval at its first line anyway, since ```contains``` is one comparison per endpoint. A per-endpoint operation becomes a substitution rather than a computation over an enum, so a future ```withEnd.<E2: Bound>(value: T): Range.<T, S, E2>``` needs no type-level function. A signature can quantify one endpoint and fix the other - ```Range.<T, Bound.Closed, E>``` is "definite inclusive start, either end" in one overload, where a single parameter would need a union of two interval values. And ```Bound.Open``` sits at the endpoint it describes, where ```OpenClosed``` asks a reader to remember which side is which. Julia's ```IntervalSets``` carries the two bounds as type parameters for the same reasons; Rust and Swift avoid the question by giving each interval its own type, which this proposal cannot do without giving up specialization.

**```Interval``` survives as derived vocabulary, never as a parameter.** The four-way name is still the useful one for display, reflection, and a ```switch``` over shapes, so it stays as an enum and as an accessor computed from the two bounds:

```js
enum Interval: uint8 { Closed, ClosedOpen, OpenClosed, Open }; // Exposed as Range.Interval

// And the aliases, so no annotation is forced through the three-argument spelling:
type ClosedRange<T> = Range.<T, Bound.Closed, Bound.Closed>;   // a..=b
type ClosedOpenRange<T> = Range.<T, Bound.Closed, Bound.Open>; // a..<b
type OpenClosedRange<T> = Range.<T, Bound.Open, Bound.Closed>; // a<..=b
type OpenRange<T> = Range.<T, Bound.Open, Bound.Open>;         // a<..<b
```

The aliases and the accessor share the enum's names, so the language has one vocabulary for the four intervals and not two. A diagnostic should prefer them: ```ClosedRange.<uint8>``` or "a closed range of ```uint8```" reads where ```Range.<uint8, Bound.Closed, Bound.Closed>``` does not, and an implementation that prints the raw parameterization where an alias exists is doing its reader no favors.

Element types come from literal propagation, as anywhere else:

```js
const a = 1..=6; // ClosedRange.<number>
const b: ClosedRange.<uint8> = 1..=6; // uint8 endpoints
0n..<10n; // ClosedOpenRange.<bigint>
Meter(0)..<Meter(10); // ClosedOpenRange.<float32.<{ m: 1 }>>
```

**An endpoint may be infinite, and that is not the same as having none.** ```0..<Infinity``` and ```0..``` contain the same values and iterate the same - an infinite end stops the iteration nowhere, exactly as an absent end does - but they are different shapes and say so: the first has an ```end```, an ```endBound``` of ```Bound.Open``` and an ```interval``` of ```Interval.ClosedOpen```, where the second reports none of the three. Only the absent form is ```isFull``` when both sides are missing, so ```-Infinity..<Infinity``` is not ```..```. Both spellings are allowed because a computed endpoint may *be* infinite without an author choosing it - a bound derived from a division, or a length that overflowed - and refusing would turn a survivable program into a throw. Write ```0..``` when you mean "no upper bound"; ```0..<Infinity``` is what a computation hands you.

**```length``` adjusts once per open endpoint.** Over a bounded integer range it is ```end - start + 1``` less one for each endpoint that excludes its own value, so ```0..=10``` has 11 members, ```0..<10``` and ```0<..=10``` have 10, and ```0<..<10``` has 9; it is never negative. **```isEmpty``` reads the bounds at equal endpoints**: ```5..=5``` holds exactly one value, while ```5..<5```, ```5<..=5```, and ```5<..<5``` hold none, since an open endpoint excludes the only value the interval could contain.

Descending ranges are **empty**, not reversed: ```10..<0``` contains nothing, and ```(0..<10).reverse()``` is how you count down. This is Rust's rule and it exists because the alternative silently produces a loop that runs backwards when a variable goes negative.

**The interface's operations are total across the four shapes**, because the interface is what a consumer of an arbitrary range holds: the [metadata](primitivemetadata.md) bounds field is a ```RangeBounds.<T>```, and narrowing reaches it through ```..``` where there is no bound yet.

```contains``` overloads on a value and on a range. Against a range it is the subset test, so an empty range is contained in every range and ```..``` contains them all. The two overloads cannot be ambiguous: a range is not ```Ordered```, so ```T``` is never itself a ```RangeBounds```.

```intersect``` is the point-set intersection, and it is what makes flow narrowing monotone rather than a rule per comparison operator. It is commutative and associative with ```..``` as its identity. Where two ranges share an endpoint the exclusive bound wins, since ```0..<10``` and ```0..=10``` agree everywhere below 10 and disagree only at it. Where they are disjoint the result is the crossed pair, the greater start with the lesser end, which is descending and therefore empty by the rule above - so an empty intersection needs no representation of its own. The result's shape depends on the operands, which is why the return type is the interface rather than a ```Range```: intersecting a ```RangeFrom``` with a ```RangeTo``` produces a ```Range```, and intersecting anything with ```..``` produces the other operand.

```scale``` multiplies both endpoints. A negative factor swaps them **and** swaps their bounds, since the image of ```[a, b)``` under negation is ```(-b, -a]```, and it swaps the one-ended shapes with each other: ```a..``` scaled by -1 is ```..=-a```. A zero factor is the case the obvious rule gets wrong - multiplying both endpoints of ```0..<10``` gives the empty ```0..<0```, where the image of a nonempty range under multiplication by zero is the single point zero - so scaling by zero yields ```0..=0```, and leaves an already-empty range empty. The unit conversions this exists for are never zero or negative; the rules are written down because the method is general.

**The arithmetic operators are interval arithmetic**, the bounds of a computed value given the bounds of what it was computed from. An unbounded side propagates as an unbounded side, so the shapes fall out rather than being special cases, and a result nothing can be said about is ```..```:

```js
(1..=3) + (10..=20);  // 11..=23
(1..=3) - (10..=20);  // -19..=-7, the cross: low minus high
-(1..<3);             // -3<..=-1
(2..) * (3..);        // 6.., a lower bound from two lower bounds
(1..=2) / (0..=4);    // .., a divisor that can be zero says nothing
```

Addition adds the corresponding endpoints; subtraction crosses them, the result's low being the left's low minus the right's **high**; negation reflects, as ```scale(-1)``` does. For all three, **a result bound is exclusive where either contributing bound is**: ```(3..) + (5<..)``` is ```8<..```, because the right operand approaches 5 without reaching it, so the sum approaches 8 without reaching it.

Multiplication is the four products of the endpoints, the least and the greatest of them being the result's. Its exclusivity rule is the one that repays stating: **a result bound is exclusive only where every product attaining it involves an exclusive source bound.** For ```(0..=1) * (0..<2)``` the least product is zero, attained both by ```0 * 0``` from two inclusive endpoints and by ```0 * 2``` touching an exclusive one - and because one attaining combination is inclusive, zero is reached and the low bound is closed. The greatest product is two, and it is attained only by ```1 * 2```, whose right endpoint is exclusive: the product approaches two without reaching it, so the high bound is open and the result is ```0..<2```. Where the operands are not both fully bounded, multiplication propagates what it can: two non-negative lower bounds still give a lower bound, which is the ```(2..) * (3..)``` line above.

Division is multiplication by the reciprocal, and it is defined only where the divisor is bounded away from zero - a strictly positive low or a strictly negative high. A divisor whose range merely excludes zero at an open endpoint, like ```0<..=1```, is not enough: its values approach zero, so the quotient is unbounded and the result is ```..```.

## Random

The [random extension](random.md) takes a range as its only bounds parameter, which is what motivated this document. ```Math.random.<uint8>(1..=6)``` is a die, ```Math.random.<float32>(0..<1)``` is the unit interval, and ```Math.random.<uint8>(..)``` is the type's full range. The interval lives in the range's type, so the four cases specialize with no runtime branch, and there is no convention to remember or misdocument.

## Ranges as Types

Here is where the feature pays for itself. A range literal with constant endpoints is a compile-time-evaluable expression, and this proposal already says that such an expression is a valid type annotation. A range is also, exactly, the value ```NumberBounds``` carries:

```js
type Die = uint8.<{ bounds: 1..=6 }>;
type Unit = float32.<{ bounds: 0..<1 }>;
type Percent = uint8.<{ bounds: 0..=100 }>;
type Angle = float32.<{ bounds: 0..<6.283185307 }>;
type Positive = float32.<{ bounds: 0<.. }>;
```

The range is written under the key that says what it means. There is no bare ```uint8.<1..=6>```: dispatch in the [metadata](primitivemetadata.md) system is by claimed key, and a bare range carries no key to route by, so admitting one would mean either a second dispatch rule or a second spelling of ```{ bounds: r }```. What the metadata gains in exchange is that a constraint has one form: the four JSON Schema fields it replaces could say the same thing two ways and could contradict themselves, and a range can do neither.

One notation now spells an interval in all four places a program needs one - as a value, as a constraint, as a random source, and as the bound the arithmetic on a constrained value carries - and they agree by construction:

```js
type Die = uint8.<{ bounds: 1..=6 }>;
const roll: Die = Math.random.<Die>(); // No arguments: the type is the range
JSON.parse.<{ roll: Die }>(body); // Validated during the parse
function play(value: Die) {} // Checked at the boundary
```

```Math.random.<Die>()``` produces a value that satisfies ```Die```'s constraint by construction, so the validation a cast would normally run at the return boundary is elided. The generator cannot produce anything else.

## Iteration

**A range iterates when it has a step.** Over the integer types the step is ```1``` and is implicit, which covers the loop everyone writes:

```js
for (const i of 0..<array.length) {} // The end is written, so it stops one short
for (const i of 0..=9) {} // 0 through 9
[...0..<5]; // [0, 1, 2, 3, 4]. Spread is prefix, so no ambiguity
```

```step``` supplies a step where there is none, and widens the stride where there is one. The step is a *difference*, so its type is the type of ```end - start```: the element type itself for a numeric range, and a ```Temporal.Duration``` for a range of instants.

```js
(0..<10).step(2); // 0, 2, 4, 6, 8
(0.0..<1.0).step(0.25); // 0, 0.25, 0.5, 0.75
(start..<end).step(Temporal.Duration.from('PT1H')); // Every hour
(0..).step(3).take(4); // 0, 3, 6, 9
```

**```step``` returns an ```Iterator```, not a stepped range**, and the reason is algebraic rather than ergonomic. ```RangeBounds``` is an *interval* abstraction: ```contains```, ```intersect```, ```scale```, and the arithmetic operators are all point-set operations, and intersection is **closed** over intervals - which is what makes narrowing monotone. A step turns the point set into a lattice, and intersection is not closed over lattices: the multiples of 2 in ```0..<10``` intersected with the multiples of 3 are the multiples of 6, which no single stepped range with one step expresses. A stepped range would therefore make ```intersect``` either wrong or union-valued. Kotlin can afford ```IntProgression``` with ```contains``` precisely because it has no interval arithmetic to keep closed; Rust's ```step_by``` returns an iterator, as here.

The nth value is ```start + n * by``` rather than the previous value plus ```by```, and iteration stops when that value reaches ```end```, tested against the value and not against a computed count. Both halves matter for a float range: ```(end - start) / by``` can round up and yield one value too many, and repeated addition accumulates error.

**An open start begins one step in.** The nth value is counted from ```n = 0``` where the start is inclusive and from ```n = 1``` where it is exclusive, because an open start excludes its own endpoint: ```0<..<4``` yields 1, 2, 3, and ```(0<..<1).step(0.25)``` yields 0.25, 0.5, and 0.75. The rule is stated because the arithmetic above was written when a closed start was the only start a literal could spell, and an open one changes where the count begins rather than how each value is computed.

```js
[...(0.0..<1.0).step(0.1)]; // Ten values, ending at 0.9
```

Repeated addition would have ended at ```0.8999999999999999```. Individual values still round, so the fourth is ```0.30000000000000004```. The rule prevents accumulation, not rounding.

A range whose element type has no natural unit, which is every type but the integers, is a TypeError to iterate without a step rather than a silent choice of ```1```:

```js
// for (const x of 0.0..<1.0) {} // TypeError: a float range has no implicit step
for (const x of (0.0..<1.0).step(0.1)) {}
```

```RangeTo``` and ```RangeFull``` are never iterable, having no start. ```RangeFrom``` is unbounded, so the iterator helpers are how it is used.

A range is an ordinary iterable, so the [standard library](standardlibrary.md)'s iterator helpers apply, and materializing one is a spread or a ```toArray```. The element type propagates into a typed array:

```js
const a: [].<uint32> = [...0..<10]; // A typed array of uint32
(0..<10).map(i => `item${i}`).toArray(); // [].<string>
Array.from(0..<10, i => i * i);
(0..).take(10).toArray();
```

Because ```Range``` is a value type and ```*operator...()``` is an ordinary iteration operator, ```for (const i of 0..<n)``` inlines to a counted loop with no iterator object allocated. It is the C-style loop, spelled once.

## Indexing and Slicing

This proposal already has user-defined index operators. A range-taking overload is all that slicing needs, and it lands on the ```window``` method the arrays section recently gained:

```js
class Array<T> {
	get operator[](range: RangeBounds.<uint32>): [].<T>; // Aliases, doesn't copy; a constant range refines to a fixed extent (below)
}

array[2..<5]; // Elements 2, 3, 4 as a view
array[2..=5]; // Elements 2 through 5
array[2..]; // The tail
array[..3]; // The head
array[..]; // The whole thing
```

```array[a..<b]``` and ```array.window(a, b)``` are the same operation, and the range form should be the one people write. Two consequences fall out for free. When the endpoints are compile-time constants the length is too, so the result is a fixed-extent view whose type says so:

```js
const head: [3].<uint8> = bytes[0..<3]; // The length is known
```

And the SIMD pass in the [entity component system](examples/ecs.md) example loses its last piece of arithmetic:

```js
const lanes = [].<float32x4>(vx[0..<whole]);
```

A runtime start with a compile-time length still wants ```window.<N>(start)```, since ```start..<start + Words``` requires symbolic arithmetic to see that the length is constant. That is an honest limit, not a gap: the two forms coexist.

## Containment, switch, and `is`

```(1..=6).contains(x)``` is the containment test. It is deliberately **not** spelled ```x in 1..=6```, Kotlin's way, because ```in``` already means "has this property" and ```3 in someRange``` is currently legal and false; overloading it would change the meaning of running code rather than of a syntax error.

Instead, containment reaches the type system, which is the more useful direction:

```js
if (x is uint8.<{ bounds: 1..=6 }>) {} // Structural test against the bounded type
if ((1..=6).contains(x)) {} // The value-level test
```

Range case labels extend ```switch``` the way sealed classes did. When a case label is a range, the clause matches if the range contains the discriminant. It is the same generalization already made for type-object labels, and it costs no new concept:

```js
switch (statusCode) {
	case 200..<300: return Result.Ok;
	case 400..<500: return Result.ClientError;
	case 500..<600: return Result.ServerError;
	default: return Result.Unknown;
}
```

Exhaustiveness is not claimed for range labels. Deciding whether a set of intervals covers a type is a different analysis from listing an enum's members, and this proposal's stated position is that exhaustiveness belongs to enums and sealed classes. Ranges don't change that.

## The Rest of the Language

- **Dimensioned quantities.** ```Meter(0)..<Meter(10)``` is a ```Range.<Meter>``` because ```float32.<D>``` defines ```operator<```. Mixing dimensions is the same TypeError it is anywhere: ```Meter(0)..<Second(10)``` fails at the range's construction.
- **Strings do not get it.** ```a[start..<end]``` is a *view*, which is the whole of why it is an operator rather than a method, and a string cannot give one - ```str[1..<3]``` could only copy. Four characters costing O(1) on one receiver and O(n) on another, with no signal at the call site, is the trap this proposal avoids elsewhere; ```slice``` already says "copy" in its name. Underneath sits a second trap: a string range index would have to choose code units or code points, and Rust's ```&s[1..3]``` panicking on a non-boundary index is the known result of choosing.
- **Temporal.** ```start..<end``` over ```Temporal.Instant``` works, and it subsumes the ad-hoc ```Interval``` record in [temporal.md](temporal.md). Like any non-integer range it needs a step, and ```(start..<end).step(Temporal.Duration.from('PT1H'))``` reads better than the loop it replaces. A non-empty requirement stays a ```where``` clause, since an empty range is a legal value, not an error.
- **decimal and rational.** Ordered, so ranged. ```Math.random.<decimal128.<{ scale: 2 }>>(0..=100)``` quantizes at the return boundary like any other decimal.
- **Dependent record types.** ```where (0..=150).contains(this.age)``` reads as the constraint it is, and the bounded-type spelling ```age: uint8.<{ bounds: 0..=150 }>``` is better still, since it validates at every boundary rather than at record construction.
- **Value type references.** ```for (const ref p of particles[0..<n])``` composes: the slice is a view, and reference iteration over a view is reference iteration.
- **SoA.** ```mesh.fields.position[0..<count]``` is the upload slice.
- **Destructuring.** ```const { start, end } = 0..<10``` works, because those are the fields.
- **Spread.** ```f(...0..<3)``` passes ```0, 1, 2```. Legal, since spread is prefix and the range is its operand.

## What This Costs

One lexical rule, one broken idiom, one family of six tokens, four small classes with three interfaces over them, and one operator overload on arrays. Two characters are added to every range that has an end, and they are added where the languages disagree rather than where they agree. In exchange: the C-style loop, ```slice```'s argument pairs, the inclusivity convention in ```Math.random```, the four JSON Schema bound fields and the collision rule they needed, the ```window``` method, ```Iterator.range```'s common case, and ```temporal.md```'s ```Interval``` record all become one notation.

The completeness test this document set for itself was whether existing features absorb ranges without special cases. They do, with two exceptions worth naming rather than hiding: containment cannot use ```in``` without breaking legal code, and exhaustiveness over ranges is not offered. Both are refusals, not gaps.

## Other Proposals

**```Iterator.range```.** These are different kinds of thing rather than competing spellings. ```Iterator.range(start, end, optionOrStep)``` returns an iterator, which is consumed once. ```a..<b``` is a value, which can be iterated any number of times, stored in a field, passed to ```Math.random```, and used as a type. So there is no conflict, only an overlap, and the two should agree where they overlap:

```js
Iterator.range(0, 10); // The values of (0..<10)[Symbol.iterator]()
Iterator.range(0, 10, 2); // The values of (0..<100).step(2)
```

```Iterator.range``` is half-open, which is what ```a..<b``` writes out; both treat a descending pair as empty, and both admit ```bigint```. Once ranges exist, ```Iterator.range``` is a function spelling of a literal, including for computed bounds, since ```a..<b``` takes expressions. That is a duplication worth deciding about rather than a conflict to resolve, and this document takes no position beyond asking that the values agree.

**A static ```range``` on typed arrays**, as in the ```uint32[].range(0, 10)``` that earlier drafts of this idea proposed, is declined. It is a second way to write ```0..<10``` that also materializes an array nobody asked for. ```[...0..<10]``` and ```(0..<10).map(f).toArray()``` cover it, and both propagate their element type.

**Slice notation**, ```a[1:3]```, is subsumed. ```a[1..<3]``` is the same operation through the index operator, and it composes with the other range forms, so ```a[..]``` and ```a[2..]``` come free. Two spellings of a slice would be worse than one.

**Pattern matching** gains range patterns from this document rather than needing its own, since a case label is an expression and a range is a value.

**The pipeline operator**, if it advances, binds looser than ```..```, so ```0..<10 |> f``` pipes the range rather than piping ```10```.

**Decimal literals and numeric separators** are unaffected. The lexical rule reads a ```.``` as part of a number only when the next character is not another ```.```, and neither ```1_000..<2_000``` nor a suffixed decimal literal has two dots in a row.

## Open Questions

- Whether a stepped range should be reachable as a value - a ```Progression```, Kotlin's name for it - so that a step can be carried and asked ```contains```. This is deliberately not ```step```'s return type, for the reason given under Iteration; it would be a separate type whose containment is lattice membership rather than interval membership.
- ```Range.of.<S, E>(a, b)``` needs its bounds as compile-time constants, so choosing a bound at runtime produces a union of range types, which a ```switch``` handles and which is rare enough that no sugar is proposed.

