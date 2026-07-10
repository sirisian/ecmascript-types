# Standard Library Signatures

This document collects typed signatures for the standard library's generic methods. These are signature listings rather than new features: every method here already exists, and the signatures state how element and key types flow through, so fully typed call sites infer their callbacks and engines can specialize the loops. Declarations use ```<...>``` and applications use ```.<...>``` per the main proposal, and the dotted ```function Object.groupBy<...>``` form below is spec-style shorthand for adding a typed overload to an existing function, as used in the primitive metadata document for ```Math.sqrt```.

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
