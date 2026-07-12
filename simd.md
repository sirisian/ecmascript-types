# SIMD

The main proposal gives the SIMD types their shape: ```vector.<T, N>```, a fixed number of lanes of a single element type, with shorthand names like ```float32x4``` and ```boolean32x4```. The operators come from [operator overloading](operatoroverloading.md), which shows what a matrix multiply needs and what an engine must do to compile it well.

This document fills the gap between the two: how to get at an individual lane, how to name several of them at once, and how masks work. These are the parts that make SIMD code readable, and the reason the SIMD types were originally proposed as distinct types rather than as arrays of four floats.

Every ```vector.<T, N>``` is a value type. It copies on assignment, it has no identity, and ```typeof``` reports ```"object"```, so it has a prototype and the accessors described below live there.

## Lanes

Two forms, and the difference between them is the difference between one instruction and several.

```js
class vector<T, N: uint32> {
	operator vector(value: T); // Broadcast: a cast operator from the lane type T, filling every lane

	lane<I: uint32>(): T where I < N; // Compile-time index
	withLane<I: uint32>(value: T): vector.<T, N> where I < N;

	get operator[](index: uint32): T; // Runtime index
	set operator[](index: uint32, value: T);
}
```

```lane.<I>()``` takes its index as a value generic argument, so the index is a constant and the extraction is a single instruction: ```extractps``` on x86, ```mov``` from a lane on ARM. ```withLane.<I>(value)``` is the insert, returning a new vector, since a vector is a value.

```operator[]``` takes a runtime index. It is the same operation, and it is slower: an index that isn't known until run time forces either a variable permute or a spill to memory and a reload. Use it when the index is data, not when the index is a constant you happened to store in a variable.

```js
const v = float32x4(1, 2, 3, 4);
v.lane.<2>(); // 3, one instruction
v[i]; // Also 3 when i is 2, but the engine cannot fold it

const w = v.withLane.<0>(10); // float32x4(10, 2, 3, 4)
let u = v;
u[0] = 10; // Same result, mutating a value in place
```

Bit vectors get lane access for free, which is what makes ```boolean8``` a usable bitfield rather than a name for a byte. Lane ```i``` is bit ```i```, counting from the least significant bit, matching how ```movmskps``` and ```vpmovmskb``` number their results:

```js
let a: boolean8 = 0b00000010; // vector.<boolean1, 8>
a[1]; // 1
a[0]; // 0
a[3] = 1;
a; // 0b00001010
```

The conversion runs both ways by this bit pattern: an integer value assigned to a boolean vector fills lane ```i``` from bit ```i```, and a boolean vector read where an integer is expected produces the integer with exactly those bits set. The literal above and the ```0b00001010``` readout are the one rule applied in each direction.

## Component Accessors

A vector of four or fewer lanes has names for its lanes, in two sets:

```js
const v = float32x4(1, 2, 3, 4);
v.x; // 1, a float32
v.w; // 4
v.rgb; // vector.<float32, 3>, holding (1, 2, 3)
v.xxxx; // float32x4(1, 1, 1, 1)
v.wzyx; // float32x4(4, 3, 2, 1)
v.bgra; // float32x4(3, 2, 1, 4)
```

The shorthand names in the main proposal cover the register widths, so ```float32x4``` has a name and a three-lane float vector does not. A narrowing accessor produces ```vector.<T, L>``` and is written that way. A library that wants ```Vector3``` gives it one, as [primitive metadata](primitivemetadata.md) does with ```vec3```.

```xyzw``` and ```rgba``` name the same lanes. GLSL has a third set, ```stpq```, for texture coordinates; HLSL and WGSL dropped it, and so does this document. Web code targets WGSL, and the third set costs a third of the names for nothing.

The rules are the ones every shading language settled on:

- The receiver's type is ```vector.<T, N>``` with ```N``` at most four. Wider vectors have no names, because the alphabet runs out. They use ```swizzle``` below.
- Every character comes from one set. ```v.xg``` mixes them and is a TypeError.
- Every character's lane index is less than ```N```. ```v.z``` on a two-lane vector is a TypeError.
- The result is ```T``` for one character and ```vector.<T, L>``` for ```L``` characters, for ```L``` up to four. A two-lane vector can produce a four-lane one: ```v.xyxy```.
- An accessor with no repeated character is assignable. One with a repeat is not, because it would name the same lane twice.

