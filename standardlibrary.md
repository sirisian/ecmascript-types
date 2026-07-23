# Standard Library Signatures

This document collects typed signatures for the standard library's generic methods. These are signature listings rather than new features: every method here already exists, and the signatures state how element and key types flow through, so fully typed call sites infer their callbacks and engines can specialize the loops. Declarations use ```<...>``` and applications use ```.<...>``` per the main proposal, and the dotted ```function Object.groupBy<...>``` form below is spec-style shorthand for adding a typed overload to an existing function, as used in the primitive metadata document for ```Math.sqrt```. The same dotted form names a nested class, as ```class Temporal.Instant``` does in the [temporal](temporal.md) extension.

## Numeric Library

The Math functions are overloaded over the numeric types with declared, checked returns. The normative listing lives in the specification's numeric library clause; the rules it follows:

- Every signature takes all of its numeric parameters at one type ```T```. Mixing two typed widths matches no signature and is a TypeError; the conversion is written. A literal beside a typed argument takes the parameter's type where it fits, so ```Math.max(a, 3)``` on a ```uint8``` is a ```uint8``` and ```Math.max(a, 300)``` does not compile.
- A declared return is a checked boundary, so an integer result that does not fit throws a RangeError. ```Math.pow(uint8(2), uint8(10))``` throws where ```uint8(2) ** uint8(10)``` wraps to 0 - the same split as ```Math.addChecked(a, 1)``` against ```a + 1```, and deliberate: the operator is the cheap wrapping form, the named function the checked one. A float result rounds at its own width and overflows to an infinity, as float arithmetic already does; a decimal result raises a RangeError outside its exponent range, as decimal arithmetic already does.
- ```Math.sqrt``` and ```Math.cbrt``` on an integer type are the integer roots, truncated toward zero exactly as integer ```/``` truncates, and as [BigInt Math](https://github.com/tc39/proposal-bigint-math) specifies for ```BigInt.sqrt``` and ```BigInt.cbrt```; ```Math.sqrt``` of a negative integer value is a RangeError, and integer ```Math.pow``` refuses a negative exponent. The bigint rows match that proposal value for value, so the two compose if it advances. The transcendentals have no integer counterpart and no integer rows; ```Math.sin(float64(n))``` writes the promotion it means.
- ```Math.clz``` counts leading zeros at the argument's own width for every sized integer type, so ```Math.clz(uint8(1))``` is 7; it agrees with ```Math.clz32``` at 32 bits and replaces it in most typed code, ```clz32``` remaining the 32-bit field count. It takes no ```bigint```, which has no width to count from the top of. ```Math.imul``` returns an ```int32``` whatever integer type its arguments carry; ```Math.clz32``` preserves its argument's type up to 32 bits; ```Math.fround``` and ```Math.f16round``` take only the float families; ```Math.random```'s typed form is ```Math.random.<T>()``` in the [random](random.md) extension; the constants stay ```number```.
- Annotating the destination of a literal call is a specialized call, not a conversion: ```const a: uint8 = Math.pow(2, 10);``` resolves at ```uint8``` and throws, while ```uint8(Math.pow(2, 10))``` is the untyped call followed by a wrapping cast and is 0. A typed argument under a different destination type needs a cast, whose placement picks the computation: ```float64(Math.sqrt(10n))``` is the integer root converted, exactly 3, and ```Math.sqrt(float64(10n))``` is the real root, 3.1622 and change.

The numeric predicates gain the same per-type answers, returning ```boolean```. ```isNaN``` and ```isFinite``` are constants at the integer, bigint, and rational types and value tests at the float and decimal types. ```Number.isNaN```, ```Number.isFinite```, ```Number.isInteger```, and ```Number.isSafeInteger``` answer for the mathematical value, so ```Number.isNaN(x)``` on a ```float32``` NaN is true and ```Number.isInteger(i)``` on an ```int32``` is true, where today both are false because a typed value is not a Number. ```Number.isSafeInteger``` of an ```int64``` past 2^53 is false because the value is out of safe range, not because a payload lost precision.

## Iterables

The iteration interfaces are the ```...``` operator from the main proposal's typed iteration section, expressed as interface requirements:

```js
interface Iterable<T> {
	*operator...(): T;
}

interface AsyncIterable<T> {
	async *operator...(): T;
}
```

An ```Iterator.<T>``` implements ```Iterable.<T>``` and is what generator functions and the helper methods below return.

## Iterator Helpers

```js
class Iterator<T> {
	map<U>(callback: (value: T, index: uint32) => U): Iterator.<U>;
	filter(callback: (value: T, index: uint32) => boolean): Iterator.<T>;
	take(limit: uint32): Iterator.<T>;
	drop(limit: uint32): Iterator.<T>;
	flatMap<U>(callback: (value: T, index: uint32) => Iterable.<U>): Iterator.<U>;
	reduce<U>(callback: (accumulator: U, value: T, index: uint32) => U, initial: U): U;
	reduce(callback: (accumulator: T, value: T, index: uint32) => T): T;
	toArray(): [].<T>;
	forEach(callback: (value: T, index: uint32) => void): void;
	some(callback: (value: T, index: uint32) => boolean): boolean;
	every(callback: (value: T, index: uint32) => boolean): boolean;
	find(callback: (value: T, index: uint32) => boolean): T | undefined;
}
```

```AsyncIterator.<T>``` mirrors these with callbacks allowed to return ```Promise```-wrapped results and the terminal methods returning promises, e.g. ```toArray(): Promise.<[].<T>, any>```.

A fully typed chain infers every callback parameter and can be fused into a single specialized loop with no intermediate allocation:

```js
function* f(): int32 { yield* [1, 2, 3]; }
const a: [].<int32> = f().map(x => x * 2).filter(x => x > 2).toArray(); // x: int32 inferred
```

## Grouping

```Object.groupBy``` produces property keys, so its key type is constrained to the property key types; ```Map.groupBy``` accepts any key type, using SameValueZero like ```Map``` itself:

```js
function Object.groupBy<K extends string | symbol, T>(
	items: Iterable.<T>,
	callback: (value: T, index: uint32) => K
): { [key: K]: [].<T> };

function Map.groupBy<K, T>(
	items: Iterable.<T>,
	callback: (value: T, index: uint32) => K
): Map.<K, [].<T>>;
```

```js
const groups = Object.groupBy([1, 2, 3, 4], (n: uint32) => n % 2 == 0 ? 'even' : 'odd');
// groups: { [key: string]: [].<uint32> }
```

## Set Operations

The result element type follows where elements can come from: ```intersection``` and ```difference``` draw only from ```this```, while ```union``` and ```symmetricDifference``` draw from both sides:

```js
class Set<T> {
	union<U>(other: Set.<U>): Set.<T | U>;
	intersection<U>(other: Set.<U>): Set.<T>;
	difference<U>(other: Set.<U>): Set.<T>;
	symmetricDifference<U>(other: Set.<U>): Set.<T | U>;
	isSubsetOf<U>(other: Set.<U>): boolean;
	isSupersetOf<U>(other: Set.<U>): boolean;
	isDisjointFrom<U>(other: Set.<U>): boolean;
}
```

When ```T``` and ```U``` are unrelated value types the compiler can constant-fold the answer: an ```intersection``` of a ```Set.<uint8>``` and a ```Set.<string>``` is empty without iterating, and ```isDisjointFrom``` is ```true```, since distinct value types share no values.

A ```Set``` of value type class instances deduplicates structurally, and a ```Map``` keyed on one compares its keys by value, per the keyed collections section of the main proposal.

## Promise Statics

```js
type PromiseSettledResult<R, E> = {
	status: string, // 'fulfilled' or 'rejected'
	value?: R,
	reason?: E
};

class Promise<R, E> {
	static withResolvers<R, E>(): {
		promise: Promise.<R, E>,
		resolve: (value: R) => void,
		reject: (reason: E) => void
	};
	static try<R, E>(callback: (...args: [].<any>) => R | Promise.<R, E>, ...args: [].<any>): Promise.<R, E>;
	static all<R, E>(promises: Iterable.<Promise.<R, E>>): Promise.<[].<R>, E>;
	static allSettled<R, E>(promises: Iterable.<Promise.<R, E>>): Promise.<[].<PromiseSettledResult.<R, E>>, undefined>;
	static any<R, E>(promises: Iterable.<Promise.<R, E>>): Promise.<R, AggregateError>;
	static race<R, E>(promises: Iterable.<Promise.<R, E>>): Promise.<R, E>;
}
```

Over a tuple of differently typed promises the combinators return tuples instead: ```Promise.all``` of ```[Promise.<uint8, Error>, Promise.<string, Error>]``` resolves to ```[uint8, string]```, matching the combinator inference note in the main proposal's typed promises section.

## Array.fromAsync

```js
function Array.fromAsync<T>(
	items: AsyncIterable.<T> | Iterable.<T | Promise.<T, any>>
): Promise.<[].<T>, any>;

function Array.fromAsync<T, U>(
	items: AsyncIterable.<T> | Iterable.<T | Promise.<T, any>>,
	mapFn: (value: T, index: uint32) => U | Promise.<U, any>
): Promise.<[].<U>, any>;
```

When the element type is a value type or a typed class, ```fromAsync``` and ```toArray``` allocate the contiguous typed storage directly as elements settle, the same single-pass property the serialization document relies on for JSON arrays.
