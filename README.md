# ECMAScript Proposal: Optional Static Typing

Current status of this proposal is -1. It's in a theoretical state at the moment to better understand how types could function in Javascript and the long-term future benefits or complications they could cause to future proposals.

## Rationale

With TypedArrays and classes finalized, ECMAScript is in a good place to finally discuss types again. The demand for types as a different approach to code has been so strong in the past few years that separate languages have been created to deal with the perceived shortcomings. Types won't be an easy discussion, nor an easy addition, since they touch a large amount of the language; however, they are something that needs rigorous discussion.

The types described below bring ECMAScript in line or surpasses the type systems in most languages. For developers it cleans up a lot of the syntax, as described later, for TypedArrays, SIMD, and working with number types (floats vs signed and unsigned integers). It also allows for new language features like function overloading and a clean syntax for operator overloading. For implementors, added types offer a way to better optimize the JIT when specific types are used. For languages built on top of Javascript this allows more explicit type usage and closer matching to hardware.

### Types Proposed

Since it would be potentially years before this would be implemented this proposal includes a new keyword ```enum``` for enumerated types and the following types:

```number```  
```bool```  
```string```  
```object```  
```symbol```  
```int8```, ```int16```, ```int32```, ```int64```  
```uint8```, ```uint16```, ```uint32```, ```uint64```  
```bigint```  
```float16```, ```float32```, ```float64```, ```float80```, ```float128```  
```decimal32```, ```decimal64```, ```decimal128```  
```bool8x16```, ```bool16x8```, ```bool32x4```, ```bool64x2```, ```bool8x32```, ```bool16x16```, ```bool32x8```, ```bool64x4```  
```int8x16```, ```int16x8```, ```int32x4```, ```int64x2```, ```int8x32```, ```int16x16```, ```int32x8```, ```int64x4```  
```uint8x16```, ```uint16x8```, ```uint32x4```, ```uint64x2```, ```uint8x32```, ```uint16x16```, ```uint32x8```, ```uint64x4```  
```float32x4```, ```float64x2```, ```float32x8```, ```float64x4```  
```rational```  
```complex```  
```any```  
```void```  

These types once imported behave like a ```const``` declaration and cannot be reassigned.

### Variable Declaration With Type

This syntax is taken from ActionScript and other proposals over the years. It's subjectively concise, readable, and consistent throughout the proposal.

```js
var foo:Type = value;
let foo:Type = value;
const foo:Type = value;
```

### typeof Operator

One of the first complications with types is ```typeof```'s behavior. All of the above types would return their string conversion including ```bool```. (I've spoken to many people now and "boolean" is seen as verbose among C++ and C# developers. Breaking this part of Java's influence probably wouldn't hurt to preserve consistency for the future).

```js
let foo:uint8 = 0; // typeof foo == "uint8"
let bar:uint8? = 0; // typeof bar == "uint8?"
let baz:uint8[] = []; // typeof baz == "object"
let foobar:(uint8):uint8 = x => x * x; // typeof foobar == "function"
```

### instanceof Operator

THIS SECTION IS A WIP

```js
if (value instanceof uint8) {}
```

Also this would be nice for function signatures.

```js
if (value instanceof (uint8):uint8) {}
```

That would imply ```Object.getPrototypeOf(value) === ((uint8):uint8).prototype```.

I'm not well versed on if this makes sense though, but it would be like each typed function has a prototype defined by the signature.

### Nullable Types

All types except ```any``` are non-nullable. The syntax below creates a nullable ```uint8``` typed variable:
```js
let foo:uint8? = null; // typeof foo == "uint8?"
```

### any Type

Using ```any?``` would result in a syntax error since ```any``` already includes nullable types. As would using ```any[]``` since it already includes array types. Using just ```[]``` would be the type for arrays that can contain anything. For example:

```js
let foo:[];
```

### Variable-length Typed Arrays

```js
let foo:uint8[]; // []
foo.push(0); // [0]
let bar:uint8[] = [0, 1, 2, 3];
let baz:uint8[]?; // null
```

The index operator doesn't perform casting just to be clear so array objects even when typed still behave like objects.

```js
let foo:uint8[] = [0, 1, 2, 3];
foo['a'] = 'foobar';
'a' in foo; // true
delete foo['a'];
```

### Fixed-length Typed Arrays

```js
let foo:uint8[4]; // [0, 0, 0, 0]
// foo.push(0); TypeError: foo is fixed-length
// foo.pop(); TypeError: foo is fixed-length
foo[0] = 1; // valid
// foo[foo.length] = 2; Out of range
let bar:uint8[4] = [0, 1, 2, 3];
let baz:uint8[4]?; // null
```

