# Pattern Matching

A pattern is three things at once: a test, a destructuring, and a proof. ```match``` runs the test, performs the destructuring, and hands the proof to the type system, so the arm that runs knows statically what the subject is, the bindings it introduced carry exact types, and a ```match``` over a closed type is checked complete the way a ```switch``` over an enum already is. This document defines the ```match``` expression and its pattern grammar for this proposal, extends the ```is``` operator so a pattern is usable as an ordinary boolean test, and then does what the [ranges](ranges.md) document did: checks the feature against everything else here - literal types, narrowing, enums and sealed classes, [composites](composites.md), [ranges](ranges.md), [typed regular expressions](regexp.md), [error handling](errorhandling.md), generics - because a pattern language is only worth its grammar if the type system makes every pattern checkable and most of them free.

```js
type Response =
  | { status: 200, body: string }
  | { status: 404 }
  | { status: uint16, error: string };

const text = match (response) {
  when { status: 200, let body }: body;
  when { status: 404 }: 'not found';
  when { status: 500..<600, let error }: throw new ServerError(error);
  when { let status, let error }: `${status}: ${error}`;
};
```

Every piece of that is typed. The literals ```200``` and ```404``` take ```uint16``` from the subject's field, so they compare against what is actually stored; ```body``` binds as ```string``` because the arm has narrowed the union to its first member; the range arm tests containment the way a range ```case``` label does; the last arm is reachable only for the third member, so ```error``` binds ```string``` with no test the earlier arms haven't already paid for; and removing the last arm is a compile-time TypeError, because the union is not covered.

## The match Expression

```match (subject) { ... }``` is an expression. Its body is one or more clauses, each ```when``` a pattern, or ```default```, followed by ```:``` and an arm body. An expression arm is terminated by a ```;```, with automatic semicolon insertion applying as it does to an expression statement; a block arm is terminated by its closing brace and takes no ```;```. Clauses are tried in source order; the first whose pattern matches evaluates its body, and that value is the value of the expression. ```default``` matches anything and must be last. If no clause matches, a ```TypeError``` is thrown - and the exhaustiveness rules below make that throw statically impossible exactly where the types can prove it.

An arm body is an expression, a ```throw``` expression, or a block. ```throw``` is admitted as an arm rather than left to a library's always-throwing helper, because an arm that reports an impossible case is the commonest arm a total ```match``` has. A block arm is a [```do``` expression's block](doexpressions.md): its value is its completion value, and it carries that form's rules, so an arm may end in an ```if```/```else```, a ```try```/```catch```, or a ```switch```, and an arm ending in a declaration, an ```if``` without an ```else```, or a loop is an error naming what it ended with. The subject is evaluated once, before any pattern.

```js
match (command) {
  when 'start': engine.start();
  when 'stop': { engine.stop(); log('stopped'); }   // void: the effects form
  default: throw new RangeError(command);
};

const size = match (request) {
  when { let width, let height }: {
    const scale = devicePixelRatio;                 // statements, then a value
    clamp(width * scale, height * scale);
  }
  default: DEFAULT_SIZE;
};
```

A ```match``` initializing several bindings at once returns a tuple or an object and is destructured, which is typed positionally or by member rather than inferred from a literal:

```js
// status: Status, an enum of Active, Paused, and Off
const [color, size] = match (status) {
  when Status.Active: ['green', 2];
  when Status.Paused: ['yellow', 1];
  when Status.Off: ['red', 0];
};                                       // color: string, size: uint8
```

A block arm is a block, not a function body, which is the whole reason to have one: ```return``` returns from the enclosing function, ```break``` and ```continue``` target an enclosing loop, ```await``` works where the enclosing function is async, and ```yield``` works inside a generator. An immediately-invoked arrow can reproduce none of those - its ```return``` lands in the arrow, its ```await``` demands an async arrow whose value is a promise - which is why a block arm carries a value rather than sending the author to a closure for one. The block is the plain ```do``` form and never ```do *```: an arm produces the arm's value, and a generator there would produce a generator.

The static type of a ```match``` expression is the union of its arm types, canonicalized, with ```throw``` arms and diverging blocks contributing nothing - the same divergence analysis the README defines for ```switch``` and that a [```do``` expression](doexpressions.md) reads its own type by. A contextual type on the ```match``` flows into each arm body, so a ```match``` in a ```uint8``` position types its arm literals as ```uint8``` by ordinary [literal propagation](README.md).

### Every matching arm

```match all (subject) { ... }``` answers with **every** arm that matched rather
than the first, as a ```[].<T>``` in arm order.

```js
const warnings = match all (character) {
  when { hp: ..<30 }: 'LOW HEALTH';
  when { mp: ..<10 }: 'LOW MANA';
  when { inCombat: true }: 'IN COMBAT';
};
```

Three things differ from ```match```, and nothing else does - patterns, guards,
bindings, per-arm narrowing, arm bodies and block decorators are all the same:

