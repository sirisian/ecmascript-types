# Memory Layout

Value types have a defined in-memory layout, which is what lets a `[N].<T>` be a contiguous buffer, a placement `new` land a class on existing bytes, a serializer walk fields by offset, and a GPU vertex descriptor be generated from a type. This document defines that layout: the size and alignment a type reports, the natural-alignment rules that place fields, and the decorators that override them for wire formats, unions, and bit-fields.

The default layout is the C, C++, and Rust rule, so a typed class is layout-compatible with the same declaration in those languages. The explicit-control decorators below are the `#[repr(C)]`, `#[repr(packed)]`, and `#[repr(align)]` of that world, plus the field offsets those languages keep internal.

## Layout properties

Every value type and value type class exposes its layout as three static properties. `byteLength` is the laid-out size in bytes, including any trailing padding required by the type's alignment; `alignment` is the byte alignment of the type; and `bitLength` is the size in bits, which is what an arbitrary-width integer needs to describe itself. A type's alignment is its byte length rounded up to a power of two, so `uint.<4>` sits at alignment one, `uint.<24>` at four, and `uint.<128>` at sixteen. There is no cap. This is the C and Rust rule — Rust aligns each scalar to its own size, and `u128` is sixteen-byte aligned to match C — and raising an alignment beyond it is what `@align` is for, as `#[repr(align(N))]` is there. A SIMD vector needs no separate rule: `float32x4` is sixteen bytes and so aligns to sixteen, which is how the register it occupies is addressed. A typed array's instance `byteLength` is its length times its element's `byteLength`.

```js
uint8.byteLength;    // 1
uint8.bitLength;     // 8
uint.<4>.bitLength;  // 4
uint.<4>.byteLength; // 1, the bits rounded up to a byte
float64.byteLength;  // 8
float64.alignment;   // 8
float32x4.byteLength; // 16
float32x4.alignment;  // 16

class Vertex {
  x: float32;
  y: float32;
  z: float32;
}
Vertex.byteLength; // 12
Vertex.alignment;  // 4

const mesh: [10].<Vertex>;
mesh.byteLength;   // 120

// Slicing a pool's byte view for upload:
Span.<uint8>(mesh).slice(0, count * Vertex.byteLength);
```

These are properties on the type object rather than a `sizeof` operator, so no grammar is added, and they work wherever a type does. A generic reads its own parameter, and dynamic code reads the type of a value:

```js
function stride<T>(): uint32 {
  return T.byteLength; // A constant once T is specialized
}
Reflect.typeOf(value).byteLength; // One property load on an interned type object
```

**They are compile-time constants.** For any type whose layout is known, `byteLength`, `bitLength`, and `alignment` are compile-time evaluable in the sense of the [type objects](typeobjects.md) extension's compile-time type expressions: they constant-fold, never compute anything at run time, and can appear anywhere a constant can, including as an array extent or a value generic argument.

```js
const scratch: [Vertex.byteLength * 1024].<uint8>; // A 12288 byte buffer
const header: [Header.byteLength].<uint8>;
```

## Which types have a layout

