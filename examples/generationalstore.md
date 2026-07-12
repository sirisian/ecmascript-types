# Generational Store

Stable references into a pool that recycles its storage. Any long-running system that hands out references to entries it also deletes has the same bug waiting: after slot 5 is freed and reused, everyone still holding "5" silently points at the new occupant - the wrong entity takes the damage, the stale texture binds to the mesh, the packet acts on someone else's session. Not a crash; a wrong answer. The generational store is the standard cure, and the reason every production entity-component system (Unity's ```EntityManager```, Bevy, EnTT, Flecs) keys entities this way: a handle is an index *plus the generation it was minted at*, freeing a slot bumps the generation, and a lookup with a stale handle fails loudly instead of hitting the impostor.

This example builds the store the way an ECS wants it - contiguous value type slots, allocation-free lookup, deferred structural changes - and uses it to test a specific question about the type system: can handles into *different* stores be made non-interchangeable at compile time, the way Rust's newtype indices are, without runtime cost? The answer turns out to be yes, through a phantom type parameter, and the notes at the end weigh what that refines about earlier findings and what gaps remain.

Features exercised, and why they matter here:

- A phantom-typed ```Handle<T>```: a two-field value class whose type parameter appears in no field, existing only to make ```Handle.<Enemy>``` and ```Handle.<Texture>``` distinct, non-converting types. Specialization identity plus nominal value classes deliver Rust's newtype-index pattern at zero bytes of cost.
- Parallel column storage - values, generations, free links - so a lookup is two indexed loads and a compare, live iteration walks one contiguous array, and freeing touches no value bytes.
- ```ref```-returning access: ```store.at(handle)``` yields a reference into the column for in-place mutation, throwing on a stale handle, with the reference-liveness rule then policing what may happen while the borrow lives.
- A command buffer for structural changes during iteration - the ECS idiom - motivated here by the liveness rule itself: ```insert``` can grow the columns, growth is forbidden while iteration holds references, and the compiler is the one that says so.
- Value type handles as ```Map``` keys: structural SameValueZero over the two fields, so per-entity side tables need no hand-written hashing or string keys.
- Defined ```uint32``` wraparound bounding the generation counter's behavior at the edge, rather than leaving 2^32 as a cliff.

## The Handle

A handle is eight bytes that name a slot at a moment in time. The type parameter is deliberately unused - a phantom, in Rust's vocabulary - and it is what makes handles from different stores different types. Two value classes never convert implicitly even when their layouts match, and two instantiations of one generic class are distinct types with distinct type objects, so ```Handle.<Enemy>``` is assignable to exactly one thing: ```Handle.<Enemy>```.

```js
// store.js
export const NULL_SLOT: uint32 = 0xffffffff;

export class Handle<T> { // T is phantom: no field uses it; it exists to make the type nominal per store
	readonly slot: uint32;
	readonly generation: uint32;
}
```

The fields are ```readonly``` because a handle is a fact, not a cursor: it names what it named when it was minted, forever. There is no constructor; the store mints handles by filling the layout, and nothing stops user code from forging one by the same spelling - the generation check below is what bounds the damage a forged or stale handle can do to "lookup fails."

## Slots and Liveness

Storage is three parallel columns plus a free-list head. A slot's generation is the liveness protocol in one integer: it starts at zero, incremented on every insert *and* every free, so a slot is live exactly when its generation is odd. A handle matches only if its generation equals the slot's current one, which after any intervening free it cannot - the bump at free instantly invalidates every outstanding handle to that slot, with no need to find them.

```js
// store.js
export class Store<T> {
	#values: [].<T>;
	#generations: [].<uint32>;
	#freeNext: [].<uint32>; // Threads the free list; meaningful only while a slot is free
	#freeHead: uint32 = NULL_SLOT;
	#liveCount: uint32 = 0;

	constructor(capacity: uint32 = 64) {
		this.#values = [].<T>.withCapacity(capacity);
		this.#generations = [].<uint32>.withCapacity(capacity);
		this.#freeNext = [].<uint32>.withCapacity(capacity);
	}

	get length(): uint32 { return this.#liveCount; }
	get capacity(): uint32 { return this.#values.capacity; }

	insert(value: T): Handle.<T> {
		let slot: uint32;
		if (this.#freeHead != NULL_SLOT) {
			slot = this.#freeHead;
			this.#freeHead = this.#freeNext[slot];
			this.#values[slot] = value;
			this.#generations[slot]++; // Even -> odd: live
		} else {
			slot = this.#values.length;
			this.#values.push(value); // May grow all three columns: no refs into the store may be live
			this.#generations.push(1); // First life of a fresh slot
			this.#freeNext.push(NULL_SLOT);
		}
		this.#liveCount++;
		return { slot, generation: this.#generations[slot] } := Handle.<T>;
	}

	has(handle: Handle.<T>): boolean {
		return handle.slot < this.#generations.length
			&& this.#generations[handle.slot] == handle.generation
			&& (handle.generation & 1) == 1;
	}

	// Copy the value out; false and out untouched when the handle is stale.
	tryGet(handle: Handle.<T>, ref out: T): boolean {
		if (!this.has(handle)) {
			return false;
		}
		out = this.#values[handle.slot];
		return true;
	}

	// A reference into the column, for in-place mutation. Stale handles throw:
	// use-after-free is an error at the lookup, never a read of the impostor.
	at(handle: Handle.<T>): ref T {
		if (!this.has(handle)) {
			throw new TypeError('Stale or foreign handle');
		}
		return ref this.#values[handle.slot];
	}

	despawn(handle: Handle.<T>): boolean {
		if (!this.has(handle)) {
			return false; // Double-despawn is a no-op, not a corruption
		}
		this.#generations[handle.slot]++; // Odd -> even: every outstanding handle to this slot is now stale
		this.#freeNext[handle.slot] = this.#freeHead;
		this.#freeHead = handle.slot;
		this.#liveCount--;
		return true;
	}
}
```

