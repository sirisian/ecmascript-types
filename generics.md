# Generics

The goal of generics would be to represent compile-time or at the least JIT optimized codepaths. In this way they're more similar to C++ templates. In a type system they allow simple generic classes for specializing for types.

Concretely, the semantics are those of full specialization. Each application - ```A.<uint8>```, ```A.<uint16>``` - is a distinct type with its own type object, its value parameters are compile-time constants within the body, and layout reads such as ```T.byteLength``` constant-fold per instantiation. An engine may share generated code between two instantiations only where the sharing is unobservable, and that specifically excludes any characteristic a program depends on for correctness: a ```[N].<T>``` extent that must be a fixed size, an ```inline``` operator that must expand. Sharing is therefore an implementation freedom, never a change in meaning. The cost is the one C++ and Rust accept openly - specialization multiplies code size - and the width-family patterns like ```write.<uint.<N>>``` in the [binary packet](examples/binarypacket.md) example are where it shows most.

A specialization is written after the expression it specializes, ```f.<uint8>(x)``` or ```new A.<uint16>()```, and never after ```?.```: an optional chain takes no type arguments.

The big picture of this section is to write out a near complete generics section to ensure types aren't implemented in a way that makes this awkward. It should be near seamless to introduce these as the main proposal relies on them in a few language feature areas.

```js
class A<T: type = uint8> {
  a: T;
  constructor(a: T) {
    this.a = a;
  }
}
const a: A = new A(5);           // A.<uint8>: the annotation names the default
const b = new A.<uint32>(1024);  // A.<uint32>
const c = new A((7 := uint16));  // A.<uint16>: T inferred from the argument
const d = new A(5);              // A.<number>: an untyped literal is a Number
```

In that example by default the field ```a``` is type ```uint8```, but the programmer foresaw someone might need to change this sometimes. Rather than hardcode this, the library exposes a generic parameter. A default is what a parameter takes when nothing else binds it: a bare ```A``` in a type position is ```A.<uint8>```, as is ```A.<>```, and so is a construction whose arguments reach ```T``` through nothing. Where an argument does reach ```T```, inference beats the default, exactly as it does in every language with both: ```new A(5)``` with no annotation is ```A.<number>```, because ```5``` is a Number, and ```new A((7 := uint16))``` is ```A.<uint16>```.

### Generic Application Syntax

Generic parameters are declared with ```<...>``` at declaration sites, as in ```class A<T: type> {}``` and ```function f<V: int32>() {}```. Every application of generic arguments, whether in a type or an expression, uses ```.<...>```, as in ```new A.<uint32>(1024)``` and ```f.<5>()```. The leading ```.``` removes the grammar ambiguity between generic argument lists and comparison operators, since ```a<b>(c)``` parses as chained comparisons today. Inside a generic argument list the tokens ```>>``` and ```>>>``` close nested lists, as in ```[].<[].<uint8>>```, rather than lexing as shift operators.