| Type | Layout |
| --- | --- |
| Numeric types, `boolean`, SIMD vectors | Yes |
| Enums | Yes, their underlying type's |
| Value type classes | Yes |
| `[N].<T>` and `SoA.<T, N>` | Yes |
| `bigint`, `string`, `any` | No. Their size is a property of the value, not the type. A string reaches a laid-out record as bytes; see [strings at a binary boundary](serialization.md#strings-at-a-binary-boundary), and [fixed-length strings](examples/fixedstring.md) for the record case |
| `T \| null` where `T` is a value type class | Yes, the optional layout below. A reference only where a cycle forces one |
| Reference types, and a nullable union of one | No. A reference's width is the engine's business |
| `[].<T>` without a length | No as a type. Its instances have a `byteLength` |
| A class with an untyped field | No |
| A union of value types | No. It has no single layout |

Reading `byteLength`, `bitLength`, or `alignment` from a type in the second group is a TypeError, which is the point: a program that asks for the size of a `string` has made a mistake a returned number would hide.

Asking WHETHER a type has a layout is a different act from asserting that it does, and `hasLayout` answers it:

```js
uint8.hasLayout;              // true
string.hasLayout;             // false, rather than throwing
Reflect.typeOf(v).hasLayout;  // what a generic serializer actually asks
```

Without it the only way to ask is to read one of the three and catch, which is control flow by exception for a question that is not a mistake - and `'byteLength' in string` is `true`, the property being present and throwing on read, so there is nothing cheaper to test. A serializer walking a heterogeneous structure is asking, not asserting; the `catch` that stands in for it also swallows a typo in the type name. `hasLayout` sits beside the three it guards, is compile-time evaluable for the same reason they are, and leaves them free to keep throwing on the mistake they are for. .NET keeps the same pair - `Marshal.SizeOf` throws where `Type.IsValueType` answers - and Rust answers it in the type system with `Sized` so it never reaches run time. The three properties reflect the *declared* layout, so the offset and endianness decorators below are accounted for.

A field's offset within its class is reflection rather than a property of the type, since it belongs to the field. `Reflect.getReflection.<Reflect.ClassFieldLayout, T>(name)` returns it, compile-time evaluable for the same reason the layout is:

```js
Reflect.getReflection.<Reflect.ClassFieldLayout, Vertex>('y').offset;     // 4
Reflect.getReflection.<Reflect.ClassFieldLayout, Vertex>('y').byteLength; // 4
```

This is the `offsetof` of C and C#, and it is what a serializer, a placement `new`, or a GPU vertex attribute descriptor needs.

What it returns is a **`ClassFieldLayoutReflection`**, and it belongs to this document rather than to the decorators one:

```js
type ClassFieldLayoutReflection = {
	// The name the slot was DECLARED under. An `accessor` reports its own name, not the private field it desugars to: the backing is unnameable, and a slot no program can name leaves a hole in a layout walk - a serializer would see bytes it could not label. This is deliberately not C#'s `<a>k__BackingField`, which leaks a compiler artifact into every reflective enumeration. A genuine `#` private field keeps its own invisibility, since it was never reachable by name to begin with.
	name: string | symbol;
	offset: int32;      // Signed bytes from the start of the instance
	byteLength: uint32;
	bitLength: uint32;
	alignment: uint32;
	offsetBit: uint32;  // Bits from the start of the allocation, which is what fixes bit order exactly
	isBitField: boolean;
};
```

Layout reflection and declaration reflection are deliberately separate, and they are reached by separate contexts: `Reflect.ClassFieldLayout` for the placement above and `Reflect.ClassField` for the declaration. They were both written as `Reflect.ClassField` here, which made one retrieval expression mean two shapes depending on which document a reader had open. `Reflect.ClassFieldLayout` is a reflection context and not a DECORATOR context - nothing decorates a placement, which is the distinction `Reflect.Type` already draws.

`Reflect.ClassField`'s decorator context in decorators.md describes what a field WAS DECLARED as — its type, visibility, and `readonly` — and carries `offset` and `byteLength` because those two are what a decorator commonly wants; the bit-level placement above is meaningful only for a class that has a layout at all, and only this extension defines what it means. Every other language keeps the same seam: .NET has `FieldInfo` for members and `Marshal.OffsetOf` for layout, C has `offsetof` unconnected to anything else, and Rust had no stable field-offset reflection at all until `offset_of!`. Asking a class with no layout for a field's placement is a TypeError, for the same reason reading `byteLength` from a `string` is.

## Optional values

A field of type `T | null` where `T` is a value type class is laid out INLINE: a discriminant followed by `T`'s layout, in the containing class's memory, with no allocation and no indirection. `class B { a: A | null }` over a one-byte `A` is two bytes, and `[1000].<B>` is two kilobytes in one allocation rather than eight kilobytes of pointers addressing a thousand separate objects.

This is what `Option<T>` is in Rust and Swift, `std::optional<T>` in C++, and `Nullable<T>` in C#. All four inline the payload; all four reserve a separate spelling for the case where the payload is reached through a pointer. This document reaches the same place from the semantics the proposal already has, which turn out to leave no room for anything else.

### Why it is inline, and why that is not a change of meaning

The representation was a reference, and the reason the change is safe rather than delicate is that **the box was never observable**. A value type class in a `T | null` slot already behaves as a value in every channel a program can reach:

```js
class A { x: uint8 = 1; }
class B { a: A | null = null; }

