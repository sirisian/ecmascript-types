# Threading Extension and Notes

ECMAScript needs real threading where any function can be spawned as a thread that runs on another core, in the same heap, sharing the same objects. This is the shared-memory model that Java, Go, and C# have - not the worker model, where every thread is a separate heap that communicates by copying. There is no structured clone, no ```postMessage```, and no need to marshal data through a ```SharedArrayBuffer``` byte buffer: you share a value by referencing it. The syntax for creating and managing threads should be minimal and effortless.

For example, you should be able to define a global ```a: shared uint32``` and atomically add to it from a thread with ```Atomics.add(ref a, 5)```, without first shuffling it into a typed array:

```js
let a: shared uint32 = 0;
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

The ```options``` bag is optional, so the first argument has to be classified. It is the bag when it is an ordinary object whose own ```signal``` property holds an actual ```AbortSignal```, or an object with no own properties at all - an explicit empty bag - *unless* the callee's declared first parameter accepts it, in which case it is an argument and there is no bag. The signature wins because it is better evidence than shape: this is the ambiguity untyped JavaScript cannot resolve and a type annotation can. Argument count plays no part; rest parameters and overloads make a function's declared length useless for this. What is left over is narrow enough to state: an untyped variadic function whose one intended argument is an object carrying a real ```AbortSignal``` under the key ```signal``` will see it taken as the bag, and either annotating the parameter or wrapping the value fixes it.

The thread ends when the function does. It runs to completion, adopts the result if it is a thenable, drains its own microtask queue, and only then posts the settlement home - so everything the thread did, including everything its trailing microtasks did, is visible to whoever awaits the handle. Work the thread scheduled with the host rather than the language, a pending timer say, is the host's business to define; what a host may not do is change what the handle means.

## The shared heap

Spawned threads run in the same realm. ```globalThis``` is the same object, there is one ```Array```, one ```Object.prototype```, one copy of each of your classes, and one already-executed module graph. ```x instanceof Foo``` is true on every thread because there is exactly one ```Foo```. Spawning a thread costs a thread - a native stack and some per-thread allocator state - not another copy of your program's startup, so a thread per request or per connection is a reasonable thing to do rather than something to avoid.

Because objects are shared by reference, a value that never leaves the thread that created it costs nothing; a value only takes on the cost of concurrency once a second thread actually touches it. The ```shared``` modifier (a contextual keyword in the main proposal) is the explicit, typed form of that boundary: it marks a binding or field whose value is expected to cross threads, which lets the compiler place it in shared storage from the start and reason statically about where synchronization is required, rather than inferring thread-locality at runtime.

```shared``` applies to value types - the sized numerics, vectors, value-typed classes, fixed arrays of value types, and ```SoA.<T>``` - because those are the things that would otherwise live in a register, a stack slot, or a thread-local nursery, and placement is what the modifier decides. An object is already shared: there is one heap, and any thread that can reach a reference can reach the object. So ```shared Map``` is not a thing to write, and it would be misleading if it were, since it would suggest a concurrent map rather than the plain one under a ```Lock``` that it would actually be. ```shared ref T``` is an error like nested ```shared``` is: a reference is a location, not a value. A [composite](composites.md) needs no marker either, being frozen.

The modifier is also a promise to the type checker, and this is where it earns more than an allocation strategy. Narrowing a binding - proving it is a ```uint32``` here even though it is declared wider there - assumes nobody rewrites it behind your back. On unmarked storage the checker assumes exactly that, and narrows, elides checks, and caches in registers the way single-threaded code always has. If another thread writes it anyway, what you get is a stale narrowing: the value is still of the declared type, because a cross-thread write goes through the same typed-storage boundary every other write does, so the failure is a branch taken on old information, never a wrong-typed value and never an unsafe one. On ```shared``` storage the checker assumes the opposite - that the world writes it - so the slot itself never narrows and every read yields the declared type. Narrow a local copy and work with that, which is what careful code does anyway. Declared types hold in both regimes, so everything the main proposal proves about eliding a redundant check at a declared type is as true here as it is in single-threaded code.

A note on what this deliberately is not. Rust's ```Send```/```Sync``` and Swift's ```Sendable``` make the type system carry a proof that nothing crosses a thread boundary unsafely, and it is fair to ask why a proposal about types does not do that. The answer is that such a marker only works as a viral, statically enforced discipline over every closure capture, which is the whole-program regime this model already declined when it chose a shared heap with memory safety guaranteed and race-freedom left to the program. A capability marker that cannot be enforced would be decoration. ```shared``` is instead a claim about *storage* - where it lives and whether the checker may assume it holds still - which is a claim the language can actually keep.

## Promises across threads

A promise is an ordinary heap object, so it is shared like everything else: any thread may settle it, and any thread may attach to it. What is *not* shared is where the reaction runs. A reaction runs on the thread that created it - the thread that called ```.then()```, or the thread suspended at the ```await```. Each thread drains its own microtask queue, and queues never interleave jobs from other threads. Settling from another thread posts the reaction to its home queue; the settle happens-before the reaction runs, so everything the settling thread did is visible to it. Attaching a handler to an already-settled promise enqueues on the attaching thread, which is what it already does today. There is no exception for the handle returned by ```callThread```: it settles wherever it was created, which is the thread that called ```callThread``` and awaited it, so a join behaves the way you expect for the same reason everything else does.

The alternative - running reactions on the settling thread, skipping the hop back - is cheaper per operation and was the earlier design here, but it does not survive contact with ```await```. Consider:

```js
const release = await lock.asyncHold();
// ...critical section...
release();
```

Under the settling-thread rule, the continuation after that ```await``` runs on whichever thread released the lock. Every waiter's critical section stacks onto the releasing thread, and a main-thread function that awaits anything another thread can settle silently continues off the main thread - fatal for a thread-affine host API, and invisible at the ```await``` that caused it. It also makes ```ThreadLocal``` meaningless across an ```await```, and would make placement a race: a handler attached before the settle would run on the settling thread and one attached after on the attaching thread, so which thread runs your callback would depend on timing. And it splits ```await``` from ```.then()```, which the language specifies in terms of each other; a threading model where rewriting one into the other changes which thread runs the body is a model programmers cannot hold in their heads. The per-operation cost of posting home is a cross-thread enqueue on the rare path, and settle-side execution is never *needed*: the settling thread can always do the work before it settles. If a program wants it anyway, that belongs in an explicit opt-in later, not in the default.

Threads here are real threads with native stacks, not virtual threads multiplexed over carriers. A Loom-style design - where a spawned function is a scheduler-managed continuation whose identity follows it across whichever carrier resumes it - is a coherent alternative, and it dissolves the question above rather than answering it, since "which thread" stops being observable. It is not the model here: blocking is real and gated on whether the embedder permits it, the pool is sized to hardware, and a thread costs a stack. That choice is what makes ```Atomics```, ```Lock```, and the memory model describable in the terms the rest of the platform already uses.

## Memory model

The language stays single-threaded in its semantics *per thread*. The engine guarantees memory safety no matter how badly a program races: no torn engine values, no corrupted object storage, no type confusion. A data race in your own code produces a stale or surprising *value*, never a corrupted heap and never a crash - races on your data are your problem, races on the engine's data are the engine's problem.

This is a weaker guarantee than Rust's and a stronger one than C++'s, and the difference is worth stating plainly. Rust's ```Send```/```Sync``` prove at compile time that a program has no data races at all: the type system carries the proof, and a racy program does not build. This proposal makes no such claim. A race here is a bug with a bounded blast radius - a stale value, never a corrupted heap or a crash - which is the position Go and Java take. What it offers in place of a proof is a discipline the types express: the ```shared``` modifier marks the state that crosses threads, and mutable shared state is reached through ```Atomics``` or under a ```Lock```. A program that keeps to that discipline is race-free; the language enforces memory safety unconditionally but leaves race-*freedom* to the program, because retrofitting ```Send```/```Sync``` onto a shared mutable heap without a borrow checker would make it a different language.

Plain operations on shared data are **not** automatically atomic. A shared ```a += 5``` performed by two threads without synchronization is a data race; the result is one of the values allowed by the relaxed memory model - the same model ECMAScript already defines for ```SharedArrayBuffer``` access - that is, a value that may have missed the other thread's update. Reads and writes of a single primitive value don't corrupt the engine, but wide value types and updates that span multiple fields can be observed torn or half-applied by another thread. A [composite](composites.md) is the exception and the idiom to reach for: it is built before it is published and frozen once it is, so publishing a multi-field snapshot is a single reference store that another thread reads whole or not at all. Atomicity is obtained exclusively through ```Atomics.*```; there is no implicit "some operations are atomic" rule, because leaving that set undefined would make racing programs unspecifiable and would tax every shared write.

```js
let a: shared uint32 = 0;
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

