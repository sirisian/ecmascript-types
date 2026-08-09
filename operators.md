# Operators by Type

What each operator yields, and of what type, family by family. The rule is one sentence and the rest of this document is its consequences:

**An operator on operands of one type yields that type.** It never widens to a larger one, never narrows to a smaller one, and never promotes to ```number```.

Comparisons are the exception, and for a reason rather than by exemption: a comparison answers a question instead of computing a quantity, so it has no result type to preserve and yields a ```boolean```. A comparison between vectors is the exception to *that*, because a lane-wise comparison has more than one useful shape; see [SIMD](simd.md).

This document covers the operators the language defines on its own types. Operators a class declares for itself are the subject of [operator overloading](operatoroverloading.md).

## Integer

```uint8``` through ```uint128```, ```int8``` through ```int128```, and the widths of ```uint.<N>``` and ```int.<N>```.

Every operator is defined: ```+ - * / % **```, the shifts ```<< >> >>>```, the bitwise ```& | ^ ~```, unary ```-```, and the comparisons.

```js
const a: uint8 = 7, b: uint8 = 2;
a / b;              // 3      - integer division truncates toward zero
a - (3 := uint8);   // 254    - arithmetic wraps at the width, it does not saturate
a * b;              // uint8  - never uint16, however large the product
a < b;              // boolean
```

Two behaviours are worth stating because a type's name does not imply them. **Division truncates**, so ```7 / 2``` is 3 and not 3.5: the result type is an integer type, and an integer type has no 3.5 to hold. **Arithmetic wraps**, so ```1 - 3``` at ```uint8``` is 254. The forms that trap or clamp instead are ```checked``` and ```saturating```, described in the main [README](README.md).

A one-bit integer is an integer, so ```boolean1``` arithmetic wraps at one bit and ```1 + 1``` is 0. The name suggests a Boolean; the family is integer.

Instructions: ```add```/```sub```/```imul``` and ```shl```/```and``` on x86-64, the same on AArch64. A width narrower than the register wraps with a mask the compiler emits; a ```uint128``` is a pair of registers and an add-with-carry.

## Binary floating-point

```float16```, ```float32```, ```float64```, ```float128```.

Defined: ```+ - * / % **```, unary ```-```, and the comparisons. **The shifts and bitwise operators are not defined**, since each would have to convert the operand to an integer type first, and the conversion is the part a reader would not see. Write it: ```(x := uint32) & mask```.

```js
const a: float32 = 7, b: float32 = 2;
a / b;              // 3.5    - float division does not truncate
a << b;             // TypeError: this operator is not defined for a binary floating-point type
```

Instructions: ```addss```/```mulss```/```divss``` and ```sqrtss``` on x86-64, ```fadd```/```fmul```/```fdiv```/```fsqrt``` on AArch64. ```float16``` is a native type on AArch64 and needs ```F16C``` or ```AVX-512FP16``` on x86-64.

## Decimal floating-point

```decimal32```, ```decimal64```, ```decimal128``` - see [decimal numbers](decimal.md).

Defined: the arithmetic operators, unary ```-```, and the comparisons, each rounding by IEEE 754-2008 decimal rules. The shifts and bitwise operators are not defined, for the reason they are not defined on a binary float.

Instructions: none on commodity hardware. A decimal operation is a library call, on POWER a hardware instruction, and the difference is why a program that does not need decimal rounding should not pay for it.

## Vector

Every SIMD type - ```float32x4```, ```int32x4```, ```uint8x16```, and the rest, and ```vector.<T, N>``` written generically.

Every operator its lane type defines, applied lane-wise, yielding a vector of the same shape: ```+ - * / %```, unary ```-```, and the shifts and bitwise operators where the lane type is an integer type. ```Math``` applies lane-wise too.

