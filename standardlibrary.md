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

## Identity

The wrapper that means *no wrapper*, used by the [higher-kinded](higherkindedtypes.md) iteration types and by anything else parameterized over one:

```js
type Identity<T> = T;
```

It is an ordinary generic alias rather than a built-in, because nothing about it is built in. ```Identity.<uint8>``` is ```uint8```.

## Iterables

The iteration interfaces are the ```...``` operator from the main proposal's typed iteration section, expressed as interface requirements. ```*operator...()``` is how a class declares ```[Symbol.iterator]```, so these are the same member the [iteration types](README.md) state:

```js
interface Iterable<T> {
	*operator...(): T;
}

interface AsyncIterable<T> {
	async *operator...(): T;
}
```

```Iterator<T, R, N>``` is the other half of the pair - it declares ```next```, returning an ```IteratorResult<T, R>``` - and ```IterableIterator<T, R, N>``` inherits both. A bare argument is the element type, so ```Iterator.<T>``` below is ```Iterator.<T, void, void>```.

## Iterator Helpers

The helpers are defined on the ```Iterator``` class, which declares that it implements ```IterableIterator<T, R, N>```. Every method returning an iterator returns the class, so a chain stays on the fast path: a declared implementation is checked at the declaration and by brand afterwards, where a hand-written iterator entering the chain pays one structural check on the way in.

```js
class Iterator<T, R = void, N = void> implements IterableIterator<T, R, N> {
	map<U>(callback: (value: T, index: uint64) => U): Iterator.<U> { /* … */ return undefined; }
	filter(callback: (value: T, index: uint64) => boolean): Iterator.<T> { /* … */ return undefined; }
	take(limit: uint32): Iterator.<T> { /* … */ return undefined; }
	drop(limit: uint32): Iterator.<T> { /* … */ return undefined; }
	flatMap<U>(callback: (value: T, index: uint64) => Iterable.<U>): Iterator.<U> { /* … */ return undefined; }
	reduce<U>(callback: (accumulator: U, value: T, index: uint64) => U, initial: U): U { /* … */ return undefined; }
	reduce(callback: (accumulator: T, value: T, index: uint64) => T): T { /* … */ return undefined; }
	toArray(): [].<T> { /* … */ return []; }
	forEach(callback: (value: T, index: uint64) => void): void { /* … */ }
	some(callback: (value: T, index: uint64) => boolean): boolean { /* … */ return false; }
	every(callback: (value: T, index: uint64) => boolean): boolean { /* … */ return false; }
	find(callback: (value: T, index: uint64) => boolean): T | undefined { /* … */ return undefined; }
}
```

```AsyncIterator<T, R, N>``` mirrors these with callbacks allowed to return ```Promise```-wrapped results and the terminal methods returning promises, e.g. ```toArray(): Promise.<[].<T>, any>```.

Two of these carry the shape the rest follow. ```map``` is the one that changes the element type, so its ```U``` is what every downstream callback infers from; ```toArray``` is the one that leaves the family, exchanging ```Iterator.<T>``` for ```[].<T>```, which is why a chain's annotation goes on the binding rather than on any step.

A fully typed chain infers every callback parameter and can be fused into a single specialized loop with no intermediate allocation:

```js
function* f(): int32 { yield* [1, 2, 3]; }
const a: [].<int32> = f().map(x => x * 2).filter(x => x > 2).toArray();
// f() is a Generator.<int32, void, void>, which satisfies IterableIterator<int32>
// map gives an Iterator.<int32> with x: int32; filter keeps it; toArray leaves for [].<int32>
```

## Building From an Iterable

```Array.from``` and ```Iterator.from``` carry an element type through. The first parameter is the same ```Iterable.<T>``` the grouping functions take, so a typed array, a collection, a generator and a string all reach it by the interface they already declare:

```js
function Array.from<T>(items: Iterable.<T>): [].<T>;

function Array.from<T, U>(
	items: Iterable.<T>,
	mapFn: (value: T, index: uint64) => U
): [].<U>;

function Iterator.from<T>(items: Iterable.<T>): Iterator.<T>;
```

The mapped overload takes its result from the callback, exactly as ```map``` does.

**A callback's index is ```uint64```, not ```uint32```**, and this document said ```uint32``` in twelve places before this change. The [index type](README.md) is one type for "every count a CONTAINER reports or accepts: an array's ```length```, its ```capacity```, an index used to read or write an element, the length of a view over a buffer, and the ```size``` of a keyed collection", and naming it once is what "keeps ```capacity``` comparable with ```length``` without a conversion". A callback's index is such a count — it indexes the very container being iterated — so writing ```uint32``` there made the one number a program is most likely to compare against a ```length``` the one number it could not. The specification's typed-statics clause already states ```uint64```, and the engine implements it; this brings the design into line rather than the reverse.

