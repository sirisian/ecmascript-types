# Structure of Arrays

```[].<T>``` stores its elements interleaved: for a ```T``` with fields ```x```, ```y```, and ```rotation```, memory reads ```x y rotation x y rotation```. That is the right default. It is also the wrong layout for a large class of programs, and the reason is the cache line rather than the language.

A loop that touches two fields of a five-field element drags the other three through cache with them. A loop that touches one field across every element strides over the rest, so a vector unit that could have loaded four consecutive values loads one and skips. A renderer that wants every ```position``` to hand to the GPU has to gather them into a staging buffer first. The fix has one shape everywhere it appears: store one array per field.

```SoA.<T>``` is that layout, as a type. Elements are stored as parallel per-field arrays, while indexing, assignment, ```ref``` bindings, iteration, and ```push``` behave exactly as they do on ```[].<T>```. The layout changes; the code does not.

```js
class Transform {
	x: float32;
	y: float32;
	rotation: float32;
}

const a: [1000].<Transform>; // x y rotation x y rotation ...
const b: SoA.<Transform, 1000>; // xxx... yyy... rrr...

b[10].x; // Reads the x column at index 10
const ref t = b[10];
t.rotation += 0.1; // Writes the rotation column at index 10
b.fields.x; // [1000].<float32>: every x, contiguous
```

## Why a Type and Not a Keyword

Every language that offers this offers it as a library or a macro. Zig's ```std.MultiArrayList(T)``` is the reference design, built on compile-time metaprogramming. Rust's ```soa_derive``` generates a companion type per struct. C++ builds proxy iterators from templates, which is how EnTT stores components. Julia's ```StructArrays.jl``` uses multiple dispatch. Of the languages with a native directive, Jai's ```#soa```, the directive buys surface syntax over what the others already have.

So ```SoA.<T>``` is a type, not new grammar. It's a built-in exotic in the same way ```[].<T>``` is: something no user-defined class could express, specified by the language and provided by the engine. Two capabilities are what put it out of userland's reach, and they're exactly what this proposal already has in the engine:

- **Storage derived from a type's fields.** Declaring one array per field of ```T``` is a mapped type, which this proposal deliberately doesn't have. The engine, which lays out ```T``` anyway, needs no new mechanism.
- **Element references into storage.** ```ref b[10]``` must be a reference *into* the columns, not a copy or a wrapper object. The [value type references](README.md) in the main proposal already denote a storage location; here that location is a column set and an index.

The alternative that other languages reach for, and this one shouldn't, is the **proxy object**: element access returns a small object holding the container and an index, forwarding field reads to the columns. It's the standard workaround, and this proposal forecloses it twice over. Proxying a layout-backed typed object is a TypeError by the rules in the main proposal, because a trap on every field access is the precise opposite of the check elision typed objects exist to enable. Even the legal form, an interface-typed proxy, checks each trapped read at the read. A ```ref``` into an ```SoA``` is the proxy pattern with the indirection compiled out: a field access on it is an indexed load from a column whose base offset is known at compile time.

## Type

```js
class SoA<T, Length: uint32 = 0> {
	constructor();
	constructor(length: uint32); // Growable arrays only

	get length(): uint32;
	get capacity(): uint32; // Growable arrays; the allocation backing every column
	get byteLength(): uint32;
	get fields(): Fields.<T>;

	push(value: T): uint32;
	pop(): T | undefined;
	reserve(n: uint32): void; // Grow every column to hold at least n elements
	fill(value: T): SoA.<T, Length>;
	*operator...(): T;

	static from<T>(values: [].<T>): SoA.<T>;
	static withCapacity<T>(n: uint32): SoA.<T>; // Empty, capacity >= n
	toArray(): [].<T>;

	static get elementByteLength(): uint32; // Per element, summed over the columns
	static get alignment(): uint32;
}
```

```Length``` follows the array types: ```0``` is growable, and a nonzero value is a fixed extent, as ```[N].<T>``` is to ```[].<T>```.

```T``` must be a value type class, since a class with a reference field has nothing to split. A primitive ```T``` is permitted and degenerates to a single column, so generic code that may or may not be handed a primitive needs no special case.