Two properties fall out of the column split and are worth noticing. A lookup is two indexed loads and a compare - no hashing, no probing, which is why an ECS keys *everything* this way. And ```despawn``` never touches ```#values```: the free link lives in its own column, so freeing a slot leaves the value bytes exactly as they were until an ```insert``` reuses them. That is not just tidiness; it is what makes despawn-during-iteration sound below.

## Iteration and Deferred Commands

Live-slot iteration is the ECS inner loop, so it takes the reference-callback form: the body receives a reference into the value column and the handle that names it, allocating nothing per element. The parity test is the skip.

```js
// store.js
partial class Store<T> {
	each(cb: (ref value: T, handle: Handle.<T>) => void) {
		for (let i: uint32 = 0; i < this.#values.length; i++) {
			const generation = this.#generations[i];
			if ((generation & 1) == 1) {
				cb(ref this.#values[i], { slot: i, generation } := Handle.<T>);
			}
		}
	}

	*operator...(): T { // Value copies, for consumers that want a plain for...of
		for (let i: uint32 = 0; i < this.#values.length; i++) {
			if ((this.#generations[i] & 1) == 1) {
				yield this.#values[i];
			}
		}
	}
}
```

What may the body of ```each``` do to the store? The liveness rule draws the line, and it draws it exactly where an ECS wants it drawn. ```despawn``` is safe: it bumps a generation and threads a free link, moves nothing, and overwrites no value bytes, so the reference the loop holds stays valid storage even if the body despawns the very element it is visiting. ```insert``` is not: it can ```push```, a push can reallocate the columns, and reallocation while a reference into them is live is a compile-time TypeError. The compiler has, in effect, rediscovered the ECS command-buffer discipline - structural *additions* are deferred to a flush point after iteration - as a consequence of array semantics rather than as a convention someone has to remember.

```js
// store.js
export class Commands<T> {
	#spawns: [].<T> = [];
	#despawns: [].<Handle.<T>> = [];

	spawn(value: T) { this.#spawns.push(value); }
	despawn(handle: Handle.<T>) { this.#despawns.push(handle); }

	flush(store: Store.<T>) {
		for (const handle of this.#despawns) {
			store.despawn(handle); // Stale entries no-op: the entity may have died twice this frame
		}
		this.#despawns.length = 0;
		for (const value of this.#spawns) {
			store.insert(value);
		}
		this.#spawns.length = 0;
	}
}
```

## Usage

Two stores, so the point about handle types has something to be a point about. The threat table shows a handle doing side-table duty as a ```Map``` key; the stale-handle sequence at the end is the bug class this structure exists to convert from silent corruption into a checked miss.

```js
// main.js
import { Store, Commands, Handle } from './store.js';

class Enemy {
	x: float32; y: float32;
	vx: float32; vy: float32;
	health: float32;
}

class Texture {
	id: uint32;
	width: uint32;
	height: uint32;
}

const enemies = new Store.<Enemy>(1024);
const textures = new Store.<Texture>(64);

const boss = enemies.insert({ x: 0, y: 0, vx: 0, vy: 0, health: 500 } := Enemy);
const grunt = enemies.insert({ x: 10, y: 5, vx: 1, vy: 0, health: 20 } := Enemy);

// In-place mutation through the ref return:
enemies.at(boss).health -= 50;

// A handle is typed by its store. The misuse is a compile error, not a garbage lookup:
// textures.at(boss); // TypeError: Handle.<Enemy> is not a Handle.<Texture>
// enemies.at(boss.slot); // TypeError: uint32 is not a Handle.<Enemy> - the raw index cannot skip the check

// Handles as keys: per-entity side data with structural equality, no string mangling.
const threat: Map.<Handle.<Enemy>, float32> = new Map.<Handle.<Enemy>, float32>();
threat.set(boss, 100);
threat.set(grunt, 5);

// A frame: iterate with refs, defer structural changes, flush after.
const commands = new Commands.<Enemy>();
function step(dt: float32) {
	enemies.each((ref e, handle) => {
		e.x += e.vx * dt;
		e.y += e.vy * dt;
		if (e.health <= 0) {
			commands.despawn(handle); // Safe mid-iteration: frees move no storage
			commands.spawn({ x: e.x, y: e.y, vx: 0, vy: 0, health: 10 } := Enemy); // Deferred: insert may grow
		}
	});
	commands.flush(enemies);
}

// The bug this exists to kill, run in slow motion:
const stale = grunt;
enemies.despawn(grunt);
const reused = enemies.insert({ x: 99, y: 99, vx: 0, vy: 0, health: 1 } := Enemy);
reused.slot == stale.slot;   // Likely true: the slot was recycled
enemies.has(stale);          // false - the generation moved on
let copy: Enemy;
enemies.tryGet(stale, ref copy); // false; copy untouched
// enemies.at(stale);        // TypeError: Stale or foreign handle
enemies.at(reused).health;   // 1 - the new occupant, reachable only through its own handle
```

