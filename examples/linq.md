# Query Comprehensions

A query language written as a decorated region: `from`, `where`, `orderby`, `group by` and `join` in the shape C# gave them, compiled by a replacement decorator into ordinary calls before the program is checked.

```js
import { linq } from "./linq.js" with { preprocessor: "true" };

const senior: Query.<string> = @linq {
  from p in people
  where p.age >= 18
  orderby p.surname, p.age descending
  select p.name
};
```

This is the case a [captured region](../decoratorreplacement.md) exists for, and a sharper one than JSX. JSX fails loudly: `<` cannot begin an expression, so a program containing it does not parse. A query fails *quietly in parts*. `from p` is two adjacent identifiers and an error, but `where p.age >= 18` is a valid expression statement, `orderby a, b` is a comma expression, and `x in xs` is a RelationalExpression that already means something else. A grammar admitting queries everywhere would not reject a malformed one; it would read it as something the author did not write.

Features exercised:

- A captured region, so `from`, `select` and `where` are ordinary identifiers everywhere outside one and no existing program changes meaning. The macro declares ```capture``` and reads the text itself; the engine provides no query grammar and needs none.
- ```stream.parse(start, end, "expression")```: a clause's operand IS ECMAScript, so the macro hands it back to the engine and gets tokens threaded from that parse - a template literal or a regular expression in a `where` arriving as one token rather than as the several a re-lex would give.
- ```constant { }``` for the query plan, so a comparer or a key list is built once per site rather than once per evaluation.
- An argumented decoration - ```@linq(sql) { ... }``` - selecting a provider, which is what lets one syntax serve an array and a database.
- [Higher-kinded types](../higherkindedtypes.md) carrying the element type through a pipeline and deciding which clauses a source admits.
- ```gensym``` for the frames a `let` or a `join` introduces, so a range variable named `p` cannot collide with the macro's own.
- Spans attributed back to the clause, so a type error in a desugared lambda reports against the query the developer wrote.

## The Syntax

Lowercase throughout. None of these are reserved words of ECMAScript, which is the mode's whole benefit.

```
Query        : `from` `await`? Binding `in` Expression QueryBody

QueryBody    : QueryClause* (SelectClause | GroupClause) Continuation?

QueryClause  : `from` `await`? Binding `in` Expression
             | `let` Binding `=` Expression
             | `where` Expression
             | `join` Binding `in` Expression `on` Expression `equals` Expression (`into` Binding)?
             | `orderby` Ordering (`,` Ordering)*
             | `index` Binding
             | `take` Expression | `skip` Expression
             | `takewhile` Expression | `skipwhile` Expression
             | `distinct` (`by` Expression)?

Ordering     : Expression (`ascending` | `descending`)? (`using` Expression)?

SelectClause : `select` Expression
GroupClause  : `group` Expression `by` Expression
Continuation : `into` Binding QueryBody
```

`Binding` is an ECMAScript binding pattern, so destructuring works where it would anywhere:

```js
@linq { from { name, age } in people where age >= 18 select name }
```

## Queries

Filter and project, which is most queries:

```js
const emails: Query.<string> = @linq { from u in users where u.active select u.email };
```

A query is a ```Query.<T>```, not an array. Deferral is visible in the type rather than hidden behind it: ```const emails: [].<string> = @linq { ... }``` is an error at the assignment, and the fix it asks for - ```.toArray()``` - is the one worth being told.

A second `from` is a cross join, and over a nested collection it is the flatten that method syntax spells worst:

```js
@linq { from o in orders from li in o.lines where li.qty > 1 select li.sku }
```

`let` is the clause that keeps a long query readable, and the one SQL and F# both lack and both work around:

```js
@linq {
  from f in files
  let kb = f.bytes / 1024
  where kb > 500
  select `${f.name}: ${kb}KB`
}
```

Ordering takes several keys with independent directions:

```js
@linq { from e in employees orderby e.dept, e.salary descending select e }
```

`into` continues a query past a grouping, which is how `HAVING` is written without a second keyword:

```js
@linq {
  from s in sales
  group s by s.region into g
  where g.count() > 10
  select { region: g.key, total: g.sum((x) => x.amount) }
}
```

A join, and a group join - the second is a left outer join, since every left row appears beside a possibly empty group:

```js
@linq { from o in orders join c in customers on o.customerId equals c.id select { o, c } }

@linq {
  from c in customers
  join o in orders on c.id equals o.customerId into theirs
  select { customer: c, orders: theirs }
}
```

Paging, a key-wise distinct, and the positional binding XQuery has as its `count` clause and C# has not at all:

```js
@linq { from p in posts orderby p.created descending skip 20 take 10 select p }
@linq { from v in visits distinct by v.userId select v.userId }
@linq { from line in lines index i select `${i + 1}: ${line}` }
```

An aggregate is a terminal on the query rather than a clause in it:

```js
const total: decimal = @linq { from li in cart select li.price * li.qty }.sum();
```

## The Source Protocol

A source declares what it can do, and a clause is legal where its source provides the operation. `W<_>` is the wrapper a source produces and ```W.<T>``` applies it, so one declaration serves the synchronous and asynchronous forms rather than two that drift:

```js
interface Source<W<_>> {
  map<T, U>(source: W.<T>, project: (value: T) => U): W.<U>;
  flatMap<T, U>(source: W.<T>, project: (value: T) => W.<U>): W.<U>;
}

interface Filterable<W<_>> extends Source.<W> {
  filter<T>(source: W.<T>, predicate: (value: T) => boolean): W.<T>;
}

interface Orderable<W<_>> extends Source.<W> {
  order<T, K>(source: W.<T>, plan: OrderPlan.<T, K>): W.<T>;
}

interface Groupable<W<_>> extends Source.<W> {
  group<T, K>(source: W.<T>, key: (value: T) => K): W.<Group.<K, T>>;
}

interface Sliceable<W<_>> extends Source.<W> {
  take<T>(source: W.<T>, count: uint32): W.<T>;
  skip<T>(source: W.<T>, count: uint32): W.<T>;
}
```

Each clause requires a capability:

| clause | requires |
| --- | --- |
| `from`, `select` | `Source` |
| a second `from` | `Source` - it is `flatMap` |
| `let`, `index` | `Source` - both are `map` |
| `where`, `takewhile`, `skipwhile` | `Filterable` |
| `orderby` | `Orderable` |
| `group by`, `distinct` | `Groupable` |
| `join` | `Groupable` - a join groups by the key and then looks up |
| `take`, `skip` | `Sliceable` |

A missing capability is an error at the **clause**, not at the source:

```js
@linq { from x in somePromise where x.ok select x }
//                            ^^^^^ Promise provides no `filter`;
//                                  `where` needs a Filterable source
```

That sentence is the reason for the protocol. Scala reaches the same fork through its for-comprehension and reports `value withFilter is not a member of scala.concurrent.Future`, which is true and says nothing about what to do.

The two shipped protocols are sequences and async sequences; `from await` selects the second, and a library may add a third without the language changing:

```js
const rows: AsyncQuery.<Row> = @linq {
  from await row in cursor
  where row.status === "open"
  take 100
  select row
};
```

## The Query Interface

```Query.<T>``` is iterable, so `for...of`, spread and destructuring work. What it is not is an array, and the terminals are named:

```js
interface Query<T> extends Iterable.<T> {
  toArray(): [].<T>;
  toSet(): Set.<T>;
  toMap<K, V>(key: (value: T) => K, value: (item: T) => V): Map.<K, V>;

  first(): T | undefined;
  last(): T | undefined;
  single(): T | undefined;

  count(): uint32;
  sum(select?: (value: T) => number): number;
  average(select?: (value: T) => number): number;
  min<K>(select?: (value: T) => K): T | undefined;
  max<K>(select?: (value: T) => K): T | undefined;
  fold<A>(seed: A, step: (accumulator: A, value: T) => A): A;

  any(predicate?: (value: T) => boolean): boolean;
  all(predicate: (value: T) => boolean): boolean;
  contains(value: T): boolean;

  union(other: Iterable.<T>): Query.<T>;
  intersect(other: Iterable.<T>): Query.<T>;
  except(other: Iterable.<T>): Query.<T>;
  zip<U, R>(other: Iterable.<U>, combine: (left: T, right: U) => R): Query.<R>;
  chunk(size: uint32): Query.<[].<T>>;
  window(size: uint32): Query.<[].<T>>;
}

interface Group<K, T> extends Query.<T> {
  key: K;
}
```

Re-iterating a ```Query.<T>``` re-runs the pipeline. Java Streams throw on reuse, which is safer and worse: it turns a performance surprise into a runtime error for code that reads correctly.

`orderby` and `group by` are **blocking**. They consume the whole source before yielding, so laziness does not survive them, and `take 10` before an `orderby` is a different query from `take 10` after one. Under a deferred design that is a correctness matter rather than a performance note, and the clause order is what says which query was meant.

## The Decorator

The macro is an ordinary preprocessor module. It receives the region's tokens, folds the clause list into a call chain, and returns tokens.

```js
function linq(tokens: TokenStream, context: Reflect.Region) {
  const clauses = parseQuery(tokens);
  return args === undefined
    ? emitCalls(clauses)
    : emitPlan(clauses, args);
}

// A query is not ECMAScript grammatically, so the region is CAPTURED and this
// macro reads its text. Without this the region would be parsed as a Block and
// refused at `from p`, which is two adjacent identifiers.
linq.capture = true;

export { linq };
```

The return carries NO annotation, and that is not an omission. A macro answers an
ARRAY of token records - the shape every macro in these documents builds - and
`TokenStream` is the nominal type of what it RECEIVES. Annotating the return
with it is refused, correctly: the two are different types, and one of them
does not have a name yet.

`parseQuery` scans the region's text for clause keywords and hands each operand back to the engine with ```tokens.parse(start, end, "expression")```. That is the only thing it cannot do itself: whether `/` in `where /^a/.test(x)` begins a regular expression or a division is not decidable lexically. Everything else - which words are clauses, what may follow each - is this macro's to decide, and a query dialect with different keywords is a different macro rather than a different engine.

`emitCalls` is a fold over the clause list, and each clause is a rewrite of the stream built so far:

```js
function emitCalls(clauses: [].<Clause>): TokenStream {
  let source = clauses[0].source;
  let frame = Frame.of(clauses[0].binding);

  for (const clause of clauses.slice(1)) {
    switch (clause.kind) {
      case ClauseKind.Where:
        source = call("_filter", source, lambda(frame, clause.predicate));
        break;
      case ClauseKind.Let:
        frame = frame.extend(clause.binding);
        source = call("_map", source, lambda(frame.previous, frame.build(clause.value)));
        break;
      case ClauseKind.OrderBy:
        source = call("_order", source, plan(clause.orderings, frame));
        break;
      case ClauseKind.Select:
        source = call("_map", source, lambda(frame, clause.projection));
        break;
      // ... one arm per clause kind
    }
  }
  return source;
}
```

### Transparent Identifiers

After a `let` or a `join`, two range variables are in scope and every later clause closes over both. The frame is an object and the lambdas destructure it:

```js
@linq {
  from p in people
  let full = p.first + " " + p.last
  where full.length < 30
  select full
}
```

compiles to:

```js
_map(
  _filter(
    _map(people, (p) => ({ p, full: p.first + " " + p.last })),
    ({ full }) => full.length < 30),
  ({ full }) => full)
```

The frame's field names come from ```gensym```, not from the source: a range variable called `p` must not collide with the frame the macro introduces, and only a fresh name is guaranteed not to. The destructuring patterns are generated to match.

### Plans as Constants

An `orderby`'s key selectors and directions are the same on every evaluation, so the plan is a ```constant { }``` - built once per site, identity-stable, and therefore usable as the key a runtime caches a compiled comparer against:

```js
_order(people, constant {
  [[(p) => p.surname, "asc"], [(p) => p.age, "desc"]];
})
```

A ```constant { }``` block must be closed: it reads nothing from outside itself. ```(p) => p.surname``` is closed; ```(p) => p.surname === target``` is not, and its plan stays at the call site. The macro hoists the closed part of a plan and leaves the rest, which is a decision it makes per key.

### Providers

A bare ```@linq { }``` emits calls. An argumented one emits a plan, and the provider translates it:

```js
import { linq, sql } from "./linq.js" with { preprocessor: "true" };

const q = @linq(sql) {
  from o in orders
  where o.total > 100
  orderby o.created descending
  take 20
  select { id: o.id, total: o.total }
};
```

compiles to:

```js
sql.compile(constant {
  ({ from: "orders",
     where: { op: ">", left: { field: "total" }, right: 100 },
     order: [{ field: "created", dir: "desc" }],
     take: 20,
     select: { id: { field: "id" }, total: { field: "total" } } });
})
```

Naming the provider is what makes an untranslatable clause a compile error at that clause. C# reaches the same fork through the source's static type, and its worst failure mode follows: a query that silently falls back to in-memory evaluation after fetching the whole table, because one clause could not be translated. A named provider cannot fall back without saying so.

### Where an Error Is Reported

A query desugars into frames nobody wrote. Without attribution a type error reports against the lambda the macro built:

```
Property 'surname' does not exist on type '{ p: Person, $f1: string }'
  at ({ p, $f1 }) => $f1.length < 30
```

which names a frame field the developer has never seen, in a lambda that is not in their file. Every emitted token carries the span of the clause it came from, so the same error reports against the query:

```
Property 'surname' does not exist on type 'Person'
  where full.length < 30
        ^^^^
```

## Coverage Notes

- **The mode is what makes the syntax possible - resolved.** Query keywords are contextual inside a region and ordinary identifiers outside one, so `const from = 1` keeps working and no existing program changes meaning. This is stronger than a global grammar would give: `x in xs` already parses as a RelationalExpression, so a query grammar admitted everywhere would silently misread rather than reject.
- **Clause operands are ordinary ECMAScript - resolved by delegation.** The macro hands each operand back through ```parse```, so a regular expression in a `where` is one token rather than a division, and a template literal is one token rather than a backtick and an identifier.
- **Deferral is in the type - resolved.** ```Query.<T>``` is not ```[].<T>```. C#'s deferral is famous as a surprise because `IEnumerable<T>` and a materialized list read alike; every language that made the distinction visible - Python's brackets, Kotlin's ```asSequence```, Elixir's `Stream`, Java's terminal operations - is not reported as a gotcha.
- **Which clauses a source admits - resolved by higher-kinded types.** ```Source<W<_>>``` and its refinements let a `Promise` source be a query source without being a filterable one, and the error lands on the `where` rather than on the source. Without higher kinds the protocol could not be written and the design would collapse to sequences only.
- **Custom comparers, deliberately narrow.** `using` exists on `Ordering` alone and always means ```(a: T, b: T) => number```. Two surveyed languages put a comparer in the comprehension itself - SQL's `COLLATE` and XQuery's `collation` - and both put it on ordering alone; none of the fourteen has one on grouping, joining or distinctness, because a key does that work and is typeable as ```(value: T) => K``` where a general equality relation is not.
- **Ordering by a type's own order - open.** ```orderby p``` on a type that defines its own comparison would need a `Comparable` protocol, which [operator overloading](../operatoroverloading.md) is the natural home for. This grammar composes with one: `using` becomes what you write when the type's order is not the one you want.
- **Aggregates inside a group - resolved by the interface, not the grammar.** ```g.sum((x) => x.amount)``` is a method on ```Group.<K, T>``` rather than a clause, which is why `into` is worth having: it is what puts a group in scope for an ordinary expression to consume.