const src = new A();
const b = new B();
b.a = src;
src.x = 7;
b.a?.x;              // 1 — the store COPIED; the field does not alias src

const out = b.a;
if (out != null) out.x = 9;
b.a?.x;              // 1 — the read COPIED; the binding does not alias the field

const b1 = new B(); b1.a = new A();
const b2 = new B(); b2.a = new A();
b1 === b2;           // true — structural, so the allocation is not an identity

new WeakRef(b1);     // TypeError — nothing may hold the box weakly
```

Store copies, read copies, equality is structural, and the value cannot be held weakly. Those four are the complete set of channels through which a heap allocation makes itself known, and each is closed. What remains is `byteLength`, the field offsets reflection reports, and the stride of an array — that is, the layout itself, which is what this section defines. A conforming program cannot otherwise distinguish the two representations, so choosing the cheaper one is a layout decision rather than a semantic one.

This is the sense in which `T | null` was never `Option<&T>`. It is `Option<T>`, spelled as though it were the first and paying for the second.

> **A tension in the current text worth resolving explicitly.** The value type copying rules say a store into "a field or an array element of that type" copies, and the recursion rules say a `T | null` field "is a reference". A field declared `A | null` holding an `A` is reached by both sentences, and they do not agree about what it is. The copying rule is the one that decides observable behaviour, and the one an implementation already follows; the recursion rule is describing a layout obligation, not an aliasing one, and the sections below say so in those terms.

### The layout

The discriminant is one byte. It precedes the payload and is placed so the payload lands at `T`'s alignment, which means it occupies trailing padding the containing class already has wherever one exists. The optional's alignment is `T`'s, and its `byteLength` is `T`'s byte length plus the discriminant, rounded up to that alignment.

```js
class A { x: uint8; }
class B { a: A | null; }
B.byteLength;        // 2
B.alignment;         // 1

class V { x: float32; y: float32; }
class C { v: V | null; }
C.byteLength;        // 12 — 1 discriminant, 3 pad, 8 payload
C.alignment;         // 4

const pool: [1000].<B>;
pool.byteLength;     // 2000
```

`null` is the discriminant's zero, which is what makes a typed declaration without an initializer already correct: a nullable union defaults to `null`, and a zero-filled allocation is a run of empty optionals with no fill pass of its own.

The discriminant is a declared byte rather than a spare bit pattern of `T`, EXCEPT where the type's own declaration says a pattern is impossible. Rust reaches a smaller `Option<&T>` by inferring a niche from whatever values the payload happens not to use, and that inference is deliberately not taken here: `byteLength` is a compile-time constant this proposal lets a program compute with, put in an array extent, and assert in a test, so a size that depends on whether some nested field's value range happens to have a hole would let a new enum member resize a struct three declarations away. Rust can afford it because `size_of` is not a layout promise there; here it is.

What is taken is the DECLARED niche, and it needs no new spelling because the type system already has one. A primitive carrying `bounds` metadata excludes values by declaration, checked at every boundary the value crosses, so the excluded pattern is genuinely unreachable rather than merely unused:

```js
type NodeIndex = uint32.<{ bounds: 0..=0xFFFFFFFE }>;
class Node { left: NodeIndex | null; right: NodeIndex | null; }
Node.byteLength;     // 8 — 0xFFFFFFFF is the niche, so neither optional pays a discriminant

