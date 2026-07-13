# Expression Parser

A tokenizer, recursive descent parser, and evaluator for arithmetic expressions - the smallest honest compiler frontend. This is the scenario that defines typed languages with algebraic data types: a token stream, a tree of node kinds, and functions that must handle every kind. Rust would write the AST as an enum, TypeScript as a discriminated union, Java as a sealed interface. This example writes it as a ```sealed``` class hierarchy with an exhaustive ```switch``` over the subclasses, which gives the same compile-time totality: adding a node kind is a compile error until every function handles it.

Features exercised:

- A ```uint8```-typed enum for token kinds, driving an exhaustive ```switch``` in the evaluator so adding an operator surfaces every site that must handle it.
- A typed generator as the tokenizer: ```function* tokenize(source: string): Token``` yields value type tokens the parser pulls on demand, no token array allocated.
- ```Token``` is a value type class - four fixed-width fields - so the generator's yields are copies, not heap objects.
- A ```sealed``` class hierarchy for the AST, discriminated by a ```switch``` whose case labels are the subclass type objects; with no ```default``` the compiler checks it covers every node kind, and each case narrows to its subclass.
- A typed error subclass caught by type: ```catch (e: ParseError)``` picks parse failures out from everything else and carries a typed position.

## Tokens

```js
enum TokenType: uint8 { Number, Plus, Minus, Star, Slash, LeftParen, RightParen, End };

class Token { // Value type: enum + float64 + one uint32
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
sealed class Node {
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

```NumberNode``` is a value type; ```Node``` is ```sealed```, which the main proposal's sealed-class rule makes a reference type, so ```UnaryNode``` and ```BinaryNode``` hold their children as plain ```Node``` references - a non-sealed value type class would need ```T | null``` for the same - and the nodes of a tree are ordinary heap objects, which is what a tree wants.

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

An exhaustive ```switch``` over the sealed hierarchy - case labels are the subclass type objects - then a ```switch``` on the operator enum inside the binary case:

```js
export function evaluate(node: Node): float64 {
	switch (node) {
		case NumberNode:
			return node.value; // node: NumberNode in this case
		case UnaryNode:
			return -evaluate(node.operand);
		case BinaryNode: {
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
	} // Exhaustive over Node's three subclasses: no default, no trailing return
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

Sum types are the feature every comparison language leads with, and the proposal now expresses them directly. Most of what earlier drafts recorded as gaps here has since been specified; these notes track what's resolved and the one pattern left deliberately to nominal typing:

- **Closed hierarchies - resolved.** ```Node``` is ```sealed```, so the evaluator's ```switch``` over its subclasses is checked exhaustive: with no ```default```, adding a node kind is a compile-time error until every evaluator handles it, the same totality Rust's enum and Java's sealed interface give. There is no ```Unknown node kind``` fallthrow because the compiler proves there is no fourth kind. The main proposal demonstrates the sealed-class switch on this very hierarchy.
- **Recursive types - resolved.** The classes are nominal and hoisted, so ```operand: Node``` refers back to the sealed base without ceremony. The union spelling ```type Node = NumberNode | UnaryNode | BinaryNode``` is equally valid now (a union of classes is a reference), and the main proposal's structural-identity rule handles the structural case coinductively, terminating each cycle at its reference position. The sealed class form used here is the closed one; a bare union is the open one.
- **Narrowing composition.** ```instanceof``` narrows (per its README section), and the enum ```switch``` narrows, but narrowing on a *field* (```if (node.op == TokenType.Plus)```) discriminating the object holding it is the TypeScript-style pattern this proposal has no rule for, and with nominal classes it doesn't need one - worth saying explicitly so it isn't assumed.
- **Enum-to-string in errors - resolved.** ```TokenType.toString(current.type)``` gives the enumerator's key for messages. The enum section now also lets a ```string```-underlying enum interpolate as its value directly; token kinds here are a ```uint8``` enum, whose interpolation is the underlying number, so ```toString``` is the right call for a readable name.
- **```:=``` for typed construction - resolved.** ```{ ... } := BinaryNode``` in a ```return``` is blessed by the typed-assignment section, which uses this same construct-and-return shape; it fills the class layout from the literal without running a constructor.
- **Generator plus lookahead - resolved.** ```tokens.next().value``` types as ```Token``` because ```tokenize```'s yield and return types coincide; the iteration section specifies ```Generator.<Y, R, N>``` and that ```next()``` returns ```{ value: Y | R, done: boolean }```, narrowing on ```done```.
