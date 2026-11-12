# Dependent Record Types

> **This document describes an EXTENSION.** The `where` clause it uses - a
> predicate over a record's own fields, as `} where this.country == 'US' { … }` -
> is not part of the core specification, which places it with "the dependent
> record type extension, checked at the boundaries `#table-check-sites`
> enumerates rather than at specialization" and lists it among "the obligations
> this specification has incurred outward, not a specification of the extensions
> themselves". An engine implementing only the core will refuse these examples,
> and that refusal is correct rather than a defect.
>
> The two `where` forms the core DOES specify - a compile-time bound over generic
> parameters, and a contract naming `return` - are unrelated to this one beyond
> sharing the keyword: they are checked at specialization and at evaluation
> respectively, and a shared keyword is not a shared rule.

In addition to [primitive metadata](https://github.com/sirisian/ecmascript-types/blob/master/primitivemetadata.md) it's possible to have minimal syntax to define dependent record types and further improve refinement typing. Taken from JSON Schema the goal is to support dependentRequired, dependentSchemas, and if/then/else. Consider the following examples:

```js
// dependentRequired
{
	"type": "object",
	"properties": {
		"name": { "type": "string" },
		"creditCard": { "type": "number" },
		"billingAddress": { "type": "string" }
	},
	"required": ["name"],
	"dependentRequired": {
		"creditCard": ["billingAddress"],
		"billingAddress": ["creditCard"]
	}
}
// dependentSchemas
{
	"type": "object",
	"properties": {
		"name": { "type": "string" },
		"creditCard": { "type": "number" }
	},
	"required": ["name"],
	"dependentSchemas": {
		"creditCard": {
			"properties": {
				"billingAddress": { "type": "string" }
			},
			"required": ["billingAddress"]
		}
	}
}
// if/then/else:
{
	"type": "object",
	"properties": {
		"streetAddress": {
			"type": "string"
		},
		"country": {
			"default": "US",
			"enum": ["US", "CA"]
		}
	},
	"if": {
		"properties": {
			"country": { "const": "US" }
		}
	},
	"then": {
		"properties": {
			"postalCode": { "pattern": "[0-9]{5}(-[0-9]{4})?" }
		}
	},
	"else": {
		"properties": {
			"postalCode": { "pattern": "[A-Z][0-9][A-Z] [0-9][A-Z][0-9]" }
		}
	}
}
```

The proposed syntax is to use ```where``` and ```is``` to define these dependencies.

```js
// dependentRequired
type Payment = {
	name: string,
	creditCard?: number,
	billingAddress?: string,
} where (this.creditCard != null) == (this.billingAddress != null);

// dependentSchemas
type PaymentWithSchema = {
	name: string,
	creditCard?: number,
	billingAddress?: string,
} where if (this.creditCard != null) {
	this.billingAddress is string
	// or
	//this is { billingAddress: string }
};

// if/then/else
type USPostalCode = string.<{ pattern: /^[0-9]{5}(-[0-9]{4})?$/ }>;
type CAPostalCode = string.<{ pattern: /^[A-Z]\d[A-Z] \d[A-Z]\d$/ }>;
type Address = {
	streetAddress: string,
	country: 'US' | 'CA',
} where if (this.country == 'US') {
	this is { postalCode: USPostalCode }
} else {
	this is { postalCode: CAPostalCode }
};
```

In the ```Address``` code it should be noted that this design means developers don't have to use discriminated unions. For reference:

```js
type Address =
	| { streetAddress: string, country: 'US', postalCode: USPostalCode }
	| { streetAddress: string, country: 'CA', postalCode: CAPostalCode }
```

Also to be clear multiple ```where``` clauses can be used.

```js
type ... = {
}
where if (...) {} else {}
where if (...) {} else {};
```

Each clause is judged under the clauses before it, whichever form they are written in. An ```if``` clause passes on what the conditional expression over the same test and branches would, the join of its branches under their test, so a later clause that no value of either branch satisfies is an error where it is written:

```js
type Level = uint8
where if (this > 100) { this > 200 } else { this < 50 }
where this > 60 where this < 90; // TypeError: no value the first clause admits lies between
```

A branch written ```false``` forbids its case and so contributes nothing, and a clause with no ```else``` holds wherever its test fails.

#### Why a conditional clause is read as the conditional expression

A valid design gives one predicate one answer however it is spelled, covers branches that read a place the test does not, and adds no second rule beside the flow join.

| Direction | Ergonomics | Prior art and limits | Real-engine cost | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| The clause establishes nothing for later clauses | A contradiction is found at the first call, while the ternary spelling of the same clause is refused where it is written | No surveyed language checks one conditional spelling and not the other | None | Contradicts judging each clause under the clauses before it | Rejected: one predicate, two answers |
| Project one ordered place, as a refinement read through a value is projected | Fixes a clause over one place; a conditional whose branches read a second place is still accepted in silence | The proposal's own projection rule, which is single-place because a refinement is read later through a value that may have changed; a clause sequence is evaluated at one instant | A second walker beside the flow join | Sound, but leaves the ```if``` form weaker than ```?:``` and a disjunction in the same sequence | Rejected: three readers of the form, two rules |
| Read it as the conditional expression: the join of each branch under its test | ```if``` and ```?:``` always agree, over any number of places | Rust has one conditional, the ```if``` expression, so the split cannot arise there; the analogy is exact on unity and says nothing about value ranges, which Rust derives from neither. JSON Schema's ```if```/```then```/```else```, where this form comes from, validates as (if and then) or (not if and else) | Reuses the existing join; no new state | It is the ordinary join of narrowing facts; an explicit ```false``` branch is an impossible result and contributes no predecessor | **Recommended and implemented** |
| Remove the form and write the ternary | One spelling fewer | Rust again has only one | Simplest | Loses what is keyed on the form: the discriminated chain, and branches that are predicates themselves, including a ```match``` with its implicit default | Rejected: the form carries meaning the ternary does not |

### Narrowing

A `where` clause validates a value at a boundary. It does not keep a mutable object valid between boundaries. Testing one member therefore cannot, on its own, prove another member's current value:

```js
type Contact = { kind: string, email: string | null }
where if (this.kind === 'email') { this.email !== null } else { true };

function edit(c: Contact) {
    c.email = null; // Temporary invalid states remain permitted.
    if (c.kind === 'email') {
        // c.email still has type string | null.
    }
}
```

An alias can perform that write too. A getter can even pass the predicate and return `null` on its next read, with no intervening assignment. `const` and a structural `readonly` spelling do not establish a stable data property. For an ordinary record, capture and check the value you intend to use:

```js
function show(c: Contact): string {
    if (c.kind === 'email') {
        const email = c.email;
        if (email !== null) return email;
    }
    return 'none';
}
```

Automatic cross-field narrowing is available for stable observations through composite data fields:

```js
type ContactShape = { kind: string, email: string | null };
type StableContact = Composite.<ContactShape>
where if (this.kind === 'email') { this.email !== null } else { true };

function showStable(c: StableContact): string {
    if (c.kind === 'email') {
        const email: string = c.email;
        return email;
    }
    return 'none';
}
```

Each intermediate field on a projected path must itself contain a composite. A composite can hold an ordinary Object through an `any` or nominal leaf; that object's mutable properties do not gain this guarantee. Optional composite fields read `undefined` when absent. A structural assertion projects required members individually, without replacing the receiver's type. A branch conjunction contributes both assertions. Nested conditional predicates and an expression arm of an ordered `where match` can select a branch; an unknown earlier match arm prevents selecting a later one.

Branch selection uses existing flow facts and a finite syntax fragment: literal equality and inequality, nullish comparisons, decided `is` type or literal-pattern tests, Boolean literals, parentheses and built-in `!`, `&&` and `||`. Match selection recognizes literal and wildcard patterns and their `not`, `and` and `or` combinations. Helpers, extractors, block arms and other opaque computations supply no inferred facts. Checking does not execute a predicate to discover its consequences. Asserted type names keep their declaration scope and generic arguments. Ordinary `Reflect.makeType` callbacks remain opaque; private provenance on a source clause's bridge is not a new public type distinction.

These are ordinary flow facts. Joins retain only facts justified on every incoming path, and rebinding, reference effects and applicable call effects invalidate them. Nothing removes predicate evaluation at required boundaries or reference-liveness checks. This qualification also applies to any payload-shape facts inferred from a discriminated `where` chain; closed discriminant coverage alone is not a proof of mutable payload contents.

#### Why this direction

A valid design must preserve temporary invalid states, getter and exception behavior, declaration-scoped type identity, and reference liveness. It must produce no fact about a replacement value, require no arbitrary predicate execution during flow inference, and impose no new object layout or per-read validation protocol.

| Direction | Ergonomics | Prior art and limits | Real-engine cost | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Infer from every declared refinement | Attractive autocomplete, but can claim a nullable field is non-null and reject reachable code | Superficially resembles TypeScript discriminant narrowing; an arbitrary mutable predicate is a weaker premise | Cheap checking, but unsafe facts cannot justify optimization | Fails mutation and getter cases | Rejected: an admission is not a permanent invariant |
| Enforce the predicate after every write | Strong field correlations, but multi-field updates cannot pass through invalid states | Rust enums preserve variant structure; they do not implement arbitrary JavaScript getter predicates | Guards on aliases, setters, descriptors and proxies; adds shape and inline-cache dependencies | Contradicts the chosen boundary-only mutation model | Rejected: changes the meaning of ordinary updates |
| Reevaluate on every projected read | Appears convenient until a read throws or repeats user effects | Unlike an explicit smart cast or a Rust pattern match | Repeated predicate execution, calls and deoptimization dependencies | Changes getter counts, exception timing and boundary placement | Rejected: observable semantic change |
| Project through stable composite fields | Precise autocomplete where justified; mutable records use an explicit local test | Kotlin's stable smart-cast requirement is a close analogy; Rust's variant and borrowing guarantees illustrate why stability matters | Static path checks and ordinary flow facts; no additional runtime property protocol | Preserves boundaries, aliases, identity and liveness | **Recommended and implemented** |
| Track mutable data-only admissions and their dependencies | Useful for a proved local object, but an ordinary parameter can still receive getters or aliases | Kotlin also permits effectively immutable local variables; Rust supplies stronger ownership evidence | Static alias and effect dependencies; no runtime guard is needed when the entire proof is static | Sound only with an established admission, data descriptors and preserved dependencies; the annotation alone supplies none of these | Rejected as a general annotation guarantee; ordinary local flow analysis remains available, and additional descriptor-based proofs can be specified separately |
| Remove all automatic cross-field narrowing | Uniform but needlessly repeats checks on stable data | Explicit guards remain valid in all three languages | Lowest implementation cost, but forfeits available facts | Correct but unnecessarily restrictive | Rejected: stable observations already justify useful narrowing |

[Rust enums](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html) associate payloads with variants, while [borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) constrains mutation through aliases. This proposal's ordinary objects have neither guarantee. [Kotlin smart casts](https://kotlinlang.org/docs/typecasts.html) exclude custom getters and mutable properties that cannot be proved stable. [TypeScript's aliased-condition analysis](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-4.html) motivates the usability goal, but its discriminant analysis is not evidence that a boundary-checked JavaScript predicate remains true.

