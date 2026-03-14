# Decorators

Very WIP: Feel free to open issues with fixes and requirements for sections.

## Introduction

https://github.com/sirisian/ecmascript-types/issues/59  
https://github.com/sirisian/ecmascript-types/issues/65

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

Decorators can target almost anything and are defined by the following target contexts:

```
namespace Reflect {
	Class<T>
	ClassField<T, TClass>
	ClassGetter<T, TClass>
	ClassGetterReturn<T, TClass>
	ClassSetter<T, TClass>
	ClassSetterParameter<T, TClass>
	ClassMethod<T, TClass>
	ClassMethodParameter<T, TMethod, TClass>
	ClassMethodReturn<T, TMethod, TClass>
	ClassOperator<T, TClass>
	ClassOperatorParameter<T, TMethod, TClass>
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
	Enum<T extends enum<TValue>, TValue = int32>
	EnumEnumerator<T extends enum<TValue>, TValue = int32>
	Tuple<T>
	Record<T>
}
```

Decorators can be specialized for different types by specifying the generic parameters. They can also be specialized for different targets.

```js
function f<TClass>(context: Reflect.ClassField<uint32, TClass>) {
	console.log('decorator on uint32');
}
function f<TClass extends A>(context: Reflect.ClassField<any, TClass>) {
	console.log('decorator on another type, extends A');
}
function f<TClass>(context: Reflect.ClassField<any, TClass>) {
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
	a: bool

	@f(0, 1, 2)
	b: uint32

	@f('a', 'b', 'c')
	c: string
}
```

An example featuring all of them:
```js
function f(context: any) {
}

@f // Reflect.Class
class A {
	@f // Reflect.ClassField, initial: 5
	a:uint32 = 5;
	@f
	#b:uint32 = 5;

	@f // Reflect.ClassGetter
	get c(): @f uint32 { // Reflect.ClassGetterReturn
	}

	@f // Reflect.ClassSetter
	set c(@f value:uint32) { // Reflect.ClassSetterParameter
	}

	@f // Reflect.ClassMethod
	d(@f a:uint32):@f uint32 {
	}

	@f // Reflect.ClassOperator
	operator+(@f rhs):@f uint32 {
	}
}

@f // Reflect.Function
function g() {}

@f // Reflect.Let, initial: 5
let a = 5;

@f // Reflect.Const
const b = @f { // Reflect.Object
	@f // Reflect.ObjectField
	a: 1
	@f // Reflect.ObjectMethod
	b: () => number;
};

@f // Reflect.Enum
enum Count {
	@f // Reflect.EnumEnumerator
	Zero,
	One,
	Two
};

const e = @f #[0]; // Reflect.Tuple

const d = @f #{ a: 1 }; // Reflect.Record
```

## Replacement

Decorators can optionally return a replacement for the decorated target. If a decorator returns `void` (or `undefined`), no replacement occurs. If it returns a value, that value replaces the original target. The return type must be compatible with the original.

| Context | Return replaces | Return type |
|---|---|---|
| `Reflect.Class<T>` | The class itself | `T` (constructor with same interface) |
| `Reflect.ClassField<T, TClass>` | The field's initial value | `T` |
| `Reflect.ClassGetter<T, TClass>` | The getter function | `() => T` |
| `Reflect.ClassSetter<T, TClass>` | The setter function | `(value: T) => void` |
| `Reflect.ClassMethod<T, TClass>` | The method | `T` (same signature) |
| `Reflect.ClassOperator<T, TClass>` | The operator function | `T` (same signature) |
| `Reflect.Function<T>` | The function | `T` (same signature) |
| `Reflect.ObjectGetter<T, TObject>` | The getter function | `() => T` |
| `Reflect.ObjectSetter<T, TObject>` | The setter function | `(value: T) => void` |
| `Reflect.ObjectMethod<T, TObject>` | The method | `T` (same signature) |

Decorators that describe sub-targets (parameters, returns) or structural positions (blocks, enums, tuples, records, let, const) do not support return replacement.

## Reflection

The following `<Context>Reflection` types define the data that is returned when reflecting a specific target. When when reflecting a `class` one can access the `name`, `type`, and `metadata`.

### Class

