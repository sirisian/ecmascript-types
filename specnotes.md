# Required Changes to the Specification:

### 6.1 ECMAScript Language Types

Would need to include the types listed above. Probably in a more verbose view than a list.

### 6.1.3 The Boolean Type

```boolean``` is an alias for ```Boolean``` when ```boolean``` is imported.

### 6.1.4 The String Type

```string``` is an alias for ```String``` when ```string``` is imported.

### 6.1.5 The Symbol Type

```symbol``` is an alias for ```Symbol``` when ```symbol``` is imported.

### 6.1.6 The Number Type

```number``` is an alias for ```Number``` when ```number``` is imported.

### 6.1.7 The Object Type

```object``` is an alias for ```Object``` when ```object``` is imported.

### 6.1.8 Integral Types

#### 6.1.8.1 Signed

```int8```, ```int16```, ```int32```, ```int64```

```int8.parse(string, radix = 10)```

#### 6.1.8.2 Unsigned

```uint8```, ```uint16```, ```uint32```, ```uint64```

```uint8.parse(string, radix = 10)```

### 6.1.9 Big Integer

```bigint```

```bigint.parse(string, radix = 10)```

### 6.1.10 Float

```float16```, ```float32```, ```float64```, ```float80```, ```float128```

TODO: Requirements in the spec? Is referring to the specs for each sufficient?

https://en.wikipedia.org/wiki/Half-precision_floating-point_format  
https://en.wikipedia.org/wiki/Single-precision_floating-point_format  
https://en.wikipedia.org/wiki/Double-precision_floating-point_format  
https://en.wikipedia.org/wiki/Quadruple-precision_floating-point_format

### 6.1.11 Decimal

```decimal32```, ```decimal64```, ```decimal128```

https://en.wikipedia.org/wiki/Decimal32_floating-point_format  
https://en.wikipedia.org/wiki/Decimal64_floating-point_format  
https://en.wikipedia.org/wiki/Decimal128_floating-point_format

### 6.1.12 SIMD

```boolean8x16```, ```boolean16x8```, ```boolean32x4```, ```boolean64x2```, ```boolean8x32```, ```boolean16x16```, ```boolean32x8```, ```boolean64x4```  
```int8x16```, ```int16x8```, ```int32x4```, ```int64x2```, ```int8x32```, ```int16x16```, ```int32x8```, ```int64x4```  
```uint8x16```, ```uint16x8```, ```uint32x4```, ```uint64x2```, ```uint8x32```, ```uint16x16```, ```uint32x8```, ```uint64x4```  
```float32x4```, ```float64x2```, ```float32x8```, ```float64x4```  

### 6.1.13 Rational

```rational```

https://en.wikipedia.org/wiki/Rational_data_type

### 6.1.14 Complex

```complex```

https://en.wikipedia.org/wiki/Complex_data_type

### 6.1.15 Any

```any```

### 6.1.16 Void

```void```

Used solely in function signatures to denote that a return value does not exist.


### 7.2.12 SameValueNonNumber ( x, y )

TODO:...

The internal comparison abstract operation SameValueNonNumber(x, y), where neither x nor y are Number values, produces true or false. Such a comparison is performed as follows:

    Assert: Type(x) is not Number.
    Assert: Type(x) is the same as Type(y).
    If Type(x) is Undefined, return true.
    If Type(x) is Null, return true.
    If Type(x) is String, then
        If x and y are exactly the same sequence of code units (same length and same code units at corresponding indices), return true; otherwise, return false.
    If Type(x) is Boolean, then
        If x and y are both true or both false, return true; otherwise, return false.
    If Type(x) is Symbol, then
        If x and y are both the same Symbol value, return true; otherwise, return false.
    If x and y are the same Object value, return true. Otherwise, return false. 



### 7.2.14 Abstract Equality Comparison

The comparison x == y, where x and y are values, produces true or false. Such a comparison is performed as follows:

1. If Type(x) is the same as Type(y), then
    a. Return the result of performing Strict Equality Comparison x === y.
2. If Type(x) has an implicit cast to Type(y), then
    a. 
3. 
2. If x is null and y is undefined, return true.
3. If x is undefined and y is null, return true.
4. If Type(x) is Number and Type(y) is String, return the result of the comparison x == ! ToNumber(y).
5. If Type(x) is String and Type(y) is Number, return the result of the comparison ! ToNumber(x) == y.
6. If Type(x) is Boolean, return the result of the comparison ! ToNumber(x) == y.
7. If Type(y) is Boolean, return the result of the comparison x == ! ToNumber(y).
8. If Type(x) is either String, Number, or Symbol and Type(y) is Object, return the result of the comparison x == ToPrimitive(y).
9. If Type(x) is Object and Type(y) is either String, Number, or Symbol, return the result of the comparison ToPrimitive(x) == y.
10. Return false. 



### 11.6.2.1

Move enum from 11.6.2.2 to 11.6.2.1.

### 11.8.3.1 Static Semantics: MV

This needs to be changed to work with more than Number. Ideally this operation is delayed until the type is determined. As an example the following should be intuitively legal without any Number conversion done.

```js
let a:uint64 = 0xffffffffffffffff;
```

The same could be true for bigint support.

### 12.5.6 The typeof Operator

The table needs to be updated with all the new types and nullable type explanation.

### 12.10 Relational Operators


### 12.11 Equality Operators

Not sure if something needs to be changed in these.

### 12.5.7 to 12.11

Theese contain the operator definitions. Would probably need to include at least a brief change to explain the behavior of all the types. SIMD ones would require the most explanation.

### 12.14.5 Destructuring Assignment

Type casting syntax described above would need to be included.

### 13.3.2 Variable Statement

Would need to cover the optional typing syntax and grammar.

### 13.15 The try Statement

CatchParameter needs to be modified to allow type constraints.

### 14 ECMAScript Language: Functions and Classes

Function overloading would need to be included and the expected operations for matching a list of arguments to parameters and types. This will also need to cover cases like ambiguous function overloading.

### 14.2 Arrow Function Definitions

The grammar rule ArrowFunction needs an optional return type and ArrowParameters needs optional type information per each parameter.

### 14.3 Method Definitions

Grammar requires typing information as defined above. Specifically MethodDefinition's first rule and the get and set ones.

### 14.5 Class Definitions

Grammar requires typing information for members and methods. Specifically ClassElement and MethodDefinition. ConstructorMethod is referenced as being a MethodDefinition so it should be fine after the MethodDefinition changes.

### 19.2 Function Objects

Needs to support type information in the constructor. Examples from the documentation with typed examples:

```js
new Function("a:string", "b:uint8", "c:int32", "return a + b + c;")
new Function("a:string, b:uint8, c:int32", "return a + b + c;") 
new Function("a:string,b:uint8", "c:int32", "return a + b + c;") 
```

Syntax to define a return type:

```js
new Function("a:string", "b:uint8[]", "c:int32", ":string", "return a + b + c;")
```

### New Sections for Each Type

As described before each type needs a parse function to turn a string into the type. Also for float and decimal .EPSILON needs defined in their section. Also for all the integer (not bigint), float, and decimal types need MIN_VALUE and MAX_VALUE defined in their section.

### 20.2 The Math Object

All the math operations need to be overloaded to work with the integer, float, and decimal types. Meaning if they take in the type they should return the same type.

### 25.2.2 Properties of the GeneratorFunction Constructor

Similar to Function the constructor needs to be changed to allow types. For example:

```js
new GeneratorFunction("a:float32", ":float32", "yield a * 2;");
```