### Discriminated where-chains

A ```where``` chain that switches on one member against constants is a discriminated union written from the inside, and where the chain qualifies it *denotes* that union. It qualifies in either of the two forms a chain is written in.

The ```if```/```else``` form qualifies when every condition is an equality test between a member of ```this``` and a literal or enumerator constant, every condition tests the same member, that member's declared type is a closed set of those constants - a literal union or an ```enum``` - and the chain is total, meaning it ends in ```else``` or its conditions exhaust the member's type. The ```where match``` form qualifies when its subject is one member of ```this``` of such a type, every clause is unguarded against a constant of it or an ```or``` of constants, no constant is named twice, and the clauses name them all; each clause is a branch, and a clause naming several constants contributes a member for each. Each branch contributes the declared members with the discriminant narrowed to the constant that branch tested, plus whatever shape the branch asserts; a branch predicate that isn't a shape assertion rides along as a refinement of that branch and contributes no members. A branch's assertions can narrow a consumer's fields only under the stable-observation requirements above. The declaration's own predicate is checked under the ordinary facts of the tests it actually performs.

```js
// Address above qualifies: one member, two literal constants, a final else.
// It denotes:
//   | { streetAddress: string, country: 'US', postalCode: USPostalCode }
//   | { streetAddress: string, country: 'CA', postalCode: CAPostalCode }
```

