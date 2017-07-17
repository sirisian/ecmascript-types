# ECMAScript Proposal: Optional Static Typing

Current status of this proposal is -1. It's in a theoretical state at the moment to better understand how types could function in Javascript and the long-term future benefits or complications they could cause to future proposals.

## Rationale

With TypedArrays and classes finalized, ECMAScript is in a good place to finally discuss types again. The demand for types as a different approach to code has been so strong in the past few years that separate languages have been created to deal with the perceived shortcomings. Types won't be an easy discussion, nor an easy addition, since they touch a large amount of the language; however, they are something that needs rigorous discussion.

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

In theory the following current keywords could be deprecated in the very long-term: Boolean, Number, String, Symbol, and the TypedArray objects. Their methods and features would be rolled into the new type system.

### Variable Declaration With Type

This syntax is taken from ActionScript and other proposals over the years. It's subjectively concise, readable, and consistent throughout the proposal.

```js
var foo:Type = value;
let foo:Type = value;
const foo:Type = value;
```

### typeof Operator

One of the first complications with types is typeof's behavior. All of the above types would return their string conversion including bool. (I've spoken to many people now and "boolean" is seen as verbose among C++ and C# developers. Breaking this part of Java's influence probably wouldn't hurt to preserve consistency for the future).

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

That would imply Object.getPrototypeOf(value) === ((uint8):uint8).prototype

I'm not well versed on if this makes sense though, but it would be like each typed function has a prototype defined by the signature.

### Nullable Types

By default all types except any are non-nullable. The syntax below creates a nullable uint8 variable:
```js
let foo:uint8? = null; // typeof foo == "uint8?"
```

### any Type

Using "any?" would result in a syntax error since "any" already includes nullable types. As would using "any[]" since it already includes array types. Using just "[]" would be the type for arrays that can contain anything. For example:

```js
let foo:[];
```

### Variable-length Typed Arrays

```js
let foo:uint8[];
foo.push(1);
let bar:uint8[] = [1, 2, 3, 4];
```

The index operator doesn't perform casting just to be clear so array objects even when typed still behave like objects.

```js
let foo:uint8[] = [1, 2, 3, 4];
foo['0'] = 'foobar'; // property '0'
'0' in foo; // true
delete foo['0'];
```

### Fixed-length Typed Arrays

```js
let foo:uint8[4];
foo.push(0); // invalid
foo.pop(); // invalid
foo[0] = 1; // valid
foo[foo.length] = 2;  // invalid
let bar:uint8[4] = [1, 2, 3, 4];
```

Typed arrays would be zero-ed at creation.

### Mixing Variable-length and Fixed-length Arrays

```js
function Foo(p:boolean):uint8[] // default case, return a resizable array
{
    let foo:uint8[4] = [1, 2, 3, 4];
    let bar:uint8[6] = [1, 2, 3, 4, 5, 6];
    return p ? foo : bar;
}

function Foo(p:boolean):uint8[6] // return a resized foo
{
    let foo:uint8[4] = [1, 2, 3, 4];
    let bar:uint8[6] = [1, 2, 3, 4, 5, 6];
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
const bar:uint8[] = [1, 2, 3, 4];
delete bar[0]; // throws TypeError
```

### Array length Type And Operations

Valid types for defining the length of an array are as follows:

int8/16/32/64  
uint8/16/32/64

By default length is uint32.

Syntax:

```js
let foo:uint8[int8];
let bar = foo.length; // int8
```

```js
let foo:uint8[10:uint64];
let bar = foo.length; // uint64 with value 10
```

```js
let n = 10;
let foo:uint8[n:uint64];
let bar = foo.length; // uint64 with value 10
```

Setting the length reallocates the array truncating when applicable.

```js
let foo:uint8[10];
foo.length = 5;
foo.length = 15;
```

### Array views:

Like TypedArray views this array syntax allows any array, even arrays of typed objects to be viewed as different objects. Stride would have performance implications but would allow for a single view to access elements with padding between elements.

