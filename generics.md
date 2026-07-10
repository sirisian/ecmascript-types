# Generics

The goal of generics would be to represent compile-time or at the least JIT optimized codepaths. In this way they're more similar to C++ templates. In a type system they allow simple generic classes for specializing for types.

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

### Constraints

Often not just any type can be passed into the generic argument. Nearly every language has a constraint system to specify what interface(s) a type must implement.

```js
class A<T extends int> {
}
```
Here ```int``` is the constraint family matching any ```int.<N>```; likewise ```uint``` matches any ```uint.<N>```. These families are only usable as constraints, not as concrete types, since they don't specify a width.

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

#### Syntax to get the value type's type

To increase readability often you want to define the type once and refer to the parameter's type. Something like ```decltype```, but a better keyword. Or is that good?
```js
class A<V: int32> {
  f(): type(V) {
  }
}
```

### Using Value Type Classes as parameters

WIP, is this necessary? Is it possible? What does it allow? They are non-dynamic, just like ```int32```, so they should be alright.

### Decorator Generics

WIP: Generic decorators should work as expected. Write a few examples.
