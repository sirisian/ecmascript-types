# Math.random Extension

The goal of this extension is to describe how a random proposal would change with types. This will cover the basic generation tasks like random numbers and seeded randoms with various methods.

Refer to these as I'll base these notes off of them: https://github.com/tc39/proposal-seeded-random and https://github.com/tc39/proposal-random-functions

The seeded proposal pins a single algorithm, ChaCha12, so that a seed produces the same numbers in every engine. That decision is load-bearing for everything below: a seeded generator whose algorithm is implementation-defined has no reproducible stream and no portable state. Here the algorithm is named by ```Math.PRNG``` rather than fixed, and ```Math.PRNG.Default``` therefore means "the engine's choice" only for the unseeded ```Math.random```. A seeded generator resolves it to a specific algorithm at construction, and that name is part of its state.

**Note**: This is a pseudorandom library identical to Math.random and not cryptographically random. It's designed around speed for things like games. Extending the Crypto library later into ECMAScript would probably be ideal to have a wide range of Crypto RNG generation.

Bounds are given as a [range](ranges.md). A range carries its endpoints and its inclusivity in its type, so a call site says which interval it means, the compiler specializes on it, and there is no convention to remember.

### Math.random.<T, Method = Math.PRNG.Default>()

The first addition is a generic version of ```Math.random``` for the float types:

```float16```, ```float32```, ```float64```

The second generic argument selects the PRNG method. Methods are namespaced as an enumeration on ```Math```, with ```Math.PRNG.Default``` being the browser's default method:

```js
enum PRNG { Default, Xoshiro256StarStar, PCG32, SplitMix64 }; // Exposed as Math.PRNG
Math.random.<float32, Math.PRNG.Xoshiro256StarStar>();
```

Rapidly generating arrays of random numbers in the range \[0, 1):

```js
const a: [100].<float32>;
Math.random.<float32>(a);
```

The array fill overload returns the filled array, allowing chaining.

When used with an integer type it will generate a random integer across the type's full range, inclusive.

```int8```, ```int16```, ```int32```, ```int64```, ```int128```  
```uint8```, ```uint16```, ```uint32```, ```uint64```, ```uint128```

```js
const a = Math.random.<uint8>();
a; // 0 to 255
```

Generating a random RGB color:
```js
const a: [3].<uint8>;
Math.random.<uint8>(a);
a; // [100, 20, 25] as an example
```

Both no-argument forms are defaults over the range form below. ```Math.random.<float32>()``` is ```Math.random.<float32>(0..1)```, and ```Math.random.<uint8>()``` is ```Math.random.<uint8>(..)```.

### Math.random.<T, Method = Math.PRNG.Default>(range)

The bounds are a range, which may be any of the [range](ranges.md) forms. Its element type is ```T``` and its interval is part of its type:

```js
function Math.random<T, R extends RangeBounds.<T>, Method: Math.PRNG = Math.PRNG.Default>(range: R): T;
function Math.random<T, R extends RangeBounds.<T>, Method: Math.PRNG = Math.PRNG.Default>(array: [].<T>, range: R): [].<T>;
```

```js
Math.random.<uint8>(1..=6); // A die. 1 through 6
Math.random.<int32>(-5..=5); // -5 through 5
Math.random.<float32>(0..1); // [0, 1)
Math.random.<float32>(-1..=1); // Both endpoints attainable
Math.random.<uint32>(0..100); // 0 through 99
Math.random.<bigint>(0n..=1000n); // Any ordered type works
```

An open-ended range takes its missing endpoint from ```T```, which therefore has to be a type with finite bounds: an integer type, or any type carrying ```minimum``` and ```maximum``` metadata. ```Math.random.<float32>(0..)``` is a TypeError, since a float's upper bound is not a useful answer.

```js
Math.random.<uint8>(..); // The full range, 0 through 255
Math.random.<int32>(0..); // 0 through int32's maximum
Math.random.<int16>(..=0); // int16's minimum through 0
```

Filling an array uses the same range:

```js
const prng = Math.seededRandom.<float32>({ seed: 0 });
const a: [100].<float32>;
prng.random(a, -1..=1);
```

An empty range produces no value. When its endpoints are literals this is a compile-time TypeError, and otherwise a ```RangeError``` when the call is made:

```js
// Math.random.<uint8>(5..5); // TypeError: the range is empty
Math.random.<uint8>(5..=5); // 5, the only value the range contains
Math.random.<uint8>(low..high); // RangeError at the call when high <= low
```

The two intervals with an open start have no literal form. They are named through ```Range.of```, and the one that matters is ```(0, 1]```, which sampling code needs whenever it takes a logarithm:

```js
const u = Math.random.<float32>(Range.of.<Range.Interval.OpenClosed>(0, 1)); // (0, 1]
const exponential = -Math.log(u) / lambda;
```

### Bounded Types

