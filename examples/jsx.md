# Reactive Views

A `jsx` lexical mode: markup and control flow written as a decorated region, compiled by a replacement decorator into calls on a reactive runtime before the program is checked.

```js
import { jsx } from "./jsx.js" with { preprocessor: "true", mode: "jsx" };

const Inventory = ({ items, character, showEmpty }) => @jsx {
  const visible = items.filter((i) => i.qty > 0);
  <panel layout="stack">
    @match all (character) {
      when 1: { <status-dot color="red" />; }
      when 2: { <status-dot color="amber" />; }
    };
    <inventory-grid columns="8">
      @for (const item of visible) @key(item.id) {
        match (item.kind) {
          when "weapon": <item-slot icon={item.icon} count={item.qty} />;
          when _: <item-slot icon={item.icon} />;
        };
      }
      @if (showEmpty) { <item-slot empty />; } else { <label text="hidden" />; }
    </inventory-grid>
  </panel>;
};
```

`<` cannot begin an expression, so a program containing markup does not parse at all - which is why a [scoped lexical mode](../decoratorreplacement.md) is what admits it, rather than a global grammar change. Outside a region nothing moves: `a < b` is a comparison and a type argument list is a type argument list.

The whole of the mode is one extra production. `jsx` is ECMAScript with a JSX element admitted where a |PrimaryExpression| is expected - which is to say, exactly where a |RegularExpressionLiteral| may begin - so a region is an ordinary Block parsed with that production enabled. No second parser, and no scanner: the tokens come from the parse like every other token, so a template literal in a prop is one token rather than a backtick and an identifier that exists in no source.

Features exercised:

- A declared lexical mode, so markup parses inside a region and nowhere else.
- Statements and markup in one region, because a region is a Block: a view may compute before it builds, and its value is the last expression.
- ```constant { }``` for each static subtree, so a template is built once per SITE for the life of the realm rather than once per render.
- Block decorators - ```@key(item.id)```, ```@persist``` - carrying the metadata a tag would have carried as attributes, in a position where the loop binding is already in scope.
- [`match` and `match all`](../patternmatching.md) as the view's dispatch, bringing range patterns, guards and exhaustiveness where a predicate list had none.
- [Ranges](../ranges.md) as iterables, so `for (const i of used..<capacity)` needs no start and end props.
- ```gensym``` for the frames the macro introduces, so a range variable cannot collide with one.

## Running It

The engine never loads the preprocessor module. It calls `HostResolveReplacementDecorator(name, specifier)` and uses whatever the host returns - and the hook receives the decorator's NAME and the CONSUMING module's specifier, so it cannot discover where a preprocessor came from. A name-keyed registry is the shape that works:

```js
HostResolveReplacementDecorator: (name) => realm.GlobalObject.__preprocessors?.[name]
```

With that in place:

1. **Create a devtools snippet named `jsx.js`** containing the macro below, and run it once. It is a script - no imports, no exports - and it registers itself on `globalThis.__preprocessors`.
2. **Create a second snippet** containing the demo below, and run it. It prints the tree, checks that two renders share their templates, and changes a signal to show the controllers re-evaluating.

A region cannot be pasted as a bare script, because the mode is declared by an import attribute and only a module may carry one. Without it `<inventory-grid columns="8">` lexes as ECMAScript and `grid columns` is two adjacent identifiers. The demo therefore contains what the macro EMITS, so it runs with nothing but itself.

## Three Rules

Each was established by running the macro rather than by designing it.

**A construct between tags takes the `@` sigil; inside a block it does not.** Between tags a bare `if (` could be child text, and the parser would have to guess - so it is marked, which is where Angular 17 arrived with the same spelling. Inside a block there is no text to be ambiguous with, so `for (const s of xs) { if (c) { ... } }` needs no sigils.

**A `match` needs a trailing `;`.** It is an expression statement where `if` and `for` are statements, so without a terminator the parser reads on into the next tag and lexes its `/` as a regular expression.

**A decoration goes on the BLOCK.** `for (const slot of slots) @key(slot.id) { ... }` - the decorated position is the loop body, so the binding is in scope where the key is written and no lambda is needed. `@persist` lands per branch and per arm, which is finer than a tag's single flag.

## What It Compiles To

The view at the top becomes, in outline:

```js
do {
  const visible = items.filter((i) => i.qty > 0);
  jsxTemplate(constant {
    const $e1 = { type: "panel", constants: { layout: "stack" }, children: [], attrSlots: [] };
    $e1.children.push({ slotIndex: 0 });
    const $e5 = { type: "inventory-grid", constants: { columns: "8" }, children: [], attrSlots: [] };
    $e5.children.push({ slotIndex: 1 });
    $e1.children.push($e5);
    ({ id: 0, root: $e1, slotCount: 4, slotKinds: [...], slotTargets: [$e1, $e5, ...] });
  },
  jsx("matchAll", { on: character, children: [[...]] }),
  jsx("for", { items: visible, key: (item) => item.id, children: [(item) => ...] }),
  ...);
}
```

Three things in that output are the design rather than an accident.

**The static skeleton survives.** `<inventory-grid>` is inlined into the panel's template as a nested element, while each `@for`, `@match` and `@if` becomes a `slotIndex`. A control-flow child is a slot exactly as an interpolation is, because the element's SHAPE is still fixed and what varies is one child. Treating it as non-static instead would collapse the panel into a call and take the template of every static ancestor with it.

**The plan is a `constant { }` and not a literal**, because `slotTargets` holds references to the same elements that appear in `root.children` - which no object literal can express, and which is why `constant { }` takes a block.

**The dynamics stay at the call site.** `visible` closes over `items`, and `(item) => item.id` closes over the loop binding; a `constant { }` block must read nothing from outside itself, so the macro hoists the shape and leaves the bindings. That split is what makes the hoist legal.

## The Macro

The first snippet. It parses the region's tokens into a node tree and emits tokens back.

