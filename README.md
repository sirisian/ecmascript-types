# ES8 Proposal: Optional Static Typing

## Introduction

With ES6's TypedArrays and classes finalized and ES7 SIMD getting experimental tests, ECMAScript is in a good place to finally discuss types again. The demand for types as a different approach to code has been so strong in the past few years that separate languages have been created to deal with the perceived shortcomings. Types won't be an easy discussion, nor an easy addition, since they touch a large amount of the language; however, they are something that needs rigorous discussion. Turning this into an official proposal discussed by TC39 is the goal.

It should be noted that the overlap in this proposal and current features in Javascript is expected. This is a gradual upgrade for the language.

### Types Proposed

Since it would be potentially years before this would be implemented this proposal includes a new keyword "enum" for enumerated types and the following types:

```
number
bool
string
object
symbol
int8/16/32/64
uint8/16/32/64
bigint
float16/32/64/80/128
decimal32/64/128
bool8x16/16x8/32x4/64x2/8x32/16x16/32x8/64x4
int8x16/16x8/32x4/64x2/8x32/16x16/32x8/64x4
uint8x16/16x8/32x4/64x2/8x32/16x16/32x8/64x4
float32x4/64x2/32x8/64x4
rational
complex
any
void
```

These types bring ECMAScript in line or surpasses the type systems in most languages. For developers it cleans up a lot of the syntax, as described later, for TypedArrays, SIMD, and working with number types (floats vs signed and unsigned integers). It also allows for new language features like function overloading and a clean syntax for operator overloading. For implementors, added types offer a way to better optimize the JIT when specific types are used. For languages built on top of Javascript this allows more explicit type usage and closer matching to hardware.

### Deprecated Keywords

In theory the following current keywords could be deprecated in the very long-term: Boolean, Number, String, Object, Symbol, and the TypedArray objects. Their methods and features would be rolled into the new type system.

### Variable Declaration With Type

This syntax is taken from ActionScript and many other proposals over the years. It's subjectively concise and readable.

```js
var foo:Type = value;
let foo:Type = value;
const foo:Type = value;
```

### typeof Operator

One of the first complications with types is typeof's behavior. All of the above types would return their string conversion including bool. (I've spoken to many people now and "boolean" is seen as verbose among C++ and C# developers. Breaking this part of Java's influence probably wouldn't hurt to preserve consistency for the future).

```js
var foo:uint8; // typeof foo == "uint8"
var bar:uint8? // typeof bar == "uint8?"
var baz:uint8[] // typeof baz == "object", ideally this would return "uint8[]", but it's not necessary
var foobar:(uint8):uint8; // typeof foobar == "function", ideally this would return "(uint8):uint8", but it's not necessary
```

### Nullable Types

By default all types except any are non-nullable. The syntax below creates a nullable uint8 variable:
```js
var foo:uint8? = null; // typeof foo == "uint8?"
```

### any Type

Using "any?" would result in a syntax error since "any" already includes nullable types. As would using "any[]" since it already includes array types. Using just "[]" would be the type for arrays that can contain anything. For example:

```js
var foo:[];
```

### Variable-length Typed Arrays

```js
var foo:uint8[];
foo.push(1);
var bar:uint8[] = [1, 2, 3, 4];
```

### Fixed-length Typed Arrays:

```js
var foo:uint8[4];
foo.push(0); // invalid
foo.pop(); // invalid
var bar:uint8[4] = [1, 2, 3, 4];
```

### Mixing Variable-length and Fixed-length Arrays

```js
function Foo(p:boolean):uint8[] // default case, return a resizable array
{
    var foo:uint8[4] = [1, 2, 3, 4];
    var bar:uint8[6] = [1, 2, 3, 4, 5, 6];
    return p ? foo : bar;
}

function Foo(p:boolean):uint8[6] // return a resized foo
{
    var foo:uint8[4] = [1, 2, 3, 4];
    var bar:uint8[6] = [1, 2, 3, 4, 5, 6];
    return p ? foo : bar;
}
```

### Any Typed Array

```js
var foo:[]; // Using any[] is a syntax error
var foo:[]? = null; // nullable array
```

### Implicit Casting

Number would convert implicitly with precedence given to decimal, float128/80/64/32/16, uint64/32/16/8, int64/32/16/8. Examples are shown later with class constructor overloading.

```js
function Foo(foo:float32) { }
function Foo(foo:uint32 { }
Foo(1); // float32 called
Foo(uint32(1)); // uint32 called
```

### Explicit Casting

```js
var foo = uint8(65535); // Cast taking the lowest 8 bits so the value 255, but note that foo is still typed as any
```

### Function signatures with constraints

```js
function Foo(a:int32, b:string, c:bigint[], callback:(bool, string) = (b, s = 'none') => b ? s : ''):int32 { }
```

### Typed Arrow Functions