Typed arrays would be zero-ed at creation. That is the allocated memory would be set to all zeroes.

### Mixing Variable-length and Fixed-length Arrays

```js
function Foo(p:boolean):uint8[] // default case, return a resizable array
{
    let foo:uint8[4] = [0, 1, 2, 3];
    let bar:uint8[6] = [0, 1, 2, 3, 4, 5];
    return p ? foo : bar;
}

function Foo(p:boolean):uint8[6] // return a resized foo
{
    let foo:uint8[4] = [0, 1, 2, 3];
    let bar:uint8[6] = [0, 1, 2, 3, 4, 5];
    return p ? foo : bar;
}
```

### Any Typed Array

```js
let foo:[]; // Using any[] is a syntax error as explained before
let foo:[]? = null; // nullable array
```

Deleting a typed array element results in a type error:

```js
const bar:uint8[] = [0, 1, 2, 3];
// delete bar[0]; TypeError
```

### Array length Type And Operations

Valid types for defining the length of an array are as follows:

```
int8/16/32/64  
uint8/16/32/64
```

By default ```length``` is ```uint32```.

Syntax:

```js
let foo:uint8[int8] = [0, 1, 2, 3, 4];
let bar = foo.length; // length is type int8
```

```js
let foo:uint8[5:uint64] = [0, 1, 2, 3, 4];
let bar = foo.length; // length is type uint64 with value 5
```

```js
let n = 5;
let foo:uint8[n:uint64] = [0, 1, 2, 3, 4];
let bar = foo.length; // length is type uint64 with value 5
```

Setting the ```length``` reallocates the array truncating when applicable.

```js
let foo:uint8[] = [0, 1, 2, 3, 4];
foo.length = 4; // [0, 1, 2, 4]
foo.length = 6; // [0, 1, 2, 4, 0, 0]
```

```js
let foo:uint8[5] = [0, 1, 2, 3, 4];
// foo.length = 4; TypeError foo is fixed-length
```

### Array views:

Like ```TypedArray``` views this array syntax allows any array, even arrays of typed objects to be viewed as different objects. Stride would have performance implications but would allow for a single view to access elements with padding between elements.

```js
let view = Type[](buffer [, byteOffset [, byteLength [, byteStride]]]);
```

```js
let foo:uint64[] = [1];
let bar = uint32[](foo, 0, 8);
```

Take special note of the lack of ```new``` in this syntax. Adding new in the above case would pass the arguments to the constructor for each element.

### Multidimensional and Jagged Array Support Via User-defined Index Operators

Rather than definining index functions for various multidimensional and jagged array implementations the user is given the ability to define their own. Any lambda parameter passed to the "index constructor" creates an indexing function. More than one can be defined as long as they have unique signatures. The signature ```(x:string)``` is reserved for keys and can't be used.

An example of a user-defined index to access a 16 element grid with ```(x, y)``` coordinates:

```js
let grid = new uint8[16:uint32, (x:uint32, y:uint32) => y * 4 + x];
// grid[0] = 10; Error, invalid arguments
grid[2, 1] = 10;
```

```js
let grid = new uint8[16:uint32, i => i, (x:uint32, y:uint32) => y * 4 + x];
grid[0] = 10;
grid[2, 1] = 10;
```

For a variable-length array it works as expected where the user drops the length:

```js
var grid = new uint8[(x, y, z) => z * 4 * 4 + y * 4 + x];
var grid2 = new uint8[uint64, (x, y, z) => z * 4 * 4 + y * 4 + x];
```

Views also work as expected allowing one to apply custom indexing to existing arrays:

```js
var gridView = new uint32[(x, y, z) => z * 4 * 4 + y * 4 + x](grid);
```

### Implicit Casting

The default numeric type Number would convert implicitly with precedence given to ```decimal128/64/32```, ```float128/80/64/32/16```, ```uint64/32/16/8```, ```int64/32/16/8```. (This is up for debate). Examples are shown later with class constructor overloading.

```js
function Foo(foo:float32) { }
function Foo(foo:uint32) { }
Foo(1); // float32 called
Foo(uint32(1)); // uint32 called
```

### Explicit Casting

```js
let foo = uint8(65535); // Cast taking the lowest 8 bits so the value 255, but note that foo is still typed as any
```

Many truncation rules have intuitive rules going from larger bits to smaller bits or signed types to unsigned types. Type casts like decimal to float or float to decimal would need to be clear.

### Function signatures with constraints

```js
function Foo(a:int32, b:string, c:bigint[], callback:(bool, string) = (b, s = 'none') => b ? s : ''):int32 { }
```

