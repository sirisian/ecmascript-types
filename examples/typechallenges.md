# Type Challenges

[type-challenges](https://github.com/type-challenges/type-challenges) is the standard obstacle course for TypeScript's type-level language: implement `Pick`, unwrap a `Promise`, take the head of a tuple, using only conditional types, mapped types, `infer`, and recursion. It is a good adversarial test of [type builders](../typeprogramming.md), because the problems were *chosen* to be natural in a type language whose only verbs are `extends` and `in`. If the builder model only won on problems selected to suit it, that would prove nothing.

This document works the warm-up, all thirteen easy challenges, and the first six medium ones, in the order the README lists them. Each entry gives the problem, the top-voted community solution in TypeScript (linked, with its score at the time of writing), and the builder solution. The builders are written from the primitives in [type programming](../typeprogramming.md) rather than by calling the `std:types` entry that already ships the answer, since implementing the utility is the whole point of the exercise.

Features exercised, and why they matter here:

- `Reflect.getReflection.<Reflect.Type>` and `Reflect.makeType` as the read and write halves of one API: every solution below is reflect, transform, rebuild.
- `Reflect.isAssignable` standing in for `T extends U`, and interned `===` standing in for the harness's own `Equal<X, Y>`, which in TypeScript is an incantation exploiting checker internals.
- Literal types with values readable back out, including **symbol literal types**, which challenge 11 requires and which no TypeScript solution can express without `unique symbol` gymnastics.
- `never` as the empty union, which challenge 14 returns for an empty tuple.
- Authored `TypeError` diagnostics where the challenges write `@ts-expect-error`, so a wrong argument reports what was wrong rather than that something was.
- Argument-bound value generics, which give challenge 12 the key *as a string* and turn a fluent builder's accumulating type into `[...properties, prop(key, V)]`.
- The proposal's fixed arrays `[N].<T>`, which give challenge 18 an answer TypeScript cannot state.
- Type objects usable at runtime, so every "test case" below is also an ordinary unit test.

Several challenges surfaced questions the type programming document implies but does not settle, and two found genuine divergences. The notes at the end record them.

## Preamble

```js
import { arms, arrayOf, firstParameter, keysOf, literal, literalValues, mapProperties,
         never, objectOf, prop, reflect, union } from 'std:types';

// One local helper the tuple challenges share.
function tupleElements(T: type): [].<Reflect.TypeTupleElement> {
  const node = reflect(T);
  if (node.kind !== 'tuple') throw new TypeError(`expected a tuple type, got ${String(T)}`);
  return node.elements;
}
```

The harness's `Expect<Equal<X, Y>>` becomes `===`, so the cases are written as plain assertions.

## 13 · Hello World

Make `HelloWorld` a `string` rather than `any`.

```ts
// TypeScript, issue #150 (+106)
type HelloWorld = string
```

```js
// Builder
type HelloWorld = string;

HelloWorld === string;         // the case, as an assertion
Reflect.typeOf('hi') === HelloWorld;
```

Identical, and deliberately so: the warm-up asks for a type, not a computation. What differs is only that the second line can run.

## 4 · Pick

Construct a type from `T` with only the properties in `K`. `MyPick<Todo, 'title' | 'invalid'>` must be an error.

```ts
// TypeScript, issue #13427 (+445)
type MyPick<T, K extends keyof T> = {
  [key in K]: T[key]
}
```

```js
// Builder
function myPick(T: type, K: type): type {
  const wanted = new Set(literalValues(K));
  const kept = reflect(T).properties.filter(p => wanted.has(p.name));
  const missing = [...wanted].filter(k => !kept.some(p => p.name === k));
  if (missing.length > 0)
    throw new TypeError(`myPick: ${String(T)} has no property ${missing.map(k => `'${String(k)}'`).join(', ')}`);
  return objectOf(kept);
}

type Todo = { title: string, description: string, completed: boolean };
type TodoPreview = myPick(Todo, type 'title' | 'completed');
TodoPreview === type { title: string, completed: boolean };
myPick(Todo, type 'title' | 'invalid');  // TypeError: myPick: Todo has no property 'invalid'
```

Keeping the property *records* rather than rebuilding from key and type is what makes this preserve `readonly`, defaults, and provenance for free. The `K extends keyof T` constraint has two counterparts: the thrown error above at the call, or, when `myPick` appears in a signature, the computed constraint `<T, K: keysOf(T)>` which checks at the declaration.

## 7 · Readonly

Make every property of `T` readonly. Shallow: a nested object stays mutable.

```ts
// TypeScript, issue #1348 (+135)
type MyReadonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

```js
// Builder
function myReadonly(T: type): type {
  return mapProperties(T, p => ({ ...p, readonly: true }));
}

type Todo = { title: string, description: string, meta: { author: string } };
type Frozen = myReadonly(Todo);
Frozen === type { readonly title: string, readonly description: string, readonly meta: { author: string } };
```

The spread is the modifier arithmetic: `readonly` is set, everything else on the record is copied, and the nested `meta` is untouched because nothing recursed into it. TypeScript's `-readonly` is `false` in the same slot.

## 11 · Tuple to Object

Turn a tuple of literals into an object type mapping each value to itself. Keys may be strings, numbers, or symbols; anything else is an error.

```ts
// TypeScript, issue #2737 (+178)
type TupleToObject<T extends readonly string[]> = {
  [P in T[number]]: P
}
```

```js
// Builder
function tupleToObject(T: type): type {
  return objectOf(reflect(T).elements.map(e => {
    const node = reflect(e.type);
    if (node.kind !== 'literal' || typeof node.value === 'boolean' || typeof node.value === 'bigint')
      throw new TypeError(`tupleToObject: ${String(e.type)} is not a valid property key`);
    return prop(typeof node.value === 'symbol' ? node.value : String(node.value), e.type);
  }));
}

type Cars = tupleToObject(type ['tesla', 'model 3']);
Cars === type { tesla: 'tesla', 'model 3': 'model 3' };

const sym1 = Symbol('1'), sym2 = Symbol('2');
type Syms = tupleToObject(type [sym1, sym2]);      // { [sym1]: sym1, [sym2]: sym2 }
Syms === type { [sym1]: sym1, [sym2]: sym2 };      // symbol literal types, no `unique symbol` needed

tupleToObject(type [[1, 2], {}]);                  // TypeError: tupleToObject: [1, 2] is not a valid property key
```

The symbol case is the interesting one. The published TypeScript solution constrains `T extends readonly string[]` and therefore fails the harness's own symbol and number cases; the answers that pass widen the constraint to `readonly (string | number | symbol)[]`. Here the key check is a line of code that names what a property key is, and symbol literal types make the symbol case fall out rather than need a workaround. On numbers, see the note below.

## 14 · First of Array

The first element's type, or `never` for an empty tuple. Non-arrays are an error.

```ts
// TypeScript, issue #16315 (+228)
type First<T extends any[]> = T extends [infer A, ...infer rest] ? A : never
```

```js
// Builder
function first(T: type): type {
  const node = reflect(T);
  if (node.kind === 'tuple') return node.elements[0]?.type ?? never;
  if (node.kind === 'array') return node.extent === 0 ? never : node.element;
  throw new TypeError(`first: ${String(T)} is not an array or tuple type`);
}

first(type [3, 2, 1]) === type 3;
first(type []) === never;
first(type [undefined]) === type undefined;   // an element that exists and is undefined, not a missing element
first([].<string>) === string;
first(type 'notArray');                       // TypeError: first: 'notArray' is not an array or tuple type
```

`infer A` here is destructuring, and reflection is the accessor, so the conditional evaporates into an array index. The `[undefined]` case is the one that catches naive solutions in both languages, and `?.` plus `??` says exactly what is meant: the element record is missing, not the element's type.

## 18 · Length of Tuple

The length of a tuple as a literal type.

```ts
// TypeScript, issue #725 (+72)
type Length<T extends readonly any[]> = T['length']
```

```js
// Builder
function length(T: type): type {
  const node = reflect(T);
  if (node.kind === 'tuple') return literal(node.elements.length);
  if (node.kind === 'array') return node.extent === undefined ? uint32 : literal(node.extent);
  throw new TypeError(`length: ${String(T)} is not an array or tuple type`);
}

length(type ['tesla', 'model 3', 'model X', 'model Y']) === type 4;
length([16].<float32>) === type 16;        // a fixed array's extent: TypeScript has no such type to ask about
length([].<string>) === uint32;            // unknown at compile time, as TS's number
length(uint8);                             // TypeError: length: uint8 is not an array or tuple type
```

TypeScript reads `'length'` off the tuple because a tuple's length happens to be a literal-typed property. The builder asks the reflection how many elements there are, which is the same question without the indirection, and which extends to the two array forms TypeScript does not have.

## 43 · Exclude

Remove from union `T` the members assignable to `U`.

```ts
// TypeScript, issue #54 (+77)
type MyExclude<T, U> = T extends U ? never : T;
```

```js
// Builder
function myExclude(T: type, U: type): type {
  return union(arms(T).filter(arm => !Reflect.isAssignable(arm, U)));
}

myExclude(type 'a' | 'b' | 'c', type 'a') === type 'b' | 'c';
myExclude(type 'a' | 'b' | 'c', type 'a' | 'b') === type 'c';
myExclude(type string | uint32 | (() => void), Function) === type string | uint32;
myExclude(type 'a', type 'a') === never;
```

The whole challenge is distribution, which is TypeScript's implicit behavior when `T` is a naked type parameter and its best-known foot-gun when it isn't. Here the distribution is the `.filter`, and `never` is what a union with no arms interns to, so the last case needs no rule of its own.

## 189 · Awaited

Unwrap nested promises, and thenables generally. Non-thenables are an error.

```ts
// TypeScript, issue #24969 (+79)
type MyAwaited<T extends PromiseLike<any>> = T extends PromiseLike<infer U>
  ? U extends PromiseLike<any>
    ? MyAwaited<U>
    : U
  : never;
```

```js
// Builder
function thenValue(T: type): type | null {
  const node = reflect(T);
  if (node.kind === 'primitive' && node.generic?.base === Promise)
    return node.generic.arguments[0];
  const then = node.kind === 'object' && node.properties.find(p => p.name === 'then');
  return then ? firstParameter(reflect(then.type).signatures[0].parameters[0].type) : null;
}
function myAwaited(T: type): type {
  const inner = thenValue(T);
  if (inner === null) throw new TypeError(`myAwaited: ${String(T)} is not a thenable`);
  return thenValue(inner) === null ? inner : myAwaited(inner);
}

myAwaited(Promise.<string>) === string;
myAwaited(Promise.<Promise.<string | uint32>>) === type string | uint32;
myAwaited(Promise.<Promise.<Promise.<string | boolean>>>) === type string | boolean;
myAwaited(type { then: (onfulfilled: (arg: uint32) => any) => any }) === uint32;
myAwaited(uint32);                        // TypeError: myAwaited: uint32 is not a thenable
```

Both solutions have the same shape, which is the point worth noticing: the recursion is genuine and neither language avoids it. What differs is the reach. `T extends PromiseLike<infer U>` is TypeScript's only way inside `Promise<T>`, and the nested `F extends (value: infer V, ...) => any` in the real `lib.d.ts` definition is how it gets at a thenable's callback parameter. The `generic` field and `firstParameter` are those two moves as accessors.

## 268 · If

Pick `T` or `F` on a boolean condition.

```ts
// TypeScript, issue #279 (+73)
type If<C extends boolean, T, F> = C extends true ? T : F;
```

```js
// Builder
function ifType(C: type, T: type, F: type): type {
  if (!Reflect.isAssignable(C, boolean)) throw new TypeError(`ifType: ${String(C)} is not a boolean`);
  const conditions = C === boolean ? [type true, type false] : arms(C);   // see the note on boolean
  return union(conditions.map(c => c === type true ? T : F));
}

ifType(type true, type 'a', type 'b') === type 'a';
ifType(type false, type 'a', type 2) === type 2;
ifType(boolean, type 'a', type 2) === type 'a' | 2;   // both branches: boolean is not decided
ifType(type null, type 'a', type 'b');                // TypeError: ifType: null is not a boolean
```

The third case is the one that makes this challenge worth including. `If<boolean, 'a', 2>` is `'a' | 2` in TypeScript because `boolean` *is* the union `true | false` there, so the conditional distributes over it and both branches survive. That is not obviously the answer a reader expects from a type named `If`, and it is the only place in these ten where the builder needs a line that TypeScript gets from a modelling decision rather than from a rule. See the note.

## 533 · Concat

Concatenate two tuples.

```ts
// TypeScript, issue #538 (+61)
type Tuple = readonly unknown[];
type Concat<T extends Tuple, U extends Tuple> = [...T, ...U];
```

```js
// Builder
function concat(A: type, B: type): type {
  return Reflect.makeType({ kind: 'tuple', elements: [...tupleElements(A), ...tupleElements(B)] });
}

concat(type [], type []) === type [];
concat(type [], type [1]) === type [1];
concat(type [1, 2], type [3, 4]) === type [1, 2, 3, 4];
concat(type ['1', 2, '3'], type [false, boolean, '4']) === type ['1', 2, '3', false, boolean, '4'];
concat(type null, type undefined);   // TypeError: expected a tuple type, got null
```

This is the first of three challenges where TypeScript's answer is shorter, and it is worth conceding plainly: variadic tuple spread is good syntax for exactly this, and `[...T, ...U]` reads better than rebuilding a node. The builder earns its extra lines only at the error case and at anything the spread cannot say.

## 898 · Includes

Does tuple `T` contain `U`? Equality is *exact*: `boolean` does not contain `false`, and `{ a: 'A' }` is not `{ readonly a: 'A' }`.

```ts
// TypeScript, issue #1568 (+150)
type IsEqual<T, U> =
  (<G>() => G extends T ? 1 : 2) extends
  (<G>() => G extends U ? 1 : 2)
    ? true
    : false;

type Includes<Value extends any[], Item> =
  IsEqual<Value[0], Item> extends true
    ? true
    : Value extends [Value[0], ...infer rest]
      ? Includes<rest, Item>
      : false;
```

```js
// Builder
function includes(T: type, U: type): type {
  return tupleElements(T).some(e => e.type === U) ? type true : type false;
}

includes(type ['Kars', 'Esidisi', 'Wamuu', 'Santana'], type 'Kars') === type true;
includes(type ['Kars', 'Esidisi', 'Wamuu', 'Santana'], type 'Dio') === type false;
includes(type [boolean, 2, 3], type false) === type false;             // boolean is not the literal false
includes(type [false, 2, 3], type false) === type true;
includes(type [{ a: 'A' }], type { readonly a: 'A' }) === type false;  // readonly differs, so identity differs
includes(type [1], type 1 | 2) === type false;
includes(type [null], type undefined) === type false;
```

This is the single clearest case in the whole set. The challenge needs **identity**, and TypeScript's only relation is assignability (`extends`), so every passing solution imports or restates the `IsEqual` incantation: two generic function signatures whose relation the checker happens to decide the right way, discovered in a 2018 issue thread and never blessed as a feature. The recursion in `Includes` then exists only because there is no way to iterate a tuple except by peeling it.

The builder is `.some(e => e.type === U)`. Interning *is* identity, so the hack has nothing to do; `.some` is the iteration. Both halves of the challenge evaporate at once, and the `readonly` case is a live demonstration of the point that [type programming](../typeprogramming.md) makes in the abstract: identity and inter-assignability genuinely differ, `===` and `Reflect.isAssignable` are two spellings, and code picks the one it means.

## 3057 · Push

Append `U` to tuple `T`.

```ts
// TypeScript, issue #3874 (+46)
type Push<T extends unknown[], U> = [...T, U]
```

```js
// Builder
function push(T: type, U: type): type {
  return Reflect.makeType({ kind: 'tuple',
    elements: [...tupleElements(T), { type: U, rest: false, initial: undefined }] });
}

push(type [], type 1) === type [1];
push(type [1, 2], type '3') === type [1, 2, '3'];
push(type ['1', 2, '3'], boolean) === type ['1', 2, '3', boolean];
push(uint32, type 1);   // TypeError: expected a tuple type, got uint32
```

## 3060 · Unshift

Prepend `U` to tuple `T`.

```ts
// TypeScript, issue #4824 (+38)
type Unshift<T extends unknown[], U> = [U, ...T]
```

```js
// Builder
function unshift(T: type, U: type): type {
  return Reflect.makeType({ kind: 'tuple',
    elements: [{ type: U, rest: false, initial: undefined }, ...tupleElements(T)] });
}

unshift(type [], type 1) === type [1];
unshift(type [1, 2], type 0) === type [0, 1, 2];
unshift(type ['1', 2, '3'], boolean) === type [boolean, '1', 2, '3'];
```

Push and Unshift are the pair where TypeScript's answer is plainly better, for the same reason Concat was: variadic spread is purpose-built syntax, and `[U, ...T]` says it in six characters. The builder is doing something more general than it needs to.

## 3312 · Parameters

The parameter list of a function type, as a tuple.

```ts
// TypeScript, issue #5112 (+37)
type MyParameters<T extends (...args: any[]) => any> = T extends (...any: infer S) => any ? S : any
```

```js
// Builder
function myParameters(F: type): type {
  const node = reflect(F);
  if (node.kind !== 'function') throw new TypeError(`myParameters: ${String(F)} is not a function type`);
  return Reflect.makeType({ kind: 'tuple',
    elements: node.signatures[0].parameters.map(p => ({ type: p.type, rest: p.rest, initial: p.initial })) });
}

function foo(arg1: string, arg2: uint32): void {}
function baz(): void {}
myParameters(Reflect.typeOf(foo)) === type [string, uint32];
myParameters(Reflect.typeOf(baz)) === type [];
myParameters(string);   // TypeError: myParameters: string is not a function type
```

`infer S` in a rest position is TypeScript reaching for the parameter list because it has no accessor for it; `signatures[0].parameters` is the accessor. The builder also carries `rest` and `initial` across, which the spread form gets for free and a naive reflection would drop.

## 2 · Get Return Type

The return type of a function type.

```ts
// TypeScript, issue #2 (+69)
type MyReturnType<T extends Function> =
  T extends (...args: any) => infer R
    ? R
    : never
```

```js
// Builder
function myReturnType(F: type): type {
  const node = reflect(F);
  if (node.kind !== 'function') throw new TypeError(`myReturnType: ${String(F)} is not a function type`);
  return node.signatures[0].return.type;
}

myReturnType(type () => string) === string;
myReturnType(type () => 123) === type 123;
myReturnType(type () => Promise.<boolean>) === Promise.<boolean>;
myReturnType(type () => () => 'foo') === type () => 'foo';
```

A field read. Note what the `signatures[0]` is quietly deciding: on an overloaded function, this takes the *first* overload, where TypeScript's `infer R` takes the last. Neither is obviously right, but only one of them is written down where a reader can see it. The `std:types` `returnType` takes the union of all overloads instead, which is a third answer, and the fact that all three are one line apart is the point.

## 3 · Omit

`T` without the properties in `K`.

```ts
// TypeScript, issue #448 (+97)
type MyOmit<T, K extends keyof T> = {[P in keyof T as P extends K ? never: P] :T[P]}
```

```js
// Builder
function myOmit(T: type, K: type): type {
  const dropped = new Set(literalValues(K));
  return mapProperties(T, p => dropped.has(p.name) ? null : p);
}

type Todo = { readonly title: string, description: string, completed: boolean };
myOmit(Todo, type 'description') === type { readonly title: string, completed: boolean };
myOmit(Todo, type 'description' | 'completed') === type { readonly title: string };
```

TypeScript's key-remapping `as` clause with a conditional that maps to `never` to delete a key is three mechanisms cooperating to express "drop these". Returning `null` from the callback is the same instruction. Both preserve `readonly` on `title`, but for different reasons: TypeScript's because a mapped type over `keyof T` earns homomorphic status and its modifier-copying rule applies, and the builder's because nobody wrote code to remove it.

## 8 · Readonly 2

Make only the properties in `K` readonly, defaulting to all of them.

```ts
// TypeScript, issue #1721 (+80)
type MyReadonly2<T, K extends keyof T = keyof T> = Omit<T, K> &
  Readonly<Pick<T, K>>;
```

```js
// Builder
function myReadonly2(T: type, K: type = keysOf(T)): type {
  const keys = new Set(literalValues(K));
  const have = new Set(literalValues(keysOf(T)));
  for (const k of keys) if (!have.has(k))
    throw new TypeError(`myReadonly2: ${String(T)} has no property '${String(k)}'`);
  return mapProperties(T, p => keys.has(p.name) ? { ...p, readonly: true } : p);
}

type Todo = { title: string, description?: string, completed: boolean };
myReadonly2(Todo, type 'title' | 'description')
  === type { readonly title: string, readonly description?: string, completed: boolean };
myReadonly2(Todo) === myReadonly(Todo);           // the default parameter is an ordinary default parameter
myReadonly2(Todo, type 'title' | 'invalid');      // TypeError: myReadonly2: Todo has no property 'invalid'
```

Two things worth noticing. The generic default `K = keyof T` becomes a JavaScript default parameter, computed from an earlier parameter, which is a thing default parameters already do. And the results differ in kind: TypeScript produces the *intersection* `Omit<T, K> & Readonly<Pick<T, K>>`, which is inter-assignable with the flat object but not identical to it, which is exactly why the harness has to weaken this challenge's assertions from `Equal` to `Alike`. The builder edits the records in place and returns the flat object, so `===` holds and no weakening is needed.

## 9 · Deep Readonly

Make `T` readonly recursively.

```ts
// TypeScript, issue #187 (+159)
type DeepReadonly<T> = keyof T extends never
  ? T
  : { readonly [k in keyof T]: DeepReadonly<T[k]> };
```

```js
// Builder
function deepReadonly(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'object':
      return objectOf(
        node.properties.map(p => ({ ...p, readonly: true, type: deepReadonly(p.type) })),
        node.indexSignatures.map(s => ({ ...s, value: deepReadonly(s.value) })));
    case 'array': return arrayOf(deepReadonly(node.element), node.extent);
    case 'tuple': return Reflect.makeType({ ...node, elements: node.elements.map(e => ({ ...e, type: deepReadonly(e.type) })) });
    case 'union': return union(node.arms.map(deepReadonly));
    default:      return T;   // primitives, literals, functions, classes, enums, parameterized
  }
}