- **The value is a list.** Every matching arm's body is evaluated, in source
  order, and the results collected. The subject is evaluated once.
- **Exhaustiveness does not apply.** No arm need match, and the value is then an
  empty list. That is an answer rather than a missing case, which is why the rule
  below is not inherited here.
- **Arm-failure subtraction does not apply.** A later clause is reached whether
  or not an earlier one matched, so it learns nothing from the earlier failure -
  where a ```match```'s ```default``` after ```when null:``` sees a non-null
  subject.

**```default``` is a Syntax Error in a ```match all```.** It and ```_``` are
synonyms in a ```match```, and a word meaning "always" in one form and "only if
nothing else did" one form over is a trap worth refusing outright. A clause that
always contributes is spelled ```when _:```, which is what ```_``` means
everywhere. What to do when nothing matched is an ordinary question about an
empty list, and is better asked after the expression than inside it.

An arm that ```throw```s aborts the collection: the arms before it have already
run, and their values are lost with the expression. Collecting failures rather
than throwing them is what the form is already for -

```js
const failures = match all (form) {
  when { email: '' }: new ValidationError('email required');
  when { age: ..<18 }: new ValidationError('must be 18');
};
```

- where the value *is* the failure and the result is already the collection.

No general-purpose language has this. Every pattern-matching construct surveyed -
Rust, OCaml, Haskell, Scala, F#, Elixir, Swift, Kotlin, Python, C#, Java, Ruby,
Elm, Racket, Clojure - is first-match. The semantics lives in rule engines, whose
agenda fires every activated rule; in the CSS cascade, where every matching
selector applies; in Prolog, where matching is non-deterministic and first-match
is the special case spelled with a cut; and in ```bash```'s ```case```, whose
```;;&``` terminator continues testing the remaining patterns. The case for the
form is the use rather than the tradition: validation collecting every failing
rule, classification where an item carries several tags, diagnostics gathering
every applicable warning, permissions accumulating grants. Each is written today
as a filter over an array of predicate-and-value pairs, which is this construct
with the patterns spelled as lambdas.

### Why not switch

```switch``` stays what it is: a statement dispatching on ```===```, on range containment, or on the type objects of a sealed hierarchy, with exhaustiveness over enums and sealed classes. ```match``` is the expression form with structure: it destructures, it binds, it composes tests, and it answers with a value. The two share their narrowing and exhaustiveness machinery, and a ```switch``` that has outgrown its labels - a guard here, a destructure there - is a ```match``` waiting to be written. Nothing is removed from ```switch``` to make room; the division is statement-and-equality against expression-and-structure.

### The grammar is an error today

This proposal only adds syntax that is currently invalid, and ```match``` needs the argument made, because ```match(x) { }``` is a valid program today: a call followed by an empty block. The construct stays valid and keeps its meaning, because a ```match``` expression requires at least one clause, and every clause begins ```when <pattern>:``` or ```default:```. ```default``` is a reserved word, so ```default:``` cannot be a label; ```when``` is an ordinary identifier, but ```when <token>:``` parses today only as the label ```when:``` with nothing between - and a clause's pattern is never empty - or as the expression ```when``` followed immediately by another expression, which is a SyntaxError. So every program the new grammar accepts is an error in the current language, and every program the current language accepts keeps its parse: ```match(x) {}```, ```match(x) { foo: bar(); }```, and bare ```match(x)``` all still mean the call. In expression position there is no overlap at all, since a call followed by ```{``` is already a SyntaxError there.

The same argument covers ```all```. ```match all (x)``` is two adjacent expressions today and so a SyntaxError, and the modifier carries a **[no LineTerminator here]** restriction so that

```js
match
all(x) { }
```

keeps the parse it has now - a statement, a call, and a block - rather than becoming a ```match all``` whose subject is on the next line. ```all``` is an ordinary identifier everywhere else, including as a binding, a property, and a function name.

## Patterns

The pattern grammar, one form at a time, each with its runtime meaning and its typing rule. Throughout, the *position type* is the static type a pattern is matched against: the subject's type at the top, a field's type inside an object pattern, an element's type inside an array pattern.

### Literal patterns

A literal - number, string, boolean, ```null```, ```undefined```, a template with no substitutions - matches by SameValue, except that a bare ```0``` matches both zeros by SameValueZero while an explicit ```+0``` or ```-0``` distinguishes them. The literal takes the position type, by the same propagation that types a literal anywhere else: ```when 27:``` against a ```uint8``` field is a ```uint8``` 27, and against a ```decimal128``` field a decimal, so the comparison is within one type and the cross-type miss that propagation exists to prevent cannot happen here either. A literal that cannot take the position type - ```when 'baz':``` against a ```uint32``` - is a compile-time TypeError, the impossible-test rule narrowing already enforces.