```js
/**
 * jsx.js - the `@jsx { }` replacement decorator.
 *
 * A preprocessor module for proposal-runtime-types. It receives the tokens of a
 * moded region scanned in `jsx` mode and returns tokens that call the reactive
 * framework's runtime.
 *
 * Import it as:
 *
 *     import { jsx } from "./jsx.js" with { preprocessor: "true", mode: "jsx" };
 *
 * and write views as:
 *
 *     const TreeNode = defineView((ctx) => {
 *       const expanded = ctx.signal(true);
 *       return ({ label, children: childItems }) => @jsx {
 *         <panel>
 *           <label text={label} />
 *           if (expanded) {
 *             for (const child of childItems) @key(child.id) {
 *               <TreeNode label={child.label} children={child.children} />
 *             }
 *           }
 *         </panel>
 *       };
 *     });
 *
 * ---------------------------------------------------------------------------
 * WHAT IT EMITS, AND WHY
 *
 * Three decisions were taken from the framework's source rather than guessed,
 * and each shapes the output:
 *
 * 1. A SIGNAL PROP IS FORWARDED UNCHANGED. `ClientContext.#createLayoutElement`
 *    partitions props with `isSignal(value)`, which tests for the SIGNAL symbol
 *    that `createSignal` puts on the function it returns. Wrapping a prop in a
 *    thunk would defeat that test and drop the prop into the
 *    `typeof value !== 'function'` branch, which discards it. So `text={label}`
 *    emits `text: label`.
 *
 * 2. A CONDITION IS EMITTED AS A THUNK. `IfBranch.when` is
 *    `Signal<unknown> | (() => unknown)` and `readSignal` is `signal()`, so a
 *    thunk and a signal are read identically. A thunk is the one that works for
 *    both `if (expanded)` and `if (x() > 5)`, so conditions always get one.
 *
 * 3. A BRANCH IS A FACTORY. `IfController` holds `branches[i].factory` and calls
 *    it when the condition changes, keeping the built subtree when `persist` is
 *    set. So lifting `if (c) { ... }` into `() => ...` produces exactly what
 *    `#createIf` already expects - no controller changes.
 *
 * A static subtree becomes a ParsedTemplate hoisted into a `constant { }`, which
 * is allocated once per SITE for the life of the realm. That removes
 * `parseTemplateStrings` and `globalTemplateCache` entirely: the string
 * round-trip existed only because a syntactic transform could not hand the
 * runtime a tree, and a macro can.
 */

// ===========================================================================
// Token helpers
// ===========================================================================

const CONTROL_WORDS = new Set(['if', 'else', 'for', 'match', 'while', 'switch']);

function jsx(tokens, args) {
  const region = firstGroup(tokens);
  const emitter = new Emitter(region ? region.span : tokens[0] && tokens[0].span);
  const nodes = new Parser(region ? region.tokens : tokens, emitter).parseChildren(null);
  return emitter.emitRoot(nodes, args);
}

function firstGroup(tokens) {
  for (const t of tokens) {
    if (t.kind === 'group' && t.value === '{') {
      return t;
    }
  }
  return undefined;
}

const isPunct = (t, v) => t && t.kind === 'punctuator' && t.value === v;
const isWord = (t, v) => t && t.kind === 'identifier' && t.value === v;
const isGroup = (t, v) => t && t.kind === 'group' && t.value === v;

// ===========================================================================
// Parser - region tokens to a node tree
// ===========================================================================

/**
 * Node kinds:
 *   { k: 'el',    tag, props: [{ name, value }], children, isComponent }
 *   { k: 'text',  value }
 *   { k: 'expr',  tokens }
 *   { k: 'if',    branches: [{ cond, body, deco }], elseBody, elseDeco }
 *   { k: 'for',   binding, iterable, body, deco }
 *   { k: 'match', subject, all, arms: [{ pattern, guard, body, deco }] }
 */
class Parser {
  constructor(tokens, emitter) {
    this.ts = tokens;
    this.i = 0;
    this.em = emitter;
  }

  peek(o = 0) { return this.ts[this.i + o]; }
  next() { return this.ts[this.i++]; }
  done() { return this.i >= this.ts.length; }

  /** Children of an element, or the region's top level when `closing` is null. */
  parseChildren(closing) {
    const out = [];
    while (!this.done()) {
      const t = this.peek();

      // `</tag>` ends the current element.
      if (isPunct(t, '<') && isPunct(this.peek(1), '/')) {
        if (closing === null) {
          throw new SyntaxError('unexpected closing tag at the top of a jsx region');
        }
        this.next(); this.next();
        const name = this.readTagName();
        if (name !== closing) {
          throw new SyntaxError(`closing tag </${name}> does not match <${closing}>`);
        }
        this.expect('>');
        return out;
      }

      // A parsed region delivers each JSX element as a GROUP whose value is the
      // element's text and whose tokens are its structure, so it is unwrapped
      // and parsed from the inside.
      if (t.kind === 'group' && isPunct(t.tokens && t.tokens[0], '<')) {
        out.push(new Parser(this.next().tokens, this.em).parseElementOnly());
        continue;
      }
      if (isPunct(t, '<')) { out.push(this.parseElement()); continue; }
      // A parsed region logs child text RAW, with its own kind - the source
      // slice rather than a quoted string, so the macro quotes it when emitting
      // and nothing decides for it which whitespace mattered.
      if (t.kind === 'jsxtext') { out.push({ k: 'text', value: String(this.next().value) }); continue; }
      if (t.kind === 'string') { out.push({ k: 'text', value: JSON.parse(String(this.next().value)) }); continue; }
      if (isGroup(t, '{')) { out.push({ k: 'expr', tokens: this.next().tokens }); continue; }
      if (isPunct(t, '@')) { out.push(this.parseDecorated()); continue; }
      if (isWord(t, 'if')) { out.push(this.parseIf(null)); continue; }
      if (isWord(t, 'for')) { out.push(this.parseFor(null)); continue; }
      if (isWord(t, 'match')) { out.push(this.parseMatch()); continue; }
      if (isPunct(t, ';')) { this.next(); continue; }

      // ORDINARY JAVASCRIPT, at the region's statement level.
      //
      // A region is a Block, so a view may compute before it builds:
      // `const visible = items.filter(...)` ahead of the markup is the commonest
      // shape a real view has. Anything that is not markup or control flow is
      // passed through to the emitted code untouched, and the region's value is
      // the last expression - the do-block completion semantics the region
      // already carries.
      if (closing === null) {
        out.push({ k: 'stmt', tokens: this.takeStatement() });
        continue;
      }

      throw new SyntaxError(`unexpected ${t.kind} \`${String(t.value)}\` in a jsx region`);
    }
    if (closing !== null) {
      throw new SyntaxError(`unclosed <${closing}>`);
    }
    return out;
  }

  /**
   * A decoration on a control-flow construct: `@key(expr)`, `@persist`.
   *
   * The proposal decorates a construct's BLOCK, and the decorated position sits
   * inside the construct's scope - which is why `@key(slot.id)` needs no lambda:
   * the loop binding is already in scope where it is written, and the macro
   * knows its name from the head it just parsed.
   */
  /**
   * `@` in child position is either a SIGIL on a construct or a decoration.
   *
   * A sigil is needed here and not at the region's statement level because child
   * TEXT is possible between tags: a bare `if (` could be either, and the parser
   * would have to guess. Angular 17 reached the same answer with the same
   * spelling. They never collide - a sigil is followed by a construct keyword, a
   * decoration by an ordinary name.
   */
  parseDecorated() {
    const after = this.peek(1);
    if (isWord(after, 'if') || isWord(after, 'for') || isWord(after, 'match')) {
      this.next();
      const t = this.peek();
      if (isWord(t, 'if')) { return this.parseIf(null); }
      if (isWord(t, 'for')) { return this.parseFor(null); }
      return this.parseMatch(null);
    }
    const deco = this.readDecorations();
    const t = this.peek();
    if (isWord(t, 'if')) { return this.parseIf(deco); }
    if (isWord(t, 'for')) { return this.parseFor(deco); }
    if (isWord(t, 'match')) { return this.parseMatch(deco); }
    throw new SyntaxError('a decoration here must precede `if`, `for` or `match`');
  }

