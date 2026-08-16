# Decorators

Very WIP: Feel free to open issues with fixes and requirements for sections.

## Introduction

Types simplify how decorators are defined. By utilizing function overloading they get rid of the requirement to return a `(value, context)` function for decorators that take arguments. This means that independent of arguments a decorator looks the same. Modifying a decorator to take arguments or take none just requires changing the parameters to the decorator. Consider these decorators that are all distinct:

```js
function f(context: Reflect.ClassField) {
	// No parameters
}
function f(x: uint32, context: Reflect.ClassField) {
	// x is 0
}
function f(x: string, context: Reflect.ClassField) {
	// x is 'a'
}

class A {
	@f
	@f(0)
	@f('a')
	a: uint32;
}
```

A bare ```@f``` and an empty ```@f()``` are equivalent; both resolve through overload resolution with zero explicit arguments. **A REPLACEMENT decorator name binds one function and is not overloaded** — overload resolution needs types, and a replacement decorator runs before the module is checked. The overload taking only the context parameter is preferred when present. If no such overload exists and multiple signatures match through default values alone, the decoration is ambiguous and throws a TypeError at class definition time.

## Order

Two phases, and they run in opposite directions. This is the rule TC39's decorators already established, and following it means a reader who knows those knows these.

**Decorator expressions are evaluated in document order** — left to right, top to bottom, interleaved with computed property names. Whatever `@f(BASE + '/x')` computes, it computes at the position where it is written. **A replacement decorator's arguments are NOT evaluated**: it runs before the module is parsed, so there is nothing to evaluate against, and `@derive(Serialize)` passes the identifier token `Serialize`.

**Decorators are applied innermost first, and in reverse source order.** Concretely:

1. Several decorators on one declaration apply in reverse source order, so the one written closest to the declaration is applied first. `@a @b @c x` applies `c`, then `b`, then `a` — Python's `a(b(c(x)))`, and TC39's rule for the same shape. **A stack of REPLACEMENT decorators runs the other way, outer first**, and the two orders are one principle in two media: both say `a` is outside `b`, and a value must exist before it is wrapped where syntax must be rewritten before it is consumed. Replacement decorators are written OUTERMOST in a mixed stack, so a replacement encloses the runtime decorations and may rewrite or remove them with what it replaces.
2. A declaration's sub-targets apply before the declaration itself: parameter decorators in parameter order, then the return's, then the method's own. A method's decorator therefore sees a method whose parts are already decorated.
3. Members apply before their container, in document order, and the container's own decorators apply last. A class decorator sees a finished class, including whatever its fields' and methods' decorators did; an object decorator sees a finished object; an enum decorator sees decorated enumerators.
4. `addInitializer` callbacks run after every decorator of that declaration has been applied, in the order they were added.
5. A BLOCK decorator runs on every ENTRY to the block, not once at the declaration. **A replacement block decorator runs ONCE, at parse time**, and the cost this rule warns about below does not apply to it — it rewrites the block and can emit the instrumentation inline, so the per-iteration work happens where the per-iteration decorator does not. A block inside a loop is evaluated each iteration, so its decorator runs each iteration - which follows from the rule above ("a decorator runs when the declaration it decorates is evaluated") and is what makes a block decorator useful for instrumentation, tracing, and scoped resources. It is also the ONE per-evaluation position in this extension: every other decorator runs once per declaration, so a block decorator in a hot loop costs a context and a call per iteration, and that cost is not visible at the declaration site.

```js
function tag(name) { return (context) => log.push(name); }

@tag('class')
class A {
	@tag('field-outer')
	@tag('field-inner')
	a: uint8;

	@tag('method')
	m(@tag('param') p: uint8): @tag('return') uint8 { return p; }
}

// log is:
//   'field-inner', 'field-outer',   // reverse source order on one declaration
//   'param', 'return', 'method',    // sub-targets before their owner
//   'class'                         // the container last
```

The evaluation phase is separate and runs top to bottom, so in `@a(f()) @b(g()) x` the call `f()` happens before `g()` even though `b` is applied before `a`.

### When

A decorator runs when its declaration is evaluated: class definition time for a class and its members, function instantiation for a function, object literal evaluation for an object, enum definition time for an enum.

**A decorated function declaration does not hoist.** `@dec function f() {}` behaves as `var f = @dec function () {};` — the value is not available above its declaration. Hoisting it would mean either evaluating the decorator expressions before the bindings they reference exist, or evaluating them out of document order, and both break the rule above. This is the approach TC39's function decorators proposal reaches for the same reason.

Block, `let`, and `const` decorators are on the other timeline: they fire when the statement executes rather than when a declaration is evaluated. A block decorator on a loop body therefore fires once per iteration, which makes block decorators the only ones that can run more than once — deliberate, since a decorator that observes a block is observing an execution rather than a declaration.

Decorators can target almost anything and are defined by the following target contexts:

```
namespace Reflect {
	Class<T>
	ClassField<T, TClass>
	ClassAccessor<T, TClass>
	ClassGetter<T, TClass>
	ClassGetterReturn<T, TClass>
	ClassSetter<T, TClass>
	ClassSetterParameter<T, TClass>
	ClassMethod<T, TClass>
	ClassMethodParameter<T, TMethod, TClass>
	ClassMethodReturn<T, TMethod, TClass>
	ClassOperator<T, TClass>
	ClassOperatorParameter<T, TMethod, TClass>
	ClassOperatorReturn<T, TMethod, TClass>
	Function<T>
	FunctionParameter<T, TFunction>
	FunctionReturn<T, TFunction>
	Let<T>
	Const<T>
	Object<T>
	ObjectField<T, TObject>
	ObjectGetter<T, TObject>
	ObjectGetterReturn<T, TObject>
	ObjectSetter<T, TObject>
	ObjectSetterParameter<T, TObject>
	ObjectMethod<T, TObject>
	ObjectMethodParameter<T, TMethod, TObject>
	ObjectMethodReturn<T, TMethod, TObject>
	Block
	IfBlock
	ElseIfBlock
	ElseBlock
	WhileBlock
	DoWhileBlock
	ForBlock
	ForInBlock
	ForOfBlock
	Enum<T extends enum.<TValue>, TValue = int32>
	EnumEnumerator<T extends enum.<TValue>, TValue = int32>
	Tuple<T>
	Record<T>
}
```

Decorators can be specialized for different types by specifying the generic parameters. They can also be specialized for different targets.

```js
function f<TClass>(context: Reflect.ClassField.<uint32, TClass>) {
	console.log('decorator on uint32');
}
function f<TClass extends A>(context: Reflect.ClassField.<any, TClass>) {
	console.log('decorator on another type, extends A');
}
function f<TClass>(context: Reflect.ClassField.<any, TClass>) {
	console.log('decorator on another type');
}

class A {
	@f // decorator on uint32
	a: uint32

	@f // decorator on another type, extends A
	b: string
}

class B {
	@f // decorator on another type
	a: string
}
```

Note that because rest parameters are allowed to be duplicated and placed anywhere this means it's legal to write:

```js
function f(...x, context: Reflect.ClassField) {
	// [], [0, 1, 2], ['a', 'b', 'c']
}

class A {
	@f
	a: boolean

	@f(0, 1, 2)
	b: uint32

	@f('a', 'b', 'c')
	c: string
}
```

An example featuring all of them:
```js
@f // Reflect.Class
class A {
	@f // Reflect.ClassField, initial: 5
	a: uint32 = 5;
	@f // Reflect.ClassAccessor, initial: 5
	accessor b: uint32 = 5;
	@f
	#c: uint32 = 5;

	@f // Reflect.ClassGetter
	get c(): @f uint32 { // Reflect.ClassGetterReturn
	}

	@f // Reflect.ClassSetter
	set c(@f value: uint32) { // Reflect.ClassSetterParameter
	}

	@f // Reflect.ClassMethod
	d(@f a: uint32): @f uint32 { // Reflect.ClassMethodParameter, Reflect.ClassMethodReturn
	}

	@f // Reflect.ClassOperator
	operator+(@f rhs): @f uint32 { // Reflect.ClassOperatorParameter, then Reflect.ClassOperatorReturn
	}
}

@f // Reflect.Function
function g(@f a: uint32, @f b: string): @f uint32 { // Reflect.FunctionParameter, Reflect.FunctionReturn
}

@f // Reflect.Let, initial: 5
let a = 5;

@f // Reflect.Const
const b = @f { // Reflect.Object
	@f // Reflect.ObjectField
	a: 1,

	@f // Reflect.ObjectMethod
	b(@f x: uint32): @f uint32 { // Reflect.ObjectMethodParameter, Reflect.ObjectMethodReturn
		return x;
	},

	@f // Reflect.ObjectGetter
	get c(): @f uint32 { // Reflect.ObjectGetterReturn
		return 1;
	},

	@f // Reflect.ObjectSetter
	set c(@f value: uint32) { // Reflect.ObjectSetterParameter
	}
};

@f // Reflect.Enum
enum Count {
	@f // Reflect.EnumEnumerator
	Zero,
	One,
	Two
};

const e = @f Composite([0]); // Reflect.Tuple

const d = @f Composite({ a: 1 }); // Reflect.Record

@f // Reflect.Block
{
	const x = 10;
}

if (true) @f { // Reflect.IfBlock

} else if (false) @f { // Reflect.ElseIfBlock

} else @f { // Reflect.ElseBlock

}

while (true) @f { // Reflect.WhileBlock
	break;
}

do @f { // Reflect.DoWhileBlock
	break;
} while (true);

for (let i = 0; i < 10; ++i) @f { // Reflect.ForBlock
}

for (const key in obj) @f { // Reflect.ForInBlock
}

for (const item of items) @f { // Reflect.ForOfBlock
}
```

## Replacement

Decorators can optionally return a replacement for the decorated target. If a decorator returns `void` (or `undefined`), no replacement occurs. If it returns a value, that value replaces the original target. The return type must be compatible with the original.

This is the **value replacement** table. A *replacement decorator* returns SYNTAX rather than a value and is governed by a second table, below - the two are different axes, and a position may appear in both.

