# Reflection-Driven Binary Codec

A binary serializer whose schema is the type system itself: register a class, and its encoder and decoder are derived from reflection - field names, field types, enum underlying types, declaration order - rather than written by hand. This is the example that drives the proposal's introspection machinery for a living: the [decorators](../decorators.md) extension's ```Reflect.getReflection```, including the ```Reflect.Type``` structural reflection it uses to walk a bare type, type objects as first-class values and ```Map``` keys, and the ```type``` prefix operator constructing exact type values at runtime. Where [binary packet](binarypacket.md) writes one wire format by hand and [serialization](../serialization.md) types the JSON built-ins, this example asks the harder question: is the reflection surface *complete enough to program against*? The answer is mostly, with specific holes, and the notes at the end name each one.

Features exercised, and why they matter here:

- ```Map.<type, Codec>``` as the codec registry: type objects compare by identity per instantiation, so ```uint32```, ```Vec2```, ```[].<uint32>```, and ```Player | null``` are each their own key, with no string mangling of type names.
- ```Reflect.getReflection<Reflect.ClassField, T>()``` and ```<Reflect.Enum, T>()``` as the schema source - the class's own declaration is the wire contract, so there is nothing to keep in sync.
- The ```type``` prefix operator building union keys at runtime: ```type E | null``` inside a generic constructs the exact nullable type value the registry needs.
- Checked enum decoding through the reverse conversion: ```Kind(raw)``` throws on a value outside the enumeration, so wire garbage becomes a ```TypeError``` at the boundary instead of a corrupt enum inside the program.
- Constructor-free decoding: a decoded object fills the class layout through the cast form, the same rule typed JSON parsing uses, so deserialization cannot trigger construction side effects.
- A fixed-size fast path: when every field of a class is fixed-width, a whole column of them encodes as one byte-view copy - declaration-order layout making the type its own wire format.

## Wire Primitives

Little machinery: a growable byte buffer with scalar writers, and its mirror reader. Floats reach their bytes through a one-element array viewed as ```uint8``` - the reinterpretation the array-view rules define - so there is no ```DataView``` and no allocation per value.

```js
// codec.js
export class Writer {
	#out: [].<uint8> = [].<uint8>.withCapacity(1024);
	#f64: [1].<float64>;
	#f64Bytes: [].<uint8> = new [].<uint8>(this.#f64); // Aliases #f64's storage
	#f32: [1].<float32>;
	#f32Bytes: [].<uint8> = new [].<uint8>(this.#f32);

	u8(value: uint8) { this.#out.push(value); }
	u16(value: uint16) {
		this.#out.push(uint8(value));
		this.#out.push(uint8(value >> 8));
	}
	u32(value: uint32) {
		this.u16(uint16(value));
		this.u16(uint16(value >> 16));
	}
	u64(value: uint64) {
		this.u32(uint32(value));
		this.u32(uint32(value >> 32));
	}
	f32(value: float32) {
		this.#f32[0] = value;
		this.raw(this.#f32Bytes);
	}
	f64(value: float64) {
		this.#f64[0] = value;
		this.raw(this.#f64Bytes);
	}
	str(value: string) {
		this.u32(value.length);
		for (let i: uint32 = 0; i < value.length; i++) {
			this.u16(uint16(value.charCodeAt(i)));
		}
	}
	raw(bytes: [].<uint8>) {
		for (const b of bytes) {
			this.#out.push(b);
		}
	}
	take(): [].<uint8> { return this.#out; }
}

export class Reader {
	#bytes: [].<uint8>;
	#pos: uint32 = 0;

	constructor(bytes: [].<uint8>) { this.#bytes = bytes; }

	u8(): uint8 { return this.#bytes[this.#pos++]; }
	u16(): uint16 { return uint16(this.u8()) | (uint16(this.u8()) << 8); }
	u32(): uint32 { return uint32(this.u16()) | (uint32(this.u16()) << 16); }
	u64(): uint64 { return uint64(this.u32()) | (uint64(this.u32()) << 32); }
	f32(): float32 {
		const scratch: [1].<float32>;
		const view = new [].<uint8>(scratch);
		for (let i: uint32 = 0; i < 4; i++) { view[i] = this.u8(); }
		return scratch[0];
	}
	f64(): float64 {
		const scratch: [1].<float64>;
		const view = new [].<uint8>(scratch);
		for (let i: uint32 = 0; i < 8; i++) { view[i] = this.u8(); }
		return scratch[0];
	}
	str(): string {
		const length = this.u32();
		let s = '';
		for (let i: uint32 = 0; i < length; i++) {
			s += String.fromCharCode(this.u16());
		}
		return s;
	}
	raw(count: uint32): [].<uint8> {
		const view = this.#bytes[this.#pos .. this.#pos + count]; // Range index: a view, not a copy
		this.#pos += count;
		return view;
	}
}
```