Which is the union printed for reference above, so the equivalence that section asserts is a rule rather than a remark. What it buys is the property an ```enum``` and a sealed class already have and this form did not: [pattern matching](patternmatching.md) checks a ```match``` over an ```Address``` exhaustive from one arm per country with no ```default```, and adding ```'MX'``` to ```country``` becomes a compile-time error at every such ```match``` instead of silently routing through a default clause.

A chain that doesn't qualify denotes nothing but itself: a ```!=``` or ordering condition, conditions over two different members, or a chain with no ```else``` over a member whose type isn't exhausted. Those keep every other behavior described here - the predicates run at boundaries, ```is``` evaluates them, stable observations can supply field correspondences - and only forgo the totality claim, so code matching on them writes a ```default```. Nothing is lost by declining to normalize; the alternative is a checker that reasons about predicates on a budget, and no other rule in this proposal asks for one.

The derived union is used for checking, not as the type's identity. ```Reflect.typeOf``` reports the dependent record type, assignability compares against it, and ```Address``` remains one type object rather than two.

A chain's tests are judged like any other test. A condition the discriminant's type cannot satisfy, such as `this.country == 'MX'` over `'US' | 'CA'`, is an error where it is written, and so is a `where match` clause whose pattern cannot match; the implicit `default: false` of a `where match` spares it only the exhaustiveness check. The last test of a discriminated chain, whose every condition is one equality comparison of the same member, is the one exception when its conditions exhaust the type, since it is certain by construction: `if (this.country == 'US') { ... } else { if (this.country == 'CA') { ... } }` is accepted, while an `else` after that last test is an error.

A generic alias may refine its parameter: `type NonEmpty<T: type> = T where this.length > 0;` declares a generic dependent record type, and `NonEmpty.<string>` refines `string` with that predicate. Each application is its own type, identified by the declaration and its arguments, its clause is judged once at the declaration through each parameter's bound, and every boundary enforces it as for a non-generic refinement. Testing a value of the base with `is` against a refinement decides a branch, and a passing test narrows the value to the refinement.

A declaration's `where` clauses read a refined subject through its refinement: with `type Small = uint8 where this < 10;`, `function f(x: Small) where x > 100` can never hold and is refused where it is written, as the same two limits written as clauses on a `uint8` are. A refinement's own clause is never assumed while that clause is being judged.

### Assignment design choices

In the previous example for ```Payment``` setting either ```creditCard``` or ```billingAddress``` would place an instance into an invalid state. (Both need to be defined).

```js
type Payment = {
	name: string,
	creditCard?: number,
	billingAddress?: string
} where (this.creditCard != null) == (this.billingAddress != null);

const payment: Payment = {
	name: 'Alice'
}; // Valid: Neither creditCard nor billingAddress are present.

payment.name = 'Bob'; // Valid: name has no constraints
```

