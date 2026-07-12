# Serialization

This extension specifies how typed values cross I/O boundaries: JSON parsing and stringification, structured clone, and binary layout. It exists because every network response, config file, and stored record is a place where untyped data enters a typed program, and today that boundary costs twice - once to build an intermediate untyped object graph and once to walk it validating and converting. With runtime types the parser can validate while it parses and write directly into the final typed representation, so the boundary check that makes the program safe is also what makes it fast.

The pieces this builds on: [primitive metadata](primitivemetadata.md) supplies per-field constraints and their `validate`/`describe` hooks, [dependent record types](dependentrecordtypes.md) supply cross-field `where` clauses checked at boundaries, [decorators](decorators.md) supply reflection over fields, and the member memory layout section of the main proposal supplies the physical layout that parsing targets.

## Typed JSON Parsing

`JSON.parse` gains a generic overload that parses and validates in one pass:

```js
JSON.parse.<T>(text: string): T
```

```js
type ServerConfig = {
	host: string.<{ pattern: /^[a-z0-9.-]+$/ }>,
	port: uint16,
	timeout?: float32.<{ exclusiveMinimum: 0 }>,
	tls: boolean,
	certificate?: string
} where !this.tls || this.certificate != undefined;

const config = JSON.parse.<ServerConfig>(text);
config.port; // uint16, already range-checked by the type itself
```

Validation is layered exactly like any other boundary in the proposal. Each field is checked against its type as it's parsed, metadata constraints run their `validate` hooks on the spot, and once the object is complete the `where` clause runs as the construction boundary check. `port: uint16` needs no metadata at all; an out-of-range number is a TypeError because the integer type is the constraint.

### Value mapping

JSON has six value forms and each maps onto the type system directly. An object maps to an interface, type literal, or class; an array maps to `[].<T>` or `[N].<T>` (the length must equal `N` or it's a TypeError); a string maps to `string` and runs any `StringBounds` metadata; `true`/`false` map to `boolean`; `null` maps to a nullable union member; and an absent property maps to an optional `?` field, which takes its declared default or the type's default value. Anything else is a TypeError.

### Numbers parse into their target type

A JSON number token is a digit sequence, and the grammar places no limit on its precision - only `JSON.parse`'s conversion to a float64 does. The typed overload applies the same rule as type propagation to literals in the main proposal, but to JSON text: the digits convert directly into the target numeric type with no float64 intermediate.

```js
type Account = { id: uint64, balance: decimal128 };
const a = JSON.parse.<Account>('{ "id": 18446744073709551615, "balance": 0.1 }');
a.id; // 18446744073709551615, exact
a.balance; // 0.1 as an exact decimal, never rounded through a double
```

`bigint` fields accept any digit sequence. A number token that doesn't fit its integer target, or a fractional token targeting an integer type, is a TypeError.

### Arrays parse into contiguous memory

A `[].<T>` field over a typed class or value type parses element by element into the array's contiguous storage. There is no intermediate array of boxed objects to build and discard; a million-vertex mesh is one allocation and one pass.

```js
class Vertex {
	x: float32;
	y: float32;
	z: float32;
}
interface Mesh {
	name: string;
	vertices: [].<Vertex>;
}
const mesh = JSON.parse.<Mesh>(text);
```

### Classes as targets

A class whose fields are all typed has a sealed, fixed layout, and `JSON.parse.<T>` allocates that layout and fills it directly. The constructor does not run, matching the existing precedent of `structuredClone`; absent optional fields take their declared defaults; class `where` clauses run once the instance is filled, as the construction boundary. A class that requires constructor logic to establish invariants should express those invariants as `where` clauses or metadata so they hold under deserialization too.

### Enums

A field typed as an enum accepts a JSON value equal to one of the enumerators' values (compared with the enum's underlying type) and parses to that enumerator. Any other value is a TypeError.

```js
enum Role: string { Admin = 'admin', User = 'user' };
type Session = { user: string, role: Role };
JSON.parse.<Session>('{ "user": "a", "role": "admin" }'); // role: Role.Admin
```

### Unions

A union field is resolved first by JSON token type, then, for object shapes, by structure using the same property-superset rule as typed destructuring. A union whose members can't be distinguished by token type or structure is a compile-time TypeError at the `JSON.parse.<T>` call, since no input could ever resolve it.

```js
type Id = uint32 | string; // token type discriminates
type Shape = { radius: float32 } | { w: float32, h: float32 }; // structure discriminates
```

### Unknown keys

Typed objects are sealed, and parsing honors that: a JSON key with no corresponding field is a TypeError. A type opts into open data explicitly with an index signature, which is also how fully dynamic JSON is written:

```js
// JSON.parse.<ServerConfig>('{ "host": "a", "port": 1, "tls": false, "extra": 1 }');
// TypeError: unknown key "extra" for type ServerConfig

type LooseConfig = { host: string, [key: string]: any };
const anything = JSON.parse.<{ [key: string]: any }>(text);
```

### Errors

Malformed JSON throws a SyntaxError as it does today. Well-formed JSON that fails the type is a TypeError whose message names the JSON path, the expected type, and the failed constraint, using each meta type's `describe` hook. The typed catch clauses from the main proposal separate the two:

```js
try {
	return JSON.parse.<ServerConfig>(text);
} catch (e: SyntaxError) {
	// not JSON at all
} catch (e: TypeError) {
	// JSON, but not a ServerConfig; e.message like:
	// "at .timeout: expected float32 where exclusiveMinimum: 0, got -1"
}
```

