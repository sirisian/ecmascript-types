# Expression Parser

A tokenizer, recursive descent parser, and evaluator for arithmetic expressions - the smallest honest compiler frontend. This is the scenario that defines typed languages with algebraic data types: a token stream, a tree of node kinds, and functions that must handle every kind. Rust would write the AST as an enum, TypeScript as a discriminated union, Java as a sealed interface. This example writes it with what the proposal has - classes, ```instanceof``` narrowing, and enums with exhaustive ```switch``` - and records where that falls short of the languages above.

Features exercised:

- A ```uint8```-typed enum for token kinds, driving an exhaustive ```switch``` in the evaluator so adding an operator surfaces every site that must handle it.
- A typed generator as the tokenizer: ```function* tokenize(source: string): Token``` yields value type tokens the parser pulls on demand, no token array allocated.
- ```Token``` is a value type class - four fixed-width fields - so the generator's yields are copies, not heap objects.
- ```instanceof``` narrowing over a class hierarchy as the AST discrimination mechanism, with the narrowed field access checked in each branch.
- A typed error subclass caught by type: ```catch (e: ParseError)``` picks parse failures out from everything else and carries a typed position.

## Tokens

```js
enum TokenType: uint8 { Number, Plus, Minus, Star, Slash, LeftParen, RightParen, End };

class Token { // Value type: enum + float64 + two uint32s
	type: TokenType;
	value: float64; // Meaningful when type is Number
	position: uint32;
}

class ParseError extends SyntaxError {
	position: uint32;
	constructor(message: string, position: uint32) {
		super(`${message} at ${position}`);
		this.position = position;
	}
}
```

## Tokenizer

```js
function* tokenize(source: string): Token {
	let index: uint32 = 0;
	while (index < source.length) {
		const start = index;
		const code = source.charCodeAt(index);
		switch (code) {
			case 0x20: // space
				++index;
				continue;
			case 0x2B: // +
				++index;
				yield { type: TokenType.Plus, value: 0, position: start };
				continue;
			case 0x2D: // -
				++index;
				yield { type: TokenType.Minus, value: 0, position: start };
				continue;
			case 0x2A: // *
				++index;
				yield { type: TokenType.Star, value: 0, position: start };
				continue;
			case 0x2F: // /
				++index;
				yield { type: TokenType.Slash, value: 0, position: start };
				continue;
			case 0x28: // (
				++index;
				yield { type: TokenType.LeftParen, value: 0, position: start };
				continue;
			case 0x29: // )
				++index;
				yield { type: TokenType.RightParen, value: 0, position: start };
				continue;
		}
		if (code >= 0x30 && code <= 0x39) { // 0-9
			while (index < source.length && ((source.charCodeAt(index) >= 0x30 && source.charCodeAt(index) <= 0x39) || source.charCodeAt(index) == 0x2E)) {
				++index;
			}
			yield { type: TokenType.Number, value: float64.parse(source.slice(start, index)), position: start };
			continue;
		}
		throw new ParseError(`Unexpected character '${source[index]}'`, index);
	}
	yield { type: TokenType.End, value: 0, position: uint32(source.length) };
}
```

An object literal where a ```Token``` is expected constructs one through the matching-shape cast, the same rule the routing examples in [decorators](../decorators.md) rely on.

## AST

```js
class Node {
	position: uint32;
}
class NumberNode extends Node {
	value: float64;
}
class UnaryNode extends Node {
	op: TokenType;
	operand: Node;
}
class BinaryNode extends Node {
	op: TokenType;
	left: Node;
	right: Node;
}
```

```NumberNode``` is a value type; ```UnaryNode``` and ```BinaryNode``` hold references, so nodes of a tree are ordinary heap objects, which is what a tree wants.

## Parser

Recursive descent with precedence climbing. The parser owns one token of lookahead pulled from the generator:

```js
export function parse(source: string): Node {
	const tokens = tokenize(source);
	let current: Token = tokens.next().value;

	function advance(): Token {
		const consumed = current;
		current = tokens.next().value;
		return consumed;
	}

	function expect(type: TokenType) {
		if (current.type != type) {
			throw new ParseError(`Expected ${TokenType.toString(type)}, found ${TokenType.toString(current.type)}`, current.position);
		}
		advance();
	}

	function precedence(type: TokenType): uint8 {
		switch (type) {
			case TokenType.Plus:
			case TokenType.Minus:
				return 1;
			case TokenType.Star:
			case TokenType.Slash:
				return 2;
			default:
				return 0;
		}
	}

	function primary(): Node {
		if (current.type == TokenType.Number) {
			const token = advance();
			return { position: token.position, value: token.value } := NumberNode;
		}
		if (current.type == TokenType.Minus) {
			const token = advance();
			return { position: token.position, op: TokenType.Minus, operand: primary() } := UnaryNode;
		}
		if (current.type == TokenType.LeftParen) {
			advance();
			const node = expression(1);
			expect(TokenType.RightParen);
			return node;
		}
		throw new ParseError('Expected a number, unary minus, or parenthesis', current.position);
	}

	function expression(minimum: uint8): Node {
		let left = primary();
		while (precedence(current.type) >= minimum) {
			const op = advance();
			const right = expression(precedence(op.type) + 1);
			left = { position: op.position, op: op.type, left, right } := BinaryNode;
		}
		return left;
	}

	const root = expression(1);
	expect(TokenType.End);
	return root;
}
```

## Evaluator

Discrimination by ```instanceof```, then an exhaustive ```switch``` on the operator enum inside each class branch:

```js
export function evaluate(node: Node): float64 {
	if (node instanceof NumberNode) {
		return node.value; // node: NumberNode in this branch
	}
	if (node instanceof UnaryNode) {
		return -evaluate(node.operand);
	}
	if (node instanceof BinaryNode) {
		const left = evaluate(node.left);
		const right = evaluate(node.right);
		switch (node.op) {
			case TokenType.Plus: return left + right;
			case TokenType.Minus: return left - right;
			case TokenType.Star: return left * right;
			case TokenType.Slash:
				if (right == 0) {
					throw new RangeError('Division by zero');
				}
				return left / right;
			default:
				throw new ParseError('Not a binary operator', node.position);
		}
	}
	throw new TypeError('Unknown node kind'); // Unreachable... says the author, not the compiler
}
```

```js
try {
	evaluate(parse('1 + 2 * (3 - 4) / -2')); // 2
	evaluate(parse('1 + '));
} catch (e: ParseError) {
	console.error(e.message, e.position); // Typed catch selects parse failures
} catch (e: RangeError) {
	console.error('math error', e.message);
}
```

## Coverage Notes

This is the example that presses hardest on what the proposal doesn't have, because sum types are the one feature every comparison language leads with:

- **No closed hierarchies.** The evaluator's final ```throw new TypeError('Unknown node kind')``` is the tell. ```Node``` has exactly three subclasses, but nothing lets the author say so, so the compiler can't verify the ```instanceof``` chain is exhaustive the way the enum ```switch``` is. A ```sealed``` modifier (or ```enum```-of-classes) would make "handled every node kind" a compile-time fact. This is the single largest expressiveness gap the examples found.
- **Recursive type aliases.** The class-based AST sidesteps it because class names are nominal and hoisted, but the union spelling ```type Node = NumberNode | UnaryNode | BinaryNode``` with those classes referring back to ```Node``` is the natural TypeScript/Rust shape, and [type records](../typerecords.md) currently notes recursive types as unresolved. The classes-work/aliases-don't asymmetry should be stated somewhere on purpose.
- **Narrowing composition.** ```instanceof``` narrows (per its README section), and the enum ```switch``` narrows, but narrowing on a *field* (```if (node.op == TokenType.Plus)```) discriminating the object holding it is the TypeScript-style pattern this proposal has no rule for, and with nominal classes it doesn't need one - worth saying explicitly so it isn't assumed.
- **Enum-to-string in errors.** ```TokenType.toString(current.type)``` comes from the enum prototype functions; it reads clumsily in an interpolation. An enum value participating in template literals via ```toString``` directly (```${current.type}``` printing ```Star```) would be a small quality addition to the enum section.
- **```:=``` for typed literals.** ```{ ... } := BinaryNode``` is doing real work here - constructing a class instance from a literal with inference. It's specified under typed assignment for bindings; its use in a ```return``` expression position (construct-and-return) is an extrapolation this example depends on and the README should bless or reject.
- **Generator plus lookahead.** ```tokens.next().value``` types as ```Token``` mid-iteration but ```Token | string``` once the generator's return type differs from its yield type; here both are ```void```-returning so it stays ```Token```. The iteration section covers ```Generator.<Y, R, N>```; a sentence on ```next()```'s result typing (```{ value: Y | R, done: boolean }``` and how it narrows on ```done```) would close the loop.