A range whose endpoints are compile-time constants is also a type, since a range is what ```NumberBounds``` describes. A bounded type therefore needs no bounds argument, and the same declaration validates the value everywhere else it travels:

```js
type Die = uint8.<1..=6>;
type Unit = float32.<0..1>;
type Percent = uint8.<0..=100>;

Math.random.<Die>(); // 1 through 6, no arguments
Math.random.<Unit>(); // [0, 1)
Math.random.<Percent>(a); // Fill an array of Percent
```

The generated value satisfies its type's constraint by construction, so the validation a cast runs at a return boundary is elided. Passing a range that doesn't fit the type's bounds is a TypeError:

```js
// Math.random.<Die>(1..=7); // TypeError: 7 is outside Die's maximum of 6
Math.random.<Die>(1..=3); // Legal, a narrower range of the same type
```

Dimensioned quantities work the same way, since the range's element type is the quantity:

```js
type Meter = float32.<{ m: 1 }>;
Math.random.<Meter>(Meter(0)..=Meter(10)); // A Meter
```

### Generation Rules

These are normative, because getting them wrong is the usual way a random library is wrong.

- Integer generation over \[a, b] is uniform. Implementations use rejection sampling, never a modulo of a full-width word, which over-represents the low end of the range.
- ```Math.random.<T>(..)``` over an integer type spans ```2^n``` values, so the count does not fit in ```T``` and must be computed in a wider type. This is the case the "full range, inclusive" wording used to hide.
- For floats, \[a, b) is ```a + u * (b - a)``` with ```u``` in \[0, 1). \(a, b] is ```b - u * (b - a)```. \[a, b] scales a 53-bit integer by ```1 / (2^53 - 1)``` rather than multiplying ```u```, since ```u``` never reaches ```1```. \(a, b) rejects a zero ```u```.
- The unadorned ```Math.random.<float32>()``` is \[0, 1) exactly, as it is today.

### Math.seededRandom.<T, Method = Math.PRNG.Default>(config)

```js
const prng = Math.seededRandom.<float32>({ seed: 0 });
const i = prng.random(); // [0, 1)
const j = prng.random(-5..=5); // [-5, 5]
const k = prng.random(0..1); // [0, 1)
```

Rapidly generating arrays of random numbers:

```js
const prng = Math.seededRandom.<float32>({ seed: 0 });
const a: [100].<float32>;
prng.random(a, -1..=1);
```

### One Parameter System

Bounds are a range and nothing else. There is no ```random(min, max)``` overload, no interval enumeration in the generic list, and no options object.