```js
type Vector2 = vector.<float32, 2>;

let v = float32x4(1, 2, 3, 4);
v.xy = Vector2(9, 8); // float32x4(9, 8, 3, 4)
v.zw = v.xy; // float32x4(9, 8, 9, 8)
// v.xx = Vector2(1, 2); // TypeError: x names the same lane twice
v.xy += Vector2(1, 1); // Compound assignment works, as it does anywhere
```

### They Are Properties, Not Syntax

Counting both sets and every result length from one to four, a four-lane vector has 680 accessor names, and the two-, three-, and four-lane shapes have 980 between them. That number is the reason to think this needs a grammar rule, and it is the reason it doesn't.

The accessors live on the prototype. They are shared by every vector of that shape, cost nothing per value, and are non-enumerable, exactly like every other built-in accessor. So:

```js
Object.keys(v); // [], as before. Prototype properties are not own properties
for (const k in v) {} // Nothing, because they are non-enumerable
'xyz' in v; // true
v['xyz']; // Works. A computed access is an ordinary property lookup
Reflect.get(v, 'wzyx'); // Works
```

The last three lines are the point. If swizzles were a grammar production keyed on the receiver's static type, then ```v.xyz``` would mean a shuffle when ```v``` is annotated and ```undefined``` when it is ```any```. That would be the only place in this proposal where adding a type annotation changes what a program does, rather than adding a check to what it already did. Prototype accessors keep typed and untyped code meaning the same thing.

None of the 680 names collides with a method on the vector prototype: the letters are drawn from ```xyzw``` and ```rgba```, and ```lane```, ```swizzle```, ```shuffle```, ```select```, ```all```, ```any```, and ```sum``` all contain a letter that is in neither set.

The one visible consequence is that ```Object.getOwnPropertyNames(float32x4.prototype)``` returns several hundred names. An engine generates them from a rule and installs them lazily, which is what Rust's ```glam``` does with a macro and what ```glm``` does with proxy structs, at compile time in both cases and at zero runtime cost in this one.

### Desugaring

A component accessor is sugar for a permutation, which is a real instruction:

```js
v.xzzw; // v.swizzle.<0, 2, 2, 3>()
v.x; // v.lane.<0>()
v.xy = w; // A blend of w into lanes 0 and 1 of v
```

So the accessors add no code generation. The intrinsic below is what an engine compiles, and the names are how a person reads it.

## Permutation

```js
partial class vector<T, N: uint32> {
	swizzle<...I: [].<uint32>>(): vector.<T, I.length>; // One source
	shuffle<...I: [].<uint32>>(other: vector.<T, N>): vector.<T, I.length>; // Two sources
}
```

Indices are value generic arguments, so they are compile-time constants. This is not a stylistic choice. The immediate operand of ```shufps``` and the lane index of ```vdupq_laneq_f32``` are encoded in the instruction, so an index known only at run time cannot compile to either.

For ```swizzle```, an index selects a lane of the receiver and must be less than ```N```. For ```shuffle```, indices ```0``` to ```N - 1``` select lanes of the receiver and ```N``` to ```2N - 1``` select lanes of the argument. Both are checked at compile time, because both are constants.

```js
const a = float32x4(1, 2, 3, 4);
const b = float32x4(5, 6, 7, 8);

a.swizzle.<0, 0, 0, 0>(); // (1, 1, 1, 1), a broadcast
a.swizzle.<3, 2, 1, 0>(); // (4, 3, 2, 1)
a.swizzle.<0, 1>(); // vector.<float32, 2>, narrowing
a.shuffle.<0, 1, 4, 5>(b); // (1, 2, 5, 6)
```

```swizzle``` is where the eight- and sixteen-lane vectors do their permuting, since they have no component names.

## Masks

A comparison between vectors produces a mask: one lane per input lane, every bit set or every bit clear. The mask's element type is the boolean type of the same width as the compared element, which is why ```boolean32``` exists as ```vector.<boolean1, 32>```.