class Plain { left: uint32 | null; }
Plain.byteLength;    // 8 — 1 discriminant, 3 pad, 4 payload
```

A field whose type is a REFERENCE (a `reference` class, or a `sealed` or `abstract` one) has the same property for free, `null` being a pattern it cannot hold, so a nullable reference is one pointer rather than a pointer and a tag.

The rule is one rule: a `T | null` costs a discriminant unless `T`'s declaration excludes a pattern, in which case that pattern is the discriminant. It is local, because the exclusion is written in the type; it is stable, because it changes only when someone edits that declaration; and it is enforced, because the boundary check that already refuses an out-of-bounds value is what makes the pattern unreachable. This is Rust's `NonZeroU32` rather than Rust's inference — the declared half, which is also the half Rust guarantees.

The convergence with the index pools of the [bounding volume hierarchy](examples/dbvh.md) example is not an accident. A pool that reserves `0xFFFFFFFF` as an absent-child sentinel has always been declaring a niche; writing it as `bounds` states to the compiler what a `NULL_NODE` constant states only to the reader, and gets a checked `| null` for the same four bytes the convention was already spending.

### Where a reference is still required, and how it is spelled

A cycle has no finite inline layout, so a class that contains itself must be a REFERENCE TYPE, declared with the `reference` modifier:

```js
reference class Node {
  value: uint32;
  next: Node | null;     // A nullable reference: one pointer, and it aliases
}