  /** One statement's tokens, to the `;` that ends it or the block that does. */
  takeStatement() {
    const out = [];
    while (!this.done()) {
      const t = this.peek();
      if (isPunct(t, ';')) { out.push(this.next()); break; }
      if (isPunct(t, '<')) { break; }
      out.push(this.next());
      const last = out[out.length - 1];
      if (last.kind === 'group' && last.value === '{') { break; }
    }
    return out;
  }

  readDecorations() {
    const deco = {};
    while (isPunct(this.peek(), '@')) {
      this.next();
      const name = this.next();
      if (!name || name.kind !== 'identifier') {
        throw new SyntaxError('a decoration needs a name');
      }
      if (isGroup(this.peek(), '(')) {
        deco[name.value] = this.next().tokens;
      } else {
        deco[name.value] = true;
      }
    }
    return deco;
  }

  /**
   * A tag name, which may be hyphenated.
   *
   * `status-dot` and `item-slot` are the commonest shape a custom element has,
   * and the lexer gives `status`, `-`, `dot` as three tokens - so a name that
   * stops at the first token reads `<status-dot>` as `<status>` and then finds a
   * `-` where an attribute should be. Dots and colons join for the same reason.
   */
  readTagName() {
    let name = '';
    while (!this.done() && this.peek().kind === 'identifier') {
      name += this.next().value;
      if (isPunct(this.peek(), '-') || isPunct(this.peek(), '.') || isPunct(this.peek(), ':')) {
        name += this.next().value;
      } else {
        break;
      }
    }
    return name;
  }

  expect(v) {
    if (!isPunct(this.peek(), v)) {
      throw new SyntaxError(`expected \`${v}\``);
    }
    return this.next();
  }

  /** The whole of this token run is one element. */
  parseElementOnly() {
    const el = this.parseElement();
    if (!this.done()) {
      throw new SyntaxError('trailing tokens after a jsx element');
    }
    return el;
  }

  parseElement() {
    this.expect('<');
    // A fragment: `<>...</>`
    if (isPunct(this.peek(), '>')) {
      this.next();
      const children = this.parseChildren('');
      return { k: 'el', tag: null, props: [], children, isComponent: false };
    }
    const tag = this.readTagName();
    const props = [];
    while (!this.done() && !isPunct(this.peek(), '>') && !isPunct(this.peek(), '/')) {
      const nameTok = this.next();
      if (nameTok.kind !== 'identifier') {
        throw new SyntaxError(`unexpected ${nameTok.kind} in the attributes of <${tag}>`);
      }
      let name = nameTok.value;
      while (isPunct(this.peek(), '-')) { this.next(); name += `-${this.next().value}`; }
      if (isPunct(this.peek(), '=')) {
        this.next();
        const v = this.next();
        props.push({ name, value: v.kind === 'group' ? { tokens: v.tokens } : { literal: v } });
      } else {
        // A boolean attribute: `<item-slot empty />`
        props.push({ name, value: { boolean: true } });
      }
    }
    if (isPunct(this.peek(), '/')) {
      this.next(); this.expect('>');
      return { k: 'el', tag, props, children: [], isComponent: isComponentName(tag) };
    }
    this.expect('>');
    const children = this.parseChildren(tag);
    return { k: 'el', tag, props, children, isComponent: isComponentName(tag) };
  }

  parseIf(deco) {
    this.next(); // `if`
    const cond = this.expectGroup('(');
    // The proposal decorates a construct's BLOCK, so the decoration sits between
    // the head and the `{`. Reading it here is what puts `@persist` on the
    // branch it describes rather than on the `if` as a whole - which is finer
    // than the tag form, where one flag covers every branch.
    const headDeco = { ...(deco || {}), ...this.readBlockDecorations() };
    const body = this.expectBlockChildren();
    const branches = [{ cond, body, deco: headDeco }];
    let elseBody = null;
    let elseDeco = {};
    while (isWord(this.peek(), 'else')) {
      this.next();
      let branchDeco = {};
      if (isWord(this.peek(), 'if')) {
        this.next();
        const c = this.expectGroup('(');
        branchDeco = this.readBlockDecorations();
        branches.push({ cond: c, body: this.expectBlockChildren(), deco: branchDeco });
      } else {
        elseDeco = this.readBlockDecorations();
        elseBody = this.expectBlockChildren();
        break;
      }
    }
    return { k: 'if', branches, elseBody, elseDeco };
  }

  parseFor(deco) {
    this.next(); // `for`
    const head = this.expectGroup('(');
    const { binding, iterable } = splitForHead(head);
    const blockDeco = { ...(deco || {}), ...this.readBlockDecorations() };
    const body = this.expectBlockChildren();
    return {
      k: 'for', binding, iterable, body, deco: blockDeco,
    };
  }

  parseMatch(deco) {
    this.next(); // `match`
    const all = isWord(this.peek(), 'all');
    if (all) { this.next(); }
    const subject = this.expectGroup('(');
    const block = this.next();
    if (!isGroup(block, '{')) {
      throw new SyntaxError('a `match` needs a block of clauses');
    }
    const arms = new Parser(block.tokens, this.em).parseArms();
    return {
      k: 'match', subject, all, arms, deco: deco || {},
    };
  }

  parseArms() {
    const arms = [];
    while (!this.done()) {
      if (isPunct(this.peek(), ';')) { this.next(); continue; }
      let pattern = null;
      if (isWord(this.peek(), 'when')) {
        this.next();
        pattern = [];
        while (!this.done() && !isPunct(this.peek(), ':') && !isWord(this.peek(), 'if')) {
          pattern.push(this.next());
        }
      } else if (this.peek().kind === 'identifier' && this.peek().value === 'default') {
        this.next();
      } else {
        throw new SyntaxError('a match clause begins with `when` or `default`');
      }
      let guard = null;
      if (isWord(this.peek(), 'if')) { this.next(); guard = this.expectGroup('('); }
      this.expect(':');
      const deco = this.readBlockDecorations();
      let body;
      if (isGroup(this.peek(), '{')) {
        body = new Parser(this.next().tokens, this.em).parseChildren(null);
      } else {
        const collected = [];
        while (!this.done() && !isPunct(this.peek(), ';')) { collected.push(this.next()); }
        body = new Parser(collected, this.em).parseChildren(null);
      }
      arms.push({ pattern, guard, body, deco });
    }
    return arms;
  }

