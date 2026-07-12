# Entity Component System

An archetype-based ECS for a networked game server: entities are rows in tables grouped by component composition, systems iterate matching tables, and sessions receive delta-replicated state for the entities their view touches. This is the design Unity DOTS, Bevy, and Flecs converge on, and it's the harshest test of a type system's generic machinery, because the whole point is storing *differently typed* columns behind *one* storage abstraction without boxing.

Two pieces come from the [generational store](generationalstore.md) example rather than being rebuilt here. The entity id is its phantom-typed `Handle` - `Handle.<Entity>`, a slot plus the generation it was minted at - so a recycled slot is unreachable through a stale reference and a raw index cannot skip the check. And structural changes route through a command buffer to flush points between phases, the discipline that example's coverage notes derive from the reference-liveness rule. The archetype layout sharpens both: swap-removal means even *destruction* moves storage here, so the buffer carries every structural operation, not just the growing ones.

Converted from a TypeScript implementation, and the conversion has a theme: **much of the original code existed to fight the allocator, and value types delete it.** The `BitSet` pool is gone because bitsets copy by assignment, and its `toKey` string hashing is gone because a value type is a structural `Map` key. The reused scratch object threaded through a generator is gone because yielding a value type is already a copy. The four parallel arrays hand-rolling structure-of-arrays for the transition log are one `[].<Transition>` because a value type array is already contiguous. The per-session map of entity id to generation is gone because the entity handle carries the generation the world minted. The `as` casts on every column access became typed accessors. The string-literal unions became enums with exhaustive switches.

Features exercised: generational entity handles through the phantom-typed `Handle` of the [generational store](generationalstore.md), a deferred command buffer for structural changes, value type classes as component storage and as bitmasks, structure-of-arrays columns through [`SoA.<T>`](../soa.md), `readonly` fields, `[N].<T>` fixed arrays with `byteLength`-based view windows, `ref` element bindings, `ref` callback parameters, and `ref`-returning accessors as the data API, enums replacing string vocabularies with sentinel-value arithmetic, type objects as `Map` keys for resources and events, typed generators, SIMD passes over field columns through ordinary array views, and a compile-time function in type position mapping component ids to component types, the generic machinery this example leans on hardest, detailed in the coverage notes.

## Components

```js
import { Handle, NULL_SLOT } from './store.js'; // The generational store example's handle and free-list sentinel

enum Component: uint8 {
	Transform, Velocity, Collider, RigidBody,
	Sprite, Light, AnimationState,
	Health, Armor, Damage,
	Interactable, Children, Parent, RelativeTransform,
	AIController, Patrol, Faction,
	Destructible, Flammable,
	SpatialCell, PreviousTransform, RenderTransform,
	Static, Projectile, Replicated, Session,
	__COUNT,

	// Disabled variants keep their data but drop out of normal queries.
	// Enum values referencing previous values places them after the reals.
	DisabledVelocity = __COUNT,
	DisabledCollider, DisabledLight, DisabledSprite, DisabledInteractable,
	__TOTAL,
};

const TotalComponents: uint8 = Component.__COUNT;
const TotalBits: uint8 = Component.__TOTAL;
const Words: uint32 = (TotalBits + 31) / 32; // Typed integer division
const MaxEntities: uint32 = 65536; // Bounds the slot, so every per-entity table is a flat array
const MSB: uint32 = 0x80000000;

// The phantom parameter for this world's handles. Handle.<Entity> is nominal:
// no raw uint32, and no handle minted by another store, converts to it.
class Entity {}
const NULL_ENTITY: Handle.<Entity>; // A typed const takes its type's default: slot 0, generation 0. Even, so never live

const disabledOf: [TotalComponents].<Component>; // Zero-filled; 0 (Transform) means none
const enabledOf: [TotalBits].<Component>;

function registerDisabledVariant(enabled: Component, disabled: Component) {
	disabledOf[enabled] = disabled;
	enabledOf[disabled] = enabled;
}
registerDisabledVariant(Component.Velocity, Component.DisabledVelocity);
registerDisabledVariant(Component.Collider, Component.DisabledCollider);
registerDisabledVariant(Component.Light, Component.DisabledLight);
registerDisabledVariant(Component.Sprite, Component.DisabledSprite);
registerDisabledVariant(Component.Interactable, Component.DisabledInteractable);
```

An entity is a `Handle.<Entity>` from the [generational store](generationalstore.md): eight bytes naming a slot at a moment in time. The slot stays below `MaxEntities`, so every per-entity structure - locations, dirty masks, replication entries - is a flat array indexed by `entity.slot`, a direct offset with no hashing, and the generation makes a recycled slot's old handles miss instead of aliasing the new occupant. The zero handle doubles as the none: its generation is even, so it can never match a live slot, which `removeEntity` and the command buffer both lean on below.

Component data are value type classes wherever the data is plain, which is what makes a column of them contiguous memory. String vocabularies from the original (`type: "box" | "circle"`, `state: string`) became enums, both for the exhaustive switches and because an enum field keeps a class a value type where a `string` field would not:

```js
class Vec2 {
	x: float32;
	y: float32;
}
class Transform {
	x: float32;
	y: float32;
	rotation: float32;
}
class Velocity {
	vx: float32;
	vy: float32;
	angularVel: float32;
}
enum ColliderType: uint8 { Box, Circle };
class Collider {
	type: ColliderType;
	width: float32;
	height: float32;
	radius: float32; // Unused for Box; value types have no optional fields
}
class RigidBody {
	mass: float32.<{ exclusiveMinimum: 0 }>;
	friction: float32;
	restitution: float32;
}
class Health {
	current: float32;
	max: float32;
} where this.current <= this.max;
class Armor {
	defense: float32;
}
class Damage {
	amount: float32;
}
class SpatialCell {
	cellId: uint32;
}
class Projectile {
	origin: Vec2;
	direction: Vec2;
	speed: float32;
	maxRange: float32;
}
enum AIState: uint8 { Idle, Patrolling, Chasing };
class AIController {
	state: AIState;
}
class Patrol {
	waypoints: [].<Vec2>; // A reference field: Patrol is not a value type, and that's fine
	currentIndex: uint32;
	speed: float32;
}
// Reference components (Session below), marker components (Static, Replicated),
// and the remaining gameplay data classes follow the same two shapes.
```

The map from a component id to its data type is a compile-time function returning a type object. Types are values in this proposal, so a `switch` over an enum can return them, and using the call in type position is what lets one generic accessor serve every component:

```js
function componentType(c: Component): type {
	switch (c) {
		case Component.Transform: return Transform;
		case Component.Velocity: return Velocity;
		case Component.Collider: return Collider;
		case Component.RigidBody: return RigidBody;
		case Component.Health: return Health;
		case Component.Armor: return Armor;
		case Component.Damage: return Damage;
		case Component.SpatialCell: return SpatialCell;
		case Component.PreviousTransform: return Transform;
		case Component.RenderTransform: return Transform;
		case Component.Session: return Session;
		case Component.DisabledVelocity: return Velocity;
		// ... one case per real and disabled component; the __COUNT/__TOTAL
		// sentinels case to throw, which keeps the switch exhaustive
	}
}
```

## BitSet

A mask over the component bits, MSB-first so `Math.clz32` walks set bits. The one field is a *fixed* array of `uint32`, which is inline storage, so `BitSet` is a **value type**: assignment copies it, and the original's pool (`acquireBitSet`/`releaseBitSet`) and `copyFrom` method are deleted outright. Being a value type also makes it a structural `Map` key, which deletes the original's `toKey()` string hashing. On `uint32`, `>>` is already a logical shift, so the original's `>>>` is gone too.

```js
class BitSet {
	readonly words: [Words].<uint32>; // Zero-filled

	add(bit: uint8) {
		this.words[bit / 32] |= MSB >> (bit & 31);
	}
	remove(bit: uint8) {
		this.words[bit / 32] &= ~(MSB >> (bit & 31));
	}
	has(bit: uint8): boolean {
		return (this.words[bit / 32] & (MSB >> (bit & 31))) != 0;
	}
	clear() {
		this.words.fill(0);
	}
	isEmpty(): boolean {
		for (const word of this.words) {
			if (word != 0) {
				return false;
			}
		}
		return true;
	}
	containsAll(other: BitSet): boolean {
		for (let w: uint32 = 0; w < Words; ++w) {
			if ((this.words[w] & other.words[w]) != other.words[w]) {
				return false;
			}
		}
		return true;
	}
	containsAny(other: BitSet): boolean {
		for (let w: uint32 = 0; w < Words; ++w) {
			if ((this.words[w] & other.words[w]) != 0) {
				return true;
			}
		}
		return false;
	}
}

function buildMask(components: [].<Component>): BitSet {
	let mask: BitSet; // Default-initialized, all zero
	for (const c of components) {
		mask.add(c);
	}
	return mask;
}
```

## Archetypes

An archetype owns one column per component in its mask. Columns hold differently typed elements, so the container is typed `any` and the typed view is recovered through `componentType` at the accessor - the honest shape of a dependently indexed store.

Per-component columns are already structure-of-arrays at component granularity. Storing each column as an [```SoA.<T>```](../soa.md) takes the layout to the field level: every ```x``` contiguous, every ```y``` contiguous, split at the element's immediate fields. A system reading two fields of a five-field component pulls only those cache lines, and each field is a dense stream a compiler can vectorize. The element API matches ```[].<T>``` - indexing, ```ref``` bindings, ```push```, and the swap-remove all work identically - so nothing below changes with the layout:

```js
class EntityLocation {
	archetype: Archetype; // Always constructed with one; a slot with no entity is null in the table instead
	row: uint32;
}

// The value-level twin of componentType. With a runtime id, SoA.<componentType(id)>
// can't be written in type position, so each column type is constructed explicitly.
function newColumn(id: Component): any {
	switch (id) {
		case Component.Transform: return new SoA.<Transform>();
		case Component.Velocity: return new SoA.<Velocity>();
		case Component.Collider: return new SoA.<Collider>();
		case Component.RigidBody: return new SoA.<RigidBody>();
		case Component.Health: return new SoA.<Health>();
		case Component.Armor: return new SoA.<Armor>();
		case Component.Damage: return new SoA.<Damage>();
		case Component.SpatialCell: return new SoA.<SpatialCell>();
		case Component.PreviousTransform: return new SoA.<Transform>();
		case Component.RenderTransform: return new SoA.<Transform>();
		case Component.Session: return new SoA.<Session>();
		case Component.DisabledVelocity: return new SoA.<Velocity>();
		// ... one case per component, the same set componentType maps
	}
}

class Archetype {
	readonly mask: BitSet; // A value type field: the mask is stored inline
	readonly #columns: [TotalBits].<any>; // SoA.<componentType(id)> or null per slot
	readonly entities: [].<Handle.<Entity>> = [];

	constructor(mask: BitSet) {
		this.mask = mask; // Copies the value
		for (let w: uint32 = 0; w < Words; ++w) {
			let word = mask.words[w];
			while (word != 0) {
				const bit = Math.clz32(word);
				const id = Component(w * 32 + bit);
				this.#columns[id] = newColumn(id);
				word ^= MSB >> bit;
			}
		}
	}

	get length(): uint32 {
		return this.entities.length;
	}

	column<C: Component>(): SoA.<componentType(C)> {
		return this.#columns[C];
	}
	// The untyped escape hatch: when the component id is a runtime value its column
	// type can't be named in type position, so callers that only move the data
	// opaquely take it as any.
	columnAny(comp: Component): any {
		return this.#columns[comp];
	}
	hasColumn(comp: Component): boolean {
		return this.#columns[comp] != null;
	}

	// data is a flat scratch indexed by component id.
	addEntity(entity: Handle.<Entity>, data: [TotalBits].<any>): uint32 {
		const row = this.entities.length;
		this.entities.push(entity);
		for (let id: uint8 = 0; id < TotalBits; ++id) {
			this.#columns[id]?.push(data[id]);
		}
		return row;
	}

	// Swap-removes the row. Returns the handle that moved into it, or the zero
	// handle - which is never live, so callers resolve it like any stale one.
	removeEntity(row: uint32): Handle.<Entity> {
		const lastRow = this.entities.length - 1;
		if (row != lastRow) {
			this.entities[row] = this.entities[lastRow];
			for (let id: uint8 = 0; id < TotalBits; ++id) {
				const column = this.#columns[id];
				if (column != null) {
					column[row] = column[lastRow]; // Copies each field column's slot
				}
			}
		}
		this.entities.pop();
		for (let id: uint8 = 0; id < TotalBits; ++id) {
			this.#columns[id]?.pop();
		}
		return row != lastRow ? this.entities[row] : NULL_ENTITY;
	}
}
```

Each field of an ```SoA``` projects as a typed array view aliasing that column, which is how one attribute streams out without touching the rest:

```js
const transforms = archetype.column.<Component.Transform>();
transforms.fields.x; // [].<float32>: every x in this archetype, aliasing storage
transforms.fields.rotation.fill(0); // Writes one field column across all rows
```

A `PreparedQuery` caches the archetypes matching a mask, and is updated when a new archetype appears:

```js
class PreparedQuery {
	readonly mask: BitSet;
	readonly withoutMask: BitSet;
	readonly archetypes: [].<Archetype> = [];

	constructor(required: [].<Component>, without: [].<Component> = []) {
		this.mask = buildMask(required);
		this.withoutMask = buildMask(without);
	}

	matches(archetype: Archetype): boolean {
		return archetype.mask.containsAll(this.mask)
			&& !archetype.mask.containsAny(this.withoutMask);
	}

	// The iteration API: ref parameters bind column slots, so the callback
	// mutates storage in place with no copies and nothing allocated.
	each<C1: Component>(callback: (entity: Handle.<Entity>, ref a: componentType(C1)) => void) {
		for (const archetype of this.archetypes) {
			const columnA = archetype.column.<C1>();
			const entities = archetype.entities;
			for (let i: uint32 = 0; i < archetype.length; ++i) {
				callback(entities[i], ref columnA[i]);
			}
		}
	}
	each<C1: Component, C2: Component>(callback: (entity: Handle.<Entity>, ref a: componentType(C1), ref b: componentType(C2)) => void) {
		for (const archetype of this.archetypes) {
			const columnA = archetype.column.<C1>();
			const columnB = archetype.column.<C2>();
			const entities = archetype.entities;
			for (let i: uint32 = 0; i < archetype.length; ++i) {
				callback(entities[i], ref columnA[i], ref columnB[i]);
			}
		}
	}
	// Three- and four-component overloads follow the same shape.
}
```

## The World

The World is the [generational store](generationalstore.md)'s protocol grown archetype columns. `entityLocations` plays the value column; a generation column implements the odd-is-live parity; the free list threads through its own fixed column, since `MaxEntities` bounds every slot and nothing here ever grows. What the store bought for one pool it buys here for every table keyed by entity: each accessor and each structural method resolves the handle through one `locationOf`, so a stale handle fails the generation check there and use-after-destroy is a checked miss everywhere, never a read of whichever entity recycled the slot.

```js
class World {
	#nextSlot: uint32;
	readonly #generations: [MaxEntities].<uint32>; // Zero-filled; odd is live, exactly the store's protocol
	readonly #freeNext: [MaxEntities].<uint32>; // Threads the free list; meaningful only while a slot is free
	#freeHead: uint32 = NULL_SLOT;
	readonly archetypes: [].<Archetype> = [];
	readonly #archetypeIndex = new Map.<BitSet, Archetype>(); // A value type key, compared structurally
	readonly entityLocations: [MaxEntities].<EntityLocation | null>; // Defaults to null
	readonly #queries: [].<PreparedQuery> = [];
	#migrateData: [TotalBits].<any>;

	// The store's has(), against this world's columns: in range, matching
	// generation, and odd. The zero handle fails the parity test.
	has(entity: Handle.<Entity>): boolean {
		return entity.slot < MaxEntities
			&& this.#generations[entity.slot] == entity.generation
			&& (entity.generation & 1) == 1;
	}
	// Every read and every structural change resolves through this, which is
	// where a stale handle - or a forged one - becomes a miss.
	locationOf(entity: Handle.<Entity>): EntityLocation | null {
		return this.has(entity) ? this.entityLocations[entity.slot] : null;
	}

	compileQuery(required: [].<Component>, without: [].<Component> = []): PreparedQuery {
		const query = new PreparedQuery(required, without);
		for (const archetype of this.archetypes) {
			if (query.matches(archetype)) {
				query.archetypes.push(archetype);
			}
		}
		this.#queries.push(query);
		return query;
	}

	#getOrCreateArchetype(mask: BitSet): Archetype {
		let archetype = this.#archetypeIndex.get(mask);
		if (archetype == null) {
			archetype = new Archetype(mask);
			this.archetypes.push(archetype);
			this.#archetypeIndex.set(mask, archetype); // Copies the mask in
			for (const query of this.#queries) {
				if (query.matches(archetype)) {
					query.archetypes.push(archetype);
				}
			}
		}
		return archetype;
	}

	spawn(components: [].<ComponentInit>): Handle.<Entity> {
		let slot: uint32;
		if (this.#freeHead != NULL_SLOT) {
			slot = this.#freeHead;
			this.#freeHead = this.#freeNext[slot];
		} else {
			slot = this.#nextSlot++;
		}
		this.#generations[slot]++; // Even -> odd: live. The column starts zeroed, so a fresh slot's first life is generation 1
		const entity = { slot, generation: this.#generations[slot] } := Handle.<Entity>;
		let mask: BitSet;
		this.#migrateData.fill(null);
		for (const { component, data } of components) {
			mask.add(component);
			this.#migrateData[component] = data;
		}
		const archetype = this.#getOrCreateArchetype(mask);
		const row = archetype.addEntity(entity, this.#migrateData);
		this.entityLocations[slot] = { archetype, row } := EntityLocation;
		return entity;
	}

	// A reference into the column, for in-place mutation. Stale handles and
	// missing components throw: this is the accessor for data the caller
	// knows is there.
	at<C: Component>(entity: Handle.<Entity>, comp: C): ref componentType(C) {
		const location = this.locationOf(entity);
		if (location == null || !location.archetype.hasColumn(comp)) {
			throw new TypeError('Stale handle or missing component');
		}
		return ref location.archetype.column.<C>()[location.row];
	}

	// Copy the value out; false and out untouched when the handle is stale
	// or the component absent.
	tryGet<C: Component>(entity: Handle.<Entity>, comp: C, ref out: componentType(C)): boolean {
		const location = this.locationOf(entity);
		if (location == null || !location.archetype.hasColumn(comp)) {
			return false;
		}
		out = location.archetype.column.<C>()[location.row];
		return true;
	}

	addComponent<C: Component>(entity: Handle.<Entity>, comp: C, data: componentType(C)) {
		this.addComponentAny(entity, comp, data); // The check happened in this signature
	}
	// The runtime-id twin, for Commands.flush, whose recorded component is a
	// value: a value generic cannot bind from a non-constant.
	addComponentAny(entity: Handle.<Entity>, comp: Component, data: any) {
		const location = this.locationOf(entity);
		if (location == null) {
			return;
		}
		if (location.archetype.mask.has(comp)) {
			location.archetype.columnAny(comp)[location.row] = data;
			return;
		}
		let mask = location.archetype.mask; // Value copy
		mask.add(comp);
		this.#migrate(entity, location, mask, comp, data);
	}

	removeComponent(entity: Handle.<Entity>, comp: Component) {
		const location = this.locationOf(entity);
		if (location == null || !location.archetype.mask.has(comp)) {
			return;
		}
		let mask = location.archetype.mask;
		mask.remove(comp);
		this.#migrate(entity, location, mask, comp, null);
	}

	// A disabled component keeps its column data under the disabled id, so
	// normal queries skip the entity and enabling restores it unchanged.
	disableComponent(entity: Handle.<Entity>, comp: Component) {
		const disabled = disabledOf[comp];
		const location = this.locationOf(entity);
		if (disabled == Component.Transform || location == null || !location.archetype.mask.has(comp)) {
			return;
		}
		const data = location.archetype.columnAny(comp)[location.row];
		let mask = location.archetype.mask;
		mask.remove(comp);
		mask.add(disabled);
		this.#migrate(entity, location, mask, comp, null, disabled, data);
	}

	enableComponent(entity: Handle.<Entity>, comp: Component) {
		const disabled = disabledOf[comp];
		const location = this.locationOf(entity);
		if (disabled == Component.Transform || location == null || !location.archetype.mask.has(disabled)) {
			return;
		}
		const data = location.archetype.columnAny(disabled)[location.row];
		let mask = location.archetype.mask;
		mask.remove(disabled);
		mask.add(comp);
		this.#migrate(entity, location, mask, disabled, null, comp, data);
	}

	destroyEntity(entity: Handle.<Entity>) {
		const location = this.locationOf(entity);
		if (location == null) {
			return; // Stale or double destroy: a no-op, not a corruption
		}
		const swapped = location.archetype.removeEntity(location.row);
		const swappedLocation = this.locationOf(swapped); // The zero handle resolves to null
		if (swappedLocation != null) {
			swappedLocation.row = location.row;
		}
		this.entityLocations[entity.slot] = null;
		this.#generations[entity.slot]++; // Odd -> even: every outstanding handle to this slot is now stale
		this.#freeNext[entity.slot] = this.#freeHead;
		this.#freeHead = entity.slot;
	}

	#migrate(entity: Handle.<Entity>, location: EntityLocation, mask: BitSet,
		transitionedComponent: Component, transitionedData: any, movedTo?: Component, movedData?: any) {
		const data = this.#migrateData;
		for (let id: uint8 = 0; id < TotalBits; ++id) {
			data[id] = id == transitionedComponent ? transitionedData
				: location.archetype.hasColumn(Component(id)) ? location.archetype.columnAny(Component(id))[location.row]
				: null;
		}
		if (movedTo != null) {
			data[movedTo] = movedData;
		}
		const swapped = location.archetype.removeEntity(location.row);
		const swappedLocation = this.locationOf(swapped);
		if (swappedLocation != null) {
			swappedLocation.row = location.row;
		}
		const archetype = this.#getOrCreateArchetype(mask);
		const row = archetype.addEntity(entity, data);
		this.entityLocations[entity.slot] = { archetype, row } := EntityLocation;
	}
}

class ComponentInit {
	component: Component;
	data: any;
}
function init<C: Component>(component: C, data: componentType(C)): ComponentInit {
	return { component, data } := ComponentInit;
}
```

