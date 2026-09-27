# Declared defaults and anonymous family constraints

A concrete application orders positional and named arguments, fills omitted arguments from declared defaults in declaration order, validates kinds and domains, and only then canonicalizes and interns. A missing required argument is an error even in an unused annotation. Lexical lookup happens first: an intrinsic spelling never overrides a user declaration or an enclosing type parameter.

| Declaration | Required arguments | Defaults |
| --- | --- | --- |
| `int`, `uint` | `N`, width 1–65536 | None |
| `rational` | `N`, width 1–65536 or `bigint` | None |
| `complex` | None | `T = number` |
| `vector` | Valid lane `T`, positive count `N` | None |
| `Range` | Element `T` | `S = Closed`, `E = Open` |
| `RangeFrom` | Element `T` | `S = Closed` |
| `RangeTo` | Element `T` | `E = Open` |
| `RangeFull`, `RangeBounds`, named interval aliases | Element `T` | None |

`complex`, `complex.<>`, and `complex.<number>` are one type, including when used as bounds. Bare `rational` has no concrete meaning: choose `rational64`, another explicit width, or `rational.<bigint>`. Internal declarations remain available for explicitly higher-kinded arguments. User-class constructor inference and injected class names keep their existing rules; numeric construction gains no inference rule.

## Bounds

In an argument position within a type parameter constraint, `_` matches exactly one argument admitted by the declaration's parameter. It introduces no source-visible name and is not a type. Each written hole is independent.

```js
function integer<T: type extends uint.<_>>(x: T): T { return x; }
function fraction<T: type extends rational.<_>>(x: T): T { return x; }
function lanes<T: type extends vector.<float32, _>>(x: T): T { return x; }
```

Omission is different from a hole. `Range.<_>` keeps the Closed/Open defaults; `Range.<_, _, _>` leaves all three positions open. For `class Pair<T: type, U: type = T> {}`, `Pair.<_>` requires equal arguments, whereas `Pair.<_, _>` does not. Defaults are evaluated forward under candidate bindings with ordinary type-evaluation budgets. An error or exhausted budget aborts checking; it never makes another overload win.

Patterns support intrinsic families, nominal classes and library applications, nesting, named arguments, and transparent forwarding aliases. `type Same<T: type> = Pair.<T, T>` used as `Same.<_>` retains the equality. A forwarding alias may use fixed arguments and direct parameter references. Structural interface existential matching, inversion of computed aliases, named captures in these bounds, variadic heads, and wildcard packs are outside this version and produce diagnostics. Ordinary concrete structural bounds remain available.

`any` keeps its existing concrete-view semantics outside a pattern. Within a pattern, a fixed `any` argument means precisely that type: `Pair.<any, _>` does not match `Pair.<uint8, string>`. It is not another wildcard.

## Matching and selection

A candidate supplies its canonical application or a supported nominal ancestor application. The intrinsic range shapes also supply their `RangeBounds.<T>` witness. Match heads by declaration identity, fixed entries by canonical argument identity, holes by parameter domain, and repeated entries by equality. Nested applications are compared recursively. Matching neither converts values nor changes the candidate's inferred type or metadata.

Specificity is proven inclusion: fixed arguments narrow wildcard positions; supported equalities and ancestry participate. An opaque computed default supplies no invented inclusion proof. Incomparable applicable overloads remain ambiguous, regardless of declaration order. A bound supplies no permission to combine different widths or to widen a reference's invariant location type; reference liveness is unchanged.

The logical pattern is separate from a concrete Type Object. Reflection of generic parameters carries a tagged `family-pattern` constraint with an application head and fixed, wildcard, or declaration-default entries. Wildcard indices preserve equality. Deferred defaults retain their declaration context. Rebuilding a reflected generic signature preserves its constraints and type identity. Passing a pattern by itself to `Reflect.makeType` is an error.

Concrete intrinsic reflection exposes the family name, complete arguments, and parameter names/domains/defaults. `Reflect.makeType({ kind: "generic", base: "rational", arguments: [64] })` is `rational64`. Neither reflection nor interning first constructs an invalid bare rational. Canonical display prefers standard aliases independently of user-defined aliases and import order.

## Deliberate boundaries

The selected direction uses declared defaults consistently and explicit holes for family-wide bounds. Rust's explicit-width aliases and separate type/const arguments are useful precedents, but Rust's `_` is an inference placeholder, not this constraint operation. General existential structural search and computed-alias inversion are deferred because they lack the direct nominal witness and bounded inclusion proof used here. Adding a default element type for `..`, a public first-class family value, mixed type/value domains for user generics, or numeric constructor inference would each change a separate contract and is deferred.