An **untyped** source yields an untyped result. Where ```T``` cannot be determined the call has no static type at all, rather than one naming ```any``` in the element position — ```[].<any>``` would claim a typed array where the program built an ordinary one.

```Array.of``` is the same element type gathered from arguments instead of from an iterable:

```js
function Array.of<T>(...items: T): [].<T>;
```

## Reading an Object's Own Properties

```js
function Object.keys(o: object): [].<string>;
function Object.values<V>(o: { [key: string]: V }): [].<V>;
function Object.entries<V>(o: { [key: string]: V }): [].<[string, V]>;
function Object.fromEntries<V>(entries: Iterable.<[string, V]>): { [key: string]: V };
```

```Object.keys``` answers strings whatever it is given, so it needs no type parameter. The other three carry the value type, and they are stated over an index signature rather than over a specific object type: a signature that named one would not describe the call a program actually writes, which is over an object whose properties are known individually. Where the argument's properties have differing types the value type is their union, which is what an index signature over that object already means.

**```Object.entries``` pairs a ```string``` with the value**, not a property-key union. The keys ```Object.keys``` reports are Strings — a Symbol-keyed property is not among them — so the pair's first position is ```string``` and not ```string | symbol```. ```Object.groupBy```'s key constraint is the other direction and is unaffected: it PRODUCES property keys, and those may be Symbols.

## Cloning

```js
function structuredClone<T>(value: T): T;
```

The clone has the type of what was cloned. This is the one signature in this document whose result depends on nothing but its argument, and it is stated because the alternative — leaving it ```any``` — loses a type across a call that is defined to preserve the value.

**Not yet implementable, and stated anyway.** ```structuredClone``` is a HTML specification function rather than an ECMAScript one, and the reference engine does not provide it: ```typeof structuredClone``` is ```'undefined'``` there. A signature is a claim that the function EXISTS, so this one must not be given to an engine that lacks it — a program written against it would type-check and then fail on the call. It is recorded because the design question is settled and only the host is missing; an implementation adds the row when the function is present, and not before.

## Grouping

```Object.groupBy``` produces property keys, so its key type is constrained to the property key types; ```Map.groupBy``` accepts any key type, using SameValueZero like ```Map``` itself:

```js
function Object.groupBy<K extends string | symbol, T>(
	items: Iterable.<T>,
	callback: (value: T, index: uint64) => K
): { [key: K]: [].<T> };

function Map.groupBy<K, T>(
	items: Iterable.<T>,
	callback: (value: T, index: uint64) => K
): Map.<K, [].<T>>;
```

```js
const groups = Object.groupBy([1, 2, 3, 4], (n: uint32) => n % 2 == 0 ? 'even' : 'odd');
// groups: { [key: string]: [].<uint32> }
```

## Keyed Collections

The keyed collections take type arguments and every member follows from them. Earlier drafts of this document typed only the set-algebra methods below, which left ```Map.<K, V>``` used in a dozen places across these documents and defined in none of them, and left ```size``` as the one count in the language with no type.

```js
class Map<K, V> implements Iterable<[K, V]> {
	constructor(entries?: Iterable.<[K, V]>);
	get size(): uint64;
	get(key: K): V | undefined { /* … */ return undefined; }
	set(key: K, value: V): Map.<K, V> { /* … */ return undefined; }
	has(key: K): boolean { /* … */ return false; }
	delete(key: K): boolean { /* … */ return false; }
	clear(): void { /* … */ }
	getOrInsert(key: K, value: V): V { /* … */ return undefined; }
	getOrInsertComputed(key: K, callback: (key: K) => V): V { /* … */ return undefined; }
	keys(): Iterator.<K> { /* … */ return undefined; }
	values(): Iterator.<V> { /* … */ return undefined; }
	entries(): Iterator.<[K, V]> { /* … */ return undefined; }
	forEach(callback: (value: V, key: K, map: Map.<K, V>) => void, thisArg?: any): void { /* … */ }
	*operator...(): [K, V];
}

class Set<T> implements Iterable<T> {
	constructor(values?: Iterable.<T>);
	get size(): uint64;
	add(value: T): Set.<T> { /* … */ return undefined; }
	has(value: T): boolean { /* … */ return false; }
	delete(value: T): boolean { /* … */ return false; }
	clear(): void { /* … */ }
	keys(): Iterator.<T> { /* … */ return undefined; }
	values(): Iterator.<T> { /* … */ return undefined; }
	entries(): Iterator.<[T, T]> { /* … */ return undefined; }
	forEach(callback: (value: T, value2: T, set: Set.<T>) => void, thisArg?: any): void { /* … */ }
	*operator...(): T;
}
```

