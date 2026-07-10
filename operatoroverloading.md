# Operator Overloading

The main proposal defines operator overloading on classes and [primitive metadata](primitivemetadata.md) defines it on primitives. This document works both through a real target: a 2D and 3D math library of the kind every renderer, physics engine, and game ships. It exists to answer three questions the shorter sections leave open.

Can an expression like ```projection * view * model * position``` be written naturally? Can the engine compile it to the same instructions a C++ or Rust program would emit? And what does the language have to guarantee for the answer to be yes rather than *probably*?

The last question turns out to be the interesting one. Operator overloading over value types is not automatically fast. It is fast under four conditions, all of which this proposal either has or needs, and they are stated at the end.

## Declaring Operators

There are three places an operator can be declared, and they differ in which operand is the receiver.

**On a class**, where the receiver is the left operand:

```js
class Vector4 {
	v: float32x4;

	operator*(rhs: float32): Vector4 {} // Vector4 * float32
	operator*(rhs: Vector4): Vector4 {} // Vector4 * Vector4, component-wise
}
```

**By reopening a class**, using the class extension syntax, which appends operators to a type declared elsewhere:

```js
class Vector4 {
	operator*(rhs: Matrix4): Vector4 {} // Added from another module
}
```

**On a primitive**, using a ```primitive``` block, where the receiver is the primitive:

```js
primitive float32 {
	operator*(rhs: Vector4): Vector4 {} // float32 * Vector4
}
```

Resolution looks only at the left operand's type. For ```a * b```, the compiler searches the operator table of ```a```'s type for a signature whose parameter accepts ```b```. It never reverses the operands to find a match. That rule is not a limitation to work around, it is the reason ```Matrix4 * Vector4``` and ```Vector4 * Matrix4``` can mean different things, which in a column-major library they do: one transforms a column vector, the other a row vector. A language that silently reverses operands to find any match makes that bug unfindable.

Return type overloading applies to operators as it does to functions, which is how a SIMD comparison yields either a mask or a vector:

```js
operator<(v: int32x4): int32x4 {}
operator<(v: int32x4): boolean32x4 {}
```

Two applicable overloads with the same parameter types and no expected return type is an ambiguity TypeError, not a coin flip.

Note that ```static operator``` is a different feature. It defines an operator on the class object itself, so that ```A += 5``` means something, and has nothing to do with the operands of an instance's operators.

## Scalars on the Left

```v * 2``` is a member operator on ```Vector4```. ```2 * v``` is not: the left operand is a number, and numbers are primitives. This is the problem that motivated the issue behind this document, and every language with operator overloading has had to answer it.

C++ uses a free function, ```operator*(float, Vector4)```, which belongs to neither type. C# and Swift declare the reversed form inside the user's class, as a static operator taking both operands. Rust writes ```impl Mul<Vector4> for f32```, which its orphan rule permits because ```Vector4``` is local. Python looks for ```__rmul__``` on the right operand after ```__mul__``` on the left fails. All four put the declaration somewhere the *library author* controls, and none of them reverse operands implicitly except Python.

The ```primitive``` block is this proposal's version, and the direct spelling works:

```js
primitive number {
	operator*(v: Vector2): Vector2 {
		return v * this;
	}
}
```

Written out per type it doesn't scale. Three scalar types times eight math types is twenty-four blocks, and every new vector type in every library adds three more. The fix is to make the block generic, constrained by an interface that says what the right operand must support. An interface can declare operator members, which is what makes the constraint expressible:

```js
interface ScalarMultiply<S, R> {
	operator*(rhs: S): R;
}

primitive float32 {
	operator*<T extends ScalarMultiply.<float32, T>>(rhs: T): T {
		return rhs * this;
	}
}
```

One block per scalar type, covering every type that defines scalar multiplication, including ones that don't exist yet. The whole math library needs three: ```number```, ```float32```, and ```float64```. Division does not commute, so no such block exists for it, and ```2 / v``` remains a TypeError - which is correct, because it isn't defined.

An untyped numeric literal propagates into the operand type as it does anywhere else, so ```2 * v``` types the literal as ```float32``` and reaches the ```primitive float32``` block. The ```primitive number``` block exists for the case where the scalar is a ```number``` binding rather than a literal, and it narrows exactly as ```v * 2``` does.

A ```primitive``` block adds to a global operator table, which is the same exposure the class extension syntax already accepts. Two libraries defining ```float32 * Vector4``` for two different ```Vector4``` types coexist, since the parameter types differ. Two definitions for the same pair of types is a TypeError at the second declaration.