class Bad {
  value: uint32;
  // next: Bad | null;   // TypeError: Bad contains itself through field "next".
                         // Declare Bad a `reference class`, or break the cycle
                         // through an array or a pool index.
}
```

A `reference` class is held and passed by reference. Its fields alias, `===` on two of its instances compares references, `[10].<Node>` is ten references, and a bare `Node` field is a non-null one. It is the third class kind beside the value type class and the `dynamic` class, and it is what `struct` versus `class` is in C# and Swift: a decision made once at the declaration rather than at each use.

**Reference-ness is declared on the class, not on the field, and that is the load-bearing choice here.** The alternative is a type constructor applied per field — Rust's `Box<T>`, `&T` and `Rc<T>` are all of this shape — and it does not survive contact with this proposal's two commitments. An owned box would have to COPY on store, since there is no move that leaves a source invalid; a linked list built from copying boxes copies its whole tail at every link, which is O(n) for an operation that must be O(1) and produces a tree of copies rather than a list. A box that instead ALIASES has acquired an identity, and it would be handing one to a value whose `===` the language has defined as structural, so two holders of "the same" payload would compare equal to two holders of equal copies while behaving differently. Neither is available. Rust can offer the use-site choice precisely because it has no collector and ownership must therefore be written down; a language whose collector owns lifetime has exactly one kind of reference, so the only question left is whether a given class is reached through one, which is a property of the class.

`sealed` and `abstract` classes are reference types already, for their own reasons, so they close a cycle without the modifier — which is what lets the [expression parser](examples/expressionparser.md) hold its children as plain `Node`. `reference` is what an OPEN, non-abstract class needs, and the gap is real independently of recursion: a typed class meant to be subclassed by consumers is a value type class today, so a field declared with its type embeds only the base slice and a subclass instance assigned to one silently loses everything the subclass added. Such a class has always needed to be a reference type and has had no way to say so except by sealing itself, which refuses the extension it exists to offer.

### What this settles elsewhere

- `[10].<A | null>` is ten inline optionals rather than ten references, so the contrast the value type class section draws between `[10].<A>` and `[10].<A | null>` becomes a contrast between ten values and ten optional values, both contiguous, rather than between values and pointers.
- `uint8 | null` and every other nullable scalar acquires the layout it did not have. The rule is one rule: a discriminant and a payload, whatever the payload is.
- A class holding an optional value type class is itself a value type class with a layout, which it already was, now for a stated reason rather than by inference from the reference width.

### Alternatives considered

**A new spelling for the inline form, leaving `T | null` a reference.** `Option.<T>` or a suffix, added beside the existing meaning, with nothing existing changing. Rejected on the proposal's own principle that the natural spelling should be the fast spelling: the shape a reader reaches for first would keep allocating, and the cheap form would be the one you had to know to ask for. It also has to answer what `Option.<T> | null` means, and it doubles the vocabulary for a distinction that, given the copying semantics above, no program can observe.

**Inferring the representation per field, boxing only to break cycles.** An inference that inlined where it could and indirected where it could not has to choose an edge in a mutual cycle `A → B → C → A`, and any choice makes one of those three layouts a function of declaration order or of the order a checker walks the graph. It also makes the choice invisible: a program that adds a field completing a cycle would find a hot structure quietly grow, with nothing at the edit site to read. Rust and C++ both refuse the recursive declaration and make the author say what the indirection is.

**A per-field indirection type, `Box.<T>`.** Ruled out above under the reference modifier: copying makes linking O(n), aliasing gives an identity to a structurally-compared value, and moving needs affine types the proposal has refused.

**Inferred niches, as Rust performs them.** Rejected for making `byteLength` non-local. The declared half is taken instead.

### Open

- **`A | B | null`, a union of more than one value type class, still has no single layout and is still a reference.** The optional is the two-variant case of an inline tagged union, so the layout machinery now exists and the generalization is mostly a question of what `===` compares and how a variant narrows, not of how bytes are placed. The interesting version is probably not the anonymous union at all but a `sealed abstract` hierarchy laid out inline where every subclass is a value type, which would give Rust's `enum` with the exhaustiveness checking this proposal already reserves to sealed hierarchies. Deferred as a feature of its own rather than a corollary of this one.
- **A variant of an inline sum type sets the size of every holder**, so one large case makes every value of the type large. This is inherent — Rust has it and answers it by indirecting the large variant — and it is the question the item above has to answer before it lands.
- **`bounds` metadata is specified but not yet implemented**, so the declared niche above is specified against a mechanism that does not run. The two should land together, since a niche that silently does not apply is a size regression nobody sees.

## Natural alignment and padding

Members are naturally aligned. Each member is placed at the next offset that is a multiple of its own alignment, a class's alignment is the largest alignment among its members, and its `byteLength` is rounded up to that alignment so that every element of an array of the class is aligned too.

```js
class A {
  a: uint8;  // Offset 0
  b: uint16; // Offset 2, not 1: uint16 is 2-byte aligned
}
A.byteLength; // 4, padded from 3 so that b stays aligned in an array
A.alignment;  // 2
const a: [10].<A>; // 40 bytes
```

Because layout follows declaration order, field order is a performance decision: `{ a: uint8, b: float64, c: uint8 }` occupies 24 bytes where `{ b: float64, a: uint8, c: uint8 }` occupies 16, since the second groups the small fields into one alignment gap. Ordering fields from largest alignment to smallest minimizes padding. The proposal does not reorder for you — views, serialization, and interop depend on the declared order — so this is guidance rather than a guarantee, and because `byteLength` is a compile-time constant a test can assert the size a class was meant to have.

## Inheritance

By default the layout of a typed class — one where every property is typed — appends to the memory of the extended class:

```js
class A {
  a: uint8;
}
class B extends A {
  b: uint8;
}
// The layout is the same as:
class AB {
  a: uint8;
  b: uint8;
}
```

The base's fields keep their offsets, and the subclass's fields follow, aligned by the same natural-alignment rule. Re-declaring an inherited field is a TypeError, since it would have no defined offset.

## Packing

A `@packed` class decorator removes the padding, placing each member immediately after the previous one and giving the class an alignment of `1`. Members may then be unaligned, which costs a little on every access and is exactly what a wire format wants in exchange for exact byte offsets. `@packed` decides member offsets; `@alignAll` still decides the alignment of the instance as a whole, so the two compose.

```js
@packed
class A {
  a: uint8;  // Offset 0
  b: uint16; // Offset 1
}
A.byteLength; // 3
A.alignment;  // 1
const a: [10].<A>; // 30 bytes
```

## Explicit offset and alignment

Two property-descriptor keys, `align` and `offset`, control a member's placement, and for consistency between codebases two reserved decorators, `@align` and `@offset`, set them with byte values.

`@align(n)` requires the member's address to be a multiple of `n`, in either direction — a member can be given a stricter alignment than its type requires, useful for cache-line boundaries and specialized move instructions. `@offset(n)` places the member `n` bytes from the start of the class allocation; the offset origin is `0` for each class, so in a subclass a positive offset lands after the base and a negative offset reaches back into it (see Unions, below).

Two object-level reserved descriptor keys, `alignAll` and `size`, control the whole instance: `@alignAll(n)` sets the instance's memory alignment, and `@size(n)` fixes the instance's allocated size, padding with zeros.

```js
@alignAll(16) // The instance is 16-byte aligned
@size(32)     // The instance is 32 bytes, zero-padded
class A {
  @offset(2)
  x: float32;   // At byte 2, within a 16-byte-aligned instance
  @align(4)
  y: float32x4; // 2 + 4 = 6, rounded up to the required 4-byte multiple: byte 8
}
```

`@align` and `@offset` apply only when every property in the class is typed and the complete prototype chain is as well; a class with an untyped field has no fixed layout to place fields within.

## Endianness

A third reserved decorator, `@endian('little')` / `@endian('big')` with the descriptor key `endian`, fixes the byte order of a multi-byte member for parsing wire formats. By default members use platform byte order, matching `TypedArray`s. `@endian` on a member overrides that member's order, so a struct can mix native fields with big-endian network fields in one declaration:

```js
@packed
class PacketHeader {
  @endian('big') length: uint32; // Network byte order
  @endian('big') checksum: uint16;
  flags: uint8;                  // Platform order
}
```

## Unions via overlapping offsets

Because `@offset` places a member explicitly, two members can be given the same offset, which maps them to the same memory — a union. A negative offset reaches into a base class's memory to overlap it:

```js
class A {
  a: uint8;
}
class B extends A {
  @offset(-1)
  b: uint8;
}
// The layout is the same as:
class AB { // 1 byte
  a: uint8;
  @offset(0)
  b: uint8;
}
const ab = new AB();
ab.a = 10;
ab.b == 10; // true
```

A tagged union — a value that is one of several layouts distinguished by a discriminant — is better expressed as a `sealed abstract class` hierarchy, which the language checks for exhaustiveness. Offset-overlap unions are for the C-style case where a program deliberately reinterprets the same bytes, such as reading a `float32`'s bits as a `uint32`.

## Bit-fields

Integer members narrower than a byte — `uint.<N>` or `int.<N>` with `N` under `8` — pack into shared bytes rather than each rounding up to a byte, the way a C bit-field does. `@offsetBit(n)` places a member `n` bits from the start of the allocation, defining the bit order explicitly so a wire format is exact; members without `@offsetBit` pack consecutively from the current bit position. Automatic packing stops at the byte: a member 8 bits or wider never packs, so a `uint.<12>` occupies two bytes at its alignment unless `@offsetBit` places it, which is what a 12-bit wire format has anyway. The byte boundary rather than `bitLength` is the line, and explicit placement is the tool past it.

```js
@packed
class RGB565 {
  r: uint.<5>; // Bits 0..<5
  g: uint.<6>; // Bits 5..<11
  b: uint.<5>; // Bits 11..<16
}
RGB565.byteLength; // 2
RGB565.bitLength;  // 16
```

`bitLength` reports the packed bit size, and `byteLength` rounds it up to the enclosing byte. Bit-fields are distinct from the SIMD bit-vector types (`boolean8` and its family): those are lane masks with one bit per lane, defined in the [SIMD](simd.md) extension, whereas a bit-field is a sub-byte integer member of a class. Reading or writing a bit-field is a shift and mask the compiler emits; taking a reference to one is a TypeError, since it is not a byte-addressable location.

## Adding fields at run time

The descriptor features apply only to fully-typed classes with a typed prototype chain. A `dynamic` class, being unsealed, can gain a field later through `Object.defineProperty` with a `type` key, but that field is not part of any fixed layout, and reflecting a layout property on a `dynamic` class is a TypeError. On a non-`dynamic` class, `Object.defineProperty` adding a typed field is itself a TypeError, since the layout is closed at declaration.

```js
class A {
  a: uint8;
  constructor(a: uint8) {
    this.a = a;
  }
}
const a: [].<A> = [0, 1, 2];
// Object.defineProperty(A, 'b', { value: 0, writable: true, type: uint8 });
// TypeError: A is sealed; its layout is fixed at declaration
```
