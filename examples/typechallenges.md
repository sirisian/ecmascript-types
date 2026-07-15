# Type Challenges

[type-challenges](https://github.com/type-challenges/type-challenges) is the standard obstacle course for TypeScript's type-level language: implement `Pick`, unwrap a `Promise`, take the head of a tuple, using only conditional types, mapped types, `infer`, and recursion. It is a good adversarial test of [type builders](../typeprogramming.md), because the problems were *chosen* to be natural in a type language whose only verbs are `extends` and `in`. If the builder model only won on problems selected to suit it, that would prove nothing.

This document works the warm-up, all thirteen easy challenges, and the first twenty-six medium ones, in the order the README lists them. Every TypeScript solution shown has been checked against the repo's own test cases with TypeScript 5.9.3; where the top-voted answer no longer passes, a verified passing answer is shown instead and the note at the end says why. Each entry gives the problem, the top-voted community solution in TypeScript (linked, with its score at the time of writing), and the builder solution. The builders are written from the primitives in [type programming](../typeprogramming.md) rather than by calling the `std:types` entry that already ships the answer, since implementing the utility is the whole point of the exercise.

Features exercised, and why they matter here:

- `Reflect.getReflection.<Reflect.Type>` and `Reflect.makeType` as the read and write halves of one API: every solution below is reflect, transform, rebuild.
- `Reflect.isAssignable` standing in for `T extends U`, and interned `===` standing in for the harness's own `Equal<X, Y>`, which in TypeScript is an incantation exploiting checker internals.
- Literal types with values readable back out, including **symbol literal types**, which challenge 11 requires and which no TypeScript solution can express without `unique symbol` gymnastics.
- `never` as the empty union, which challenge 14 returns for an empty tuple.
- Authored `TypeError` diagnostics where the challenges write `@ts-expect-error`, so a wrong argument reports what was wrong rather than that something was.
- Argument-bound value generics, which give challenge 12 the key *as a string*, turn a fluent builder's accumulating type into `[...properties, prop(key, V)]`, and reduce the five string challenges to `trimStart`, `trim`, `toUpperCase`, `replace`, and `replaceAll`.
- The proposal's fixed arrays `[N].<T>`, which give challenge 18 an answer TypeScript cannot state.
- Type objects usable at runtime, so every "test case" below is also an ordinary unit test.

Several challenges surfaced questions the type programming document implies but does not settle, and two found genuine divergences. The notes at the end record them.

## Preamble

```js
import { arms, arrayOf, awaited, firstParameter, keysOf, literal, literalValues, mapProperties,
         never, objectOf, prop, reflect, tupleOf, union } from 'std:types';

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

## 15 · Last of Array

The last element's type, or `never` for an empty tuple.

```ts
// TypeScript, issue #38 (+54). The top-voted answer no longer passes; see the note.
type Last<T extends any[]> = T extends [...infer _, infer L] ? L : never
```

```js
// Builder
function last(T: type): type {
  return tupleElements(T).at(-1)?.type ?? never;
}

last(type [3, 2, 1]) === type 1;
last(type [2]) === type 2;
last(type []) === never;
last(uint32);   // TypeError: expected a tuple type, got uint32
```

## 16 · Pop

The tuple without its last element. `Pop<[]>` is `[]`.

```ts
// TypeScript, issue #8622 (+2). The top-voted answer no longer passes; see the note.
type Pop<T extends unknown[]> = T extends [ ...infer O, infer L ] ? O : []
```

```js
// Builder
function pop(T: type): type {
  return Reflect.makeType({ kind: 'tuple', elements: tupleElements(T).slice(0, -1) });
}

pop(type [3, 2, 1]) === type [3, 2];
pop(type ['a', 'b', 'c', 'd']) === type ['a', 'b', 'c'];
pop(type []) === type [];
pop(uint32);   // TypeError: expected a tuple type, got uint32
```

`slice(0, -1)` on an empty list is empty, and a non-tuple never reaches the slice. The note below is about why that sentence is the whole finding.

## 20 · Promise.all

Type `PromiseAll`, which takes a list of values or promises and resolves to the list of their settled types.

```ts
// TypeScript, issue #211 (+62)
declare function PromiseAll<T extends any[]>(values: readonly [...T]):
  Promise<{ [K in keyof T]: T[K] extends Promise<infer R> ? R : T[K] }>;
```

```js
// Builder
function settled(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'tuple': return Reflect.makeType({ ...node, elements: node.elements.map(e => ({ ...e, type: awaited(e.type) })) });
    case 'array': return arrayOf(awaited(node.element), node.extent);
    default: throw new TypeError(`promiseAll: ${String(T)} is not an array or tuple type`);
  }
}

