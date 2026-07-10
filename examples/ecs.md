# Entity Component System

An archetype-based ECS for a networked game server: entities are rows in tables grouped by component composition, systems iterate matching tables, and sessions receive delta-replicated state for the entities their view touches. This is the design Unity DOTS, Bevy, and Flecs converge on, and it's the harshest test of a type system's generic machinery, because the whole point is storing *differently typed* columns behind *one* storage abstraction without boxing.

Converted from a TypeScript implementation, and the conversion has a theme: **much of the original code existed to fight the allocator, and value types delete it.** The `BitSet` pool is gone because bitsets copy by assignment, and its `toKey` string hashing is gone because a value type is a structural `Map` key. The reused scratch object threaded through a generator is gone because yielding a value type is already a copy. The four parallel arrays hand-rolling structure-of-arrays for the transition log are one `[].<Transition>` because a value type array is already contiguous. The `as` casts on every column access became typed accessors. The string-literal unions became enums with exhaustive switches.

Features exercised: value type classes as component storage and as bitmasks, structure-of-arrays columns through [`SoA.<T>`](../soa.md), `readonly` fields, `[N].<T>` fixed arrays with `byteLength`-based view windows, `ref` element bindings and `ref` callback parameters as the iteration API, enums replacing string vocabularies with sentinel-value arithmetic, type objects as `Map` keys for resources and events, typed generators, SIMD passes over field columns through ordinary array views, and a compile-time function in type position mapping component ids to component types - the load-bearing extrapolation this example is built to stress, detailed in the coverage notes.

## Components

```js
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
const MaxEntities: uint32 = 65536;
const MSB: uint32 = 0x80000000;

type EntityId = uint16; // MaxEntities is 2^16, so ids fit exactly

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
		// ... one case per component; exhaustiveness keeps it honest
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
	archetype: Archetype | null;
	row: uint32;
}

class Archetype {
	readonly mask: BitSet; // A value type field: the mask is stored inline
	readonly #columns: [TotalBits].<any>; // SoA.<componentType(id)> or null per slot
	readonly entities: [].<EntityId> = [];

	constructor(mask: BitSet) {
		this.mask = mask; // Copies the value
		for (let w: uint32 = 0; w < Words; ++w) {
			let word = mask.words[w];
			while (word != 0) {
				const bit = Math.clz32(word);
				const id = Component(w * 32 + bit);
				this.#columns[id] = new SoA.<componentType(id)>();
				word ^= MSB >> bit;
			}
		}
	}

	get length(): uint32 {
		return this.entities.length;
	}

	column<C: Component>(comp: C): SoA.<componentType(C)> {
		return this.#columns[comp];
	}
	hasColumn(comp: Component): boolean {
		return this.#columns[comp] != null;
	}

	// data is a flat scratch indexed by component id.
	addEntity(entityId: EntityId, data: [TotalBits].<any>): uint32 {
		const row = this.entities.length;
		this.entities.push(entityId);
		for (let id: uint8 = 0; id < TotalBits; ++id) {
			this.#columns[id]?.push(data[id]);
		}
		return row;
	}

	// Swap-removes the row. Returns the entity that moved into it, if any.
	removeEntity(row: uint32): EntityId | null {
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
		return row != lastRow ? this.entities[row] : null;
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
	each<C1: Component>(callback: (entityId: EntityId, ref a: componentType(C1)) => void) {
		for (const archetype of this.archetypes) {
			const as = archetype.column.<C1>();
			const entities = archetype.entities;
			for (let i: uint32 = 0; i < archetype.length; ++i) {
				callback(entities[i], ref as[i]);
			}
		}
	}
	each<C1: Component, C2: Component>(callback: (entityId: EntityId, ref a: componentType(C1), ref b: componentType(C2)) => void) {
		for (const archetype of this.archetypes) {
			const as = archetype.column.<C1>();
			const bs = archetype.column.<C2>();
			const entities = archetype.entities;
			for (let i: uint32 = 0; i < archetype.length; ++i) {
				callback(entities[i], ref as[i], ref bs[i]);
			}
		}
	}
	// Three- and four-component overloads follow the same shape.
}
```

