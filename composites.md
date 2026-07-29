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

This document does five things. It states the adopted semantics and where this proposal deviates. It works out what interning means under a typed runtime, where SameValueZero is type-sensitive and the signed-zero problem turns out to be one instance of a general canonicalization rule. It catalogs where composites get used, each use with its cost. It draws the line between a composite and a value type class, which overlap enough that the comparison decides whether composites pull their weight at all. And it defines how composites behave as pattern matching subjects, where interning buys a compilation no other object subject can have.

## The Design as Adopted

The upstream semantics carry over unchanged except where the next sections say otherwise:

- ```Composite(source)``` requires an object - ```Composite(null)``` throws - and reads its **own enumerable string-keyed** properties; the source itself is never converted, it only supplies values. Inherited and non-enumerable properties are ignored, getters are invoked eagerly exactly once, and an own enumerable symbol key is a TypeError. Symbols remain fine as *values*.
- The result is **frozen**, has a **null [[Prototype]]**, ```typeof``` ```"object"```, and is not a constructor; ```new Composite(...)``` throws. ```Composite.isComposite(v)``` answers membership, and a Proxy over a composite is not a composite.
- Keys are stored **sorted** - integer-indexed keys first in ascending numeric order, then the remaining string keys lexicographically - so a composite is a canonical form independent of the argument's enumeration order.
- Two creations intern to the same object when the sources have the same keys and **SameValueZero**-equal values per key. Values held by a composite that are not themselves composites keep their ordinary semantics: an object field compares by identity, so ```Composite({ v: {} }) !== Composite({ v: {} })``` while two mentions of the *same* object are one key.
- A composite can hold **any value**. It is shallowly immutable: deeply immutable exactly when everything it holds is.
- Composites are refused in every **weak position** - a ```WeakMap``` key, a ```WeakSet``` value, a ```WeakRef``` target, a ```FinalizationRegistry``` target or unregister token - since a structurally-reachable value has no identity-based lifetime to observe. This is the same rule this proposal already applies to value types in the weak references section, for the same reason.
- The intern registry belongs to the **heap**, so composites are ```===``` across the realms sharing one, like ```Symbol.for``` symbols, and the null prototype is what keeps a composite from being affiliated with the realm that happened to create it. Under the worker model a heap is an agent and the two spellings coincide; where a heap is shared by several threads the registry is shared with it, because the invariant is a statement about objects and cannot be scoped more finely than objects are reachable.
- Cycles cannot form: a composite is frozen at birth and can only reference values that already existed, so nested composites are trees terminating in non-composite leaves. Interning recursion likewise stops at non-composite objects.
- **Interning consults no user code.** The source's own values are read once, getters included, and everything after that - canonicalization, hashing, and the comparison that decides a hit - is computed by the implementation from the field names and the stored values. A composite's identity is a function of its contents and nothing else: no hash or equality protocol participates in it, and admitting one would make creation reentrant and an identity depend on when a method was installed. What a *collection* treats as the same key is a separate question from what a composite is, and only the second one is settled here.

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

SameValueZero being type-sensitive *across* types still leaves its behavior *within* a type, and this proposal has already settled that in the equality clauses: within a numeric type, SameValueZero compares numerical value while SameValue distinguishes representations. Binary floats have ```-0``` and ```+0```, and a NaN carries a sign and a payload. The decimal types have entire cohorts - ```1.0``` and ```1.00``` are distinct ```decimal128``` representations, SameValue-distinct, SameValueZero-equal.

Interning forces the same decision for each of these that upstream faced for ```-0```: SameValueZero-equal sources must intern to one object, and that object stores one representative, so the representative must not depend on creation order. The rule, stated once:

**A composite stores the canonical representative of each field value's SameValueZero equivalence class.** Three classes have more than one member, and each gets a representative:

- **Signed zeros.** A zero stores as ```+0```, at every float width, matching upstream's normalization of ```-0```.
- **NaNs.** A NaN stores quiet, positive, with a zero payload. Untyped JavaScript never had to say this because it cannot observe a payload; this proposal can, by reading the field into a value class and viewing its bytes, so without a representative the payload of whichever call created the composite first would be visible to every later holder.
- **Decimal cohorts.** Where the field's type declares a scale, the value arrives at that scale - quantization at an assignment boundary is the [decimal](decimal.md) rule, and a composite's field is such a boundary - so the cohort has collapsed before interning sees it and the composite stores exactly the declared scale. Where the type declares no scale, the reduced member is stored: trailing zeros are stripped, the one member computable from the numerical value alone, independent of the width, and the value the hash had to be taken over in any case.

```js
Composite({ (v: float32): -0 }).v;                                      // float32 +0
Composite({ (v: float32): -0 }) === Composite({ (v: float32): 0 });     // true
Composite({ (v: float64): NaN }) === Composite({ (v: float64): NaN });  // true

type Cents = decimal128.<{ currency: 'USD', scale: 2 }>;
Composite({ (price: Cents): 19.9 }).price;                              // 19.90, the type's scale
Composite({ (price: Cents): 19.9 }) === Composite({ (price: Cents): 19.90 }); // true
Composite({ (v: decimal128): 1.00 }).v;                                 // 1, reduced
```

The division of labour there is the general one: canonicalization is the type's job wherever the type has an opinion and the composite's only where it doesn't. A money value carries its significance because ```Cents``` says so, and an unconstrained ```decimal128``` in a key is not the place to carry significance - which is the same advice the decimal types give anywhere else, since that value was going to be quantized at the first typed boundary it crossed. ```rational``` needs no rule at all: the type keeps every value in lowest terms, so each rational number has exactly one representation.

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

**Interface satisfaction is a check, never a cast.** Elsewhere in this proposal, passing an object literal where an interface is expected casts it - attaches the interface's member types - and the interfaces section guarantees an object can't be mutated out of an implementation. A composite is frozen and *shared*: the same interned object may be in unrelated hands anywhere in the heap, so writing anything onto it at a check would be action at a distance. None is needed. Frozenness makes check-only sound - a frozen object trivially can't be modified out of an implementation - and a typed composite already carries exact field types, so checking ```Composite.<{ x: int32; y: int32 }>``` against ```IPoint``` is a subtype test on interned type objects, constant time. An *untyped* composite checked against ```IPoint``` fails on its ```number``` fields by the no-implicit-conversion rule, which is the boundary hazard above wearing its interface clothes; the fix is the same, type the creation.

**An optional member's default is filled at creation, never at a check.** A default (```c?: uint8 = 0```) belongs to construction - it is written wherever a value is being built, which is an object literal checked against the interface, a ```:=``` fill, a typed parse, and ```Composite.<T>({...})```. For a composite the default is written before freezing, so it is part of the contents that intern, and the two spellings of one key stay together:

```js
interface CacheKey { id: uint32; page?: uint8 = 0; }

const k = Composite.<CacheKey>({ id: 7 });
k.page;                                          // 0, filled at creation
k === Composite.<CacheKey>({ id: 7, page: 0 });  // true, same contents
```

Checking a composite that *already exists* writes nothing to it. An optional member is optional, so ```Composite({ (id: uint32): 7 })``` - built without the member and frozen before it ever met the interface - satisfies ```CacheKey``` with the member absent, reads ```undefined``` for ```page```, and answers false to ```'page' in c```. It is a different composite from ```k``` above, which is correct: they have different contents. These are one rule - a default fills a value under construction, a check tests a value that exists - and composites are where the difference stops being academic, because a frozen shared object is the case where filling at a check is not merely undesirable but impossible.

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

