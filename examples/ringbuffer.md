# Lock-Free Ring Buffer

Bounded queues over shared memory: a single-producer single-consumer ring that two threads use as a wait-free channel, and a Vyukov-style multi-producer multi-consumer ring built from per-cell sequence numbers and compare-exchange. This is the example that exercises the [threading](../threading.md) extension's memory model for real - not a counter incremented from two threads, but an algorithm whose correctness *is* its ordering claims. Lock-free code is where a memory model's precision gets tested: which reads may be plain, which writes publish, what the atomics can address, and what synchronizes a thread's birth. Several of those questions turned out to have answers the documents imply but do not state, and the notes at the end record them.

Features exercised, and why they matter here:

- ```shared``` value type storage: a fixed ```[Capacity].<T>``` of payload slots and ```uint32``` indices, mutated by two or more threads under a protocol rather than a lock.
- The by-reference ```Atomics``` forms - ```Atomics.store(ref this.#tail.value, ...)```, ```Atomics.compareExchange(ref this.#head.value, ...)``` - as the entire synchronization vocabulary, each a single sequentially consistent step.
- Plain reads of shared integers as the relaxed tier: the memory model defines a racy read of a single primitive as a stale-but-whole value, which is exactly what index caching wants, so the classic SPSC optimization is expressible without a relaxed atomic.
- A padded cell - a value class of one field at ```@align(64)``` - as the type of the head and the tail, so the producer's line and the consumer's line never ping-pong, and a class-level ```where``` clause holding ```Capacity``` to a power of two.
- Defined unsigned wraparound as a load-bearing feature: the indices are monotonic ```uint32``` counters that wrap, occupancy is a wrapping subtraction, and the MPMC lag test reinterprets a wrapped difference as ```int32``` - all of it specified arithmetic rather than undefined behavior to step around.
- ```ref``` out-parameters for ```tryPop```, so a failed pop costs nothing and a successful one costs one value type copy, with no allocation and no nullable box.

## Layout and the Shape of Sharing

Everything the two sides contend on gets its own cache line, and everything one side owns is grouped away from what the other side owns. The payload array is fixed-length, which matters twice: a fixed array never reallocates, so references into it carry no liveness restriction, and its capacity being a compile-time value generic lets the power-of-two requirement be a ```where``` clause on the class rather than a constructor assertion.

The queue is an object. It has an identity - a copy of it would be a second queue - and the threads reach the one queue by reference, so it is a class and not a ```value class```, and a class held by reference has no layout for a decorator to place a field in. The alignment therefore belongs to the type of what is stored: a value class of one field, whose size rounds up to its alignment, so no two cells share a line once each is stored at its alignment - which shared storage is, since it has the size and alignment of what it holds, and which an engine is expected to do for the two unshared caches as well.

```js
// ringbuffer.js
// One value on a cache line of its own: byteLength and alignment are both 64.
export value class Padded<T: type> {
	@align(64) value: T;
}

export class SPSCQueue<T: type, Capacity: uint32> where (Capacity & (Capacity - 1)) == 0 {
	#slots: shared [Capacity].<T>;

	// Each index owns a cache line; each side's private cache owns one too.
	#head: shared Padded.<uint32>;  // Next unread slot. Written only by the consumer.
	#tail: shared Padded.<uint32>;  // Next free slot. Written only by the producer.
	#headCache: Padded.<uint32>;    // Producer's last-seen #head. Producer-only.
	#tailCache: Padded.<uint32>;    // Consumer's last-seen #tail. Consumer-only.
}
```

This is Rust's ```CachePadded<T>``` and the padded struct inside .NET's concurrent queue, and it reaches further than a decorator on the field would: ```let hits: shared Padded.<uint64>``` pads a binding, and ```[Threads].<Padded.<uint64>>``` gives each thread's counter its own line.

The indices are not slot numbers; they are monotonic counters that wrap at 2^32. A counter maps to a slot as ```index & (Capacity - 1)```, and occupancy is ```tail - head``` as wrapping ```uint32``` subtraction, which stays correct across the wrap exactly because ```Capacity``` divides 2^32 - the ```where``` clause is not a style preference, it is what makes the arithmetic sound. In C this idiom leans on unsigned being the one integer type without undefined overflow; in Rust it is ```wrapping_sub``` at every site; here wrapping is what the integer types mean, so the natural spelling is the correct one.

## Single Producer, Single Consumer

Each side owns one index and reads the other's. The push sequence is: write the payload with a plain store, then publish it with a sequentially consistent store to ```#tail```. The pop sequence mirrors it. The interesting part is which reads get to be *plain*.