```NaN``` needs no special case here, which is worth saying because in an untyped matcher it needs one. It is a name, so it matches as a constant, and constants compare within a type by that type's SameValue, under which a NaN equals a NaN. What follows from the same rule is that a ```float32``` NaN does not match a ```float64``` one: they are values of different types, and no comparison in this proposal crosses types.

One case needs a rule rather than inference: a numeric literal against a union of numeric types, ```when 5:``` against ```uint8 | float32```, has two types it could take and matching only one would be a silent half-answer. It is a type error, and the fix is to say which: ```when uint8 and 5:``` narrows first and lets the literal take the narrowed type, or ```${uint8(5)}``` states the value outright.

### Interpolation patterns

```${expression}``` evaluates the expression and matches by SameValue against the result, whatever the result is - a constant, a composite, a Type Object matched for identity rather than membership. It is the escape hatch from every cleverer rule below: where an expression pattern would consult a matcher or test a type, ```${...}``` compares. It is a type error if the expression's type and the position type share no values, since the test could never succeed.

### Wildcard patterns

```_``` matches anything and binds nothing. It is what a position the arm doesn't care about is spelled: ```[_, let y]``` takes the second element, ```{ id: _, let name }``` requires ```id``` to be present without reading it into a binding, and ```when _:``` is a catch-all in a nested or a top-level position alike. The identifier ```_``` is the wildcard wherever a pattern is expected, so a program that binds the name ```_``` and wants to compare against its value writes ```${_}```.

### Binding patterns

```let name``` and ```const name``` match anything and bind the matched value at the position type, narrowed by everything the enclosing pattern has established. The binding is scoped to the arm - its guard and its body. An annotated binding is a test and a bind in one: ```let e: TypeError``` matches when the value is of the type and binds it narrowed, which is the ```catch (e: TypeError)``` form of [error handling](errorhandling.md) appearing where it was always going to reappear. ```let``` binds mutably and ```const``` immutably, as they do in any other declaration; a pattern is a binding form and the keywords keep their meaning. There are no bare-identifier bindings; an identifier without ```let``` or ```const``` is an expression pattern, below, which is what lets a constant be a pattern without a sigil.

A binding on the right of ```and``` names the whole value a pattern just matched, which is what other languages spell as an *as-pattern* or an ```@``` binding:

```js
// result: ['ok', Payload] | ['error', string]
match (result) {
  when ['ok', _] and let success: forward(success);   // success: ['ok', Payload]
  when ['error', let message]: report(message);
}
```

### Type patterns

A pattern that is a type - a predefined name, a class, an interface, a parameterized type, ```Composite.<T>```, a union written inline - matches when the subject is of the type, by the same ```IsOfType``` test the ```is``` operator and every boundary perform, and narrows the position to it. A dependent record type runs its ```where``` clauses here, because membership in such a type is defined by them. A union of literal types is the or-pattern for constants with no combinator spelled: ```when 'GET' | 'HEAD':``` is one type pattern whose test is two SameValue comparisons and whose narrowing is the union of the two literal types.

```js
match (value) {
  when uint8: value + 1;                  // value: uint8 here
  when 'GET' | 'HEAD': readOnly(value);   // value: 'GET' | 'HEAD'
  when Circle: value.radius;              // instanceof, narrowed
  when Payment: charge(value);            // where clauses evaluated
  default: reject(value);
}
```

### Type-subject patterns

```when extends T:``` matches when the subject is a *type* - a type object - that is within ```T```. Every other pattern here matches a value against a shape; this one matches a type against a pattern of types, which is what a program dispatching on reflection needs, since a decorator, a schema emitter, and a validator generator are all handed a type and must ask what kind it is.

```js
function constraintsFor(type: type): ConstraintDoc | undefined {
  return match (type) {
    when extends float32.<B: NumberBounds.<float32>>: ({ ...B });
    when extends string.<S: StringBounds>: ({ ...S, pattern: S.pattern?.toString() });
    default: undefined;
  };
}
```

```B``` and ```S``` are **slots**: written ```name: Constraint``` in a type-argument position, each binds the component of the subject standing where it does and is checked against its constraint. A slot binds a type object where that position holds a type (```Map.<K: type, V: type>```) and the value where it holds metadata (```float32.<B: NumberBounds>``` binds the metadata object, which is why ```{ ...B }``` is an ordinary spread). Its type in the arm is the constraint it was checked against, so ```S.pattern``` is a member access the checker verifies. Slots are bindings like any other - immutable, scoped to the guard and arm, and subject to the same duplicate-name and ```or```-consistency rules.

The keyword is doing real work. Without slots the test is assignability, so ```when extends string:``` matches ```string``` and every refinement of one; with slots the test is unification, because a slot must bind the component that stands where it is written. The consequence: a slotted pattern matches structurally where an unslotted one matches up to subtyping, so ```extends [].<E: type>``` does not match a ```[5].<uint8>``` although ```extends [].<uint8>``` does. Unification up to subtyping is a much larger operation that this proposal performs nowhere; a pattern is written against the shape it expects.