```js
namespace Reflect {
	type ClassReflection = {
		name: string | undefined;
		type: Function;
		metadata: ClassMetadata;
	};

	type ClassFieldReflection<T = any> = {
		type: T;
		name: string | symbol;
		static: boolean;
		private: boolean;
		initial: T | undefined;
		metadata: ClassFieldMetadata;
	};

	type ClassGetterReflection<T = any> = {
		type: () => T;
		name: string | symbol;
		metadata: ClassGetterMetadata;
	};

	type ClassGetterReturnReflection<T = any> = {
		type: T;
		metadata: ClassGetterReturnMetadata;
	};

	type ClassSetterReflection<T = any> = {
		type: (value: T) => void;
		name: string | symbol;
		metadata: ClassSetterMetadata;
	};

	type ClassSetterParameterReflection<T = any> = {
		type: T;
		key: string;
		initial: T | undefined;
		metadata: ClassSetterParameterMetadata;
	};

	type ClassMethodReflection<T extends (...args: [].<any>) => any = (...args: [].<any>) => any> = {
		type: T;
		name: string | symbol;
		static: boolean;
		private: boolean;
		metadata: ClassMethodMetadata;
	};

	type ClassMethodParameterReflection<T = any> = {
		type: T;
		key: string;
		index: uint32;
		initial: T | undefined;
		metadata: ClassMethodParameterMetadata;
	};

	type ClassMethodReturnReflection<T = any> = {
		type: T;
		metadata: ClassMethodReturnMetadata;
	};

	type ClassOperatorReflection<T = any> = {
		type: T;
		operator: Operator;
		metadata: ClassOperatorMetadata;
	};

	type ClassOperatorParameterReflection<T = any> = {
		type: T;
		key: string;
		index: uint32;
		initial: T | undefined;
		metadata: ClassOperatorParameterMetadata;
	};
}
```

### Function

```js
namespace Reflect {
	type FunctionReflection<T extends (...args: [].<any>) => any = (...args: [].<any>) => any> = {
		type: T;
		name: string | symbol | undefined;
		metadata: FunctionMetadata;
	};

	type FunctionParameterReflection<T = any> = {
		type: T;
		key: string;
		index: uint32;
		initial: T | undefined;
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
	};

	type ConstReflection<T = any> = {
		type: T;
		name: string;
		initial: T;
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
		metadata: ObjectSetterParameterMetadata;
	};

	type ObjectMethodReflection<T extends (...args: [].<any>) => any = (...args: [].<any>) => any> = {
		type: T;
		name: string | symbol;
		metadata: ObjectMethodMetadata;
	};

	type ObjectMethodParameterReflection<T = any> = {
		type: T;
		name: string | symbol;
		index: uint32;
		initial: T | undefined;
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
		block: Expression;
	};

	type IfBlockReflection = {
		label?: string;
		block: Expression;
		condition: Expression;
	};

	type ElseIfBlockReflection = {
		label?: string;
		block: Expression;
		condition: Expression;
	};

	type ElseBlockReflection = {
		label?: string;
		block: Expression;
	};

	type WhileBlockReflection = {
		label?: string;
		block: Expression;
		condition: Expression;
	};

	type DoWhileBlockReflection = {
		label?: string;
		block: Expression;
		condition: Expression;
	};

	type ForBlockReflection = {
		label?: string;
		block: Expression;
		initializer?: Expression;
		condition?: Expression;
		update?: Expression;
	};

	type ForInBlockReflection = {
		label?: string;
		block: Expression;
		binding: string | symbol;
	};

	type ForOfBlockReflection = {
		label?: string;
		block: Expression;
		binding: string | symbol;
	};
}
```

### Enum

```js
namespace Reflect {
	type EnumReflection<T extends enum<TValue>, TValue = int32> = {
		type: T;
		valueType: TValue;
		size: uint32;
		metadata: EnumMetadata;
	};

	type EnumEnumeratorReflection<T extends enum<TValue>, TValue = int32> = {
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

	type RecordReflection<T extends Record<string | symbol, any> = Record<string | symbol, any>> = {
		type: T;
	};
}
```

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

	// Methods
	getReflection<Reflect.ClassMethod, T>(): { [name: string | symbol]: Reflect.ClassMethodReflection };
	getReflection<Reflect.ClassMethod, T>(name: string | symbol): Reflect.ClassMethodReflection;

	// Method parameters
	getReflection<Reflect.ClassMethodParameter, T>(method: string | symbol): { [key: string | uint32]: Reflect.ClassMethodParameterReflection };
	getReflection<Reflect.ClassMethodParameter, T>(method: string | symbol, param: string | uint32): Reflect.ClassMethodParameterReflection;
	getReflectionByIndex<Reflect.ClassMethodParameter, T>(method: string | symbol): [].<Reflect.ClassMethodParameterReflection>;

	// Method return
	getReflection<Reflect.ClassMethodReturn, T>(method: string | symbol): Reflect.ClassMethodReturnReflection;

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

	getReflection<Reflect.FunctionParameter, T>(): { [key: string | uint32]: Reflect.FunctionParameterReflection };
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
	getReflection<Reflect.Object>(instance): Reflect.ObjectReflection;

	// Fields
	getReflection<Reflect.ObjectField>(instance): { [name: string | symbol]: Reflect.ObjectFieldReflection };
	getReflection<Reflect.ObjectField>(instance, name: string | symbol): Reflect.ObjectFieldReflection;

	// Methods
	getReflection<Reflect.ObjectMethod>(instance): { [name: string | symbol]: Reflect.ObjectMethodReflection };
	getReflection<Reflect.ObjectMethod>(instance, name: string | symbol): Reflect.ObjectMethodReflection;

	// Method parameters
	getReflection<Reflect.ObjectMethodParameter>(instance, method: string | symbol): { [key: string | uint32]: Reflect.ObjectMethodParameterReflection };
	getReflection<Reflect.ObjectMethodParameter>(instance, method: string | symbol, param: string | uint32): Reflect.ObjectMethodParameterReflection;
	getReflectionByIndex<Reflect.ObjectMethodParameter>(instance, method: string | symbol): [].<Reflect.ObjectMethodParameterReflection>;

	// Method return
	getReflection<Reflect.ObjectMethodReturn>(instance, method: string | symbol): Reflect.ObjectMethodReturnReflection;

	// Getters
	getReflection<Reflect.ObjectGetter>(instance): { [name: string | symbol]: Reflect.ObjectGetterReflection };
	getReflection<Reflect.ObjectGetter>(instance, name: string | symbol): Reflect.ObjectGetterReflection;

	// Getter return
	getReflection<Reflect.ObjectGetterReturn>(instance, getter: string | symbol): Reflect.ObjectGetterReturnReflection;

	// Setters
	getReflection<Reflect.ObjectSetter>(instance): { [name: string | symbol]: Reflect.ObjectSetterReflection };
	getReflection<Reflect.ObjectSetter>(instance, name: string | symbol): Reflect.ObjectSetterReflection;

	// Setter parameter
	getReflection<Reflect.ObjectSetterParameter>(instance, setter: string | symbol): Reflect.ObjectSetterParameterReflection;
}
```

## `Reflect.getMetadata` Signatures

`Reflect.getMetadata` is sugar over `Reflect.getReflection`, returning only the `.metadata` field. Only targets whose reflection structures carry metadata have a corresponding `Reflect.getMetadata` overload.

### Class

```js
namespace Reflect {
	getMetadata<Reflect.Class, T>(): ClassMetadata;

