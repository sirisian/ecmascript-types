# Rust Scripting

A Rust application - a game engine, an editor, a server, a spreadsheet - owns every byte of its data, and embeds V8 through Deno to run the scripts. The host creates the world; the scripts read it and write into it. This is the oldest division of labor in software, and the reason it is painful today is not the embedding, which `deno_core` makes routine, but the boundary: JavaScript has no memory layout, so every byte that crosses has to be described twice and marshaled once.

The shape every such embedding grows is worth stating, because the sections below exist to delete it. The host tags its structs `#[repr(C)]`, a proc macro walks them with `offset_of!` and `size_of`, an exporter binary prints a TypeScript file of cursor classes wrapping `DataView.getFloat32(offset, true)` calls, and a developer reruns the generator after every component edit. The generated code works - `offset_of!` guarantees it - and everything about it is compensation. The offsets cross the boundary because only one side can compute them. The `DataView` calls exist because buffers are untyped and every read boxes into a double. The cursor object with its `setIndex` state exists because the language cannot say *a reference into memory*. The toolchain exists because a type cannot cross the boundary as a value.

This proposal removes the compensations at the root. The [memory layout](../memorylayout.md) extension makes layout normative - the default rule *is* the C, C++, and Rust rule - so both sides compute the same offsets from the same declarations and nothing needs to be exported. Array views bind directly to host memory, and an [```SoA``` view](../soa.md#views) binds a whole column block at once, so the host's storage *is* the script's storage: a column of the host's floats is a ```[N].<float32>```, a read of it is a typed load, and there is no per-field glue between them. Value types cross the boundary as data: an entity handle, an event, a command are eight or sixteen bytes both sides agree on. And because a type is a runtime value, the last piece of glue - checking that the two sides still agree - is an interned identity comparison at startup instead of a generated file and a prayer.

Features exercised: [```SoA.<T, N>``` views](../soa.md#views) over host-provided ```SharedArrayBuffer``` memory as the whole of the binding, ```fields``` projections and ```ref``` element bindings into that host storage, array views and ```window``` over an ```ArrayBuffer```, the normative layout rule with its compile-time ```byteLength``` and ```alignment```, ```Reflect.getReflection.<Reflect.ClassField, T>()``` reflecting declarations for the schema handshake, interned type identity as the comparison that handshake turns on, SIMD passes through ```float32x4``` views with scalar broadcast, enums with underlying-type layout crossing the boundary as bytes, generational ```Handle.<T>``` values from the [generational store](generationalstore.md) as the entity currency, typed rings of value types for commands and events, and exhaustive ```switch``` over event kinds.

## The Binding

The host exposes one operation, called once at startup. Everything else the scripts touch lives inside the memory it maps.

```js
const { buffer, schema, capacity } = Deno.core.ops.op_world_map();
```

```buffer``` is a ```SharedArrayBuffer``` over memory the host allocated and keeps; ```schema``` is the component table as data - for each component the base offset of its column block, that block's byte length, and a field list of names and primitive type names; and ```capacity``` is the host's row bound, the same ```MaxEntities``` choice the [entity component system](ecs.md) makes. This document keeps one flat table - the entity's slot is its row - so the boundary stays in focus; the archetype machinery that maps slots to moving rows is [ecs.md](ecs.md) and layers on unchanged.

The components are declared once, in the language that scripts them:

```js
import { Handle } from './store.js'; // The generational store example's handle

class Entity {} // The phantom parameter: Handle.<Entity> is nominal, so no bare uint32 becomes one

class Transform { x: float32; y: float32; rotation: float32; }
class Velocity { vx: float32; vy: float32; angularVel: float32; }
class Health { current: uint32; max: uint32; armorType: uint8; }
```

An entity is a ```Handle.<Entity>``` - a slot and the generation it was minted at - which is what a script names an entity by when it talks back to the host. Its ```slot``` is the row, so a handle indexes the columns directly and a stale one is detectable rather than silently pointing at whoever recycled the row.

A component's storage is one ```SoA``` per table, and the host's bytes are already in that layout, so binding is [the view form](../soa.md#views): a call on the type with the buffer and the block's offset.

```js
const MaxEntities: uint32 = 65536; // The host's capacity; a type's extent, so a constant on both sides
```

There is no binding code, because there is nothing left to bind. An ```SoA.<Transform, 65536>``` places its columns by the rule in the [SoA layout](../soa.md) section, the host placed its columns by the same rule, and the view is that layout over the host's base. No offsets are exported, no per-field views are constructed, no factory maps types to constructors, and nothing is generated.

Before trusting any of it, the script checks that the host's schema and its own declarations are the same types. A class is nominal - two identical declarations are different types by design, per [type objects](../typeobjects.md) - so the check compares fields, and it compares them by interned identity, which is a pointer comparison on type objects:

```js
const primitive = { float32, float64, uint8, uint16, uint32, int8, int16, int32 };

// Both halves matter. The field walk catches a renamed or retyped field. The byteLength catches a
// host that ran the layout rule differently - padded a column, swapped two fields, aligned to a
// different width - because SoA.<C, N>.byteLength is a compile-time constant and a different rule
// produces a different number.
function check<C>(block: { offset: uint32, byteLength: uint32, fields: [].<{ name: string, type: string }> }): void {
	const fields = Object.entries(Reflect.getReflection.<Reflect.ClassField, C>());
	if (fields.length != block.fields.length) {
		throw new TypeError(`${typename(C)}: host declares ${block.fields.length} fields, script declares ${fields.length}`);
	}
	for (let i: uint32 = 0; i < fields.length; ++i) {
		const [name, field] = fields[i];
		if (name != block.fields[i].name) { // Order is layout, so a swap is drift even when the set matches
			throw new TypeError(`${typename(C)}: field ${i} is '${name}' here and '${block.fields[i].name}' in the host`);
		}
		if (field.type !== primitive[block.fields[i].type]) { // Interned identity: one pointer comparison
			throw new TypeError(`${typename(C)}.${name}: host says ${block.fields[i].type}, script says ${typename(field.type)}`);
		}
	}
	if (block.byteLength != SoA.<C, MaxEntities>.byteLength) {
		throw new TypeError(
			`${typename(C)}: host carved ${block.byteLength} bytes, the type lays out ${SoA.<C, MaxEntities>.byteLength}`);
	}
}

// An extent is part of a type, so the script's storage is SoA.<C, MaxEntities> and the host's
// capacity is data. They have to be the same number, and this is where that is established.
if (capacity != MaxEntities) {
	throw new TypeError(`host capacity ${capacity}, script built for ${MaxEntities}`);
}
check.<Transform>(schema.Transform); // One call per component: a generic argument is compile-time,
check.<Velocity>(schema.Velocity);   // so this list is written out rather than looped over
check.<Health>(schema.Health);

// Everything agrees, so the views can be taken.
const world = {
	Transform: SoA.<Transform, MaxEntities>(buffer, schema.Transform.offset),
	Velocity: SoA.<Velocity, MaxEntities>(buffer, schema.Velocity.offset),
	Health: SoA.<Health, MaxEntities>(buffer, schema.Health.offset),
};
world.Transform.fields.x; // [65536].<float32>, aliasing the host's column
```

Editing a component on either side without the other now fails at startup with a named field, instead of running and corrupting. That one function is the replacement for the entire generated-bindings pipeline, and it is checked where the old pipeline was hoped.

Counts and cursors live inside the buffer too, in a header struct at offset zero, so the host never pushes state and the script never polls an op. A ```ref``` into a fixed view names the storage itself, and fixed views never move, so the binding is taken once:

```js
class WorldHeader {
	entityCount: uint32;
	commandHead: uint32; commandTail: uint32; // Script writes tail, host consumes head
	eventHead: uint32; eventTail: uint32;     // Host writes tail, script consumes head
}
const ref header = [].<WorldHeader>(buffer)[0]; // The header is at offset 0; a ref names the storage
header.entityCount; // Reads the host's count out of the host's memory
```

Growth is a policy, not a problem. With a bounded ```capacity``` nothing resizes, which is the choice above. A host that must grow uses a resizable buffer instead: views over one are length-tracking, growth never invalidates a view, and shrinking below a fixed view's extent detaches it so a stale access is a TypeError rather than a wild read.

## Scenarios

The scripting surface of a real host is not one thing. Eleven recur, across engines and applications alike: per-frame gameplay systems, AI behaviors, spawning and structural mutation, reacting to engine events, live tuning and inspection, modding with hot reload, UI logic, network packet validation, particle and VFX authoring, application automation such as user formulas and macros, and asset-pipeline hooks.

The five below were chosen to span the boundary rather than the genre. The first two are the two loop shapes - dense arithmetic over columns and branchy per-entity logic - which between them are most of what game scripts do. The third and fourth are the two directions of the boundary: scripts requesting structural changes the host must perform, and scripts consuming events the host produced. The fifth is the same machinery in an application rather than a game, with the reflection-driven inspector that generated code could never write. Of the rest, packet validation is [binarypacket.md](binarypacket.md), particles are [particlesystem.md](particlesystem.md), and hot reload turns out to need a paragraph rather than an example - it falls out of where the state lives, in the coverage notes.

## 1. Gameplay Systems Over Host Columns

The per-frame system is the reason the layout matters. Integration is one field read and one field written across every entity, and with the columns bound above it is the loop that would be written by hand:

```js
export function integrate(dt: float32): void {
	const t = world.Transform.fields, v = world.Velocity.fields;
	for (let i: uint32 = 0; i < header.entityCount; ++i) {
		t.x[i] += v.vx[i] * dt;
		t.y[i] += v.vy[i] * dt;
	}
}
```

Every access is a typed load or store on the host's memory. ```float32``` stays ```float32``` through the arithmetic, ```rotation``` and ```angularVel``` are other columns and never enter cache, and there is no cursor, no ```setIndex```, no boxing into a double and back. Where the body is pure arithmetic, the columns vectorize the way any ```[].<float32>``` does - a view of the four-aligned prefix, a broadcast constant, a scalar tail:

```js
const Gravity: float32 = -9.8;

export function applyGravity(dt: float32): void {
	const vy = world.Velocity.vy;
	const count = header.entityCount;
	const whole = count - count % 4;
	const lanes = [].<float32x4>(vy.window(0, whole)); // Same storage, four lanes
	const dv: float32x4 = Gravity * dt;                // A scalar assigned to a SIMD binding broadcasts
	for (let j: uint32 = 0; j < lanes.length; ++j) {
		lanes[j] += dv;
	}
	for (let i: uint32 = whole; i < count; ++i) { // Tail
		vy[i] += Gravity * dt;
	}
}
```

Two guarantees come from structure rather than analysis, as the [SoA](../soa.md) document derives: within one ```SoA``` the columns are disjoint extents at compile-time offsets, so a pass reading ```vy``` and writing ```y``` needs no runtime overlap check to vectorize, and the ```window``` view costs nothing because it is the same storage with a length. Both hold for a view exactly as they hold for an allocation, since a viewed ```SoA``` differs only in where its base came from.

A system that wants the element rather than the column takes a reference, which is a column set and an index and never a copy:

```js
export function damp(factor: float32): void {
	for (const ref v of world.Velocity) { // Reference iteration over the host's storage
		v.vx *= factor;
		v.vy *= factor;
	}
}
```

The host calls ```integrate``` and ```applyGravity``` by name each fixed step; the calls carry one typed scalar and no data, because the data never left.

## 2. Behavior Scripts

AI is the other loop shape: a branch per entity, state machines, early exits. The honest division the [ECS example](ecs.md) draws applies here - columns for arithmetic, scalar iteration for control flow - and the enum crossing the boundary is just bytes, since an enum has its underlying type's layout:

```js
enum AIState: uint8 { Idle, Seek, Flee, Attack }

class Brain {
	state: AIState;
	target: Handle.<Entity>; // Who the brain is tracking; .slot is that entity's row
	aggroRange: float32;
	fleeBelow: float32;      // Health fraction that flips Seek to Flee
}
check.<Brain>(schema.Brain);
world.Brain = SoA.<Brain, MaxEntities>(buffer, schema.Brain.offset);

export function think(dt: float32): void {
	const h = world.Health.fields;
	for (let i: uint32 = 0; i < header.entityCount; ++i) {
		const ref b = world.Brain[i]; // One reference, four columns, no copy
		switch (b.state) {
			case AIState.Idle: {
				if (distance(i, b.target.slot) < b.aggroRange) b.state = AIState.Seek;
				break;
			}
			case AIState.Seek: {
				if (float32(h.current[i]) < float32(h.max[i]) * b.fleeBelow) { b.state = AIState.Flee; break; }
				steer(i, b.target.slot, +40.0);
				break;
			}
			case AIState.Flee: {
				steer(i, b.target.slot, -60.0);
				break;
			}
			case AIState.Attack: {
				break; // Damage flows through the command ring in section 3
			}
		}
	}
}

function distance(i: uint32, j: uint32): float32 {
	const t = world.Transform.fields;
	const dx = t.x[j] - t.x[i], dy = t.y[j] - t.y[i];
	return Math.sqrt(dx * dx + dy * dy);
}

function steer(i: uint32, target: uint32, speed: float32): void {
	const t = world.Transform.fields, v = world.Velocity.fields;
	const dx = t.x[target] - t.x[i], dy = t.y[target] - t.y[i];
	const len = Math.max(Math.sqrt(dx * dx + dy * dy), 0.0001);
	v.vx[i] = dx / len * speed;
	v.vy[i] = dy / len * speed;
}
```

```b.state``` and ```b.aggroRange``` are reads of two different columns at the same index, which is what a reference into an ```SoA``` is, and assigning ```b.state``` writes the state column at that row of the host's memory. The state machine's whole vocabulary - the enum, the tuning fields, the switch - is ordinary typed script. Designers edit ```aggroRange``` and ```fleeBelow``` per entity from the host's tools, the script reads the columns live, and the exhaustive switch means a new state added to the enum is a compile error at every ```switch``` that ignores it.

## 3. Structural Changes Through a Command Ring

Scripts must not spawn or despawn by poking memory: a structural change moves storage, and the host owns storage. The [ECS example](ecs.md) routes all structural work through a command buffer applied at flush points between phases; across an embedding boundary the same discipline becomes a typed ring in the shared buffer - the [ring buffer](ringbuffer.md) example's structure, at its minimal single-producer size - script as producer and host as consumer. The command is a value type both sides declared, the entity currency is the generational ```Handle.<Entity>``` from the [generational store](generationalstore.md) - eight bytes, slot plus generation - so a stale despawn misses instead of killing the slot's new occupant:

```js
enum Cmd: uint8 { Spawn, Despawn, DealDamage }

class Command {
	kind: Cmd;
	entity: Handle.<Entity>; // Ignored by Spawn
	x: float32;              // Spawn position, or damage amount in x
	y: float32;
}

const RingSize: uint32 = 4096; // Power of two; the host sized the region for it
const commands = [RingSize].<Command>(buffer, schema.commandRing.offset);

export function enqueue(command: Command): boolean {
	if (header.commandTail - header.commandHead == RingSize) return false; // Full: the host drains between phases
	commands[header.commandTail & (RingSize - 1)] = command;               // A value type scatters into the slot
	header.commandTail += 1;
	return true;
}

// The Attack case from section 2, now that the vocabulary exists:
enqueue({ kind: Cmd.DealDamage, entity: b.target[i], x: 12.5, y: 0.0 } := Command);
```

The host drains ```commandHead``` to ```commandTail``` at its flush point, validates each handle's generation, performs the mutation, and advances the head. A command naming an entity that died earlier in the same frame fails that check and is dropped, which is the whole reason the currency is a handle rather than a row: a script cannot damage whoever inherited the row. Ordering is the same-thread embedding's gift: the host calls the script, the script returns, the host drains - no atomics, no torn command, one writer per cursor. The threaded upgrade is mechanical and covered in the notes. What matters architecturally is that the script's *authority* is exactly the command vocabulary: a script can request anything in the enum and nothing outside it, which is a capability list the host wrote as a type.

## 4. Consuming Engine Events

Events run the same ring the other direction. Physics contacts, damage resolution, pickups - the host produces them during its own phases, and the script reacts in its next call. The event is again a value type, the handles again generation-checked, and the reaction writes columns or enqueues commands, closing the loop with section 3:

```js
enum EventKind: uint8 { Damage, Collision, Pickup }

class GameEvent {
	kind: EventKind;
	a: Handle.<Entity>;
	b: Handle.<Entity>; // Damage source, collision partner, or the item
	amount: float32;
}

const events = [RingSize].<GameEvent>(buffer, schema.eventRing.offset);

export function pump(): void {
	const health = world.Health;
	for (; header.eventHead != header.eventTail; header.eventHead += 1) {
		const e = events[header.eventHead & (RingSize - 1)]; // A value copy: the slot is reusable immediately
		switch (e.kind) {
			case EventKind.Damage: {
				const row = e.a.slot;
				const dealt = uint32(Math.min(float32(health.current[row]), e.amount));
				health.current[row] -= dealt;
				if (health.current[row] == 0) enqueue({ kind: Cmd.Despawn, entity: e.a, x: 0.0, y: 0.0 } := Command);
				break;
			}
			case EventKind.Collision: {
				break; // Gameplay rules go here; the physics already resolved
			}
			case EventKind.Pickup: {
				break;
			}
		}
	}
}
```

Note what did not happen: no event object was allocated, no JSON crossed the boundary, no callback was registered per event. The host wrote sixteen bytes into a ring it owns; the script read them as the type it declared. Reading an element of a value type view copies it, so the ring slot is free the moment the read completes, and the copy lives in registers for the length of a switch arm.

## 5. An Application's Formulas

Nothing above is about games. Replace the world with a data grid - a Rust analytics tool, a report engine, a spreadsheet - and the same binding gives user formulas that are compact, typed, and allocation-free. The host owns the columns, including the output column the formula fills:

```js
class Ledger { revenue: float64; cost: float64; margin: float64; }
check.<Ledger>(schema.Ledger);
const rows = SoA.<Ledger, MaxEntities>(buffer, schema.Ledger.offset);

// The entire user script:
export function recompute(count: uint32): void {
	for (const ref r of rows) {
		r.margin = r.revenue == 0.0 ? 0.0 : (r.revenue - r.cost) / r.revenue;
	}
}

export function totalRevenue(count: uint32): float64 {
	const revenue = rows.fields.revenue; // The column, not the rows: cost and margin stay out of cache
	let sum: float64 = 0.0;
	for (let i: uint32 = 0; i < count; ++i) sum += revenue[i];
	return sum; // Declared float64: the accumulator never widens and never boxes
}
```

Three lines of loop is the whole formula, and it runs at column speed because it is a column loop. The declared ```float64``` return is about the *script's* arithmetic rather than the boundary - the accumulator stays a machine double through the loop instead of being a number the engine re-speculates on every add. What the embedder receives at the call is still whatever the host's binding API hands back, which is why the results that matter go in the buffer. Money that must not accumulate binary error points at the [decimal](../decimal.md) extension; the binding is identical, only the element type changes. And because the schema is a value, the inspector that every such application grows - the property panel, the column legend, the debug dump - is a loop over reflection rather than a code generator:

```js
export function describe<C>(): void {
	for (const [name, field] of Object.entries(Reflect.getReflection.<Reflect.ClassField, C>())) {
		console.log(`${name}: ${typename(field.type)}, ${field.type.byteLength} bytes per row`);
	}
	console.log(`${MaxEntities} rows, ${SoA.<C, MaxEntities>.byteLength} bytes of columns`);
}
describe.<Ledger>();
// revenue: float64, 8 bytes per row
// cost: float64, 8 bytes per row
// margin: float64, 8 bytes per row
// 65536 rows, 1572864 bytes of columns
```

One thing the inspector deliberately does not print is ```field.offset```. ```Reflect.getReflection.<Reflect.ClassField, Ledger>('cost').offset``` is ```8```, the field's offset inside one interleaved ```Ledger```, which is the right answer to a question this storage isn't asking: under columns a field's address is a column base and a row index, and the base belongs to the ```SoA``` rather than to the field. The declared offsets still matter - they are what a packet or a file uses - which is why reflection reports the declaration and lets the storage decide what to make of it.

That loop is the exporter binary from the old pipeline, inverted: instead of Rust printing what JavaScript cannot know, JavaScript reads what both sides always knew.

## The Host Side

What remains in Rust is one allocation, one op, and a drain - and a simplification of its own. Under columns, the host's storage is ```Vec<f32>``` per field or one carved region; the ```#[repr(C)]``` structs, the ```offset_of!``` calls, the proc-macro crate, and the exporter binary all served the interleaved layout and the untyped consumer, and none survives the change of either.

```rust
// deno_core, sketched. One region, carved by the rule SoA.<T, N> lays out - which is what
// makes the script's view of it legal, and what its byteLength check verifies.
fn column_offsets(fields: &[(usize, usize)], cap: usize, column_align: usize) -> (Vec<usize>, usize) {
	let mut offset = HEADER_AND_RINGS; // WorldHeader + the two rings, laid out first
	let bases = fields.iter().map(|&(size, align)| {
		let a = align.max(column_align);
		offset = (offset + a - 1) & !(a - 1);
		let base = offset;
		offset += size * cap;
		base
	}).collect();
	(bases, offset)
}

#[op2]
fn op_world_map(state: &mut OpState) -> WorldMap {
	// The buffer is a SharedArrayBuffer whose backing store the host allocated and retains.
	// The schema is the same four-line table the offsets came from: data, not generated code.
	WorldMap { buffer, schema, capacity: MAX_ENTITIES, column_align: VECTOR_ALIGN }
}

// Per frame: call the phases, drain the ring at the flush point. `call` here stands in for
// the embedder's usual dance - resolve the module's export once, cache the v8::Function,
// invoke it with a scope - which is unchanged by any of this and so is elided.
call(&mut runtime, "think", dt)?;
call(&mut runtime, "integrate", dt)?;
drain_commands(&mut world, header); // Validates each handle's generation before acting
call(&mut runtime, "pump", ())?;
```

The schema table is the one artifact that must exist twice, once as these Rust field tuples and once as the script's class declarations, and the startup handshake is what makes that duplication safe: generate either from the other, or from a common four-line source, and a mismatch is a named TypeError before the first frame rather than corruption during it.

## Coverage Notes

**Reflection order is load-bearing here, and unstated.** The handshake walks ```Reflect.getReflection.<Reflect.ClassField, C>()``` and compares it position by position against the host's field list, so the walk must yield fields in declaration order or it reports drift that isn't there. That the *layout* follows declaration order is normative - [memory layout](../memorylayout.md) says the proposal never reorders, because views, serialization, and interop depend on the declared order, and [SoA](../soa.md) says a byte view sees columns in declaration order - but the [decorators](../decorators.md) extension doesn't say what order reflection reports members in, and a walk in any other order would make the handshake report a swap on every startup. The view form contains the damage - the engine derives the column layout internally, from the rule it owns, so a misordered reflection can no longer mis-bind anything silently - but the check still needs the order to mean something. The fix is a sentence in the reflection API rather than anything here: members are reported in declaration order, which is the order the layout rule already gives them.

**The view form is what makes this document short.** Everything above rests on [```SoA```'s view constructor](../soa.md#views): the host lays out columns by the rule, the script names the same layout as a type, and the binding is a call. Without it the script would have to project one ```[N].<F>``` view per field, which is the same bytes but not the same type - no ```fields```, no element gather or scatter, no ```ref``` iteration, and a compile-time type function plus a per-type factory to build the record at all, because a field's type is a runtime value inside a loop over reflection. That is roughly forty lines of binding, an ```any``` in the middle of it, and none of the element API. The one thing the view form asks in exchange is that the host keep a component's columns in one block, which is what a chunked ECS already does and what the ```byteLength``` check enforces.

**The flush discipline is real, and it is the same one.** A script must not hold a ```ref``` into storage across a host structural change, exactly as the [ECS example](ecs.md) forbids holding references across its own flush points. The embedding makes the discipline easy to keep - the host mutates only between script calls, and a ```ref``` bound inside a call dies with it - but it is an architecture, not an accident, and the header-plus-rings design above exists to give structural work exactly one door.

**Nominal classes made the handshake honest.** Two identical class declarations are different types on purpose, so the schema check compares fields by name and interned type identity rather than comparing class objects. That is the right granularity anyway: it reports *which field* drifted, and interned identity means each comparison is one pointer test.

**Same thread now, ```shared``` later.** Everything above assumes the common embedding, where the host thread calls the script and no one else touches the buffer meanwhile, so the ring cursors are plain fields. Moving a system to another thread is the [threading](../threading.md) extension's territory: the buffer is already shared memory, the cursors become ```Atomics``` operations on the header's fields, and the columns' one structural gift carries over - two threads writing different fields of the same entities never share a cache line, because different fields are different columns.

**Strings do not cross by view, and should not.** A ```string``` has no layout - the [memory layout](../memorylayout.md) table is explicit that its size is a property of the value - so names, dialogue, and paths cross as ids and enums, with the text living on whichever side displays it. A host that must ship variable bytes through the buffer uses a length-prefixed ```uint8``` region and the [serializer](serializer.md) example's machinery; the rings above deliberately carry fixed-size value types so a slot is a slot.

**Hot reload is a re-import, because the state never lived in the script.** Every durable byte - positions, health, brains, the rings, the counts - is in the host's buffer. Reloading a module rebinds views over unchanged memory and re-runs the schema check; a reload after a component edit fails the handshake by name, which is the moment a developer wants to hear about it. The scripts are stateless functions over shared state, which is what makes them scripts.

**The op surface is the sandbox.** A script's write authority is the columns it was given and the command vocabulary the host declared; everything else - files, network, process - is a Deno op the host chose to register or did not. The security review of a mod is the schema and the enum, which fit on one screen.