declare function promiseAll<T>(values: T): Promise.<settled(T)>;

promiseAll.<type [1, 2, 3]>;                        // returns Promise.<[1, 2, 3]>
promiseAll.<type [1, 2, Promise.<uint32>]>;         // returns Promise.<[1, 2, uint32]>
promiseAll.<[].<uint32 | Promise.<string>>>;        // returns Promise.<[].<uint32 | string>>
```

`{ [K in keyof T]: ... }` over a tuple is TypeScript's most surprising rule: a mapped type over an array or tuple is special-cased to produce an array or tuple rather than an object with numeric keys, which is why this signature works at all and why nothing in the syntax says so. The builder's `switch` says so. Note also that `awaited` already means "unwrap if thenable, else pass through", so `T[K] extends Promise<infer R> ? R : T[K]` is a call rather than a conditional.

The harness's first and third cases differ only by `as const`, checking that a widened `[1, 2, Promise<3>]` still maps element-wise. This proposal marks `as const` obviated, since literals keep their literal types and value types are immutable by layout, so the two cases collapse into one and there is nothing left to test.

## 62 · Type Lookup

Select the member of a tagged union whose `type` field matches.

```ts
// TypeScript, issue #149 (+104)
type LookUp<U, T> = U extends {type: T} ? U : never;
```

```js
// Builder
function lookUp(U: type, T: type): type {
  return union(arms(U).filter(arm => Reflect.isAssignable(arm, objectOf([prop('type', T)]))));
}

interface Cat { type: 'cat'; breeds: 'Abyssinian' | 'Shorthair' }
interface Dog { type: 'dog'; breeds: 'Hound' | 'Boxer'; color: 'brown' | 'white' }
type Animal = Cat | Dog;

lookUp(Animal, type 'dog') === Dog;
lookUp(Animal, type 'cat') === Cat;
lookUp(Animal, type 'bird') === never;
```

A one-line translation, and the closest correspondence in the whole set: `extends` becomes `Reflect.isAssignable`, distribution becomes `.filter`, and the `never` branch becomes the empty union. `std:types` ships this as `byKind(T, k, tag)`, with the tag a parameter rather than hardcoded to `'type'`.

## 106 · Trim Left

Strip leading whitespace from a string literal type.

```ts
// TypeScript, issue #346 (+90)
type Space = ' ' | '\n' | '\t'
type TrimLeft<S extends string> = S extends `${Space}${infer R}` ? TrimLeft<R> : S
```

```js
// Builder
function trimLeft(s: string): type {
  return literal(s.trimStart());
}

trimLeft('     str     ') === type 'str     ';
trimLeft('   \n\t foo bar ') === type 'foo bar ';
trimLeft(' \n\t') === type '';
```

## 108 · Trim

Strip whitespace from both ends.

```ts
// TypeScript, issue #481 (+107)
type Space = ' ' | '\t' | '\n';
type Trim<S extends string> = S extends `${Space}${infer T}` | `${infer T}${Space}` ? Trim<T> : S;
```

```js
// Builder
function trim(s: string): type {
  return literal(s.trim());
}

trim('     str     ') === type 'str';
trim('   \n\t foo bar \t') === type 'foo bar';
trim(' \n\t ') === type '';
```

## 110 · Capitalize

Uppercase the first character.

```ts
// TypeScript, issue #759 (+32)
type MyCapitalize<S extends string> = S extends `${infer x}${infer tail}` ? `${Uppercase<x>}${tail}` : S;
```

```js
// Builder
function myCapitalize(s: string): type {
  return literal(s.charAt(0).toUpperCase() + s.slice(1));
}

