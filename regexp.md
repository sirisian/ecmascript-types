# Typed Regular Expressions

A regular expression literal is a compile-time constant that engines already parse before the program runs. This extension puts that parse to work in the type system: the number and kind of capture groups, the names of named groups, and the flags all become part of the literal's type. Match results then have exact shapes instead of being growable arrays with an attached dictionary, callbacks passed to ```replace``` get typed parameters instead of positional ```arguments```, and a mistyped group name is a compile-time TypeError rather than an ```undefined``` discovered in production.

The performance goal is served directly. When the shape of a match is known at the call site an engine can allocate the result as a fixed tuple with a fixed-shape ```groups``` object, elide ```groups``` and ```indices``` entirely when the program never reads them, and compile the matcher once at the literal rather than on first use.

## Pattern Types

A regular expression has type ```RegExp.<Captures, Groups>```, where ```Captures``` is a tuple of the capture group types in source order and ```Groups``` is an object type of the named groups. Both are inferred from a literal:

```js
const date = /(\d{4})-(\d{2})-(\d{2})/;
// date: RegExp.<[string, string, string], {}>

const named = /(?<year>\d{4})-(?<month>\d{2})/;
// named: RegExp.<[string, string], { year: string, month: string }>
```

A named group also occupies a numbered position, so it appears in both ```Captures``` and ```Groups```. A pattern with no groups is ```RegExp.<[], {}>```, which is the type of ```RegExp``` used without type arguments.

### Capture Types

A capture that can fail to participate in a match is ```string | undefined```. That covers optional groups, groups behind a quantifier that may match zero times, and groups in an alternation branch that isn't taken:

```js
const a = /(\d+)(\.\d+)?/;
// a: RegExp.<[string, string | undefined], {}>

const b = /(?:(cat)|(dog))/;
// b: RegExp.<[string | undefined, string | undefined], {}>

const c = /(?<protocol>https?):\/\/(?<port>\d+)?/;
// c: RegExp.<[string, string | undefined], { protocol: string, port: string | undefined }>
```

The rule is syntactic and decidable from the pattern alone: a group is non-optional when every path through the pattern that produces a match enters it exactly once.

## Match Results

```js
type RegExpExecArray<Captures extends [] = [], Groups = {}> = [string, ...Captures] & {
	index: uint32,
	input: string,
	groups: Groups
};

class RegExp<Captures extends [] = [], Groups = {}> {
	exec(value: string): RegExpExecArray.<Captures, Groups> | null;
	test(value: string): boolean;
	get source(): string;
	get flags(): string;
	get lastIndex(): uint32;
	set lastIndex(value: uint32);
}
```

Element ```0``` is always the whole match, so the result tuple is the captures prefixed with a ```string```. Indexing past the end of the tuple is a compile-time TypeError, and ```groups``` is a fixed-shape object rather than a null-prototype dictionary:

```js
const match = /(?<year>\d{4})-(\d{2})/.exec(input);
if (match != null) {
	const whole: string = match[0];
	const year: string = match[1];
	const month: string = match[2];
	// match[3]; // TypeError: index 3 is out of range for [string, string, string]
	match.groups.year; // string
	// match.groups.day; // TypeError: no group named 'day'
}
```

```exec``` returns a nullable union, so the null check is not optional; reading a property off the result without narrowing it is a TypeError, and the short-circuiting operators work as they do anywhere else:

```js
const year = /(?<year>\d{4})/.exec(input)?.groups.year ?? '0000';
```

### String Methods

```js
class String {
	match<C extends [], G>(pattern: RegExp.<C, G>): RegExpExecArray.<C, G> | null;
	matchAll<C extends [], G>(pattern: RegExp.<C, G>): Iterator.<RegExpExecArray.<C, G>>;
	search(pattern: RegExp): int32;
	split<C extends [], G>(pattern: RegExp.<C, G>, limit?: uint32): [].<string>;
}
```

With the ```g``` flag, ```match``` returns ```[].<string> | null``` instead, since the per-match capture information isn't retained. That difference is visible in the type, which is a good reason to prefer ```matchAll```:

```js
const all = input.matchAll(/(?<key>\w+)=(?<value>\w+)/g);
for (const m of all) {
	m.groups.key; // string, inferred through the iterator
}
```

## Replacement

The callback form of ```replace``` currently receives the whole match, then one argument per capture group, then the offset, the input, and possibly a groups object - a positional signature that no type system built on annotations can express. Here the captures are a typed rest parameter drawn from the pattern:

```js
class String {
	replace<C extends [], G>(
		pattern: RegExp.<C, G>,
		replacement: string | ((match: string, ...captures: C, offset: uint32, input: string, groups: G) => string)
	): string;
	replaceAll<C extends [], G>(
		pattern: RegExp.<C, G>,
		replacement: string | ((match: string, ...captures: C, offset: uint32, input: string, groups: G) => string)
	): string;
}
```