```extends``` is a reserved word, so this form costs the grammar nothing - unlike ```match``` itself, which needed care to stay compatible. And it removes an ambiguity the value forms would otherwise leave: a bare name denoting a type tests *membership*, so ```when float32:``` asks whether the subject is a float, and against a type object that is always false. ```extends``` is how a program says it means the type.

A type subject has no closed set of cases - the types are an open universe - so a ```match``` over one needs a catch-all, which is what the ```default``` above is.

### Expression patterns

A pattern that is an identifier or member expression - not a literal, not a binding, not a call - evaluates it and matches by what the result is. A Type Object is a type pattern, which is what makes ```when Circle:``` and ```when Count.Zero:``` read as the tests they are - a class is its type object, an enumerator is a constant, and both spellings do the expected thing, the enumerator comparing by SameValue as the constant it is. A value with a ```[Symbol.customMatcher]``` method is matched through it, below. Anything else is a constant compared by SameValue, which for an interned [composite](composites.md) is one pointer comparison. A call expression is never a constant pattern - a call in pattern position is the extractor form below, and a computed constant is spelled ```${f(x)}```. Because the dispatch is on the value, a *named* range or regular expression matches as its literal spelling would, by containment and by whole-subject match: ```when validRange:``` and ```when datePattern:``` test rather than compare, and identity, where meant, is ```${validRange}```.

### Object patterns

```{ key: pattern, ..., ...let rest }``` matches a subject that has each named member, each member's value matching its sub-pattern; ```{ let x }``` is shorthand for ```{ x: let x }```. Presence is the ```in``` test, so an optional member that is absent fails the pattern rather than matching ```undefined```, and matching the *absence* is written with a guard or by matching the member against a union with ```undefined``` where the type says that is possible. A pattern names the member the *declaration* names. Where a [decorator](decorators.md) has given a member a different name on the wire - ```@field('street_address') streetAddress: string``` - it is ```streetAddress``` a pattern matches, never ```street_address```, because the typed parse ([serialization](serialization.md)) produced the object the program declared rather than the document it received. That is the whole of what the boundary buys, seen from the inside. A member the pattern does not name is ignored - a pattern is a subset test, as an interface check is, and for the same reason: this type system has width subtyping and no notion of an exact object type, so there is nothing for a pattern to say about the members it did not name and no syntax it must spell to stay silent about them. A member whose value doesn't matter is named against ```_```, which tests presence and reads nothing. Unless ```...let rest``` appears, which binds an object of the remaining named members at the type the position type gives them; the rest is a fresh ordinary object, built like an object spread.

The typing rule is where a union subject earns its keep: an object pattern is matched against each member of a union position type, the members it cannot match are discarded, and the sub-patterns and bindings type against what remains. That is how the opening example's ```let body``` binds ```string``` - the ```status: 200``` field pattern eliminated the members without it - and it is the [dependent record types](dependentrecordtypes.md) narrowing arriving through a pattern instead of an ```if```: ```when { country: 'US', let postalCode }:``` types ```postalCode``` as ```USPostalCode``` because the discriminant chose the member. A pattern no union member can match is the impossible test again, a compile-time TypeError.

Reads are cached per match evaluation: the presence test and the read are one cached touch per key - one ```in``` and at most one get - however many patterns name it, so a getter runs once and arms agree about what they saw. This is what makes an accessor-backed subject matchable at all rather than a documented hazard: a pattern language that re-reads a property per test turns a lazily computed member into a different value in each clause, and one that forbids getters cannot match the objects a program actually has. A composite subject makes the cache unnecessary rather than the guarantee weaker, since its every read is idempotent - the [composites](composites.md) document owns that note.

The guarantee covers a member named by a ```{ ... }``` TYPE as much as one named by an object pattern, because the two spellings are meant to be interchangeable and a reader cannot see which path a member takes. A structural object type is therefore matched member by member through the same cache rather than by a whole-value membership test, and the difference is not merely how often a getter runs: given a getter whose value changes, ```when { g: 2 }``` followed by ```when { g: 1 }``` reads twice and matches NEITHER, where a single read matches the second - two spellings of one test choosing different arms. Three kinds of object type keep the whole-value test because a member-by-member one cannot express them: an OPTIONAL member is satisfied by its absence where this requires presence, an INDEX SIGNATURE names no members to walk, and a NOMINAL type rejects a value that merely has the right members. An interface is structural and is not among them.

### Array patterns

```[p1, p2, ..., ...let rest]``` obtains an iterator from the subject and matches elements positionally. Iteration rather than an array test is what lets the form reach everything array-shaped in this proposal: a ```[N].<T>``` need not be an Array exotic object, a tuple [composite](composites.md) is iterable by kind rather than by prototype, and a typed view answers no array predicate at all. A pattern that meant `Array.isArray` would match the one shape that needed it least. Without a rest element the pattern requires the iterator to be exhausted at the pattern's length: ```[let a, let b]``` matches exactly two. With ```...let rest``` the remainder is collected into an array. Iteration results are memoized per match evaluation, so alternatives over array patterns of different lengths pull each element once.

