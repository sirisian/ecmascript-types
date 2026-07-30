# API Client

A typed HTTP API client: response discrimination, retry with backoff, header parsing, and a request cache. This is the scenario [pattern matching](../patternmatching.md) exists for - a boundary where every value arrives as a union of shapes and the program is one long "which case is this" - and it validates the design by using nearly every pattern form on problems a client actually has, with the exhaustiveness check standing where a missed case would otherwise ship. The types come from the main proposal's unions and literal types, [composites](../composites.md) key the cache, [typed regular expressions](../regexp.md) parse the headers, [ranges](../ranges.md) classify status codes, and [error handling](../errorhandling.md)'s typed errors drive the retry policy.

Features exercised:

- An exhaustive ```match``` over a union of response shapes discriminated by a literal ```status``` field, so adding a response kind to the type breaks every site that must handle it.
- Range patterns classifying status codes - ```when { status: 500..600 }:``` - the containment test a numeric family needs where enumeration is absurd.
- A generic ```Result.<T, E>``` matched through extractor patterns, ```Ok(let value)``` and ```Err(let error)```, with the bindings typed by ordinary generic inference from the subject.
- Regular expression patterns with typed named groups parsing ```Retry-After``` and ```Link``` headers, where a misspelled group name is a compile-time TypeError.
- Composite request keys as constant patterns: a match over interned keys is pointer comparisons, and the cache's ```Map``` needs no custom hashing.
- Typed error patterns in the retry loop - the ```catch (e: T)``` form reappearing as ```when let e: NetworkError:``` - deciding retry against give-up by type.
- A block arm that computes - statements then a final expression - where a closure would have broken ```await``` and ```return```.
- ```is``` with a pattern as a loop condition, binding and narrowing in the body.
- An enum-typed connection state matched exhaustively, including the sentinel cased to ```throw```.

## Response Shapes

The server speaks a small protocol: success carries a body, redirects carry a location, client errors carry a message, server errors carry nothing trustworthy. The union writes that down, and the literal and range discriminants make each member recognizable by pattern.

```js
type Response =
	| { status: 200, body: string, headers: Headers }
	| { status: 301 | 302, location: string }
	| { status: 404 }
	| { status: 429, headers: Headers }
	| { status: uint16, message?: string };

class NetworkError extends Error { retriable: boolean = true; }
class ProtocolError extends Error {
	status: uint16;
	constructor(message: string, status: uint16) { super(message); this.status = status; }
}
class Redirect extends Error {
	location: string;
	constructor(location: string) { super('redirect'); this.location = location; }
}
class RateLimited extends Error {
	response: Response;
	constructor(response: Response) { super('rate limited'); this.response = response; }
}
```

## Discriminating a Response

One ```match```, one arm per shape, checked complete. The union has five members and the arms cover five members; delete one and the ```match``` is a compile-time TypeError, which is the entire point of writing the type first.

```js
function interpret(response: Response): Result.<string, ProtocolError> {
	return match (response) {
		when { status: 200, let body }: new Ok(body);
		when { status: 301 | 302, let location }: throw new Redirect(location);
		when { status: 404 }: new Err(new ProtocolError('not found', 404));
		when { status: 429 }: throw new RateLimited(response);
		when { let status }: {
			const detail = response.message ?? 'server error';
			log.warn(`unhandled status ${status}: ${detail}`);
			new Err(new ProtocolError(detail, status));
		}
	};
}
```

The literal ```200``` takes ```uint16``` from the field, so the comparison is against what the field stores. ```let body``` binds ```string``` because the first field pattern narrowed the union to its first member. The last arm names only ```status```, and that is the presence rule doing its work: an object pattern requires the members it names, ```message``` is optional in the final member, so ```{ let status, let message }``` would fail a response that lacks it and the arm would not cover the member - a compile-time TypeError under the exhaustiveness check, caught before it shipped as a runtime one. The body reads ```response.message``` instead, at ```string | undefined```, off the subject the arm has narrowed. That arm is also the block form carrying a value: two statements and a final expression, whose value is the arm's, with no closure between the ```log``` call and the result.

## Status Classes by Range

Logging wants classes, not codes, and a class of codes is a range. A float would work as well as a ```uint16``` here - containment needs only an ordering - which is what separates range patterns from case labels over an enumerable type.

```js
function logClass(status: uint16): void {
	match (status) {
		when 100..200: log.debug('informational');
		when 200..300: log.debug('success');
		when 300..400: log.info('redirect');
		when 400..500: log.warn('client error');
		when 500..600: log.error('server error');
		default: log.error('unregistered status');
	};
}
```

The subject is open - ```uint16``` has 65536 values and the ranges name 500 of them - so the ```default``` is required, and its presence is the honest statement that the classification is partial.

## Result and the Extractors

The client's public reads return ```Result.<T, ProtocolError>``` rather than throwing, because a missing document is data to a caller, not an exception. The extractor protocol is one static method per case; the generic instantiates from the subject, so every binding below is typed without an annotation anywhere.

```js
sealed abstract class Result<T, E> {}
class Ok<T, E> extends Result.<T, E> {
	value: T;
	constructor(value: T) { super(); this.value = value; }
	static [Symbol.customMatcher]<T, E>(subject: Result.<T, E>): [T] | null {
		return subject instanceof Ok.<T, E> ? [subject.value] : null;
	}
}
class Err<T, E> extends Result.<T, E> {
	error: E;
	constructor(error: E) { super(); this.error = error; }
	static [Symbol.customMatcher]<T, E>(subject: Result.<T, E>): [E] | null {
		return subject instanceof Err.<T, E> ? [subject.error] : null;
	}
}

const title = match (await client.get('/article/1')) {
	when Ok(let body): extractTitle(body);        // body: string
	when Err(let error) if (error.status == 404): 'Untitled';
	when Err(let error): throw error;             // error: ProtocolError
};
```

