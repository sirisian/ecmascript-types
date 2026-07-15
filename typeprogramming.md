# Type Programming with JavaScript Type Builders

This document responds to the one substantial gap identified in the TypeScript feature comparison ([#64](https://github.com/sirisian/ecmascript-types/issues/64#issuecomment-4955364388)): type-level programming. TypeScript ships a second, bespoke language — conditional types, mapped types, `infer`, template-literal types, and the utility library built on them — that runs inside the checker. This proposal deliberately has no such language. The question examined here is whether that entire surface can instead be covered by *type builders*: ordinary JavaScript functions that take type objects, inspect them with reflection, and return new type objects, evaluated by the engine at compile time under the same rules the proposal already uses for [primitive metadata](primitivemetadata.md) hooks, `where` clauses, and [compile-time type expressions](typeobjects.md).

The short answer, argued in detail below: yes, with six specific additions to the existing machinery. A builder model covers the practical TypeScript type-programming surface, exceeds it in several dimensions (real arithmetic, real string manipulation, first-class type functions, testable and debuggable type code, authored error messages), and concedes exactly three things that are inherent to computing types with opaque functions — inference *through* a type function's result, definition-site checking of generic bodies, and participation in the checker's own inference. Each concession has a workable bridge — and §6, written under a maximalist mandate (any feature, minimal new syntax), closes most of what the bridges leave, under one discipline: builders may propose, the engine must verify. The plan is: what exists today (§1), what TypeScript's type language actually consists of and which of its semantic properties matter (§2), the builder design delta (§3), an exhaustive side-by-side catalog (§4), what JavaScript alone cannot do (§5), how far each of those limits moves when nothing is off the table (§6), recommendations (§7), open questions (§8), and the verdict (§9).

A note on the premise. TypeScript's refusal to consider this approach rests on a design constraint: the checker never executes user code. That constraint buys TypeScript editor safety on untrusted repositories, deterministic and portable `.d.ts` semantics, and a checker whose performance is bounded by its own algorithms rather than arbitrary programs. It is a reasonable constraint *for an erasure compiler that is itself a third-party tool*. This proposal spent that constraint long ago: `meta` protocol hooks (`subtype`, `validate`, `narrow`, `conversionFactor`) are user functions the compiler calls; operator return types like `float32.<multiplyDimensions(D, D2)>` evaluate user arithmetic in type position; `where` clauses evaluate user predicates; `componentType(C)` is already a user function producing a type in an annotation. The compile-time evaluator is mandatory infrastructure here whether or not type builders exist. The marginal cost of type builders is therefore *zero new execution machinery* — it is purely an API-completeness question: reflection can already read a type's structure, and the missing half is constructing one.

## 1. What already exists in this proposal

The pieces below are already specified across [typeobjects.md](typeobjects.md), [decorators.md](decorators.md), [generics.md](generics.md), and [primitivemetadata.md](primitivemetadata.md). Type builders are a small delta on top of them, so it is worth being precise about the base.

**Types are first-class interned values.** A type name in expression position evaluates to its type object; structural types intern by canonical form, so `[].<uint8> === [].<uint8>` and two independently written but structurally identical object types are one object. Nominal types (`class`, `enum`) are identified by declaration. Interning is load-bearing for everything below: it makes type identity `===`, makes types usable as `Map` keys, and makes memoizing a builder call trivial.

**The `type` operator and `type NAME = ...`.** Type expressions whose grammar collides with value grammar are lifted into expression position with the `type` prefix: `const C = type 'a' | 'b' | 'c'` is a union type object; `type (uint8) => uint8` is a function type object. The statement form binds the object to a name and introduces an alias usable in type position.

**Compile-time type expressions.** The rule from [typeobjects.md](typeobjects.md) that this entire document leans on: *an expression that evaluates to a type object at compile time is a valid type annotation*. `componentType(C)` in a return-type position is the worked example — a `switch` over an enum value generic, evaluated at specialization, exhaustiveness-checked so it provably yields a type. The stated rules: evaluation happens at specialization when every generic parameter the expression reads is known; the expression must be *compile-time evaluable*; it must produce a type object; diagnostics name the expression rather than only its value.

**Compile-time evaluability.** The same notion `where` clauses and metadata annotations use: a function is evaluable when its body reads only its parameters, constants, and other evaluable functions. This is a static purity discipline rather than a runtime sandbox — an evaluable function *cannot name* ambient mutable state, I/O, or nondeterminism, so there is nothing to escape from. It is the proposal's answer to "running user JS in the compiler," and it is stricter and cheaper than a jail.

**Generic parameters are values.** A type generic parameter in scope evaluates to the type it was specialized with (`this.#values.set(T, value)`), and value generics (`<N: uint32>`, `<Name: string>`) are compile-time constants with SameValue identity. Both are exactly what a builder needs to receive as arguments. Generics are fully specialized — each application is a distinct type, checked and compiled per instantiation — which fixes *when* builders run: at specialization, once per distinct argument tuple.

**Structural reflection.** `Reflect.typeOf(v)` returns a value's type object; `Reflect.getReflection.<Reflect.Type>(t)` cracks a type object open into a node discriminated by `kind`. The node model as currently specified in [decorators.md](decorators.md):

```js
namespace Reflect {
  type TypeReflection =
    | { kind: 'primitive'; type: type; }                              // uint8, string, or a nominal class/enum reference
    | { kind: 'union'; arms: [].<type>; }
    | { kind: 'intersection'; members: [].<type>; }
    | { kind: 'tuple'; elements: [].<TypeTupleElement>; }
    | { kind: 'array'; element: type; extent: uint32 | undefined; }
    | { kind: 'object'; properties: [].<TypePropertyReflection>; }
    | { kind: 'function'; signatures: [].<FunctionSignatureReflection>; }
    | { kind: 'reference'; name: string; };                           // recursive back-edge

  type TypeTupleElement = { type: type; rest: boolean; };
  type TypePropertyReflection = { name: string | symbol; type: type; optional: boolean; };
}
```

**`keyof`.** Already a built-in compile-time type expression producing a union of literal types.

**Recursive types with coinductive assignability.** Aliases may be self- and mutually recursive provided every cycle passes through a reference position; equality and assignability across cycles assume the pair under comparison and look for contradiction. Builders that produce recursive types (§3.4) reuse this machinery rather than adding new theory.

What is *missing* from this base is exactly five pieces, none of them large:

1. **Construction.** Reflection is read-only. There is no `Reflect.makeType` to turn a node back into an interned type object, so a function cannot build an object type from a computed property list. This is the single load-bearing addition.
2. **Node completeness.** The current node model cannot round-trip every type: literal types have no node kind carrying their value, properties lack `readonly` and default-initializer information, index signatures don't appear on `object` nodes, metadata-parameterized primitives (`float32.<{ m: 1 }>`) have no structured form, and a generic application (`Map.<string, uint8>`) surfaces only as an opaque nominal `primitive` with no way to read `string` and `uint8` back out.
3. **Relations as callable predicates.** The checker's assignability judgment exists but isn't exposed; builders that filter unions (`Exclude`/`Extract`) need `Reflect.isAssignable(source, target)`.
4. **A bottom type.** The comparison table lists `never` as partial (divergence covers the return-type case). Union-filtering algebra needs the empty union to *be* a type: `never` is the identity of union and the natural result of `exclude` removing every arm. The checker already possesses this type internally — divergence is typed with it — so this is naming, not new theory.
5. **Recursion knot-tying.** A builder mapping over a recursive type (`deepPartial(Category)` where `Category` references itself) re-enters itself with the same argument; something must tie that cycle into a legal recursive type instead of diverging.

Section 3 specifies each. (Bookkeeping against the intro's *six additions*: items 1–2 travel together as the first, items 3–5 are the next three, and the remaining two — computed constraints with literal inference, §3.5, and the pattern-metadata route for infinite strings, §4.4 — are a placement rule and a recipe rather than missing API.)

## 2. What TypeScript's type language consists of

For the comparison to be exhaustive, here is the inventory of TypeScript's type-programming machinery, grouped by mechanism, together with the *semantic properties* of that machinery that any replacement must consciously match or consciously drop. The catalog in §4 walks the concrete constructs one by one; this section is about the shape of the system.

**The mechanisms.**

1. *Generics with constraints, defaults, and inference.* `<T extends U = D>`; type arguments inferred from value arguments, from contextual (expected) types, and — for homomorphic mapped types — by inverting the mapping.
2. *Operators over types.* `keyof T`, `typeof x`, indexed access `T[K]`, `readonly`/`?` in object members, tuple spreads `[...A, ...B]`.
3. *Conditional types.* `T extends U ? X : Y`, with three special behaviors: they stay **deferred** (unevaluated but still a type) while `T` involves an unresolved type parameter; they **distribute** over unions when `T` is a naked type parameter; and they host **`infer`**, which introduces fresh type variables bound by structural unification against `U`'s shape, optionally constrained (`infer S extends string`).
4. *Mapped types.* `{ [K in Keys]: F<K> }` with modifier arithmetic (`+?`, `-?`, `+readonly`, `-readonly`) and key remapping (`as NewKey`, where mapping a key to `never` deletes it). A mapped type over `keyof T` is **homomorphic**: it preserves each source property's modifiers and remains invertible for inference.
5. *Template-literal types and the four intrinsics.* `` `on${K}` `` denotes a set of strings by concatenation grammar — a *possibly infinite* set when holes are `string`/`number` — plus compiler-intrinsic `Uppercase`/`Lowercase`/`Capitalize`/`Uncapitalize`.
6. *Recursion.* Type aliases may recurse (with tail-recursion elimination on conditional types and hard limits: instantiation depth and count caps that surface as "type instantiation is excessively deep and possibly infinite").
7. *The utility library.* `Partial`, `Required`, `Readonly`, `Record`, `Pick`, `Omit`, `Exclude`, `Extract`, `NonNullable`, `Parameters`, `ConstructorParameters`, `ReturnType`, `InstanceType`, `Awaited`, `NoInfer`, `ThisParameterType`, `OmitThisParameter`, `ThisType`, and the string intrinsics — every one a composition of mechanisms 2–5.

**The semantic properties, and what the builder model does with each.**

- **P1 — Deferral.** `Partial<T>` with `T` unresolved is a real type the checker can carry, compare for identity, and instantiate later. *Builder answer:* an unevaluated call `partial(T)` is likewise carried as an interned application — identical calls are the identical type object — and evaluates at specialization. What is lost relative to TypeScript is any reasoning *finer than identity* before specialization; §5.2 examines the consequences.
- **P2 — Implicit distribution.** Naked-parameter conditionals silently map over union arms; suppressing it requires the `[T] extends [U]` idiom. *Builder answer:* distribution is an explicit `union(arms(T).map(f))`. This is a deliberate improvement — one of TypeScript's best-known foot-guns becomes a visible line of code — at the cost of writing that line.
- **P3 — Homomorphism.** Mapped types over `keyof T` copy modifiers automatically and can be inverted during inference (`declare function f<T>(p: Partial<T>): T` infers `T` from the argument). *Builder answer:* modifier preservation falls out of copying property records and editing only the field you mean to change (§4.2). Inversion does **not** carry over: a builder is an opaque function and the engine will not run it backwards. This is genuine gap #1 (§5.1).
- **P4 — Checker participation.** Conditional types and `infer` are evaluated *during* inference and overload resolution; `ThisType<T>` and `NoInfer<T>` are pure checker-behavior markers with no structural content. *Builder answer:* builders run after inference has bound their arguments, never during it. The sanctioned checker-integration points remain the `meta` protocol hooks (`subtype`, `narrow`, `validate`) — user code the checker calls at defined seams — rather than arbitrary types influencing inference. Genuine gap #2 (§5.3).
- **P5 — Tooling display.** Hovers and errors print mapped/conditional types structurally, sometimes helpfully, often as a wall of expanded conditionals. *Builder answer:* diagnostics already must name the originating expression (`componentType(Component.Health)`); the tooling plan (§7, R10) pairs the call with its canonical expansion, and builders may `throw` to produce *authored* error text — something TypeScript's type language famously cannot do.
- **P6 — Termination by limits.** TypeScript's type language is (accidentally) Turing-complete and is kept finite by depth and instantiation-count caps. *Builder answer:* the same, honestly: builders are deliberately Turing-complete and bounded by an evaluation budget (§3.4). Neither system offers guaranteed termination; TypeScript's precedent shows limits are an acceptable answer.
- **P7 — No user-code execution.** *Builder answer:* replaced by the static evaluability discipline plus the budget, as argued in the premise above. This is the one property the proposal rejects by design, and it rejected it before this document existed.

## 3. The design: type builders

A *type builder* is nothing new syntactically: it is a compile-time evaluable function whose return type is `type`, called in type position under the existing compile-time type expression rule. `componentType` already is one. What this section adds is the API that makes builders able to do general shape transformation, and the evaluation semantics that make them well-defined on recursive types and unbound generics.

### 3.1 `Reflect.makeType` and the completed node model

`Reflect.makeType(node)` is the inverse of `Reflect.getReflection.<Reflect.Type>`: it takes a plain node (or any object matching a node's shape — the input is structural), canonicalizes it under the same rules interning already uses (unions flattened, de-duplicated, deterministically ordered; nested types expanded to canonical form), and returns the interned type object. The round-trip law:

```js
Reflect.makeType(Reflect.getReflection.<Reflect.Type>(T)) === T; // for every T
```

Canonicalization is also where invalidity is caught: a node describing an infinite value-type layout, an intersection of distinct value types, or an index signature with an illegal key type is a `TypeError` from `makeType`, with the same message the equivalent source declaration would produce.

Round-tripping every type requires completing the node model. The revised `Reflect.TypeReflection`, a superset of the one in [decorators.md](decorators.md) (additions marked):

```js
namespace Reflect {
  type TypeReflection =
    | { kind: 'primitive'; type: type; generic: GenericApplication | undefined; } // generic added
    | { kind: 'literal'; value: string | number | boolean | bigint; base: type; } // added: 'a', 42, true, 5n
    | { kind: 'union'; arms: [].<type>; }                                         // arms: [] is never (§3.3)
    | { kind: 'intersection'; members: [].<type>; }
    | { kind: 'tuple'; elements: [].<TypeTupleElement>; }
    | { kind: 'array'; element: type; extent: uint32 | undefined; }
    | { kind: 'object'; properties: [].<TypePropertyReflection>;
        indexSignatures: [].<TypeIndexSignature>; }                               // indexSignatures added
    | { kind: 'function'; signatures: [].<FunctionSignatureReflection>; }
    | { kind: 'parameterized'; base: type; metadata: object; }                    // added: float32.<{ m: 1 }>
    | { kind: 'reference'; name: string; };                                       // recursive back-edge (reflection output only)

  type TypeTupleElement = { type: type; rest: boolean; initial: any | undefined; };
  type TypePropertyReflection = {
    name: string | symbol;
    type: type;
    optional: boolean;
    readonly: boolean;            // added, mirroring readonly fields
    initial: any | undefined;     // added: the a?: T = value default
  };
  type TypeIndexSignature = { key: type; value: type; };                          // added
  type GenericApplication = { base: type; arguments: [].<type | any>; };          // added: Map.<string, uint8> → { base: Map, arguments: [string, uint8] }
  type FunctionSignatureReflection = {                                            // from decorators.md, completed
    parameters: [].<FunctionParameterReflection>;
    return: { type: type; metadata: object | undefined; };                        // FunctionReturnReflection
    thisType: type | undefined;                                                   // added: the declared `this` type, for ThisType-style builders (§4.5)
  };
  type FunctionParameterReflection = {                                            // from decorators.md, completed
    type: type;
    name: string | undefined;                                                     // non-canonical: identity ignores it (§3.1)
    index: uint32;
    optional: boolean;                                                            // added: a `p?: T` parameter, distinct from one with a default
    rest: boolean;                                                                // added: a rest parameter was indistinguishable from a plain one
    initial: any | undefined;
    metadata: object | undefined;
  };
}
```

Notes on the additions:

- **`literal`** carries the value and its base primitive. Today a literal type is reachable only as an opaque type object; builders need to read `'circle'` back out of the type `'circle'` (`literalValues` in §4.1 is built on this) and to mint literal types from computed strings (key remapping, §4.2).
- **`generic` on `primitive`** exposes a nominal generic application's arguments. This is what replaces most uses of `infer`: TypeScript writes `T extends Promise<infer U> ? U : never` because it has no other way to reach inside `Promise<T>`; here `node.generic` hands back `{ base: Promise, arguments: [U] }` directly. Value generic arguments appear as values, type arguments as type objects.
- **`parameterized`** is the metadata half of the same story: `Reflect.typeOf` already returns the full parameterization for a metadata-carrying value, so the node model must be able to express and rebuild it. Builders can thereby *manipulate metadata* — strip units, tighten bounds — with the same tools they use on shapes.
- **`readonly`, `initial`, `indexSignatures`** exist in the type grammar (readonly fields, `a?: T = []` optional defaults, `[T, U = d]` trailing tuple defaults, index signatures) and merely weren't surfaced. The same completion pass adds `rest: boolean` to `FunctionParameterReflection` — a rest parameter is currently indistinguishable from a plain one, which `parameters` (§4.3) needs — and `optional` and `thisType` (to `FunctionParameterReflection` and `FunctionSignatureReflection` respectively), without which optional-parameter builders and `withThisType`/`thisParameterType` (§4.5) have nothing to read or write, and a `Reflect.ClassConstructor` reflection context, since `ClassReflection` today carries `name`/`type`/`abstract`/`metadata` and every member kind has a context *except* construct signatures. Without `readonly` in the node, `Readonly`/`Mutable` builders are unwritable and every mapped builder would silently strip the flag — the completeness of this record is exactly what makes builders homomorphic by default (§4.2).
- Constructed nodes never contain `reference` nodes; cycles in *construction* come from the fixpoint mechanism in §3.4, and `reference` remains what a *reader* sees when walking an already-cyclic type.

Two small conveniences round this out. Type objects get a canonical `toString` (the canonical source form — `String(type 'a' | 'b')` is `"'a' | 'b'"`), because builders throwing authored `TypeError`s need to print types. And `Reflect.never` names the empty union (§3.3) so kit code doesn't spell it as a construction.

### 3.2 Relations: identity and assignability

Identity is already solved by interning: `A === B` is structural equality (nominal for classes and enums), which single-handedly replaces one of TypeScript's ugliest idioms (§4.8). The missing relation is the checker's assignability judgment as a callable, evaluable predicate:

```js
Reflect.isAssignable(source: type, target: type): boolean;
```

This is the same coinductive judgment the checker runs on every assignment, exposed. It is what conditional-type replacements branch on: TypeScript's `T extends U ? X : Y` becomes `Reflect.isAssignable(T, U) ? X : Y` inside a builder. It is deliberately the *assignability* relation, not a new one — builders that want mutual assignability compose it, and builders that want identity use `===`.

### 3.3 `never` as a real, computable type

The empty union is admitted as a type object: `Reflect.makeType({ kind: 'union', arms: [] })` returns it, `Reflect.never` names it, and canonicalization gives it the algebra the utility library needs — it is the identity of union (`union([T, never]) === T`), it annihilates intersection, it is assignable to every type, and no value inhabits it. The checker already has this type internally (divergence satisfies any return type by producing it), so this is a naming decision plus one question the main proposal must answer: whether `never` becomes *writable* in annotations, or remains reachable only by computation. Since a computed type is a valid annotation, builders make it writable indirectly anyway (`let x: exclude(T, T)` would be legal and uninhabitable); the recommendation (§7, R3) is to admit it openly with the standard meaning rather than pretend otherwise, while keeping the existing stance that `switch` exhaustiveness stays reserved to enums and sealed classes.

### 3.4 Evaluation semantics

**When.** A builder call in type position evaluates at specialization, when every type and value generic parameter it reads is bound — the existing compile-time type expression rule, restated to include *type* generic parameters explicitly (today's text names value generics; type parameters are already type objects at specialization, so this is clarification, not change). At module top level, where there are no unbound parameters, calls evaluate during compilation of the module, before any of its code runs.

**Memoization by interning.** A call is keyed by (function identity, argument tuple) with type arguments compared by `===` (interned) and primitive value arguments by SameValue — the identical key rule value generics already use. Compound constant arguments need one extra rule: `pick(User, ['id', 'name'])` passes an array whose *reference* differs at every occurrence, so compound compile-time constants canonicalize into the key by structural value — precisely the value semantics the [composites](README.md#composites) extension gives tuples and records, and consistent with generics rejecting reference-identity arguments. (Callers who prefer already-interned arguments can pass a union of literals instead; the kit's overloads in §4.2 accept both.) Each distinct key evaluates once per program; every subsequent occurrence, in any module, yields the same interned type object. Consequently `partial(User) === partial(User)` everywhere, and an unevaluated application inside a generic declaration (`partial(T)` with `T` unbound) is itself carried as an interned deferred application with that same identity rule, which is what makes P1-style deferral coherent: two mentions of `partial(T)` in one generic body are the same type even before anyone knows what `T` is.

**Recursion and knot-tying (in-flight fixpoint).** When evaluation of `f(args)` re-enters `f` with an *identical* key before the outer call returns, the inner call immediately returns a fresh *placeholder type object* instead of recursing. When the outer call returns node `N`, the placeholder is resolved to the canonicalization of `N` — producing exactly the kind of named-back-edge cyclic type that `type Tree = { value: uint32, children: [].<Tree> }` produces from syntax, validated by the same rule (every cycle must pass through a reference position, or `TypeError`). Two failure modes are diagnosed: a call whose entire result *is* its own placeholder (`function f(T) { return f(T); }`) is an unproductive-recursion `TypeError`, and a recursion that never repeats a key (a genuinely infinite family, like `paths` on a cyclic type, §4.6) runs into the budget. This mechanism is what makes `deepPartial` work on recursive types with zero API surface — the walkthrough is in §4.6 — and it is the computational twin of the coinductive assignability the proposal already specifies.

**Purity and determinism.** Builders are compile-time evaluable functions, so the static discipline applies: they read only parameters, constants, and other evaluable functions. *Local* mutation (a `Set` of seen keys, an accumulator array) is fine; *shared* module-level mutable state is not evaluable and is rejected statically — which also disposes of the tempting-but-wrong pattern of a hand-rolled module-level memo cache, since the engine's interning memoization (above) is the supported mechanism and is observably equivalent. Evaluability rules out `Date.now`, `Math.random`, I/O, and ambient reads by construction, so builds are deterministic and the "sandbox" is a property of what the code can name rather than a wall around what it does.

**Budget.** Evaluation carries a fuel budget (spec'd default, host-overridable) covering steps and constructed-type count per top-level type-position evaluation. Exhaustion is a compile-time `TypeError` naming the outermost call and the deepest frames — the moral equivalent of TypeScript's instantiation-depth diagnostic, with a real stack attached.

**Errors.** A builder may `throw` (typically `TypeError`) and the thrown message becomes the compile-time diagnostic, prefixed with the originating type-position expression per the existing diagnostics rule. This is worth pausing on: TypeScript library authors resort to elaborate tricks to make misuse produce a readable error; a builder writes `throw new TypeError(`pick: ${String(T)} has no property '${key}'`)` and is done.

**Dual use.** A builder is an ordinary function. Called at runtime with runtime type objects (from `Reflect.typeOf` or values of type `type`), it does exactly the same thing and returns the same interned objects. Type-level code therefore gets unit tests in a normal test runner, `console.log`-style debugging in a scratch script, coverage, and profiling — none of which TypeScript's type language has ever had. This is not a side benefit; it is a structural consequence of the type language being *the* language.

### 3.5 Where builder calls may appear

Everywhere a type may: variable and field annotations, parameter and return types, generic argument lists (`Map.<string, partial(User)>`), the right side of `is` and `as`, alias declarations (`type Draft = partial(User)`), and — the one placement that needs a sentence of new specification — **generic constraints**. A constraint may be a compile-time call over *earlier* parameters in the same list, evaluated left to right:

```js
function pluck<T, K: keysOf(T)>(o: T, key: K): indexed(T, K) {
  return o[key];
}
```

`keysOf(T)` evaluates once `T` is bound (inferred from `o`), and the resulting union-of-literals constraint does double duty: it checks `key`, and it drives *literal inference* for `K` — an argument `'name'` binds `K` to the literal type `'name'` rather than widening to `string`, exactly as `K extends keyof T` cues TypeScript to keep the literal. That inference rule (a parameter constrained to a union of literals infers literally) is recommendation R5 and is the only inference change builders ask for.

### 3.6 The standard kit ships as JavaScript

Everything in §4 is written against `Reflect.makeType`, `Reflect.getReflection.<Reflect.Type>`, `Reflect.isAssignable`, and `Reflect.never` — no construct below is engine magic. That is the point: the entire TypeScript utility library, plus the mapped/conditional/template machinery it is built from, lands as roughly two hundred lines of ordinary evaluable JavaScript. The recommendation (R6) is to ship those two hundred lines as a standard module (spelled `import { partial, pick, keysOf, ... } from 'std:types';` below, final home to be decided with [standardlibrary.md](standardlibrary.md)) so the ecosystem shares one interned vocabulary, but nothing prevents user-space from writing every one of them today — which is also the compatibility story: a codebase can polyfill the kit itself.

## 4. The catalog

Each entry pairs the TypeScript construct with the builder equivalent. The builder code is written in this proposal's syntax and is *complete* — it runs against the §3 API, and because builders are ordinary functions, every definition below is also runtime-testable as written. TypeScript definitions of the built-in utilities are reproduced from their standard published definitions where showing the mechanism matters.

### 4.0 The shared prelude

Fifteen lines of helpers the rest of the catalog reuses. This is the bottom of the proposed `std:types` module.

```js
export function reflect(T: type): Reflect.TypeReflection {
  return Reflect.getReflection.<Reflect.Type>(T);
}
export const never: type = Reflect.never;

export function literal(value: string | number | boolean | bigint): type {
  return Reflect.makeType({ kind: 'literal', value, base: Reflect.typeOf(value) });
}
export function union(armList: [].<type>): type {
  return Reflect.makeType({ kind: 'union', arms: armList }); // flattens, dedupes, sorts; [] → never, [T] → T
}
export function arms(T: type): [].<type> {
  const node = reflect(T);
  return node.kind === 'union' ? node.arms : [T];
}
export function literalValues(T: type): [].<string | number | boolean | bigint> {
  return arms(T).map(arm => {
    const node = reflect(arm);
    if (node.kind !== 'literal') throw new TypeError(`expected a union of literals, got ${String(arm)}`);
    return node.value;
  });
}
export function prop(name: string | symbol, type: type,
    { optional = false, readonly = false, initial = undefined } = {}): Reflect.TypePropertyReflection {
  return { name, type, optional, readonly, initial };
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

And the workhorse — the builder analog of a homomorphic mapped type. It walks an object type's property records and rebuilds the type from whatever the callback returns; returning the record unchanged preserves everything (type, `optional`, `readonly`, defaults), spreading it and overriding one field edits exactly that field, and returning `null` deletes the property. Distribution over a union of object types is one visible line:

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

Where TypeScript makes homomorphism a special status a mapped type earns by iterating `keyof T` in the right syntactic form, here it is just the default behavior of copying a record: you cannot accidentally strip `readonly` from properties you never mentioned, because you would have to write the code that strips it.

### 4.1 The operators: `keyof`, `typeof`, indexed access

`keyof` already exists as a built-in over syntactic type expressions. Builders need the same operation over a type object held in a variable, which is a four-line function rather than new syntax — and it doubles as the specification of what `keyof` means on unions and intersections (TypeScript: keys of a union are the *common* keys; keys of an intersection are the union of keys):

```ts
// TypeScript
type Point = { x: number; y: number };
type Axis = keyof Point;                       // 'x' | 'y'
type Common = keyof (A | B);                   // keyof A & keyof B
declare function pluck<T, K extends keyof T>(o: T, key: K): T[K];
```

```js
// Builder
export function keysOf(T: type): type {
  const node = reflect(T);
  switch (node.kind) {
    case 'object':       return union([
      ...node.properties.filter(p => typeof p.name === 'string').map(p => literal(p.name)),
      ...node.indexSignatures.map(s => s.key),   // keyof { [key: string]: T } includes string itself
    ]);
    case 'intersection': return union(node.members.map(keysOf));
    case 'union': {                                // common keys: intersect the per-arm key unions
      const keyUnions = node.arms.map(keysOf);
      return keyUnions.reduce((acc, next) => extract(acc, next)); // extract (§4.3): keeps 'x' against 'x', and 'x' against string
    }
    default: throw new TypeError(`keysOf: ${String(T)} has no keys`);
  }
}

export function indexed(T: type, K: type): type {   // TypeScript's T[K]
  return union(arms(T).flatMap(t => {
    const node = reflect(t);
    return literalValues(K).map(key => {
      const p = node.properties.find(p => p.name === key);
      if (!p) throw new TypeError(`${String(t)} has no property '${String(key)}'`);
      return p.optional ? union([p.type, type undefined]) : p.type;  // policy: optional access admits undefined
    });
  }));
}

function pluck<T, K: keysOf(T)>(o: T, key: K): indexed(T, K) {
  return o[key];
}
const name = pluck(user, 'name'); // K = 'name' (literal inference via the computed constraint, §3.5)
```

Two things worth noticing. The `undefined`-on-optional-access decision that TypeScript gates behind a compiler flag is a one-line, readable *policy choice* inside `indexed`, and a codebase that wants the other policy writes the other line. And symbol keys: property records carry `name: string | symbol`, so `pick`/`omit`/`mapProperties` handle symbol-keyed members by identity, but `keysOf` cannot mint a *literal type* for a symbol — literal types cover strings, numbers, booleans, and bigints, and the proposal has no `unique symbol`. Symbol keys are therefore reachable to builders but absent from `keyof`-style unions — the definition above filters them, and folds index-signature *key types* in wholesale, mirroring TypeScript's `keyof { [k: string]: T } = string` — matching the existing `unique symbol` gap row rather than widening it, until §6.6 closes that row.

`typeof x` needs no builder: types are values, so `Reflect.typeOf(x)` in type position is the type query, and for a binding whose declared type you want without a value, the declared name itself is already the type object.

### 4.2 Mapped types

**The modifier family.** TypeScript's definitions, then the builders — note that `-readonly` (`Mutable`), which TypeScript's modifier grammar supports but whose utility `lib.d.ts` never shipped, costs the same one line as the others:

```ts
// TypeScript
type Partial<T>  = { [P in keyof T]?: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
type Readonly<T> = { readonly [P in keyof T]: T[P] };
type Mutable<T>  = { -readonly [P in keyof T]: T[P] };
```

```js
// Builder
export function partial(T: type): type  { return mapProperties(T, p => ({ ...p, optional: true  })); }
export function required(T: type): type { return mapProperties(T, p => ({ ...p, optional: false })); }
export function readonly(T: type): type { return mapProperties(T, p => ({ ...p, readonly: true  })); }
export function mutable(T: type): type  { return mapProperties(T, p => ({ ...p, readonly: false })); }

type User = { id: uint32, name: string, readonly createdAt: string };
type Draft = partial(User);   // { id?: uint32, name?: string, readonly createdAt?: string }
let d: Draft = { name: 'a' };
```

Modifier *arithmetic* (`+?`/`-?`/`+readonly`/`-readonly`) is spread-and-override; homomorphic preservation is the default (§4.0). TypeScript's subtlety that `Partial` distributes over a union operand because homomorphic maps do — `Partial<A | B>` is `Partial<A> | Partial<B>` — is the explicit `union` case inside `mapProperties`. One deliberate simplification: `mapProperties` passes index signatures through untouched, so the modifier builders edit declared properties only — `deepPartial` (§4.6) shows signatures being walked when a builder means to include them.

**`Pick`, `Omit`, `Record`.** The proposal's function overloading (distinct bodies) lets the kit accept either TypeScript's union-of-literals style or a plain key array, whichever reads better at the call:

```ts
// TypeScript
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
type Record<K extends keyof any, T> = { [P in K]: T };

type Credentials = Pick<User, 'id' | 'name'>;
type Sanitized  = Omit<User, 'password'>;
type Flags      = Record<'read' | 'write', boolean>;
type Scores     = Record<string, uint32>;
```

```js
// Builder
export function pick(T: type, K: type | [].<string | symbol>): type {
  const wanted = new Set(Array.isArray(K) ? K : literalValues(K));
  const have = new Set(literalValues(keysOf(T))); // union/intersection-safe key set (§4.1)
  for (const key of wanted) if (!have.has(key))
    throw new TypeError(`pick: ${String(T)} has no property '${String(key)}'`); // TS: an opaque constraint failure
  return mapProperties(T, p => wanted.has(p.name) ? p : null);
}
export function omit(T: type, K: type | [].<string | symbol>): type {
  const dropped = new Set(Array.isArray(K) ? K : literalValues(K));
  return mapProperties(T, p => dropped.has(p.name) ? null : p);
}
export function record(K: type, V: type): type {
  const node = reflect(K);
  if (node.kind === 'literal' || node.kind === 'union' && node.arms.every(a => reflect(a).kind === 'literal'))
    return objectOf(literalValues(K).map(name => prop(name, V)));   // Record<'read' | 'write', T> → concrete properties
  return objectOf([], [{ key: K, value: V }]);                      // Record<string, T> → index signature
}

type Credentials = pick(User, ['id', 'name']);
type Sanitized  = omit(User, ['password']);
type Flags      = record(type 'read' | 'write', boolean);
type Scores     = record(string, uint32);                           // { [key: string]: uint32 }
```

`record` collapsing to the two forms the comparison table already names — concrete properties or an index signature — is a good example of a builder *encoding a design decision as inspectable code* instead of a checker rule.

**Key remapping.** TypeScript's `as` clause, including its idiom of remapping to `never` to delete a key, and the classic getters example. Computed key names, which TypeScript reaches through template-literal types plus the `Capitalize` intrinsic, are ordinary string expressions here:

```ts
// TypeScript
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type RemoveKind<T> = { [K in keyof T as Exclude<K, 'kind'>]: T[K] };
type PickByValue<T, V> = { [K in keyof T as T[K] extends V ? K : never]: T[K] };
```

```js
// Builder
const capitalize = (s: string): string => s.charAt(0).toUpperCase() + s.slice(1);

export function getters(T: type): type {
  return mapProperties(T, p => prop(`get${capitalize(p.name)}`, fn([], p.type), { readonly: true }));
}
export function removeKind(T: type): type {
  return omit(T, ['kind']);
}
export function pickByValue(T: type, V: type): type {
  return mapProperties(T, p => Reflect.isAssignable(p.type, V) ? p : null);
}

type PointGetters = getters(type { x: float32, y: float32 }); // { readonly getX: () => float32, readonly getY: () => float32 }
type Strings = pickByValue(User, string);                     // only the string-typed members of User
```

The comparison is stark in a way worth stating plainly: `as T[K] extends V ? K : never` is a conditional type, an indexed access, distribution, and the never-deletes-a-key rule composed inside a remap clause — four mechanisms to express "keep the property if its type fits." The builder is `filter` by another name. This is the recurring pattern of the whole catalog: TypeScript's mechanisms are ingenious encodings forced by having no functions; given functions, the encodings dissolve into the standard library of a normal language.

### 4.3 Conditional types, distribution, and `infer`

**The distributive filters.** `Exclude`, `Extract`, and `NonNullable` are the utility library's face of distributive conditional types. The builder versions make the distribution a visible `filter` and use the exposed assignability judgment as the branch test:

```ts
// TypeScript
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;
type NonNullable<T> = T & {};

type Shape = Circle | Rect | Line;
type NotLine = Exclude<Shape, Line>;
type Events = Extract<'click' | 'focus' | 42, string>;   // 'click' | 'focus'
```

```js
// Builder
export function exclude(T: type, U: type): type {
  return union(arms(T).filter(arm => !Reflect.isAssignable(arm, U)));
}
export function extract(T: type, U: type): type {
  return union(arms(T).filter(arm => Reflect.isAssignable(arm, U)));
}
export function nonNullable(T: type): type {
  return exclude(T, type null | undefined);
}

type NotLine = exclude(Shape, Line);
type Events = extract(type 'click' | 'focus' | 42, string);
```

`never` earns its keep here: `union([])` on a fully filtered union has to *be* something, and it is the same something TypeScript's `never` is (§3.3).

**The distribution foot-gun, dissolved.** In TypeScript, whether a conditional distributes depends on whether `T` appears naked, and suppressing distribution requires the tuple-wrapping idiom — a genuine trap that every advanced TypeScript user has been bitten by:

```ts
// TypeScript
type ToArray<T> = T extends any ? T[] : never;        // distributes: string | number → string[] | number[]
type ToArrayAll<T> = [T] extends [any] ? T[] : never; // does not:     string | number → (string | number)[]
```

```js
// Builder — both intents are literal
export function toArrayEach(T: type): type { return union(arms(T).map(arm => arrayOf(arm))); }
export function toArrayAll(T: type): type  { return arrayOf(T); }
```

There is no implicit behavior to suppress; the map over arms either appears in the code or it doesn't.

**`infer` as reflection.** Most uses of `infer` are destructuring — reaching a component of a type the language gives you no other accessor for. Reflection *is* the accessor, so the conditional-plus-`infer` scaffolding evaporates:

```ts
// TypeScript
type Flatten<T> = T extends (infer U)[] ? U : T;
type FirstParameter<F> = F extends (x: infer A, ...rest: any[]) => any ? A : never;
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
```

```js
// Builder
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
}   // p.rest is an R1 addition to parameter records; a default lands as the trailing tuple form [T, U = d]
```

One deliberate divergence is visible in `returnType`: on an overloaded function TypeScript resolves `infer R` to the *last* overload's return type — a rule users discover by surprise. The builder takes the union, and a codebase that prefers TypeScript's rule writes `returns.at(-1)` instead. Overload policy becomes a decision the code states rather than trivia the checker imposes.

**`infer` on generic applications.** The other big `infer` family reaches inside nominal generics — `Promise<infer U>`, `Map<infer K, infer V>`. The `generic` field added to `primitive` nodes (§3.1) is the direct accessor, shown here on `Awaited`, whose real TypeScript definition is a five-line thenable unwrap:

```ts
// TypeScript (the actual lib definition, reformatted)
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    (F extends (value: infer V, ...args: infer _) => any ? Awaited<V> : never) :
  T;
```

```js
// Builder
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

The recursion is the same shape as TypeScript's; what disappears is the encoding of "reach the callback's first parameter" as nested conditional inference.

**`ConstructorParameters` and `InstanceType`.** Half of this pair dissolves. In this proposal a class's type object *is* the class and the class name *is* its instance type — there is no `typeof C` constructor-type / instance-type split — so `InstanceType<typeof C>` has no job: you already hold `C`. What remains meaningful is reflecting construct signatures — which the declaration-reflection model does not yet surface: `ClassReflection` carries `name`, `type`, `abstract`, and `metadata`, and fields, accessors, methods, and operators each have a context, but the constructor has none. R1 therefore adds a `Reflect.ClassConstructor` context returning the constructor's overload list in the existing `FunctionSignatureReflection` shape:

```ts
// TypeScript
type ConstructorParameters<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: infer P) => any ? P : never;
type InstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;
```

```js
// Builder
export function constructorParameters(C: type): type {
  const { signatures } = Reflect.getReflection.<Reflect.ClassConstructor, C>(); // added by R1; overload policy as in returnType
  return Reflect.makeType({ kind: 'tuple',
    elements: signatures[0].parameters.map(p => ({ type: p.type, rest: p.rest, initial: p.initial })) });
}

function build<C>(...args: constructorParameters(C)): C {
  return new C(...args); // a class's type object is its constructor, so a specialized C constructs
}
const p = build.<Player>(10, 20);
```

**The residual `infer`: pattern matching as an API, not a keyword.** After reflection and `generic` access, what is left of `infer` is matching an *abstract pattern* against an unknown structure — rare in practice, but for completeness a library-level unifier covers it (recommendation R8):

```js
const K = Reflect.inferSlot('K'), V = Reflect.inferSlot('V');
const m = Reflect.matchType(genericApplication(Map, [K, arrayOf(V)]), T); // pattern for Map.<K, [].<V>>
if (m) { /* m.K and m.V are the bound type objects */ }
```

`Reflect.matchType(pattern, subject)` performs one-sided structural unification and returns bindings or `null`; `infer S extends string`-style constraints are an `isAssignable` check on the binding afterwards. It is proposed as optional (the catalog needed it exactly zero times), and its existence is mostly an argument-closer: nothing `infer` does is out of reach.

**Deferral.** Inside a generic declaration, `T extends U ? X : Y` and `partial(T)` are in the same boat: unevaluable until `T` binds. TypeScript carries the deferred conditional and can occasionally reason about it (its constraint is `X | Y`; identical deferred conditionals match). The builder model carries the deferred *application* `partial(T)`, interned so identical applications match (§3.4), with declared result knowledge limited to `: type` — effectively "some type." The practical difference shows up only when checking generic bodies before specialization, taken up in §5.2–5.3; under this proposal's full-specialization semantics every use eventually reaches a concrete `T`, where the difference vanishes.

### 4.4 Template-literal types and the string intrinsics

**Finite string computation.** Over unions of literals — which is what event maps, key remapping, and prefixing actually operate on — template-literal types and the four intrinsics are ordinary string code:

```ts
// TypeScript
type EventName<K extends string> = `on${Capitalize<K>}`;
type Listeners<T> = { [K in keyof T & string as `on${Capitalize<K>}Changed`]: (value: T[K]) => void };
type Loud<T extends string> = Uppercase<T>;
```

```js
// Builder
export function mapLiterals(T: type, f: (string) => string): type {
  return union(literalValues(T).map(v => literal(f(String(v)))));
}
export function uppercase(T: type): type   { return mapLiterals(T, s => s.toUpperCase()); }
export function lowercase(T: type): type   { return mapLiterals(T, s => s.toLowerCase()); }
export function capitalized(T: type): type { return mapLiterals(T, capitalize); }
export function uncapitalized(T: type): type {
  return mapLiterals(T, s => s.charAt(0).toLowerCase() + s.slice(1));
}

export function listeners(T: type): type {
  return mapProperties(T, p => prop(`on${capitalize(p.name)}Changed`, fn([p.type], type void)));
}

type Settings = { volume: uint8, theme: string };
type SettingsListeners = listeners(Settings);
// { onVolumeChanged: (uint8) => void, onThemeChanged: (string) => void }
```

Cross products, the other template-literal trick (`` `${Vertical}-${Horizontal}` `` producing nine literals from three-by-three), are a nested `flatMap` over two `literalValues` calls — code so unremarkable it needs no example.

**The showcase: parsing.** The strongest advertisement for builders is anything that *parses* in type position. Extracting route parameters is the community's canonical hard template-literal exercise; compare a correct recursive-conditional solution to `split`:

```ts
// TypeScript
type RouteParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param]: string } & RouteParams<`/${Rest}`>
    : Path extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

declare function get<P extends string>(
  path: P, handler: (params: RouteParams<P>) => Response): void;
```

```js
// Builder
export function routeParams(path: string): type {
  return objectOf(path.split('/')
    .filter(segment => segment.startsWith(':'))
    .map(segment => prop(segment.slice(1), string)));
}

function get<P: string>(path: P, handler: (params: routeParams(P)) => Response): void {}

get('/users/:id/posts/:postId', ({ id, postId }) => { ... }); // id: string, postId: string
```

`P` is a value generic bound implicitly from the first argument (the argument-binding rule in [generics.md](generics.md)), so the handler's parameter type is computed from the actual route string at the call. And because `routeParams` is real code, the version every router actually wants — typed segments like `'/users/:id(uint32)'` mapping `id` to `uint32` — is a `Map` lookup away, where in TypeScript it is another research project. The same argument applies to every parse-shaped problem the TypeScript community has heroically encoded (SQL result types, printf argument checking, JSON path access): builders reduce them from encodings to programs.

**Infinite string sets: a different mechanism, on purpose.** `` `${number}px` `` and `` `on${string}` `` denote *infinite* languages; no finite union of literals expresses them, and this is the one part of template-literal types a builder over today's type kinds cannot produce. The proposal already contains the right home for it: a refined string is a [primitive metadata](primitivemetadata.md) parameterization, and the [regexp](regexp.md) extension supplies typed patterns. A `StringPattern` meta type — `string.<{ pattern: /^\d+px$/ }>` — gives assignment-time and runtime validation through the standard `validate` hook, and its `subtype` hook decides pattern-vs-pattern assignability. That last judgment is where honesty is required: regular-language inclusion is decidable but expensive in general and undecidable once patterns exceed regular power, so `subtype` should be conservative (syntactic containment for the concatenation-of-literals-and-classes forms template types use; otherwise only reflexive), with `validate` carrying the rest at runtime. That trade — a slightly conservative static judgment for a strictly more expressive pattern language plus real runtime enforcement — is a better deal than template-literal types, which validate nothing at runtime and cap out at concatenation grammar. Builders interoperate directly, since `parameterized` nodes round-trip:

```js
type Pixels = string.<{ pattern: /^\d+px$/ }>;
export function suffixed(suffix: string): type {
  return Reflect.makeType({ kind: 'parameterized', base: string,
    metadata: { pattern: new RegExp(`^.*${RegExp.escape(suffix)}$`) } });
}
let margin: suffixed('px') = '12px';
```

### 4.5 Tuples, variadics, and type-level arithmetic

Tuple surgery in TypeScript is variadic spreads plus recursive conditionals; on reflection nodes it is array methods:

```ts
// TypeScript
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type Tail<T extends any[]> = T extends [any, ...infer R] ? R : never;
type Concat<A extends any[], B extends any[]> = [...A, ...B];
type Reverse<T extends any[]> = T extends [infer H, ...infer R] ? [...Reverse<R>, H] : [];
type Zip<A extends any[], B extends any[]> =
  A extends [infer AH, ...infer AR] ? B extends [infer BH, ...infer BR]
    ? [[AH, BH], ...Zip<AR, BR>] : [] : [];
```

```js
// Builder
const elementTypes = (T: type): [].<type> => reflect(T).elements.map(e => e.type);

export function head(T: type): type    { return elementTypes(T)[0] ?? never; }
export function tail(T: type): type    { return tupleOf(elementTypes(T).slice(1)); }
export function concat(A: type, B: type): type { return tupleOf([...elementTypes(A), ...elementTypes(B)]); }
export function reverse(T: type): type { return tupleOf(elementTypes(T).toReversed()); }
export function zip(A: type, B: type): type {
  const a = elementTypes(A), b = elementTypes(B);
  return tupleOf(a.slice(0, Math.min(a.length, b.length)).map((t, i) => tupleOf([t, b[i]])));
}
```

Named tuple elements, per the comparison table, are the intersection form `[T, U] & { x: 0, y: 1 }` here rather than TypeScript's labels; a builder that attaches or reads names manipulates that intersection with the same tools.

**Arithmetic.** The starkest single comparison in this document. TypeScript has no numbers at the type level, so addition is performed by materializing tuples and reading their lengths — the standard technique, with its standard failure modes (limits near a thousand elements, integer-only, painful subtraction):

```ts
// TypeScript
type BuildTuple<N extends number, T extends unknown[] = []> =
  T['length'] extends N ? T : BuildTuple<N, [...T, unknown]>;
type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]['length'];

declare function concatArrays<A extends unknown[], B extends unknown[]>(a: A, b: B): [...A, ...B];
```

```js
// Builder — numbers are numbers; value generics already carry them through signatures
function concatArrays<A: uint32, B: uint32, T>(a: [A].<T>, b: [B].<T>): [A + B].<T> {}
function reshape<R: uint32, C: uint32, T>(m: [R * C].<T>): [R].<[C].<T>> {}
```

`A + B` in the extent is a compile-time evaluable expression over value generics — the same evaluation `multiplyDimensions(D, D2)` already performs on metadata. There is nothing to build; the capability is a corollary of value generics plus evaluability, and it covers ground (fixed-size linear algebra shapes, bit-width computation, ring-buffer sizing) that TypeScript's tuple trick cannot reach at all.

### 4.6 Recursive builders

**`DeepPartial`, and why the fixpoint matters.** The TypeScript community definition and the builder:

```ts
// TypeScript (common community form; note it also maps over methods and mangles class instances)
type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;
```

```js
// Builder
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
    default:      return T; // primitives, literals, functions, classes, enums pass through
  }
}
```

The `default` arm quietly fixes two things the TypeScript version gets wrong by accident: functions and class instances pass through instead of being mapped into partial method bags, because the builder *states* its domain. Now the interesting case — a recursive input:

```js
type Category = { name: string, parent: Category | null };
type Draft = deepPartial(Category);
```

Evaluation, step by step under §3.4: `deepPartial(Category)` begins and its key is marked in-flight. Walking `parent`'s type `Category | null` distributes into `deepPartial(Category)` — the identical key — which returns a placeholder instead of recursing. The outer call finishes building `{ name?: string, parent?: ⟨placeholder⟩ | null }`, the placeholder is resolved to the canonicalized result, and the outcome is precisely the type that `type Draft = { name?: string, parent?: Draft | null }` declares by hand: a legal cycle through a nullable-union reference position, interned by the same named-back-edge canonicalization. TypeScript, for comparison, handles this by lazily deferring instantiation — and when a recursive type manages to defeat that laziness, answers with its depth-limit error. Both systems lean on a fixpoint; this one is specified rather than emergent.

**`Paths` — recursion that grows.** Dot-notation key paths, a staple of typed `get(obj, 'a.b.c')` APIs:

```ts
// TypeScript
type Paths<T> = T extends object
  ? { [K in keyof T & string]: T[K] extends object ? K | `${K}.${Paths<T[K]>}` : K }[keyof T & string]
  : never;
```

```js
// Builder
export function paths(T: type): type {
  const node = reflect(T);
  if (node.kind !== 'object') return never;
  return union(node.properties.flatMap(p => {
    const nested = paths(p.type);
    const suffixes = nested === never ? [] : literalValues(nested);
    return [literal(p.name), ...suffixes.map(rest => literal(`${p.name}.${rest}`))];
  }));
}

function getPath<T, P: paths(T)>(o: T, path: P): any { ... } // P checked against the computed union
```

On a *cyclic* type the path set is genuinely infinite; the keys never repeat (each call receives a different property type as it descends), so the fixpoint never fires and the budget (§3.4) terminates evaluation with a diagnostic naming `paths` and the offending cycle. TypeScript fails the same input with its depth error. Neither system can enumerate an infinite union; the difference is that the builder's failure message can say so in the author's words — `paths` can trivially carry a `seen: Set.<type>` parameter and `throw new TypeError('paths: T is recursive; path strings are unbounded')`, an option the TypeScript version does not have.

**`JSON`** requires no builder in either system — it is a plain recursive alias, and the proposal's version was already writable: `type Json = null | boolean | float64 | string | [].<Json> | { [key: string]: Json }`.

### 4.7 Discriminated-union tooling

The comparison table notes discriminated unions themselves are covered (tag fields, plus `sealed` hierarchies with real exhaustiveness). What TypeScript layers on top is *computed* handler shapes. The canonical pattern, both ways:

```ts
// TypeScript
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rect'; width: number; height: number };

type Handlers<T extends { kind: string }, R> =
  { [K in T['kind']]: (value: Extract<T, { kind: K }>) => R };

function match<T extends { kind: string }, R>(value: T, handlers: Handlers<T, R>): R {
  return handlers[value.kind as T['kind']](value as any); // the correlation cast, as usual
}
```

```js
// Builder
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

function match<T, R>(value: T, table: handlers(T, R)): R {
  switch (value.kind) { /* per-arm dispatch */ }
}
type CircleOnly = byKind(Shape, 'circle');
let area: handlers(Shape, float64) = {
  circle: c => Math.PI * c.radius ** 2,
  rect: r => r.width * r.height,
}; // all keys required → structurally total, without touching the enum/sealed exhaustiveness rule
```

Honesty note: the *body* of `match` faces the correlated-union problem — proving that `table[value.kind]` accepts *this* `value` — in both systems. TypeScript papers over it with the `as any`; here a `switch` over the narrowed arms (or the same cast) is required per instantiation. Builders compute the *shape*; they neither worsen nor fix checker-side correlation, and it is worth the comparison document saying so rather than implying the builder version checks something TypeScript's cannot. Meanwhile the required-keys handler object delivers TypeScript-style structural totality for literal-tagged unions *without* disturbing the proposal's rule that `switch` exhaustiveness belongs to enums and sealed classes — the totality lives in the object type, not the control flow.

### 4.8 Identity and relations

TypeScript has no type-equality operator, so the ecosystem uses a famous incantation that exploits how the checker relates generic function signatures:

```ts
// TypeScript
type IsEqual<A, B> =
  (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;
type Same = IsEqual<{ a: 1 }, { a: 1 }>; // true
```

```js
// Builder
A === B;                                              // structural identity via interning (nominal for class/enum)
Reflect.isAssignable(A, B) && Reflect.isAssignable(B, A); // mutual assignability, when equivalence-up-to-subtyping is meant
```

Interned identity is not merely shorter; it is *specified* (canonical forms, the recursive back-edge rewriting) where `IsEqual` is an artifact of checker internals that has changed behavior across TypeScript releases. The two spellings also make an actual distinction TypeScript users blur: identity and inter-assignability differ (e.g., across `readonly`), and the builder picks the one it means.

### 4.9 Higher-order type functions

TypeScript's type language has no lambdas: a type alias cannot be passed to another type alias. Abstracting over a type constructor (`F<_>` such that the same code works for `Array<_>`, `Promise<_>`, `Option<_>`) is the higher-kinded-types problem, and the community's answer — fp-ts-style URI registries — is an encoding heavy enough to be its own tutorial genre:

```ts
// TypeScript — the shape of the HKT workaround (abridged)
interface URItoKind<A> { Array: A[]; Promise: Promise<A>; }
type URIS = keyof URItoKind<any>;
type Kind<F extends URIS, A> = URItoKind<A>[F];
declare function lift<F extends URIS>(F: F): <A, B>(f: (a: A) => B) => (fa: Kind<F, A>) => Kind<F, B>;
// ...plus module augmentation to register each new constructor
```

Builders are functions, so type functions are first-class, composable, partially applicable, and storable — full stop:

```js
// Builder
export function mapUnion(T: type, f: (type) => type): type { return union(arms(T).map(f)); }
export function compose(...fs: [].<(type) => type>): (type) => type {
  return T => fs.reduceRight((result, f) => f(result), T);
}

const looseDraft = compose(partial, mutable);   // a stored, first-class type function
type Editable = looseDraft(User);
type AllDrafts = mapUnion(Shape, deepPartial);

export function deepMap(T: type, leaf: (type) => type): type { // HKT-flavored: parameterize the traversal by a type function
  const node = reflect(T);
  switch (node.kind) {
    case 'object': return objectOf(node.properties.map(p => ({ ...p, type: deepMap(p.type, leaf) })), node.indexSignatures);
    case 'array':  return arrayOf(deepMap(node.element, leaf), node.extent);
    case 'union':  return union(node.arms.map(arm => deepMap(arm, leaf)));
    default:       return leaf(T);
  }
}
type Stringly = deepMap(Config, () => string); // every leaf becomes string
```

Closures qualify as evaluable when everything they capture is (parameters, constants, evaluable functions), so `compose(partial, mutable)` is itself a compile-time value usable in type position. This is a capability class TypeScript does not have at all, and it changes how a type library is *organized*: instead of one bespoke mapped type per transformation, a handful of traversals parameterized by functions.

### 4.10 The obviated set, confirmed

The comparison table marks several TypeScript constructs **obviated**; the builder analysis does not reopen them, but it is worth confirming none of them secretly needs a builder. `satisfies` is a typed binding that checks without widening — `let config: Routes = {...}` already does this here because declared types don't erase the literal's own type. `as const` is obviated because literals propagate literal types and value types are immutable by layout. `<const T>` is obviated by value generics with SameValue identity. `NoInfer<T>` exists to *suppress* an inference site; with inference confined to explicit parameter positions and computed constraints (§3.5), the failure mode it patches — an inference site you didn't want participating — barely arises, and when it does, pinning the parameter explicitly (`f.<T>(...)`) is the direct spelling — and the identity builder `noInfer` reproduces TypeScript's suppression exactly (§6.3). `ThisType<T>` is pure checker behavior (contextual `this` inside object literals) with no structural content; it is out of scope for builders by the P4 boundary and needs its own checker feature — which §6.3 supplies declaratively, as a `thisType` field on signature nodes. `unknown` stays obviated by runtime-checked `any` boundaries.

### 4.11 Coverage summary

| TypeScript construct | Status under builders | Where |
|---|---|---|
| `keyof T` | built-in already; `keysOf` for computed operands | §4.1 |
| `typeof x` | `Reflect.typeOf` (existing) | §4.1 |
| Indexed access `T[K]` | `indexed(T, K)` | §4.1 |
| Mapped types, modifiers `±?`/`±readonly` | `mapProperties` + record edits | §4.2 |
| Key remapping (`as`, `as never` deletion) | computed `name`, `null` deletion | §4.2 |
| Homomorphic modifier preservation | default (records are copied) | §4.0, §4.2 |
| `Partial`/`Required`/`Readonly`/(`Mutable`) | one line each | §4.2 |
| `Pick`/`Omit`/`Record` | `pick`/`omit`/`record` (+ authored errors) | §4.2 |
| Conditional `T extends U ? X : Y` | `Reflect.isAssignable(T, U) ? X : Y` | §4.3 |
| Distribution (and suppressing it) | explicit `arms().map`/`filter` | §4.3 |
| `Exclude`/`Extract`/`NonNullable` | `exclude`/`extract`/`nonNullable` | §4.3 |
| `infer` (destructuring uses) | reflection accessors | §4.3 |
| `infer` on generic applications | `generic` node field | §4.3 |
| `infer` (abstract patterns) | optional `Reflect.matchType` | §4.3 |
| `ReturnType`/`Parameters`/`FirstParameter` | reflection | §4.3 |
| `ConstructorParameters` | constructor reflection | §4.3 |
| `InstanceType` | obviated (class *is* the instance type) | §4.3 |
| `Awaited` | `awaited` via `generic` + thenable reflection | §4.3 |
| Template literals over literal unions | string code | §4.4 |
| `Uppercase`/`Lowercase`/`Capitalize`/`Uncapitalize` | `mapLiterals` one-liners | §4.4 |
| Template literals over infinite sets | pattern metadata (`string.<{ pattern }>`), conservative `subtype` | §4.4 |
| Tuple surgery, variadic spreads | array methods on elements | §4.5 |
| Type-level arithmetic | actual arithmetic (value generics) | §4.5 |
| Recursive type functions (`DeepPartial`) | in-flight fixpoint | §4.6 |
| Growing recursion (`Paths`) | works; infinite cases hit budget (as TS hits depth) | §4.6 |
| Discriminated-union computation | `discriminants`/`byKind`/`handlers` | §4.7 |
| Type equality (`IsEqual` hack) | `===` / mutual `isAssignable` | §4.8 |
| Higher-kinded / type-level lambdas | first-class functions (TS **cannot**) | §4.9 |
| `satisfies`, `as const`, `<const T>`, `NoInfer`, `unknown` | obviated, confirmed | §4.10 |
| `ThisType` | declarative `thisType` + one contextual rule | §6.3 |
| `ThisParameterType` / `OmitThisParameter` | one-liners over the `thisType` field | §6.3 |
| Inference *through* a utility (`f<T>(p: Partial<T>)`) | inherent as stated; closed by §6.1's ladder | §5.1, §6.1 |
| Definition-site generic body checking | recovered in bounded form via checked contracts | §5.2, §6.2 |
| Checker-participating types | re-drawn: declarative facts + verified proposals | §5.3, §6.3 |

## 5. What JavaScript alone cannot do

The catalog establishes coverage; this section establishes limits. Each gap states why it is inherent to "types computed by opaque functions" rather than an API omission, how much it matters, and the bridge.

### 5.1 Inference through a builder's result

TypeScript can run a homomorphic mapped type *backwards* during inference:

```ts
declare function apply<T>(patch: Partial<T>): T;
const user = apply({ name: 'a' }); // TS infers T = { name: string } by inverting Partial
```

A builder is a black box; given a value of type `partial(T)`'s shape, no engine can reconstruct `T` by running `partial` in reverse, and no engine should try. This is the single most fundamental gap, and it is worth being precise about its size: TypeScript's inversion works only for *homomorphic mapped types* (not conditionals, not remapped keys, not most of the wider zoo), and the failing pattern requires `T` to appear **only** under the utility. The moment `T` also appears plainly — `merge<T>(base: T, patch: Partial<T>): T`, which is how such APIs are usually shaped — inference proceeds from the plain position and the builder result is merely checked, which works fine here.

Bridges, in order of preference: design APIs so the parameter appears at least once uninverted; require explicit specialization for the rare remainder (`apply.<User>(patch)` — noisier than TypeScript, never wrong); and for library authors who insist, `Reflect.matchType` supports writing an explicit inverse and dispatching on it. **Rejected bridge (as stated):** letting a builder *register* an inverse the engine consults *and trusts* during inference — its soundness is unverifiable (nothing forces the registered inverse to actually invert), and it reintroduces checker participation (§5.3) through the back door. Severity: low-to-moderate, concentrated in `.d.ts`-style API ports. §6.1 closes the remainder with three forward-verified mechanisms, the last of which repairs this rejected bridge's soundness defect rather than trusting it.

### 5.2 Definition-site checking of generic bodies, and identity-only deferral

TypeScript checks a generic function's body *once*, abstractly, against the parameter's constraint; errors surface at the declaration. This proposal's generics are fully specialized — bodies are checked per instantiation, C++-template style — and that stance predates builders. But builders amplify it: inside

```js
function sanitize<T>(value: T): omit(T, ['password']) { ... }
```

the type `omit(T, ['password'])` is, before specialization, an interned opaque application. The checker knows `omit(T, ['password']) === omit(T, ['password'])`, knows its declared kind is `type`, and knows nothing else — not that it is an object type, not its variance in `T` (which must therefore be treated as invariant pre-specialization), not whether the body's `return` value matches it. TypeScript's deferred conditional and mapped types are *also* mostly opaque in this position, but TypeScript retains slivers of reasoning (a deferred conditional's constraint is the union of its branches; homomorphic maps relate covariantly) that opaque applications surrender.

The consequences are real but bounded by the specialization model itself: every use in a running program reaches a concrete `T`, at which point everything is checked with full precision. What is lost is *early* diagnosis inside unspecialized library code — the same trade C++ made for two decades before concepts. Bridges: constraints give definition-site checking for the constrained surface (`<T: HasPassword>` lets the body use what `HasPassword` promises); a testing convention of *exemplar instantiations* (`sanitize.<TestUser>` in the library's own tests — cheap, since builders are testable functions, §3.4) catches body errors before users do; and checked contracts — `where` postconditions on builders, assumed before specialization because every evaluation re-verifies them — are the scoped extension, specified in §6.2. Severity: moderate for library authors, invisible to application code.

### 5.3 Participation in the checker's own inference

TypeScript evaluates conditional types *during* inference and overload resolution, and ships types whose entire content is checker behavior (`ThisType`, `NoInfer`). Builders run strictly *after* inference binds their arguments — a deliberate phase separation this design refuses to blur, because "user code runs inside the inference fixpoint" is where determinism, termination, and comprehensibility go to die. The proposal's sanctioned checker-integration points already exist and stay: `meta` protocol hooks (`subtype`, `narrow`, `validate`, `conversionFactor`) are user functions the checker calls at *specified* seams with *specified* obligations. The one inference behavior builders genuinely need — literal inference driven by computed constraints — is R5 and is a rule about constraints, not a hook. Severity: low; the constructs this forecloses are TypeScript's most fragile corner. §6.3 revisits the boundary and re-draws it — as a direction of trust — rather than removing it.

### 5.4 Infinite string languages

Covered in §4.4: not expressible as unions of literals, deliberately routed to pattern metadata instead of a new structural type kind, at the price of a conservative static `subtype` between patterns. The residue after that bridge is small — pattern-vs-pattern assignability weaker than TypeScript's template-literal cross-matching in some cases — and is repaid with regex-power patterns and runtime validation TypeScript cannot offer. §6.4 then sharpens the static judgment into an exact decision procedure on the regular fragment. Severity: low.

### 5.5 No minting of nominal types

Builders construct *structural* types. They cannot declare a class, an enum, or a `sealed` hierarchy, because nominal identity is declaration identity — a builder-minted class would be a macro system, with hygiene, layout, and toolchain consequences this proposal has consistently declined (the same line drawn at declaration merging: `partial class` adds methods, never fields). Branding, the main thing TypeScript users mint `unique symbol`s and fake-nominal wrappers for, is served *better* by primitive metadata (`decimal128.<{ currency: 'USD' }>` is a checked brand with conversion semantics, not a lie to the checker). Severity: low, and the boundary is a feature; §6.5 gives branding the first-class mechanism that makes it moot.

### 5.6 Symbol keys in computed key sets

Property records carry symbol names, so shape transformations handle them; literal types don't include symbols, so `keysOf`-style unions can't. Inherited from the `unique symbol` gap in the comparison table; a builder-era resolution would be admitting symbol literal types — a main-proposal question on which §6.6 now takes a position: admit them. Severity: low, then zero.

### 5.7 Termination, resources, and supply-chain exposure

Builders are Turing-complete on purpose, so compile time is now programmable — including by a hostile or merely careless dependency. Three honest observations. First, TypeScript is *already* in this position in practice: its type language is Turing-complete-with-limits, type-level bombs exist, and editors dutifully hang on them; the difference is degree and candor. Second, the static evaluability discipline means a builder cannot exfiltrate, phone home, or read the environment — the damage class is compile-time resource burn, not I/O. Third, the budget (§3.4) is the enforcement: spec'd defaults, host overrides, per-evaluation attribution so tooling can name the guilty call. What JavaScript alone cannot provide is a *termination guarantee*, and this design declines to pretend otherwise — as does TypeScript. Severity: manageable with budgets; must be in the spec, not left to hosts. §6.7 adds the one hard rule that keeps them sound: fuel decides whether compilation fails, never what it produces.

### 5.8 The tooling burden

Every tool that answers "what is the type here" — language servers, linters, bundlers-with-checks — must embed the compile-time evaluator. This is the real cost of the whole approach and it is honest to state it plainly, with its two mitigations. It is *not a new cost*: `componentType(C)` in an annotation, `multiplyDimensions(D, D2)` in an operator return, and every `meta` hook already require exactly this evaluator; builders add call surface, not machinery. And interning makes results cacheable across a session and serializable across builds (the interning table keyed by module-graph hash is the natural cache). What is genuinely new work: hover and error printers should show *both* the application and its expansion (`omit(User, ['password'])` = `{ id: uint32, name: string }`), go-to-definition on a computed type should reach the builder, and a stack of builder frames should attach to budget and thrown-error diagnostics. All specified in R10. Severity: significant engineering, zero novel research. §6.8's expansion artifact takes the evaluator off the common path.

### 5.9 Ecosystem interop

`.d.ts` files lean on mapped and conditional types, and nothing here executes TypeScript syntax. The bridge is mechanical for the majority: the catalog is, in effect, a translation table (homomorphic mapped type → `mapProperties`; distributive conditional → `arms().filter`; `infer` destructuring → reflection accessor; utilities → kit calls), and a converter can target it. The residue that does not translate: `ThisType`, `NoInfer`, declaration merging and global augmentation (non-goals here), assertion-function signatures (an existing table gap), and inference-inverted APIs (§5.1), which convert but demand explicit specialization at call sites. Untranslated boundaries remain what they are today: `any`/`dynamic`. Severity: moderate, tooling-shaped, and independent of the language design. §6.8 scopes the artifact and the two-way `.d.ts` bridge.

## 6. Closing the gaps: the maximal design

Section 5 was written under this document's conservative defaults. This section reopens each gap under a different mandate: adopt any feature that closes it, provided it adds little or no new syntax and keeps the properties that make builders trustworthy — determinism, termination under budget, and soundness. Working through the gaps one by one surfaces a single unifying discipline, so it is worth stating up front:

> **Builders may propose; the engine must verify.** User code never makes a checker decision the engine takes on faith. Every mechanism below is one of three shapes: *declarative data* the checker reads (never executes), a *proposal* the checker confirms by ordinary forward evaluation and assignability, or a *contract* that is assumed early only because every concrete instantiation re-checks it.

Under that discipline, the phase boundary of §5.3 does not have to be a wall; it becomes a direction of trust. Nothing here lets a builder *decide* anything — a lying builder can only make compilation fail, never make it wrongly succeed.

### 6.1 Inference through results — a three-rung ladder

**Rung 1: compute in the return, not the parameter (zero additions).** The inversion problem exists only when the computed type sits in a *parameter* position, where the engine would have to run the builder backwards during inference. Move it to the *return* position and everything is forward: the argument's type is inferred plainly, and the "inverse" is just another builder the author writes as ordinary structural code.

```js
// TypeScript: declare function apply<T>(patch: Partial<T>): T;
function apply<Patch>(patch: Patch): required(Patch) { ... }

const user = apply({ name: 'a' });
// Patch = { name: string } by plain forward inference; return type = required({ name: string })
```

For `Partial` the explicit inverse is the kit's own `required`; for a bespoke builder, the author writes its inverse the same way the builder itself was written — reflect, transform, `makeType`. No unification, no reversal, no new machinery. This covers the majority of real `.d.ts` signatures of this shape, because what callers usually want back is precisely a function of the argument they passed.

One correction this rung pins down for R8: `Reflect.matchType` patterns must be *structural* — nodes and slots. A deferred opaque application like `partial(⟨slot⟩)` is not a legal pattern, because evaluating a builder against an unbound slot is exactly the inversion nobody can perform. An "inverse via matchType" is therefore always written against the *shape the builder produces* (optional properties, a wrapped element, a tagged member), never against the builder application itself.

**Rung 2: trial specialization over closed candidate sets (an inference rule, no syntax).** When a generic parameter appears *only* under deferred applications, ordinary inference has nothing to bind it from — but if its constraint denotes a *closed, finite* set of types, the engine does not need inversion at all. It can evaluate forward per candidate. `sealed` classes already make such sets first-class (the direct-subclass set is fixed when the declaring module finishes evaluating), and a union constraint or an `enum`-of-types is the same thing spelled structurally:

```js
sealed class Shape { }
class Circle extends Shape { kind: 'circle' = 'circle'; radius: float64; }
class Rect extends Shape { kind: 'rect' = 'rect'; width: float64; height: float64; }

function apply<T: Shape>(patch: partial(T)): T { ... }

apply({ kind: 'circle', radius: 3 });
// trials: { kind: 'circle', ... } <: partial(Circle) ✓ ; <: partial(Rect) ✗ (kind conflicts) ⇒ T = Circle
```

The rule: enumerate the constraint's members in declaration order; for each candidate, evaluate the deferred applications forward (memoized) and check every argument; exactly one survivor infers, zero or several is a compile error that asks for explicit `.<T>`. It is overload resolution's logic pointed at specialization — deterministic, terminating in `|candidates|` memoized evaluations, and sound because nothing is accepted that forward checking did not accept. Two honest notes. Selectivity depends on the computed types being discriminating: an all-optional shape under width subtyping matches everything *unless* object literals are checked freshly (excess properties rejected), so this rule should ship together with literal freshness checking against candidate applications — a rule the proposal's typed layouts already impose for struct-backed types. And the candidate set must be closed; an open constraint like `<T: HasKind>` never trials. Multi-parameter sites need one more rule, and it is a determinism rule as much as a cost rule: trials are **separable**. A parameter is trialed against exactly the arguments whose computed types mention it alone, so `merge<A: Shape, B: Shape>(a: partial(A), b: partial(B))` runs |Shape| trials for `A` plus |Shape| for `B` — a sum, never a Cartesian product — and memoization makes every repeated `partial(Circle)` a table lookup after the first. An argument whose type mentions two or more un-inferred parameters (`x: pair(A, B)`) is not separable and does not trial; that site asks for explicit arguments. The whole phase runs under a **spec-fixed trial ceiling** — fixed, not host-tunable, for the same reason §6.7 fixes everything observable: whether inference succeeds must not vary by machine. Past the ceiling, the site bails to explicit `.<...>` with a diagnostic naming the constraint that blew it.

**Rung 3: verified inverses, declared at the builder (opt-in, last resort).** Section 5.1 rejected registered inverses because their soundness was unverifiable. That objection has a repair — never trust the inverse; treat it as an *oracle whose answer is checked* — and the binding has one right place: the builder's own declaration. An imperative registry call (`Reflect.setInverse(partial, ...)`) would make inference behavior depend on module evaluation order, invite conflicting registrations from unrelated modules, and hide a checker-relevant fact somewhere other than the declaration the checker is reading. A decorator has none of those failure modes — one possible author, no timeline, visible at the declaration — and it is existing machinery: [decorators.md](decorators.md) already decorates standalone functions (`@f // Reflect.Function`) and reserves `FunctionMetadata` for exactly this kind of declaration-attached, evaluable fact.

```js
@inverse(required)                              // or @inverse((Result: type) => ...) for a bespoke one
function partial(T: type): type { ... }                  // §4.2, quoted

function apply<T>(patch: partial(T)): T { ... }
const user = apply({ name: 'a' });
// 1. ordinary inference for T fails (T occurs only under partial)
// 2. propose: T₀ = inverse({ name: string }) = { name: string }
// 3. verify forward: { name: string } <: partial(T₀) ✓ ⇒ T = T₀; on ✗, fall through to "explicit .<T> required"
```

The engine consults a declared inverse only when ordinary inference (and any Rung-2 trial) has failed, calls it once per site (memoized, budgeted), and then performs the *same* forward evaluation and assignability check an explicitly specialized call would face. A wrong inverse can therefore cause inference to fail — never to succeed wrongly — which dissolves the soundness objection and leaves only the participation objection, answered by the propose/verify discipline above. For builders with several inferable slots, the inverse returns a record keyed by parameter name and the proposals are verified jointly; any miss falls back to explicit. Because this is the one rung that genuinely lets user code into inference (as a proposer), it is deliberately ranked last: adopt it when `.d.ts` porting demonstrates the need, not before.

### 6.2 Definition-site checking — checked contracts

The [dependent record types](dependentrecordtypes.md) extension already attaches `where` clauses to functions as evaluable predicates over parameters. One extension — allowing `return` as an expression inside such a clause, naming the returned value, a position where the token is otherwise ungrammatical — turns the same syntax into postconditions. On a builder, a postcondition is a fact about the *type it returns*:

```js
function omit(T: type, keys: [].<string | symbol>): type   // §4.2's omit, in its array form, quoted
    where Reflect.isAssignable(T, return)        // dropping properties widens: every T value is a result value
    where reflect(return).kind === 'object' || reflect(return).kind === 'union'
{
  const dropped = new Set(keys);
  return mapProperties(T, p => dropped.has(p.name) ? null : p);
}
```

The semantics are two-sided, and both sides matter. **Verified:** at every concrete evaluation, the clauses are checked against the actual result; a violation is a compile-time TypeError naming the builder and its arguments. Contracts are never trusted. **Assumed:** before specialization, the checker takes the clauses as known facts about the deferred application, which is sound precisely *because* of the verification side — any instantiation that would falsify the assumption is stopped at the builder, before code that relied on it runs. The net effect is the C++-concepts trade recovered inside the specialization model: definition-site checking against declared bounds, with per-instantiation verification as the backstop.

Direction is everything here, and it is easy to get backwards. An *upper* bound (`return <: partial(T)`) serves **consumers** of the result — code holding an `omit(T, keys)` value may use it as a `partial(T)`. Checking a generic body that **produces** the result needs a *lower* bound, and for `omit` the true one is `T <: return` (width subtyping: removing properties yields a supertype). That is what lets the motivating example finally check at its declaration:

```js
function sanitize<T>(value: T): omit(T, ['password']) {
  return { ...value };   // typed T; the contract's Reflect.isAssignable(T, return-of-omit) admits it pre-specialization
}
```

Contracts should ship *on the standard kit itself* — every §4 builder carries its natural bounds and kind facts, dogfooding the feature and giving downstream generic code a reasoned surface out of the box. Two limits, stated plainly: a contract is a per-evaluation proposition, so universally quantified knowledge — variance of `omit(T, K)` in `T`, parametricity — cannot be expressed as a checkable contract and remains absent before specialization (arms compared after specialization are simply evaluated and compared, so nothing is lost where it matters). And contracts do not verify the *builder's* body ahead of time either; for that, the exemplar convention gains a zero-syntax handle — a `@exemplars(TestUser, Point)` decorator (decorators exist) that forces specialization of a generic declaration at compile time, turning "test your library's generics" from advice into a checked annotation.

Beneath contracts sits the escape hatch the proposal already owns: the cast. `value as omit(T, ['password'])` inside a generic body is legal with a *deferred* application as its target — the cast is carried like any other unevaluated type and resolves at specialization, where it becomes the proposal's ordinary cast: discharged statically when the concrete relation holds, and a *checked* conversion with a diagnostic naming the cast at the consumer's instantiation when it does not. That is the honest division of labor. Contracts state what the author can promise and the engine can re-verify; `as` covers the remainder the author must assert — and because casts in this proposal are checked rather than erased, a wrong assertion is bounded to a TypeError at the cast instead of TypeScript's quiet unsoundness.

### 6.3 Checker participation, re-drawn

Three pieces close the remaining distance, each conforming to the propose/verify discipline.

**`noInfer` is already free.** Under R5, inference reads *plain* parameter positions; a computed position is opaque until arguments bind. An identity builder therefore reproduces TypeScript's `NoInfer` exactly, with zero additions:

```js
export function noInfer(T: type): type { return T; }

function update<T>(original: T, patch: noInfer(T)): void { ... }
update({ a: 1 }, { a: 2, b: 3 });
// T infers from `original` only ⇒ T = { a: float64 }; patch is then *checked* against noInfer(T) = T
```

The kit gains `noInfer`, and the idiom generalizes: any argument an author wants checked-but-not-inferred-from is wrapped in an identity application.

**`this` typing as constructed data.** `ThisType<T>`'s job — giving methods of an options object a contextual `this` — needs no checker-resident marker if function signature nodes carry an optional `thisType` and one contextual rule exists: a function literal checked against an expected signature adopts that signature's `thisType` for body checking (arrow functions excluded; their `this` stays lexical). Then the Vue-options pattern is a builder that *constructs* method types with the right `thisType`, and the checker merely reads the data:

```js
export function withThisType(F: type, Self: type): type {
  const node = reflect(F);
  return Reflect.makeType({ ...node, signatures: node.signatures.map(s => ({ ...s, thisType: Self })) });
}

export function options(Data: type, Methods: type): type {
  const self = Reflect.makeType({ kind: 'intersection', members: [Data, Methods] }); // the eventual instance shape
  return objectOf([
    prop('data', fn([], Data)),
    ...reflect(Methods).properties.map(p => prop(p.name, withThisType(p.type, self))),
  ]);
}
```

Two more `lib.d.ts` entries fall out of the field for free, and one boundary stays where the comparison table drew it:

```js
export function thisParameterType(F: type): type {
  return reflect(F).signatures[0].thisType ?? any;   // TypeScript says unknown; the runtime-checked any is the analog here
}
export function omitThisParameter(F: type): type {
  const node = reflect(F);
  return Reflect.makeType({ ...node, signatures: node.signatures.map(({ thisType, ...s }) => s) });
}
```

Polymorphic `this` — the fluent-builder return type — is *not* covered by `thisType` and remains the comparison table's separate small gap: it is an implicitly generic self type the checker rebinds per receiver, a checker feature in its own right, whereas `thisType` only records what a signature demands of its receiver.

**Narrowing signatures as declarative facts.** TypeScript's `asserts x is T` / `x is T` are not executed by its checker either — they are declared facts consumed by control-flow analysis. The same shape fits here: signature nodes gain an optional `narrows` record (`{ parameter, to }`, with an `asserts` flag for the throwing form), the checker applies it after a call exactly as it applies `instanceof` or `is`, and soundness stays where TypeScript put it — on the author of the function body, with the difference that here the declared fact is also *runtime-checkable* at the boundary if the host wants belt-and-braces. This closes the assertion-functions row of the comparison table, and it is worth noticing what it is not: no builder runs during narrowing; the checker reads a field.

With these in place, R9's invariant is restated rather than retired: **the inference fixpoint remains builder-free; every checker-visible effect of user type code is declarative data, a forward-verified proposal, or a per-instantiation-checked contract.** That sentence is the whole security model, and it is checkable against every mechanism in this section.

### 6.4 Infinite strings — decide the regular fragment

R7's conservative pattern-subtype judgment can be sharpened without giving up determinism. Patterns that stay inside the genuinely regular fragment — no backreferences, no lookaround — compile to finite automata, and `L(A) ⊆ L(B)` is then *decidable exactly* (complement `B`, product with `A`, test emptiness). The needed tiering rule is about size, not fuel: the spec fixes a constant (say, an automaton state bound after construction); pairs within the bound get the exact answer, pairs beyond it get the conservative syntactic answer. Because the tier is decided by syntactic size rather than by remaining budget, the judgment is identical on every host — the same determinism rule §6.7 enforces elsewhere. TypeScript's template-literal grammar (literal runs and `${string}`/`${number}` holes) sits comfortably inside the regular fragment, so this upgrade makes the pattern-metadata route *strictly stronger* than template-literal matching on TypeScript's own turf, while keeping the two things it already had over TypeScript: full regex expressiveness in `validate`, and real runtime enforcement. The kit exposes the general constructor alongside §4.4's `suffixed`:

```js
export function stringPattern(pattern: RegExp | TemplateStringsArray, ...holes: [].<type>): type {
  // Callable two ways: stringPattern(/^\d+px$/) with a regex, or as a template tag,
  // stringPattern`${string}-${uint32}`, the analog of a template literal type. In tag form each
  // literal chunk is escaped and each hole contributes the sub-pattern its type matches
  // (a literal contributes its escaped value, string contributes [\s\S]*, a numeric primitive -?\d+(\.\d+)?).
  const regex = pattern instanceof RegExp ? pattern : tagToRegExp(pattern, holes);
  return Reflect.makeType({ kind: 'parameterized', base: string, metadata: { pattern: regex } });
}
type Px = stringPattern(/^\d+px$/);   // Px <: stringPattern(/^.*px$/) now decided exactly, not conservatively
```

### 6.5 Branding — generalized parameterization

Branding was already routed to primitive metadata; the maximal design finishes the thought by generalizing the `parameterized` node to **any base type**, not just primitives, with the metadata protocol's `subtype`/`validate` hooks deciding relations between parameterizations. The default relation is itself the branding rule: `parameterized(T, M)` is assignable *to* `T` (upcasting sheds the brand freely) but `T` is not assignable to `parameterized(T, M)` unless a hook admits it — construction goes through the metadata boundary, where `validate` runs. The kit builder is three lines, and interning is what makes it a real newtype: `brand(uint32, 'UserId')` is *one* type everywhere it is written, in any module, without a registry.

```js
export function brand(T: type, tag: string | symbol): type {
  return Reflect.makeType({ kind: 'parameterized', base: T, metadata: { brand: tag } });
}
type UserId = brand(uint32, 'UserId');
function getUser(id: UserId) { ... }
getUser(7);            // TypeError: uint32 is not assignable to brand(uint32, 'UserId')
getUser(UserId(7));    // the metadata construction boundary — checked, not asserted
```

Unlike TypeScript's `unique symbol` intersection trick, the brand is not a lie told to the checker: the parameterization is visible to reflection, participates in `validate`, and can carry conversion semantics (`conversionFactor`) when the brand is a unit rather than a mere tag. The line against true nominal *minting* — classes, enums, `sealed` hierarchies from a builder — holds for the layout and hygiene reasons of §5.5; what this section establishes is that the thing people mint nominal types *for* did not need minting.

### 6.6 Symbol literal types — adopt them

The clean resolution to §5.6 is to stop treating it as open: admit `symbol` into the literal kind (`{ kind: 'literal', value: symbol, base: symbol }`), identity-compared like every other literal value. `keysOf` becomes total over symbol-keyed shapes, `pick`/`omit` accept symbol keys through the same overloads, and the comparison table's `unique symbol` row closes — a declared `const s = Symbol()` used in type position *is* the unique symbol type, without a keyword. Two consequences to specify rather than discover: canonical ordering (symbols sort after strings, by first-interning order within the agent — stable per session, which is all interning needs), and serialization (`Symbol.for` keys round-trip through the wire format by key; a type mentioning a unique symbol is agent-local, and the serializer says so by naming the symbol rather than failing opaquely).

### 6.7 Budgets — one hard rule

This subsection is a correction, because the tempting design is wrong. Making budget exhaustion *catchable inside evaluation* — `try { return paths(T) } catch (e) { return string }` — would make a program's **types depend on host fuel settings**: the same source elaborates differently on a large-budget CI box and a small-budget laptop, and the memoization table happily caches whichever answer was computed first. That is nondeterminism smuggled in through the error channel, and it breaks the property every other mechanism in this section was shaped to preserve. The rule, stated once: **fuel decides whether compilation fails, never what it produces.** Budget exhaustion is uncatchable from evaluated code and surfaces as an attributed diagnostic. Graceful degradation is still fully available — it just has to be *deterministic*, which means explicit and structural: builders take spec'd-value caps (`paths(T, { maxDepth: 4 })` computes the identical type on every host), carry visited-sets for cycle policy (§4.6 showed `paths` throwing an authored error on recursive input), or expose a `*Bounded` kit variant. Same ergonomics, no observable fuel.

Interactive tooling gets one further allowance, scoped so it cannot leak into semantics: **cancellation, never truncation**. Because builders are pure and memoized, an evaluator can checkpoint at memo boundaries, yield, resume, or abandon an in-flight evaluation at will — an editor keystroke never has to wait out `paths(T)`. What a language server does with an abandoned evaluation is display-side only: hover falls back to the unexpanded application form (`partial(User)`, which is already the specified display alongside its expansion), completion omits members not yet computed, and the request re-runs against the memo table a moment later. What it may not do is feed a partial result back into checking, or report diagnostics the build would not report. The editor is permitted to be *behind* the build, never *different from* it.

### 6.8 Distribution — the expansion artifact

The tooling costs of §5.8–5.9 shrink with one build artifact, honestly scoped. Because evaluation is deterministic and results are interned canonical forms, a package can ship its **expansion table**: the serialized interning table for every *concrete* type its public surface produces — including every closed builder application, pre-evaluated — keyed by a hash of the module graph that produced it. A consumer toolchain verifies the hash and reads types out of the table instead of evaluating; on mismatch it re-derives, which determinism makes reproducible. What the artifact deliberately cannot contain is the *open* applications — `partial(T)` inside an exported generic evaluates against the consumer's `T`, at the consumer — so the evaluator stays embedded in tools (R10 stands) and the artifact's job is to take the common path (a dependency's concrete API surface) off it. The same expansion machinery powers the `.d.ts` bridge in both directions: R11's converter translates TypeScript's mapped/conditional subset *into* kit calls, and the reverse emitter *prints* a `.d.ts` for TypeScript consumers by evaluating the public surface and writing out expansions — mechanical precisely where the translation table of §5.9 is.

The native-tooling objection deserves its own answer, because "every tool embeds an evaluator" reads as "every Rust language server embeds V8," and it should not. The evaluable subset is deliberately tiny *as a language*: no I/O, no ambient state, no `eval` or `Function` constructor, no `async`, no generators, no proxies — straight-line functions, closures over evaluable bindings, the immutable core library surface, and regular expressions (which R18 obliges a conformant evaluator to carry anyway). That is a small tree-walking interpreter in any implementation language, not an embedded JavaScript engine; and because evaluation is memoized and the artifact pre-expands dependencies, an interactive session evaluates each distinct local application once. R21 turns this from a hope into a guarantee: specify the subset's grammar and semantics normatively, as an implementation target in its own right, with a conformance suite — the determinism rules of §3.4 and §6.7 are precisely what make "my Rust evaluator agrees with the engine" a checkable claim.

### 6.9 Provenance — documentation through the round trip

A builder that cracks `User` open and rebuilds it as `partial(User)` should not cost the editor its memory of where `name` came from: hover on a `partial(User)` value's `.name` ought to still surface the doc comment and deprecation notice written on `User.name`, as TypeScript preserves through homomorphic mapped types. The tempting fix — a `jsdoc: string` field on `TypePropertyReflection` — is wrong in an instructive way: anything inside the node rides the round trip into `makeType`, and *type identity must not depend on comments*. Either the field joins the canonical form, and two structurally identical types with different documentation become different types (catastrophic), or interning must crown one winner's documentation for the shared object (silent, order-dependent data loss). Documentation is tooling data *about declarations*, not structure *of types*, and the channel has to say so.

The design that threads the needle is **provenance, not payload**: property records (and top-level nodes) carry an optional, explicitly *non-canonical* `origin` — a reference to the declaring site. Canonicalization ignores it for identity and *unions* it on merge, so an interned type accumulates the set of declarations it came from. Because `mapProperties` spreads records, origin rides through every homomorphic builder with zero effort — `partial(User)`'s `name` still points at `User.name` — while a property a builder *mints* (`getters`, key remapping) has no origin, and tooling falls back to the answer the diagnostics rule already mandates: the builder application that produced it. Doc comments themselves stay where they always lived, with the source, resolved from origins by the language server at display time; for dependencies without sources on disk, the expansion artifact carries extracted doc strings (§8).

## 7. Recommendations

**R1 — Add `Reflect.makeType` and complete the node model.** The inverse of `Reflect.getReflection.<Reflect.Type>`, canonicalizing and interning, with the round-trip law of §3.1. Extend nodes with: a `literal` kind carrying value and base; `readonly` and `initial` on property records; `indexSignatures` on object nodes; a `parameterized` kind for metadata parameterizations; `generic` (base plus arguments) on nominal applications; `initial` on tuple elements (the `[T, U = d]` trailing-default form); `rest` on `FunctionParameterReflection`; and a `Reflect.ClassConstructor` reflection context, since `ClassReflection` currently omits construct signatures. Give type objects a canonical `toString`. This single recommendation is the load-bearing one; everything in §4 is downstream of it.

**R2 — Expose assignability: `Reflect.isAssignable(source, target)`.** The existing coinductive judgment as an evaluable predicate. With interned `===` already covering identity, this completes the relations builders branch on.

**R3 — Admit `never`.** The empty union as a real, interned type object named `Reflect.never`, with the standard algebra (identity of union, annihilator of intersection, assignable to all, uninhabited), writable in annotations for honesty since computation makes it reachable regardless. Explicitly preserve the rule that `switch` exhaustiveness remains reserved to enums and sealed classes.

**R4 — Specify builder evaluation semantics.** In [typeobjects.md](typeobjects.md)'s compile-time type expression section: evaluation at specialization with *type* generic parameters named alongside value generics; memoization keyed on (function identity, SameValue/interned arguments); unevaluated applications carried as interned deferred types; the in-flight fixpoint with the unproductive-recursion `TypeError` and the reference-position validity check; the evaluation budget with spec'd defaults, host override, and attributed diagnostics; thrown exceptions surfacing as compile-time diagnostics prefixed by the originating type-position expression.

**R5 — Computed constraints, and literal inference through them.** A generic constraint may be a compile-time expression over earlier parameters in its list (`<T, K: keysOf(T)>`), and a value or type parameter constrained to a union of literal types infers *literally* from its argument. This is the only inference rule builders request and it mirrors the cue `K extends keyof T` gives TypeScript.

**R6 — Ship the kit as a standard module of plain evaluable JavaScript.** The §4 definitions — `keysOf`, `indexed`, `mapProperties`, `partial`, `required`, `readonly`, `mutable`, `pick`, `omit`, `record`, `exclude`, `extract`, `nonNullable`, `returnType`, `parameters`, `constructorParameters`, `awaited`, `flatten`, the tuple set, `mapLiterals` and the four case functions, `listeners`-style helpers, `deepPartial`, `paths`, `discriminants`/`byKind`/`handlers`, `mapUnion`, `compose`, and §6's `noInfer`, `brand`, `stringPattern`, `options`, `thisParameterType`, and `omitThisParameter` — under a `std:types`-style specifier reconciled with [standardlibrary.md](standardlibrary.md). Shipping it *as source* is the dogfood proof that no construct is engine magic, gives the ecosystem one interned vocabulary, and doubles as the reference documentation for R1's API.

**R7 — Route infinite string types to pattern metadata.** Specify a `StringPattern` meta type over the [regexp](regexp.md) extension: `validate` by matching; `subtype` conservative (syntactic containment for literal/class concatenations, reflexive otherwise), with the conservatism documented as intentional. Provide `suffixed`/`prefixed`-style kit builders. Do not add a structural template-literal type kind.

**R8 — Add `Reflect.matchType` and `Reflect.inferSlot` as optional completeness.** One-sided structural unification returning bindings or `null`. Low priority — the catalog never needed it — but it closes the abstract-`infer` question and gives §5.1's determined library author an explicit-inverse escape hatch. Do **not** add *trusted* engine-consulted inverses; the verified propose-and-check form is R15, and `matchType` patterns are structural only — never opaque applications (§6.1).

**R9 — Keep the phase boundary.** Builders never run inside inference, overload resolution, or narrowing; checker-integrated user code remains exclusively the `meta` protocol hooks with their specified obligations. State this as a design invariant in the spec text so future ergonomic pressure has something to push against — in the precise form §6.3 gives it: the inference fixpoint stays builder-free, and every checker-visible effect of user type code is declarative data, a forward-verified proposal, or a per-instantiation-checked contract.

**R10 — Write the tooling plan into the extension.** Language servers embed the evaluator (already implied by metadata and `where`); computed types display as application *plus* canonical expansion; go-to-definition on a computed type targets the builder; budget and thrown-error diagnostics carry builder stacks; the interning table is the cross-build cache, keyed by module-graph hash; and the recommended library workflow is unit-testing builders at runtime (§3.4), which no erased type system can offer. Two later refinements fold in: evaluation in interactive contexts is cancellable but never truncated (§6.7), and hover resolves documentation through the provenance channel (§6.9, R22).

**R11 — Build the `.d.ts` translation table as a tool, not a language feature.** A converter targeting the R6 kit for the mechanical majority (§5.9), with a documented unsupported residue. This is where TS-ecosystem compatibility is won or lost, and it needs none of the engine work to begin.

**R12 — Update the comparison document.** With R1–R7 accepted, the issue #64 table's **gap** rows for conditional types, mapped types and their modifiers, key remapping, `infer`, template-literal types, the string intrinsics, `Partial`/`Required`/`Readonly`/`Pick`/`Omit`, and `Exclude`/`Extract` generality become "via type builders (see this document)"; `NoInfer` folds into obviated; with §6 (R13–R20) accepted as well, the `unique symbol` and assertion-function rows close, §5.1 and §5.3 become opt-in features rather than gaps, and the honest residuals shrink to §5.2's universally-quantified knowledge (variance and parametricity before specialization) plus the deliberate non-goals.

**R13 — Specify checked contracts: `where` postconditions on builders.** Extend the function-level `where` clause of [dependent record types](dependentrecordtypes.md) so `return` names the returned value inside the clause. Semantics per §6.2: verified at every concrete evaluation (violation is a compile-time TypeError naming the builder and arguments), assumed over the deferred application before specialization. Ship the R6 kit *with* contracts on every builder — lower bounds where they hold (`T <: omit(T, keys)`), kind facts everywhere — and add the `@exemplars(...)` decorator that forces compile-time specialization of a generic declaration.

**R14 — Add trial specialization over closed constraint sets.** When a generic parameter occurs only in computed positions and its constraint denotes a closed finite set (a union of types, an enum of types, or a `sealed` class and its fixed subclass set), infer by forward-evaluating each candidate and checking; one survivor infers, otherwise require explicit `.<T>`. Deterministic, terminating, forward-verified. Ship alongside a literal freshness (excess-property) rule for object literals checked against candidate applications, without which all-optional shapes cannot discriminate. Trials are separable per parameter — arguments whose types mention two or more un-inferred parameters do not trial — and the phase runs under a spec-fixed ceiling, a sum across parameters and never a Cartesian product, so both the cost and the outcome are machine-independent.

**R15 — Add declared, verified inverses, last and opt-in.** An `@inverse(fn)` decorator on the builder declaration — standalone functions are already decoratable with a `FunctionMetadata` slot in [decorators.md](decorators.md) — rather than any imperative registry: the association is part of the declaration, has one possible author, and cannot vary with module evaluation order. Consulted only after ordinary inference and R14 trials fail; the proposal is confirmed by forward evaluation and assignability before it binds, so an incorrect inverse can only fail inference, never mislead it. Multi-slot builders propose via a record keyed by parameter name, verified jointly. Adopt when `.d.ts` porting demonstrates need.

**R16 — Admit symbol literal types.** `{ kind: 'literal', value: symbol, base: symbol }`, identity-compared; `keysOf` becomes total; the `unique symbol` comparison-table row closes. Specify canonical ordering (after strings, by first-interning order per agent) and serialization behavior (`Symbol.for` round-trips by key; unique-symbol types are agent-local with a named diagnostic). Supersedes the former open question.

**R17 — Generalize `parameterized` to any base type, and ship `brand`.** Metadata parameterization over objects, classes, functions — not only primitives — with the default relation being the branding rule (parameterized-to-base free, base-to-parameterized only through the metadata boundary) and hooks able to admit more. Layout-transparent by specification: metadata never changes representation. The kit gains `brand(T, tag)` and `stringPattern(regex)`.

**R18 — Upgrade R7's pattern subtyping to a decision procedure on the regular fragment.** Backreference- and lookaround-free pattern pairs within a spec-fixed automaton size bound get exact language-inclusion answers (complement–product–emptiness); beyond the bound, the conservative syntactic judgment applies. The tier is decided by syntactic size, never by remaining fuel, so the judgment is host-independent. This makes pattern metadata strictly stronger than template-literal matching on TypeScript's own grammar.

**R19 — Add declarative checker facts to signature nodes: `thisType` and `narrows`.** One contextual rule for each: a function literal checked against an expected signature adopts its `thisType` for body checking (arrows excluded), and a call whose signature declares `narrows` applies the declared narrowing afterwards, exactly as `is` and `instanceof` do. Data the checker reads, never code it runs; closes `ThisType`, `ThisParameterType`/`OmitThisParameter`, and the assertion-functions row.

**R20 — Define the expansion artifact.** A serialized, module-graph-hash-keyed interning table covering a package's concrete public types and closed builder applications; consumers verify the hash and read instead of evaluating, re-deriving on mismatch. Open applications still evaluate at the consumer, so R10's embedded evaluator remains; the artifact removes it from the common path and doubles as the engine of the reverse `.d.ts` emitter in R11's bridge.

**R21 — Specify the evaluable subset as a normative interpreter target.** Pin the grammar and semantics of compile-time-evaluable code exactly: no I/O or ambient reads (evaluability already says so), and explicitly no `eval`/`Function`, `async`, generators, or proxies; closures over evaluable bindings, the immutable core library, and regular expressions in. Ship a conformance suite so native toolchains implement the evaluator directly — a small tree-walking interpreter, not an embedded JavaScript engine — and can prove agreement with the engine, which the determinism rules make a checkable claim.

**R22 — Add the provenance channel.** An optional, non-canonical `origin` on property records and nodes: ignored by identity, unioned by interning, spread-preserved through builders. Documentation resolves from origins at display time; minted members fall back to the originating builder application; the R20 artifact carries extracted doc strings for dependency origins. Explicitly reject documentation *inside* the node model — type identity must never depend on comments.

## 8. Open questions

**Contract vocabulary creep.** R13 deliberately allows any evaluable predicate; watch whether practice demands universally quantified forms (variance declarations) that per-evaluation checking cannot honor — those would need a different, proof-shaped mechanism, or should stay absent.

**Inverse arity and overloads.** R15's record-of-proposals shape for multi-slot builders, and how inverse consultation interacts with overloaded builders: propose per overload and verify jointly, or refuse to consult?

**Freshness scope.** R14 needs excess-property (freshness) checking of object literals against candidate applications; decide whether that rule is trial-local or the language's general literal-checking rule — the typed-layout stance suggests general.

**Contextual `this` and lexical capture.** R19's adoption rule must be specified against arrow functions (excluded — their `this` stays lexical) and against methods extracted as values.

**Provenance across the wire.** R22's origins are declaration references; the R20 artifact must carry extracted documentation strings for them so hover works without dependency sources on disk, which makes doc extraction part of the artifact format — decide how much (raw comment text, or a structured subset) and how origins serialize alongside [serialization](serialization.md)'s forms.

**Property order.** Canonicalization must fix a deterministic property order for interning; source order is meaningful to humans and reflection. Proposal: intern on sorted order, *display and reflect* in construction order — needs a line of spec either way.

**Cross-realm and serialized types.** Interning is per-agent today; computed types that flow through [serialization](serialization.md) or workers need the canonical form to be the wire format, with re-interning on arrival. The named-back-edge canonical form appears sufficient; confirm against the threading extension.

**`keyof` in expression position.** `type keyof Point` works; should `keysOf` be spelled `keyof` uniformly (operator accepting a type-object operand) instead of shipping a kit function? Pure surface decision.

**Metadata inside walks.** `deepPartial` over a field typed `float32.<{ m: 1 }>` currently passes the parameterization through untouched (a `parameterized` node hits the default arm). Confirm that is the wanted default and that meta types can veto structural edits where they must (e.g., a builder should not be able to strip `nonZero` from a type used as a divisor without the change being an ordinary — checked — assignability question, which it is).

**Budget defaults.** Concrete numbers for the step and construction budgets, and whether they scale per module, per top-level evaluation, or per program. TypeScript's depth-50/count-5,000,000 style limits are prior art to calibrate against. R18's automaton size tier and R14's trial ceiling need the same treatment: one spec'd constant each.

## 9. Verdict

Measured against the issue #64 table, the builder model plus six additions — `Reflect.makeType` with a completed node model, `Reflect.isAssignable`, `never`, specified evaluation semantics with the in-flight fixpoint, computed constraints with literal inference, and the pattern-metadata route for infinite strings — covers every type-programming row currently marked **gap**, and does so with less mechanism than it replaces: no conditional-type semantics, no mapped-type syntax, no `infer` binder, no template-literal grammar, no intrinsic types. Where TypeScript built a second language and then a standard library inside it, this proposal builds a reflection API and lets the first language be the second. The exchange rate runs in the builder model's favor on expressiveness (arithmetic, parsing, higher-order type functions, authored diagnostics, testable type code) and against it on exactly three inherent points — inference through results, definition-site body checking, checker participation. Section 5 bounds and bridges them; §6 then closes them nearly outright under a single discipline — builders may propose, the engine must verify — through return-position inverses, trial specialization, verified inverse registration, checked `where` contracts, and declarative signature facts, with symbol literals, generalized branding, exact pattern subtyping, and the expansion artifact folding in the smaller residues. What survives as truly irreducible is universally quantified knowledge about a builder's unspecialized output: variance and parametricity live only as contracts re-checked at every instantiation, never as definition-time proofs. TypeScript's reason for not doing this — never execute user code in the checker — is a good reason that this proposal has already, deliberately, paid to remove: primitive metadata cannot exist without the evaluator. Type builders are the rest of what that purchase buys.