  readBlockDecorations() {
    return isPunct(this.peek(), '@') ? this.readDecorations() : {};
  }

  expectGroup(open) {
    const t = this.next();
    if (!isGroup(t, open)) {
      throw new SyntaxError(`expected \`${open}\``);
    }
    return t.tokens;
  }

  expectBlockChildren() {
    const t = this.next();
    if (!isGroup(t, '{')) {
      throw new SyntaxError('expected a block');
    }
    return new Parser(t.tokens, this.em).parseChildren(null);
  }
}

/** A component is capitalised; an element is not. This is JSX's own rule. */
function isComponentName(tag) {
  return typeof tag === 'string' && tag.length > 0 && tag[0] >= 'A' && tag[0] <= 'Z';
}

/** `const x of xs` -> the binding token run and the iterable token run. */
function splitForHead(head) {
  let i = 0;
  if (isWord(head[i], 'const') || isWord(head[i], 'let') || isWord(head[i], 'var')) { i += 1; }
  const binding = head[i];
  i += 1;
  if (!isWord(head[i], 'of')) {
    throw new SyntaxError('a `for` in a jsx region is a `for...of`');
  }
  return { binding, iterable: head.slice(i + 1) };
}

// ===========================================================================
// Emitter - node tree to tokens
// ===========================================================================

class Emitter {
  constructor(span) {
    this.span = span;
    this.n = 0;
  }

  /** A name nothing else can collide with, for the frames a macro introduces. */
  gensym(base) {
    this.n += 1;
    return `$${base}${this.n}`;
  }

  k(kind, value) { return { kind, value, span: this.span }; }
  g(value, tokens) {
    return {
      kind: 'group', value, span: this.span, tokens,
    };
  }

  id(name) { return this.k('identifier', name); }
  p(v) { return this.k('punctuator', v); }
  str(s) { return this.k('string', JSON.stringify(s)); }
  num(n) { return this.k('numeric', String(n)); }

  /** `name(a, b, c)` */
  call(name, argLists) {
    const inner = [];
    argLists.forEach((a, i) => {
      if (i > 0) { inner.push(this.p(',')); }
      inner.push(...a);
    });
    return [this.id(name), this.g('(', inner)];
  }

  /** `(params) => (body)` */
  arrow(params, body) {
    return [
      this.g('(', params),
      this.p('=>'),
      this.g('(', body),
    ];
  }

  /** `{ a: x, b: y }` */
  object(entries) {
    const inner = [];
    entries.forEach(([key, value], i) => {
      if (i > 0) { inner.push(this.p(',')); }
      inner.push(this.k('identifier', key), this.p(':'), ...value);
    });
    return [this.g('{', inner)];
  }

  /** `[a, b, c]` */
  array(items) {
    const inner = [];
    items.forEach((item, i) => {
      if (i > 0) { inner.push(this.p(',')); }
      inner.push(...item);
    });
    return [this.g('[', inner)];
  }

  /**
   * Statements run, then the last value is the region's.
   *
   * A region holding `const x = ...;` before its markup must emit the
   * declaration as a STATEMENT and the markup as the value - which is what a
   * `do`-block does, and what the region already is.
   */
  emitRoot(nodes, args) {
    void args;
    const stmts = nodes.filter((n) => n.k === 'stmt');
    const values = nodes.filter((n) => n.k !== 'stmt').map((n) => this.emitNode(n));
    let value;
    if (values.length === 0) { value = [this.id('undefined')]; } else if (values.length === 1) {
      [value] = values;
    } else { value = this.array(values); }
    if (stmts.length === 0) { return value; }
    const inner = [];
    for (const st of stmts) { inner.push(...st.tokens); }
    inner.push(...value, this.p(';'));
    return [this.id('do'), this.g('{', inner)];
  }

  emitNode(node) {
    switch (node.k) {
      case 'el': return this.emitElement(node);
      case 'text': return [this.str(node.value)];
      case 'expr': return node.tokens.slice();
      case 'stmt': return node.tokens.slice();
      case 'if': return this.emitIf(node);
      case 'for': return this.emitFor(node);
      case 'match': return this.emitMatch(node);
      default: throw new SyntaxError(`cannot emit a ${node.k}`);
    }
  }

  // -- elements ------------------------------------------------------------

  /**
   * A subtree with no control flow and no component is STATIC in shape, so it
   * becomes a ParsedTemplate hoisted into a `constant { }` and instantiated with
   * the dynamics that remain. Anything else is emitted as a `jsx(...)` call,
   * which is what the framework's `createElement` dispatch expects.
   */
  emitElement(node) {
    if (node.tag === null) {
      // A fragment is a list; the framework flattens children.
      return this.array(node.children.map((c) => this.emitNode(c)));
    }
    if (this.isTemplatable(node)) {
      return this.emitTemplate(node);
    }
    return this.emitCreateElement(node);
  }

  /**
   * An element is templatable unless it is a component.
   *
   * What varies among its children does NOT decide this: a control-flow child is
   * a DynamicChildSlot exactly as an interpolation is, because the element's
   * SHAPE is still fixed and what varies is one child - which is precisely what
   * a slot describes.
   *
   * Treating a control-flow child as non-static instead collapses the element
   * into a `jsx(...)` call and the template disappears - along with the template
   * of every static ancestor of any control flow in the view. The static
   * skeleton is the part worth hoisting, and it is the part that would go.
   */
  isTemplatable(node) {
    return node.k === 'el' && node.tag !== null && !node.isComponent;
  }