```js
const a = float32x4(1, 2, 3, 4);
const b = float32x4(4, 3, 2, 1);
const m: boolean32x4 = a < b; // (true, true, false, false)

partial class vector<T, N: uint32> {
	sum(): T; // Horizontal add across the lanes
}

partial class vector<boolean1, N: uint32> { // Mask operations: boolean lanes only
	all(): boolean; // Every lane set. movmskps + cmp, or uminv
	any(): boolean; // Some lane set
	select<U>(whenSet: vector.<U, N>, whenClear: vector.<U, N>): vector.<U, N>;
}
```

```select``` is how branchless code is written. It is ```blendvps``` on x86 and ```bsl``` on ARM, and it evaluates both arguments, which is the trade a branch would have made anyway when the lanes disagree:

```js
const clamped = m.select(a, b); // Lanes of a where a < b, lanes of b elsewhere
if (m.any()) {} // Reduce a mask to a boolean
```

Masks are vectors, so they swizzle and index like any other:

```js
m.x; // boolean32
m[2]; // boolean32
m.xyxy; // boolean32x4
```

Return type overloading gives a comparison both of its useful shapes, which is how Intel's separate ```_mm_cmpeq_epi32``` and ```_mm_cmpeq_epi32_mask``` intrinsics become one operator:

```js
const asVector: int32x4 = int32x4(0, 1, 2, 3) == int32x4(0, 1, 3, 2);
const asMask: boolean32x4 = int32x4(0, 1, 2, 3) == int32x4(0, 1, 3, 2);
```

## Wrapping a Vector

A math library has to decide whether its ```Vector4``` is a class holding a ```float32x4``` or an alias for one. Component accessors decide it.

```js
type Vector4 = vector.<float32, 4>; // Swizzles. Operators from the primitive vector block
class Vector4 { v: float32x4; } // A Vector4 instance does not swizzle; its float32x4 field does
```

A class does not inherit its field's prototype, so a wrapper has no ```.xyz``` unless it declares one, and declaring 680 of them is not a plan. This pushes a library toward the alias for its vector types, where operators come from the ```primitive vector<...>``` block in [primitive metadata](primitivemetadata.md) rather than from class members.

The cost is nominal distinction. ```Quaternion``` and ```Vector4``` are both four floats, and as aliases they would be the same type, so ```Vector4 * Quaternion``` could not be told from ```Vector4 * Vector4```. The resolution is that they want different things: a quaternion needs ```x```, ```y```, ```z```, and ```w``` and nothing else, so it stays a class with four accessors, while a vector wants all 680 and so stays an alias. Rust's ```glam``` makes exactly this split, with ```Vec4``` swizzling and ```Quat``` not.

Stated as a rule: alias a vector type for storage and the full component-accessor set, and make it a nominal class when it carries an operator vocabulary of its own. The reason is the same global operator table the [operator overloading](operatoroverloading.md) extension describes - an alias's operators live on the shared lane type, so a quaternion aliasing ```float32x4``` would have nowhere to put a Hamilton-product ```*``` distinct from the component-wise one, and two libraries aliasing the same lane type would collide on every operator they both define. A class keeps its operators to itself.

```js
type Vector2 = vector.<float32, 2>;
type Vector3 = vector.<float32, 3>;
type Vector4 = vector.<float32, 4>;

class Quaternion {
	v: float32x4;
	get x(): float32 { return this.v.x; }
	get y(): float32 { return this.v.y; }
	get z(): float32 { return this.v.z; }
	get w(): float32 { return this.v.w; }
}
```

## Portable Widths

```vector.<T, N>``` takes any lane count, which is the portable shape - a kernel is written once against a width - but the width that runs best differs by target: 128 bits on WebAssembly and baseline x86-64, 256 on AVX2, 512 on AVX-512, 128 on AArch64. ```vector.preferredLanes(T)``` is a compile-time constant giving the platform's natural lane count for element type ```T```, so a kernel can pick the width the machine wants without branching on it:

```js
const Lanes = vector.preferredLanes(float32); // 4, 8, or 16 depending on the target
type FloatBatch = vector.<float32, Lanes>;
```