## Layout

An ```SoA.<T>``` stores one array per **immediate field** of ```T```. A field that is itself a value type stays one column, interleaved within itself:

```js
class Vec2 {
	x: float32;
	y: float32;
}
class Projectile {
	origin: Vec2;
	direction: Vec2;
	speed: float32;
}

const p: SoA.<Projectile, 4>;
// origin:    x y x y x y x y
// direction: x y x y x y x y
// speed:     s s s s
```

Splitting one level rather than recursively to the leaves is deliberate, and it's the same choice ```MultiArrayList``` makes. A consumer that wants ```origin``` as a contiguous stream of ```Vec2``` - a GPU vertex attribute, a SIMD pass over positions - gets it. A consumer that wants every ```origin.x``` contiguous can flatten ```Projectile``` into ```originX```, ```originY```, and so on. Which is wanted is a property of the data's use, and inlining the decision into the layout rule would take it away from the programmer who knows.

There is no interior padding between an element's fields, because an element's fields are no longer adjacent. Each column is padded and aligned on its own, so a ```T``` whose interleaved layout pads to a larger stride has an ```SoA.byteLength``` smaller than its ```[].<T>``` equivalent. ```SoA.<T, Length>.byteLength``` is that full laid-out size, following the type-byteLength convention and the layout table in the main proposal; ```elementByteLength``` is the per-element sum of column strides.

Columns are aligned to at least the platform's vector width, so a lane load at an aligned index is an aligned load.

## Elements

Every operation that reads or writes an element behaves as it does on ```[].<T>```.

```js
const particles: SoA.<Particle, 1024>;

particles[0]; // Gathers a Particle value from the columns
particles[0] = spawned; // Scatters the fields into the columns
particles.push(spawned); // Appends to every column
particles.length; // The element count, not a column length
```

A ```ref``` binding is a reference to the element, which for an ```SoA``` is a column set and an index. Field accesses through it compile to a load or store on one column:

```js
const ref p = particles[i];
p.velocity += gravity * dt; // velocity column at i
p.lifetime -= dt; // lifetime column at i

for (const ref p of particles) { // Reference iteration, as on any typed array
	p.position += p.velocity * dt;
}
```

Because a reference names storage rather than an object, ```SoA``` and ```[].<T>``` present the same interface to every function that takes a ```ref Particle```. A system written against one storage works against the other, including one written against the reference callback parameters in the main proposal, where the container supplies the references and never says which layout produced them.

A reference into an ```SoA``` pins the container as well as the element: a ```push``` that reallocates moves every column, so growing an ```SoA``` while a reference into it is live is a TypeError, exactly as changing an array's length during ```ref``` iteration is.

## Fields

```fields``` projects each of ```T```'s immediate fields as an array view aliasing that field's column. The views are live: writes through them are visible through the element API and the reverse.

```js
class Vertex {
	position: vec3;
	uv: vec2;
	color: uint32;
}
const mesh: SoA.<Vertex, 65536>;

mesh.fields.position; // [65536].<vec3>
mesh.fields.color; // [65536].<uint32>
mesh.fields.color.fill(0xFFFFFFFF); // Writes one column across every element

// Upload one attribute with no gather and no staging copy:
device.queue.writeBuffer(positionBuffer, 0, [].<uint8>(mesh.fields.position));
```

A growable ```SoA.<T>``` projects growable ```[].<F>``` views; a fixed ```SoA.<T, N>``` projects ```[N].<F>```. Nested value type fields project as columns of that type, so ```p.fields.origin``` is a ```[].<Vec2>``` and ```p.fields.origin.x``` doesn't exist; flatten the class if that's what's wanted.

The projections live under ```fields``` rather than on the container so a field named ```length``` or ```push``` collides with nothing.

## Conversion

```SoA.<T>``` and ```[].<T>``` are distinct types with distinct layouts, and neither is assignable to the other. Conversion is explicit and copies:

```js
const array: [].<Transform> = [/* ... */];
const columns = SoA.from(array);
const back = columns.toArray();
```