```Result``` is sealed, so the ```match``` is over a closed hierarchy: the two extractor arms cover the two subclasses and the checker knows it - the guarded ```Err``` arm proves nothing, the unguarded one after it restores coverage. Reordering the two ```Err``` arms is the unreachability error, since an unguarded pattern above a guarded twin leaves the guard nothing to see.

## Header Parsing

```Retry-After``` is seconds or an HTTP date; ```Link``` is a comma-separated list of URI references with parameters. Both are strings with structure, which is the regular expression pattern's job, and the typed groups mean the bindings exist exactly when the pattern says they do.

```js
function retryDelay(header: string): float64 {
	return match (header) {
		when /(?<seconds>\d+)/ { groups: { let seconds } }:
			float64.parse(seconds);
		when /.+ GMT/:
			Date.parse(header) - Date.now();
		default: 1000;
	};
}

function nextPage(link: string): string | null {
	return match (link) {
		when /<(?<url>[^>]+)>;\s*rel="next".*/ { groups: { let url } }: url;
		default: null;
	};
}
```

A pattern matches the entire subject - the whole-string discipline every string constraint in this proposal uses - so the ```Link``` pattern spells its own trailing ```.*``` rather than inheriting a silent search. Misspell ```url``` as ```uri``` in the destructure and the arm is a compile-time TypeError against the match result's ```groups``` type; that error is this extension earning its keep at a boundary regexes have owned since the web began.

## The Request Cache

A request's identity is its method, path, and variant headers - a compound key, which is a [composite](../composites.md). Interning makes the ```Map``` correct with no hashing protocol, and it makes the hot requests *constants*, matchable by pointer.

```js
interface RequestKey { method: 'GET' | 'HEAD'; path: string; variant?: string = ''; }

const HEALTH = Composite.<RequestKey>({ method: 'GET', path: '/health' });
const cache = new Map.<Composite.<RequestKey>, Response>();

function lookup(key: Composite.<RequestKey>): Response | null {
	return match (key) {
		when HEALTH: synthetic200;          // one pointer comparison, no fetch
		default: cache.get(key) ?? null;
	};
}
```

```HEALTH``` in pattern position is an expression pattern: an interned constant compared by identity, which for a composite is the structural comparison already paid for at creation. The defaulted ```variant``` means ```Composite.<RequestKey>({ method: 'GET', path: '/health', variant: '' })``` *is* ```HEALTH``` - the two spellings interned together - so the pattern hits however the key was built.

## Retry by Error Type

The retry loop is a ```match``` over the caught value. ```catch``` stays untyped here deliberately: the decision is data - retriable or not, and after how long - so the error goes to ```match``` as a value and the annotated bindings do what typed ```catch``` clauses would, in an expression that computes the delay.

```js
async function withRetry(request: Composite.<RequestKey>): Response {
	for (const attempt: uint8 of 0..5) {
		try {
			return await send(request);
		} catch (e) {
			const delay: float64 = match (e) {
				when let err: NetworkError if (err.retriable): 250 * 2 ** attempt;
				when RateLimited { let response }:
					retryDelay(response.headers.get('Retry-After') ?? '1');
				default: throw e;
			};
			await sleep(delay);
		}
	}
	throw new NetworkError('exhausted retries');
}
```

```when let err: NetworkError``` is the test-and-bind form - ```catch (e: NetworkError)``` relocated into an expression - and the juxtaposed ```RateLimited { let response }``` narrows to the class and destructures it in one clause. The ```default: throw e``` keeps the arm honest: everything unmatched propagates, typed rethrow being the one thing a partial handler owes its caller.

## Connection State

The client's transport is a small machine, and its state is an enum because the states are a closed set that ```match``` should check. The sentinel enumerator is cased to ```throw```, which keeps the exhaustiveness check instead of buying it off with a ```default```.

```js
enum ConnState: uint8 { Idle, Connecting, Ready, Draining, Count };

function tick(state: ConnState): ConnState {
	return match (state) {
		when ConnState.Idle: connect();
		when ConnState.Connecting: poll();
		when ConnState.Ready: ConnState.Ready;
		when ConnState.Draining: drain();
		when ConnState.Count: throw new RangeError('sentinel is not a state');
	};
}
```

## Draining the Socket

```is``` with a pattern is the one-arm ```match```, and a read loop is its natural home: test, bind, and narrow in the condition, use the binding in the body.

```js
while (socket.read() is Ok(let chunk)) {
	buffer.append(chunk);          // chunk: [].<uint8>, in scope exactly where the test held
}
```

## What This Example Establishes

The client never tested a ```status``` twice, never read a header field the type didn't prove, and never wrote an ```instanceof``` ladder - each ```match``` is the control flow the shape of the data already implied. Three checks did standing work: the response union and the ```Result``` hierarchy are exhaustive, so a new response kind or result case is a build break at every site that must care; the regex group names are checked against the patterns that define them; and the one ```default``` over an open subject (```uint16```) is present because the type says it must be. The pieces composed without a special case anywhere - extractors are methods, constants are values, ranges are values, types are patterns - which is what the pattern matching document means when it claims the type system was ready.