```js
// ringbuffer.js
partial class SPSCQueue<const T, const Capacity: uint32> {
	tryPush(value: T): boolean {
		const tail = this.#tail.value; // Plain: reading our own thread's last write
		if (tail - this.#headCache.value == Capacity) { // Apparently full; refresh our view of the consumer
			this.#headCache.value = Atomics.load(ref this.#head.value);
			if (tail - this.#headCache.value == Capacity) {
				return false; // Genuinely full
			}
		}
		this.#slots[tail & (Capacity - 1)] = value; // Plain write of the payload
		Atomics.store(ref this.#tail.value, tail + 1); // Publish: a load that sees this sees the payload above
		return true;
	}

	tryPop(ref out: T): boolean {
		const head = this.#head.value; // Plain: our own write
		if (head == this.#tailCache.value) {
			this.#tailCache.value = Atomics.load(ref this.#tail.value);
			if (head == this.#tailCache.value) {
				return false; // Empty
			}
		}
		out = this.#slots[head & (Capacity - 1)]; // Plain read of a published slot, copied through the ref
		Atomics.store(ref this.#head.value, head + 1); // Return the slot to the producer
		return true;
	}
}
```

Three tiers of access are in play, and the memory model supports each by name. The **sequentially consistent pair** - the ```Atomics.store``` to an index and the ```Atomics.load``` that observes it - is the synchronization: a load that sees the store also sees every plain write sequenced before it, which is what makes the payload write safe to leave plain. The **own-index plain reads** need no atomicity at all, since a thread always sees its own writes in program order. And the **cached-index refresh** exists so the hot path does not touch the other side's cache line: ```#headCache``` goes stale the moment it is read, which is fine, because a stale view only ever makes the queue look *more* full or *more* empty than it is - conservative in both directions. That staleness tolerance is also why the probe could even have been a plain racy read of ```#head``` itself: the model defines a racy read of a single primitive as some whole previously written value, never a torn one, which is precisely a relaxed load. The relaxed tier of this algorithm is spelled as ordinary reads.

What a thread may *not* do is share the payload copy: ```out = this.#slots[i]``` is a plain multi-field copy, and the model is explicit that wide value types can be observed torn if raced on. Nothing tears here because the index protocol guarantees the producer is finished with a slot before the consumer's copy begins and vice versa - the algorithm provides the exclusion the model does not. That is the same division of labor C++ and Rust queues live with; the difference is only that a torn read here would be a wrong value rather than undefined behavior.

## Waiting Without Spinning

```tryPop``` returning ```false``` in a loop is correct and wasteful. The futex forms wait on an integer's value directly, and the consumer already has an integer whose change means "there is more": the producer's ```#tail```.

```js
// ringbuffer.js
partial class SPSCQueue<const T, const Capacity: uint32> {
	// Producer side: push, then wake a sleeping consumer if there is one.
	pushNotify(value: T): boolean {
		if (!this.tryPush(value)) {
			return false;
		}
		Atomics.notify(ref this.#tail.value, 1);
		return true;
	}

	// Consumer side: sleep until #tail moves past what we last observed.
	async popWait(): Promise.<T> {
		let value: T;
		while (!this.tryPop(ref value)) {
			const r = Atomics.waitAsync(ref this.#tail.value, this.#tailCache.value);
			if (r.async) {
				await r.value;
			}
		}
		return value; // A reference cannot span an await, so the waiting form returns by value
	}
}
```

```waitAsync``` compares the current value of ```#tail``` against the consumer's last-seen ```#tailCache``` and only sleeps if they still match, so a push that lands between the failed ```tryPop``` and the wait does not strand the consumer - the lost-wakeup race is closed by the futex's compare, the same way it is in every futex protocol. The ```wait```/```notify``` pair requires an integer target because a futex compares bit patterns; the payload slots, being multi-field value types, could not be waited on, and do not need to be.

## Many Producers, Many Consumers

With multiple producers, ```#tail``` must be *claimed* rather than advanced, and a claimed cell needs a way to say "the payload is actually in me now" that is distinct from the global index. Vyukov's design gives every cell its own sequence number: a producer may fill cell ```i``` when the cell's sequence equals the position it claimed, and it publishes by setting the sequence one past that; a consumer may drain it when the sequence is one past the position, and recycles it a full lap ahead.

The sequence numbers need per-element atomic access, and that requirement decides the layout: they live in their own ```[Capacity].<uint32>``` beside the payload array, because the typed-array form ```Atomics.load(array, i)``` is the form the extension defines for elements. The structure-of-arrays split the queue wanted for cache behavior anyway is here forced by what the atomics can address - the notes return to this.

