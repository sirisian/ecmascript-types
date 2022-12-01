# Generics

The goal of generics would be represent compile-time or at the least JIT optimized codepaths. In this way they're more similar to C++ templates. In a type system they allow simple generic classes for specializing for types.

The big pictue of this section is to write out a near complete generics section to ensure types aren't implemented in a way that makes this awkward. It should be near seamless to introduce these as the main proposal relies on them in a few language feature areas.

```js
class A<T = uint8> {
  a:T;
  constructor(a:T) {
    this.a = a;
  }
}
const a = new A(5);
const b = new A<uint32>(1024);
```

In that example by default the field ```a``` is type ```uint8```, but the programmer foresaw someone might need to change this sometimes. Rather than hardcode this, the library exposes a generic parameter.

### Constraints

Often not just any type can be passed into the generic argument. Nearly every language has a constraint system to specify what interface(s) a type must implement.

One possible issue with my current type system is there's no base/interface ```int```, ```uint```, or ```float```. I don't think it's a huge issue since one can write ```int8|int16|int32|int64```. In any case if you wanted to say a ```T``` needed to be an integer.

```js
class A<T extends int8|int16|int32|int64> {
}
```
Simple syntax, but often you want to apply multiple interface constraints. TypeScript uses ```&```.

```js
class A<T extends B & C> {
}
```
I think that's sufficient and covers all use cases.

### Value Type Generic Parameters

A value can be passed into generics like a function argument. The only caveat is they must be const and will be treated like const variables that are compiled away.

```js
class A<V:int32> {
  f():int32 {
    return 2**V;
  }
}
const a = new A<5>();
a.f();
```

In this examples 5 is passed into the class creating essentially a unique implementation. For all purposes the first pass of the JIT would see it something like this:

```js
class A5 e {
  f():int32 {
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
  const a = new A<v>();
  const b = new A<v>();
}
```

If this value needs to be defined outside of our function we can make it generic:
```js
function f<V:int32>() {
  const a = new A<V>();
  const b = new A<V>();
}
f<5>();
```
This preserves the requirement that the argument needs to be const.

#### Syntax to get the value type's type

To increase readability often you want to define the type once and refer to the parameter's type. Something like ```decltype```, but a better keyword. Or is that good?
 ```js
class A<V:int32> {
  f():type(V) {
  }
}
```

### Using Value Type Classes as parameters

WIP, is this necessary? Is it possible? What does it allow? They are non-dynamic, just like ```int32```, so they should be alright.

### Decorator Generics

WIP Generic decorators should work as expected. Write a few examples.
