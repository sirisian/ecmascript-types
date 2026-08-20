# Higher-Kinded Types

A generic parameter normally stands for a type: ```Box<T>``` where ```T``` is ```uint8```. A **higher-kinded** parameter stands for something that *becomes* a type once applied — a generic declaration awaiting its own arguments. This extension adds them, so that one declaration can serve a family that differs only in a wrapper.

It builds on [generics](generics.md) and changes nothing about first-order parameters, which behave exactly as they did.

## The problem

The [iteration types](README.md) are six interfaces, and the asynchronous three resemble the synchronous three closely:

```js
interface Iterator<T, R = void, N = void> {
  next(value?: N): IteratorResult<T, R>;
}
interface AsyncIterator<T, R = void, N = void> {
  next(value?: N): Promise.<IteratorResult<T, R>, any>;
}
```

**They differ on two axes, and a higher-kinded parameter addresses one of them.** Saying so plainly is more useful than a motivating example that does not survive being checked:

1. `next`'s **return is wrapped** in a promise. This is the axis a kind removes, and the unified declaration below is what removes it.
2. The **member key** differs — ```[Symbol.iterator]``` against ```[Symbol.asyncIterator]```. A kind abstracts over the *type* a member has and never over the *key* it is stored under, so ```Iterable``` and ```AsyncIterable``` do not unify by this feature — and because ```IterableIterator``` extends one while ```AsyncIterableIterator``` extends the other, that pair does not either.

**So the family goes from six declarations to five: one merge, in the smallest of the three pairs.** That is the honest measure of what this buys, and a reader deciding whether the feature is worth its weight should have it rather than an estimate.

A third difference is often listed and is not one. The synchronous ```Iterator``` declares optional ```return``` and ```throw``` where the asynchronous one declares neither — but they are *optional*, so an interface declaring them is satisfied by a value with neither, and the asynchronous form's omission is a gap in its description rather than a difference in the protocol.

The larger deduplication here is not this feature's. Abstracting over the member key would collapse all six to two, and it needs a **value** generic rather than a kind — a symbol parameter, as ```interface Iterable<K: symbol, W<_>, T> { [K](): Iterator.<W, T>; }```. That form is unwritten and would cost the use site its readability, since ```Iterable.<Symbol.iterator, Identity, uint8>``` stands where ```Iterable.<uint8>``` did, and defaults cannot rescue it because the defaultable parameters come first. It is recorded here because a reader who sees ```Iterable``` left un-unified will ask why, and because it is the larger prize.

## The declaration

A parameter is higher-kinded when its own parameter list is written with holes:

```js
interface Iterator<W<_>, T, R = void, N = void> {
  next(value?: N): W.<IteratorResult<T, R>>;
}
```

The declaration above is the unification, and it is worth reading against the two it replaces: `next`, `return`, and `throw` are written once, with `W.<…>` where the synchronous form had a bare result and the asynchronous form a promise. `Iterator.<Identity, T>` is the first and `Iterator.<Promise, T>` the second.

```W<_>``` takes one argument. ```W<_, _>``` takes two. **Arity is written, never inferred**, so a declaration says how it will use its parameter and a reader need not scan the body to find out.

```_``` is the [pattern matching](patternmatching.md) wildcard, and the reuse is deliberate. This design already distinguishes two meanings of one token by position: ```%``` is the remainder operator between two operands and the [pipeline](pipelineoperator.md) topic where an operand is expected. A type parameter list is a position where a pattern can never appear, so ```_``` there is unambiguous, and it already means *a hole* everywhere else it is written.

Three other spellings were considered and rejected. ```<W<~>>``` is the tilde of TypeScript's own proposal, which reads as nothing in particular here. ```<W<*>>``` follows Scala, but ```*``` is an operator this design lets a class declare. ```<W<1>>``` writes the arity as a number, which is unambiguous and says *how many* where every other declaration site says *what shape*.

## Applying it

Inside the declaration, a higher-kinded parameter is applied like any other generic:

```js
next(value?: N): W.<IteratorResult<T, R>>;
```

At the use site, the argument is a generic declaration with its arguments left off:

```js
function drain(it: Iterator.<Identity, uint8>): [].<uint8> {}
function drainAsync(it: Iterator.<Promise, uint8>): Promise.<[].<uint8>, any> {}
```

A bare ```W``` — before it is applied — is **not a type**. It is a constructor of one, and it may only appear where this extension expects a constructor: as an argument to a higher-kinded parameter, or applied. Writing ```const x: W = …``` is an error, and the message says that ```W``` takes an argument.

## What may be supplied

Any generic declaration of matching arity: a class, an interface, or a type alias.

```js
type Identity<T> = T;

Iterator.<Identity, uint8>   // the synchronous form
Iterator.<Promise, uint8>    // the asynchronous form
Iterator.<uint8, uint8>      // TypeError: uint8 takes no type arguments; W expects one
Iterator.<Map, uint8>        // TypeError: Map takes two type arguments; W expects one
```