	getMetadata<Reflect.ClassField, T>(): { [name: string | symbol]: ClassFieldMetadata };
	getMetadata<Reflect.ClassField, T>(name: string | symbol): ClassFieldMetadata;

	getMetadata<Reflect.ClassMethod, T>(): { [name: string | symbol]: ClassMethodMetadata };
	getMetadata<Reflect.ClassMethod, T>(name: string | symbol): ClassMethodMetadata;

	getMetadata<Reflect.ClassMethodParameter, T>(method: string | symbol): { [key: string | uint32]: ClassMethodParameterMetadata };
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
	getMetadataByIndex<Reflect.ClassOperatorParameter, T>(op: Operator): [].<ClassOperatorParameterMetadata>;
}
```

### Function

```js
namespace Reflect {
	getMetadata<Reflect.Function, T>(): FunctionMetadata;

	getMetadata<Reflect.FunctionParameter, T>(): { [key: string | uint32]: FunctionParameterMetadata };
	getMetadata<Reflect.FunctionParameter, T>(param: string | uint32): FunctionParameterMetadata;
	getMetadataByIndex<Reflect.FunctionParameter, T>(): [].<FunctionParameterMetadata>;

	getMetadata<Reflect.FunctionReturn, T>(): FunctionReturnMetadata;
}
```

### Object (instance-based)

```js
namespace Reflect {
	getMetadata<Reflect.Object>(instance): ObjectMetadata;

	getMetadata<Reflect.ObjectField>(instance): { [name: string | symbol]: ObjectFieldMetadata };
	getMetadata<Reflect.ObjectField>(instance, name: string | symbol): ObjectFieldMetadata;

	getMetadata<Reflect.ObjectMethod>(instance): { [name: string | symbol]: ObjectMethodMetadata };
	getMetadata<Reflect.ObjectMethod>(instance, name: string | symbol): ObjectMethodMetadata;

	getMetadata<Reflect.ObjectMethodParameter>(instance, method: string | symbol): { [key: string | uint32]: ObjectMethodParameterMetadata };
	getMetadata<Reflect.ObjectMethodParameter>(instance, method: string | symbol, param: string | uint32): ObjectMethodParameterMetadata;
	getMetadataByIndex<Reflect.ObjectMethodParameter>(instance, method: string | symbol): [].<ObjectMethodParameterMetadata>;

	getMetadata<Reflect.ObjectMethodReturn>(instance, method: string | symbol): ObjectMethodReturnMetadata;

	getMetadata<Reflect.ObjectGetter>(instance): { [name: string | symbol]: ObjectGetterMetadata };
	getMetadata<Reflect.ObjectGetter>(instance, name: string | symbol): ObjectGetterMetadata;

	getMetadata<Reflect.ObjectGetterReturn>(instance, getter: string | symbol): ObjectGetterReturnMetadata;