myCapitalize('foo bar') === type 'Foo bar';
myCapitalize('FOOBAR') === type 'FOOBAR';
myCapitalize('') === type '';
```

## 116 · Replace

Replace the first occurrence of `From` in `S` with `To`. An empty `From` changes nothing.

```ts
// TypeScript, issue #407 (+40)
type Replace<S extends string, From extends string, To extends string> =
      From extends ''
      ? S
      : S extends `${infer V}${From}${infer R}`
        ? `${V}${To}${R}`
        : S
```

```js
// Builder
function replace(s: string, from: string, to: string): type {
  return literal(from === '' ? s : s.replace(from, () => to));
}

replace('foobarbar', 'bar', 'foo') === type 'foofoobar';
replace('foobarbar', 'bar', '') === type 'foobar';
replace('foobarbar', 'bra', 'foo') === type 'foobarbar';
replace('foobarbar', '', 'foo') === type 'foobarbar';
```

## 119 · ReplaceAll

Replace every occurrence.

```ts
// TypeScript, issue #367 (+56)
type ReplaceAll<S extends string, From extends string, To extends string> = From extends ''
  ? S
  : S extends `${infer R1}${From}${infer R2}`
  ? `${R1}${To}${ReplaceAll<R2, From, To>}`
  : S
```

```js
// Builder
function replaceAll(s: string, from: string, to: string): type {
  return literal(from === '' ? s : s.replaceAll(from, () => to));
}

replaceAll('foobarbar', 'bar', 'foo') === type 'foofoofoo';
replaceAll('t y p e s', ' ', '') === type 'types';
replaceAll('foobarfoobar', 'ob', 'b') === type 'fobarfobar';
replaceAll('foboorfoboar', 'bo', 'b') === type 'foborfobar';
replaceAll('foobarbar', '', 'foo') === type 'foobarbar';
```

Five string challenges in a row, and the builder answers are `trimStart`, `trim`, `toUpperCase`, `replace`, and `replaceAll`. There is nothing to explain, which is the finding. Two details are worth pointing at anyway. The replacement is passed as `() => to` rather than `to`, because `String.prototype.replace` gives `$&` and `$1` special meaning in a replacement *string*, and a type-level `Replace<S, 'a', '$&'>` must insert the two characters `$&`; the function form is the one that means what the challenge means. And the empty-`From` guard is needed because `'abc'.replaceAll('', 'x')` interpolates, where the challenge wants identity. Both are cases where the builder author has to know JavaScript's string semantics, which is a real cost, and both are one conditional.

The `Space` alias in 106 and 108 is worth a second look. It is `' ' | '\n' | '\t'`, three characters, because in TypeScript every whitespace character to be trimmed has to be enumerated into a union and matched by a template pattern, so the definition is bounded by the author's patience. `trimStart` is defined against Unicode's `White_Space` property. The builder is not just shorter here; it is the only one of the two that is *correct* for `'\r\n str'`.

## 191 · Append Argument

Add a parameter to the end of a function type.

```ts
// TypeScript, issue #222 (+67)
type AppendArgument<Fn, A> = Fn extends (...args: infer R) => infer T ? (...args: [...R, A]) => T : never
```

```js
// Builder
function appendArgument(F: type, A: type): type {
  const node = reflect(F);
  if (node.kind !== 'function') throw new TypeError(`appendArgument: ${String(F)} is not a function type`);
  return Reflect.makeType({ ...node, signatures: node.signatures.map(sig => ({
    ...sig,
    parameters: [...sig.parameters,
      { type: A, name: 'appended', index: sig.parameters.length, rest: false, initial: undefined, metadata: {} }],
  })) });
}