The full set carries over from the SharedArrayBuffer atomics: ```load```, ```store```, ```add```, ```sub```, ```and```, ```or```, ```xor```, ```exchange```, ```compareExchange```, ```wait```, ```waitAsync```, and ```notify```. Each is a single sequentially-consistent step on the target. ```compareExchange``` compares with SameValueZero, so compare-and-swap loops that cycle through ```NaN``` behave. The bitwise operations - ```and```, ```or```, ```xor``` - require an integer target. ```wait``` and ```notify``` require an integer target as well, since a futex compares bit patterns. ```load```, ```store```, ```exchange```, and ```compareExchange``` apply to any value type where the operation is meaningful.

What the operations do *not* look at is whether the target is marked ```shared```. The restrictions are about the type - integer for the bitwise operations and for ```wait```/```notify```, a typed own data property for the object form, since an ```any``` slot has no width to be atomic over - and not about the modifier. Requiring the marker would buy nothing: an unmarked binding captured by a function another thread runs is reachable from that thread regardless, so plain racy writes to unmarked storage are possible whether or not the synchronized ones are permitted, and forbidding only the synchronized ones would be a strange place to draw a line. It would also fracture generic code, since a function taking ```ref uint32``` would have to know whether its caller's storage was marked. What the marker changes is cost, not legality: marked storage is shared-capable from the start, while unmarked storage is promoted the first time another thread touches it, at higher cost, in the rare case - and a ```wait``` on it may force that promotion eagerly.