```size``` is the **index type**, ```uint64```, the same type an array's ```length``` and ```capacity``` report. The reason is not the one that fixed the width for arrays - that argument is about a view's length coming from a buffer, and a collection has no view form - but the other property the index type exists for: one type for every count means a count from one container is comparable with a count from another. ```map.size < array.length``` is a sentence a program wants to write, and it is unwriteable if the two are different types, exactly as *a capacity is at least a length* is unwriteable if those two are.

```get``` answers ```V | undefined``` rather than ```V```, because a lookup that finds nothing answers ```undefined```; a binding of type ```V``` therefore does not take a lookup's result without a test. ```getOrInsert``` and ```getOrInsertComputed``` answer ```V``` and never ```undefined```, since they insert what they did not find.

```keys```, ```values``` and ```entries``` return ```Iterator``` rather than the ```IterableIterator``` interface, so a chain of iterator helpers stays typed: ```m.values().map(f).toArray()``` carries ```V``` through. There are no per-collection iterator types - no ```MapIterator```, no ```SetIterator``` - because the bare-argument shorthand already carries what TypeScript needs those names for.

A ```Map``` iterates as ```[K, V]``` pairs and a ```Set``` as its elements, which is what the declared ```Iterable``` gives them; ```WeakMap``` and ```WeakSet``` declare neither and are not iterable.

### Untyped Collections Are Untouched

**A ```Map``` or ```Set``` written with no type arguments is an ordinary JavaScript ```Map``` or ```Set``` and stays one.** ```size``` is a Number, keys and values are unconstrained, ```for..of``` binds at ```any```, and no program that does not use these types can observe any of the above. This is the same carve-out an array has - a plain ```[1, 2, 3]``` reports a ```length``` that is a Number - and it is stated here because a collection has no other place to state it.

```js
const m = new Map();          // An ordinary Map
m.set(1, "a");                // Any key, any value
m.size + 1;                   // A Number, and it mixes with Numbers

const t = new Map.<string, uint8>();
t.size + 1;                   // A uint64; the literal takes the type
// t.set(1, 2);               // TypeError: 1 is not a string
```

The two coexist in one program without interacting: a specialization adds nothing to the shared prototype and changes nothing about the constructor.

## Set Operations

The result element type follows where elements can come from: ```intersection``` and ```difference``` draw only from ```this```, while ```union``` and ```symmetricDifference``` draw from both sides:

```js
class Set<T> {
	union<U>(other: Set.<U>): Set.<T | U> { /* … */ return undefined; }
	intersection<U>(other: Set.<U>): Set.<T> { /* … */ return undefined; }
	difference<U>(other: Set.<U>): Set.<T> { /* … */ return undefined; }
	symmetricDifference<U>(other: Set.<U>): Set.<T | U> { /* … */ return undefined; }
	isSubsetOf<U>(other: Set.<U>): boolean { /* … */ return false; }
	isSupersetOf<U>(other: Set.<U>): boolean { /* … */ return false; }
	isDisjointFrom<U>(other: Set.<U>): boolean { /* … */ return false; }
}
```

Where the other side's element type is not known, the result's is not either and the result carries none: a union with an unconstrained set is unconstrained, and answering ```Set.<T>``` would state more than the values support. The bound on ```other``` is ```Set.<any>```, the top of the collection family, which is admissible for the reason ```[].<any>``` is admissible as the top of the array family - a store is checked against the receiver's own element type at run time, so writing through the wider view is refused whatever the static type permitted.

When ```T``` and ```U``` are unrelated value types the compiler can constant-fold the answer: an ```intersection``` of a ```Set.<uint8>``` and a ```Set.<string>``` is empty without iterating, and ```isDisjointFrom``` is ```true```, since distinct value types share no values.

These take a **set-like** rather than only a ```Set``` - any object with ```size```, ```has``` and ```keys``` - as they do today. A set-like's ```size``` is read as a count and checked rather than coerced, the treatment ```reserve``` already gives an array's.

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
	static try<R, E>(callback: (...args: [].<any>) => R | Promise.<R, E>, ...args: [].<any>): Promise.<R, E> { /* … */ return undefined; }
	static all<R, E>(promises: Iterable.<Promise.<R, E>>): Promise.<[].<R>, E> { /* … */ return undefined; }
	static allSettled<R, E>(promises: Iterable.<Promise.<R, E>>): Promise.<[].<PromiseSettledResult.<R, E>>, undefined> { /* … */ return undefined; }
	static any<R, E>(promises: Iterable.<Promise.<R, E>>): Promise.<R, AggregateError> { /* … */ return undefined; }
	static race<R, E>(promises: Iterable.<Promise.<R, E>>): Promise.<R, E> { /* … */ return undefined; }
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
	mapFn: (value: T, index: uint64) => U | Promise.<U, any>
): Promise.<[].<U>, any>;
```

When the element type is a value type or a typed class, ```fromAsync``` and ```toArray``` allocate the contiguous typed storage directly as elements settle, the same single-pass property the serialization document relies on for JSON arrays.
