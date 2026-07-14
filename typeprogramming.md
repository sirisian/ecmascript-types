# Type Programming with Type Builders

Erased type systems that want computed types (a type derived from another type's keys, a partial view of a shape, the parameters of a function) grow a second language to compute them in: conditional types, mapped types, inference binders, template-literal grammars, all evaluated inside the checker. This proposal does not grow one. Types here are [first-class interned values](typeobjects.md); the compiler already contains a [compile-time evaluator](typeobjects.md), since `meta` protocol hooks, `where` clauses, operator return types like `float32.<multiplyDimensions(D, D2)>`, and `componentType(C)`-style annotations all run user functions during compilation; and **an expression that evaluates to a type object at compile time is a valid type annotation**. Type programming is therefore ordinary JavaScript: a ***type builder*** is a compile-time evaluable function that takes type objects (and constants), inspects them with reflection, and returns a new type object.

```js
import { mapProperties } from 'std:types';

function partial(T: type): type {
  return mapProperties(T, p => ({ ...p, optional: true }));
}

type User = { id: uint32, name: string };
type Draft = partial(User);        // { id?: uint32, name?: string }
let d: Draft = { name: 'a' };
let e: partial(User) = {};         // a builder call is a type annotation
```

This document specifies the small API delta that makes builders able to construct any type the language can express, the evaluation semantics that make them well-defined on recursive types and unbound generics, the inference rules that let them participate safely, and the standard library, `std:types`, that covers in ordinary code the ground erased systems cover with bespoke type-level languages. Each library entry below includes a usage example and, collapsed, the TypeScript construction it replaces, since that is the reference point most readers will bring.

One discipline governs every mechanism in this document, and it is what keeps a Turing-complete type language trustworthy:

> **Builders may propose; the engine must verify.** User code never makes a checker decision the engine takes on faith. Every checker-visible effect of type code is one of three shapes: *declarative data* the checker reads (never executes), a *proposal* the checker confirms by ordinary forward evaluation and assignability, or a *contract* that is assumed early only because every concrete instantiation re-checks it. The inference fixpoint itself never runs a builder.

A lying builder can make compilation fail; it cannot make compilation wrongly succeed.

## Construction: `Reflect.makeType`

Reflection is currently read-only: [decorators.md](decorators.md) defines `Reflect.getReflection.<Reflect.Type>(t)`, which cracks a type object into a plain node discriminated by `kind`. `Reflect.makeType(node)` is its inverse. It takes a node (or any object matching a node's shape, since the input is structural), canonicalizes it under the same rules [interning](typeobjects.md) already uses (unions flattened, de-duplicated, deterministically ordered; nested types expanded to canonical form), and returns the interned type object. The round-trip law:

```js
Reflect.makeType(Reflect.getReflection.<Reflect.Type>(T)) === T; // for every T
```

Canonicalization is where invalidity is caught: a node describing an impossible value-type layout, an intersection of distinct value types, or an index signature with an illegal key type is a `TypeError` from `makeType`, with the same message the equivalent source declaration would produce. Canonical identity is order-independent where the type constructor is, so properties, arms, and members intern the same regardless of construction order, while reflection and display preserve construction order, because source order is meaningful to humans.

Type objects also gain a canonical `toString` (the canonical source form: `String(type 'a' | 'b')` is `"'a' | 'b'"`), because builders that throw authored diagnostics need to print types.

### The completed reflection model

Round-tripping every type requires completing the node model. The revised `Reflect.TypeReflection`, a superset of the one in [decorators.md](decorators.md), with additions marked:

```js
namespace Reflect {
  type TypeReflection =
    | { kind: 'primitive'; type: type; generic: GenericApplication | undefined; } // generic added
    | { kind: 'literal'; value: string | number | boolean | bigint | symbol;     // added: 'a', 42, true, 5n, and
        base: type; }                                                             //   symbol literal types (below)
    | { kind: 'union'; arms: [].<type>; }                                         // arms: [] is never (below)
    | { kind: 'intersection'; members: [].<type>; }
    | { kind: 'tuple'; elements: [].<TypeTupleElement>; }
    | { kind: 'array'; element: type; extent: uint32 | undefined; }
    | { kind: 'object'; properties: [].<TypePropertyReflection>;
        indexSignatures: [].<TypeIndexSignature>; }                               // indexSignatures added
    | { kind: 'function'; signatures: [].<FunctionSignatureReflection>; }
    | { kind: 'parameterized'; base: type; metadata: object; }                    // added: metadata parameterizations,
    | { kind: 'reference'; name: string; };                                       //   over any base type

  type TypeTupleElement = { type: type; rest: boolean; initial: any | undefined; };   // initial added: [T, U = d]
  type TypePropertyReflection = {
    name: string | symbol;
    type: type;
    optional: boolean;
    readonly: boolean;                    // added, mirroring readonly fields
    initial: any | undefined;             // added: the a?: T = value default
    origin: DeclarationOrigin | undefined; // added: provenance; non-canonical, see below
  };
  type TypeIndexSignature = { key: type; value: type; };                          // added
  type GenericApplication = { base: type; arguments: [].<type | any>; };          // added: Map.<string, uint8> →
}                                                                                 //   { base: Map, arguments: [string, uint8] }
```

Notes on the additions, each of which exists because some type the grammar can write was previously unreachable or unrebuildable:

- **`literal`** carries the value and its base primitive, and **symbol literal types are admitted**: a `const s = Symbol()` used in type position is the type inhabited by exactly that symbol, identity-compared like every other literal value. This is what makes `keysOf` total over symbol-keyed shapes and closes the `unique symbol` gap without a keyword. Symbols sort after strings in canonical order, by first-interning order within the agent; a type mentioning a unique symbol is agent-local under [serialization](serialization.md) (the serializer names the symbol), while `Symbol.for` keys round-trip by key.
- **`generic` on `primitive`** exposes a nominal generic application's arguments: `Map.<string, uint8>` reflects with `generic: { base: Map, arguments: [string, uint8] }`. This is the direct accessor that replaces most uses of an `infer` binder: there is no other way to reach inside `Promise.<T>` in erased systems, and here there is.
- **`parameterized`** expresses [primitive metadata](primitivemetadata.md) parameterizations structurally, and the base is *any* type, not only a primitive, which is what makes branding a library function (see `brand` below). The metadata protocol's `subtype`/`validate` hooks decide relations between parameterizations; the default relation is that `parameterized(T, M)` is assignable to `T` while `T` is not assignable to `parameterized(T, M)` except through the metadata construction boundary. Metadata never changes representation.
- **`readonly`, `initial`, `indexSignatures`, tuple `initial`** exist in the type grammar (readonly fields, `a?: T = []` optional defaults, `[T, U = d]` trailing tuple defaults, index signatures) and merely weren't surfaced. Their presence is what makes shape-transforming builders *homomorphic by default*: a builder copies property records and edits only the field it means to change, so modifiers it never mentions survive.
- **`origin`** is provenance: a reference to the declaring site, explicitly **non-canonical**. Identity ignores it; interning *unions* it when structurally identical types merge; spreading a record preserves it. `DeclarationOrigin` is an opaque host reference to a declaration site (module and span). It exists so documentation survives the round trip (see the provenance section), and it is deliberately not a payload: type identity must never depend on comments.
- Constructed nodes never contain `reference` nodes; cycles in *construction* come from the fixpoint mechanism below, and `reference` remains what a reader sees when walking an already-cyclic type.

Two completions land beside the type nodes. `FunctionParameterReflection` gains `rest: boolean` (a rest parameter was previously indistinguishable from a plain one), and `FunctionSignatureReflection` gains two optional, declarative fields the checker *reads* and never executes:

```js
type FunctionSignatureReflection = {
  parameters: [].<FunctionParameterReflection>;
  return: FunctionReturnReflection;
  thisType: type | undefined;                                    // added: what the signature demands of its receiver
  narrows: { parameter: uint32; to: type; asserts: boolean } | undefined; // added: declared narrowing, applied post-call
};
```

A function literal checked against an expected signature adopts that signature's `thisType` for body checking (arrow functions excluded; their `this` stays lexical), which is how an options-object API gives its methods a typed `this` without any checker-resident marker. A call whose signature declares `narrows` applies the declared narrowing afterwards, exactly as `is` and `instanceof` do; the declared fact is also runtime-checkable at the boundary. Polymorphic `this`, the fluent-builder return type that rebinds per receiver, is *not* covered by `thisType` and remains a separate feature: it is an implicitly generic self type, a checker feature in its own right.

Finally, a `Reflect.ClassConstructor` reflection context joins the declaration-reflection family, returning a class's construct signatures in the same `FunctionSignatureReflection` shape: `ClassReflection` carries `name`, `type`, `abstract`, and `metadata`, and every member kind had a context except the constructor.

## Relations: identity, assignability, and `never`

**Identity is `===`.** Interning makes structural equality pointer equality (nominal for classes and enums), which is both the memoization key for builders and the type-equality operator erased systems lack.

**Assignability is a predicate.** The checker's coinductive assignability judgment, the one it runs on every assignment, including across recursive types, is exposed as an evaluable function:

```js
Reflect.isAssignable(source: type, target: type): boolean;
```

This is what conditional logic branches on: where an erased system writes `T extends U ? X : Y`, a builder writes `Reflect.isAssignable(T, U) ? X : Y`. Builders that mean *equivalence up to subtyping* compose it both ways; builders that mean identity use `===`. The two genuinely differ (for instance across `readonly`), and code states which one it means.

**`never` is the empty union.** `Reflect.makeType({ kind: 'union', arms: [] })` returns it, `Reflect.never` names it, and it is writable in annotations with the standard algebra: identity of union (`union([T, never]) === T`), annihilator of intersection, assignable to every type, uninhabited. The checker already possessed this type internally, since divergence is typed with it, so this is naming, not new theory. Admitting it changes nothing about exhaustiveness: `switch` exhaustiveness stays reserved to enums and sealed classes, exactly as the [main proposal](README.md) specifies; a union of literals is still not a switchable closed set.

## Pattern matching over types

For the rare shape question that reflection accessors don't answer directly, matching an abstract pattern against an unknown structure, a one-sided structural unifier is provided:

```js
const K = Reflect.inferSlot('K'), V = Reflect.inferSlot('V');
const m = Reflect.matchType(pattern, subject);   // bindings object, or null
if (m) { /* m.K and m.V are the bound type objects */ }
```

Patterns are built of nodes and slots and are **structural only**: a deferred builder application is not a legal pattern, because evaluating a builder against an unbound slot is precisely the inversion no engine can perform. Constrained slots are an `isAssignable` check on the binding afterwards. In practice the standard library below never needs `matchType`; it exists for completeness and as the tool with which a builder author writes an explicit inverse (see declared inverses).

## Evaluation semantics

**When.** A builder call in type position evaluates at specialization, when every type and value generic parameter it reads is bound. This is the existing [compile-time type expression](typeobjects.md) rule, with type generic parameters named alongside value generics. At module top level, where there are no unbound parameters, calls evaluate during compilation of the module, before any of its code runs.

**Memoization by interning.** A call is keyed by (function identity, argument tuple): type arguments compare by `===`, primitive value arguments by SameValue (the identical rule [value generics](generics.md) use), and compound compile-time constants (`pick(User, ['id', 'name'])` passes a fresh array at every occurrence) canonicalize into the key by structural value, the semantics the [composites](README.md#composites) extension gives tuples and records. Each distinct key evaluates once per program; every subsequent occurrence, in any module, yields the same interned type object, so `partial(User) === partial(User)` everywhere.

**Deferral.** An application whose arguments are not yet bound, such as `partial(T)` inside a generic declaration, is itself carried as an *interned deferred type* under the same identity rule: two mentions of `partial(T)` in one generic body are one type before anyone knows `T`. Before specialization the checker knows a deferred application's identity and its declared kind (`type`) and, absent a contract (below), nothing finer; in particular it is treated as invariant in its parameters. Every use in a running program reaches concrete arguments, where checking has full precision.

**Recursion: the in-flight fixpoint.** When evaluation of `f(args)` re-enters `f` with an *identical* key before the outer call returns, the inner call immediately returns a fresh *placeholder type object* instead of recursing. When the outer call returns node `N`, the placeholder resolves to the canonicalization of `N`, producing exactly the named-back-edge cyclic type that `type Tree = { value: uint32, children: [].<Tree> }` produces from syntax, validated by the same rule: every cycle must pass through a reference position, or `TypeError`. A call whose entire result *is* its own placeholder (`function f(T) { return f(T); }`) is an unproductive-recursion `TypeError`. This mechanism is what makes `deepPartial` below work on recursive types with zero API surface, and it is the computational twin of the coinductive assignability the proposal already specifies.

**Purity.** Builders are compile-time evaluable functions, so the static discipline applies: a body reads only its parameters, constants, and other evaluable functions. *Local* mutation (a `Set` of visited types, an accumulator) is fine; module-level mutable state is not evaluable and is rejected statically, which also disposes of hand-rolled memo caches, since the engine's interning memoization is the supported mechanism and observably equivalent. `Date.now`, `Math.random`, I/O, and ambient reads are unnameable by construction, so evaluation is deterministic.

**Budget.** Evaluation carries a fuel budget (spec'd default, host-overridable) covering steps and constructed-type count per top-level type-position evaluation. Exhaustion is a compile-time `TypeError` naming the outermost call and the deepest frames. One rule is load-bearing: **fuel decides whether compilation fails, never what it produces.** Budget exhaustion is uncatchable from evaluated code; a builder that could catch it and return a fallback type would make a program's types depend on host fuel settings: the same source elaborating differently on a CI box and a laptop, with the memo table caching whichever answer came first. Graceful degradation is written deterministically instead: explicit structural caps (`paths(T, ...)` below carries its own cycle policy), spec'd-value depth parameters, `*Bounded` variants.

Interactive tooling gets one further allowance, scoped so it cannot leak into semantics: **cancellation, never truncation**. Because builders are pure and memoized, an evaluator can checkpoint at memo boundaries, yield, resume, or abandon an in-flight evaluation at will, so an editor keystroke never waits out a deep recursion. What a language server does with an abandoned evaluation is display-side only: hover falls back to the unexpanded application form, completion omits members not yet computed, the request re-runs against the memo table a moment later. What it may not do is feed a partial result back into checking or report diagnostics the build would not report. The editor is permitted to be *behind* the build, never *different from* it.

**Authored errors.** A builder may `throw` (typically `TypeError`), and the thrown message becomes the compile-time diagnostic, prefixed with the originating type-position expression per the existing diagnostics rule. Misuse of a type-level API produces error text its author wrote:

```js
throw new TypeError(`pick: ${String(T)} has no property '${String(key)}'`);
```

**Dual use.** A builder is an ordinary function. Called at runtime with runtime type objects (from `Reflect.typeOf`, or values of type `type`), it does the same thing and returns the same interned objects. Type-level code therefore gets unit tests in a normal test runner, `console.log` debugging, coverage, and profiling. This is a structural consequence of the type language being *the* language.

## Type positions, inference, and generic bodies

**Placement.** A builder call may appear everywhere a type may: variable and field annotations, parameter and return types, generic argument lists (`Map.<string, partial(User)>`), the right side of `is` and `as`, and alias declarations (`type Draft = partial(User)`).

**Computed constraints, and literal inference.** A generic constraint may be a compile-time call over *earlier* parameters in the same list, evaluated left to right, and a parameter constrained to a union of literal types infers *literally* from its argument:

```js
function pluck<T, K: keysOf(T)>(o: T, key: K): indexed(T, K) {
  return o[key];
}
const name = pluck(user, 'name'); // K = 'name', the literal type, not string; return type is indexed(User, 'name')
```

`keysOf(T)` evaluates once `T` binds (inferred from `o`); the resulting union both checks `key` and cues the literal inference, the same cue `K extends keyof T` gives erased checkers.

**Inference sites, and suppressing them.** Inference reads *plain* parameter positions. A computed position, meaning any builder application, is opaque until its arguments bind, so it is not an inference site. That single rule yields inference suppression for free, as an identity builder:

```js
import { noInfer } from 'std:types';

function update<T>(original: T, patch: noInfer(T)): void { ... }
update({ a: 1 }, { a: 2, b: 3 });
// T infers from `original` only ⇒ T = { a: float64 }; `patch` is then checked against noInfer(T) = T
```

**Contracts: `where` postconditions on builders.** The function-level `where` clause of [dependent record types](dependentrecordtypes.md) extends to postconditions: inside a `where` clause, `return` names the returned value (a position where the token is otherwise ungrammatical). On a builder, a postcondition is a fact about the type it returns:

```js
export function omit(T: type, keys: [].<string | symbol>): type
    where Reflect.isAssignable(T, return)   // dropping properties widens: every T value is a result value
{ ... }
```

The semantics are two-sided. **Verified:** at every concrete evaluation the clauses are checked against the actual result; a violation is a compile-time `TypeError` naming the builder and its arguments. Contracts are never trusted. **Assumed:** before specialization, the checker takes the clauses as known facts about the deferred application. That is sound precisely *because* of the verification side, since any instantiation that would falsify the assumption is stopped at the builder. This recovers definition-site checking of generic bodies in bounded form:

```js
function sanitize<T>(value: T): omit(T, ['password']) {
  return { ...value };   // admitted at the declaration: the contract supplies T <: omit(T, keys)
}
```

Direction matters. A *lower* bound on the result (`T` assignable to `return`) serves the **producer** above; an *upper* bound (`return` assignable to `partial(T)`, say) serves **consumers** holding the result. The standard library ships with contracts on every builder for which useful bounds hold. What contracts cannot express is universally quantified knowledge, such as variance of `omit(T, K)` in `T`, or parametricity, because a contract is a per-evaluation proposition; that knowledge simply does not exist before specialization, and after specialization it isn't needed. A companion `@exemplars(TestUser, Point)` decorator forces compile-time specialization of a generic declaration, turning "test your library's generics" into a checked annotation.

**Casts on deferred types.** Beneath contracts sits the escape hatch the proposal already owns. `value as omit(T, ['password'])` inside a generic body is legal with a deferred application as its target: the cast is carried like any other unevaluated type and resolves at specialization into the proposal's ordinary cast: discharged statically when the concrete relation holds, a *checked* conversion with a diagnostic naming the cast when it does not. A wrong assertion is bounded to a `TypeError` at the cast, never silent unsoundness.

**Trial specialization over closed sets.** When a generic parameter appears *only* under deferred applications, ordinary inference has nothing to bind it from. But if its constraint denotes a closed, finite set of types (a union of types, an enum of types, or a [`sealed`](README.md) class with its fixed direct-subclass set), the engine evaluates forward per candidate instead of inverting:

```js
sealed class Shape { }
class Circle extends Shape { kind: 'circle' = 'circle'; radius: float64; }
class Rect extends Shape { kind: 'rect' = 'rect'; width: float64; height: float64; }

function apply<T: Shape>(patch: partial(T)): T { ... }

apply({ kind: 'circle', radius: 3 });
// trials: <: partial(Circle) ✓ ; <: partial(Rect) ✗ (kind conflicts) ⇒ T = Circle
```

Candidates are enumerated in declaration order; exactly one survivor infers, zero or several is a compile error asking for explicit `.<T>`. Trials are **separable**: a parameter is trialed against exactly the arguments whose computed types mention it alone, so two independent parameters cost a sum of trials, never a Cartesian product, and memoization makes repeated candidates table lookups. An argument whose type mentions two or more un-inferred parameters does not trial; that site asks for explicit arguments. The whole phase runs under a **spec-fixed trial ceiling**, fixed rather than host-tunable because whether inference succeeds must not vary by machine, past which the site bails to explicit arguments with a diagnostic naming the constraint. Trial selection pairs with literal freshness: an object literal checked against a candidate application is checked exactly (excess properties rejected), without which all-optional candidates could not discriminate.

**Declared inverses.** For the residual signature shape (a parameter typed by a builder application, no plain occurrence of the parameter, no closed candidate set), a builder may declare its own inverse, as an oracle whose every answer is checked. The binding lives on the declaration ([functions are decoratable](decorators.md), with a `FunctionMetadata` slot), not in any registry, so it has one possible author and no dependence on module evaluation order:

```js
@inverse(required)
export function partial(T: type): type { ... }

function apply<T>(patch: partial(T)): T { ... }
const user = apply({ name: 'a' });
// 1. ordinary inference for T fails (T occurs only under partial)
// 2. propose: T₀ = inverse({ name: string }) = { name: string }
// 3. verify forward: { name: string } <: partial(T₀) ✓ ⇒ T = T₀; on ✗, fall through to "explicit .<T> required"
```

The engine consults a declared inverse only after ordinary inference and any trial has failed, calls it once per site (memoized, budgeted), and then performs the same forward evaluation and assignability check an explicitly specialized call would face. A wrong inverse can cause inference to fail, never to succeed wrongly. Builders with several inferable slots propose via a record keyed by parameter name, verified jointly.

## The standard library: `std:types`

Everything below is written against `Reflect.makeType`, `Reflect.getReflection.<Reflect.Type>`, `Reflect.isAssignable`, and `Reflect.never`. No entry is engine magic, and shipping the library *as source* (final home to be reconciled with [standardlibrary.md](standardlibrary.md)) is the proof. A codebase can polyfill any of it. Because builders intern, one shared vocabulary matters: `partial(User)` is the same type object in every module that imports the same `partial`.

```js
import {
  reflect, never, literal, union, arms, literalValues, prop, objectOf, fn, tupleOf, arrayOf,
  mapProperties, keysOf, indexed,
  partial, required, readonly, mutable, pick, omit, record, pickByValue,
  exclude, extract, nonNullable,
  flatten, firstParameter, returnType, parameters, constructorParameters, awaited,
  mapLiterals, uppercase, lowercase, capitalized, uncapitalized,
  head, tail, concat, reverse, zip,
  deepPartial, deepMap, paths,
  discriminants, byKind, handlers,
  mapUnion, compose,
  noInfer, brand, stringPattern, prefixed, suffixed,
  withThisType, options, thisParameterType, omitThisParameter,
} from 'std:types';
```

Every entry is a compile-time evaluable function usable in type position and at runtime; examples show both the definition and a use. Library builders carry `where` contracts (elided below except where instructive) and the internal string helper `const capitalize = (s: string): string => s.charAt(0).toUpperCase() + s.slice(1);` is module-private.

### Core constructors and accessors

The foundation: reflect a type, and build the basic forms.

```js
export function reflect(T: type): Reflect.TypeReflection {
  return Reflect.getReflection.<Reflect.Type>(T);
}
export const never: type = Reflect.never;

export function literal(value: string | number | boolean | bigint | symbol): type {
  return Reflect.makeType({ kind: 'literal', value, base: Reflect.typeOf(value) });
}
export function union(armList: [].<type>): type {
  return Reflect.makeType({ kind: 'union', arms: armList }); // flattens, dedupes, sorts; [] → never, [T] → T
}
export function arms(T: type): [].<type> {
  const node = reflect(T);
  return node.kind === 'union' ? node.arms : [T];
}
export function literalValues(T: type): [].<string | number | boolean | bigint | symbol> {
  return arms(T).map(arm => {
    const node = reflect(arm);
    if (node.kind !== 'literal') throw new TypeError(`expected a union of literals, got ${String(arm)}`);
    return node.value;
  });
}
export function prop(name: string | symbol, type: type,
    { optional = false, readonly = false, initial = undefined } = {}): Reflect.TypePropertyReflection {
  return { name, type, optional, readonly, initial, origin: undefined }; // minted: no provenance
}
export function objectOf(properties: [].<Reflect.TypePropertyReflection>,
    indexSignatures: [].<Reflect.TypeIndexSignature> = []): type {
  return Reflect.makeType({ kind: 'object', properties, indexSignatures });
}
export function fn(parameterTypes: [].<type>, returnType: type): type {
  return Reflect.makeType({ kind: 'function', signatures: [{
    parameters: parameterTypes.map((type, index) => ({ type, name: `p${index}`, index, rest: false, initial: undefined, metadata: {} })),
    return: { type: returnType, metadata: {} }
  }] });
}
export function tupleOf(types: [].<type>): type {
  return Reflect.makeType({ kind: 'tuple', elements: types.map(type => ({ type, rest: false, initial: undefined })) });
}
export function arrayOf(element: type, extent: uint32 | undefined = undefined): type {
  return Reflect.makeType({ kind: 'array', element, extent });
}
```

```js
type Level = union([literal('debug'), literal('info'), literal('warn')]); // 'debug' | 'info' | 'warn'
literalValues(Level);                                                     // ['debug', 'info', 'warn']
type Point = objectOf([prop('x', float32), prop('y', float32)]);
Point === type { x: float32, y: float32 };                                // true, interned
type Pair = tupleOf([string, uint32]);                                    // [string, uint32]
type Names = arrayOf(string);                                             // [].<string>
type Grid = arrayOf(float32, 16);                                         // [16].<float32>
type Handler = fn([string], type void);                                   // (string) => void
```

<details>
<summary>TypeScript reference</summary>

These forms are written syntactically in TypeScript (`'debug' | 'info' | 'warn'`, `{ x: number, y: number }`, `[string, number]`, `string[]`, `(s: string) => void`) but cannot be *constructed from computed parts*: there is no expression that builds a union from an array of computed literal types, or an object type from a computed property list. That constructor role is played indirectly by mapped types, conditional types, and template-literal types, which the rest of this library replaces one for one.

</details>

### `mapProperties`: the shape-transform workhorse

The builder analog of a homomorphic mapped type: walk an object type's property records and rebuild from what the callback returns. Returning the record unchanged preserves everything (type, `optional`, `readonly`, defaults, provenance); spreading it and overriding one field edits exactly that field; returning `null` deletes the property. Distribution over a union is a visible line, and an intersection maps member-wise:

```js
export function mapProperties(T: type,
    f: (p: Reflect.TypePropertyReflection) => Reflect.TypePropertyReflection | null): type {
  const node = reflect(T);
  if (node.kind === 'union') return union(node.arms.map(arm => mapProperties(arm, f)));
  if (node.kind === 'intersection')   // A & B: map each member; equivalent up to assignability to mapping the flattened shape
    return Reflect.makeType({ kind: 'intersection', members: node.members.map(m => mapProperties(m, f)) });
  if (node.kind !== 'object') throw new TypeError(`mapProperties expects an object type, got ${String(T)}`);
  return objectOf(node.properties.map(f).filter(p => p !== null), node.indexSignatures);
}
```

Index signatures pass through untouched: the modifier builders below edit declared properties only, and a builder that means to include signatures walks them explicitly, as `deepPartial` does. Homomorphism is not a special status a construct earns; it is the default consequence of copying a record, and stripping a modifier from a property you never mentioned requires writing the code that strips it.

```js
const cap = (s: string): string => s.charAt(0).toUpperCase() + s.slice(1);
type Settings = { volume: uint8, theme: string };
type SettingsListeners = mapProperties(Settings,
  p => prop(`on${cap(p.name)}Changed`, fn([p.type], type void)));
// { onVolumeChanged: (uint8) => void, onThemeChanged: (string) => void }
```

<details>
<summary>TypeScript reference</summary>

```ts
type Listeners<T> = { [K in keyof T & string as `on${Capitalize<K>}Changed`]: (value: T[K]) => void };
```

A mapped type with key remapping and a template-literal key. Homomorphic modifier preservation in TypeScript is a special status the mapped type earns by iterating `keyof T` in the right syntactic form.

</details>

### Keys and indexing: `keysOf`, `indexed`

`keyof` exists as a [built-in](typeobjects.md) over syntactic type expressions; `keysOf` is the same operation over a type object held in a variable, and doubles as the specification of `keyof` on index signatures (the signature's key type joins wholesale: `keysOf` of `{ [key: string]: T }` includes `string` itself), intersections (union of the members' keys), and unions (keys common to every arm). With symbol literal types admitted, symbol-keyed members participate like any others.

```js
export function keysOf(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'object': return union([
      ...node.properties.map(p => literal(p.name)),
      ...node.indexSignatures.map(s => s.key),
    ]);
    case 'intersection': return union(node.members.map(keysOf));
    case 'union':        return node.arms.map(keysOf).reduce((acc, next) => extract(acc, next)); // common keys
    default: throw new TypeError(`keysOf: ${String(T)} has no keys`);
  }
}

export function indexed(T: type, K: type): type {   // the T[K] of erased systems, as a function
  return union(arms(T).flatMap(t => {
    const node = reflect(t);
    return literalValues(K).map(key => {
      const p = node.properties.find(p => p.name === key);
      if (!p) throw new TypeError(`${String(t)} has no property '${String(key)}'`);
      return p.optional ? union([p.type, type undefined]) : p.type;  // policy: optional access admits undefined
    });
  }));
}
```

```js
type User = { id: uint32, name: string, email?: string };
type Key = keysOf(User);            // 'id' | 'name' | 'email'
type NameT = indexed(User, type 'name');            // string
type EmailT = indexed(User, type 'email');          // string | undefined, from the policy line above

function pluck<T, K: keysOf(T)>(o: T, key: K): indexed(T, K) { return o[key]; }
```

The `undefined`-on-optional-access decision erased systems gate behind a compiler flag is a readable policy line inside `indexed`; a codebase wanting the other policy writes the other line. `typeof x` needs no builder: `Reflect.typeOf(x)` in type position is the type query.

<details>
<summary>TypeScript reference</summary>

```ts
type Key = keyof User;                                 // 'id' | 'name' | 'email'
type NameT = User['name'];                             // indexed access
declare function pluck<T, K extends keyof T>(o: T, key: K): T[K];
type Common = keyof (A | B);                           // keyof of a union: the common keys
type All = keyof (A & B);                              // keyof of an intersection: the union of keys
type Sig = keyof { [k: string]: unknown };             // string | number
```

</details>

### Modifier builders: `partial`, `required`, `readonly`, `mutable`

Modifier arithmetic is spread-and-override; each is one line, including the `-readonly` form that erased systems support in grammar but never shipped as a utility.

```js
export function partial(T: type): type  { return mapProperties(T, p => ({ ...p, optional: true  })); }
export function required(T: type): type { return mapProperties(T, p => ({ ...p, optional: false })); }
export function readonly(T: type): type { return mapProperties(T, p => ({ ...p, readonly: true  })); }
export function mutable(T: type): type  { return mapProperties(T, p => ({ ...p, readonly: false })); }
```

```js
type User = { id: uint32, name: string, readonly createdAt: string };
type Draft = partial(User);      // { id?: uint32, name?: string, readonly createdAt?: string }; readonly survives
let d: Draft = { name: 'a' };
type Frozen = readonly(User);
type Thawed = mutable(Frozen);
type Complete = required(Draft);
partial(type User | Account) === union([partial(User), partial(Account)]); // distribution, visible in mapProperties
```

<details>
<summary>TypeScript reference</summary>

```ts
type Partial<T>  = { [P in keyof T]?: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
type Readonly<T> = { readonly [P in keyof T]: T[P] };
type Mutable<T>  = { -readonly [P in keyof T]: T[P] };   // grammar supports it; lib.d.ts never shipped it
```

`Partial<A | B>` distributes because homomorphic mapped types do: an invisible rule that becomes a visible `union` case in the builder.

</details>

### Selection and remapping: `pick`, `omit`, `record`, `pickByValue`

Overloads accept either a union-of-literals key type or a plain key array, whichever reads better at the call. Misuse produces an authored diagnostic instead of an opaque constraint failure.

```js
export function pick(T: type, K: type): type { return pick(T, literalValues(K)); }
export function pick(T: type, keys: [].<string | symbol>): type
    where Reflect.isAssignable(T, return)
{
  const wanted = new Set(keys);
  const have = new Set(literalValues(keysOf(T)));
  for (const key of wanted) if (!have.has(key))
    throw new TypeError(`pick: ${String(T)} has no property '${String(key)}'`);
  return mapProperties(T, p => wanted.has(p.name) ? p : null);
}
export function omit(T: type, K: type): type { return omit(T, literalValues(K)); }
export function omit(T: type, keys: [].<string | symbol>): type
    where Reflect.isAssignable(T, return)
{
  const dropped = new Set(keys);
  return mapProperties(T, p => dropped.has(p.name) ? null : p);
}
export function record(K: type, V: type): type {
  const node = reflect(K);
  if (node.kind === 'literal' || node.kind === 'union' && node.arms.every(a => reflect(a).kind === 'literal'))
    return objectOf(literalValues(K).map(name => prop(name, V)));   // literal keys → concrete properties
  return objectOf([], [{ key: K, value: V }]);                      // open keys → an index signature
}
export function pickByValue(T: type, V: type): type {
  return mapProperties(T, p => Reflect.isAssignable(p.type, V) ? p : null);
}
```

```js
type Credentials = pick(User, ['id', 'name']);
type Sanitized  = omit(User, ['password']);
type Flags      = record(type 'read' | 'write', boolean);   // { read: boolean, write: boolean }
type Scores     = record(string, uint32);                    // { [key: string]: uint32 }
type Strings    = pickByValue(User, string);                 // only the string-typed members

// Key remapping is ordinary string computation on the property record:
const cap = (s: string): string => s.charAt(0).toUpperCase() + s.slice(1);
type PointGetters = mapProperties(type { x: float32, y: float32 },
  p => prop(`get${cap(p.name)}`, fn([], p.type), { readonly: true }));
// { readonly getX: () => float32, readonly getY: () => float32 }
```

<details>
<summary>TypeScript reference</summary>

```ts
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
type Record<K extends keyof any, T> = { [P in K]: T };
type PickByValue<T, V> = { [K in keyof T as T[K] extends V ? K : never]: T[K] };
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type RemoveKind<T> = { [K in keyof T as Exclude<K, 'kind'>]: T[K] };   // remap-to-never deletes a key
```

`PickByValue` composes a conditional type, an indexed access, distribution, and the never-deletes-a-key rule inside a remap clause: four mechanisms for "keep the property if its type fits." The builder is `filter` by another name.

</details>

### Union filters: `exclude`, `extract`, `nonNullable`

Filtering a union is `filter`, with `Reflect.isAssignable` as the test and `never` as the empty result. Distribution does not exist here as implicit behavior. In erased systems it is a well-known foot-gun, where whether a conditional maps over union arms depends on the operand being a "naked" parameter, and suppressing it needs a tuple-wrapping idiom. Here the map over arms either appears in the code or it doesn't.

```js
export function exclude(T: type, U: type): type {
  return union(arms(T).filter(arm => !Reflect.isAssignable(arm, U)));
}
export function extract(T: type, U: type): type {
  return union(arms(T).filter(arm => Reflect.isAssignable(arm, U)));
}
export function nonNullable(T: type): type {
  return exclude(T, type null | undefined);
}
```

```js
type Shape = Circle | Rect | Line;
type NotLine = exclude(Shape, Line);
type Events = extract(type 'click' | 'focus' | 42, string);   // 'click' | 'focus'
type Present = nonNullable(string | null | undefined);        // string
exclude(string, string) === never;                            // the empty union is a real type

// Both distribution intents are literal; nothing to suppress:
function toArrayEach(T: type): type { return union(arms(T).map(arm => arrayOf(arm))); } // string | uint8 → [].<string> | [].<uint8>
function toArrayAll(T: type): type  { return arrayOf(T); }                              // string | uint8 → [].<string | uint8>
```

<details>
<summary>TypeScript reference</summary>

```ts
type Exclude<T, U> = T extends U ? never : T;   // distributes over T
type Extract<T, U> = T extends U ? T : never;
type NonNullable<T> = T & {};

type ToArray<T> = T extends any ? T[] : never;        // distributes: string | number → string[] | number[]
type ToArrayAll<T> = [T] extends [any] ? T[] : never; // tuple-wrapped to suppress distribution
```

</details>

### Function and class introspection: `flatten`, `firstParameter`, `returnType`, `parameters`, `constructorParameters`

Most uses of an `infer` binder are destructuring: reaching a component the language gives no other accessor for. Reflection *is* the accessor.

```js
export function flatten(T: type): type {
  const node = reflect(T);
  return node.kind === 'array' ? node.element : T;
}
export function firstParameter(F: type): type {
  const node = reflect(F);
  if (node.kind !== 'function') return never;
  return union(node.signatures.map(s => s.parameters[0]?.type ?? never));
}
export function returnType(F: type): type {
  const node = reflect(F);
  if (node.kind !== 'function') throw new TypeError(`returnType: ${String(F)} is not a function type`);
  const returns = node.signatures.map(s => s.return.type);
  return returns.length === 1 ? returns[0] : union(returns);
}
export function parameters(F: type): type {
  const [signature] = reflect(F).signatures;
  return Reflect.makeType({ kind: 'tuple',
    elements: signature.parameters.map(p => ({ type: p.type, rest: p.rest, initial: p.initial })) });
}
export function constructorParameters(C: type): type {
  const { signatures } = Reflect.getReflection.<Reflect.ClassConstructor, C>();
  return Reflect.makeType({ kind: 'tuple',
    elements: signatures[0].parameters.map(p => ({ type: p.type, rest: p.rest, initial: p.initial })) });
}
```

```js
type F = (name: string, count?: uint32 = 1) => Promise.<User>;
type R = returnType(F);        // Promise.<User>
type A0 = firstParameter(F);   // string
type P = parameters(F);        // [string, uint32 = 1]; the default carries into the tuple form
type Row = flatten([].<User>); // User

function build<C>(...args: constructorParameters(C)): C {
  return new C(...args);       // a class's type object is its constructor, so a specialized C constructs
}
const player = build.<Player>(10, 20);
```

On an overloaded function, `returnType` takes the union of the overloads' returns; a codebase preferring the last-overload rule erased systems apply writes `returns.at(-1)`: overload policy is a decision the code states. `InstanceType` has no entry because it has no job here: there is no constructor-type / instance-type split, so the class name *is* the instance type and you already hold it.

<details>
<summary>TypeScript reference</summary>

```ts
type Flatten<T> = T extends (infer U)[] ? U : T;
type FirstParameter<F> = F extends (x: infer A, ...rest: any[]) => any ? A : never;
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
type ConstructorParameters<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: infer P) => any ? P : never;
type InstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;
```

On overloads, TypeScript's `infer R` resolves to the *last* overload's return type, a rule users discover by surprise.

</details>

### `awaited`

The other large `infer` family reaches inside nominal generics (`Promise<infer U>`), which the `generic` field answers directly; the structural-thenable half walks the `then` signature by reflection.

```js
export function awaited(T: type): type {
  const node = reflect(T);
  if (node.kind === 'union') return union(node.arms.map(awaited));
  if (node.kind === 'primitive' && node.generic?.base === Promise)
    return awaited(node.generic.arguments[0]);                    // Promise.<V> → recurse on V
  const then = node.kind === 'object' && node.properties.find(p => p.name === 'then');
  if (then) {                                                     // structural thenable: unwrap onfulfilled's first parameter
    const onfulfilled = reflect(then.type).signatures[0]?.parameters[0];
    return onfulfilled ? awaited(firstParameter(onfulfilled.type)) : never;
  }
  return T;
}
```

```js
type A = awaited(Promise.<Promise.<uint32>>);        // uint32
type B = awaited(uint32 | Promise.<string>);         // uint32 | string
async function all<T>(ps: [].<T>): Promise.<[].<awaited(T)>> { ... }
```

<details>
<summary>TypeScript reference</summary>

```ts
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    (F extends (value: infer V, ...args: infer _) => any ? Awaited<V> : never) :
  T;
```

The recursion is the same shape; what disappears is encoding "reach the callback's first parameter" as nested conditional inference.

</details>

### Finite string types: `mapLiterals`, `uppercase`, `lowercase`, `capitalized`, `uncapitalized`

Over unions of literals, which is what event maps, key prefixes, and label transforms actually operate on, string type computation is string code. Cross products are a `flatMap`.

```js
export function mapLiterals(T: type, f: (string) => string): type {
  return union(literalValues(T).map(v => literal(f(String(v)))));
}
export function uppercase(T: type): type     { return mapLiterals(T, s => s.toUpperCase()); }
export function lowercase(T: type): type     { return mapLiterals(T, s => s.toLowerCase()); }
export function capitalized(T: type): type   { return mapLiterals(T, capitalize); }
export function uncapitalized(T: type): type {
  return mapLiterals(T, s => s.charAt(0).toLowerCase() + s.slice(1));
}
```

```js
type Kind = type 'click' | 'focus';
type EventName = mapLiterals(capitalized(Kind), k => `on${k}`);  // 'onClick' | 'onFocus'
type Loud = uppercase(Kind);                                     // 'CLICK' | 'FOCUS'

type V = type 'top' | 'middle' | 'bottom';
type H = type 'left' | 'center' | 'right';
type Anchor = union(literalValues(V).flatMap(v =>
  literalValues(H).map(h => literal(`${v}-${h}`))));             // the nine 'top-left' … 'bottom-right'
```

<details>
<summary>TypeScript reference</summary>

```ts
type EventName<K extends string> = `on${Capitalize<K>}`;
type Loud<T extends string> = Uppercase<T>;                      // with Lowercase / Capitalize / Uncapitalize intrinsics
type Anchor = `${'top' | 'middle' | 'bottom'}-${'left' | 'center' | 'right'}`;
```

</details>

### Infinite string types: `stringPattern`, `prefixed`, `suffixed`

`` `${number}px` `` names an *infinite* set of strings; no finite union of literals expresses it, and no new structural type kind is added for it. A refined string is a [primitive metadata](primitivemetadata.md) parameterization whose pattern comes from the [regexp](regexp.md) extension: `validate` matches at assignment and runtime; `subtype` decides pattern-vs-pattern assignability. That judgment is tiered deterministically: for patterns inside the genuinely regular fragment (no backreferences, no lookaround) and within a spec-fixed automaton size bound, language inclusion is decided *exactly* (complement, product, emptiness); beyond the bound or the fragment, a conservative syntactic containment applies. Because the tier is decided by syntactic size rather than remaining fuel, the judgment is identical on every host. This makes pattern types strictly stronger than concatenation-grammar template types on their own turf, with full regex power and real runtime enforcement besides.

```js
export function stringPattern(regex: RegExp): type {
  return Reflect.makeType({ kind: 'parameterized', base: string, metadata: { pattern: regex } });
}
export function prefixed(prefix: string): type {
  return stringPattern(new RegExp(`^${RegExp.escape(prefix)}`));
}
export function suffixed(suffix: string): type {
  return stringPattern(new RegExp(`${RegExp.escape(suffix)}$`));
}
```

```js
type Px = stringPattern(/^\d+px$/);
let margin: Px = '12px';                 // validated on assignment, and at runtime boundaries
let bad: Px = '12em';                    // TypeError, at compile time here, from the validate hook
type Handler = prefixed('on');           // the `on${string}` of template-type systems
Reflect.isAssignable(Px, stringPattern(/px$/)); // true, decided exactly on the regular fragment
```

<details>
<summary>TypeScript reference</summary>

```ts
type Px = `${number}px`;
type Handler = `on${string}`;
```

Template-literal types validate nothing at runtime and cap out at concatenation grammar; pattern-vs-pattern assignability is a structural cross-match over that grammar.

</details>

### Computed types from values: parsing in type position

Because value generics bind from arguments and builders are real code, anything that *parses* (routes, format strings, query fragments) computes its types with the parser you would write anyway:

```js
function routeParams(path: string): type {
  return objectOf(path.split('/')
    .filter(segment => segment.startsWith(':'))
    .map(segment => prop(segment.slice(1), string)));
}

function get<P: string>(path: P, handler: (params: routeParams(P)) => Response): void { ... }

get('/users/:id/posts/:postId', ({ id, postId }) => { ... }); // id: string, postId: string, from the actual route
```

`P` is a value generic bound implicitly from the first argument ([generics.md](generics.md)); the handler's parameter type is computed from the route string at the call. The version every router actually wants, typed segments like `'/users/:id(uint32)'` mapping `id` to `uint32`, is a lookup table away, because this is a program, not an encoding.

<details>
<summary>TypeScript reference</summary>

```ts
type RouteParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param]: string } & RouteParams<`/${Rest}`>
    : Path extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

declare function get<P extends string>(path: P, handler: (params: RouteParams<P>) => Response): void;
```

The community's canonical hard template-literal exercise; typed segments make it a research project.

</details>

### Tuples: `head`, `tail`, `concat`, `reverse`, `zip`

Tuple surgery on reflection nodes is array methods. Named tuple elements are the intersection form `[T, U] & { x: 0, y: 1 }` in this proposal, manipulated with the same tools.

```js
const elementTypes = (T: type): [].<type> => reflect(T).elements.map(e => e.type); // module-private

export function head(T: type): type    { return elementTypes(T)[0] ?? never; }
export function tail(T: type): type    { return tupleOf(elementTypes(T).slice(1)); }
export function concat(A: type, B: type): type { return tupleOf([...elementTypes(A), ...elementTypes(B)]); }
export function reverse(T: type): type { return tupleOf(elementTypes(T).toReversed()); }
export function zip(A: type, B: type): type {
  const a = elementTypes(A), b = elementTypes(B);
  return tupleOf(a.slice(0, Math.min(a.length, b.length)).map((t, i) => tupleOf([t, b[i]])));
}
```

```js
type T = [string, uint32, boolean];
type H = head(T);              // string
type Rest = tail(T);           // [uint32, boolean]
type Both = concat(T, type [float32]); // [string, uint32, boolean, float32]
type Z = zip(type [string, uint32], type ['a', 'b']); // [[string, 'a'], [uint32, 'b']]
```

<details>
<summary>TypeScript reference</summary>

```ts
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type Tail<T extends any[]> = T extends [any, ...infer R] ? R : never;
type Concat<A extends any[], B extends any[]> = [...A, ...B];
type Reverse<T extends any[]> = T extends [infer H, ...infer R] ? [...Reverse<R>, H] : [];
type Zip<A extends any[], B extends any[]> =
  A extends [infer AH, ...infer AR] ? B extends [infer BH, ...infer BR]
    ? [[AH, BH], ...Zip<AR, BR>] : [] : [];
```

</details>

### Type-level arithmetic

Not a library entry, because there is nothing to build: value generics are numbers and compile-time expressions over them are already evaluable, being the same evaluation `float32.<multiplyDimensions(D, D2)>` performs in [primitive metadata](primitivemetadata.md). Fixed-size shape algebra follows as a corollary:

```js
function concatArrays<A: uint32, B: uint32, T>(a: [A].<T>, b: [B].<T>): [A + B].<T> { ... }
function reshape<R: uint32, C: uint32, T>(m: [R * C].<T>): [R].<[C].<T>> { ... }
```

<details>
<summary>TypeScript reference</summary>

```ts
type BuildTuple<N extends number, T extends unknown[] = []> =
  T['length'] extends N ? T : BuildTuple<N, [...T, unknown]>;
type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]['length'];
```

Addition by materializing tuples and reading their lengths: integer-only, with limits near a thousand elements and painful subtraction. Multiplication for `reshape` is out of practical reach.

</details>

### Recursive builders: `deepPartial`, `deepMap`, `paths`

A recursive builder is a recursive function; the in-flight fixpoint (evaluation semantics above) ties legal cycles automatically. `deepPartial` states its domain in a `switch`, which quietly fixes what ad-hoc erased definitions get wrong by accident: functions and class instances pass through instead of being mapped into partial method bags, and metadata parameterizations pass through the `default` arm untouched:

```js
export function deepPartial(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'object':
      return objectOf(
        node.properties.map(p => ({ ...p, optional: true, type: deepPartial(p.type) })),
        node.indexSignatures.map(s => ({ ...s, value: deepPartial(s.value) })));
    case 'array': return arrayOf(deepPartial(node.element), node.extent);
    case 'tuple': return Reflect.makeType({ ...node, elements: node.elements.map(e => ({ ...e, type: deepPartial(e.type) })) });
    case 'union': return union(node.arms.map(deepPartial));
    default:      return T; // primitives, literals, functions, classes, enums, parameterized: pass through
  }
}

export function deepMap(T: type, leaf: (type) => type): type {   // the traversal, parameterized by a type function
  const node = reflect(T);
  switch (node.kind) {
    case 'object': return objectOf(node.properties.map(p => ({ ...p, type: deepMap(p.type, leaf) })), node.indexSignatures);
    case 'array':  return arrayOf(deepMap(node.element, leaf), node.extent);
    case 'union':  return union(node.arms.map(arm => deepMap(arm, leaf)));
    default:       return leaf(T);
  }
}
```

On a cyclic input the fixpoint does the knot-tying:

```js
type Category = { name: string, parent: Category | null };
type Draft = deepPartial(Category);
// evaluation: deepPartial(Category) marks its key in-flight; walking `parent` re-enters with the identical
// key and receives a placeholder; the outer call builds { name?: string, parent?: ⟨placeholder⟩ | null };
// the placeholder resolves to the canonicalized result: precisely the type this declares by hand:
type ByHand = { name?: string, parent?: ByHand | null };
Draft === ByHand; // true
```

`paths` is recursion that *grows*: on a cyclic type its result is genuinely infinite, so the library version carries its cycle policy explicitly and throws an authored diagnostic rather than running into fuel:

```js
export function paths(T: type, ancestors: Set.<type> = new Set()): type {
  const node = reflect(T);
  if (node.kind !== 'object') return never;
  if (ancestors.has(T)) throw new TypeError(`paths: ${String(T)} is recursive; its path set is unbounded`);
  const below = new Set([...ancestors, T]);   // on-stack detection: same type in sibling branches is fine
  return union(node.properties.filter(p => typeof p.name === 'string').flatMap(p => {
    const nested = paths(p.type, below);
    const suffixes = nested === never ? [] : literalValues(nested);
    return [literal(p.name), ...suffixes.map(rest => literal(`${p.name}.${rest}`))];
  }));
}
```

```js
type Config = { server: { host: string, port: uint16 }, debug: boolean };
type Path = paths(Config);     // 'server' | 'server.host' | 'server.port' | 'debug'
function getPath<T, P: paths(T)>(o: T, path: P): any { ... }
getPath(config, 'server.port');   // checked against the computed union
paths(Category);                  // TypeError: paths: Category is recursive; its path set is unbounded
```

<details>
<summary>TypeScript reference</summary>

```ts
type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;
// (community form; also maps over methods and mangles class instances, having no way to state its domain)

type Paths<T> = T extends object
  ? { [K in keyof T & string]: T[K] extends object ? K | `${K}.${Paths<T[K]>}` : K }[keyof T & string]
  : never;
```

Recursive aliases are kept finite by instantiation-depth limits; a cyclic input to `Paths` fails with "type instantiation is excessively deep and possibly infinite": the same wall, without the option of an authored message.

</details>

### Discriminated unions: `discriminants`, `byKind`, `handlers`

Discriminated unions themselves are main-proposal territory (tag fields; `sealed` hierarchies get real exhaustiveness). What computes on top of them is the handler-table shape:

```js
export function discriminants(T: type, tag: string = 'kind'): [].<string> {
  return arms(T).map(arm => {
    const p = reflect(arm).properties.find(p => p.name === tag);
    if (!p) throw new TypeError(`discriminants: ${String(arm)} lacks a '${tag}' discriminant`);
    return literalValues(p.type)[0];
  });
}
export function byKind(T: type, k: string, tag: string = 'kind'): type {
  return extract(T, objectOf([prop(tag, literal(k))]));
}
export function handlers(T: type, R: type, tag: string = 'kind'): type {
  return objectOf(discriminants(T, tag).map(k => prop(k, fn([byKind(T, k, tag)], R))));
}
```

```js
type Shape =
  | { kind: 'circle', radius: float64 }
  | { kind: 'rect', width: float64, height: float64 };

type CircleOnly = byKind(Shape, 'circle');
let area: handlers(Shape, float64) = {
  circle: c => Math.PI * c.radius ** 2,
  rect: r => r.width * r.height,
}; // every key required ⇒ structurally total, without touching the rule that switch
   // exhaustiveness stays reserved to enums and sealed classes; totality lives in the object type
```

A generic `match(value, table)` body still faces the correlated-union step, proving `table[value.kind]` accepts *this* `value`, per instantiation, as every structural checker does; the builders compute the *shape*, which is the part that was previously inexpressible.

<details>
<summary>TypeScript reference</summary>

```ts
type Handlers<T extends { kind: string }, R> =
  { [K in T['kind']]: (value: Extract<T, { kind: K }>) => R };

function match<T extends { kind: string }, R>(value: T, handlers: Handlers<T, R>): R {
  return handlers[value.kind as T['kind']](value as any); // the correlation cast, as usual
}
```

</details>

### Higher-order type functions: `mapUnion`, `compose`

Builders are functions, so type functions are first-class: composable, partially applicable, storable. Closures qualify as evaluable when everything they capture is, so a composed pipeline is itself a compile-time value usable in type position. This is the higher-kinded-types capability erased type languages structurally lack.

```js
export function mapUnion(T: type, f: (type) => type): type { return union(arms(T).map(f)); }
export function compose(...fs: [].<(type) => type>): (type) => type {
  return T => fs.reduceRight((result, f) => f(result), T);
}
```

```js
const looseDraft = compose(partial, mutable);   // a stored, first-class type function
type Editable = looseDraft(User);
type AllDrafts = mapUnion(Shape, deepPartial);
type Stringly = deepMap(Config, () => string);  // every leaf becomes string: a traversal taking a type function
```

<details>
<summary>TypeScript reference</summary>

```ts
// No equivalent: a type alias cannot be passed to a type alias. The nearest encoding (fp-ts-style):
interface URItoKind<A> { Array: A[]; Promise: Promise<A>; }
type URIS = keyof URItoKind<any>;
type Kind<F extends URIS, A> = URItoKind<A>[F];
declare function lift<F extends URIS>(F: F): <A, B>(f: (a: A) => B) => (fa: Kind<F, A>) => Kind<F, B>;
// ...plus module augmentation to register each new constructor
```

</details>

### Inference control: `noInfer`

The identity builder. Because computed positions are not inference sites, wrapping a parameter type in `noInfer` checks the argument without inferring from it. That is the whole feature, in one line of library code.

```js
export function noInfer(T: type): type { return T; }
```

```js
function update<T>(original: T, patch: noInfer(T)): void { ... }
update({ a: 1 }, { a: 2, b: 3 }); // T = { a: float64 } from `original` only; `patch` merely checked
```

<details>
<summary>TypeScript reference</summary>

```ts
declare function update<T>(original: T, patch: NoInfer<T>): void; // NoInfer is a compiler intrinsic
```

</details>

### Branding: `brand`

A brand is a metadata parameterization, which the generalized `parameterized` node supports over any base. The default relation is the branding rule: the branded type is assignable to its base, while the base reaches the branded type only through the metadata construction boundary, where `validate` runs. A brand is therefore checked, visible to reflection, and can carry conversion semantics (`conversionFactor`) when it is a unit rather than a mere tag. Interning is what makes it a real newtype: `brand(uint32, 'UserId')` is one type everywhere it is written, in any module, without a registry. True nominal *minting*, a builder declaring a class, enum, or `sealed` hierarchy, is deliberately excluded: nominal identity is declaration identity, and the thing brands are minted *for* did not need minting.

```js
export function brand(T: type, tag: string | symbol): type {
  return Reflect.makeType({ kind: 'parameterized', base: T, metadata: { brand: tag } });
}
```

```js
type UserId = brand(uint32, 'UserId');
function getUser(id: UserId) { ... }
getUser(7);            // TypeError: uint32 is not assignable to brand(uint32, 'UserId')
getUser(UserId(7));    // through the metadata construction boundary: checked, not asserted
let n: uint32 = UserId(7); // fine: upcasting sheds the brand
```

<details>
<summary>TypeScript reference</summary>

```ts
declare const userIdBrand: unique symbol;
type UserId = number & { readonly [userIdBrand]: true };  // a lie told to the checker: no such property exists
const id = 7 as UserId;                                   // and an unchecked cast to mint one
```

</details>

### `this` typing: `withThisType`, `options`, `thisParameterType`, `omitThisParameter`

The `thisType` signature field is constructed data; these are its library face. `options` is the options-object pattern, where methods receive a `this` that is the eventual instance shape, with the checker's sole involvement being the contextual-adoption rule stated with the node model.

```js
export function withThisType(F: type, Self: type): type {
  const node = reflect(F);
  return Reflect.makeType({ ...node, signatures: node.signatures.map(s => ({ ...s, thisType: Self })) });
}
export function options(Data: type, Methods: type): type {
  const self = Reflect.makeType({ kind: 'intersection', members: [Data, Methods] });
  return objectOf([
    prop('data', fn([], Data)),
    ...reflect(Methods).properties.map(p => prop(p.name, withThisType(p.type, self))),
  ]);
}
export function thisParameterType(F: type): type {
  return reflect(F).signatures[0].thisType ?? any;   // erased systems say unknown; the runtime-checked any is the analog
}
export function omitThisParameter(F: type): type {
  const node = reflect(F);
  return Reflect.makeType({ ...node, signatures: node.signatures.map(({ thisType, ...s }) => s) });
}
```

```js
type CounterOptions = options(
  type { count: uint32 },
  type { increment: () => void, report: () => string });

let counter: CounterOptions = {
  data: () => ({ count: 0 }),
  increment() { this.count += 1; },          // this: { count: uint32 } & { increment ..., report ... }
  report() { return `at ${this.count}`; },
};
```

<details>
<summary>TypeScript reference</summary>

```ts
type ObjectDescriptor<D, M> = {
  data?: D;
  methods?: M & ThisType<D & M>;             // ThisType is a checker-behavior marker with no structural content
};
type ThisParameterType<T> = T extends (this: infer U, ...args: never) => any ? U : unknown;
type OmitThisParameter<T> = unknown extends ThisParameterType<T> ? T
  : T extends (...args: infer A) => infer R ? (...args: A) => R : T;
```

</details>

## Type equality

Erased systems have no type-equality operator, and their ecosystems use an incantation that exploits checker internals. Here identity is interned `===`, and equivalence-up-to-subtyping is mutual assignability: two spellings for two things that genuinely differ (for instance across `readonly`), so code states which one it means:

```js
A === B;                                                  // structural identity (nominal for classes and enums)
Reflect.isAssignable(A, B) && Reflect.isAssignable(B, A); // inter-assignable
```

<details>
<summary>TypeScript reference</summary>

```ts
type IsEqual<A, B> =
  (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;
```

An artifact of how the checker relates generic signatures, whose behavior has shifted across releases.

</details>

## Provenance: documentation through the round trip

A builder that cracks `User` open and rebuilds it as `partial(User)` does not cost the editor its memory of where `name` came from. The `origin` field on property records is the channel: a reference to the declaring site, **non-canonical**. Identity ignores it, interning *unions* it when structurally identical types merge, and record-spreading preserves it, so every homomorphic builder carries provenance with zero effort. Hover on a `partial(User)` value's `.name` resolves the doc comment and deprecation notice from `User.name`'s origin at display time; a property a builder *mints* (`getters`-style remapping) has no origin, and tooling falls back to the answer the diagnostics rule already mandates: the builder application that produced it. Documentation itself never enters the node model: anything inside a node rides the round trip into `makeType`, and type identity must never depend on comments. A `jsdoc: string` field would either fork identical types by their comments or silently crown one declaration's docs for a shared object. For dependencies without sources on disk, the expansion artifact below carries extracted doc strings for the origins it mentions.

## Tooling and distribution

**Display.** A computed type displays as the application *plus* its canonical expansion (`omit(User, ['password'])` = `{ id: uint32, name: string }`); go-to-definition on a computed type targets the builder, and budget or thrown-error diagnostics carry the builder stack. Interactive evaluation is cancellable, never truncated, as the evaluation-semantics section specifies.

**The evaluator is an interpreter target, not an embedded engine.** Every tool that answers "what is the type here" evaluates builders, using the same evaluator [metadata](primitivemetadata.md) hooks and `where` clauses already require. That does not mean a native-language toolchain embeds a JavaScript engine: the evaluable subset is tiny *as a language*, with no I/O, no ambient state, no `eval`/`Function`, no `async`, no generators, and no proxies; straight-line functions, closures over evaluable bindings, the immutable core library surface, and regular expressions (which the pattern-subtype procedure requires anyway). The subset's grammar and semantics are specified normatively as an implementation target with a conformance suite, so a Rust or Go language server implements a small tree-walking evaluator and *proves* agreement with the engine, the determinism rules being what make that a checkable claim. Memoization means an interactive session evaluates each distinct application once.

**The expansion artifact.** Because evaluation is deterministic and results are interned canonical forms, a package can ship its expansion table: the serialized interning table for every *concrete* type its public surface produces, with closed builder applications pre-evaluated, keyed by a hash of the module graph, with extracted doc strings for provenance origins. A consumer toolchain verifies the hash and reads instead of evaluating, re-deriving on mismatch. Open applications (`partial(T)` inside an exported generic) necessarily evaluate at the consumer against the consumer's `T`; the artifact takes the common path off the evaluator, it does not replace it.

**Interop with erased declarations.** The library above is, in effect, a translation table for `.d.ts` files: homomorphic mapped type → `mapProperties`; distributive conditional → `arms().filter`; `infer` destructuring → a reflection accessor; utilities → their entries here. A converter targets it mechanically in one direction; in the other, a `.d.ts` emitter *prints* declarations for erased-system consumers by evaluating the public surface and writing out expansions, which is the same machinery as the artifact. The residue that does not translate is small and named: declaration merging and global augmentation (non-goals of this proposal), and inference-inverted signatures without a declared inverse, which convert but require explicit specialization at call sites.

## Limits

Stated plainly, because a design is trusted for what it admits it cannot do.

**Universally quantified knowledge does not exist before specialization.** A contract is a per-evaluation proposition, re-verified at every instantiation; variance of `omit(T, K)` in `T`, or parametricity of a builder, is not expressible as one and is not otherwise available. Deferred applications are invariant in their parameters until specialization, where everything is concrete and checked with full precision. This is the trade full specialization made before builders existed, softened by contracts and `@exemplars`, never removed.

**Inference through an opaque result needs a declared route.** When a generic parameter occurs only under builder applications, inference proceeds by trial over a closed constraint set or a declared, verified inverse; absent both, the call site writes explicit `.<T>`, noisier than reverse mapped types but never wrong. Coupled arguments (a type mentioning two or more un-inferred parameters) always require explicit arguments.

**No nominal minting, no merging.** Builders construct structural types and metadata parameterizations; classes, enums, and `sealed` hierarchies come only from declarations, and declarations do not merge. `brand` covers what fake-nominal wrappers are minted for.

**Polymorphic `this`** is an implicitly generic self type, a checker feature of its own that is not derivable from `thisType`, and remains a separate proposal item.

**Termination is bounded, not guaranteed.** Builders are deliberately Turing-complete; the budget is the answer, with the two determinism rules (uncatchable fuel; cancellation, never truncation) keeping it out of the language's observable semantics.

## Coverage

The reference map from erased-system constructs to this document's mechanisms:

| Construct | Here | Where |
|---|---|---|
| `keyof T` | built-in `keyof`; `keysOf` over computed operands | Keys and indexing |
| `typeof x` | `Reflect.typeOf` | Keys and indexing |
| Indexed access `T[K]` | `indexed(T, K)` | Keys and indexing |
| Mapped types, `±?`/`±readonly`, key remapping (`as`, delete-via-`never`) | `mapProperties` record edits; computed names; `null` deletes | `mapProperties`; selection |
| Homomorphic modifier preservation | the default: records are copied | `mapProperties` |
| `Partial`/`Required`/`Readonly`/(`Mutable`) | one line each | Modifier builders |
| `Pick`/`Omit`/`Record` | `pick`/`omit`/`record`, with authored errors | Selection |
| Conditional `T extends U ? X : Y` | `Reflect.isAssignable(T, U) ? X : Y` | Relations |
| Distribution, and suppressing it | explicit `arms().map`/`filter` | Union filters |
| `Exclude`/`Extract`/`NonNullable` | `exclude`/`extract`/`nonNullable` | Union filters |
| `infer` (destructuring; generic arguments; abstract patterns) | reflection accessors; the `generic` field; `Reflect.matchType` | Introspection; pattern matching |
| `ReturnType`/`Parameters`/`ConstructorParameters` | reflection (`ClassConstructor` context) | Introspection |
| `InstanceType` | obviated: the class *is* the instance type | Introspection |
| `Awaited` | `awaited` | `awaited` |
| Template literals over literal unions; the four intrinsics | string code; `mapLiterals` + case builders | Finite string types |
| Template literals over infinite sets | pattern metadata, exact on the regular fragment | Infinite string types |
| Tuple surgery, variadic spreads | array methods on elements | Tuples |
| Type-level arithmetic | arithmetic over value generics | Type-level arithmetic |
| Recursive type functions | the in-flight fixpoint | Recursive builders |
| Discriminated-union computation | `discriminants`/`byKind`/`handlers` | Discriminated unions |
| Type equality (`IsEqual`) | `===` / mutual `isAssignable` | Type equality |
| Higher-kinded / type-level lambdas | first-class functions | Higher-order |
| `NoInfer` | `noInfer`, the identity builder | Inference control |
| `ThisType`, `ThisParameterType`, `OmitThisParameter` | `thisType` field + one contextual rule; library one-liners | `this` typing |
| Assertion functions (`asserts x is T`, `x is T`) | declarative `narrows` on signatures | The completed reflection model |
| `unique symbol` | symbol literal types | The completed reflection model |
| Nominal branding | `brand` over generalized parameterization | Branding |
| Inference through a utility (`f<T>(p: Partial<T>)`) | trials over closed sets; declared verified inverses; else explicit | Type positions and inference |
| Definition-site generic body checking | checked `where` contracts + `@exemplars` + deferred casts | Type positions and inference |
| `satisfies`, `as const`, `<const T>`, `unknown`, non-null `!` | obviated by the main proposal's semantics | [main proposal](README.md) |

## Open questions

**`keyof` in expression position.** `type keyof Point` works today; whether `keysOf` should instead be the `keyof` operator accepting a type-object operand is a pure surface decision.

**Constants.** The evaluation budget's defaults, the pattern-subtype automaton size bound, and the trial-specialization ceiling each need one spec'd number; depth-and-count limits in erased systems are the prior art to calibrate against.

**Inverse arity and overloads.** The record-of-proposals shape for multi-slot builders, and how a declared inverse interacts with an overloaded builder: propose per overload and verify jointly, or decline to consult.

**Freshness scope.** Trial specialization requires exact (excess-property) checking of object literals against candidate applications; whether that freshness rule is trial-local or the language's general literal-checking rule, where the typed-layout stance suggests general.

**Metadata inside walks.** `deepPartial` passes a `parameterized` node through untouched; confirm that default, noting that a builder stripping metadata a use-site depends on is caught as an ordinary assignability failure, which is the wanted behavior.

**Contract vocabulary.** `where` postconditions admit any evaluable predicate; if practice demands universally quantified forms (variance declarations), those need a proof-shaped mechanism or should stay absent, since per-evaluation checking cannot honor them.

**Cross-realm and serialized types.** Interning is per-agent; computed types crossing [serialization](serialization.md) or workers use the canonical form as the wire format with re-interning on arrival, and the artifact's provenance doc strings need a decided shape (raw comment text or a structured subset).

**Contextual `this` at a distance.** The adoption rule excludes arrow functions; methods extracted as values and passed onward need a stated behavior.