## Layout

The library is **column-major**, matching GLSL, WGSL, GLM, and therefore WebGL and WebGPU. ```Matrix4``` stores four columns, ```m * v``` transforms a column vector, and a matrix uploaded to the GPU needs no transpose. Nothing about this is forced by the language, and a row-major library would work as well, but the convention has to be stated once and loudly because getting it wrong produces code that is correct for square symmetric matrices and wrong for everything else.

```js
class Vector2 {
	x: float32;
	y: float32;
}

// Three floats, twelve bytes, tight in a vertex buffer.
class Vector3 {
	x: float32;
	y: float32;
	z: float32;
}

// Four lanes, sixteen bytes, w unused. Use this for math, Vector3 for storage.
class Vector3A {
	v: float32x4;
}

class Vector4 {
	v: float32x4;
}

class Quaternion {
	v: float32x4; // x, y, z, w
}

class Matrix2 {
	c: float32x4; // [m00, m10, m01, m11], two columns packed in one register
}

class Matrix3 {
	c0: float32x4;
	c1: float32x4;
	c2: float32x4; // Fourth lane of each column is unused
}

class Matrix4 {
	c0: float32x4;
	c1: float32x4;
	c2: float32x4;
	c3: float32x4;
}
```

The ```Vector3``` and ```Vector3A``` split is the one Rust's ```glam``` crate makes, and for the same reason: a three-float vector packs vertex data densely, and a four-lane one does arithmetic in one register. They convert explicitly. A library that only offers the packed form spends a shuffle on every operation, and one that only offers the padded form wastes a quarter of every vertex buffer.

Alignment is automatic. ```float32x4``` aligns to sixteen bytes, a class's alignment is the largest alignment among its members, so ```Matrix4.alignment``` is sixteen and ```Matrix4.byteLength``` is sixty-four with no annotation. This is what the natural alignment rule in the member memory layout section buys. Conversely, a ```@packed``` math type would defeat it, forcing unaligned loads on every access, so these types are never packed.

## Intrinsics

Operators alone can't express a matrix multiply efficiently. Four more operations are needed, and each maps to a single instruction on both major architectures. They are defined in the [SIMD](simd.md) extension and summarized here, because the code below depends on them.

**Lane permutation.** ```swizzle``` takes its lane indices as value generic arguments, so they are compile-time constants. That is not a stylistic choice: the immediate operand of ```_mm_shuffle_ps``` and the lane index of ```vdupq_laneq_f32``` must be encoded in the instruction, so an index that is only known at runtime cannot be compiled to either.

```js
v.swizzle.<0, 0, 0, 0>(); // Broadcast lane 0. SSE: shufps / AVX: vbroadcastss / NEON: vdupq_laneq_f32
v.swizzle.<2, 3, 0, 1>(); // Arbitrary permutation of one vector
a.shuffle.<0, 1, 4, 5>(b); // Two-source. Indices 0 to 3 select lanes of a, 4 to 7 of b
v.lane.<0>(); // Extract a lane as float32. SSE4.1: extractps
```

Lane-index arguments must be in range for the vector's width, which the compiler checks, since they are constants.

**Selection.** ```mask.select(a, b)``` chooses lanes from ```a``` where the mask is set and from ```b``` where it isn't. SSE4.1 ```blendvps```, NEON ```vbslq_f32```. With the comparison operators returning masks, this is how branchless code is written. ```mask.all()``` and ```mask.any()``` reduce a mask to a ```boolean```.

**Fused multiply-add.** ```Math.fma(a, b, c)``` computes ```a * b + c``` with a single rounding. It is overloaded for the scalar and vector types.

**Horizontal reduction.** ```v.sum()``` adds the lanes. It is the operation SIMD is worst at, and a library should use it sparingly. A matrix multiply written in terms of dot products is a row-major matrix multiply in disguise and is slower than the column form below.

The rest are ordinary overloads on ```Math``` over the vector types, lane-wise: ```sqrt```, ```abs```, ```min```, ```max```, ```floor```, and ```ceil```. ```Math.rsqrt(x)``` is exactly ```1 / Math.sqrt(x)```, correctly rounded, so it does not lower to a bare ```rsqrtps```, which is a twelve-bit approximation. An explicitly approximate reciprocal square root would be a deliberate exception to the strictness rule below, and is listed as an open question rather than smuggled in here.