### Typed Arrow Functions

```js
let foo:(int32, string):string; // hold a reference to a signature of this type
let foo:(); // void is the default return type for a signature without a return type
let foo = (s:string, x:int32) => s + x; // implicit return type of string
let foo = (x:uint8, y:uint8):uint16 => x + y; // explicit return type
let foo = x:uint8 => x + y; // single parameter
```

### Integer Binary Shifts

```js
let a:int8 = -128;
a >> 1; // -64, sign extension
let b:uint8 = 128;
b >> 1; // 64, no sign extension as would be expected with an unsigned type
```

### Integer Division

```js
let a:int32 = 3;
a /= 2; // 1
```

### Destructing Assignment Casting

Array destructuring with default values:

```js
[a:uint32 = 1, b:float32 = 2] = Foo();
```

Object destructuring with default values:

```js
{ a = 1, b = 2 }:{ a:uint8, b:uint8 } = { a: 2 };
```

Object destructuring with default value and new name:

```js
let { a: b = 1 }: { a:uint8 } = { a: 2 }; // b is 2
```

Alternatively assigning to an already declared variable:

```js
let b:uint8;
({ a: b = 1 }:{ a:uint8 } = { a: 2 }); // b is 2
```

Destructuring with functions:

```js
(function({a: b = 0, b: a = 0}:{ a:uint8, b:uint8 }, [c:uint8])
{
    // a = 2, b = 1, c = 0
})({a: 1, b: 2}, [0]);
```

Deep object destructuring:

```js
const o: { a: { a2: uint8 } } = { a: { a2: 1 } };
const { a: { a2: b } }: { a: { a2: uint32 } } = o; // b is 1 and type uint32
```

### Typed Assignment

A variable by default is typed ```any``` meaning its dynamic and its type changes depending on the last assigned value. As an example one can write:

```js
let a = new MyType();
a = 5; // a is type any and is 5
```
If one wants to constrain the variable type they can write:
```js
let a:MyType = new MyType();
// a = 5; // TypeError, a is type MyType
```

This redundancy in declaring types for the variable can be removed with a typed assignment:

```js
let a := new MyType(); // a is type MyType
// a = 5; // TypeError, a is type MyType
```

This new form of assignment is useful with both ```var``` and ```let``` declarations. With ```const``` it has no uses:

```js
const a = new MyType(); // a is type MyType
const b:MyType = new MyType(); // Redundant, b is type MyType even without explicitly specifying the type
const c := new MyType(); // Redundant, c is type MyType even without explicitly specifying the type
const d:MyType = 1; // Calls a matching constructor
const e:uint8 = 1; // Without the type this would have been typed Number
class A {}
class B extends A {}
const f:A = new B(); // This might not even be useful to allow
```

This assignment also works with destructuring:

```js
let { a, b } := { a:uint8: 1, b:uint32: 2 }; // a is type uint8 and b is type uint32
```

### Function Overloading

```js
function Foo(x:int32[]) { return "int32"; }
function Foo(s:string[]) { return "string"; }
Foo(["test"]); // "string"
```

Up for debate is if accessing the separate functions is required. Functions are objects so using a key syntax with a string isn't ideal. Something like ```Foo["(int32[])"]``` wouldn't be viable. It's possible ```Reflect``` could have something added to it to allow access.

### Object Typing

Syntax:

```js
let o = { a:uint8: 1 };
```

This syntax works with any arrays:

```js
let o = { a:[] }; // Normal array syntax works as expected
let o = { a:[]: [] }; // With typing this is identical to the above
```

```Object.defineProperty``` and ```Object.defineProperties``` have a type property in the descriptor that accepts a type or string representing a type:

```js
Object.defineProperty(o, 'foo', { type: uint8 }); // using the type
Object.defineProperty(o, 'foo', { type: 'uint8' }); // using a string representing the type
```

```js
Object.defineProperties(o,
{
    'foo':
    {
        type: uint8,
        value: 0,
        writable: true
    },
    'bar':
    {
        type: string,
        value: 'Hello',
        writable: true
    }
});
```

### Constructor Overloading

```js
class MyType
{
    x:float32; // Able to define members outside of the constructor
    constructor(x:float32)
    {
        this.x = x;
    }
    constructor(y:uint32)
    {
        this.x = float32(y) * 2;
    }
}
```

Implicit casting using the constructors:
```js
let t:MyType = 1; // float32 constructor call
let t:MyType = uint32(1); // uint32 constructor called
```

Constructing arrays all of the same type:
```js
let t = new MyType[5](1);
```

