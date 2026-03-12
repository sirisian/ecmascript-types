# Decorators

Very WIP: Feel free to open issues with fixes and requirements for sections.

## Introduction

https://github.com/sirisian/ecmascript-types/issues/59  
https://github.com/sirisian/ecmascript-types/issues/65

Types simplify how decorators are defined. By utilizing function overloading they get rid of the requirement to return a ```(value, context)``` function for decorators that take arguments. This means that independent of arguments a decorator looks the same. Modifying a decorator to take argument or take none just requires changing the parameters to the decorator. Consider these decorators that are all distinct:

```js
function f(context) {
	// No parameters
}
function f(x:uint32, context) {
	// x is 0
}
function f(x:string, context) {
	// x is 'a'
}

class A {
	@f
	@f(0)
	@f('a')
	a:uint32;
}
```

In the above the ```context``` is type ```any```. In actual usage one would use one of these types:

```
ClassDecorator<T>
ClassFieldDecorator<T, TClass>
ClassGetterDecorator<T, TClass>
ClassGetterReturnDecorator<T, TClass>
ClassSetterDecorator<T, TClass>
ClassSetterParameterDecorator<T, TClass>
ClassMethodDecorator<T, TClass>
ClassMethodParameterDecorator<T, TMethod, TClass>
ClassMethodReturnDecorator<T, TMethod, TClass>
ClassOperatorDecorator<T, TClass>
ClassOperatorParameterDecorator<T, TMethod, TClass>
FunctionDecorator<T>
FunctionParameterDecorator<T, TFunction>
FunctionReturnDecorator<T, TFunction>
LetDecorator<T>
ConstDecorator<T>
ObjectDecorator<T>
ObjectFieldDecorator<T, TObject>
ObjectGetterDecorator<T, TObject>
ObjectGetterReturnDecorator<T, TObject>
ObjectSetterDecorator<T, TObject>
ObjectSetterParameterDecorator<T, TMethod, TObject>
ObjectMethodDecorator<T, TObject>
ObjectMethodParameterDecorator<T, TMethod, TObject>
ObjectMethodReturnDecorator<T, TMethod, TObject>
BlockDecorator
IfBlockDecorator
ElseIfBlockDecorator
ElseBlockDecorator
WhileBlockDecorator
DoWhileBlockDecorator
ForBlockDecorator
ForInBlockDecorator
ForOfBlockDecorator
EnumDecorator<T extends enum<TValue>, TValue = int32>
EnumEnumeratorDecorator<T extends enum<TValue>, TValue = int32>
TupleDecorator<T>
RecordDecorator<T>
```

Decorators can be specialized for different types by specifying the generic parameters. They can also be specialized for different targets.

```js
function f<TClass>(context: ClassFieldDecorator<uint32, TClass>) {
	console.log('decorator on uint32');
}
function f<TClass extends A>(context: ClassFieldDecorator<any, TClass>) {
	console.log('decorator on another type, extends A');
}
function f<TClass>(context: ClassFieldDecorator<any, TClass>) {
	console.log('decorator on another type');
}

class A {
	@f // decorator on uint32
	a:uint32

	@f // decorator on another type, extends A
	b:string
}

class B {
	@f // decorator on another type
	a:string
}
```

Note that because rest parameters are allowed to be duplicated and placed anywhere this means it's legal to write:

```js
function f(...x, context) {
	// [], [0, 1, 2], ['a', 'b', 'c']
}

class A {
	@f
	a:bool

	@f(0, 1, 2)
	b:uint32

	@f('a', 'b', 'c')
	c:string
}
```

An example featuring all of them:
```js
function f(context: any) {
}

@f // ClassDecorator
class A {
	@f // ClassFieldDecorator, initial: 5
	a:uint32 = 5;
	@f
	#b:uint32 = 5;

	@f // ClassGetterDecorator
	get c():@f uint32 { // ReturnDecorator
	}

	@f // ClassSetterDecorator
	set c(@f value:uint32) { // ParameterDecorator
	}

	@f // ClassMethodDecorator
	d(@f a:uint32):@f uint32 {
	}

	@f // ClassOperatorDecorator
	operator+(@f rhs):@f uint32 {
	}
}

@f // FunctionDecorator
function g() {}

@f
let a = 5; // LetDecorator, initial

@f // ConstDecorator
const b = @f { // BlockDecorator
	@f // ObjectFieldDecorator
	a: 1
	@f // ObjectMethodDecorator
	b: () => number;
};

@f // EnumDecorator
enum Count { @f Zero, One, Two }; // EnumEnumeratorDecorator

const e = @f #[0]; // TupleDecorator

const d = @f #{ a: 1 }; // RecordDecorator
```