## The Codec Registry

A codec is an encode function, a decode function, and a fixed size when it has one. The registry keys codecs by the type object itself. Types are values that compare by identity per instantiation - ```[].<uint32>``` is the same key everywhere it is written, and a different key from ```[].<uint16>``` - which is what makes a ```Map.<type, Codec>``` sound with no name-based lookup at all.

```js
// codec.js
export class Codec<T = any> {
	fixedSize: uint32; // 0 means variable-length
	encode: (value: T, w: Writer) => void;
	decode: (r: Reader) => T;

	constructor(fixedSize: uint32, encode: (value: T, w: Writer) => void, decode: (r: Reader) => T) {
		this.fixedSize = fixedSize;
		this.encode = encode;
		this.decode = decode;
	}
}

const registry: Map.<type, Codec> = new Map.<type, Codec>();

export function codecForType(t: type): Codec {
	const hit = registry.get(t);
	if (hit == undefined) {
		throw new TypeError('No codec registered for this type; derive or register it first');
	}
	return hit;
}

export function codecFor<T>(): Codec.<T> {
	return codecForType(T); // T in expression position is its type object
}

function set(t: type, codec: Codec) {
	registry.set(t, codec);
}

// The primitives seed the registry.
set(uint8,   new Codec(1, (v, w) => w.u8(v),   r => r.u8()));
set(uint16,  new Codec(2, (v, w) => w.u16(v),  r => r.u16()));
set(uint32,  new Codec(4, (v, w) => w.u32(v),  r => r.u32()));
set(uint64,  new Codec(8, (v, w) => w.u64(v),  r => r.u64()));
set(float32, new Codec(4, (v, w) => w.f32(v),  r => r.f32()));
set(float64, new Codec(8, (v, w) => w.f64(v),  r => r.f64()));
set(boolean, new Codec(1, (v, w) => w.u8(v ? 1 : 0), r => r.u8() != 0));
set(string,  new Codec(0, (v, w) => w.str(v),  r => r.str()));
```

## Classes by Reflection

```deriveClass.<T>()``` reads the class's field map from reflection and builds a codec that encodes fields in declaration order. Field codecs resolve *lazily*, on first use rather than at derivation - that one decision is what lets a class refer to itself (through a nullable) or to a class registered later, because the registry acts as the indirection a recursive type needs, the same role the ```reference``` form plays in the type-record format.

```js
// codec.js
export function deriveClass<T>(): Codec.<T> {
	const fields = Reflect.getReflection<Reflect.ClassField, T>();
	const names: [].<string> = [];
	const types: [].<type> = [];
	for (const name in fields) { // Insertion order, taken here to be declaration order (see notes)
		names.push(name);
		types.push(fields[name].type);
	}
	const resolved: [].<Codec | null> = [];
	resolved.length = types.length; // null until first use
	function fieldCodec(i: uint32): Codec {
		let c = resolved[i];
		if (c == null) {
			c = codecForType(types[i]);
			resolved[i] = c;
		}
		return c;
	}

	// Fixed-size when every field is: the layout is the format.
	let fixed: uint32 = 0;
	for (const t of types) {
		const c = registry.get(t);
		if (c == undefined || c.fixedSize == 0) {
			fixed = 0;
			break;
		}
		fixed += c.fixedSize;
	}

	const codec = new Codec.<T>(fixed,
		(value, w) => {
			for (let i: uint32 = 0; i < names.length; i++) {
				fieldCodec(i).encode(value[names[i]], w);
			}
		},
		r => {
			const o = {};
			for (let i: uint32 = 0; i < names.length; i++) {
				o[names[i]] = fieldCodec(i).decode(r);
			}
			return T(o); // Cast fills the class layout; no constructor runs
		});
	set(T, codec);
	return codec;
}
```

The decode line carries the semantics that make reflection-driven deserialization safe to specify: the cast from a matching shape fills the class's layout directly - every field supplied or defaulted, ```where``` clauses validated - and runs no constructor, which is the same rule the serialization extension uses for typed JSON. A class whose constructor opens a file cannot be tricked into opening one by a byte stream.

Enums derive from their reflection too. Encoding uses the one-way implicit conversion to the underlying type; decoding uses the *checked* reverse conversion, so the enum boundary validates the wire:

```js
// codec.js
export function deriveEnum<T extends enum>(): Codec.<T> {
	const info = Reflect.getReflection<Reflect.Enum, T>();
	const underlying = codecForType(info.valueType); // The enum's underlying value type
	const codec = new Codec.<T>(underlying.fixedSize,
		(value, w) => underlying.encode(value, w), // enum -> underlying is the one-way implicit direction
		r => T(underlying.decode(r))); // T(raw) throws unless raw names an enumerator
	set(T, codec);
	return codec;
}
```