The prototype objections go away because iteration stops being a prototype lookup. In typed code the ```...``` iteration operator is type-directed: ```for..of```, spread, and destructuring of a ```Composite.<[...]>``` compile to the index loop the main proposal already specifies as observably equivalent for its typed arrays. In untyped code, ```for..of``` and spread recognize the tuple-composite kind and iterate the elements directly, so no ```Symbol.iterator``` has to live on an object whose prototype is deliberately null; the manual protocol spelling is ```Array.prototype.values.call(m)```. ```length``` being own and non-enumerable keeps ```Object.keys``` to the elements. Array generics that don't mutate - ```map```, ```slice```, ```includes```, ```indexOf``` - work applied explicitly (```Array.prototype.map.call(m, f)```) or through the typed signatures the [standard library](standardlibrary.md) extension gives them; the mutating ones throw on a frozen receiver as they do for any frozen array.

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

**JSON.** A composite stringifies as its data - a record as an object, a tuple as an array - and interning is deliberately not on the wire; it is a heap-local identity, not a serialization format. The typed parse direction re-interns: ```JSON.parse.<Composite.<ServerConfig>>(text)``` validates against ```ServerConfig``` per the [serialization](serialization.md) rules and interns the result, so "has the config changed since last poll" is ```===``` between two parses. The mapping is stated in [serialization](serialization.md) alongside the other targets.

**Structured clone and workers.** ```structuredClone``` and ```postMessage``` carry a composite's kind and contents and re-intern on the receiving agent. Equality is preserved *within* each agent - two messages carrying equal composites are ```===``` after arrival - but a composite is never ```===``` across agents, because the registries are disjoint. A job-dedup table on the worker side works; comparing a worker's composite against the main thread's by identity is a category error the types can flag, since agents don't share references at all outside the [threading](threading.md) extension. That extension's shared heap is a different case from a worker: it forces a registry spanning threads rather than merely permitting one, since thread-local registries would put two composites of equal contents, both reachable from both threads, in one heap.

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

Two unifying observations close the comparison. First, the keyed-collections rule that a value type key is *copied into* the collection and compared structurally is describing an internal canonical value - a composite is that value made reachable, and an implementation has latitude to implement value-type keys *by* interning them, at which point the two rows of the table above converge inside the engine. Second, the registry composites require is machinery this proposal already demands: type objects are interned structural values with a table of the same scope, and composites are the same table shape with weak entries. An engine implementing this proposal builds one structural interner and points two features at it.

And the case against, stated fairly, since this document was written to decide it: composites add a global registry with GC obligations, a second spelling of "structural value" next to value classes, and an identity that depends on which agent you ask. If this proposal had only fully-typed programs to serve, value classes plus the keyed-collections rule would cover most of the catalog above, and composites would be ergonomics. The reasons that isn't the conclusion: the catalog's biggest entries - ad hoc map keys, heterogeneous keys, reference-holding keys, canonical compile-time constants - are outside what value classes can express at all; untyped and mixed programs, which this proposal is explicitly additive over, get value semantics from composites with no annotations anywhere; and the upstream proposal exists independently, so the choice for this proposal is not "add composites or don't" but "define how types make them precise, or leave the two features to collide later." This document is the former.

## Pattern Matching