## Metadata

Some contexts have metadata which is on the type (class constructor) and/or target. The same is true for enum types. For objects the metadata is on the instance. So each instance created has its own unique metadata that is not shared.

All Metadata uses a partial class which can be appended with typed symbol fields.

```js
const myMetadata = Symbol('myMetadata');
partial class ClassMetadata {
	[myMetadata]: string;
};
```

Each decorator context has a reference to the target metadata:

```js
function f<T>({ metadata }: ClassDecorator<T>) {
	metadata[myMetadata] = 'f';
}

@f
class A {}

const metadata = Reflect.getMetadata<ClassDecorator, A>();
metadata[myMetadata]; // 'f'
```

This would need to also work on fields.

```js
const myMetadata = Symbol('myMetadata');
partial class ClassFieldMetadata {
	[myMetadata]: string;
};

function f<T, TClass>({ metadata }: ClassFieldDecorator<T, TClass>) {
	metadata[myMetadata] = 'f';
}

class A {
	@f
	a: uint8;
}

const metadata = Reflect.getMetadata<ClassFieldDecorator, A>('a');
metadata[myMetadata]; // 'f'
```

```Reflect.getMetadata<DecoratorContext, ...>()``` accesses the metadata:

```js
// Class-level
Reflect.getMetadata<ClassDecorator, T>(): ClassMetadata

// Class members — no key argument returns all, key argument returns one
Reflect.getMetadata<ClassFieldDecorator, T>(): { [name: string | symbol]: ClassFieldMetadata }
Reflect.getMetadata<ClassFieldDecorator, T>(name: string | symbol): ClassFieldMetadata
Reflect.getMetadata<ClassMethodDecorator, T>(): { [name: string | symbol]: ClassMethodMetadata }
Reflect.getMetadata<ClassMethodDecorator, T>(name: string | symbol): ClassMethodMetadata
Reflect.getMetadata<ClassGetterDecorator, T>(): { [name: string | symbol]: ClassGetterMetadata }
Reflect.getMetadata<ClassGetterDecorator, T>(name: string | symbol): ClassGetterMetadata
Reflect.getMetadata<ClassSetterDecorator, T>(): { [name: string | symbol]: ClassSetterMetadata }
Reflect.getMetadata<ClassSetterDecorator, T>(name: string | symbol): ClassSetterMetadata
Reflect.getMetadata<ClassOperatorDecorator, T>(): { [op: Operator]: ClassOperatorMetadata }
Reflect.getMetadata<ClassOperatorDecorator, T>(op: Operator): ClassOperatorMetadata

// Class member sub-targets (parameters, returns)
Reflect.getMetadata<ClassMethodParameterDecorator, T>(method: string | symbol): { [key: string | uint32]: ClassMethodParameterMetadata }
Reflect.getMetadata<ClassMethodParameterDecorator, T>(method: string | symbol, param: string | uint32): ClassMethodParameterMetadata
Reflect.getMetadataByIndex<ClassMethodParameterDecorator, T>(method: string | symbol): [].<ClassMethodParameterMetadata>
Reflect.getMetadata<ClassSetterParameterDecorator, T>(setter: string | symbol): ClassSetterParameterMetadata
Reflect.getMetadata<ClassGetterReturnDecorator, T>(getter: string | symbol): ClassGetterReturnMetadata
Reflect.getMetadata<ClassMethodReturnDecorator, T>(method: string | symbol): ClassMethodReturnMetadata
Reflect.getMetadata<ClassOperatorParameterDecorator, T>(op: Operator): { [index: uint32]: ClassOperatorParameterMetadata }
Reflect.getMetadata<ClassOperatorParameterDecorator, T>(op: Operator, param: string | uint32): ClassOperatorParameterMetadata
Reflect.getMetadataByIndex<ClassOperatorParameterDecorator, T>(op: Operator): [].<ClassOperatorParameterMetadata>

// Enum
Reflect.getMetadata<EnumDecorator, T>(): EnumMetadata
Reflect.getMetadata<EnumEnumeratorDecorator, T>(): { [name: string]: EnumEnumeratorMetadata }
Reflect.getMetadata<EnumEnumeratorDecorator, T>(value: T): EnumEnumeratorMetadata
// Goes directly from an enum string name to the metadata
Reflect.getMetadataByName<EnumEnumeratorDecorator, T>(name: string): EnumEnumeratorMetadata

// Function
Reflect.getMetadata<FunctionDecorator, T>(): FunctionMetadata
Reflect.getMetadata<FunctionParameterDecorator, T>(): { [key: string | uint32]: FunctionParameterMetadata }
Reflect.getMetadata<FunctionParameterDecorator, T>(param: string | uint32): FunctionParameterMetadata
Reflect.getMetadataByIndex<FunctionParameterDecorator, T>(): [].<FunctionParameterMetadata>
Reflect.getMetadata<FunctionReturnDecorator, T>(): FunctionReturnMetadata

// Object instance
Reflect.getMetadata<ObjectDecorator>(instance): ObjectMetadata
Reflect.getMetadata<ObjectFieldDecorator>(instance): { [key: string | symbol]: ObjectFieldMetadata }
Reflect.getMetadata<ObjectFieldDecorator>(instance, key: string | symbol): ObjectFieldMetadata
Reflect.getMetadata<ObjectMethodDecorator>(instance): { [key: string | symbol]: ObjectMethodMetadata }
Reflect.getMetadata<ObjectMethodDecorator>(instance, key: string | symbol): ObjectMethodMetadata
Reflect.getMetadata<ObjectGetterDecorator>(instance): { [key: string | symbol]: ObjectGetterMetadata }
Reflect.getMetadata<ObjectGetterDecorator>(instance, key: string | symbol): ObjectGetterMetadata
Reflect.getMetadata<ObjectSetterDecorator>(instance): { [key: string | symbol]: ObjectSetterMetadata }
Reflect.getMetadata<ObjectSetterDecorator>(instance, key: string | symbol): ObjectSetterMetadata
Reflect.getMetadata<ObjectMethodParameterDecorator>(instance, method: string | symbol): { [key: string | uint32]: ObjectMethodParameterMetadata }
Reflect.getMetadata<ObjectMethodParameterDecorator>(instance, method: string | symbol, param: string | uint32): ObjectMethodParameterMetadata
Reflect.getMetadataByIndex<ObjectMethodParameterDecorator>(instance, method: string | symbol): [].<ObjectMethodParameterMetadata>
Reflect.getMetadata<ObjectMethodReturnDecorator>(instance, method: string | symbol): ObjectMethodReturnMetadata
Reflect.getMetadata<ObjectSetterParameterDecorator>(instance, setter: string | symbol): ObjectSetterParameterMetadata
Reflect.getMetadata<ObjectGetterReturnDecorator>(instance, getter: string | symbol): ObjectGetterReturnMetadata
```