```js
int32x4(1, 2, 3, 4) + int32x4(1, 1, 1, 1);   // int32x4, lane-wise    paddd / add v.4s
Math.sqrt(float32x4(1, 4, 9, 16));           // (1, 2, 3, 4)          sqrtps / fsqrt v.4s
-float32x4(1, 2, 3, 4);                      // negate each lane      xorps sign bit / fneg
```

A comparison is overloaded on its result type, because the hardware produces more than one shape and a program wants each: a wide mask, a compact mask, or the compared vector type with all-ones lanes. That is Intel's ```_mm_cmpeq_epi32``` and ```_mm_cmpeq_epi32_mask``` folded into one operator, and it is why a vector comparison left unannotated is an error - the result type is what picks the form. The predicate each operator means, and what a NaN lane does, are in [SIMD](simd.md).

```mask.select(a + b, src)``` is how some lanes are computed and the rest left alone, which is the write-masked form of an instruction on a target that has one. See [SIMD](simd.md) for the masking model and [operator overloading](operatoroverloading.md) for ```swizzle```, ```shuffle```, ```lane```, ```Math.fma```, and the rest of the intrinsic surface.

## bigint

Defined: the arithmetic operators, the shifts and bitwise operators, unary ```-```, and the comparisons. A ```bigint``` does not combine with a ```number``` or with a sized integer type; ```bigint(a) + 1n``` converts explicitly.

Instructions: none. A ```bigint``` is a heap-allocated magnitude and every operation is a library call, which is the cost a fixed-width integer type exists to avoid.

## string

```+``` concatenates and yields a ```string```. The relational operators compare code unit sequences and yield a ```boolean```. The remaining operators apply the existing coercion, so ```'a' * 'b'``` is ```NaN``` exactly as it is today.

## enum

An enum member computes at the enum's underlying type and yields that type, so a member of an enum over ```number``` adds to a ```number``` and a member of an enum over ```uint8``` wraps at eight bits. Bitwise operators work, which is what a flags enum is for.

## Ranges and refined types

A type refined by [primitive metadata](primitivemetadata.md), such as ```uint8.<{ bounds: 1..=6 }>```, is its underlying type for every operator: the refinement constrains what may be stored, not what an operator does. The sum of two ```uint8.<{ bounds: 1..=6 }>``` values is a ```uint8```, and whether it is back in range is a question for the next assignment. A range is written under the key that says what it means, as [ranges](ranges.md) requires; there is no bare ```uint8.<1..=6>```.

## References

An operand read through a reference is read through to its referent, so a ```ref``` to a ```uint8``` adds like a ```uint8```. Assignment through a reference writes the referent. See [references and borrowing](references.md).

## Classes

A class defines its own operators, and the result type is the return type it declares - see [operator overloading](operatoroverloading.md). A class that declares none behaves as an ordinary object does today, so ```new D() + new D()``` is string concatenation rather than an error.

## Summary

| Family | Arithmetic | Shifts and bitwise | Comparison | Notes |
| --- | --- | --- | --- | --- |
| integer | operand type, wrapping | operand type | ```boolean``` | ```/``` truncates |
| binary float | operand type | **not defined** | ```boolean``` | |
| decimal | operand type | **not defined** | ```boolean``` | library call |
| rational | operand type; no ```%``` | not defined | ```boolean``` | ```**``` for integer exponents |
| complex | operand type; no ```%``` | not defined | equality only | not ordered |
| vector | operand type, lane-wise | operand type, integer lanes | three forms | overloaded on return type |
| ```bigint``` | ```bigint``` | ```bigint``` | ```boolean``` | no mixing with ```number``` |
| ```string``` | ```+``` concatenates | existing coercion | ```boolean``` | |
| enum | underlying type | underlying type | ```boolean``` | |
| refined | underlying type | underlying type | ```boolean``` | refinement is not an operator rule |
| reference | referent's type | referent's type | referent's type | |
| class | as declared | as declared | as declared | [operator overloading](operatoroverloading.md) |

The normative statement of this table is in the specification, under *The Result of an Operator*.