A one-argument SIMD constructor broadcasts, so ```float32x4(s)``` is ```float32x4(s, s, s, s)```. This is the implicit SIMD constructor from the main proposal written explicitly.

## Floating Point Contraction

```Math.fma``` has to exist because the compiler is not allowed to introduce it. ```a * b + c``` and ```fma(a, b, c)``` produce different results: the first rounds the product before adding, the second doesn't. Contracting the former into the latter changes observable output.

C and C++ permit the contraction by default, under ```FP_CONTRACT```. Rust forbids it and provides ```f32::mul_add```. C# forbids it and provides ```Math.FusedMultiplyAdd```. Java forbids it and provides ```Math.fma```. JavaScript has no notion of relaxed floating point and should not acquire one: ```a * b + c``` means what it says, and a program that wants the fused form asks for it.

The same rule forbids reassociation. ```a + b + c``` is ```(a + b) + c``` and an engine may not vectorize a float reduction by reordering it. This is why ```sum()``` is an intrinsic rather than something a loop gets turned into, and why the [threading](threading.md) extension notes that atomic float addition is not associative.

The cost of the rule is exactly one instruction per multiply-add that the programmer forgot to write as ```fma```. The benefit is that a matrix multiply produces the same bits on every engine, which is the difference between a replay system that works and one that desynchronizes on a different laptop.

## Vector4

```js
class Vector4 {
	v: float32x4;

	constructor(x: float32, y: float32, z: float32, w: float32) {
		this.v = float32x4(x, y, z, w);
	}

	// A wrapper class does not inherit its field's accessors, so these are
	// written out. A library that wants the full set of component accessors
	// aliases the vector type instead, as the SIMD extension describes.
	get x(): float32 { return this.v.x; }
	get y(): float32 { return this.v.y; }
	get z(): float32 { return this.v.z; }
	get w(): float32 { return this.v.w; }

	operator+(rhs: Vector4): Vector4 { return { v: this.v + rhs.v } := Vector4; }
	operator-(rhs: Vector4): Vector4 { return { v: this.v - rhs.v } := Vector4; }
	operator*(rhs: Vector4): Vector4 { return { v: this.v * rhs.v } := Vector4; }
	operator/(rhs: Vector4): Vector4 { return { v: this.v / rhs.v } := Vector4; }

	operator*(rhs: float32): Vector4 { return { v: this.v * float32x4(rhs) } := Vector4; }
	operator/(rhs: float32): Vector4 { return { v: this.v / float32x4(rhs) } := Vector4; }

	operator-(): Vector4 { return { v: -this.v } := Vector4; }

	// Lane-wise, and it agrees with === here, since a value type's identity
	// is its fields and this class has one field.
	operator==(rhs: Vector4): boolean { return (this.v == rhs.v).all(); }

	operator+=(rhs: Vector4): Vector4 { this.v += rhs.v; return this; }
	operator*=(rhs: float32): Vector4 { this.v *= float32x4(rhs); return this; }
}

function dot(a: Vector4, b: Vector4): float32 {
	return (a.v * b.v).sum();
}
function length(a: Vector4): float32 {
	return Math.sqrt(dot(a, a));
}
function normalize(a: Vector4): Vector4 {
	return a * Math.rsqrt(dot(a, a));
}
```

```dot``` is a library function rather than an intrinsic: it is a lane-wise multiply followed by ```sum```, and naming it doesn't change what it compiles to.

```js
```

```float32x4(rhs)``` broadcasts a scalar to every lane, which is the implicit SIMD constructor from the main proposal written explicitly.

## Matrix4

The column-major transform is a linear combination of the matrix's columns, weighted by the vector's components. Each weight is a broadcast of one lane, and each accumulation is a fused multiply-add.

