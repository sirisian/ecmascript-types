# Invoicing

Money, dates, and invariants: the domain that made Java's ```BigDecimal```, C#'s ```decimal```, and every ORM's validation layer. An invoice is a bundle of cross-field rules - totals must equal their lines, due dates follow issue dates, paid invoices don't mutate - crossing JSON boundaries in both directions. This example builds it from [primitive metadata](../primitivemetadata.md) (a ```Currency``` meta type in the mold of ```Dimensions```), [dependent record types](../dependentrecordtypes.md) (the invariants), [Temporal](../temporal.md) (the dates), [Serialization](../serialization.md) (the boundaries), and the enum machinery in the main proposal (the status workflow).

Features exercised:

- A user-defined meta type: ```Currency``` claims one metadata field and makes mixing currencies a compile error, demonstrating that ```Dimensions``` is an instance of a pattern, not a special case. Deliberately, it defines no ```conversionFactor```: unlike kilometers to meters, dollars never convert to euros implicitly.
- ```decimal128``` as the money representation, parsed exactly from JSON digits with no float64 round trip.
- Two ```where``` clauses on one type, one relating Temporal dates and one summing lines with a typed ```reduce```.
- A string enum for status with the exhaustive ```switch``` rule catching unhandled transitions.
- ```Map.groupBy``` and dimensionless aggregation from the [standard library](../standardlibrary.md).

## Currency as a Meta Type

```js
type Currency = {
	currency: string, // ISO 4217 code; '' is the default, meaning unconstrained
};

meta Currency {
	default = { currency: '' };

	// A constrained amount is assignable where an unconstrained one is expected.
	// Two different constraints never are: no conversionFactor is defined, so
	// there is no implicit path between currencies, unlike Dimensions' ratios.
	subtype(sub: Currency, sup: Currency): boolean {
		return sup.currency == '' || sub.currency == sup.currency;
	}

	describe(constraint: Currency): string {
		return constraint.currency || 'currency-less';
	}
}

primitive decimal128<C: Currency> {
	// Same-currency arithmetic. The parameter reuses C, so a mismatched
	// currency fails subtype() at the argument boundary: a compile error
	// when types are known, a TypeError when dynamic.
	operator+(rhs: decimal128.<C>): decimal128.<C> {
		return this + rhs;
	}
	operator-(rhs: decimal128.<C>): decimal128.<C> {
		return this - rhs;
	}
	// Scaling by a dimensionless quantity preserves the currency.
	operator*(rhs: decimal128): decimal128.<C> {
		return this * rhs;
	}
}

type USD = decimal128.<{ currency: 'USD' }>;
type EUR = decimal128.<{ currency: 'EUR' }>;
```

```js
const a: USD = 19.99;
const b: USD = 5.01;
a + b; // USD 25.00, exact decimal arithmetic
// const c: EUR = a; // TypeError: Currency.subtype({ currency: 'USD' }, { currency: 'EUR' }) failed
// a + EUR(1); // TypeError: no implicit path between currencies

// Conversion is an explicit function that takes the rate; the currencies are
// value generics, so both ends are named at the call site.
function convert<From: string, To: string>(
	amount: decimal128.<{ currency: From }>,
	rate: decimal128
): decimal128.<{ currency: To }> {
	return decimal128.<{ currency: To }>(amount * rate);
}
const c = convert.<'USD', 'EUR'>(a, 0.86);
```

## The Invoice

```js
enum Status: string { Draft = 'draft', Sent = 'sent', Paid = 'paid', Void = 'void' };

type LineItem = {
	description: string.<{ minLength: 1, maxLength: 200 }>,
	quantity: uint32.<{ minimum: 1 }>,
	unitPrice: USD
};

type Invoice = {
	id: uint64,
	status: Status,
	issued: Temporal.PlainDate,
	due: Temporal.PlainDate,
	lines: [].<LineItem>.<{ minLength: 1 }>,
	total: USD
}
where Temporal.PlainDate.compare(this.issued, this.due) <= 0
where this.total == this.lines.reduce((sum: USD, line) => sum + line.unitPrice * line.quantity, USD(0));
```

The clauses are checked at the boundaries the dependent record types document defines - construction, function calls, assignment - so an invoice whose total drifted from its lines cannot cross one:

```js
const invoice: Invoice = {
	id: 1,
	status: Status.Draft,
	issued: Temporal.PlainDate.from('2026-07-01'),
	due: Temporal.PlainDate.from('2026-07-31'),
	lines: [
		{ description: 'Consulting', quantity: 10, unitPrice: 150.00 },
		{ description: 'Hosting', quantity: 1, unitPrice: 25.50 }
	],
	total: 1525.50
};
// total: 1525.49 // TypeError at construction: the second where clause failed
```

## The Boundary

Both directions in one pass each. Parsing validates the string constraints, the currency-typed decimals (exact from the digits), the Temporal ISO strings through their cast operators, the enum membership of ```status```, and finally the ```where``` clauses:

```js
const invoice = JSON.parse.<Invoice>(body);
// Malformed JSON: SyntaxError. Well-formed but wrong: TypeError naming the path,
// e.g. "at .lines[0].quantity: expected uint32 where minimum: 1, got 0"

JSON.stringify(invoice);
// decimal128 emits exact digits, Temporal emits ISO strings via toJSON,
// Status emits its underlying string
```

## The Workflow

```js
enum Event: string { Send = 'send', Pay = 'pay', Cancel = 'cancel' };

function transition(status: Status, event: Event): Status {
	switch (status) {
		case Status.Draft:
			switch (event) {
				case Event.Send: return Status.Sent;
				case Event.Cancel: return Status.Void;
				case Event.Pay: throw new TypeError('Cannot pay a draft');
			} // Exhaustive over Event: no default required
		case Status.Sent:
			switch (event) {
				case Event.Pay: return Status.Paid;
				case Event.Cancel: return Status.Void;
				case Event.Send: return Status.Sent; // Resending is idempotent
			}
		case Status.Paid:
		case Status.Void:
			throw new TypeError(`${Status.toString(status)} is terminal`);
	} // Exhaustive over Status: adding an enumerator breaks this switch loudly
}
```

## Reporting

```js
function outstanding(invoices: [].<Invoice>, today: Temporal.PlainDate): Map.<Status, USD> {
	const byStatus = Map.groupBy(invoices, invoice => invoice.status);
	const totals = new Map.<Status, USD>();
	for (const [status, group] of byStatus) {
		totals.set(status, group.reduce((sum: USD, invoice) => sum + invoice.total, USD(0)));
	}
	return totals;
}

const overdue = invoices
	.filter(invoice => invoice.status == Status.Sent
		&& Temporal.PlainDate.compare(invoice.due, today) < 0);
```

## Coverage Notes

- **```readonly``` doesn't exist.** A paid invoice should be immutable, and typed languages express that per field (```readonly``` in TypeScript and C#, ```final``` in Java). The proposal has ```const``` bindings and the dependent-field rule (fields named in ```where``` clauses check at boundaries), but no way to say a field never changes after construction. This came up on every type in this example and is the clearest missing modifier.
- **Rounding is unspecified.** ```unitPrice * quantity``` on ```decimal128``` is exact here, but ```convert``` multiplies by a rate and real money code must then round to the currency's minor unit under a named mode (half-even, usually). The type list includes the decimal types without defining their rounding control; either operations take a context, or a metadata field carries scale-and-mode the way ```minimum``` carries bounds. Until decided, financial code can't actually be written against this.
- **Metadata on array types.** ```lines: [].<LineItem>.<{ minLength: 1 }>``` is this example's invention: the metadata application syntax attached to an array type, wanting an ```ArrayBounds``` meta type parallel to ```StringBounds```. Nothing defines whether ```.<>``` chains like that or whether array types accept metadata at all. Either bless it or the constraint moves into a ```where``` clause.
- **String-valued generic parameters.** ```convert<From: string, To: string>``` binds value generics of type ```string``` and uses them inside metadata objects. Value generics are established with integers (```Size: uint32``` throughout); strings are an extrapolation the generics document should confirm, since currency-style meta types depend on it.
- **Nested exhaustive switches fall through.** ```case Status.Draft:``` ends in an exhaustive inner switch whose every arm returns or throws, so control never reaches the next case - but the outer ```switch``` has no ```break```, and nothing in the control structures section says an exhaustive-and-diverging inner switch satisfies the outer case. TypeScript solves the class of problem with ```never```; here it likely wants a rule that a case body whose control provably diverges needs no ```break```.
- **Enum values in template literals.** Same note as the parser example: ```${status}``` printing ```paid``` directly instead of requiring ```Status.toString(status)``` in error messages.