Element positions type from the position type: a tuple type gives each position its own type, an array type gives every position the element type, and ```rest``` binds ```[].<T>``` over the element type or the tuple's remainder. A tuple's defaults are trailing - an element with a default may not precede one without, and none may follow a rest - so a length the type admits leaves unsupplied only positions that carry one, and a pattern shorter than the tuple is testing a length rather than discovering a hole. What an array pattern does *not* do is narrow an array's length into its type: a ```[].<uint8>``` that matched ```[let a, let b]``` is still a ```[].<uint8>```, because a dynamic-length array and a ```[2].<uint8>``` are distinct types with distinct values and no test converts one into the other. The length was tested; the elements were typed; the subject's type is unchanged. This is stated under limitations because it is one.

### Range patterns

A range literal is a pattern matching by containment, exactly as a range ```case``` label does in the [ranges](ranges.md) extension, and with the same requirement: the position type must be ordered. ```when 500..<600:``` against a ```uint16``` is two comparisons; against a float it is the form that makes a float subject matchable at all, since a float has no cases to enumerate. A range pattern narrows through metadata where the interval is constant, the refinement the ranges document already gives a ```for``` counter.

### Regular expression patterns

A regular expression literal is a pattern over a ```string``` position. It matches when the pattern matches the *entire* subject - the whole-string discipline this proposal uses everywhere a pattern constrains a string, from ```StringPattern``` metadata to parsing - and a search is spelled by writing the pattern as a search, ```/.*error.*/```. Its typed match result is available to a juxtaposed object pattern, so the capture types of [typed regular expressions](regexp.md) flow into bindings:

```js
match (line) {
  when /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/ { groups: { let year, let month } }:
    Date.parse(year, month);       // year, month: string, and the group names are checked
  when /\s*/: skip();
  default: throw new SyntaxError(line);
}
```

A misspelled group name in that pattern is a compile-time TypeError, because the match result's ```groups``` has an exact object type; this is the difference between a pattern language with types and one with conventions.

### Extractor patterns

```Expr(p1, p2, ...)``` evaluates ```Expr``` and matches through its ```[Symbol.customMatcher]```. The typed protocol is a method, usually static, from the subject to a tuple or ```null```:

```js
class Some<T> {
  value: T;
  static [Symbol.customMatcher]<T>(subject: Option.<T>): [T] | null {
    return subject instanceof Some.<T> ? [subject.value] : null;
  }
}

class None<T> extends Option.<T> {}

match (find(id)) {                    // find returns Option.<User>
  when Some(let user): greet(user);   // user: User, and the arm narrows to Some.<User>
  when None: signIn();                // a class is a type pattern; no matcher needed
}
```

```null``` is no match; a tuple is a match whose elements the sub-patterns match positionally, each typing from the tuple's element types, and a runtime TypeError where the counts disagree, so an extractor reached through ```any``` fails loudly rather than part-matching. An extractor whose head names a type narrows the arm to that type, so ```when Some(let user):``` narrows exactly as ```when Some:``` would and covers the ```Some``` case of a sealed ```Option```. That narrowing is a claim the matcher's author makes and the checker takes, the same trust a declared narrowing predicate receives; a matcher on a head that is not a type extracts without narrowing. The protocol is an ordinary method, so everything ordinary applies: overloads select on the subject's type, a generic matcher infers its parameters from the subject as any generic call does - which is how ```let user``` above binds ```User``` with nothing annotated - and the declared return type is the whole contract, checked where the matcher is written rather than trusted where it is used. A matcher may instead return ```boolean```, in which case the pattern takes no parentheses and is a plain membership test - the form ```Composite``` uses, and the form a head that is *not* a type needs, since a class or interface name already tests membership as a type pattern without consulting any matcher; a boolean matcher with parentheses, or a tuple matcher without them, is a type error. Matchers run user code, so unlike every other pattern they can observe order and throw; a throw propagates out of the ```match```.

### Juxtaposition

A type pattern followed directly by an object or array pattern matches both against the same value: the type first, then the structure, with the structure typing against the *narrowed* subject. ```Circle { let radius }``` is the variant form every language with sealed hierarchies converges on, and here it is not a special form but the composition it looks like - sugar for ```Circle and { let radius }```:

```js
// shape: a sealed hierarchy of Circle and Rect
match (shape) {
  when Circle { let radius }: PI * radius ** 2;
  when Rect { let w, let h }: w * h;
}
```

### Combinators