type X = { a: () => 22, b: string, c: { d: boolean, e: { g: { h: { i: true } } } } };
deepReadonly(X) === type {
  readonly a: () => 22,
  readonly b: string,
  readonly c: { readonly d: boolean, readonly e: { readonly g: { readonly h: { readonly i: true } } } }
};
deepReadonly(type { a: string } | { b: uint32 }) === type { readonly a: string } | { readonly b: uint32 };
```

`keyof T extends never ? T : ...` is a load-bearing hack rather than a base case: it is how the TypeScript solution stops at a function property (`a: () => 22`), because `keyof (() => 22)` happens to be `never`. It works, and it is not what it says. The builder's `switch` states its domain, and the `default` arm is the base case that a reader can check against the list of type kinds.

The recursion into cycles needs no code in either solution, but for different reasons: TypeScript defers instantiation lazily and hits its depth limit when that fails, while the builder's in-flight fixpoint ties the knot, so `deepReadonly` on a self-referential type terminates with the cyclic type rather than an error. See the note on readonly arrays for where this solution genuinely diverges.

## 10 · Tuple to Union

The union of a tuple's element types.

```ts
// TypeScript, issue #7 (+34)
export type TupleToUnion<T> = T extends Array<infer ITEMS> ? ITEMS : never
```

```js
// Builder
function tupleToUnion(T: type): type {
  const node = reflect(T);
  if (node.kind === 'tuple') return union(node.elements.map(e => e.type));
  if (node.kind === 'array') return node.element;
  throw new TypeError(`tupleToUnion: ${String(T)} is not an array or tuple type`);
}

