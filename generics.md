# Generics

The goal of generics would be to represent compile-time or at the least JIT optimized codepaths. In this way they're more similar to C++ templates. In a type system they allow simple generic classes for specializing for types.

Concretely, the semantics are those of full specialization. Each application - ```A.<uint8>```, ```A.<uint16>``` - is a distinct type with its own type object, its value parameters are compile-time constants within the body, and layout reads such as ```T.byteLength``` constant-fold per instantiation. An engine may share generated code between two instantiations only where the sharing is unobservable, and that specifically excludes any characteristic a program depends on for correctness: a ```[N].<T>``` extent that must be a fixed size, an ```inline``` operator that must expand. Sharing is therefore an implementation freedom, never a change in meaning. The cost is the one C++ and Rust accept openly - specialization multiplies code size - and the width-family patterns like ```write.<uint.<N>>``` in the [binary packet](examples/binarypacket.md) example are where it shows most.

A specialization is written after the expression it specializes, ```f.<uint8>(x)``` or ```new A.<uint16>()```, and never after ```?.```: an optional chain takes no type arguments.

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

### Named Generic Arguments

A type argument may be supplied by name, mirroring named call arguments. This matters where a generic has parameters with defaults, since supplying only a later one otherwise means repeating the earlier ones:

```js
type Grid<T = float64, Rows: uint32 = 4, Cols: uint32 = 4> = { cells: [Rows * Cols].<T> };

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
class A<T extends int> {
}
```
Here ```int``` is the constraint family matching any ```int.<N>```; likewise ```uint``` matches any ```uint.<N>```, and ```enum``` matches any enumeration - written ```enum.<TValue>``` to bound it to enumerations over a given underlying type, as the [decorators](decorators.md) reflection API does. These families are only usable as constraints, not as concrete types, since they don't specify a width (or, for ```enum```, a member set).

A ```static``` member is not parameterized by its class's type parameters, so it declares its own. A static that works over the class's element type takes that type as a fresh parameter - ```static of<T, S: Bound, E: Bound>(start: T, end: T): Range.<T, S, E>``` and ```static from<T>(values: [].<T>): SoA.<T>``` - rather than referring to a bare ```T``` that isn't in scope.

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

A ```partial class``` or ```partial interface``` may specialize a generic parameter the same way, so its members exist only on the matching instantiations. What a specialization does in general is *narrow* the parameter, always against the primary declaration's own bound rather than in place of it: a concrete type is the narrowest narrowing, a constraint family is an intermediate one, and an interface bound is the general case, admitting every instantiation whose argument satisfies the primary's constraint and the narrowing both. The [SIMD](simd.md) extension uses a concrete narrowing to put the mask operations on ```partial class vector<boolean1, N: uint32>``` alone, leaving the general ```vector<T, N>``` without them; the [ranges](ranges.md) extension uses an interface narrowing to put ```scale``` on ```partial interface RangeBounds<T: Scalable.<T>>```, leaving it off the ranges whose element type has an ordering but no arithmetic. A specialized ```partial``` adds members only - it changes no layout, per the class extension rules - and a member that would collide with the primary declaration's is a TypeError as usual.

A member added this way is present on an instantiation only where the declaring module is loaded, which is true of every partial and is why a narrowing the language itself relies on belongs to the standard library rather than to a program.

Specialization mixes freely with open parameters. A selector type followed by an open one is the shape the [decorators](decorators.md) reflection API uses throughout, one overload per reflection kind:

```js
getReflection<Reflect.Class, T>(): Reflect.ClassReflection;
getReflection<Reflect.ClassField, T>(name: string | symbol): Reflect.ClassFieldReflection;
```