```js
var foo:(int32, string):string; // hold a reference to a signature of this type
var foo:(); // void is the default return type for a signature without a return type
var foo = (s:string, x:int32) => s + x; // implicit return type of string
var foo = (x:uint8, y:uint8):uint16 => x + y; // explicit return type
var foo = x:uint8 => x + y; // single parameter
```

### Integer Binary Shifts

```js
var a:int8 = -128;
a >> 1; // -64, sign extension
var b:uint8 = 128;
b >> 1; // 64, no sign extension as would be expected with an unsigned type
```

### Integer Division

```js
var a:int32 = 3;
a /= 2; // 1
```

### Destructing Assignment Casting

```js
[a:uint32, b:float32] = Foo();
```

### Function Overloading

```js
function Foo(x:int32[]) { return "int32"; }
function Foo(s:string[]) { return "string"; }
Foo(["test"]); // "string"
```

Up for debate is if accessing the separate functions is required. Functions are objects so using a key syntax with a string isn't ideal. Something like Foo["(int32[])"] wouldn't be viable. It's possible Reflect could have something added to it to allow access.

### Object Typing

Syntax:

```js
var o = { a:uint8:1 };
```

This syntax works with any arrays:

```js
var o = { a:[] }; // Normal array syntax works as expected and the type is inferred to be any[]
var o = { a:[]:[] }; // With typing this is identical to the above
```

### Constructor Overloading

```js
// 4 byte object
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
var t:MyType = 1; // float32 constructor call
var t:MyType = uint32(1); // uint32 constructor called
```

### parseFloat and parseInt For Each New Type

For integers (including bigint) the parse function would have the signature parse(string, radix = 10).

```js
var foo:uint8 = uint8.parse('1', 10);
var foo:uint8 = uint8.parse('1'); // Same as the above with a default 10 for radix
var foo:uint8 = '1'; // Calls parse automatically making it identical to the above
```

For floats, decimals, and rational the signature is just parse(string)

```js
var foo:float32 = float32.parse('1.2');
```

### Implicit SIMD Constructors

Going from a scalar to a vector:

```js
var v:float32x4 = 1; // Equivalent to an ES7 SIMD splat, so var v = float32x4(1, 1, 1, 1);
```

### Implicit Array Cast

```js
var t:MyType[] = [1, 2, 3, uint32(1)];
```

### Decorators

Types would function exactly like you'd expect with decorators in ES7, but with the addition that they can be overloaded:
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
    x: float32;
    y: float32;
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

### Partial Class

Example defined in say MyClass.js defining extensions to Vector2d defined above:

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

### SIMD Operators

All SIMD types would have operator overloading added when used with the same type.
```js
var a = uint32x4(1, 2, 3, 4) + uint32x4(5, 6, 7, 8); // uint32x4
var b = uint32x4(1, 2, 3, 4) < uint32x4(5, 6, 7, 8); // bool32x4
```
It's also possible to overload class operators to work with them, but the optimizations would be implementation specific if they result in SIMD instructions.

### enum Type

