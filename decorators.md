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
ClassSetterDecorator<T, TClass>
ClassMethodDecorator<T, TClass>
ClassMethodParameterDecorator<T, TMethod, TClass>
ClassOperatorDecorator<T, TClass>
ClassOperatorParameterDecorator<T, TMethod, TClass>
FunctionDecorator<T>
FunctionParameterDecorator<T, TFunction>
ParameterDecorator<T>
LetDecorator<T>
ConstDecorator<T>
ObjectDecorator<T>
ObjectFieldDecorator<T, TObject>
ObjectGetterDecorator<T, TObject>
ObjectSetterDecorator<T, TObject>
ObjectMethodDecorator<T, TObject>
ObjectMethodParameterDecorator<T, TMethod, TObject>
BlockDecorator
IfBlockDecorator
IfElseBlockDecorator
ElseBlockDecorator
WhileBlockDecorator
DoWhileBlockDecorator
ForBlockDecorator
ForInBlockDecorator
ForOfBlockDecorator
InitializerDecorator<T>
ReturnDecorator<T>
EnumDecorator<T>
EnumEnumeratorDecorator<T>
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
	@f // ClassFieldDecorator
	a:uint32 @f = 5 // InitializerDecorator
	@f
	#b:uint32 @f = 5

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
let @f a @f = 5; // LetDecorator, InitializerDecorator

const @f b @f = @f { // ConstDecorator, InitializerDecorator, BlockDecorator
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

const metadata: ClassMetadata = Reflect.getMetadata<ClassDecorator, A>();
metadata[myMetadata]; // 'f'
```

This would need to also work on fields.

```js
function f<T, TClass>({ metadata }: ClassFieldDecorator<T, TClass>) {
	metadata[myMetadata] = 'f';
}
class A {
	@f
	a: uint8;
}

const metadata: ClassFieldMetadata = Reflect.getMetadata<ClassFieldDecorator, A, 'a'>();
metadata[myMetadata]; // 'f'
```

```Reflect.getMetadata<DecoratorContext, ...>()``` accesses the metadata:

```js
Reflect.getMetadata<ClassDecorator, T>(): ClassMetadata
Reflect.getMetadata<ClassFieldDecorator, T>(): { [name]: ClassFieldMetadata }
Reflect.getMetadata<ClassMethodDecorator, T>(): { [name]: ClassMethodMetadata }
Reflect.getMetadata<ClassGetterDecorator, T>(): { [name]: ClassGetterMetadata }
Reflect.getMetadata<ClassSetterDecorator, T>(): { [name]: ClassSetterMetadata }
Reflect.getMetadata<ClassOperatorDecorator, T>(): { [op]: ClassOperatorMetadata }
Reflect.getMetadata<EnumEnumeratorDecorator, T>(): { [name]: EnumEnumeratorMetadata }

// Filtered by decorator
Reflect.getMetadata<ClassFieldDecorator, T, decoratorName>(): { [name]: ClassFieldMetadata }
Reflect.getMetadata<ClassMethodDecorator, T, decoratorName>(): { [name]: ClassMethodMetadata }

// Single target
Reflect.getMetadata<ClassFieldDecorator, T, 'propertyName' or Symbol instance>(): ClassFieldMetadata

// Parameter metadata (keyed by index or name)
Reflect.getMetadata<ClassMethodParameterDecorator, T, 'constructor'>(): { [index]: ClassMethodParameterMetadata }
Reflect.getMetadata<FunctionParameterDecorator, typeof getUser>(): { [index]: FunctionParameterMetadata }

// Object instances (per-instance metadata)
Reflect.getMetadata<ObjectFieldDecorator>(instance): { [key]: ObjectFieldMetadata }
```

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
	metadata: FieldMetadata;
	addInitializer(initializer: () => void): void;
}
```

<details>
	<summary>Expand for example</summary>
	
```js
const metadataKey = Symbol('log');
function logField<T, TClass>({ classContext: { name: className, metadata: classMetadata } name, type, static, private, metadata }: ClassFieldDecorator<T, TClass>) {
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

### ClassMethodDecorator
```js
interface ClassMethodDecorator<T extends (...args:[].<any>) => any, TClass> {
	classContext: ClassDecorator<TClass>;
	type: T;
	name: string | symbol;
	metadata: MethodMetadata;
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
	index: number;
	metadata: ClassMethodParameterMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ClassOperatorDecorator
```js
enum Operator: Symbol {
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
	type: (...args:[].<any>) => any;
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
	index: number;
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
interface ParameterDecorator<T, TMethod> {
	functionContext: FunctionDecorator<TMethod>;
	type: T;
	key: string;
	index: number;
	metadata: Metadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### LetDecorator
```js
interface LetDecorator<T> {
	type: T;
	name: string;
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
	metadata: Metadata;
	addInitializer(initializer: () => void): void;
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
	metadata: Metadata;
	addInitializer(initializer: () => void): void;
}
```

<details>
	<summary>Expand for example</summary>

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
interface ParameterDecorator<T, TMethod, TObject> {
	objectContext: ObjectMethodDecorator<TMethod, TObject>;
	type: T;
	key: string | symbol;
	index: number;
	metadata: ObjectMethodParameterMetadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### Block Decorators

Note: That ```Expression``` is not defined here. Macro AST is out of scope.

WIP: Probably remove the ```insertBefore```, ```insertAfter```, etc. Almost need a mock AST object with non-controversial operations like ```wrap```. I know macro stuff is largely out of scope, but I don't want to make a design that makes that awkward later.

```js
interface BlockDecorator {
	label?: string;
	insertBefore(callback: () => void): void;
	insertAfter(callback: () => void): void;
	wrap(wrapper: (body: () => BlockCompletion) => BlockCompletion): void;
}

interface IfBlockDecorator extends BlockDecorator {
	condition: Expression; // the test expression, for static analysis
}

interface IfElseBlockDecorator extends BlockDecorator {
	condition: Expression; // the test expression, for static analysis
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

### InitializerDecorator
```js
interface InitializerDecorator<T> {
	target: Object;
	type: () => void; // Function that initializes something
}
```

<details>
	<summary>Expand for example</summary>

</details>

### ReturnDecorator
```js
interface ReturnDecorator<T> {
	target: Object;
	propertyKey: string | symbol;
	type: T;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### EnumDecorator
```js
interface EnumDecorator<T> {
	type: T;
	metadata: Metadata;
}
```

<details>
	<summary>Expand for example</summary>

</details>

### EnumEnumeratorDecorator
```js
interface EnumEnumeratorDecorator<T, TEnum> {
	enumContext: EnumDecorator<TEnum>;
	type: T;
	name: string;
	index: number;
	metadata: Metadata;
}
```

<details>
	<summary>Expand for example</summary>

```js
function label<T, TEnum>({ enumContext: { type: enumType }, type, name, index }: EnumEnumeratorDecorator<T, TEnum>) {
	// TODO
}

enum Count {
	@label('Setting 1')
	Setting1,
	@label('Setting 2')
	Setting2
};
```

```js
enum Count: string {
	@label('None')
	Zero = (index, name) => name.toLowerCase(),
	@label('1 item')
	One,
	@label('2 items')
	Two
}
```
</details>

### TupleDecorator

WIP: Bring inline with composites proposal

```js
interface TupleDecorator<T extends any[]> {
	target: Object;
	type: T;
}
```

### RecordDecorator

WIP: Bring inline with composites proposal

```js
interface RecordDecorator<K extends string | number | symbol, V> {
	target: Object;
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

interface NameAndValidator<T> {
	name: keyof T,
	validator: (value: T) => boolean
};

partial class ClassFieldMetadata {
	[validatorsSymbol]: [].<NameAndValidator<T>> = [];
}

function addValidators<T>(({ name, classContext: { metadata } }: ClassFieldDecorator<string, T>), validator: (value: T) => boolean) {
	metadata[validatorsSymbol]).push({ name, validator });
}

function Length<TClass>(min: uint32, max: uint32, context: ClassFieldDecorator<string, TClass>) { // Can only be placed on string
	addValidators(context, (value: string) => value.length is >= min and <= max);
}
function Includes<TClass>(searchString: string, context: ClassFieldDecorator<string, TClass>) {
	addValidators(context, (value: string) => value.includes(searchString) });
}
function Min<T extends int, TClass>(min: T, context: ClassFieldDecorator<T, TClass>) {
	addValidators(context, (value: T) => value >= min });
}
function Max<T extends int, TClass>(max: T, context: ClassFieldDecorator<T, TClass>) {
	addValidators(context, (value: T) => value <= max });
}
function IsEmail<TClass>(context: ClassFieldDecorator<T, TClass>) {
	addValidators(context, (value: string) => value.includes('@') }); // :)
}
// ...

