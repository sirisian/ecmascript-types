# Generics

The goal of generics would be to represent compile-time or at the least JIT optimized codepaths. In this way they're more similar to C++ templates. In a type system they allow simple generic classes for specializing for types.

Concretely, the semantics are those of full specialization. Each application - ```A.<uint8>```, ```A.<uint16>``` - is a distinct type with its own type object, its value parameters are compile-time constants within the body, and layout reads such as ```T.byteLength``` constant-fold per instantiation. An engine may share generated code between two instantiations only where the sharing is unobservable, and that specifically excludes any characteristic a program depends on for correctness: a ```[N].<T>``` extent that must be a fixed size, an ```inline``` operator that must expand. Sharing is therefore an implementation freedom, never a change in meaning. The cost is the one C++ and Rust accept openly - specialization multiplies code size - and the width-family patterns like ```write.<uint.<N>>``` in the [binary packet](examples/binarypacket.md) example are where it shows most.

The big picture of this section is to write out a near complete generics section to ensure types aren't implemented in a way that makes this awkward. It should be near seamless to introduce these as the main proposal relies on them in a few language feature areas.

```js
class A<T = uint8> {
  a: T;
  constructor(a: T) {
    this.a = a;
  }
}
const a = new A(5);
const b = new A.<uint32>(1024);
```

In that example by default the field ```a``` is type ```uint8```, but the programmer foresaw someone might need to change this sometimes. Rather than hardcode this, the library exposes a generic parameter.

### Generic Application Syntax

Generic parameters are declared with ```<...>``` at declaration sites, as in ```class A<T> {}``` and ```function f<V: int32>() {}```. Every application of generic arguments, whether in a type or an expression, uses ```.<...>```, as in ```new A.<uint32>(1024)``` and ```f.<5>()```. The leading ```.``` removes the grammar ambiguity between generic argument lists and comparison operators, since ```a<b>(c)``` parses as chained comparisons today. Inside a generic argument list the tokens ```>>``` and ```>>>``` close nested lists, as in ```[].<[].<uint8>>```, rather than lexing as shift operators.

Operator declarations are the exception to the bare-```<...>``` at declaration sites: an operator's generic parameter list uses ```.<...>``` too, as in ```operator*.<T extends Ring>(rhs: T)```. The operator token may itself end in ```<``` or ```>```, so ```operator<.<D2: Dimensions>``` lexes unambiguously where ```operator< <D2: Dimensions>``` would collide with the ```<<``` token.

### Constraints

Often not just any type can be passed into the generic argument. Nearly every language has a constraint system to specify what interface(s) a type must implement.

```js
class A<T extends int> {
}
```
Here ```int``` is the constraint family matching any ```int.<N>```; likewise ```uint``` matches any ```uint.<N>```, and ```enum``` matches any enumeration - written ```enum.<TValue>``` to bound it to enumerations over a given underlying type, as the [decorators](decorators.md) reflection API does. These families are only usable as constraints, not as concrete types, since they don't specify a width (or, for ```enum```, a member set).

A ```static``` member is not parameterized by its class's type parameters, so it declares its own. A static that works over the class's element type takes that type as a fresh parameter - ```static of<T, I: Interval>(start: T, end: T): Range.<T, I>``` and ```static from<T>(values: [].<T>): SoA.<T>``` - rather than referring to a bare ```T``` that isn't in scope.

Simple syntax, but often you want to apply multiple interface constraints. TypeScript uses ```&```.

```js
class A<T extends B & C> {
}
```
I think that's sufficient and covers common use cases.

### Type Generic Parameters as Values

A generic type parameter is a type object, so in expression position it evaluates to the type it was specialized with, exactly as a type name does. This is what lets generic code key a collection or a registry on its own parameter:

```js
class EventBus {
	#channels = new Map.<type, any>();
	emit<T>(event: T) {
		this.#channels.get(T)?.push(event);
	}
	read<T>(): [].<T> {
		return this.#channels.get(T) ?? [];
	}
}
```

The main proposal's compile-time type expressions cover the type position counterpart, where an expression yielding a type object is a valid annotation.

### Value Type Generic Parameters

A value can be passed into generics like a function argument. The only caveat is they must be const and will be treated like const variables that are compiled away.

A generic value parameter may be declared with any primitive value type: the integer types, the float types, the decimal and rational types, ```boolean```, ```string```, and enum types. Two applications name the same specialization when their arguments are the same value under SameValue, the comparison ```Object.is``` performs. Reference values are not permitted as generic arguments, since specialization identity would then depend on object identity.

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

What ```V: int32``` binds, primitively, is a type: the literal type of the supplied constant over ```int32```. The value reading is the view through it, ```V```'s value being that literal's value, so one binding serves both positions and the checker holds one notion. This is also why an untyped literal argument satisfies a value-typed constraint directly: against ```W: uint32``` the literal ```4``` takes the literal type ```4``` over ```uint32``` rather than over ```number```.

#### Inferring from the expected type

When an application leaves a generic parameter unpinned and the surrounding context supplies an expected type, the parameter is inferred from it — from the annotation on a binding, or from a function's declared return type:

```js
type Acceleration3 = vec3.<{ m: 1, s: -2 }>; // vec3<D: Dimensions> from primitive metadata

const gravity: Acceleration3 = vec3(0, -9.81, 0); // D inferred from Acceleration3
function fall(): Acceleration3 {
  return vec3(0, -9.81, 0); // D inferred from the return type
}
```