tupleToUnion(type [123, '456', true]) === type 123 | '456' | true;
tupleToUnion(type [123]) === type 123;                 // union of one arm is that arm
tupleToUnion([].<string | uint32>) === type string | uint32;
```

TypeScript's `T[number]` and `T extends Array<infer I> ? I : never` are two spellings of "widen the positional element types into a union", and both are indirections around a list the checker already has. `union(elements.map(...))` is that list. The one-element case needs no special handling because canonicalization already collapses a single-arm union.

## 12 · Chainable Options

Type a fluent builder: `option(key, value)` accumulates a config type, `get()` returns it. The same key must not be set twice.

```ts
// TypeScript, issue #13951 (+85)
type Chainable<T = {}> = {
  option: <K extends string, V>(key: K extends keyof T ?
    V extends T[K] ? never : K
    : K, value: V) => Chainable<Omit<T, K> & Record<K, V>>
  get: () => T
}
```

```js
// Builder
function withKey(T: type, key: string, V: type): type {
  if (reflect(T).properties.some(p => p.name === key))
    throw new TypeError(`option: '${key}' is already set on ${String(T)}`);
  return objectOf([...reflect(T).properties, prop(key, V)]);
}

interface Chainable<T = type {}> {
  option<K: string, V>(key: K, value: V): Chainable.<withKey(T, K, V)>;
  get(): T;
}