The first argument (```Reflect.Class```, ```Reflect.ClassField```, and so on) selects the overload; the second, ```T```, is the class being reflected. The [binary packet](examples/binarypacket.md) writer and reader are the fuller worked example, with a dozen ```write``` and ```read``` overloads resolved this way.

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
each<...Cs extends [].<Component>>(cb: (e: Entity, ref ...refs: Cs) => void): void;
world.each((e, ref t: Transform, ref v: Velocity) => { t.x += v.x; });  // Cs inferred: [Transform, Velocity]
```

References are not values, so a ```ref``` rest binds no array. Its name is usable in exactly three forms, each a direct use of one collected reference and none a store: ```...name``` forwarded into another call's ref-rest position, ```name[k]``` with a compile-time-constant ```k```, and ```name.length```. Everything else — assigning it, passing it whole, a runtime index — is the escape error a single ```ref``` parameter already has, applied per element. The [ECS example](examples/ecs.md) is the worked case.

**Binding a pack from arguments.** Inference reaches a pack wherever it reaches a scalar — the ladder is the same, with no carve-outs. A rest parameter typed by the pack binds it from the call's values (each a compile-time constant, as every value-generic argument is); a whole-tuple parameter binds it from one tuple; a written tuple pattern matches into it with the same greedy rule, ```pairUp<T, ...Rest>(p: [T, ...Rest])```; a callback's signature binds it structurally, as ```each``` above shows; and a builder standing between the pack and the arguments inverts only through a declared ```@inverse```, never by search. A spread argument binds a pack only when its length is static. Most inversions are better avoided than declared: declare the pack as *what the caller passes* and derive the rest forward —

```js
function all<...Ps extends [].<PromiseLike>>(...ps: Ps): Promise.<awaitedAll(Ps)> {}
```

— which needs only structural matching, and, types being structural, yields the same types the inverted spelling would.

**Growth is metered.** A specialization chain through a tuple, ```read<T>(): Reader.<[...Ts, T]>``` in the [binary packet](examples/binarypacket.md) example, is finite per call site and free. A function that specializes *itself* over a longer pack recurses without end and is stopped by the compile-time evaluation budget, as Rust stops the same program with its recursion limit — a type error naming the budget, never a stack overflow.

### Generic Function Types and Signatures

A function type may declare type parameters, and an interface's call and method signatures may too. This is the type of a generic function, and it is what makes a generic strategy or bus an ordinary interface:

```js
let g: <T>(x: T) => T;
interface Mapper { <T>(x: T): T; }
interface Bus { on<T extends Event>(name: string, h: (e: T) => void): void; }
```

Two generic signatures are the same type when they are the same up to renaming: parameter names are carried for tooling and named arguments, never compared, so ```<T>(x: T) => T``` and ```<U>(x: U) => U``` are one type, while a constraint, a default, a variance annotation, or the parameter order makes two. A class satisfies a generic interface signature by shape under the same reading, so an implementation is free to pick its own parameter names.

Assignability follows one question — could a caller reading the target's type be misled? A generic function is assignable *to* a concrete signature by instantiation, so ```const g: (uint8) => uint8 = id``` holds ```id.<uint8>```: the specialization happens where the assignment is written, and every call through ```g``` is a direct call of one body. A generic function is assignable to a generic signature that is no more general than it is. A concrete function is *not* assignable to a generic signature — ```<T>(x: T) => T``` promises every ```T```, and ```(x: uint8) => uint8``` would return a ```uint8``` for a ```string``` — with the untyped catch-all as the one exception it already is everywhere.

A specialization is a value. ```id.<uint8>``` in expression position is a function object, interned per function and ordered bindings, so two spellings are one value — ```id.<uint8> === id.<uint8>```, a ```Map``` keyed on it round-trips, ```removeEventListener``` works — and ```arr.map(id.<uint8>)``` runs the specialized body. Its ```where``` clauses run once, at specialization. The bare name keeps the generic signature: ```Reflect.typeOf(id)``` is ```<T>(x: T) => T```; ```Reflect.typeOf(id.<uint8>)``` is ```(uint8) => uint8```.

Calling through a binding of *generic* function type — ```let m: Mapper = id; m.<uint8>(1)``` — selects the specialization of whatever the binding holds, by that callee's identity and the call's bindings. Which body runs is not known where the call is written; it is not known for any indirect call either, and the cost is that of one: an indirect call plus a cache keyed on callee and bindings, the dispatch C# performs for generic virtual methods. The type checks are still compiled away; only the callee is dynamic, and only where it already was. Crossing into a *concrete* signature — the common case — moved that decision to the assignment above and costs the call nothing.

An overload set may mix concrete signatures, generic signatures, and packs. A generic signature is viable when its parameters bind, from explicit arguments or by inference; ranking uses the instantiated signature; and where more than one is viable, a concrete position beats a type parameter and a fixed parameter beats a pack, so ```route(e: Click)```, ```route<T extends Event>(e: T)```, and ```route<...Es extends [].<Event>>(...es: Es)``` layer from most to least specific. Two signatures that are the same up to renaming are one signature written twice, and are reported as the duplicate they are.

### Using Value Type Classes as parameters

WIP, is this necessary? Is it possible? What does it allow? They are non-dynamic, just like ```int32```, so they should be alright.

### Decorator Generics

Generic decorators work as expected, and the [decorators](decorators.md) extension specifies them in full: decorator factories parameterized by type and value generics, together with the specialized-overload reflection API shown above. See that document for worked examples.
