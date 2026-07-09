# Temporal

Temporal's objects are immutable and its arithmetic is explicit, which makes it a natural fit for typing. Most of the work is signatures, but two parts are interesting enough to justify a document. Units become an enumeration instead of a vocabulary of strings, so a misspelling is a compile-time error rather than a RangeError. And a duration is a physical quantity, so it participates in the ```Dimensions``` meta type from [primitive metadata](primitivemetadata.md): dividing a distance by a duration produces a velocity, checked by the compiler and carrying its unit scale.

Temporal objects are reference types. They are frozen, so a typed field holding one needs no defensive copy, and ```Reflect.typeOf``` reports the exact class.

## Units

Every Temporal API that takes a unit takes a string today. As an enumeration the same call sites keep their spelling and gain checking:

```js
enum Unit: string { // Exposed as Temporal.Unit
	Nanosecond = 'nanosecond',
	Microsecond = 'microsecond',
	Millisecond = 'millisecond',
	Second = 'second',
	Minute = 'minute',
	Hour = 'hour',
	Day = 'day',
	Week = 'week',
	Month = 'month',
	Year = 'year'
};

duration.total(Temporal.Unit.Second);
// duration.total('secnod'); // TypeError: not a Temporal.Unit
```

The time units, ```Nanosecond``` through ```Hour```, denote fixed spans. The calendar units, ```Day``` through ```Year```, do not: their length depends on the reference point, which is why Temporal requires ```relativeTo``` for them. That distinction is what makes the dimensioned overloads below sound.

## Signatures

```js
class Temporal.Instant {
	static from(value: string | Temporal.Instant): Temporal.Instant;
	static fromEpochNanoseconds(value: bigint): Temporal.Instant;
	static compare(a: Temporal.Instant, b: Temporal.Instant): int32;

	get epochNanoseconds(): bigint;
	get epochMilliseconds(): int64;

	add(duration: Temporal.Duration): Temporal.Instant;
	subtract(duration: Temporal.Duration): Temporal.Instant;
	since(other: Temporal.Instant): Temporal.Duration;
	until(other: Temporal.Instant): Temporal.Duration;
	equals(other: Temporal.Instant): boolean;
	toZonedDateTimeISO(timeZone: string): Temporal.ZonedDateTime;
	toJSON(): string;
}

class Temporal.Duration {
	years: int32;
	months: int32;
	weeks: int32;
	days: int32;
	hours: int32;
	minutes: int32;
	seconds: int32;
	milliseconds: int32;
	microseconds: int32;
	nanoseconds: int32;

	get sign(): int32;
	get blank(): boolean;

	add(other: Temporal.Duration): Temporal.Duration;
	subtract(other: Temporal.Duration): Temporal.Duration;
	negated(): Temporal.Duration;
	abs(): Temporal.Duration;
}

class Temporal.Now {
	static instant(): Temporal.Instant;
	static zonedDateTimeISO(timeZone?: string): Temporal.ZonedDateTime;
	static plainDateISO(timeZone?: string): Temporal.PlainDate;
}
```

```Temporal.PlainDate```, ```Temporal.PlainTime```, ```Temporal.PlainDateTime```, ```Temporal.ZonedDateTime```, ```Temporal.PlainYearMonth```, and ```Temporal.PlainMonthDay``` follow the same shape: typed integer fields, ```compare``` returning ```int32```, ```since```/```until``` returning ```Temporal.Duration```, and ```equals``` returning ```boolean```.

## Durations as Dimensioned Quantities

A duration measured in a fixed time unit is a quantity with dimension ```{ s: 1 }``` and a ratio giving its scale, exactly like the unit aliases in the primitive metadata document. The ```Dimensions``` operator blocks there are written for ```float32```; the same blocks are declared for ```float64```, which is what nanosecond-scale totals need.

```js
type Second = float64.<{ s: 1 }>;
type Millisecond = float64.<{ s: 1, ratio: 1/1000 }>;
type Nanosecond = float64.<{ s: 1, ratio: 1/1000000000 }>;
type Hour = float64.<{ s: 1, ratio: 3600 }>;
```

```total``` maps a fixed time unit to the matching dimensioned type. The ratio comes from a compile-time function over the enumeration, the same way ```multiplyDimensions``` computes a return type in the primitive metadata document:

```js
function unitRatio(unit: Temporal.Unit): rational {
	switch (unit) {
		case Temporal.Unit.Nanosecond: return 1/1000000000;
		case Temporal.Unit.Microsecond: return 1/1000000;
		case Temporal.Unit.Millisecond: return 1/1000;
		case Temporal.Unit.Second: return 1;
		case Temporal.Unit.Minute: return 60;
		case Temporal.Unit.Hour: return 3600;
	}
}

class Temporal.Duration {
	// Fixed time units carry a dimension.
	total<U: Temporal.Unit>(unit: U): float64.<{ s: 1, ratio: unitRatio(U) }>
		where U <= Temporal.Unit.Hour;

	// Calendar units need a reference point and produce a plain count.
	total(options: { unit: Temporal.Unit, relativeTo: Temporal.ZonedDateTime | Temporal.PlainDate }): float64;
}
```