```add``` and ```sub``` also accept ```float32``` and ```float64``` targets. They are specified as a sequentially consistent compare-exchange loop - read, add, compare-exchange, retry - which is how hardware without a native atomic float add implements it, and how C++20's ```atomic<double>::fetch_add``` is defined. Because ```compareExchange``` uses SameValueZero, the loop terminates when the observed value is ```NaN```, rather than spinning forever as strict equality would, since ```NaN !== NaN``` makes the compare-and-swap fail against the very value it just read.

SameValueZero is the choice here rather than the three alternatives. Strict equality livelocks on ```NaN```, as above. A bitwise comparison - what C++'s ```compare_exchange``` and C#'s ```Interlocked``` use - would terminate, but it is not specifiable: an implementation is permitted to write an implementation-chosen ```NaN``` encoding when it stores a float, so whether ```compareExchange(ref x, NaN, v)``` succeeds would vary by engine. This extension could pin a canonical ```NaN``` for typed float storage and rescue the bitwise form, but a canonicalized bitwise compare is then SameValue with extra steps, and it inherits SameValue's flaw: SameValue keeps ```+0``` and ```-0``` distinct, so a computed ```-0``` - one ```-1 * 0``` is enough - fails to match a ```0``` sentinel, and a claim loop intermittently refuses a slot that is arithmetically zero. SameValueZero terminates ```NaN``` loops, matches ```-0``` to ```0```, and is the language's own storage equality, the predicate behind ```Map```, ```Set```, and ```includes``` - which is exactly the role an expected-value comparison plays. On integer targets all four candidates agree, so the existing ```Atomics``` surface is unaffected either way.

```js
let total: shared float64 = 0;
function accumulate(values: [].<float64>) {
  for (const v of values) {
    Atomics.add(ref total, v);
  }
}
```