Spawning reads as a list of typed pairs, each checked against its component's data type, and returns the handle:

```js
const player = world.spawn([
	init(Component.Transform, { x: 100, y: 50, rotation: 0 }),
	init(Component.Velocity, { vx: 0, vy: 0, angularVel: 0 }),
	init(Component.Health, { current: 100, max: 100 }),
	init(Component.Replicated, {}),
]);
// init(Component.Health, { current: 100 }); // TypeError: max is required
// init(Component.Health, { current: 200, max: 100 }); // TypeError: the where clause failed

world.at(player, Component.Health).current = 75; // In place, through the ref return
// world.at(player.slot, Component.Health); // TypeError: uint32 is not a Handle.<Entity>
```

## Structural Commands

The store's coverage notes record that the reference-liveness rule *rediscovered* the command buffer: growth is forbidden while iteration holds references, so structural additions defer to a flush point. The archetype layout sharpens that finding into the full ECS discipline. Here even removal moves storage - `removeEntity` swap-removes, writing the last row over the removed one and popping every column - so *no* structural change is legal while `each` holds references into the columns: `world.destroyEntity` mid-iteration is the same compile-time TypeError `world.addComponent` is. The store could despawn during iteration because freeing touched no value bytes; an archetype cannot, and the compiler says so at the call. Everything structural therefore records into `Commands` and replays at a flush point where no references are live.

```js
enum CommandType: uint8 { Spawn, Add, Remove, Enable, Disable, Destroy };

class Command {
	type: CommandType;
	entity: Handle.<Entity>; // NULL_ENTITY for Spawn
	component: Component; // Component.Transform (0) where unused; never read for Spawn and Destroy
	data: any; // The typed data for Add, the [].<ComponentInit> for Spawn, null otherwise
}

class Commands {
	#buffer: [].<Command> = [];

	spawn(components: [].<ComponentInit>) {
		this.#buffer.push({ type: CommandType.Spawn, entity: NULL_ENTITY, component: Component.Transform, data: components });
	}
	// Typed where user code sits: data is checked against componentType(C)
	// here, at the record site, so the opaque replay below gives nothing up.
	add<C: Component>(entity: Handle.<Entity>, comp: C, data: componentType(C)) {
		this.#buffer.push({ type: CommandType.Add, entity, component: comp, data });
	}
	remove(entity: Handle.<Entity>, comp: Component) {
		this.#buffer.push({ type: CommandType.Remove, entity, component: comp, data: null });
	}
	enable(entity: Handle.<Entity>, comp: Component) {
		this.#buffer.push({ type: CommandType.Enable, entity, component: comp, data: null });
	}
	disable(entity: Handle.<Entity>, comp: Component) {
		this.#buffer.push({ type: CommandType.Disable, entity, component: comp, data: null });
	}
	destroy(entity: Handle.<Entity>) {
		this.#buffer.push({ type: CommandType.Destroy, entity, component: Component.Transform, data: null });
	}

	// One ordered log rather than per-kind buffers: replay in submission order
	// preserves what the frame meant when it adds to and then destroys the
	// same entity, and any command whose handle went stale earlier in the
	// flush no-ops at the world's generation check.
	flush(world: World) {
		for (const command of this.#buffer) {
			switch (command.type) {
				case CommandType.Spawn:
					world.spawn(command.data);
					break;
				case CommandType.Add:
					world.addComponentAny(command.entity, command.component, command.data);
					break;
				case CommandType.Remove:
					world.removeComponent(command.entity, command.component);
					break;
				case CommandType.Enable:
					world.enableComponent(command.entity, command.component);
					break;
				case CommandType.Disable:
					world.disableComponent(command.entity, command.component);
					break;
				case CommandType.Destroy:
					world.destroyEntity(command.entity);
					break;
			} // Exhaustive over CommandType
		}
		this.#buffer.length = 0;
	}
}
```

Replay is why `addComponentAny` exists. A recorded component id is a runtime value, and a value generic cannot bind from a non-constant, so `flush` calls the value-level twin that the typed `addComponent<C>` delegates to - the same two-level shape as `column`/`columnAny`. Nothing is given up: `commands.add` is generic and checks `data` against `componentType(C)` at the record site, which is where user code sits. `Commands` registers as a type-keyed resource, and the system runner flushes it after each phase's systems have run.

## The Game World

