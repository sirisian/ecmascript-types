# Type Challenges

[type-challenges](https://github.com/type-challenges/type-challenges) is the standard obstacle course for TypeScript's type-level language: implement `Pick`, unwrap a `Promise`, take the head of a tuple, using only conditional types, mapped types, `infer`, and recursion. It is a good adversarial test of [type builders](../typeprogramming.md), because the problems were *chosen* to be natural in a type language whose only verbs are `extends` and `in`. If the builder model only won on problems selected to suit it, that would prove nothing.

This document works every challenge in the repository: the warm-up, all thirteen easy, all one hundred and four medium, all fifty-five hard, and all seventeen extreme, in the order the README lists them. One hundred and ninety in total. Every TypeScript solution shown has been checked against the repo's own test cases with TypeScript 5.9.3; where the top-voted answer no longer passes, a verified passing answer is shown instead and the note at the end says why. Each entry gives the problem, the top-voted community solution in TypeScript (linked, with its score at the time of writing), and the builder solution. The builders are written from the primitives in [type programming](../typeprogramming.md) rather than by calling the `std:types` entry that already ships the answer, since implementing the utility is the whole point of the exercise.

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
import { arms, arrayOf, awaited, firstParameter, fn, keysOf, literal, literalValues, mapProperties,
         never, objectOf, partial, prop, readonly, reflect, required, returnType, stringPattern,
         tupleOf, union, withThisType } from 'std:types';

// Two local helpers the challenges below share.
function tupleElements(T: type): [].<Reflect.TypeTupleElement> {
  const node = reflect(T);
  if (node.kind !== 'tuple') throw new TypeError(`expected a tuple type, got ${String(T)}`);
  return node.elements;
}

function perms(xs: [].<type>): [].<[].<type>> {
  return xs.length === 0 ? [[]] : xs.flatMap((x, i) => perms(xs.toSpliced(i, 1)).map(rest => [x, ...rest]));
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
// TypeScript, issue #5896 (+34)
type TupleToObject<T extends readonly (string | symbol | number)[]> = {
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

The symbol case is the interesting one. The most-copied TypeScript solutions constrain `T extends readonly string[]` and fail the harness's own symbol and number cases; the answer shown widens the constraint to `readonly (string | number | symbol)[]`, and that widening is the entire fix. Here the key check is a line of code that names what a property key is, and symbol literal types make the symbol case fall out rather than need a workaround. On numbers, see the note below.

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
// TypeScript, issue #2066 (+19)
type MyEqual<X, Y> = (<T>() => T extends X ? 1 : 2 ) extends (<T>() => T extends Y ? 1 : 2) ? true : false

type Includes<T extends readonly any[], U> = true extends {
  [I in keyof T]: MyEqual<T[I], U>
}[number] ? true : false
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

This is the single clearest case in the whole set. The challenge needs **identity**, and TypeScript's only relation is assignability (`extends`), so every passing solution imports or restates the `IsEqual` incantation: two generic function signatures whose relation the checker happens to decide the right way, discovered in a 2018 issue thread and never blessed as a feature. The mapped type in `Includes` then exists only because there is no way to ask a tuple a question directly: turn every element into a verdict, index by `number`, and let `true extends` scan the union.

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
// TypeScript, issue #52 (+37)
type DeepReadonly<T> = {
  readonly [k in keyof T]: T[k] extends Record<any, any>
    ? T[k] extends Function
      ? T[k]
      : DeepReadonly<T[k]>
    : T[k]
}
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

The TypeScript answer stops at a function property (`a: () => 22`) by saying so twice: `T[k] extends Record<any, any>` is how the language asks whether something is an object, a question it has no word for, and `T[k] extends Function` walks that back for the callables the first test also catches. The builder's `switch` states its domain, and the `default` arm is the base case that a reader can check against the list of type kinds.

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
// TypeScript, issue #598 (+19)
type Chainable<T = {}> = {
  option<K extends string, V>(key: Exclude<K, keyof T>, value: V): Chainable<Omit<T,K> & Record<K, V>>;
  get(): T;
};
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

The last challenge in this batch is the one that most resembles a real API, and it separates the two models cleanly. `K` is an argument-bound value generic, so the *string* `'foo'` is available to `withKey` as a string, and the accumulation is `[...properties, prop(key, V)]`. TypeScript cannot pass a key to a function, so it encodes the duplicate check into the parameter's own type: `Exclude<K, keyof T>` makes the parameter `never` when the key is already present, so the *argument* fails to be assignable and the call errors. It works, and the error it produces is that `'name'` is not assignable to `never`.

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
// TypeScript, issue #508 (+27)
type Awaited<T> = T extends Promise<infer R> ? Awaited<R> : T

declare function PromiseAll<T extends any[]>(values: readonly [...T]): Promise<{
  [P in keyof T]: Awaited<T[P]>
}>
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

`{ [K in keyof T]: ... }` over a tuple is TypeScript's most surprising rule: a mapped type over an array or tuple is special-cased to produce an array or tuple rather than an object with numeric keys, which is why this signature works at all and why nothing in the syntax says so. The builder's `switch` says so. Note also that `awaited` already means "unwrap if thenable, else pass through", so the answer's recursive helper `Awaited<T> = T extends Promise<infer R> ? Awaited<R> : T` is a call rather than a declaration.

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
// TypeScript, issue #3553 (+2)
type AppendArgument<Fn extends (...args:any) => void, A> = (...args:[...Parameters<Fn>,A]) => ReturnType<Fn>
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

Read the first assertion again: the builder names its new parameter `appended`, the expected type names it `x`, and they are equal. That is only true if parameter names are not part of a function type's identity, which the note below takes up. Note also that the builder appends to *every* overload, which the `Parameters`/`ReturnType` form cannot express at all: on an overloaded function those utilities silently pick the last signature and discard the rest.

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
  return union(perms(arms(T)).map(tupleOf));   // perms is in the preamble
}

permutation(type 'A') === type ['A'];
permutation(type 'A' | 'B' | 'C')
  === type ['A','B','C'] | ['A','C','B'] | ['B','A','C'] | ['B','C','A'] | ['C','A','B'] | ['C','B','A'];
permutation(boolean) === type [false, true] | [true, false];
permutation(never) === type [];
```

The most-upvoted answer in the whole repository, and it deserves the votes: it is genuinely clever. Look at what the cleverness is *for*, though. `K extends K` is a conditional that is always true. It computes nothing. It exists solely because a conditional type over a naked parameter distributes, so writing a tautology is the only way to say "iterate the arms of this union", and the `: never` branch is unreachable filler needed to complete a syntax form. `[T] extends [never]` is the tuple-wrapping idiom, present because the author needs the *opposite* of distribution for the base case. Two idioms, both about controlling a behavior nobody asked for, wrapped around the actual algorithm.

The builder's `perms` (in the preamble, and reused verbatim by challenge 21220) is the permutation function anyone would write, and `.flatMap` is the iteration. `permutation(never)` needs no base case at all: `arms(never)` is the empty list, `perms([])` is `[[]]`, and the union of one tuple is that tuple.

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

## 1042 · IsNever

Is `T` exactly `never`?

```ts
// TypeScript, issue #1796 (+25)
type IsNever<T> = [T] extends [never] ? true : false
```

```js
// Builder
function isNever(T: type): type {
  return T === never ? type true : type false;
}

isNever(never) === type true;
isNever(type never | string) === type false;   // never is the union identity, so this is just string
isNever(type '') === type false;
isNever(type []) === type false;
```

The tuple wrapping is not decoration. `T extends never ? true : false` distributes, and distributing over `never` means mapping over a union with no arms, which produces no arms: `IsNever<never>` would be `never`, not `true`. So the answer to "is this the empty union" is unreachable by the mechanism that would naturally ask it, and `[T] extends [never]` is the workaround. `T === never` is a pointer comparison.

## 1097 · IsUnion

Is `T` a union?

```ts
// TypeScript, issue #1140 (+86)
type IsUnionImpl<T, C extends T = T> = (T extends T ? C extends T ? true : unknown : never) extends true ? false : true;
type IsUnion<T> = IsUnionImpl<T>;
```

```js
// Builder
function isUnion(T: type): type {
  const node = reflect(T);
  return node.kind === 'union' && node.arms.length > 1 ? type true : type false;
}

isUnion(string) === type false;
isUnion(type string | uint32) === type true;
isUnion(type 'a' | 'b' | 'c') === type true;
isUnion(type { a: string | uint32 }) === type false;
isUnion(never) === type false;                 // the empty union has no arms
isUnion(type string | never) === type false;   // never vanishes, leaving one arm
```

`IsUnionImpl` is the most indirect solution in the document. It smuggles a copy of `T` into `C` before distribution starts, distributes `T` while holding `C` fixed, and then detects union-ness from whether the result is uniformly `true`, because *whether a conditional distributed* is the only observable that reveals arm count. There is no `kind` to read. The builder reads `kind`.

This challenge also asks a question of the proposal that the [type programming](../typeprogramming.md) document does not answer, which the note below takes up: the harness expects `IsUnion<string | 'a'>` to be `false`.

## 1130 · ReplaceKeys

In a union of object types, replace the types of the named keys, per a lookup object. A key named but absent from the lookup becomes `never`.

```ts
// TypeScript, issue #3570 (+17)
type ReplaceKeys<U, T, Y> = { [K in keyof U]: K extends T ? K extends keyof Y ? Y[K] : never : U[K] }
```

```js
// Builder
function replaceKeys(U: type, T: type, Y: type): type {
  const targets = new Set(literalValues(T));
  const replacements = reflect(Y).properties;
  return mapProperties(U, p => targets.has(p.name)
    ? { ...p, type: replacements.find(r => r.name === p.name)?.type ?? never }
    : p);
}

type NodeA = { type: 'A', name: string, flag: uint32 };
type NodeB = { type: 'B', id: uint32, flag: uint32 };
replaceKeys(type NodeA | NodeB, type 'name' | 'flag', type { name: uint32, flag: string })
  === type { type: 'A', name: uint32, flag: string } | { type: 'B', id: uint32, flag: string };
```

Distribution over the union is `mapProperties`, which does it because someone wrote the line. The `?? never` and the `: never` are the same decision spelled two ways.

## 1367 · Remove Index Signature

Drop index signatures, keep declared members.

```ts
// TypeScript, issue #14662 (+26)
type RemoveIndexSignature<T, P=PropertyKey> = {
  [K in keyof T as P extends K? never : K extends P ? K : never]: T[K]
}
```

```js
// Builder
function removeIndexSignature(T: type): type {
  return objectOf(reflect(T).properties, []);
}

type Foo = { [key: string]: any, foo(): void };
removeIndexSignature(Foo) === type { foo(): void };

const foobar = Symbol('foobar');
type FooBar = { [key: symbol]: any, [foobar](): void };
removeIndexSignature(FooBar) === type { [foobar](): void };   // the symbol-keyed member survives
```

The best single vindication of a node model decision in the whole document. TypeScript's `keyof T` *merges* index signature keys and declared keys into one union, so `'foo' | string` collapses to `string` and the information about which key came from where is destroyed. Recovering it requires `P extends K`, which asks "is `K` so wide that `PropertyKey` fits inside it", because being wide is the only surviving trace of having come from an index signature. Then `K extends P` handles the symbol case, and the double conditional is really a hand-rolled discriminator for a distinction the type language threw away.

In the reflection model, `properties` and `indexSignatures` are separate fields on the object node. They were never merged, so nothing needs recovering: pass the properties, pass an empty signature list. This is also why `keysOf` has to fold signature key types in explicitly, which looked like a wart when it was written and is the same fact seen from the other side.

## 1978 · Percentage Parser

Split a string into sign, number, and percent sign.

```ts
// TypeScript, issue #3788 (+56)
type CheckPrefix<T> = T extends '+' | '-' ? T : never;
type CheckSuffix<T> =  T extends `${infer P}%` ? [P, '%'] : [T, ''];
type PercentageParser<A extends string> = A extends `${CheckPrefix<infer L>}${infer R}` ? [L, ...CheckSuffix<R>] : ['', ...CheckSuffix<A>];
```

```js
// Builder
function percentageParser(s: string): type {
  const [, sign, digits, percent] = /^([+-]?)(.*?)(%?)$/.exec(s);   // every part optional: always matches
  return tupleOf([literal(sign), literal(digits), literal(percent)]);
}

percentageParser('+100%') === type ['+', '100', '%'];
percentageParser('-100') === type ['-', '100', ''];
percentageParser('%') === type ['', '', '%'];
percentageParser('') === type ['', '', ''];
```

`CheckPrefix<infer L>` is worth staring at: an `infer` nested inside a call to another conditional type, inside a template pattern. It works because `infer L` binds first and `CheckPrefix` then filters it, which is a mechanism most TypeScript users could not describe on request. The builder's regex says the same grammar in the notation designed for grammars.

## 2070 · Drop Char

Remove every occurrence of a character.

```ts
// TypeScript, issue #2074 (+48)
type DropChar<S, C extends string> = S extends `${infer L}${C}${infer R}` ? DropChar<`${L}${R}`, C> : S;
```

```js
// Builder
function dropChar(s: string, c: string): type {
  return literal(s.replaceAll(c, ''));
}

dropChar('butter fly!', ' ') === type 'butterfly!';
dropChar(' b u t t e r f l y ! ', 't') === type ' b u   e r f l y ! ';
```

## 2257 · MinusOne

`n` minus one.

```ts
// TypeScript, issue #13507 (+48)
type ParseInt<T extends string> = T extends `${infer Digit extends number}` ? Digit : never
type ReverseString<S extends string> = S extends `${infer First}${infer Rest}` ? `${ReverseString<Rest>}${First}` : ''
type RemoveLeadingZeros<S extends string> = S extends '0' ? S : S extends `${'0'}${infer R}` ? RemoveLeadingZeros<R> : S
type InternalMinusOne<
  S extends string
> = S extends `${infer Digit extends number}${infer Rest}` ?
    Digit extends 0 ?
      `9${InternalMinusOne<Rest>}` :
    `${[9, 0, 1, 2, 3, 4, 5, 6, 7, 8][Digit]}${Rest}`:
  never
type MinusOne<T extends number> = ParseInt<RemoveLeadingZeros<ReverseString<InternalMinusOne<ReverseString<`${T}`>>>>>
```

```js
// Builder
function minusOne(n: float64): type {
  return literal(n - 1);
}

minusOne(1) === type 0;
minusOne(55) === type 54;
minusOne(1101) === type 1100;
minusOne(9_007_199_254_740_992) === type 9_007_199_254_740_991;
```

The comparison that needs no commentary, so here is a little anyway. TypeScript has no arithmetic on numbers, so it stringifies the literal, reverses the digits, walks them least-significant first, borrows through zeros by emitting `9` and recursing, decrements the first non-zero digit through a lookup table indexed by the digit itself, reverses back, strips leading zeros, and parses the result to a number. Five helper types, a digit table, and three passes over a string, to subtract one.

The usual defense of this kind of thing is that the encoding buys range, and here it buys none: `${T}` starts from a `number` literal, so the input is a float64 in both languages and `MinusOne<9007199254740993>` is already `9007199254740992` before either solution runs. The builder's parameter can simply be declared `int64` or a bigint if exactness past 2^53 is wanted, and `n - 1` does not change. That is the difference between a language with numbers and a language that has to spell them.

## 2595 · PickByType

Keep the properties whose type is assignable to `U`.

```ts
// TypeScript, issue #2768 (+18)
type PickByType<T, U> = { [P in keyof T as T[P] extends U ? P : never]: T[P] }
```

```js
// Builder
function pickByType(T: type, U: type): type {
  return mapProperties(T, p => Reflect.isAssignable(p.type, U) ? p : null);
}

interface Model { name: string; count: uint32; isReadonly: boolean; isEnable: boolean }
pickByType(Model, boolean) === type { isReadonly: boolean, isEnable: boolean };
pickByType(Model, string) === type { name: string };
```

`std:types` ships this as `pickByValue`. Filtering a list is filtering a list.

## 2688 · StartsWith

```ts
// TypeScript, issue #2690 (+16)
type StartsWith<T extends string, U extends string> = T extends `${U}${string}`?true:false
```

```js
// Builder
function startsWith(s: string, prefix: string): type {
  return s.startsWith(prefix) ? type true : type false;
}

startsWith('abc', 'ab') === type true;
startsWith('abc', 'ac') === type false;
startsWith('abc', '') === type true;
```

## 2693 · EndsWith

```ts
// TypeScript, issue #19997 (+2)
type EndsWith<T,b extends string> = T extends `${infer f}${b}`?true:false
```

```js
// Builder
function endsWith(s: string, suffix: string): type {
  return s.endsWith(suffix) ? type true : type false;
}

endsWith('abc', 'bc') === type true;
endsWith('abc', 'ac') === type false;
endsWith('abc', '') === type true;
```

The pair is worth showing together for one detail: `StartsWith` can be written with a plain pattern, `` `${U}${string}` ``, but `EndsWith` cannot, because a template literal type may not begin with an unanchored `${string}` followed by a bound parameter. The answer is `` `${infer f}${b}` ``, which introduces an inference variable `f` that is never used, purely to occupy the leading position. `String.prototype.startsWith` and `endsWith` are symmetric because nothing about the underlying question is asymmetric.

## 2757 · PartialByKeys

Make only the named keys optional, defaulting to all of them.

```ts
// TypeScript, issue #20885 (+1)
type Flatten<T> = {
  [key in keyof T]: T[key];
};

type PartialByKeys<T, K extends keyof T = keyof T> = Flatten<
  Omit<T, K> & {
    [P in K]?: T[P];
  }
>;
```

```js
// Builder
function partialByKeys(T: type, K: type = keysOf(T)): type {
  const keys = new Set(literalValues(K));
  return mapProperties(T, p => keys.has(p.name) ? { ...p, optional: true } : p);
}

interface User { name: string; age: uint32; address: string }
partialByKeys(User, type 'name') === type { name?: string, age: uint32, address: string };
partialByKeys(User, type 'name' | 'age') === type { name?: string, age?: uint32, address: string };
partialByKeys(User) === partial(User);
```

`Flatten` is a no-op that exists to be a no-op: mapping an intersection over its own keys forces the checker to flatten `A & B` into a single object type, because the challenge's expected value is flat and `A & B` is not identical to it. That is the same fact challenge 8 hit, met here with a helper instead of a weakened assertion. The builder never forms the intersection, so there is nothing to flatten.

## 2759 · RequiredByKeys

The mirror image.

```ts
// TypeScript, issue #3180 (+10)
type RequiredByKeys<
  T,
  K extends keyof T = keyof T,
  O = Omit<T, K> & { [P in K]-?: T[P] }
> = { [P in keyof O]: O[P] }
```

```js
// Builder
function requiredByKeys(T: type, K: type = keysOf(T)): type {
  const keys = new Set(literalValues(K));
  return mapProperties(T, p => keys.has(p.name) ? { ...p, optional: false } : p);
}

interface User { name?: string; age?: uint32; address?: string }
requiredByKeys(User, type 'name') === type { name: string, age?: uint32, address?: string };
requiredByKeys(User) === required(User);
```

The two challenges are one function with `optional` set to `true` or `false`, which is what a modifier being a field on a record means. TypeScript needs `Omit`, an intersection, `-?`, and a third type parameter defaulted to the intermediate result so the final mapped type can flatten it.

## 2793 · Mutable

Strip `readonly`.

```ts
// TypeScript, issue #27920 (+2)
type Mutable<T extends object> = {
  -readonly[K in keyof T]: T[K]
}
```

```js
// Builder
function mutable(T: type): type {
  return mapProperties(T, p => ({ ...p, readonly: false }));
}

interface Todo { title: string; description: string; completed: boolean }
mutable(readonly(Todo)) === Todo;
```

The fourth member of the modifier family, and the fourth one-liner. `std:types` ships it, and the challenge exists because `lib.d.ts` does not.

## 2852 · OmitByType

Drop the properties whose type is assignable to `U`.

```ts
// TypeScript, issue #3021 (+7)
type OmitByType<T, U> = {
  [P in keyof T as T[P] extends U ? never : P]: T[P]
}
```

```js
// Builder
function omitByType(T: type, U: type): type {
  return mapProperties(T, p => Reflect.isAssignable(p.type, U) ? null : p);
}

interface Model { name: string; count: uint32; isReadonly: boolean; isEnable: boolean }
omitByType(Model, boolean) === type { name: string, count: uint32 };
```

`pickByType` with the branches swapped, in both languages.

## 2946 · ObjectEntries

The union of an object's key-value pairs. An optional property's value type includes `undefined`.

```ts
// TypeScript, issue #19507 (+4). The top-voted answer (#14052, +18) strips `undefined`
// and no longer passes; eight of the next nine fail the same way. See the note.
type ObjectEntries<T extends Record<PropertyKey, any>, K = keyof T> =
  K extends keyof T
    ? [K, T[K]]
    : never
```

```js
// Builder
function objectEntries(T: type): type {
  return union(reflect(T).properties.map(p =>
    tupleOf([literal(p.name), p.optional ? union([p.type, type undefined]) : p.type])));
}

interface Model { name: string; age: uint32; locations: [].<string> | null }
objectEntries(Model) === type ['name', string] | ['age', uint32] | ['locations', [].<string> | null];
objectEntries(partial(Model))
  === type ['name', string | undefined] | ['age', uint32 | undefined] | ['locations', [].<string> | null | undefined];
objectEntries(type { key?: undefined }) === type ['key', undefined];
```

The `optional ? union([p.type, undefined]) : p.type` line is the same policy decision `indexed` makes in [type programming](../typeprogramming.md), and it is worth noticing that this challenge is *about* that decision. TypeScript reaches the same answer through `T[K]`, whose behavior on an optional property is fixed by a compiler flag rather than by the code in front of you, which is most of why the published answers disagree about it.

## 3062 · Shift

The tuple without its first element. `Shift<[]>` is `[]`.

```ts
// TypeScript, issue #6732 (+0). The top-voted answer (#3134, +4) returns `never`
// for the empty tuple; fifteen of the twenty answers do. See the note.
type Shift<T extends any[]> = T extends [infer F, ...infer R] ? R : [];
```

```js
// Builder
function shift(T: type): type {
  return Reflect.makeType({ kind: 'tuple', elements: tupleElements(T).slice(1) });
}

shift(type [3, 2, 1]) === type [2, 1];
shift(type [1]) === type [];
shift(type []) === type [];
shift(uint32);   // TypeError: expected a tuple type, got uint32
```

## 3188 · Tuple to Nested Object

`['a', 'b']` and `number` become `{ a: { b: number } }`.

```ts
// TypeScript, issue #3282 (+27)
type TupleToNestedObject<T, U> = T extends [infer F,...infer R]?
  {
    [K in F&string]:TupleToNestedObject<R,U>
  }
  :U
```

```js
// Builder
function tupleToNestedObject(T: type, U: type): type {
  return tupleElements(T).reduceRight((acc, e) => objectOf([prop(literalValues(e.type)[0], acc)]), U);
}

tupleToNestedObject(type ['a'], string) === type { a: string };
tupleToNestedObject(type ['a', 'b', 'c'], boolean) === type { a: { b: { c: boolean } } };
tupleToNestedObject(type [], boolean) === boolean;
```

A fold, and `reduceRight` is the fold that builds inward-out, so the base case is the seed and the empty tuple returns `U` without a branch. `F & string` in the TypeScript version is a cast in disguise: `F` is known to be a key but the mapped type's `in` clause requires something the checker will accept as a key type, so intersecting with `string` narrows it. `literalValues(e.type)[0]` reads the key.

## 3192 · Reverse

```ts
// TypeScript, issue #3210 (+21)
type Reverse<T extends any[]> = T extends [infer F, ...infer Rest] ? [...Reverse<Rest>, F] : T;
```

```js
// Builder
function reverse(T: type): type {
  return Reflect.makeType({ kind: 'tuple', elements: tupleElements(T).toReversed() });
}

reverse(type ['a', 'b', 'c']) === type ['c', 'b', 'a'];
reverse(type []) === type [];
```

## 3196 · Flip Arguments

Reverse a function's parameters.

```ts
// TypeScript, issue #6735 (+7)
type Reverse<T extends unknown[]> = T extends [infer F, ...infer R] ? [...Reverse<R>, F] : [];

type FlipArguments<T extends (...args: any[]) => any> = T extends (...args: infer P) => infer U
? (...args: Reverse<P>) => U
: never;
```

```js
// Builder
function flipArguments(F: type): type {
  const node = reflect(F);
  if (node.kind !== 'function') throw new TypeError(`flipArguments: ${String(F)} is not a function type`);
  return Reflect.makeType({ ...node, signatures: node.signatures.map(sig => ({
    ...sig, parameters: sig.parameters.toReversed().map((p, index) => ({ ...p, index })),
  })) });
}

flipArguments(type (arg0: string, arg1: uint32, arg2: boolean) => void)
  === type (arg0: boolean, arg1: uint32, arg2: string) => void;
flipArguments(type () => boolean) === type () => boolean;
flipArguments(type 'string');   // TypeError: flipArguments: 'string' is not a function type
```

An honest tax on the builder: parameter records carry an `index`, so reversing them means renumbering, and forgetting the `.map((p, index) => ({ ...p, index }))` would produce a signature whose parameters disagree with their own positions. TypeScript's `Reverse<P>` on a tuple has no such field to keep consistent. Richer reflection means more invariants to maintain, and this is the smallest possible example of that bill arriving.

## 3243 · FlattenDepth

Flatten, but only `depth` levels.

```ts
// TypeScript, issue #15373 (+56)
type FlattenDepth<
  T extends any[],
  S extends number = 1,
  U extends any[] = []
> = U['length'] extends S
  ? T
  : T extends [infer F, ...infer R]
  ? F extends any[]
    ? [...FlattenDepth<F, S, [...U, 1]>, ...FlattenDepth<R, S, U>]
    : [F, ...FlattenDepth<R, S, U>]
  : T
```

```js
// Builder
function flattenDepth(T: type, depth: uint32 = 1): type {
  const flat = (elements: [].<Reflect.TypeTupleElement>, d: uint32): [].<Reflect.TypeTupleElement> =>
    d === 0 ? elements : elements.flatMap(e =>
      reflect(e.type).kind === 'tuple' ? flat(reflect(e.type).elements, d - 1) : [e]);
  return Reflect.makeType({ kind: 'tuple', elements: flat(tupleElements(T), depth) });
}

flattenDepth(type [1, [2]]) === type [1, 2];
flattenDepth(type [1, 2, [3, 4], [[[5]]]], 2) === type [1, 2, 3, 4, [5]];
flattenDepth(type [1, 2, [3, 4], [[[5]]]]) === type [1, 2, 3, 4, [[5]]];
flattenDepth(type [1, [2, [3, [4, [5]]]]], 19_260_817) === type [1, 2, 3, 4, 5];
```

`U extends any[] = []` is a counter. Not a number: a *tuple*, carried through the recursion purely so that `U['length']` can be compared against `S`, and incremented by `[...U, 1]`, because a tuple's length is the only number TypeScript can produce by construction. The builder's counter is `d`, and it decrements.

The last case is worth explaining rather than gawking at, because it is the one that makes the TypeScript version defensible. A depth of 19,260,817 does not build a 19-million-element tuple: `U` only grows when the recursion descends into a nested array, so the counter is bounded by the input's nesting depth, and the structure runs out long before the limit does. It is a correct and rather elegant piece of engineering. It is also a counter made of tuples.

## 3326 · BEM style string

Build `block__element--modifier` strings from a block, a list of elements, and a list of modifiers.

```ts
// TypeScript, issue #5369 (+30)
type BEM<B extends string, E extends string[],M extends string[]> = `${B}${E extends [] ? '' : `__${E[number]}`}${M extends [] ? '' : `--${M[number]}`}`
```

```js
// Builder
function bem(block: string, elements: [].<string>, modifiers: [].<string>): type {
  const parts = (list: [].<string>, sep: string): [].<string> =>
    list.length === 0 ? [''] : list.map(x => `${sep}${x}`);
  return union(parts(elements, '__').flatMap(e =>
    parts(modifiers, '--').map(m => literal(`${block}${e}${m}`))));
}

bem('btn', ['price'], []) === type 'btn__price';
bem('btn', ['price'], ['warning', 'success']) === type 'btn__price--warning' | 'btn__price--success';
bem('btn', [], ['small', 'medium', 'large']) === type 'btn--small' | 'btn--medium' | 'btn--large';
```

`E[number]` is TypeScript's tuple-to-union, and putting it inside a template literal is what produces the cross product, because a template literal over a union distributes into every combination. That is a real feature and it is doing genuine work here. The builder's cross product is a `flatMap` over a `map`, which is the same shape written out.

## 3376 · InorderTraversal

Inorder traversal of a binary tree type.

```ts
// TypeScript, issue #5220 (+24)
interface TreeNode {
  val: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

type InorderTraversal<T extends TreeNode | null, NT extends TreeNode = NonNullable<T>> = T extends null
  ? []
  : [...InorderTraversal<NT['left']>, NT['val'], ...InorderTraversal<NT['right']>]
```

```js
// Builder
interface TreeNode { val: float64; left: TreeNode | null; right: TreeNode | null }

function inorderTraversal(T: type): type {
  const field = (node, name) => node.properties.find(p => p.name === name).type;
  const walk = (t: type): [].<type> => {
    if (t === type null) return [];
    const node = reflect(t);
    return [...walk(field(node, 'left')), field(node, 'val'), ...walk(field(node, 'right'))];
  };
  return tupleOf(walk(T));
}

type Tree = { val: 1, left: null, right: { val: 2, left: { val: 3, left: null, right: null }, right: null } };
inorderTraversal(Tree) === type [1, 3, 2];
inorderTraversal(type null) === type [];
```

A textbook tree walk in both languages, and the shapes match closely enough that this is a fair draw on structure. The one artifact is `NT extends TreeNode = NonNullable<T>`: a second type parameter defaulted to a computed value, existing because there is no `const` binding in a type alias, so a value used twice must be introduced as a parameter. The builder's `node` is a `const`.

## 4179 · Flip

Swap keys and values.

```ts
// TypeScript, issue #14094 (+22)
type Flip<T extends Record<string, string | number | boolean>> = {
  [P in keyof T as `${T[P]}`]: P
}
```

```js
// Builder
function flip(T: type): type {
  return objectOf(reflect(T).properties.map(p => {
    const node = reflect(p.type);
    if (node.kind !== 'literal') throw new TypeError(`flip: ${String(p.type)} cannot be a key`);
    return prop(String(node.value), literal(p.name));
  }));
}

flip(type { pi: 'a' }) === type { a: 'pi' };
flip(type { pi: 3.14, bool: true }) === type { 3.14: 'pi', true: 'bool' };
flip(type { prop1: 'val1', prop2: 'val2' }) === type { val1: 'prop1', val2: 'prop2' };
```

The `` `${T[P]}` `` is not cosmetic: a key must be a string, number, or symbol, and `T[P]` might be a boolean, so the template literal is a *cast to string* wearing a template literal's clothes. `String(node.value)` is the same cast, and it is spelled as one. The published answer's first attempt omits the constraint and does not compile.

## 4182 · Fibonacci Sequence

The nth Fibonacci number.

```ts
// TypeScript, issue #4204 (+29)
type Fibonacci<T extends number, CurrentIndex extends any[] = [1], Prev extends any[] = [], Current extends any[] = [1]> = CurrentIndex['length'] extends T
  ? Current['length']
  : Fibonacci<T, [...CurrentIndex, 1], Current, [...Prev, ...Current]>
```

```js
// Builder
function fibonacci(n: uint32): type {
  let [prev, current] = [0, 1];
  for (let i = 1; i < n; i++) [prev, current] = [current, prev + current];
  return literal(current);
}

fibonacci(1) === type 1;
fibonacci(3) === type 2;
fibonacci(8) === type 21;
```

Three of the four type parameters are accumulators, and all three are tuples standing in for numbers: `CurrentIndex` is the loop counter, `Prev` and `Current` are the running values, `[...Prev, ...Current]` is addition performed by concatenating lists whose lengths are the operands, and `Current['length']` at the end is how a tuple is converted back into a number. Every idea in the algorithm is present; every quantity in it is a list. The builder writes the loop, and `Fibonacci<100>` is a compile error in one language and instant in the other.

## 4260 · AllCombinations

Every combination of a string's characters, in any order, without repeats.

```ts
// TypeScript, issue #5339 (+26)
type String2Union<S extends string> =
  S extends `${infer C}${infer REST}`
  ? C | String2Union<REST>
  : never;

type AllCombinations<
  STR extends string,
  S extends string = String2Union<STR>,
> = [S] extends [never]
  ? ''
  : '' | {[K in S]: `${K}${AllCombinations<never, Exclude<S, K>>}`}[S];
```

```js
// Builder
function allCombinations(s: string): type {
  const combos = (chars: [].<string>): [].<string> =>
    ['', ...chars.flatMap((c, i) => combos(chars.toSpliced(i, 1)).map(rest => `${c}${rest}`))];
  return union([...new Set(combos([...s]))].map(literal));
}

allCombinations('') === type '';
allCombinations('A') === type '' | 'A';
allCombinations('AB') === type '' | 'A' | 'B' | 'AB' | 'BA';
```

`{[K in S]: ...}[S]` is the mapped-type-then-index idiom, which is how you map over a union when the language only lets you map over keys: build an object keyed by the union, then immediately index it back out. The object is scaffolding, discarded in the same expression that creates it. `flatMap` needs no scaffolding, and the `Set` is there because the builder must deduplicate what union canonicalization would have deduplicated anyway.

## 4425 · Greater Than

Is `T` greater than `U`?

```ts
// TypeScript, issue #21721 (+6). Every tuple-counting answer, including the top-voted
// (#5077, +11), fails the large case with "excessively deep"; see the note.
type GreaterThan<T extends number | string, U extends number | string, Res = false> =
  `${T}` extends `${infer TF}${infer TR}`
    ? `${U}` extends `${infer UF}${infer UR}`
      ? [Res, TF & UF] extends [false, never] // Res == false and TF != UF
        ? GreaterThan<TR, UR, '0123456789' extends `${string}${TF}${string}${UF}${string}` ? false : true>
        : GreaterThan<TR, UR, Res>
      : true
    : U extends '' ? Res : false
```

```js
// Builder
function greaterThan(a: float64, b: float64): type {
  return a > b ? type true : type false;
}

greaterThan(1, 0) === type true;
greaterThan(20, 20) === type false;
greaterThan(10, 100) === type false;
greaterThan(1_234_567_891_011, 1_234_567_891_010) === type true;
```

The best single artifact in this document. Read the digit comparison: to decide whether `TF` is greater than `UF`, it asks whether the literal string `'0123456789'` matches the pattern `` `${string}${TF}${string}${UF}${string}` ``, which is true exactly when `TF` appears before `UF` in the digits. It compares two numbers by searching for them in a string of all the digits, in order. The line above it, `[Res, TF & UF] extends [false, never]`, tests whether the two digits differ by intersecting them and checking for `never`, since two distinct literal types have an empty intersection.

None of this is bad code. It is the *correct* answer, and the obvious answers are all wrong: the natural approach builds a tuple of length `N` and compares lengths, which the harness kills with `GreaterThan<1234567891011, 1234567891010>`, a case that exists precisely to force a real algorithm. Given a language with no comparison and no arithmetic, digit-wise decimal comparison via substring search in `'0123456789'` is genuine ingenuity. The builder is `a > b`.

## 4471 · Zip

```ts
// TypeScript, issue #4495 (+23)
type Zip<T extends any[], U extends any[]> =
[T, U] extends [[infer L, ...infer RestT], [infer R, ...infer RestU]]
? [[L, R], ...Zip<RestT, RestU>]
: []
```

```js
// Builder
function zip(A: type, B: type): type {
  const [a, b] = [tupleElements(A), tupleElements(B)];
  return tupleOf(a.slice(0, Math.min(a.length, b.length)).map((e, i) => tupleOf([e.type, b[i].type])));
}

zip(type [], type []) === type [];
zip(type [1, 2], type [true, false]) === type [[1, true], [2, false]];
zip(type [1, 2, 3], type ['1', '2']) === type [[1, '1'], [2, '2']];
```

`[T, U] extends [[...], [...]]` wraps both operands in a tuple so they can be destructured in one pattern, which is the type-level equivalent of destructuring two arrays in a single statement, and the shorter-list case falls out of the match failing. `Math.min` says it directly.

## 4484 · IsTuple

Is `T` a tuple, as opposed to an array or anything else?

```ts
// TypeScript, issue #10917 (+9). The top-voted answer (#4491, +20) fails on `never`; see the note.
type IsTuple<T>= [T] extends [never] ?
  false :
  T extends ReadonlyArray<unknown> ?
    number extends T['length'] ?
      false :
      true :
    false;
```

```js
// Builder
function isTuple(T: type): type {
  return reflect(T).kind === 'tuple' ? type true : type false;
}

isTuple(type []) === type true;
isTuple(type [uint32]) === type true;
isTuple(type { length: 1 }) === type false;
isTuple([].<uint32>) === type false;
isTuple(never) === type false;
```

`number extends T['length']` is the actual test, and it is an inference: a tuple's `length` is a literal, an array's `length` is `number`, so asking whether `number` fits inside `T['length']` distinguishes them. The `[T] extends [never]` guard in front is the anti-distribution wrapper again, needed because the top-voted answer without it returns `never` for `IsTuple<never>` and fails. `kind === 'tuple'` is the question the challenge is named after.

## 4499 · Chunk

Split a tuple into chunks of `N`.

```ts
// TypeScript, issue #4513 (+20)
type Chunk<T extends any[], N extends number, Swap extends any[] = []> =
Swap['length'] extends N
  ? [Swap, ...Chunk<T, N>]
  : T extends [infer K, ...infer L]
    ? Chunk<L, N, [...Swap, K]>
    : Swap extends [] ? Swap : [Swap]
```

```js
// Builder
function chunk(T: type, n: uint32): type {
  const elements = tupleElements(T);
  const out = [];
  for (let i = 0; i < elements.length; i += n)
    out.push(tupleOf(elements.slice(i, i + n).map(e => e.type)));
  return tupleOf(out);
}

chunk(type [], 1) === type [];
chunk(type [1, 2, 3], 2) === type [[1, 2], [3]];
chunk(type [1, 2, 3, 4], 2) === type [[1, 2], [3, 4]];
chunk(type [1, 2, 3, 4], 5) === type [[1, 2, 3, 4]];
```

`Swap` is an accumulator holding the chunk under construction, and `Swap['length'] extends N` is the only way to ask whether it is full. The trailing `Swap extends [] ? Swap : [Swap]` handles the final partial chunk, and it must distinguish "empty accumulator, emit nothing" from "partial accumulator, emit it", which `slice` does by producing a shorter last slice and nothing at all when the input is empty.

## 4518 · Fill

Replace elements between `Start` and `End` with `N`.

```ts
// TypeScript, issue #14102 (+32)
type Fill<
  T extends unknown[],
  N,
  Start extends number = 0,
  End extends number = T['length'],
  Count extends any[] = [],
  Flag extends boolean = Count['length'] extends Start ? true : false
> = Count['length'] extends End
  ? T
  : T extends [infer R, ...infer U]
    ? Flag extends false
      ? [R, ...Fill<U, N, Start, End, [...Count, 0]>]
      : [N, ...Fill<U, N, Start, End, [...Count, 0], Flag>]
    : T
```

```js
// Builder
function fill(T: type, N: type, start: uint32 = 0, end: uint32 = tupleElements(T).length): type {
  return tupleOf(tupleElements(T).map((e, i) => i >= start && i < end ? N : e.type));
}

fill(type [], type 0) === type [];
fill(type [1, 2, 3], type 0) === type [0, 0, 0];
fill(type [1, 2, 3], type true, 0, 1) === type [true, 2, 3];
fill(type [1, 2, 3], type true, 1, 3) === type [1, true, true];
fill(type [1, 2, 3], type true, 10, 20) === type [1, 2, 3];
```

Six type parameters, two of which are the loop's private state and one of which, `Flag`, is a latch: once the counter reaches `Start` it is passed forward explicitly so the "we are inside the range" fact survives the recursion, because there is nowhere else to keep it. The builder compares `i` to two numbers.

## 4803 · Trim Right

```ts
// TypeScript, issue #11866 (+11)
type TrimRight<S extends string>
  = S extends `${infer Left}${' ' | '\n' | '\t'}` ? TrimRight<Left> : S
```

```js
// Builder
function trimRight(s: string): type {
  return literal(s.trimEnd());
}

trimRight('     str     ') === type '     str';
trimRight('   \n\t foo bar ') === type '   \n\t foo bar';
```

## 5117 · Without

Remove elements matching a value or a list of values.

```ts
// TypeScript, issue #14118 (+27)
type ToUnion<T> = T extends any[] ? T[number] : T
type Without<T, U> =
  T extends [infer R, ...infer F]
    ? R extends ToUnion<U>
      ? Without<F, U>
      : [R, ...Without<F, U>]
    : T
```

```js
// Builder
function without(T: type, U: type): type {
  const drop = new Set(reflect(U).kind === 'tuple' ? tupleElements(U).map(e => e.type) : [U]);
  return tupleOf(tupleElements(T).map(e => e.type).filter(t => !drop.has(t)));
}

without(type [1, 2], type 1) === type [2];
without(type [1, 2, 4, 1, 5], type [1, 2]) === type [4, 5];
without(type [2, 3, 2, 3, 2, 3, 2, 3], type [2, 3]) === type [];
```

`new Set` of type objects works, and it is worth pausing on why: interning makes a type object's identity a pointer, so types drop into hash sets and maps exactly like strings and numbers do. Half the builders in this batch are one line for that reason. `ToUnion` exists because `R extends U` needs `U` to be a union and the caller may pass a tuple, which is the same normalization the `Set` constructor is doing.

## 5140 · Trunc

Truncate toward zero, as a string.

```ts
// TypeScript, issue #34091 (+2). The top-voted answer (#8258, +3) returns `''` for `'.3'`; see the note.
type Trunc<T extends number | string> = `${T}` extends `${infer R}.${any}` ? (R extends '' | '-' ? `${R}0` : `${R}`) : `${T}`
```

```js
// Builder
function trunc(v: number | string): type {
  const n = Math.trunc(Number(v));
  return literal(Object.is(n, -0) ? '-0' : String(n));   // String(-0) is '0'
}

trunc(12.345) === type '12';
trunc(-5.1) === type '-5';
trunc('.3') === type '0';
trunc('-.3') === type '-0';
```

An honest tax, and a nice one. `Trunc<'-.3'>` must be `'-0'`, and the obvious builder fails it: `Math.trunc(-0.3)` is `-0`, and `String(-0)` is `'0'`, because JavaScript's number-to-string conversion drops the sign of negative zero. `Object.is` is the fix. The TypeScript solution never has this problem because it never converts to a number: it does string surgery, so `'-.3'` splits into `'-'` and then gets a `'0'` glued on by the `R extends '' | '-'` branch, which exists for exactly the same edge case seen from the other side. Both languages pay for negative zero; the builder pays in a JavaScript quirk it inherited, TypeScript pays in a branch. Worth remembering the next time a builder looks like a free win: JavaScript's semantics come along with JavaScript's expressiveness.

## 5153 · IndexOf

The first index of `U` in `T`, or `-1`.

```ts
// TypeScript, issue #13143 (+4)
type IndexOf<T extends any[], U, Pass extends any[] = []> =
  T extends [infer F, ...infer Rest]
    ? Equal<F, U> extends true
      ? Pass['length']
      : IndexOf<Rest, U, [...Pass, F]>
    : -1
```

```js
// Builder
function indexOf(T: type, U: type): type {
  return literal(tupleElements(T).findIndex(e => e.type === U));
}

indexOf(type [1, 2, 3], type 2) === type 1;
indexOf(type [0, 0, 0], type 2) === type -1;
indexOf(type [string, 1, uint32, 'a'], uint32) === type 2;   // identity, not assignability: 1 does not match uint32
```

Note the import. The published answer needs `Equal` from `@type-challenges/utils`, which is the checker-internals incantation the harness itself is built from, because `IndexOf<[string, 1, number, 'a'], number>` must be `2` and an `extends` test would match `1` at index 1. `findIndex` with `===` gives both the search and the `-1`, since that is what `findIndex` returns when nothing matches.

## 5310 · Join

```ts
// TypeScript, issue #33930 (+2). The top-voted answer (#7225, +14) has no default separator
// and returns `never` for the empty tuple; see the note.
type Stringifiable = string | number | bigint | boolean | null | undefined;

type Join<
  T extends readonly Stringifiable[],
  U extends Stringifiable = ",",
> = T extends []
  ? ""
  : T extends [infer First extends Stringifiable]
  ? `${First}`
  : T extends [
    infer First extends Stringifiable,
    infer Second extends Stringifiable,
    ...infer Rest extends Stringifiable[],
  ]
  ? Join<[`${First}${U}${Second}`, ...Rest], U>
  : string;
```

```js
// Builder
function join(T: type, separator: string | number = ','): type {
  return literal(tupleElements(T).map(e => String(literalValues(e.type)[0])).join(String(separator)));
}

join(type ['a', 'p', 'p', 'l', 'e'], '-') === type 'a-p-p-l-e';
join(type ['2', '2', '2'], 1) === type '21212';
join(type ['o'], 'u') === type 'o';
join(type [], 'u') === type '';
join(type ['1', '1', '1']) === type '1,1,1';
```

`Array.prototype.join` is the challenge. The empty case, the single-element case, and the default separator are all things `join` already does, and they are exactly the three cases the top-voted answer gets wrong.

## 5317 · LastIndexOf

```ts
// TypeScript, issue #29043 (+2). The top-voted answer (#5330, +13) uses `extends` and
// matches the wrong element; see the note.
type MyEqual<X, Y> = (<T>() => T extends X ? 1 : 2) extends <T>() => T extends Y
  ? 1
  : 2
  ? true
  : false

type LastIndexOf<T, U extends string | number> = T extends [
  ...infer Head,
  infer Tail
]
  ? MyEqual<Tail, U> extends true
    ? Head["length"]
    : LastIndexOf<Head, U>
  : -1
```

```js
// Builder
function lastIndexOf(T: type, U: type): type {
  return literal(tupleElements(T).findLastIndex(e => e.type === U));
}

lastIndexOf(type [1, 2, 3, 2, 1], type 2) === type 3;
lastIndexOf(type [0, 0, 0], type 2) === type -1;
lastIndexOf(type [string, 2, uint32, 'a', uint32, 1], uint32) === type 4;
```

The third solution in this batch to carry the `IsEqual` incantation, and the top-voted answer is the one that tried `L extends U` instead: on `[string, 2, number, 'a', number, 1]` searching for `number`, `1 extends number` is true, so it answers `5`. Assignability was the wrong question and the language offers no other.

## 5360 · Unique

Deduplicate a tuple, keeping first occurrences.

```ts
// TypeScript, issue #21766 (+12)
type Unique<T, U = never> =
  T extends [infer F, ...infer R]
    ? true extends (U extends U ? Equal<U, [F]> : never)
      ? Unique<R, U>
      : [F, ...Unique<R, U | [F]>]
    : []
```

```js
// Builder
function unique(T: type): type {
  return tupleOf([...new Set(tupleElements(T).map(e => e.type))]);
}

unique(type [1, 1, 2, 2, 3, 3]) === type [1, 2, 3];
unique(type [1, 'a', 2, 'b', 2, 'a']) === type [1, 'a', 2, 'b'];
unique(type [string, uint32, 1, 'a', 1, string, 2, 'b', 2, uint32]) === type [string, uint32, 1, 'a', 2, 'b'];
```

The high point of the batch. TypeScript has no set, so `U` is a union used as one; it has no membership test, so `U extends U ? Equal<U, [F]> : never` distributes over the accumulator and asks each member; it has no equality, so `Equal` is the incantation again; and each element is wrapped as `[F]` before being added, because a union of raw types would collapse `1 | 1` and lose the distinction the algorithm depends on. Four workarounds, one per missing primitive.

`new Set` is a set. `===` is equality. Insertion order is preserved because that is what `Set` guarantees, which is the "keep first occurrences" requirement, unrequested.

## 5821 · MapTypes

Replace property types according to a union of `{ mapFrom, mapTo }` rules.

```ts
// TypeScript, issue #14152 (+22)
type MapTypes<T, R extends { mapFrom: any; mapTo: any }> = {
  [K in keyof T]: T[K] extends R['mapFrom']
  ? R extends { mapFrom: T[K] }
    ? R['mapTo']
    : never
  : T[K]
}
```

```js
// Builder
function mapTypes(T: type, R: type): type {
  const field = (r: type, name: string): type => reflect(r).properties.find(p => p.name === name).type;
  const rules = arms(R).map(r => ({ from: field(r, 'mapFrom'), to: field(r, 'mapTo') }));
  return mapProperties(T, p => {
    const hits = rules.filter(r => r.from === p.type);
    return hits.length === 0 ? p : { ...p, type: union(hits.map(r => r.to)) };
  });
}

mapTypes(type { stringToNumber: string, skipParsingMe: boolean }, type { mapFrom: string, mapTo: uint32 })
  === type { stringToNumber: uint32, skipParsingMe: boolean };
mapTypes(type { date: string }, type { mapFrom: string, mapTo: Date } | { mapFrom: string, mapTo: null })
  === type { date: Date | null };
mapTypes(type { name: string }, type { mapFrom: boolean, mapTo: never }) === type { name: string };
```

The two-rules-match case is why the TypeScript version reads the way it does: `R extends { mapFrom: T[K] }` distributes over the rule union and keeps the matching arms, and the surrounding `T[K] extends R['mapFrom']` is a pre-filter deciding whether *any* rule applies at all. Two passes over the same union, one to ask "is there a hit" and one to collect the hits, because there is no way to hold the intermediate. `hits` is a `const`.

## 7544 · Construct Tuple

A tuple of the given length.

```ts
// TypeScript, issue #14153 (+18)
type ConstructTuple<L extends number, U extends unknown[] = []> =
  U['length'] extends L
    ? U
    : ConstructTuple<L, [...U, unknown]>
```

```js
// Builder
function constructTuple(n: uint32): type {
  return tupleOf(Array.from({ length: n }, () => any));
}

constructTuple(0) === type [];
constructTuple(2) === type [any, any];
reflect(constructTuple(999)).elements.length === 999;
reflect(constructTuple(1000)).elements.length === 1000;   // no limit here
```

The harness contains this line, and it is the most candid thing in the repository:

```ts
Expect<Equal<ConstructTuple<999>['length'], 999>>,
// @ts-expect-error
Expect<Equal<ConstructTuple<1000>['length'], 1000>>,
```

The test asserts that the challenge *stops working* at one thousand. That is not a bug being documented; it is TypeScript's recursion limit promoted to a specified behavior of the exercise, because there is no way to construct a tuple except by recursing once per element. `Array.from({ length: n })` has no such number, and the challenge's own boundary case is not expressible as a failure.

## 8640 · Number Range

The union of integers from `L` to `H` inclusive.

```ts
// TypeScript, issue #9988 (+18)
type Utils<L, C extends any[] = [], R = L> =
  C['length'] extends L
      ? R
      : Utils<L, [...C, 0], C['length'] | R>

type NumberRange<L, H> = L | Exclude<Utils<H>, Utils<L>>
```

```js
// Builder
function numberRange(low: uint32, high: uint32): type {
  return union(Array.from({ length: high - low + 1 }, (_, i) => literal(low + i)));
}

numberRange(2, 9) === type 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
numberRange(0, 2) === type 0 | 1 | 2;
```

`Utils<H>` builds every number from `0` to `H` by counting a tuple up and unioning the lengths along the way, `Utils<L>` does the same to `L`, and `Exclude` subtracts one from the other. Two counts and a set difference, to express a range. The harness's largest case runs to 90, which is not a coincidence.

## 8767 · Combination

Every ordering of every non-empty subset of the words, space-joined.

```ts
// TypeScript, issue #11027 (+33)
type Combination<T extends string[], All = T[number], Item = All>
  = Item extends string
    ? Item | `${Item} ${Combination<[], Exclude<All, Item>>}`
    : never
```

```js
// Builder
function combination(words: [].<string>): type {
  const go = (rest: [].<string>): [].<string> =>
    rest.flatMap((w, i) => [w, ...go(rest.toSpliced(i, 1)).map(tail => `${w} ${tail}`)]);
  return union(go(words).map(literal));
}

combination(['foo', 'bar', 'baz'])
  === type 'foo' | 'bar' | 'baz'
    | 'foo bar' | 'foo bar baz' | 'foo baz' | 'foo baz bar'
    | 'bar foo' | 'bar foo baz' | 'bar baz' | 'bar baz foo'
    | 'baz foo' | 'baz foo bar' | 'baz bar' | 'baz bar foo';
```

`Item = All` is the copy-the-parameter trick from `IsUnion`, here so that `Item` can distribute while `All` stays whole for the `Exclude` to subtract from. The first parameter is discarded on the recursive call (`Combination<[], ...>`) because after the first step the tuple has served its purpose and only the union matters, so a required parameter gets passed a dummy. `go` takes the list it needs.

## 8987 · Subsequence

All subsequences, order preserved.

```ts
// TypeScript, issue #8995 (+11)
type Subsequence<T extends any[],Prefix extends any[] = []> = T extends [infer F,...infer R]? Subsequence<R,Prefix | [...Prefix,F]>:Prefix
```

```js
// Builder
function subsequence(T: type): type {
  const go = (elements: [].<type>): [].<[].<type>> =>
    elements.length === 0 ? [[]] : go(elements.slice(1)).flatMap(rest => [rest, [elements[0], ...rest]]);
  return union(go(tupleElements(T).map(e => e.type)).map(tupleOf));
}

subsequence(type [1, 2]) === type [] | [1] | [2] | [1, 2];
subsequence(type [1, 2, 3]) === type [] | [1] | [2] | [1, 2] | [3] | [1, 3] | [2, 3] | [1, 2, 3];
```

A tidy answer on both sides, and one of the closest matches in the document: `Prefix | [...Prefix, F]` is the power-set step, and the union accumulator is doing exactly what a list of partial results would do. Worth noting because it is the shape the type language is actually good at, and it is no accident that the challenge is about unions.

## 9142 · CheckRepeatedChars

Does the string contain a repeated character?

```ts
// TypeScript, issue #28150 (+15)
type CheckRepeatedChars<T extends string> = T extends `${infer F}${infer E}`
  ? E extends `${string}${F}${string}`
    ? true
    : CheckRepeatedChars<E>
  : false
```

```js
// Builder
function checkRepeatedChars(s: string): type {
  const chars = [...s];
  return new Set(chars).size !== chars.length ? type true : type false;
}

checkRepeatedChars('abc') === type false;
checkRepeatedChars('abb') === type true;
checkRepeatedChars('cbc') === type true;
checkRepeatedChars('') === type false;
```

## 9286 · FirstUniqueCharIndex

The index of the first character that appears exactly once, or `-1`.

```ts
// TypeScript, issue #20488 (+9)
type FirstUniqueCharIndex<
  T extends string,
  _Acc extends string[] = []
> = T extends ''
  ? -1
  : T extends `${infer Head}${infer Rest}`
  ? Head extends _Acc[number]
    ? FirstUniqueCharIndex<Rest, [..._Acc, Head]>
    : Rest extends `${string}${Head}${string}`
    ? FirstUniqueCharIndex<Rest, [..._Acc, Head]>
    : _Acc['length']
  : never
```

```js
// Builder
function firstUniqueCharIndex(s: string): type {
  const chars = [...s];
  return literal(chars.findIndex(c => chars.indexOf(c) === chars.lastIndexOf(c)));
}

firstUniqueCharIndex('leetcode') === type 0;
firstUniqueCharIndex('loveleetcode') === type 2;
firstUniqueCharIndex('aabb') === type -1;
firstUniqueCharIndex('') === type -1;
```

`_Acc` is carrying two jobs at once: it is the set of characters already seen (queried by `Head extends _Acc[number]`) and it is the loop counter (read by `_Acc['length']` when the answer is found). One accumulator, two purposes, because there is nowhere to put a second. `indexOf(c) === lastIndexOf(c)` is the uniqueness test, `findIndex` is the loop, and `-1` is what `findIndex` returns.

## 9616 · Parse URL Params

Extract `:param` segments from a path.

```ts
// TypeScript, issue #30353 (+10)
type ParseUrlParams<T> = T extends `${string}:${infer R}`
  ? R extends `${infer P}/${infer L}`
    ? P | ParseUrlParams<L>
    : R
  : never
```

```js
// Builder
function parseUrlParams(path: string): type {
  return union(path.split('/').filter(seg => seg.startsWith(':')).map(seg => literal(seg.slice(1))));
}

parseUrlParams('') === never;
parseUrlParams('posts/:id') === type 'id';
parseUrlParams('posts/:id/:user/like') === type 'id' | 'user';
```

`split`, `filter`, `map`. The empty case is `never` because the union of no arms is `never`, which is what the challenge asks for and what the TypeScript version needs an explicit branch to say.

## 9896 · GetMiddleElement

The middle one or two elements.

```ts
// TypeScript, issue #18255 (+27)
type GetMiddleElement<T extends any[]> =
  T['length'] extends 0 | 1 | 2?
    T:
    T extends [any,...infer M,any]?
      GetMiddleElement<M>:never
```

```js
// Builder
function getMiddleElement(T: type): type {
  const elements = tupleElements(T);
  const start = Math.max(0, Math.ceil(elements.length / 2) - 1);
  return tupleOf(elements.slice(start, elements.length - start).map(e => e.type));
}

getMiddleElement(type []) === type [];
getMiddleElement(type [1, 2, 3, 4, 5]) === type [3];
getMiddleElement(type [1, 2, 3, 4, 5, 6]) === type [3, 4];
getMiddleElement(type [never]) === type [never];
```

Both peel from both ends; the TypeScript version does it one pair at a time by recursion, the builder does it in one `slice` by computing where the peeling would have stopped. The `T['length'] extends 0 | 1 | 2` base case has to enumerate three lengths as a union because there is no `<=`.

## 9898 · Appear only once

The elements that appear exactly once.

```ts
// TypeScript, issue #34727 (+1). The top-voted answer (#24748, +5) uses `extends` and
// mistakes `1` for a duplicate of `number`; see the note.
type IncludesItem<T, U> = U extends U ? [U] extends [T] ? true : false : false
type FindEles<T extends any[], U extends any[] = []> = T extends [infer A, ...infer R] ?
  IncludesItem<A, U[number] | R[number]> extends false ? [A, ...FindEles<R, U>] : FindEles<R, [...U, A]>
  : T
```

```js
// Builder
function findEles(T: type): type {
  const types = tupleElements(T).map(e => e.type);
  return tupleOf(types.filter(t => types.indexOf(t) === types.lastIndexOf(t)));
}

findEles(type [1, 2, 2, 3, 3, 4, 5, 6, 6, 6]) === type [1, 4, 5];
findEles(type [2, 2, 3, 3, 6, 6, 6]) === type [];
findEles(type [1, 2, uint32]) === type [1, 2, uint32];        // 1 is not a duplicate of uint32
findEles(type [1, 2, uint32, uint32]) === type [1, 2];
```

The fifth challenge that needs identity and the fourth whose top-voted answer used assignability instead. `F extends R[number]` looks like a membership test and is a subtype test, so on `[1, 2, number]` it decides that `1` is already present because `1 extends number`, and drops both literals. The passing answer builds its own identity check out of `[U] extends [T]` with the operands wrapped to suppress distribution. `indexOf(t) === lastIndexOf(t)` is the membership test, and it is the same one anyone would write for an array of strings.

## 9989 · Count Element Number To Object

Flatten, then count occurrences into an object.

```ts
// TypeScript, issue #28355 (+8)
type Flatten<T,R extends any[] = []> =
  T extends [infer F,...infer L]?
    [F] extends [never]?
      Flatten<L,R>:
      F extends any[]?
        Flatten<L,[...R,...Flatten<F>]  >
        :Flatten<L,[...R,F]>
    :R

type Count<
  T,
  R extends Record<string | number,any[]> = {}
> =
T extends [infer F extends string | number,...infer L]?
  F extends keyof R?
    Count<L, Omit<R,F>& Record<F,[...R[F],0] > >
    : Count<L, R & Record<F,[0]>>
  :{
    [K in keyof R]:R[K]['length']
  }

type CountElementNumberToObject<T> = Count<Flatten<T>>
```

```js
// Builder
function countElementNumberToObject(T: type): type {
  const counts = new Map();
  const walk = (elements: [].<Reflect.TypeTupleElement>): void => {
    for (const e of elements) {
      const node = reflect(e.type);
      if (node.kind === 'tuple') walk(node.elements);
      else counts.set(node.value, (counts.get(node.value) ?? 0) + 1);
    }
  };
  walk(tupleElements(T));
  return objectOf([...counts].map(([value, n]) => prop(String(value), literal(n))));
}

countElementNumberToObject(type [1, 2, 3, 4, 5]) === type { 1: 1, 2: 1, 3: 1, 4: 1, 5: 1 };
countElementNumberToObject(type [1, 2, 3, 4, 5, [1, 2, 3]]) === type { 1: 2, 2: 2, 3: 2, 4: 1, 5: 1 };
```

The counting accumulator is `Record<string | number, any[]>`: every count is a *tuple*, incrementing is `[...R[F], 0]`, and the final mapped type converts each tuple back to a number with `R[K]['length']`. Updating one key means `Omit<R, F> & Record<F, ...>`, since there is no way to replace a property. A `Map` and `+ 1`.

## 10969 · Integer

Accept only integer literals.

```ts
// TypeScript, issue #18773 (+23)
type Integer<T extends number> = `${T}` extends `${bigint}` ? T : never
```

```js
// Builder
function integer(T: type): type {
  const node = reflect(T);
  return node.kind === 'literal' && Number.isInteger(node.value) ? T : never;
}

integer(type 1) === type 1;
integer(type 1.1) === never;
integer(type 28.00) === type 28;      // 28.00 and 28 are one literal
integer(float64) === never;           // not a literal at all
```

`` `${T}` extends `${bigint}` `` is a lovely piece of misdirection: it stringifies the number and asks whether the result looks like a bigint literal, which is true exactly when there is no decimal point. `bigint` is being used as a *pattern for a string of digits*, and the type has nothing to do with bigints. `Number.isInteger` is the predicate the challenge is named after. See the note on what the challenge's last two cases are really testing.

## 16259 · ToPrimitive

Widen literals to their primitives, recursively.

```ts
// TypeScript, issue #22057 (+24)
type ToPrimitive<T> = T extends object ? (
  T extends (...args: never[]) => unknown ? Function : {
    [Key in keyof T]: ToPrimitive<T[Key]>
  }
) : (
  T extends { valueOf: () => infer P } ? P : T
)
```

```js
// Builder
function toPrimitive(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'literal':  return node.base;
    case 'object':   return objectOf(node.properties.map(p => ({ ...p, type: toPrimitive(p.type) })), node.indexSignatures);
    case 'tuple':    return Reflect.makeType({ ...node, elements: node.elements.map(e => ({ ...e, type: toPrimitive(e.type) })) });
    case 'function': return Function;
    default:         return T;
  }
}

type PersonInfo = {
  name: 'Tom', age: 30, married: false,
  addr: { home: '123456', phone: '13111111111' },
  hobbies: ['sing', 'dance'], fn: () => any,
};
toPrimitive(PersonInfo) === type {
  name: string, age: float64, married: boolean,
  addr: { home: string, phone: string },
  hobbies: [string, string], fn: Function,
};
```

The last line of the TypeScript solution is the one to look at: `T extends { valueOf: () => infer P } ? P : T`. To get from the literal `'Tom'` to `string`, it asks what `valueOf` returns *on the boxed wrapper*, because `String.prototype.valueOf(): string` is the only place in the type system where the relationship between a string literal and `string` is written down in a readable form. It is a genuinely inspired route to the answer, and it is a route because the direct road does not exist.

The reflection node for a literal carries `base`. That is the same fact, stated once, where the literal is.

(The harness's `readonlyArr: readonly ['test']` field is omitted above: this proposal has no readonly array type, which the note from challenge 9 already records.)

## 17973 · DeepMutable

Strip `readonly` recursively.

```ts
// TypeScript, issue #18259 (+5)
type DeepMutable<T extends Record<keyof any,any>> =
  T extends (...args:any[])=>any?
    T:
    {
      - readonly [K in keyof T]:DeepMutable<T[K]>
    }
```

```js
// Builder
function deepMutable(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'object': return objectOf(node.properties.map(p => ({ ...p, readonly: false, type: deepMutable(p.type) })),
                                   node.indexSignatures);
    case 'array':  return arrayOf(deepMutable(node.element), node.extent);
    case 'tuple':  return Reflect.makeType({ ...node, elements: node.elements.map(e => ({ ...e, type: deepMutable(e.type) })) });
    case 'union':  return union(node.arms.map(deepMutable));
    default:       return T;
  }
}

type X = { readonly a: () => 22, readonly b: string, readonly c: { readonly d: boolean } };
deepMutable(X) === type { a: () => 22, b: string, c: { d: boolean } };
```

The mirror of challenge 9, and worth putting beside it. `DeepReadonly`'s top-voted answer stops at functions with `keyof T extends never`, which works by accident; this author writes `T extends (...args: any[]) => any ? T` and says what is meant. The builder's `switch` is the same statement in both directions, and `readonly: false` against `readonly: true` is the whole diff.

## 18142 · All

Are all elements assignable to `N`?

```ts
// TypeScript, issue #33471 (+3)
type All<A extends unknown[], Elt> =
  Equal<Equal<A[number], A[0]> & Equal<A[0], Elt>, true>
```

```js
// Builder
function all(T: type, N: type): type {
  return tupleElements(T).every(e => e.type === N) ? type true : type false;
}

all(type [1, 1, 1], type 1) === type true;
all(type [1, 1, 2], type 1) === type false;
all(type ['1', '1', '1'], type 1) === type false;
all(type [never], never) === type true;
all(type [any], unknown) === type false;   // assignable both ways, identical neither
all(type [1, 1, 2], type 1 | 2) === type false;
```

A neat one on both sides, and the harness makes the theme explicit: `All<[any], unknown>` and `All<[unknown], any>` must both be `false`, so assignability in either direction is the wrong relation and the answer reaches for `Equal`. `Equal<A[number], A[0]>` asks whether the union of all elements is *identical* to the first, which is true exactly when the tuple is uniform, and `Equal<A[0], Elt>` pins that representative to the target; intersecting the two verdicts and testing against `true` folds the pair of questions into one. The builder is `.every(e => e.type === N)`. Interned types make identity a language operator, so the challenge's hard case costs nothing: `any` and `unknown` are simply different objects.

## 18220 · Filter

Keep the elements assignable to `P`.

```ts
// TypeScript, issue #20070 (+10)
type Filter<T extends unknown[], P> = T extends [infer F, ...infer R]
  ? F extends P
    ? [F, ...Filter<R, P>]
    : Filter<R, P>
  : [];
```

```js
// Builder
function filter(T: type, P: type): type {
  return tupleOf(tupleElements(T).map(e => e.type).filter(t => Reflect.isAssignable(t, P)));
}

type Falsy = false | 0 | '' | null | undefined;
filter(type [0, 1, 2], type 2) === type [2];
filter(type [0, 1, 2], type 0 | 1) === type [0, 1];
filter(type [0, 1, 2], Falsy) === type [0];
```

## 19749 · IsEqual

Return whether two types are equal.

```ts
// No answer has been posted for this challenge: a search for `label:19749 label:answer`
// returns zero issues. The solution below is the repo's own `Equal`, from utils/index.d.ts.
type IsEqual<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false
```

```js
// Builder
function isEqual(A: type, B: type): type {
  return A === B ? type true : type false;
}

isEqual(float64, string) === type false;
isEqual(type 1, type 1) === type true;
isEqual(any, type 1) === type false;
isEqual(type 1 | 2, type 1) === type false;
isEqual(type [any], type [uint32]) === type false;
```

The challenge is to implement `===`, and it is filed under medium.

Three things about it are worth stating plainly. The first is that the answer already ships in the repository, in `utils/index.d.ts`, because the test harness needs it to grade every other challenge; the exercise is to reconstruct the tool being used to check your work. The second is that nobody has posted a solution. Ninety challenges into this document, every other one has answers running from single digits into the hundreds, and this one has zero, which is not surprising for a puzzle whose answer is quotable from the framework's own source and originates in a 2018 issue comment about checker internals rather than in anything a person could derive.

The third is the mechanism. `(<T>() => T extends X ? 1 : 2)` is a generic function type whose body nobody will ever call. Two of them are compared for assignability, and the checker decides that comparison by structurally comparing the deferred conditionals inside, which it can only do by comparing `X` and `Y` for identity. The identity relation exists; it is reachable only as an observable side effect of how function assignability is implemented, and it is not specified to work this way. That is what four other challenges in this document import, what one reinvents with `[U] extends [T]`, what two got wrong by reaching for `extends` instead, and what every `Expect<Equal<...>>` in the harness depends on.

Interned type objects have identity because that is what interning means. `A === B` is the whole solution, and it is not a solution to a puzzle. It is the operator.

## 21104 · FindAll

Every index where `P` occurs in `T`, including overlaps.

```ts
// TypeScript, issue #29712 (+11)
type NormalFindAll<
  T extends string,
  S extends string,
  P extends any[] = [],
  R extends number[] = [],
> =
T extends `${string}${infer L}`?
  T extends `${S}${string}`?
    NormalFindAll<L,S,[...P,0],[...R,P['length']]>
    :NormalFindAll<L,S,[...P,0],R>
  :R

type FindAll<
  T extends string,
  P extends string,
> =
P extends ''?
  []:NormalFindAll<T,P>
```

```js
// Builder
function findAll(s: string, pattern: string): type {
  if (pattern === '') return type [];
  const hits = [];
  for (let i = s.indexOf(pattern); i !== -1; i = s.indexOf(pattern, i + 1)) hits.push(literal(i));
  return tupleOf(hits);
}

findAll('Collection of TypeScript type challenges', 'Type') === type [14];
findAll('Collection of TypeScript type challenges', 'pe') === type [16, 27];
findAll('AAAA', 'AA') === type [0, 1, 2];   // overlapping
findAll('', 'Type') === type [];
```

`P extends any[] = []` is the position counter and `R extends number[] = []` is the results, so the accumulator pair is a tuple whose length is the index and a tuple of the indices found. `indexOf` takes a start position, and stepping it by one rather than by the pattern's length is what makes the overlapping case work in both solutions.

## 21106 · Combination key type

Ordered pairs of modifier keys.

```ts
// TypeScript, issue #24564 (+11)
type Combs<T extends string[] = ModifierKeys> = T extends [infer F extends string, ...infer R extends string[]] ? `${F} ${R[number]}` | Combs<R> : never;
```

```js
// Builder
function combs(keys: [].<string>): type {
  return union(keys.flatMap((a, i) => keys.slice(i + 1).map(b => literal(`${a} ${b}`))));
}

combs(['cmd', 'ctrl', 'opt', 'fn'])
  === type 'cmd ctrl' | 'cmd opt' | 'cmd fn' | 'ctrl opt' | 'ctrl fn' | 'opt fn';
```

`` `${F} ${R[number]}` `` is the good version of template-literal distribution: one head against the union of every tail, producing the whole row of pairs in one expression. `slice(i + 1)` is the same idea, and this is close to a draw.

## 21220 · Permutations of Tuple

```ts
// TypeScript, issue #29713 (+10)
type Insert<
  T extends unknown[],
  U
> =
T extends [infer F,...infer L]
  ? [F,U,...L] | [F,...Insert<L,U> ]
  : [U]

type PermutationsOfTuple<
  T extends unknown[],
  R extends unknown[] = []
> =
T extends [infer F,...infer L]?
  PermutationsOfTuple<L,Insert<R,F> | [F,...R] >
  :R
```

```js
// Builder
function permutationsOfTuple(T: type): type {
  return union(perms(tupleElements(T).map(e => e.type)).map(tupleOf));   // the same perms as challenge 296
}

permutationsOfTuple(type [1, 2, 3])
  === type [1,2,3] | [1,3,2] | [2,1,3] | [2,3,1] | [3,1,2] | [3,2,1];
permutationsOfTuple(type [1]) === type [1];
permutationsOfTuple(type []) === type [];
```

Challenge 296 permuted a union and this permutes a tuple, and the builder's `perms` is the same function both times: the only difference is whether the items come from `arms(T)` or `tupleElements(T)`. TypeScript cannot share that way. `Permutation` distributes over a union with `K extends K`; `PermutationsOfTuple` peels a tuple and threads an accumulator through `Insert`. Same algorithm, two encodings, because the *shape* of the input decides which of the language's two iteration mechanisms is available.

## 25170 · Replace First

Replace the first element matching `S`.

```ts
// TypeScript, issue #25366 (+5)
type ReplaceFirst<T extends readonly unknown[], S, R> = T extends readonly [infer F, ...infer Rest] ? F extends S ? [R, ...Rest] : [F, ...ReplaceFirst<Rest, S, R>] : [];
```

```js
// Builder
function replaceFirst(T: type, S: type, R: type): type {
  const types = tupleElements(T).map(e => e.type);
  const i = types.findIndex(t => Reflect.isAssignable(t, S));
  return tupleOf(i === -1 ? types : types.toSpliced(i, 1, R));
}

replaceFirst(type [1, 2, 3], type 3, type 4) === type [1, 2, 4];
replaceFirst(type [1, 'two', 3], string, type 2) === type [1, 2, 3];   // 'two' is assignable to string
replaceFirst(type ['six', 'eight', 'ten'], type 'eleven', type 'twelve') === type ['six', 'eight', 'ten'];
```

Put this next to challenge 9898. There, `F extends R[number]` was a membership test and needed identity, and the top-voted answer was wrong because `extends` gave assignability. Here `F extends S` is a *match* test and assignability is exactly right: `ReplaceFirst<[1, 'two', 3], string, 2>` must replace `'two'` because `'two'` is a string. Same operator, opposite requirements, and TypeScript has no way to say which one it means. The builder writes `Reflect.isAssignable` here and `===` there, and the reader can see which is intended without running the tests.

## 25270 · Transpose

```ts
// TypeScript, issue #25297 (+31)
type Transpose<M extends number[][],R = M['length'] extends 0?[]:M[0]> = {
  [X in keyof R]:{
    [Y in keyof M]:X extends keyof M[Y]?M[Y][X]:never
  }
}
```

```js
// Builder
function transpose(M: type): type {
  const rows = tupleElements(M).map(r => tupleElements(r.type).map(e => e.type));
  return tupleOf((rows[0] ?? []).map((_, x) => tupleOf(rows.map(row => row[x]))));
}

transpose(type []) === type [];
transpose(type [[1, 2], [3, 4]]) === type [[1, 3], [2, 4]];
transpose(type [[1, 2, 3], [4, 5, 6]]) === type [[1, 4], [2, 5], [3, 6]];
```

The nested mapped types are indexing by *key*, and the keys of a tuple are `'0' | '1' | ...`, so `[X in keyof R]` iterates positions by iterating stringified indices and `M[Y][X]` reads a cell with two of them. It relies on the array special case from challenge 20 to come back as a tuple rather than an object. The second parameter `R` defaulted to `M['length'] extends 0 ? [] : M[0]` is the first row, hoisted into a parameter because there is no `const`, with the empty case folded into the default. `rows[0] ?? []` is that line.

## 26401 · JSON Schema to TypeScript

Convert a JSON Schema type into the type it describes.

```ts
// TypeScript, issue #26864 (+19)
type Primitives = {
  string: string;
  number: number;
  boolean: boolean;
};

type HandlePrimitives<T, Type extends keyof Primitives> = T extends {
  enum: unknown[];
}
  ? T['enum'][number]
  : Primitives[Type];

type HandleObject<T> = T extends {
  properties: infer Properties extends Record<string, unknown>;
}
  ? T extends { required: infer Required extends unknown[] }
    ? Omit<
        {
          [K in Required[number] & keyof Properties]: JSONSchema2TS<
            Properties[K]
          >;
        } & {
          [K in Exclude<keyof Properties, Required[number]>]?: JSONSchema2TS<
            Properties[K]
          >;
        },
        never
      >
    : {
        [K in keyof Properties]?: JSONSchema2TS<Properties[K]>;
      }
  : Record<string, unknown>;

type HandleArray<T> = T extends { items: infer Items }
  ? JSONSchema2TS<Items>[]
  : unknown[];

type JSONSchema2TS<T> = T extends { type: infer Type }
  ? Type extends keyof Primitives
    ? HandlePrimitives<T, Type>
    : Type extends 'object'
    ? HandleObject<T>
    : HandleArray<T>
  : never;
```

```js
// Builder
const primitives = { string, number: float64, boolean };

function jsonSchema2TS(schema: type): type {
  const field = (name: string): type | undefined =>
    reflect(schema).properties.find(p => p.name === name)?.type;
  const kind = field('type');
  if (kind === undefined) return never;

  switch (literalValues(kind)[0]) {
    case 'object': {
      const properties = field('properties');
      if (properties === undefined) return type Record.<string, any>;
      const requiredList = field('required');
      const required = requiredList === undefined
        ? new Set()
        : new Set(tupleElements(requiredList).map(e => literalValues(e.type)[0]));
      return objectOf(reflect(properties).properties.map(p =>
        ({ ...prop(p.name, jsonSchema2TS(p.type)), optional: !required.has(p.name) })));
    }
    case 'array': {
      const items = field('items');
      return arrayOf(items === undefined ? any : jsonSchema2TS(items));
    }
    default: {
      const values = field('enum');
      return values === undefined
        ? primitives[literalValues(kind)[0]]
        : union(tupleElements(values).map(e => e.type));
    }
  }
}

jsonSchema2TS(type { type: 'string' }) === string;
jsonSchema2TS(type { type: 'number', enum: [1, 2, 3] }) === type 1 | 2 | 3;
jsonSchema2TS(type { type: 'object' }) === type Record.<string, any>;
jsonSchema2TS(type { type: 'array', items: { type: 'string' } }) === [].<string>;
jsonSchema2TS(type {
  type: 'object',
  properties: { req1: { type: 'string' }, add1: { type: 'string' } },
  required: ['req1'],
}) === type { req1: string, add1?: string };
```

The largest challenge in the document, and the fairest fight in it: this is a real task, a schema compiler, and both solutions are recognizably the same program. Read them side by side and the differences are all in the joints. `Primitives` is a lookup object in both, indexed by the schema's `type` string, and TypeScript's version is a *type* used as a map only because a mapped type is the only dictionary available. The `Omit<..., never>` wrapper around the intersection is the flattening no-op from challenge 2757, third sighting. `T extends { required: infer R }` is presence-testing a property by pattern-matching a type against a shape, where the builder writes `field('required') === undefined`, and having to name `infer Required` to find out whether the field exists is the same move as `NT extends TreeNode = NonNullable<T>` back at challenge 3376: a binding introduced through the only construct that can introduce one.

What the builder gains here is not brevity, since the two are within a few lines of each other. It is that `required` is a `Set`, `field` is a function, the recursion is a call, and someone who has never read this document can debug it.

## 27133 · Square

`n` squared.

```ts
// TypeScript, issue #30300 (+5). The top-voted answer (#29275, +5) overflows on `Square<101>`;
// see the note. This one does long multiplication instead.
type Digit = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
type Tuple<Length extends Digit, Result extends unknown[] = []> = Result["length"] extends Length
    ? Result
    : Tuple<Length, [...Result, unknown]>;
type Carry<N> = N extends number
    ? `${N}` extends `${infer X extends Digit}${infer Y extends Digit}`
        ? [X, Y]
        : `${N}` extends `${infer X extends Digit}`
            ? [0, X]
            : never
    : never;
type AddDigits<N extends Digit, M extends Digit, C extends Digit = 0> = Carry<[...Tuple<N>, ...Tuple<M>, ...Tuple<C>]["length"]>;
type SplitNumber<N extends number | string> = `${N}` extends `${infer X extends Digit}${infer Y}`
    ? [X, ...SplitNumber<Y>]
    : `${N}` extends `${infer X extends Digit}`
        ? [X]
        : [];
type ParseInt<N extends string> = N extends `${infer X extends number}` ? X : never;
type JoinNumberHelper<N> = N extends [infer X extends Digit, ...infer XS extends Digit[]]
    ? `${X}${JoinNumberHelper<XS>}`
    : "";
type JoinNumber<N extends Digit[]> = ParseInt<JoinNumberHelper<N>>;
type PadList<L extends unknown[], N extends number, P = 0, I extends unknown[] = []> = I["length"] extends N
    ? L
    : L[I["length"]] extends undefined
        ? PadList<[P, ...L], N, P, [unknown, ...I]>
        : PadList<L, N, P, [unknown, ...I]>;
type DePadList<L extends unknown[], P = 0> = L extends [P, ...infer XS]
    ? DePadList<XS, P>
    : L;
type AddListsHelper<
    A extends number[],
    B extends number[],
    C extends Digit = 0,
    L = PadList<A, B["length"]>,
    R = PadList<B, A["length"]>
> = L extends [...infer XS extends number[], infer X extends Digit]
    ? R extends [...infer YS extends number[], infer Y extends Digit]
        ? AddDigits<X, Y, C> extends [infer ThisCarry extends Digit, infer ThisDigit extends Digit]
            ? [...AddListsHelper<XS, YS, ThisCarry>, ThisDigit]
            : []
        : []
    : [];
type AddLists<A extends number[], B extends number[]> = DePadList<AddListsHelper<[0, ...A], [0, ...B]>>;
type AddListXTimes<A extends number[], X extends number, O extends number[] = A, I extends unknown[] = [unknown]> = X extends 0
    ? [0]
    : I["length"] extends X
        ? A
        : AddListXTimes<AddLists<A, O>, X, O, [...I, unknown]>;
type Multiply<N extends number, M extends number> = JoinNumber<AddListXTimes<SplitNumber<N>, M>>;
type Abs<N extends number> = `${N}` extends `-${infer X extends number}` ? X : N;
type Square<N extends number, M extends number = Abs<N>> = Multiply<M, M>;
```

```js
// Builder
function square(n: float64): type {
  return literal(n * n);
}

square(3) === type 9;
square(101) === type 10201;
square(-31) === type 961;
```

Sixteen helper types, and every one of them is load-bearing. This is decimal long multiplication: `SplitNumber` turns `101` into `[1, 0, 1]`, `AddDigits` adds two digits by concatenating tuples of their lengths and reading the result off with `Carry`, `PadList` and `DePadList` align the operands and strip the leading zeros, `AddListsHelper` walks the digits right to left threading a carry, and `AddListXTimes` multiplies by repeated addition. Then `JoinNumber` glues the digits back into a string and `ParseInt` turns the string back into a number.

The story of *why* is in the note. In short: the obvious answer squares by building an `n * n`-element tuple and reading its length, and the top-voted answer is a clever refinement of that which factors out trailing zeros so that round numbers stay small. The harness includes `Square<101>` and the refinement dies on it, because 10,201 is larger than a tuple type is allowed to be. Getting past that ceiling requires implementing arithmetic, and this is what implementing arithmetic looks like.

## 27152 · Triangular number

The sum of `1` through `n`.

```ts
// TypeScript, issue #28010 (+10)
type Triangular<N extends number, P extends number[] = [], A extends number[] = []> = P['length'] extends N ? A['length'] : Triangular<N, [0, ...P], [...A, ...P, 0]>
```

```js
// Builder
function triangular(n: uint32): type {
  return literal(n * (n + 1) / 2);
}

triangular(3) === type 6;
triangular(10) === type 55;
triangular(100) === type 5050;
```

Two accumulators, both tuples: `P` counts up to `N` and `A` collects the running sum as a list of that many elements, so `[...A, ...P, 0]` is `A + P + 1` performed as concatenation. `Triangular<100>` builds a five-thousand-element tuple to hold the answer `5050`. The builder can use the closed form, because it has multiplication and division.

## 27862 · CartesianProduct

```ts
// TypeScript, issue #28095 (+5)
// Union<2 | 3> -> [2] | [3]
type Union<T> = T extends T ? [T] : never;

// [1, ...Union<2 | 3>] -> [1, 2] | [1, 3]
type CartesianProduct<T, U> = T extends T ? [T, ...Union<U>] : never;
```

```js
// Builder
function cartesianProduct(T: type, U: type): type {
  return union(arms(T).flatMap(t => arms(U).map(u => tupleOf([t, u]))));
}

cartesianProduct(type 1 | 2, type 'a' | 'b') === type [1, 'a'] | [1, 'b'] | [2, 'a'] | [2, 'b'];
cartesianProduct(type 1 | 2, type 'a' | never) === type [1, 'a'] | [2, 'a'];
cartesianProduct(type 'a', type Function | string) === type ['a', Function] | ['a', string];
```

Two `T extends T` tautologies, one per axis, because a nested loop over two unions needs distribution triggered twice and that is how you trigger it. The comments in the published answer are doing the work the code cannot: they explain that `Union<T>` means "distribute", which is not visible in `T extends T ? [T] : never`. `flatMap` over `map` is the nested loop.

## 27932 · MergeAll

Merge a tuple of objects, unioning the types of shared keys.

```ts
// TypeScript, issue #29394 (+13)
type MergeAll<
  XS extends object[],
  U = XS[number],
  Keys extends PropertyKey = U extends U ? keyof U : never
> = {
  [K in Keys]: U extends U ? U[K & keyof U] : never
}
```

```js
// Builder
function mergeAll(XS: type): type {
  const byName = new Map();
  for (const e of tupleElements(XS))
    for (const p of reflect(e.type).properties)
      byName.set(p.name, [...(byName.get(p.name) ?? []), p.type]);
  return objectOf([...byName].map(([name, types]) => prop(name, union(types))));
}

mergeAll(type []) === type {};
mergeAll(type [{ a: 1 }, { a: 2 }]) === type { a: 1 | 2 };
mergeAll(type [{ a: string }, { a: string }]) === type { a: string };
mergeAll(type [{}, { a: string }]) === type { a: string };
```

Note the difference from `merge` at challenge 599, which was last-wins; this one unions. `U extends U` appears twice, once to collect the keys and once to collect each key's types, so the union is traversed twice because there is no way to hold the grouping. The builder groups once into a `Map` and unions at the end, and the `{ a: string }` case deduplicates because that is what `union` does.

## 27958 · CheckRepeatedTuple

Does the tuple contain a repeated type?

```ts
// TypeScript, issue #28602 (+6). The top-voted answer (#28027, +10) uses `extends` and
// reports a repeat in `[number, 1, string, '1', boolean, true, false, unknown, any]`; see the note.
type Includes<T extends readonly any[], U> = T extends [infer F, ...infer R] ? (IsEqual<U, F> extends true ? true : Includes<R, U>) : false
type IsEqual<A, B> = ((<T>() => T extends A ? true : false) extends (<T>() => T  extends B ? true : false) ? true : false )

type CheckRepeatedTuple<T extends unknown[], U extends unknown[] = []> = T extends [infer F, ...infer Rest]
  ? Includes<U, F> extends true
    ? true
    : CheckRepeatedTuple<Rest, [...U, F]>
  : false
```

```js
// Builder
function checkRepeatedTuple(T: type): type {
  const node = reflect(T);
  if (node.kind !== 'tuple') return type false;   // an array is not a tuple, and not an error here
  const types = node.elements.map(e => e.type);
  return new Set(types).size !== types.length ? type true : type false;
}

checkRepeatedTuple(type [number, number, string, boolean]) === type true;
checkRepeatedTuple(type [1, 2, 3]) === type false;
checkRepeatedTuple(type []) === type false;
checkRepeatedTuple([].<string>) === type false;
checkRepeatedTuple(type [number, 1, string, '1', boolean, true, false, any]) === type false;
checkRepeatedTuple(type [never, any, never]) === type true;
```

The sixth challenge that needs identity, and the fifth whose top-voted answer used `extends` and got it wrong: `L extends R[number]` reports a repeat in `[number, 1, string, '1', boolean, true, false, unknown, any]` because `number extends any`. The passing answer restates the `IsEqual` incantation and builds an `Includes` on top of it, which is challenge 898 reimplemented inside challenge 27958. Two `Set` operations.

## 28333 · Public Type

Drop underscore-prefixed properties.

```ts
// TypeScript, issue #28419 (+4)
type PublicType<T extends object> = { [P in keyof T as P extends `_${any}` ? never : P]: T[P] };
```

```js
// Builder
function publicType(T: type): type {
  return mapProperties(T, p => typeof p.name === 'string' && p.name.startsWith('_') ? null : p);
}

publicType(type { a: uint32 }) === type { a: uint32 };
publicType(type { d: string, _e: string }) === type { d: string };
publicType(type { readonly c?: uint32 }) === type { readonly c?: uint32 };
publicType(type { g: '_g' }) === type { g: '_g' };   // the value starts with _, the key does not
```

`startsWith` against a template pattern, and the `typeof p.name === 'string'` guard is doing what TypeScript's `P extends \`_${any}\`` does implicitly by failing to match symbols.

## 29650 · ExtractToObject

Inline a nested object's properties into the parent.

```ts
// TypeScript, issue #30450 (+8)
type ExtractToObject<T, P extends keyof T> = Omit<Omit<T, P> & T[P], never>
```

```js
// Builder
function extractToObject(T: type, P: type): type {
  const key = literalValues(P)[0];
  const properties = reflect(T).properties;
  const nested = properties.find(p => p.name === key);
  return objectOf([...properties.filter(p => p.name !== key), ...reflect(nested.type).properties]);
}

extractToObject(type { id: '1', myProp: { foo: '2' } }, type 'myProp') === type { id: '1', foo: '2' };
extractToObject(type { id: '1', prop1: { zoo: '2' }, prop2: { foo: '4' } }, type 'prop2')
  === type { id: '1', prop1: { zoo: '2' }, foo: '4' };
```

The outer `Omit<..., never>` omits nothing. It is the intersection-flattening no-op again, fourth sighting in this document, and by now it reads as punctuation: the way you write "and please make this an object" in a language where `A & B` is not one.

## 29785 · Deep Omit

Omit by dotted path.

```ts
// TypeScript, issue #30825 (+11)
type DeepOmit<O, P extends string> = P extends `${infer K}.${infer Rest}` ? {
  [key in keyof O]: key extends K ? DeepOmit<O[key], Rest> : O[key]
} : Omit<O, P>
```

```js
// Builder
function deepOmit(O: type, path: string): type {
  const [head, ...rest] = path.split('.');
  return rest.length === 0
    ? objectOf(reflect(O).properties.filter(p => p.name !== head))
    : mapProperties(O, p => p.name === head ? { ...p, type: deepOmit(p.type, rest.join('.')) } : p);
}

type Obj = { person: { name: string, age: { value: uint32 } } };
deepOmit(Obj, 'person') === type {};
deepOmit(Obj, 'person.name') === type { person: { age: { value: uint32 } } };
deepOmit(Obj, 'person.age.value') === type { person: { name: string, age: {} } };
deepOmit(Obj, 'name') === Obj;
```

Close on both sides. `` P extends `${infer K}.${infer Rest}` `` is a split on the first dot, and `split('.')` is a split on all of them, which is why the builder rejoins the tail. A fair draw, and a good example of template literal patterns being the right tool: parsing a path is what they are for.

## 30301 · IsOdd

```ts
// TypeScript, issue #32994 (+6). The top-voted answer (#30334, +14) reports `2.3` as odd; see the note.
type IsOdd<T extends number>
  = `${T}` extends `${any}e${any}` | `${any}.${any}` ? false
  : `${T}` extends `${any}${1 | 3 | 5 | 7 | 9}` ? true : false
```

```js
// Builder
function isOdd(T: type): type {
  const node = reflect(T);
  return node.kind === 'literal' && Number.isInteger(node.value) && node.value % 2 !== 0
    ? type true : type false;
}

isOdd(type 5) === type true;
isOdd(type 2023) === type true;
isOdd(type 1926) === type false;
isOdd(type 2.3) === type false;
isOdd(type 3e23) === type false;   // an integer, but 3e23 % 2 is 0 at this magnitude
isOdd(type 3e0) === type true;     // 3e0 is 3
isOdd(float64) === type false;     // not a literal
```

The passing answer's first line is the whole lesson: before it can look at the last digit, it has to rule out the two ways a number's *string form* can lie about it, namely exponent notation and a decimal point. `3e23` ends in `3` and is even; `2.3` ends in `3` and is not an integer. The top-voted answer skips that line and reports `2.3` as odd. `% 2` does not care how the number was written, because it is looking at the number.

## 30430 · Tower of Hanoi

The move list for `n` rings.

```ts
// TypeScript, issue #30769 (+5)
type Hanoi<Rings extends number, FromRod extends string = 'A', ToRod extends string = 'B', IntermediateRod extends string = 'C', Acc extends '🇺🇦'[] = []> = Acc['length'] extends Rings ? [] : [ ...Hanoi<Rings, FromRod, IntermediateRod, ToRod, [...Acc, '🇺🇦']>, [FromRod, ToRod], ...Hanoi<Rings, IntermediateRod, ToRod, FromRod, [...Acc, '🇺🇦']> ]
```

```js
// Builder
function hanoi(rings: uint32, from: string = 'A', to: string = 'B', via: string = 'C'): type {
  const moves = (n: uint32, f: string, t: string, v: string): [].<type> =>
    n === 0 ? [] : [...moves(n - 1, f, v, t), tupleOf([literal(f), literal(t)]), ...moves(n - 1, v, t, f)];
  return tupleOf(moves(rings, from, to, via));
}

hanoi(0) === type [];
hanoi(1) === type [['A', 'B']];
hanoi(2) === type [['A', 'C'], ['A', 'B'], ['C', 'B']];
hanoi(3) === type [['A', 'B'], ['A', 'C'], ['B', 'C'], ['A', 'B'], ['C', 'A'], ['C', 'B'], ['A', 'B']];
```

The nicest draw in the document. Hanoi is three lines in any language, and both of these are the same three lines: move `n-1` aside, move one, move `n-1` back. Nothing is encoded and nothing is worked around, because the algorithm is recursion over a structure and recursion over a structure is what conditional types do well.

Except for the counter. `Acc extends '🇺🇦'[] = []` counts the rings, one flag per level, and the type of the filler is arbitrary because only the length is ever read. That the author picked a Ukrainian flag is the friendliest thing in this repository, and it is also the tell: when the element type of your counter can be a flag emoji, the counter is not a number.

## 30958 · Pascal's triangle

```ts
// TypeScript, issue #31537 (+0)
// Create the part of the row between the first and last 1s, but as tuples of that length.
// i.e the [3, 3] in [1, 3, 3, 1], but as [[x, x, x], [x, x, x]]
// I use tuples so I can easily sum them like `[...t1, ...t2]`
type RowMiddle<PrevRow extends any[][]> = PrevRow extends [
  infer H extends any[],
  infer H2 extends any[],
  ...infer R extends any[][],
]
  ? [[...H, ...H2], ...RowMiddle<[H2, ...R]>]
  : [];

type Lengths<Counters extends any[][]> = Counters extends [
  infer H extends any[],
  ...infer R extends any[][],
]
  ? [H["length"], ...Lengths<R>]
  : [];

type Pascal<
  N extends number,
  Result extends number[][] = [],
  PrevRow extends any[][] = [],
> = Result["length"] extends N
  ? Result
  : Result extends []
    ? Pascal<N, [[1]], [[any]]>
    : [[any], ...RowMiddle<PrevRow>, [any]] extends infer newRow extends any[][]
      ? Pascal<N, [...Result, Lengths<newRow>], newRow>
      : never;
```

```js
// Builder
function pascal(n: uint32): type {
  const rows = [];
  for (let i = 0; i < n; i++) {
    const prev = rows[i - 1] ?? [];
    rows.push(Array.from({ length: i + 1 }, (_, j) => j === 0 || j === i ? 1 : prev[j - 1] + prev[j]));
  }
  return tupleOf(rows.map(row => tupleOf(row.map(literal))));
}

pascal(1) === type [[1]];
pascal(3) === type [[1], [1, 1], [1, 2, 1]];
pascal(5) === type [[1], [1, 1], [1, 2, 1], [1, 3, 3, 1], [1, 4, 6, 4, 1]];
```

Read the author's comment: *"I use tuples so I can easily sum them like `[...t1, ...t2]`."* Every number in the triangle is a tuple of that many `unknown`s, addition is `[...H, ...H2]`, and `Lengths` is the pass at the end that converts a triangle of lists back into a triangle of numbers. Twelfth entry in the arithmetic column, and the most literal one: the encoding is stated in a comment, cheerfully, as a technique. `prev[j - 1] + prev[j]` is `+`.

## 30970 · IsFixedStringLiteralType

Is `S` exactly one string literal?

```ts
// TypeScript, issue #33984 (+7)
type IsFixedStringLiteralType<S extends string> = {} extends Record<S, 1> ? false : Equal<[S], S extends unknown ? [S] : never>
```

```js
// Builder
function isFixedStringLiteralType(S: type): type {
  const node = reflect(S);
  return node.kind === 'literal' && typeof node.value === 'string' ? type true : type false;
}

isFixedStringLiteralType(type 'ABC') === type true;
isFixedStringLiteralType(string) === type false;              // not a literal
isFixedStringLiteralType(type 'ABC' | 'DEF') === type false;  // not one literal
isFixedStringLiteralType(never) === type false;               // the empty union
isFixedStringLiteralType(stringPattern`${string}`) === type false;
isFixedStringLiteralType(literal(`${true}`)) === type true;   // that is the literal 'true'
```

Two separate hacks in one line, because the question is really two questions. `{} extends Record<S, 1>` filters out the wide types: a `Record` keyed by `string` or by a template pattern has no required keys, so `{}` satisfies it, while a `Record<'ABC', 1>` does not. Then `Equal<[S], S extends unknown ? [S] : never>` filters out unions, by distributing `S` and checking whether the result is still the same single thing. Seventh challenge to need the identity incantation. The reflection node answers both parts with `kind` and a `typeof`, because "is it one literal" is a question about the type's shape and the shape is right there.

## 34007 · Compare Array Length

```ts
// TypeScript, issue #34089 (+9)
type CompareArrayLength<T extends unknown[], U extends unknown[]> = T['length'] extends U['length']
  ? 0
  : `${U['length']}` extends keyof T ? 1 : -1;
```

```js
// Builder
function compareArrayLength(T: type, U: type): type {
  return literal(Math.sign(tupleElements(T).length - tupleElements(U).length));
}

compareArrayLength(type [1, 2, 3, 4], type [5, 6]) === type 1;
compareArrayLength(type [1, 2], type [3, 4, 5, 6]) === type -1;
compareArrayLength(type [], type []) === type 0;
```

A small gem of a workaround. Having no `<`, it asks whether `U`'s length is a valid *index* of `T`: if `U` has 2 elements and `T` has 4, then `'2'` is a key of `T`, so `T` is longer. Comparison by key membership. `Math.sign` of a subtraction.

## 34857 · Defined Partial Record

Every non-empty subset of the properties.

```ts
// TypeScript, issue #35291 (+11)
type DefinedPartial<T, K extends keyof T = keyof T> = K extends unknown
  ? T | DefinedPartial<Omit<T, K>>
  : never;
```

```js
// Builder
function definedPartial(T: type): type {
  const properties = reflect(T).properties;
  const subsets = properties.reduce((acc, p) => [...acc, ...acc.map(set => [...set, p])], [[]]);
  return union(subsets.filter(set => set.length > 0).map(set => objectOf(set)));
}

definedPartial(type Record.<'a' | 'b', string>)
  === type { a: string } | { b: string } | { a: string, b: string };
```

`K extends unknown` is `K extends K` wearing a different hat: another tautology whose only job is to make the conditional distribute over the key union, so that `Omit<T, K>` runs once per key and the recursion sweeps out the power set. The builder's `reduce` is the standard power-set fold, and `filter(set => set.length > 0)` is the word "defined" in the challenge's title.

## 35045 · Longest Common Prefix

```ts
// TypeScript, issue #35251 (+5)
type LongestCommonPrefix<T extends string[], P extends string = ''>
  = T extends [`${P}${infer Next}${any}`, ...any]
  ? T extends `${P}${Next}${any}`[]
    ? LongestCommonPrefix<T, `${P}${Next}`>
    : P // the longest
  : P   // T is empty or end of T[0]
```

```js
// Builder
function longestCommonPrefix(T: type): type {
  const strings = tupleElements(T).map(e => literalValues(e.type)[0]);
  const first = strings[0] ?? '';
  let i = 0;
  while (i < first.length && strings.every(s => s[i] === first[i])) i++;
  return literal(first.slice(0, i));
}

longestCommonPrefix(type ['flower', 'flow', 'flight']) === type 'fl';
longestCommonPrefix(type ['dog', 'racecar', 'race']) === type '';
longestCommonPrefix(type ['a', 'a', '']) === type '';
longestCommonPrefix(type ['a', 'a', 'a']) === type 'a';
```

The second line is the clever one: `` T extends `${P}${Next}${any}`[] `` asks whether *every* element of the tuple starts with the prefix, by testing the tuple against an array type whose element is a pattern. A tuple is assignable to `X[]` exactly when all its elements are assignable to `X`, so assignability is being used as a universal quantifier. `.every` is the quantifier.

## 35191 · Trace

The diagonal of a square matrix, as a union.

```ts
// TypeScript, issue #35247 (+9)
type Trace<T extends any[][]> = {[P in keyof T]: T[P][P & keyof T[P]]}[number]
```

```js
// Builder
function trace(M: type): type {
  return union(tupleElements(M).map((row, i) => tupleElements(row.type)[i].type));
}

trace(type [[1, 2], [3, 4]]) === type 1 | 4;
trace(type [[0, 1, 1], [2, 0, 2], [3, 3, 0]]) === type 0;
trace(type [['a', 'b', ''], ['c', '', ''], ['d', 'e', 'f']]) === type 'a' | '' | 'f';
```

Concise on both sides. `T[P][P & keyof T[P]]` indexes the row by the same key that selected it, which is the diagonal, and the `& keyof T[P]` is a proof obligation rather than an idea: the compiler needs convincing that `P` is a valid index of the row. `map` over an index and `[i]` twice. The second case collapsing to `0` is union deduplication doing the right thing without being asked.

## 35252 · IsAlphabet

```ts
// TypeScript, issue #35336 (+4)
type IsAlphabet<S extends string> = Uppercase<S> extends Lowercase<S> ? false : true;
```

```js
// Builder
function isAlphabet(s: string): type {
  return s.toUpperCase() !== s.toLowerCase() ? type true : type false;
}

isAlphabet('A') === type true;
isAlphabet('z') === type true;
isAlphabet('9') === type false;
isAlphabet('😂') === type false;
isAlphabet('') === type false;
```

A genuine draw, and a nice one: both sides notice that a letter is exactly a character whose two cases differ. The only difference is that `Uppercase` and `Lowercase` are intrinsics the compiler implements in C++ and cannot be written in the type language, which is what the next challenge is about.

## 35991 · MyUppercase

Implement `Uppercase` without using it.

```ts
// TypeScript, issue #36596 (+8)
interface Mapping {
  a: 'A'
  b: 'B'
  c: 'C'
  d: 'D'
  e: 'E'
  f: 'F'
  g: 'G'
  h: 'H'
  i: 'I'
  j: 'J'
  k: 'K'
  l: 'L'
  m: 'M'
  n: 'N'
  o: 'O'
  p: 'P'
  q: 'Q'
  r: 'R'
  s: 'S'
  t: 'T'
  u: 'U'
  v: 'V'
  w: 'W'
  x: 'X'
  y: 'Y'
  z: 'Z'
}

type MyUppercase<T extends string> = T extends `${infer F}${infer R}`
  ? `${F extends keyof Mapping ? Mapping[F] : F}${MyUppercase<R>}`
  : ''
```

```js
// Builder
function myUppercase(s: string): type {
  return literal(s.toUpperCase());
}

myUppercase('a') === type 'A';
myUppercase('Z') === type 'Z';
myUppercase('A z h yy 😃cda\n\t  a   ') === type 'A Z H YY 😃CDA\n\t  A   ';
```

Twenty-six lines of table, one per letter, written out by hand, because there is no operation that maps a character to another character. It is the right answer to the question asked, and the question is only askable in a language where `Uppercase` is a compiler intrinsic rather than a function.

The two are not equivalent, and the difference favors neither side automatically. `toUpperCase` is Unicode-correct: `myUppercase('ß')` is `'SS'` and `myUppercase('i')` in a Turkish locale is a discussion. The table is ASCII by construction, which is wrong for most of the world's alphabets and exactly right for this test. But the builder's version is *the same function the runtime already has*, and if it is wrong for a caller's purposes they can pass their own. The table cannot be fixed; it can only be extended, letter by letter, forever.

## 6 · Simple Vue

The first hard challenge. Type an options object where `computed` sees `data`'s shape as `this`, and `methods` sees data, methods, and the *results* of the computed getters.

```ts
// TypeScript, issue #14322 (+15). The top-voted answer (#11, +20) no longer passes; see the note.
type GetComputed<TComputed> = {
  [key in keyof TComputed]: TComputed[key] extends () => infer Result ? Result : never;
};

type Options<TData, TComputed, TMethods> = {
  data: (this: void) => TData;
  computed: TComputed & ThisType<TData>;
  methods: TMethods & ThisType<TData & GetComputed<TComputed> & TMethods>;
};

declare function SimpleVue<TData, TComputed, TMethods>(
  options: Options<TData, TComputed, TMethods>
): unknown;
```

```js
// Builder
function computedResults(C: type): type {
  return mapProperties(C, p => ({ ...p, type: returnType(p.type) }));
}

function withThisOnMethods(O: type, Self: type): type {
  return mapProperties(O, p => ({ ...p, type: withThisType(p.type, Self) }));
}

function vueOptions(D: type, C: type, M: type): type {
  const self = Reflect.makeType({ kind: 'intersection', members: [D, computedResults(C), M] });
  return objectOf([
    prop('data', withThisType(fn([], D), type void)),
    prop('computed', withThisOnMethods(C, D)),
    prop('methods', withThisOnMethods(M, self)),
  ]);
}

declare function simpleVue<D, C, M>(options: vueOptions(D, C, M)): any;

simpleVue({
  data() { return { firstname: 'Type', lastname: 'Challenges', amount: 10 }; },
  computed: {
    fullname() { return `${this.firstname} ${this.lastname}`; },   // this: D
  },
  methods: {
    getRandom() { return Math.random(); },
    hi() { return this.fullname.toLowerCase(); },                  // this: D & computed results & M
    test() { const fullname: string = this.fullname; },           // this.fullname checks as string
  },
});
```

A near-draw, and the closest thing to a real API in the document. Both sides need the same three facts and state them in the same order. `ThisType<T>` is a marker interface the checker recognizes by name and erases; `withThisType` writes the `thisType` field the signature already has. The intersection is an intersection in both.

The one real difference is `GetComputed` against `computedResults`: TypeScript maps over the computed object and pattern-matches each property against `() => infer Result` to extract the return type, while the builder calls `returnType`. And `data: (this: void) => TData` against `withThisType(fn([], D), void)` is the same statement twice, which is what a draw looks like.

## 17 · Currying 1

```ts
// TypeScript, issue #28634 (+20)
// See https://stackoverflow.com/a/72244704/388951
type FirstAsTuple<T extends any[]> = T extends [any, ...infer R]
  ? T extends [...infer F, ...R]
    ? F
    : never
  : never

type Curried<F> = F extends (...args: infer Args) => infer Return
  ? Args['length'] extends 0 | 1
    ? F
    : Args extends [any, ...infer Rest]
    ? (...args: FirstAsTuple<Args>) => Curried<(...rest: Rest) => Return>
    : never
  : never

declare function Currying<T extends Function>(fn: T): Curried<T>
```

```js
// Builder
function curried(F: type): type {
  const node = reflect(F);
  if (node.kind !== 'function') throw new TypeError(`curried: ${String(F)} is not a function type`);
  const signature = node.signatures[0];
  if (signature.parameters.length <= 1) return F;
  const [first, ...rest] = signature.parameters;
  const tail = fn(rest.map(p => p.type), signature.return.type);
  return Reflect.makeType({ kind: 'function', signatures: [{
    ...signature,
    parameters: [first],
    return: { ...signature.return, type: curried(tail) },
  }] });
}

declare function currying<T>(f: T): curried(T);

const curried1 = currying((a: string, b: float64, c: boolean) => true);
Reflect.typeOf(curried1) === type (a: string) => (b: float64) => (c: boolean) => true;

const curried3 = currying(() => true);
Reflect.typeOf(curried3) === type () => true;
```

`FirstAsTuple` is the artifact here, and it comes with a link to Stack Overflow in the published answer, which is worth noting on its own. What it does: to get a one-element tuple containing the first parameter, it takes the whole list, infers the tail, and then infers what has to be *in front of* the tail. It computes the first element by subtracting the rest from the whole.

The reason is that `[Args[0]]` would be wrong: it produces `[string]`, and the parameter's name, `a`, is gone. So the trick exists to carry the label along, and the label is the interesting part, because names are not part of a function type's identity. I checked: the naive version without `FirstAsTuple` passes every test in this challenge, and `Equal<[a: string], [string]>` is `true`. So the trick is not needed to be *correct*; it is needed so that the type displayed on hover says `(a: string) => ...` rather than `(arg: string) => ...`.

That is a real thing to want, and it is precisely why this proposal's parameter records carry `name` as non-canonical data rather than as part of the type: names matter to the reader, not to the checker. The builder passes `first` through untouched, and the name rides along in the record because there is a field for it. See the note from challenge 191, which needed the opposite thing from the same fact.

## 55 · Union to Intersection

```ts
// TypeScript, issue #122 (+29)
type UnionToIntersection<U> = (U extends any ? (arg: U) => any : never) extends ((arg: infer I) => void) ? I : never
```

```js
// Builder
function unionToIntersection(U: type): type {
  return Reflect.makeType({ kind: 'intersection', members: arms(U) });
}

unionToIntersection(type 'foo' | 42 | true) === type 'foo' & 42 & true;
unionToIntersection(type (() => 'foo') | ((i: 42) => true)) === type (() => 'foo') & ((i: 42) => true);
```

One of the two or three most-cited incantations in TypeScript, and it earns its reputation. Read what it does: distribute the union into a union of *function types taking each arm as a parameter*, then infer that parameter back out of the whole union at once. The answer is an intersection because function parameters are contravariant, so a value assignable to all of `(arg: 'foo') => any`, `(arg: 42) => any`, and `(arg: true) => any` must accept all three, and inference in that position collects the constraints by intersecting them.

Nothing in that chain is about unions or intersections. It is a variance rule, observed indirectly, through a function type nobody calls, to compute a set operation. `arms` and a node with `kind: 'intersection'` is the same operation named.

## 57 · Get Required

```ts
// TypeScript, issue #285 (+18)
type GetRequired<T> = { [P in keyof T as T[P] extends Required<T>[P] ? P : never]: T[P] };
```

```js
// Builder
function getRequired(T: type): type {
  return objectOf(reflect(T).properties.filter(p => !p.optional));
}

getRequired(type { foo: uint32, bar?: string }) === type { foo: uint32 };
getRequired(type { foo: undefined, bar?: undefined }) === type { foo: undefined };
```

## 59 · Get Optional

```ts
// TypeScript, issue #286 (+6)
type GetOptional<T> = {[P in keyof T as T[P] extends Required<T>[P] ? never: P]: T[P]}
```

```js
// Builder
function getOptional(T: type): type {
  return objectOf(reflect(T).properties.filter(p => p.optional));
}

getOptional(type { foo: uint32, bar?: string }) === type { bar?: string };
getOptional(type { foo: undefined, bar?: undefined }) === type { bar?: undefined };
```

## 89 · Required Keys

```ts
// TypeScript, issue #2664 (+7)
type RequiredKeys<T , K = keyof T> = K extends keyof T ? T extends Required<Pick<T,K>> ? K : never
  :never
```

```js
// Builder
function requiredKeys(T: type): type {
  return union(reflect(T).properties.filter(p => !p.optional).map(p => literal(p.name)));
}

requiredKeys(type { a: uint32, b?: string }) === type 'a';
requiredKeys(type { a: undefined, b?: undefined, c: string, d: null }) === type 'a' | 'c' | 'd';
requiredKeys(type {}) === never;
```

## 90 · Optional Keys

```ts
// TypeScript, issue #210 (+3)
type OptionalKeys<T> = {[P in keyof T]-?: {} extends Pick<T,P> ? P:never}[keyof T]
```

```js
// Builder
function optionalKeys(T: type): type {
  return union(reflect(T).properties.filter(p => p.optional).map(p => literal(p.name)));
}

optionalKeys(type { a: uint32, b?: string }) === type 'b';
optionalKeys(type { a: undefined, b?: undefined, c?: string, d?: null }) === type 'b' | 'c' | 'd';
optionalKeys(type {}) === never;
```

Four challenges, one bit. All four ask whether a property is optional, and none of them can ask. Count the ways the published answers find to work around that:

- `T[P] extends Required<T>[P]` (57, 59): build the fully-required version of the whole object, index it, and compare the property's type against its own de-optionalized self. An optional `bar?: string` has type `string | undefined`, which does not fit `Required<T>['bar']`, which is `string`.
- `T extends Required<Pick<T, K>>` (89): pick the single key, require it, and ask whether the original object still satisfies the result.
- `{} extends Pick<T, P>` (90): pick the single key and ask whether the empty object satisfies it, which is true exactly when the key is optional and therefore not needed.

Three distinct constructions, all of them building a whole new object type in order to look at one flag on one property, and the first one only works because optionality and `| undefined` happen to be entangled in the type's meaning. `TypePropertyReflection` has a field called `optional`. It is a boolean. The builders read it.

## 112 · Capitalize Words

```ts
// TypeScript, issue #19571 (+8). The top-voted answer (#2896, +8) recurses once per
// character and dies on the fifty-character test case; see the note.
type CapitalizeWords<
  S extends string,
  W extends string = ''
> = S extends `${infer A}${infer B}`
  ? Uppercase<A> extends Lowercase<A>
    ? `${Capitalize<`${W}${A}`>}${CapitalizeWords<B>}`
    : CapitalizeWords<B, `${W}${A}`>
  : Capitalize<W>
```

```js
// Builder
function capitalizeWords(s: string): type {
  return literal(s.replace(/\p{L}+/gu, w => `${w[0].toUpperCase()}${w.slice(1)}`));
}

capitalizeWords('foobar') === type 'Foobar';
capitalizeWords('FOOBAR') === type 'FOOBAR';
capitalizeWords('foo bar.hello,world') === type 'Foo Bar.Hello,World';
capitalizeWords('aa!bb@cc#dd$ee%ff^gg&hh*ii(jj)kk_ll+mm{nn}oo|pp🤣qq')
  === type 'Aa!Bb@Cc#Dd$Ee%Ff^Gg&Hh*Ii(Jj)Kk_Ll+Mm{Nn}Oo|Pp🤣Qq';
```

`Uppercase<A> extends Lowercase<A>` is challenge 35252's letter test, reused as a word-boundary detector: a character is punctuation exactly when its two cases agree. The `W` accumulator exists to make the common branch a tail call, which is the whole reason this answer passes and the top-voted one does not. `\p{L}+` says "runs of letters" and the regex engine finds the boundaries.

## 114 · CamelCase

```ts
// TypeScript, issue #30845 (+10)
type CamelCase<S extends string> = S extends `${infer L}_${infer R1}${infer R2}`
	? Uppercase<R1> extends Lowercase<R1>
		? `${Lowercase<L>}_${CamelCase<`${R1}${R2}`>}`
		: `${Lowercase<L>}${Uppercase<R1>}${CamelCase<R2>}`
	: Lowercase<S>;
```

```js
// Builder
function camelCase(s: string): type {
  return literal(s.toLowerCase().replace(/_(\p{L})/gu, (_, c) => c.toUpperCase()));
}

camelCase('FOOBAR') === type 'foobar';
camelCase('foo_bar') === type 'fooBar';
camelCase('foo__bar') === type 'foo_Bar';       // the second _ has no letter after it
camelCase('foo_$bar') === type 'foo_$bar';      // $ is not a letter
camelCase('foo_bar_$') === type 'fooBar_$';
camelCase('😎') === type '😎';
```

The same letter test a third time, now deciding whether an underscore introduces a word or is just an underscore. `_(\p{L})` is that condition as a pattern, and the three-way split of `${infer L}_${infer R1}${infer R2}` is what a regex capture group does.

## 147 · C-printf Parser

```ts
// TypeScript, issue #370 (+10)
type ControlsMap = {
  c: 'char'
  s: 'string'
  d: 'dec'
  o: 'oct'
  h: 'hex'
  f: 'float'
  p: 'pointer'
}

type ParsePrintFormat<S extends string> = S extends `${infer Start}%${infer Letter}${infer Rest}`
  ? (Letter extends keyof ControlsMap
      ? [ControlsMap[Letter], ...ParsePrintFormat<Rest>]
      : ParsePrintFormat<Rest>)
  : []
```

```js
// Builder
const controlsMap = { c: 'char', s: 'string', d: 'dec', o: 'oct', h: 'hex', f: 'float', p: 'pointer' };

function parsePrintFormat(s: string): type {
  const out = [];
  for (let i = 0; i < s.length - 1; i++) {
    if (s[i] !== '%') continue;
    const letter = s[i + 1];
    if (letter === '%') { i++; continue; }             // %% is an escaped percent
    if (letter in controlsMap) out.push(literal(controlsMap[letter]));
  }
  return tupleOf(out);
}

parsePrintFormat('') === type [];
parsePrintFormat('The result is %d.') === type ['dec'];
parsePrintFormat('The result is %%d.') === type [];
parsePrintFormat('The result is %%%d.') === type ['dec'];
parsePrintFormat('Hello %s: score is %d.') === type ['string', 'dec'];
```

A close one. `ControlsMap` is a lookup table in both, and the only reason TypeScript's is a *type* is that a type is the only kind of value a type-level program can hold. `Letter extends keyof ControlsMap` is `letter in controlsMap`. Note how the `%%` case works on the TypeScript side: the first `%` matches, `Letter` binds to the second `%`, which is not a key of `ControlsMap`, so nothing is emitted and the scan continues past both. The escape is handled by accident, correctly, and the author would have had to think about it either way.

## 213 · Vue Basic Props

Type a Vue options object, inferring prop types from their constructors.

```ts
// TypeScript, issue #12600 (+3). The top-voted answer (#215, +5) does not compile; see the note.
type GetComputed<TComputed> = {
  [key in keyof TComputed]: TComputed[key] extends () => infer Result
    ? Result
    : never;
};

type Union<TValue> = TValue extends Array<unknown> ? TValue[number] : TValue;

type MyReturnType<TFunction> = TFunction extends () => infer Result
  ? Result
  : TFunction extends new (...params: any[]) => infer Result
  ? Result
  : any;

type InferProps<TProps> = {
  [key in keyof TProps]: MyReturnType<
    Union<TProps[key] extends { type: infer Type } ? Type : TProps[key]>
  >;
};

type Options<TProps, TData, TComputed, TMethods> = {
  props: TProps;
  data: (this: InferProps<TProps>) => TData;
  computed: TComputed & ThisType<TData>;
  methods: TMethods &
    ThisType<GetComputed<TComputed> & TMethods & InferProps<TProps>>;
};

declare function VueBasicProps<TProps, TData, TComputed, TMethods>(
  options: Options<TProps, TData, TComputed, TMethods>
): unknown;
```

```js
// Builder
function inferPropType(P: type): type {
  const node = reflect(P);
  const declared = node.kind === 'object'
    ? node.properties.find(p => p.name === 'type')?.type ?? any
    : P;
  const each = (C: type): type => {
    const node = reflect(C);
    if (node.kind === 'tuple') return union(tupleElements(C).map(e => each(e.type)));
    return node.kind === 'function' ? returnType(C) : C;   // String's call signature says string; a class's type object already denotes its instances
  };
  return each(declared);
}

function vueProps(Props: type, D: type, C: type, M: type): type {
  const props = mapProperties(Props, p => ({ ...p, type: inferPropType(p.type) }));
  const self = Reflect.makeType({ kind: 'intersection', members: [props, D, computedResults(C), M] });   // both helpers from challenge 6
  return objectOf([
    prop('props', Props),
    prop('data', withThisType(fn([], D), props)),
    prop('computed', withThisOnMethods(C, D)),
    prop('methods', withThisOnMethods(M, self)),
  ]);
}

declare function vueBasicProps<P, D, C, M>(options: vueProps(P, D, C, M)): any;

class ClassA {}
vueBasicProps({
  props: { propA: {}, propB: { type: String }, propD: { type: ClassA }, propE: { type: [String, Number] } },
  data() {
    Reflect.typeOf(this.propB) === string;
    Reflect.typeOf(this.propD) === ClassA;
    Reflect.typeOf(this.propE) === type string | float64;
    return { toggle: false };
  },
  computed: { reversedPropB() { return this.propB.split('').reverse().join(''); } },
  methods: { toggleIt() { this.toggle = !this.toggle; } },
});
```

Challenge 6 with a second layer: the props are declared as *constructors*, and the type of `propB: { type: String }` is whatever `new String()` produces. `MyReturnType` handles that by asking for the return type of the call signature first and the construct signature second, `Union` spreads an array of constructors into a union, and `InferProps` maps the whole thing. The builder's `each` is those two in three lines: a constructor function's call signature answers for the wrapped primitives, a class's type object already denotes its instances and passes through unchanged, and the tuple branch is `Union`.

This is a big program in both languages and neither is embarrassed. What separates them is that `inferPropType` can be read, tested, and stepped through, and `InferProps` can only be run.

## 223 · IsAny

```ts
// TypeScript, issue #232 (+31)
type IsAny<T> = 0 extends (1 & T) ? true : false;
```

```js
// Builder
function isAny(T: type): type {
  return T === any ? type true : type false;
}

isAny(any) === type true;
isAny(undefined) === type false;
isAny(never) === type false;
isAny(string) === type false;
```

The last of the identity family, and the most elegant hack in the document. `1 & T` is `1` for almost every `T`, and `0 extends 1` is false. But `1 & any` is `any`, and `0 extends any` is true. So the test is: *intersect with a literal and see whether the literal survives*, because `any` is the one type that eats an intersection. It is three tokens and it is correct and it is completely opaque, and it works because of a special case in intersection reduction that exists for unrelated reasons.

`T === any` is the same question. `any` is a type object; there is one of it; you can compare against it.

## 270 · Typed Get

Read a value out of an object by dotted path.

```ts
// TypeScript, issue #397 (+2). The top-voted answer (#368, +13) splits on the dot first
// and misses the literal key; see the note.
type Get<T, K> = K extends keyof T
                    ? T[K]
                    : K extends `${infer First}.${infer Rest}`
                        ? First extends keyof T
                                    ? Get<T[First], Rest>
                                    : never
                        : never
```

```js
// Builder
function get(T: type, path: string): type {
  const at = (name: string): type | undefined =>
    reflect(T).properties.find(p => p.name === name)?.type;
  const exact = at(path);
  if (exact !== undefined) return exact;      // a literal key beats the dotted reading
  const dot = path.indexOf('.');
  if (dot === -1) return never;
  const head = at(path.slice(0, dot));
  return head === undefined ? never : get(head, path.slice(dot + 1));
}

type Data = {
  foo: { bar: { value: 'foobar', count: 6 }, included: true },
  'foo.baz': false,
  hello: 'world',
};
get(Data, 'hello') === type 'world';
get(Data, 'foo.bar.count') === type 6;
get(Data, 'foo.baz') === type false;        // the key 'foo.baz' exists
get(Data, 'no.existed') === never;
```

The trap is in the fixture, and it is a good one: `Data` contains a key literally named `'foo.baz'`, so `Get<Data, 'foo.baz'>` must find the key rather than walk the path. The top-voted answer tests the dotted pattern first, splits into `'foo'` and `'baz'`, finds no `baz` under `foo`, and answers `never`. Order of the two branches is the entire challenge, and both solutions get it right by putting the exact-key test first.

## 300 · String to Number

```ts
// TypeScript, issue #398 (+6). The top-voted answer (#499, +15) counts up from zero and
// never terminates on the non-numeric case; see the note.
type Num = readonly 0[]

type NumToNumber<N extends Num> = N['length']

type Digit = '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'

type MultiplyToDigit<D extends Digit, N extends Num> = {
  '0': [],
  '1': N,
  '2': [...N, ...N],
  '3': [...N, ...N, ...N],
  '4': [...N, ...N, ...N, ...N],
  '5': [...N, ...N, ...N, ...N, ...N],
  '6': [...N, ...N, ...N, ...N, ...N, ...N],
  '7': [...N, ...N, ...N, ...N, ...N, ...N, ...N],
  '8': [...N, ...N, ...N, ...N, ...N, ...N, ...N, ...N],
  '9': [...N, ...N, ...N, ...N, ...N, ...N, ...N, ...N, ...N],
}[D]

type One = [0]

type Sum<N extends Num, M extends Num> = [...N, ...M]

type MultiplyTo10<N extends Num> = MultiplyToDigit<'2', MultiplyToDigit<'5', N>>

type GetBase<Number extends string> = Number extends Digit
  ? MultiplyTo10<One>
  : (Number extends `${infer D}${infer Rest}` ? MultiplyTo10<GetBase<Rest>> : never)

type StrToNum<Str extends string> = Str extends Digit
  ? MultiplyToDigit<Str, One>
  : (Str extends `${infer D}${infer Rest}`
    ? (D extends Digit
      ? Sum<MultiplyToDigit<D, GetBase<Rest>>, StrToNum<Rest>>
      : never
    )
    : never
  )

type ToNumber<S extends string> = NumToNumber<StrToNum<S>>
```

```js
// Builder
function toNumber(s: string): type {
  return /^\d+$/.test(s) ? literal(Number(s)) : never;
}

toNumber('0') === type 0;
toNumber('27') === type 27;
toNumber('18@7_$%') === never;
```

Positional decimal notation, implemented. `MultiplyToDigit` is a nine-way lookup table whose entries are the operand spread that many times; `MultiplyTo10` multiplies by ten by multiplying by five and then by two, because there is no `[...N, ...N, ...N, ...N, ...N, ...N, ...N, ...N, ...N, ...N]` entry; `GetBase` computes the place value of the leading digit by recursing down the string and multiplying by ten on the way back up; `StrToNum` is `digit * base + rest`. This is what "parse an integer" costs.

The top-voted answer is the one everybody writes: count a tuple up from zero until its length stringifies to `S`. That is fine for `'27'`, and on `'18@7_$%'` it counts forever, which is why the harness includes it and why the compiler says the instantiation is excessively deep. `Number(s)` and a regex.

## 399 · Tuple Filter

Drop elements matching `F`.

```ts
// TypeScript, issue #456 (+9)
type FilterOut<T extends any[], F>
  = T extends [infer R, ...infer Rest]
      ? [R] extends [F]
        ? FilterOut<Rest, F>
        : [R, ...FilterOut<Rest, F>]
      : []
```

```js
// Builder
function filterOut(T: type, F: type): type {
  const drop = new Set(arms(F));
  return tupleOf(tupleElements(T).map(e => e.type).filter(t => arms(t).every(arm => drop.has(arm))
    ? false : true));
}

filterOut(type [1, never, 'a'], never) === type [1, 'a'];
filterOut(type [never, 1, 'a', undefined, false, null], type never | null | undefined) === type [1, 'a', false];
filterOut(type [float64 | null | undefined, never], type never | null | undefined) === type [float64 | null | undefined];
```

Both wrappers in `[R] extends [F]` are load-bearing and for different reasons: the left one stops `R` distributing so that `never` is testable at all, and the right one stops the *union* `never | null | undefined` from being taken apart, so that `number | null | undefined` is not dropped just because two of its three arms match. The last test case exists to catch exactly that. The builder's `arms(t).every(arm => drop.has(arm))` says the same thing in the affirmative: drop an element when every arm of it is in the drop set.

## 472 · Tuple to Enum Object

```ts
// TypeScript, issue #492 (+21)
/**
 * TupleKeys<[3, 4]> = 0 | 1.
 */
type TupleKeys<T extends readonly unknown[]> = T extends readonly [
  infer Head,
  ...infer Tail
]
  ? TupleKeys<Tail> | Tail["length"]
  : never;

type Enum<T extends readonly string[], N extends boolean = false> = {
  readonly [K in TupleKeys<T> as Capitalize<T[K]>]: N extends true ? K : T[K]
};
```

```js
// Builder
function enumObject(names: [].<string>, numeric: boolean = false): type {
  return objectOf(names.map((name, i) =>
    ({ ...prop(`${name[0].toUpperCase()}${name.slice(1)}`, literal(numeric ? i : name)), readonly: true })));
}

enumObject([]) === type {};
enumObject(['macOS', 'Windows', 'Linux'])
  === type { readonly MacOS: 'macOS', readonly Windows: 'Windows', readonly Linux: 'Linux' };
enumObject(['macOS', 'Windows', 'Linux'], true)
  === type { readonly MacOS: 0, readonly Windows: 1, readonly Linux: 2 };
```

`TupleKeys` exists to produce `0 | 1 | 2` from a three-element tuple, and it does it by recursing down the tuple collecting `Tail['length']` at each step, because there is no way to ask for a tuple's indices. `map` hands you `i`.

## 545 · printf

```ts
// TypeScript, issue #2830 (+10)
type MapDict = {
  s: string
  d: number
}

type Format<T extends string> = T extends `${string}%${infer M}${infer R}`
  ? M extends keyof MapDict
    ? (x: MapDict[M]) => Format<R>
    : Format<R>
  : string
```

```js
// Builder
const mapDict = { s: string, d: float64 };

function format(f: string): type {
  const specs = [...f.matchAll(/%(.)/g)].map(m => m[1]).filter(c => c in mapDict);
  return specs.reduceRight((rest, c) => fn([mapDict[c]], rest), string);
}

format('') === string;
format('%s') === type (x: string) => string;
format('%d') === type (x: float64) => string;
format('%d%s') === type (x: float64) => (x: string) => string;
```

A curried function type, built one argument at a time. The recursion in `Format` is doing the same job as `reduceRight`, and both bottom out at `string`. Close, and pleasant on both sides.

## 553 · Deep object to unique

Brand every nested object with its path, so that structurally identical siblings become distinguishable.

```ts
// TypeScript, issue #581 (+6)
type Key = string | number | symbol;

declare const KEY: unique symbol;

type DeepObjectToUniq<
  O extends object,
  Parent = O,
  Path extends readonly Key[] = []
> = {
  [K in keyof O]: O[K] extends object
    ? DeepObjectToUniq<O[K], O, [...Path, K]>
    : O[K];
} & {
  readonly [KEY]?: readonly [Parent, Path];
};
```

```js
// Builder
const KEY = Symbol('uniq');

function deepObjectToUniq(O: type, parent: type = O, path: [].<string | symbol> = []): type {
  const node = reflect(O);
  const branded = node.properties.map(p =>
    reflect(p.type).kind === 'object'
      ? { ...p, type: deepObjectToUniq(p.type, O, [...path, p.name]) }
      : p);
  return objectOf([...branded,
    { ...prop(KEY, tupleOf([parent, tupleOf(path.map(literal))])), optional: true, readonly: true }]);
}

type O = { foo: { bar: 1 }, baz: { bar: 1 } };
deepObjectToUniq(O) === deepObjectToUniq(O);   // deterministic, so interned equality still works
deepObjectToUniq(O) !== O;                     // but the branded type is not the raw one
reflect(deepObjectToUniq(O)).properties.find(p => p.name === 'foo').type
  !== reflect(deepObjectToUniq(O)).properties.find(p => p.name === 'baz').type;  // same shape, distinct paths
```

The one challenge so far where a `symbol` key is the point rather than an edge case, and the node model has been carrying `name: string | symbol` on every property since the beginning for exactly this. The brand is a phantom property: optional, so no value ever has to provide it, and typed as the path, so two sibling objects with identical shapes get different types. Both sides do the same thing and the builder's is a `Symbol()` rather than a `unique symbol` declaration, which is the same idea with one fewer concept in it, since a symbol is already unique.

## 651 · Length of String 2

Like challenge 298, but it has to work on long strings.

```ts
// TypeScript, issue #3485 (+4)
type LengthOfString<S extends string,C extends number[] = []> = S extends `${infer F0}${infer F1}${infer F2}${infer F3}${infer F4}${infer F5}${infer F6}${infer F7}${infer F8}${infer F9}${infer R}`? LengthOfString<R,[...C,0,1,2,3,4,5,6,7,8,9]>
  : S extends `${infer F}${infer R}`?
    LengthOfString<R,[...C,0]>
    :C['length']
```

```js
// Builder
function lengthOfString(s: string): type {
  return literal([...s].length);
}

lengthOfString('') === type 0;
lengthOfString('1234567') === type 7;
```

Look at what the difficulty rating bought. Challenge 298 is the same function at medium; this is hard, and the only difference is that the input is longer. The answer is a *manual loop unroll*: match ten characters at a time, add ten to the counter, and keep the one-at-a-time version as the remainder loop. It is the exact optimization a compiler backend would do, written by hand, in a type, to buy a factor of ten against a recursion limit.

`[...s].length` does not have a length limit, and it iterates code points, which is the other thing challenge 298's note said.

## 730 · Union to Tuple

```ts
// TypeScript, issue #737 (+30)
/**
 * UnionToIntersection<{ foo: string } | { bar: string }> =
 *  { foo: string } & { bar: string }.
 */
type UnionToIntersection<U> = (
  U extends unknown ? (arg: U) => 0 : never
) extends (arg: infer I) => 0
  ? I
  : never;

/**
 * LastInUnion<1 | 2> = 2.
 */
type LastInUnion<U> = UnionToIntersection<
  U extends unknown ? (x: U) => 0 : never
> extends (x: infer L) => 0
  ? L
  : never;

/**
 * UnionToTuple<1 | 2> = [1, 2].
 */
type UnionToTuple<U, Last = LastInUnion<U>> = [U] extends [never]
  ? []
  : [...UnionToTuple<Exclude<U, Last>>, Last];
```

```js
// Builder
function unionToTuple(U: type): type {
  return tupleOf(arms(U));
}

reflect(unionToTuple(type 'a' | 'b')).elements.length === 2;
union(tupleElements(unionToTuple(type 'a' | 'b')).map(e => e.type)) === type 'a' | 'b';
unionToTuple(any) === type [any];
```

The most-cited construction in the whole repository, and worth reading slowly. `LastInUnion` builds on `UnionToIntersection` by wrapping each arm in a function *twice*: distribute to get `((x: 1) => 0) | ((x: 2) => 0)`, intersect that to get `((x: 1) => 0) & ((x: 2) => 0)`, and then infer `x` out of the intersection. An intersection of function types is an overload set, inference against an overload set picks the *last* signature, so you get `2`. Then `UnionToTuple` peels that arm off with `Exclude` and recurses.

The tuple's order therefore comes from overload resolution order, which comes from the order the intersection's members were written, which comes from the order the union's arms were declared. None of that is specified. The harness knows: every test asserts on `['length']` or on the union of the elements, and not one of them checks the order.

That is the honest reading of this challenge. It asks for a tuple from a union, which requires an ordering, and the language does not have one, so the answer extracts an ordering from a chain of three implementation details and the tests carefully do not look at what came out. `arms(U)` returns a list in the order [type programming](../typeprogramming.md) specifies for union canonicalization, and a builder that wants a different order writes `.sort()`.

## 847 · String Join

```ts
// TypeScript, issue #850 (+6)
type Tuple = readonly string[];

/**
 * Tail<['1', '2', '3']> = ['2', '3'].
 */
type Tail<T extends Tuple> = T extends readonly [infer Head, ...infer Rest] ? Rest : [];

/**
 * Join<['1', '2'], " - "> = '1 - 2'.
 * Join<['1'], " - "> = '1'.
 * Join<[], 'x'> = ''.
 */
type Join<T extends Tuple, Separator extends string> = T extends readonly []
    ? ''
    : T extends readonly [infer Head]
    ? Head
    : `${T[0]}${Separator}${Join<Tail<T>, Separator>}`;

declare function join<D extends string>(delimiter: D): <P extends Tuple>(...parts: P) => Join<P, D>;
```

```js
// Builder
declare function join<D extends string>(delimiter: D):
  <P extends [].<string>>(...parts: P) => literal(tupleElements(P).map(e => literalValues(e.type)[0]).join(delimiter));

Reflect.typeOf(join('-')('a', 'b', 'c')) === type 'a-b-c';
Reflect.typeOf(join('-')()) === type '';
Reflect.typeOf(join('')('a', 'b', 'c')) === type 'abc';
```

Challenge 5310 again at the hard tier, this time as a real curried function rather than a bare alias. The interesting part is the signature: `<P extends Tuple>(...parts: P)` captures the argument tuple, and the return type is computed from it. The builder does the same thing, with `Array.prototype.join` in the return position.

## 956 · DeepPick

Pick a set of dotted paths, preserving structure.

```ts
// TypeScript, issue #1019 (+6)
type TypeGet<T, Paths> = Paths extends `${infer A}.${infer B}`
  ? A extends keyof T ? { [K in A]: TypeGet<T[A], B> } : never
  : Paths extends keyof T ? { [K in Paths]: T[Paths] } : never

type UnionToIntercetion<U> = (U extends any ? (arg: U) => any : never) extends ((arg: infer I) => any) ? I : never

type DeepPick<T, PathUnion extends string> =
  UnionToIntercetion<PathUnion extends infer Keys ? TypeGet<T, Keys> : never>
```

```js
// Builder
function pickPath(T: type, path: string): type {
  const dot = path.indexOf('.');
  const head = dot === -1 ? path : path.slice(0, dot);
  const p = reflect(T).properties.find(x => x.name === head);
  if (p === undefined) return never;
  return objectOf([prop(head, dot === -1 ? p.type : pickPath(p.type, path.slice(dot + 1)))]);
}

function deepPick(T: type, paths: [].<string>): type {
  return Reflect.makeType({ kind: 'intersection', members: paths.map(path => pickPath(T, path)) });
}

type O = { user: { name: 'ann', age: 30 }, active: true };
deepPick(O, ['active']) === type { active: true };
deepPick(O, ['user.name', 'active']) === type { user: { name: 'ann' } } & { active: true };
```

Three of the batch's ten challenges reach for `UnionToIntersection`, and this is the clearest look at why: the natural way to combine several picked paths is to merge them, the type language has no merge, and an intersection is the closest available thing, so you distribute over the path union, build one object per path, and intersect the lot. The contravariance trick from challenge 55 is load-bearing infrastructure here, not a curiosity. The builder's `members` is a list, and it is a list because that is what an intersection node holds.

## 1290 · Pinia

Challenge 6 grown up: a store with state, getters that see readonly state, and actions that see everything.

```ts
// TypeScript, issue #27536 (+14)
type AnyObject = Record<string, unknown>
type BaseGetters = Record<string, Function> ;
type BaseActions = Record<string, (...args:any[]) => any>;

type ComputedGetters<G extends BaseGetters> = {
  readonly [K in keyof G]: G[K] extends (...args:any[]) => any ? ReturnType<G[K]> : never;
};

interface StoreOptions<State extends AnyObject, Getters extends BaseGetters, Actions extends BaseActions> {
  id: string;
  state: () => State,
  getters: Getters & ThisType<ComputedGetters<Getters> & Readonly<State>>,
  actions: Actions & ThisType<Actions & State & ComputedGetters<Getters>>
}

type Store<
  State extends AnyObject,
  Getters extends BaseGetters,
  Actions extends BaseActions
>= {
  init(): void;
  reset(): true;
} & Actions & State & ComputedGetters<Getters>;

declare function defineStore<
  State extends AnyObject,
  Getters extends BaseGetters,
  Actions extends BaseActions
>(store: StoreOptions<State, Getters, Actions>): Store<State, Getters, Actions>;
```

```js
// Builder
const all = (...members: [].<type>): type => Reflect.makeType({ kind: 'intersection', members });

function storeOptions(S: type, G: type, A: type): type {
  return objectOf([
    prop('id', string),
    prop('state', fn([], S)),
    prop('getters', withThisOnMethods(G, all(computedResults(G), readonly(S)))),   // helpers from challenge 6
    prop('actions', withThisOnMethods(A, all(A, S, readonly(computedResults(G))))),
  ]);
}

function store(S: type, G: type, A: type): type {
  return all(A, S, computedResults(G));
}

declare function defineStore<S, G, A>(options: storeOptions(S, G, A)): store(S, G, A);
```

The third `ThisType` challenge and the one where the pattern has clearly become a library. Both sides say the same four things: getters see their own results plus readonly state, actions see themselves plus state plus readonly getter results, the store is all of it merged, and `ComputedGetters` is `ReturnType` mapped over the getters. `computedResults` and `withThisOnMethods` were written back at challenge 6 and are reused here unchanged, which is the whole argument in one line: the builder's helpers compose because they are functions.

## 1383 · Camelize

Snake-case keys to camel case, recursively.

```ts
// TypeScript, issue #1403 (+8)
type SnakeToCamel<S extends string, Cap extends boolean = false> =
  S extends `${infer Head}_${infer Tail}`
    ? `${
        Cap extends true
          ? Capitalize<Head>
          : Head
      }${SnakeToCamel<Tail, true>}`
    : Cap extends true
      ? Capitalize<S>
      : S;

type TerminalTypes =
  | number
  | boolean
  | symbol
  | bigint
  | Function

type Camelize<T> = {
  default: {
    [K in keyof T as Camelize<K>]: Camelize<T[K]>
  },
  array: T extends [infer Head, ...(infer Tail)]
    ? [Camelize<Head>, ...Camelize<Tail>]
    : [],
  string: SnakeToCamel<T & string>
  terminal: T,
}[
  T extends any[]
  ? 'array'
  : T extends TerminalTypes
  ? 'terminal'
  : T extends string
  ? 'string'
  /** default */
  : 'default'
];
```

```js
// Builder
const snakeToCamel = (s: string): string => s.replace(/_(\p{L})/gu, (_, c) => c.toUpperCase());

function camelize(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'object': return objectOf(node.properties.map(p => ({
      ...p,
      name: typeof p.name === 'string' ? snakeToCamel(p.name) : p.name,
      type: camelize(p.type),
    })));
    case 'tuple': return tupleOf(node.elements.map(e => camelize(e.type)));
    default: return T;
  }
}

camelize(type {
  some_prop: string,
  prop: { another_prop: string },
  array: [{ snake_case: string }, { another_element: { yet_another_prop: string } }],
}) === type {
  someProp: string,
  prop: { anotherProp: string },
  array: [{ snakeCase: string }, { anotherElement: { yetAnotherProp: string } }],
};
```

The dispatch is the thing to look at: an object literal with four entries, indexed by a conditional that computes which entry to take. That is a `switch`, spelled as a dictionary lookup, because the type language has no `switch` and a mapped type is the only dictionary it has. Fourth sighting of the mapped-type-as-lookup idiom and the first where it is doing genuine control flow rather than data. The builder's `switch (node.kind)` is the same shape, and `kind` is a string that already exists.

`snakeToCamel` is challenge 114's `camelCase` without the `.toLowerCase()`, which is exactly the difference between the two challenges.

## 2059 · Drop String

Drop every character that appears in `R`.

```ts
// TypeScript, issue #3308 (+7)
type DropOne<S extends string,R extends string> = S extends `${infer A}${R}${infer B}`?
  DropOne<`${A}${B}`,R>
  :S

type DropString<S extends string, R extends string> = R extends `${infer F}${infer L}`?
  DropString<DropOne<S,F>,L>
  :S
```

```js
// Builder
function dropString(s: string, drop: string): type {
  return literal([...s].filter(c => !drop.includes(c)).join(''));
}

dropString('butter fly!', '') === type 'butter fly!';
dropString('butter fly!', ' ') === type 'butterfly!';
dropString('butter fly!', 'but') === type 'er fly!';
dropString(' b u t t e r f l y ! ', 'tub') === type '     e r f l y ! ';
```

Two nested loops, because there is one loop construct and it needs two: `DropString` walks the characters of `R`, and `DropOne` walks `S` removing one of them at a time. `filter` runs one loop and asks `includes` per character. The `'but'` and `'tub'` cases are the same answer in both languages, and the harness includes both to say that the drop set is a set.

## 2822 · Split

```ts
// TypeScript, issue #30920 (+6). The top-voted answer (#3721, +7) has no default separator;
// see the note, which is about the template rather than the answer.
type Split<T extends string, SEP extends string = never> = T extends `${infer P}${SEP}${infer L}`
 ? [P , ...Split<L, SEP>]
 : T extends `${infer _}`
   ? T extends SEP ? [] : [T]
   : string []
```

```js
// Builder
function split(S: type, separator: string = undefined): type {
  const node = reflect(S);
  if (node.kind !== 'literal') return [].<string>;      // Split<string, 'whatever'> is string[]
  const parts = separator === undefined ? [node.value] : node.value.split(separator);
  return tupleOf(parts.map(literal));
}

split(type 'Hi! How are you?') === type ['Hi! How are you?'];
split(type 'Hi! How are you?', ' ') === type ['Hi!', 'How', 'are', 'you?'];
split(type 'The sine in cosine', 'in') === type ['The s', 'e ', ' cos', 'e'];
split(type '', '') === type [];
split(type '', 'z') === type [''];
split(string, 'whatever') === [].<string>;
```

`String.prototype.split` is the challenge, including both empty-string cases: `''.split('')` is `[]` and `''.split('z')` is `['']`, which are exactly the two the harness asks about. Twenty-five years of that method being specified is why nobody has to think about it.

The interesting failure here is not the answer's. The challenge's own template declares

```ts
type Split<S extends string, SEP extends string> = any
```

with two required parameters, and its own test file calls `Split<'Hi! How are you?'>` and `Split<''>` with one. The template does not typecheck against the tests it ships with, and every passing answer has to add a default that the template does not have. The top-voted answer copied the template's signature.

## 2828 · ClassPublicKeys

```ts
// TypeScript, issue #2843 (+19)
// :D
type ClassPublicKeys<A> = keyof A
```

```js
// Builder
function classPublicKeys(A: type): type {
  return keysOf(A);
}

class A {
  str = 'naive';
  #bool = true;
  getNum() { return Math.random(); }
}
classPublicKeys(A) === type 'str' | 'getNum';
```

A draw, and the funniest answer in the repository: nineteen votes and a smiley, because `keyof` already excludes private members and there is nothing to do. Worth including precisely because it is a challenge where the language's built-in is right.

One honest note. The harness's class has a `protected num` and expects it excluded, and JavaScript has no `protected`: `#bool` is genuinely private and never appears in reflection, but a field that TypeScript would mark `protected` is just a field, and `keysOf` would return it. The challenge tests an access modifier that exists only in TypeScript's static layer, so the builder above uses `#bool` and answers the same question honestly rather than pretending the third case exists.

## 2857 · IsRequiredKey

```ts
// TypeScript, issue #3732 (+9)
type IsRequiredKey<
  Type,
  Keys extends keyof Type
> = Pick<Type, Keys> extends Required<Pick<Type, Keys>>
  ? true
  : false
```

```js
// Builder
function isRequiredKey(T: type, keys: type): type {
  const names = new Set(arms(keys).map(k => literalValues(k)[0]));
  return reflect(T).properties.filter(p => names.has(p.name)).every(p => !p.optional)
    ? type true : type false;
}

isRequiredKey(type { a: uint32, b?: string }, type 'a') === type true;
isRequiredKey(type { a: uint32, b?: string }, type 'b') === type false;
isRequiredKey(type { a: uint32, b?: string }, type 'b' | 'a') === type false;
isRequiredKey(type { a: undefined, b: undefined }, type 'b' | 'a') === type true;
```

The fifth challenge asking whether a property is optional and the third distinct workaround: pick the keys, require the picked object, and ask whether the original still fits. `.every(p => !p.optional)` is what "are all of these required" says.

## 2949 · ObjectFromEntries

```ts
// TypeScript, issue #3382 (+15)
type ObjectFromEntries<T extends [string, any]> = {
  [K in T[0]]: T extends [ K, any ] ? T[1] : never
}
```

```js
// Builder
function objectFromEntries(T: type): type {
  return objectOf(arms(T).map(entry => {
    const [key, value] = tupleElements(entry).map(e => e.type);
    return prop(literalValues(key)[0], value);
  }));
}

type ModelEntries = ['name', string] | ['age', float64] | ['locations', [].<string> | null];
objectFromEntries(ModelEntries) === type { name: string, age: float64, locations: [].<string> | null };
```

The mirror of ObjectEntries at challenge 2946, and much shorter than it, because going this direction does not need the `undefined` handling that one did. `T extends [K, any] ? T[1] : never` inside the mapped type is a filter over the union: for each key, find the arm whose first element is that key and take its second. It reads as a lookup and it is a scan. `Object.fromEntries` is right there in the name of the challenge, and the builder is the loop that method already is.

## 4037 · IsPalindrome

```ts
// TypeScript, issue #4093 (+14)
type IsPalindrome<T extends string | number, K = `${T}`> =
  K extends `${infer L}${infer R}` ?
  R extends '' ? true :
  K extends `${L}${infer S}${L}` ? IsPalindrome<S> : false
  : true
```

```js
// Builder
function isPalindrome(v: string | float64): type {
  const s = `${v}`;
  return s === [...s].reverse().join('') ? type true : type false;
}

isPalindrome('abc') === type false;
isPalindrome('abcba') === type true;
isPalindrome(121) === type true;
isPalindrome(19260817) === type false;
```

A nice one. `` K extends `${L}${infer S}${L}` `` uses the *same* binding twice in one pattern, so the match succeeds only when the first and last characters are equal, and `S` comes back as the middle. That is a backreference, and pattern matching against a template literal type is the one place the type language has them. The builder reverses the string and compares.

## 5181 · Mutable Keys

```ts
// TypeScript, issue #5221 (+9)
type MyEqual<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false

type MutableKeys<T> = keyof {
  [Key in keyof T as MyEqual<Pick<T, Key>, Readonly<Pick<T, Key>>> extends true ? never : Key ]: any
}
```

```js
// Builder
function mutableKeys(T: type): type {
  return union(reflect(T).properties.filter(p => !p.readonly).map(p => literal(p.name)));
}

mutableKeys(type { a: uint32, readonly b: string }) === type 'a';
mutableKeys(type { a: undefined, readonly b?: undefined, c: string, d: null }) === type 'a' | 'c' | 'd';
mutableKeys(type {}) === never;
```

The `readonly` version of challenges 89 and 90, and it needs the identity incantation to do it: pick one key, make the picked object readonly, and ask whether that changed anything. If `Readonly<Pick<T, K>>` is *identical* to `Pick<T, K>` then the key was already readonly. Eighth challenge in this document to reach for `Equal`, and the reason is the same as it was for `optional`: the flag is right there on the property and there is no way to read it.

`TypePropertyReflection` has `readonly`. It is a boolean.

## 5423 · Intersection

Intersect a list of tuples and unions.

```ts
// TypeScript, issue #11756 (+12)
type Intersection<T>
  = {[I in keyof T]: T[I] extends any[] ? T[I] : [T[I]]} extends {fill: ($: infer I) => any}[]
  ? I : never;
```

```js
// Builder
function intersection(T: type): type {
  const sets = tupleElements(T).map(e => {
    const node = reflect(e.type);
    return new Set(node.kind === 'tuple' ? node.elements.map(x => x.type) : arms(e.type));
  });
  if (sets.length === 0) return never;
  return union([...sets[0]].filter(t => sets.every(set => set.has(t))));
}

intersection(type [[1, 2], [2, 3], [2, 2]]) === type 2;
intersection(type [[1, 2, 3], [2, 3, 4], [2, 2, 3]]) === type 2 | 3;
intersection(type [[1, 2], [3, 4], [5, 6]]) === never;
intersection(type [[1, 2, 3], 2 | 3 | 4, 2 | 3]) === type 2 | 3;
intersection(type [[1, 2, 3], 2, 3]) === never;
```

Two lines, and the second one is the most audacious thing in the document. It normalizes every element to an array, then matches the whole tuple of arrays against `{fill: ($: infer I) => any}[]`, and reads out `I`.

Why that works: `Array.prototype.fill` takes a parameter of the array's element type. Matching a tuple of arrays against an array of `{fill: ($: infer I) => any}` infers `$` from every element's `fill` at once, and a parameter is a contravariant position, so the inferences are *intersected*. `(1 | 2) & (2 | 3) & 2` reduces to `2`, and that is the answer. It is challenge 55's contravariance trick again, except the function whose parameter is being abused is not one the author wrote. It is a method on the standard library's array interface, chosen because its signature happens to mention the element type in the right position.

Nothing about `fill` has anything to do with intersecting sets. `sets.every(set => set.has(t))` is the same computation, and a reader who has never seen it before will still know what it does.

## 6141 · Binary to Decimal

```ts
// TypeScript, issue #6349 (+21)
type BinaryToDecimal<
  S extends string,
  R extends any[] = []
> =
  S extends `${infer F}${infer L}`?
    F extends '0'? BinaryToDecimal<L,[...R,...R]>:BinaryToDecimal<L,[...R,...R,1]>
    :R['length']
```

```js
// Builder
function binaryToDecimal(s: string): type {
  return literal(Number.parseInt(s, 2));
}

binaryToDecimal('0011') === type 3;
binaryToDecimal('11111111') === type 255;
binaryToDecimal('10101010') === type 170;
```

Horner's method, in tuples. `[...R, ...R]` doubles the accumulator, `[...R, ...R, 1]` doubles and adds one, and the digits are consumed left to right, which is the standard way to read a binary numeral and is genuinely the right algorithm. It is also `parseInt(s, 2)`, and the accumulator for `'11111111'` is a 255-element tuple.

## 7258 · Object Key Paths

Every path into an object, including array-index syntax.

```ts
// TypeScript, issue #7939 (+9)
type GenNode<K extends string | number,IsRoot extends boolean> = IsRoot extends true? `${K}`: `.${K}` | (K extends number? `[${K}]` | `.[${K}]`:never)

type ObjectKeyPaths<
  T extends object,
  IsRoot extends boolean = true,
  K extends keyof T = keyof T
> =
K extends string | number ?
  GenNode<K,IsRoot> | (T[K] extends object? `${GenNode<K,IsRoot>}${ObjectKeyPaths<T[K],false>}`:never)
  : never
```

```js
// Builder
function objectKeyPaths(T: type, isRoot: boolean = true): type {
  const node = reflect(T);
  const entries = node.kind === 'tuple'
    ? node.elements.map((e, i) => [i, e.type])
    : node.properties.filter(p => typeof p.name === 'string').map(p => [p.name, p.type]);

  return union(entries.flatMap(([key, type]) => {
    const heads = isRoot ? [`${key}`]
      : typeof key === 'number' ? [`.${key}`, `[${key}]`, `.[${key}]`]
      : [`.${key}`];
    const nested = reflect(type).kind === 'object' || reflect(type).kind === 'tuple'
      ? arms(objectKeyPaths(type, false)).map(a => literalValues(a)[0])
      : [];
    return heads.flatMap(h => [literal(h), ...nested.map(n => literal(`${h}${n}`))]);
  }));
}

type T = { a: { b: 1 }, list: [{ c: 2 }] };
objectKeyPaths(T) === type 'a' | 'a.b' | 'list'
  | 'list.0' | 'list[0]' | 'list.[0]'
  | 'list.0.c' | 'list[0].c' | 'list.[0].c';
```

`GenNode` produces three spellings for a numeric key, because `a[0]`, `a.0`, and `a.[0]` are all things people write. Both sides enumerate the same cross product; the difference is that the builder's `heads` is an array of strings and TypeScript's is a union that has to be built with a conditional inside a template literal so that it distributes. The recursion is the same shape in both.

## 8804 · Two Sum

Do any two elements sum to the target?

```ts
// TypeScript, issue #9119 (+4)
// 1. get permutations from an array
type Permutation<T> = T extends [infer One, ...infer Rest]
  ? [One, ...Permutation<Rest>] | [...Permutation<Rest>, One]
  : []

// 3. get first two elems from an array
type GetFirstTwo<T> = T extends T ? T extends [infer One, infer Two, ...infer Rest] ? [One, Two] : never : never

// 4. combine GetFirstTwo and Permutations then we get
//    all posibles of number addition in an array
type GetAllPosibles<T> = GetFirstTwo<Permutation<T>>

// 6. get result of add two number
type CreateTupple<S extends number, Res extends 1[] = []> = Res['length'] extends S ? Res : CreateTupple<S, [...Res, 1]>
type Add<A extends number, B extends number> = [...CreateTupple<A>, ...CreateTupple<B>]['length']

// 8. GetAllPosibles as a union R,
//    expand union R then find if Add<One, Two> extends target number,
//    then we got result like: true | never | never | never
type FindPosible<T, U, R = GetAllPosibles<T>> = R extends R
  ? R extends [infer One, infer Two]
    ? Add<One & number, Two & number> extends U ? true : never
    : never
  : never

// 9. if result contains not never, then it contains a true
type TwoSum<T extends number[], U extends number> = [FindPosible<T, U>] extends [never] ? false : FindPosible<T, U>
```

```js
// Builder
function twoSum(T: type, target: float64): type {
  const values = tupleElements(T).map(e => literalValues(e.type)[0]);
  return values.some((a, i) => values.slice(i + 1).some(b => a + b === target))
    ? type true : type false;
}

twoSum(type [3, 3], 6) === type true;
twoSum(type [3, 2, 4], 6) === type true;
twoSum(type [2, 7, 11, 15], 15) === type false;
twoSum(type [1, 2, 3], 0) === type false;
```

The published answer generates *every permutation of the whole array*, takes the first two elements of each, and checks those pairs, which is a factorial amount of work to enumerate the quadratically many pairs. It is not incompetence; it is that the two loop constructs available are "peel a tuple" and "distribute a union", and permuting is how you turn the first into the second. Note `Add`: build a tuple of length `A`, build a tuple of length `B`, concatenate, take the `length`. That is addition, and this document has now seen the same three lines in nine different challenges.

The nested `some` is the two loops, written as two loops.

## 9155 · ValidDate

Is `MMDD` a real date?

```ts
// TypeScript, issue #9174 (+15)
type Num = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9

type MM = `0${Num}` | `1${0 | 1 | 2}`

type AllDate =
  | `${MM}${`${0}${Num}` | `${1}${0 | Num}` | `2${0 | Exclude<Num, 9>}`}`
  | `${Exclude<MM, '02'>}${29 | 30}`
  | `${Exclude<MM, '02' | '04' | '06' | '09' | '11'>}${31}`

type ValidDate<T extends string> = T extends AllDate ? true : false
```

```js
// Builder
const monthLengths = [31, 29, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];   // February without a leap year

function validDate(s: string): type {
  if (!/^\d{4}$/.test(s)) return type false;
  const [month, day] = [Number(s.slice(0, 2)), Number(s.slice(2))];
  return month >= 1 && month <= 12 && day >= 1 && day <= monthLengths[month - 1] - (month === 2 ? 1 : 0)
    ? type true : type false;
}

validDate('0102') === type true;
validDate('1231') === type true;
validDate('0229') === type false;      // no leap years here
validDate('0100') === type false;
validDate('0132') === type false;
```

This one is genuinely clever and I want to be fair to it: rather than compute, it *enumerates every valid date as a type*. `MM` is the twelve months as a union of literals; the first arm of `AllDate` is days 01 through 28 for any month; the second adds 29 and 30 for every month except February; the third adds 31 for the seven long months, spelled as `Exclude<MM, '02' | '04' | '06' | '09' | '11'>`. Then the test is a single `extends` against a union of three hundred and thirty-seven strings, which the compiler checks instantly.

That is a real technique and in a language with no arithmetic it is the right call. It is also why the challenge quietly has no leap years: `'0229'` is `false`, not because February 29th does not exist but because the year is not in the input and the union has to commit. The builder has the same constraint and says so in a comment on the table.

## 9160 · Assign

`Object.assign` for types.

```ts
// TypeScript, issue #36059 (+5)
type Assign<T extends Record<string, unknown>, U extends unknown[]> =
  U extends [infer F, ...infer R extends Record<string, unknown>[]]
    ? Assign<Omit<T, keyof F> & F, R>
    : Omit<T, never>
```

```js
// Builder
function assign(T: type, sources: [].<type>): type {
  const byName = new Map(reflect(T).properties.map(p => [p.name, p]));
  for (const source of sources)
    for (const p of reflect(source).properties) byName.set(p.name, p);
  return objectOf([...byName.values()]);
}

assign(type {}, [type { a: 'a' }]) === type { a: 'a' };
assign(type { a: 'a', b: 'b' }, [type { a: 1 }, type { c: 'c' }]) === type { a: 1, b: 'b', c: 'c' };
```

`Omit<T, keyof F> & F` is last-wins assignment: remove the keys the new object defines, then intersect it in. The trailing `Omit<T, never>` is the flattening no-op, fifth sighting, and here it is load-bearing rather than cosmetic, because without it the result is a chain of intersections and `Equal` would say no. `Map.set` overwrites, which is what assignment is.

## 9384 · Maximum

```ts
// TypeScript, issue #21928 (+8)
// " 1|20|200|150 extends 20 ? never : U " ==>> " 1|200|150 "
type Maximum<T extends any[], U = T[number], N extends any[] = []>
    = T extends [] ? never
    : Equal<U, N['length']> extends true ? U
    : Maximum<T, (U extends N['length'] ? never : U), [...N, unknown]>
```

```js
// Builder
function maximum(T: type): type {
  const values = tupleElements(T).map(e => literalValues(e.type)[0]);
  return values.length === 0 ? never : literal(Math.max(...values));
}

maximum(type []) === never;
maximum(type [0, 2, 1]) === type 2;
maximum(type [1, 20, 200, 150]) === type 200;
```

Read the algorithm. It counts `N` up from zero, one element at a time, and at each step removes that number from the union; the last number standing when the union has one member left is the maximum. To find the largest of `[1, 20, 200, 150]` it counts to two hundred. It needs `Equal` to know when the union is down to a single member, which is the ninth appearance of the incantation.

`Math.max` exists. This is the arithmetic column's fifteenth entry and its plainest: the challenge is not hard, the language cannot compare two numbers.

## 9775 · Capitalize Nest Object Keys

```ts
// TypeScript, issue #16801 (+7)
type CapitalizeNestObjectKeys<T> = T extends readonly any[]
  ? {
      [K in keyof T]: CapitalizeNestObjectKeys<T[K]>;
    }
  : T extends Record<keyof any, any>
  ? {
      [K in keyof T as Capitalize<K & string>]: CapitalizeNestObjectKeys<T[K]>;
    }
  : T;
```

```js
// Builder
function capitalizeNestObjectKeys(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'object': return objectOf(node.properties.map(p => ({
      ...p,
      name: typeof p.name === 'string' ? `${p.name[0].toUpperCase()}${p.name.slice(1)}` : p.name,
      type: capitalizeNestObjectKeys(p.type),
    })));
    case 'tuple': return tupleOf(node.elements.map(e => capitalizeNestObjectKeys(e.type)));
    default: return T;
  }
}

type T = { foo: 1, bar: { baz: [{ deep: 2 }] } };
capitalizeNestObjectKeys(T) === type { Foo: 1, Bar: { Baz: [{ Deep: 2 }] } };
```

Challenge 1383 with `Capitalize` instead of `SnakeToCamel`, and the same two-branch recursion. `Capitalize<K & string>` needs the `& string` because `keyof T` can include symbols and `Capitalize` cannot take one; the builder's `typeof p.name === 'string'` is that guard, said in the affirmative and leaving symbol keys alone rather than intersecting them away.

## 13580 · Replace Union

```ts
// TypeScript, issue #28560 (+2). The top-voted answer (#20711, +2) returns `never` when a
// replacement pair does not match; see the note.
type UnionReplace<T, U extends [any, any][]> = U extends [
  infer F extends [any, any],
  ...infer R extends [any, any][]
]
  ? UnionReplace<T extends F[0] ? F[1] : T, R>
  : T;
```

```js
// Builder
function unionReplace(T: type, pairs: [].<[type, type]>): type {
  return union(arms(T).map(arm => pairs.find(([from]) => from === arm)?.[1] ?? arm));
}

unionReplace(type float64 | string, [[string, type null]]) === type float64 | null;
unionReplace(type float64 | string, [[string, type null], [Date, Function]]) === type float64 | null;
unionReplace(type Function | Date | object, [[Date, string], [Function, type undefined]])
  === type undefined | string | object;
```

`T extends F[0] ? F[1] : T` distributes over the union and swaps the matching arm, and the pairs are consumed one at a time by the outer recursion. The top-voted answer instead pattern-matches `F extends [T, infer Replace]` and answers `never` for any pair whose source is not in the union, which the second test case catches: `[Date, Function]` is a pair that simply does not apply.

`find` returns `undefined` when nothing matches, and `?? arm` is "leave it alone".

## 14080 · FizzBuzz

```ts
// TypeScript, issue #24661 (+5)
type IsDivByThree<T extends unknown[]> = T extends [...infer Start, unknown, unknown, unknown]
? Start extends []
  ? 'Fizz'
  : IsDivByThree<Start>
: false

type IsDivByFive<T extends unknown[]> = T extends [...infer Start, unknown, unknown, unknown, unknown, unknown]
? Start extends []
  ? 'Buzz'
  : IsDivByFive<Start>
: false

type FizzBuzz<N extends number, Acc extends unknown[] = []> = N extends Acc['length']
? Acc
: FizzBuzz<N, [...Acc,
    `${
        ( IsDivByThree<[...Acc, '']> | IsDivByFive<[...Acc, '']> ) extends false
          ? [...Acc, '']['length'] extends number ? [...Acc, '']['length'] : never
          : ''
      }${
        IsDivByThree<[...Acc, '']> extends string ? 'Fizz' : ''
      }${
        IsDivByFive<[...Acc, '']> extends string ? 'Buzz' : ''
      }`
  ]>
```

```js
// Builder
function fizzBuzz(n: uint32): type {
  return tupleOf(Array.from({ length: n }, (_, i) => {
    const v = i + 1;
    const word = `${v % 3 === 0 ? 'Fizz' : ''}${v % 5 === 0 ? 'Buzz' : ''}`;
    return literal(word === '' ? `${v}` : word);
  }));
}

fizzBuzz(1) === type ['1'];
fizzBuzz(5) === type ['1', '2', 'Fizz', '4', 'Buzz'];
```

FizzBuzz is the canonical example of a problem that is trivial in every language. Here, divisibility by three is a recursive type that removes three elements from the end of a tuple until it either empties or does not, and divisibility by five is the same type written again with five `unknown`s, because you cannot pass the three. Then `IsDivByThree<[...Acc, '']>` is evaluated three separate times in the body, because there is nowhere to put the result.

`v % 3 === 0`. It is worth sitting with the fact that this challenge is rated hard.

## 14188 · Run-length encoding

```ts
// TypeScript, issue #14254 (+3)
namespace RLE {
  type EncodeHelper<S extends string, P extends string = '', L extends 1[] = []> = S extends `${infer F}${infer R}`
    ? F extends P
      ? EncodeHelper<R, P, [1, ...L]>
      : P extends ''
        ? EncodeHelper<R, F, [1]>
        : L['length'] extends 1
          ? `${P}${EncodeHelper<R, F, [1]>}`
          : `${L['length']}${P}${EncodeHelper<R, F, [1]>}`
    : P extends ''
      ? ''
      : L['length'] extends 1
        ? P
        : `${L['length']}${P}`
  export type Encode<S extends string> = EncodeHelper<S>

  type DecodeHelper<S extends string, L extends string = ''> = S extends `${infer F}${infer R}`
    ? F extends `${number}`
      ? DecodeHelper<R, `${L}${F}`>
      : `${Repeat<F, L extends '' ? '1' : L>}${DecodeHelper<R, ''>}`
    : ''
  type Repeat<S extends string, L extends string, C extends 1[] = []> = `${C['length']}` extends L ? '' : `${S}${Repeat<S, L, [1, ...C]>}`
  export type Decode<S extends string> = DecodeHelper<S>
}
```

```js
// Builder
const RLE = {
  encode(s: string): type {
    return literal([...s.matchAll(/(.)\1*/gsu)]
      .map(([run, ch]) => `${run.length === 1 ? '' : run.length}${ch}`).join(''));
  },
  decode(s: string): type {
    return literal([...s.matchAll(/(\d*)(\D)/gu)]
      .map(([, n, ch]) => ch.repeat(n === '' ? 1 : Number(n))).join(''));
  },
};

RLE.encode('AAABCCXXXXXXY') === type '3AB2C6XY';
RLE.decode('3AB2C6XY') === type 'AAABCCXXXXXXY';
```

Both directions, and the encoder is the interesting half. `L extends 1[]` is the run-length counter, `L['length'] extends 1` is the "omit the count when it is one" rule, and `Repeat` on the decode side counts a second tuple up to the parsed number because there is no `repeat`.

The builder's encoder is a regex with a backreference: `(.)\1*` matches a character followed by more of itself, which *is* a run. `String.prototype.repeat` is the decoder. Two lines against a namespace, and the honest note is that the regex is doing real work here, not just standing in for a loop.

## 15260 · Tree path array

Every path into a nested object, as a tuple.

```ts
// TypeScript, issue #18292 (+5)
type Path<T> = T extends Record<PropertyKey, unknown>
  ? {
      [P in keyof T]: [P, ...Path<T[P]>] | [P];
    }[keyof T]
  : never;
```

```js
// Builder
function paths(T: type): type {
  const node = reflect(T);
  if (node.kind !== 'object') return never;
  return union(node.properties.flatMap(p => {
    const key = literal(p.name);
    return [tupleOf([key]), ...arms(paths(p.type)).map(a =>
      tupleOf([key, ...tupleElements(a).map(e => e.type)]))];
  }));
}

type T = { a: { b: 1, c: 2 }, d: 3 };
paths(T) === type ['a'] | ['a', 'b'] | ['a', 'c'] | ['d'];
```

Six lines against six lines, and a good example of a leaf case falling out rather than being written: `Path<string>` is `never`, so `[P, ...Path<string>]` is `never` too and only `[P]` survives that union arm. The builder gets the same thing from `arms(never)` being the empty list, so the `flatMap` contributes only `[key]`. The `{...}[keyof T]` is challenge 90's map-then-index idiom again: build a mapped type purely to index it away, because collapsing keys to a union has no other spelling.

## 19458 · SnakeCase

```ts
// TypeScript, issue #20928 (+8)
type SnakeCase<T> = T extends `${infer A}${infer R}`
  ? Uppercase<A> extends A
    ? `_${Lowercase<A>}${SnakeCase<R>}`
    : `${A}${SnakeCase<R>}`
  : '';
```

```js
// Builder
function snakeCase(T: type): type {
  return union(arms(T).map(a =>
    literal(literalValues(a)[0].replace(/\p{Lu}/gu, c => `_${c.toLowerCase()}`))));
}

snakeCase(type 'hello') === type 'hello';
snakeCase(type 'userName') === type 'user_name';
snakeCase(type 'getElementById') === type 'get_element_by_id';
snakeCase(type 'getElementById' | 'getElementByClassNames')
  === type 'get_element_by_id' | 'get_element_by_class_names';
```

`Uppercase<A> extends A` is the letter test from challenge 35252 for the fourth time, now asking "is this an uppercase letter" rather than "is this a letter". `\p{Lu}` is that character class by name. The union case works on the TypeScript side because the conditional distributes, and on the builder side because `arms` returned a list and `map` is a loop.

## 25747 · IsNegativeNumber

```ts
// TypeScript, issue #27809 (+5)
type IsNegativeNumber<T extends number, U extends T = T> =
  number extends T
    ? never
    : T extends U
      ? [U] extends [T]
        ? `${T}` extends `-${infer _}`
          ? true
          : false
        : never
      : never
```

```js
// Builder
function isNegativeNumber(T: type): type {
  const node = reflect(T);
  if (node.kind !== 'literal') return never;   // float64 and unions are not one literal
  return node.value < 0 ? type true : type false;
}

isNegativeNumber(type 0) === type false;
isNegativeNumber(float64) === never;
isNegativeNumber(type -1 | -2) === never;
isNegativeNumber(type -1.9) === type true;
isNegativeNumber(type -100_000_000) === type true;
```

The actual test is the last line: stringify the number and look for a minus sign. Everything above it is scaffolding to reject the two non-answers: `number extends T` catches the wide type, and `T extends U ? [U] extends [T] : never` is the is-this-one-literal check from challenge 30970 inlined, spending a distribution and a tuple-wrap to find out whether the union has one member. `node.kind !== 'literal'` is both of those, and `< 0` beats reading the sign out of a string.

## 28143 · OptionalUndefined

Make properties whose type includes `undefined` optional.

```ts
// TypeScript, issue #28200 (+8)
type Merge<T> = {
  [K in keyof T]:T[K]
}

type OptionalUndefined<
  T,
  Props extends keyof T = keyof T,
  OptionsProps extends keyof T =
    Props extends keyof T?
      undefined extends T[Props]?
        Props:never
      :never
> =
  Merge<{
    [K in OptionsProps]?:T[K]
  } & {
    [K in Exclude<keyof T,OptionsProps>    ]:T[K]
  }>
```

```js
// Builder
function optionalUndefined(T: type, keys: type = keysOf(T)): type {
  const names = new Set(arms(keys).map(k => literalValues(k)[0]));
  return mapProperties(T, p => {
    if (!names.has(p.name) || !arms(p.type).includes(type undefined)) return p;
    const rest = arms(p.type).filter(arm => arm !== type undefined);
    return { ...p, optional: true, type: rest.length > 0 ? union(rest) : type undefined };
  });
}

optionalUndefined(type { value: string | undefined, desc: string }, type 'value')
  === type { value?: string, desc: string };
optionalUndefined(type { value: string | undefined, desc: string | undefined })
  === type { value?: string, desc?: string };
optionalUndefined(type { value: string, desc: string }, type 'value')
  === type { value: string, desc: string };
```

Three type parameters, two of which are computed defaults, and `Merge` at the top is the flattening no-op wearing its own name (sixth sighting). The shape of the thing is: split the keys into two groups, build one object with `?` and one without, intersect them, and flatten. `mapProperties` sets the flag on the properties that need it and folds the `undefined` arm into the optionality it now expresses, leaving the rest alone, because a property is a record and both the flag and the type are fields of it.

`undefined extends T[Props]` is `arms(p.type).includes(type undefined)`, and it is worth noticing that TypeScript has to phrase a membership test as an assignability test one more time.

## 30178 · Unique Items

A function that only accepts tuples with no duplicates.

```ts
// TypeScript, issue #33600 (+2)
type Ensure<T,  _P extends unknown[] = []> =
  T extends [ infer F, ...infer R ]
    ? Ensure<R, [ ..._P, F extends _P[number] ? never : F ]>
    : _P extends T ? _P : _P

function uniqueItems<const T>(items: Ensure<T>) {
  return items
}
```

```js
// Builder
function ensureUnique(T: type): type {
  const types = tupleElements(T).map(e => e.type);
  return tupleOf(types.map((t, i) => types.indexOf(t) === i ? t : never));
}

function uniqueItems<const T>(items: ensureUnique(T)) {
  return items;
}

uniqueItems([1, 2, 3]);
uniqueItems([1, 2, 2, 3]);   // error: parameter type is [1, 2, never, 3]
```

A different shape from everything else in this document: the type is not computed for its own sake, it is computed to make a *call* fail. `Ensure` rebuilds the tuple with `never` in the position of any repeat, and since nothing is assignable to `never`, the argument no longer matches and the call errors. Both versions work the same way and both rely on `const` type parameters to get literal types out of an array argument.

The builder's `types.indexOf(t) === i` is the classic "keep first occurrence" test, and the only reason it can be written that way is that interned type objects compare with `===`. Compare the top-voted answer's `F extends _P[number]`, which is the assignability-as-membership problem from challenge 9898 all over again; it survives here because the harness's inputs happen not to contain a literal and its supertype.

## 30575 · BitwiseXOR

```ts
// TypeScript, issue #32992 (+4)
type BitwiseXOR<A extends string, B extends string, X extends string = ''>
  = `${A}^${B}` extends `${infer C}1^${infer D}0` | `${infer C}0^${infer D}1` ? BitwiseXOR<C, D, `1${X}`>
  : `${A}^${B}` extends `${infer C}1^${infer D}1` | `${infer C}0^${infer D}0` ? BitwiseXOR<C, D, `0${X}`>
  : `${A}${B}${X}`
```

```js
// Builder
function bitwiseXOR(a: string, b: string): type {
  const width = Math.max(a.length, b.length);
  const [x, y] = [a.padStart(width, '0'), b.padStart(width, '0')];
  return literal([...x].map((c, i) => c === y[i] ? '0' : '1').join(''));
}

bitwiseXOR('0', '1') === type '1';
bitwiseXOR('1', '1') === type '0';
bitwiseXOR('10', '1') === type '11';
bitwiseXOR('101', '11') === type '110';
```

A lovely trick. To walk two strings in step you need to match them in one pattern, and the way to do that is to *glue them together with a separator that cannot appear in either*: `` `${A}^${B}` `` against `` `${infer C}1^${infer D}0` ``. The `^` is a fencepost, and matching the last bit of each side at once is what the challenge needs. Four cases become two patterns by unioning them, since `1^0` and `0^1` both give a `1`.

The unequal-length case falls out because the shorter string runs out and stops matching, leaving `` `${A}${B}${X}` `` to concatenate the remainder onto the front, which works because one of `A` and `B` is empty by then. That is clever and it is also why `padStart` exists.

## 31797 · Sudoku

Is a solved Sudoku grid valid?

```ts
// TypeScript, issue #35572 (+1)
type RN = [number];
type R1 = [0, 1, 2, 3, 4, 5, 6, 7, 8];
type R3 = [0 | 1 | 2, 3 | 4 | 5, 6 | 7 | 8];
type Values = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;

type MapBool = { [N: number]: true; x: false };
type MapX = { [N: number]: "x" } & Record<Values, never>;
type NotAllX<V extends number, D extends number = Values> = {
  [K in D]: MapX[K & V];
}[D];

type Check<M extends number[][], RI extends number[], RJ extends number[]> = {
  [I in keyof RI]: { [J in keyof RJ]: NotAllX<M[RI[I]][RJ[J]]> }[number];
}[number];

type ValidSudoku1<M extends number[][]> = MapBool[
  | Check<M, R1, RN>
  | Check<M, RN, R1>
  | Check<M, R3, R3>];

type Flatten<T extends number[][]> = T extends [
  infer L extends number[],
  ...infer R extends number[][]
]
  ? [...L, ...Flatten<R>]
  : [];
type Get<M extends number[][][], K extends number[][] = []> = M extends [
  infer L extends number[][],
  ...infer R extends number[][][]
]
  ? Get<R, [...K, Flatten<L>]>
  : K;
type SudokuSolved<
  M extends number[][][],
  K extends number[][] = Get<M>
> = ValidSudoku1<K>;
```

```js
// Builder
function sudokuSolved(M: type): type {
  const rows = tupleElements(M).map(band =>
    tupleElements(band.type).flatMap(cell => tupleElements(cell.type).map(v => literalValues(v.type)[0])));

  const complete = (cells: [].<uint32>): boolean => new Set(cells).size === 9;
  const at = (i: uint32, j: uint32): uint32 => rows[i][j];
  const indices = [0, 1, 2, 3, 4, 5, 6, 7, 8];

  const ok = indices.every(i => complete(indices.map(j => at(i, j))))
          && indices.every(j => complete(indices.map(i => at(i, j))))
          && indices.every(b => complete(indices.map(k =>
               at(Math.floor(b / 3) * 3 + Math.floor(k / 3), (b % 3) * 3 + k % 3))));
  return ok ? type true : type false;
}

type Solved = [
  [[1, 2, 3], [4, 5, 6], [7, 8, 9]],
  [[4, 5, 6], [7, 8, 9], [1, 2, 3]],
  [[7, 8, 9], [1, 2, 3], [4, 5, 6]],
  [[2, 3, 1], [5, 6, 4], [8, 9, 7]],
  [[5, 6, 4], [8, 9, 7], [2, 3, 1]],
  [[8, 9, 7], [2, 3, 1], [5, 6, 4]],
  [[3, 1, 2], [6, 4, 5], [9, 7, 8]],
  [[6, 4, 5], [9, 7, 8], [3, 1, 2]],
  [[9, 7, 8], [3, 1, 2], [6, 4, 5]],
];
sudokuSolved(Solved) === type true;
const rows = tupleElements(Solved).map(band => band.type);
sudokuSolved(tupleOf([rows[0], rows[0], ...rows.slice(2)])) === type false;   // a duplicated row fails the columns
```

The hardest thing in the document to read, and the author's own comment on it is *"method 2: more ingenious"*.

Work out `NotAllX`. `MapX` is an index signature saying every number maps to `'x'`, intersected with `Record<1|2|...|9, never>` saying the nine digits map to `never`. So `MapX[K & V]` is `never` when `K` and `V` are the same digit, and `'x'` otherwise. The mapped type runs that over all nine digits and indexes the result out, so `NotAllX<V>` is `'x'` if `V` is not one of the nine and `never` if it is: the whole thing is a *set-membership test built out of an index signature being shadowed by a Record*. Then `Check` maps that over a row and a column selector, `[number]` collapses each to a union, and `MapBool[...]` turns the presence of any `'x'` back into a boolean. `R3` is three groups of three, so `Check<M, R3, R3>` checks the boxes. `RN` is `[number]`, used as a wildcard selector meaning "all of them".

There is a real algorithm in there and it is correct. It is also nine helper types, three of which exist to do a set lookup, and the fact that the box check works at all depends on `R3`'s arms being unions in exactly the right groups.

`new Set(cells).size === 9`.

## 31824 · Length of String 3

The same challenge a third time, now for very long strings.

```ts
// TypeScript, issue #33269 (+5)
type ToInt<S> = S extends `0${infer X}` ? ToInt<X> : S extends `${infer N extends number}` ? N : 0;

type Q1 = string;
type Q10 = `${Q1}${Q1}${Q1}${Q1}${Q1}${Q1}${Q1}${Q1}${Q1}${Q1 & {}}`;
type Q100 = `${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}${Q10}`;
type Q1000 = `${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}${Q100}`;
type Q10k = `${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}${Q1000}`;
type Q100k = `${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}${Q10k}`;

type Len<S, Q extends string, R extends 1[] = []> = S extends `${Q}${infer T}`
  ? Len<T, Q, [...R, 1]>
  : [R['length'], S];

type LengthOfString<S extends string> = Len<S, Q100k> extends [infer A extends number, infer S1]
  ? Len<S1, Q10k> extends [infer B extends number, infer S2]
    ? Len<S2, Q1000> extends [infer C extends number, infer S3]
      ? Len<S3, Q100> extends [infer D extends number, infer S4]
        ? Len<S4, Q10> extends [infer E extends number, infer S5]
          ? Len<S5, Q1> extends [infer F extends number, string]
            ? ToInt<`${A}${B}${C}${D}${E}${F}`>
            : 0
          : 0
        : 0
      : 0
    : 0
  : 0;
```

```js
// Builder
function lengthOfString(s: string): type {
  return literal([...s].length);
}

lengthOfString('foo') === type 3;
lengthOfString('') === type 0;
lengthOfString('\u{1F603}\u{1F603}') === type 2;      // code points, not UTF-16 units
lengthOfString('x'.repeat(5000)) === type 5000;    // the whole point: no recursion budget to blow
```

Third time. Challenge 298 counts one character at a time. Challenge 651 unrolls ten at a time. This one builds *patterns*: `Q10` is a template type of ten `string`s in a row, `Q100` is ten of those, up to `Q100k`, and then it strips off as many hundred-thousands as it can, then as many ten-thousands, then thousands, hundreds, tens, ones. That is long division in base ten, performed by pattern matching, and the six remainders `A` through `F` are literally the six decimal digits of the answer, which the last line concatenates into a string and parses. It measures a string up to a million characters long by decomposing it positionally.

The `Q1 & {}` in `Q10` is load-bearing: without it `` `${string}${string}` `` collapses to `string` and the whole edifice evaporates. It is a no-op intersection whose only purpose is to stop a simplification.

This is an extremely good answer and it is worth being clear about that. It is also the only challenge in this document whose test cases cannot be checked with a stock compiler. The harness builds a one-million-character string literal type to test against, and `tsc` dies with `JavaScript heap out of memory` and exit code 134; it passes cleanly with `--max-old-space-size=8192`. `[...s].length` runs in microseconds and iterates code points, which is the thing all three challenges are actually about.

## 32427 · Unbox

Unwrap functions, arrays, and promises, to a given depth.

```ts
// TypeScript, issue #32469 (+6)
type Unbox<T, Depth = 0, Count extends any[] = [1]> = T extends ((...args: any[]) => infer R) | (infer R)[] | Promise<infer R>
  ? Count['length'] extends Depth ? R : Unbox<R, Depth, [...Count, 1]>
  : T
```

```js
// Builder
function unbox(T: type, depth: uint32 = 0): type {
  const peel = (t: type): type | undefined => {
    const node = reflect(t);
    if (node.kind === 'function') return node.signatures[0].return.type;
    if (node.kind === 'array' || node.kind === 'tuple') return node.element ?? union(node.elements.map(e => e.type));
    if (node.generic?.base === Promise) return node.generic.arguments[0];
    return undefined;
  };
  let out = T;
  for (let i = 1; ; i++) {
    const next = peel(out);
    if (next === undefined || (depth !== 0 && i > depth)) return out;
    out = next;
    if (i === depth) return out;
  }
}

unbox(type () => Promise.<() => [].<Promise.<boolean>>>) === boolean;
unbox(type () => () => () => () => float64, 2) === type () => () => float64;
unbox(type () => () => () => () => float64, 4) === float64;
```

`((...args: any[]) => infer R) | (infer R)[] | Promise<infer R>` is three patterns in one, with the same `infer R` in all three, so whichever matches binds `R` and the other two are ignored. That is a genuinely elegant use of union patterns and there is nothing clumsy about it. The builder's `peel` is a three-branch `if`, which is the same three cases with the shared variable made explicit rather than implicit.

`Count extends any[] = [1]` is the depth counter, tuple-length again, and `Depth = 0` doubles as "all the way down" because the counter starts at one and never equals zero.

## 32532 · Binary Addition

```ts
// TypeScript, issue #33275 (+0)
type Bit = 1 | 0

type LastBit<T extends Bit[]> = T extends [...Bit[], infer B extends Bit] ? B : never
type PoppedBits<T extends Bit[]> = T extends [...infer Rest extends Bit[], Bit] ? Rest : never
type FilterOne<T extends Bit[]> = T extends [] ? [] : [...(LastBit<T> extends 0 ? [] : [1]), ...FilterOne<PoppedBits<T>>]
type NumOfOne<T extends Bit[]> = FilterOne<T>['length']

type BinaryAdd<
  A extends Bit[],
  B extends Bit[],
  Carry extends Bit = 0,
  Result extends Bit[] = [],
  // Internals
  CurrentBitA extends Bit = LastBit<A>,
  CurrentBitB extends Bit = LastBit<B>,
  CurrentBitResult extends Bit = NumOfOne<[CurrentBitA, CurrentBitB, Carry]> extends (0 | 2) ? 0 : 1,
  NextA extends Bit[] = PoppedBits<A> extends [] ? [0] : PoppedBits<A>,
  NextB extends Bit[] = PoppedBits<B> extends [] ? [0] : PoppedBits<B>,
  NextCarry extends Bit = NumOfOne<[CurrentBitA, CurrentBitB, Carry]> extends (2 | 3) ? 1 : 0,
> = [A, B, Carry] extends [[0], [0], 0]
  ? Result
  : BinaryAdd<
      NextA,
      NextB,
      NextCarry,
      [CurrentBitResult, ...Result]
    >
```

```js
// Builder
function binaryAdd(A: type, B: type): type {
  const bits = (T: type): [].<uint32> => tupleElements(T).map(e => literalValues(e.type)[0]);
  const [a, b] = [bits(A), bits(B)];
  const out = [];
  let carry = 0;
  for (let i = 1; i <= Math.max(a.length, b.length) || carry; i++) {
    const sum = (a.at(-i) ?? 0) + (b.at(-i) ?? 0) + carry;
    out.unshift(literal(sum % 2));
    carry = sum >> 1;
  }
  return tupleOf(out);
}

binaryAdd(type [1], type [1]) === type [1, 0];
binaryAdd(type [0], type [1]) === type [1];
```

Eight type parameters, five of them marked `// Internals` by the author, which is a comment saying "these are locals and I have nowhere else to put them". `NumOfOne<[A, B, Carry]>` counts the set bits of a three-element tuple by *filtering it and taking the length*, and it is called twice, once for the sum bit and once for the carry, because the result of the first call cannot be reused. `extends (0 | 2)` and `extends (2 | 3)` are the truth table, spelled as membership in a union of counts, and they are correct and quite pretty.

The builder is the addition loop from any textbook. `carry = sum >> 1`.

## 33763 · Union to Object from key

Keep the arms of a union that have a given key.

```ts
// TypeScript, issue #35025 (+4)
type UnionToObjectFromKey<Union, Key> = Union extends Union
  ? Key extends keyof Union
    ? Union
    : never
  : never
```

```js
// Builder
function unionToObjectFromKey(U: type, key: type): type {
  const name = literalValues(key)[0];
  return union(arms(U).filter(a => reflect(a).properties.some(p => p.name === name)));
}

type Foo = { foo: string, common: boolean };
type Bar = { bar: float64, common: boolean };
unionToObjectFromKey(type Foo | Bar, type 'foo') === Foo;
unionToObjectFromKey(type Foo | Bar, type 'common') === type Foo | Bar;
```

`Union extends Union` one more time, and by now it needs no explanation: a `filter` over a union is spelled as a tautology plus a conditional, because distribution is the only iteration a union has. The dropped arms become `never` and vanish when the results are re-unioned, which is why the filter works at all.

## 34286 · Take Elements

Take `n` from the front, or from the back if `n` is negative.

```ts
// TypeScript, issue #37396 (+3)
type Take<
  N extends number,
  Arr extends any[],
  Abs = `${N}` extends `-${infer NN extends number}` ? NN : N,
  Res extends any[] = []
> = Res["length"] extends Abs
  ? Res
  : Arr extends (N extends Abs ? [infer First, ...infer Rest] : [...infer Rest, infer Last])
  ? Take<N, Rest, Abs, N extends Abs ? [...Res, First] : [Last, ...Res]>
  : Res;
```

```js
// Builder
function take(n: int32, T: type): type {
  const types = tupleElements(T).map(e => e.type);
  return tupleOf(n >= 0 ? types.slice(0, n) : types.slice(n));
}

take(2, type [1, 2, 3]) === type [1, 2];
take(-2, type [1, 2, 3]) === type [2, 3];
take(0, type [1, 2, 3]) === type [];
take(5, type [1, 2, 3]) === type [1, 2, 3];
```

`N extends Abs` is the sign test, and it appears three times in five lines: once to choose which end to destructure, once to choose which end to accumulate, and it is recomputed each time because there is nowhere to keep it. `Abs` is a computed type parameter, which is this language's `const`. `slice` takes negative indices, so `n >= 0` is the only branch the builder needs and the clamping cases fall out.

## 35314 · Valid Sudoku

The last hard challenge, and a repeat of 31797 with a flat grid.

```ts
// TypeScript, issue #35322 (+5)
type RN = [number];
type R1 = [0, 1, 2, 3, 4, 5, 6, 7, 8];
type R3 = [0 | 1 | 2,  3 | 4 | 5,  6 | 7 | 8];
type Values = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;

type MapBool = {[N: number]: true; x: false};
type MapX = {[N: number]: 'x'} & Record<Values, never>;
type NotAllX<V extends number, D extends number = Values> = {[K in D]: MapX[K & V]}[D];

type Check<M extends number[][], RI extends number[], RJ extends number[]> = {
  [I in keyof RI]: {[J in keyof RJ]: NotAllX<M[RI[I]][RJ[J]]>}[number];
}[number];

type ValidSudoku<M extends number[][]> = MapBool[
  | Check<M, R1, RN>
  | Check<M, RN, R1>
  | Check<M, R3, R3>];
```

```js
// Builder
function validSudoku(M: type): type {
  const grid = tupleElements(M).map(row => tupleElements(row.type).map(c => literalValues(c.type)[0]));
  const complete = (cells: [].<uint32>): boolean => new Set(cells).size === 9;
  const i9 = [0, 1, 2, 3, 4, 5, 6, 7, 8];
  return i9.every(i => complete(grid[i]))
      && i9.every(j => complete(i9.map(i => grid[i][j])))
      && i9.every(b => complete(i9.map(k =>
           grid[Math.floor(b / 3) * 3 + Math.floor(k / 3)][(b % 3) * 3 + k % 3])))
    ? type true : type false;
}

type Grid = [
  [1, 2, 3, 4, 5, 6, 7, 8, 9],
  [4, 5, 6, 7, 8, 9, 1, 2, 3],
  [7, 8, 9, 1, 2, 3, 4, 5, 6],
  [2, 3, 1, 5, 6, 4, 8, 9, 7],
  [5, 6, 4, 8, 9, 7, 2, 3, 1],
  [8, 9, 7, 2, 3, 1, 5, 6, 4],
  [3, 1, 2, 6, 4, 5, 9, 7, 8],
  [6, 4, 5, 9, 7, 8, 3, 1, 2],
  [9, 7, 8, 3, 1, 2, 6, 4, 5],
];
validSudoku(Grid) === type true;
const gridRows = tupleElements(Grid).map(r => r.type);
validSudoku(tupleOf([gridRows[0], gridRows[0], ...gridRows.slice(2)])) === type false;
```

The same nine helper types as challenge 31797, verbatim, because it is the same problem with the input reshaped: 31797 takes a 9x3x3 nesting and flattens it first, this one takes the 9x9 directly. The set-membership test built out of an index signature shadowed by a `Record` is unchanged, and so is the builder: `new Set(cells).size === 9` for rows, columns, and boxes.

That completes the hard tier: fifty-five challenges.

## 5 · Get Readonly Keys

The first extreme challenge.

```ts
// TypeScript, issue #139 (+21)
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends <T>() => T extends Y ? 1 : 2 ? true : false;

type GetReadonlyKeys<
  T,
  U extends Readonly<T> = Readonly<T>,
  K extends keyof T = keyof T
> = K extends keyof T ? Equal<Pick<T, K>, Pick<U, K>> extends true ? K : never : never;
```

```js
// Builder
function getReadonlyKeys(T: type): type {
  return union(reflect(T).properties.filter(p => p.readonly).map(p => literal(p.name)));
}

interface Todo { readonly title: string, description: string, completed: boolean }
getReadonlyKeys(Todo) === type 'title';
```

This is rated **extreme**. It is the hardest tier the repository has, above the fifty-five hard challenges and the hundred and four medium ones, and the task is to find out which properties are `readonly`.

It is extreme because there is no way to ask. The answer imports the identity incantation, builds `Readonly<T>` for the whole object, picks each key from both versions, and compares the two one-key objects for identity: if making it readonly changed nothing, it was already readonly. That is challenge 5181's technique, and 5181 is rated hard, and the only difference between the two challenges is which half of the answer you keep.

`p.readonly` is a boolean on a property record. The builder is a `filter`, and it is the same `filter` as challenges 57, 59, 89, 90, 2857, and 5181, with a different field name.

## 151 · Query String Parser

```ts
// TypeScript, issue #21419 (+9). Thirty lines; the head is shown.
type ParseQueryString<S extends string> = S extends '' ? {} : MergeParams<SplitParams<S>>;

// e.g. 'k1=v1&k2=v2&k2=v3&k1' => ['k1=v1', 'k2=v2', 'k2=v3', 'k1']
type SplitParams<S extends string> =
  S extends `${infer E}&${infer Rest}`
    ? [E, ...SplitParams<Rest>]
    : [S]
;

// e.g. ['k1=v1', 'k2=v2', 'k2=v3', 'k1']
// => { k1: 'v1' } => { k1: 'v1', k2: ['v2', 'v3'] } => { k1: ['v1', true], k2: ['v2', 'v3'] }
type MergeParams<T extends string[], M = {}> =
  T extends [infer E, ...infer Rest extends string[]]
    ? E extends `${infer K}=${infer V}`
      ? MergeParams<Rest, SetProperty<M, K, V>>
      : E extends `${infer K}`
        ? MergeParams<Rest, SetProperty<M, K>>
        : never
    : M
;

// ... SetProperty, which folds each parsed pair into the result object, follows
```

```js
// Builder
function parseQueryString(s: string): type {
  const groups = new Map();
  for (const part of s.split('&').filter(x => x !== '')) {
    const eq = part.indexOf('=');
    const [key, value] = eq === -1 ? [part, true] : [part.slice(0, eq), part.slice(eq + 1)];
    const seen = groups.get(key) ?? [];
    if (!seen.includes(value)) groups.set(key, [...seen, value]);
  }
  return objectOf([...groups].map(([key, values]) =>
    prop(key, values.length === 1 ? literal(values[0]) : tupleOf(values.map(literal)))));
}

parseQueryString('') === type {};
parseQueryString('k1') === type { k1: true };
parseQueryString('k1&k1') === type { k1: true };
parseQueryString('k1=v1&k1=v2') === type { k1: ['v1', 'v2'] };
parseQueryString('k1=v1&k2=v2&k1=v2') === type { k1: ['v1', 'v2'], k2: 'v2' };
```

`SplitParams` is `String.prototype.split`, `MergeParams` is the `for` loop, and `SetProperty` (omitted above, another dozen lines) is the `Map` with its dedupe and its one-value-versus-list rule. This is a real parser and it is a fair fight in structure, but count the pieces: four type aliases and about thirty lines, against a `split`, a `Map`, and an `indexOf`.

## 216 · Slice

```ts
// TypeScript, issue #22110 (+18)
type ToPositive<N extends number, Arr extends unknown[]> =
  `${N}` extends `-${infer P extends number}`
  ? Slice<Arr, P>['length']
  : N

// get the initial N items of Arr
type InitialN<Arr extends unknown[], N extends number, _Acc extends unknown[] = []> = 
  _Acc['length'] extends N | Arr['length']
  ? _Acc
  : InitialN<Arr, N, [..._Acc, Arr[_Acc['length']]]>

type Slice<Arr extends unknown[], Start extends number = 0, End extends number = Arr['length']> = 
  InitialN<Arr, ToPositive<End, Arr>> extends [...InitialN<Arr, ToPositive<Start, Arr>>, ...infer Rest]
  ? Rest
  : []
```

```js
// Builder
function slice(T: type, start: int32 = 0, end: int32 = tupleElements(T).length): type {
  return tupleOf(tupleElements(T).map(e => e.type).slice(start, end));
}

slice(type [1, 2, 3, 4, 5], 2, 4) === type [3, 4];
slice(type [1, 2, 3, 4, 5], 0, -1) === type [1, 2, 3, 4];
slice(type [1, 2, 3, 4, 5], -3, -1) === type [3, 4];
slice(type [1, 2, 3, 4, 5], 10) === type [];
slice(type [1, 2, 3, 4, 5], 1, 0) === type [];
slice(type [1, 2, 3, 4, 5]) === type [1, 2, 3, 4, 5];
```

Rated extreme. `Array.prototype.slice` has taken negative indices and clamped out-of-range ones since 1997, and the entire difficulty of this challenge is reimplementing that: `ToPositive` turns `-3` into `2` by *slicing the array by three and taking its length*, which is a recursive call to the thing being defined. Every one of the harness's "invalid" cases is a line in ECMA-262 that `slice` already handles.

## 274 · Integers Comparator

```ts
// TypeScript, issue #11444 (+13). Thirty-six lines; the head is shown.
type Comparator<A extends number | string, B extends number | string>
  = `${A}` extends `-${infer AbsA}`
    ? `${B}` extends `-${infer AbsB}`
      ? ComparePositives<AbsB, AbsA>
      : Comparison.Lower
    : `${B}` extends `-${number}`
      ? Comparison.Greater
      : ComparePositives<`${A}`, `${B}`>

// ... ComparePositives, CompareByLength, and CompareByDigits follow
```

```js
// Builder
function comparator(a: float64 | string, b: float64 | string): type {
  const [x, y] = [BigInt(a), BigInt(b)];
  return x < y ? Comparison.Lower : x > y ? Comparison.Greater : Comparison.Equal;
}

comparator(5, 5) === Comparison.Equal;
comparator(-5, 0) === Comparison.Lower;
comparator(0, -5) === Comparison.Greater;
```

The sign handling is the readable part and it is correct: negative beats nothing, and two negatives compare by their absolute values reversed. `ComparePositives` underneath is challenge 4425's digit-wise decimal comparison again, because the challenge takes strings so it can exceed float64 and the tuple encodings are hopeless past ten thousand anyway. `BigInt` is the answer to "numbers larger than a float64", and it has been in the language since 2020.

## 462 · Currying 2

Curry with any argument split.

```ts
// TypeScript, issue #3697 (+16)
type Curry<A, R, D extends unknown[] = []> =
    A extends [infer H, ...infer T]
      ? T extends []
        ? (...args: [...D, H]) => R
        : ((...args: [...D, H]) => Curry<T, R>) & Curry<T, R, [...D, H]>
      : () => R

declare function DynamicParamsCurrying<A extends unknown[], R>(fn: (...args: A) => R): Curry<A, R>
```

```js
// Builder
function curry(Params: type, R: type): type {
  const elements = tupleElements(Params);
  if (elements.length === 0) return fn([], R);
  return Reflect.makeType({ kind: 'intersection', members: elements.map((_, i) => {
    const [head, tail] = [elements.slice(0, i + 1), elements.slice(i + 1)];
    return fn(head.map(e => e.type), tail.length === 0 ? R : curry(tupleOf(tail.map(e => e.type)), R));
  })});
}

declare function dynamicParamsCurrying<A, R>(f: (...args: A) => R): curry(A, R);

const curried = dynamicParamsCurrying((a: string, b: number, c: boolean) => true);
curried('a')(1)(true) === true;
curried('a', 1)(true) === true;
curried('a')(1, true) === true;
```

An overload set covering every way to split the arguments, and the recursion produces it by intersecting "take the first `k`" with "take one more". Both versions are the same idea. The builder's `members` is a list produced by a `map` over the split points, which makes the structure visible: there are exactly `n` ways to make the first call, and each leaves a smaller curry behind.

This is one of the better challenges in the repository and one of the closer results in this document. `ThisType`, overload sets, and variadic tuples are real features doing real work, and the difference is only that one side can name the split point.

## 476 · Sum

```ts
// TypeScript, issue #506 (+6). Eighty-six lines; the first twenty are shown.
type Or<left extends boolean, right extends boolean> =
  left extends true ? true : right extends true ? true : false;

type CoalesceToString<n extends string | number | bigint> = n extends string ? n : `${n}`;

type Digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";

type SubOneFromDigit<digit extends Digit> =
  digit extends "1" ? "0"
  : digit extends "2" ? "1"
  : digit extends "3" ? "2"
  : digit extends "4" ? "3"
  : digit extends "5" ? "4"
  : digit extends "6" ? "5"
  : digit extends "7" ? "6"
  : digit extends "8" ? "7"
  : digit extends "9" ? "8"
  : digit extends "10" ? "9"
  : never;

// ... AddOneToDigit, AddDigits, PadWithZeroes, AddSameLengthNumbers, and Sum follow
```

```js
// Builder
function sum(a: string | float64 | bigint, b: string | float64 | bigint): type {
  return literal(String(BigInt(a) + BigInt(b)));
}

sum(2, 3) === type '5';
sum('13', '21') === type '34';
sum(9999, 1) === type '10000';
sum(1_000_000_000_000n, '123') === type '1000000000123';
sum(4325234, '39532') === type '4364766';
```

## 517 · Multiply

```ts
// TypeScript, issue #11580 (+1). Seventy-one lines; the head is shown. The top-voted answer
// (#5814, +13) does not compile: its `Reverse<A>` interpolates an unconstrained `A`.
type Multiply<
  A extends number | string | bigint, B extends number | string | bigint,
  ATuple extends number[] = ToTuple<A>, BTuple extends number[] = ToTuple<B>,
> = ToString<TrimZeros<Sum<MultiplyToSum<ATuple, BTuple>>>>;

type MultiplyToSum<
  A extends number[],
  B extends number[],
  Result extends number[][] = [],
  FullB extends number[] = B,
  ABTrailingZeros extends 0[] = [],
  ATrailingZeros extends 0[] = ABTrailingZeros
> =
  A extends [...infer AS extends number[], infer AL extends number]
  ? B extends [...infer BS extends number[], infer BL extends number]
    ? MultiplyToSum<A, BS, [...Result, [...MultTable[AL][BL], ...ABTrailingZeros]], FullB, [0, ...ABTrailingZeros]>
    : MultiplyToSum<AS, FullB, Result, FullB, [0, ...ATrailingZeros], [0, ...ATrailingZeros]>
  : Result;

// ... MultTable, ToTuple, ToString, TrimZeros, Sum follow
```

```js
// Builder
function multiply(a: string | float64 | bigint, b: string | float64 | bigint): type {
  return literal(String(BigInt(a) * BigInt(b)));
}

multiply(2, 3) === type '6';
multiply(0, 16) === type '0';
multiply('13', '21') === type '273';
multiply('43423', 321543n) === type '13962361689';
multiply(9999, 1) === type '9999';
```

These two are the end of the road the whole arithmetic column has been on, and the repository is honest about where it leads: they are rated extreme, they are eighty-six and seventy-one lines, and they are `+` and `*`.

Every technique this document has catalogued is in them. Digit lookup tables, per-digit carry propagation, `PadWithZeroes` to align operands, `TrimZeros` to clean up after, a nine-by-nine multiplication table, and long multiplication accumulating partial products with trailing zeros for place value. The work is good. The authors of these answers implemented decimal arithmetic correctly, from nothing, in a language with no numbers, and got it right.

The signatures take `string | number | bigint` and return a string, which is the tell: the challenges are about arithmetic on values too large to be a `number`, and every answer's real subject is the representation, not the operation. JavaScript shipped `BigInt` in 2020, so `BigInt(a) * BigInt(b)` is exact at any size, and `String()` puts it back. One line each.

## 697 · Tag

Nominal tags that are order-insensitive and idempotent.

```ts
// TypeScript, issue #1947 (+5). One hundred and nine lines; the head is shown.
type IsNever<T> = [T] extends [never] ? true : false
type IsAny<T> = 0 extends (1 & T) ? true : false
type Is<T, K> = [T] extends [K] ? [K] extends [T] ? true : false : false

type StartsWith<T extends any[], K extends any[]> =
  T extends [infer T0, ...infer Tr]
  ? K extends [infer K0, ...infer Kr]
    ? Is<T0, K0> extends true ? StartsWith<Tr, Kr> : false
    : true
  : IsNever<K[0]>

// ... Insert, Sorted, TagList, Tagged, Tag, UnTag follow
```

```js
// Builder
const TAGS = Symbol('tags');

function tag(T: type, name: string): type {
  const existing = reflect(T).properties.find(p => p.name === TAGS);
  const names = existing === undefined ? [] : literalValues(existing.type).map(String);
  const merged = [...new Set([...names, name])].sort();
  return objectOf([
    ...reflect(T).properties.filter(p => p.name !== TAGS),
    { ...prop(TAGS, union(merged.map(literal))), optional: true, readonly: true },
  ]);
}

function unTag(T: type): type {
  return objectOf(reflect(T).properties.filter(p => p.name !== TAGS));
}

interface I { foo: string }
tag(tag(I, 'a'), 'b') === tag(tag(I, 'b'), 'a');   // order does not matter
tag(tag(I, 'a'), 'a') === tag(I, 'a');             // tagging twice does not
unTag(tag(tag(I, 'c'), 'b')) === I;
```

The requirement is that `Tag<Tag<I, 'a'>, 'b'>` and `Tag<Tag<I, 'b'>, 'a'>` be the *same type*, which means the tags have to be canonicalized. TypeScript has no set and no sort, so the answer builds both: `Insert` puts a tag into a sorted list by walking it and comparing, `Sorted` checks the invariant, `StartsWith` compares two lists element by element with `Is` (the double-tuple-wrapped identity test), and `IsNever` and `IsAny` guard the edges. One hundred and nine lines to keep a set of strings in order.

`[...new Set(names)].sort()` is a set and a sort. The builder puts them behind a symbol key, which is what challenge 553 established: a phantom property nobody has to provide, carrying data the type system compares structurally. Interning does the rest, since two objects built from the same sorted list are the same object.

## 734 · Inclusive Range

```ts
// TypeScript, issue #6111 (+5)
type InclusiveRange<Lower extends number,Higher extends number, C extends any[] = [], I = false, R extends number[] = []> =
  I extends true ?
    C["length"] extends Higher ?
      [...R, Higher] :
    InclusiveRange<Lower, Higher, [...C, 1], true, [...R, C["length"]]> :
  C["length"] extends Lower ?
    InclusiveRange<Lower, Higher, C, true, R> :
  C["length"] extends Higher ?
    [] :
  InclusiveRange<Lower, Higher, [...C, 1], I, R>
```

```js
// Builder
function inclusiveRange(low: int32, high: int32): type {
  return tupleOf(Array.from({ length: Math.max(0, high - low + 1) }, (_, i) => literal(low + i)));
}

inclusiveRange(200, 1) === type [];
inclusiveRange(5, 5) === type [5];
inclusiveRange(0, 10) === type [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
```

Five type parameters, of which `I` is a boolean flag meaning "we have reached the lower bound and are now collecting", threaded through every call because there is nowhere else to keep it. The counter walks from zero to `Higher` regardless of where `Lower` is, so `InclusiveRange<1, 200>` counts to two hundred twice over: once in `C` and once in `R`. `Math.max(0, high - low + 1)` is the length, and the `[]` case for an inverted range falls out of it.

## 741 · Sort

```ts
// TypeScript, issue #856 (+9). Forty-seven lines; the head is shown.
type Iterator<n, iterator extends any[] = []> =
  iterator['length'] extends n
    ? iterator
    : Iterator<n, [any, ...iterator]>

type LessThanOrEqual<a extends any[], b extends any[]> =
  [a, b] extends [[], [any, ...any]]
    ? true
    : [a, b] extends [[any, ...any], []]
      ? false
      : [a, b] extends [[any, ...infer as], [any, ...infer bs]]
        ? LessThanOrEqual<as, bs>
        : true

// ... Insert, Sort follow
```

```js
// Builder
function sort(T: type, descending: boolean = false): type {
  const values = tupleElements(T).map(e => literalValues(e.type)[0]);
  return tupleOf([...values].sort((a, b) => descending ? b - a : a - b).map(literal));
}

sort(type []) === type [];
sort(type [3, 2, 1]) === type [1, 2, 3];
sort(type [2, 4, 7, 6, 6, 6, 5, 8, 9], true) === type [9, 8, 7, 6, 6, 6, 5, 4, 2];
```

An insertion sort, and the comparison at the bottom of it is the thing to look at: `LessThanOrEqual` converts both numbers to tuples of that length and then removes one element from each until one of them runs out. To decide `7 <= 9` it builds a seven-element tuple and a nine-element tuple and peels them in lockstep. Every comparison in the sort does this again, from scratch.

`Array.prototype.sort` takes a comparator. `descending ? b - a : a - b` is the whole difference between the two halves of the test suite.

## 869 · DistributeUnions

Turn a structure containing unions into a union of structures.

```ts
// TypeScript, issue #11761 (+9)
type DistributeUnions<T>
  = T extends unknown[] ? DistributeArray<T>
  : T extends object ? Merge<DistributeObject<T>>
  : T

type DistributeArray<A extends unknown[]>
  = A extends [infer H, ...infer T]
  ? ArrHelper<DistributeUnions<H>, T>
  : []
type ArrHelper<H, T extends unknown[]> = H extends H ? [H, ...DistributeArray<T>] : never

type DistributeObject<O extends object, K extends keyof O = keyof O>
  = [K] extends [never] ? {}
  : K extends K ? ObjHelper<K, DistributeUnions<O[K]>> & DistributeObject<Omit<O, K>>
  : never
type ObjHelper<K, V> = V extends V ? { [k in K & string]: V } : never

type Merge<O> = { [K in keyof O]: O[K] }
```

```js
// Builder
function distributeUnions(T: type): type {
  const cross = (columns: [].<[].<type>>): [].<[].<type>> =>
    columns.length === 0 ? [[]]
      : cross(columns.slice(1)).flatMap(rest => columns[0].map(head => [head, ...rest]));

  const node = reflect(T);
  if (node.kind === 'tuple')
    return union(cross(node.elements.map(e => arms(distributeUnions(e.type)))).map(tupleOf));
  if (node.kind === 'object')
    return union(cross(node.properties.map(p => arms(distributeUnions(p.type))))
      .map(row => objectOf(node.properties.map((p, i) => ({ ...p, type: row[i] })))));
  return T;
}

distributeUnions(type [1 | 2, 'a' | 'b'])
  === type [1, 'a'] | [1, 'b'] | [2, 'a'] | [2, 'b'];
distributeUnions(type { a: 1 | 2, b: 'x' }) === type { a: 1, b: 'x' } | { a: 2, b: 'x' };
```

A cross product over every field, which is the same shape as challenge 27862 and challenge 8767, and here it needs two helper types (`ArrHelper`, `ObjHelper`) whose entire job is to be a place where `H extends H` and `V extends V` can be written, because distribution only happens on a naked type parameter and the value being distributed arrives as the result of a call. `Merge` at the end is the flattening no-op, seventh sighting.

`cross` is the cross product, and it is the same six lines as `perms` in the preamble.

## 925 · Assert Array Index

An index type that lets you read an array without `| undefined`.

```ts
// TypeScript, issue #7058 (+4). Sixty-eight lines; the head is shown.
type HashMapHelper<T extends number, R extends unknown[] = []> = R['length'] extends T
  ? R
  : HashMapHelper<T, [...R, unknown]>

type HashMap = {
  '0': HashMapHelper<0>
  '1': HashMapHelper<1>
  '2': HashMapHelper<2>
  // ... through '9'
}

// ... StringToTuple, Add, Index, assertArrayIndex follow
```

```js
// Builder
function at(A: type, index: uint32): type {
  const node = reflect(A);
  if (node.kind !== 'tuple') throw new TypeError(`at: ${String(A)} is not a tuple type`);
  if (index >= node.elements.length)
    throw new TypeError(`at: index ${index} is out of bounds for ${String(A)}`);
  return node.elements[index].type;   // never `| undefined`: out of bounds is an error, not a value
}

type Matrix = [[1, 2], [3, 4], [5, 6]];
at(Matrix, 1) === type [3, 4];
at(at(Matrix, 1), 0) === type 3;
at(Matrix, 9);   // TypeError: at: index 9 is out of bounds for [[1, 2], [3, 4], [5, 6]]
```

The one challenge in the repository that needs a compiler option changed: its test cases only typecheck with `noUncheckedIndexedAccess` on, which `tsconfig.base.json` does not set. The README says so, so this is a documented requirement rather than a bug, but it does mean the challenge cannot be checked by the repo's own configuration.

The answer's `HashMap` is a lookup table from each digit character to a tuple of that length, and `StringToTuple` and `Add` build on it, because the index type has to do arithmetic to stay in bounds and the arithmetic has to be built first. Sixteen challenges in this document build a tuple purely to count with it, and this is one more. The builder does not need any of it: `at` evaluates the comparison the checker cannot, so out of bounds is an error at build time and an in-bounds read never carries `| undefined`. The `asserts` brand and the digit table were never the point; `i < length` was.

## 6228 · JSON Parser

```ts
// TypeScript, issue #6329 (+16)
//My answer is only for the test cases
type Parse<T extends string> = Eval<T> extends [infer V, infer U] ? V : never

type Eval<T>
  = T extends `${' '|'\n'}${infer U}` ? Eval<U>
  : T extends `true${infer U}` ? [true, U]
  : T extends `false${infer U}` ? [false, U]
  : T extends `null${infer U}` ? [null, U]
  : T extends `"${infer V}"${infer U}` ? [V, U]
  : T extends `[${infer U}` ? EvalArray<U>
  : T extends `{${infer U}` ? EvalObject<U>
  : never

// ... EvalArray, EvalObject follow
```

```js
// Builder
function parse(source: string): type {
  const build = (v): type =>
    v === null ? type null
    : Array.isArray(v) ? tupleOf(v.map(build))
    : typeof v === 'object' ? objectOf(Object.entries(v).map(([k, x]) => prop(k, build(x))))
    : literal(v);
  return build(JSON.parse(source));
}

parse('{"a": "b", "b": false, "c": [true, false, "hello"], "nil": null}')
  === type { a: 'b', b: false, c: [true, false, 'hello'], nil: null };
```

A recursive-descent parser, written as conditional types, and it is legitimately impressive: whitespace skipping, three literal keywords, quoted strings, arrays, objects, and a `[value, rest]` pair threaded through every production to carry the remaining input, which is exactly how you write a parser combinator by hand in any language. The author's first line is `//My answer is only for the test cases`, which is a fair warning: it does not handle numbers, escapes, or nesting depth beyond what the harness asks.

`JSON.parse` has been in the language since 2009 and handles all of it. The builder's remaining work is turning the parsed value into a type, which is the recursion `build` does in four lines, and the fact that `parse` runs at compile time is the entire point of the proposal: the compiler runs your JavaScript.

## 7561 · Subtract

The last arithmetic challenge in the repository.

```ts
// TypeScript, issue #11216 (+8)
type Tuple<T, Res extends 1[] = []> = 0 extends 1 ? never : Res['length'] extends T ? Res : Tuple<T, [...Res, 1]>;

type Subtract<M extends number, S extends number> = Tuple<M> extends [...Tuple<S>, ...infer Rest] ? Rest['length'] : never
```

```js
// Builder
function subtract(m: uint32, s: uint32): type {
  return m >= s ? literal(m - s) : never;
}

subtract(1, 1) === type 0;
subtract(2, 1) === type 1;
subtract(1, 2) === never;
subtract(1000, 999) === type 1;
```

Three lines, and the best thing in the document, for two reasons.

The first is `0 extends 1 ? never : ...`. That condition is always false, so the branch is dead and the type means exactly what it would without it. Its only purpose is to defer instantiation, and it works: I checked. With the guard, `Subtract<1000, 999>` evaluates to `1`. Remove it, change nothing else, and the compiler answers `error TS2589: Type instantiation is excessively deep and possibly infinite`. A no-op conditional is the difference between the answer working and not working.

The second is the harness:

```ts
Expect<Equal<Subtract<1, 1>, 0>>,
Expect<Equal<Subtract<2, 1>, 1>>,
Expect<Equal<Subtract<1, 2>, never>>,
// @ts-expect-error
Expect<Equal<Subtract<1000, 999>, 1>>,
```

The last line says that `1000 - 999` must **fail**. Not "may be slow", not "is left as an exercise": the test asserts an error, so the exercise formally specifies that subtracting nine hundred and ninety-nine from a thousand does not work. And because this answer's guard beats the ceiling and computes `1`, the directive goes unused and `tsc` reports `error TS2578: Unused '@ts-expect-error' directive`. **The top-voted answer fails the test suite because it is correct.**

The challenge is `-`. It is rated extreme. `m - s`.

## 31447 · CountReversePairs

Count the inversions: pairs where an earlier element is greater than a later one.

```ts
// TypeScript, issue #31605 (+2)
type SimpleGreaterThan<A extends number, B extends number, _StackA extends 1[] = [], _StackB extends 1[] = []> =
  `${A}${B}` extends `-${infer NA extends number}-${infer NB extends number}` ?
  SimpleGreaterThan<NB, NA> : //Negative
  `${A}` extends `-${number}` ? false : //only A Negative
  `${B}` extends `-${number}` ? true : //only B Negative
  _StackA['length'] extends A ? false :
  _StackB['length'] extends B ? true :
  SimpleGreaterThan<A, B, [..._StackA, 1], [..._StackB, 1]>

// ... CountReversePairs follows
```

```js
// Builder
function countReversePairs(T: type): type {
  const values = tupleElements(T).map(e => literalValues(e.type)[0]);
  return literal(values.reduce((n, a, i) => n + values.slice(i + 1).filter(b => a > b).length, 0));
}

countReversePairs(type [5, 2, 6, 1]) === type 4;
countReversePairs(type [1, 2, 3, 4]) === type 0;
countReversePairs(type [-1, -1]) === type 0;
```

`SimpleGreaterThan` counts two tuples up in parallel and reports whichever runs out first, with the sign cases peeled off in front by string matching, and note the third one: `` `${A}${B}` extends `-${infer NA}-${infer NB}` `` concatenates the two numbers to test both signs in a single pattern, which is challenge 30575's fencepost trick without a fencepost, and works here because a minus sign can only appear at the start.

`a > b`.

## 31997 · Parameter Intersection

Merge two parameter lists, intersecting positionally and handling rest elements.

```ts
// TypeScript, issue #33210 (+3)
type Arr = readonly unknown[];

type Intr<A, B> = [A, B] extends [object, object] ? {[K in keyof (A & B)]: (A & B)[K]} : A & B;

type IntersectParameters<X extends Arr, Y extends Arr, R extends Arr = []> = X extends readonly []
  ? [...R, ...Y]
  : Y extends readonly []
    ? [...R, ...X]
    : [X, Y] extends [readonly [(infer A)?, ...infer XT], readonly [(infer B)?, ...infer YT]]
      ? '0' extends (keyof X) | (keyof Y) 
        ? IntersectParameters<XT, YT, [[], []] extends [X, Y] ? [...R, Intr<A, B>?] : [...R, Intr<A, B>]>
        : [...R, ...Intr<A, B>[]]
      : never;
```

```js
// Builder
function intersectParameters(X: [].<Reflect.FunctionParameterReflection>, Y: [].<Reflect.FunctionParameterReflection>):
    [].<Reflect.FunctionParameterReflection> {
  const both = (a, b) => Reflect.makeType({ kind: 'intersection', members: [a.type, b.type] });
  const out = [];
  for (let i = 0; i < Math.max(X.length, Y.length); i++) {
    const [x, y] = [X[i], Y[i]];
    if (x === undefined || y === undefined) out.push({ ...(x ?? y), index: i });
    else out.push({ ...x, index: i, type: both(x, y), optional: x.optional && y.optional });
  }
  return out;
}

type F = (a: { x: 1 }, b: string) => void;
type G = (a: { y: 2 }) => void;
const merged = intersectParameters(reflect(F).signatures[0].parameters, reflect(G).signatures[0].parameters);
merged.length === 2;
merged[0].type === type { x: 1 } & { y: 2 };
merged[1].type === string;
```

The last extreme challenge and one of the most honest ones: parameter lists have positions, optionality, and rest elements, and merging two of them means handling all three. The TypeScript version reads optionality by three tricks stacked: `(infer A)?` matches a head whether it is required or optional, `'0' extends (keyof X) | (keyof Y)` distinguishes real heads from a bare rest element, and `[[], []] extends [X, Y]` decides whether the merged position keeps its question mark, because a tuple accepts the empty tuple exactly when everything in it is optional. Optionality cannot be read, so it is triangulated. `Intr` is the flattening no-op again (eighth sighting), applied per position.

The builder walks two lists with an index, and this is the one place in the document where the parameter records' `index` field earns its keep rather than costing: the loop assigns positions because positions are what it is computing. Challenge 3196 charged the builder a renumbering tax for the same field, and this is the refund.

## 33345 · Dynamic Route

Next.js route parameters.

```ts
// TypeScript, issue #33631 (+5). Thirty-six lines; the head is shown.
type SlashSplit<T extends string> =
  T extends `` ? [] :
  T extends `/${infer TSingle}` ? [...SlashSplit<TSingle>] :
  T extends `${infer TDouble1}/${infer TDouble2}` ? [TDouble1, ...SlashSplit<TDouble2>] :
  [T];

type PartialRoute<S extends string> = S extends `[[...]]` ? { [K in `...`]?: string } :
  // ... catch-all, optional catch-all, and plain segments follow
```

```js
// Builder
function dynamicRoute(path: string): type {
  const properties = path.split('/').filter(Boolean).flatMap(segment => {
    const inner = /^\[(.+)\]$/.exec(segment)?.[1];
    if (inner === undefined) return [];
    const optional = /^\[(.+)\]$/.exec(inner);
    const name = optional?.[1] ?? inner;
    const rest = name.startsWith('...');
    const key = rest ? name.slice(3) : name;
    if (key === '') return [];
    return [{ ...prop(key, rest ? [].<string> : string), optional: optional !== null }];
  });
  return objectOf(properties);
}

dynamicRoute('/shop') === type {};
dynamicRoute('/shop/[]') === type {};
dynamicRoute('/shop/[slug]') === type { slug: string };
dynamicRoute('/shop/[slug]/[foo]') === type { slug: string, foo: string };
```

The last challenge in the repository, and a good one to end on because it is a real thing people actually want: Next.js has this exact feature and the types for it are written this way in production. `SlashSplit` is `split('/')`, the bracket cases are a regex, and `[[...slug]]` being optional is a flag on a property record.

Both solutions are the same program. One is thirty-six lines of pattern matching and the other is nine lines of string handling, and the honest summary of the whole document is in that ratio: not that TypeScript's authors are doing it wrong, but that they are doing it without `split`, without `Set`, without `===`, without `+`, and without anywhere to put a variable.

## Notes

**Identity was the whole game, and the harness proves it.** Challenge 898 is the clearest result across all twenty. Its expected values are chosen precisely to break assignability-based answers (`boolean` excludes `false`, `{ a }` excludes `{ readonly a }`, `[1]` excludes `1 | 2`), so every passing TypeScript solution carries the `IsEqual` incantation: two generic function signatures whose relation the checker happens to decide the right way, found in a 2018 issue comment and never blessed as a feature. With interned type objects it is `===`, and the challenge collapses to `.some`. The same incantation is what the harness's own `Expect<Equal<X, Y>>` is built from, which is why every case in this document is written as a plain assertion instead: the challenges' tooling is the first thing the builder model deletes, and each of those assertions is also a line that runs. Challenge 898 additionally settles a claim [type programming](../typeprogramming.md) only asserts in the abstract, that `readonly` is where identity and inter-assignability come apart. The harness agrees.

**`boolean` and its literals.** Challenge 268 is decided by whether `boolean` is the union `true | false`. TypeScript models it that way, so `If<boolean, 'a', 2>` distributes to `'a' | 2`. The [type programming](../typeprogramming.md) document admits boolean literal types and defines `arms` to return `[T]` for a non-union, but never says which of these `boolean` is, so `ifType` above carries an explicit line for it. The recommendation is that **`boolean === type true | false`**, interned as the union of its two literal types: it costs nothing, since a two-inhabitant primitive and the union of its two inhabitants are the same set of values; it makes `arms(boolean)` return both literals and deletes the special case; and it keeps `exclude(boolean, type true) === type false` working, which a reader will expect the moment literal booleans exist. The alternative, keeping `boolean` a primitive node, is defensible for layout reasons but must then say so explicitly, because the question is not decidable from the current text.

**Numeric property keys.** Challenge 11's harness expects `TupleToObject<[1, 2]>` to be `{ 1: 1, 2: 2 }` with numeric keys. JavaScript has no numeric property keys: `{ 1: 1 }` and `{ '1': 1 }` are one object, and the node model's `name: string | symbol` says so. `tupleToObject(type [1, 2])` therefore yields `{ '1': 1, '2': 2 }`, whose *values* are still the number literals. This is a TypeScript-ism rather than a gap, and the node model is right to not reproduce it.

**Readonly arrays are a real divergence.** Challenge 9's expected output contains `readonly ['hi', { readonly m: readonly ['hey'] }]`. TypeScript has `readonly T[]` as a *type*; this proposal has `readonly` as a *field modifier* and reaches immutable arrays through frozen instances and value semantics instead, which the comparison table already calls out as a different mechanism. So `deepReadonly` marks fields readonly and recurses through element types, but cannot mark the array itself readonly, and the challenge's tuple cases pass only up to that difference. This is not a builder limitation: no reflection API can construct a modifier the type grammar does not have. If `readonly` on array and tuple types is ever wanted it belongs in the main proposal, and the node model would carry it as a flag on the `array` and `tuple` nodes, which `deepReadonly` would set in the spread it already uses for properties.

**The node model has no generic signatures.** `FunctionSignatureReflection` carries parameters and a return, but no type parameters, so a builder cannot *construct* `<K, V>(key: K, value: V) => R` with `makeType`, and a generic function type cannot round-trip through reflection. Challenge 12 shows why this has not bitten: generic methods are declared in ordinary syntax and builders compute their *arguments*, which is the natural division of labor. Recording it as a known limit rather than an oversight, the open question being whether reflection should carry type parameters for completeness. Nothing in twenty challenges has needed it.

**The published answers rot, and the rot is the finding.** Two of this batch's top-voted solutions fail the repo's current tests, which I checked with TypeScript 5.9.3 against its own `Equal`. Challenge 15's `[any, ...T][T['length']]` (+149) evaluates to `any` for `Last<[]>` where the test wants `never`; challenge 16's `T extends [...infer I, infer _] ? I : never` (+20) gives `never` for `Pop<[]>` where the test wants `[]`. These are not careless answers, and the authors are not the story: the test cases were tightened *after* the answers were posted, and the answers were correct when written. The story is why the empty tuple is where they broke. In a conditional type, the else-branch is doing double duty: it means both "`T` is not a tuple with a last element" and "`T` is the empty tuple", because pattern matching against `[...infer I, infer _]` is the only iteration primitive available and a failed match is the only way to notice either fact. Writing `: never` there looks like marking an impossible branch; it is silently also answering the empty case. Of the nineteen distinct `Pop` answers I tested, twelve return `never` and fail, and the passing ones differ only in writing `: []`. The builder cannot make this mistake, and not because its author is more careful: `tupleElements` throws for a non-tuple and `slice(0, -1)` returns `[]` for the empty one, so the two facts are two code paths and neither can stand in for the other.

Challenge 3062's `Shift` is `Pop` seen in a mirror, and the numbers are worse: fifteen of the twenty published answers return `never` for the empty tuple, including every one with a vote, and the five that pass all write `: []`. Between them, `Pop` and `Shift` are twenty-seven wrong answers to the same question, and the question is not hard. It is that the else-branch of a conditional type has two jobs and the language provides no way to tell them apart.

A third answer rotted for a related reason. Challenge 949's top-voted `AnyOf` (+83) tests `T[number] extends 0 | '' | false | [] | {[key: string]: never}`, and fails now that the harness added `undefined` and `null` to its all-falsy case. The builder is not immune to *that* one, since the falsy set must be enumerated by hand in both languages, and the passing answer is the same union plus two members. But it belongs to the same family: a `never` or a falsy list standing in for a case nobody enumerated, invisible until a test names it. Which points at the shape underneath. Across this batch, `Permutation` ends in `: never` on a branch that cannot be reached, `Merge` ends in `: never` on a branch that cannot be reached, and `Permutation` opens with `K extends K`, a tautology whose only job is to trigger distribution. None of those tokens mean anything; they are there because a conditional type needs an else and a distribution needs a naked parameter. Unreachable code that the language forces you to write is unreachable code nobody checks, and it is exactly where these answers rot.

The next batch supplies the rest of the family. `IsNever` wraps its operand in a tuple to *stop* distribution, since `T extends never` distributes over the empty union and answers `never` instead of `true`. `IsUnion` smuggles a copy of its parameter into a second parameter so it can compare the distributed copy against the fixed one, because whether a conditional distributed is the only way to observe how many arms a union has. `EndsWith` binds an `infer f` it never uses, because a template pattern may not start with an unanchored `${string}`. Each of these is a competent answer to the question the language allows, rather than the question asked, and the reason is always the same: the union arms, the arm count, and the string's tail are all facts the checker knows and offers no way to read. `arms`, `node.kind`, and `endsWith` read them.

**Does a union absorb its subtypes?** Challenge 1097 asks the proposal a question it has not answered. The harness expects `IsUnion<string | 'a'>` to be `false`, which holds in TypeScript because it reduces `string | 'a'` to `string`: the literal is a subtype of `string`, the union of a type with its own subtype is that type, and the arm disappears. The [type programming](../typeprogramming.md) document specifies union canonicalization as flattening, deduplicating, and ordering, where deduplication is by interned identity. Under that text `type string | 'a'` keeps two arms, `isUnion` answers `true`, and the two languages part company.

Both answers are defensible and the choice is not local. Reducing is what a reader expects from interning, since `string | 'a'` and `string` are inhabited by exactly the same values and the promise of `===` is that structurally identical types are one object; declining to reduce means two type objects with identical value sets are different types, which weakens `===` from "same type" to "same spelling". But reducing means canonicalization must run assignability, and assignability is defined coinductively over interned types, so the two become mutually recursive at the exact point where the fixpoint machinery already lives. It also spreads: does `float32.<{ nonZero: true }> | float32` reduce to `float32`, and does a metadata `subtype` hook get consulted during interning? The recommendation is to reduce, and to specify the recursion, because `arms` is the foundation of every union builder in the catalog and its behavior on `string | 'a'` should not be an accident of the order the arms were written in. What matters most is that the document say which, since forty of these fifty challenges lean on `arms` or `union` somewhere.

**Thirty-one of these hundred and ninety top-voted answers no longer pass.** The full list is challenges 9, 12, 15, 16, 20, 191, 898, 949, 2757, 2793, 2946, 3062, 4425, 4484, 5140, 5310, 5317, 9898, 18142, 27133, 27958, 30301, 6, 112, 213, 270, 300, 2822, 13580, 517, and 7561, checked against the repo's current test cases with TypeScript 5.9.3. That is one in six, on a corpus whose whole purpose is to be checked, with hundreds of votes between them. Challenge 2757 is the extreme case: twelve posted answers, eleven of which fail the same `@ts-expect-error` line, because exactly one of the twelve constrains `K extends keyof T`. The last of them fails for a reason worth its own paragraph, below.

**The ceiling has a number, and it is 10,000.** Challenge 27133 asks for `n` squared, and it is the sharpest thing in the document about what the tuple encodings actually cost. The obvious answer builds an `n * n`-element tuple and reads its length. The top-voted answer is a genuinely clever refinement of that: `SplitZeroes` factors trailing zeros out of the input, so `Square<100>` becomes `Square<1>` with `'0000'` glued on the end, and every round number stays small. The harness includes `Square<101>`, and the refinement dies:

```
error TS2799: Type produces a tuple type that is too large to represent.
```

10,201 is over TypeScript's tuple ceiling of 10,000, and `101` has no trailing zeros to factor. So the passing answer implements decimal long multiplication instead: sixteen helper types for digit splitting, digit addition via tuple concatenation, carry extraction, operand padding, right-to-left digit walking, and reassembly through a string. It is excellent work and it exists entirely to get out from under a limit that exists entirely because the language counts with lists. `Square<101>` sits in the harness beside `GreaterThan<1234567891011, 1234567891010>` and `ConstructTuple<1000>`, three cases in three unrelated challenges, all placed to force the same admission. The builder writes `n * n`.

The hard tier has the other kind of ceiling, on recursion depth rather than tuple size, and two challenges are shaped entirely by it. Capitalize Words (112) recurses once per character, and its top-voted answer dies on the harness's fifty-character string with "type instantiation is excessively deep": the recursive call sits inside a template literal, so it is not a tail call, and TypeScript's tail-recursion elimination does not apply. The passing answer threads an accumulator for no reason other than to make the common branch a tail call. Length of String 2 (651) is the same limit met head-on: it is challenge 298 with a longer input and a harder difficulty rating, and the answer is a **manual loop unroll**, matching ten characters at a time and adding ten to the counter, with the one-at-a-time version kept as the remainder loop. That is the optimization a compiler backend performs, written out by hand, in a type, to buy a factor of ten against a limit. `[...s].length` has no limit and iterates code points.

The whole arc is visible in one function asked three times. **Length of String** is challenge 298 at medium, 651 at hard, and 31824 at hard again, and the only thing that changes between them is how long the input is. 298 counts one character at a time. 651 unrolls the loop ten-fold by hand. 31824 gives up on counting entirely and does long division in base ten *by pattern matching*: `Q10` is a template type of ten `string`s in a row, `Q100` is ten of those, up to `Q100k`, and the answer strips off as many hundred-thousands as it can, then ten-thousands, then thousands, hundreds, tens, ones, so that the six remainders are literally the six decimal digits, which the last line concatenates and parses. The `Q1 & {}` in its definition is load-bearing: without the no-op intersection, `` `${string}${string}` `` collapses to `string` and the edifice evaporates. It is an extremely good answer. It is also the only challenge in this document whose tests cannot be checked with a stock compiler: the harness builds a one-million-character string literal type, and `tsc` exits 134 with `JavaScript heap out of memory` unless given `--max-old-space-size=8192`. `[...s].length` is the same three times, runs in microseconds, and iterates code points.

**And one answer fails because it is correct.** Challenge 7561 is subtraction, rated extreme, and its top-voted answer is three lines. The harness reads:

```ts
Expect<Equal<Subtract<1, 1>, 0>>,
Expect<Equal<Subtract<2, 1>, 1>>,
Expect<Equal<Subtract<1, 2>, never>>,
// @ts-expect-error
Expect<Equal<Subtract<1000, 999>, 1>>,
```

The last line asserts an *error*. Not that it is slow, not that it is out of scope: the exercise formally specifies that subtracting nine hundred and ninety-nine from one thousand does not work. That is the same admission as `ConstructTuple<1000>` and `Square<101>`, except this time the answer beats it. Its `Tuple` opens with `0 extends 1 ? never : ...`, a condition that is always false, so the branch is dead and the type means exactly what it would without it. Its only function is to defer instantiation, and I checked what it buys: with the guard, `Subtract<1000, 999>` evaluates to `1`; delete it and change nothing else, and the compiler answers `error TS2589: Type instantiation is excessively deep and possibly infinite`. A no-op conditional is the difference between the answer working and not.

So the answer computes `1`, the `@ts-expect-error` goes unused, `tsc` reports `error TS2578: Unused '@ts-expect-error' directive`, and the top-voted answer fails the test suite. The challenge is `-`. The builder is `m - s`.

**Two of them are traps the harness set on purpose.** Challenge 4425's `GreaterThan` is the sharpest case in the document. The obvious answer builds a tuple of length `N` and compares lengths, and both published tuple answers do exactly that, including the top-voted one. The harness kills them with `GreaterThan<1234567891011, 1234567891010>`, which asks for a tuple of 1.2 trillion elements and gets "type instantiation is excessively deep" instead. That case was added to force a real algorithm, and the only answers that survive it compare decimal digits by substring search. Challenge 4484's `IsTuple` fails on `never` for the reason `IsNever` exists: the top-voted answer omits the `[T] extends [never]` wrapper, so the conditional distributes over the empty union and returns `never` rather than `false`. Both are the same story as `Pop` and `Shift`: the failing answers are the ones a competent person writes first, and the language gives no signal that the case they miss exists.

**The harness's own hack is load-bearing inside the answers.** Ten challenges (898 Includes, 5153 IndexOf, 5317 LastIndexOf, 5360 Unique, 9898 Appear only once, 27958 CheckRepeatedTuple, 30970 IsFixedStringLiteralType, 5181 Mutable Keys, 9384 Maximum, 5 Get Readonly Keys) cannot be solved without type equality, and TypeScript has none, so each published answer imports `Equal` from `@type-challenges/utils`, restates it inline, or reinvents it: challenge 9898's passing answer builds `[U] extends [T]` with both operands tuple-wrapped to suppress distribution, which is a third spelling of the same missing operator.

And then there is challenge 19749, which asks you to implement `Equal` itself, and is the quiet centre of this whole document. Its answer already ships in the repository, in `utils/index.d.ts`, because the harness needs it to grade the other ninety-nine. It has **zero** posted solutions: every other challenge here has answers running from single digits into the hundreds, and this one has none, which makes sense for a puzzle whose answer is quotable from the framework's own source and originates in a 2018 comment about checker internals rather than in anything a person could derive. The mechanism is worth stating once, plainly: `(<T>() => T extends X ? 1 : 2)` is a generic function type nobody will call, two of them are compared for assignability, and the checker settles that comparison by structurally comparing the deferred conditionals inside, which it can only do by comparing `X` and `Y` for identity. The identity relation exists. It is reachable only as an observable side effect of how function assignability happens to be implemented, and it is not specified to work this way. Interned type objects have identity because that is what interning means, and `A === B` is not a solution to a puzzle. It is the operator.

The hard tier adds two more of the same shape, and both are famous. Challenge 223 asks whether a type is `any`, and the answer is `0 extends (1 & T)`: `1 & T` is `1` for nearly everything and `0 extends 1` is false, but `1 & any` is `any` and `0 extends any` is true, so the test is whether a literal survives being intersected. It works because of a special case in intersection reduction that exists for other reasons entirely. Challenge 55 turns a union into an intersection by distributing it into function types, one per arm, and inferring the parameter back out of the whole union at once: the answer is an intersection because parameters are contravariant, so inference in that position collects constraints by intersecting them. Nothing in that chain is about unions or intersections. It is a variance rule, observed indirectly, through a function type nobody calls, to compute a set operation. The builders are `T === any` and a node with `kind: 'intersection'`.

And challenge 55 is not a curiosity: it is infrastructure. Three challenges in the hard tier's second ten depend on it. DeepPick (956) needs to merge several picked paths, the language has no merge, so it distributes over the path union, builds one object per path, and intersects the lot. Union to Tuple (730) builds on it twice over, and is the sharpest case in the document of a challenge asking for something the language does not have. To get a tuple from a union you need an ordering. `LastInUnion` obtains one by wrapping each arm in a function, intersecting the results into an overload set, and inferring the parameter back out, because inference against an overload set picks the *last* signature. The order of the tuple therefore comes from overload resolution order, which comes from the order the intersection's members were written, which comes from the order the union's arms were declared, and none of those three steps is specified. The harness knows: every one of its tests asserts on `['length']` or on the union of the elements, and not one looks at the order. `arms(U)` returns a list, in the order [type programming](../typeprogramming.md) specifies for union canonicalization, and a builder that wants a different order calls `.sort()`.

The fourth use is the one to end on. Challenge 5423 intersects a list of tuples in two lines: normalize each element to an array, then match the whole thing against `{fill: ($: infer I) => any}[]` and read out `I`. It works because `Array.prototype.fill` takes a parameter of the array's element type, and inferring that parameter from every element at once intersects the results, because parameters are contravariant. The function whose signature is being abused is not one the author wrote. It is a method on the standard library's array interface, picked because it happens to mention the element type in the right position. Nothing about `fill` has anything to do with intersecting sets. `sets.every(set => set.has(t))` is the same computation and a reader who has never seen it will still know what it does. That is the `(<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2)` construction: a pair of generic function signatures whose assignability the checker happens to decide by comparing `X` and `Y` for identity, found in a 2018 issue comment, never specified, and now a dependency of both the test framework and the solutions it grades. The failures follow from the same gap, and there are three of them. LastIndexOf's top-voted answer used `L extends U` and answers `5` instead of `4` on `[string, 2, number, 'a', number, 1]`, because `1 extends number`. Appear-only-once's top-voted answer used `F extends R[number]` as a membership test and drops `1` and `2` from `[1, 2, number]` for the same reason. CheckRepeatedTuple's reports a repeat in `[number, 1, string, '1', boolean, true, false, unknown, any]` because `number extends any`. All three read like membership tests and are subtype tests, and CheckRepeatedTuple's passing answer ends up restating the incantation and rebuilding challenge 898's `Includes` on top of it, inside challenge 27958. The builder writes `===` and `findLastIndex`, and gets `-1` for free because that is what `findLastIndex` returns.

The flip side is worth stating in the same breath, since it is the same fact. `new Set` and `Map` work on type objects, because interning makes identity a pointer comparison and type objects hash like strings. `unique` is `[...new Set(elements)]`, `without` is a `Set` and a `filter`, and both keep insertion order because `Set` does. TypeScript's `Unique` needs four workarounds in five lines: a union standing in for a set, distribution standing in for membership, `Equal` standing in for `===`, and `[F]` wrapping each element so the union does not deduplicate the very thing being tested.

**What does a mutable binding infer?** Challenge 10969 ends with two cases that are not about integers at all:

```ts
let x = 1
let y = 1 as const
Expect<Equal<Integer<typeof x>, never>>,   // typeof x is number
Expect<Equal<Integer<typeof y>, 1>>,       // typeof y is 1
```

The challenge is testing TypeScript's literal widening: a `let` initialized from a numeric literal infers `number`, and `as const` is how you keep the literal type. [Type programming](../typeprogramming.md)'s coverage table lists `as const` as obviated by the main proposal's semantics, and the reason given elsewhere is that literals propagate their literal types and value types are immutable by layout. But the table does not say what `let x = 1` infers, and the two readings differ: if a mutable binding kept the literal type `1`, then `x = 2` would not typecheck, so it must widen; a `const` binding has no such pressure and can keep `1`. That is very likely the intent, and it is a better answer than TypeScript's, because the distinction falls out of `let` versus `const` rather than needing an annotation to recover what the checker just threw away. But `Reflect.typeOf(x)` is observable and a builder can branch on it, so the rule belongs in the text rather than in the reader's inference. This is the third question this exercise has raised that the documents imply and do not state, after `boolean === true | false` and union subtype absorption.

**`extends` is two questions, and the code cannot say which one it is asking.** Challenges 9898 and 25170 sit twenty-eight apart and want opposite things from the same operator. In `FindEles`, `F extends R[number]` is a *membership* test and needs identity, so on `[1, 2, number]` the top-voted answer decides `1` is already present, because `1 extends number`, and returns the wrong tuple. In `ReplaceFirst`, `F extends S` is a *match* test and needs assignability, so `ReplaceFirst<[1, 'two', 3], string, 2>` correctly replaces `'two'`. Identical syntax, opposite requirements, no way to write down which was meant, and a reader cannot tell the correct use from the incorrect one without running the tests. The builder writes `types.indexOf(t) === types.lastIndexOf(t)` in the first and `Reflect.isAssignable(t, S)` in the second. That the two relations are separately spellable is the point [type programming](../typeprogramming.md) makes when it says builders pick the one they mean; these two challenges are what it looks like when you cannot.

**Four challenges, one boolean.** Get Required (57), Get Optional (59), Required Keys (89), and Optional Keys (90) all ask the same question: is this property optional? None of them can ask it, and the three published answers find three different ways around:

```ts
T[P] extends Required<T>[P]        // 57, 59: compare a property against its own de-optionalized self
T extends Required<Pick<T, K>>     // 89: pick one key, require it, re-test the whole object
{} extends Pick<T, P>              // 90: pick one key and see if the empty object satisfies it
```

IsRequiredKey (2857) later adds a fourth: `Pick<Type, Keys> extends Required<Pick<Type, Keys>>`. Every one of them constructs a whole new object type to inspect one flag on one property, and the first only works because optionality and `| undefined` are entangled in what an optional property *means*, which is why `GetRequired<{ foo: undefined, bar?: undefined }>` is a test case rather than a curiosity. `TypePropertyReflection` has a field named `optional` and it holds a boolean. All five builders are a `filter`.

The same story runs again for `readonly`. Mutable Keys (5181) has to reach past assignability entirely and use the identity incantation: pick one key, make the picked object `Readonly`, and ask whether that changed anything, because if `Readonly<Pick<T, K>>` is identical to `Pick<T, K>` the key was already readonly. That is the eighth challenge in this document to need `Equal`, and it needs it to read a boolean that is sitting on the property. `TypePropertyReflection` has `readonly` too.

And then there is challenge 5, which is the same question inverted and is rated **extreme**. It sits above the fifty-five hard challenges and the hundred and four medium ones, in the hardest tier the repository has, and it asks which properties are `readonly`. It is extreme because there is no way to ask: the answer imports the identity incantation, builds `Readonly<T>`, picks each key from both versions, and compares the one-key objects for identity, so that "making it readonly changed nothing" stands in for "it was already readonly". Seven challenges across three difficulty tiers (57, 59, 89, 90, 2857, 5181, and 5) are the same `filter` over the same two boolean fields, and the tier they land in is decided entirely by how far the language makes you go to read a flag that is already there.

**One challenge needs a compiler option the repository does not set.** Assert Array Index (925) is built entirely around `noUncheckedIndexedAccess`, and its test cases only typecheck when that flag is on. `tsconfig.base.json` does not turn it on. The challenge's README says to enable it, so this is a documented requirement rather than a bug, but it does mean the challenge cannot be checked by the repository's own configuration, and the answer's sixty-eight lines include yet another digit-to-tuple lookup table, because an index type that stays in bounds has to do arithmetic and the arithmetic has to be built first.

**One challenge does not typecheck against its own tests.** Split (2822) ships a template that declares `type Split<S extends string, SEP extends string> = any`, with two required parameters, and a test file that calls `Split<'Hi! How are you?'>` and `Split<''>` with one. Compiling the template against the tests it comes with produces `error TS2314: Generic type 'Split' requires 2 type argument(s)`. Every passing answer quietly adds a default the template does not have, and the top-voted one copied the template's signature and fails. This is a different failure from the stale answers above: nothing rotted, the two files never agreed.

**Not all rot is the language's fault.** Challenge 2946 broke differently and it is worth separating out, because the previous note would otherwise take credit it has not earned. Nine of `ObjectEntries`' top ten answers fail, including all four highest, and the cause is that the challenge changed its mind: it used to strip `undefined` from an optional property's value type and now preserves it, so every answer carrying a `RemoveUndefined` helper is answering the old question correctly. That is ordinary specification drift. A builder would have rotted identically, since `p.optional ? union([p.type, undefined]) : p.type` is a policy line and the policy flipped. The distinction matters: the `never` family fails because the language forced an unreachable branch that silently answered a real case, and no amount of care avoids it; `ObjectEntries` fails because the goalposts moved, which is a thing that happens to code. Only the first is evidence about type languages.

**Parameter names cannot be part of a function type's identity.** Challenge 191 requires `appendArgument((a: uint32, b: string) => uint32, boolean)` to equal `(a: uint32, b: string, x: boolean) => uint32`. The builder appends a parameter and has no way to know it should be called `x`; under an identity that included parameter names, the challenge would be unpassable by any builder, and TypeScript passes it because function type identity ignores names. So `makeType` must canonicalize parameter names away, while `FunctionParameterReflection` must keep them, since diagnostics, hovers, and `parameters` all want them. That is exactly the shape of the provenance problem [type programming](../typeprogramming.md) already solves for documentation: information that reflection carries and identity ignores. Parameter names belong in the same non-canonical class as `origin`, and the same question follows them, namely what reflection reports after two differently-named signatures intern to one object. The provenance rule (union on merge, construction order for display) is the available answer and should be stated for names too.

**Mapped types over tuples are a special case, and the special case is invisible.** Challenge 20's entire solution rests on `{ [K in keyof T]: ... }` producing a *tuple* when `T` is a tuple, rather than an object with keys `'0' | '1' | '2' | 'length' | 'map' | ...`. Nothing in that syntax says so; it is a rule in the checker that mapped types over array and tuple types are rewritten element-wise. The builder's `switch` on `node.kind` states the same rule in the place where it applies, which is the difference between a language feature you must know and a line you can read.

**Intersections versus flat objects.** Challenge 8 weakens its assertions from `Equal` to `Alike` because `Omit<T, K> & Readonly<Pick<T, K>>` is inter-assignable with the intended object but not identical to it. The builder edits records in place, returns the flat object, and passes the stronger assertion. That the harness had to weaken a test to accommodate the idiomatic TypeScript answer is small but telling: the type language's shape leaks into what the test is allowed to say.

**Constraints versus checks.** These challenges are stated as generic *aliases* with constraints (`MyPick<T, K extends keyof T>`), and the builders are *functions*, so a bad argument is a thrown `TypeError` at the call rather than a constraint violation at the declaration. The two are not far apart: when a builder appears in a signature, the computed constraint form (`<T, K: keysOf(T)>`) restores the declaration-site check, and the thrown message is available in both cases. What the builder gives up is being checkable before its arguments are known; what it gains is saying which property was missing.

**Where TypeScript wins, or draws.** Concat, Push, and Unshift: three of a hundred and ninety, and all three the same win. Variadic tuple spread is purpose-built syntax, `[U, ...T]` says it in six characters, and no reflection call will be shorter. Two more are honest draws. Absolute (529) stringifies and strips a minus in both languages, because the challenge specifies a string result and neither is doing arithmetic. AnyOf (949) is the more interesting one: truthiness is a property of values, but the challenge has no values, so the falsy set is enumerated by hand either way and all that differs is `.some` against recursion. When a problem is genuinely about types rather than about values wearing types, the two models converge, and that is worth knowing about the ones where they do not. The far end of the same scale is MinusOne (2257), where the encoding costs five helper types, a digit lookup table, and three passes over a reversed string to compute `n - 1`, and buys no range for the trouble, since the input literal is a float64 in both languages before either solution starts.

Arithmetic is now the document's largest single theme, and it is worth collecting: MinusOne subtracts one with a digit table and three string passes; FlattenDepth (3243) counts depth with a tuple whose length is the counter; Fibonacci (4182) carries three tuple accumulators and performs addition by concatenating lists whose lengths are the operands; Chunk (4499) and Fill (4518) each keep a tuple counter as private loop state, and Fill additionally threads a boolean latch because there is nowhere to put a local; GreaterThan (4425) compares two numbers by searching for their digits inside `'0123456789'`. Construct Tuple (7544) builds a tuple by recursing once per element, Number Range (8640) produces a range by counting a tuple to `H`, counting another to `L`, and subtracting the two sets, and Count Element Number To Object (9989) stores every count as a tuple in a `Record`, increments with `[...R[F], 0]`, and converts back with `R[K]['length']` at the end. Triangular number (27152) sums one to `n` by concatenating the running total onto itself as a list, so `Triangular<100>` builds a five-thousand-element tuple to hold `5050`. Tower of Hanoi (30430) counts its rings with `Acc extends '🇺🇦'[]`, one Ukrainian flag per level, which is both the friendliest thing in the repository and the clearest statement of the problem: when the element type of your counter can be a flag emoji, the counter is not a number. And Pascal's triangle (30958) represents every number in the triangle as a tuple of that many `unknown`s, sums them with `[...H, ...H2]`, and converts the whole triangle back with a `Lengths` pass at the end; the author's own comment says it plainly: *"I use tuples so I can easily sum them."* And String to Number (300) is the one that says it out loud: to parse `'27'` you implement positional decimal notation, with a nine-entry lookup table whose entries spread the operand that many times, a multiply-by-ten built out of a multiply-by-five and a multiply-by-two because there is no ten-entry row, and a place-value recursion down the string. The hard tier adds four more and does not soften them. Binary to Decimal (6141) is Horner's method with `[...R, ...R]` for the doubling, so `'11111111'` builds a 255-element tuple. Two Sum (8804) enumerates every permutation of the array to reach the quadratically many pairs, because the two loop constructs available are "peel a tuple" and "distribute a union", and permuting is how you turn the first into the second. FizzBuzz (14080) writes divisibility-by-three as a type that removes three elements from a tuple's end until it empties, then writes it again with five `unknown`s, because you cannot pass the three. And Maximum (9384) is the plainest of all: to find the largest of `[1, 20, 200, 150]` it counts a tuple up from zero to two hundred, deleting each number from the union as it passes, and needs `Equal` to notice when one member is left. BitwiseXOR (30575) and Binary Addition (32532) close the column out at the bit level, the latter with eight type parameters, five of which its author marked `// Internals` because they are locals with nowhere to live, and a `NumOfOne` that counts set bits by filtering a tuple and taking its length, called twice because the first result cannot be reused. And then the extreme tier states the conclusion outright. **Sum (476) and Multiply (517) are `+` and `*`**, they are rated extreme, and they are eighty-six and seventy-one lines. Every technique this document has catalogued is in them: digit tables, per-digit carry propagation, `PadWithZeroes` to align the operands, `TrimZeros` to clean up after, a nine-by-nine multiplication table, long multiplication accumulating partial products with trailing zeros for place value. The work is good and correct. Both signatures take `string | number | bigint` and return a string, which is the tell: the subject is the representation, not the operation, because the answers are for numbers too large to be a `number`. JavaScript shipped `BigInt` in 2020. Twenty-three challenges, one cause, and the last two are the ones to remember. Sort (741) decides `7 <= 9` by building a seven-element tuple and a nine-element tuple and peeling them in lockstep, once per comparison, from scratch. And Subtract (7561) is `-`, rated extreme, three lines. The builder answers are `n - 1`, `d - 1`, a `for` loop, `slice`, `i >= start && i < end`, `a > b`, `Array.from({ length: n })`, `Array.from({ length: high - low + 1 })`, a `Map` with `+ 1`, `n * n`, `n * (n + 1) / 2`, `prev[j - 1] + prev[j]`, `Number(s)`, `parseInt(s, 2)`, `a + b === target`, `v % 3 === 0`, `Math.max`, `[...x].map((c, i) => c === y[i] ? '0' : '1')`, `carry = sum >> 1`, `BigInt(a) + BigInt(b)`, `BigInt(a) * BigInt(b)`, `.sort((a, b) => a - b)`, and `m - s`. This proposal's value generics make the arithmetic that the tuple encodings simulate available directly, and the encodings are not a failure of the people writing them; they are what a type language without numbers requires.

Two genuine taxes on the builder are worth recording against all this. FlipArguments (3196) reverses a parameter list, and parameter records carry an `index`, so the builder must renumber them; TypeScript's `Reverse<P>` over a tuple has no such field to keep consistent. Richer reflection means more invariants to maintain, and that bill will arrive in larger amounts than this one. Trunc (5140) is the other: `Trunc<'-.3'>` is `'-0'`, and the obvious builder returns `'0'`, because `Math.trunc(-0.3)` is negative zero and `String(-0)` drops the sign. `Object.is` fixes it in one call, but the TypeScript answer never had the problem, since it does string surgery and never touches a number. JavaScript's semantics arrive with JavaScript's expressiveness, and negative zero is in the box. The pattern across the set is consistent. Where TypeScript has syntax for an operation on types, it beats a builder; where it must *encode* a computation (identity as a signature relation, iteration as recursive peeling, a duplicate-key check as a parameter that becomes `never`, a base case as `keyof T extends never`, whitespace as a hand-enumerated union of three characters), the encoding is precisely what the builder deletes.