```js
const elapsed = end.since(start);
const s = elapsed.total(Temporal.Unit.Second); // Second
const ms = elapsed.total(Temporal.Unit.Millisecond); // Millisecond
```

With a cast operator, a duration converts to its seconds quantity implicitly wherever one is expected:

```js
class Temporal.Duration {
	operator Second() {
		return this.total(Temporal.Unit.Second);
	}
}
```

Then the arithmetic the unit system exists for typechecks end to end:

```js
type Meter = float64.<{ m: 1 }>;
type Velocity = float64.<{ m: 1, s: -1 }>;

const distance: Meter = 400;
const elapsed = end.since(start);

const speed: Velocity = distance / Second(elapsed); // m/s
// const wrong: Velocity = distance * Second(elapsed); // TypeError: { m: 1, s: 1 } is not { m: 1, s: -1 }
```

Because the ratio rides along, mixing scales stays correct without the programmer normalizing anything. Dividing a distance by a duration totalled in milliseconds yields ```float64.<{ m: 1, s: -1, ratio: 1000 }>```, which converts to ```Velocity``` at the assignment boundary by the conversion rules in the primitive metadata document.

## Precision

```epochNanoseconds``` is a ```bigint``` because Temporal's range spans roughly ±273,972 years, about 8.64e21 nanoseconds, which needs 74 bits. That exceeds ```int64```, but not ```int128```, so an ```Instant``` whose storage is a single ```int128``` field is a value type class: an array of instants is contiguous memory with no boxing, and comparing two instants is a 128-bit integer comparison.

```js
class Temporal.Instant {
	#epochNanoseconds: int128; // The whole instant, exactly
	get epochNanoseconds(): bigint {
		return bigint(this.#epochNanoseconds);
	}
}
```

```bigint``` remains the type of the public accessor for compatibility with Temporal as specified; the ```int128``` is the storage.

```Temporal.Duration```, by contrast, has only fixed-width integer fields, so it satisfies the value type class rule in the main proposal: an array of durations is contiguous memory with no boxing, which is what a scheduler or an animation timeline wants.

```total``` in seconds returns a ```float64```, whose 53-bit mantissa represents nanosecond totals exactly for durations up to about 104 days. Beyond that, or when exactness is required regardless, use ```decimal128``` or work in ```epochNanoseconds```:

```js
class Temporal.Duration {
	total<U: Temporal.Unit>(unit: U): decimal128.<{ s: 1, ratio: unitRatio(U) }>;
}
const exact: decimal128.<{ s: 1 }> = elapsed.total(Temporal.Unit.Second);
```

Overloading on return type selects between these, as described in the main proposal.

## Ordering and Ranges

```compare``` returns ```int32```, and with operator overloading the comparisons read the way the arithmetic already does. Defining them on the classes is optional sugar over ```compare``` and ```since```:

```js
class Temporal.Instant {
	operator<(other: Temporal.Instant): boolean {
		return Temporal.Instant.compare(this, other) < 0;
	}
	operator-(other: Temporal.Instant): Temporal.Duration {
		return other.until(this);
	}
	operator+(other: Temporal.Duration): Temporal.Instant {
		return this.add(other);
	}
}

if (start < end) {}
const elapsed = end - start; // Temporal.Duration
```

Note that ```===``` is never overloadable, so identity comparison of two Temporal objects remains reference identity; ```equals``` is the value comparison, as it is today.

A range whose validity depends on both endpoints is a dependent record type, checked at boundaries by the rules in [dependent record types](dependentrecordtypes.md):

```js
type Interval = {
	start: Temporal.Instant,
	end: Temporal.Instant
} where this.start < this.end;

function schedule(interval: Interval) {}
// schedule({ start: b, end: a }); // TypeError: the where clause failed
```

## Serialization

Every Temporal type has an ISO 8601 string form and a ```toJSON```. A cast operator from ```string``` is what lets a typed parse construct one directly, so a Temporal-typed field validates during the parse described in [Serialization](serialization.md) rather than in a post-processing pass:

```js
class Temporal.Instant {
	operator Temporal.Instant(value: string) {
		return Temporal.Instant.from(value); // RangeError on a malformed string
	}
}

type Event = {
	name: string,
	at: Temporal.Instant,
	duration: Temporal.Duration
};

const event = JSON.parse.<Event>('{ "name": "a", "at": "2026-07-09T12:00:00Z", "duration": "PT1H30M" }');
event.at; // Temporal.Instant
JSON.stringify(event); // toJSON is respected, round-tripping the ISO strings
```

The reflection examples in [decorators](decorators.md) already type fields as ```Temporal.Instant``` and ```Temporal.ZonedDateTime```; those work unchanged once these signatures exist.

## Open Questions

- ```unitRatio``` is total only over the time units. The ```where U <= Temporal.Unit.Hour``` constraint relies on enum ordering, which the enum section now defines for enumerations over ordered underlying types.
- Calendar arithmetic is not dimensioned and cannot be: a month has no fixed length. Durations mixing calendar and time components have no ```{ s: 1 }``` reading, and their ```total``` overload correctly requires ```relativeTo```.