```js
// ringbuffer.js
export class MPMCQueue<T: type, Capacity: uint32> where (Capacity & (Capacity - 1)) == 0 {
	#sequence: shared [Capacity].<uint32>;
	#slots: shared [Capacity].<T>;
	#head: shared Padded.<uint32>;
	#tail: shared Padded.<uint32>;

	constructor() {
		for (let i: uint32 = 0; i < Capacity; i++) {
			this.#sequence[i] = i; // Plain: nothing else can see the queue until construction completes
		}
	}

	tryPush(value: T): boolean {
		let pos = Atomics.load(ref this.#tail.value);
		while (true) {
			const i = pos & (Capacity - 1);
			const seq = Atomics.load(this.#sequence, i);
			const lag = int32(seq - pos); // Wrapping difference, read as signed
			if (lag == 0) { // The cell is ready for exactly this position
				if (Atomics.compareExchange(ref this.#tail.value, pos, pos + 1) == pos) {
					this.#slots[i] = value; // We own the cell; plain write
					Atomics.store(this.#sequence, i, pos + 1); // Publish to consumers
					return true;
				}
				pos = Atomics.load(ref this.#tail.value); // Lost the claim; reread and retry
			} else if (lag < 0) {
				return false; // The cell is a full lap behind: the queue is full
			} else {
				pos = Atomics.load(ref this.#tail.value); // Another producer got here first; catch up
			}
		}
	}

	tryPop(ref out: T): boolean {
		let pos = Atomics.load(ref this.#head.value);
		while (true) {
			const i = pos & (Capacity - 1);
			const seq = Atomics.load(this.#sequence, i);
			const lag = int32(seq - (pos + 1));
			if (lag == 0) { // Published for exactly this position
				if (Atomics.compareExchange(ref this.#head.value, pos, pos + 1) == pos) {
					out = this.#slots[i];
					Atomics.store(this.#sequence, i, pos + Capacity); // Recycle for one lap later
					return true;
				}
				pos = Atomics.load(ref this.#head.value);
			} else if (lag < 0) {
				return false; // Empty
			} else {
				pos = Atomics.load(ref this.#head.value);
			}
		}
	}
}
```

The ```lag``` line deserves a pause: ```seq - pos``` is wrapping ```uint32``` subtraction, and the ```int32(...)``` conversion reinterprets that wrapped difference as signed, turning "how far is this cell's sequence from my position" into a small window around zero that stays correct across the 2^32 wrap. That two-step - wrap unsigned, read signed - is the standard sequence-comparison idiom from TCP onward, and both steps are defined conversions here rather than implementation-defined casts.

```compareExchange``` is the claim: whichever producer swaps ```#tail``` from ```pos``` owns cell ```i``` exclusively until it publishes, so the plain payload write races with nothing. Everything is sequentially consistent because that is the only strength the extension offers; the notes weigh what that costs.

## Usage

One producer thread, one consumer thread, a million samples through a 1024-slot ring. ```callThread``` shares the queue by reference - same heap, no serialization boundary - and joining the returned promise is the shutdown.

```js
// main.js
import { SPSCQueue } from './ringbuffer.js';

value class Sample {
	sequence: uint32;
	channel: uint32;
	reading: float64;
}

const COUNT: uint32 = 1000000;
const queue = new SPSCQueue.<Sample, 1024>();

function consume(): uint64 { // Not async: it blocks on a dedicated core, and callThread wraps the Promise
	let sum: uint64 = 0;
	let sample: Sample;
	for (let received: uint32 = 0; received < COUNT; ) {
		if (queue.tryPop(ref sample)) {
			sum += uint64(sample.sequence);
			received++;
		}
	}
	return sum;
}

const consumer = consume.callThread();

let sample: Sample;
for (let i: uint32 = 0; i < COUNT; i++) {
	sample.sequence = i;
	sample.channel = i & 7;
	sample.reading = float64(i) * 0.5;
	while (!queue.tryPush(sample)) {} // Full means the consumer is behind; spin, it owns a core
}

const sum = await consumer; // Join; settles on this thread's loop
```

The consumer spins on purpose - it is a pinned core in this design, and ```popWait``` is the variant for a consumer that shares its thread with an event loop. An MPMC deployment differs only in spawning several of each: ```Promise.all([producer.callThread(), producer.callThread(), drain.callThread()])``` with an ```MPMCQueue``` behind them.

## Coverage Notes

This example exists to press on the threading extension's exact wording, and it found two places where the wording stops short of what the code needs, plus several places where the model is stronger than it first appears.

