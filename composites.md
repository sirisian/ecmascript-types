# Composites

The [Composites proposal](https://github.com/tc39/proposal-composites) adds ```Composite(obj)```: a frozen, prototype-less object built from the argument's own enumerable string-keyed values, **interned** so that two calls with equal contents return the same object. Structural equality then costs nothing to provide, because it isn't a new comparison: ```===``` already holds between an object and itself, ```Map```, ```Set```, and ```Array.prototype.includes``` already use SameValueZero, and an interned composite reaches all of them with no change to any collection or operator.

```js
const a = Composite({ x: 1, y: 4 });
const b = Composite({ y: 4, x: 1 });
a === b;                      // true, one object
new Set([a, b]).size;         // 1
new Map([[a, 'ship']]).get(Composite({ x: 1, y: 4 })); // 'ship'
```

An earlier revision of the proposal instead kept every composite distinct and compared them through a ```Composite.equal``` walk, with ```Map```, ```Set```, and the array search methods special-cased to use it. The pivot to interning ([#32](https://github.com/tc39/proposal-composites/pull/32)) is what this document integrates, and everything below assumes it. The pivot matters here for a specific reason: it moves all of the cost and all of the semantics into one operation, creation, and creation is exactly the operation a type system can see the shape of.

This document does four things. It states the adopted semantics and where this proposal deviates. It works out what interning means under a typed runtime, where SameValueZero is type-sensitive and the signed-zero problem turns out to be one instance of a general canonicalization rule. It catalogs where composites get used, each use with its cost. And it draws the line between a composite and a value type class, which overlap enough that the comparison decides whether composites pull their weight at all.

## The Design as Adopted

The upstream semantics carry over unchanged except where the next sections say otherwise:

- ```Composite(source)``` requires an object - ```Composite(null)``` throws - and reads its **own enumerable string-keyed** properties; the source itself is never converted, it only supplies values. Inherited and non-enumerable properties are ignored, getters are invoked eagerly exactly once, and an own enumerable symbol key is a TypeError. Symbols remain fine as *values*.
- The result is **frozen**, has a **null [[Prototype]]**, ```typeof``` ```"object"```, and is not a constructor; ```new Composite(...)``` throws. ```Composite.isComposite(v)``` answers membership, and a Proxy over a composite is not a composite.
- Keys are stored **sorted** - integer-indexed keys first in ascending numeric order, then the remaining string keys lexicographically - so a composite is a canonical form independent of the argument's enumeration order.
- Two creations intern to the same object when the sources have the same keys and **SameValueZero**-equal values per key. Values held by a composite that are not themselves composites keep their ordinary semantics: an object field compares by identity, so ```Composite({ v: {} }) !== Composite({ v: {} })``` while two mentions of the *same* object are one key.
- A composite can hold **any value**. It is shallowly immutable: deeply immutable exactly when everything it holds is.
- Composites are refused in every **weak position** - a ```WeakMap``` key, a ```WeakSet``` value, a ```WeakRef``` target, a ```FinalizationRegistry``` target or unregister token - since a structurally-reachable value has no identity-based lifetime to observe. This is the same rule this proposal already applies to value types in the weak references section, for the same reason.
- The intern registry is **per-agent**, so composites are ```===``` across same-agent realms, like ```Symbol.for``` symbols, and the null prototype is what keeps a composite from being affiliated with the realm that happened to create it.
- Cycles cannot form: a composite is frozen at birth and can only reference values that already existed, so nested composites are trees terminating in non-composite leaves. Interning recursion likewise stops at non-composite objects.

Two consequences worth stating. ```Composite``` is idempotent - ```Composite(c)``` on a composite reads its entries and interns to ```c``` itself. And the collections integration requires no specification text at all: ```map.get(key)``` finds an interned key because it is the same object, which is the part of the pivot that deletes the old design's special cases rather than relocating them.

## Interning Under a Typed Runtime

Upstream, SameValueZero compares numbers as one type. Here it does not. This proposal's value types are distinct at runtime - ```uint8(1)```, ```uint16(1)```, and the plain ```number``` ```1``` are three values of three types, ```===```-unequal, with ```==``` alone comparing mathematical value across them - and SameValue and SameValueZero require the same runtime type before they compare payloads, which is what makes ```uint8(1)``` and ```1``` distinct ```Map``` keys. Interning inherits that:

```js
const a = Composite({ (x: uint8): 1 });
const b = Composite({ x: 1 });
a === b;              // false: a uint8 field and a number field are different keys
a.x;                  // uint8 1 - exactly what was stored, deterministically
Composite({ (x: uint8): 1 }) === a; // true
```

This is the right rule, and it is worth seeing why the alternative fails. Suppose interning ignored types and ```Composite({ (x: uint8): 1 })``` interned with ```Composite({ x: 1 })```. The two calls store different values - a ```uint8``` and a ```number``` - so what ```.x``` reads back would depend on which call ran first anywhere in the agent. That is precisely the nondeterminism the upstream proposal identifies for ```-0``` and resolves by normalization, and there is no normalization available here: the types are not convertible into each other implicitly anywhere else in this proposal, and picking one would silently discard the other's type. Type-sensitive interning dissolves the problem: equal-by-SameValueZero implies same type, so the stored value is fully determined by any of the calls that produce it.

### The Signed Zero Rule Generalizes

SameValueZero being type-sensitive *across* types still leaves its behavior *within* a type, and this proposal has already settled that in the equality clauses: within a numeric type, SameValueZero compares numerical value while SameValue distinguishes representations. Binary floats have ```-0``` and ```+0```. The decimal types have entire cohorts - ```1.0``` and ```1.00``` are distinct ```decimal128``` representations, SameValue-distinct, SameValueZero-equal.

Interning forces the same decision for each of these that upstream faced for ```-0```: SameValueZero-equal sources must intern to one object, and that object stores one representative, so the representative must not depend on creation order. The rule, stated once:

**A composite stores the canonical representative of each field value's SameValueZero equivalence class.** For binary floats that is ```+0``` for a zero, matching upstream's normalization of ```-0```. For decimals it is the cohort's canonical member. NaN needs no choice a program can observe. Every other value is alone in its class.

```js
Composite({ (v: float32): -0 }).v;        // float32 +0
Composite({ (v: decimal128): 1.00 }).v;   // the canonical 1, whichever cohort member arrived
Composite({ (v: float64): NaN }) === Composite({ (v: float64): NaN }); // true
```

The upstream FAQ floats a ```preserveNegativeZero``` options bag for callers who need the distinction. This proposal drops it: an options argument that changes interning identity fragments the registry into flag-keyed families, and a program that needs a signed zero or a specific cohort member to be part of a key can carry it as data - a sign field, an exponent field - which keys correctly with no special case. The general lesson is the reason to write this section at all: ```-0``` was never a quirk, it was the one SameValueZero equivalence class with more than one member that untyped JavaScript could see. A typed runtime has more of them, and interning is only coherent alongside canonicalization.

### The Typed and Untyped Boundary

Type-sensitive interning has one hazard, and it should be named rather than discovered. A typed producer and an untyped consumer do not meet:

```js
// module A, typed
cache.set(Composite({ (id: uint32): 7, (page: uint8): 2 }), results);

// module B, untyped
cache.get(Composite({ id: 7, page: 2 })); // undefined - number fields, different keys
```

The miss is silent, which is the worst kind. The mitigation is that the boundary is exactly where this proposal already puts annotations: give the key a named shape and create it through that shape on both sides, and the types propagate to the literals identically wherever the creation happens.

```js
interface CacheKey { id: uint32; page: uint8; }
cache.set(Composite.<CacheKey>({ id: 7, page: 2 }), results);
cache.get(Composite.<CacheKey>({ id: 7, page: 2 })); // results
```

A contextual type flows into the call the same way - ```const k: Composite.<CacheKey> = Composite({ id: 7, page: 2 })``` types the literal fields through the signature ```Composite<T extends object>(source: T): Composite.<T>``` and overload resolution's return-type direction. The flip side of that convenience is that the *same source text* ```Composite({ id: 7, page: 2 })``` denotes different interned objects in different contexts, because the literals take different types. That is literal propagation behaving exactly as it does everywhere else in this proposal, but composites make it an identity question rather than a representation question, so it is called out here: **an unannotated ```Composite``` call in typed code produces ```number``` fields**, and code that means anything else should say so at the creation site.

## Typed Composites

```Composite.<T>``` for an object type ```T``` is the type of composites whose fields are exactly ```T```'s members at ```T```'s types. It sits in the type grammar like any parameterized type - in annotations, unions, generic arguments, ```Map.<Composite.<K>, V>``` - and ```Composite``` unparameterized is the family's top, the type of every composite, which is what an untyped or heterogeneous creation infers to.

```js
interface IPoint { x: int32; y: int32; }

const p = Composite.<IPoint>({ x: 1, y: 2 }); // p: Composite.<IPoint>, fields are int32
const q: IPoint = p;                          // fine: a composite satisfies the interface
Reflect.typeOf(p);                            // Composite.<{ x: int32; y: int32 }>, interned
p instanceof Composite;                       // true
p instanceof Composite.<IPoint>;              // true, and narrows from Composite or any
```

The pieces behind that example:

**Fields carry their types in the ordinary way.** A composite's properties are own data properties, and this proposal's property descriptors carry a ```type```; ```Object.getOwnPropertyDescriptor(p, 'x').type``` is ```int32```. Nothing composite-specific is added to reflection - ```Reflect.typeOf``` returns the interned structural type object, the same one ```Composite.<{ x: int32; y: int32 }>``` written in source denotes, so type-object identity works: two composites of the same shape have ```===``` runtime types.

**Interface satisfaction is a check, never a cast.** Elsewhere in this proposal, passing an object literal where an interface is expected casts it - attaches the interface's member types - and the interfaces section guarantees an object can't be mutated out of an implementation. A composite is frozen and *shared*: the same interned object may be in unrelated hands across the agent, so writing anything onto it at a check would be action at a distance. None is needed. Frozenness makes check-only sound - a frozen object trivially can't be modified out of an implementation - and a typed composite already carries exact field types, so checking ```Composite.<{ x: int32; y: int32 }>``` against ```IPoint``` is a subtype test on interned type objects, constant time. An *untyped* composite checked against ```IPoint``` fails on its ```number``` fields by the no-implicit-conversion rule, which is the boundary hazard above wearing its interface clothes; the fix is the same, type the creation.

One interface feature doesn't reach composites: an optional member's default (```c?: uint8 = 0```) is described as supplying the value where a lacking object is checked, and a frozen composite can't receive it. The coherent reading for composites is that the member simply stays absent and reads as absent. Whether defaults should materialize on *any* checked object or be virtual at the read is feedback this comparison generates for the interfaces section; composites merely make the mutating reading impossible.

**Value class instances convert in both directions.** ```Composite(v)``` on a value type class instance reads its public fields - own, enumerable, typed - and interns a composite of them. For the way back, this document extends the ```:=``` literal rule to composite sources: ```composite := Vector2``` fills the class layout directly and runs no constructor, the treatment [serialization](serialization.md) already gives parsed runtime data, and a frozen canonical composite is as good a source as the literal that made it. The bridge is two spellings, both explicit:

```js
class Vector2 { x: float32; y: float32; }
const v = new Vector2();
const key = Composite(v);            // Composite.<{ x: float32; y: float32 }>
const w = key := Vector2;            // a fresh Vector2 copy of those fields
```

Two prints of that bridge to keep in view. Private fields are not properties, so they don't reach the composite: two instances differing only in ```#hidden``` state produce the *same* composite, which is correct for a key but surprising for a fingerprint. And the class's nominal brand is dropped - ```Composite``` is structural, so two different value classes with identical field names and types intern together. A program that wants the brand in the key puts the type in the key, and interned type objects make that free: ```Composite({ kind: Vector2, x: v.x, y: v.y })```.

## Tuple Composites

The upstream proposal declines an array form for now, sketching only a ```Composite.of``` that would spell ```{ 0: 'a', 1: 'b', 2: 'c', length: 3 }```, and its stated reservations are that the result would have an enumerable ```length```, no ```Symbol.iterator```, and a creation cost that grows with the element count. This proposal keeps the array form, because the objections are prototype problems and cost problems, and a typed runtime dissolves the first and prices the second. One honesty note first: upstream's ```Composite``` applied to an array already returns something today - the index-keyed *record*, since ```length``` is own but non-enumerable and is dropped - so the tuple kind below changes the result of an existing call rather than filling a hole, and the deviations list flags it as such.

```Composite([...])``` - ```Array.isArray``` on the argument decides the kind - creates a **tuple composite**: an exotic array object, frozen, null-prototyped, ```Array.isArray``` true, integer-indexed, with an own non-enumerable ```length```. Its static type is ```Composite.<[T1, T2, ...]>``` over this proposal's tuple types, and a homogeneous ```Composite.<[].<T>>``` covers the variable-length case.

```js
const m = Composite([uint8(42), uint8(12), uint8(67)]); // Composite.<[uint8, uint8, uint8]>
m[0];                  // uint8 42
m.length;              // 3
Array.isArray(m);      // true
Composite([...m, uint8(99)]) === Composite.<[4].<uint8>>([42, 12, 67, 99]); // true

for (const x: uint8 of m) { /* typed iteration */ }
```

The prototype objections go away because iteration stops being a prototype lookup. In typed code the ```...``` iteration operator is type-directed: ```for..of```, spread, and destructuring of a ```Composite.<[...]>``` compile to the index loop the main proposal already specifies as observably equivalent for its typed arrays. In untyped code, ```for..of``` and spread recognize the tuple-composite kind and iterate the elements directly - a spec addition this document owns - so no ```Symbol.iterator``` has to live on an object whose prototype is deliberately null; the manual protocol spelling is ```Array.prototype.values.call(m)```. ```length``` being own and non-enumerable keeps ```Object.keys``` to the elements. Array generics that don't mutate - ```map```, ```slice```, ```includes```, ```indexOf``` - work applied explicitly (```Array.prototype.map.call(m, f)```) or through the typed signatures the [standard library](standardlibrary.md) extension gives them; the mutating ones throw on a frozen receiver as they do for any frozen array.

Because ```length``` doesn't participate in enumerable-key equality, tuple and record composites must not intern into one namespace - ```Composite([1])``` and ```Composite({ 0: 1 })``` would otherwise collide while disagreeing about shape. The intern key therefore includes the kind: **a tuple composite and a record composite are never equal**. This also keeps the reflection split in [decorators](decorators.md) crisp: ```Reflect.Tuple``` reflects the array-backed kind, ```Reflect.Record``` the object-backed kind.

The cost reservation is real and stays: creation hashes every element, so an unbounded list is a poor key, and upstream's advice to prefer small named shapes stands. What types add is that the *common* tuple key - a pair, a triple, a fixed row - has statically known length and element types, so its creation compiles to a fixed sequence of typed hashes with no shape discovery, which is the fast path the performance section prices.

## Where Composites Are Used

The catalog below is the answer to "what is this feature for", grouped, each entry with its cost shape. The recurring pattern is worth stating first: **a composite is created once and compared many times**. Creation is the O(fields) operation; every comparison after it is a pointer. Uses that fit that pattern love interning; uses that create fresh never-repeated composites pay creation each time and get nothing back for it.

### Keys and Membership

**Multi-column Map keys.** The proposal's motivating case: a key with more than one part, where today's workarounds are string concatenation or nested maps. Spatial hashes and chunk stores keyed on coordinates, sparse matrices on ```(row, col)```, permissions on ```(userId, resourceId)```, graph edge weights on ```(from, to)```:

```js
const chunks = new Map.<Composite.<{ cx: int32; cy: int32 }>, Chunk>();
chunks.set(Composite({ (cx: int32): x >> 4, (cy: int32): y >> 4 }), chunk);
```

Cost: one creation per lookup *expression*, O(1) map probe after. The lookup-side creation is the hot cost and the performance section returns to it.

**Memoization and cache keys.** A function's argument tuple as the key - the dynamic-programming state, the query key of the fetch-cache pattern ```(endpoint, params)```, request coalescing on ```(method, url)```. Interning makes the second occurrence of a key free to compare and lets the cache be a plain ```Map```.

**Deduplication and visited sets.** BFS/DFS visited states ```(x, y, direction)```, unique ```(r, g, b)``` palette entries, seen record dedup during import. A ```Set``` of composites is the whole implementation.

**Array membership.** ```positions.includes(Composite({ x, y }))``` and ```indexOf``` work unchanged - ```includes```' SameValueZero and ```indexOf```'s strict equality both land on the interned pointer - so a small collection doesn't need to graduate to a ```Set``` to get value semantics. O(n) pointer scan after one O(fields) creation.

**Grouping and joining.** ```Map.groupBy(rows, r => Composite({ dept: r.dept, role: r.role }))``` groups on a compound key directly; a hash join between two datasets builds one side's ```Map``` on the join-column composite and probes it with the other's. These are the database shapes, and composites are the missing compound key.

**Interned tokens.** A composite is a value that behaves like a shared symbol with data: ```Composite({ kind: 'error', code: 404 })``` matches by ```===``` anywhere in the agent, across realms, with no registry of constants to import. ```switch``` on composites works by the same token, since ```case``` comparison is strict equality. Type-and-value registry keys fall out too, since type objects are interned references: ```Composite({ event: PointerDown, phase: 'capture' })``` keys a handler table with no strings to misspell.

### Value Objects and Change Detection

**Small immutable value objects.** Points, sizes, ranges ```{ start, end }```, colors, version triples, ```{ amount, currency }``` money, date parts ```{ y, m, d }``` as calendar keys. Where the object exists to *be a value* - compared, deduplicated, keyed - a composite is the direct spelling. Where it exists to *compute* - methods, operators, bulk storage - the value type class comparison below takes over.

**Multi-value results worth comparing.** A ```{ quotient, remainder }``` return that a caller checks against an expected pair, a parser position, a reducer's ```{ state, effects }``` pair. When results are composites, "did anything change" is ```===```.

**Change detection and memoized rendering.** The props-comparison pattern: hold the last composite of inputs, recompute when the new one differs. Interning turns a field-by-field shallow compare into a pointer compare, and the dependency-array idiom becomes a tuple composite. The cost moved, not vanished - creation hashes the fields the compare used to walk - but it's paid once per *distinct* value rather than per comparison site.

**Snapshots and assertions.** A test's expected structure as a composite makes deep-equal into ```===``` for the value-shaped parts; property-based testing dedups generated cases in a ```Set```. Leaves that are ordinary objects still compare by identity, which is the right default; the upstream FAQ's purity and reliability grounds for refusing an equality-customization protocol stand unchanged here.

### Compile Time and Meta

**Canonical compound constants.** The [type programming](typeprogramming.md) extension keys compile-time function memoization on argument tuples and canonicalizes compound constant arguments "by structural value — precisely the value semantics the composites extension gives tuples and records." A ```pick(User, ['id', 'name'])``` whose array argument differs by reference at every occurrence needs exactly one canonical identity per contents, which is interning verbatim: composites at compile time are the identity rule value generics already use, extended to aggregates. Specialization caches keyed on type-argument tuples are the same shape.

**Cross-realm tokens.** Per-agent interning makes a composite the value to hand across a same-origin iframe boundary when both sides need to recognize it later without a shared module instance.

### Boundaries

**JSON.** A composite stringifies as its data - a record as an object, a tuple as an array - and interning is deliberately not on the wire; it is an agent-local identity, not a serialization format. The typed parse direction re-interns: ```JSON.parse.<Composite.<ServerConfig>>(text)``` validates against ```ServerConfig``` per the [serialization](serialization.md) rules and interns the result, so "has the config changed since last poll" is ```===``` between two parses. This answers the open question serialization.md records for composites.

**Structured clone and workers.** ```structuredClone``` and ```postMessage``` carry a composite's kind and contents and re-intern on the receiving agent. Equality is preserved *within* each agent - two messages carrying equal composites are ```===``` after arrival - but a composite is never ```===``` across agents, because the registries are disjoint. A job-dedup table on the worker side works; comparing a worker's composite against the main thread's by identity is a category error the types can flag, since agents don't share references at all outside the [threading](threading.md) extension. Whether that extension's shared heap wants a process-wide registry is an open question below, and the default answer is no: a shared registry serializes every creation in every thread through one synchronized table.

### Where Not To Use Them

Each of these has a better tool in this proposal, and naming them is half the value-class comparison:

- **Bulk element storage.** A million points is ```[1000000].<Vector2>``` - eight megabytes, contiguous - not a million registry-resident heap objects. Composites are keys and tokens, not columns.
- **Hot mutation.** An accumulator, a particle, a running total: a value type local or field, mutated in place and copied by assignment. Deriving composite-after-composite in a loop is an interning-table workout with nothing looked up twice.
- **Layout, wire formats, GPU upload.** A composite has no ```byteLength``` and no defined memory image; ```@packed``` value classes and array views exist for exactly this.
- **Weak caching.** Composites throw in weak positions. A cache that must not retain its keys uses ordinary objects, or holds the composite's *constituent* trackable objects weakly, as the upstream FAQ suggests.
- **Wide or unbounded keys.** Creation is O(size) every time the key expression runs. A key of hundreds of fields or an arbitrary-length list is better restructured - a small named shape, or an explicit content hash the program manages.
- **Cross-agent identity.** Per-agent registries, as above; bytes and value types cross threads, references don't.

## Composites Against Value Type Classes

The overlap is real, so start with it. For a shape of pure value-typed fields, a composite and a value type class instance have **the same observable equality**: ```===``` compares contents (fieldwise for the class, by interned identity for the composite), ```Map``` and ```Set``` key structurally, ```==``` agrees where defined, and both are refused in weak positions. This program can't tell which it's holding without asking:

```js
class V2 { x: float32; y: float32; }
interface P2 { x: float32; y: float32; }

const a = new V2(); a.x = 1; a.y = 4;
const b = new V2(); b.x = 1; b.y = 4;
a === b;                                  // true, fieldwise
new Set([a, b]).size;                     // 1, structural key, copied in

const c = Composite.<P2>({ x: 1, y: 4 });
const d = Composite.<P2>({ x: 1, y: 4 });
c === d;                                  // true, one interned object
new Set([c, d]).size;                     // 1, pointer key
```

That lack of difference is the point: this proposal's keyed-collections rule already committed ```Map``` to structural value keys, so composites introduce no second equality regime here - they are the same regime with a different implementation strategy and a wider domain. The differences are capability and cost:

| | Value type class | Composite |
|---|---|---|
| Declaration | Nominal, declared ahead | Structural, ad hoc at the expression |
| Field types | Value types only (a reference position makes the class a reference type) | Any values: references, functions, symbols, mixed |
| Shape | Fixed per class | Any shape per creation; sorted canonical keys |
| Mutation | Fields writable; copy-on-assign contains it | Frozen at birth |
| Behavior | Methods, accessors, overloaded operators | Data only; null prototype |
| ```===``` | Fieldwise compare, O(fields) each time | Pointer compare, O(1); O(fields) paid once at creation |
| ```Map```/```Set``` key | Hash fieldwise per operation; key copied in | Interned pointer; O(1) per operation |
| Storage | Inline in ```[N].<T>``` and SoA, ```byteLength```, views, ```@packed``` | Individual heap object plus registry entry |
| Identity | None; a value is its fields | Object identity that *coincides* with contents |
| Weak positions | TypeError | TypeError |
| Realms/agents | Values copy anywhere bytes go, threads included | ```===``` per agent, across realms; re-interned across agents |
| ```typeof``` / prototype | ```"object"```, class prototype | ```"object"```, null |

The decision rule falls out. **Declared shape doing arithmetic, living in bulk, or crossing a binary boundary: value type class.** It has the layout, the operators, the storage, and comparison costs that don't involve a global table. **Ad hoc shape, heterogeneous or reference-holding contents, keys compared repeatedly, tokens shared across modules and realms, or plain untyped code: composite.** It needs no declaration, holds what classes can't, and amortizes comparison to a pointer. The bridge between them is one call in each direction, shown in the typed-composites section, so data can live as a class and key as a composite without either side contorting.

Two unifying observations close the comparison. First, the keyed-collections rule that a value type key is *copied into* the collection and compared structurally is describing an internal canonical value - a composite is that value made reachable, and an implementation has latitude to implement value-type keys *by* interning them, at which point the two rows of the table above converge inside the engine. Second, the registry composites require is machinery this proposal already demands: type objects are interned structural values with a per-agent table, and composites are the same table shape with weak entries. An engine implementing this proposal builds one structural interner and points two features at it.

And the case against, stated fairly, since this document was written to decide it: composites add a global registry with GC obligations, a second spelling of "structural value" next to value classes, and an identity that depends on which agent you ask. If this proposal had only fully-typed programs to serve, value classes plus the keyed-collections rule would cover most of the catalog above, and composites would be ergonomics. The reasons that isn't the conclusion: the catalog's biggest entries - ad hoc map keys, heterogeneous keys, reference-holding keys, canonical compile-time constants - are outside what value classes can express at all; untyped and mixed programs, which this proposal is explicitly additive over, get value semantics from composites with no annotations anywhere; and the upstream proposal exists independently, so the choice for this proposal is not "add composites or don't" but "define how types make them precise, or leave the two features to collide later." This document is the former.

## Performance

The model, then the numbers-shaped claims an engine can be held to.

**Creation, miss:** sort or know the keys, hash each ```(key, field type, canonical value)``` triple, probe the registry, allocate the composite and its cached hash, insert a weak registry entry. O(fields), plus a hash-table insert whose cost tracks the registry's load factor.

**Creation, hit:** same hash and probe, one fieldwise compare against the candidate, return the existing object. O(fields), **no allocation** - an implementation services a hit by hashing the source in place, which matters because the lookup side of every map access in the catalog is a hit after the first time.

**Comparison:** pointer equality. ```Map```/```Set``` operations on a composite key: the stored hash and a pointer compare, O(1).

**The value-class comparison in numbers.** A value type key pays a fieldwise hash on every collection operation and a fieldwise compare on bucket collisions, allocating nothing, ever. A composite pays fieldwise once per creation *expression* and a pointer after. For a key whose expression runs once and is then compared k times, the composite does F + k field-units of work against the value class's F·k - the composite wins for k ≥ 2 and the win compounds. For streaming keys compared once each, the value class wins on constants: no registry probe, no GC story, no allocation even on novel keys. Hot lookups that re-create the key expression each call sit in between and are the case to engineer: the creation is a hit, so it's F work and no allocation, against the value class's F - a wash on work, with the composite paying the registry probe and the value class paying nothing. Hoisting the composite out of the loop converts it to pure pointer operations, which no value-class key can reach.

**Memory.** A ```{ x: float64, y: float64 }``` value copy inline in a collection is 16 bytes. The composite is an object header, two slots, a cached hash, and a registry entry - on the order of 48-72 bytes in a typical 64-bit layout - held once per *distinct* value rather than per occurrence. Dedup-heavy workloads come out ahead; storage-heavy workloads are the anti-use case already named.

**The registry's GC.** Entries are weak: a composite with no references outside the table is collected and its entry swept, the standard hash-consing arrangement. The failure mode to benchmark is the treadmill - a workload creating never-repeated composites (the memoizer whose keys never recur) allocates, inserts, and sweeps at full rate and would have been better off with value-type keys or an explicit hash. The engine262 work should measure exactly this before the design hardens, since it is the one workload interning makes *worse* than the pre-pivot design.

**What types buy the engine.** A creation site with a statically known shape - ```Composite.<CacheKey>({...})```, a typed literal, a tuple of known arity - compiles to a fixed sequence: no key sorting (the order is known), no descriptor walk (the fields are data slots), monomorphic per-field hashes (the types are known), a precomputed shape for the allocation on miss. The untyped call keeps the generic walk. This is the recurring shape of this proposal - the same semantics either way, with annotations converting dynamic discovery into straight-line code - and it is why no ```#{}``` syntax is needed here for performance: ```Composite({...})``` with a static type already carries everything a syntax form would, though if the upstream follow-on adds the syntax it composes with these types unchanged.

**Concurrency.** Per-agent registries need no synchronization. The [threading](threading.md) extension's shared heap is where a shared registry would be proposed, and the default position above is that it shouldn't be: creation would serialize threads through one table, and cross-thread keys are better carried as value types and re-interned, which is O(fields) at the boundary - the same cost the message-passing story already pays.

## Deviations From the Upstream Proposal

For feedback in both directions, the deltas in one place:

- **Interning is type-sensitive**, inheriting this proposal's SameValueZero. ```Composite({ (x: uint8): 1 })``` and ```Composite({ x: 1 })``` are distinct. Upstream has one number type and no such distinction to make.
- **Canonicalization is stated as a rule** - the stored value is its SameValueZero class's canonical representative - of which upstream's ```-0``` normalization is the binary-float instance and decimal cohorts are a second instance. The ```preserveNegativeZero``` options bag is dropped.
- **The tuple kind is kept**: array sources make frozen exotic arrays with null prototypes, typed as ```Composite.<[...]>```, iterated through the type-directed operator in typed code and by kind recognition in untyped code. Upstream defers a list form, and its current call on an array yields the index-keyed record (```length``` is non-enumerable and dropped), so this is a behavioral change to an existing call, flagged for upstream feedback. The intern key includes the kind, so tuples and records never collide.
- **Typed creation and the ```Composite.<T>``` types** are added; interface satisfaction on composites is check-only; ```Reflect.typeOf``` returns interned structural composite types; ```Reflect.Tuple``` and ```Reflect.Record``` in [decorators](decorators.md) reflect the two kinds.
- **JSON and structured clone mappings** are defined (data on the wire, re-intern on read), answering serialization.md's open question.
- Everything else - creation semantics, sorted keys, symbol-key refusal, frozenness, null prototype, weak-position refusal, per-agent cross-realm identity, cycle impossibility, ```Composite.isComposite``` - is adopted as upstream specifies.

## Open Questions

- **Canonical decimal representative.** Which cohort member a composite stores for a decimal field - exponent zero, the shortest form, or IEEE's canonical encoding - should be pinned alongside the decimal extension rather than here. The requirement from this side is only that it be a function of the numerical value.
- **SameValueZero on typed floats in the implementation.** The engine262 branch currently routes typed operands of SameValueZero through the SameValue path, which makes ```float32(-0)``` and ```float32(0)``` distinct ```Map``` keys, contrary to the equality clauses this document builds on. Composite interning needs the spec'd behavior; the fix belongs in the comparison operations, not in ```Composite```.
- **Should value-type ```Map``` keys be specified as interning?** The keyed-collections copy-in rule and composite interning converge; specifying the former as the latter would let ```map.getKey```-style APIs return the canonical key object. Latitude today; possibly a guarantee later.
- **Optional-member defaults on frozen objects.** Whether an interface default is virtual at the read or materialized at a check needs one answer for all frozen objects; composites only expose the question.
- **A shared-heap registry.** Refused above by default; if the [threading](threading.md) extension finds a compelling cross-thread key workload, a per-process registry with sharded locks is the design to evaluate against re-interning at the boundary.
- **Registry pressure benchmarks.** The never-repeating-key treadmill and high-load-factor creation costs should be measured in the engine262 implementation and reported upstream, where "performance expectations" currently notes the load-factor dependence without numbers.
- **Pattern matching.** If the match proposal advances, composites are natural scrutinees - frozen, canonical, sorted keys - and ```Composite.<T>``` patterns would narrow like ```instanceof```; coordination noted, nothing needed yet.