## Combinators for Parameterized Types

Arrays, fixed-extent arrays, and nullables are built by combinators whose real work is constructing the right *key*. Inside a generic, a type expression over the parameter evaluates to the exact type value: ```[].<E>``` and ```[N].<E>``` directly, and the union through the ```type``` prefix operator, since ```E | null``` in expression position would otherwise parse as bitwise-or.

```js
// codec.js
export function deriveArray<E>(): Codec.<[].<E>> {
	const elem = codecForType(E);
	const codec = new Codec.<[].<E>>(0,
		(value, w) => {
			w.u32(value.length);
			for (const v of value) {
				elem.encode(v, w);
			}
		},
		r => {
			const length = r.u32();
			const out = [].<E>.withCapacity(length);
			for (let i: uint32 = 0; i < length; i++) {
				out.push(elem.decode(r));
			}
			return out;
		});
	set([].<E>, codec); // The array instantiation is its own key
	return codec;
}

export function deriveFixedArray<E, N: uint32>(): Codec.<[N].<E>> {
	const elem = codecForType(E);
	const codec = new Codec.<[N].<E>>(elem.fixedSize * N, // 0 propagates: variable elements make it variable
		(value, w) => {
			for (const v of value) {
				elem.encode(v, w); // Extent is in the type; no length prefix on the wire
			}
		},
		r => {
			const out: [N].<E>;
			for (let i: uint32 = 0; i < N; i++) {
				out[i] = elem.decode(r);
			}
			return out;
		});
	set([N].<E>, codec);
	return codec;
}

export function deriveNullable<E>(): Codec {
	const K = type E | null; // The prefix operator builds the exact union key
	const elem = codecForType(E);
	const codec = new Codec(0,
		(value, w) => {
			if (value == null) {
				w.u8(0);
			} else {
				w.u8(1);
				elem.encode(value, w);
			}
		},
		r => r.u8() == 0 ? null : elem.decode(r));
	set(K, codec);
	return codec;
}
```

These combinators exist because they have to: given the *value* of an array type at runtime, nothing exposes its element type or extent, so a walker cannot derive ```[].<uint32>``` from the type object alone. The combinator sidesteps the opacity by being told ```E``` as a generic parameter and reconstructing the instantiation. The notes weigh this properly - it is the second of the two real holes this example found.

## Columns in One Copy

When a class is fixed-size, its declaration-order layout *is* a wire format, and a whole array of instances round-trips as one byte-view copy - the same reinterpretation the Writer uses for a single float, applied to a column. This is the path the interpreter overhead cannot touch.

```js
// codec.js
export function encodeColumn<T>(values: [].<T>, w: Writer) {
	const c = codecFor.<T>();
	w.u32(values.length);
	if (c.fixedSize != 0) {
		w.raw(new [].<uint8>(values)); // One contiguous copy of the whole column
	} else {
		for (const v of values) {
			c.encode(v, w);
		}
	}
}

export function decodeColumn<T>(r: Reader): [].<T> {
	const c = codecFor.<T>();
	const length = r.u32();
	const out = [].<T>.withCapacity(length);
	if (c.fixedSize != 0) {
		const view = new [].<T>(r.raw(length * c.fixedSize)); // Reinterpret the wire bytes as elements
		for (let i: uint32 = 0; i < length; i++) {
			out.push(view[i]);
		}
	} else {
		for (let i: uint32 = 0; i < length; i++) {
			out.push(c.decode(r));
		}
	}
	return out;
}
```

Views read and write in platform byte order, so this fast path is exact on one machine and correct across machines only when their endianness agrees; a format that must cross architectures pins its fields with the ```@endian``` member decorator and stays on the per-field path. The per-field encoders happen to write little-endian, so on little-endian hardware the two paths produce identical bytes for a ```@packed```-layout class - the fast path is an optimization, not a second format.

## Usage

A small game-state slice: an enum, a fixed-size vector, and a player that references another player. Derivations run once at module load, in dependency order for the eager pieces, with the lazy field resolution absorbing the one genuine cycle (```Player``` containing ```Player | null```).

