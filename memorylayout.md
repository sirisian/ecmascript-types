# Memory Layout

Value types have a defined in-memory layout, which is what lets a `[N].<T>` be a contiguous buffer, a placement `new` land a class on existing bytes, a serializer walk fields by offset, and a GPU vertex descriptor be generated from a type. This document defines that layout: the size and alignment a type reports, the natural-alignment rules that place fields, and the decorators that override them for wire formats, unions, and bit-fields.

The default layout is the C, C++, and Rust rule, so a typed class is layout-compatible with the same declaration in those languages. The explicit-control decorators below are the `#[repr(C)]`, `#[repr(packed)]`, and `#[repr(align)]` of that world, plus the field offsets those languages keep internal.

## Layout properties

Every value type and value type class exposes its layout as three static properties. `byteLength` is the laid-out size in bytes, including any trailing padding required by the type's alignment; `alignment` is the byte alignment of the type; and `bitLength` is the size in bits, which is what an arbitrary-width integer needs to describe itself. A width no named type has aligns to the smallest power of two at least its byte length, capped at eight, so `uint.<4>` sits at alignment one and `uint.<24>` at four. A typed array's instance `byteLength` is its length times its element's `byteLength`.

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
[].<uint8>(mesh).slice(0, count * Vertex.byteLength);
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
| `bigint`, `string`, `any` | No. Their size is a property of the value, not the type |
| Reference types, including a nullable union of a value type class | No. A reference's width is the engine's business |
| `[].<T>` without a length | No as a type. Its instances have a `byteLength` |
| A class with an untyped field | No |
| A union of value types | No. It has no single layout |

Reading `byteLength`, `bitLength`, or `alignment` from a type in the second group is a TypeError, which is the point: a program that asks for the size of a `string` has made a mistake a returned number would hide. The three properties reflect the *declared* layout, so the offset and endianness decorators below are accounted for.

A field's offset within its class is reflection rather than a property of the type, since it belongs to the field. `Reflect.getReflection.<Reflect.ClassField, T>(name)` returns an `offset`, compile-time evaluable for the same reason the layout is:

```js
Reflect.getReflection.<Reflect.ClassField, Vertex>('y').offset;     // 4
Reflect.getReflection.<Reflect.ClassField, Vertex>('y').byteLength; // 4
```

This is the `offsetof` of C and C#, and it is what a serializer, a placement `new`, or a GPU vertex attribute descriptor needs.

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
  r: uint.<5>; // Bits 0..5
  g: uint.<6>; // Bits 5..11
  b: uint.<5>; // Bits 11..16
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