#### Solution

Fields referenced in a ```where``` clause are dependent: they cannot be validated on independent assignment, so the clause is checked at boundaries instead - construction, function calls, and object assignment. This is the rule the examples in this document assume.

Two more choices the same rule settles. A dependent record type's identity includes its declaration activation and generic arguments: the predicate is a function value compared by SameValue, so two textually identical ```where``` blocks in two modules are two types, and sharing one means declaring once and importing. And a ```where``` block is one predicate: conjunction is written by nesting one refined type in another, and a multi-assertion block would need a separator the grammar does not have.

The spread operator can be used to set a fully valid object:
```js
const updatedPayment: Payment = {
	...payment,
	creditCard: 4111222233334444,
	billingAddress: '123 Main St'
}; // Valid: Both fields are provided. The where clause evaluates to true.

const invalidPayment: Payment = {
	...payment,
	creditCard: 1234123412341234
}; // Compile or runtime error: 'where' clause validation failed at construction. (In this case it would be a compile time error).
```

#### Intersecting refinements

An intersection of refinements of one base admits the values every member admits. It stays an intersection: each refinement keeps the identity of its declaration, so a value of ```A & C``` is still a value of ```A``` and of ```C``` by name.

```js
type A = uint8 where this < 10;
type B = uint8 where this > 100;
type C = uint8 where this < 50;

type E = A & B;              // TypeError: no value is of both A and B
let x: A & C = 5;            // admitted: 5 satisfies both
let y: A & C = 20;           // TypeError: 20 fails A
function f(v: A & C) where v > 20 {} // TypeError: v is read through both, and nothing below 10 is above 20
function g(v: A): void { if (v is B) {} } // not judged: a test in a body writes no intersection
```

Two refinements written in source are judged together as one sequence of clauses over ```this```. Where they leave some place no value the pair is a contradiction written in source, and is an error where it is written; an intersection a computation constructs is not judged.

A valid design makes the checker and the run time agree on every value, admits the values an intersection has, refuses a contradiction where it is written, keeps a declared refinement's identity, and never lets a type's identity depend on a prover.

| Direction | Ergonomics | Prior art and limits | Real-engine cost | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Never compare the two refinements | ```let x: A & C = 5``` is refused though 5 is of both, and ```A & B``` with no values is accepted | None | Two predicate lists reached through an intersection at every check | The checker and the run time disagree on ```A & C``` | Rejected: the defect this section closes, in both directions |
| Give ```where``` a ```meet``` that answers one refinement holding both predicate lists | ```A & C``` would behave as ```uint8 where this < 10 where this < 50``` | In Rust ```T: A + B``` is the conjunction of two bounds; Liquid Haskell and Ada conjoin predicates when a refinement is refined again | One refinement record in place of two | A declared refinement has its declaration's identity. One merged type is neither ```A``` nor ```C```, so a value of it would stop being accepted where ```A``` is asked for | Rejected: it was the first recommendation, and it does not survive declaration identity |
| Keep the intersection, read every member wherever a refinement is read, and judge a written pair as one clause sequence | The examples above | The proposal's own rule that a contradiction written in source is refused while a computed empty type is not reduced | An intersection walk in the three static paths that read a refinement: admitting a constant, projecting clauses, and judging the written pair | Identity is untouched; the run time already checks each member | **Implemented** |
| A ```meet``` that answers no type where the two sets are disjoint | A constructed empty pair silently becomes ```never``` | TypeScript reduces ```string & number``` to ```never```, by structure and not by proof | The interval reasoning would have to run when types are built | Makes a type's identity depend on a prover, and two engines would have to agree on it at run time | Rejected |
| Forbid intersecting two refinements of one base | The user writes one refinement by hand | None | Simplest | Generic code that intersects two type parameters could not be instantiated with two refinements | Rejected: breaks composition |

#### A constant is decided where it is written

When the value crossing into a refinement is a constant, the predicate is evaluated at the declaration, and a value that fails it is an error at the place it is written, whatever kind of boundary that is:

```js
type Small = uint8 where this < 10;

let x: Small = 20;                  // TypeError: a binding's initializer
function g(x: Small) {} g(20);      // TypeError: an argument
function f(x: Small = 20) {}        // TypeError: a parameter's default
function f(): Small { return 20; }  // TypeError: a return
const h = (): Small => 20;          // TypeError: an arrow's expression body
class K { a: Small = 20; }          // TypeError: a field's initializer, static or not
x = 20;                             // TypeError: an assignment
20 := Small;                        // TypeError: a conversion
function f<N: Small = 20>() {}      // TypeError: a generic parameter's default
function* g(): Generator.<Small> { yield 20; } // TypeError: a yield
const a: [2].<Small> = [1, 20];     // TypeError: a constant of a fresh array, tuple or object literal
const o: { a: Small } = { a: 20 };  // TypeError, likewise
function h(a: [2].<Small>) {} h([1, 20]); // TypeError: the same at an argument, and at a return
function k<N: Small>() {} k.<20>(); // TypeError: a type argument for a refined domain
function f(v: uint8): Small { return v := Small; } // checked when it runs: v is not a constant
function f(): Small { return 1; return 20; }       // not judged: the second return is never reached
```

