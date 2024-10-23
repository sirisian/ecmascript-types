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
// ClassOperatorParameterDecorator ?
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

Some contexts have metadata which is on the type essentially for classes. (This can be thought of as on the constructor). The same is true for enum types. For objects the metadata is on the instance. So each instance created has its own unique metadata that is not shared.

```js
type Metadata = Record<string | number | symbol | ???, unknown>;
```

TODO: I want Metadata applied to types to be typed. Essentially I need a way for code to broadcast that they've registered a variable. IDEs would be able to see this.
```js
@Metadata<T, { f: string }>
function f<T>(context: ClassDecorator<T>) {
  context.metadata.f = 'f';
}
@f
class A {}

const metadata: Metadata<A> = getMetadata<A>();
metadata.f;
```
Obviously this would breakdown if a user dynamically generated a class.

This would need to also work on properties.

```js
@Metadata<TClass, { [context.name]: T }, { f: string }>
function f<T, TClass>(context: ClassFieldDecorator<T, TClass>) {
  context.metadata.f = 'f';
}
class A {
  @f
  a: uint8;
}

const metadata: Metadata<A, { a }> = getMetadata<A, { a }>();
metadata.f;
```

The big picture of having these be typed is they could be treated as constant immutable records. If they were truly constant the values could be used with generics. An engine would also be free to optimize using the data.

This might be possible independently if the programmer is diligent about utilizing records and tuples to store things. In common serialization scenarios the tuples would be iterated over and a very ambitious engine could customize the code path expanding the loop and inlining everything.

## Decorator Contexts

Overloading parameter types and contexts allow defining specialized decorators for every situation.

### ClassDecorator
```js
interface ClassDecorator<T extends { new (...args: [].<any>): any }> {
  name: string | undefined;
  type: Function;
  metadata: Metadata;
  addInitializer(initializer: () => void): void;
}
```

### ClassFieldDecorator
```js
interface ClassFieldDecorator<T, TClass> {
  type: T;
  classType: TClass;
  classContext: ClassDecorator<TClass>;
  name: string | symbol;
  static: boolean;
  private: boolean;
  metadata: Metadata;
  initial: T | undefined; // If a constant exists, then it'll be stored here
  addInitializer(initializer: () => void): void;
}
```

```js
const metadataKey = Symbol('log');
function logField<T, TClass>({ target, name, type, static, private, metadata }: ClassFieldDecorator<T, TClass>) {
  console.log('name:', name);
  console.log('class:', targetContext.name);
  console.log('type:', type);
  console.log('static:', static);
  console.log('private:', private);
  // Similar TypeScript syntax: Reflect.defineMetadata(metadataKey, name, target.constructor, name);
  metadata[metadataKey] = name;
}
```

### ClassGetterDecorator
```js
interface ClassGetterDecorator<T, TClass> {
  type: () => T;
  classType: TClass;
  classContext: ClassDecorator<TClass>;
  name: string | symbol;
  metadata: Metadata;
  addInitializer(initializer: () => void): void;
}
```

### ClassSetterDecorator
```js
interface ClassSetterDecorator<T, TClass> {
  type: (value: T) => void;
  classType: TClass;
  classContext: ClassDecorator<TClass>;
  name: string | symbol;
  metadata: Metadata;
  addInitializer(initializer: () => void): void;
}
```

### ClassMethodDecorator
```js
interface ClassMethodDecorator<T extends (...args:[].<any>) => any, TClass> {
  type: T;
  classType: TClass;
  classContext: ClassDecorator<TClass>;
  name: string | symbol;
  metadata: Metadata;
}
```

### ClassMethodParameterDecorator
```js
interface ParameterDecorator<T, TMethod, TClass> {
  type: T;
  methodType: TMethod;
  methodContext: ClassMethodDecorator<TMethod, TClass>
  key: string | symbol;
  index: number;
}
```

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

interface ClassOperatorDecorator {
  operator: Operator;
  type: (...args:[].<any>) => any;
  metadata: Metadata;
}
```

```ClassOperatorParameterDecorator``` ?

### FunctionDecorator
```js
interface FunctionDecorator<T extends (...args:[].<any>) => any> {
  type: T;
  name: string | symbol | undefined;
  metadata: Metadata;
}
```

### FunctionParameterDecorator
```js
interface ParameterDecorator<T, TMethod> {
  type: T;
  methodType: TMethod;
  methodContext: FunctionDecorator<TMethod>;
  key: string | symbol;
  index: number;
  metadata: Metadata;
}
```

### LetDecorator
```js
interface LetDecorator<T> {
  type: T;
  name: string;
}
```

### ConstDecorator
```js
interface ConstDecorator<T> {
  type: T;
  name: string;
}
```

### ObjectDecorator
```js
interface ObjectDecorator<T> {
  type: T;
  metadata: Metadata; // on the instance
}
```

```js
function f<T>(context: ObjectDecorator<T>) {
  // ???
}