```js
class Matrix4 {
	c0: float32x4;
	c1: float32x4;
	c2: float32x4;
	c3: float32x4;

	// m * v, transforming a column vector. Eight instructions:
	// four broadcasts, one multiply, three fused multiply-adds.
	operator*(v: Vector4): Vector4 {
		return { v: this.#transformColumn(v.v) } := Vector4;
	}

	// m * n, one column of the result per column of n. Thirty-two instructions,
	// which is what a hand-written SSE routine emits.
	operator*(n: Matrix4): Matrix4 {
		return {
			c0: this.transformColumn(n.c0),
			c1: this.transformColumn(n.c1),
			c2: this.transformColumn(n.c2),
			c3: this.transformColumn(n.c3)
		} := Matrix4;
	}

	#transformColumn(c: float32x4): float32x4 {
		let r = this.c0 * c.swizzle.<0, 0, 0, 0>();
		r = Math.fma(this.c1, c.swizzle.<1, 1, 1, 1>(), r);
		r = Math.fma(this.c2, c.swizzle.<2, 2, 2, 2>(), r);
		r = Math.fma(this.c3, c.swizzle.<3, 3, 3, 3>(), r);
		return r;
	}

	operator*(s: float32): Matrix4 {
		const b = float32x4(s);
		return { c0: this.c0 * b, c1: this.c1 * b, c2: this.c2 * b, c3: this.c3 * b } := Matrix4;
	}

	operator+(n: Matrix4): Matrix4 {
		return { c0: this.c0 + n.c0, c1: this.c1 + n.c1, c2: this.c2 + n.c2, c3: this.c3 + n.c3 } := Matrix4;
	}

	// Eight shuffles, the same count as Intel's _MM_TRANSPOSE4_PS macro.
	transpose(): Matrix4 {
		const t0 = this.c0.shuffle.<0, 1, 4, 5>(this.c1);
		const t1 = this.c0.shuffle.<2, 3, 6, 7>(this.c1);
		const t2 = this.c2.shuffle.<0, 1, 4, 5>(this.c3);
		const t3 = this.c2.shuffle.<2, 3, 6, 7>(this.c3);
		return {
			c0: t0.shuffle.<0, 2, 4, 6>(t2),
			c1: t0.shuffle.<1, 3, 5, 7>(t2),
			c2: t1.shuffle.<0, 2, 4, 6>(t3),
			c3: t1.shuffle.<1, 3, 5, 7>(t3)
		} := Matrix4;
	}

	// get operator[] gives dynamic column access, and returns a reference,
	// so m[2] = v writes into the matrix rather than into a copy.
	get operator[](column: uint32): ref float32x4 {
		switch (column) {
			case 0: return ref this.c0;
			case 1: return ref this.c1;
			case 2: return ref this.c2;
			case 3: return ref this.c3;
		}
		throw new RangeError('column');
	}

	static identity(): Matrix4 {
		return {
			c0: float32x4(1, 0, 0, 0),
			c1: float32x4(0, 1, 0, 0),
			c2: float32x4(0, 0, 1, 0),
			c3: float32x4(0, 0, 0, 1)
		} := Matrix4;
	}
}
```

```Matrix4``` satisfies ```ScalarMultiply.<float32, Matrix4>``` through its scalar overload, so the ```primitive float32``` block declared earlier already gives ```2 * m``` without another line.

An object literal cast with ```:=``` fills the class's layout directly rather than calling its constructor, which is the same rule [serialization](serialization.md) uses for parsing. That is what makes these operators allocation-free: the result is four registers, never an object.

An affine inverse, the common case, is a transposed rotation and a negated translation, and needs no general cofactor expansion:

```js
class Matrix4 {
	// Valid when the matrix is a rotation and a translation with no scale.
	inverseAffine(): Matrix4 {
		const r = { c0: this.c0, c1: this.c1, c2: this.c2, c3: float32x4(0, 0, 0, 1) } := Matrix4;
		const t = r.transpose();
		const translation = this.c3;
		let back = t.c0 * translation.swizzle.<0, 0, 0, 0>();
		back = Math.fma(t.c1, translation.swizzle.<1, 1, 1, 1>(), back);
		back = Math.fma(t.c2, translation.swizzle.<2, 2, 2, 2>(), back);
		return { c0: t.c0, c1: t.c1, c2: t.c2, c3: float32x4(0, 0, 0, 1) - back } := Matrix4;
	}
}
```

## Quaternion

```js
class Quaternion {
	v: float32x4; // x, y, z, w

	// Hamilton product. Each component of a scales a permuted, sign-flipped b,
	// and the four terms accumulate with fused multiply-adds. Branchless.
	operator*(q: Quaternion): Quaternion {
		const a = this.v;
		const b = q.v;
		const b1 = b.swizzle.<3, 2, 1, 0>() * float32x4(1, -1, 1, -1); // (bw, -bz, by, -bx)
		const b2 = b.swizzle.<2, 3, 0, 1>() * float32x4(1, 1, -1, -1); // (bz, bw, -bx, -by)
		const b3 = b.swizzle.<1, 0, 3, 2>() * float32x4(-1, 1, 1, -1); // (-by, bx, bw, -bz)
		let r = a.swizzle.<3, 3, 3, 3>() * b;
		r = Math.fma(a.swizzle.<0, 0, 0, 0>(), b1, r);
		r = Math.fma(a.swizzle.<1, 1, 1, 1>(), b2, r);
		r = Math.fma(a.swizzle.<2, 2, 2, 2>(), b3, r);
		return { v: r } := Quaternion;
	}

	operator*(v: Vector3A): Vector3A {} // Rotate a vector
	operator*(s: float32): Quaternion { return { v: this.v * float32x4(s) } := Quaternion; }
}
```