- **The ```Atomics``` forms do not compose to all typed storage, and the gap shaped the data structure.** The extension lists three targets: a typed binding via ```ref```, an object's own property by name, and a typed-array element by index. A private field - ```this.#tail``` - fits none of them literally: the property form takes a string key, which cannot name a private field, and a ```ref``` of a private member is itself refused, a private member not being a property of an ordinary base. The index is reachable because it sits in a cell: ```this.#tail.value``` is a field of the value the private field holds, which a ```ref``` may name, so this example writes ```Atomics.load(ref this.#tail.value)``` and reads "typed binding" as "any ```ref```-addressable typed storage." A field *of an array element* - the ```sequence``` field a cell-per-struct layout would want - has no form at all, which is why the MPMC queue stores sequences in a parallel ```[Capacity].<uint32>``` and addresses them with the element form. The structure-of-arrays split happens to be good for the payload's cache behavior, but the honest reason it is mandatory is that the atomics cannot reach into a struct in an array. The clean fix is to respecify ```Atomics``` over a single target kind - a reference to typed storage - which the ```ref``` machinery already knows how to denote for bindings, fields, and elements alike.
- **Thread creation and join need stated synchronization edges.** The MPMC constructor initializes every cell's sequence with plain writes, and the usage code builds the queue before spawning threads that read it. That is only correct if ```callThread``` establishes a happens-before edge from the spawning thread's prior writes to the new thread's start (and symmetrically from a thread's writes to the observation of its settled promise). Every threading system guarantees exactly this - it is why constructors do not need atomics - but the memory model section never says it. Without the edge, nothing written here before the spawn is formally visible to the child. One sentence in the memory model would close it: *spawn publishes everything sequenced before it; settlement publishes everything the thread did.*
- **The two-tier model reaches further than its size suggests.** Sequential consistency plus defined plain races turns out to cover this entire algorithm family: plain reads serve as the relaxed tier (own indices, staleness-tolerant probes), and the seq-cst pair provides publication, which subsumes acquire/release for these patterns. On current hardware the cost of the missing middle is small - a sequentially consistent load and store lower to the same acquire/release instructions ARM64 uses anyway - and where it genuinely bites is elsewhere: algorithms built on standalone fences or on relaxed *read-modify-writes* (statistics counters that want ```Atomics.add``` without ordering) pay full strength with no way to ask for less. That is a real but bounded gap, worth a sentence in the extension rather than a redesign.
- **Defined overflow is doing quiet correctness work in four places.** Monotonic ```uint32``` counters that wrap, occupancy by wrapping subtraction, slot mapping by mask, and the signed-window reinterpretation ```int32(seq - pos)``` - each is specified arithmetic here. The same lines in C depend on choosing unsigned types to dodge undefined behavior, and in Rust on remembering ```wrapping_sub``` at every site; a reviewer can check this code against the model instead of against a checklist of traps. The class-level ```where (Capacity & (Capacity - 1)) == 0``` completes it by making the power-of-two assumption a compile-time fact the arithmetic is entitled to.
- **Exclusion for wide payloads comes from the protocol, not the model - by design.** A multi-field ```T``` copies as multiple plain accesses, and the model says a raced copy may be observed torn. The sequence discipline ensures no slot is ever read while written, so the copies never race; the model's job is only to make the *indices* sound and to bound the failure if the protocol were wrong (a stale or mixed value, never heap corruption). This is the same contract C++ and Rust lock-free code operates under, with the blast radius of a bug reduced from undefined behavior to a wrong value.
- **The queue is written in parts, and the parts share its private fields.** Each method group above is a ```partial class``` beside the text that explains it, and every one of them reads ```#head```, ```#tail``` and ```#slots```. A part may name a private member of its class where it is in the file that declares the class, which all of these are; a part in another file could not, so the fields are as private as they would be in one class body.
- **An object places a field through the field's type.** The queue is shared by reference, so it is a class and not a ```value class```, and it has no layout: ```@align(64)``` written on its own fields would have nothing to place, and is an error for that reason rather than a request silently dropped. The alignment lives in ```Padded.<T>```, a value class whose layout is one field at 64 and whose size therefore rounds to 64, and shared storage keeps its target's size and alignment. That is one mechanism for a field of an object, a binding and an element of an array, where a decorator on the object's field would have covered only the first.
- **```ref``` out-parameters give the pop the shape lock-free APIs want.** ```tryPop(ref out)``` writes the payload through the caller's reference: no allocation on any path, no ```T | null``` box for value types that have no natural null, and a failed attempt touches nothing. The cost of success is one value type copy - the same copy any design must pay to get the payload out of the shared slot before recycling it. The one place the shape breaks is suspension: a reference cannot live across an ```await```, which is why ```popWait``` returns by value instead of writing through a caller's ref - the second-class rule holding at an async boundary the reference section never explicitly mentions.