Dirty tracking, the transition log, and resources. The original hand-rolled the log as four parallel arrays plus a reused scratch object its generator yielded to avoid allocating per event. `Transition` is a value type here, so `[].<Transition>` *is* the structure-of-arrays-grade storage, and yielding from it copies a few words - both hacks dissolve. The dirty-mask indexing also lost its casts on one side and gained one on the other: a handle's slot is already `uint32`, while the component enum converts implicitly only to its underlying `uint8` and so widens explicitly.

```js
enum TransitionType: uint8 { Add, Remove, Enable, Disable, Destroy };

class Transition { // Value type: a handle and three small fields
	entity: Handle.<Entity>;
	type: TransitionType;
	component: Component;
	tick: uint32;
}

class GameWorld extends World {
	worldTick: uint32;
	readonly #dirtyMasks: [MaxEntities * Words].<uint32>;
	readonly #transitions: [].<Transition> = [];
	readonly #resources = new Map.<any, any>(); // Keyed by type objects

	// Resources are keyed by their type. Wrapper classes disambiguate scalars.
	setResource<T>(value: T) {
		this.#resources.set(T, value);
	}
	getResource<T>(): T {
		return this.#resources.get(T);
	}

	markDirty(entity: Handle.<Entity>, comp: Component) {
		this.#dirtyMasks[entity.slot * Words + uint32(comp) / 32] |= MSB >> (comp & 31);
	}
	isCompDirty(entity: Handle.<Entity>, comp: Component): boolean {
		return (this.#dirtyMasks[entity.slot * Words + uint32(comp) / 32] & (MSB >> (comp & 31))) != 0;
	}
	// A fixed-extent window over this slot's words, indexed by element.
	// Reads and writes alias the backing storage.
	dirtyWords(entity: Handle.<Entity>): [Words].<uint32> {
		return this.#dirtyMasks.window.<Words>(entity.slot * Words);
	}
	isDirty(entity: Handle.<Entity>): boolean {
		return !this.dirtyWords(entity).every(word => word == 0);
	}
	clearDirtyFlags() {
		this.#dirtyMasks.fill(0);
	}

	*transitionsThisTick(): Transition {
		yield* this.#transitions; // Each yield copies a value type
	}
	clearTransitionLog() {
		this.#transitions.length = 0;
	}

	// getMut's replacement: run the body against a reference into the column,
	// and stamp the dirty bit only if the component was actually there.
	mutate<C: Component>(entity: Handle.<Entity>, comp: C, body: (ref value: componentType(C)) => void): boolean {
		const location = this.locationOf(entity);
		if (location == null || !location.archetype.hasColumn(comp)) {
			return false;
		}
		body(ref location.archetype.column.<C>()[location.row]);
		this.markDirty(entity, comp);
		return true;
	}

	addComponentAny(entity: Handle.<Entity>, comp: Component, data: any) {
		const location = this.locationOf(entity);
		if (location == null) {
			return; // A stale handle no-ops before anything is logged or marked
		}
		const had = location.archetype.mask.has(comp);
		super.addComponentAny(entity, comp, data);
		this.markDirty(entity, comp);
		if (!had) {
			this.#transitions.push({ entity, type: TransitionType.Add, component: comp, tick: this.worldTick });
		}
	}
	// removeComponent, disableComponent, enableComponent, and destroyEntity
	// wrap super the same way, logging their transition types; destroyEntity
	// also clears the slot's dirty words and its replication entry so a
	// reused slot never inherits stale flags.
}
```

Overriding `addComponentAny` rather than the typed `addComponent` is deliberate: the typed method delegates down, so one override sees every path - direct calls and `Commands.flush` alike - and the stale-handle guard sits before the logging, where the original could stamp a dirty bit and log an `Add` for an entity that no longer existed.

## Sessions and Replication

A `Session` is a component holding a connection and the interest state for one client. The event bus is keyed by event type, the same pattern as resources, replacing the original's string channels and `read<T>` casts. Events carry handles, so an event that outlives its entity - read a phase after a destroy flushed - resolves to a miss at the world rather than to whichever entity recycled the slot:

```js
class CollisionEvent { // Value type: handles are value types too
	entityA: Handle.<Entity>;
	entityB: Handle.<Entity>;
	normal: Vec2;
	depth: float32;
	point: Vec2;
}

class EventBus {
	#channels = new Map.<any, [].<any>>();
	emit<T>(event: T) {
		let channel = this.#channels.get(T);
		if (channel == null) {
			channel = new [].<T>();
			this.#channels.set(T, channel);
		}
		channel.push(event);
	}
	read<T>(): [].<T> {
		return this.#channels.get(T) ?? [];
	}
	clear() {
		for (const channel of this.#channels.values()) {
			channel.length = 0; // Retain backing storage
		}
	}
}

interface Connection {
	send(packet: OutgoingPacket): void; // Must serialize synchronously
}
```

The original tracked a per-session map of entity id to generation - how many times this session had learned the id - because a raw id says nothing about *which* entity it currently names. The handle is that map, minted once by the world instead of once per session: a `Set.<Handle.<Entity>>` compares structurally, so it is instance-precise for free, and the eight-byte handle is the wire entity id, so a client's references carry the same staleness the server's do.

```js
class Session {
	readonly connection: Connection;
	viewRadius: float32.<{ exclusiveMinimum: 0 }>;
	observedCells = new Set.<GridCell>();
	readonly knownEntities = new Set.<Handle.<Entity>>();
	readonly pendingLearns = new Set.<Handle.<Entity>>();
	readonly pendingForgets = new Set.<Handle.<Entity>>();
	lastAckedWorldTick: uint32;
	readonly lastAckedVersions = new Map.<Handle.<Entity>, AckedState>();

	constructor(connection: Connection, viewRadius: float32) {
		this.connection = connection;
		this.viewRadius = viewRadius;
	}
	learnEntity(entity: Handle.<Entity>) {
		if (this.pendingForgets.delete(entity)) {
			return;
		}
		this.knownEntities.add(entity);
		this.pendingLearns.add(entity);
	}
	forgetEntity(entity: Handle.<Entity>) {
		if (this.pendingLearns.delete(entity)) {
			this.knownEntities.delete(entity);
			return;
		}
		this.knownEntities.delete(entity);
		this.lastAckedVersions.delete(entity);
		this.pendingForgets.add(entity);
	}
}

class ReplicationEntry {
	readonly versions: [TotalComponents].<uint32>; // Last tick each component changed
	archetypeVersion: uint32;
	componentMask: BitSet;
}
class AckedState {
	readonly versions: [TotalComponents].<uint32>;
	archetypeVersion: uint32;
}

class ReplicationState {
	// Flat, indexed by slot: a direct offset, no hashing. The generation check
	// lives at the world's accessors; destroy clears the slot's entry.
	readonly entries: [MaxEntities].<ReplicationEntry | null>;
	getOrCreate(entity: Handle.<Entity>): ReplicationEntry {
		return this.entries[entity.slot] ??= new ReplicationEntry();
	}
	delete(entity: Handle.<Entity>) {
		this.entries[entity.slot] = null;
	}
}
```