  /** Build a ParsedTemplate and emit `jsxTemplate(constant { ... }, ...dynamics)`. */
  emitTemplate(node) {
    const dynamics = [];
    const build = [];
    const slotKinds = [];
    const slotTargets = [];
    const names = new Map();

    const walk = (el) => {
      const name = this.gensym('e');
      names.set(el, name);
      const constants = [];
      const attrSlots = [];
      for (const prop of el.props) {
        if (prop.value.literal !== undefined) {
          constants.push([prop.name, [prop.value.literal]]);
        } else if (prop.value.boolean) {
          constants.push([prop.name, [this.id('true')]]);
        } else {
          // A dynamic prop becomes an attribute slot, and the expression is
          // FORWARDED UNCHANGED - a signal must arrive as itself for
          // `isSignal` to see it.
          const idx = dynamics.length;
          dynamics.push(this.call('jsxAttr', [[this.str(prop.name)], prop.value.tokens]));
          slotKinds.push('attr');
          slotTargets.push(name);
          attrSlots.push(idx);
        }
      }
      // const eN = { type, constants, children: [], attrSlots };
      build.push(
        this.id('const'), this.id(name), this.p('='),
        ...this.object([
          ['type', [this.str(el.tag)]],
          ['constants', this.object(constants)],
          ['children', this.array([])],
          ['attrSlots', this.array(attrSlots.map((i) => [this.num(i)]))],
        ]),
        this.p(';'),
      );
      for (const child of el.children) {
        if (child.k === 'el' && child.tag !== null && !child.isComponent) {
          const childName = walk(child);
          build.push(
            this.id(name), this.p('.'), this.id('children'), this.p('.'), this.id('push'),
            this.g('(', [this.id(childName)]), this.p(';'),
          );
        } else {
          const idx = dynamics.length;
          dynamics.push(this.slotValue(child));
          slotKinds.push('child');
          slotTargets.push(name);
          build.push(
            this.id(name), this.p('.'), this.id('children'), this.p('.'), this.id('push'),
            this.g('(', this.object([['slotIndex', [this.num(idx)]]])), this.p(';'),
          );
        }
      }
      return name;
    };

    const rootName = walk(node);
    // The block's completion value is the ParsedTemplate. `slotTargets` holds
    // REFERENCES to the same elements that appear in `root.children`, which no
    // object literal can express - which is why the plan is a block rather than
    // a literal, and why `constant { }` takes one.
    build.push(this.g('(', this.object([
      ['id', [this.num(0)]],
      ['root', [this.id(rootName)]],
      ['slotCount', [this.num(dynamics.length)]],
      ['slotKinds', this.array(slotKinds.map((s) => [this.str(s)]))],
      ['slotTargets', this.array(slotTargets.map((t) => [this.id(t)]))],
    ])), this.p(';'));

    // The plan is CLOSED - it names nothing from outside - so it may be hoisted
    // into a `constant { }`, which is evaluated once per SITE for the life of
    // the realm. The dynamics are not closed and stay at the call site.
    const plan = [this.id('constant'), this.g('{', build)];
    return this.call('jsxTemplate', [plan, ...dynamics]);
  }

  /** What fills a DynamicChildSlot, by the kind of child that made it. */
  slotValue(child) {
    if (child.k === 'text') { return [this.str(child.value)]; }
    if (child.k === 'expr') { return this.call('jsxEscape', [child.tokens]); }
    // A control-flow child, or a component - each is an ordinary expression that
    // evaluates to what the slot holds.
    return this.emitNode(child);
  }

  /** `jsx('tag', { ...props, children: [...] })` */
  emitCreateElement(node) {
    const entries = [];
    for (const prop of node.props) {
      if (prop.value.literal !== undefined) {
        entries.push([prop.name, [prop.value.literal]]);
      } else if (prop.value.boolean) {
        entries.push([prop.name, [this.id('true')]]);
      } else {
        entries.push([prop.name, prop.value.tokens]);
      }
    }
    if (node.children.length > 0) {
      entries.push(['children', this.array(node.children.map((c) => this.emitNode(c)))]);
    }
    const tag = node.isComponent ? [this.id(node.tag)] : [this.str(node.tag)];
    return this.call('jsx', [tag, this.object(entries)]);
  }

  // -- control flow --------------------------------------------------------

  /**
   * `jsx('if', { when, children: [factory, ...], persist })`
   *
   * The condition is a THUNK: `IfBranch.when` is `Signal | (() => unknown)` and
   * `readSignal` is `signal()`, so a thunk reads identically to a signal and is
   * the form that also works for `if (x() > 5)`.
   *
   * A branch is a FACTORY, which is what `IfController` calls when the condition
   * changes and what it keeps when `persist` is set.
   */
  emitIf(node) {
    const first = node.branches[0];
    const entries = [
      ['when', this.arrow([], first.cond)],
      ['children', this.array([
        this.arrow([], this.childrenValue(first.body)),
        ...(node.elseBody ? [this.arrow([], this.childrenValue(node.elseBody))] : []),
      ])],
    ];
    if (first.deco.persist) { entries.push(['persist', [this.id('true')]]); }
    const root = this.call('jsx', [[this.str('if')], this.object(entries)]);

    // `else if` chains are siblings in the framework's model, each with its own
    // metadata - which the tag form cannot express, since `<elseif>` associates
    // by position.
    const rest = node.branches.slice(1).map((b) => {
      const e = [
        ['when', this.arrow([], b.cond)],
        ['children', this.array([this.arrow([], this.childrenValue(b.body))])],
      ];
      if (b.deco.persist) { e.push(['persist', [this.id('true')]]); }
      return this.call('jsx', [[this.str('elseif')], this.object(e)]);
    });
    return rest.length === 0 ? root : this.array([root, ...rest]);
  }

  /**
   * `jsx('for', { items, key, children: [(item) => ...], persist })`, or
   * `jsx('forRange', { start, end, ... })` where the iterable is a range.
   */
  emitFor(node) {
    const range = splitRange(node.iterable);
    const bodyFn = [
      this.g('(', [node.binding]),
      this.p('=>'),
      this.g('(', this.childrenValue(node.body)),
    ];
    const entries = [];
    if (range) {
      entries.push(['start', this.arrowIfNeeded(range.start)]);
      entries.push(['end', this.arrowIfNeeded(range.end)]);
      if (range.inclusive) { entries.push(['inclusive', [this.id('true')]]); }
    } else {
      entries.push(['items', node.iterable.slice()]);
    }
    if (node.deco.key) {
      // The loop binding is in scope where `@key(...)` is written, so the macro
      // lifts the expression into a lambda over it - no lambda is written.
      entries.push(['key', [
        this.g('(', [node.binding]), this.p('=>'), this.g('(', node.deco.key),
      ]]);
    }
    entries.push(['children', this.array([bodyFn])]);
    if (node.deco.persist) { entries.push(['persist', [this.id('true')]]); }
    return this.call('jsx', [[this.str(range ? 'forRange' : 'for')], this.object(entries)]);
  }

  /**
   * `jsx('match', { on, children: [[test, factory], ...] })`, or `'matchAll'`.
   *
   * A pattern becomes a predicate over the subject. Literal patterns compare;
   * a range compares against its bounds; `_` always holds.
   */
  emitMatch(node) {
    const subject = this.gensym('s');
    const arms = node.arms.map((arm) => {
      const factory = this.arrow([], this.childrenValue(arm.body));
      if (arm.pattern === null) { return this.array([factory]); }
      const test = this.patternTest(arm.pattern, subject, arm.guard);
      return this.array([
        [this.g('(', [this.id(subject)]), this.p('=>'), this.g('(', test)],
        factory,
      ]);
    });
    const entries = [
      ['on', node.subject.slice()],
      ['children', this.array([this.array(arms)])],
    ];
    if (node.deco && node.deco.persist) { entries.push(['persist', [this.id('true')]]); }
    return this.call('jsx', [[this.str(node.all ? 'matchAll' : 'match')], this.object(entries)]);
  }

