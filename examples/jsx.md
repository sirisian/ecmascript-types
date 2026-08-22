# Reactive Views

Markup and control flow written as a captured region, compiled by a replacement decorator into calls on a reactive runtime before the program is checked. The engine knows nothing about JSX: the macro reads its own syntax and delegates only the ranges that are ECMAScript.

```js
import jsx from "./jsx.js" with { preprocessor: "true" };

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

`<` cannot begin an expression, so a program containing markup does not parse at all - which is why a [captured region](../decoratorreplacement.md) is what admits it, rather than a global grammar change. Outside a region nothing moves: `a < b` is a comparison and a type argument list is a type argument list.

The whole of the mode is one extra production. `jsx` is ECMAScript with a JSX element admitted where a |PrimaryExpression| is expected - which is to say, exactly where a |RegularExpressionLiteral| may begin - so a region is an ordinary Block parsed with that production enabled. No second parser, and no scanner: the tokens come from the parse like every other token, so a template literal in a prop is one token rather than a backtick and an identifier that exists in no source.

Features exercised:

- A captured region, so markup is read inside one and nowhere else. The macro declares ```capture``` and scans the text itself - tags, attributes, children, the sigil - so a VARIANT of this syntax is a different macro rather than a different engine.
- Statements and markup in one region, because a region is a Block: a view may compute before it builds, and its value is the last expression.
- ```constant { }``` for each static subtree, so a template is built once per SITE for the life of the realm rather than once per render.
- Block decorators - ```@key(item.id)```, ```@persist``` - carrying the metadata a tag would have carried as attributes, in a position where the loop binding is already in scope.
- [`match` and `match all`](../patternmatching.md) as the view's dispatch, bringing range patterns, guards and exhaustiveness where a predicate list had none.
- [Ranges](../ranges.md) as iterables, so `for (const i of used..<capacity)` needs no start and end props.
- ```gensym``` for the frames the macro introduces, so a range variable cannot collide with one.

## Running It

The engine loads the preprocessor module itself. `sec-preprocessor-modules` says it is fetched and evaluated before the importing module is parsed, so it arrives through the ordinary module loader and its exports are the macros - the host needs no registry and no hook of its own.

1. **Create a devtools snippet named `jsx.js`** containing the macro below. It is a module, and `export default function jsx` is what the import binds — `sec-static-semantics-replacementdecoratornames` reads a default import's binding as well as a named one, and a preprocessor module providing a single macro is what a default export is for.
2. **Create a second snippet** containing the demo below, and run it. It prints the tree, checks that two renders share their templates, and changes a signal to show the controllers re-evaluating.

The import and the code using the macro must be in the SAME compilation unit. Expansion collects macro names from the parsed body's own imports and runs before evaluation, so a macro imported by one snippet cannot expand a decoration in another - and a dynamic `import()` cannot feed the expander at all, resolving as it does during evaluation, after expansion is over.

A region cannot be a bare SCRIPT: the mode is declared by an import attribute and only a module may carry one. Without it `<inventory-grid columns="8">` lexes as ECMAScript and `grid columns` is two adjacent identifiers, which is the error a paste produces. Run the demo as a module and the import does its work.

## Four Rules

Each was established by running the macro rather than by designing it.

**A construct between tags takes the `@` sigil; inside a block it does not.** Between tags a bare `if (` could be child text, and the parser would have to guess - so it is marked, which is where Angular 17 arrived with the same spelling. Inside a block there is no text to be ambiguous with, so `for (const s of xs) { if (c) { ... } }` needs no sigils.

**A `match` needs a trailing `;`.** It is an expression statement where `if` and `for` are statements, so without a terminator the parser reads on into the next tag and lexes its `/` as a regular expression.

**A decoration goes on the BLOCK.** `for (const slot of slots) @key(slot.id) { ... }` - the decorated position is the loop body, so the binding is in scope where the key is written and no lambda is needed. `@persist` lands per branch and per arm, which is finer than a tag's single flag.

**A macro cannot emit a call to its own name.** The runtime dispatch here is `jsxCreate`, not `jsx`, and the name is forced rather than chosen. `import { jsx } from "./jsx.js" with { preprocessor: "true" }` is an ordinary import that happens to carry an attribute: it binds `jsx` in module scope, that binding is the macro function, and [`sec-replacement-decorators`](../decoratorreplacement.md) makes it a Syntax Error for anything else in the module to bind the same name. So `jsx` inside this module is the macro, at run time as much as at parse time - emitting `jsx("matchAll", ...)` calls the MACRO with a string, and publishing a runtime `jsx` on `globalThis` does not help, the import binding shadowing the global like any other. React's automatic runtime imports its dispatch as `_jsx` for the same reason, and arrives at the same shape from the other direction.

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
  jsxCreate("matchAll", { on: character, children: [[...]] }),
  jsxCreate("for", { items: visible, key: (item) => item.id, children: [(item) => ...] }),
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
// jsx.js - a JSX macro that owns its grammar.
//
// The region is CAPTURED, so the engine parses none of it. This macro scans the
// text, decides every syntax question itself, and delegates exactly one thing:
// ranges that are ECMAScript go back to the engine through
// `stream.parse(start, end, goal)`, which answers tokens threaded from that
// parse. That is the only thing a macro cannot do for itself - whether `/`
// begins a regular expression or a division is not decidable lexically.
//
// Everything else here is a choice, not a rule. A variant wanting `{#if}`
// instead of `@if`, or a different interpolation delimiter, changes this file
// and nothing else.

export default function jsx(stream: TokenStream, context: Reflect.Region): [].<Token> {
  // The SOURCE text, not `String(stream)`. `toString` renders the TOKENS, so it
  // differs from the source by whatever is not a token - a comment, most
  // obviously - and `stream.parse(start, end)` indexes the source. Scanning the
  // rendering while delegating against the source is off by exactly those
  // characters, and the delegated range then starts mid-comment.
  const span = stream[0] ? stream[0].span : undefined;
  const text = span && span.source && typeof span.source.text === 'string'
    ? span.source.text
    : String(stream);
  const open = text.indexOf('{');
  const close = text.lastIndexOf('}');
  if (open < 0 || close < open) { throw new SyntaxError('a jsx region is a block'); }
  const out = new Out(stream, span, text);
  const nodes = new Scanner(text, open + 1, close, out).scanRegion();
  return out.emitRegion(nodes);
}

// ===========================================================================
// Scanner - text to nodes. Every syntax decision lives here.
// ===========================================================================

const CONSTRUCTS = ['if', 'for', 'match'];

class Scanner {
  constructor(text, from, to, out) {
    this.text = text;
    this.i = from;
    this.end = to;
    this.out = out;
  }

  /**
   * Whitespace AND comments.
   *
   * A region is source, so it has comments in it - and skipping only whitespace
   * left `// like this` at the head of a statement run, which was then delegated
   * to the engine starting mid-comment and refused. Comments are trivia to every
   * consumer here: a statement run must not begin with one, and a construct must
   * be findable past one.
   */
  ws() {
    for (;;) {
      while (this.i < this.end && /\s/.test(this.text[this.i])) { this.i += 1; }
      if (this.at('//')) {
        while (this.i < this.end && this.text[this.i] !== '\n') { this.i += 1; }
        continue;
      }
      if (this.at('/*')) {
        const close = this.text.indexOf('*/', this.i + 2);
        this.i = close < 0 ? this.end : close + 2;
        continue;
      }
      return;
    }
  }

  at(s) { return this.text.startsWith(s, this.i); }

  /** The region's top level: statements, with markup where a `<` begins one. */
  scanRegion() {
    const nodes = [];
    for (;;) {
      this.ws();
      if (this.i >= this.end) { return nodes; }
      if (this.at('<')) { nodes.push(this.element()); continue; }
      if (this.at('@') && this.construct(1)) { nodes.push(this.constructNode(1)); continue; }
      // A BARE construct, inside a block. No sigil is needed here - there is no
      // child text to be ambiguous with - and it must be recognised rather than
      // swallowed into a statement run: its arms and branches hold markup, which
      // the engine does not parse and would refuse.
      if (this.construct(0)) { nodes.push(this.constructNode(0)); continue; }
      // Ordinary JavaScript. Collected to the end of the statement and handed
      // BACK to the engine - the macro does not parse JS, it delegates it.
      const start = this.i;
      const stop = this.statementEnd();
      // A run that is only whitespace or a stray `;` is not a statement. Emitting
      // it anyway put an empty statement at the head of every region - harmless
      // and wrong, and it makes the output unreadable.
      if (this.text.slice(start, stop).replace(/[\s;]/g, '') !== '') {
        nodes.push({ k: 'js', from: start, to: stop });
      }
      this.i = stop === start ? stop + 1 : stop;
    }
  }

  /** A statement ends at its `;`, or where markup begins at brace depth 0. */
  statementEnd() {
    let depth = 0;
    let i = this.i;
    while (i < this.end) {
      const c = this.text[i];
      if (c === '"' || c === "'" || c === '`') { i = this.skipString(i); continue; }
      if (c === '(' || c === '[' || c === '{') { depth += 1; i += 1; continue; }
      if (c === ')' || c === ']' || c === '}') { depth -= 1; i += 1; continue; }
      if (c === '/' && (this.text[i + 1] === '/' || this.text[i + 1] === '*')) {
        // A comment inside a statement run is fine - it is the engine's to skip -
        // but one that STARTS a run is not, and `ws` has already handled that.
        i = this.text[i + 1] === '/'
          ? (this.text.indexOf('\n', i) < 0 ? this.end : this.text.indexOf('\n', i))
          : (this.text.indexOf('*/', i + 2) < 0 ? this.end : this.text.indexOf('*/', i + 2) + 2);
        continue;
      }
      if (depth === 0 && c === ';') { return i + 1; }
      if (depth === 0 && c === '<' && /[A-Za-z]/.test(this.text[i + 1] || '')) { return i; }
      // A sigil'd construct ends the run before it, or the whole construct is
      // swallowed into a statement and handed to the engine, which cannot parse
      // `@if` in that position.
      if (depth === 0 && c === '@' && CONSTRUCTS.some((w) => new RegExp(`^${w}\\b`).test(this.text.slice(i + 1)))) { return i; }
      i += 1;
    }
    return this.end;
  }

  skipString(i) {
    const quote = this.text[i];
    let j = i + 1;
    while (j < this.text.length) {
      if (this.text[j] === '\\') { j += 2; continue; }
      if (this.text[j] === quote) { return j + 1; }
      j += 1;
    }
    return this.text.length;
  }

  construct(offset) {
    const rest = this.text.slice(this.i + offset);
    return CONSTRUCTS.some((w) => new RegExp(`^${w}\\b`).test(rest));
  }

  // -- markup --------------------------------------------------------------

  element() {
    this.i += 1; // `<`
    if (this.at('>')) { this.i += 1; return { k: 'frag', children: this.children('') }; }
    const tag = this.name();
    const props = [];
    for (;;) {
      this.ws();
      if (this.i >= this.end || this.at('>') || this.at('/')) { break; }
      const name = this.name();
      this.ws();
      if (this.at('=')) {
        this.i += 1;
        this.ws();
        if (this.at('{')) {
          props.push({ name, expr: this.balanced('{', '}') });
        } else {
          const q = this.i;
          this.i = this.skipString(this.i);
          props.push({ name, literal: this.text.slice(q, this.i) });
        }
      } else {
        props.push({ name, boolean: true });
      }
    }
    if (this.at('/')) { this.i += 2; return { k: 'el', tag, props, children: [] }; }
    this.i += 1; // `>`
    return { k: 'el', tag, props, children: this.children(tag) };
  }

  name() {
    const start = this.i;
    while (this.i < this.end && /[-A-Za-z0-9_$.:]/.test(this.text[this.i])) { this.i += 1; }
    return this.text.slice(start, this.i);
  }

  children(tag) {
    const nodes = [];
    for (;;) {
      if (this.i >= this.end) { throw new SyntaxError(`unclosed <${tag}>`); }
      if (this.at('</')) {
        this.i += 2;
        const closing = tag === '' ? '' : this.name();
        if (closing !== tag) { throw new SyntaxError(`</${closing}> does not close <${tag}>`); }
        this.ws();
        this.i += 1; // `>`
        return nodes;
      }
      if (this.at('<')) { nodes.push(this.element()); continue; }
      if (this.at('{')) { nodes.push({ k: 'expr', range: this.balanced('{', '}') }); continue; }
      // The `@` sigil. A marker is needed BETWEEN TAGS because child text is
      // possible there and a bare `if (` could be either - which every framework
      // that faced this concluded. It is this macro's rule, not the language's.
      if (this.at('@') && this.construct(1)) { nodes.push(this.constructNode(1)); continue; }
      const start = this.i;
      while (this.i < this.end && !this.at('<') && !this.at('{')
        && !(this.at('@') && this.construct(1))) { this.i += 1; }
      if (this.i > start) { nodes.push({ k: 'text', value: this.text.slice(start, this.i) }); }
    }
  }

  /** `{ ... }` or `( ... )`, answering the range INSIDE the delimiters. */
  balanced(o, c) {
    const start = this.i;
    let depth = 0;
    while (this.i < this.text.length) {
      const ch = this.text[this.i];
      if (ch === '"' || ch === "'" || ch === '`') { this.i = this.skipString(this.i); continue; }
      if (ch === o) { depth += 1; }
      if (ch === c) {
        depth -= 1;
        if (depth === 0) { this.i += 1; return { from: start + 1, to: this.i - 1 }; }
      }
      this.i += 1;
    }
    throw new SyntaxError(`unbalanced ${o}`);
  }

  // -- constructs ----------------------------------------------------------

  constructNode(offset) {
    this.i += offset;
    const word = this.name();
    if (word === 'if') { return this.ifNode(); }
    if (word === 'for') { return this.forNode(); }
    return this.matchNode();
  }

  decorations() {
    const deco = {};
    for (;;) {
      this.ws();
      if (!this.at('@')) { return deco; }
      this.i += 1;
      const name = this.name();
      deco[name] = this.at('(') ? this.balanced('(', ')') : true;
    }
  }

  ifNode() {
    this.ws();
    const cond = this.balanced('(', ')');
    const deco = this.decorations();
    this.ws();
    const body = this.blockChildren();
    let alt = null;
    let altDeco = {};
    this.ws();
    if (/^else\b/.test(this.text.slice(this.i))) {
      this.i += 4;
      this.ws();
      if (/^if\b/.test(this.text.slice(this.i))) { this.i += 2; alt = [this.ifNode()]; } else {
        altDeco = this.decorations();
        this.ws();
        alt = this.blockChildren();
      }
    }
    return {
      k: 'if', cond, body, deco, alt, altDeco,
    };
  }

  forNode() {
    this.ws();
    const head = this.balanced('(', ')');
    const deco = this.decorations();
    this.ws();
    const body = this.blockChildren();
    return {
      k: 'for', head, body, deco,
    };
  }

  matchNode() {
    this.ws();
    const all = /^all\b/.test(this.text.slice(this.i));
    if (all) { this.i += 3; this.ws(); }
    const subject = this.balanced('(', ')');
    this.ws();
    const block = this.balanced('{', '}');
    const arms = new Scanner(this.text, block.from, block.to, this.out).scanArms();
    this.i = block.to + 1;
    this.ws();
    if (this.at(';')) { this.i += 1; }
    return { k: 'match', all, subject, arms };
  }

  blockChildren() {
    const block = this.balanced('{', '}');
    const inner = new Scanner(this.text, block.from, block.to, this.out);
    const nodes = inner.scanRegion();
    this.i = block.to + 1;
    return nodes;
  }

  scanArms() {
    const arms = [];
    for (;;) {
      this.ws();
      if (this.i >= this.end) { return arms; }
      if (this.at(';')) { this.i += 1; continue; }
      let pattern = null;
      if (/^when\b/.test(this.text.slice(this.i))) {
        this.i += 4;
        const start = this.i;
        while (this.i < this.end && !this.at(':')) { this.i += 1; }
        pattern = { from: start, to: this.i };
      } else if (/^default\b/.test(this.text.slice(this.i))) {
        this.i += 7;
      } else {
        throw new SyntaxError('a match clause begins with `when` or `default`');
      }
      this.i += 1; // `:`
      const deco = this.decorations();
      this.ws();
      // An arm's body is ONE node, and markup is the commonest. `statementEnd`
      // stops at a `<` because at the region's top level markup begins a new
      // thing - inside an arm the markup IS the body, so it must not.
      let body;
      if (this.at('{')) {
        body = this.blockChildren();
      } else if (this.at('<')) {
        body = [this.element()];
        this.ws();
        if (this.at(';')) { this.i += 1; }
      } else if (this.at('@') && this.construct(1)) {
        body = [this.constructNode(1)];
      } else {
        const start = this.i;
        const stop = this.statementEnd();
        body = start === stop ? [] : [{ k: 'js', from: start, to: stop }];
        this.i = stop === start ? stop + 1 : stop;
      }
      arms.push({ pattern, body, deco });
    }
  }
}

// ===========================================================================
// Out - nodes to tokens. Every ECMAScript range goes back to the engine.
// ===========================================================================

class Out {
  constructor(stream, span, text) {
    this.stream = stream;
    this.span = span;
    this.text = text;
    this.n = 0;
  }

  /**
   * The delegation. A range that is ECMAScript is handed to the engine, and what
   * comes back is threaded from THAT parse - so a regular expression in a prop
   * is one token, and a template literal is one token. Re-lexing the slice here
   * would give four tokens and a backtick, and no way to tell a regular
   * expression from a division.
   */
  parse(range, goal) {
    return Array.from(this.stream.parse(range.from, range.to, goal || 'expression'));
  }

  gensym(base) { this.n += 1; return `$${base}${this.n}`; }

  k(kind, value) { return { kind, value, span: this.span }; }
  g(value, tokens) { return { kind: 'group', value, span: this.span, tokens }; }
  id(n) { return this.k('identifier', n); }
  p(v) { return this.k('punctuator', v); }
  str(s) { return this.k('string', JSON.stringify(s)); }
  num(n) { return this.k('numeric', String(n)); }

  call(name, args) {
    const inner = [];
    args.forEach((a, i) => { if (i > 0) { inner.push(this.p(',')); } inner.push(...a); });
    return [this.id(name), this.g('(', inner)];
  }

  object(entries) {
    const inner = [];
    entries.forEach(([key, value], i) => {
      if (i > 0) { inner.push(this.p(',')); }
      inner.push(this.id(key), this.p(':'), ...value);
    });
    return [this.g('{', inner)];
  }

  array(items) {
    const inner = [];
    items.forEach((item, i) => { if (i > 0) { inner.push(this.p(',')); } inner.push(...item); });
    return [this.g('[', inner)];
  }

  arrow(params, body) { return [this.g('(', params), this.p('=>'), this.g('(', body)]; }

  /** Statements run, then the last value is the region's. */
  emitRegion(nodes) {
    const stmts = nodes.filter((n) => n.k === 'js');
    const values = nodes.filter((n) => n.k !== 'js').map((n) => this.emit(n));
    let value;
    if (values.length === 0) { value = [this.id('undefined')]; } else if (values.length === 1) {
      [value] = values;
    } else { value = this.array(values); }
    if (stmts.length === 0) { return value; }
    const inner = [];
    for (const s of stmts) { inner.push(...this.parse(s, 'statements')); }
    inner.push(...value, this.p(';'));
    return [this.id('do'), this.g('{', inner)];
  }

  emit(node) {
    switch (node.k) {
      case 'el': return this.element(node);
      case 'frag': return this.array(node.children.map((c) => this.emit(c)));
      case 'text': return [this.str(node.value)];
      case 'expr': return this.parse(node.range);
      case 'js': return this.parse(node, 'statements');
      case 'if': return this.ifCall(node);
      case 'for': return this.forCall(node);
      case 'match': return this.matchCall(node);
      default: throw new SyntaxError(`cannot emit ${node.k}`);
    }
  }

  templatable(node) { return node.k === 'el' && !/^[A-Z]/.test(node.tag); }

  /** A static subtree becomes a ParsedTemplate hoisted into a `constant { }`. */
  element(node) {
    if (!this.templatable(node)) { return this.createElement(node); }
    const dynamics = [];
    const build = [];
    const slotKinds = [];
    const slotTargets = [];
    const walk = (el) => {
      const name = this.gensym('e');
      const constants = [];
      const attrSlots = [];
      for (const prop of el.props) {
        if (prop.literal !== undefined) { constants.push([prop.name, [this.k('string', prop.literal)]]); } else if (prop.boolean) { constants.push([prop.name, [this.id('true')]]); } else {
          const idx = dynamics.length;
          dynamics.push(this.call('jsxAttr', [[this.str(prop.name)], this.parse(prop.expr)]));
          slotKinds.push('attr');
          slotTargets.push(name);
          attrSlots.push(idx);
        }
      }
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
        if (child.k === 'text' && child.value.trim() === '') { continue; }
        if (this.templatable(child)) {
          const childName = walk(child);
          build.push(this.id(name), this.p('.'), this.id('children'), this.p('.'), this.id('push'), this.g('(', [this.id(childName)]), this.p(';'));
        } else {
          const idx = dynamics.length;
          dynamics.push(this.emit(child));
          slotKinds.push('child');
          slotTargets.push(name);
          build.push(this.id(name), this.p('.'), this.id('children'), this.p('.'), this.id('push'), this.g('(', this.object([['slotIndex', [this.num(idx)]]])), this.p(';'));
        }
      }
      return name;
    };
    const root = walk(node);
    build.push(this.g('(', this.object([
      ['id', [this.num(0)]],
      ['root', [this.id(root)]],
      ['slotCount', [this.num(dynamics.length)]],
      ['slotKinds', this.array(slotKinds.map((s) => [this.str(s)]))],
      ['slotTargets', this.array(slotTargets.map((t) => [this.id(t)]))],
    ])), this.p(';'));
    return this.call('jsxTemplate', [[this.id('constant'), this.g('{', build)], ...dynamics]);
  }

  createElement(node) {
    const entries = [];
    for (const prop of node.props) {
      if (prop.literal !== undefined) { entries.push([prop.name, [this.k('string', prop.literal)]]); } else if (prop.boolean) { entries.push([prop.name, [this.id('true')]]); } else { entries.push([prop.name, this.parse(prop.expr)]); }
    }
    if (node.children.length > 0) {
      entries.push(['children', this.array(node.children.filter((c) => !(c.k === 'text' && c.value.trim() === '')).map((c) => this.emit(c)))]);
    }
    const tag = /^[A-Z]/.test(node.tag) ? [this.id(node.tag)] : [this.str(node.tag)];
    return this.call('jsxCreate', [tag, this.object(entries)]);
  }

  childrenValue(nodes) {
    const values = nodes.map((n) => this.emit(n));
    if (values.length === 0) { return [this.id('undefined')]; }
    if (values.length === 1) { return values[0]; }
    return this.array(values);
  }

  ifCall(node) {
    const entries = [['when', this.arrow([], this.parse(node.cond))]];
    const branches = [this.arrow([], this.childrenValue(node.body))];
    if (node.alt) { branches.push(this.arrow([], this.childrenValue(node.alt))); }
    entries.push(['children', this.array(branches)]);
    if (node.deco.persist) { entries.push(['persist', [this.id('true')]]); }
    return this.call('jsxCreate', [[this.str('if')], this.object(entries)]);
  }

  forCall(node) {
    // `const x of xs` - split by the macro, and the iterable delegated.
    const head = this.text.slice(node.head.from, node.head.to);
    const of = /\bof\b/.exec(head);
    if (!of) { throw new SyntaxError('a `for` in a jsx region is a `for...of`'); }
    const binding = head.slice(0, of.index).replace(/^\s*(const|let|var)\s+/, '').trim();
    const iterable = { from: node.head.from + of.index + 2, to: node.head.to };
    const entries = [['items', this.parse(iterable)]];
    if (node.deco.key) {
      entries.push(['key', [this.g('(', [this.id(binding)]), this.p('=>'), this.g('(', this.parse(node.deco.key))]]);
    }
    entries.push(['children', this.array([[this.g('(', [this.id(binding)]), this.p('=>'), this.g('(', this.childrenValue(node.body))]])]);
    if (node.deco.persist) { entries.push(['persist', [this.id('true')]]); }
    return this.call('jsxCreate', [[this.str('for')], this.object(entries)]);
  }

  matchCall(node) {
    const subject = this.gensym('s');
    const arms = node.arms.map((arm) => {
      const factory = this.arrow([], this.childrenValue(arm.body));
      if (arm.pattern === null) { return this.array([factory]); }
      // `when _:` is the wildcard, not a reference to a binding called `_`.
      // Emitting it as one produced `"_" is not defined` at RUN time - the arm
      // compiled, and failed when the controller tested it.
      if (this.text.slice(arm.pattern.from, arm.pattern.to).trim() === '_') {
        return this.array([[this.g('(', []), this.p('=>'), this.g('(', [this.id('true')])], factory]);
      }
      const pattern = this.parse(arm.pattern);
      return this.array([
        [this.g('(', [this.id(subject)]), this.p('=>'), this.g('(', [this.id(subject), this.p('==='), ...pattern])],
        factory,
      ]);
    });
    const entries = [
      ['on', this.parse(node.subject)],
      ['children', this.array([this.array(arms)])],
    ];
    return this.call('jsxCreate', [[this.str(node.all ? 'matchAll' : 'match')], this.object(entries)]);
  }
}

// `context: Reflect.Region` declares WHERE this macro applies: `@jsx class C {}`
// is refused at the decoration rather than failing inside the scanner. It is
// optional - a macro declaring no context works in any position - and this one
// only ever means anything on a region.
//
// The context carries `kind` and nothing else, because a replacement decorator
// receives the tokens of what it decorates: everything else a runtime context
// carries syntactically is already in them.

// The region is CAPTURED: its text is not ECMAScript, so the engine must not try
// to parse it. That is the whole of what the engine needs to be told - there is
// no grammar to name, because there is no grammar in the engine to name.
jsx.capture = true;
```

## The Demo

The second snippet, run as a MODULE. Runtime, the view as written, and a driver. The `@jsx { }` region is the real thing - the macro compiles it at parse time, which is what the static import above makes possible.

```js
// =============================================================================
// A COMPLETE, RUNNABLE EXAMPLE - the real `@jsx { }` source, not its expansion.
//
// Run the `jsx.js` snippet first, then this one as a MODULE.
//
// The import declares no grammar, and there is none to declare: the engine
// provides no lexical modes. Being a preprocessor decoration is what makes
// `@jsx { ... }` a region, and the MACRO says whether its text is ECMAScript -
// `jsx.js` sets `capture: true`, so it reads the region itself and hands the
// ranges that ARE ECMAScript back through `stream.parse`.
//
// The import must be STATIC, which is why this file is a module: the
// preprocessor import is found by a textual prescan before parsing, and a script
// has no way to write one. The engine loads the module named there through the
// ordinary loader and reads the macro out of its exports, so the specifier is
// whatever your devtools calls the jsx.js snippet and nothing else is needed.
// =============================================================================

import jsx from "./jsx.js" with { preprocessor: "true" };

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
function jsxCreate(type, props) {
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

// The macro emits bare `jsxCreate(...)`, `jsxTemplate(...)`, `jsxAttr(...)` and
// `jsxEscape(...)` calls, and each is a module-scope declaration above - so the
// emitted code resolves them the way it resolves any other name in the module,
// and nothing has to be published anywhere for it to work.
//
// The dispatch is NOT called `jsx`. It cannot be: see the fourth rule.

// =============================================================================
// 4. THE VIEW - the real thing, compiled by the macro at parse time
// =============================================================================

const Inventory = ({
  items, character, capacity, showEmpty,
}) => @jsx {
  // Ordinary JavaScript: a region is a Block, so a view may compute before it
  // builds, and the region's value is the last expression.
  const visible = items.filter((i) => i.qty > 0);
  const used = visible.length;

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
    @match (used) {
      when 0: <label text="Empty" />;
      when _: <label text={used} />;
    };
  </panel>;
};

// Three rules the view above follows, each established by running the macro:
//
//   - a construct BETWEEN TAGS takes the `@` sigil, because child text is
//     possible there and a bare `if (` could be either. Inside a block there is
//     no text to be ambiguous with, so the `match` in the loop body has none.
//   - a `match` needs a trailing `;`, being an expression statement where `if`
//     and `for` are statements.
//   - `@key(item.id)` names the loop binding directly: the decorated position is
//     the loop BODY, so the binding is already in scope where the key is written.

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
const items = [
  { id: 'a', kind: 'weapon', icon: 'sword', qty: 1 },
  { id: 'b', kind: 'potion', icon: 'flask', qty: 3 },
  { id: 'c', kind: 'armor', icon: 'shield', qty: 0 },
];

const tree = Inventory({
  items, character, capacity: 8, showEmpty: true,
});

console.log('--- initial tree ---');
console.log(render(tree));

// Each static subtree is a `constant { }`, allocated once per SITE for the life
// of the realm - so repeated calls build trees over the same templates.
const a = Inventory({ items, character, capacity: 8, showEmpty: true });
const b = Inventory({ items, character, capacity: 8, showEmpty: true });
console.log('');
console.log('--- checks ---');
console.log('two renders, same shape :', render(a) === render(b));
console.log('weapon slot has count   :', render(a).indexOf('count=1') >= 0);
console.log('zero-qty item filtered  :', render(a).indexOf('shield') === -1);

// A signal drives the controllers: changing it rebuilds only what reads it.
console.log('');
console.log('--- character 1 -> 2, matchAll re-evaluates ---');
console.log('before:', render(tree).indexOf('"red"') >= 0 ? 'red' : 'none');
character.set(2);
console.log('after :', render(tree).indexOf('"amber"') >= 0 ? 'amber' : 'none');
```

Verified end to end: the module compiles - which is what runs the macro over the real `@jsx { }` source - and its expansion executes to the tree below, including the signal change that makes the `matchAll` controller re-evaluate. Running it prints:

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
- **Four of this macro's bugs were found by RUNNING it, not by compiling it.** A `//` comment in a region was swallowed into a statement run and delegated mid-comment; a bare `match` inside a `@for` body was handed to the engine, whose arms hold markup it does not parse; `when _:` emitted `_` as a reference and failed at run time with `"_" is not defined`; and the macro scanned `String(stream)` while `parse` indexes the source, which is a silent misdelegation rather than an error. Compiling a view exercises none of them.
- **Whitespace between tags reaches the macro - deliberately.** Every run is kept, including runs that are only spaces, because which whitespace is significant is the consumer's rule and not the parser's. The macro drops whitespace-only runs between elements, which is JSX's own convention; a different consumer may not.