It is a value generic argument like any other constant, so a generic pass over a column reads it once and specializes. A vector wider than the hardware supports is not an error: an operation on it lowers to a sequence over the hardware width - two ```float32x4``` operations for a ```float32x8``` on a 128-bit target - so ```vector.<float32, 8>``` is always defined, just not always one instruction. Writing ```vector.<float32, vector.preferredLanes(float32)>``` is how a kernel stays both portable and optimal.

Gather, scatter, and masked load and store are not in the first version. They are the operations that vectorize an indirect or predicated loop - reading ```data[index[i]]``` across lanes, or storing only where a mask is set - and they matter for exactly the entity-component sweeps this proposal is aimed at, but WebAssembly's ```simd128``` does not have them either, so a portable baseline cannot assume them. They are named here as a deliberate gap rather than an oversight; a later version would add them as methods on ```vector.<T, N>``` that lower to the portable width the same way the arithmetic operators already do.

## Instructions

| Expression | x86-64 | AArch64 |
| --- | --- | --- |
| ```v.lane.<0>()``` | ```extractps``` | ```mov w0, v0.s[0]``` |
| ```v[i]``` | variable permute, or spill and reload | ```mov``` with a computed lane, or spill |
| ```v.withLane.<0>(s)``` | ```insertps``` | ```mov v0.s[0], w0``` |
| ```v.xxxx``` | ```shufps``` or ```vbroadcastss``` | ```dup v0.4s, v1.s[0]``` |
| ```v.wzyx``` | ```shufps``` | ```rev64``` plus ```ext```, or ```tbl``` |
| ```a.shuffle.<0, 1, 4, 5>(b)``` | ```shufps``` | ```zip1``` |
| ```v.xy = w``` | ```blendps``` or ```movsd``` | ```mov``` of two lanes |
| ```a < b``` to a mask | ```cmpltps``` | ```fcmgt v.4s``` |
| ```m.select(a, b)``` | ```blendvps``` | ```bsl v.16b``` |
| ```m.all()``` | ```movmskps``` plus ```cmp``` | ```uminv``` plus ```fmov``` |
| ```m.any()``` | ```movmskps``` plus ```test``` | ```umaxv``` plus ```fmov``` |

## Other Languages

| | Component accessors | Permutation | Masks |
| --- | --- | --- | --- |
| GLSL | ```xyzw```, ```rgba```, ```stpq```, in the grammar | Built in | ```bvec4```, ```any```, ```all``` |
| HLSL, WGSL | ```xyzw```, ```rgba```, in the grammar | Built in | ```bool4```, ```any```, ```all``` |
| Rust ```glam``` | Macro-generated trait methods | ```shuffle``` | ```Vec4Mask```, ```any```, ```all``` |
| C++ ```glm``` | ```GLM_FORCE_SWIZZLE```, proxy structs | ```_mm_shuffle_ps``` | ```bvec4``` |
| Zig | None | ```@shuffle``` with index vectors | ```@select```, ```@reduce``` |
| C# | ```.X```, ```.Y```, ```.Z```, ```.W``` only | ```Vector128.Shuffle``` | ```Vector128<T>``` as a mask |
| Swift | ```.x``` through ```.w```, plus halves | ```SIMD``` subscripts | ```SIMDMask``` |
| Java | None | ```rearrange(VectorShuffle)``` | ```VectorMask``` |

Two observations. The shading languages put component accessors in the grammar because they have no prototypes to hang them on; a language with prototypes doesn't need the grammar. And ```glm``` is the cautionary case: it implements swizzles with unions of proxy objects, pays for them in compile time, and gates them behind a macro that most projects leave off.

## Open Questions

- Component accessors for eight- and sixteen-lane vectors. There are no names, and inventing some would be worse than ```swizzle.<...>()```. Deferred permanently unless a convention appears.
- A dynamic swizzle, where the permutation is data rather than a constant. ```pshufb``` and ```tbl``` exist, and Java's ```VectorShuffle``` is a runtime value. It is a different operation with a different cost, and it deserves its own name rather than a computed property.
- Bit fields. ```uint.<4>``` members packing into a class, with ```@offsetBit``` and a defined bit order, are a separate feature from bit vectors, and belong with the member memory layout rules rather than here.
- Whether ```v[i] = value``` on a value type should be an error rather than an in-place mutation, given that ```withLane``` exists and is cheaper. The array section already permits it for value type elements, so consistency says keep it.