The alternatives were considered and each loses to a range on its own terms. A ```(min, max)``` pair cannot say which endpoints it includes, so the convention lives in prose, and prose is where it goes wrong: this document previously said floats were half-open and then documented two float calls as closed. An interval generic (```Math.random.<float32, Math.Interval.Closed>(-1, 1)```) says it, but separates the interval from the endpoints it describes, and it forces named generic arguments so a caller can select a PRNG without also spelling an interval. A runtime options bag (```{ inclusive: true }```, NumPy's ```endpoint``` flag) cannot specialize, so the branch survives into the generated code. Python needs three functions - ```randint```, ```randrange```, ```uniform``` - because it has no way to say the interval, and ```uniform``` documents its upper endpoint as included depending on rounding.

A range is one value carrying both endpoints and their inclusivity, in a type, at the call site. It removes an axis from the overload set: what would be ```T``` × interval × method × array-fill is ```T``` × method × array-fill. It is also why the generic list here stays short enough that named generic arguments aren't needed.

### Parallel Streams

Sharing a seeded generator across Web Workers isn't allowed, and copying its state into two threads is worse: both would produce the same numbers. The portable answer is the one every PRNG library ships, which is a way to derive independent streams from one generator.

```js
const prng = Math.seededRandom.<float32, Math.PRNG.Xoshiro256StarStar>({ seed: 0 });
const worker = prng.jump(); // Advances prng by 2^128 draws and returns a generator at the old position
```

```jump``` is defined for every method, as the operation that partitions the period: ```xoshiro256**``` and its relatives have ```jump```/```longJump```, ```pcg32``` has ```advance(n)```, and the ChaCha family increments a stream identifier. Each thread takes a jumped generator and never touches another's.

Where a generator is created per thread instead, seed them from one ```splitmix64```, which is what its authors intended it for.

### Saving and Restoring State

A generator's state is a value. It can be read, stored, sent, and used to construct a generator that continues the original's stream exactly.

```js
class PRNGState {  // Exposed as Math.PRNGState
	readonly method: string; // The algorithm's canonical name
	readonly version: uint8; // The layout version of that algorithm's state
	readonly words: [].<uint8>; // The algorithm's state, big-endian

	toBytes(): [].<uint8>; // Self-describing, for a socket or a file
	static fromBytes(value: [].<uint8>): PRNGState;

	toJSON(): { method: string, version: uint8, state: string }; // state is base64
	operator PRNGState(value: string); // Cast from base64, for JSON.parse
}

class SeededPRNG<T, Method: Math.PRNG> {
	get state(): PRNGState;
	set state(value: PRNGState);
	jump(): SeededPRNG.<T, Method>;
}
```

```js
const prng = Math.seededRandom.<float32, Math.PRNG.Xoshiro256StarStar>({ seed: 0 });
prng.random(); // Advance it a while
const saved = prng.state;

const resumed = Math.seededRandom.<float32>({ state: saved });
resumed.random() === prng.random(); // true, the same stream
```

#### What the state is

For each method the state is exactly what that algorithm's reference implementation calls its state, in the same order, with no extra fields:

| Method | State | Constraint |
| --- | --- | --- |
| ```splitmix64``` | 1 x uint64 | none |
| ```xoshiro256**``` | 4 x uint64 | not all zero |
| ```pcg32``` | uint64 state, uint64 inc | inc is odd |
| ```chacha12``` | 32 byte seed, uint64 stream, uint128 wordPos | wordPos below 2^68 |

Restoring validates. An all-zero ```xoshiro256**``` state is a fixed point that emits zeros forever, and an even ```pcg32``` increment collapses the generator's period, so both are ```RangeError``` rather than a generator that silently misbehaves. These are ordinary type constraints:

```js
class Pcg32State {
	state: uint64;
	inc: uint64;
} where this.inc % 2 == 1;
```

An unknown method name, a state length that doesn't match the named method, or a version this implementation doesn't know are each a ```TypeError```: the bytes are not a state of the kind they claim to be.

#### No hidden buffering

This is the rule that makes a state portable, and it constrains generation rather than serialization. Nothing in this extension caches leftover engine output. ```Math.random.<uint8>()``` consumes a whole engine output and discards the unused bits; a normal distribution, if one is ever added, does not stash the second value Box-Muller produces. A generator's state is therefore the algorithm's state and nothing more.

The alternative is what NumPy does, whose state carries ```has_uint32```, ```uinteger```, and a cached gaussian, and whose state consequently interchanges with nothing. C++ takes the same medicine differently, requiring engines and distributions to be serialized separately, because its distributions hold state.

Note the distinction this buys and the one it doesn't. Saving state gets you resumption of a stream. Getting the *same numbers* in another language additionally requires that language to derive its results from engine outputs the same way, which is what the generation rules above specify: rejection sampling with Lemire's method, the 53-bit float construction, and so on. State portability without derivation portability restores the engine but not the sequence you cared about.

#### Bytes

```toBytes``` is self-describing, so a state can never be restored into the wrong algorithm. Multi-byte fields are big-endian, which is the wire convention and what Go's ```rand/v2``` marshalers use.

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | Magic, the ASCII bytes ```PRNG``` |
| 4 | 1 | Container version, currently 1 |
| 5 | 1 | Method name length, n |
| 6 | n | Method name, ASCII |
| 6+n | 1 | Method state version |
| 7+n | 2 | State length, m |
| 9+n | m | State, big-endian |

```js
socket.send(prng.state.toBytes()); // A binary frame
const restored = Math.PRNGState.fromBytes([].<uint8>(event.data));
```

The [binary packet](examples/binarypacket.md) example writes one with ```write.<uint16>(bytes.length)``` followed by the bytes.

#### JSON and localStorage

```toJSON``` emits the method and version as fields and the state as base64, because JSON numbers cannot hold a ```uint64``` and every language's parser rounds them differently. Base64 costs a third over the raw bytes; hex costs double, and is worth it only when a human has to read the value.

```js
localStorage.setItem('rng', JSON.stringify(prng.state));
// { "method": "xoshiro256**", "version": 1, "state": "3q2+7wAAAAA..." }

const state = JSON.parse.<Math.PRNGState>(localStorage.getItem('rng'));
const prng = Math.seededRandom.<float32>({ state });
```

```JSON.parse.<Math.PRNGState>``` runs the cast operator on the base64 string and then the method's validation, so a corrupted or hand-edited entry fails at the parse rather than at the thousandth draw. ```structuredClone``` copies a state as it copies any typed record, and a state travels to a worker that way.

#### Why not the alternatives

Serializing the generator object rather than its state, as Java does, ties the format to a class layout and to one language. Emitting the raw state words with no header, as some libraries do, means a state saved from one algorithm restores into another as plausible-looking garbage. Emitting ```uint64``` words as JSON numbers loses them above 2^53 in any parser that reads JSON numbers as doubles, which is most of them. Naming the algorithm by its ```Math.PRNG``` ordinal rather than its name breaks the moment the enumeration gains a member.

What remains is the shape NumPy converged on and Go writes in binary: a tagged, versioned, byte-exact state whose tag is a name.