function validate<T>(o: T): boolean {
	return (Reflect.getMetadata<T>(validatorsSymbol) as NameAndValidator[]).every(({ name, validator }) => validator(o[name]));
}

class Post {
	@Length(10, 20)
	title: string;

	@Contains('hello')
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

```js
const binary = Symbol('binary');

function data(context: ClassFieldDecorator<boolean, any>) {
	context.metadata[binary] = (packet, value) => packet.write<boolean>(value);
}

function data<N: uint32>(context: ClassFieldDecorator<uint<N>, any>) {
	context.metadata[binary] = (packet, value) => packet.write<uint<N>>(value);
}
function data<LengthType extends uint = uint16>(context: ClassFieldDecorator<string, any>) {
	context.metadata[binary] = (packet, value: string) => packet.write<string, LengthType>(value);
}

@data
class Room {
	id: uint64;
	@data(255)
	name: string;
	@data // Implicit generic uint<n>
	test: uint<7>;
	@data
	currentlyBooked: boolean;
}

@data
class Building {
	rooms: [].<Room>;
}

function binarySerialize<T>(buffer: Packet, item: T) {
	const fields = Reflect.context<T,
	for (const key of fields) {
	
	}
}
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

type InjectionSite = {
    method: string | symbol,
    index: uint32,
    token: string | symbol
};

partial class ClassMetadata {
    [injectKey]: [].<InjectionSite> = [];
}

function inject<T, TMethod, TClass>(
    token: string | symbol,
    { key, index, methodContext: { classContext: { metadata } } }: ClassMethodParameterDecorator<T, TMethod, TClass>
) {
    metadata[injectKey].push({ method: key, index, token });
}

function resolve<T>(cls: { new(...args: any): T }, container: Container): T {
    const sites = Reflect.getMetadata<T>()[injectKey];
    const ctorArgs = sites
        .filter(s => s.method == 'constructor')
        .sort((a, b) => a.index - b.index)
        .map(s => container.get(s.token));
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
class Room {
	id: uint64;
	@validate(/[A-Za-z ]{1, 80}/)
	name: string;
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
```