## The World

```js
class World {
	#nextId: EntityId;
	readonly #freeIds: [].<EntityId> = [];
	readonly archetypes: [].<Archetype> = [];
	readonly #archetypeIndex = new Map.<BitSet, Archetype>(); // A value type key, compared structurally
	readonly entityLocations: [MaxEntities].<EntityLocation | null>; // Defaults to null
	readonly #queries: [].<PreparedQuery> = [];
	#migrateData: [TotalBits].<any>;

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

	spawn(components: [].<ComponentInit>): EntityId {
		const id = this.#freeIds.pop() ?? this.#nextId++;
		let mask: BitSet;
		this.#migrateData.fill(null);
		for (const { component, data } of components) {
			mask.add(component);
			this.#migrateData[component] = data;
		}
		const archetype = this.#getOrCreateArchetype(mask);
		const row = archetype.addEntity(id, this.#migrateData);
		this.entityLocations[id] = { archetype, row } := EntityLocation;
		return id;
	}

	getRef<C: Component>(entityId: EntityId, comp: C): componentType(C) | null {
		const location = this.entityLocations[entityId];
		if (location == null || !location.archetype.hasColumn(comp)) {
			return null;
		}
		return location.archetype.column.<C>()[location.row];
	}

	addComponent<C: Component>(entityId: EntityId, comp: C, data: componentType(C)) {
		const location = this.entityLocations[entityId];
		if (location == null) {
			return;
		}
		if (location.archetype.mask.has(comp)) {
			location.archetype.column.<C>()[location.row] = data;
			return;
		}
		let mask = location.archetype.mask; // Value copy
		mask.add(comp);
		this.#migrate(entityId, location, mask, comp, data);
	}

	removeComponent(entityId: EntityId, comp: Component) {
		const location = this.entityLocations[entityId];
		if (location == null || !location.archetype.mask.has(comp)) {
			return;
		}
		let mask = location.archetype.mask;
		mask.remove(comp);
		this.#migrate(entityId, location, mask, comp, null);
	}

	// A disabled component keeps its column data under the disabled id, so
	// normal queries skip the entity and enabling restores it unchanged.
	disableComponent(entityId: EntityId, comp: Component) {
		const disabled = disabledOf[comp];
		const location = this.entityLocations[entityId];
		if (disabled == Component.Transform || location == null || !location.archetype.mask.has(comp)) {
			return;
		}
		const data = location.archetype.column.<comp>()[location.row];
		let mask = location.archetype.mask;
		mask.remove(comp);
		mask.add(disabled);
		this.#migrate(entityId, location, mask, comp, null, disabled, data);
	}

	enableComponent(entityId: EntityId, comp: Component) {
		const disabled = disabledOf[comp];
		const location = this.entityLocations[entityId];
		if (disabled == Component.Transform || location == null || !location.archetype.mask.has(disabled)) {
			return;
		}
		const data = location.archetype.column.<disabled>()[location.row];
		let mask = location.archetype.mask;
		mask.remove(disabled);
		mask.add(comp);
		this.#migrate(entityId, location, mask, disabled, null, comp, data);
	}

	destroyEntity(entityId: EntityId) {
		const location = this.entityLocations[entityId];
		if (location == null) {
			return;
		}
		const swapped = location.archetype.removeEntity(location.row);
		if (swapped != null) {
			this.entityLocations[swapped].row = location.row;
		}
		this.entityLocations[entityId] = null;
		this.#freeIds.push(entityId);
	}

	#migrate(entityId: EntityId, location: EntityLocation, mask: BitSet,
		removed: Component, added: any, movedTo?: Component, movedData?: any) {
		const data = this.#migrateData;
		for (let id: uint8 = 0; id < TotalBits; ++id) {
			data[id] = id == removed ? added
				: location.archetype.hasColumn(Component(id)) ? location.archetype.column.<Component(id)>()[location.row]
				: null;
		}
		if (movedTo != null) {
			data[movedTo] = movedData;
		}
		const swapped = location.archetype.removeEntity(location.row);
		if (swapped != null) {
			this.entityLocations[swapped].row = location.row;
		}
		const archetype = this.#getOrCreateArchetype(mask);
		const row = archetype.addEntity(entityId, data);
		this.entityLocations[entityId] = { archetype, row } := EntityLocation;
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

Spawning reads as a list of typed pairs, each checked against its component's data type:

```js
const player = world.spawn([
	init(Component.Transform, { x: 100, y: 50, rotation: 0 }),
	init(Component.Velocity, { vx: 0, vy: 0, angularVel: 0 }),
	init(Component.Health, { current: 100, max: 100 }),
	init(Component.Replicated, {}),
]);
// init(Component.Health, { current: 100 }); // TypeError: max is required
// init(Component.Health, { current: 200, max: 100 }); // TypeError: the where clause failed
```

## The Game World

Dirty tracking, the transition log, and resources. The original hand-rolled the log as four parallel arrays plus a reused scratch object its generator yielded to avoid allocating per event. `Transition` is a value type here, so `[].<Transition>` *is* the structure-of-arrays-grade storage, and yielding from it copies a few words - both hacks dissolve:

```js
enum TransitionType: uint8 { Add, Remove, Enable, Disable, Destroy };