Atomic float addition is not associative, so a parallel reduction over floats produces a result that depends on thread interleaving. Where reproducibility matters, give each thread its own sub-range and its own partial, and sum the partials on the joining thread in a fixed order; that is also faster, since it touches shared memory once per thread rather than once per value. Reproducibility comes from the fixed split and the fixed combining order, not from the threads - which is the same reason ```Thread.parallelReduce``` below fixes its partition.

A ```shared SoA.<T>``` from the [structure of arrays](soa.md) extension allocates its columns in shared memory. Because different fields occupy different columns, threads writing different fields of the same elements never contend for a cache line, which interleaved storage cannot avoid.

## Synchronization

Atomics cover single-location updates. For anything larger - guarding a multi-field update, a shared collection, or a handoff between threads - the extension provides ordinary objects with blocking and async methods:

```js
const lock = new Lock();
lock.hold(() => { /* critical section, released like a finally */ });
using guard = lock.acquire(); // released at the end of the enclosing block
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

```Lock``` is non-recursive. Reentrant monitors - Java's and C#'s - buy tolerance of self-nesting at the price of owner-and-count bookkeeping on the fast path, a ```Condition``` that has to release and restore a nesting depth, and the silent tolerance of layering violations that a non-reentrant lock reports the moment they appear. Go, Rust, C++'s default, Swift's ```Mutex```, and Web Locks all landed on non-reentrant, and so does this.

Since it is non-recursive, acquiring it while already holding it has to mean something, and what it means depends on whether the acquisition can still make progress. A *blocking* self-acquire cannot: the thread that would have to release is the thread now parked, so it is a certain deadlock, and it throws a ```TypeError``` rather than hanging - the same choice the platform already made when ```Atomics.wait``` on the main thread became an error rather than a frozen tab. The owner is known anyway, because ```Condition.wait``` has to check it. An ```asyncHold``` while holding is *not* a deadlock - the acquisition simply queues, and the current holder may release from a later job before anyone awaits it - so it is allowed and queues. The ```release``` from ```asyncHold``` may be called once, by the thread that received it: a second call or a call from another thread is a ```TypeError```, because tolerating a stale release would unlock a critical section that by then belongs to somebody else, which is the one failure a lock exists to prevent.

```hold``` returns whatever its callback returns, and releases on the way out whether the callback returned or threw. The closure passed to ```hold``` does not escape the call, so an engine allocates nothing for it and may inline the critical section, though the closure cannot be marked ```inline``` for the reason a ```ref``` callback cannot. ```acquire``` is the same lock without the callback: it returns a disposable guard, and ```using``` releases it at the end of the block, which reads better when the critical section wants an early ```return``` or spans a ```try```. Disposing a guard twice is a ```TypeError``` for the same reason releasing twice is. A guard is also where the type system can help in a way a callback cannot: a guard type the checker requires to be consumed by ```using``` turns forgetting to release into a type error rather than a deadlock at runtime.

```wait```/```notify``` are the textbook condition-variable handshake - and since the mailbox can be any shared object with real methods, a producer/consumer handoff is expressible directly rather than as an index into a byte buffer. On threads where the embedder forbids blocking, for instance the main thread of a browser, the blocking forms - ```hold```, ```acquire```, ```Condition.wait```, and ```Atomics.wait``` - throw a ```TypeError```, and the ```async``` forms are used instead. They throw rather than quietly becoming their own async forms: a ```hold``` that returned ```T``` on one thread and ```Promise.<T>``` on another would have a return type that depends on which thread is running, which is not a thing a typed language can say.

## Parallel iteration

Spawning one thread per element is spawning too many threads. Splitting a range across a fixed pool and joining is the common shape - the ```#integrate(begin, end, dt)``` slice loops in the [entity component system](examples/ecs.md) example are exactly this done by hand - so the extension provides it directly over a shared work-stealing pool:

```js
// Runs the body over the range, partitioned across the pool, and joins before returning.
Thread.parallelFor(0, particles.length, (i: uint32) => {
  const ref p = particles[i];
  p.position += p.velocity * dt;
});

// The reduction form combines per-slice partials in slice order, so the result is deterministic.
const total = Thread.parallelReduce(
  0, rows.length,
  0.0,
  (i: uint32) => rows[i].mass, // per-element
  (a: float64, b: float64) => a + b // combine
);
```