## Use Cases

**A model-view-projection chain**, the most common expression in graphics:

```js
const mvp = projection * view * model;
const clip = mvp * position;
```

Three matrix multiplies and one transform, all value types, no allocation. The intermediate ```projection * view``` lives in four registers and never reaches memory.

**A transform hierarchy**, where each node composes its parent's world matrix:

```js
class Node {
	local: Matrix4;
	parent: int32; // -1 for a root, and parents always precede children
}

function updateWorlds(nodes: SoA.<Node>, worlds: [].<Matrix4>) {
	for (let i: uint32 = 0; i < nodes.length; ++i) {
		const ref node = nodes[i];
		worlds[i] = node.parent < 0 ? node.local : worlds[uint32(node.parent)] * node.local;
	}
}
```

**Skinning**, a weighted sum of four bone matrices, which is scalar multiplication on the left of a matrix and is exactly the operator the ```primitive float32``` block provides:

```js
function skin(bones: [].<Matrix4>, indices: uint32x4, weights: Vector4, position: Vector4): Vector4 {
	const m = weights.x * bones[indices.lane.<0>()]
		+ weights.y * bones[indices.lane.<1>()]
		+ weights.z * bones[indices.lane.<2>()]
		+ weights.w * bones[indices.lane.<3>()];
	return m * position;
}
```

**Frustum culling** with masks and no branches, four spheres at a time:

```js
function cullSpheres(plane: Vector4, x: float32x4, y: float32x4, z: float32x4, radius: float32x4): boolean32x4 {
	let d = x * float32x4(plane.x);
	d = Math.fma(y, float32x4(plane.y), d);
	d = Math.fma(z, float32x4(plane.z), d);
	d += float32x4(plane.w);
	return d >= -radius; // A mask. mask.any() and mask.all() reduce it
}
```

Note the shape of that last one. The inputs are four parallel lanes of *different objects*, not the components of one vector. That is the layout the [structure of arrays](soa.md) extension produces, and it is four times faster than transforming one ```Vector4``` at a time because no lane is wasted and no shuffle is needed. The ```Matrix4``` code above is the right answer when there is one transform to do; ```SoA``` is the right answer when there are a million.

## What the Engine Must Do

The claim to defend is that ```mvp * position``` compiles to eight instructions. Here is what that depends on, in order of how much of it the proposal already has.

**Value types, which it has.** ```Matrix4``` is four ```float32x4``` fields with a fixed layout, so its instances live in registers. Nothing is allocated, nothing is collected, and there is no header to skip on a field access.

**Natural alignment, which it now has.** ```Matrix4``` inherits sixteen-byte alignment from its members, so loading a column is ```movaps``` rather than ```movups```, and an array of matrices is aligned at every element.

**Compile-time lane indices, which it has.** ```swizzle.<0, 0, 0, 0>()``` puts the permutation in a value generic, so it specializes to an instruction with an immediate operand. Runtime indices cannot compile to ```shufps``` at all.

**No aliasing, which it has.** Value semantics mean two ```Matrix4``` bindings never refer to the same storage, so the engine needs no alias analysis to keep columns in registers across an operator call.

**Guaranteed inlining, which it does not yet have.** Every operator here is a method call. If the engine inlines them, the temporaries are registers and scalar replacement removes the objects entirely. If it doesn't, ```projection * view * model``` is three calls, each returning sixty-four bytes through memory, and the whole exercise fails. Today this is a heuristic. C++ and Rust do not leave it to chance: ```glam``` marks essentially every operator ```#[inline(always)]```, and Eigen's expression templates exist to force the same outcome through the type system. This is the last piece. What it needs is a way to mark a function as inlined at every call site, guaranteed by the language rather than hoped for from the optimizer, which is a general mechanism this proposal does not yet have and which operators are only the sharpest motivation for.

**Multi-register return.** A ```Matrix4``` returned by value is four vector registers. An ABI that returns it through a hidden memory pointer costs a store and a load per operator. This is a codegen requirement rather than a language one, but it is worth writing down, because a naive implementation of value type returns will destroy the performance the value types were introduced to deliver.