```and``` matches when both sides match, evaluated left to right, with the right side typed against the position as narrowed by the left - which is the rule that made ```uint8 and 5``` work above, and makes ```Payment and { let creditCard }``` destructure a validated value. ```or``` matches when either side does, tried in order; a binding that appears in one alternative must appear in all, and it binds at the union of its types across them, so ```{ ok: let v: uint8 } or { fallback: let v: float32 }``` binds ```v: uint8 | float32```. ```not``` binds tightest, then ```and```, then ```or```, and parentheses group. ```not``` matches when its operand does not, binds nothing - a binding inside a failed pattern has no value to carry - and narrows by subtraction where the operand's type has a subtractable representation: ```not null``` removes ```null``` from a union, ```not Circle``` removes a sealed member. Where it does not - ```not { let x }``` against an open object type - the test runs and the type is unchanged, because this proposal has no negation types to record "lacks ```x```" in; that too is a limitation stated below rather than a rule pretended.

### Guards

```when Pattern if (expr):``` runs the guard after the pattern matches, with the pattern's bindings in scope and the subject narrowed; a falsy guard fails the arm and matching continues. A guard is an ordinary expression - it may not ```await```, since a ```match``` evaluates synchronously - and its own narrowing tests compose: ```when { let user } if (user is Admin):``` types ```user``` as ```Admin``` in the body. A guarded arm proves nothing to exhaustiveness, since the checker does not evaluate guards.

## Narrowing and Exhaustiveness

Each arm narrows the subject to what its pattern established, through the same machinery that serves ```instanceof```, ```is```, the brand check, and the sealed ```switch```; failing an arm narrows the *next* arm's view of the subject by subtraction where the failed pattern is subtractable, so a ```default``` after ```when null:``` sees the subject non-null. Nothing outlives the expression: the subject's type after the ```match``` is what it was before, since which arm ran is not visible outside.

**Every ```match``` must be exhaustive**, and it is a compile-time TypeError when it is not. A ```match all``` is the exception: no arm need match there, since its value is the list of those that did and an empty list is an answer. A ```match``` is exhaustive when it has an unguarded catch-all clause - ```default```, ```_```, or an unannotated binding - or when its subject's type is *closed* and every case of it is covered. The consequence is the one worth stating plainly: a ```match``` whose subject holds a value of its static type never throws for want of a clause, so the runtime TypeError the previous section describes is unreachable from typed code, and the analysis is what makes it so. The closed-subject half is the ```switch``` rule extended to what patterns can prove. The closed subjects are: an enum; a sealed class, covered by covering its direct subclasses and itself where instantiable; ```boolean```, covered by ```true``` and ```false```; ```null``` and ```undefined```; a union of two or more members each of which is one of these, an object type, a tuple type, or a [composite](composites.md) type, covered by covering every member; a [dependent record type](dependentrecordtypes.md) whose ```where``` chain discriminates on one member against a closed set of constants, which denotes such a union and is covered by covering it; and an intersection one of whose members is a union, which distributes into one - ```(FloatType | IntType) & Shared``` has the two cases a reader sees in it, and an arm naming ```FloatType``` covers ```FloatType & Shared```.

The last two are *derived* forms, and derived for this check alone: neither changes what the type is. ```Reflect.typeOf``` reports the dependent record type and the intersection, identity and interning are untouched, and assignability compares against the type as written. All they change is that the checker sees the cases the reader sees, which is the whole job of the check. An arm covers a member when its unguarded pattern provably matches every value of it: a type pattern naming the member or a supertype, an enumerator constant for that enumerator, a binding, an object or tuple pattern whose every sub-pattern covers the member's corresponding field - the subsumption the impossible-test rule already computes, run in the other direction - and a guard forfeits the arm's contribution, since the checker does not evaluate guards. That is what checks the opening example, a union of object types discriminated by a field, and what makes ```Node | null``` - the shape every ```tryParse``` in this proposal returns - exhaustive from ```when null:``` plus the subclass arms, with no ```default``` standing in for the case the types already name.

Two subjects stay deliberately outside the check. A *lone* object or tuple type is not checked, absent a qualifying ```where``` chain - a subset pattern is the point of matching on one, and a partial ```match``` over an event object is a program, not an oversight - which is why the union rule asks for two members; totality over one structural type is otherwise what ```is``` and an ```if``` are for. And a union containing string or number *literal-type* members is not checked, for the reason the README gives ```switch```: a closed set of strings that wants the check is an ```enum``` over ```string```, and this document keeps that decision rather than reopening it with a second exhaustiveness regime. Literal types remain patterns and remain discriminants inside object members; they do not become an enumerable subject by appearing in a union.

Unreachability is the check's other half, and it is an error, not a lint: an arm whose pattern cannot match what the preceding arms have left - subsumed by an earlier pattern, or impossible against the narrowed subject - is a compile-time TypeError, the dead-```case``` rule generalized. A ```default``` under a fully covered closed subject is unreachable by this rule, which pairs with the exhaustiveness rule to make the two halves one statement: **exactly one of "this ```match``` needs a catch-all" and "this ```match``` must not have one" holds**, and the checker says which. A closed subject covered case by case rejects the ```default``` it does not need; anything else demands one.

