# Threading Extension and Notes

ECMAScript needs real threading where any function can be spawned as a thread that runs on another core, in the same heap, sharing the same objects. This is the shared-memory model that Java, Go, and C# have - not the worker model, where every thread is a separate heap that communicates by copying. There is no structured clone, no ```postMessage```, and no need to marshal data through a ```SharedArrayBuffer``` byte buffer: you share a value by referencing it. The syntax for creating and managing threads should be minimal and effortless.

For example, you should be able to define a global ```a: uint32``` and atomically add to it from a thread with ```Atomics.add(ref a, 5)```, without first shuffling it into a typed array:

```js
let a: uint32 = 0;
function A() {
  Atomics.add(ref a, 5);
}
async function B() {
  A();
  Atomics.add(ref a, 5);
}
// callThread runs the function on another thread and returns a joinable handle.
await Promise.all([A.callThread(), B.callThread()]); // Join
a; // 15
```

```callThread``` runs the function on another thread and returns a Promise that settles with the function's return value, or rejects with the exception it threw. Internally it spawns a thread that closes over the state the function references - the closure sees the same variables, objects, and imported bindings it would see on the calling thread, because there is one heap. In fully typed code the compiler knows statically which state is shared and can make this close to free; dynamic code works too, at higher cost, but that case is rare.

Its signature places an optional options bag before the forwarded arguments:

```js
// On any function; Return is the function's own return type.
callThread(options?: { signal?: AbortSignal }, ...args): Promise.<Return, any>;
```

The ```options``` bag is optional and distinguished from the function's own arguments by shape, so a function whose own first parameter is itself an ```{ signal }```-shaped object is the one ambiguous case; it is called with its arguments and no bag.

## The shared heap

Spawned threads run in the same realm. ```globalThis``` is the same object, there is one ```Array```, one ```Object.prototype```, one copy of each of your classes, and one already-executed module graph. ```x instanceof Foo``` is true on every thread because there is exactly one ```Foo```. Spawning a thread costs a thread - a native stack and some per-thread allocator state - not another copy of your program's startup, so a thread per request or per connection is a reasonable thing to do rather than something to avoid.

Because objects are shared by reference, a value that never leaves the thread that created it costs nothing; a value only takes on the cost of concurrency once a second thread actually touches it. The ```shared``` modifier (a contextual keyword in the main proposal) is the explicit, typed form of that boundary: it marks a binding or field whose value is expected to cross threads, which lets the compiler place it in shared storage from the start and reason statically about where synchronization is required, rather than inferring thread-locality at runtime.

## Promises across threads

A promise is an ordinary heap object, so it is shared like everything else. If one thread registers a ```.then()``` (or is suspended at an ```await```) and another thread settles the promise, the reaction runs on the settling thread's microtask queue - there is no hop back to the registering thread. Each thread drains its own microtask queue, and queues never interleave jobs from other threads. The handle returned by ```callThread``` is the exception to the settling-thread rule: like an explicit join, it settles on the thread that called ```callThread```, tied to that thread's event loop, so awaiting a spawned function behaves the way you expect.

## Memory model

The language stays single-threaded in its semantics *per thread*. The engine guarantees memory safety no matter how badly a program races: no torn engine values, no corrupted object storage, no type confusion. A data race in your own code produces a stale or surprising *value*, never a corrupted heap and never a crash - races on your data are your problem, races on the engine's data are the engine's problem.

This is a weaker guarantee than Rust's and a stronger one than C++'s, and the difference is worth stating plainly. Rust's ```Send```/```Sync``` prove at compile time that a program has no data races at all: the type system carries the proof, and a racy program does not build. This proposal makes no such claim. A race here is a bug with a bounded blast radius - a stale value, never a corrupted heap or a crash - which is the position Go and Java take. What it offers in place of a proof is a discipline the types express: the ```shared``` modifier marks the state that crosses threads, and mutable shared state is reached through ```Atomics``` or under a ```Lock```. A program that keeps to that discipline is race-free; the language enforces memory safety unconditionally but leaves race-*freedom* to the program, because retrofitting ```Send```/```Sync``` onto a shared mutable heap without a borrow checker would make it a different language.