The two refusals carry different messages because they are different mistakes, and a reader who sees only "invalid type argument" has to work out which.

## ```Identity```

The unification needs a wrapper that means *no wrapper*, and it is an ordinary alias:

```js
type Identity<T> = T;
```

Nothing else is required. A generic alias may already be applied, so ```Identity.<uint8>``` is ```uint8``` and ```Iterator.<Identity, uint8>``` is the synchronous iterator. This proposal ships the alias in the standard library rather than making it a built-in type, because there is nothing built-in about it.

## Defaults, and where the parameter goes

A higher-kinded parameter may carry a default like any other, and doing so decides where it sits in the list. The wrapper is the *least* interesting parameter at most use sites — almost every annotation wants the synchronous form — so it goes **last** and defaults to ```Identity```:

```js
interface Iterator<T, R = void, N = void, W<_> = Identity> {
  next(value?: N): W.<IteratorResult<T, R>>;
  return?(value?: R): W.<IteratorResult<T, R>>;
  throw?(e?: any): W.<IteratorResult<T, R>>;
}
```

```Iterator.<uint8>``` therefore reads exactly as it did before this extension existed, which matters more than it sounds: the alternative is that every annotation naming an iteration type in the standard library grows a wrapper argument to say what it already said.

Putting it last is not a style preference — it is what the ordinary rule requires. A parameter carrying a default may not precede one that does not, so a leading ```W<_> = Identity``` followed by a required ```T``` would be ill-formed. **This extension adds no exception to that rule**, and a design needing one should be read as a sign the parameter is in the wrong position.

The asynchronous form is then a name rather than a second declaration:

```js
type AsyncIterator<T, R = void, N = void> = Iterator<T, R, N, Promise>;
type AsyncIterator<T, R = void, N = void> = Iterator.<T, R, N, Promise>;
```

which keeps ```AsyncIterator.<uint8>``` writable while the members it describes are declared once. The duplication this extension removes is of *content*, not of names — names are cheap, and a reader looking for ```AsyncIterator``` should find it.

## Constraints

A ```where``` clause on a higher-kinded parameter constrains its **applied form at a stated argument**:

```js
function collect<W<_>, T>(it: Iterator.<W, T>): [].<T>
  where W.<T> is Iterable.<T> {}
```

What may not be written is a constraint quantified over every argument — "any ```W``` such that ```W.<X>``` is ```Ordered<X>``` for every ```X```". That is refused, and the reason is not that it is hard.

Type-level evaluation in this design is **metered**: a budget of evaluation steps and constructed types bounds every type-position evaluation, and exhausting it fails compilation rather than hanging. A budget bounds a *computation* — it stops and reports. A quantified constraint is not a computation but a *search* over every type that could be supplied, and a budget that truncates a search yields a **wrong answer** rather than no answer: whether the constraint held would depend on how much budget was left when the search stopped. That turns a limit on time into a bug in meaning, which is why the refusal is a rule and not a caution.

## Variance

A higher-kinded parameter carries the same variance annotations a first-order one may, meaning the same thing at the applied form, and is **invariant** where it carries none — the same default and the same reasoning as [generics](generics.md).

```js
class Shape {} class Circle extends Shape {}
function draw(xs: Iterator.<Identity, Shape>): void {}
// draw(circles); // TypeError: Iterator.<Identity, Circle> is not Iterator.<Identity, Shape>
```

The wrapper parameter is invariant on the same terms: an ```Iterator.<Identity, T>``` is not an ```Iterator.<Promise, T>```, which is what keeps a synchronous iterator from being passed where an asynchronous one is expected.

## Inference is not attempted

A higher-kinded parameter is **always supplied explicitly**. It is never inferred from an argument, and this is a decision rather than an unimplemented case.

Inferring ```W``` and ```T``` together from one argument means searching for a decomposition — given a ```Promise.<uint8>```, deciding whether ```W``` is ```Promise``` and ```T``` is ```uint8```, or ```W``` is ```Identity``` and ```T``` is the whole promise. Both are consistent. Choosing needs a search, and the argument of the previous section applies unchanged: this design meters computation and cannot meter a search without making the answer depend on the budget.

Every language that infers higher-kinded parameters pays for it in checking time, and the languages that decline are declining this. Explicit application costs a reader one type argument at the call site and buys a checker that always terminates with the same answer.

## What this is not

This extension does not ship ```Functor```, ```Monad```, ```Applicative```, or ```Traversable```, and does not intend to. Those are the motivation wherever higher-kinded types are discussed in an erased language, and they are *possible* here now, but they are not why this exists. The reason is one duplication in one part of the standard library, and a reader arriving expecting category theory should know that up front.

It also does not add kinds of arity greater than the parameter lists that use them, higher-rank types, or constraints that quantify over a parameter's argument. Each is a coherent feature and none has a case in this design today.
