# Particle System

A fixed-capacity particle simulation of the kind that sits inside every game engine, written the way it would be in C# or C++: a contiguous pool of value type particles, integrated in place, updated across threads, and handed to the GPU as raw bytes. It exists to stress the proposal's performance core - value type classes, fixed-length arrays, references, dimensioned arithmetic from [primitive metadata](../primitivemetadata.md), and the [threading](../threading.md) extension - in one loop.

Features exercised, and why they matter here:

- A value type class makes ```[Capacity].<Particle>``` one allocation of interleaved fields with no per-particle object headers; assignment between slots is a byte copy, which is what makes swap-removal cheap.
- ```ref``` bindings mutate pool slots in place; plain reads copy, and the code relies on both behaviors deliberately.
- The ```Dimensions``` meta type turns the integration step into checked physics: ```Acceleration3 * Second``` is a ```Velocity3```, and transposing the update is a compile error, not a subtly wrong simulation.
- ```shared``` storage plus ```callThread``` splits integration across cores with no copying; the compaction phase stays single-threaded because it isn't race-free.
- A view over the pool is the GPU upload path: no serialization step, the simulation memory is the vertex buffer source.

## Types

```js
type Second = float32.<{ s: 1 }>;
type Position = vec3.<{ m: 1 }>;
type Velocity3 = vec3.<{ m: 1, s: -1 }>;
type Acceleration3 = vec3.<{ m: 1, s: -2 }>;

class Particle { // Every field is a value type, so Particle is one
	position: Position;
	velocity: Velocity3;
	lifetime: Second;
	size: float32.<{ exclusiveMinimum: 0 }>;
}
```

## The Pool

```js
export class ParticleSystem<Capacity: uint32 = 65536> {
	// One contiguous shared allocation: position, velocity, lifetime, size, repeated.
	#particles: shared [Capacity].<Particle>;

	// Written only on the owning thread; other threads read stale values harmlessly.
	#alive: uint32; // Defaults to 0

	#prng = Math.seededRandom.<float32>({ seed: 0 });

	@doc('Spawns a particle if capacity allows. Returns whether it did.')
	spawn(position: Position, velocity: Velocity3, lifetime: Second): boolean {
		if (this.#alive == Capacity) {
			return false;
		}
		const ref p = this.#particles[this.#alive++];
		p.position = position;
		p.velocity = velocity + vec3(
			this.#prng.random(-0.5, 0.5),
			this.#prng.random(-0.5, 0.5),
			this.#prng.random(-0.5, 0.5)
		);
		p.lifetime = lifetime;
		p.size = this.#prng.random(0.1, 1);
		return true;
	}

	@doc('Integrates a slice of the pool. Safe to run concurrently over disjoint slices.')
	#integrate(begin: uint32, end: uint32, dt: Second) {
		const gravity: Acceleration3 = vec3(0, -9.81, 0);
		for (let i: uint32 = begin; i < end; ++i) {
			const ref p = this.#particles[i];
			p.velocity += gravity * dt; // Acceleration3 * Second -> Velocity3
			p.position += p.velocity * dt; // Velocity3 * Second -> { m: 1 }
			p.lifetime -= dt;
			// p.position += p.velocity; // TypeError: { m: 1, s: -1 } is not assignable to { m: 1 }
		}
	}

	@doc('Removes expired particles by copying the last live particle into the hole.')
	#compact() {
		for (let i: uint32 = 0; i < this.#alive; ) {
			if (this.#particles[i].lifetime <= 0) {
				// Value semantics: this assignment copies the particle's bytes.
				this.#particles[i] = this.#particles[this.#alive - 1];
				--this.#alive;
				continue; // Re-run this index against the copied-in particle.
			}
			++i;
		}
	}

	@doc('One simulation step: integrate across threads, then compact on this one.')
	async update(dt: Second, threads: uint32 = 4) {
		// Typed integer division: chunk is a whole number of particles.
		const chunk = (this.#alive + threads - 1) / threads;
		const jobs: [].<Promise.<void, any>> = [];
		for (let t: uint32 = 0; t < threads; ++t) {
			const begin = t * chunk;
			const end = Math.min(begin + chunk, this.#alive);
			jobs.push((() => this.#integrate(begin, end, dt)).callThread());
		}
		await Promise.all(jobs); // Join before touching #alive again
		this.#compact();
	}

	@doc('The pool as raw bytes for the GPU: positions and sizes are already interleaved.')
	get vertexBytes(): [].<uint8> {
		return [].<uint8>(this.#particles).slice(0, this.#alive * Particle.byteLength);
	}
}
```

## The Frame Loop

A fixed timestep accumulates real frame time and steps the simulation in constant increments, the standard decoupling of simulation rate from display rate:

```js
const system = new ParticleSystem.<100000>();
const step: Second = 1 / 240;
let accumulator: Second = 0;
let last = performance.now();

async function frame(now: float64) {
	accumulator += Second((now - last) / 1000);
	last = now;
	while (accumulator >= step) {
		await system.update(step);
		accumulator -= step;
	}
	device.queue.writeBuffer(vertexBuffer, 0, system.vertexBytes); // WebGPU upload
	requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

## Coverage Notes

Writing this surfaced the following, most of which the proposal has since absorbed:

- **Reference iteration.** Resolved: the value type references section defines ```for (const ref p of this.#particles)```, so the hot loop iterates by reference with no index bookkeeping, as C#'s ```foreach (ref var p in span)``` does. Iterating several arrays in step, mutating one element of each, is the reference callback parameter idiom in the same section.
- **```Particle.byteLength```.** Resolved: value types and value type classes expose static ```byteLength``` and ```alignment```, which is what ```vertexBytes``` slices with.
- **Structure-of-arrays.** This example's pool is array-of-structures, which is what ```[N].<T>``` means. Declaring it ```shared SoA.<Particle, Capacity>``` instead, per the [structure of arrays](../soa.md) extension, changes no line of the code above: the element API and the ```ref``` bindings are identical. It buys a contiguous ```particles.fields.position``` to upload with no gather, a vectorizable pass per field, and no false sharing between threads writing different fields.
- **Dimensioned Math functions.** Resolved: ```min```, ```max```, ```abs```, ```clamp```, and the rounding functions preserve their argument's dimension in the [primitive metadata](../primitivemetadata.md) document, alongside ```sqrt``` and ```hypot```, while the transcendental functions stay dimensionless.
- **Atomic floats.** Resolved: ```Atomics.add``` and ```Atomics.sub``` accept float targets through a specified compare-exchange loop. A parallel reduction can use them, though this example would still accumulate per-thread partials and join them in a fixed order, since atomic float addition is not associative.
- **```#alive``` across threads.** Still open. Worker slices are computed from ```#alive``` before spawning and it's only written after the join, so the example is race-free by construction, but nothing *checks* that discipline. A way to mark a field as owned by a thread, the inverse of ```shared```, is deferred pending a second motivating case.