**Decoration cannot falsify a coverage claim.** A decorator may replace what it decorates, so the question is fair, and three rules of the [decorators](decorators.md) extension answer it - one per way a case could go missing. An atom set is read from a declaration and no context rewrites one: enums, enumerators, tuples, records, blocks, and bindings admit no return replacement at all, so the cases a ```match``` covers are the ones the declaration lists. A class replacement is a subclass: ```Reflect.Class``` does admit one, and a ```@singleton``` returning ```class extends T``` is the design's own example, but the declared return type is ```T``` and a class type is nominal, so only that declaration or a subclass satisfies it - and coverage is closed downward, so the arm naming the class matches whatever came back. Decorating a ```sealed``` base rather than one of its cases is the interesting one: a decorator in another module cannot return a subclass at all, since that is what ```sealed``` forbids, and one in the same module declares a case the checker reads like any other, so the check tightens rather than breaks. And a field replacement is a *value*: ```Reflect.ClassField``` replaces an initial value at the field's type, accessors replace at their declared types, so a discriminant still holds a value of its literal type and a member a pattern reads still has the type it was declared with.

What sits outside that claim is what sits outside every claim here: a decorator running arbitrary code can write a value that has drifted out of its type, and the backstop is the boundary check, not a property of ```match```.

```js
sealed abstract class Node {}
class Num extends Node { value: float64; }
class Neg extends Node { operand: Node; }
class Bin extends Node { op: Op; left: Node; right: Node; }

function evaluate(node: Node): float64 {
  return match (node) {
    when Num { let value }: value;
    when Neg { let operand }: -evaluate(operand);
    when Bin { let op, let left, let right }: apply(op, evaluate(left), evaluate(right));
  }; // Closed and covered: no default, no missing-return, and a new subclass breaks the build here
}
```

## The is Operator Takes a Pattern

```is``` is this proposal's boolean test - ```value is Payment``` from [dependent record types](dependentrecordtypes.md), running the structural check and the ```where``` clauses, narrowing on success. This document widens its right side from a type to a pattern, of which a type is now one form, so nothing existing changes meaning and one operator serves both questions:

```js
if (response is { status: 200, let body }) {
  render(body);                       // body in scope, typed as the narrowed member's field
}
while (input is not '') { ... }       // subtraction-narrowed in the loop body
```

Bindings from an ```is``` pattern scope to where the test being true implies they matched: the consequent of the ```if```, the right of ```&&```, the loop body - the narrowing positions the README already enumerates for boolean tests. In a position where truth implies nothing, the bindings do not exist. ```is``` with a pattern is the one-arm ```match```, and the equivalence is exact: ```subject is P``` matches, binds, and narrows precisely as ```when P``` would.

## What the Rest of the Proposal Provides

Checking the feature against its neighbours, which is where a design either compounds or contradicts:

**Composites** are the best-behaved subject the grammar has - frozen, canonical, getter-free - and the cheapest constant it can name: an interned composite in an expression pattern is one pointer comparison, and a ```match``` over composite constants compiles to the hash dispatch ```switch``` has always had for primitives, reaching compound keys. ```Composite``` carries the boolean matcher that makes ```when Composite:``` a membership test, ```Composite.<T>``` is a type pattern that narrows, tuple composites meet array patterns through their iterability, and the read-caching above is waived for a subject that cannot change. The [composites](composites.md) document works each of these through.

**Enums** are the closed sets. An enumerator is a constant pattern; a ```match``` over an enum is checked exhaustive; a string enum is how a closed set of strings gets both the ergonomics of ```when Status.Active:``` and the check the literal union declines. Sentinel enumerators appear in coverage as they do in a ```switch```, cased explicitly - typically to ```throw``` - to keep the check.

**Sealed classes** are the algebraic data types, and juxtaposition is their eliminator: ```when Bin { let op }:``` tests, narrows, and destructures in one clause, which is the shape the ```switch``` chapter's evaluator wanted and had to spell as a case plus member reads. Value type classes match structurally through their fieldwise ```===```: a value-class constant in an expression pattern compares by contents, so ```when origin:``` against a ```Vector2``` is the structural test with no protocol.

**Dependent record types** discriminate from the inside, and a ```match``` is one of the two forms a ```where``` chain is written in - ```where match (this.status) { ... }``` - carrying an implicit ```default: false;``` there, since a predicate that matches no clause has said the value is not of the type. A chain switching on one member against constants, in either form, denotes the union of its branches, so ```when { country: 'US', let postalCode }:``` narrows ```postalCode``` to ```USPostalCode``` because the discriminant chose the branch, and an arm per country is checked complete with no ```default```. That is the same coverage the opening example gets from a hand-written union, reaching the form written to avoid one.