A compound assignment is not such a boundary: in ```x += 20``` the result depends on ```x```, which is not a constant.

A refinement of a refinement holds both predicates, and a constant enters it at the innermost base:

```js
type Low = uint8 where this < 10;
type Mid = Low where this > 3;
let m: Mid = 5;   // admitted: 5 is built as a uint8, then satisfies both
let n: Mid = 20;  // TypeError: fails Low
let k: Mid = 2;   // TypeError: fails Mid
```

#### A refinement's default

A refinement has its base's default where the predicate holds of it, and none otherwise. Over a primitive base that default is a constant, so the question is answered where the type is used without an initializer:

```js
type Small = uint8 where this < 10;
type Big = uint8 where this > 5;

let x: Small;                    // 0: the base's default satisfies the predicate
class K { a: Small; }            // likewise for a field
let y: Big;                      // TypeError: 0 fails `this > 5`, so Big has no default
const a = new [2].<Small>();     // two elements of 0
const b = new [2].<Big>();       // TypeError: there is no default to fill it with
```

The same reading of "constant" decides a function's own clauses at a call and at a return. ```null``` and ```undefined``` are constants as a number is, a fresh array, tuple or object literal of constants is one, and a function that can end without a ```return``` returns the constant ```undefined```:

```js
function g(x: uint8 | null) where x !== null {}
g(null);                            // TypeError at the call
function g(t: [uint8, uint8]) where t[0] > 10 {}
g([5, 6]);                          // TypeError: a fresh literal of constants
const o = { v: 5 };
function k(x: uint8) where x > 10 {}
k(o.v);                             // checked when it runs: const fixes the binding, not the object
function f(): uint8 | null where return !== null { return null; } // TypeError at the return
function f(): uint8 | undefined where return !== undefined {}     // TypeError: the implicit exit returns undefined
```

An enumerator is a constant too. The enums a source text declares are processed before its clauses are evaluated, as its type aliases and interfaces are, so a clause that names an enumerator, or that an enumerator decides, is judged where the call, the return or the default is written:

```js
enum Level { low, mid, high }
function g(e: Level) where e > Level.low {}
g(Level.low);                                            // TypeError at the call
function d(e: Level = Level.low) where e > Level.low {}  // TypeError: the default can never be taken
function r(): Level where return !== Level.low { return Level.low; } // TypeError at the return
g(Level.mid);                                            // fine
```

An enum that is decorated, or whose initializer reads a binding other than an earlier enumerator, is not processed ahead of the program, and a clause it would decide is evaluated when the call runs.

A valid design refuses such a call where it is written, evaluates the clause with the enum the program has and not a copy, changes no order of evaluation a program can observe, and decides every clause form.

| Direction | Ergonomics | Prior art and limits | Real-engine cost | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Leave an enumerator to run time | ```g(5)``` is refused where it is written and ```g(Level.low)``` is not, though an enum is the commonest closed set a contract names | Rust's enum variants are constants in any const context, and Swift, C# and Java treat theirs the same; TypeScript inlines a ```const enum``` | The check runs at every such call | Consistent, but narrower than the language's own definition of compile-time evaluable | Rejected: the gap this closes |
| Evaluate the clauses after the enums are processed | The examples above | The enums are items known before the program runs, as in Rust; the analogy is close, since an initializer here must be compile-time evaluable | None at run time: the pass already processed enums, only later than the clauses | The same enum object the program uses; no evaluation moves that a program can observe | **Recommended and implemented** |
| Evaluate the enum a second time for the clause | Works for comparisons | None | A second enum per check | Two objects for one enum: an ```is``` test in a clause would see a different type | Rejected |
| Substitute each enumerator's underlying constant | Works for comparisons | TypeScript's ```const enum``` | None | An enumerator is a value of its enum, not its underlying constant | Rejected |
| Judge the clause without evaluating it, reading the parameter as the enumerator | Decides comparisons and equality; an opaque predicate is left out | Rust's exhaustiveness checking over variants reasons this way | None | Sound, and what an argument of a narrower type will need in any case | Rejected as the answer here: it does not decide every clause form |

A valid design reports a wrong constant where it is written, in every position alike, never guesses about a value that is not a constant, and evaluates the predicate exactly as the boundary would.

