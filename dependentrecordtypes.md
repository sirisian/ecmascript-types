# Dependent Record Types

In addition to [primitive metadata](https://github.com/sirisian/ecmascript-types/blob/master/primitivemetadata.md) it's possible to have minimal syntax to define dependent record types and further improve refinement typing. Taken from JSON Schema the goal is to support dependentRequired, dependentSchemas, and if/then/else. Consider the following examples:

```js
// dependentRequired
{
	"type": "object",
	"properties": {
		"name": { "type": "string" },
		"creditCard": { "type": "number" },
		"billingAddress": { "type": "string" }
	},
	"required": ["name"],
	"dependentRequired": {
		"creditCard": ["billingAddress"],
		"billingAddress": ["creditCard"]
	}
}
// dependentSchemas
{
	"type": "object",
	"properties": {
		"name": { "type": "string" },
		"creditCard": { "type": "number" }
	},
	"required": ["name"],
	"dependentSchemas": {
		"creditCard": {
			"properties": {
				"billingAddress": { "type": "string" }
			},
			"required": ["billingAddress"]
		}
	}
}
// if/then/else:
{
	"type": "object",
	"properties": {
		"streetAddress": {
			"type": "string"
		},
		"country": {
			"default": "US",
			"enum": ["US", "CA"]
		}
	},
	"if": {
		"properties": {
			"country": { "const": "US" }
		}
	},
	"then": {
		"properties": {
			"postalCode": { "pattern": "[0-9]{5}(-[0-9]{4})?" }
		}
	},
	"else": {
		"properties": {
			"postalCode": { "pattern": "[A-Z][0-9][A-Z] [0-9][A-Z][0-9]" }
		}
	}
}
```

The proposed syntax is to use ```where``` and ```is``` to define these dependencies.

```js
// dependentRequired
type Payment = {
	name: string,
	creditCard?: number,
	billingAddress?: string,
} where (this.creditCard != null) == (this.billingAddress != null);

// dependentSchemas
type PaymentWithSchema = {
	name: string,
	creditCard?: number,
	billingAddress?: string,
} where if (this.creditCard != null) {
	this.billingAddress is string
	// or
	//this is { billingAddress: string }
};

// if/then/else
type USPostalCode = string.<{ pattern: /^[0-9]{5}(-[0-9]{4})?$/ }>;
type CAPostalCode = string.<{ pattern: /^[A-Z]\d[A-Z] \d[A-Z]\d$/ }>;
type Address = {
	streetAddress: string,
	country: 'US' | 'CA',
} where if (this.country == 'US') {
	this is { postalCode: USPostalCode }
} else {
	this is { postalCode: CAPostalCode }
};
```

In the ```Address``` code it should be noted that this design means developers don't have to use discriminated unions. For reference:

```js
type Address =
	| { streetAddress: string, country: 'US', postalCode: USPostalCode }
	| { streetAddress: string, country: 'CA', postalCode: CAPostalCode }
```

Also to be clear multiple ```where``` clauses can be used.

```js
type ... = {
}
where if (...) {} else {}
where if (...) {} else {};
```

### Narrowing

```js
const addr: Address = { streetAddress: '123 Main', country: 'CA', postalCode: 'M4W 3R8' };

if (addr.country == 'US') {
	// Compiler narrows addr.postalCode to USPostalCode
}
```

### Assignment design choices

In the previous example for ```Payment``` setting either ```creditCard``` or ```billingAddress``` would place an instance into an invalid state. (Both need to be defined).

```js
type Payment = {
	name: string,
	creditCard?: number,
	billingAddress?: string
} where (this.creditCard != null) == (this.billingAddress != null);

const payment: Payment = {
	name: 'Alice'
}; // Valid: Neither creditCard nor billingAddress are present.

payment.name = 'Bob'; // Valid: name has no constraints
```

#### Solution

Fields referenced in a ```where``` clause are dependent: they cannot be validated on independent assignment, so the clause is checked at boundaries instead - construction, function calls, and object assignment. This is the rule the examples in this document assume.

The spread operator can be used to set a fully valid object:
```js
const updatedPayment: Payment = {
	...payment,
	creditCard: 4111222233334444,
	billingAddress: '123 Main St'
}; // Valid: Both fields are provided. The where clause evaluates to true.

const invalidPayment: Payment = {
	...payment,
	creditCard: 1234123412341234
}; // Compile or runtime error: 'where' clause validation failed at construction. (In this case it would be a compile time error).
```

Dependent fields are checked at boundaries:

```js
const payment: Payment = { name: 'Alice' }; // Checked, construction
payment.creditCard = 4111111111111111; // Not checked
payment.billingAddress = '123 Main St'; // Not checked
validate(payment); // Checked, function call
const payment2: Payment = payment; // Checked, object assignment
```

This allows an object to be put into an invalid state momentarily while the compiler ensures it doesn't stay invalid across a boundary. When validity cannot be determined at compile time, a runtime check of the ```where``` clause is inserted at the boundary and throws on failure. To branch on validity instead of throwing, test explicitly:

```js
if (payment is Payment) { 
	example(payment);
}
```

#### Limitations

Consider the following code that allows a name to be a 'strict' or 'loose' mode.

```js
type User = {
	mode: 'strict' | 'loose',
	name: string
} where if (this.mode == 'strict') {
	this.name is string.<{ pattern: /^[a-z]+$/ }> // lowercase only
} else {
	this.name is string.<{ pattern: /^[a-zA-Z]+$/ }> // mixed case
};

const user: User = { mode: 'strict', name: 'alice' }; // Pretend 'alice' is a dynamic value
// Compiler knows:
// user.mode is 'strict'
// user.name is string.<{ pattern: /^[a-z]+$/ }>

user.mode = 'loose';

example(user);
// The compiler cannot prove the where clause still holds, so a runtime
// check is inserted at the call boundary. It passes here since 'alice'
// matches the loose pattern, but the check cannot be elided.
```

If the compiler were smart it would know that strings matching the 'strict' regex are a subset of those matching the 'loose' regex and elide the check. It's unlikely such a subset operation would be added (or implemented by a user in the metadata subtype operation, at least in general), so the compiler falls back to the inserted runtime check. To branch on validity instead:

```js
if (user is User) {
	example(user);
}
```

Note: There's a comment about 'alice' being a dynamic value because a hardcoded literal would propagate and validate. That is, the compiler would be totally fine with the above example and correctly determine that no runtime check is required.

## Examples

### JSON Serialization

A native, fused version of this boundary (parsing, validation, and typed allocation in one pass) is specified in [Serialization](serialization.md). The reflective pattern below remains the way to express wire-name mappings and custom policies.

```js
// String constraints like pattern/minLength/maxLength come from the
// StringBounds meta type defined in the primitive metadata proposal.

const schemaKey = Symbol('schema');

type SerializeData = {
	name: string,
	wireName: string
};

partial class ClassMetadata {
	[schemaKey]: [].<SerializeData> = [];
}

// @field() registers a field for serialization with an optional wire name
function field<T, TClass>(
	{ name, metadata }: Reflect.ClassField.<T, TClass>
) where typeof name == 'string' {
	metadata[schemaKey].push({ name, wireName: name });
}
function field<T, TClass>(
	wireName: string,
	{ name, metadata }: Reflect.ClassField.<T, TClass>
) where typeof name == 'string' {
	metadata[schemaKey].push({ name, wireName });
}

function serialize<T>(instance: T): { [key: string]: any } {
	const result: { [key: string]: any } = {};
	for (const { name, wireName } of Reflect.getMetadata.<Reflect.Class, T>()[schemaKey]) {
		result[wireName] = instance[name];
	}
	return result;
}

function deserialize<T>(cls: { new(): T }, data: { [key: string]: any }): T {
	const instance = new cls();
	
	// 1. Mutation phase: Populate the fields.
	// Scalar bounds (like string or union types) are checked immediately on assignment.
	// However, cross-field 'where' clauses are not evaluated yet, allowing us to assign fields in any order without triggering false validation errors.
	for (const { name, wireName } of Reflect.getMetadata.<Reflect.Class, T>()[schemaKey]) {
		instance[name] = data[wireName];
	}

	// 2. Boundary phase: Now that the object is fully populated, we assert the cross-field invariants. The `is` operator evaluates the `where` clause.
	if (!(instance is T)) {
		throw new TypeError(`Cross-field validation failed for ${cls.name}`);
	}

	return instance;
}

class AddressResponse {
	@field('street_address')
	streetAddress: string;

	@field
	country: 'US' | 'CA';

	// The base type is just string
	@field('postal_code')
	postalCode: string;
} where if (this.country == 'US') {
	this.postalCode is USPostalCode
} else {
	this.postalCode is CAPostalCode
};
```

This does raise a question though about decorators usage in dependent record types. In the above usage all decorator usage with ```postalCode``` are identical. I mention this to say this isn't allowed:

```js
class AddressResponse {
	@field('street_address')
	streetAddress: string;

	@field
	country: 'US' | 'CA';
} where if (this.country == 'US') {
	this is { @field('zip_code') postalCode: USPostalCode };
} else {
	this is { @field('postal_code') postalCode: CAPostalCode }
};
```

Instead this form would be used, as decorators are only allowed in the class body:

```js
class AddressResponse {
	@field('street_address') streetAddress: string;
	@field country: 'US' | 'CA';

	@field('zip_code') usZip?: USPostalCode;
	@field('postal_code') caZip?: CAPostalCode;
} where if (this.country == 'US') {
	this.usZip !== undefined && this.caZip === undefined;
} else {
	this.caZip !== undefined && this.usZip === undefined;
};
```

### Network Messages

Showing ```where match``` syntax. The ```match```/```when``` forms follow the [pattern matching proposal](https://github.com/tc39/proposal-pattern-matching); ```where match``` inherits its semantics.

```js
type NetworkState = {
	status: 'idle' | 'loading' | 'success' | 'error',
	data?: any,
	errorMessage?: string
} where match (this.status) {
	when 'idle' | 'loading': 
		this.data === undefined && this.errorMessage === undefined;
	when 'success': 
		this.data !== undefined && this.errorMessage === undefined;
	when 'error': 
		this.errorMessage !== undefined && this.data === undefined;
};

function renderUI(state: NetworkState) {
	match (state) {
		when { status: 'success' }:
			renderData(state.data); // state.data cannot be undefined
		when { status: 'error' }: 
			renderError(state.errorMessage); // state.errorMessage cannot be undefined
		when { status: 'idle' | 'loading' }:
			renderSpinner();
	}
}
```

### Business Logic

```js
type DatabaseCommand = {
	userRole: 'admin' | 'editor' | 'viewer',
	action: 'insert' | 'update' | 'delete' | 'read',
	targetTable: string
} where match (this.userRole) {
	when 'viewer': 
		this.action === 'read';
	when 'editor': 
		this.action !== 'delete';
	when 'admin': 
		true; // Admins can do anything
};
```

On construction, this would throw if userRole doesn't match the allowed action.

### Misc Example

```js
type Email = {
	to: string.<{ pattern: /@/ }>,
	subject?: string,
	body?: string
} where (this.subject?.length ?? 0) > 0 || (this.body?.length ?? 0) > 0;

// The optional-field draft shape, written out since utility types like
// TypeScript's Partial aren't part of the proposal:
type EmailDraft = {
	to?: string.<{ pattern: /@/ }>,
	subject?: string,
	body?: string
};

function draftEmail(): EmailDraft {
	return {};
}

function sendEmail(ref draft: EmailDraft) {
	if (!(draft is Email)) {
		throw new Error("Invalid email draft");
	}
	// Sending...
}

const msg = draftEmail();
msg.to = "alice@example.com";
// No subject or body

sendEmail(ref msg); // throws
```