Enumerations with enum that support any type except function signatures.
```js
enum Count { Zero, One, Two }; // Starts at 0
var c:Count = Count.Zero;

enum Count { One = 1, Two, Three }; // Two is 2 since these are sequential
var c:Count = Count.One;

enum Count:float32 { Zero, One, Two };
```
Custom sequential function (these aren't closures):
```js
enum Count:float32 { Zero = (index, name) => index * 100, One, Two }; // 0, 100, 200
enum Count:string { Zero = (index, name) => name, One, Two = (index, name) => name.toLowerCase(), Three }; // "Zero", "One", "two", "three"
```
Index operator:
```js
enum Count { Zero, One, Two };
Count[0];
```
Get enum value as string:

// Not sure what the syntax for this would be.

### Rest Parameters (ES6)

```js
function Foo(a:string, ...args:uint32) {}
Foo('foo', 0, 1, 2, 3);
```
Rest parameters are valid for signatures:
```js
var foo:(...:uint8);
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

### Global Objects

The following global objects could be used as types:

DataView, Date, Error, EvalError, InternalError, Map, Promise, Proxy, RangeError, ReferenceError, RegExp, Set, SyntaxError, TypeError, URIError, WeakMap, WeakSet

## Undecided Topics

### TypedArray Views

Should there be a new syntax for TypedArray views like:
```js
var a1:uint64[] = [2, 0, 1, 3];
var a2:uint32[] = a1(1, 2); // Creates a view of a1 at offset 1 with 2 elements. So [0, 1].
```
That might be asking too much though. In a very compact form:
```js
var foo = (uint64[](a1(1, 2))[0]; // foo would be 1
```
Bit conversions aren't much cleaner. Right now according to the changes going from an int8 to an uint8 requires:
```js
uint8(int8[]([value]).buffer)[0]
```
Ideally there should be a very elegant way to do a bitconversion from any type that's the same number of bits.

This has been brought up before, but possible solutions due to compatability issues would be to introduce "use types"; or since ES6 has them Brenden once suggested something like:
```js
import {int8, int16, int32, int64} from "@valueobjects";
//import "@valueobjects";
```

## Overview of Future Considerations and Concerns

### Generic Functions

This section is to show that a generic syntax can be seamlessly added with no syntax issues. A generic function example:

```js
function Foo<T>(foo:T):T
{
    var bar:T;
}
```

A generic class example:

```js
class Vector2d<T>
{
    x: T;
    y: T;
    constructor(x:T = 0, y:T = 0) // T could be inferred, but that might be asking too much. In any case T must have a constructor supporting a parameter 0 if this is a class.
    {
        this.x = x;
        this.y = y;
    }
}
```

Generic constraints aren't defined here but would need to be. TypeScript has their extends type syntax. Being able to constrain T to an interface seems like an obvious requirement. Also being able to constrain to a list of specific types or specifically to numeric, floating point, or integer types. Another consideration is being able to support a default type. Also generic specialization for specific types that require custom definitions. There's probably more to consider, but those are the big ideas for generics.

Typedefs or aliases for types are a requirement. Not sure what the best syntax is for proposing these. There's a lot of ways to approach them. TypeScript has a system, but I haven't see alternatives so it's hard for me to judge if it's the best or most ideal syntax.

### Value Type Classes

I left value type classes out of this discussion since I'm still not sure how they'll be proposed. Doesn't sound like they have a strong proposal still or syntax.

### Union Types

Having function overloading removes most use cases for TypeScript's union types and optional parameters. Future proposals can try to justify them since none of their syntax conflicts with anything proposed.

# Example:  
Packet bit writer/reader https://gist.github.com/sirisian/dbc628dde19771b54dec

# Previous discussions

Current Mailing List Thread: https://esdiscuss.org/topic/es8-proposal-optional-static-typing

This one contains a lot of my old thoughts (at least the stuff from 8 months ago):   https://esdiscuss.org/topic/proposal-for-new-floating-point-and-integer-data-types  
https://esdiscuss.org/topic/optional-strong-typing  
https://esdiscuss.org/topic/optional-argument-types  

# Required Changes to the Specification:

### 6.1 ECMAScript Language Types

Would need to include the types listed above. Probably in a more verbose view than a list.

### 6.1.?

New sections would need to be added to cover every new type proposed above. Not too bad, most of the types are very well defined regarding ranges and behavior. The ones that need a bit of work are bigint, rational, and complex.

### 11.6.2.1

Move enum from 11.6.2.2 to 11.6.2.1.

### 12.5.6 The typeof Operator

The table needs to be updated with all the new types and nullable type explanation.

### 12.5.7 to 12.11

Theese contain the operator definitions. Would probably need to include at least a brief change to explain the behavior of all the types. SIMD ones would require the most explanation.

### 12.14.5 Destructuring Assignment

Type casting syntax described above would need to be included.

### 13.3.2 Variable Statement

Would need to cover the optional typing syntax and grammar.

### 14 ECMAScript Language: Functions and Classes

This sections grammar would need to have FunctionDeclaration in the grammar redefined to include an optional typed return value. FormalParameter would require optional types added.

### 14.2 Arrow Function Definitions

The grammar rule ArrowFunction needs an optional return type and ArrowParameters needs optional type information per each parameter.

### 14.3 Method Definitions

Grammar requires typing information as defined above. Specifically MethodDefinition's first rule and the get and set ones.

### 14.5 Class Definitions

Grammar requires typing information for members and methods. Specifically ClassElement and MethodDefinition. ConstructorMethod is referenced as being a MethodDefinition so it should be fine after the MethodDefinition changes.

### 19.2 Function Objects

Needs to support type information in the constructor. Examples from the documentation with typed examples:

```js
new Function("a:string", "b:uint8", "c:int32", "return a+b+c")
new Function("a:string, b:uint8, c:int32", "return a+b+c") 
new Function("a:string,b:uint8", "c:int32", "return a+b+c") 
```

Syntax to define a return type:

```js
new Function("a:string", "b:uint8[]", "c:int32", ":string", "return a+b+c")
```

### New Sections for Each Type

As described before each type needs a parse function to turn a string into the type. Also for float and decimal .EPSILON needs defined in their section. Also for all the integer (not bigint), float, and decimal types need MIN_VALUE and MAX_VALUE defined in their section.

### 20.2 The Math Object

All the math operations need to be overloaded to work with the integer, float, and decimal types. Meaning if they take in the type they should return the same type.

### 25.2.2 Properties of the GeneratorFunction Constructor

Similar to Function the constructor needs to be changed to allow types. For example:

```js
var g = new GeneratorFunction("a:float32", ":float32", "yield a * 2");
```

### A.1 Lexical Grammar 