### parseFloat and parseInt For Each New Type

For integers (including bigint) the parse function would have the signature ```parse(string, radix = 10)```.

```js
let foo:uint8 = uint8.parse('1', 10);
let foo:uint8 = uint8.parse('1'); // Same as the above with a default 10 for radix
let foo:uint8 = '1'; // Calls parse automatically making it identical to the above
```

For floats, decimals, and rational the signature is just ```parse(string)```.

```js
let foo:float32 = float32.parse('1.2');
```

### Implicit SIMD Constructors

Going from a scalar to a vector:

```js
let foo:float32x4 = 1; // Equivalent to let foo = float32x4(1, 1, 1, 1);
```

### Implicit Array Cast

```js
let foo:MyType[] = [1, 2, 3, uint32(1)];
```

### Initializer List for Array of Class Instances

Implicit array casting already exists for single variables as defined above. It's possible one might want to compactly create instances. The following syntax is proposed:

```js
let foo = new MyType[] = [(10, 20), (30, 40), 10];
```
This would be equivalent to:
```js
let foo = new MyType[] = [new MyType(10, 20), new MyType(30, 40), 10];
```

Due to the very specialized syntax it can't be introduced later. In ECMAScript the parentheses have defined meaning such that ```[(10, 20), 30]``` is ```[20, 30]``` when evaluated. This special syntax takes into account that an array is being created requiring more grammar rules to specialize this case.

Initializer lists work well with SIMD to create compact arrays of vectors:

```js
let foo = new float32x4[] =
[
    (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4),
    (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4),
    (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4)
];
```

The syntax also has to work with typed functions intuitively:
```js
function Foo(foo:float32x4)
{
}

Foo([(1, 2, 3, 4)]);
```

### Decorators

Types would function exactly like you'd expect with decorators, but with the addition that they can be overloaded.
```js
function AlwaysReturnValue(value:uint32)
{
    return function (target, name, descriptor)
    {
        descriptor.get = () => value;
        return descriptor;
    }
}
function AlwaysReturnValue(value:float32) { /* ... */ }
```

### Classes and Operator Overloading

```js
class Vector2d
{
    x:float32;
    y:float32;
    constructor(x:float32 = 0, y:float32 = 0)
    {
        this.x = x;
        this.y = y;
    }
    Length():float32
    {
        return Math.sqrt(x * x + y * y); // uses Math.sqrt(v:float32):float32 due to input and return type
    }
    get X():float64 // return implicit cast
    {
        return this.x;
    }
    set X(x:float64)
    {
        this.x = x / 2;
    }
    operator +(v:Vector2d)
    {
        return new vector2d(this.x + v.x, this.y + v.y);
    }
    operator ==(v:Vector2d)
    {
        // equality check between this and v
    }
}
```

### Class Extension

Example defined in say ```MyClass.js``` defining extensions to ```Vector2d``` defined above:

```js
class Vector2d
{
    operator ==(v:MyClass)
    {
        // equality check between this and MyClass
    }
    operator +(v:MyClass)
    {
        return v + this; // defined in terms of the MyClass operator
    }
}
```

Note that no members may be defined in an extension class. The new methods are simply appended to the existing class definition.

### SIMD Operators

All SIMD types would have operator overloading added when used with the same type.
```js
let a = uint32x4(1, 2, 3, 4) + uint32x4(5, 6, 7, 8); // uint32x4
let b = uint32x4(1, 2, 3, 4) < uint32x4(5, 6, 7, 8); // bool32x4
```
It's also possible to overload class operators to work with them, but the optimizations would be implementation specific if they result in SIMD instructions.

### enum Type