Plain operations on shared data are **not** automatically atomic. A shared ```a += 5``` performed by two threads without synchronization is a data race; the result is one of the values allowed by the relaxed memory model - the same model ECMAScript already defines for ```SharedArrayBuffer``` access - that is, a value that may have missed the other thread's update. Reads and writes of a single primitive value don't corrupt the engine, but wide value types and updates that span multiple fields (a record, a composite) can be observed torn or half-applied by another thread. Atomicity is obtained exclusively through ```Atomics.*```; there is no implicit "some operations are atomic" rule, because leaving that set undefined would make racing programs unspecifiable and would tax every shared write.

```js
let a: uint32 = 0;
function A() {
  while (true) {
    a += 5; // data race: not atomic
  }
}
A.callThread();
await new Promise(resolve => setTimeout(resolve, 100));
a; // Unspecified value: concurrent unsynchronized writes. Use Atomics.add(ref a, 5) for a defined result.
```

## Atomics on typed values

Today ```Atomics``` operates only on integer typed arrays: ```Atomics.add(typedArray, index, value)```. The extension keeps that form and adds two more, so the same operations apply to a typed binding or an object's own property:

```js
Atomics.add(ref a, value);         // atomic RMW on a typed binding
Atomics.add(obj, 'count', value);  // atomic RMW on an own data property
Atomics.add(typedArray, i, value); // unchanged
```

The full set carries over from the SharedArrayBuffer atomics: ```load```, ```store```, ```add```, ```sub```, ```and```, ```or```, ```xor```, ```exchange```, ```compareExchange```, ```wait```, ```waitAsync```, and ```notify```. Each is a single sequentially-consistent step on the target. ```compareExchange``` compares with SameValueZero, so compare-and-swap loops that cycle through ```NaN``` behave. The bitwise operations - ```and```, ```or```, ```xor``` - require an integer target. ```wait``` and ```notify``` require an integer target as well, since a futex compares bit patterns. ```load```, ```store```, ```exchange```, and ```compareExchange``` apply to any shared value type where the operation is meaningful.

```add``` and ```sub``` also accept ```float32``` and ```float64``` targets. They are specified as a sequentially consistent compare-exchange loop - read, add, compare-exchange, retry - which is how hardware without a native atomic float add implements it, and how C++20's ```atomic<double>::fetch_add``` is defined. Because ```compareExchange``` uses SameValueZero, the loop terminates when the observed value is ```NaN```, rather than spinning forever as a bitwise comparison would.

```js
let total: shared float64 = 0;
function accumulate(values: [].<float64>) {
  for (const v of values) {
    Atomics.add(ref total, v);
  }
}
```

Atomic float addition is not associative, so a parallel reduction over floats produces a result that depends on thread interleaving. Where reproducibility matters, accumulate a per-thread partial and sum the partials on the joining thread in a fixed order; that is also faster, since it touches shared memory once per thread rather than once per value.

A ```shared SoA.<T>``` from the [structure of arrays](soa.md) extension allocates its columns in shared memory. Because different fields occupy different columns, threads writing different fields of the same elements never contend for a cache line, which interleaved storage cannot avoid.

## Synchronization

Atomics cover single-location updates. For anything larger - guarding a multi-field update, a shared collection, or a handoff between threads - the extension provides ordinary objects with blocking and async methods:

```js
const lock = new Lock();
lock.hold(() => { /* critical section, released like a finally */ });
const release = await lock.asyncHold(); // non-blocking acquire, call release() when done

const cond = new Condition();
lock.hold(() => {
  while (!ready) {
    cond.wait(lock); // atomically releases the lock and parks, spurious wakeups allowed
  }
});
cond.notify(); // or cond.notifyAll()
await cond.asyncWait(lock); // promise resolves holding the lock again

const tls = new ThreadLocal.<uint32>(); // ThreadLocal.<T>: .value is a T, independent per thread
```