## Coverage Notes

This example was written to settle a question two earlier examples raised from opposite sides, and its findings adjust one of them.

- **Phantom type parameters deliver the newtype-index pattern, and it costs nothing.** ```Handle<T>``` uses its parameter in no field, yet ```Handle.<Enemy>``` and ```Handle.<Texture>``` are fully distinct types: instantiations are identity-distinct per the generics semantics, and value classes never convert implicitly even at identical layout. Cross-store misuse and raw-index smuggling are both compile errors, which is precisely what Rust's one-line newtype wrappers buy - here it is one generic class for *all* stores rather than one wrapper per index type. This refines the finding from the bounding-volume example: the missing lightweight nominal alias matters specifically for *arithmetic-bearing* indices, the bare ```uint32``` a tree does ```& mask``` and ```+ 1``` on, where a wrapper's unwrap-rewrap ceremony fights every line. An opaque handle never does arithmetic, so the class wrapper has no ceremony to pay, and the pattern is simply available today.
- **There is no way to say "T must be a value type," and this store's guarantees silently assume it.** Contiguous columns, two-load lookups, copy-out ```tryGet```, despawn leaving value bytes untouched - every performance and soundness claim above holds because ```Enemy``` is a value type class. Instantiate ```Store.<SomeDynamicClass>``` and it still typechecks: the columns quietly become arrays of references, the locality story evaporates, and nothing warns. Rust encodes this exact requirement as implicit ```Sized``` plus explicit ```Copy``` bounds; this proposal has constraint families for widths (```extends uint```) but no ```extends value``` for the property that actually gates layout. A value-type bound is the one missing constraint this example genuinely needed.
- **The liveness rule rediscovered the command buffer.** Despawn-during-iteration is sound *by layout* - the free link lives in its own column, so freeing writes nothing a held reference can see - while insert-during-iteration is rejected *by the compiler*, because ```push``` can move the columns a live reference points into. That split (mutate in place freely, defer growth) is the discipline every ECS adopts by convention and Rust enforces by borrow; here it falls out of the array-liveness rule with the error message pointing at the exact call. The classic slotmap alternative, overlapping the free link onto the value bytes with an ```@offset``` union, would save the four bytes per slot and *lose* this property, since freeing would then scribble on storage a reference may still watch: the parallel-column layout is safer specifically under held references, and the cost of that safety is one ```uint32``` column.
- **Use-after-free is a checked runtime error, and that is the honest ceiling.** ```at``` on a stale handle throws; ```tryGet``` returns false; the impostor in a recycled slot is unreachable through the old handle. What no amount of generation counting provides is what an affine type system provides: making the *stale handle itself* unrepresentable, the way Rust's borrow checker retires a freed key's borrows at compile time. The store converts the silent-wrong-answer bug into a loud boundary error, which is the strongest guarantee available to a language without linear types, and it is worth stating as the deliberate trade rather than leaving a reviewer to discover the limit.
- **Structural value-type keys make handles first-class ```Map``` citizens for free.** ```Map.<Handle.<Enemy>, float32>``` compares keys field-wise with SameValueZero and copies them on insert, so the eight-byte handle is hashed as eight bytes with no boxing, no ```slot * BIG + generation``` encoding, and no accidental identity semantics. Side tables keyed by entity are the second-most-common ECS structure after the components themselves; they need zero support code here.
- **The generation counter's edge is defined, and small.** Generations are ```uint32``` under parity, so a single slot supports 2^31 insert/free cycles before its counter wraps and a truly ancient handle could match again. The wrap is defined arithmetic rather than overflow UB, the exposure window requires holding one handle across two billion reuses of one slot, and the standard hardening - widen to ```uint64```, or retire a slot at max generation - is a local change. Stated because ABA analyses die by omission, not because the risk is live.