### Metadata Inheritance

WIP: If ```class B extends A {}``` then does ```Reflect.getMetadata<ClassFieldDecorator, B>()``` include A's field metadata?

## Decorator Contexts

Overloading parameter types and contexts allow defining specialized decorators for every situation.

### ClassDecorator
```js
interface ClassDecorator<T extends { new (...args: [].<any>): any }> {
	name: string | undefined;
	type: Function;
	metadata: ClassMetadata;
	addInitializer(initializer: () => void): void;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassFieldDecorator
```js
interface ClassFieldDecorator<T, TClass> {
	classContext: ClassDecorator<TClass>;
	type: T;
	name: string | symbol;
	static: boolean;
	private: boolean;
	initial: T | undefined; // If a constant exists, then it'll be stored here
	metadata: ClassFieldMetadata;
	addInitializer(initializer: () => void): void;
}
```

<details>
	<summary>Expand for example</summary>
	
```js
const metadataKey = Symbol('log');
function logField<T, TClass>({ classContext: { name: className, metadata: classMetadata }, name, type, static, private, metadata }: ClassFieldDecorator<T, TClass>) {
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

### ClassGetterDecorator
```js
interface ClassGetterDecorator<T, TClass> {
	classContext: ClassDecorator<TClass>;
	type: () => T;
	name: string | symbol;
	metadata: ClassGetterMetadata;
	addInitializer(initializer: () => void): void;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassGetterReturnDecorator

```js
interface ClassGetterReturnDecorator<T, TClass> { // Note: T and TMethod would be the same, so just T is used.
	getterContext: ClassGetterDecorator<T, TClass>;
	type: T;
	metadata: ClassGetterReturnMetadata;
}
```

<details>
	<summary>Expand for example</summary>

```js

```
</details>

### ClassSetterDecorator
```js
interface ClassSetterDecorator<T, TClass> {
	classContext: ClassDecorator<TClass>;
	type: (value: T) => void;
	name: string | symbol;
	metadata: ClassSetterMetadata;
	addInitializer(initializer: () => void): void;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassSetterParameterDecorator

```js
interface ClassSetterParameterDecorator<T, TClass> {
	setterContext: ClassSetterDecorator<T, TClass>;
	type: T;
	key: string;
	metadata: ClassSetterParameterMetadata;
}
```

<details>
	<summary>Expand for example</summary>

```js
function clamp<T extends number, TClass>(
	min: T,
	max: T,
	{ setterContext }: ClassSetterParameterDecorator<T, TClass>
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

### ClassMethodDecorator
```js
interface ClassMethodDecorator<T extends (...args:[].<any>) => any, TClass> {
	classContext: ClassDecorator<TClass>;
	type: T;
	name: string | symbol;
	metadata: ClassMethodMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassMethodParameterDecorator
```js
interface ClassMethodParameterDecorator<T, TMethod, TClass> {
	methodContext: ClassMethodDecorator<TMethod, TClass>;
	type: T;
	key: string;
	index: uint32;
	metadata: ClassMethodParameterMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassMethodReturnDecorator

```js
interface ClassMethodReturnDecorator<T, TMethod, TClass> {
	methodContext: ClassMethodDecorator<TMethod, TClass>;
	type: T;
	metadata: ClassMethodReturnMetadata;
}
```

### ClassOperatorDecorator
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

interface ClassOperatorDecorator<T, TClass> {
	classContext: ClassDecorator<TClass>;
	type: T;
	operator: Operator;
	metadata: ClassOperatorMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassOperatorParameterDecorator

```js
interface ClassOperatorParameterDecorator<T, TMethod, TClass> {
	operatorContext: ClassOperatorDecorator<TMethod, TClass>;
	type: T;
	key: string;
	index: uint32;
	metadata: ClassOperatorParameterMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### FunctionDecorator
```js
interface FunctionDecorator<T extends (...args:[].<any>) => any> {
	type: T;
	name: string | symbol | undefined;
	metadata: FunctionMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### FunctionParameterDecorator
```js
interface FunctionParameterDecorator<T, TFunction> {
	functionContext: FunctionDecorator<TFunction>;
	type: T;
	key: string;
	index: uint32;
	metadata: FunctionParameterMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### FunctionReturnDecorator

```js
interface FunctionReturnDecorator<T, TFunction> {
	functionContext: FunctionDecorator<TFunction>;
	type: T;
	metadata: FunctionReturnMetadata;
}
```

### LetDecorator
```js
interface LetDecorator<T> {
	type: T;
	name: string;
	initial: T | undefined;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ConstDecorator
```js
interface ConstDecorator<T> {
	type: T;
	name: string;
	initial: T;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectDecorator
```js
interface ObjectDecorator<T> {
	type: T;
	metadata: ObjectMetadata; // on the instance
}
```

<details>
	<summary>Expand for example</summary>
	
```js
function f<T>(context: ObjectDecorator<T>) {
	// ???
}

const a = @f {
	(b: number): 10
};
```
</details>

### ObjectFieldDecorator
```js
interface ObjectFieldDecorator<T, TObject> {
	objectContext: ObjectDecorator<TObject>;
	type: T;
	key: string | symbol;
	metadata: ObjectFieldMetadata; // on the instance
}
```

<details>
	<summary>Expand for example</summary>

```js
function f<T>(context: ObjectFieldDecorator<T, any>) {
	// ???
}

const a = {
	@f
	(b: number): 10
};
```
</details>

### ObjectGetterDecorator
```js
interface ObjectGetterDecorator<T, TObject> {
	objectContext: ObjectDecorator<TObject>;
	type: () => T;
	name: string | symbol;
	metadata: ObjectGetterMetadata;
	addInitializer(initializer: () => void): void;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectGetterReturnDecorator

```js
interface ObjectGetterReturnDecorator<T, TObject> { // Note: T and TMethod would be the same, so just T is used.
	getterContext: ObjectGetterDecorator<T, TObject>;
	type: T;
	metadata: ObjectGetterReturnMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectSetterDecorator
```js
interface ObjectSetterDecorator<T, TObject> {
	objectContext: ObjectDecorator<TObject>;
	type: (value: T) => void;
	name: string | symbol;
	metadata: ObjectSetterMetadata;
	addInitializer(initializer: () => void): void;
}
```

<details>
	<summary>Expand for example</summary>

```js

```
</details>

### ObjectSetterParameterDecorator

```js
interface ObjectSetterParameterDecorator<T, TObject> {
	setterContext: ObjectSetterDecorator<T, TObject>;
	type: T;
	key: string;
	metadata: ObjectSetterParameterMetadata;
}
```

<details>
	<summary>Expand for example</summary>

```js

```
</details>

### ObjectMethodDecorator
```js
interface ObjectMethodDecorator<T extends (...args:[].<any>) => any, TObject> {
	objectContext: ObjectDecorator<TObject>;
	type: T;
	key: string | symbol;
	metadata: ObjectMethodMetadata; // on the instance
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectMethodParameterDecorator
```js
interface ObjectMethodParameterDecorator<T, TMethod, TObject> {
	methodContext: ObjectMethodDecorator<TMethod, TObject>;
	type: T;
	key: string | symbol;
	index: uint32;
	metadata: ObjectMethodParameterMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ObjectMethodReturnDecorator
```js
interface ObjectMethodReturnDecorator<T, TObject> {
	methodContext: ObjectMethodDecorator<T, TObject>;
	type: T;
	metadata: ObjectMethodReturnMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Block Decorators

Note: That ```Expression``` is not defined here. Macro AST is out of scope. The Expression is a placeholder.

```js
interface BlockDecorator {
	label?: string;
	block: Expression;
}

interface IfBlockDecorator extends BlockDecorator {
	condition: Expression;
}

interface ElseIfBlockDecorator extends BlockDecorator {
	condition: Expression;
}

interface ElseBlockDecorator extends BlockDecorator {
}

interface WhileBlockDecorator extends BlockDecorator {
	condition: Expression;
}

interface DoWhileBlockDecorator extends BlockDecorator {
	condition: Expression;
}

interface ForBlockDecorator extends BlockDecorator {
	// Expose loop structure for analysis/transformation
	initializer?: Expression;
	condition?: Expression;
	update?: Expression;
}

interface ForInBlockDecorator extends BlockDecorator {
	binding: string | symbol; // the iteration variable name
}

interface ForOfBlockDecorator extends BlockDecorator {
	binding: string | symbol;
}
```

<details>
	<summary>Expand for example</summary>

```js
function f(context: BlockDecorator) {
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

### EnumDecorator
```js
interface EnumDecorator<T extends enum<TValue>, TValue = int32> {
	type: T;
	valueType: TValue;
	metadata: EnumMetadata;
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
	{ metadata }: EnumDecorator<T>,
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

### EnumEnumeratorDecorator
```js
interface EnumEnumeratorDecorator<T extends enum<TValue>, TValue = int32> {
	enumContext: EnumDecorator<T, TValue>;
	name: string;
	value: TValue;
	index: uint32;
	metadata: EnumEnumeratorMetadata;
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
	{ metadata }: EnumEnumeratorDecorator<T, TValue>
) {
	metadata[enumLabelKey][locale] = label;
}

function getLabel<T extends enum<TValue>, TValue>(value: T, locale: string = 'en'): string {
	return Reflect.getMetadata<EnumEnumeratorDecorator, T>(value)[enumLabelKey][locale];
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

Note: ```getLabel``` can be evaluated at compile time. Essentially both ```getLabel``` lines can be turned into their string literals, but if the function is redefined it would need to update those lines.

### TupleDecorator

WIP: Bring inline with composites proposal

```js
interface TupleDecorator<T extends any[]> {
	type: T;
}
```

### RecordDecorator

WIP: Bring inline with composites proposal

```js
interface RecordDecorator<K extends string | number | symbol, V> {
	type: Record<K, V>;
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

function addValidators<T, TClass>({ name, metadata }: ClassFieldDecorator<T, TClass>, validator: (value: T) => boolean) {
	metadata[validatorsSymbol].push(validator);
}

function Length<TClass>(min: uint32, max: uint32, context: ClassFieldDecorator<string, TClass>) { // Can only be placed on string
	addValidators(context, (value: string) => value.length is >= min and <= max);
}
function Includes<TClass>(searchString: string, context: ClassFieldDecorator<string, TClass>) {
	addValidators(context, (value: string) => value.includes(searchString));
}
function Min<T extends int, TClass>(min: T, context: ClassFieldDecorator<T, TClass>) {
	addValidators(context, (value: T) => value >= min);
}
function Max<T extends int, TClass>(max: T, context: ClassFieldDecorator<T, TClass>) {
	addValidators(context, (value: T) => value <= max);
}
function IsEmail<TClass>(context: ClassFieldDecorator<string, TClass>) {
	addValidators(context, (value: string) => value.includes('@')); // :)
}
// ... IsFQDN and IsZonedDateTime 

function validate<T>(o: T): boolean {
	const fields = Reflect.getMetadata<ClassFieldDecorator, T>();
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
function data<TClass>({ metadata }: ClassFieldDecorator<boolean, TClass>) {
	metadata[binaryWriter] = (packet, value) => packet.write<boolean>(value);
}

// uint<N>
function data<N: uint32, TClass>({ metadata }: ClassFieldDecorator<uint<N>, TClass>) {
	metadata[binaryWriter] = (packet, value) => packet.write<uint<N>>(value);
}

// string
function data<LengthType extends uint = uint16, TClass>({ metadata }: ClassFieldDecorator<string, TClass>) {
	metadata[binaryWriter] = (packet, value: string) => packet.write<string, LengthType>(value);
}

// [].<T>
function data<T, TClass>({ metadata }: ClassFieldDecorator<[].<T>, TClass>) {
	metadata[binaryWriter] = (packet, value: [].<T>) => {
		packet.write<uint32>(value.length);
		for (const item of value) {
			binarySerialize(packet, item);
		}
	}
}

function binarySerialize<T>(packet: Packet, item: T) {
	// Naively iterate all fields
	const fields = Reflect.getMetadata<ClassFieldDecorator, T>();
	for (const [key, metadata] of Object.entries(fields)) {
		metadata[binaryWriter](packet, item[key]);
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
function register<T>(tag: string, { addInitializer }: ClassDecorator<T>) {
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
	{ metadata }: ClassMethodParameterDecorator<T, TMethod, TClass>,
) {
	metadata[injectKey] = { token };
}

function resolve<T>(cls: { new(...args: any): T }, container: Container): T {
	const params = Reflect.getMetadataByIndex<ClassMethodParameterDecorator, T>('constructor');
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
	{ metadata }: ClassDecorator<T>,
) {
	metadata[routeKey] = basePath;
}

function get<T extends (...args: any) => any, TClass>(
	path: string,
	{ name, metadata }: ClassMethodDecorator<T, TClass>,
) {
	metadata[routesKey] = { method: 'GET', path, handler: name };
}

function post<T extends (...args: any) => any, TClass>(
	path: string,
	{ name, metadata }: ClassMethodDecorator<T, TClass>,
) {
	metadata[routesKey] = { method: 'POST', path, handler: name };
}

function mountRoutes<T>(controller: T, router: Router) {
	const basePath = Reflect.getMetadata<ClassDecorator, T>()[routeKey] ?? '';
	const methods = Reflect.getMetadata<ClassMethodDecorator, T>();
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