The typed overloads take no reviver. A reviver observes and rewrites intermediate untyped values, which is exactly the graph this feature avoids building; transformations belong to the type system through casts, metadata, and decorators.

## Typed JSON Modules

A JSON module import can annotate its binding, validating the module once at evaluation with the same machinery, using the typed-import form specified in the main proposal's import types section. A failure is a load-time TypeError identifying the file and path:

```js
import config: ServerConfig from './config.json' with { type: 'json' };
```

## JSON.stringify

`JSON.stringify` learns the new types on the output side. The numeric value types emit plain number tokens, with `int64`, `uint64`, `decimal128`, and `bigint` emitting their full digits - valid JSON, though a note of caution: consumers parsing with an untyped double-based parser will round them, so string-encoding via a decorator remains the interop-safe choice for foreign endpoints. `bigint` serializing to digits is a behavior change from today's TypeError, in the same spirit as the `BigInt` change in the type propagation section. Enum values stringify as their underlying value. Typed class instances stringify their fields in declaration order, which is well-defined because the layout is; `toJSON` is respected as always, and that remains the hook for custom formats.

```js
const id: uint64 = 18446744073709551615;
JSON.stringify({ (id: uint64): id }); // '{"id":18446744073709551615}'
JSON.stringify(10n); // '10', previously a TypeError
```

Types with no natural JSON form, like `rational`, `complex`, and the SIMD types, are a TypeError to stringify without a `toJSON`, rather than silently choosing an encoding.

## structuredClone and Typed Values

Within an agent cluster, `structuredClone` preserves types: a typed class instance clones to an instance of the same class with the same layout, which for classes containing only value types is a memory copy with no per-field work. `[N].<T>` arrays clone, and transfer, like the TypedArrays they interoperate with. A `shared` array is already shared and is passed by reference rather than cloned. Between threads none of this is usually needed - the [threading](threading.md) extension's shared heap means a thread shares an object by referencing it - but `structuredClone` remains available for deliberate snapshots.

Outside the agent cluster, in persistence like IndexedDB or messages to another process, type identity can't be assumed on the other side, so values are stored structurally as untyped data. Reading typed data back is then an ordinary boundary: the host API or the program casts with a target type, and the same validation that backs `JSON.parse.<T>` runs. Hosts are encouraged to expose that directly, e.g. a typed read like `store.get.<Player>(key)`.

## Binary Layout

The main proposal already contains the primitives for binary serialization: a typed class has a defined layout from its declaration order and the member memory alignment and offset rules, `T.byteLength` and a field's reflected `offset` expose that layout as compile-time constants, `@packed` removes the natural alignment padding so members land at the byte offsets a format specifies, placement `new` constructs instances inside an existing buffer, array views alias a class's memory as `[].<uint8>`, and the `@endian` decorator fixes byte order for wire formats. Writing a typed value to a socket is therefore a view and a copy, and reading one is a placement `new` over the received buffer followed by the type's boundary validation - the where clauses and metadata checks are what make a raw byte reinterpretation safe to hand to the rest of the program. Schema evolution across versions is deliberately left to userland, where decorators and reflection can record versions and migrations; the language's guarantee is only that a given class declaration has one layout.

## Interaction with Decorators and Reflection

Wire-name mapping, field omission, and versioning stay in userland. The [dependent record types](dependentrecordtypes.md) document shows the pattern: an `@field('wireName')` decorator records mappings into class metadata and a reflective `serialize`/`deserialize` walks them. `JSON.parse.<T>` itself maps JSON keys to same-named fields only. An open direction is a standard decorator vocabulary the native parser understands, so renamed fields keep the fused fast path instead of falling back to reflective code.

## Streaming

A large JSON array shouldn't have to be resident in memory as text and again as values. `JSON.parseStream` takes a stream of chunks and, for a top-level array type, yields each element as it completes, validated:

```js
JSON.parseStream.<T>(stream: AsyncIterable.<string | [].<uint8>>): AsyncIterator.<elementType(T)>
```

```js
type Reading = { sensor: string, value: float32, at: Temporal.Instant };

for await (const reading: Reading of JSON.parseStream.<[].<Reading>>(response.body)) {
	// Each element is validated as it completes; the array is never materialized
	record(reading);
}
```

```elementType(T)``` is the array type's element, so `parseStream.<[].<Reading>>` yields `Reading`; for a non-array `T` it is `T` itself. A malformed chunk throws a SyntaxError from the iterator, and an element that fails its type throws a TypeError naming the index and path, both surfacing at the `for await` that pulled them. Streaming a non-array top-level type yields exactly one value when the stream ends, which is the degenerate case rather than an error.

## Host APIs

Host APIs that produce values from bytes should adopt the same overload shape rather than each inventing one. The pattern is a generic parameter naming the expected type and a rejection type that includes the parse and validation failures:

```js
class Response {
	json<T = any>(): Promise.<T, SyntaxError | TypeError>;
	text(): Promise.<string, any>;
}

const config = await (await fetch('/config.json')).json.<ServerConfig>();
```

The same shape applies to `structuredClone.<T>(value)`, to storage reads such as an IndexedDB `get.<Player>(key)`, and to `WebSocket` message events over binary or text frames. These are specified by their host, not here; what this document fixes is the shape they should copy, so that a validated read looks the same wherever the bytes came from.

## Open Questions

- Composites: once the records and tuples section aligns with the Composites proposal, `#{}`/`#[]` need a JSON mapping here.
