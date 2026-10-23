# Imported method returns and namespace contracts

Import spelling does not change an established declaration's contract. A namespace member resolves through the module's export binding, including renaming, default export names and re-export chains. When that resolution establishes a class under the existing binding-stability rules, construction and static-member access retain the producer's class identity, generic environment and published member contracts. A namespace object is not an instance of each class it exports.

Participation remains separate from knowing a body's contributions. `class E { get() { return "ok"; } }` has no annotation-derived anchor and publishes no return contract. A Boolean destination checks the returned value at runtime, both locally and after import. Merely using a class receiver or exporting the class does not introduce participation. By contrast, a declared String result, a typed parameter that makes the method participate, or a return derived from a typed local, capture or field can establish a published result. An incompatible consumer of that known result is rejected at the applicable source-checking boundary, including in an uncalled function.

```js
// dep
export class E {
  text: string = "ok";
  get() { return this.text; }
}
// main: a type error before either module body begins
import * as ns from "dep";
const result: boolean = new ns.E().get();
```

The same rejection applies through `import { E }`, a renamed export of that declaration, or a namespace re-export. Compatible code keeps ordinary initialization order, temporal-dead-zone behaviour and runtime admission. No body is executed merely to discover a contract. Computed access must establish the export key under the existing constant-key rules; an arbitrary object or unknown key does not establish namespace identity.

## Participation directions

| Direction | Ergonomics / autocomplete / wrong guess | Prior art including Rust | Real-engine performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Publish every known method return | More completion detail, but ordinary JavaScript methods acquire enforced contracts and formerly runtime failures become source errors. | TypeScript infers returns broadly; its checks are erased. Rust function items use explicit output contracts, with unit when omitted. Neither supplies this proposal's gradual participation rule. | Expands analysis and runtime return checks; no hidden-class or IC benefit requires it. | Violates annotation-seeded participation; a known literal alone is not declaration provenance. | Reject. |
| Let an importing typed consumer make the producer participate | A method's displayed contract changes with its callers; independently checked consumers can disagree. | Rust declarations are not retyped by importers. TypeScript contextual typing is not permission to give one exported method incompatible caller-dependent contracts. | Requires cross-consumer coordination or contract variants without a shape benefit. | Confuses a destination's check with the producer's contract and generic environment. | Reject. |
| Require every exported method to write a return annotation | Clear interfaces, but exporting a class disables useful inference; the wrong guess costs a source rejection. | Rust's explicit outputs are a useful API analogy, but Rust does not preserve legacy untyped JavaScript. TypeScript allows inferred exports. | Reduces inference work, with no runtime layout advantage. | Removes supported participating inference and changes existing programs. | Reject independently of effort. |
| Preserve declaration-derived participation across import forms | Tools show the same established contract locally and through imports; untyped code remains dynamic. Known incompatible assignments fail at the existing boundary. | Rust illustrates declaration-owned contracts; TypeScript illustrates inference across modules. Their inference and enforcement policies differ here. | Retain compiler metadata and resolved export edges; no new object shape, IC family or runtime contract for untyped methods. | Preserves participation, nominal/generic identity, effects and reference liveness. | **Choose independently of effort.** |

## Namespace resolution directions

A valid solution must use the resolved export, distinguish constructor values from instances, preserve producer context, respect mutable bindings and shadowing, and avoid evaluating user code to discover a type.

| Direction | Ergonomics / autocomplete / wrong guess | Prior art including Rust | Real-engine performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat namespace class access as unknown | Import refactoring removes completion information and postpones known failures until execution. | Rust module paths and TypeScript namespace imports preserve resolved declarations. Their runtime namespaces differ, but losing a known contract follows from neither. | Less checking work, more runtime fallback; no shape or IC justification. | Conflicts with import-spelling invariance. | Reject. |
| Match public property names against producer local names | Simple names work; renaming, default names and barrels unexpectedly lose contracts. | Rust paths and ECMAScript export resolution distinguish public names from local bindings. | A name map can cache completed resolution; spelling alone is insufficient. | Conflates names with binding identity and loses re-export provenance. | Reject. |
| Treat any matching object property as a permanent class origin | Convenient completion for lookalike objects, but mutation can replace the constructor under the assumed type. | Rust module paths are not mutable object fields. A TypeScript structural shape is not namespace identity proof. | A property IC alone cannot establish the required source binding proof. | Violates mutation, shadowing and declaration-identity rules. | Reject. |
| Evaluate values or constructors to learn their contracts | Appears precise after execution but can trigger effects or TDZ failures while checking. | Rust and TypeScript source checking do not run ordinary constructor bodies to type imports. | Executes arbitrary work and complicates effect/cache handling. | Violates the checking phase and can consume or invalidate references. | Reject. |
| Resolve namespace exports and retain proven declaration metadata | Named and namespace access expose the same contract; immutable aliases preserve evidence while arbitrary mutable values retain dynamic checks. | Rust paths are a limited identity analogy; ECMAScript export resolution supplies the actual binding relation. | Linkers can memoize export edges and compiler records. No program-visible property or new IC family is needed. | Preserves nominal/generic environments, binding stability, evaluation and reference liveness. | **Choose independently of effort.** |

These rules do not claim that every constructor-valued binding has a proven exact class origin. Complete the applicable binding and publication judgments; an unavailable analysis result is not a new semantic exemption. Getter receiver inference and publication of exported constructor aliases remain distinct audit obligations.

Sources: [Rust function contracts](https://doc.rust-lang.org/reference/items/functions.html), [TypeScript inference](https://www.typescriptlang.org/docs/handbook/type-inference.html), and [ECMAScript export resolution](https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#sec-resolveexport). Participation and liveness policy remain this proposal's own.