The spatial grid, replicated event log, learn/forget packet structures, and the wire encoding are unchanged in shape from the original, so the identifiers they reference - ```OutgoingPacket```, ```GridCell```, ```HierarchicalGrid```, and the ```narrowphaseTest``` helper - appear at their use sites without their definitions repeated here; [Binary Packet](binarypacket.md) is the natural encoding for ```OutgoingPacket```, with the ```changedMask``` selecting which component writes each delta carries and the eight-byte handle written as the entity id.

## Systems

A system declares its phase and order, compiles its queries once, and iterates columns. The string phases became an enum, ordered so the runner can iterate phases in declaration order. The runner also owns the flush point: after a phase's systems run, no iteration holds references, so the structural commands they recorded apply there.

```js
enum Phase: uint8 { PreUpdate, FixedUpdate, PostFixedUpdate, NetworkTick, PreRender };

interface System {
	readonly name: string;
	readonly phase: Phase;
	readonly order: uint32;
	enabled: boolean;
	init?(world: GameWorld): void;
	run(world: GameWorld): void;
}

class SystemRunner {
	readonly #world: GameWorld;
	readonly #phases = new Map.<Phase, [].<System>>();
	constructor(world: GameWorld) {
		this.#world = world;
	}
	addSystem(system: System) {
		let systems = this.#phases.get(system.phase);
		if (systems == null) {
			systems = new [].<System>();
			this.#phases.set(system.phase, systems);
		}
		systems.push(system);
		systems.sort((a, b) => int32(a.order) - int32(b.order));
	}
	initAll() {
		for (const systems of this.#phases.values()) {
			for (const system of systems) {
				system.init?.(this.#world);
			}
		}
	}
	runPhase(phase: Phase) {
		for (const system of this.#phases.get(phase) ?? []) {
			if (system.enabled) {
				system.run(this.#world);
			}
		}
		// The flush point: the structural changes this phase recorded apply
		// here, so the next phase queries the migrated world.
		this.#world.getResource.<Commands>().flush(this.#world);
	}
}
```

Scalar resources get wrapper classes so type-keyed lookup stays unambiguous:

```js
class FixedDeltaTime { value: float32.<{ exclusiveMinimum: 0 }>; }
class Gravity { value: Vec2; }
class InterpolationAlpha { value: float32.<{ minimum: 0, maximum: 1 }>; }
```

The movement pair shows the `each` API - `ref` parameters bind the column slots, so the bodies write storage directly:

```js
class GravitySystem implements System {
	readonly name = 'Gravity';
	readonly phase = Phase.FixedUpdate;
	readonly order: uint32 = 1;
	enabled: boolean = true;
	#query: PreparedQuery;

	init(world: GameWorld) {
		this.#query = world.compileQuery([Component.Velocity, Component.RigidBody], [Component.Static]);
	}
	run(world: GameWorld) {
		const dt = world.getResource.<FixedDeltaTime>().value;
		const g = world.getResource.<Gravity>().value;
		this.#query.each.<Component.Velocity>((entity, ref velocity) => {
			velocity.vx += g.x * dt;
			velocity.vy += g.y * dt;
			world.markDirty(entity, Component.Velocity);
		});
	}
}

class MovementSystem implements System {
	readonly name = 'Movement';
	readonly phase = Phase.FixedUpdate;
	readonly order: uint32 = 2;
	enabled: boolean = true;
	#query: PreparedQuery;

	init(world: GameWorld) {
		this.#query = world.compileQuery([Component.Transform, Component.Velocity], [Component.Static]);
	}
	run(world: GameWorld) {
		const dt = world.getResource.<FixedDeltaTime>().value;
		this.#query.each.<Component.Transform, Component.Velocity>((entity, ref transform, ref velocity) => {
			if (velocity.vx == 0 && velocity.vy == 0 && velocity.angularVel == 0) {
				return;
			}
			transform.x += velocity.vx * dt;
			transform.y += velocity.vy * dt;
			transform.rotation += velocity.angularVel * dt;
			world.markDirty(entity, Component.Transform);
		});
	}
}
```

### Vectorized Systems

```each``` passes a reference per element, which suits any system whose body branches, calls out, or touches the world. A system whose body is dense arithmetic over one or two fields can do better by dropping to the columns. That's what ```SoA``` is for: ```fields.vx``` is a ```[].<float32>``` with unit stride, and viewing it as ```[].<float32x4>``` is an ordinary array view, so SIMD needs no new API.

Gravity is the clean case. Every entity in the query gets the same constant added, with no branch:

```js
class GravitySystem implements System {
	readonly name = 'Gravity';
	readonly phase = Phase.FixedUpdate;
	readonly order: uint32 = 1;
	enabled: boolean = true;
	#query: PreparedQuery;

	init(world: GameWorld) {
		this.#query = world.compileQuery([Component.Velocity, Component.RigidBody], [Component.Static]);
	}
	run(world: GameWorld) {
		const dt = world.getResource.<FixedDeltaTime>().value;
		const g = world.getResource.<Gravity>().value;

		// A scalar assigned to a SIMD binding broadcasts to every lane.
		const dvx: float32x4 = g.x * dt;
		const dvy: float32x4 = g.y * dt;

		for (const archetype of this.#query.archetypes) {
			const velocities = archetype.column.<Component.Velocity>();
			const vx = velocities.fields.vx; // [].<float32>, dense
			const vy = velocities.fields.vy; // angularVel is never loaded
			const count = archetype.length;
			const whole = count - count % 4;

			// Four elements per iteration. The view aliases the column.
			const vxLanes = [].<float32x4>(vx.window(0, whole));
			const vyLanes = [].<float32x4>(vy.window(0, whole));
			for (let j: uint32 = 0; j < vxLanes.length; ++j) {
				vxLanes[j] += dvx;
				vyLanes[j] += dvy;
			}
			for (let i: uint32 = whole; i < count; ++i) { // Tail
				vx[i] += g.x * dt;
				vy[i] += g.y * dt;
			}
			for (const entity of archetype.entities) {
				world.markDirty(entity, Component.Velocity);
			}
		}
	}
}
```

Three properties of this loop come from the layout rather than from the code. ```angularVel``` is a third column and is never touched, so it never enters cache. ```vx``` has unit stride, so a four-lane load is one aligned load rather than a gather; interleaved, its stride would be ```Velocity.byteLength``` and the vector unit would sit idle. And the ```window``` view costs nothing at runtime, since it is the same storage with a length.