```Lock``` is non-recursive. ```wait```/```notify``` are the textbook condition-variable handshake - and since the mailbox can be any shared object with real methods, a producer/consumer handoff is expressible directly rather than as an index into a byte buffer. On threads where the embedder forbids blocking, for instance the main thread of a browser, the ```async``` forms are used instead of the blocking ones. The closure passed to ```hold``` does not escape the call, so an engine allocates nothing for it and may inline the critical section, though the closure cannot be marked ```inline``` for the reason a ```ref``` callback cannot.

## Parallel iteration

Spawning one thread per element is spawning too many threads. Splitting a range across a fixed pool and joining is the common shape - the ```#integrate(begin, end, dt)``` slice loops in the [entity component system](examples/ecs.md) example are exactly this done by hand - so the extension provides it directly over a shared work-stealing pool:

```js
// Runs the body over the range, partitioned across the pool, and joins before returning.
Thread.parallelFor(0, particles.length, (i: uint32) => {
  const ref p = particles[i];
  p.position += p.velocity * dt;
});

// The reduction form sums per-thread partials in a fixed order, so the result is deterministic.
const total = Thread.parallelReduce(
  0, rows.length,
  0.0,
  (i: uint32) => rows[i].mass, // per-element
  (a: float64, b: float64) => a + b // combine
);
```

Each worker owns a contiguous sub-range, so the reference-liveness rule applies per slice: a reference into an element is valid while that slice runs, and nothing may change the array's length for the duration. Because the sub-ranges are disjoint, writes to distinct elements never contend, and a ```shared SoA.<T>``` makes writes to distinct *fields* non-contending as well, since each field is its own column. ```parallelReduce``` exists rather than a plain ```parallelFor``` with a shared accumulator because atomic floating-point addition is not associative: accumulating a per-thread partial and combining the partials in a fixed order is both reproducible and faster, touching shared memory once per thread rather than once per element. The pool is sized to the hardware by default and reused across calls, so ```parallelFor``` is cheap enough to place inside a frame loop rather than something to set up once.

## Cancellation

A thread is cancelled with an ```AbortSignal```, the same mechanism the rest of the platform uses. (The Cancelable Promise proposal this file once assumed was withdrawn.)

```js
const controller = new AbortController();
const t = search.callThread({ signal: controller.signal });
// ...
controller.abort(); // the thread observes the abort at its next check point
```

A cooperative check is often just a shared boolean the winner flips, which every other thread sees on its next read. Cancellation is data like anything else.

## Applications

* Game algorithms like pathfinding.
* Parsing large binary data formats when using binary WebSocket or WebTransport tasks.

## Future Applications

* Building DOM nodes in a separate thread then appending in the main thread. This is intuitive for programmers, but currently is not possible. In an ideal web environment this would just work where you could document.createElement in a function and as long as you didn't try to reference the active DOM you'd be fine.
  * In a large single page application multiple threads could be spun up creating different sections of the DOM that are then joined and appended to the document.
* In cases where you're waiting for data from a REST call and get a JSON object back you then need to process the data. A thread could do the REST call, perform the JSON.parse, processing, then return the data without having to postMessage.

## Concurrent Data Structures

Because a ```Map``` or ```Set``` is a shared object like any other, one instance guarded by a ```Lock``` is directly usable as a shared cache across threads. This removes any per-thread copies, cache-server thread, or hand-rolled hashmap over a byte buffer that might have been used. Native concurrent (lock-free or fine-grained) implementations of these structures would be valuable so the guarding is built in.

# Node.js, Deno, etc

Node.js would use this as well, where offloading parsing and expensive operations to threads is very beneficial and where the shared-heap model avoids the serialization that Worker threads currently pay.