appendArgument(type (a: uint32, b: string) => uint32, boolean) === type (a: uint32, b: string, x: boolean) => uint32;
appendArgument(type () => void, type undefined) === type (x: undefined) => void;
appendArgument(uint32, boolean);   // TypeError: appendArgument: uint32 is not a function type
```

Read the first assertion again: the builder names its new parameter `appended`, the expected type names it `x`, and they are equal. That is only true if parameter names are not part of a function type's identity, which the note below takes up. Note also that the builder appends to *every* overload, which the `infer R` form cannot express at all: on an overloaded function it silently picks the last signature and discards the rest.

## 296 · Permutation

Every ordering of a union, as a union of tuples.

```ts
// TypeScript, issue #614 (+379)
type Permutation<T, K=T> =
    [T] extends [never]
      ? []
      : K extends K
        ? [K, ...Permutation<Exclude<T, K>>]
        : never
```

```js
// Builder
function permutation(T: type): type {
  const perms = (xs: [].<type>): [].<[].<type>> =>
    xs.length === 0 ? [[]] : xs.flatMap((x, i) => perms(xs.toSpliced(i, 1)).map(rest => [x, ...rest]));
  return union(perms(arms(T)).map(tupleOf));
}

permutation(type 'A') === type ['A'];
permutation(type 'A' | 'B' | 'C')
  === type ['A','B','C'] | ['A','C','B'] | ['B','A','C'] | ['B','C','A'] | ['C','A','B'] | ['C','B','A'];
permutation(boolean) === type [false, true] | [true, false];
permutation(never) === type [];
```

The most-upvoted answer in the whole repository, and it deserves the votes: it is genuinely clever. Look at what the cleverness is *for*, though. `K extends K` is a conditional that is always true. It computes nothing. It exists solely because a conditional type over a naked parameter distributes, so writing a tautology is the only way to say "iterate the arms of this union", and the `: never` branch is unreachable filler needed to complete a syntax form. `[T] extends [never]` is the tuple-wrapping idiom, present because the author needs the *opposite* of distribution for the base case. Two idioms, both about controlling a behavior nobody asked for, wrapped around the actual algorithm.

The builder's `perms` is the permutation function anyone would write, and `.flatMap` is the iteration. `permutation(never)` needs no base case at all: `arms(never)` is the empty list, `perms([])` is `[[]]`, and the union of one tuple is that tuple.

## 298 · Length of String

The length of a string literal type, as a number.

```ts
// TypeScript, issue #359 (+34)
type LengthOfString<
  S extends string,
  T extends string[] = []
> = S extends `${infer F}${infer R}`
  ? LengthOfString<R, [...T, F]>
  : T['length'];
```

```js
// Builder
function lengthOfString(s: string): type {
  return literal(s.length);
}

lengthOfString('Sound! Euphonium') === type 16;
lengthOfString('') === type 0;
```

TypeScript has no way to count, so it peels the string one character at a time into an accumulator tuple and reads the tuple's `length`, which is the only length that exists at the type level. This is also why the challenge has a practical ceiling around a thousand characters. `s.length` has no ceiling and no accumulator.

## 459 · Flatten

Flatten a nested tuple type, to any depth.

```ts
// TypeScript, issue #1314 (+28)
type Flatten<S extends any[], T extends any[] = []> =  S extends [infer X, ...infer Y] ?
  X extends any[] ?
   Flatten<[...X, ...Y], T> : Flatten<[...Y], [...T, X]>
  : T
```

```js
// Builder
function flatten(T: type): type {
  const flat = (elements: [].<Reflect.TypeTupleElement>): [].<Reflect.TypeTupleElement> =>
    elements.flatMap(e => reflect(e.type).kind === 'tuple' ? flat(reflect(e.type).elements) : [e]);
  return Reflect.makeType({ kind: 'tuple', elements: flat(tupleElements(T)) });
}

flatten(type [1, 2, [3, 4], [[[5]]]]) === type [1, 2, 3, 4, 5];
flatten(type [{ foo: 'bar' }, 'foobar']) === type [{ foo: 'bar' }, 'foobar'];
flatten(type []) === type [];
flatten(type '1');   // TypeError: expected a tuple type, got '1'
```

The accumulator in the TypeScript version is not incidental: with no `flatMap` and no local variables, the only way to build a result is to thread it through the recursion as a second type parameter with a default. The builder's `flat` is the same recursion without the plumbing. (`std:types` also exports a `flatten`, which is the unrelated shallow `[].<U>` unwrap; this one is the challenge's deep tuple flatten.)

## 527 · Append to object

Add a key and value type to an object type.

```ts
// TypeScript, issue #536 (+49)
type AppendToObject<T, U extends keyof any, V> = {
  [K in keyof T | U]: K extends keyof T ? T[K] : V;
};
```

```js
// Builder
function appendToObject(T: type, key: string | symbol, V: type): type {
  return objectOf([...reflect(T).properties, prop(key, V)]);
}