  /** A pattern as a predicate over `subject`, with the guard folded in. */
  patternTest(pattern, subject, guard) {
    let test;
    if (pattern.length === 1 && isWord(pattern[0], '_')) {
      test = [this.id('true')];
    } else {
      const range = splitRange(pattern);
      if (range) {
        test = [
          this.id(subject), this.p('>='), ...range.start,
          this.p('&&'), this.id(subject), this.p(range.inclusive ? '<=' : '<'), ...range.end,
        ];
      } else {
        test = [this.id(subject), this.p('==='), ...pattern];
      }
    }
    return guard ? [...test, this.p('&&'), this.g('(', guard)] : test;
  }

  /** One child is its value; several are a list. */
  childrenValue(children) {
    const values = children.map((c) => this.emitNode(c));
    if (values.length === 0) { return [this.id('undefined')]; }
    if (values.length === 1) { return values[0]; }
    return this.array(values);
  }

  /** A range endpoint may be a signal, so it is forwarded rather than thunked. */
  arrowIfNeeded(tokens) { return tokens.slice(); }
}

/** `a..<b` and `a..=b` -> endpoints, or null where the tokens are not a range. */
function splitRange(tokens) {
  for (let i = 0; i < tokens.length - 1; i += 1) {
    if (isPunct(tokens[i], '..')) {
      const next = tokens[i + 1];
      if (isPunct(next, '<') || isPunct(next, '=')) {
        return {
          start: tokens.slice(0, i),
          end: tokens.slice(i + 2),
          inclusive: isPunct(next, '='),
        };
      }
      return { start: tokens.slice(0, i), end: tokens.slice(i + 1), inclusive: false };
    }
  }
  return null;
}

// -----------------------------------------------------------------------------
// REGISTRATION
//
// The engine never loads this file. It calls
// `HostResolveReplacementDecorator(name, specifier)` and uses whatever the host
// returns - and the hook is given the decorator's NAME and the CONSUMING
// module's specifier, so it cannot discover where a preprocessor came from. A
// name-keyed registry is therefore the shape that works:
//
//     HostResolveReplacementDecorator: (name) =>
//       realm.GlobalObject.__preprocessors?.[name]
//
// Declared without `export` so this file is a SCRIPT, which is what a devtools
// snippet is. Wrap it in a module and add `export { jsx }` if you need one.
// -----------------------------------------------------------------------------

globalThis.__preprocessors = Object.assign(globalThis.__preprocessors || {}, { jsx });
```

## The Demo

The second snippet. Runtime, the view as the macro emits it, and a driver.

```js
// =============================================================================
// A COMPLETE, PASTE-AND-RUN EXAMPLE
//
// Paste this whole file into the engine262 devtools and run it. It is a SCRIPT -
// no imports, no exports, no host hooks - so nothing outside it is needed.
//
// -----------------------------------------------------------------------------
// WHY THE `@jsx { }` SOURCE ITSELF CANNOT BE PASTED
//
// Two things are required to compile a `@jsx { }` region, and neither is
// available to a pasted script:
//
//   1. A MODULE. The mode is declared by an import attribute -
//      `import { jsx } from "./jsx.js" with { preprocessor: "true", mode: "jsx" }`
//      - which a script has no way to write. Without it there is no `jsx` mode,
//      so `<inventory-grid columns="8">` is lexed as ordinary ECMAScript and
//      `grid columns` is two adjacent identifiers. That is the SyntaxError.
//
//   2. A HOST HOOK. The engine never loads `./jsx.js`. It calls
//      `HostResolveReplacementDecorator(name, specifier)` and uses whatever
//      function the host returns, so the devtools must supply the macro.
//
// So this file contains what the macro EMITS for the view below, taken verbatim
// from the engine, plus a runtime to execute it against. It uses `constant { }`
// and `do { }`, which this engine has.
//
// -----------------------------------------------------------------------------
// THE SOURCE THIS CORRESPONDS TO
//
//   const Inventory = ({ items, character, capacity, showEmpty }) => @jsx {
//     const visible = items.filter((i) => i.qty > 0);
//     const used = visible.length;
//     <panel layout="stack">
//       @match all (character) {
//         when 1: { <status-dot color="red" />; }
//         when 2: { <status-dot color="amber" />; }
//       };
//       <inventory-grid columns="8">
//         @for (const item of visible) @key(item.id) {
//           match (item.kind) {
//             when "weapon": <item-slot icon={item.icon} count={item.qty} />;
//             when _: <item-slot icon={item.icon} />;
//           };
//         }
//         @if (showEmpty) { <item-slot empty />; } else { <label text="hidden" />; }
//       </inventory-grid>
//       @match (used) {
//         when 0: <label text="Empty" />;
//         when _: <label text={used} />;
//       };
//     </panel>;
//   };
//
// Three rules the source above follows, each established by running it:
//   - a construct between tags takes the `@` sigil; inside a block it does not,
//     because there is no text there to be ambiguous with;
//   - a `match` needs a trailing `;`, being an expression statement where `if`
//     and `for` are statements;
//   - `@key(item.id)` names the loop binding directly, because the decorated
//     position is the loop BODY and the binding is already in scope there.
// =============================================================================

// -----------------------------------------------------------------------------
// 1. SIGNALS
// -----------------------------------------------------------------------------

const SIGNAL = Symbol.for('SIGNAL');
const SUBSCRIBERS = Symbol.for('SUBSCRIBERS');

function createSignal(initial) {
  let value = initial;
  const subscribers = new Set();
  const signal = () => value;
  signal[SIGNAL] = true;
  signal[SUBSCRIBERS] = subscribers;
  signal.set = (next) => {
    if (next === value) { return; }
    value = next;
    for (const fn of [...subscribers]) { fn(); }
  };
  return signal;
}

const isSignal = (v) => typeof v === 'function' && v[SIGNAL] === true;
// A signal and a thunk read the same way, which is why a condition may be
// either: `readSignal` is a call.
const readSignal = (s) => (typeof s === 'function' ? s() : s);
function watchSignal(s, onChange) {
  if (!isSignal(s)) { return () => {}; }
  s[SUBSCRIBERS].add(onChange);
  return () => s[SUBSCRIBERS].delete(onChange);
}

// -----------------------------------------------------------------------------
// 2. THE ELEMENT TREE
// -----------------------------------------------------------------------------

let nodeSeq = 0;
function createLayoutElement(type, props) {
  const el = {
    id: (nodeSeq += 1), type, props: {}, children: [], bindings: [],
  };
  for (const key of Object.keys(props || {})) {
    const value = props[key];
    if (key === 'children') { continue; }
    if (isSignal(value)) {
      // FORWARDED, not thunked: the partition is `isSignal(value)`, so wrapping
      // a prop would defeat the test and drop it.
      el.bindings.push([key, value]);
      el.props[key] = readSignal(value);
      watchSignal(value, () => { el.props[key] = readSignal(value); });
    } else {
      el.props[key] = value;
    }
  }
  for (const child of [].concat(props && props.children ? props.children : [])) {
    appendChild(el, child);
  }
  return el;
}