```js
const swapped = '2026-07-09'.replace(
	/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/,
	(match, year, month, day, offset, input, groups) => `${day}/${month}/${year}`
	// year, month, day: string. offset: uint32. groups: { year: string, month: string, day: string }
);
```

Named references in a string replacement are checked against ```Groups``` as well, so a typo fails at compile time rather than inserting the literal text:

```js
'a'.replace(/(?<x>a)/, '$<x>'); // 'a'
// 'a'.replace(/(?<x>a)/, '$<y>'); // TypeError: no group named 'y' in the pattern
```

## Flags

Flags that change the shape of a result are reflected in the type. The ```d``` flag adds ```indices```, a tuple parallel to the match with an optional ```groups``` of its own:

```js
type RegExpIndices<Captures extends [], Groups> = [[uint32, uint32], ...Captures] & {
	groups: Groups
};
```

```js
const m = /(?<year>\d{4})/d.exec(input);
if (m != null) {
	const [start, end] = m.indices[1]; // [uint32, uint32]
	m.indices.groups.year; // [uint32, uint32] | undefined
}
```

Without ```d```, reading ```indices``` is a TypeError rather than ```undefined```. A capture typed ```string | undefined``` has its index entry typed ```[uint32, uint32] | undefined``` to match.

The remaining flags don't change result shapes, but ```g``` and ```y``` make ```lastIndex``` meaningful and select the ```match``` overload above, and ```u```/```v``` are checked when the pattern is parsed, so an invalid escape is a compile-time SyntaxError at the literal.

## Dynamic Patterns

A pattern assembled at runtime has no literal to parse, so it takes the untyped shape - captures of ```string | undefined``` and an index-signature groups object:

```js
const dynamic = new RegExp(userSupplied);
// dynamic: RegExp.<[].<string | undefined>, { [key: string]: string | undefined }>
const m = dynamic.exec(input);
m?.[1]; // string | undefined
m?.groups?.['year']; // string | undefined
```

A program that knows more than the compiler can say so, and the assertion is checked when the ```RegExp``` is constructed. If the compiled pattern's group count or names disagree with the type arguments the constructor throws a TypeError, which is the ordinary runtime-types tradeoff: the check happens once, at the boundary, instead of at every read:

```js
const parser = new RegExp.<[string, string], { year: string, month: string }>(patternText);
// TypeError at construction if patternText doesn't have exactly these two named groups
parser.exec(input)?.groups.year; // string | undefined, typed thereafter
```

A pattern built from a template with an interpolated ```RegExp``` composes the group tuples of its parts, which is how a dynamically assembled pattern can stay typed.

## Narrowing Strings

A regular expression is also a constraint. The ```StringBounds``` meta type in [primitive metadata](primitivemetadata.md) carries a ```pattern``` field, and ```string.<{ pattern: /.../ }>``` is the type of strings known to match it. A successful ```test``` against a constant pattern narrows its argument to that type in the true branch, the same way ```instanceof``` narrows in the main proposal:

```js
const EmailPattern = /^[^@\s]+@[^@\s]+$/;
type Email = string.<{ pattern: EmailPattern }>;

function send(to: Email) {}

function f(input: string) {
	// send(input); // TypeError: string is not assignable to Email
	if (EmailPattern.test(input)) {
		send(input); // input: Email in this branch
	}
}
```

This is the nominal counterpart to the structural ```is``` operator, which tests the same constraint directly:

```js
if (input is Email) {
	send(input);
}
```

Because the constraint lives in the type, a cast validates and an interface field declares it once for every boundary that touches the value, including the parsers described in [Serialization](serialization.md):

```js
const address = Email(input); // TypeError if it doesn't match
type Signup = { email: Email, age: uint8 };
JSON.parse.<Signup>(body); // the pattern runs during the parse
```

## Custom Matchers

The well-known symbol methods let a class stand in for a pattern. Typing them is what makes a custom matcher's results as precise as a literal's:

```js
class Delimited {
	constructor(separator: string) {}

	[Symbol.split](value: string, limit?: uint32): [].<string> {}
	[Symbol.match](value: string): RegExpExecArray.<[], {}> | null {}
	[Symbol.search](value: string): int32 {}
	[Symbol.replace](value: string, replacement: string | ((match: string) => string)): string {}
}

'a:b:c'.split(new Delimited(':')); // [].<string>
```

## Notes

- The compiler parses regular expression literals to derive these types. Engines already do this work, so the cost is in the specification rather than at runtime, and it's the same analysis that reports an invalid pattern today.
- Backreferences and lookaround don't add captures and don't change the tuple. Duplicate named groups in separate alternation branches, allowed by the ```v``` flag rules, contribute a single ```string | undefined``` entry to ```Groups```.
- Nothing here changes the matching semantics of a regular expression. Every rule above is about the *shape* of what a match produces, which is exactly the part that is currently untyped.