type Test = { key: 'cat', value: 'green' };
appendToObject(Test, 'home', boolean) === type { key: 'cat', value: 'green', home: boolean };
```

A mapped type over `keyof T | U` has to re-derive, for every key, which of the two sources it came from, because the mapped form is the only way to build an object type and it must produce all keys in one pass. Appending to a list is appending to a list.

## 529 · Absolute

The absolute value of a number, string, or bigint literal, as a string.

```ts
// TypeScript, issue #10386 (+47)
type Absolute<T extends number | string | bigint> = `${T}` extends `-${infer U}` ? U : `${T}`
```

```js
// Builder
function absolute(v: number | string | bigint): type {
  return literal(String(v).replace(/^-/, ''));
}

absolute(-5) === type '5';
absolute(-0) === type '0';
absolute(-1_000_000n) === type '1000000';
absolute('-5') === type '5';
```

An honest near-draw, and worth including for that. Both solutions do the same thing for the same reason: the result is specified as a *string*, so both stringify and strip a leading minus rather than doing arithmetic. TypeScript's `` `${T}` `` is its stringify, and the pattern match is its strip. The builder is not cleverer here; it is the same idea with `String` and a regex instead of a template pattern.

## 531 · String to Union

The union of a string's characters.

```ts
// TypeScript, issue #537 (+41)
type StringToUnion<T extends string> = T extends `${infer Letter}${infer Rest}`
  ? Letter | StringToUnion<Rest>
  : never;
```

```js
// Builder
function stringToUnion(s: string): type {
  return union([...s].map(literal));
}

stringToUnion('hello') === type 'h' | 'e' | 'l' | 'o';
stringToUnion('t') === type 't';
stringToUnion('') === never;
```

The empty case falls out twice over: `[...'']` is empty and the union of no arms is `never`, which is the answer the challenge wants. Note the harness writes the `'hello'` case as `'h' | 'e' | 'l' | 'l' | 'o'` with `'l'` twice, which is the same type as the deduplicated one in both systems.

## 599 · Merge

Merge two object types; the second wins on conflicts.

```ts
// TypeScript, issue #608 (+25)
type Merge<F, S> = {
  [K in keyof F | keyof S]: K extends keyof S
    ? S[K]
    : K extends keyof F
    ? F[K]
    : never;
};
```

```js
// Builder
function merge(F: type, S: type): type {
  const second = reflect(S).properties;
  const overridden = new Set(second.map(p => p.name));
  return objectOf([...reflect(F).properties.filter(p => !overridden.has(p.name)), ...second]);
}

type Foo = { a: uint32, b: string };
type Bar = { b: uint32, c: boolean };
merge(Foo, Bar) === type { a: uint32, b: uint32, c: boolean };
```

The `: never` branch in the TypeScript version is unreachable, since `K` came from `keyof F | keyof S` and cannot be outside both. It is there because a conditional needs an else. That is the third unreachable `never` in this batch alone, and the note below is about what they have in common.

## 612 · KebabCase

`FooBarBaz` becomes `foo-bar-baz`.

```ts
// TypeScript, issue #664 (+64)
type KebabCase<S extends string> = S extends `${infer S1}${infer S2}`
  ? S2 extends Uncapitalize<S2>
  ? `${Uncapitalize<S1>}${KebabCase<S2>}`
  : `${Uncapitalize<S1>}-${KebabCase<S2>}`
  : S;
```

```js
// Builder
function kebabCase(s: string): type {
  return literal([...s].map((c, i) => {
    const lower = c.toLowerCase();
    return lower !== c && i > 0 ? `-${lower}` : lower;
  }).join(''));
}