| Context | Return replaces | Return type |
|---|---|---|
| `Reflect.Class.<T>` | The class itself | `T` (the class or a subclass: a class type is nominal, so a structurally identical class of another declaration is not a `T`) |
| `Reflect.ClassField.<T, TClass>` | The field's initial value | `T` |
| `Reflect.ClassAccessor.<T, TClass>` | The accessor's get/set pair | `{ get(): T, set(value: T): void }` |
| `Reflect.ClassGetter.<T, TClass>` | The getter function | `() => T` |
| `Reflect.ClassSetter.<T, TClass>` | The setter function | `(value: T) => void` |
| `Reflect.ClassMethod.<T, TClass>` | The method | `T` (same signature) |
| `Reflect.ClassOperator.<T, TClass>` | The operator function | `T` (same signature) |
| `Reflect.Function.<T>` | The function | `T` (same signature) |
| `Reflect.ObjectGetter.<T, TObject>` | The getter function | `() => T` |
| `Reflect.ObjectSetter.<T, TObject>` | The setter function | `(value: T) => void` |
| `Reflect.ObjectMethod.<T, TObject>` | The method | `T` (same signature) |
| `Reflect.DoBlock.<T>` | The [`do` expression's](doexpressions.md) value | `T` |
| `Reflect.DoGeneratorBlock.<Y, R, N>` | The `do *` expression's generator | `Generator.<Y, R, N>` |

Decorators that describe sub-targets (parameters, returns) or the remaining structural positions (blocks other than a `do`'s, enums, tuples, records, let, const) do not support return replacement. For a block that was never about its being structural: a block produces nothing, so there was nothing to replace. A `do` block produces a value and a `do *` block produces a generator, which is why those two rows exist and the others do not.

### Syntax replacement

A **replacement decorator** - one imported with `with { preprocessor: true }`, see [decoratorreplacement.md](decoratorreplacement.md) - returns a `TokenStream` that replaces the SYNTAX it decorates, before the module is checked. That is a different axis from the table above, and the reasoning that excludes blocks there does not carry: a block produces no value, but it has syntax.

Every decorable position can be syntax-replaced, including every position the value table excludes. The constraint is grammatical rather than typed:

Each row's context reports its own name as `kind`, the same string the reflection
carries, so a replacement decorator dispatches on the position exactly as a
runtime one does. `Reflect.Region` is the one context that is not a reflection
above: a captured region is a position this table has and the reflection list
does not.

| Context | `kind` | Replacement must parse as | Also value-replaceable |
|---|---|---|---|
| `Reflect.Region` | `'Region'` | any statement list, since the region is replaced whole | no |
| `Reflect.Class` | `'Class'` | a class declaration or expression, as the position was | yes |
| `Reflect.ClassField` | `'ClassField'` | a class field definition | yes |
| `Reflect.ClassMethod` | `'ClassMethod'` | a method definition | yes |
| `Reflect.ClassAccessor` | `'ClassAccessor'` | an `accessor` field definition | yes |
| `Reflect.ClassGetter` / `Reflect.ClassSetter` | `'ClassGetter'` / `'ClassSetter'` | a getter / setter definition | yes |
| `Reflect.ClassOperator` | `'ClassOperator'` | an operator definition | yes |
| `Reflect.Function` | `'Function'` | a function declaration or expression, as the position was | yes |
| `Reflect.ObjectMethod` / `ObjectGetter` / `ObjectSetter` | the matching name | the corresponding object member | yes |
| `Reflect.ClassMethodParameter` and the other parameter contexts | the matching name | a formal parameter | no |
| `Reflect.ClassMethodReturn` and the other return contexts | the matching name | a type annotation | no |
| `Reflect.Block` and the eleven other block contexts | the matching name | the statement form decorated | only `DoBlock` and `DoGeneratorBlock` |
| `Reflect.Enum` / `Reflect.Tuple` / `Reflect.Record` | the matching name | the corresponding declaration | no |
| `Reflect.Let` / `Reflect.Const` | `'Let'` / `'Const'` | a lexical declaration | no |

An EMPTY replacement needs no separate permission: it is legal exactly where an empty token stream parses, so `@cfg` removing a statement or a class member works and removing a parameter from the middle of a list does not.

A replacement decorator is called `(tokens, context, args)`. Its context carries
`kind` and nothing else: it receives the TOKENS of what it decorates, so a name,
`static`, a binding and a pattern are already in them, and a runtime decorator
needs them in its context only because it is handed no tokens. Typing that
parameter declares where the decorator applies, and is optional.

`DoBlock` and `DoGeneratorBlock` appear in both tables. What a return means at those positions is decided by WHICH KIND of decorator returned it, which the import that introduced its name settles.

## Reflection

The following `<Context>Reflection` types define the data that is returned when reflecting a specific target. When reflecting a `class` one can access the `name`, `type`, and `metadata`.

Every `type` field below holds a type object - interned by structural identity, the same value `Reflect.typeOf` returns for a value of that type. Those type objects are opaque to property access; to walk a type's own structure (a union's arms, an array's element and extent, an object type's properties, a function's overload signatures) reflect it with the `Reflect.Type` context defined at the end of this section. `Reflect.Type` is the one reflection target that is not also a decorator context - a bare type expression carries no decorator - so it appears in the reflection structures and `getReflection` signatures but not in the replacement, `addInitializer`, or decorator-context tables.

Every reflection below carries a `kind`, a string naming the context it came from - `'ClassField'`, `'FunctionParameter'`, and so on. A reader that is handed a reflection can dispatch on it rather than inferring which context it holds from which fields happen to be present, which matters most for a decorator whose parameter is a union of contexts.

The fields listed for a reflection are all of them, and all of them are present. There is no field an implementation may leave out, because the point of the facility is that a decorator reads a shape it can rely on: a `readonly` or an `offset` that some hosts have is one every reader has to feature-test for, and feature tests are how a portable API becomes a single-implementation one. Where an implementation wants to expose something of its own, it goes under `host`, a single reserved property that is absent unless the implementation puts something there. So every bare property of a reflection is one of these, and anything else is under one key a reader can see and ignore.

Reflecting the same thing twice gives two objects, not one. A reflection is a report about a declaration rather than the declaration itself, and it is an ordinary object: extensible, not frozen, and not interned the way a type object is. The `metadata` object is the exception, and is stable: reflecting a declaration twice reaches the same metadata object, which is what makes a decorator's write visible to a later reader and what lets a subclass's metadata inherit prototypically from its base's. Identity belongs to the thing that carries state, not to the report about it. Block contexts show why this could not be otherwise: a block decorator runs on every entry to its block, so its reflection is produced per evaluation and there is nothing to intern it against.

A field written with a `?` below is present and *undefined* where it does not apply, not absent. A `for` with no update clause reports `update` as *undefined*, and a block with no label reports `label` as *undefined*; the `?` says the value may be undefined rather than that the property may be missing. A reader therefore walks one shape per context rather than testing which form of a construct it was handed, which is the same reason `offset` and `byteLength` are *undefined* on a field that has no laid-out placement rather than absent from it.

`metadata` is present and empty where nothing has written to it. A declaration with no decorators anywhere still reflects with a metadata object, so a reader never has to guard the read, and a missing key is a missing key rather than a TypeError on the ordinary case. Contexts whose family carries no metadata at all - bindings, blocks, tuples, and records - have no `metadata` field, which is a different thing from an empty one.

`initial` captures CONSTANT values only: a non-constant initializer reports *undefined*, because evaluating it would run user code at class definition rather than per call. `initializer` carries the same declaration as a `TokenStream`, so `x: uint32 = f()` is readable as what was written even though no value exists yet. The pair is a value and the expression that produced it, not two spellings of one thing - see [decoratorreplacement.md](decoratorreplacement.md).

### Class

```js
namespace Reflect {
	type ClassReflection = {
		name: string | undefined;
		type: Function;
		abstract: boolean;
		metadata: ClassMetadata;
	};

	type ClassFieldReflection<T = any> = {
		type: T;
		name: string | symbol;
		static: boolean;
		private: boolean;
		protected: boolean;
		readonly: boolean;
		// The DECLARED default: a typed field's zero value, or a constant initializer. A field's initializer runs per INSTANCE at construction while a field decorator fires at class definition, so there is no instance value to report here; `addInitializer` is what reaches one. (`inspect.Parameter.default` and `ParameterInfo.DefaultValue` report a declared default for the same reason.)
		initial: T | undefined;
		initializer: TokenStream | undefined;
		// Layout, present when the declaring class has one. A static field is not part of an instance's layout, so both are undefined for it. The full layout of a field - bit-level placement included - is `ClassFieldLayoutReflection` in memorylayout.md; these two are here because they are the two a decorator commonly wants.
		offset: int32 | undefined; // Signed bytes from the start of the instance; a negative offset overlaps a base
		byteLength: uint32 | undefined;
		metadata: ClassFieldMetadata;
	};

	type ClassAccessorReflection<T = any> = {
		type: T;
		name: string | symbol;
		static: boolean;
		private: boolean;
		protected: boolean;
		readonly: boolean;
		initial: T | undefined;
		initializer: TokenStream | undefined;
		// The pair this accessor generated, over its own backing field. A decorator that REPLACES the accessor returns a new `{ get, set }`; delegating to this one keeps the replacement over the SLOT the layout already allotted, instead of closing over storage of its own and leaving that slot dead. The slot exists either way: a layout is compile-time evaluable and must not depend on whether a decorator ran.
		access: { get(): T, set(value: T): void };
		metadata: ClassAccessorMetadata;
	};

	type ClassGetterReflection<T = any> = {
		type: () => T;
		name: string | symbol;
		static: boolean;
		private: boolean;
		protected: boolean;
		metadata: ClassGetterMetadata;
	};

	type ClassGetterReturnReflection<T = any> = {
		type: T;
		metadata: ClassGetterReturnMetadata;
	};

	type ClassSetterReflection<T = any> = {
		type: (value: T) => void;
		name: string | symbol;
		static: boolean;
		private: boolean;
		protected: boolean;
		metadata: ClassSetterMetadata;
	};

	// A setter takes exactly one parameter, so this reflection carries no `index`
	// where the other parameter reflections do - an index that is always 0 reports
	// nothing.
	type ClassSetterParameterReflection<T = any> = {
		type: T;
		name: string;
		initial: T | undefined;
		initializer: TokenStream | undefined;
		metadata: ClassSetterParameterMetadata;
	};

	type ClassMethodReflection<T extends (...args: [].<any>) => any = (...args: [].<any>) => any> = {
		type: T;
		name: string | symbol;
		static: boolean;
		private: boolean;
		protected: boolean;
		abstract: boolean; // True for an abstract method: a signature with no implementation
		signatures: [].<FunctionSignatureReflection>; // Length 1 when not overloaded
		metadata: ClassMethodMetadata;
	};

	type ClassMethodParameterReflection<T = any> = {
		type: T;
		name: string;
		index: uint32;
		initial: T | undefined;
		initializer: TokenStream | undefined;
		metadata: ClassMethodParameterMetadata;
	};

	type ClassMethodReturnReflection<T = any> = {
		type: T;
		metadata: ClassMethodReturnMetadata;
	};

	type ClassOperatorReflection<T = any> = {
		type: T;
		operator: Operator;
		static: boolean;
		signatures: [].<FunctionSignatureReflection>; // Length 1 when not overloaded; return-type overloads appear as multiple entries
		metadata: ClassOperatorMetadata;
	};

	type ClassOperatorParameterReflection<T = any> = {
		type: T;
		name: string;
		index: uint32;
		initial: T | undefined;
		initializer: TokenStream | undefined;
		metadata: ClassOperatorParameterMetadata;
	};

	type ClassOperatorReturnReflection<T = any> = {
		type: T;
		metadata: ClassOperatorReturnMetadata;
	};
}
```

### Function

```js
namespace Reflect {
	// One entry per overload signature. The neutral Function parameter and return
	// reflections carry each signature's shape; the context-specific flat accessors
	// (getReflection.<Reflect.FunctionParameter>, etc.) serve the single-signature case.
	type FunctionSignatureReflection = {
		parameters: [].<FunctionParameterReflection>;
		return: FunctionReturnReflection;
	};

	type FunctionReflection<T extends (...args: [].<any>) => any = (...args: [].<any>) => any> = {
		type: T;
		name: string | symbol | undefined;
		signatures: [].<FunctionSignatureReflection>; // Length 1 when not overloaded
		metadata: FunctionMetadata;
	};

	type FunctionParameterReflection<T = any> = {
		type: T;
		name: string;
		index: uint32;
		initial: T | undefined;
		initializer: TokenStream | undefined;
		metadata: FunctionParameterMetadata;
	};

	type FunctionReturnReflection<T = any> = {
		type: T;
		metadata: FunctionReturnMetadata;
	};
}
```

### Let / Const

```js
namespace Reflect {
	type LetReflection<T = any> = {
		type: T;
		name: string;
		initial: T | undefined;
		initializer: TokenStream | undefined;
	};

	type ConstReflection<T = any> = {
		type: T;
		name: string;
		initial: T;
		initializer: TokenStream | undefined;
	};
}
```

### Object

```js
namespace Reflect {
	type ObjectReflection<T = any> = {
		type: T;
		metadata: ObjectMetadata;
	};

	type ObjectFieldReflection<T = any> = {
		type: T;
		name: string | symbol;
		metadata: ObjectFieldMetadata;
	};

	type ObjectGetterReflection<T = any> = {
		type: () => T;
		name: string | symbol;
		metadata: ObjectGetterMetadata;
	};

	type ObjectGetterReturnReflection<T = any> = {
		type: T;
		metadata: ObjectGetterReturnMetadata;
	};

	type ObjectSetterReflection<T = any> = {
		type: (value: T) => void;
		name: string | symbol;
		metadata: ObjectSetterMetadata;
	};

	type ObjectSetterParameterReflection<T = any> = {
		type: T;
		name: string;
		initial: T | undefined;
		initializer: TokenStream | undefined;
		metadata: ObjectSetterParameterMetadata;
	};

	type ObjectMethodReflection<T extends (...args: [].<any>) => any = (...args: [].<any>) => any> = {
		type: T;
		name: string | symbol;
		signatures: [].<FunctionSignatureReflection>; // Length 1 when not overloaded
		metadata: ObjectMethodMetadata;
	};

	type ObjectMethodParameterReflection<T = any> = {
		type: T;
		// A parameter is named by an identifier, so `string` - as the other
		// parameter reflections have it. A `symbol` here would be a member name,
		// which a parameter does not have.
		name: string;
		index: uint32;
		initial: T | undefined;
		initializer: TokenStream | undefined;
		metadata: ObjectMethodParameterMetadata;
	};

	type ObjectMethodReturnReflection<T = any> = {
		type: T;
		metadata: ObjectMethodReturnMetadata;
	};
}
```

### Block

```js
namespace Reflect {
	type BlockReflection = {
		label?: string;
		block: TokenStream;
	};

	type IfBlockReflection = {
		label?: string;
		block: TokenStream;
		condition: TokenStream;
	};

	type ElseIfBlockReflection = {
		label?: string;
		block: TokenStream;
		condition: TokenStream;
	};

	type ElseBlockReflection = {
		label?: string;
		block: TokenStream;
	};

	type WhileBlockReflection = {
		label?: string;
		block: TokenStream;
		condition: TokenStream;
	};

	type DoWhileBlockReflection = {
		label?: string;
		block: TokenStream;
		condition: TokenStream;
	};

	type ForBlockReflection = {
		label?: string;
		block: TokenStream;
		initializer?: TokenStream;
		condition?: TokenStream;
		update?: TokenStream;
	};

	type ForInBlockReflection = {
		label?: string;
		block: TokenStream;
		binding: string | symbol;
	};

	type ForOfBlockReflection = {
		label?: string;
		block: TokenStream;
		binding: string | symbol;
	};

	type DoBlockReflection = {
		label?: string;
		block: TokenStream;
	};

	type DoGeneratorBlockReflection = {
		label?: string;
		block: TokenStream;
		async: boolean;   // `async do *` rather than `do *`
	};

	type MatchArmBlockReflection = {
		label?: string;
		block: TokenStream;
		subject: TokenStream;   // the match's argument
		pattern?: TokenStream;  // absent for a `default` clause
		guard?: TokenStream;    // absent where the clause is unguarded
		index: uint32;         // the clause's position among its siblings
	};
}
```

### Enum

```js
namespace Reflect {
	type EnumReflection<T extends enum.<TValue>, TValue = int32> = {
		type: T;
		name: string;
		valueType: TValue;
		size: uint32;
		metadata: EnumMetadata;
	};

	type EnumEnumeratorReflection<T extends enum.<TValue>, TValue = int32> = {
		name: string;
		value: TValue;
		index: uint32;
		metadata: EnumEnumeratorMetadata;
	};
}
```

### Tuple / Record

```js
namespace Reflect {
	type TupleReflection<T extends [].<any> = [].<any>> = {
		type: T;
	};

	type RecordReflection<T = any> = {
		type: T;
	};
}
```

### Type

`Reflect.Type` reflects a *type object* - as opposed to `Reflect.Tuple` and `Reflect.Record`, which reflect Composite *values*. Its reflection is discriminated by `kind` over the structural forms a type can take. Every `type`, `element`, and `arm` field is itself a type object, so a walker recurses by reflecting it again; a cycle terminates at a `reference` node, since every recursive cycle passes through a reference position.

```js
namespace Reflect {
	type TypeReflection =
		| { kind: 'primitive'; type: type; }                              // uint8, string, or a nominal class/enum reference
		| { kind: 'union'; arms: [].<type>; }
		| { kind: 'intersection'; members: [].<type>; }
		| { kind: 'tuple'; elements: [].<TypeTupleElement>; }
		| { kind: 'array'; element: type; extent: uint32 | undefined; }   // [].<T> => extent undefined; [N].<T> => N
		| { kind: 'object'; properties: [].<TypePropertyReflection>; }    // object-literal or interface type
		| { kind: 'function'; signatures: [].<FunctionSignatureReflection>; }
		| { kind: 'reference'; name: string; };                           // recursive back-edge

	type TypeTupleElement = {
		type: type;
		rest: boolean;                                                    // true for a spread position
	};

	type TypePropertyReflection = {
		name: string | symbol;
		type: type;
		optional: boolean;
	};
}
```

An `enum` or `class` type surfaces as a `primitive` node whose `type` is the nominal type; its members are reached through the existing `Reflect.Enum` and `Reflect.Class` contexts, so `Reflect.Type` does not duplicate enum or class member reflection. A `function` node's `signatures` is the same overload list the function reflection carries, so a bare function *type* and a reflected function *declaration* expose their overloads through one shape.

## `Reflect.getReflection` Signatures

The following `Reflect.getReflection` function is able to reflect any class, function, object, enum, record, or tuple feature.

### Class

```js
namespace Reflect {
	// Class-level
	getReflection<Reflect.Class, T>(): Reflect.ClassReflection;

	// Fields
	getReflection<Reflect.ClassField, T>(): { [name: string | symbol]: Reflect.ClassFieldReflection };
	getReflection<Reflect.ClassField, T>(name: string | symbol): Reflect.ClassFieldReflection;

	// Accessors
	getReflection<Reflect.ClassAccessor, T>(): { [name: string | symbol]: Reflect.ClassAccessorReflection };
	getReflection<Reflect.ClassAccessor, T>(name: string | symbol): Reflect.ClassAccessorReflection;

	// Methods
	getReflection<Reflect.ClassMethod, T>(): { [name: string | symbol]: Reflect.ClassMethodReflection };
	getReflection<Reflect.ClassMethod, T>(name: string | symbol): Reflect.ClassMethodReflection;

	// Method parameters
	getReflection<Reflect.ClassMethodParameter, T>(method: string | symbol): { [name: string | uint32]: Reflect.ClassMethodParameterReflection };
	getReflection<Reflect.ClassMethodParameter, T>(method: string | symbol, param: string | uint32): Reflect.ClassMethodParameterReflection;
	getReflectionByIndex<Reflect.ClassMethodParameter, T>(method: string | symbol): [].<Reflect.ClassMethodParameterReflection>;

	// Method return
	getReflection<Reflect.ClassMethodReturn, T>(method: string | symbol): Reflect.ClassMethodReturnReflection;
	getReflection<Reflect.ClassOperatorReturn, T>(operator: Operator): Reflect.ClassOperatorReturnReflection;

	// Getters
	getReflection<Reflect.ClassGetter, T>(): { [name: string | symbol]: Reflect.ClassGetterReflection };
	getReflection<Reflect.ClassGetter, T>(name: string | symbol): Reflect.ClassGetterReflection;

	// Getter return
	getReflection<Reflect.ClassGetterReturn, T>(getter: string | symbol): Reflect.ClassGetterReturnReflection;

	// Setters
	getReflection<Reflect.ClassSetter, T>(): { [name: string | symbol]: Reflect.ClassSetterReflection };
	getReflection<Reflect.ClassSetter, T>(name: string | symbol): Reflect.ClassSetterReflection;

	// Setter parameter
	getReflection<Reflect.ClassSetterParameter, T>(setter: string | symbol): Reflect.ClassSetterParameterReflection;

	// Operators
	getReflection<Reflect.ClassOperator, T>(): { [op: Operator]: Reflect.ClassOperatorReflection };
	getReflection<Reflect.ClassOperator, T>(op: Operator): Reflect.ClassOperatorReflection;

	// Operator parameters
	getReflection<Reflect.ClassOperatorParameter, T>(op: Operator): { [index: uint32]: Reflect.ClassOperatorParameterReflection };
	getReflection<Reflect.ClassOperatorParameter, T>(op: Operator, param: string | uint32): Reflect.ClassOperatorParameterReflection;
	getReflectionByIndex<Reflect.ClassOperatorParameter, T>(op: Operator): [].<Reflect.ClassOperatorParameterReflection>;
}
```

### Function

```js
namespace Reflect {
	getReflection<Reflect.Function, T>(): Reflect.FunctionReflection;

	getReflection<Reflect.FunctionParameter, T>(): { [name: string | uint32]: Reflect.FunctionParameterReflection };
	getReflection<Reflect.FunctionParameter, T>(param: string | uint32): Reflect.FunctionParameterReflection;
	getReflectionByIndex<Reflect.FunctionParameter, T>(): [].<Reflect.FunctionParameterReflection>;

	getReflection<Reflect.FunctionReturn, T>(): Reflect.FunctionReturnReflection;
}
```

### Let / Const

```js
namespace Reflect {
	getReflection<Reflect.Let, T>(): Reflect.LetReflection;
	getReflection<Reflect.Const, T>(): Reflect.ConstReflection;
}
```

### Object (instance-based)

```js
namespace Reflect {
	getReflection<Reflect.Object>(instance: any): Reflect.ObjectReflection;

	// Fields
	getReflection<Reflect.ObjectField>(instance: any): { [name: string | symbol]: Reflect.ObjectFieldReflection };
	getReflection<Reflect.ObjectField>(instance, name: string | symbol): Reflect.ObjectFieldReflection;

	// Methods
	getReflection<Reflect.ObjectMethod>(instance: any): { [name: string | symbol]: Reflect.ObjectMethodReflection };
	getReflection<Reflect.ObjectMethod>(instance, name: string | symbol): Reflect.ObjectMethodReflection;

	// Method parameters
	getReflection<Reflect.ObjectMethodParameter>(instance, method: string | symbol): { [name: string | uint32]: Reflect.ObjectMethodParameterReflection };
	getReflection<Reflect.ObjectMethodParameter>(instance, method: string | symbol, param: string | uint32): Reflect.ObjectMethodParameterReflection;
	getReflectionByIndex<Reflect.ObjectMethodParameter>(instance, method: string | symbol): [].<Reflect.ObjectMethodParameterReflection>;

	// Method return
	getReflection<Reflect.ObjectMethodReturn>(instance, method: string | symbol): Reflect.ObjectMethodReturnReflection;

	// Getters
	getReflection<Reflect.ObjectGetter>(instance: any): { [name: string | symbol]: Reflect.ObjectGetterReflection };
	getReflection<Reflect.ObjectGetter>(instance, name: string | symbol): Reflect.ObjectGetterReflection;

	// Getter return
	getReflection<Reflect.ObjectGetterReturn>(instance, getter: string | symbol): Reflect.ObjectGetterReturnReflection;

	// Setters
	getReflection<Reflect.ObjectSetter>(instance: any): { [name: string | symbol]: Reflect.ObjectSetterReflection };
	getReflection<Reflect.ObjectSetter>(instance, name: string | symbol): Reflect.ObjectSetterReflection;

	// Setter parameter
	getReflection<Reflect.ObjectSetterParameter>(instance, setter: string | symbol): Reflect.ObjectSetterParameterReflection;
}
```

### Block

Block reflection is not accessed via `getReflection` in normal user code. Block decorator contexts receive the reflection structure directly. These signatures are reserved for macro / compiler-plugin APIs.

```js
namespace Reflect {
	getReflection<Reflect.Block>(label: string): Reflect.BlockReflection;
	getReflection<Reflect.IfBlock>(label: string): Reflect.IfBlockReflection;
	getReflection<Reflect.ElseIfBlock>(label: string): Reflect.ElseIfBlockReflection;
	getReflection<Reflect.ElseBlock>(label: string): Reflect.ElseBlockReflection;
	getReflection<Reflect.WhileBlock>(label: string): Reflect.WhileBlockReflection;
	getReflection<Reflect.DoWhileBlock>(label: string): Reflect.DoWhileBlockReflection;
	getReflection<Reflect.ForBlock>(label: string): Reflect.ForBlockReflection;
	getReflection<Reflect.ForInBlock>(label: string): Reflect.ForInBlockReflection;
	getReflection<Reflect.ForOfBlock>(label: string): Reflect.ForOfBlockReflection;
	getReflection<Reflect.DoBlock>(label: string): Reflect.DoBlockReflection;
	getReflection<Reflect.DoGeneratorBlock>(label: string): Reflect.DoGeneratorBlockReflection;
	getReflection<Reflect.MatchArmBlock>(label: string): Reflect.MatchArmBlockReflection;
}
```

### Enum

```js
namespace Reflect {
	getReflection<Reflect.Enum, T>(): Reflect.EnumReflection.<T>;

	getReflection<Reflect.EnumEnumerator, T>(): { [name: string]: Reflect.EnumEnumeratorReflection };
	getReflection<Reflect.EnumEnumerator, T>(value: T): Reflect.EnumEnumeratorReflection;
	getReflectionByName<Reflect.EnumEnumerator, T>(name: string): Reflect.EnumEnumeratorReflection;
}
```

### Tuple / Record

```js
namespace Reflect {
	getReflection<Reflect.Tuple>(instance: any): Reflect.TupleReflection;
	getReflection<Reflect.Record>(instance: any): Reflect.RecordReflection;
}
```

### Type

`getReflection.<Reflect.Type>` takes a type object and returns its structural node. It is the retrieval verb for walking a type - a union, alias, tuple, array, or object type - that is not a class, function, or enum declaration.

```js
namespace Reflect {
	getReflection<Reflect.Type>(t: type): Reflect.TypeReflection;
}
```

When a function or method is overloaded, its parameters cannot be reached through the flat accessors above, because a parameter name or index does not identify which signature it belongs to. `getReflection.<Reflect.FunctionParameter, T>('x')` and its `ClassMethodParameter`, `ClassOperatorParameter`, and `ObjectMethodParameter` counterparts throw on an overloaded target; read `signatures[i].parameters` instead. This mirrors the call-site rule that an ambiguous overloaded reference is a TypeError, and the flat accessors remain defined for the single-signature case.

## `Reflect.getMetadata` Signatures

`Reflect.getMetadata` is sugar over `Reflect.getReflection`, returning only the `.metadata` field. Only targets whose reflection structures carry metadata have a corresponding `Reflect.getMetadata` overload.

### Class

```js
namespace Reflect {
	getMetadata<Reflect.Class, T>(): ClassMetadata;

	getMetadata<Reflect.ClassField, T>(): { [name: string | symbol]: ClassFieldMetadata };
	getMetadata<Reflect.ClassField, T>(name: string | symbol): ClassFieldMetadata;

	getMetadata<Reflect.ClassAccessor, T>(): { [name: string | symbol]: ClassAccessorMetadata };
	getMetadata<Reflect.ClassAccessor, T>(name: string | symbol): ClassAccessorMetadata;

	getMetadata<Reflect.ClassMethod, T>(): { [name: string | symbol]: ClassMethodMetadata };
	getMetadata<Reflect.ClassMethod, T>(name: string | symbol): ClassMethodMetadata;

	getMetadata<Reflect.ClassMethodParameter, T>(method: string | symbol): { [name: string | uint32]: ClassMethodParameterMetadata };
	getMetadata<Reflect.ClassMethodParameter, T>(method: string | symbol, param: string | uint32): ClassMethodParameterMetadata;
	getMetadataByIndex<Reflect.ClassMethodParameter, T>(method: string | symbol): [].<ClassMethodParameterMetadata>;

	getMetadata<Reflect.ClassMethodReturn, T>(method: string | symbol): ClassMethodReturnMetadata;

	getMetadata<Reflect.ClassGetter, T>(): { [name: string | symbol]: ClassGetterMetadata };
	getMetadata<Reflect.ClassGetter, T>(name: string | symbol): ClassGetterMetadata;

	getMetadata<Reflect.ClassGetterReturn, T>(getter: string | symbol): ClassGetterReturnMetadata;

	getMetadata<Reflect.ClassSetter, T>(): { [name: string | symbol]: ClassSetterMetadata };
	getMetadata<Reflect.ClassSetter, T>(name: string | symbol): ClassSetterMetadata;

	getMetadata<Reflect.ClassSetterParameter, T>(setter: string | symbol): ClassSetterParameterMetadata;

	getMetadata<Reflect.ClassOperator, T>(): { [op: Operator]: ClassOperatorMetadata };
	getMetadata<Reflect.ClassOperator, T>(op: Operator): ClassOperatorMetadata;

	getMetadata<Reflect.ClassOperatorParameter, T>(op: Operator): { [index: uint32]: ClassOperatorParameterMetadata };
	getMetadata<Reflect.ClassOperatorParameter, T>(op: Operator, param: string | uint32): ClassOperatorParameterMetadata;
	getMetadata<Reflect.ClassOperatorReturn, T>(op: Operator): ClassOperatorReturnMetadata;
	getMetadataByIndex<Reflect.ClassOperatorParameter, T>(op: Operator): [].<ClassOperatorParameterMetadata>;
}
```

### Function

```js
namespace Reflect {
	getMetadata<Reflect.Function, T>(): FunctionMetadata;

	getMetadata<Reflect.FunctionParameter, T>(): { [name: string | uint32]: FunctionParameterMetadata };
	getMetadata<Reflect.FunctionParameter, T>(param: string | uint32): FunctionParameterMetadata;
	getMetadataByIndex<Reflect.FunctionParameter, T>(): [].<FunctionParameterMetadata>;

	getMetadata<Reflect.FunctionReturn, T>(): FunctionReturnMetadata;
}
```

### Object (instance-based)

```js
namespace Reflect {
	getMetadata<Reflect.Object>(instance: any): ObjectMetadata;

	getMetadata<Reflect.ObjectField>(instance: any): { [name: string | symbol]: ObjectFieldMetadata };
	getMetadata<Reflect.ObjectField>(instance, name: string | symbol): ObjectFieldMetadata;

	getMetadata<Reflect.ObjectMethod>(instance: any): { [name: string | symbol]: ObjectMethodMetadata };
	getMetadata<Reflect.ObjectMethod>(instance, name: string | symbol): ObjectMethodMetadata;

	getMetadata<Reflect.ObjectMethodParameter>(instance, method: string | symbol): { [name: string | uint32]: ObjectMethodParameterMetadata };
	getMetadata<Reflect.ObjectMethodParameter>(instance, method: string | symbol, param: string | uint32): ObjectMethodParameterMetadata;
	getMetadataByIndex<Reflect.ObjectMethodParameter>(instance, method: string | symbol): [].<ObjectMethodParameterMetadata>;

	getMetadata<Reflect.ObjectMethodReturn>(instance, method: string | symbol): ObjectMethodReturnMetadata;

	getMetadata<Reflect.ObjectGetter>(instance: any): { [name: string | symbol]: ObjectGetterMetadata };
	getMetadata<Reflect.ObjectGetter>(instance, name: string | symbol): ObjectGetterMetadata;

	getMetadata<Reflect.ObjectGetterReturn>(instance, getter: string | symbol): ObjectGetterReturnMetadata;

	getMetadata<Reflect.ObjectSetter>(instance: any): { [name: string | symbol]: ObjectSetterMetadata };
	getMetadata<Reflect.ObjectSetter>(instance, name: string | symbol): ObjectSetterMetadata;

	getMetadata<Reflect.ObjectSetterParameter>(instance, setter: string | symbol): ObjectSetterParameterMetadata;
}
```

### Enum

```js
namespace Reflect {
	getMetadata<Reflect.Enum, T>(): EnumMetadata;

	getMetadata<Reflect.EnumEnumerator, T>(): { [name: string]: EnumEnumeratorMetadata };
	getMetadata<Reflect.EnumEnumerator, T>(value: T): EnumEnumeratorMetadata;
	getMetadataByName<Reflect.EnumEnumerator, T>(name: string): EnumEnumeratorMetadata;
}
```

No `getMetadata` overloads exist for `Reflect.Let`, `Reflect.Const`, `Reflect.Tuple`, `Reflect.Record`, or block contexts, as their reflection structures do not carry metadata.

## Metadata

Some contexts have metadata which is on the type (class constructor) and/or target. The same is true for enum types. For objects the metadata is on the instance.

All metadata uses a partial interface which can be appended with typed, symbol-keyed members. The metadata interfaces themselves are intrinsic: one exists per context whose reflection carries a `metadata` member (`ClassMetadata`, `ClassFieldMetadata`, and so on through `EnumEnumeratorMetadata`), each declaring nothing, so the members a program's partial declarations contribute are the only members there are. Metadata shapes are extended with `partial interface`, which the main proposal's class extension section specifies: an interface declares a shape and adds no instance state, so a partial one may contribute members where a partial class may not. An interface rather than a class because a metadata object must be an ordinary object - the inheritance rules below have a subclass's metadata inherit prototypically, and an instance of a class with a typed field is not extensible and so cannot be prototypically linked at all.

```js
const myMetadata = Symbol('myMetadata');
partial interface ClassMetadata {
	[myMetadata]: string;
};
```

Each decorator context has a reference to the target metadata:

```js
function f<T>({ metadata }: Reflect.Class.<T>) {
	metadata[myMetadata] = 'f';
}

@f
class A {}

const metadata = Reflect.getMetadata.<Reflect.Class, A>();
metadata[myMetadata]; // 'f'
```

This would need to also work on fields.

```js
const myMetadata = Symbol('myMetadata');
partial interface ClassFieldMetadata {
	[myMetadata]: string;
};

function f<T, TClass>({ metadata }: Reflect.ClassField.<T, TClass>) {
	metadata[myMetadata] = 'f';
}

class A {
	@f
	a: uint8;
}

const metadata = Reflect.getMetadata.<Reflect.ClassField, A>('a');
metadata[myMetadata]; // 'f'
```

### Metadata Inheritance

Consider `class B extends A {}`.

Reflection includes inherited members by default. `Reflect.getReflection.<Reflect.ClassField, B>()` returns all fields on B, including those inherited from its superclasses. The same applies to methods, getters, setters, and operators.

Each member's metadata is inherited through the prototype chain. If A declares a field with metadata, and B extends A without redeclaring that field, B inherits A's metadata as-is. Symbol key lookups fall through the prototype. If B redeclares the field and applies its own decorators, B gets a new metadata object (prototypically inheriting from A's) where B's decorators write their values, shadowing A's without mutating them.

```js
class A {
	@tag('base')
	id: uint64 = 0;
}

class B extends A {
	@tag('override')
	id: uint64 = 0; // Redeclares and redecorates
}

class C extends A {
	// Does not redeclare id
}
```

To query only the members a class declares itself, pass `{ own: true }`:
 
```js
Reflect.getReflection.<Reflect.ClassField, B>() // All fields including inherited
Reflect.getReflection.<Reflect.ClassField, B>({ own: true }) // Only fields B declares
```

An enumeration answers members of one staticness. `{ static: true }` asks for the static ones; with neither option the instance ones come back:

```js
class A { m() {} static m() {} }

Reflect.getReflection.<Reflect.ClassMethod, A>()                 // { constructor, m } - the instance m
Reflect.getReflection.<Reflect.ClassMethod, A>({ static: true }) // { m }              - the static one
```

The result is keyed by name, and a name does not identify a member: `m` and `static m` are both legal in one body and both reach the same owner. A program wanting both asks twice and merges by VALUE, each reflection carrying its own `name` and `static`:

```js
const all = [...Object.values(Reflect.getReflection.<Reflect.ClassMethod, A>()),
             ...Object.values(Reflect.getReflection.<Reflect.ClassMethod, A>({ static: true }))];

const broken = { ...instance, ...statics };   // keyed by name, so `m` collides again
```

The enumerated methods include `constructor` - a class always has one, written or not - so a program listing the methods it means to expose skips that name.

## addInitializer

Present on contexts that represent declaration sites where initialization logic can be injected:

| Has `addInitializer` | Does not |
|---|---|
| `Reflect.Class` | `Reflect.ClassMethodParameter` |
| `Reflect.ClassField` | `Reflect.ClassMethodReturn` |
| `Reflect.ClassAccessor` | `Reflect.ClassGetterReturn` |
| `Reflect.ClassGetter` | `Reflect.ClassSetterParameter` |
| `Reflect.ClassSetter` | `Reflect.ClassOperatorParameter` |
| `Reflect.ClassMethod` | `Reflect.Function` |
| `Reflect.ClassOperator` | `Reflect.FunctionParameter` |
| `Reflect.ObjectMethod` | `Reflect.FunctionReturn` |
| `Reflect.ObjectGetter` | `Reflect.Let` |
| `Reflect.ObjectSetter` | `Reflect.Const` |
| | `Reflect.ObjectField` |
| | `Reflect.ObjectMethodParameter` |
| | `Reflect.ObjectMethodReturn` |
| | `Reflect.ObjectGetterReturn` |
| | `Reflect.ObjectSetterParameter` |
| | `Reflect.Enum` / `Reflect.EnumEnumerator` |
| | `Reflect.Tuple` / `Reflect.Record` |
| | All block contexts |

## Decorator Contexts

Defining specialized target decorators is done by overloading a decorator's parameter types and context.

### Class
```js
namespace Reflect {
	interface Class<T extends { new (...args: [].<any>): any }> extends Reflect.ClassReflection {
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

#### Singleton

```js
const singletonKey = Symbol('singleton');

partial interface ClassMetadata {
	[singletonKey]?: { instance: any };
}

function singleton<T>({ metadata }: Reflect.Class.<T>): T {
	metadata[singletonKey] = { instance: undefined };
	let instance: T | undefined;
	return class extends T {
		constructor(...args: [].<any>) {
			if (instance) return instance;
			super(...args);
			instance = this;
			metadata[singletonKey].instance = this;
		}
	};
}

@singleton
class AppConfig {
	debug: boolean = false;
	version: string = '1.0.0';
}

const a = new AppConfig();
const b = new AppConfig();
```

#### Sealed

```js
function sealed<T>({ type }: Reflect.Class.<T>) {
	Object.seal(type);
	Object.seal(type.prototype);
}

@sealed
class Api {
	version: string = '1.0';
}
```
</details>

### ClassField
```js
namespace Reflect {
	interface ClassField<T, TClass> extends Reflect.ClassFieldReflection.<T> {
		classContext: Reflect.Class.<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>
	
```js
const metadataKey = Symbol('log');
function logField<T, TClass>({ classContext: { name: className, metadata: classMetadata }, name, type, static, private, metadata }: Reflect.ClassField.<T, TClass>) {
	console.log('name:', name);
	console.log('class:', className);
	console.log('type:', type);
	console.log('static:', static);
	console.log('private:', private);
	classMetadata[metadataKey] = name; // Just an example
	metadata[metadataKey] = name;
}
```
</details>

### ClassAccessor

```js
namespace Reflect {
	interface ClassAccessor<T, TClass> extends Reflect.ClassAccessorReflection.<T> {
		classContext: Reflect.Class.<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>
	
```js
import { Signal, Memo } from 'signals';

function signal<T, TClass>(
	{ name, initial }: Reflect.ClassAccessor.<T, TClass>,
): { get(): T, set(value: T): void } {
	const signals = new WeakMap.<object, Signal.<T>>();
	function getSignal(instance: object): Signal.<T> {
		let sig = signals.get(instance);
		if (!sig) {
			sig = new Signal.<T>(initial);
			signals.set(instance, sig);
		}
		return sig;
	}
	return {
		get(): T { return getSignal(this).get(); },
		set(value: T) { getSignal(this).set(value); },
	};
}

function memo<T, TClass>(
	{ type: originalGetter }: Reflect.ClassGetter.<T, TClass>,
): () => T {
	const memos = new WeakMap.<object, Memo.<T>>();
	return function(): T {
		let m = memos.get(this);
		if (!m) {
			m = new Memo(() => originalGetter.call(this));
			memos.set(this, m);
		}
		return m.get();
	};
}

class Sum {
	@signal accessor a: int32 = 1;
	@signal accessor b: int32 = 2;
	@signal static accessor instanceCount: uint32 = 0;
	@signal accessor #internal: int32 = 0;

	@memo
	get sum(): int32 {
		return this.a + this.b;
	}
}

const s = new Sum();
s.sum;  // 3
s.a = 10;
s.sum;  // 12
s.b = 5;
s.sum;  // 15
```

Note: Accessor is required so that all decorators see the same context. Consider:

```js
class A {
	@validate
	@signal
	accessor a: int32 = 1;
}
```

`signal` runs before `validate` and both see an accessor.
</details>

### ClassGetter
```js
namespace Reflect {
	interface ClassGetter<T, TClass> extends Reflect.ClassGetterReflection.<T> {
		classContext: Reflect.Class.<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassGetterReturn

```js
namespace Reflect {
	interface ClassGetterReturn<T, TClass> extends Reflect.ClassGetterReturnReflection.<T> {
		getterContext: Reflect.ClassGetter.<T, TClass>;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js

```
</details>

### ClassSetter
```js
namespace Reflect {
	interface ClassSetter<T, TClass> extends Reflect.ClassSetterReflection.<T> {
		classContext: Reflect.Class.<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const setterLogKey = Symbol('setterLog');

partial interface ClassSetterMetadata {
	[setterLogKey]?: { logged: boolean };
}

function logged<T, TClass>(
	{ name, type: originalSetter, metadata }: Reflect.ClassSetter.<T, TClass>,
): (value: T) => void {
	metadata[setterLogKey] = { logged: true };
	return function(value: T): void {
		console.log(`${name} = ${value}`);
		originalSetter.call(this, value);
	};
}

class Theme {
	#primary: string = '#000';

	@logged
	set primaryColor(value: string) {
		this.#primary = value;
	}
}

const t = new Theme();
t.primaryColor = '#fff'; // Logs: "primaryColor = #fff"

const setterMeta = Reflect.getMetadata.<Reflect.ClassSetter, Theme>('primaryColor');
setterMeta[setterLogKey]; // { logged: true }
```
</details>

### ClassSetterParameter

```js
namespace Reflect {
	interface ClassSetterParameter<T, TClass> extends Reflect.ClassSetterParameterReflection.<T> {
		setterContext: Reflect.ClassSetter.<T, TClass>;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
function clamp<T extends number, TClass>(
	min: T,
	max: T,
	{ setterContext }: Reflect.ClassSetterParameter.<T, TClass>
) {
	// Access setter name via setterContext.name
	// Access class metadata via setterContext.classContext.metadata
}

class Sensor {
	#temperature: float32 = 0;

	set temperature(@clamp(-273.15, 1000) value: float32) {
		this.#temperature = value;
	}
}
```
</details>

### ClassMethod
```js
namespace Reflect {
	interface ClassMethod<T extends (...args: [].<any>) => any, TClass> extends Reflect.ClassMethodReflection.<T> {
		classContext: Reflect.Class.<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const deprecatedKey = Symbol('deprecated');

partial interface ClassMethodMetadata {
	[deprecatedKey]?: { message: string, since: string };
}

function deprecated<T extends (...args: [].<any>) => any, TClass>(
	message: string,
	since: string,
	{ name, type: original, metadata }: Reflect.ClassMethod.<T, TClass>,
): T {
	metadata[deprecatedKey] = { message, since };
	let warned = false;
	return function(...args: [].<any>) {
		if (!warned) {
			console.warn(`${name} is deprecated since ${since}: ${message}`);
			warned = true;
		}
		return original.call(this, ...args);
	} := T;
}

class Api {
	@deprecated('Use fetchV2 instead', '2.0.0')
	fetch(url: string): Response {
		return httpGet(url);
	}
}

const api = new Api();
api.fetch('/data'); // Warns: "fetch is deprecated since 2.0.0: Use fetchV2 instead"

const methodMeta = Reflect.getMetadata.<Reflect.ClassMethod, Api>('fetch');
methodMeta[deprecatedKey]; // { message: 'Use fetchV2 instead', since: '2.0.0' }
```
</details>

### ClassMethodParameter
```js
namespace Reflect {
	interface ClassMethodParameter<T, TMethod, TClass> extends Reflect.ClassMethodParameterReflection.<T> {
		methodContext: Reflect.ClassMethod.<TMethod, TClass>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassMethodReturn

```js
namespace Reflect {
	interface ClassMethodReturn<T, TMethod, TClass> extends Reflect.ClassMethodReturnReflection.<T> {
		methodContext: Reflect.ClassMethod.<TMethod, TClass>;
	}
}
```

### ClassOperator
```js
enum Operator: symbol {
	AdditionAssignment
	SubtractionAssignment
	MultiplicationAssignment
	DivisionAssignment
	RemainderAssignment
	ExponentiationAssignment
	LeftShiftAssignment
	RightShiftAssignment
	UnsignedRightShiftAssignment
	BitwiseANDAssignment
	BitwiseXORAssignment
	BitwiseORAssignment
	Addition
	Subtraction
	Multiplication
	Division
	Remainder
	Exponentiation
	LeftShift
	RightShift
	UnsignedRightShift
	BitwiseAND
	BitwiseOR
	BitwiseXOR
	BitwiseNOT
	Equal
	NotEqual
	LessThan
	LessThanOrEqual
	GreaterThan
	GreaterThanOrEqual
	LogicalAND
	LogicalOR
	LogicalNOT
	Increment
	Decrement
	UnaryNegation
	UnaryPlus
};

namespace Reflect {
	interface ClassOperator<T, TClass> extends Reflect.ClassOperatorReflection.<T> {
		classContext: Reflect.Class.<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const profiledOpsKey = Symbol('profiledOps');

partial interface ClassOperatorMetadata {
	[profiledOpsKey]?: { calls: uint64, totalTime: float64 } = { calls: 0, totalTime: 0 };
}

function profiled<T, TClass>(
	{ operator, type: original, metadata }: Reflect.ClassOperator.<T, TClass>,
): T {
	return function(...args: [].<any>) {
		const start = performance.now();
		const result = original.call(this, ...args);
		metadata[profiledOpsKey].totalTime += performance.now() - start;
		metadata[profiledOpsKey].calls += 1;
		return result;
	} := T;
}

class Matrix4 {
	@profiled
	operator*(rhs: Matrix4): Matrix4 {
		// expensive multiplication
	}
}

const a = new Matrix4();
const b = new Matrix4();
const c = a * b;

const opMeta = Reflect.getMetadata.<Reflect.ClassOperator, Matrix4>(Operator.Multiplication);
opMeta[profiledOpsKey]; // { calls: 1, totalTime: ... }
```
</details>

### ClassOperatorParameter

```js
namespace Reflect {
	interface ClassOperatorParameter<T, TMethod, TClass> extends Reflect.ClassOperatorParameterReflection.<T> {
		operatorContext: Reflect.ClassOperator.<TMethod, TClass>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Function
```js
namespace Reflect {
	interface Function<T extends (...args: [].<any>) => any> extends Reflect.FunctionReflection.<T> {
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const memoKey = Symbol('memo');

partial interface FunctionMetadata {
	[memoKey]: { maxSize: uint32 } = { maxSize: 1000 };
}

function memo<T extends (...args: [].<any>) => any>(
	{ type: original, metadata }: Reflect.Function.<T>,
): T {
	const cache = new Map.<string, any>();
	return function(...args: [].<any>) {
		const key = JSON.stringify(args);
		if (cache.has(key)) {
			return cache.get(key);
		}
		const result = original(...args);
		if (cache.size >= 1000) {
			cache.clear();
		}
		cache.set(key, result);
		return result;
	} := T;
}

@memo
function fibonacci(n: uint32): uint64 {
	if (n <= 1) return n;
	return fibonacci(n - 1) + fibonacci(n - 2);
}

fibonacci(50);
```
</details>

### FunctionParameter
```js
namespace Reflect {
	interface FunctionParameter<T, TFunction> extends Reflect.FunctionParameterReflection.<T> {
		functionContext: Reflect.Function.<TFunction>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### FunctionReturn

```js
namespace Reflect {
	interface FunctionReturn<T, TFunction> extends Reflect.FunctionReturnReflection.<T> {
		functionContext: Reflect.Function.<TFunction>;
	}
}
```

### Let
```js
namespace Reflect {
	interface Let<T> extends Reflect.LetReflection.<T> {
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Const
```js
namespace Reflect {
	interface Const<T> extends Reflect.ConstReflection.<T> {
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Object
```js
namespace Reflect {
	interface Object<T> extends Reflect.ObjectReflection.<T> {
	}
}
```

<details>
	<summary>Expand for example</summary>
	
```js
function f<T>(context: Reflect.Object.<T>) {
	// ???
}

const a = @f {
	(b: number): 10
};
```
</details>

### ObjectField
```js
namespace Reflect {
	interface ObjectField<T, TObject> extends Reflect.ObjectFieldReflection.<T> {
		objectContext: Reflect.Object.<TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
function f<T>(context: Reflect.ObjectField.<T, any>) {
	// ???
}

const a = {
	@f
	(b: number): 10
};
```
</details>

### ObjectGetter
```js
namespace Reflect {
	interface ObjectGetter<T, TObject> extends Reflect.ObjectGetterReflection.<T> {
		objectContext: Reflect.Object.<TObject>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectGetterReturn

```js
namespace Reflect {
	interface ObjectGetterReturn<T, TObject> extends Reflect.ObjectGetterReturnReflection.<T> {
		getterContext: Reflect.ObjectGetter.<T, TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectSetter
```js
namespace Reflect {
	interface ObjectSetter<T, TObject> extends Reflect.ObjectSetterReflection.<T> {
		objectContext: Reflect.Object.<TObject>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js

```
</details>

### ObjectSetterParameter

```js
namespace Reflect {
	interface ObjectSetterParameter<T, TObject> extends Reflect.ObjectSetterParameterReflection.<T> {
		setterContext: Reflect.ObjectSetter.<T, TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js

```
</details>

### ObjectMethod
```js
namespace Reflect {
	interface ObjectMethod<T extends (...args: [].<any>) => any, TObject> extends Reflect.ObjectMethodReflection.<T> {
		objectContext: Reflect.Object.<TObject>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectMethodParameter
```js
namespace Reflect {
	interface ObjectMethodParameter<T, TMethod, TObject> extends Reflect.ObjectMethodParameterReflection.<T> {
		methodContext: Reflect.ObjectMethod.<TMethod, TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectMethodReturn
```js
namespace Reflect {
	interface ObjectMethodReturn<T, TMethod, TObject> extends Reflect.ObjectMethodReturnReflection.<T> {
		methodContext: Reflect.ObjectMethod.<TMethod, TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Block Decorators

Note: `TokenStream` is defined in [decoratorreplacement.md](decoratorreplacement.md). It is the token stream of what the decorator decorates - kinds, values, and spans saying where each token came from - deliberately below an AST: ECMAScript's lexical grammar is already normative, where a syntax tree would have to be invented and versioned. A decorator READS one here; a *replacement* decorator also RETURNS one, which is that document's subject.

```js
namespace Reflect {
	interface Block extends Reflect.BlockReflection {
	}

	interface IfBlock extends Reflect.IfBlockReflection {
	}

	interface ElseIfBlock extends Reflect.ElseIfBlockReflection {
	}

	interface ElseBlock extends Reflect.ElseBlockReflection {
	}

	interface WhileBlock extends Reflect.WhileBlockReflection {
	}

	interface DoWhileBlock extends Reflect.DoWhileBlockReflection {
	}

	interface ForBlock extends Reflect.ForBlockReflection {
	}

	interface ForInBlock extends Reflect.ForInBlockReflection {
	}

	interface ForOfBlock extends Reflect.ForOfBlockReflection {
	}

	interface DoBlock extends Reflect.DoBlockReflection {
	}

	interface DoGeneratorBlock extends Reflect.DoGeneratorBlockReflection {
	}

	interface MatchArmBlock extends Reflect.MatchArmBlockReflection {
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
function f(context: Reflect.Block) {
}

@f
foo: // Should I be able to access labels in context?
{
	const a = 10;
}

if (true) @f {
	
} else @f {
	
}

while (true) @f {
	
}

do @f {

} while(true);

loop:
for (let i = 0; i < 10; ++i) @f {

}

match (command) {
	when 'start': @g { start(); }
	default: @g { }
}

// A `do` block and a `do *` block are blocks too, and are the only ones whose
// decorator may RETURN a replacement, since they are the only ones with a value.
const config = @memo do { expensiveParse(readFile(path)) };
for (const v of @take(3) do * { yield* head(); yield* tail(); }) { use(v); }
```

A match arm is the position the per-entry rule pays for. A block decorator runs on every entry rather than once at the declaration, so decorating arms counts how often each case is actually taken - the measurement that tells an author their arms are in the wrong order, which is the one thing about a ```match``` that source order controls and static analysis cannot predict:

```js
const counts: Map.<uint32, uint32> = new Map();

function g({ index }: Reflect.MatchArmBlock) {
	counts.set(index, (counts.get(index) ?? 0) + 1);
}
```

```index``` is what identifies an arm, since an arm carries no label of its own; ```pattern``` is absent on a ```default``` clause, which is how the two clause kinds are told apart without a context each.

Could include in the context all sorts of information like the scope information for current variables declarations/references.

Loop blocks could also include their own context like the kind of loop, the initialization, condition, and increment expressions?

</details>

### Enum
```js
namespace Reflect {
	interface Enum<T extends enum.<TValue>, TValue = int32> extends Reflect.EnumReflection.<T, TValue> {
	}
}
```

How does this decorator work with typed decorators? It's like a union of types?

```js
enum example1 {
	example
}

enum example2: string {
	example
}
```

<details>
	<summary>Expand for example</summary>

```js
const enumInfoKey = Symbol('enumInfo');

type EnumInfo = {
	description: string
};

partial interface EnumMetadata {
	[enumInfoKey]?: EnumInfo;
}

function describe<T>(
	description: string,
	{ metadata }: Reflect.Enum.<T>,
) {
	metadata[enumInfoKey] = { description };
}

@describe('HTTP status codes')
enum Status {
	Ok,
	NotFound,
	InternalError
}
```
</details>

### EnumEnumerator
```js
namespace Reflect {
	interface EnumEnumerator<T extends enum.<TValue>, TValue = int32> extends Reflect.EnumEnumeratorReflection.<T, TValue> {
		enumContext: Reflect.Enum.<T, TValue>;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const enumLabelKey = Symbol('enumLabels');

partial interface EnumEnumeratorMetadata {
	[enumLabelKey]: Map.<string, string> = new Map();
}

function label<T extends enum.<TValue>, TValue>(
	label: string,
	locale: string = 'en',
	{ metadata }: Reflect.EnumEnumerator.<T, TValue>
) {
	metadata[enumLabelKey].set(locale, label);
}

function getLabel<T extends enum.<TValue>, TValue>(value: T, locale: string = 'en'): string {
	return Reflect.getMetadata.<Reflect.EnumEnumerator, T>(value)[enumLabelKey].get(locale);
}

enum Status {
	@label('Success')
	@label('Erfolg', 'de')
	Ok,
	@label('Not Found')
	@label('Nicht gefunden', 'de')
	NotFound,
	@label('Server Error')
	@label('Serverfehler', 'de')
	InternalError
}

getLabel(Status.NotFound, 'de'); // 'Nicht gefunden'
getLabel(Status.Ok); // 'Success'
```
</details>

Note: `getLabel` can be evaluated at compile time. Essentially both `getLabel` lines can be turned into their string literals, but if the function is redefined it would need to update those lines.

### Tuple

The array-backed composite shape, whose elements are reflected through `TupleReflection`.

```js
namespace Reflect {
	interface Tuple<T extends [].<any>> extends Reflect.TupleReflection.<T> {
	}
}
```

### Record

The object-backed composite shape, whose properties are reflected through `RecordReflection`.

```js
namespace Reflect {
	interface Record<T> extends Reflect.RecordReflection.<T> {
	}
}
```

## Examples

WIP, this should be an exhaustive list of common decorator uses including metadata uses.

### Validation

Note that [primitive metadata](https://github.com/sirisian/ecmascript-types/blob/master/primitivemetadata.md) drastically simplifies validator code. Some examples on this page can be simplified using that type system feature.  
WIP: Look into just using primitive metadata in the whole spec so as not to show more verbose syntax.

An existing TypeScript library for reference: https://github.com/typestack/class-validator

A naive example below that assumes we only want to run a validation for the whole object.

```js
const validatorsSymbol = Symbol('validators');

partial interface ClassFieldMetadata {
	[validatorsSymbol]: [].<(value: any) => boolean> = [];
}

function addValidators<T, TClass>({ name, metadata }: Reflect.ClassField.<T, TClass>, validator: (value: T) => boolean) {
	metadata[validatorsSymbol].push(validator);
}

function Length<TClass>(min: uint32, max: uint32, context: Reflect.ClassField.<string, TClass>) { // Can only be placed on string
	addValidators(context, (value: string) => value.length >= min && value.length <= max);
}
function Includes<TClass>(searchString: string, context: Reflect.ClassField.<string, TClass>) {
	addValidators(context, (value: string) => value.includes(searchString));
}
function Min<T extends int, TClass>(min: T, context: Reflect.ClassField.<T, TClass>) {
	addValidators(context, (value: T) => value >= min);
}
function Max<T extends int, TClass>(max: T, context: Reflect.ClassField.<T, TClass>) {
	addValidators(context, (value: T) => value <= max);
}
function IsEmail<TClass>(context: Reflect.ClassField.<string, TClass>) {
	addValidators(context, (value: string) => value.includes('@')); // :)
}
// ... IsFQDN and IsZonedDateTime 

function validate<T>(o: T): boolean {
	const fields = Reflect.getMetadata.<Reflect.ClassField, T>();
	for (const [name, metadata] of Object.entries(fields)) {
		for (const validator of metadata[validatorsSymbol]) {
			if (!validator(o[name])) {
				return false;
			}
		}
	}
	return true;
}

class Post {
	@Length(10, 20)
	title: string;

	@Includes('hello')
	text: string;

	//@IsInt() // Unnecessary
	@Min(0)
	@Max(10)
	rating: int32;

	@IsEmail()
	email: string;

	@IsFQDN()
	site: string;

	@IsZonedDateTime()
	createDate: Temporal.ZonedDateTime;
}

const post = new Post();
post.title = 'My post with a title...';
// ...
validate(post); // true or false
```

### Serialization

WIP: Trying to keep this example simple. This is a bit expensive in practice.

```js
const binaryWriter = Symbol('binary');

partial interface ClassFieldMetadata {
	[binaryWriter]: (packet: Packet, value: any) => void = (packet, value) => {};
}

// boolean
function data<TClass>({ metadata }: Reflect.ClassField.<boolean, TClass>) {
	metadata[binaryWriter] = (packet, value) => packet.write.<boolean>(value);
}

// uint.<N>
function data<N: uint32, TClass>({ metadata }: Reflect.ClassField.<uint.<N>, TClass>) {
	metadata[binaryWriter] = (packet, value) => packet.write.<uint.<N>>(value);
}

// string
function data<LengthType extends uint = uint16, TClass>({ metadata }: Reflect.ClassField.<string, TClass>) {
	metadata[binaryWriter] = (packet, value: string) => packet.write.<string, LengthType>(value);
}

// [].<T>
function data<T, TClass>({ metadata }: Reflect.ClassField.<[].<T>, TClass>) {
	metadata[binaryWriter] = (packet, value: [].<T>) => {
		packet.write.<uint32>(value.length);
		for (const item of value) {
			binarySerialize(packet, item);
		}
	}
}

function binarySerialize<T>(packet: Packet, item: T) {
	// Naively iterate all fields
	const fields = Reflect.getMetadata.<Reflect.ClassField, T>();
	for (const [name, metadata] of Object.entries(fields)) {
		metadata[binaryWriter](packet, item[name]);
	}
}

class Room {
	@data
	id: uint64 = 0;
	@data
	name: string = '';
	@data // Implicit generic uint.<N>
	test: uint.<7> = 0;
	@data
	currentlyBooked: boolean = false;
}

class Building {
	@data
	rooms: [].<Room> = [];
}

const packet = new Packet();
const building = new Building();
building.rooms.push(new Room());
binarySerialize(packet, building);
```

### Web component definition

```js
function register<T>(tag: string, { addInitializer }: Reflect.Class.<T>) {
	addInitializer(() => customElements.define(tag, type));
}

@register('ui-tree')
export class UITree extends HTMLElement {
	constructor() {
		super();
	}
}
```

### Dependency injection

```js
const injectKey = Symbol('inject');

partial interface ClassMethodParameterMetadata {
	[injectKey]?: { token: string | symbol };
}

function inject<T, TMethod, TClass>(
	token: string | symbol,
	{ metadata }: Reflect.ClassMethodParameter.<T, TMethod, TClass>,
) {
	metadata[injectKey] = { token };
}

function resolve<T>(cls: { new(...args: [].<any>): T }, container: Container): T {
	const params = Reflect.getReflectionByIndex.<Reflect.ClassMethodParameter, typeof cls>('constructor');
	const ctorArgs = params.map(p => {
		// Explicit token takes priority
		const token = p.metadata[injectKey]?.token;
		if (token != null) {
			return container.get(token);
		}
		// Fall back to auto-resolution by type
		return container.getByType(p.type);
	});
	return new cls(...ctorArgs);
}

class OrderService {
	#db: Database;
	#mailer: Mailer;
	#logger: Logger;

	constructor(
		@inject('database') db: Database,
		@inject('mailer') mailer: Mailer,
		logger: Logger, // No @inject — resolved by type
	) {
		this.#db = db;
		this.#mailer = mailer;
		this.#logger = logger;
	}
}

const container = new Container();
container.register('database', () => new PostgresDatabase());
container.register('mailer', () => new SmtpMailer());
container.registerByType.<Logger>(() => new ConsoleLogger());

const service = resolve(OrderService, container);
```

### Declarative routing

```js
const routeKey = Symbol('route');
const routesKey = Symbol('routes');

partial interface ClassMetadata {
	[routeKey]?: string;
}

type RouteEntry = {
	method: 'GET' | 'POST' | 'PUT' | 'DELETE',
	path: string,
	handler: string | symbol
};

partial interface ClassMethodMetadata {
	[routesKey]?: RouteEntry;
}

function route<T>(
	basePath: string,
	{ metadata }: Reflect.Class.<T>,
) {
	metadata[routeKey] = basePath;
}

function get<T extends (...args: [].<any>) => any, TClass>(
	path: string,
	{ name, metadata }: Reflect.ClassMethod.<T, TClass>,
) {
	metadata[routesKey] = { method: 'GET', path, handler: name };
}

function post<T extends (...args: [].<any>) => any, TClass>(
	path: string,
	{ name, metadata }: Reflect.ClassMethod.<T, TClass>,
) {
	metadata[routesKey] = { method: 'POST', path, handler: name };
}

function mountRoutes<T>(controller: T, router: Router) {
	const basePath = Reflect.getMetadata.<Reflect.Class, T>()[routeKey] ?? '';
	const methods = Reflect.getMetadata.<Reflect.ClassMethod, T>();
	for (const [name, metadata] of Object.entries(methods)) {
		const entry = metadata[routesKey];
		if (!entry) continue;
		const fullPath = `/${basePath}/${entry.path}`.replace(/\/+/g, '/');
		router.add(entry.method, fullPath, (...args) => controller[name](...args));
	}
}

class Room {
	id: uint64;
	name: string.<{ minLength: 1, maxLength: 80 }>; // See primitive metadata
}

const rooms: [].<Room> = [
	{ id: 0, name: 'a' }
];
let nextRoomId: uint64 = 1;

@route('rooms')
class Rooms {
	@get('')
	getRooms(): [].<Room> {
		return rooms;
	}

	@get(':id')
	getRoomById(id: uint64): Room {
		const room = rooms.find(room => room.id === id);
		if (room === undefined) {
			throw new Error('id does not exist');
		}
		return room;
	}

	@post('')
	addRoom({ name }: { name: string }): Room {
		if (rooms.find(room => room.name.toLowerCase() === name.toLowerCase())) {
			throw new Error('room name already exists');
		}
		const room: Room = { id: nextRoomId++, name };
		rooms.push(room);
		return room;
	}

	@post(':id')
	editRoom(id: uint64, { name }: { name: string }): Room {
		const room = rooms.find(room => room.id === id);
		if (room === undefined) {
			throw new Error('id does not exist');
		}
		room.name = name;
		return room;
	}
}

const router = new Router();
mountRoutes(new Rooms(), router);
```

### Documentation Generation

A good test of this would be to see if a documentation JSON can be created. Say a server wanted to automatically generate its documentation and serve it on a route. This would do that with no build step.

This documentation generation is basically reflecting a class to access its full definition.

A ```typename``` operator goes from a type to a string, which documentation generation uses to render a type.

```js
type NumberBounds<T: Ordered.<T>> = {
	bounds?: RangeBounds.<T>,
	nonZero?: boolean,
};

type StringBounds = {
	pattern?: RegExp,
	minLength?: uint32,
	maxLength?: uint32,
};

const docKey = Symbol('doc');

// Metadata

partial interface ClassMetadata {
	[docKey]?: string;
}
partial interface ClassFieldMetadata {
	[docKey]?: string;
}
partial interface ClassMethodMetadata {
	[docKey]?: string;
}
partial interface ClassMethodParameterMetadata {
	[docKey]?: string;
}
partial interface ClassGetterMetadata {
	[docKey]?: string;
}
partial interface ClassSetterMetadata {
	[docKey]?: string;
}
partial interface ClassSetterParameterMetadata {
	[docKey]?: string;
}

// Decorators

function doc<T>(description: string, { metadata }: Reflect.Class.<T>) {
	metadata[docKey] = description;
}

function doc<T, TClass>(description: string, { metadata }: Reflect.ClassField.<T, TClass>) {
	metadata[docKey] = description;
}

function doc<T extends (...args: [].<any>) => any, TClass>(description: string, { metadata }: Reflect.ClassMethod.<T, TClass>) {
	metadata[docKey] = description;
}

function doc<T, TMethod, TClass>(description: string, { metadata }: Reflect.ClassMethodParameter.<T, TMethod, TClass>) {
	metadata[docKey] = description;
}

function doc<T, TClass>(description: string, { metadata }: Reflect.ClassGetter.<T, TClass>) {
	metadata[docKey] = description;
}

function doc<T, TClass>(description: string, { metadata }: Reflect.ClassSetter.<T, TClass>) {
	metadata[docKey] = description;
}

function doc<T, TClass>(description: string, { metadata }: Reflect.ClassSetterParameter.<T, TClass>) {
	metadata[docKey] = description;
}

type ConstraintDoc = {
	minimum?: float64,
	maximum?: float64,
	exclusiveMinimum?: float64,
	exclusiveMaximum?: float64,
	minLength?: uint32,
	maxLength?: uint32,
	pattern?: string,
};

// `ConstraintDoc` stays in JSON Schema's vocabulary, because that is what a
// generated schema speaks. The metadata no longer does, so the decorator
// translates rather than spreading - which is the point of a claimed key: the
// consumer owns the translation into whatever names it needs.
function schemaFromBounds(b: NumberBounds.<float32>): ConstraintDoc {
	const r = b.bounds, doc: ConstraintDoc = {};
	if (r.start != null)
		r.startBound == Bound.Open ? doc.exclusiveMinimum = r.start : doc.minimum = r.start;
	if (r.end != null)
		r.endBound == Bound.Open ? doc.exclusiveMaximum = r.end : doc.maximum = r.end;
	return doc;
}

function constraintsFor(type: type): ConstraintDoc | undefined {
	return match (type) {
		when extends float32.<B: NumberBounds.<float32>>:
			schemaFromBounds(B); // Translate the range into JSON Schema's four fields
		when extends string.<S: StringBounds>:
			({ ...S, pattern: S.pattern?.toString() }); // Replace the pattern with the string representation
		default:
			undefined;
	};
}

// Output types

type ParamDoc = {
	name: string,
	type: string,
	description: string,
	initial?: any,
	constraints?: ConstraintDoc
};

type FieldDoc = {
	name: string,
	type: string,
	description: string,
	static: boolean,
	private: boolean,
	initial?: any,
	constraints?: ConstraintDoc
};

type AccessorDoc = {
	name: string,
	type: string,
	description: string,
	constraints?: ConstraintDoc
};

type MethodDoc = {
	name: string,
	description: string,
	returnType: string,
	parameters: [].<ParamDoc>
};

type ClassDoc = {
	name: string,
	description: string,
	fields: [].<FieldDoc>,
	getters: [].<AccessorDoc>,
	setters: [].<AccessorDoc>,
	methods: [].<MethodDoc>
};

function generateDocs<T>(): ClassDoc {
	const classRefl = Reflect.getReflection.<Reflect.Class, T>();

	const result: ClassDoc = {
		name: classRefl.name ?? '(anonymous)',
		description: classRefl.metadata[docKey] ?? '',
		fields: [],
		getters: [],
		setters: [],
		methods: []
	};

	// Fields
	const fields = Reflect.getReflection.<Reflect.ClassField, T>();
	for (const [name, field] of Object.entries(fields)) {
		result.fields.push({
			name,
			type: typename(field.type),
			description: field.metadata[docKey] ?? '',
			static: field.static,
			private: field.private,
			initial: field.initial,
			constraints: constraintsFor(field.type)
		});
	}

	// Getters, return type extracted via ClassGetterReturn
	const getters = Reflect.getReflection.<Reflect.ClassGetter, T>();
	for (const [name, getter] of Object.entries(getters)) {
		const returnRefl = Reflect.getReflection.<Reflect.ClassGetterReturn, T>(name);
		result.getters.push({
			name,
			type: typename(returnRefl.type),
			description: getter.metadata[docKey] ?? '',
			constraints: constraintsFor(returnRefl.type)
		});
	}

	// Setters, type and constraints come from the parameter
	const setters = Reflect.getReflection.<Reflect.ClassSetter, T>();
	for (const [name, setter] of Object.entries(setters)) {
		const paramRefl = Reflect.getReflection.<Reflect.ClassSetterParameter, T>(name);
		result.setters.push({
			name,
			type: typename(paramRefl.type),
			description: setter.metadata[docKey] ?? '',
			constraints: constraintsFor(paramRefl.type)
		});
	}

	// Methods
	const methods = Reflect.getReflection.<Reflect.ClassMethod, T>();
	for (const [name, method] of Object.entries(methods)) {
		const returnRefl = Reflect.getReflection.<Reflect.ClassMethodReturn, T>(name);
		const params = Reflect.getReflectionByIndex.<Reflect.ClassMethodParameter, T>(name);

		result.methods.push({
			name,
			description: method.metadata[docKey] ?? '',
			returnType: typename(returnRefl.type),
			parameters: params.map(p => ({
				name: p.name,
				type: typename(p.type),
				description: p.metadata[docKey] ?? '',
				initial: p.initial,
				constraints: constraintsFor(p.type)
			}))
		});
	}

	return result;
}

@doc('Represents a sensor device with readings and calibration.')
class Sensor {
	#offset: float32 = 0;

	@doc('Unique identifier for the sensor.')
	id: uint64;

	@doc('Human-readable sensor label.')
	label: string.<{ minLength: 1, maxLength: 120 }> = 'Unnamed Sensor';

	@doc('Current temperature reading in Celsius.')
	temperature: float32.<{ bounds: -273.15..=1000 }> = 20.0;

	@doc('Humidity percentage.')
	humidity: float32.<{ bounds: 0..=100 }> = 50.0;

	@doc('Whether the sensor is currently active.')
	active: boolean = true;

	@doc('Number of readings taken since last reset.')
	private readingCount: uint32 = 0;

	@doc('Returns the current temperature.')
	get currentTemp(): float32.<{ bounds: -273.15..=1000 }> {
		return this.temperature;
	}

	@doc('Sets the calibration offset applied to readings.')
	set calibrationOffset(
		@doc('Offset in Celsius.')
		value: float32.<{ bounds: -50..=50 }>
	) {
		this.#offset = value;
	}

	@doc('Records a new temperature and humidity reading.')
	record(
		@doc('Temperature value in Celsius.')
		temp: float32.<{ bounds: -273.15..=1000 }>,
		@doc('Humidity value as a percentage.')
		humid: float32.<{ bounds: 0..=100 }>,
		@doc('Optional timestamp override.')
		timestamp: Temporal.Instant = Temporal.Now.instant(),
	): void {
		this.temperature = temp;
		this.humidity = humid;
		this.readingCount += 1;
	}

	@doc('Resets the reading counter and optionally sets a new label.')
	reset(
		@doc('New label for the sensor.')
		newLabel: string.<{ minLength: 1, maxLength: 120 }> = this.label,
	): void {
		this.readingCount = 0;
		this.label = newLabel;
	}
}

const docs = generateDocs.<Sensor>();
```

Would generate this for `docs`:

```json
{
	"name": "Sensor",
	"description": "Represents a sensor device with readings and calibration.",
	"fields": [
		{ "name": "id", "type": "uint64", "description": "Unique identifier for the sensor.", "static": false, "private": false },
		{ "name": "label", "type": "string", "description": "Human-readable sensor label.", "static": false, "private": false, "initial": "Unnamed Sensor", "constraints": { "minLength": 1, "maxLength": 120 } },
		{ "name": "temperature", "type": "float32", "description": "Current temperature reading in Celsius.", "static": false, "private": false, "initial": 20.0, "constraints": { "minimum": -273.15, "maximum": 1000 } },
		{ "name": "humidity", "type": "float32", "description": "Humidity percentage.", "static": false, "private": false, "initial": 50.0, "constraints": { "minimum": 0, "maximum": 100 } },
		{ "name": "active", "type": "boolean", "description": "Whether the sensor is currently active.", "static": false, "private": false, "initial": true },
		{ "name": "readingCount", "type": "uint32", "description": "Number of readings taken since last reset.", "static": false, "private": true, "initial": 0 }
	],
	"getters": [
		{ "name": "currentTemp", "type": "float32", "description": "Returns the current temperature.", "constraints": { "minimum": -273.15, "maximum": 1000 } }
	],
	"setters": [
		{ "name": "calibrationOffset", "type": "float32", "description": "Sets the calibration offset applied to readings.", "constraints": { "minimum": -50, "maximum": 50 } }
	],
	"methods": [
		{
			"name": "record",
			"description": "Records a new temperature and humidity reading.",
			"returnType": "void",
			"parameters": [
				{ "name": "temp", "type": "float32", "description": "Temperature value in Celsius.", "constraints": { "minimum": -273.15, "maximum": 1000 } },
				{ "name": "humid", "type": "float32", "description": "Humidity value as a percentage.", "constraints": { "minimum": 0, "maximum": 100 } },
				{ "name": "timestamp", "type": "Temporal.Instant", "description": "Optional timestamp override." }
			]
		},
		{
			"name": "reset",
			"description": "Resets the reading counter and optionally sets a new label.",
			"returnType": "void",
			"parameters": [
				{ "name": "newLabel", "type": "string", "description": "New label for the sensor.", "constraints": { "minLength": 1, "maxLength": 120 } }
			]
		}
	]
}
```

## FAQ

### Initial only works for constants?

Yes.

```js
function f(a: uint32, b: uint32 = a * 2) {}
```

`initial` captures CONSTANT values only, as the [Reflection](#reflection) section defines it: a non-constant initializer reports *undefined*, because evaluating it would run user code at class definition rather than per call, and `initializer` carries the declaration as a `TokenStream` instead.