| Direction | Ergonomics | Prior art and limits | Real-engine cost | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Decide a constant only at a binding's initializer and an argument | The same constant is an error in ```let x: Small = 20``` and silent in ```return 20```; a default that can never be taken ships | None | A run-time check is emitted where its outcome is already known | A refinement admits a value only where every predicate holds, and that sentence names no position | Rejected: the defect this section closes |
| Decide it at every boundary whose source is a constant | A wrong constant is reported where it is written; a right one costs nothing | Rust's ```unconditional_panic``` and ```arithmetic_overflow``` lints are deny by default and fire on a constant in any position, a ```return``` included, and Ada checks a static expression against a static predicate at compile time wherever it appears. Rust's own refinement-like types, such as ```NonZeroU8::new(0).unwrap()```, are decided only in const contexts, so there the analogy is weaker | A decided check is one never emitted | The predicate is evaluated as the boundary would evaluate it; one with an input that cannot be read yet keeps its deferral | **Recommended and implemented** |
| Decide it at declarations only: defaults and field initializers | Catches what running the code may never reach, and leaves ```return 20``` | None | Fewer sites | An arbitrary line between boundaries the rules treat alike | Rejected |
| Count only numbers, strings, Booleans and bigints as constants | ```g(5)``` is refused and ```g(null)``` is not, for the same kind of clause; not-null, the commonest contract, is never decided early | Kotlin reports a ```null``` literal passed to a non-null parameter at compile time and C# warns on one; Rust's const evaluation treats ```None``` as any other constant | The check runs at every such call | Narrower than the language's own definition of compile-time evaluable | Rejected |
| Also count a read through a ```const``` object | ```const o = { v: 5 }; g(o.v)``` would be refused too | TypeScript narrows through ```const``` object members and is unsound once one is written | Needs escape analysis | ```const``` fixes the binding, not the object | Rejected: not a constant |

Dependent fields are checked at boundaries:

```js
const payment: Payment = { name: 'Alice' }; // Checked, construction
payment.creditCard = 4111111111111111; // Not checked
payment.billingAddress = '123 Main St'; // Not checked
validate(payment); // Checked, function call
const payment2: Payment = payment; // Checked, object assignment
```

This allows an object to be put into an invalid state momentarily while the compiler ensures it doesn't stay invalid across a boundary. When validity cannot be determined at compile time, a runtime check of the ```where``` clause is inserted at the boundary and throws on failure. To branch on validity instead of throwing, test explicitly:

```js
if (payment is Payment) { 
	example(payment);
}
```

#### Limitations

Consider the following code that allows a name to be a 'strict' or 'loose' mode.

```js
type User = {
	mode: 'strict' | 'loose',
	name: string
} where if (this.mode == 'strict') {
	this.name is string.<{ pattern: /^[a-z]+$/ }> // lowercase only
} else {
	this.name is string.<{ pattern: /^[a-zA-Z]+$/ }> // mixed case
};

const user: User = { mode: 'strict', name: 'alice' }; // Pretend 'alice' is a dynamic value
// Compiler knows:
// user.mode is 'strict'
// user.name is string.<{ pattern: /^[a-z]+$/ }>

user.mode = 'loose';

example(user);
// The compiler cannot prove the where clause still holds, so a runtime
// check is inserted at the call boundary. It passes here since 'alice'
// matches the loose pattern, but the check cannot be elided.
```

If the compiler were smart it would know that strings matching the 'strict' regex are a subset of those matching the 'loose' regex and elide the check. It's unlikely such a subset operation would be added (or implemented by a user in the metadata subtype operation, at least in general), so the compiler falls back to the inserted runtime check. To branch on validity instead:

```js
if (user is User) {
	example(user);
}
```

Note: There's a comment about 'alice' being a dynamic value because a hardcoded literal would propagate and validate. That is, the compiler would be totally fine with the above example and correctly determine that no runtime check is required.

## Examples

### JSON Serialization

A native, fused version of this boundary (parsing, validation, and typed allocation in one pass) is specified in [Serialization](serialization.md). The reflective pattern below remains the way to express wire-name mappings and custom policies.

```js
// String constraints like pattern/minLength/maxLength come from the
// StringBounds meta type defined in the primitive metadata proposal.

const schemaKey = Symbol('schema');

type SerializeData = {
	name: string,
	wireName: string
};

partial class ClassMetadata {
	[schemaKey]: [].<SerializeData> = [];
}

// @field() registers a field for serialization with an optional wire name
function field<T: type, TClass: type>(
	{ name, metadata }: Reflect.ClassField.<T, TClass>
) where typeof name == 'string' {
	metadata[schemaKey].push({ name, wireName: name });
}
function field<T: type, TClass: type>(
	wireName: string,
	{ name, metadata }: Reflect.ClassField.<T, TClass>
) where typeof name == 'string' {
	metadata[schemaKey].push({ name, wireName });
}

function serialize<T: type>(instance: T): { [key: string]: any } {
	const result: { [key: string]: any } = {};
	for (const { name, wireName } of Reflect.getMetadata.<Reflect.Class, T>()[schemaKey]) {
		result[wireName] = instance[name];
	}
	return result;
}

function deserialize<T: type>(cls: { new(): T }, data: { [key: string]: any }): T {
	const instance = new cls();
	
	// 1. Mutation phase: Populate the fields.
	// Scalar bounds (like string or union types) are checked immediately on assignment.
	// However, cross-field 'where' clauses are not evaluated yet, allowing us to assign fields in any order without triggering false validation errors.
	for (const { name, wireName } of Reflect.getMetadata.<Reflect.Class, T>()[schemaKey]) {
		instance[name] = data[wireName];
	}

	// 2. Boundary phase: Now that the object is fully populated, we assert the cross-field invariants. The `is` operator evaluates the `where` clause.
	if (!(instance is T)) {
		throw new TypeError(`Cross-field validation failed for ${cls.name}`);
	}

	return instance;
}

class AddressResponse {
	@field('street_address')
	streetAddress: string;

	@field
	country: 'US' | 'CA';

	// The base type is just string
	@field('postal_code')
	postalCode: string;
} where if (this.country == 'US') {
	this.postalCode is USPostalCode
} else {
	this.postalCode is CAPostalCode
};
```