kebabCase('FooBarBaz') === type 'foo-bar-baz';
kebabCase('fooBarBaz') === type 'foo-bar-baz';
kebabCase('Foo-Bar') === type 'foo--bar';
kebabCase('ABC') === type 'a-b-c';
kebabCase('😎') === type '😎';
```

Both solutions test "is this character uppercase" the same way, by asking whether lowercasing changes it, because that is the definition that works for every alphabet rather than for `A` through `Z`. TypeScript spells it `S2 extends Uncapitalize<S2>` and the builder spells it `lower !== c`. The builder iterates with `[...s]`, which yields code points, so the `'😎'` case is handled by the iteration rather than in spite of it.

## 645 · Diff

The symmetric difference of two object types.

```ts
// TypeScript, issue #3014 (+95)
type Diff<O, O1> = Omit<O & O1, keyof (O | O1)>
```

```js
// Builder
function diff(A: type, B: type): type {
  const inA = new Set(reflect(A).properties.map(p => p.name));
  const inB = new Set(reflect(B).properties.map(p => p.name));
  return objectOf([
    ...reflect(A).properties.filter(p => !inB.has(p.name)),
    ...reflect(B).properties.filter(p => !inA.has(p.name)),
  ]);
}

type Foo = { name: string, age: string };
type Coo = { name: string, gender: uint32 };
diff(Foo, Coo) === type { age: string, gender: uint32 };
```

`Omit<O & O1, keyof (O | O1)>` is the sharpest one-liner in the set, and it is worth understanding rather than dismissing: `O & O1` has every key, `keyof (O | O1)` is the *common* keys because a union's keys are the ones every arm has, and omitting the second from the first leaves the symmetric difference. It is exact, it is four tokens, and it relies on the reader knowing that `keyof` inverts across union and intersection. The builder is longer and says what it does.

## 949 · AnyOf

True if any element of the tuple is truthy. Falsy means `0`, `''`, `false`, `[]`, `{}`, `null`, or `undefined`.

```ts
// TypeScript, issue #15212 (+1). The top-voted answer (#954, +83) is this without
// `null | undefined` in Falsy and no longer passes; see the note.
type EmptyObject = { [i in string]: never };
type Falsy = 0 | '' | false | null | undefined | [] | EmptyObject
type AnyOf<T extends readonly any[]> = T[number] extends Falsy ? false : true;
```

```js
// Builder
const falsy = type 0 | '' | false | null | undefined | [] | { [key: string]: never };

function anyOf(T: type): type {
  return tupleElements(T).some(e => !Reflect.isAssignable(e.type, falsy)) ? type true : type false;
}