Given those, the mapping is direct:

| Expression | x86-64 | AArch64 |
| --- | --- | --- |
| ```a + b``` on ```float32x4``` | ```addps``` | ```fadd v.4s``` |
| ```v.swizzle.<0,0,0,0>()``` | ```shufps``` / ```vbroadcastss``` | ```dup v.4s, v[0]``` |
| ```Math.fma(a, b, c)``` | ```vfmadd213ps``` | ```fmla v.4s``` |
| ```m.select(a, b)``` | ```blendvps``` | ```bsl v.16b``` |
| ```a == b``` to a mask | ```cmpeqps``` | ```fcmeq v.4s``` |
| ```mask.all()``` | ```movmskps``` + ```cmp``` | ```uminv``` + ```fmov``` |
| ```v.sum()``` | ```haddps``` or shuffles + ```addps``` | ```faddp``` |

## Other Languages

| Language | Vector types | Operators | Intrinsics | Inlining | Auto FMA |
| --- | --- | --- | --- | --- | --- |
| C++ | ```__m128```, plus GLM and Eigen | Yes | ```_mm_*``` | ```inline```, and expression templates | Yes, by default |
| Rust | ```core::arch```, ```std::simd```, glam | Yes, via traits | ```_mm_*``` unsafe | ```#[inline(always)]``` | No, ```mul_add``` |
| C# | ```Vector4```, ```Matrix4x4```, ```Vector128<T>``` | Yes | ```Sse.*```, ```AdvSimd.*``` | ```AggressiveInlining``` | No, ```FusedMultiplyAdd``` |
| Zig | ```@Vector(4, f32)```, native operators | No, for user types | ```@shuffle```, ```@reduce``` | ```inline fn``` | ```@mulAdd``` |
| Swift | ```SIMD4<Float>```, ```simd_float4x4``` | Yes | ```simd``` framework | ```@inline(__always)``` | ```addingProduct``` |
| Java | Vector API | No | Species-based methods | JIT heuristic | ```Math.fma``` |
| GLSL, WGSL | ```vec4```, ```mat4``` native | Built in | Built in | Whole-program | Driver-defined |

Two designs work, and they are not the same design. **C#, Zig, and the shading languages make the vector and matrix types known to the compiler**, which then emits instructions directly. C#'s ```Matrix4x4``` is not a library type that happens to be fast; the JIT recognizes it. **C++, Rust, and Swift make the types ordinary and lean on guaranteed inlining**, so that a user-defined ```Matrix4``` is exactly as fast as a built-in one would be.

This proposal is the second kind, and it should stay that way, because the first kind privileges one library's idea of a matrix forever. But the second kind has a prerequisite that the first does not, which is that inlining cannot be a suggestion. Every language in the second group provides a way to demand it. That is the gap.

Two smaller observations from the table. Zig has native vectors and no operator overloading, so its matrix code reads as function calls; the combination here is strictly more expressive. And C++ is the only entry that contracts to FMA automatically, which is why the same C++ source produces different results at different optimization levels, and why every other language on the list decided against it.

## Open Questions

- Component accessors, ```v.xyzw``` and ```v.rgba```, are specified in the [SIMD](simd.md) extension as prototype accessors that desugar to ```swizzle```. They are why a math library aliases its vector types rather than wrapping them.
- Whether engines will special-case a standard ```Vector4``` and ```Matrix4```, as .NET does. They probably will, and the language's job is to make sure the same code written by hand is not slower for it.
- ```Matrix3``` wastes a lane per column. A three-column, three-row packed form is thirty-six bytes and needs a shuffle per operation. Both are useful; only the padded one is fast.
- A general ```inverse()``` needs a cofactor expansion and is numerically delicate. The affine case above covers transforms; the general case belongs in a library rather than in this document.
- Whether ```dot``` should return ```float32``` or a broadcast ```float32x4```. Returning the broadcast avoids a lane extract when the result feeds back into vector math, and return type overloading lets both exist.
- An approximate reciprocal square root. ```rsqrtps``` and ```frsqrte``` are twelve-bit and eight-bit approximations, an order of magnitude faster than an exact reciprocal square root, and normalizing a vector rarely needs the last bits. Exposing one is the only place in this document where the strict floating point rule is tempting to break, and it should be broken explicitly, with a named intrinsic and a specified error bound, or not at all.