```MovementSystem``` stays on ```each```, and the reason is visible in its body: it skips entities whose velocity is zero. A branch per element is the thing SIMD cannot do without masking, and the masked version reads worse than the scalar one for a predicate that is usually false. That is the honest division between the two forms - the callback for systems with control flow, the columns for arithmetic.

Collision events flow through the typed bus, and the health system shows disabled variants doing their job - a dead entity keeps its data but leaves the physics queries. The disables go through the command buffer, because the direct call inside ```each``` migrates the entity out of the very archetype the callback holds references into - the compile-time TypeError the liveness rule promises. The broadphase's pair key became a two-handle value class, retiring the ```* 65536``` packing along with its coupling to ```MaxEntities```:

```js
class EntityPair { // A value type key: the unordered pair, ordered by slot
	a: Handle.<Entity>;
	b: Handle.<Entity>;
}

class BroadphaseSystem implements System {
	readonly name = 'Broadphase';
	readonly phase = Phase.FixedUpdate;
	readonly order: uint32 = 6;
	enabled: boolean = true;
	#query: PreparedQuery;
	readonly #tested = new Set.<EntityPair>();
	readonly #near: [].<Handle.<Entity>> = [];

	init(world: GameWorld) {
		this.#query = world.compileQuery([Component.Transform, Component.Collider, Component.Velocity]);
	}
	run(world: GameWorld) {
		const grid = world.getResource.<HierarchicalGrid>();
		const events = world.getResource.<EventBus>();
		this.#tested.clear();
		this.#query.each.<Component.Transform, Component.Collider>((entity, ref transform, ref collider) => {
			grid.queryNear(transform, collider, this.#near);
			for (const other of this.#near) {
				if (other == entity) { // Structural equality on value types
					continue;
				}
				const pair = entity.slot < other.slot
					? { a: entity, b: other } := EntityPair
					: { a: other, b: entity } := EntityPair;
				if (!this.#tested.has(pair)) {
					this.#tested.add(pair);
					let otherTransform: Transform;
					let otherCollider: Collider;
					if (world.tryGet(other, Component.Transform, ref otherTransform)
							&& world.tryGet(other, Component.Collider, ref otherCollider)) {
						const contact = narrowphaseTest(collider, transform, otherCollider, otherTransform);
						if (contact != null) {
							events.emit({ entityA: entity, entityB: other, normal: contact.normal, depth: contact.depth, point: contact.point } := CollisionEvent);
						}
					}
				}
			}
		});
	}
}

class HealthSystem implements System {
	readonly name = 'Health';
	readonly phase = Phase.FixedUpdate;
	readonly order: uint32 = 8;
	enabled: boolean = true;
	#query: PreparedQuery;

	init(world: GameWorld) {
		this.#query = world.compileQuery([Component.Health]);
	}
	run(world: GameWorld) {
		const events = world.getResource.<EventBus>();
		const commands = world.getResource.<Commands>();
		for (const { entityA, entityB } of events.read.<CollisionEvent>()) {
			this.#applyDamage(world, entityA, entityB);
			this.#applyDamage(world, entityB, entityA);
		}
		this.#query.each.<Component.Health>((entity, ref health) => {
			if (health.current <= 0) {
				// Deferred: disabling migrates the entity, which swap-removes from
				// the columns this callback holds references into. The direct call
				// is the liveness TypeError; the command applies at the phase flush.
				commands.disable(entity, Component.Velocity);
				commands.disable(entity, Component.Collider);
			}
		});
	}
	#applyDamage(world: GameWorld, source: Handle.<Entity>, target: Handle.<Entity>) {
		let damage: Damage;
		if (!world.tryGet(source, Component.Damage, ref damage)) {
			return;
		}
		let armor: Armor; // Zero-filled: defense is 0 when the component is absent
		world.tryGet(target, Component.Armor, ref armor);
		world.mutate(target, Component.Health, (ref health) => {
			health.current -= Math.max(0, damage.amount - armor.defense);
		});
	}
}
```

```#applyDamage``` is the split in one method: in-place mutation through ```mutate``` is legal here because no iteration holds references at this point in ```run```, while the structural disables above defer. A handle in a ```CollisionEvent``` that went stale - the entity destroyed by an earlier phase's flush - fails ```tryGet``` and ```mutate``` at the generation check and the damage is dropped, which is the correct answer and previously a silent read of a recycled id.

The replication version system walks each dirty entity's mask words with `clz32`, stamping the world tick per changed component - the same bit walk the archetype constructor uses - and the packet build system diffs those stamps against each session's acked versions to emit deltas. The interpolation, snapshot, patrol, parent-child, animation, and spatial maintenance systems from the original all reduce to the same column-loop or `each` shapes shown above.

## The Main Loop

Unchanged in structure: a fixed timestep accumulator driving `FixedUpdate`, a slower network tick, and interpolation for rendering.

```js
function mainLoop(world: GameWorld, runner: SystemRunner) {
	const fixedDt: float32 = 1 / 60;
	const netDt: float32 = 1 / 30;
	const maxSteps: uint32 = 8;
	let physicsAccumulator: float32 = 0;
	let netAccumulator: float32 = 0;
	let last = performance.now();

	function tick(now: float64) {
		const dt = float32(Math.min((now - last) / 1000, 0.1));
		last = now;
		physicsAccumulator += dt;
		netAccumulator += dt;
		runner.runPhase(Phase.PreUpdate);
		let steps: uint32 = 0;
		while (physicsAccumulator >= fixedDt) {
			if (++steps > maxSteps) {
				physicsAccumulator = 0;
				break;
			}
			++world.worldTick;
			runner.runPhase(Phase.FixedUpdate);
			runner.runPhase(Phase.PostFixedUpdate);
			world.clearDirtyFlags();
			world.clearTransitionLog();
			world.getResource.<EventBus>().clear();
			physicsAccumulator -= fixedDt;
			if (netAccumulator >= netDt) {
				runner.runPhase(Phase.NetworkTick);
				netAccumulator -= netDt;
			}
		}
		world.getResource.<InterpolationAlpha>().value = physicsAccumulator / fixedDt;
		runner.runPhase(Phase.PreRender);
		requestAnimationFrame(tick);
	}
	requestAnimationFrame(tick);
}
```

## Coverage Notes

The TypeScript original leans on four type-level features this proposal doesn't have - indexed access types (`ComponentDataMap[C]`), mapped types with `Partial` (the `spawn` literal), `keyof` (the resource map), and string literal unions (transition types, phases). Two of the four have *better* answers here (type-keyed resources and events; enums), one has an adequate one (spawn as typed pairs), and one required the extrapolation everything else in this example stands on:

- **Structure-of-arrays storage.** Resolved as the [structure of arrays](../soa.md) extension: the columns are `SoA.<T>`, one parallel array per immediate field behind the element API of `[].<T>`. Nothing in the systems below changed when the layout did, which was the design's whole claim. The [particle system](particlesystem.md) wanted the same thing for GPU attribute upload.
- **Compile-time functions in type position.** Resolved for the typed paths: where the component is a compile-time constant - `column.<C>()`, `each.<C1>()`, `at.<C>()`, `tryGet.<C>()` - `componentType(C)` used as a type replaces `ComponentDataMap[C]`, since the main proposal blesses an expression evaluating to a type object as a valid type annotation, with `type` naming the type such a function returns. Applying `componentType` to a *runtime* id can't go in type position, because the proposal requires a compile-time function's arguments to be constants, so the sites that construct, read, or write a column from a dynamic id use value-level fallbacks: an exhaustive `newColumn(id)` switch factory in the constructor, and the untyped `columnAny`/`addComponentAny` pair for the opaque data-shuttling in `#migrate`, `disableComponent`, `enableComponent`, and `Commands.flush`. Dependent dispatch over a runtime enum-indexed type function - which would retire the factory and both untyped accessors - is the open wish. Without the typed form the alternative was one accessor overload per component, fifty-five declarations edited in lockstep with the enum.
- **Generic type parameters in expression position.** Resolved alongside the above: `getResource<T>()` does `this.#resources.get(T)`, and the runtime type objects section now states that a generic type parameter in scope evaluates to the type object it was specialized with.
- **Type objects as Map keys** are endorsed in the keyed collections section, with the scalar wrapper-class convention this example uses. They replace both the string-keyed `ResourceMap`/`keyof` machinery and the string-channel event bus with something strictly better: `getResource.<HierarchicalGrid>()` cannot be misspelled.
- **Value type classes as Map keys** are resolved by the keyed collections section: value type keys compare structurally with SameValueZero and are copied into the collection on insert, so the archetype index is a `Map.<BitSet, Archetype>` and `toKey()` is deleted. The copy-in rule also means `#getOrCreateArchetype` can hand its working mask straight to `set` without defensively copying it first. The same rule makes `Handle.<Entity>` a `Set` member and a `Map` key for the session tables, and the broadphase's pair key a two-handle `EntityPair` in a `Set` instead of a `slot * 65536 + slot` packing coupled to `MaxEntities`.
- **The generational store's handle transplanted without modification, and it deleted code on arrival.** `Handle.<Entity>` is the [generational store](generationalstore.md)'s phantom-typed class imported as-is - one marker class, `Entity`, is the entire per-world cost. The World adopted the store's parity protocol over fixed columns: `MaxEntities` bounds the slot, so the generation and free-list columns are flat `[MaxEntities]` arrays, the free list threads with no growable side stack, and every accessor resolves through one `locationOf`. A stale handle anywhere - in a `CollisionEvent` read after the entity's destroy flushed, in a session's known set, in a late client packet - degrades to a checked miss instead of an aliased read of the slot's new occupant, which is the store's guarantee holding across every table this server keys by entity. The largest deletion is the per-session id-to-generation map: the handle *is* that map, minted once by the world rather than once per session, and the eight-byte handle doubles as the wire entity id.
- **The liveness rule's command-buffer rediscovery, stricter.** The store found that references held during iteration forbid growth, so structural *additions* defer while despawn stays safe by layout. Archetype storage sharpens the rule: removal swap-removes, writing and popping the very columns a query's callbacks hold references into, so here *nothing* structural is legal mid-iteration - `HealthSystem`'s in-loop `disableComponent`, legal-looking in the original, is the compile-time TypeError, with the message at the exact call. `Commands` therefore carries every structural operation and replays at a per-phase flush where no references are live, which is the sync-point discipline Bevy and Unity adopt by convention, arrived at here as a consequence of array semantics. One ordered log rather than per-kind buffers keeps an add-then-destroy of the same entity meaning what the frame meant, and a handle gone stale earlier in the flush no-ops at the generation check, so replay order can't corrupt.
- **A nullable union of a value type class is the reference form, and two APIs changed shape to respect it.** The original `getRef<C>(...): componentType(C) | null` returned, for a value class, a detached boxed copy: reads paid an allocation per call in the broadphase inner loop, and writes vanished - `getMut` marked Health dirty while the damage landed in a copy the next line discarded. The store's accessors are the honest shapes and replaced it: `at` returns a `ref` and throws, `tryGet` copies out through a `ref` parameter, and `mutate` runs a callback against the column reference. In the other direction, returning `Handle.<Entity> | null` from `removeEntity` would box eight bytes to say nothing moved; the parity protocol supplies a sentinel for free - the zero handle's generation is even, so it is never live and every resolver already rejects it - which the swap-remove returns and the command buffer stores for spawns. A nullable value-class in a signature is worth reading twice: it is a reference, with a reference's allocation, and mutating through one mutates the box.
- **The spawn literal.** TypeScript's `Partial<{[C in Component]: ComponentDataMap[C]}>` types an object literal keyed by enum with per-key data types. The pair-helper pattern (`init(Component.Health, {...})`) recovers per-pair checking and reads fine, but it is a workaround. The direct feature would be a dependent index signature - `{ [key: C: Component]: componentType(C) }` - which is a large ask; recommend blessing the pair pattern and deferring the signature unless it recurs.
- **Variadic generic parameters.** `each` is written as one overload per arity. Tuple spread already exists in type arguments (`[...ReadTypes, T]` in the packet reader), so `each<...Cs: [].<Component>>` with the callback's ref parameters derived from the tuple is expressible in spirit; recommending it as the follow-up that makes query APIs scale past hand-written arities.
- **`ref` callback parameters as the iteration idiom.** Resolved: the value type references section now presents `each((entity, ref t, ref v) => ...)` as the way to iterate several arrays in step and mutate an element of each, composing `ref` parameters with `ref` element access. References have no identity and cannot escape, so passing one allocates nothing; that much is guaranteed. Whether the *call* is inlined is an optimizer's decision, not the language's, which is why the vectorized systems above drop to the columns where the arithmetic justifies it. Making inlining a guarantee needs a mechanism the proposal doesn't have - `inline` functions or monomorphization over the callback's type - which is the last open item in this example's list.
- **Small wins worth recording:** `array.window.<N>(start)` replaces `subarray` and takes element indices, so `dirtyWords` needs no byte arithmetic, and a handle's slot being `uint32` already retires the `uint32(entityId)` casts that indexing needed; enum sentinel arithmetic (`DisabledVelocity = __COUNT`) works as written via value references, and the enum section now records that `Reflect.getReflection.<Reflect.Enum, T>().size` supplies a count where no sentinel is wanted; `>>` being logical on `uint32` retires `>>>`; structural `==` on value classes is what lets the broadphase compare handles directly; and the deleted-code list from the intro - the BitSet pool, `copyFrom`, `toKey`, the scratch-object generator, the parallel-array transition log, and the session generation map - is the value-type performance story measured in removed lines rather than added ones.