Making the two silently interchangeable, as Julia's ```StructArrays``` does, reads well until a function needs the concrete layout, and then the abstraction has to be undone. Keeping the layout in the type means every call site knows which it has, and the engine compiles a fixed ABI for each. That property is what the next section rests on.

A byte view over an ```SoA``` sees the columns in declaration order, one after another. That is also its serialization order, and it's why ```byteLength``` is a sum of column lengths.

## SIMD and Optimization

Nothing about an ```SoA``` is dynamic once the type is specialized. ```SoA.<Transform>``` fixes the column set, each column's element type, and each field's column offset at compile time. A field access through a reference is one indexed load with a constant base: no hashing, no traps, no shape guard, no branch.

A ```fields``` projection is an ordinary typed array. Everything that vectorizes over ```[].<float32>``` vectorizes over a column, and nothing special is needed to make it so:

```js
// Autovectorizable: a dense read and a dense write, unit stride.
const xs = transforms.fields.x;
for (let i: uint32 = 0; i < xs.length; ++i) {
	xs[i] *= scale;
}

// Or explicitly. A column of float32 viewed as float32x4 needs no new API:
// it's an array view, and `window` takes the four-aligned prefix.
const whole = xs.length - xs.length % 4;
const lanes = [].<float32x4>(xs.window(0, whole));
const factor: float32x4 = scale; // Implicit SIMD constructor broadcasts
for (let j: uint32 = 0; j < lanes.length; ++j) {
	lanes[j] *= factor;
}
for (let i: uint32 = whole; i < xs.length; ++i) {
	xs[i] *= scale; // Tail
}
```

The interleaved equivalent can do neither: a stride of ```Transform.byteLength``` defeats the vector unit and pulls two unused fields into every cache line.

A pass that reads one column and writes another - ```position``` from ```velocity```, the shape of every integrator - needs no runtime overlap check to vectorize, because distinct fields are distinct allocations and so provably cannot alias. This is the single-threaded counterpart of the cache-line fact below: the same premise, that each field is its own column, lets a vector unit load and store two columns without proving at run time that they do not overlap. An engine cannot make that assumption about two arbitrary ```[].<T>``` arguments, since they might be the same array, nor about two ```ref``` parameters, since ```zip(a, a, ...)``` aliases them deliberately; the columns of one ```SoA``` are the case where the guarantee holds by construction rather than by analysis.

Growth reallocates all columns together under one capacity, amortized as ```[].<T>``` is.

Under the [threading](threading.md) extension, ```shared SoA.<Particle, Capacity>``` allocates its columns in shared memory. Two threads writing *different fields* of the same elements never touch the same cache line, because different fields are different columns. Interleaved storage cannot offer that; false sharing on adjacent fields is one of the standard reasons data-oriented code moves to this layout in the first place.

```js
let particles: shared SoA.<Particle, 65536>;
```

## Generic Code

```SoA.<T>``` composes with the compile-time type expressions in the main proposal: when the component is a compile-time constant, ```SoA.<componentType(C)>``` names that component's exact column type, which is what lets one storage abstraction hold differently typed columns behind a typed accessor. Construction, where the id is a runtime value, goes through a value-level factory instead. The [entity component system](examples/ecs.md) example is built on the combination:

```js
class Archetype {
	#columns: [TotalBits].<any>;

	constructor(mask: BitSet) {
		for (const id of mask) {
			this.#columns[id] = newColumn(id); // Per-type factory; a runtime id can't index a type expression
		}
	}

	// C is a compile-time constant here, so componentType(C) is a valid type expression.
	column<C: Component>(): SoA.<componentType(C)> {
		return this.#columns[C];
	}
}
```

## Notes

- Sorting scatters. ```toSorted``` on an ```SoA``` is specified to compute a permutation and apply it per column, which touches each column once, rather than gathering and rewriting elements.
- ```SoA.<T>``` participates in ```Reflect.typeOf``` and ```instanceof``` as any other type. It is not a subtype of ```[].<T>```.
- A ```T``` with a single field is an ```SoA``` with a single column, which is a correct if pointless instantiation, and useful for generic code that parameterizes over layout.