	getMetadata<Reflect.ObjectSetter>(instance): { [name: string | symbol]: ObjectSetterMetadata };
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

All Metadata uses a partial class which can be appended with typed symbol fields.

```js
const myMetadata = Symbol('myMetadata');
partial class ClassMetadata {
	[myMetadata]: string;
};
```

Each decorator context has a reference to the target metadata:

```js
function f<T>({ metadata }: Reflect.Class<T>) {
	metadata[myMetadata] = 'f';
}

@f
class A {}

const metadata = Reflect.getMetadata<Reflect.Class, A>();
metadata[myMetadata]; // 'f'
```

This would need to also work on fields.

```js
const myMetadata = Symbol('myMetadata');
partial class ClassFieldMetadata {
	[myMetadata]: string;
};

function f<T, TClass>({ metadata }: Reflect.ClassField<T, TClass>) {
	metadata[myMetadata] = 'f';
}

class A {
	@f
	a: uint8;
}

const metadata = Reflect.getMetadata<Reflect.ClassField, A>('a');
metadata[myMetadata]; // 'f'
```

### Metadata Inheritance

WIP: If `class B extends A {}` then does `Reflect.getMetadata<Reflect.ClassField, B>()` include A's field metadata?

## addInitializer

Present on contexts that represent declaration sites where initialization logic can be injected:

| Has `addInitializer` | Does not |
|---|---|
| `Reflect.Class` | `Reflect.ClassMethodParameter` |
| `Reflect.ClassField` | `Reflect.ClassMethodReturn` |
| `Reflect.ClassMethod` | `Reflect.ClassGetterReturn` |
| `Reflect.ClassGetter` | `Reflect.ClassSetterParameter` |
| `Reflect.ClassSetter` | `Reflect.ClassOperatorParameter` |
| `Reflect.ClassOperator` | `Reflect.Function` |
| `Reflect.ObjectMethod` | `Reflect.FunctionParameter` |
| `Reflect.ObjectGetter` | `Reflect.FunctionReturn` |
| `Reflect.ObjectSetter` | `Reflect.Let` / `Reflect.Const` |
| | `Reflect.ObjectField` |
| | `Reflect.ObjectMethodParameter` |
| | `Reflect.ObjectMethodReturn` |
| | `Reflect.ObjectGetterReturn` |
| | `Reflect.ObjectSetterParameter` |
| | `Reflect.Enum` / `Reflect.EnumEnumerator` |
| | `Reflect.Tuple` / `Reflect.Record` |
| | All block contexts |

## Decorator Contexts

Overloading a decorator's parameter types and contexts allow defining specialized decorators for every situation.

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

partial class ClassMetadata {
	[singletonKey]?: { instance: any };
}

function singleton<T>({ metadata }: Reflect.Class<T>): T {
	metadata[singletonKey] = { instance: undefined };
	let instance: T | undefined;
	return class extends T {
		constructor(...args: any) {
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
function sealed<T>({ type }: Reflect.Class<T>) {
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
	interface ClassField<T, TClass> extends Reflect.ClassFieldReflection<T> {
		classContext: Reflect.Class<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>
	
```js
const metadataKey = Symbol('log');
function logField<T, TClass>({ classContext: { name: className, metadata: classMetadata }, name, type, static, private, metadata }: Reflect.ClassField<T, TClass>) {
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

### ClassGetter
```js
namespace Reflect {
	interface ClassGetter<T, TClass> extends Reflect.ClassGetterReflection<T> {
		classContext: Reflect.Class<TClass>;
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
	interface ClassGetterReturn<T, TClass> extends Reflect.ClassGetterReturnReflection<T> {
		getterContext: Reflect.ClassGetter<T, TClass>;
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
	interface ClassSetter<T, TClass> extends Reflect.ClassSetterReflection<T> {
		classContext: Reflect.Class<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const setterLogKey = Symbol('setterLog');

partial class ClassSetterMetadata {
	[setterLogKey]?: { logged: boolean };
}

function logged<T, TClass>(
	{ name, type: originalSetter, metadata }: Reflect.ClassSetter<T, TClass>,
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

const setterMeta = Reflect.getMetadata<Reflect.ClassSetter, Theme>('primaryColor');
setterMeta[setterLogKey]; // { logged: true }
```
</details>

### ClassSetterParameter

```js
namespace Reflect {
	interface ClassSetterParameter<T, TClass> extends Reflect.ClassSetterParameterReflection<T> {
		setterContext: Reflect.ClassSetter<T, TClass>;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
function clamp<T extends number, TClass>(
	min: T,
	max: T,
	{ setterContext }: Reflect.ClassSetterParameter<T, TClass>
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
	interface ClassMethod<T extends (...args: [].<any>) => any, TClass> extends Reflect.ClassMethodReflection<T> {
		classContext: Reflect.Class<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const deprecatedKey = Symbol('deprecated');

partial class ClassMethodMetadata {
	[deprecatedKey]?: { message: string, since: string };
}

function deprecated<T extends (...args: any) => any, TClass>(
	message: string,
	since: string,
	{ name, type: original, metadata }: ClassMethodDecorator<T, TClass>,
): T {
	metadata[deprecatedKey] = { message, since };
	let warned = false;
	return function(...args: any) {
		if (!warned) {
			console.warn(`${name} is deprecated since ${since}: ${message}`);
			warned = true;
		}
		return original.call(this, ...args);
	} as T;
}

class Api {
	@deprecated('Use fetchV2 instead', '2.0.0')
	fetch(url: string): Response {
		return httpGet(url);
	}
}

const api = new Api();
api.fetch('/data'); // Warns: "fetch is deprecated since 2.0.0: Use fetchV2 instead"

const methodMeta = Reflect.getMetadata<Reflect.ClassMethod, Api>('fetch');
methodMeta[deprecatedKey]; // { message: 'Use fetchV2 instead', since: '2.0.0' }
```
</details>

### ClassMethodParameter
```js
namespace Reflect {
	interface ClassMethodParameter<T, TMethod, TClass> extends Reflect.ClassMethodParameterReflection<T> {
		methodContext: Reflect.ClassMethod<TMethod, TClass>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassMethodReturn

```js
namespace Reflect {
	interface ClassMethodReturn<T, TMethod, TClass> extends Reflect.ClassMethodReturnReflection<T> {
		methodContext: Reflect.ClassMethod<TMethod, TClass>;
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
	interface ClassOperator<T, TClass> extends Reflect.ClassOperatorReflection<T> {
		classContext: Reflect.Class<TClass>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const profiledOpsKey = Symbol('profiledOps');

partial class ClassOperatorMetadata {
	[profiledOpsKey]?: { calls: uint64, totalTime: float64 } = { calls: 0, totalTime: 0 };
}

function profiled<T, TClass>(
	{ operator, type: original, metadata }: Reflect.ClassOperator<T, TClass>,
): T {
	return function(...args: any) {
		const start = performance.now();
		const result = original.call(this, ...args);
		metadata[profiledOpsKey].totalTime += performance.now() - start;
		metadata[profiledOpsKey].calls += 1;
		return result;
	} as T;
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

const opMeta = Reflect.getMetadata<ClassOperator, Matrix4>(Operator.Multiplication);
opMeta[profiledOpsKey]; // { calls: 1, totalTime: ... }
```
</details>

### ClassOperatorParameter

```js
namespace Reflect {
	interface ClassOperatorParameter<T, TMethod, TClass> extends Reflect.ClassOperatorParameterReflection<T> {
		operatorContext: Reflect.ClassOperator<TMethod, TClass>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Function
```js
namespace Reflect {
	interface Function<T extends (...args: [].<any>) => any> extends Reflect.FunctionReflection<T> {
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const memoKey = Symbol('memo');

partial class FunctionMetadata {
	[memoKey]: { maxSize: uint32 } = { maxSize: 1000 };
}

function memo<T extends (...args: any) => any>(
	{ type: original, metadata }: Reflect.Function<T>,
): T {
	const cache = new Map<string, any>();
	return function(...args: any) {
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
	} as T;
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
	interface FunctionParameter<T, TFunction> extends Reflect.FunctionParameterReflection<T> {
		functionContext: Reflect.Function<TFunction>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### FunctionReturn

```js
namespace Reflect {
	interface FunctionReturn<T, TFunction> extends Reflect.FunctionReturnReflection<T> {
		functionContext: Reflect.Function<TFunction>;
	}
}
```

### Let
```js
namespace Reflect {
	interface Let<T> extends Reflect.LetReflection<T> {
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Const
```js
namespace Reflect {
	interface Const<T> extends Reflect.ConstReflection<T> {
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Object
```js
namespace Reflect {
	interface Object<T> extends Reflect.ObjectReflection<T> {
	}
}
```

<details>
	<summary>Expand for example</summary>
	
```js
function f<T>(context: Reflect.Object<T>) {
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
	interface ObjectField<T, TObject> extends Reflect.ObjectFieldReflection<T> {
		objectContext: Reflect.Object<TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
function f<T>(context: Reflect.ObjectField<T, any>) {
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
	interface ObjectGetter<T, TObject> extends Reflect.ObjectGetterReflection<T> {
		objectContext: Reflect.Object<TObject>;
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
	interface ObjectGetterReturn<T, TObject> extends Reflect.ObjectGetterReturnReflection<T> {
		getterContext: Reflect.ObjectGetter<T, TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectSetter
```js
namespace Reflect {
	interface ObjectSetter<T, TObject> extends Reflect.ObjectSetterReflection<T> {
		objectContext: Reflect.Object<TObject>;
		addInitializer(initializer: () => void): void;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js

```
</details>

### ObjectSetterParameterDecorator

```js
namespace Reflect {
	interface ObjectSetterParameter<T, TObject> extends Reflect.ObjectSetterParameterReflection<T> {
		setterContext: Reflect.ObjectSetter<T, TObject>;
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
	interface ObjectMethod<T extends (...args: [].<any>) => any, TObject> extends Reflect.ObjectMethodReflection<T> {
		objectContext: Reflect.Object<TObject>;
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
	interface ObjectMethodParameter<T, TMethod, TObject> extends Reflect.ObjectMethodParameterReflection<T> {
		methodContext: Reflect.ObjectMethod<TMethod, TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectMethodReturn
```js
namespace Reflect {
	interface ObjectMethodReturn<T, TMethod, TObject> extends Reflect.ObjectMethodReturnReflection<T> {
		methodContext: Reflect.ObjectMethod<TMethod, TObject>;
	}
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Block Decorators

Note: That `Expression` is not defined here. Macro AST is out of scope. The Expression is a placeholder.

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
```

Could include in the context all sorts of information like the scope information for current variables declarations/references.

Loop blocks could also include their own context like the kind of loop, the initialization, condition, and increment expressions?

</details>

### Enum
```js
namespace Reflect {
	interface Enum<T extends enum<TValue>, TValue = int32> extends Reflect.EnumReflection<T, TValue> {
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

partial class EnumMetadata {
	[enumInfoKey]?: EnumInfo;
}

function describe<T>(
	description: string,
	{ metadata }: Reflect.Enum<T>,
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
	interface EnumEnumerator<T extends enum<TValue>, TValue = int32> extends Reflect.EnumEnumeratorReflection<T, TValue> {
		enumContext: Reflect.Enum<T, TValue>;
	}
}
```

<details>
	<summary>Expand for example</summary>

```js
const enumLabelKey = Symbol('enumLabels');

partial class EnumEnumeratorMetadata {
	[enumLabelKey]: Record<string, string> = {};
}

function label<T extends enum<TValue>, TValue>(
	label: string,
	locale: string = 'en',
	{ metadata }: Reflect.EnumEnumerator<T, TValue>
) {
	metadata[enumLabelKey][locale] = label;
}

function getLabel<T extends enum<TValue>, TValue>(value: T, locale: string = 'en'): string {
	return Reflect.getMetadata<Reflect.EnumEnumerator, T>(value)[enumLabelKey][locale];
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

WIP: Bring inline with composites proposal

```js
namespace Reflect {
	interface Tuple<T extends [].<any>> extends Reflect.TupleReflection<T> {
	}
}
```

### RecordDecorator

WIP: Bring inline with composites proposal

```js
namespace Reflect {
	interface Record<T extends Record<string | symbol, any>> extends Reflect.RecordReflection<T> {
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

partial class ClassFieldMetadata {
	[validatorsSymbol]: [].<(value: any) => boolean> = [];
}

function addValidators<T, TClass>({ name, metadata }: Reflect.ClassField<T, TClass>, validator: (value: T) => boolean) {
	metadata[validatorsSymbol].push(validator);
}

function Length<TClass>(min: uint32, max: uint32, context: Reflect.ClassField<string, TClass>) { // Can only be placed on string
	addValidators(context, (value: string) => value.length >= min && value.length <= max);
}
function Includes<TClass>(searchString: string, context: Reflect.ClassField<string, TClass>) {
	addValidators(context, (value: string) => value.includes(searchString));
}
function Min<T extends int, TClass>(min: T, context: Reflect.ClassField<T, TClass>) {
	addValidators(context, (value: T) => value >= min);
}
function Max<T extends int, TClass>(max: T, context: Reflect.ClassField<T, TClass>) {
	addValidators(context, (value: T) => value <= max);
}
function IsEmail<TClass>(context: Reflect.ClassField<string, TClass>) {
	addValidators(context, (value: string) => value.includes('@')); // :)
}
// ... IsFQDN and IsZonedDateTime 

function validate<T>(o: T): boolean {
	const fields = Reflect.getMetadata<Reflect.ClassField, T>();
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

partial class ClassFieldMetadata {
	[binaryWriter]: (packet: Packet, value: any) => void = (packet, value) => {};
}

// boolean
function data<TClass>({ metadata }: Reflect.ClassField<boolean, TClass>) {
	metadata[binaryWriter] = (packet, value) => packet.write<boolean>(value);
}

// uint<N>
function data<N: uint32, TClass>({ metadata }: Reflect.ClassField<uint<N>, TClass>) {
	metadata[binaryWriter] = (packet, value) => packet.write<uint<N>>(value);
}

// string
function data<LengthType extends uint = uint16, TClass>({ metadata }: Reflect.ClassField<string, TClass>) {
	metadata[binaryWriter] = (packet, value: string) => packet.write<string, LengthType>(value);
}

// [].<T>
function data<T, TClass>({ metadata }: Reflect.ClassField<[].<T>, TClass>) {
	metadata[binaryWriter] = (packet, value: [].<T>) => {
		packet.write<uint32>(value.length);
		for (const item of value) {
			binarySerialize(packet, item);
		}
	}
}

function binarySerialize<T>(packet: Packet, item: T) {
	// Naively iterate all fields
	const fields = Reflect.getMetadata<Reflect.ClassField, T>();
	for (const [name, metadata] of Object.entries(fields)) {
		metadata[binaryWriter](packet, item[name]);
	}
}

class Room {
	@data
	id: uint64 = -1;
	@data
	name: string = '';
	@data // Implicit generic uint<n>
	test: uint<7> = 0;
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
function register<T>(tag: string, { addInitializer }: Reflect.Class<T>) {
	addInitializer(() => customElements.define(tag, T));
}

@register('ui-tree')
export class UITree extends HTMLElement {
	constructor() {
		super();
	}
}
```

### Dependency injection

WIP: Is this the best implementation of this pattern?

```js
const injectKey = Symbol('inject');

partial class ClassMethodParameterMetadata {
	[injectKey]?: { token: string | symbol };
}

function inject<T, TMethod, TClass>(
	token: string | symbol,
	{ metadata }: Reflect.ClassMethodParameter<T, TMethod, TClass>,
) {
	metadata[injectKey] = { token };
}

function resolve<T>(cls: { new(...args: any): T }, container: Container): T {
	const params = Reflect.getMetadataByIndex<Reflect.ClassMethodParameter, T>('constructor');
	const ctorArgs = params.map(p => container.get(p[injectKey].token));
	return new cls(...ctorArgs);
}

class OrderService {
	#db: Database;
	#mailer: Mailer;

	constructor(
		@inject('database') db: Database,
		@inject('mailer') mailer: Mailer
	) {
		this.#db = db;
		this.#mailer = mailer;
	}
}

const service = resolve(OrderService, container);
```

### Declarative routing

```js
const routeKey = Symbol('route');
const routesKey = Symbol('routes');

partial class ClassMetadata {
	[routeKey]?: string;
}

type RouteEntry = {
	method: 'GET' | 'POST' | 'PUT' | 'DELETE',
	path: string,
	handler: string | symbol
};

partial class ClassMethodMetadata {
	[routesKey]?: RouteEntry;
}

function route<T>(
	basePath: string,
	{ metadata }: Reflect.Class<T>,
) {
	metadata[routeKey] = basePath;
}

function get<T extends (...args: any) => any, TClass>(
	path: string,
	{ name, metadata }: Reflect.ClassMethod<T, TClass>,
) {
	metadata[routesKey] = { method: 'GET', path, handler: name };
}

function post<T extends (...args: any) => any, TClass>(
	path: string,
	{ name, metadata }: Reflect.ClassMethod<T, TClass>,
) {
	metadata[routesKey] = { method: 'POST', path, handler: name };
}

function mountRoutes<T>(controller: T, router: Router) {
	const basePath = Reflect.getMetadata<Reflect.Class, T>()[routeKey] ?? '';
	const methods = Reflect.getMetadata<Reflect.ClassMethod, T>();
	for (const [name, metadata] of Object.entries(methods)) {
		const entry = metadata[routesKey];
		if (!entry) continue;
		const fullPath = `/${basePath}/${entry.path}`.replace(/\/+/g, '/');
		router.add(entry.method, fullPath, (...args) => controller[name](...args));
	}
}

class Room {
	id: uint64;
	name: string<{ minLength: 1, maxLength: 80 }>; // See primitive metadata
}

typed RoomWithoutId = Omit<Room, { id }>;

const rooms: [].<Room> = [
	{ id: 0, name: 'a' }
];

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
		rooms.push({ name });
	}

	@post(':id')
	editRoom(id: uint64, { name }: Room | RoomWithoutId): Room {
		const room = rooms.find(room => room.id === id);
		if (room === undefined) {
			throw new Error('id does not exist');
		}
		room.name = name;
	}
}

const router = new Router();
mountRoutes(new Rooms(), router);
```

### Documentation Generation

A good test of this would be to see if a documentation JSON can be created. Say a server wanted to automatically generate its documentation and serve it on a route. This would do that with no build step.

This documentation generation is basically reflecting a class to access its full definition.

I'm going to include typename from this suggestion to go from a type to a string: https://github.com/microsoft/TypeScript/issues/29944

```js
type NumberBounds = {
	minimum?: float32,
	maximum?: float32,
	exclusiveMinimum?: float32,
	exclusiveMaximum?: float32,
};

type StringBounds = {
	pattern?: RegExp,
	minLength?: uint32,
	maxLength?: uint32,
};

const docKey = Symbol('doc');

// Metadata

partial class ClassMetadata {
	[docKey]?: string;
}
partial class ClassFieldMetadata {
	[docKey]?: string;
}
partial class ClassMethodMetadata {
	[docKey]?: string;
}
partial class ClassMethodParameterMetadata {
	[docKey]?: string;
}
partial class ClassGetterMetadata {
	[docKey]?: string;
}
partial class ClassSetterMetadata {
	[docKey]?: string;
}
partial class ClassSetterParameterMetadata {
	[docKey]?: string;
}

// Decorators

function doc<T>(description: string, { metadata }: Reflect.Class<T>) {
	metadata[docKey] = description;
}

function doc<T, TClass>(description: string, { metadata }: Reflect.ClassField<T, TClass>) {
	metadata[docKey] = description;
}

function doc<T extends (...args: any) => any, TClass>(description: string, { metadata }: Reflect.ClassMethod<T, TClass>) {
	metadata[docKey] = description;
}

function doc<T, TMethod, TClass>(description: string, { metadata }: Reflect.ClassMethodParameter<T, TMethod, TClass>) {
	metadata[docKey] = description;
}

function doc<T, TClass>(description: string, { metadata }: Reflect.ClassGetter<T, TClass>) {
	metadata[docKey] = description;
}

function doc<T, TClass>(description: string, { metadata }: Reflect.ClassSetter<T, TClass>) {
	metadata[docKey] = description;
}

function doc<T, TClass>(description: string, { metadata }: Reflect.ClassSetterParameter<T, TClass>) {
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

function constraintsFor(type): ConstraintDoc | undefined {
	return match (type) {
		when extends number<B: NumberBounds>: // This would be new syntax for pattern matching
			break { ...B };
		when extends string<S: StringBounds>:
			const c: ConstraintDoc = {};
			if (S.minLength != null) c.minLength = S.minLength;
			if (S.maxLength != null) c.maxLength = S.maxLength;
			if (S.pattern != null) c.pattern = S.pattern.source;
			break c;
		default:
			break undefined;
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
	const classRefl = Reflect.getReflection<Reflect.Class, T>();

	const result: ClassDoc = {
		name: classRefl.name ?? '(anonymous)',
		description: classRefl.metadata[docKey] ?? '',
		fields: [],
		getters: [],
		setters: [],
		methods: []
	};

	// Fields
	const fields = Reflect.getReflection<Reflect.ClassField, T>();
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
	const getters = Reflect.getReflection<Reflect.ClassGetter, T>();
	for (const [name, getter] of Object.entries(getters)) {
		const returnRefl = Reflect.getReflection<Reflect.ClassGetterReturn, T>(name);
		result.getters.push({
			name,
			type: typename(returnRefl.type),
			description: getter.metadata[docKey] ?? '',
			constraints: constraintsFor(returnRefl.type)
		});
	}

	// Setters, type and constraints come from the parameter
	const setters = Reflect.getReflection<Reflect.ClassSetter, T>();
	for (const [name, setter] of Object.entries(setters)) {
		const paramRefl = Reflect.getReflection<Reflect.ClassSetterParameter, T>(name);
		result.setters.push({
			name,
			type: typename(paramRefl.type),
			description: setter.metadata[docKey] ?? '',
			constraints: constraintsFor(paramRefl.type)
		});
	}

	// Methods
	const methods = Reflect.getReflection<Reflect.ClassMethod, T>();
	for (const [name, method] of Object.entries(methods)) {
		const returnRefl = Reflect.getReflection<Reflect.ClassMethodReturn, T>(name);
		const params = Reflect.getReflectionByIndex<Reflect.ClassMethodParameter, T>(name);

		result.methods.push({
			name,
			description: method.metadata[docKey] ?? '',
			returnType: typename(returnRefl.type),
			parameters: params.map(p => ({
				name: p.key,
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
	label: string<{ minLength: 1, maxLength: 120 }> = 'Unnamed Sensor';

	@doc('Current temperature reading in Celsius.')
	temperature: float32<{ minimum: -273.15, maximum: 1000 }> = 20.0;

	@doc('Humidity percentage.')
	humidity: float32<{ minimum: 0, maximum: 100 }> = 50.0;

	@doc('Whether the sensor is currently active.')
	active: boolean = true;

	@doc('Number of readings taken since last reset.')
	private readingCount: uint32 = 0;

	@doc('Returns the current temperature.')
	get currentTemp(): float32<{ minimum: -273.15, maximum: 1000 }> {
		return this.temperature;
	}

	@doc('Sets the calibration offset applied to readings.')
	set calibrationOffset(
		@doc('Offset in Celsius.')
		value: float32<{ minimum: -50, maximum: 50 }>
	) {
		this.#offset = value;
	}

	@doc('Records a new temperature and humidity reading.')
	record(
		@doc('Temperature value in Celsius.')
		temp: float32<{ minimum: -273.15, maximum: 1000 }>,
		@doc('Humidity value as a percentage.')
		humid: float32<{ minimum: 0, maximum: 100 }>,
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
		newLabel: string<{ minLength: 1, maxLength: 120 }> = this.label,
	): void {
		this.readingCount = 0;
		this.label = newLabel;
	}
}

const docs = generateDocs<Sensor>();
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

Similar to `Reflect.ClassField` the parameter decorators only capture constant values. This is a limitation. Ideally one could capture the `Expression`, but I'm trying not to directly implement AST into this yet. Just leaving it as an extension.