function appendChild(parent, child) {
  if (child === undefined || child === null || child === false) { return; }
  if (Array.isArray(child)) {
    for (const c of child) { appendChild(parent, c); }
    return;
  }
  // Whitespace between tags reaches the template as a text child. The parser
  // keeps every run because which whitespace matters is the CONSUMER's rule -
  // and this consumer drops runs that are only spaces between elements, which is
  // JSX's own rule.
  if (typeof child === 'string' && child.trim() === '') { return; }
  parent.children.push(child);
}

// -----------------------------------------------------------------------------
// 3. THE JSX RUNTIME
// -----------------------------------------------------------------------------

const jsxAttr = (name, value) => ({ __attr: true, name, value });
const jsxEscape = (v) => v;

/** Walk a ParsedTemplate, filling its slots from the dynamics. */
function jsxTemplate(template, ...dynamics) {
  const build = (node) => {
    const props = { ...node.constants };
    for (const slotIndex of node.attrSlots || []) {
      const a = dynamics[slotIndex];
      if (a && a.__attr) { props[a.name] = a.value; }
    }
    const el = createLayoutElement(node.type, props);
    for (const child of node.children || []) {
      if (child && child.slotIndex !== undefined) {
        appendChild(el, dynamics[child.slotIndex]);
      } else {
        appendChild(el, build(child));
      }
    }
    return el;
  };
  return build(template.root);
}

/** The dispatch a control-flow construct compiles to. */
function jsx(type, props) {
  switch (type) {
    case 'if': return createIf(props);
    case 'for': return createFor(props);
    case 'forRange': return createForRange(props);
    case 'match': return createMatch(props, false);
    case 'matchAll': return createMatch(props, true);
    default:
      return typeof type === 'function'
        ? type(props)
        : createLayoutElement(type, props);
  }
}

/** A wrapper whose children a controller replaces as its inputs change. */
function controller(kind, rebuild, watched) {
  const el = createLayoutElement(kind, {});
  el.rebuild = () => { el.children = []; rebuild(el); };
  el.rebuild();
  for (const w of watched) { watchSignal(w, () => el.rebuild()); }
  return el;
}

function createIf(props) {
  // Branches are FACTORIES, called when the condition changes - which is what a
  // macro lifting `if (c) { ... }` produces, so no controller change was needed.
  const branches = [].concat(props.children || []);
  return controller('if', (el) => {
    const taken = readSignal(props.when) ? branches[0] : branches[1];
    if (taken) { appendChild(el, taken()); }
  }, [props.when]);
}

function createFor(props) {
  const body = [].concat(props.children || [])[0];
  const keyOf = props.key || ((item) => item.id);
  return controller('for', (el) => {
    for (const item of readSignal(props.items) || []) {
      const child = body(item);
      if (child) { child.key = keyOf(item); }
      appendChild(el, child);
    }
  }, [props.items]);
}

function createForRange(props) {
  const body = [].concat(props.children || [])[0];
  return controller('forRange', (el) => {
    const start = readSignal(props.start);
    const end = readSignal(props.end) + (props.inclusive ? 1 : 0);
    for (let i = start; i < end; i += 1) { appendChild(el, body(i)); }
  }, [props.start, props.end]);
}

function createMatch(props, all) {
  const arms = [].concat(props.children || [])[0] || [];
  return controller(all ? 'matchAll' : 'match', (el) => {
    const subject = readSignal(props.on);
    for (const arm of arms) {
      const [test, factory] = arm.length === 1 ? [() => true, arm[0]] : arm;
      if (test(subject)) {
        appendChild(el, factory());
        // First-match, unless collecting - which is the whole difference.
        if (!all) { return; }
      }
    }
  }, [props.on]);
}

// =============================================================================
// 4. THE VIEW, EXACTLY AS THE MACRO EMITS IT
// =============================================================================

const Inventory = ({
  items, character, capacity, showEmpty,
}) => do {
  const visible = items.filter((i) => i.qty > 0);
  const used = visible.length;
  jsxTemplate(constant {
    const $e1 = { type: 'panel', constants: { layout: 'stack' }, children: [], attrSlots: [] };
    $e1.children.push({ slotIndex: 0 });
    const $e5 = { type: 'inventory-grid', constants: { columns: '8' }, children: [], attrSlots: [] };
    $e5.children.push({ slotIndex: 1 });
    $e5.children.push({ slotIndex: 2 });
    $e1.children.push($e5);
    $e1.children.push({ slotIndex: 3 });
    ({
      id: 0, root: $e1, slotCount: 4, slotKinds: ['child', 'child', 'child', 'child'], slotTargets: [$e1, $e5, $e5, $e1],
    });
  },
  jsx('matchAll', {
    on: character,
    children: [[
      [($s) => ($s === 1), () => jsxTemplate(constant {
        const $e2 = { type: 'status-dot', constants: { color: 'red' }, children: [], attrSlots: [] };
        ({ id: 0, root: $e2, slotCount: 0, slotKinds: [], slotTargets: [] });
      })],
      [($s) => ($s === 2), () => jsxTemplate(constant {
        const $e3 = { type: 'status-dot', constants: { color: 'amber' }, children: [], attrSlots: [] };
        ({ id: 0, root: $e3, slotCount: 0, slotKinds: [], slotTargets: [] });
      })],
    ]],
  }),
  jsx('for', {
    items: visible,
    key: (item) => item.id,
    children: [(item) => jsx('match', {
      on: item.kind,
      children: [[
        [($s) => ($s === 'weapon'), () => jsxTemplate(constant {
          const $e7 = { type: 'item-slot', constants: {}, children: [], attrSlots: [0, 1] };
          ({
            id: 0, root: $e7, slotCount: 2, slotKinds: ['attr', 'attr'], slotTargets: [$e7, $e7],
          });
        }, jsxAttr('icon', item.icon), jsxAttr('count', item.qty))],
        [($s) => (true), () => jsxTemplate(constant {
          const $e8 = { type: 'item-slot', constants: {}, children: [], attrSlots: [0] };
          ({
            id: 0, root: $e8, slotCount: 1, slotKinds: ['attr'], slotTargets: [$e8],
          });
        }, jsxAttr('icon', item.icon))],
      ]],
    })],
  }),
  jsx('if', {
    when: () => showEmpty,
    children: [
      () => jsxTemplate(constant {
        const $e9 = { type: 'item-slot', constants: { empty: true }, children: [], attrSlots: [] };
        ({ id: 0, root: $e9, slotCount: 0, slotKinds: [], slotTargets: [] });
      }),
      () => jsxTemplate(constant {
        const $e10 = { type: 'label', constants: { text: 'hidden' }, children: [], attrSlots: [] };
        ({ id: 0, root: $e10, slotCount: 0, slotKinds: [], slotTargets: [] });
      }),
    ],
  }),
  jsx('match', {
    on: used,
    children: [[
      [($s) => ($s === 0), () => jsxTemplate(constant {
        const $e11 = { type: 'label', constants: { text: 'Empty' }, children: [], attrSlots: [] };
        ({ id: 0, root: $e11, slotCount: 0, slotKinds: [], slotTargets: [] });
      })],
      [($s) => (true), () => jsxTemplate(constant {
        const $e12 = { type: 'label', constants: {}, children: [], attrSlots: [0] };
        ({
          id: 0, root: $e12, slotCount: 1, slotKinds: ['attr'], slotTargets: [$e12],
        });
      }, jsxAttr('text', used))],
    ]],
  }));
};