This does raise a question though about decorators usage in dependent record types. In the above usage all decorator usage with ```postalCode``` are identical. I mention this to say this isn't allowed:

```js
class AddressResponse {
	@field('street_address')
	streetAddress: string;

	@field
	country: 'US' | 'CA';
} where if (this.country == 'US') {
	this is { @field('zip_code') postalCode: USPostalCode };
} else {
	this is { @field('postal_code') postalCode: CAPostalCode }
};
```

Instead this form would be used, as decorators are only allowed in the class body:

```js
class AddressResponse {
	@field('street_address') streetAddress: string;
	@field country: 'US' | 'CA';

	@field('zip_code') usZip?: USPostalCode;
	@field('postal_code') caZip?: CAPostalCode;
} where if (this.country == 'US') {
	this.usZip !== undefined && this.caZip === undefined;
} else {
	this.caZip !== undefined && this.usZip === undefined;
};
```

### Network Messages

Showing ```where match``` syntax. A ```match``` is an expression, so it is a predicate like any other; in this position it carries an implicit ```default: false;```, since a predicate that matches no clause has answered that the value is not of the type. Its semantics are otherwise [pattern matching](patternmatching.md)'s.

"This position" is the whole predicate, or the whole predicate of an ```if``` branch. A ```match``` anywhere else in a predicate is an ordinary expression: it has no implicit default, so it must be exhaustive as any ```match``` is, and the predicate around it is judged as any test is. A test of one pattern that composes with other operands is written with ```is```:

```js
type A = { k: 'a' | 'b', n: uint8 } where this.n > 3 && match (this.k) { when 'a': true };                 // TypeError: missing 'b'
type B = { k: 'a' | 'b', n: uint8 } where this.n > 3 && match (this.k) { when 'a': true; default: false }; // exhaustive
type C = { k: 'a' | 'b', n: uint8 } where this.n > 3 && this.k is 'a';                                     // the composable test
type D = { k: 'a' | 'b', n: uint8 } where this.n > 3 where match (this.k) { when 'a': true };              // a second clause keeps the default
```

A ```match``` clause passes on to the clauses after it the join of its arms, each under its pattern: an arm that answers ```false```, like the implicit default, contributes nothing, and a range arm bounds its subject as the same range in an ```is``` test does. So ```where match (k) { when 'a': true; when 'b': false } where k === 'b'``` and ```where match (x) { when 0..=10: true } where x > 50``` are errors where they are written.

#### Why the implicit default stays with the whole predicate

A valid design never lets a boundary throw where it is specified to answer, gives a missing case a message at the declaration, and adds no second Boolean-test form beside ```is```.

| Direction | Ergonomics | Prior art and limits | Real-engine cost | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Whole predicate or whole branch only; a nested ```match``` is ordinary | A missing case is an error naming the case; the fix is ```default: false```, a second ```where```, or ```is``` | Rust requires ```match``` to be exhaustive everywhere and gives the Boolean test its own form, ```matches!```, which expands to a ```match``` ending ```_ => false```. The analogy is close: the whole-predicate position is this proposal's ```matches!``` and ```is``` its composable test. C# splits the same way between ```switch``` expressions and ```is``` | One rule and no context flag on a ```match``` | A nested ```match``` narrows and is checked as in a statement | **Recommended and implemented** |
| Through ```&&```, ```\|\|```, ```?:``` arms and parentheses, not ```!``` | Reads naturally under ```&&```; a user must learn where the default stops | No surveyed language gives a ```match``` a default by Boolean context | A context flag on each ```match``` | Adds ```false``` to a nested match's type and needs more rules for ```??```, the comma and arguments | Rejected: a hidden rule where an explicit form exists |
| As above, and through ```!``` | Under ```!```, a value that matches no clause is admitted | None | The same flag | Reverses the reason for the default, that a value no clause matches is not of the type | Rejected |
| Every ```match``` inside a predicate | ```g(match ...)``` and ```match (...) === 3``` receive a Boolean nobody wrote | None | The same flag | An arm type that is not Boolean gains ```false``` | Rejected |
| Forbid a nested ```match``` in a predicate | Blocks a legitimate helper argument, and a disjunction with a ```match``` cannot be split into two clauses | None; Rust allows ```match``` in any expression position | Simplest | Consistent but arbitrary: no other expression is barred from a predicate | Rejected: over-restricts ordinary expressions |
| Leave a nested ```match``` unjudged and without a default | A missing case throws from inside an ```is``` test or a boundary check | C# reaches this for a non-exhaustive ```switch``` expression, with a compile-time warning | A throwing path in every evaluation | A check specified to answer true or false throws instead | Rejected: the defect this section closes |