declare const a: Chainable;
const result = a
  .option('foo', 123)
  .option('bar', { value: 'Hello World' })
  .option('name', 'type-challenges')
  .get();
// result: { foo: float64, bar: { value: string }, name: string }

a.option('name', 'x').option('name', 'y');   // TypeError: option: 'name' is already set on { name: string }
```

The last challenge in this batch is the one that most resembles a real API, and it separates the two models cleanly. `K` is an argument-bound value generic, so the *string* `'foo'` is available to `withKey` as a string, and the accumulation is `[...properties, prop(key, V)]`. TypeScript cannot pass a key to a function, so it encodes the duplicate check into the parameter's own type: `K extends keyof T ? V extends T[K] ? never : K : K` makes the parameter `never` when the key is a duplicate, so the *argument* fails to be assignable and the call errors. It works, and the error it produces is that `'name'` is not assignable to `never`.

Note what the builder did **not** need to do: it never constructs a generic function type. `option` is declared with its type parameters in ordinary syntax, and the builder only computes the return type's argument. That division is worth stating, because the node model could not have done otherwise, and the note below records why that is fine.

## Notes

**Identity was the whole game, and the harness proves it.** Challenge 898 is the clearest result across all twenty. Its expected values are chosen precisely to break assignability-based answers (`boolean` excludes `false`, `{ a }` excludes `{ readonly a }`, `[1]` excludes `1 | 2`), so every passing TypeScript solution carries the `IsEqual` incantation: two generic function signatures whose relation the checker happens to decide the right way, found in a 2018 issue comment and never blessed as a feature. With interned type objects it is `===`, and the challenge collapses to `.some`. The same incantation is what the harness's own `Expect<Equal<X, Y>>` is built from, which is why every case in this document is written as a plain assertion instead: the challenges' tooling is the first thing the builder model deletes, and each of those assertions is also a line that runs. Challenge 898 additionally settles a claim [type programming](../typeprogramming.md) only asserts in the abstract, that `readonly` is where identity and inter-assignability come apart. The harness agrees.

**`boolean` and its literals.** Challenge 268 is decided by whether `boolean` is the union `true | false`. TypeScript models it that way, so `If<boolean, 'a', 2>` distributes to `'a' | 2`. The [type programming](../typeprogramming.md) document admits boolean literal types and defines `arms` to return `[T]` for a non-union, but never says which of these `boolean` is, so `ifType` above carries an explicit line for it. The recommendation is that **`boolean === type true | false`**, interned as the union of its two literal types: it costs nothing, since a two-inhabitant primitive and the union of its two inhabitants are the same set of values; it makes `arms(boolean)` return both literals and deletes the special case; and it keeps `exclude(boolean, type true) === type false` working, which a reader will expect the moment literal booleans exist. The alternative, keeping `boolean` a primitive node, is defensible for layout reasons but must then say so explicitly, because the question is not decidable from the current text.

**Numeric property keys.** Challenge 11's harness expects `TupleToObject<[1, 2]>` to be `{ 1: 1, 2: 2 }` with numeric keys. JavaScript has no numeric property keys: `{ 1: 1 }` and `{ '1': 1 }` are one object, and the node model's `name: string | symbol` says so. `tupleToObject(type [1, 2])` therefore yields `{ '1': 1, '2': 2 }`, whose *values* are still the number literals. This is a TypeScript-ism rather than a gap, and the node model is right to not reproduce it.

**Readonly arrays are a real divergence.** Challenge 9's expected output contains `readonly ['hi', { readonly m: readonly ['hey'] }]`. TypeScript has `readonly T[]` as a *type*; this proposal has `readonly` as a *field modifier* and reaches immutable arrays through frozen instances and value semantics instead, which the comparison table already calls out as a different mechanism. So `deepReadonly` marks fields readonly and recurses through element types, but cannot mark the array itself readonly, and the challenge's tuple cases pass only up to that difference. This is not a builder limitation: no reflection API can construct a modifier the type grammar does not have. If `readonly` on array and tuple types is ever wanted it belongs in the main proposal, and the node model would carry it as a flag on the `array` and `tuple` nodes, which `deepReadonly` would set in the spread it already uses for properties.

**The node model has no generic signatures.** `FunctionSignatureReflection` carries parameters and a return, but no type parameters, so a builder cannot *construct* `<K, V>(key: K, value: V) => R` with `makeType`, and a generic function type cannot round-trip through reflection. Challenge 12 shows why this has not bitten: generic methods are declared in ordinary syntax and builders compute their *arguments*, which is the natural division of labor. Recording it as a known limit rather than an oversight, the open question being whether reflection should carry type parameters for completeness. Nothing in twenty challenges has needed it.

**Intersections versus flat objects.** Challenge 8 weakens its assertions from `Equal` to `Alike` because `Omit<T, K> & Readonly<Pick<T, K>>` is inter-assignable with the intended object but not identical to it. The builder edits records in place, returns the flat object, and passes the stronger assertion. That the harness had to weaken a test to accommodate the idiomatic TypeScript answer is small but telling: the type language's shape leaks into what the test is allowed to say.

**Constraints versus checks.** These challenges are stated as generic *aliases* with constraints (`MyPick<T, K extends keyof T>`), and the builders are *functions*, so a bad argument is a thrown `TypeError` at the call rather than a constraint violation at the declaration. The two are not far apart: when a builder appears in a signature, the computed constraint form (`<T, K: keysOf(T)>`) restores the declaration-site check, and the thrown message is available in both cases. What the builder gives up is being checkable before its arguments are known; what it gains is saying which property was missing.

**Where TypeScript wins.** Concat, Push, and Unshift: three of twenty, and all three the same win. Variadic tuple spread is purpose-built syntax, `[U, ...T]` says it in six characters, and no reflection call will be shorter. The pattern across the set is consistent. Where TypeScript has syntax for an operation on types, it beats a builder; where it must *encode* a computation (identity as a signature relation, iteration as recursive peeling, a duplicate-key check as a parameter that becomes `never`, a base case as `keyof T extends never`), the encoding is precisely what the builder deletes.