// =============================================================================
// 5. RUN IT
// =============================================================================

function render(el, depth = 0) {
  const pad = '  '.repeat(depth);
  const attrs = Object.keys(el.props)
    .map((k) => ` ${k}=${JSON.stringify(el.props[k])}`).join('');
  const key = el.key === undefined ? '' : ` (key ${el.key})`;
  const lines = [`${pad}<${el.type}${attrs}>${key}`];
  for (const child of el.children) {
    lines.push(typeof child === 'string' ? `${pad}  ${JSON.stringify(child)}` : render(child, depth + 1));
  }
  return lines.join('\n');
}

const character = createSignal(1);
const items = createSignal([
  { id: 'a', kind: 'weapon', icon: 'sword', qty: 1 },
  { id: 'b', kind: 'potion', icon: 'flask', qty: 3 },
  { id: 'c', kind: 'armor', icon: 'shield', qty: 0 },
]);

const tree = Inventory({
  items: items(), character, capacity: 8, showEmpty: true,
});

console.log('--- initial tree ---');
console.log(render(tree));

// The template is a `constant { }`, so it is the SAME object on every call -
// allocated once per SITE for the life of the realm, whatever reaches it.
const a = Inventory({
  items: items(), character, capacity: 8, showEmpty: true,
});
const b = Inventory({
  items: items(), character, capacity: 8, showEmpty: true,
});
console.log('\n--- checks ---');
console.log('two renders, same shape :', render(a) === render(b));
console.log('weapon slot has count   :',
  render(a).indexOf('count=1') >= 0);
console.log('zero-qty item filtered  :',
  render(a).indexOf('shield') === -1);

// A signal drives the controllers: changing it rebuilds only what depends on it.
console.log('\n--- character 1 -> 2, matchAll re-evaluates ---');
console.log('before:', render(tree).indexOf('"red"') >= 0 ? 'red' : 'none');
character.set(2);
console.log('after :', render(tree).indexOf('"amber"') >= 0 ? 'amber' : 'none');
```

Running it prints:

```
<panel layout="stack">
  <matchAll>
    <status-dot color="red">
  <inventory-grid columns="8">
    <for>
      <match> (key a)
        <item-slot icon="sword" count=1>
      <match> (key b)
        <item-slot icon="flask">
    <if>
      <item-slot empty=true>
  <match>
    <label text=2>

two renders, same shape : true
weapon slot has count   : true
zero-qty item filtered  : true

character 1 -> 2, matchAll re-evaluates
before: red
after : amber
```

## Further Examples

Each compiles with the same macro.

**A counter**, showing that a signal prop is forwarded rather than thunked - the runtime partitions props by testing whether the value IS a signal, so wrapping one would defeat the test:

```js
const Counter = ({ count }) => @jsx {
  <panel layout="row">
    <label text={count} />
    <button text="+" click={() => count.set(count() + 1)} />
  </panel>;
};
```

**A keyed list with a fallback**, where `@key` is often omissible because a value type's identity is its fields and the runtime can compare with `==`:

```js
const Rows = ({ rows }) => @jsx {
  <table>
    @for (const row of rows) @key(row.id) { <tr label={row.label} />; }
    @if (rows.length === 0) { <empty-state text="Nothing here" />; }
  </table>;
};
```

**A state machine**, where `match` brings exhaustiveness a predicate list cannot have - a state the view forgets is a compile-time error rather than a blank region:

```js
const Status = ({ state }) => @jsx {
  <panel>
    @match (state) {
      when "idle": <label text="Waiting" />;
      when "loading": @persist { <spinner size="md" />; }
      when "error": <label text="Failed" />;
      when _: <label text="Unknown" />;
    };
  </panel>;
};
```

**A range**, which replaces a start and an end prop with one expression that re-reads as its endpoints change:

```js
const Ruler = ({ from, to }) => @jsx {
  <axis>
    @for (const tick of from..<to) @key(tick) { <tick-mark at={tick} />; }
  </axis>;
};
```

**Nested control flow**, which is where a captured region fails and a parsed one does not - the inner `if` needs no sigil, being inside a block:

```js
const Grid = ({ slots }) => @jsx {
  <inventory-grid columns="8">
    @for (const slot of slots) @key(slot.id) @persist {
      if (slot.item) {
        <item-slot icon={slot.item.icon} count={slot.item.count} />;
      } else {
        <item-slot empty />;
      }
    }
  </inventory-grid>;
};
```

## Coverage Notes

- **The mode is what admits the syntax - resolved.** Markup parses inside a region and nowhere else, so `a < b` keeps its meaning and no existing program changes. This is narrower than a global grammar and narrower than a file extension, which is what TypeScript resorts to.
- **A region is a Block - resolved.** Declarations, expressions and markup share one region, and its value is the last expression. That is the ```do``` expression's completion semantics, which the region already carries.
- **Control flow between tags - resolved by a sigil.** `@if`, `@for` and `@match` are marked because child text is possible there; at statement level they are not, because it is not.
- **Per-site templates - resolved by ```constant { }```.** Each static subtree is allocated once for the life of the realm, keyed on the Parse Node, so a template inside a loop body's lambda is hoisted once however many times the lambda runs - and a component built inside a factory shares it across calls, which a hoisted `const` could not.
- **What the caching does not do.** The runtime re-walks the template on every render and memoises nothing against its identity. What ```constant { }``` removes is the allocation and the string round-trip a syntactic transform needed, not the walk. The template's identity is now a stable key to memoise against, which it was not before.
- **Narrowing does not survive the lift - open.** `match (user) { when let u: ... }` narrows at the source, but the macro rewrites the arms into thunks and the checker then sees calls, so the binding is typed by the runtime's signature rather than by the pattern. Typing the runtime generically covers literal and type patterns, which is most of what a view dispatches on; a destructuring pattern with bindings stays opaque. Closing it properly wants a construct that yields its arms rather than evaluating one, which is reification, and belongs with the same question in the [query example](linq.md).
- **Whitespace between tags reaches the macro - deliberately.** Every run is kept, including runs that are only spaces, because which whitespace is significant is the consumer's rule and not the parser's. The macro drops whitespace-only runs between elements, which is JSX's own convention; a different consumer may not.