A composite is the best-behaved subject the [pattern matching proposal](https://github.com/tc39/proposal-pattern-matching) can be handed - frozen, canonical, sorted, getter-free, null-prototyped - and one addition is needed to make the two features meet correctly.

**```Composite``` carries its own ```Symbol.customMatcher```**, returning ```Composite.isComposite(subject)```. This is required rather than decorative: the matcher installed on ```Function.prototype``` invokes a non-constructor function as a predicate, so absent one of its own, ```when Composite:``` would *call* ```Composite(subject)```, match every object because the result is truthy, run the subject's getters, and intern a composite as a side effect of a test.

```js
match (value) {
  when Composite: 'a composite';              // Composite.isComposite(value)
  when Composite.<IPoint>: 'a typed one';     // type test, narrows to Composite.<IPoint>
  when { kind: 'key', let code }: handle(code);
}
```

The type form in that second arm relies on ```is``` being one operator whose right side is the pattern grammar, with a type as one of its forms: ```value is Composite.<IPoint>``` is then the structural test this proposal defines and ```value is { let x }``` is the pattern proposal's, rather than two boolean operators sharing a name and both taking something brace-shaped.

**Object patterns need no special casing.** Every key of a composite is own, enumerable, and non-configurable, there are no getters and no prototype, so a subset pattern tests exactly what it names and a rest pattern collects the remaining fields. The object-pattern caching the proposal specifies - guarding against getters that are expensive or non-idempotent - is unnecessary for a composite subject, whose every read is idempotent by construction, so an implementation may skip it.

**Array patterns reach tuple composites through iteration.** They obtain an iterator from the subject, which is what the tuple kind's iterability is for: without it ```when [let x, let y]:``` would silently fail to match every tuple composite, which is the difference between a feature and a trap.

**A composite constant is the cheapest pattern in the language.** A variable pattern tests SameValue against the resolved value, and for an interned composite that is a pointer comparison. A ```match``` whose arms are composite constants compiles to a sequence of pointer compares, and a long one to a hash dispatch on the pointer - the jump table ```switch``` has always had for primitives, reaching compound keys for the first time.

```js
const DISMISS = Composite({ kind: 'key', code: 27 });
match (event) {
  when DISMISS: dismiss();     // one pointer comparison
  default: pass(event);
}
```

**Literal patterns take their field types from the subject**, by the same propagation that types a creation site: ```when { code: 27 }``` against a ```Composite.<{ kind: string; code: uint8 }>``` tests a ```uint8``` 27, not a ```number``` 27, which would otherwise never match. Where the subject's type is unknown the literal is a ```number``` and the mismatch is a static error rather than a silent non-match. The pattern proposal's rule that a bare ```0``` matches with SameValueZero while ```+0``` and ```-0``` match with SameValue carries over per type.

Exhaustiveness is unchanged: composite patterns are structural, and this proposal checks exhaustiveness only over an ```enum``` and a sealed class. A shape that wants the check puts one of those in a discriminating field - ```Composite({ kind: Phase.Running, ... })``` - and switches on the field.

## Performance

The model, then the numbers-shaped claims an engine can be held to.

**Creation, miss:** sort or know the keys, hash each ```(key, field type, canonical value)``` triple, probe the registry, allocate the composite and its cached hash, insert a weak registry entry. O(fields), plus a hash-table insert whose cost tracks the registry's load factor.

**Creation, hit:** same hash and probe, one fieldwise compare against the candidate, return the existing object. O(fields), **no allocation** - an implementation services a hit by hashing the source in place, which matters because the lookup side of every map access in the catalog is a hit after the first time.

**Comparison:** pointer equality. ```Map```/```Set``` operations on a composite key: the stored hash and a pointer compare, O(1).

**The value-class comparison in numbers.** A value type key pays a fieldwise hash on every collection operation and a fieldwise compare on bucket collisions, allocating nothing, ever. A composite pays fieldwise once per creation *expression* and a pointer after. For a key whose expression runs once and is then compared k times, the composite does F + k field-units of work against the value class's F·k - the composite wins for k ≥ 2 and the win compounds. For streaming keys compared once each, the value class wins on constants: no registry probe, no GC story, no allocation even on novel keys. Hot lookups that re-create the key expression each call sit in between and are the case to engineer: the creation is a hit, so it's F work and no allocation, against the value class's F - a wash on work, with the composite paying the registry probe and the value class paying nothing. Hoisting the composite out of the loop converts it to pure pointer operations, which no value-class key can reach.

**Memory.** A ```{ x: float64, y: float64 }``` value copy inline in a collection is 16 bytes. The composite is an object header, two slots, a cached hash, and a registry entry - on the order of 48-72 bytes in a typical 64-bit layout - held once per *distinct* value rather than per occurrence. Dedup-heavy workloads come out ahead; storage-heavy workloads are the anti-use case already named.

**The registry's GC.** Entries are weak: a composite with no references outside the table is collected and its entry swept, the standard hash-consing arrangement. The failure mode is the treadmill - a workload creating never-repeated composites (the memoizer whose keys never recur) allocates, inserts, and sweeps at full rate and would have been better off with value-type keys or an explicit hash. It is worse than it first looks: an ordinary frozen object in that loop is a candidate for scalar replacement, since it demonstrably doesn't escape, while a composite must escape by definition - publishing it in the registry is what the feature *is* - so the optimization that makes the untyped equivalent free is unavailable in principle. This is the one workload interning makes *worse* than the pre-pivot design, and the guidance is blunt: a key expression whose value never recurs should be a value type key or an explicit hash, and a key that does recur should be hoisted out of the loop that compares it.

**What types buy the engine.** A creation site with a statically known shape - ```Composite.<CacheKey>({...})```, a typed literal, a tuple of known arity - compiles to a fixed sequence: no key sorting (the order is known), no descriptor walk (the fields are data slots), monomorphic per-field hashes (the types are known), a precomputed shape for the allocation on miss. The untyped call keeps the generic walk. This is the recurring shape of this proposal - the same semantics either way, with annotations converting dynamic discovery into straight-line code - and it is why no ```#{}``` syntax is needed here for performance: ```Composite({...})``` with a static type already carries everything a syntax form would, though if the upstream follow-on adds the syntax it composes with these types unchanged.

**Concurrency.** The registry belongs to the heap, so under the worker model - one heap per agent - creation needs no synchronization at all. The [threading](threading.md) extension shares one heap across threads, which makes one registry across threads not a choice but the same rule applied to a different runtime: thread-local registries would put two composites of equal contents, both reachable from both threads, in the heap at once. Creation is then concurrent, and the arrangement that keeps it cheap is lock-free reads with sharded or compare-and-swap insertion behind a small per-thread cache on each creation site, so hits never touch shared memory. A composite is also the tear-free way to publish a multi-field snapshot between threads, since it is built before it is published and publishing it is a single reference store.

## Deviations From the Upstream Proposal

For feedback in both directions, the deltas in one place:

- **The registry is scoped to the heap** rather than to the agent. The two coincide under the worker model, which is the only one upstream has; the [threading](threading.md) extension shares one heap across threads, and a registry scoped more finely than objects are reachable would put two composites of equal contents, both reachable from both threads, in one heap.
- **Interning is type-sensitive**, inheriting this proposal's SameValueZero. ```Composite({ (x: uint8): 1 })``` and ```Composite({ x: 1 })``` are distinct. Upstream has one number type and no such distinction to make.
- **Canonicalization is stated as a rule** - the stored value is its SameValueZero class's canonical representative - of which upstream's ```-0``` normalization is the binary-float instance and decimal cohorts are a second instance. The ```preserveNegativeZero``` options bag is dropped.
- **The tuple kind is kept**: array sources make frozen exotic arrays with null prototypes, typed as ```Composite.<[...]>```, iterated through the type-directed operator in typed code and by kind recognition in untyped code. Upstream defers a list form, and its current call on an array yields the index-keyed record (```length``` is non-enumerable and dropped), so this is a behavioral change to an existing call rather than a new one. The intern key includes the kind, so tuples and records never collide.
- **Typed creation and the ```Composite.<T>``` types** are added; interface satisfaction on composites is check-only; ```Reflect.typeOf``` returns interned structural composite types; ```Reflect.Tuple``` and ```Reflect.Record``` in [decorators](decorators.md) reflect the two kinds.
- **```Composite[Symbol.customMatcher]```** is added, without which the default matcher for a callable would make ```when Composite:``` invoke ```Composite``` as a predicate - matching every object and interning one as a side effect. This one is independent of types and applies to the upstream proposal as it stands.
- **JSON and structured clone mappings** are defined: data on the wire, re-interned on read, which is the mapping [serialization](serialization.md) had left open for composites.
- Everything else - creation semantics, sorted keys, symbol-key refusal, frozenness, null prototype, weak-position refusal, cross-realm identity within one registry, cycle impossibility, ```Composite.isComposite``` - is adopted as upstream specifies.
