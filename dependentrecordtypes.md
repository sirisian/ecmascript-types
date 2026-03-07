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
			"default": "United States of America",
			"enum": ["United States of America", "Canada"]
		}
	},
	"if": {
		"properties": {
			"country": { "const": "United States of America" }
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
type USZip = string<{ pattern: /^[0-9]{5}(-[0-9]{4})?$/ }>;
type CAZip = string<{ pattern: /^[A-Z]\d[A-Z] \d[A-Z]\d$/ }>;
type Address = {
    streetAddress: string,
    country: 'United States of America' | 'Canada',
} where if (this.country == 'United States of America') {
    this is { postalCode: USZip }
} else {
    this is { postalCode: CAZip }
};
```

In the ```Address``` code it should be noted that this design means developers don't have to use discriminated unions. For reference:

```js
type Address =
	| { streetAddress: string, country: 'United States of America', postalCode: USZip }
	| { streetAddress: string, country: 'Canada', postalCode: CAZip }
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
const addr: Address = { streetAddress: '123 Main', country: 'Canada', postalCode: 'M4W 3R8' };

if (addr.country == 'United States of America') {
    // Compiler narrows addr.postalCode to USZip
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

#### Possible Solution

Mark fields in the ```where``` clauses that cannot be updated independently.

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

For marked fields check at boundaries, like construction and function calls.

```js
const payment: Payment = { name: 'Alice' }; // Checked, construction
payment.creditCard = 4111111111111111; // Not checked
payment.billingAddress = '123 Main St'; // Not checked
validate(payment); // Checked, function call
const payment2: Payment = payment; // Checked, object assignment
```

This allows an object to be put into an invalid state momentarily and the compiler ensures it doesn't stay in an invalid state. If this cannot be determined at compile-time however it up to the user to add runtime checks:

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
    this.name is string<{ pattern: /^[a-z]+$/ }> // lowercase only
} else {
    this.name is string<{ pattern: /^[a-zA-Z]+$/ }> // mixed case
};

const user: User = { mode: 'strict', name: 'alice' }; // Pretend 'alice' is a dynamic value
// Compiler's knows: 
// user.mode is 'strict'
// user.name is string<{ pattern: /^[a-z]+$/ }>

user.mode = 'loose';

example(user); // throws as the compiler cannot infer that user is valid
```

If the compiler was smart then it would know that the 'loose' regex is a subset of the 'strict' regex. It's unlikely such a subset operation would be added (or implemented by a user in the metadata subtype operation at least in general) so the compiler would require a runtime check.

```js
if (user is User) {
	example(user);
}
```

Note: There's a comment about 'alice' being a dynamic value because a hardcoded literal would propagate and validate. That is the compiler would be totally fine with the above example and correctly determine that no runtime check is required.

## Examples