The range is cut into disjoint contiguous slices, so the reference-liveness rule applies per slice: a reference into an element is valid while that slice runs, and nothing may change the array's length for the duration. Because the slices are disjoint, writes to distinct elements never contend, and a ```shared SoA.<T>``` makes writes to distinct *fields* non-contending as well, since each field is its own column. ```parallelReduce``` exists rather than a plain ```parallelFor``` with a shared accumulator because atomic floating-point addition is not associative: accumulating a partial per slice and combining the partials in a fixed order is both reproducible and faster, touching shared memory once per slice rather than once per element. The pool is sized to the hardware by default and reused across calls, so ```parallelFor``` is cheap enough to place inside a frame loop rather than something to set up once.

Determinism is a property of the *partition*, not of which thread runs what. How the range is cut is left to the implementation - grain size is a tuning decision - but the cut is a function of ```begin``` and ```end``` alone: core count, current load, and how many workers happen to be idle may not influence it. Work stealing then moves whole slices between threads freely, which changes who computes a partial but never which elements are in it, and the partials are combined in ascending slice order. So the contract is: the same implementation, given the same range and the same body, produces the same result - on a busy machine or an idle one, on four cores or sixty-four. It is not a promise across implementations; an engine that cuts differently gets a different (equally deterministic) answer. Where cross-implementation reproducibility is needed, a future explicit option can pin the partition. The contract is also only as strong as the body: a body that consults ```ThreadLocal```, thread identity, or observed scheduling reintroduces the nondeterminism the partition rule removes.

The calling thread participates. It executes slices itself and, when none are left to claim, waits out the slices still in flight without parking on a wait queue, so a join is never a blocking operation and ```parallelFor``` is legal on threads where the embedder forbids blocking - the browser main thread included, which is where a frame loop lives. Two properties follow. Sequential execution is a conforming implementation: with no workers available the caller runs every slice itself, in order, so a host with no threads to give still runs the program correctly. And nesting cannot deadlock by exhausting the pool, since a body that itself calls ```parallelFor``` participates in its inner range the same way; the worst case is that the inner loop runs sequentially on the thread that entered it.

If the body throws, slices after the failing one are cancelled, slices before it are allowed to finish, and the error from the lowest-numbered failing slice is the one that propagates. That is the error a sequential run would have produced, since a sequential run reaches the lowest failing index first and stops there - so failures are as reproducible as results. ```parallelReduce``` follows the same rule, and a ```combine``` that throws is attributed to the later of the two slices it was combining.

## Cancellation

A thread is cancelled with an ```AbortSignal```, the same mechanism the rest of the platform uses. (The Cancelable Promise proposal this file once assumed was withdrawn.)

```js
const controller = new AbortController();
const t = search.callThread({ signal: controller.signal });
// ...
controller.abort(); // the thread observes the abort at its next check point
```

Abort is observed at the points where the thread is already suspending or about to run something: at ```callThread``` itself, where an already-aborted signal rejects the handle without spawning anything; when the thread picks up a job; at the resumption of an ```await```; and in the waits - ```Atomics.wait``` and ```waitAsync```, ```Lock``` acquisition, and ```Condition.wait```. A wait already parked is *woken* by the abort rather than left until someone happens to notify it, since a cancellation that cannot reach a parked thread cancels nothing.

Observing it throws ```signal.reason``` out of whatever operation was pending. That is an ordinary abrupt completion: ```finally``` blocks run, ```using``` declarations dispose, locks taken by ```hold``` release, and if it reaches the top of the thread function the handle rejects with it. Cancellation does not tear the thread down where it stands - the model that ```Thread.stop``` and ```Thread.Abort``` were both removed from their languages for, because destroying a thread mid-update leaves exactly the invariants a lock was protecting broken.

Abort is not checked at loop back-edges, since that would tax every loop in the language to serve a rare case. A compute loop with no suspension points cancels itself cooperatively, by reading ```signal.aborted``` - which works across threads because the signal is a shared object, and a read after another thread's ```abort()``` is synchronized. A cooperative check is often just a shared boolean the winner flips, which every other thread sees on its next read. Cancellation is data like anything else.

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