```js
let view = Type[](buffer [, byteOffset [, byteLength [, byteStride]]]);
```

```js
let foo:uint64[] = [1];
let bar = uint32[](foo, 0, 8);
```

Take special note of the lack of "new" in this syntax. Adding new in the above case would pass the arguments to the constructor for each element.

### Multidimensional and Jagged Array Support Via User-defined Index Operators

Rather than definining index functions for various multidimensional and jagged array implementations the user is given the ability to define their own. Any lambda parameter passed to the "index constructor" creates an indexing function. More than one can be defined as long as they have unique signatures.

An example of a user-defined index to access a 16 element grid with (x, y) coordinates:

```js
let grid = new uint8[16:uint32, (x:uint32, y:uint32) => y * 4 + x];
// grid[0] = 10; // throws Error, invalid arguments
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

The default numeric type Number would convert implicitly with precedence given to decimal128/64/32, float128/80/64/32/16, uint64/32/16/8, int64/32/16/8. (This is up for debate). Examples are shown later with class constructor overloading.

```js
function Foo(foo:float32) { }
function Foo(foo:uint32 { }
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
{ a:uint8 = 1, b:uint8 = 2 } = { a: 2 };
```

Object destructuring with default value and new name:

```js
let { a:uint8: b = 1 } = { a: 2 }; // b is 2
```

Alternatively assigning to an already declared variable:

```js
let b:uint8;
({ a:uint8: b = 1 } = { a: 2 }); // b is 2
```

Also for completeness using destructuring with functions:

```js
(function({a:uint8: b = 0, b:uint8: a = 0}, [c:uint8])
{
    // a = 2, b = 1, c = 0
})({a: 1, b: 2}, [0]);
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
let o = { a:uint8: 1 };
```

This syntax works with any arrays:

```js
let o = { a:[] }; // Normal array syntax works as expected
let o = { a:[]: [] }; // With typing this is identical to the above
```

Object.defineProperty and Object.defineProperties have a type property in the descriptor that accepts a type or string representing a type:

```js
Object.defineProperty(o, 'foo', { type:uint8 }); // using the type
Object.defineProperty(o, 'foo', { type: 'uint8' }); // using a string representing the type
```

```js
Object.defineProperties(o,
{
    'foo':
    {
        type:uint8,
        value: 0,
        writable: true
    },
    'bar':
    {
        type:string,
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

For integers (including bigint) the parse function would have the signature parse(string, radix = 10).

```js
let foo:uint8 = uint8.parse('1', 10);
let foo:uint8 = uint8.parse('1'); // Same as the above with a default 10 for radix
let foo:uint8 = '1'; // Calls parse automatically making it identical to the above
```

For floats, decimals, and rational the signature is just parse(string)

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

Due to the very specialized syntax it can't be introduced later. In ECMAScript the parentheses have defined meaning such that [(10, 20), 30] is [20, 30] when evaluated. This special syntax takes into account that an array is being created requiring more grammar rules to specialize this case.

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

Types would function exactly like you'd expect with decorators, but with the addition that they can be overloaded:
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

Note that no members may be defined in an extension class. The new methods are simply appended to the existing class definition.

### SIMD Operators

All SIMD types would have operator overloading added when used with the same type.
```js
let a = uint32x4(1, 2, 3, 4) + uint32x4(5, 6, 7, 8); // uint32x4
let b = uint32x4(1, 2, 3, 4) < uint32x4(5, 6, 7, 8); // bool32x4
```
It's also possible to overload class operators to work with them, but the optimizations would be implementation specific if they result in SIMD instructions.

### enum Type

Enumerations with enum that support any type including functions.
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

Get enum value as string:

```js
Count.toString(Count.Zero); // 'Zero'
```

It seems like there needs to be an expression form also. Something akin to Function or GeneratorFunction which allows the construction of features with strings. It's not clear to me if this is required or beneficial, but it could be. I guess the syntax would look like:

```js
new enum('a', 0, 'b', 1);
new enum(':uint8', 'a', 0, 'b', 1);
new enum(':string', 'None', 'none', 'Flag1', '(index, name) => name', 'Flag2', 'Flag3'); // This doesn't make much sense though since the value pairing is broken. Need a different syntax
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

A table should be included here with every type and which values evaluate to executing. At first glance it might just be 0 and NaN do not execute and all other values do. SIMD types probably would not implicitly cast to boolean and attempting to would produce a TypeError indicating no implicit cat is available.

## switch

The variable used in a switch statement must be integral. Specifically int8/16/32/64, uint8/16/32/64, and Number. Most languages do not allow floating point case statements unless they also support ranges. This could be considered later.

Enumerations can be used dependent on if their type is integral.
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

### Global Objects

The following global objects could be used as types:

DataView, Date, Error, EvalError, InternalError, Map, Promise, Proxy, RangeError, ReferenceError, RegExp, Set, SyntaxError, TypeError, URIError, WeakMap, WeakSet

## Undecided Topics

### Import Types

This has been brought up before, but possible solutions due to compatability issues would be to introduce special imports. Brenden once suggested something like:
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

Generic constraints aren't defined here but would need to be. TypeScript has their extends type syntax. Being able to constrain T to an interface seems like an obvious requirement. Also being able to constrain to a list of specific types or specifically to numeric, floating point, or integer types. Another consideration is being able to support a default type. Also generic specialization for specific types that require custom definitions. There's probably more to consider, but those are the big ideas for generics.

Typedefs or aliases for types are a requirement. Not sure what the best syntax is for proposing these. There's a lot of ways to approach them. TypeScript has a system, but I haven't seen alternatives so it's hard for me to judge if it's the best or most ideal syntax.

### Value Type Classes

I left value type classes out of this discussion since I'm still not sure how they'll be proposed. Doesn't sound like they have a strong proposal still or syntax.

### Union Types

Having function overloading removes most use cases for TypeScript's union types and optional parameters. Future proposals can try to justify them since none of their syntax conflicts with anything proposed. (I imagine someone will create an interfaces proposal before that happens though).

### Numeric Literals

Many languages have numeric literal suffixes to indicate a number is a specific data type. This isn't necessary if explicit casts are used. Since there are so many types, the use of suffixes would not be overly readable. The goal was to keep type names to a minimum number of characters also so that literals are less needed.

### Private, Public, and Static types and methods

Unlike other proposals adding types allows for robust type checking that allows for private, public, and static member and method checks. It can be added like this:

https://github.com/sirisian/ecmascript-public-private-static

### Partial Class

```js
partial class MyType
{
}
```

Partial classes are when you define a single class into multiple pieces. When using partial classes the ordering members would be undefined. What this means is you cannot create views of a partial class using the normal array syntax and this would throw an error.

# Example:  
Packet bit writer/reader https://gist.github.com/sirisian/dbc628dde19771b54dec

# Previous discussions

Current Mailing List Thread: https://esdiscuss.org/topic/optional-static-typing-part-2  

Original thread: https://esdiscuss.org/topic/es8-proposal-optional-static-typing  
This one contains a lot of my old thoughts (at least the stuff from 8 months ago):   https://esdiscuss.org/topic/proposal-for-new-floating-point-and-integer-data-types  
https://esdiscuss.org/topic/optional-strong-typing  
https://esdiscuss.org/topic/optional-argument-types  

# Required Changes to the Specification:

### 6.1 ECMAScript Language Types

Would need to include the types listed above. Probably in a more verbose view than a list.

### 6.1.?

New sections would need to be added to cover every new type proposed above. Not too bad, most of the types are very well defined regarding ranges and behavior. The ones that need a bit of work are bigint, rational, and complex.

https://en.wikipedia.org/wiki/Half-precision_floating-point_format  
https://en.wikipedia.org/wiki/Single-precision_floating-point_format  
https://en.wikipedia.org/wiki/Double-precision_floating-point_format  
https://en.wikipedia.org/wiki/Quadruple-precision_floating-point_format

https://en.wikipedia.org/wiki/Decimal32_floating-point_format  
https://en.wikipedia.org/wiki/Decimal64_floating-point_format  
https://en.wikipedia.org/wiki/Decimal128_floating-point_format

https://en.wikipedia.org/wiki/Rational_data_type

https://en.wikipedia.org/wiki/Complex_data_type

### 11.6.2.1

Move enum from 11.6.2.2 to 11.6.2.1.

### 11.8.3.1 Static Semantics: MV’s

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

### A.1 Lexical Grammar 

THIS SECTION IS A WIP

*Keyword* :: **one of**  
&nbsp;&nbsp;&nbsp;&nbsp;**break** **do** **in** **typeof** **case** **else** **instanceof** **var** **catch** **export** **new** **void** **class** **extends** **return** **while** **const** **finally** **super** **with** **continue** **for** **switch** **yield** **debugger** **function** **this** **default** **if** **throw** **delete** **import** **try** **enum**

*FutureReservedWord* ::  
&nbsp;&nbsp;&nbsp;&nbsp;**await**

*Type* :  
&nbsp;&nbsp;&nbsp;&nbsp;**:** *ReservedTypes*<sub>opt</sub> *TypeArray*<sub>opt</sub> **?**<sub>opt</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;**:** *Identifier*<sub>opt</sub> *TypeArray*<sub>opt</sub> **?**<sub>opt</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;**:** *FunctionSignature* **?**<sub>opt</sub>

*TypeArray* :  
&nbsp;&nbsp;&nbsp;&nbsp;**[** *TypeArrayExpression*<sub>opt</sub> **]**

*TypeArrayExpression* :  
&nbsp;&nbsp;&nbsp;&nbsp;*DecimalDigits*
&nbsp;&nbsp;&nbsp;&nbsp;*AssignmentExpression*

*FunctionSignature* :  
&nbsp;&nbsp;&nbsp;&nbsp;**(** *FunctionSignatureElementList* **)**

*FunctionSignatureElementList* :  
&nbsp;&nbsp;&nbsp;&nbsp;*FunctionSignatureElement*  
&nbsp;&nbsp;&nbsp;&nbsp;*FunctionSignatureElementList* **,** *FunctionSignatureElement*

*FunctionSignatureElement* :  
&nbsp;&nbsp;&nbsp;&nbsp;*Identifier* *Type*<sub>opt</sub>

*VariableDeclaration*<sub>[In, Yield]</sub>:  
&nbsp;&nbsp;&nbsp;&nbsp;*BindingIdentifier*<sub>[?Yield]</sub> *Type*<sub>opt</sub> *Initializer*<sub>[?In, ?Yield]opt</sub>  
&nbsp;&nbsp;&nbsp;&nbsp;*BindingPattern*<sub>[?Yield]</sub> *Type*<sub>opt</sub> *Initializer*<sub>[?In, ?Yield]</sub>  


*EnumDeclaration* :  
&nbsp;&nbsp;&nbsp;&nbsp;**enum** *Identifier* *Type*<sub>opt</sub> **{** *EnumElementList* **}**

*EnumElementList* :  
&nbsp;&nbsp;&nbsp;&nbsp;*EnumElement*  
&nbsp;&nbsp;&nbsp;&nbsp;*EnumElementList* **,** *EnumElement*  

*EnumElement* :  
&nbsp;&nbsp;&nbsp;&nbsp;*Identifier*  
&nbsp;&nbsp;&nbsp;&nbsp;*Identifier* **=** *Literal*  
&nbsp;&nbsp;&nbsp;&nbsp;*Identifier* **=** *ArrowFunction*