Inference runs after explicit arguments and argument-bound parameters, and it is a ```TypeError``` when the context fixes no type.

#### Referring to a value parameter's type

A value generic's type is its declared constraint, so ```V: int32``` can be named ```int32``` directly. Where the type is inferred, or you would rather not repeat it, ```Reflect.typeOf(V)``` is a compile-time type expression that yields it, per the runtime type objects and compile-time type expression sections:

```js
class A<V: int32> {
  f(): Reflect.typeOf(V) { // int32
  }
}
```

No ```decltype```-style keyword is needed: ```Reflect.typeOf``` in type position is the general form, and it works for type parameters and ordinary bindings alike.

### Specialized Overloads

A generic parameter list may pin some parameters to concrete types and leave others open, and a function or method may declare several such signatures. This is how one name serves many types with no ```any``` in sight: the compiler selects the signature whose generic parameter list matches the arguments at the call, the same resolution methods and functions already use on their value parameters.

A parameter is *specialized* when it names a concrete type rather than a fresh identifier, and its signature applies only when that generic argument is that type:

```js
class PacketWriter {
  write<boolean>(value: boolean): PacketWriter {}   // Selected by write.<boolean>(...)
  write<float32>(value: float32): PacketWriter {}
  write<float32, minimum: float32, maximum: float32, bits: uint32>(value: float32): PacketWriter {}
}

const w = new PacketWriter();
w.write.<boolean>(true);
w.write.<float32, -1024, 1024, 18>(x); // The four-parameter float32 overload
```

The first generic argument selects among the overloads; the remaining parameters may be open type parameters or value generics, so ```write.<float32, -1024, 1024, 18>``` picks the signature whose first parameter is ```float32``` and whose next three are the value generics ```minimum```, ```maximum```, and ```bits```.

A specialized parameter may be a *constraint family* rather than a single type, matching any member of the family and binding its variable:

```js
write<uint<N: uint32>>(value: uint.<N>): PacketWriter {}                   // Any uint.<N>, N bound
write<uint<N: uint32>, minimum: uint32, maximum: uint32>(value: uint.<N>) {}
```

```uint<N: uint32>``` matches ```uint.<8>```, ```uint.<12>```, and so on, binding ```N``` to the width for the parameter and body. This is the declaration-site counterpart of the ```extends uint``` constraint: ```extends``` bounds an open parameter, while a family in the specialization position both selects the overload and binds its width.

A ```partial class``` may specialize a generic parameter the same way, to a concrete type or a constraint family, so its members exist only on the matching instantiations. The [SIMD](simd.md) extension uses this to put the mask operations on ```partial class vector<boolean1, N: uint32>``` alone, leaving the general ```vector<T, N>``` without them. A specialized ```partial``` adds members only - it changes no layout, per the class extension rules - and a member that would collide with the primary declaration's is a TypeError as usual.

Specialization mixes freely with open parameters. A selector type followed by an open one is the shape the [decorators](decorators.md) reflection API uses throughout, one overload per reflection kind:

```js
getReflection<Reflect.Class, T>(): Reflect.ClassReflection;
getReflection<Reflect.ClassField, T>(name: string | symbol): Reflect.ClassFieldReflection;
```

The first argument (```Reflect.Class```, ```Reflect.ClassField```, and so on) selects the overload; the second, ```T```, is the class being reflected. The [binary packet](examples/binarypacket.md) writer and reader are the fuller worked example, with a dozen ```write``` and ```read``` overloads resolved this way.

### Variadic Generic Parameters

A generic parameter written ```...Name: [].<T>``` collects zero or more constant arguments into a tuple whose element type is ```T``` and whose ```length``` is a compile-time constant. This is what lets a permutation or projection take its indices as generic arguments rather than a runtime array:

```js
// On the vector type:
swizzle<...I: [].<uint32>>(): vector.<T, I.length>;                     // One source
shuffle<...I: [].<uint32>>(other: vector.<T, N>): vector.<T, I.length>; // Two sources

a.swizzle.<0, 0, 0, 0>(); // I is [0, 0, 0, 0], I.length is 4
a.swizzle.<0, 1>();        // I is [0, 1], result narrows to vector.<T, 2>
b.shuffle.<0, 1, 4, 5>(c);
```

The pack is a tuple, so ```I.length``` is a value generic and the result type ```vector.<T, I.length>``` depends on how many indices were passed. Bounds on the individual values use a ```where``` clause, checked at compile time because every element is a constant:

```js
swizzle<...I: [].<uint32>>(): vector.<T, I.length> where I.every(i => i < N);
```

A variadic parameter is the last in its list, and its arguments are the trailing ones once any fixed parameters are matched. The same form expresses a query API that takes a variable number of component types, ```each<...Cs: [].<Component>>```, with the callback's parameters derived from the pack.

### Using Value Type Classes as parameters

WIP, is this necessary? Is it possible? What does it allow? They are non-dynamic, just like ```int32```, so they should be alright.

### Decorator Generics

Generic decorators work as expected, and the [decorators](decorators.md) extension specifies them in full: decorator factories parameterized by type and value generics, together with the specialized-overload reflection API shown above. See that document for worked examples.