class Transition { // Value type: four small fields
	entityId: EntityId;
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

	markDirty(entityId: EntityId, comp: Component) {
		this.#dirtyMasks[uint32(entityId) * Words + comp / 32] |= MSB >> (comp & 31);
	}
	isCompDirty(entityId: EntityId, comp: Component): boolean {
		return (this.#dirtyMasks[uint32(entityId) * Words + comp / 32] & (MSB >> (comp & 31))) != 0;
	}
	// A fixed-extent window over this entity's words, indexed by element.
	// Reads and writes alias the backing storage.
	dirtyWords(entityId: EntityId): [Words].<uint32> {
		return this.#dirtyMasks.window.<Words>(uint32(entityId) * Words);
	}
	isDirty(entityId: EntityId): boolean {
		return !this.dirtyWords(entityId).every(word => word == 0);
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

	// getRef with dirty marking, for mutation.
	getMut<C: Component>(entityId: EntityId, comp: C): componentType(C) | null {
		const data = this.getRef.<C>(entityId, comp);
		if (data != null) {
			this.markDirty(entityId, comp);
		}
		return data;
	}

	addComponent<C: Component>(entityId: EntityId, comp: C, data: componentType(C)) {
		const had = this.entityLocations[entityId]?.archetype.mask.has(comp) ?? false;
		super.addComponent(entityId, comp, data);
		this.markDirty(entityId, comp);
		if (!had) {
			this.#transitions.push({ entityId, type: TransitionType.Add, component: comp, tick: this.worldTick });
		}
	}
	// removeComponent, disableComponent, enableComponent, and destroyEntity
	// wrap super the same way, logging their transition types; destroyEntity
	// also clears this entity's dirty words and its replication entry so a
	// reused id never inherits stale state.
}
```

## Sessions and Replication

A `Session` is a component holding a connection and the interest state for one client. The event bus is keyed by event type, the same pattern as resources, replacing the original's string channels and `read<T>` casts:

```js
class CollisionEvent { // Value type
	entityA: EntityId;
	entityB: EntityId;
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

class Session {
	readonly connection: Connection;
	viewRadius: float32.<{ exclusiveMinimum: 0 }>;
	observedCells = new Set.<GridCell>();
	readonly knownEntities = new Map.<EntityId, uint32>(); // id -> generation
	readonly pendingLearns = new Set.<EntityId>();
	readonly pendingForgets = new Set.<EntityId>();
	lastAckedWorldTick: uint32;
	readonly lastAckedVersions = new Map.<EntityId, AckedState>();
	readonly #generations = new Map.<EntityId, uint32>();

	constructor(connection: Connection, viewRadius: float32) {
		this.connection = connection;
		this.viewRadius = viewRadius;
	}
	learnEntity(entityId: EntityId) {
		if (this.pendingForgets.delete(entityId)) {
			return;
		}
		const generation = (this.#generations.get(entityId) ?? 0) + 1;
		this.#generations.set(entityId, generation);
		this.knownEntities.set(entityId, generation);
		this.pendingLearns.add(entityId);
	}
	forgetEntity(entityId: EntityId) {
		if (this.pendingLearns.delete(entityId)) {
			this.knownEntities.delete(entityId);
			return;
		}
		this.knownEntities.delete(entityId);
		this.lastAckedVersions.delete(entityId);
		this.pendingForgets.add(entityId);
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
	// Flat, indexed by EntityId: a direct offset, no hashing.
	readonly entries: [MaxEntities].<ReplicationEntry | null>;
	getOrCreate(entityId: EntityId): ReplicationEntry {
		return this.entries[entityId] ??= new ReplicationEntry();
	}
	delete(entityId: EntityId) {
		this.entries[entityId] = null;
	}
}
```

The spatial grid, replicated event log, learn/forget packet structures, and the wire encoding are unchanged in shape from the original; [Binary Packet](binarypacket.md) is the natural encoding for `OutgoingPacket`, with the `changedMask` selecting which component writes each delta carries.

## Systems

A system declares its phase and order, compiles its queries once, and iterates columns. The string phases became an enum, ordered so the runner can iterate phases in declaration order:

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
		this.#query.each.<Component.Velocity>((entityId, ref velocity) => {
			velocity.vx += g.x * dt;
			velocity.vy += g.y * dt;
			world.markDirty(entityId, Component.Velocity);
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
		this.#query.each.<Component.Transform, Component.Velocity>((entityId, ref transform, ref velocity) => {
			if (velocity.vx == 0 && velocity.vy == 0 && velocity.angularVel == 0) {
				return;
			}
			transform.x += velocity.vx * dt;
			transform.y += velocity.vy * dt;
			transform.rotation += velocity.angularVel * dt;
			world.markDirty(entityId, Component.Transform);
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
			for (const entityId of archetype.entities) {
				world.markDirty(entityId, Component.Velocity);
			}
		}
	}
}
```

Three properties of this loop come from the layout rather than from the code. ```angularVel``` is a third column and is never touched, so it never enters cache. ```vx``` has unit stride, so a four-lane load is one aligned load rather than a gather; interleaved, its stride would be ```Velocity.byteLength``` and the vector unit would sit idle. And the ```window``` view costs nothing at runtime, since it is the same storage with a length.

```MovementSystem``` stays on ```each```, and the reason is visible in its body: it skips entities whose velocity is zero. A branch per element is the thing SIMD cannot do without masking, and the masked version reads worse than the scalar one for a predicate that is usually false. That is the honest division between the two forms - the callback for systems with control flow, the columns for arithmetic.

Collision events flow through the typed bus, and the health system shows disabled variants doing their job - a dead entity keeps its data but leaves the physics queries:

```js
class BroadphaseSystem implements System {
	readonly name = 'Broadphase';
	readonly phase = Phase.FixedUpdate;
	readonly order: uint32 = 6;
	enabled: boolean = true;
	#query: PreparedQuery;
	readonly #tested = new Set.<uint32>();
	readonly #near: [].<EntityId> = [];

	init(world: GameWorld) {
		this.#query = world.compileQuery([Component.Transform, Component.Collider, Component.Velocity]);
	}
	run(world: GameWorld) {
		const grid = world.getResource.<HierarchicalGrid>();
		const events = world.getResource.<EventBus>();
		this.#tested.clear();
		this.#query.each.<Component.Transform, Component.Collider>((entityId, ref transform, ref collider) => {
			grid.queryNear(transform, collider, this.#near);
			for (const other of this.#near) {
				if (other == entityId) {
					continue;
				}
				const pairKey = uint32(Math.min(entityId, other)) * 65536 + Math.max(entityId, other);
				if (!this.#tested.has(pairKey)) {
					this.#tested.add(pairKey);
					const otherTransform = world.getRef(other, Component.Transform);
					const otherCollider = world.getRef(other, Component.Collider);
					if (otherTransform != null && otherCollider != null) {
						const contact = narrowphaseTest(collider, transform, otherCollider, otherTransform);
						if (contact != null) {
							events.emit({ entityA: entityId, entityB: other, normal: contact.normal, depth: contact.depth, point: contact.point } := CollisionEvent);
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
		for (const { entityA, entityB } of events.read.<CollisionEvent>()) {
			this.#applyDamage(world, entityA, entityB);
			this.#applyDamage(world, entityB, entityA);
		}
		this.#query.each.<Component.Health>((entityId, ref health) => {
			if (health.current <= 0) {
				world.disableComponent(entityId, Component.Velocity);
				world.disableComponent(entityId, Component.Collider);
			}
		});
	}
	#applyDamage(world: GameWorld, source: EntityId, target: EntityId) {
		const damage = world.getRef(source, Component.Damage);
		if (damage == null) {
			return;
		}
		const health = world.getMut(target, Component.Health);
		if (health == null) {
			return;
		}
		const defense = world.getRef(target, Component.Armor)?.defense ?? 0;
		health.current -= Math.max(0, damage.amount - defense);
	}
}
```

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
- **Compile-time functions in type position.** Resolved: `componentType(C)` used as a type replaces `ComponentDataMap[C]`, and the main proposal's compile-time type expressions bless exactly this - an expression evaluating to a type object is a valid type annotation, with `type` naming the type such a function returns. Without it the alternative was one accessor overload per component, fifty-five declarations edited in lockstep with the enum.
- **Generic type parameters in expression position.** Resolved alongside the above: `getResource<T>()` does `this.#resources.get(T)`, and the runtime type objects section now states that a generic type parameter in scope evaluates to the type object it was specialized with.
- **Type objects as Map keys** are endorsed in the keyed collections section, with the scalar wrapper-class convention this example uses. They replace both the string-keyed `ResourceMap`/`keyof` machinery and the string-channel event bus with something strictly better: `getResource.<HierarchicalGrid>()` cannot be misspelled.
- **Value type classes as Map keys** are resolved by the keyed collections section: value type keys compare structurally with SameValueZero and are copied into the collection on insert, so the archetype index is a `Map.<BitSet, Archetype>` and `toKey()` is deleted. The copy-in rule also means `#getOrCreateArchetype` can hand its working mask straight to `set` without defensively copying it first.
- **The spawn literal.** TypeScript's `Partial<{[C in Component]: ComponentDataMap[C]}>` types an object literal keyed by enum with per-key data types. The pair-helper pattern (`init(Component.Health, {...})`) recovers per-pair checking and reads fine, but it is a workaround. The direct feature would be a dependent index signature - `{ [key: C: Component]: componentType(C) }` - which is a large ask; recommend blessing the pair pattern and deferring the signature unless it recurs.
- **Variadic generic parameters.** `each` is written as one overload per arity. Tuple spread already exists in type arguments (`[...ReadTypes, T]` in the packet reader), so `each<...Cs: [].<Component>>` with the callback's ref parameters derived from the tuple is expressible in spirit; recommending it as the follow-up that makes query APIs scale past hand-written arities.
- **`ref` callback parameters as the iteration idiom.** Resolved: the value type references section now presents `each((entityId, ref t, ref v) => ...)` as the way to iterate several arrays in step and mutate an element of each, composing `ref` parameters with `ref` element access. References have no identity and cannot escape, so passing one allocates nothing; that much is guaranteed. Whether the *call* is inlined is an optimizer's decision, not the language's, which is why the vectorized systems above drop to the columns where the arithmetic justifies it. Making inlining a guarantee needs a mechanism the proposal doesn't have - `inline` functions or monomorphization over the callback's type - which is the last open item in this example's list.
- **Small wins worth recording:** `array.window.<N>(start)` replaces `subarray` and takes element indices, so `dirtyWords` needs no byte arithmetic; enum sentinel arithmetic (`DisabledVelocity = __COUNT`) works as written via value references, and the enum section now records that `Reflect.getReflection.<Reflect.Enum, T>().size` supplies a count where no sentinel is wanted; `>>` being logical on `uint32` retires `>>>`; and the deleted-code list from the intro - the BitSet pool, `copyFrom`, `toKey`, the scratch-object generator, and the parallel-array transition log - is the value-type performance story measured in removed lines rather than added ones.