Operator declarations are the exception to the bare-```<...>``` at declaration sites: an operator's generic parameter list uses ```.<...>``` too, as in ```operator*.<T: type extends Ring>(rhs: T)```. The operator token may itself end in ```<``` or ```>```, so ```operator<.<D2: Dimensions>``` lexes unambiguously where ```operator< <D2: Dimensions>``` would collide with the ```<<``` token. An operator's list follows the rules of [Specialized Overloads](#specialized-overloads) below: ```operator+.<T: type>(rhs: T)``` declares a parameter, and ```operator+.<uint32>(rhs: uint32)``` fixes the argument.

### Declaring Parameters

Every entry of a declaration's parameter list states what kind of argument it takes. A type parameter is written ```T: type``` and a value parameter ```V: uint32```, and the two are one form: ```type``` is the domain of Type Objects, so a type parameter is a compile-time constant whose value is a type, which is what it already evaluates to in expression position. A bound follows the domain, ```T: type extends Ordered.<T>```, and a default follows both, ```T: type = uint8```. The kind is read from how the domain is written: ```type```, or ```[].<type>``` on a pack, takes types, and any other domain takes values. An alias of ```type``` is therefore refused with the spelling it stands for (```type Kind = type; function f<T: Kind>()``` is a TypeError asking for ```T: type```), which keeps the kind visible in the declaration itself, as Rust's ```const N: usize``` and C++'s ```typename T``` do, without resolving a name that may be declared later or imported. An alias of a value type is a value domain like the type it names: ```type Small = uint8; function f<N: Small>()``` takes a ```uint8``` constant.

```js
class Grid<T: type = float64, Rows: uint32 = 4, Cols: uint32 = 4> {}
function max<T: type extends Ordered.<T>>(a: T, b: T): T {}
function tuple<...Ts: [].<type>>(...values: Ts): Ts {}
function lanes<...I: [].<uint32>>() {}
function each<...Cs: [].<type> extends [].<Component>>(cb: (e: Entity, ref ...refs: Cs) => void): void {}
interface Iterator<W<_>: type, T: type, R: type = void, N: type = void> {}
```

A pack's annotation is its collection type, as a rest parameter's is: ```...I: [].<uint32>``` collects ```uint32``` constants and ```...Ts: [].<type>``` collects types. A bound on a pack of types bounds the tuple, so ```...Cs: [].<type> extends [].<Component>``` admits a pack whose every element is a ```Component```. A higher-kinded parameter keeps its holes, and its annotation says what an application of it yields: ```W<_>: type``` becomes a type when given one argument, and ```W``` alone is still not a type.

Roles are decided by the syntax, never by what a name resolves to. An entry with a ```:``` declares a parameter; a bare name is a *reference* to something already in scope, which in a declaration's list is how a [specialization](#specialization) names a fixed argument. So ```class Store<T: type> {}``` declares a family and ```class Store<uint32> {}``` specializes it, and neither reading depends on whether some ```T``` or ```uint32``` happens to be in scope. Earlier drafts of this document declared a type parameter with a bare name and treated a name that resolved to a type as a specialization, which made a declaration's meaning depend on its imports.

Three spellings are early errors that name their correction rather than alternatives:

- ```T: B```, where ```B``` is not a value domain - an interface, a class, ```object``` or an object shape, such as ```T: Ordered.<T>``` - declares a value parameter whose values would be objects, which a generic argument cannot be. The correction is ```T: type extends B```. Rust, Swift, and Kotlin all write a bound as ```T: B```, so this is the first mistake a reader from those languages will make, and it is caught where it is written rather than at some later application.
- ```T extends B``` with no domain is the same error with the same correction. One spelling per parameter means that deleting a bound cannot turn a parameter into a reference.
- A domain that admits both Type Objects and other values, such as ```V: any``` or ```V: type | uint32```, is refused, since an argument could bind it either way (see [Binding a value generic from an argument](#binding-a-value-generic-from-an-argument)).

A parameter may be named after a predefined type - ```uint32: type``` declares a parameter named ```uint32``` - and it then shadows that type by the ordinary lexical rule, in the signature and the body alike. Tooling warns, since it is rarely meant.

A generic function, method, or operator may not rebind one of its own type parameters at its own level: not as one of its parameters, not in a declaration at the top of its body, and not with a ```var``` anywhere in it, since the ```var``` hoists to the top. It is the rule an ordinary parameter already has (```function f(a) { let a; }``` is a SyntaxError), closed over ```var```, which for an ordinary parameter reuses the parameter's binding and for a type parameter would replace it for the whole body. A nested block, callback, or function may shadow it:

```js
function f<T: type>() { let T; }                // SyntaxError: T is a type parameter of f
function f<T: type>(T) {}                       // SyntaxError
function f<T: type>() { if (c) { var T; } }     // SyntaxError: the var hoists over T
function f<T: type>() { { let T = 5; } }        // OK: a nested block shadows it
class Pairs<K: type, V: type> {
  sum(m) { m.forEach((V, K) => { /* ... */ }); } // OK: the callback's own parameters
}
```

Generic function types, interface call signatures, and generic function and arrow expressions declare their parameters the same way: ```<T: type>(x: T) => T```.


### Named Generic Arguments

A type argument may be supplied by name, mirroring named call arguments. This matters where a generic has parameters with defaults, since supplying only a later one otherwise means repeating the earlier ones:

```js
type Grid<T: type = float64, Rows: uint32 = 4, Cols: uint32 = 4> = { cells: [Rows * Cols].<T> };

let a: Grid.<float64, 4, 8>;  // repeats the two defaults
let b: Grid.<Cols: 8>;        // says what differs
```

The separator is ```:```, matching named call arguments rather than the ```=``` of a parameter's default — in a declaration ```=``` means *the default if none is supplied*, and in an application it would mean *the value being supplied*, which are opposite senses at the same position.

Positional arguments come first, so a positional argument's meaning never depends on the names used after it. Among the names, order is free. A parameter a named list skips takes its default whether it sits before, between, or after the named ones, so ```Grid.<Cols: 8>``` and ```Grid.<T: float32, Cols: 8>``` both leave ```Rows``` at 4.

```js
let c: Grid.<float64, Cols: 8>;   // fine
// let d: Grid.<Cols: 8, float64>;  // does not compile: positional after named
```

Three mistakes are errors rather than guesses, and each for the same reason: a name that silently did something else would change what the program means without a diagnostic. A name that matches no parameter is refused rather than ignored, since a misspelling would otherwise take the parameter's default. The same rule refuses a name on a form that declares no parameters, such as ```[4].<uint8>```, whose extent and element the grammar does not name. A parameter may not be supplied twice, whether by two names or by a name and a position.

Names work at every application site — a type annotation, an expression, ```new```, a heritage clause, an explicit call — and through every declaration form: aliases, interfaces, classes, functions, methods, statics, generators. A method's argument list addresses the method's own parameters; the enclosing class's are bound at the instance and are not names a call can supply, so ```v.lane.<I: 1>()``` binds the method's ```I``` and ```v.lane.<N: 2>()``` is the unknown-name error even though the class declares an ```N```.

Two spellings that bind the same arguments are one specialization: ```Grid.<Cols: 8>```, ```Grid.<float64, Cols: 8>```, and ```Grid.<float64, 4, 8>``` intern to one type, compare ```===``` in expression position, and share one specialized body. Identity is the ordered bindings, never the spelling.

The standard library's generics carry the parameter names their signatures are written with — ```Map.<K, V>```, ```Set.<T>```, ```vector.<T, N>```, ```int.<N>``` — so ```Map.<V: uint8, K: string>``` means what it says and ```Map.<Z: uint8>``` is the unknown-name error, exactly as for a user declaration. Reflection exposes every declaration's type parameter names, which is what lets tooling complete them at an application site.

A named argument may address a variadic parameter, opening a run: see [Variadic Generic Parameters](#variadic-generic-parameters).

### Constraints

Often not just any type can be passed into the generic argument. Nearly every language has a constraint system to specify what interface(s) a type must implement.

```js
class A<T: type extends int> {
}
```
Here ```int``` is the constraint family matching any ```int.<N>```; likewise ```uint``` matches any ```uint.<N>```, and ```enum``` matches any enumeration - written ```enum.<TValue>``` to bound it to enumerations over a given underlying type, as the [decorators](decorators.md) reflection API does. These families are only usable as constraints, not as concrete types, since they don't specify a width (or, for ```enum```, a member set).

A ```static``` member is not parameterized by its class's type parameters, so it declares its own. A static that works over the class's element type takes that type as a fresh parameter - ```static of<T: type extends Ordered.<T>, S: Bound, E: Bound>(start: T, end: T): Range.<T, S, E>``` and ```static from<T: type>(values: [].<T>): SoA.<T>``` - rather than referring to a bare ```T``` that isn't in scope.

Simple syntax, but often you want to apply multiple interface constraints. TypeScript uses ```&```.

```js
class A<T: type extends B & C> {
}
```
I think that's sufficient and covers common use cases.

### Type Generic Parameters as Values

A generic type parameter is a type object, so in expression position it evaluates to the type it was specialized with, exactly as a type name does. This is what lets generic code key a collection or a registry on its own parameter:

```js
class EventBus {
	#channels = new Map.<type, any>();
	emit<T: type>(event: T) {
		this.#channels.get(T)?.push(event);
	}
	read<T: type>(): [].<T> {
		return this.#channels.get(T) ?? [];
	}
}
```

The main proposal's compile-time type expressions cover the type position counterpart, where an expression yielding a type object is a valid annotation.

### Value Type Generic Parameters

A value can be passed into generics like a function argument. The only caveat is they must be const and will be treated like const variables that are compiled away.

A generic value parameter may be declared with any primitive value type: the integer types, the float types, the decimal and rational types, ```boolean```, ```string```, and enum types. Two applications name the same specialization when their arguments are the same value under SameValue, the comparison ```Object.is``` performs. Reference values are not permitted as generic arguments, since specialization identity would then depend on object identity.

The one record-shaped domain is a metadata type of [primitive metadata](primitivemetadata.md), such as ```Dimensions``` or ```NumberBounds.<float32>```. Its values are the normalized, immutable metadata records its ```meta``` declaration accepts, compared by that facility's identity rather than by object identity, which is what lets ```decimal128.<{ currency: To }>``` below name one type however it is spelled. No other object type is a value domain: together with ```type```, the primitive value types, enumerations, and literal types and unions of them, these are the domains a generic parameter may declare, and the domains the correction ```T: type extends B``` is offered against.

An array or tuple of value types is also a value domain, so ```P: [].<string>``` takes a list of string constants. Its argument is identified by content, the tuple of its elements' literal types, never by the array's identity. An array of objects is not a value domain.

Two parameters of one list may not share a name, and a higher-kinded parameter's domain is written ```type```, a TypeError otherwise.

```js
class Buffer<Size: uint32, Name: string> {}
const a = new Buffer.<1024, 'input'>();

enum Endian: uint8 { Little, Big };
function read<E: Endian>(bytes: [].<uint8>): uint32 {}
read.<Endian.Big>(bytes);
```

String parameters make user-defined meta types practical, since a metadata object can be built from them:

```js
function convert<From: string, To: string>(
  amount: decimal128.<{ currency: From }>,
  rate: decimal128
): decimal128.<{ currency: To }> {
  return decimal128.<{ currency: To }>(amount * rate);
}
const euros = convert.<'USD', 'EUR'>(dollars, 0.86);
```

Float parameters are allowed for consistency, with two consequences worth knowing rather than prohibiting. SameValue makes ```A.<0>``` and ```A.<-0>``` distinct specializations, and it makes ```A.<NaN>``` a usable one, since ```Object.is(NaN, NaN)``` is true. Both follow from the identity rule above rather than being special cases, and a float argument must still be an exact compile-time constant like any other.

```js
class A<V: int32> {
  f(): int32 {
    return 2**V;
  }
}
const a = new A.<5>();
a.f();
```

In this example 5 is passed into the class creating essentially a unique implementation. For all purposes the first pass of the JIT would see it something like this:

```js
class A5 {
  f(): int32 {
    return 32;
  }
}
const a = new A5();
a.f();
```

#### Passing generic value type arguments

Since the goal is optimization it is impossible to pass a non-const to a generic value type. This is fine as long as the right expression is also completely const:

```js
function f() {
  const v = 5;
  const a = new A.<v>();
  const b = new A.<v>();
}
```

If this value needs to be defined outside of our function we can make it generic:
```js
function f<V: int32>() {
  const a = new A.<V>();
  const b = new A.<V>();
}
f.<5>();
```
This preserves the requirement that the argument needs to be const.

#### Binding a value generic from an argument

A value generic can also be bound implicitly, by a parameter whose type *is* that generic. The argument supplies the value, and because a value generic argument must be a compile-time constant, that argument must be one too:

```js
enum Component: uint8 { Transform, Velocity, Health };

function init<C: Component>(component: C, data: componentType(C)): ComponentInit {
  return { component, data } := ComponentInit;
}

init(Component.Transform, { x: 0, y: 0, rotation: 0 }); // C is bound to Component.Transform
// init(runtimeComponent, data); // TypeError: C's argument is not a constant
```

```C``` is fixed by the first argument, so the second parameter's type ```componentType(C)``` is a concrete type at the call and the object literal is checked against that specific component. This is the same specialization the explicit ```.<>``` form performs, reading the value off an argument instead of an angle-bracket list; a non-constant argument is a ```TypeError``` here for the same reason it is in a type position.

A type parameter is bound from an argument the same way, and its domain decides what is read. A parameter whose domain is ```type``` binds the argument's static type - ```function id<T: type>(x: T): T``` called as ```id((3 := uint16))``` binds ```T``` to ```uint16``` - while any other domain binds the argument's constant value, as ```C``` does above. A domain admitting both would have to guess between the two readings, which is why it is refused.

What ```V: int32``` binds, primitively, is a type: the literal type of the supplied constant over ```int32```. The value reading is the view through it, ```V```'s value being that literal's value, so one binding serves both positions and the checker holds one notion. This is also why an untyped literal argument satisfies a value-typed constraint directly: against ```W: uint32``` the literal ```4``` takes the literal type ```4``` over ```uint32``` rather than over ```number```.

#### Inferring from the expected type

When the surrounding context supplies an expected type — the annotation on a binding, a parameter's declared type, a function's declared return type, the target of an assignment, a field's type — and that type is an instantiation of the declaration being applied, its arguments bind the parameters in their positions:

```js
type Acceleration3 = vec3.<{ m: 1, s: -2 }>; // vec3<D: Dimensions> from primitive metadata

const gravity: Acceleration3 = vec3(0, -9.81, 0); // D inferred from Acceleration3
function fall(): Acceleration3 {
  return vec3(0, -9.81, 0); // D inferred from the return type
}

class Box<T: type> { v: T; constructor(v: T) { this.v = v; } }
const b: Box.<uint8> = new Box(1);  // T bound to uint8 from the annotation; the 1 is read at uint8
const c: Box.<uint8> = new Box("s"); // TypeError at the argument, as new Box.<uint8>("s") would be
```

The order is: explicit arguments first, then the expected type, then the value arguments, then defaults; a parameter left by all four is a ```TypeError``` naming it. A binding the expected type makes is fixed before the arguments are looked at and is treated exactly as an explicit one, so the arguments are checked against it. That order, and not the reverse, is what makes the ordinary spelling work: an untyped literal binds ```number``` when it is looked at first, and ```Box.<number>``` is not a ```Box.<uint8>```; read first, the annotation fixes ```T``` and the literal takes it, which is what writing ```new Box.<uint8>(1)``` does. The expected type binds only an instantiation of the same declaration (or a union containing one); an argument written ```any``` in it binds nothing in its position, and an interface, a supertype, or a structural type contributes nothing — this proposal's inference is positional.

#### Referring to a value parameter's type

A value generic's type is its declared constraint, so ```V: int32``` can be named ```int32``` directly. Where the type is inferred, or you would rather not repeat it, ```Reflect.typeOf(V)``` is a compile-time type expression that yields it, per the runtime type objects and compile-time type expression sections:

```js
class A<V: int32> {
  f(): Reflect.typeOf(V) { // int32
  }
}
```

No ```decltype```-style keyword is needed: ```Reflect.typeOf``` in type position is the general form, and it works for type parameters and ordinary bindings alike.

#### Constructing a generic class

A construction of a generic class always constructs a specialization — never the declaration, whose parameters are bound in no frame and whose typed fields would therefore check nothing. ```new Box(x)``` binds ```T``` by the same ladder a generic call uses (explicit arguments, the expected type, the value arguments, the defaults), and then proceeds as ```new Box.<...>(x)``` at those bindings: ```new.target``` is the specialization, the instance's prototype is the specialization's, and ```new Box((1 := uint8))``` and ```new Box.<uint8>(1)``` reach one class object and one type. Target-typed construction, ```const b: Box.<uint8> = new.(1)```, is the same rule with the name omitted; ```Reflect.construct(Box, args)``` binds the same way; and ```Reflect.construct(Box, args, Unrelated)``` is a ```TypeError```, since running the declaration's body against a foreign prototype would produce the open instance by another route.

A parameter that nothing reaches — no argument, no expected type, no formal annotated with it, no default — is a ```TypeError``` naming the parameter and the declaration, never ```any```:

```js
class Registry<T: type> { #entries = new Map.<string, T>(); }
new Registry();            // TypeError: T of Registry is not determined and has no default
new Registry.<Foo>();      // Registry.<Foo>
const r: Registry.<Foo> = new Registry(); // Registry.<Foo>, from the annotation
```

A ```Registry.<any>``` the program never named is an unchecked specialization the program cannot see it has, which is the failure this proposal exists to prevent; Rust and C++ ask for the argument here and their users write it. The same rule holds for a call of a generic function.

#### A parameter is opaque within its declaration

Inside the declaration that binds it, ```T``` is a subtype of itself and of its constraint and nothing else relates to it: a body is checked once, over its parameters, not per instantiation. So ```function f<T: type>(x: T) { let v: T = 5; }``` is a type error — a Number is not known to be a ```T```, which may be instantiated at ```string``` — and a field is the same position: ```class A<T: type> { value: T = 0; }``` and ```value: T = null``` are refused, and the value arrives through the constructor (```value: T; constructor(v: T) { this.value = v; }```) or is written at a type (```value: T | null = null```). This is Rust's rule, where an unconstrained ```T``` cannot be built from a literal at all, and TypeScript's. Checking the initializer at each instantiation instead (C++'s model) would move the error from the declaration to whichever ```new A.<string>()``` first cannot convert it.

#### Bare generic names and the family

A generic declaration's bare name in a type position — an annotation, a parameter or return type, a heritage clause, ```is```, a ```when``` pattern — names the application at its defaults, ```Box.<>```, and is a type error naming the parameter where one has no default. ```A```, ```A.<>``` and ```A.<uint8>``` are one type for a ```class A<T: type = uint8>```; ```let b: Box``` for a ```class Box<T: type>``` is refused; ```class S extends Box {}``` is refused, and ```class S<U: type> extends Box.<U>``` or ```extends Box.<uint8>``` is how it is written. There is no bare instantiation for a bare name to denote, since every construction yields a specialization, and a name that meant the family would make a default meaningless in a type position.

The family has a spelling of its own. An argument written ```any``` admits any instantiation in its position, as it already does for the collections, so ```Box.<any>``` is a Box of some element type and ```Pair.<any, string>``` a Pair whose first type is unknown and whose second is a string. A read through the wider view is ```any```; a store through it is checked against the instance's own field type at run time, which is what runtime types are for.

The one position where a bare generic name is the declaration rather than an application is as a type argument, where it binds a higher-kinded parameter. In expression position the name is the constructor, which stands for the declaration wherever a declaration is a value: ```Reflect.makeType({ kind: "generic", base: Box, arguments: [uint32] })```.

```instanceof``` sees the family through the constructor. A specialization is a distinct class object whose prototype chain does not pass through ```Box.prototype```, so ```x instanceof Box``` is extended: it is ```true``` when ```x``` is an instance of any specialization of ```Box```, or of a class extending one, while ```x instanceof Box.<uint8>``` is the ordinary prototype check against that specialization.

```js
new Box(1) instanceof Box;          // true
new Box.<uint8>(1) instanceof Box;  // true
new Box.<uint8>(1) instanceof Box.<uint16>; // false
```

### Specialization

A declaration whose list holds arguments rather than parameters specializes a family declared elsewhere. The primary declaration introduces the parameters callers bind; a specialization supplies a complete definition for the applications it matches:

```js
class Box<T: type, N: uint32 = 4> {}   // Primary: the family and its public parameters
class Box<uint32, 8> {}                 // Exact: Box.<uint32, 8> uses this body
class Box<const T, 16> {}               // Pattern: any element type with extent 16
partial class Box<_, 32> {}             // Wildcard: members added to every extent-32 Box
```

A specialization's list is positional, one entry per primary parameter, and each entry is a fixed argument, ```_```, or a ```const``` capture. A capture binds the argument in its position for the specialization's signature, heritage, and body - a type capture reads as its Type Object and a value capture as its constant - and ```const``` is the keyword a [pattern](patternmatching.md#binding-patterns) binds with, for the same reason: a bare name is a reference, and a binding says so. A capture takes its position's domain, so ```const T``` suffices where the primary declares ```T: type```. A written domain restates that domain and must match it, ```const N: uint32``` where the primary declares ```N: uint32```, as a Rust implementation header restates ```const N: usize```; the one exception is a metadata position, where the written meta type selects which metadata the capture binds, ```float32.<const D: Dimensions>```. Narrowing is spelled differently, ```const T extends Serializable``` for a type and a ```where``` clause for a value, so a written domain can never silently shrink what a specialization covers. Omitting a defaulted argument means its default, not a wildcard, so with the primary above ```class Box<uint32> {}``` specializes ```Box.<uint32, 4>```. A capture has no default and no variance, which are promises only a primary makes. And the list is positional even though an application may name arguments: ```class Box<uint32, N: 8> {}``` does not select ```N```, because ```name: domain``` in a declaration's list always declares a parameter, and a class has one primary. It is written ```class Box<uint32, 8> {}```.

A nested application within a specialization is a pattern over that constructor's parameters, and may use their names. Each line below is a separate example:

```js
class Store<Map.<string, const Element>> {}        // string-keyed maps, element captured
class Store<Map.<K: string, V: const Element>> {}  // the same pattern, by Map's own names
class Pair<const T, T> {}                          // equal component types
class Matrix<float32, const N: uint32, N> {}       // square float32 matrices
class Transfer<InputBuffer.<const T>, OutputBuffer.<T>> {}
class Width<uint.<const N>> {}                     // any uint.<N>, width captured
class ArrayCase<[const N].<const Element>> {}      // N takes the index type, uint64
```

A capture is declared once per header and used any number of times, and each further use is an equality the arguments must satisfy, under SameType for types and SameValue for values: ```Pair<const T, T>``` matches ```Pair.<uint8, uint8>``` and not ```Pair.<uint8, uint16>```. Declaring one name twice is an early error even when the two annotations agree, since a reader could not tell whether equality or two unrelated bindings was meant; TypeScript's repeated ```infer U``` combines its candidates silently, and this design declines to guess. A capture's scope is the whole header, the specialization's signature or heritage, and its body, so a use may precede the declaration, which is what keeps reordered names meaning one thing: ```Map.<V: T, K: const T>``` is ```Map.<K: const T, V: T>```. Only structural positions expose a component to capture - constructor arguments, tuple and array elements, fixed extents, width families - so a capture inside a type-building call is an error; a builder is evaluated forward from captures bound elsewhere instead. A pack is captured with ```...const Ts``` in a variadic position, reads as its tuple as any pack does, and a repeated ```...Ts``` requires the same length and elementwise identity: ```class Append<Tuple.<...const Ts>, Tuple.<...Ts>> {}```.

Selection binds the primary first and matches afterwards. Explicit arguments, the expected type, the value arguments, and the defaults bind the primary's parameters in their usual order, and the specializations are matched against that ordered binding, so ```Box.<uint32, N: 8>``` and ```Box.<uint32, 8>``` reach the same specialization and a capture's name never becomes an argument a caller can supply. The most specific matching specialization is selected by a finite comparison of fixed arguments, constructors, captures, repeated-capture equalities, and bounds whose inclusion is decidable. Where two match and neither is more specific - ```Pair<uint32, _>``` and ```Pair<const T, T>``` both match ```Pair.<uint32, uint32>``` - the application is a TypeError naming both, and a specialization for their intersection, ```Pair<uint32, uint32>```, resolves it. Declaration order never decides. Where none matches, the primary is used.

A specialization replaces the primary's body for the applications it matches, and keeps the primary's public contract: required members, constructors, variance, and reference permissions, so code written against ```Box.<T>``` stays correct whichever body runs. It may change its private representation, and layout is computed after selection, so generic code learns no size or offset from the primary's body that a specialization could invalidate. A specialization belongs to the declaration group that declares its primary: a program cannot replace an imported or intrinsic family's representation, though it may extend one additively. An alias family specializes the same way, ```type Storage<T: type> = T;``` with ```type Storage<boolean> = uint8;```, and generic code does not assume that an open ```Storage.<T>``` is ```T```, since a binding may select the other case. This is the rule Rust applies to a ```default``` associated type.

A ```partial class``` or ```partial interface``` target is a pattern too, and its members exist only on the matching instantiations. What a partial specialization does is *narrow* the family, always against the primary declaration's own bound rather than in place of it: a fixed argument is the narrowest narrowing, a constructor pattern such as ```uint.<const N>``` an intermediate one, and a bound on a capture the general case, admitting every instantiation whose argument satisfies both the primary's bound and the capture's. The [SIMD](simd.md) extension uses a fixed argument to put the mask operations on ```partial class vector<boolean1, const N: uint32>``` alone, leaving the general ```vector.<T, N>``` without them; the [ranges](ranges.md) extension uses a bound to put ```scale``` on ```partial interface RangeBounds<const T extends Scalable.<T>>```, leaving it off the ranges whose element type has an ordering but no arithmetic. A partial adds members only - it changes no layout, per the class extension rules - and a member that would collide with the primary declaration's is a TypeError as usual.

A member added this way is present on an instantiation only where the declaring module is loaded, which is true of every partial and is why a narrowing the language itself relies on belongs to the standard library rather than to a program.

### Specialized Overloads

Functions, methods, and operators already overload, so their generic lists follow the model C++ uses for function templates rather than for class templates: every generic list declares an overload of its own. Within it, a fixed argument or capture is an unnamed position and a ```name: domain``` entry is a named public parameter, and the two mix freely:

```js
class PacketWriter {
  write<boolean>(value: boolean): PacketWriter {}
  write<uint.<const N>>(value: uint.<N>): PacketWriter {}
  write<float32, maximum: float32, bits: uint32>(value: float32): PacketWriter {}
  write<float32, minimum: float32, maximum: float32, bits: uint32>(value: float32): PacketWriter {}
}

const w = new PacketWriter();
w.write.<boolean>(true);
w.write.<uint.<12>>(id);                       // N is 12
w.write.<float32, -1024, 1024, 18>(x);         // the four-parameter float32 overload
w.write.<float32, maximum: 1024, bits: 18>(x); // the three-parameter one, by name
// w.write.<float16>(h);                       // TypeError: no declared signature accepts it
```

Overload resolution ranks these as it ranks value signatures, a fixed position beating a parameter and a fixed parameter beating a pack. An application that no overload accepts is the ordinary "no declared signature" error, statically wherever the arguments are static. The float32 overloads with three and four positions have different public names, which is why they are separate overloads rather than cases of one generic signature. A capture at the top level of such an overload would observe nothing - it is a parameter in disguise - so it is an error that asks for ```name: domain```.

A same-named signature whose list holds only parameters is an *owner*. An overload whose list holds only fixed arguments and captures attaches to the owner whose list accepts it: each entry lands in one of the owner's parameters by position, and a fixed argument there is of that parameter's kind and admitted by its domain and bound, while a capture or ```_``` takes the parameter's domain. An overload the owner's contract does not admit, ```read<string>``` beside ```read<T: type extends uint>```, does not attach; it is a standalone case, reached by explicit application and never through the owner, since the owner's generic function value promises only what its bound admits. An attached overload borrows the owner's parameter names, so named and positional calls reach it alike, and where its signature is the owner's at those arguments it replaces the owner's body for them, keeps the owner's contract, and is reached through the generic function value too.

```js
function category<T: type>(): string { return "general"; }
function category<uint32>(): string { return "uint32"; }
function category<Map.<string, const Element>>(): string { return "string-keyed map"; }

category.<Map.<string, uint8>>(); // "string-keyed map"
category.<Map.<uint32, uint8>>(); // "general"
category.<T: uint32>();           // "uint32": the overload borrows the owner's name T
```

An owner may be declared without a body, as an abstract method is, and then an application that no attached overload matches is an error rather than a fallback. A generic body can forward an open argument to it, checked once against the owner's signature and selected per specialization:

```js
class PacketReader {
  read<T: type>(): T;
  read<boolean>(): boolean {}
  read<uint.<const N>>(): uint.<N> {}
}
class Accumulating<T: type> extends PacketReader {
  next(): T { return super.read.<T>(); } // checked against read<T: type>(): T
}
```

Without an owner, an open argument reaches a set of overloads only where its bound proves one of them applicable to every binding it admits. ```this.write.<LengthType>(...)```, for ```LengthType: type extends uint```, is checked against ```write<uint.<const N>>```, and any more specific overload that a particular binding selects must have that overload's signature at the binding. Anything else is a type error asking for an owner.

The [decorators](decorators.md) reflection API is a set of such overloads, one per reflection kind, each fixing the kind and naming the reflected class:

```js
getReflection<Reflect.Class, T: type>(): Reflect.ClassReflection;
getReflection<Reflect.ClassField, T: type>(name: string | symbol): Reflect.ClassFieldReflection;
```

The [binary packet](examples/binarypacket.md) writer and reader are the fuller worked example, with a dozen ```write``` and ```read``` overloads resolved this way.


### Variadic Generic Parameters

A generic parameter written with ```...``` collects any number of arguments, exactly as a rest parameter collects any number of values. Its annotation is the *collection* type, the same habit a rest parameter teaches — ```...args: [].<uint32>``` on one line, ```...I: [].<uint32>``` on the next, one idea with one spelling:

```js
// On the vector type:
swizzle<...I: [].<uint32>>(): vector.<T, I.length> where I.every(i => i < N);   // One source
shuffle<...I: [].<uint32>>(other: vector.<T, N>): vector.<T, I.length>;         // Two sources

a.swizzle.<0, 0, 0, 0>(); // I is [0, 0, 0, 0], I.length is 4
a.swizzle.<0, 1>();       // I is [0, 1], result narrows to vector.<T, 2>
b.shuffle.<0, 1, 4, 5>(c);
```

What a pack binds is a tuple: ```swizzle.<0, 1>``` binds ```I``` to the tuple type ```[0, 1]```, the same record the written type denotes. The ```...``` is only the collection marker — a pack *is* a tuple-typed parameter, so ```<...Ts>``` applied ```.<uint8, string>``` and ```<Ts extends [].<any>>``` applied ```.<[uint8, string]>``` bind the same thing, differing only in how the application spells it, and everything a tuple already does — spread, indexing in type position, reflection — applies to the binding with no new rule. The annotation admits everything a collection type can say: ```[].<uint32>``` for any count, ```[4].<uint8>``` for exactly four, ```[Entity, ...[].<Component>]``` for a shaped prefix, a bare ```...Ts``` or ```...Ts extends [].<Bound>``` for a pack of types.

In the body, a *value* pack reads as a frozen fixed-extent array, one per specialization: ```I.length``` and ```I[k]``` are compile-time constants an engine folds, ```I.every(i => i < N)``` is an ordinary array method over constants, the ```where``` clause above is checked once per specialization, and the same array answers every read — ```I === I``` across calls of one specialization. A *type* pack reads as the type object of its tuple, as any type parameter reads as its type. This is the scalar rule of the sections above applied pointwise: a value parameter reads as its value, so a tuple of values reads as their array.

Any number of packs may appear, anywhere in the list, and the binding rule is the rest-parameter rule verbatim: an argument run ends where the next argument fails the pack's element type, assignment is greedy with give-back for the parameters after, and names override everything. A parameter after a pack is reachable positionally — its neighbours' bounds are what stop the runs — or by name:

```js
function two<...A: [].<uint32>, ...B: [].<string>>() {}
two.<0, 1, 'x'>();                 // A is [0, 1], B is ['x'] — the bounds split the runs
function q<...I: [].<uint32>, N: uint32 = 4>() {}
q.<0, 1, 2>();                     // I takes all three, N defaults
q.<0, 1, 2, N: 8>();               // the name reaches past the greedy pack
q.<I: 0, 1, N: 8>();               // a named pack OPENS A RUN: I is [0, 1]
```

Two packs with no usable boundary between them are an error, the same sentence rest parameters use: a pack with no element bound admits everything, so a second pack adjacent to it could never receive an argument positionally. Two adjacent packs of the *same* element type are allowed — the first takes what both admit, and the parameter after them says where the run must stop — and are as unrecommended here as ```f(...a: [].<uint32>, ...b: [].<uint32>, c)``` is for values; names are the readable spelling of both. A pack whose bound reads a parameter declared at or after the first pack cannot help split — such a bound admits everything *for the split* and is checked in full once the assignment is fixed, which is the same conservatism argument binding already applies to a parameter type that mentions a not-yet-bound type parameter.

One consequence to know rather than trip on: a literal is a type too, so an unbounded *type* parameter admits one positionally. In ```<...Ts, N: uint32>``` applied ```.<A, B, 4>```, the greedy pack yields the last argument to ```N``` by arity — but in ```<T, ...I: [].<uint32>>``` applied ```.<0, 1>```, the fixed ```T``` takes the literal type ```0``` and ```I``` gets ```[1]```. Writing ```N: 4``` or ```T: uint8``` says what was meant; the positional reading is deterministic either way.

A pack may declare a tuple default — ```<...I: [].<uint32> = [0, 1, 2]>``` — and counts as defaulted for the trailing-defaults rule, which restarts after it, so ```<...A: [].<uint32>, N: uint32>``` is legal with ```N``` required. Variance annotates a pack as it does a scalar parameter and applies per element position.

**Spread.** ```...``` in an argument list splices a tuple — a written one, an alias of one, or a pack in scope — element by element before anything binds, and the spellings intern to one specialization: ```swizzle.<...Pair>``` *is* ```swizzle.<0, 1>``` for ```type Pair = [0, 1]```. The operand's length must be statically known; a dynamic ```[].<uint32>``` has none and is refused. The same token splices inside a tuple type, ```[...Ts, T]```, and a pack forwards into another application the same way, ```Map.<...Ts>```.

**Deriving from a pack.** A value pack is an array in the body, so transforming one is plain code — ```I.map(i => i * 2)``` inside a compile-time function. A type pack restructures with spread and transforms per element with a [type-programming](typeprogramming.md) builder, written once and named, rather than with a mapped-type sub-language:

```js
type promisesOf(Ts: type): type {
  const elements = Reflect.getReflection(Ts).elements.map((e) => ({ type: Promise.<e.type> }));
  return Reflect.makeType({ kind: 'tuple', elements });
}
```

One modifier distributes rather than maps: ```ref``` on a rest parameter makes each parameter the rest collects a ```ref```, in declarations and in function types alike. This is what types a query callback without any builder at all — the pack appears directly:

```js
each<...Cs: [].<type> extends [].<Component>>(cb: (e: Entity, ref ...refs: Cs) => void): void;
world.each((e, ref t: Transform, ref v: Velocity) => { t.x += v.x; });  // Cs inferred: [Transform, Velocity]
```

References are not values, so a ```ref``` rest binds no array. Its name is usable in exactly three forms, each a direct use of one collected reference and none a store: ```...name``` forwarded into another call's ref-rest position, ```name[k]``` with a compile-time-constant ```k```, and ```name.length```. Everything else — assigning it, passing it whole, a runtime index — is the escape error a single ```ref``` parameter already has, applied per element. The [ECS example](examples/ecs.md) is the worked case.

**Binding a pack from arguments.** Inference reaches a pack wherever it reaches a scalar — the ladder is the same, with no carve-outs. A rest parameter typed by the pack binds it from the call's values (each a compile-time constant, as every value-generic argument is); a whole-tuple parameter binds it from one tuple; a written tuple pattern matches into it with the same greedy rule, ```pairUp<T: type, ...Rest: [].<type>>(p: [T, ...Rest])```; a callback's signature binds it structurally, as ```each``` above shows; and a builder standing between the pack and the arguments inverts only through a declared ```@inverse```, never by search - the inverse of a builder over a pack receives the tuple of the collected arguments' types and returns the pack's tuple, so a builder written elementwise inverts elementwise. A spread argument binds a pack only when its length is static. Most inversions are better avoided than declared: declare the pack as *what the caller passes* and derive the rest forward —

```js
function all<...Ps: [].<type> extends [].<PromiseLike>>(...ps: Ps): Promise.<awaitedAll(Ps)> {}
```

— which needs only structural matching, and, types being structural, yields the same types the inverted spelling would.

**Growth is metered.** A specialization chain through a tuple, ```read<T: type>(): Reader.<[...Ts, T]>``` in the [binary packet](examples/binarypacket.md) example, is finite per call site and free. A function that specializes *itself* over a longer pack recurses without end and is stopped by the compile-time evaluation budget, as Rust stops the same program with its recursion limit — a type error naming the budget, never a stack overflow.

### Generic Function Types and Signatures

A function type may declare type parameters, and an interface's call and method signatures may too. This is the type of a generic function, and it is what makes a generic strategy or bus an ordinary interface:

```js
let g: <T: type>(x: T) => T;
interface Mapper { <T: type>(x: T): T; }
interface Bus { on<T extends Event>(name: string, h: (e: T) => void): void; }
```

Two generic signatures are the same type when they are the same up to renaming: parameter names are carried for tooling and named arguments, never compared, so ```<T: type>(x: T) => T``` and ```<U: type>(x: U) => U``` are one type, while a constraint, a default, a variance annotation, or the parameter order makes two. A class satisfies a generic interface signature by shape under the same reading, so an implementation is free to pick its own parameter names.

Assignability follows one question — could a caller reading the target's type be misled? A generic function is assignable *to* a concrete signature by instantiation, so ```const g: (uint8) => uint8 = id``` holds ```id.<uint8>```: the specialization happens where the assignment is written, and every call through ```g``` is a direct call of one body. A generic function is assignable to a generic signature that is no more general than it is. A concrete function is *not* assignable to a generic signature — ```<T: type>(x: T) => T``` promises every ```T```, and ```(x: uint8) => uint8``` would return a ```uint8``` for a ```string``` — with the untyped catch-all as the one exception it already is everywhere.

A specialization is a value. ```id.<uint8>``` in expression position is a function object, interned per function and ordered bindings, so two spellings are one value — ```id.<uint8> === id.<uint8>```, a ```Map``` keyed on it round-trips, ```removeEventListener``` works — and ```arr.map(id.<uint8>)``` runs the specialized body. Its ```where``` clauses run once, at specialization. The bare name keeps the generic signature: ```Reflect.typeOf(id)``` is ```<T: type>(x: T) => T```; ```Reflect.typeOf(id.<uint8>)``` is ```(uint8) => uint8```.

Calling through a binding of *generic* function type — ```let m: Mapper = id; m.<uint8>(1)``` — selects the specialization of whatever the binding holds, by that callee's identity and the call's bindings. Which body runs is not known where the call is written; it is not known for any indirect call either, and the cost is that of one: an indirect call plus a cache keyed on callee and bindings, the dispatch C# performs for generic virtual methods. The type checks are still compiled away; only the callee is dynamic, and only where it already was. Crossing into a *concrete* signature — the common case — moved that decision to the assignment above and costs the call nothing.

An overload set may mix concrete signatures, generic signatures, and packs. A generic signature is viable when its parameters bind, from explicit arguments or by inference; ranking uses the instantiated signature; and where more than one is viable, a concrete position beats a type parameter and a fixed parameter beats a pack, so ```route(e: Click)```, ```route<T: type extends Event>(e: T)```, and ```route<...Es: [].<type> extends [].<Event>>(...es: Es)``` layer from most to least specific. Two signatures that are the same up to renaming are one signature written twice, and are reported as the duplicate they are.

### Using Value Type Classes as parameters

WIP, is this necessary? Is it possible? What does it allow? They are non-dynamic, just like ```int32```, so they should be alright.

### Decorator Generics

Generic decorators work as expected, and the [decorators](decorators.md) extension specifies them in full: decorator factories parameterized by type and value generics, together with the specialized-overload reflection API shown above. See that document for worked examples.