anyOf(type [1, 'test', true, [1], { name: 'test' }]) === type true;
anyOf(type [0, '', false, [], { name: 'test' }]) === type true;
anyOf(type [0, '', false, [], {}, undefined, null]) === type false;
anyOf(type []) === type false;
```

The fairest challenge in the batch, and the builder wins almost nothing. Truthiness is a property of *values*, but this challenge has no values: it asks whether a list of **types** contains one that is not in a falsy set, so the falsy set has to be enumerated by hand in both languages and the answer is an assignability question either way. What is left is `.some` against recursion, and `Reflect.isAssignable` against `extends`. Worth including precisely because the pattern of the previous thirty-nine does not hold here: when a problem is genuinely about types rather than about values wearing types, the two models converge.

## Notes

**Identity was the whole game, and the harness proves it.** Challenge 898 is the clearest result across all twenty. Its expected values are chosen precisely to break assignability-based answers (`boolean` excludes `false`, `{ a }` excludes `{ readonly a }`, `[1]` excludes `1 | 2`), so every passing TypeScript solution carries the `IsEqual` incantation: two generic function signatures whose relation the checker happens to decide the right way, found in a 2018 issue comment and never blessed as a feature. With interned type objects it is `===`, and the challenge collapses to `.some`. The same incantation is what the harness's own `Expect<Equal<X, Y>>` is built from, which is why every case in this document is written as a plain assertion instead: the challenges' tooling is the first thing the builder model deletes, and each of those assertions is also a line that runs. Challenge 898 additionally settles a claim [type programming](../typeprogramming.md) only asserts in the abstract, that `readonly` is where identity and inter-assignability come apart. The harness agrees.

**`boolean` and its literals.** Challenge 268 is decided by whether `boolean` is the union `true | false`. TypeScript models it that way, so `If<boolean, 'a', 2>` distributes to `'a' | 2`. The [type programming](../typeprogramming.md) document admits boolean literal types and defines `arms` to return `[T]` for a non-union, but never says which of these `boolean` is, so `ifType` above carries an explicit line for it. The recommendation is that **`boolean === type true | false`**, interned as the union of its two literal types: it costs nothing, since a two-inhabitant primitive and the union of its two inhabitants are the same set of values; it makes `arms(boolean)` return both literals and deletes the special case; and it keeps `exclude(boolean, type true) === type false` working, which a reader will expect the moment literal booleans exist. The alternative, keeping `boolean` a primitive node, is defensible for layout reasons but must then say so explicitly, because the question is not decidable from the current text.

**Numeric property keys.** Challenge 11's harness expects `TupleToObject<[1, 2]>` to be `{ 1: 1, 2: 2 }` with numeric keys. JavaScript has no numeric property keys: `{ 1: 1 }` and `{ '1': 1 }` are one object, and the node model's `name: string | symbol` says so. `tupleToObject(type [1, 2])` therefore yields `{ '1': 1, '2': 2 }`, whose *values* are still the number literals. This is a TypeScript-ism rather than a gap, and the node model is right to not reproduce it.

**Readonly arrays are a real divergence.** Challenge 9's expected output contains `readonly ['hi', { readonly m: readonly ['hey'] }]`. TypeScript has `readonly T[]` as a *type*; this proposal has `readonly` as a *field modifier* and reaches immutable arrays through frozen instances and value semantics instead, which the comparison table already calls out as a different mechanism. So `deepReadonly` marks fields readonly and recurses through element types, but cannot mark the array itself readonly, and the challenge's tuple cases pass only up to that difference. This is not a builder limitation: no reflection API can construct a modifier the type grammar does not have. If `readonly` on array and tuple types is ever wanted it belongs in the main proposal, and the node model would carry it as a flag on the `array` and `tuple` nodes, which `deepReadonly` would set in the spread it already uses for properties.

**The node model has no generic signatures.** `FunctionSignatureReflection` carries parameters and a return, but no type parameters, so a builder cannot *construct* `<K, V>(key: K, value: V) => R` with `makeType`, and a generic function type cannot round-trip through reflection. Challenge 12 shows why this has not bitten: generic methods are declared in ordinary syntax and builders compute their *arguments*, which is the natural division of labor. Recording it as a known limit rather than an oversight, the open question being whether reflection should carry type parameters for completeness. Nothing in twenty challenges has needed it.

**The published answers rot, and the rot is the finding.** Two of this batch's top-voted solutions fail the repo's current tests, which I checked with TypeScript 5.9.3 against its own `Equal`. Challenge 15's `[any, ...T][T['length']]` (+149) evaluates to `any` for `Last<[]>` where the test wants `never`; challenge 16's `T extends [...infer I, infer _] ? I : never` (+20) gives `never` for `Pop<[]>` where the test wants `[]`. These are not careless answers, and the authors are not the story: the test cases were tightened *after* the answers were posted, and the answers were correct when written. The story is why the empty tuple is where they broke. In a conditional type, the else-branch is doing double duty: it means both "`T` is not a tuple with a last element" and "`T` is the empty tuple", because pattern matching against `[...infer I, infer _]` is the only iteration primitive available and a failed match is the only way to notice either fact. Writing `: never` there looks like marking an impossible branch; it is silently also answering the empty case. Of the nineteen distinct `Pop` answers I tested, twelve return `never` and fail, and the passing ones differ only in writing `: []`. The builder cannot make this mistake, and not because its author is more careful: `tupleElements` throws for a non-tuple and `slice(0, -1)` returns `[]` for the empty one, so the two facts are two code paths and neither can stand in for the other.

A third answer has since rotted the same way. Challenge 949's top-voted `AnyOf` (+83) tests `T[number] extends 0 | '' | false | [] | {[key: string]: never}`, and fails now that the harness added `undefined` and `null` to its all-falsy case. The builder is not immune to *that* one, since the falsy set must be enumerated by hand in both languages, and the passing answer is the same union plus two members. But it belongs to the same family: a `never` or a falsy list standing in for a case nobody enumerated, invisible until a test names it. Which points at the shape underneath. Across this batch, `Permutation` ends in `: never` on a branch that cannot be reached, `Merge` ends in `: never` on a branch that cannot be reached, and `Permutation` opens with `K extends K`, a tautology whose only job is to trigger distribution. None of those tokens mean anything; they are there because a conditional type needs an else and a distribution needs a naked parameter. Unreachable code that the language forces you to write is unreachable code nobody checks, and it is exactly where these answers rot.

**Parameter names cannot be part of a function type's identity.** Challenge 191 requires `appendArgument((a: uint32, b: string) => uint32, boolean)` to equal `(a: uint32, b: string, x: boolean) => uint32`. The builder appends a parameter and has no way to know it should be called `x`; under an identity that included parameter names, the challenge would be unpassable by any builder, and TypeScript passes it because function type identity ignores names. So `makeType` must canonicalize parameter names away, while `FunctionParameterReflection` must keep them, since diagnostics, hovers, and `parameters` all want them. That is exactly the shape of the provenance problem [type programming](../typeprogramming.md) already solves for documentation: information that reflection carries and identity ignores. Parameter names belong in the same non-canonical class as `origin`, and the same question follows them, namely what reflection reports after two differently-named signatures intern to one object. The provenance rule (union on merge, construction order for display) is the available answer and should be stated for names too.

**Mapped types over tuples are a special case, and the special case is invisible.** Challenge 20's entire solution rests on `{ [K in keyof T]: ... }` producing a *tuple* when `T` is a tuple, rather than an object with keys `'0' | '1' | '2' | 'length' | 'map' | ...`. Nothing in that syntax says so; it is a rule in the checker that mapped types over array and tuple types are rewritten element-wise. The builder's `switch` on `node.kind` states the same rule in the place where it applies, which is the difference between a language feature you must know and a line you can read.

**Intersections versus flat objects.** Challenge 8 weakens its assertions from `Equal` to `Alike` because `Omit<T, K> & Readonly<Pick<T, K>>` is inter-assignable with the intended object but not identical to it. The builder edits records in place, returns the flat object, and passes the stronger assertion. That the harness had to weaken a test to accommodate the idiomatic TypeScript answer is small but telling: the type language's shape leaks into what the test is allowed to say.

**Constraints versus checks.** These challenges are stated as generic *aliases* with constraints (`MyPick<T, K extends keyof T>`), and the builders are *functions*, so a bad argument is a thrown `TypeError` at the call rather than a constraint violation at the declaration. The two are not far apart: when a builder appears in a signature, the computed constraint form (`<T, K: keysOf(T)>`) restores the declaration-site check, and the thrown message is available in both cases. What the builder gives up is being checkable before its arguments are known; what it gains is saying which property was missing.

**Where TypeScript wins, or draws.** Concat, Push, and Unshift: three of forty, and all three the same win. Variadic tuple spread is purpose-built syntax, `[U, ...T]` says it in six characters, and no reflection call will be shorter. Two more are honest draws. Absolute (529) stringifies and strips a minus in both languages, because the challenge specifies a string result and neither is doing arithmetic. AnyOf (949) is the more interesting one: truthiness is a property of values, but the challenge has no values, so the falsy set is enumerated by hand either way and all that differs is `.some` against recursion. When a problem is genuinely about types rather than about values wearing types, the two models converge, and that is worth knowing about the ones where they do not. The pattern across the set is consistent. Where TypeScript has syntax for an operation on types, it beats a builder; where it must *encode* a computation (identity as a signature relation, iteration as recursive peeling, a duplicate-key check as a parameter that becomes `never`, a base case as `keyof T extends never`, whitespace as a hand-enumerated union of three characters), the encoding is precisely what the builder deletes.