```js
match (addr) {                                    // addr: Address
  when { country: 'US', let postalCode }: lookupZip(postalCode);
  when { country: 'CA', let postalCode }: lookupFSA(postalCode);
}                                                 // Covered: adding 'MX' breaks this
```

**Decorators** meet pattern matching from both sides. A ```match``` arm's body is a block, so it takes a block decorator with no new syntax - ```when P: @count { ... }``` - and ```Reflect.MatchArmBlock``` is that position's [context](decorators.md). Block decorators are the one per-entry position in that extension, running on every entry rather than once at the declaration, which makes an arm the natural thing to count: how often each case is actually taken is what tells an author their arms are in the wrong order, and source order is the one thing about a ```match``` that the author controls and the compiler cannot second-guess. In the other direction, a decorator is handed a *type* and must dispatch on its structure to emit a schema, a validator, or documentation, which is what type-subject patterns above are for; the two features need each other in opposite directions.

**Errors** split the work with typed ```catch```: ```catch (e: RangeError)``` remains the form at a ```try```, and ```match``` takes over when an error value is *data* - already caught, carried in a result field, matched on its type and destructured for its payload in one arm. The annotated binding is deliberately the same syntax in both.

**Generics** type the extractor protocol, as ```Some.<T>``` showed: inference from the subject is ordinary inference, and an ```Option```/```Result``` library is patterns all the way down with no machinery this document had to add. That the protocol needed nothing new is the strongest evidence the type system was ready.

**Types themselves** are matchable three ways, and the distinction is worth keeping straight. An expression pattern producing a type object tests *membership*, because that is what naming a type in a test position means everywhere else here. Identity - "is this exactly the type object ```uint8```" - is the interpolation ```${uint8}```, SameValue on interned objects. And ```extends``` asks the question a program dispatching on reflection actually has, whether the subject type is within another, binding its components as it goes. So matching over ```v``` wants type patterns, and matching over ```Reflect.typeOf(v)``` wants ```extends```. ```Reflect.matchType``` in [type programming](typeprogramming.md) is the same unification reached reflectively rather than syntactically - the relation ```is``` has to ```IsOfType``` - and stays the escape hatch for what the syntax does not spell.

## What the Engine Compiles

The recurring shape of this proposal - annotations converting dynamic discovery into straight-line code - lands hard here, because a ```match``` is a decision tree the source hands over whole. An enum subject compiles to the jump table its ```switch``` would; a sealed subject to a brand-tag dispatch, one load and one bounded table; composite and enumerator constants to pointer compares and, past a handful, a hash on the pointer; literal patterns over a typed field to monomorphic compares with no type check inside; object patterns over a known type to direct slot loads in one pass, the caching guarantee costing nothing because a known layout has nothing to cache; a discriminated union to a test of the discriminant followed by the member's straight-line destructure. The order-sensitivity budget is confined to where the source spent it: guards and custom matchers run user code in arm order, everything else is a pure test the compiler may reorder, factor, and share between arms, because patterns cannot observe each other.

## Limitations

Stated because they were found, not designed:

- **A guard belongs to a clause, not to a pattern.** ```is``` takes a pattern, so it cannot express "matches ```P1``` under this guard or ```P2``` under that one" - that needs ```x is P1 && g1 || x is P2 && g2```, which names the subject twice. The combinators cover every guard-free case, which is why this has not forced a second matching form; if the shape recurs, the answer is guards in patterns rather than one.

- **No negation types.** ```not``` and arm-failure narrow only what subtraction can represent - union members, sealed subclasses, literals, ```null```. A failed structural pattern narrows nothing, so ```default``` below ```when { let x }:``` still sees a subject that may have ```x```. Recording "lacks a member" would be a new kind of type, and no other feature here has needed one; the guard ```if (!('x' in v))``` is the workaround and is honest about being a runtime fact.
- **Array length does not narrow.** ```[let a, let b]``` tests length 2 and types the elements, but cannot retype a ```[].<uint8>``` as ```[2].<uint8>```, because those are distinct types with distinct values, not one type at two levels of knowledge. Code that wants a length-indexed type constructs one; a pattern only proves things a value's own type can express.
- **Literal-union exhaustiveness stays declined.** A ```match``` over ```'a' | 'b'``` needs a ```default``` however plainly the arms cover it. This is the standing decision restated, and the pressure a pattern language puts on it is real and is the strongest argument the enum-over-string form has ever had; if that decision is ever revisited, this document inherits the answer without changing.
- **Where-clause knowledge flows one way.** A type pattern naming a dependent record type runs its predicates; a structural pattern that happens to establish what a ```where``` clause would conclude - matching ```{ creditCard: let c }``` against ```Payment``` - narrows by the built-in field correspondences of [dependent record types](dependentrecordtypes.md) and no further. Patterns evaluate types' predicates; they do not prove new consequences from them. The exhaustiveness check reads a ```where``` chain in exactly one shape, the discriminating chain that denotes a union, and that reading is a syntactic normalization stated with the types rather than reasoning performed here.