```js
// main.js
import { Writer, Reader, deriveClass, deriveEnum, deriveArray, deriveNullable, codecFor, encodeColumn, decodeColumn } from './codec.js';

enum Kind { Civilian, Guard, Merchant }

class Vec2 {
	x: float32;
	y: float32;
}

class Player {
	id: uint32;
	kind: Kind;
	name: string;
	position: Vec2;
	tags: [].<uint32>;
	mentor: Player | null;
}

deriveEnum.<Kind>();      // 4 bytes on the wire: int32 underlying
deriveClass.<Vec2>();     // fixedSize 8
deriveArray.<uint32>();   // key: [].<uint32>
deriveClass.<Player>();   // fields resolve lazily, so the self-reference below is fine
deriveNullable.<Player>(); // key: Player | null

const mentor = { id: 1, kind: Kind.Guard, name: 'Aldric', position: { x: 3, y: 4 } := Vec2, tags: [], mentor: null } := Player;
const player = { id: 2, kind: Kind.Merchant, name: 'Bren', position: { x: 10, y: 2 } := Vec2, tags: [7, 9], mentor } := Player;

const w = new Writer();
codecFor.<Player>().encode(player, w);

const back = codecFor.<Player>().decode(new Reader(w.take()));
back.kind;          // Kind.Merchant
back.mentor?.name;  // 'Aldric'
back.position.y;    // 2

// A corrupted enum byte is a boundary error, not a corrupt value:
// Kind(9) throws a TypeError, so decode rejects the stream instead of storing 9 in a Kind.

// The bulk path: a fixed-size column in one copy each way.
const path: [].<Vec2> = [{ x: 0, y: 0 } := Vec2, { x: 1, y: 1 } := Vec2, { x: 2, y: 4 } := Vec2];
const cw = new Writer();
encodeColumn(path, cw);                       // 4 + 3 * 8 bytes
const path2 = decodeColumn.<Vec2>(new Reader(cw.take()));
```

## Coverage Notes

The point of this example was to find out whether the reflection surface is complete enough to build against. It is close, and the two places it is not are precise enough to fix with signatures rather than redesigns.

- **Reflection cannot be re-entered with a type value, which forces per-class derivation.** ```Reflect.getReflection<Reflect.ClassField, T>()``` takes its target as a *generic parameter*. The derivation walker, holding a field's type as a runtime *value* (```fields[name].type```), has no way back in: it cannot call the generic form with a value, so it cannot auto-derive a nested class it discovers. This example's answer is explicit registration per class plus lazy field resolution through the registry, which works but pushes a compiler-shaped job onto module-load ordering. The missing piece is a value-taking overload - ```Reflect.getReflection(target: type, kind)``` - and it is the same runtime-dependent-dispatch seam the entity-component example hit from the other side: type parameters and type values are two currencies, and the reflection API only accepts one.
- **Type objects are now walkable through `Reflect.Type` - resolved.** Given ```[].<uint32>``` or ```Player | null```, ```Reflect.getReflection.<Reflect.Type>(t)``` exposes the element or the arms - the named retrieval this example previously lacked - and the ```array``` node carries the ```extent``` that fixed-length ```[N].<T>``` needs, so the two fixes this note once asked for are both in the [decorators](../decorators.md) extension. The combinator reconstruction below (```deriveArray.<E>``` rebuilding ```[].<E>``` from a parameter) is therefore the pre-```Reflect.Type``` workaround: with the structural nodes available a codec can walk a type directly instead of rebuilding its instantiations, and the union key it needs is the ```union``` node's arms rather than a value assembled with the ```type``` prefix operator.
- **Live reflection and structural types are one facility - resolved.** Classes and enums are served by ```getReflection```'s declaration contexts; aliases, unions, tuples, arrays, and recursion are served by the same ```getReflection``` through ```Reflect.Type```. A serializer needs both halves and now gets them from one call, which is the merge this note previously argued for: a codec generator is precisely the consumer that wants fields *and* structure, and one entry point supplies both.
- **There is no compile-time iteration over fields, so derived codecs are interpreters.** Rust's ```serde``` derive and Zig's ```comptime``` walk a type's fields *during compilation* and emit straight-line code per type; here the walk happens at module load and the result is a loop over name and codec arrays with dynamic property access - a shape no specialization can turn into per-type code, because the field list is data, not structure the compiler sees. The ceiling is real and the recovery lane is too: the fixed-size column path replaces the interpreter with one byte-view copy exactly where declaration-order layout makes the type its own format, and ```byteLength``` being a compile-time constant is what lets a test pin a format by asserting a size.
- **What worked without friction is worth the list.** Type identity as ```Map``` keys held up across primitives, classes, instantiations, and a constructed union. Checked enum decode - ```T(raw)``` throwing on a non-enumerator - turns wire corruption into a typed boundary error for free. Constructor-free decode via the cast fill means byte streams cannot run code, which is a security property inherited from a conversion rule rather than engineered here. And declaration order doubling as the wire order makes the class declaration the single source of truth - with the one caveat that this example *assumes* the reflected field map iterates in declaration order, which the decorators document should state rather than leave to convention.