const a = @f {
  (b: number): 10
};
```

### ObjectFieldDecorator
```js
interface ObjectFieldDecorator<T, TObject> {
  type: T;
  objectType: TObject;
  objectContext: ObjectDecorator<TObject>;
  key: string | symbol;
  metadata: Metadata; // on the instance
}
```

```js
function f<T>(context: ObjectFieldDecorator<T, any>) {
  // ???
}

const a = {
  @f
  (b: number): 10
};
```

### ObjectGetterDecorator
```js
interface ObjectGetterDecorator<T, TObject> {
  type: () => T;
  objectType: TObject;
  objectContext: ObjectDecorator<TObject>;
  name: string | symbol;
  metadata: Metadata;
  addInitializer(initializer: () => void): void;
}
```

### ObjectSetterDecorator
```js
interface ObjectSetterDecorator<T, TObject> {
  type: (value: T) => void;
  objectType: TObject;
  objectContext: ObjectDecorator<TObject>;
  name: string | symbol;
  metadata: Metadata;
  addInitializer(initializer: () => void): void;
}
```

### ObjectMethodDecorator
```js
interface ObjectMethodDecorator<T extends (...args:[].<any>) => any, TObject> {
  type: T;
  objectType: TObject;
  objectContext: ObjectDecorator<TObject>;
  key: string | symbol;
  metadata: Metadata; // on the instance
}
```

### ObjectMethodParameterDecorator
```js
interface ParameterDecorator<T, TMethod, TObject> {
  type: T;
  objectType: TMethod;
  objectContext: ObjectMethodDecorator<TMethod, TObject>
  key: string | symbol;
  index: number;
}
```

### BlockDecorator
```js
interface BlockDecorator {
  label?: string; // ???
  insertBefore // ???
  insertAfter // ???
  wrap // ???
}
```

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

### InitializerDecorator
```js
interface InitializerDecorator<T> {
  target: Object;
  type: () => void;  // Function that initializes something
}
```

### ReturnDecorator
```js
interface ReturnDecorator<T> {
  target: Object;
  propertyKey: string | symbol;
  type: T;
}
```

### EnumDecorator
```js
interface EnumDecorator<T> {
  type: T;
  metadata: Metadata;
}
```

### EnumEnumeratorDecorator
```js
interface EnumEnumeratorDecorator<T, TEnum> {
  type: T;
  enumType: TEnum;
  enumContext: EnumDecorator<TEnum>;
  name: string;
  index: number;
  metadata: Metadata;
}
```

```js
function label<T, TEnum>({ enumType, type, name, index }: EnumEnumeratorDecorator<T, TEnum>) {
  // Should I have the ability to reflect on T? reflectMetadata<T>
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

### TupleDecorator
```js
interface TupleDecorator<T extends any[]> {
  target: Object;
  type: T;
}
```

### RecordDecorator
```js
interface RecordDecorator<K extends string | number | symbol, V> {
  target: Object;
  type: Record<K, V>;
}
```

## Examples

WIP, this should be an exhaustive list of common decorator uses including metadata uses.

### Validation

An existing TypeScript library for reference: https://github.com/typestack/class-validator

A naive example below that assumes we only want to run a validation for the whole object.

```js
const validatorsSymbol = Symbol('validators');

type NameAndValidator = { name: keyof T, validator: (value:T) => boolean };

function addValidators({ name, classContext: { metadata } }, validator) {
  (metadata[validatorsSymbol] ??= [].<NameAndValidator>).push({ name, validator });
}

function Length<TClass>(min: uint32, max: uint32, context: ClassFieldDecorator<string, TClass>) { // Can only be placed on string
  addValidators(context, value: string => value.length is >= min and <= max);
}
function Includes<TClass>(searchString: string, context: ClassFieldDecorator<string, TClass>) {
  addValidators(context, value: string => value.includes(searchString) });
}
function Min<T extends int, TClass>(min: T, context: ClassFieldDecorator<T, TClass>) {
  addValidators(context, value: T => value >= min });
}
function Max<T extends int, TClass>(max: T, context: ClassFieldDecorator<T, TClass>) {
  addValidators(context, value: T => value <= max });
}
function IsEmail<TClass>(context: ClassFieldDecorator<T, TClass>) {
  addValidators(context, value: string => value.includes('@') }); // :)
}
// ...

function validate<T>(o: T) {
  return (Reflect.getMetadata<T>(validatorsSymbol) as NameAndValidator[]).every(({ name, validator }) => validator(o[name]));
}

export class Post {
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
```

### Serialization

const binary = Symbol('binary');

```js
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