Enumerations with ```enum``` that support any type including functions.
```js
enum Count { Zero, One, Two }; // Starts at 0
let c:Count = Count.Zero;

enum Count { One = 1, Two, Three }; // Two is 2 since these are sequential
let c:Count = Count.One;

enum Count:float32 { Zero, One, Two };

enum Counter:(float32):float32 { Zero = x => 0, One = x => x + 1, Two = x => x + 2 }
```
Custom sequential functions for numerical and string types (these aren't closures):
```js
enum Count:float32 { Zero = (index, name) => index * 100, One, Two }; // 0, 100, 200
enum Count:string { Zero = (index, name) => name, One, Two = (index, name) => name.toLowerCase(), Three }; // "Zero", "One", "two", "three"
enum Flags:uint32 { None = 0, Flag1 = (index, name) => 1 << index, Flag2, Flag3 } // 0, 1, 2, 4
```
Index operator:
```js
enum Count { Zero, One, Two };
Count[0]; // Count.Zero
Count['Zero']; // Count.Zero
```

Get ```enum``` value as string:

```js
Count.toString(Count.Zero); // 'Zero'
```

It seems like there needs to be an expression form also. Something akin to Function or GeneratorFunction which allows the construction of features with strings. It's not clear to me if this is required or beneficial, but it could be. I guess the syntax would look like:

```js
new enum('a', 0, 'b', 1);
new enum(':uint8', 'a', 0, 'b', 1);
new enum(':string', 'None', 'none', 'Flag1', '(index, name) => name', 'Flag2', 'Flag3'); // This doesn't make much sense though since the value pairing is broken. Need a different syntax
```

Similar to ```Array``` there would be a number of reserved functions:

```js
enum.prototype.keys() // Array Iterator with the string keys
enum.prototype.values() // Array Iterator with the values
enum.prototype.entries() // Array Iterator with [key, value]
enum.prototype.forEach((key, value, enumeration) => {})
enum.prototype.filter((key, value, enumeration) => {}) // returns an Array
enum.prototype.map((key, value, enumeration) => {}) // returns an Array
enum.prototype[@@iterator]()
```

Iteration would work like this:

```js
enum Count { Zero, One, Two };

for (let [key, value] of Count)
{
    // key = 'Zero', value = 0
}
```

### Rest Parameters

```js
function Foo(a:string, ...args:uint32) {}
Foo('foo', 0, 1, 2, 3);
```
Rest parameters are valid for signatures:
```js
let foo:(...:uint8);
```
Multiple rest parameters can be used:
```js
function Foo(a:string, ...args:uint32, ...args2:string, callback:()) {}
Foo('foo', 0, 1, 2, 'foo', 'bar', () => {});
```
Dynamic types have less precedence than typed parameters:
```js
function Foo(...args1, callback:(), ...args2, callback:()) {}
Foo('foo', 1, 1.0, () => {}, 'bar', 2, 2.0, () => {});
```

### Try Catch

Catch clauses can be typed allowing for minimal conditional catch clauses.

```js
try
{
    // Statement that throws
}
catch (e:TypeError)
{
    // Statements to handle TypeError exceptions
}
catch (e:RangeError)
{
    // Statements to handle RangeError exceptions
}
catch (e:EvalError)
{
    // Statements to handle EvalError exceptions
}
catch (e)
{
    // Statements to handle any unspecified exceptions
}
```

### Placement New

Arbitrary arrays can be allocated into using the placement new syntax. This works with both a single instance and array of instances.

```js
let foo = new(buffer, byteOffset) MyType(20);
```

```js
let foo = new(buffer, byteOffset, byteStride) MyType[10](20);
```

### Control Structures

## if else

A table should be included here with every type and which values evaluate to executing. At first glance it might just be 0 and NaN do not execute and all other values do. SIMD types probably would not implicitly cast to boolean and attempting to would produce a TypeError indicating no implicit cast is available.

## switch

The variable when typed in a switch statement must be integral or a string type. Specifically ```int8/16/32/64```, ```uint8/16/32/64```, ```number```, and ```string```. Most languages do not allow floating point case statements unless they also support ranges. (This could be considered later with causing backwards compatability issues).

Enumerations can be used dependent on if their type is integral or string.
```js
let foo:uint32 = 10;
switch (foo)
{
    case 10:
        break;
    case `baz`: // TypeError unexpected string literal, expected uint32 literal
        break;
}
```

```js
let foo:float32 = 1.23;
switch (foo) // TypeError float32 cannot be used in the switch variable
{
}
```

### Member memory alignment and offset

Two new keys would be added to the Object.defineProperty called ```align``` and ```offset```. For consistency between codebases two reserved decorators would be created called ```@align``` and ```@offset``` that would set the underlying keys with byte values. Align defines the memory address to be a multiple of a given number. On some software architectures specialized move operations and cache boundaries can use these for small advantages. Offset is always defined as the number of bytes from the start of the instance allocation in memory. It's possible to create a union by defining overlapping offsets.

Along with the member decorators, two class reserved properties would be created, ```align``` and ```size```. These would control the allocated memory alignment of the instances and the allocated size of the instances.

```js
@align(16) // Foo.align = 16; Defines the class memory alignment to be 16 byte aligned
@size(32) // Foo.size = 32; Defines the class as 32 bytes. Pads with zeros when allocating
class Foo
{
    @offset(2)
    x:float32; // Aligned to 16 bytes because of the class alignment and offset by 2 bytes because of the property alignment
    @align(4)
    y:float32x4; // 2 (from the offset above) + 4 (for x) is 6 bytes and we said it has to be aligned to 4 bytes so 8 bytes offset from the start of the allocation. Instead of @align(4) we could have put @offset(8)
}
```

These language features only apply if all the properties in a class are typed along with the complete prototype chain. Adding properties later with ```Object.defineProperty``` is allowed, but new properties are appended to the end. Modifying properties in a ```super``` class would not be allowed. It's likely that one would need to remove and readd all the properties if the goal is to change the structure. Or ```offset``` could be modified with ```Object.defineProperty``` to rearrange the properties.

WIP: The behavior of modifying plain old data classes that are already used in arrays needs to be well-defined.

An example of overlapping properties using ```offset``` creating a union where both properties map to the same memory:

```js
class Foo // 16 bytes
{
    @offset(0)
    x:float32x4;
    @offset(0)
    y:int32x4;
}
```

### Global Objects

The following global objects could be used as types:

```DataView```, ```Date```, ```Error```, ```EvalError```, ```InternalError```, ```Map```, ```Promise```, ```Proxy```, ```RangeError```, ```ReferenceError```, ```RegExp```, ```Set```, ```SyntaxError```, ```TypeError```, ```URIError```, ```WeakMap```, ```WeakSet```

## Undecided Topics

### Import Types

This has been brought up before, but possible solutions due to compatability issues would be to introduce special imports. Brenden once suggested something like:
```js
import {int8, int16, int32, int64} from "@valueobjects";
//import "@valueobjects";
```

## Overview of Future Considerations and Concerns

### Generics

This section is to show that a generic syntax can be seamlessly added with no syntax issues. A generic function example:

```js
function Foo<T>(foo:T):T
{
    let bar:T;
}
```

A generic class example:

```js
class Vector2d<T>
{
    x:T;
    y:T;
    constructor(x:T = 0, y:T = 0) // T could be inferred, but that might be asking too much. In any case T must have a constructor supporting a parameter 0 if this is a class.
    {
        this.x = x;
        this.y = y;
    }
}
```

Generic constraints aren't defined here but would need to be. TypeScript has their extends type syntax. Being able to constrain ```T``` to an interface seems like an obvious requirement. Also being able to constrain to a list of specific types or specifically to numeric, floating point, or integer types. Another consideration is being able to support a default type. Also generic specialization for specific types that require custom definitions. There's probably more to consider, but those are the big ideas for generics.

Typedefs or aliases for types are a requirement. Not sure what the best syntax is for proposing these. There's a lot of ways to approach them. TypeScript has a system, but I haven't seen alternatives so it's hard for me to judge if it's the best or most ideal syntax.

### Interfaces

Taken from TypeScript an interface allows for structural subtyping, essentially structure matching.

See: https://www.typescriptlang.org/docs/handbook/interfaces.html

### Value Type Classes

I left value type classes out of this discussion since I'm still not sure how they'll be proposed. Doesn't sound like they have a strong proposal still or syntax.

### Union Types

Having function overloading removes most use cases for TypeScript's union types and optional parameters. Future proposals can try to justify them since none of their syntax conflicts with anything proposed.

```js
let a:string|uint32 = `hello`;
```

### Numeric Literals

Many languages have numeric literal suffixes to indicate a number is a specific data type. This isn't necessary if explicit casts are used. Since there are so many types, the use of suffixes would not be overly readable. The goal was to keep type names to a minimum number of characters also so that literals are less needed.

### Private, Public, and Static types and methods

Unlike other proposals adding types allows for robust type checking that allows for private, public, and static member and method checks. It can be added like this:

https://github.com/sirisian/ecmascript-class-member-modifiers

### Partial Class

```js
partial class MyType
{
}
```

Partial classes are when you define a single class into multiple pieces. When using partial classes the ordering members would be undefined. What this means is you cannot create views of a partial class using the normal array syntax and this would throw an error.

### Switch ranges

If case ranges were added and switches were allowed to use non-integral and non-string types then the following syntax could be used in future proposals without conflicting since this proposal would throw a ```TypeError``` restricting all cases of its usage keeping the behavior open for later ideas.

```js
let foo:float32 = 1 / 5;
switch (foo)
{
    case 0..0.99:
        break;
}
```

### Bit-fields

These are incredibly niche. That said I've had at least one person mention them to me in an issue. C itself has many rules like that property order is undefined allowing properties to be rearranged more optimally meaning the memory layout isn't defined. This would all have to be defined probably since some view this as a benefit and others view it as a design problem. Controlling options with decorators might be ideal. Packing and other rules would also need to be clearly defined.

Would allow the following types to define bit lengths.

```
bool
int8/16/32/64
uint8/16/32/64
```

```js
class Vector2d
{
    x:uint8(4); // 4 bits
    y:uint8(4); // 4 bits
}
```

### Exception filters

See https://github.com/sirisian/ecmascript-types/issues/22

A very compact syntax can be used later for exception filters:

```js
catch (e:Error => e.message == `foo`)
```

Or

```js
catch (e:Error => console.log(e.message))
```

This accomplishes exception filters without requiring a keyword like "when". That said it would probably not be a true lambda and instead be limited to only expressions.

### Named Arguments

Named arguments comes up once in a while as a compact way to skip default parameters.

```js
function f(a:uint8, b:string = 0, ...args:string) {}
f(8, args:`a`, `b`);

function g(option1:string, option2:string) {}
// g(option2: `a`); Error no signature for g matches (option2:string)
```

The above syntax is probably what would be used and it has no obvious conflicts with types.

# Example:  
Packet bit writer/reader https://gist.github.com/sirisian/dbc628dde19771b54dec

# Previous discussions

Current Mailing List Thread: https://esdiscuss.org/topic/proposal-optional-static-typing-part-3

Second Thread: https://esdiscuss.org/topic/optional-static-typing-part-2  
Original thread: https://esdiscuss.org/topic/es8-proposal-optional-static-typing  
This one contains a lot of my old thoughts:   https://esdiscuss.org/topic/proposal-for-new-floating-point-and-integer-data-types  
https://esdiscuss.org/topic/optional-strong-typing  
https://esdiscuss.org/topic/optional-argument-types  

# Required Changes to the Specification:

### 6.1 ECMAScript Language Types

Would need to include the types listed above. Probably in a more verbose view than a list.

### 6.1.3 The Boolean Type

```bool``` is an alias for ```Boolean``` when ```bool``` is imported.

### 6.1.4 The String Type

```string``` is an alias for ```String``` when ```string``` is imported.

### 6.1.5 The Symbol Type

```symbol``` is an alias for ```Symbol``` when ```symbol``` is imported.

### 6.1.6 The Number Type

```number``` is an alias for ```Number``` when ```number``` is imported.

### 6.1.7 The Object Type

```object``` is an alias for ```Number``` when ```object``` is imported.

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

```bool8x16```, ```bool16x8```, ```bool32x4```, ```bool64x2```, ```bool8x32```, ```bool16x16```, ```bool32x8```, ```bool64x4```  
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


*EnumDeclaration* :  
&nbsp;&nbsp;&nbsp;&nbsp;**enum** *Identifier* *ColonType*<sub>opt</sub> **{** *EnumElementList* **}**

*EnumElementList* :  
&nbsp;&nbsp;&nbsp;&nbsp;*EnumElement*  
&nbsp;&nbsp;&nbsp;&nbsp;*EnumElementList* **,** *EnumElement*  

*EnumElement* :  
&nbsp;&nbsp;&nbsp;&nbsp;*Identifier*  
&nbsp;&nbsp;&nbsp;&nbsp;*Identifier* **=** *Literal*  
&nbsp;&nbsp;&nbsp;&nbsp;*Identifier* **=** *ArrowFunction*

#### A.2 Expressions

*BindingIdentifier*<sub>[Yield, Await]</sub> :  
*Identifier* *ColonType*<sub>opt</sub>  
**yield**  
**await**  

*PropertyDefinition*<sub>[Yield, Await]</sub> :  
&nbsp;&nbsp;&nbsp;&nbsp;*IdentifierReference*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*CoverInitializedName*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*PropertyName*<sub>[?Yield, ?Await]</sub> *ColonType*<sub>opt</sub> **:** *AssignmentExpression*<sub>[+In, ?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*MethodDefinition*<sub>[?Yield, ?Await]</sub>  

*ColonIntegralType* :  
&nbsp;&nbsp;&nbsp;&nbsp;**:** *IntegralType*  

*TypedArrayIndexList* :  
&nbsp;&nbsp;&nbsp;&nbsp;*ArrowFunction*  
&nbsp;&nbsp;&nbsp;&nbsp;*ArrowFunction* **,** *TypedArrayIndexList*  

*ColonTypedArrayIndexList* :  
&nbsp;&nbsp;&nbsp;&nbsp;**,** *TypedArrayIndexList*  

*TypedArrayIndexParameters* :  
&nbsp;&nbsp;&nbsp;&nbsp;*AssignmentExpression*<sub>opt</sub> *ColonIntegralType*<sub>opt</sub> *ColonTypedArrayIndexList*<sub>opt</sub>  

*PlacementNew*<sub>[Yield, Await]</sub> :  
&nbsp;&nbsp;&nbsp;&nbsp;**(** *AssignmentExpression*<sub>[?Yield, ?Await]</sub> **)**  
&nbsp;&nbsp;&nbsp;&nbsp;**(** *AssignmentExpression*<sub>[?Yield, ?Await]</sub> **,** *AssignmentExpression*<sub>[?Yield, ?Await]</sub> **)**  
&nbsp;&nbsp;&nbsp;&nbsp;**(** *AssignmentExpression*<sub>[?Yield, ?Await]</sub> **,** *AssignmentExpression*<sub>[?Yield, ?Await]</sub> **,** *AssignmentExpression*<sub>[?Yield, ?Await]</sub> **)**  

*MemberExpression*<sub>[Yield, Await]</sub> :  
&nbsp;&nbsp;&nbsp;&nbsp;*PrimaryExpression*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*MemberExpression*<sub>[?Yield, ?Await]</sub> **[** *Expression*<sub>[+In, ?Yield, ?Await]</sub> **]**  
&nbsp;&nbsp;&nbsp;&nbsp;*MemberExpression*<sub>[?Yield, ?Await]</sub> **.** *IdentifierName*  
&nbsp;&nbsp;&nbsp;&nbsp;*MemberExpression*<sub>[?Yield, ?Await]</sub> *TemplateLiteral*<sub>[?Yield, ?Await, +Tagged]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*SuperProperty*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*MetaProperty*  
&nbsp;&nbsp;&nbsp;&nbsp;**new** *MemberExpression*<sub>[?Yield, ?Await]</sub> *Arguments*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;**new** *PlacementNew*<sub>[?Yield, ?Await] opt</sub> *Idenitifier* **[** *TypedArrayIndexParameters* **]**

#### A.3 Statements

*BindingProperty*<sub>[Yield, Await]</sub> :
&nbsp;&nbsp;&nbsp;&nbsp;*SingleNameBinding*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*PropertyName*<sub>[?Yield, ?Await]</sub> *ColonType*<sub>opt</sub> **:** *BindingElement*<sub>[?Yield, ?Await]</sub>  

*SingleNameBinding*<sub>[Yield, Await]</sub> :  
&nbsp;&nbsp;&nbsp;&nbsp;*BindingIdentifier*<sub>[?Yield, ?Await]</sub> *Initializer*<sub>[+In, ?Yield, ?Await] opt<sub>  

*BindingRestElement*<sub>[Yield, Await]</sub> :  
&nbsp;&nbsp;&nbsp;&nbsp;**...** *BindingIdentifier*<sub>[?Yield, ?Await]</sub> *ColonType*<sub>opt</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;**...** *BindingPattern*<sub>[?Yield, ?Await]</sub>

*CatchParameter*<sub>[Yield, Await]</sub> :  
&nbsp;&nbsp;&nbsp;&nbsp;*BindingIdentifier*<sub>[?Yield, ?Await]</sub> *ColonType*<sub>opt</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*BindingPattern*<sub>[?Yield, ?Await]</sub>  


#### A.4 Functions and Classes

*FunctionDeclaration*[Yield, Await, Default] :  
&nbsp;&nbsp;&nbsp;&nbsp;**function** *BindingIdentifier*[?Yield, ?Await] **(** *FormalParameters*<sub>[~Yield, ~Await]</sub> **)** *ColonType*<sub>opt</sub> **{** *FunctionBody*<sub>[~Yield, ~Await]</sub> **}**  
&nbsp;&nbsp;&nbsp;&nbsp;[+Default]*function* **(** *FormalParameters*<sub>[~Yield, ~Await]</sub> **)** *ColonType*<sub>opt</sub> **{** *FunctionBody*<sub>[~Yield, ~Await]</sub> **}**  

*FunctionExpression* :  
&nbsp;&nbsp;&nbsp;&nbsp;**function** *BindingIdentifier*<sub>[~Yield, ~Await] opt</sub> **(** *FormalParameters*<sub>[~Yield, ~Await]</sub> **)** *ColonType*<sub>opt</sub> **{** *FunctionBody*<sub>[~Yield, ~Await]</sub> **}**  

*ClassElement*<sub>[Yield, Await]</sub> :  
&nbsp;&nbsp;&nbsp;&nbsp;*MemberDefinition*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*MethodDefinition*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*staticMethodDefinition*<sub>[?Yield, ?Await]</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;;

*MemberDefinition* :  
&nbsp;&nbsp;&nbsp;&nbsp;*Identifier* *ColonType*<sub>opt</sub> **;**