```js
type NetworkState = {
	status: 'idle' | 'loading' | 'success' | 'error',
	data?: any,
	errorMessage?: string
} where match (this.status) {
	when 'idle' | 'loading': 
		this.data === undefined && this.errorMessage === undefined;
	when 'success': 
		this.data !== undefined && this.errorMessage === undefined;
	when 'error': 
		this.errorMessage !== undefined && this.data === undefined;
};

function renderUI(state: NetworkState) {
	match (state) {
		when { status: 'success' }: {
			const data = state.data;
			if (data !== undefined) renderData(data);
		}
		when { status: 'error' }: {
			const message = state.errorMessage;
			if (message !== undefined) renderError(message);
		}
		when { status: 'idle' | 'loading' }:
			renderSpinner();
	}
}
```

A ```where match``` on one member against constants is a discriminating chain in the sense above, so ```NetworkState``` denotes the union of its four branches and the ```match``` in ```renderUI``` is checked exhaustive with no ```default```. A branch assertion such as ```this.data !== undefined``` contributes a consumer fact only through stable observations. For a mutable ```NetworkState```, capture and test ```state.data``` explicitly before using it; the tag alone does not prove that its payload remains present.

### Business Logic

```js
type DatabaseCommand = {
	userRole: 'admin' | 'editor' | 'viewer',
	action: 'insert' | 'update' | 'delete' | 'read',
	targetTable: string
} where match (this.userRole) {
	when 'viewer': 
		this.action === 'read';
	when 'editor': 
		this.action !== 'delete';
	when 'admin': 
		true; // Admins can do anything
};
```

On construction, this would throw if userRole doesn't match the allowed action.

### Misc Example

```js
type Email = {
	to: string.<{ pattern: /@/ }>,
	subject?: string,
	body?: string
} where (this.subject?.length ?? 0) > 0 || (this.body?.length ?? 0) > 0;

// The optional-field draft shape, written out since utility types like
// TypeScript's Partial aren't part of the proposal:
type EmailDraft = {
	to?: string.<{ pattern: /@/ }>,
	subject?: string,
	body?: string
};

function draftEmail(): EmailDraft {
	return {};
}

function sendEmail(ref draft: EmailDraft) {
	if (!(draft is Email)) {
		throw new Error("Invalid email draft");
	}
	// Sending...
}

const msg = draftEmail();
msg.to = "alice@example.com";
// No subject or body

sendEmail(ref msg); // throws
```


## Constructing refinements with Reflect.makeType

A builder uses the ordinary parameterized node and a unary callback:

```js
function refine(base, predicate) {
    return Reflect.makeType({ kind: 'parameterized', base, metadata: { where: predicate } });
}
const positive = value => value > 0;
type Positive = refine(number, positive);
```

The callback receives the candidate as its only argument. It is called with an `undefined` receiver using its ordinary JavaScript receiver rules, and its result is converted with `ToBoolean`. A supplied arrow must use its argument; its lexical `this` is not rebound. Source `where this > 0` creates a declaration-bound unary callback which supplies the source clause's `this`, retaining lexical scope, private scope, and converted generic arguments. Both forms use the same metadata validation and identity rules. Repeating a source specialization within one declaration activation reuses its predicate; another activation or argument tuple has a distinct predicate.

Admission inspects the callback without running it. It requires one non-rest, non-reference formal parameter on an ordinary synchronous function. Default and destructuring parameters retain their ordinary behavior. Class constructors, async functions, generators, bound functions, proxies and opaque host callbacks are rejected. The body and dependencies must satisfy the compile-time-evaluable fragment: local computation is allowed, while mutable captures, writes through the candidate, unknown callbacks, ambient receivers and nondeterministic calls are refused. Deterministic library calls are admitted by actual intrinsic identity. Capture and call-target guards remain in force across getter and argument effects; re-admitting a function cannot reset the evidence protecting an existing type.

The original function is retained without cloning, freezing or altering it. Two constructions with the same base and callback are one type. Identical source text does not make two closures equal. Reflection returns the same callback, so `Reflect.makeType(Reflect.getReflection(Positive)) === Positive`.

Construction does not assert that a value inhabits the type:

```js
type Triple = [uint8, uint8, uint8];
const twoElements = value => value.length === 2;
type EmptyTriple = refine(Triple, twoElements); // valid construction; no matching value
```

This computed refinement retains its callback identity and is not rewritten to `never`. Existing source impossible-test diagnostics still apply when a contradiction is written directly. Callback admission is required even for a base of `void` or `never`; an empty base cannot conceal an invalid callback.

At a boundary, the base is admitted or converted first. Nested predicates run from the innermost refinement outward and stop at the first false result. Ordinary candidate getters, conversions, exceptions and reference-liveness checks remain observable. A false result means rejection; a predicate exception propagates. For a default, false means no default and an exception remains an exception. No extra predicate invocation probes default availability. A mutable object's earlier membership result is not a permanent fact about that object.

A refinement changes no representation or object shape. Engines can share base layouts and specialize guards around a retained callback without attaching per-type properties to candidates. `Where` has an empty default and no arithmetic operator block, so fallback arithmetic sheds the predicate instead of promising that its result still satisfies it.

A recursive alias retains the predicate at every recursive occurrence. Refining an alias does not supply a representation for a cycle made only of alias forwarding and refinement bases; such a cycle is still rejected. For generic aliases with multiple clauses, a clause that does not read `this` remains a specialization condition, while candidate-dependent clauses refine the resulting value type.
