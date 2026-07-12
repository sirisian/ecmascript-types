# Dynamic Bounding Volume Hierarchy

A broadphase collision structure of the Box2D lineage: a binary tree of fattened AABBs over moving entities, incrementally updated as they move, queried for overlap pairs and segment hits every frame, and layered over a coarse grid so a world can hold ten thousand entities of wildly different sizes. Performance broadphase is organized around allocation avoidance - a node pool, preallocated traversal stacks, and arithmetic that stays in registers - which makes it a test of the proposal's performance core from the data-structure side rather than the numeric side. The example is split into ```dbvh.js``` for the tree and ```world.js``` for the layered world that drives it, and the notes at the end record which allocation-avoidance patterns the type system absorbs into ordinary code and which frictions it adds in their place.

Features exercised, and why they matter here:

- An index-based node pool in one contiguous ```[].<Node>```: value type nodes, ```uint32``` children, and a free list threaded through a reused field - the data-oriented spelling of a tree, with no per-node object header and no pointer chasing across the heap.
- ```inline``` AABB operations returning by value. Guaranteed construction-in-place means ```AABB.merge(a, b)``` is written like math and compiles into the destination slot, so the tree stays allocation-free with no scratch boxes and no output-parameter functions.
- The reference liveness rule from the value type references section, which forbids holding a ```ref``` into the pool across an allocation that could move it - the same discipline Rust's borrow checker imposes on a ```Vec```, arrived at here from array semantics alone.
- ```[].<T>.withCapacity``` and ```reserve``` for steady-state zero allocation: the moved buffer, the hit list, and the pool itself reach their working size once and never touch the allocator again.
- A fixed ```[4096].<SAHEntry>``` traversal stack of value type entries, so the surface-area-heuristic search pushes and pops without allocating an object per step in its hottest loop.
- A ```uint64``` pair key built from two shifted ids, exact across the full id range rather than a float product that loses precision past 2^53.

## AABB

The box comes first. Four ```float32``` fields make it a sixteen-byte value type; every operation on it is ```inline```, so the arithmetic lands in registers at the call site and the functions below cost what the expressions inside them cost.

```js
// dbvh.js
export class AABB {
	minX: float32;
	minY: float32;
	maxX: float32;
	maxY: float32;

	inline overlaps(b: AABB): boolean {
		return this.minX <= b.maxX && this.maxX >= b.minX && this.minY <= b.maxY && this.maxY >= b.minY;
	}
	inline contains(inner: AABB): boolean {
		return this.minX <= inner.minX && this.maxX >= inner.maxX && this.minY <= inner.minY && this.maxY >= inner.maxY;
	}
	inline perimeter(): float32 {
		return 2 * ((this.maxX - this.minX) + (this.maxY - this.minY));
	}
	static inline merge(a: AABB, b: AABB): AABB {
		return {
			minX: Math.min(a.minX, b.minX), minY: Math.min(a.minY, b.minY),
			maxX: Math.max(a.maxX, b.maxX), maxY: Math.max(a.maxY, b.maxY)
		} := AABB;
	}
	// The insertion cost metric: the perimeter the merged box would have, without building it.
	static inline mergedPerimeter(a: AABB, b: AABB): float32 {
		return 2 * ((Math.max(a.maxX, b.maxX) - Math.min(a.minX, b.minX))
			+ (Math.max(a.maxY, b.maxY) - Math.min(a.minY, b.minY)));
	}
}
```

Returning ```{ minX: ... }``` from a function would ordinarily allocate an object per call, which is why hand-tuned broadphase threads output parameters through ```mergeInto(out, a, b)``` helpers instead. Here ```AABB.merge``` returns a value type, and the typed assignment rule guarantees the ```:=``` literal is constructed directly in the destination - a node's ```aabb``` field, a local, the caller's slot - with no temporary and, being ```inline```, no call. The natural spelling is the fast spelling, which is the difference between a language where allocation avoidance is a style and one where it is the default.

## The Node Pool

Nodes live in one growable array and refer to each other by index. ```NULL_NODE``` is the sentinel for an absent link, and a node is a leaf exactly when it has no left child, which frees ```entity``` to be an index into the world's entity list.

```js
// dbvh.js
export const NULL_NODE: uint32 = 0xffffffff;

class Node {
	aabb: AABB;
	parent: uint32 = NULL_NODE; // Doubles as the free list's next pointer while the node is free
	left: uint32 = NULL_NODE;
	right: uint32 = NULL_NODE;
	entity: uint32 = NULL_NODE; // Index into the world's entities; meaningful only on leaves
	height: int32 = 0;
}

const nodes: [].<Node> = [].<Node>.withCapacity(16384);
let freeHead: uint32 = NULL_NODE;

inline function isLeaf(i: uint32): boolean {
	return nodes[i].left == NULL_NODE;
}

function allocNode(): uint32 {
	if (freeHead != NULL_NODE) {
		const i = freeHead;
		freeHead = nodes[i].parent;
		nodes[i] = new Node(); // Reset the slot; a sixteen-plus-four-times-four byte copy
		return i;
	}
	nodes.push(new Node()); // May reallocate: no ref into nodes may be live across this call
	return nodes.length - 1;
}

function freeNode(i: uint32) {
	nodes[i].parent = freeHead;
	nodes[i].entity = NULL_NODE;
	nodes[i].left = NULL_NODE;
	freeHead = i;
}
```

Every field of ```Node``` is a value type, so ```[].<Node>``` is a single contiguous allocation, and recycling a node is an index push onto the free list. Because ```AABB``` is embedded rather than referenced, a node is thirty-six bytes in one place instead of two objects in two, and walking the tree touches one array.

The comment on ```push``` is load-bearing. ```allocNode``` can grow the array, growth can move the storage, and the liveness rule makes holding a ```ref``` into ```nodes``` across that call a compile-time TypeError. The consequence is a code shape: everything that *restructures* the tree works in indices and takes references only briefly, between allocations, while the *queries* further down - which never allocate - take ```ref``` bindings freely and keep them for whole traversals. The alternative representation, nodes as reference-typed objects with ```Node | null``` links, would sidestep the rule entirely, because heap objects never move - at the cost of a pointer dereference per hop and an object header per node. The index pool buys contiguity and pays with a rule, and it is the same rule Rust would impose on a ```Vec<Node>```, here discovered by the compiler rather than the author.

## The Tree

Two insertion strategies, selected per tree: greedy descent for the dynamic trees that churn every frame, and a branch-and-bound surface area heuristic for the static trees built once. The SAH search is the hottest loop in construction, so its stack entry is a value type in a fixed array and the search allocates nothing per push.

```js
// dbvh.js
class SAHEntry {
	node: uint32;
	inherited: float32;
}

const stack: [4096].<uint32>;      // Shared traversal stack for queries
const sahStack: [4096].<SAHEntry>; // Insertion search stack

export class DBVH {
	root: uint32 = NULL_NODE;
	count: uint32 = 0;
	#useGreedy: boolean;

	constructor(useGreedy: boolean) {
		this.#useGreedy = useGreedy;
	}

	get height(): int32 {
		return this.root != NULL_NODE ? nodes[this.root].height : 0;
	}

	createLeaf(entity: uint32, aabb: AABB): uint32 {
		const i = allocNode();
		nodes[i].entity = entity;
		nodes[i].aabb = aabb;
		return i;
	}

	insert(leaf: uint32) {
		this.count++;
		if (this.root == NULL_NODE) {
			this.root = leaf;
			return;
		}
		const sibling = this.#useGreedy ? this.#siblingGreedy(leaf) : this.#siblingSAH(leaf);
		this.#makeParent(leaf, sibling);
	}

	#siblingGreedy(leaf: uint32): uint32 {
		const leafBox = nodes[leaf].aabb;
		let n = this.root;
		while (!isLeaf(n)) {
			const ref node = nodes[n]; // No allocation below: the ref is safe for the walk
			const merged = AABB.mergedPerimeter(node.aabb, leafBox);
			const delta = merged - node.aabb.perimeter();
			const costLeft = AABB.mergedPerimeter(nodes[node.left].aabb, leafBox) + (isLeaf(node.left) ? 0 : delta);
			const costRight = AABB.mergedPerimeter(nodes[node.right].aabb, leafBox) + (isLeaf(node.right) ? 0 : delta);
			if (merged < costLeft && merged < costRight) {
				break;
			}
			n = costLeft < costRight ? node.left : node.right;
		}
		return n;
	}

	#siblingSAH(leaf: uint32): uint32 {
		const leafBox = nodes[leaf].aabb;
		const leafPerimeter = leafBox.perimeter();
		sahStack[0] = { node: this.root, inherited: 0 } := SAHEntry;
		let sp: uint32 = 1;
		let best = this.root;
		let bestCost: float32 = Infinity;
		while (sp > 0) {
			const entry = sahStack[--sp]; // A copy; the stack slot is reused immediately
			const direct = AABB.mergedPerimeter(nodes[entry.node].aabb, leafBox);
			const cost = direct + entry.inherited;
			if (cost < bestCost) {
				bestCost = cost;
				best = entry.node;
			}
			if (!isLeaf(entry.node)) {
				const childInherited = entry.inherited + direct - nodes[entry.node].aabb.perimeter();
				if (leafPerimeter + childInherited < bestCost) {
					sahStack[sp++] = { node: nodes[entry.node].left, inherited: childInherited } := SAHEntry;
					sahStack[sp++] = { node: nodes[entry.node].right, inherited: childInherited } := SAHEntry;
				}
			}
		}
		return best;
	}

	#makeParent(leaf: uint32, sibling: uint32) {
		const oldParent = nodes[sibling].parent;
		const branch = allocNode(); // Allocation: take no refs across it
		nodes[branch].aabb = AABB.merge(nodes[leaf].aabb, nodes[sibling].aabb);
		nodes[branch].parent = oldParent;
		nodes[branch].left = leaf;
		nodes[branch].right = sibling;
		nodes[branch].height = 1 + Math.max(nodes[leaf].height, nodes[sibling].height);
		nodes[leaf].parent = branch;
		nodes[sibling].parent = branch;
		if (oldParent != NULL_NODE) {
			if (nodes[oldParent].left == sibling) {
				nodes[oldParent].left = branch;
			} else {
				nodes[oldParent].right = branch;
			}
			this.#refit(oldParent);
		} else {
			this.root = branch;
		}
	}

	remove(leaf: uint32) {
		this.count--;
		if (leaf == this.root) {
			this.root = NULL_NODE;
			return;
		}
		const parent = nodes[leaf].parent;
		const sibling = nodes[parent].left == leaf ? nodes[parent].right : nodes[parent].left;
		const grandparent = nodes[parent].parent;
		nodes[sibling].parent = grandparent;
		if (grandparent != NULL_NODE) {
			if (nodes[grandparent].left == parent) {
				nodes[grandparent].left = sibling;
			} else {
				nodes[grandparent].right = sibling;
			}
			this.#refit(grandparent);
		} else {
			this.root = sibling;
		}
		nodes[leaf].parent = NULL_NODE;
		freeNode(parent);
	}

	#refit(from: uint32) {
		let n = from;
		while (n != NULL_NODE) {
			if (!isLeaf(n)) {
				nodes[n].aabb = AABB.merge(nodes[nodes[n].left].aabb, nodes[nodes[n].right].aabb);
				nodes[n].height = 1 + Math.max(nodes[nodes[n].left].height, nodes[nodes[n].right].height);
				const balance = nodes[nodes[n].left].height - nodes[nodes[n].right].height;
				if (balance > 1) {
					this.#rotate(n, nodes[n].left, nodes[n].right);
				} else if (balance < -1) {
					this.#rotate(n, nodes[n].right, nodes[n].left);
				}
			}
			n = nodes[n].parent;
		}
	}

	#rotate(node: uint32, tall: uint32, short: uint32) {
		if (isLeaf(tall)) {
			return;
		}
		const tl = nodes[tall].left;
		const tr = nodes[tall].right;
		if (AABB.mergedPerimeter(nodes[tr].aabb, nodes[short].aabb) < AABB.mergedPerimeter(nodes[tl].aabb, nodes[short].aabb)) {
			nodes[tall].left = short;
			nodes[short].parent = tall;
			if (nodes[node].left == tall) {
				nodes[node].right = tl;
			} else {
				nodes[node].left = tl;
			}
			nodes[tl].parent = node;
		} else {
			nodes[tall].right = short;
			nodes[short].parent = tall;
			if (nodes[node].left == tall) {
				nodes[node].right = tr;
			} else {
				nodes[node].left = tr;
			}
			nodes[tr].parent = node;
		}
		nodes[tall].aabb = AABB.merge(nodes[nodes[tall].left].aabb, nodes[nodes[tall].right].aabb);
		nodes[tall].height = 1 + Math.max(nodes[nodes[tall].left].height, nodes[nodes[tall].right].height);
		nodes[node].aabb = AABB.merge(nodes[nodes[node].left].aabb, nodes[nodes[node].right].aabb);
		nodes[node].height = 1 + Math.max(nodes[nodes[node].left].height, nodes[nodes[node].right].height);
	}
}
```

```nodes[nodes[n].left].aabb``` is the readability cost of the index representation: a reference representation would spell the same thing ```n.left.aabb```, and the type system does not smooth the difference away. What the types add on top of the representation is that each of those accesses is a bounds-checked ```uint32``` index into one array with a compile-time element layout - a field read is base plus index times thirty-six plus a constant, with no hidden class check - and that the height arithmetic is ```int32``` throughout, so the ```balance``` subtraction cannot underflow the way it silently would had ```height``` been declared unsigned.

## Queries

Queries never allocate, so they hold references across whole traversals. The overlap query feeds pair tracking; the segment query is the slab test, used for projectiles that would tunnel through anything thinner than their per-frame motion; the point query is what a renderer's hit test calls.

```js
// dbvh.js
export type PairCallback = (a: uint32, b: uint32) => void;
export type HitCallback = (entity: uint32) => void;

partial class DBVH {
	queryPairs(leaf: uint32, cb: PairCallback) {
		if (this.root == NULL_NODE) {
			return;
		}
		const leafBox = nodes[leaf].aabb;
		const leafEntity = nodes[leaf].entity;
		stack[0] = this.root;
		let sp: uint32 = 1;
		while (sp > 0) {
			const n = stack[--sp];
			if (n == leaf) {
				continue;
			}
			const ref node = nodes[n]; // Queries allocate nothing, so the ref spans the visit
			if (!node.aabb.overlaps(leafBox)) {
				continue;
			}
			if (node.left == NULL_NODE) {
				cb(leafEntity, node.entity);
			} else {
				stack[sp++] = node.left;
				stack[sp++] = node.right;
			}
		}
	}

	querySegment(x0: float32, y0: float32, x1: float32, y1: float32, radius: float32, exclude: uint32, cb: HitCallback) {
		if (this.root == NULL_NODE) {
			return;
		}
		const dx = x1 - x0;
		const dy = y1 - y0;
		const invDx: float32 = 1 / (dx != 0 ? dx : 1e-7);
		const invDy: float32 = 1 / (dy != 0 ? dy : 1e-7);
		stack[0] = this.root;
		let sp: uint32 = 1;
		while (sp > 0) {
			const ref node = nodes[stack[--sp]];
			let t0x = (node.aabb.minX - radius - x0) * invDx;
			let t1x = (node.aabb.maxX + radius - x0) * invDx;
			if (invDx < 0) {
				const t = t0x; t0x = t1x; t1x = t;
			}
			let t0y = (node.aabb.minY - radius - y0) * invDy;
			let t1y = (node.aabb.maxY + radius - y0) * invDy;
			if (invDy < 0) {
				const t = t0y; t0y = t1y; t1y = t;
			}
			const tEnter = Math.max(t0x, t0y);
			const tExit = Math.min(t1x, t1y);
			if (tExit < tEnter || tEnter > 1 || tExit < 0) {
				continue;
			}
			if (node.left == NULL_NODE) {
				if (node.entity != exclude) {
					cb(node.entity);
				}
			} else {
				stack[sp++] = node.left;
				stack[sp++] = node.right;
			}
		}
	}

	queryPoint(x: float32, y: float32, cb: HitCallback) {
		if (this.root == NULL_NODE) {
			return;
		}
		stack[0] = this.root;
		let sp: uint32 = 1;
		while (sp > 0) {
			const ref node = nodes[stack[--sp]];
			if (x < node.aabb.minX || x > node.aabb.maxX || y < node.aabb.minY || y > node.aabb.maxY) {
				continue;
			}
			if (node.left == NULL_NODE) {
				cb(node.entity);
			} else {
				stack[sp++] = node.left;
				stack[sp++] = node.right;
			}
		}
	}
}
```

The callbacks are the one part of a query the ```inline``` guarantee cannot reach: an ```inline``` function's value cannot be taken, so a callback parameter is by definition not one, and whether ```cb``` disappears is the engine's monomorphic-call bet. The projectile system below sidesteps the question by writing hits into a preallocated buffer instead of calling out per hit.

## The World

```world.js``` owns entities, the layered grids, pair tracking, and the frame step. Entities have identity and hold an array, so they are ordinary reference objects; everything per-frame and per-node stays in value types.

```js
// world.js
import { AABB, DBVH, NULL_NODE } from './dbvh.js';

export const WORLD_W: float32 = 12000;
export const WORLD_H: float32 = 12000;

enum Layer { Small, Medium, Large }

const LAYER0_CELL: float32 = 1000;
const LAYER1_CELL: float32 = 4000;
const LAYER0_PROMOTE: float32 = 115;
const LAYER0_DEMOTE: float32 = 105;
const LAYER1_PROMOTE: float32 = 215;
const LAYER1_DEMOTE: float32 = 205;

export class Entity {
	id: uint32;
	x: float32; y: float32;
	hw: float32; hh: float32; // Half extents
	vx: float32; vy: float32;
	isStatic: boolean;
	isProjectile: boolean;
	layer: Layer = Layer.Small;
	cellX: uint16; cellY: uint16;
	node: uint32 = NULL_NODE;
	pairs: [].<Pair> = [];
	projRadius: float32;
	prevX: float32; prevY: float32;
}

export class Pair {
	a: Entity;
	b: Entity;
	constructor(a: Entity, b: Entity) {
		this.a = a;
		this.b = b;
	}
}

class Cell {
	bounds: AABB;
	staticTree: DBVH = new DBVH(false); // SAH: built rarely, queried constantly
	dynamicTree: DBVH = new DBVH(true); // Greedy: rebuilt around moving leaves
	safe: AABB; // Shrunk bounds inside which no neighbor cell's contents reach
}

class Grid {
	cellSize: float32;
	cols: uint32;
	rows: uint32;
	cells: [].<Cell> = [];

	constructor(cellSize: float32) {
		this.cellSize = cellSize;
		this.cols = uint32(Math.ceil(WORLD_W / cellSize));
		this.rows = uint32(Math.ceil(WORLD_H / cellSize));
		this.cells.reserve(this.cols * this.rows);
		for (let cx: uint32 = 0; cx < this.cols; cx++) {
			for (let cy: uint32 = 0; cy < this.rows; cy++) {
				const cell = new Cell();
				cell.bounds = { minX: cx * cellSize, minY: cy * cellSize, maxX: (cx + 1) * cellSize, maxY: (cy + 1) * cellSize } := AABB;
				this.cells.push(cell);
			}
		}
	}

	inline cell(cx: uint32, cy: uint32): Cell {
		return this.cells[cx * this.rows + cy];
	}
	inline clampX(x: float32): uint32 {
		const c = uint32(Math.max(0, Math.trunc(x / this.cellSize))); // Clamp below zero before the unsigned cast
		return Math.min(this.cols - 1, c);
	}
	inline clampY(y: float32): uint32 {
		const c = uint32(Math.max(0, Math.trunc(y / this.cellSize)));
		return Math.min(this.rows - 1, c);
	}
}
```

The grid is flat with an ```inline``` accessor rather than an array of arrays, since two multiplies beat a pointer hop and the proposal leaves true multidimensional indexing to user-defined index operators. The two trees per cell split by mobility: static entities in an SAH tree that is expensive to build and cheap to query, movers in a greedy tree with fattened leaves that absorb motion.

Fattening and layer assignment are ordinary value-returning helpers, the AABB constructors building their result in place rather than through an output parameter:

```js
// world.js
const JITTER_PADDING: float32 = 5;
const LOOK_AHEAD_FRAMES: float32 = 3;

inline function tightAABB(e: Entity): AABB {
	return { minX: e.x - e.hw, minY: e.y - e.hh, maxX: e.x + e.hw, maxY: e.y + e.hh } := AABB;
}

inline function expandedAABB(e: Entity): AABB {
	let box = { minX: e.x - e.hw - JITTER_PADDING, minY: e.y - e.hh - JITTER_PADDING,
		maxX: e.x + e.hw + JITTER_PADDING, maxY: e.y + e.hh + JITTER_PADDING } := AABB;
	const px = e.vx * LOOK_AHEAD_FRAMES;
	const py = e.vy * LOOK_AHEAD_FRAMES;
	if (px < 0) { box.minX += px; } else { box.maxX += px; }
	if (py < 0) { box.minY += py; } else { box.maxY += py; }
	return box;
}

inline function maxHalf(e: Entity): float32 {
	let mh = Math.max(e.hw, e.hh);
	if (!e.isStatic) {
		mh += JITTER_PADDING + Math.max(Math.abs(e.vx), Math.abs(e.vy)) * LOOK_AHEAD_FRAMES;
	}
	return mh;
}

function targetLayer(e: Entity): Layer {
	const mh = maxHalf(e);
	switch (e.layer) {
		case Layer.Small:
			return mh > LAYER0_PROMOTE ? Layer.Medium : Layer.Small;
		case Layer.Medium: {
			if (mh < LAYER0_DEMOTE) {
				return Layer.Small;
			}
			return mh > LAYER1_PROMOTE ? Layer.Large : Layer.Medium;
		}
		case Layer.Large:
			return mh < LAYER1_DEMOTE ? Layer.Medium : Layer.Large;
	} // Exhaustive over Layer: no default, no trailing return
}

function freshLayer(e: Entity): Layer {
	const mh = maxHalf(e);
	if (mh > LAYER1_PROMOTE) {
		return Layer.Large;
	}
	return mh > LAYER0_PROMOTE ? Layer.Medium : Layer.Small;
}
```

The hysteresis switch is exhaustive over the ```Layer``` enum, so adding a fourth layer is a compile error at every function that dispatches on it - the promotion thresholds cannot silently miss a tier the way a chain of numeric comparisons could.

## Migration and Pair Tracking

An entity lives in exactly one tree at a time - its layer's grid cell (or the global large-object trees), split static from dynamic. Movement between cells or layers is remove-reinsert with a refattened leaf. The moved buffer and the hit list are growable arrays warmed to capacity once, then reused by resetting ```length```; after the first frames at working size, a step performs no allocation.

```js
// world.js
export class World {
	entities: [].<Entity> = [];
	#grid0: Grid = new Grid(LAYER0_CELL);
	#grid1: Grid = new Grid(LAYER1_CELL);
	#globalStatic: DBVH = new DBVH(false);
	#globalDynamic: DBVH = new DBVH(true);
	#tracked: Map.<uint64, Pair> = new Map.<uint64, Pair>();
	#moved: [].<Entity> = [].<Entity>.withCapacity(16384);
	#nextId: uint32 = 0;
	projectileHits: [].<ProjectileHit> = [].<ProjectileHit>.withCapacity(512);
	// Callbacks are hoisted to fields so pair queries allocate no closure per call.
	// A partial class adds no fields, so these live here in the primary declaration
	// even though only the later sections use them.
	#pairCB: PairCallback = (a: uint32, b: uint32) => this.#addPair(a, b);
	#currentProjectile: Entity | null = null;
	#projHitCB: HitCallback = (hitId: uint32) => {
		this.projectileHits.push(new ProjectileHit(this.#currentProjectile, this.entities[hitId]));
	};

	get pairs(): Iterator.<Pair> {
		return this.#tracked.values();
	}

	#trees(e: Entity): Cell | null { // null means the global layer
		if (e.layer == Layer.Large) {
			return null;
		}
		const g = e.layer == Layer.Small ? this.#grid0 : this.#grid1;
		return g.cell(e.cellX, e.cellY);
	}

	#insertToTree(e: Entity) {
		let tree: DBVH;
		if (e.layer == Layer.Large) {
			tree = e.isStatic ? this.#globalStatic : this.#globalDynamic;
		} else {
			const g = e.layer == Layer.Small ? this.#grid0 : this.#grid1;
			e.cellX = uint16(g.clampX(e.x));
			e.cellY = uint16(g.clampY(e.y));
			const cell = g.cell(e.cellX, e.cellY);
			tree = e.isStatic ? cell.staticTree : cell.dynamicTree;
		}
		e.node = tree.createLeaf(e.id, e.isStatic ? tightAABB(e) : expandedAABB(e));
		tree.insert(e.node);
	}

	#removeFromTree(e: Entity) {
		const cell = this.#trees(e);
		const tree = cell == null
			? (e.isStatic ? this.#globalStatic : this.#globalDynamic)
			: (e.isStatic ? cell.staticTree : cell.dynamicTree);
		tree.remove(e.node);
	}

	#migrateLayer(e: Entity, layer: Layer) {
		this.#removeFromTree(e);
		e.layer = layer;
		this.#insertToTree(e); // Reinsertion refattens the leaf for the new tree
	}

	spawn(x: float32, y: float32, hw: float32, hh: float32, vx: float32, vy: float32, isStatic: boolean): Entity {
		const e = new Entity();
		e.id = this.#nextId++;
		e.x = x; e.y = y; e.hw = hw; e.hh = hh; e.vx = vx; e.vy = vy;
		e.isStatic = isStatic;
		e.layer = freshLayer(e);
		this.entities.push(e);
		this.#insertToTree(e);
		return e;
	}

	spawnProjectile(x: float32, y: float32, vx: float32, vy: float32, radius: float32): Entity {
		const e = this.spawn(x, y, Math.max(2, radius), Math.max(2, radius), vx, vy, false);
		e.isProjectile = true;
		e.projRadius = radius;
		e.prevX = x;
		e.prevY = y;
		return e;
	}
}
```

Pair identity is a ```uint64``` built from the two entity ids, smaller id in the high half. The untyped idiom for this is a float product, ```a.id * 1000000 + b.id``` - exact only while ids stay under a million and the product under 2^53 - hashed as a boxed number per lookup. The shifted key is exact for the full ```uint32``` id range, and the ```Map``` is typed on a value type, hashing eight bytes with no boxing.

```js
// world.js
inline function pairKey(a: Entity, b: Entity): uint64 {
	const lo = Math.min(a.id, b.id);
	const hi = Math.max(a.id, b.id);
	return (uint64(lo) << 32) | uint64(hi);
}

partial class World {
	#addPair(aId: uint32, bId: uint32) {
		const a = this.entities[aId];
		const b = this.entities[bId];
		const key = pairKey(a, b);
		if (!this.#tracked.has(key)) {
			const pair = new Pair(a, b);
			this.#tracked.set(key, pair);
			a.pairs.push(pair);
			b.pairs.push(pair);
		}
	}

	#queryCell(cell: Cell, e: Entity) {
		cell.dynamicTree.queryPairs(e.node, this.#pairCB);
		cell.staticTree.queryPairs(e.node, this.#pairCB);
	}

	#queryOverlappingCells(g: Grid, box: AABB, e: Entity) {
		const sx = g.clampX(box.minX);
		const sy = g.clampY(box.minY);
		const ex = g.clampX(box.maxX);
		const ey = g.clampY(box.maxY);
		for (let cx = sx; cx <= ex; cx++) {
			for (let cy = sy; cy <= ey; cy++) {
				this.#queryCell(g.cell(cx, cy), e);
			}
		}
	}

	#addPairsFor(e: Entity) {
		const box = nodes[e.node].aabb;
		if (e.layer != Layer.Large) {
			const g = e.layer == Layer.Small ? this.#grid0 : this.#grid1;
			const cell = g.cell(e.cellX, e.cellY);
			this.#queryCell(cell, e);
			// Neighbors only when the fattened box escapes the cell's safe interior
			if (box.minX < cell.safe.minX || box.maxX > cell.safe.maxX
				|| box.minY < cell.safe.minY || box.maxY > cell.safe.maxY) {
				this.#queryOverlappingCells(g, box, e);
			}
		} else {
			this.#globalDynamic.queryPairs(e.node, this.#pairCB);
			this.#globalStatic.queryPairs(e.node, this.#pairCB);
		}
		// Cross-layer: up into coarser storage, down into finer
		if (e.layer == Layer.Small) {
			this.#queryOverlappingCells(this.#grid1, box, e);
		}
		if (e.layer != Layer.Large) {
			this.#globalDynamic.queryPairs(e.node, this.#pairCB);
			this.#globalStatic.queryPairs(e.node, this.#pairCB);
		}
		if (e.layer != Layer.Small) {
			this.#queryOverlappingCells(this.#grid0, box, e);
		}
		if (e.layer == Layer.Large) {
			this.#queryOverlappingCells(this.#grid1, box, e);
		}
	}
}
```

The safe-interior test replaces eight direction-by-direction neighbor checks with one containment test against a per-cell ```safe``` box, computed each frame as the cell bounds shrunk by whatever the neighbors' roots overhang. That box is refreshed by one pass over the grid using the merged roots of each cell's two trees:

```js
// world.js
partial class World {
	#computeSafeBounds(g: Grid) {
		for (let cx: uint32 = 0; cx < g.cols; cx++) {
			for (let cy: uint32 = 0; cy < g.rows; cy++) {
				const cell = g.cell(cx, cy);
				let safe = cell.bounds;
				// Each neighbor that has contents may push one edge inward
				for (let nx = cx > 0 ? cx - 1 : cx; nx <= (cx < g.cols - 1 ? cx + 1 : cx); nx++) {
					for (let ny = cy > 0 ? cy - 1 : cy; ny <= (cy < g.rows - 1 ? cy + 1 : cy); ny++) {
						if (nx == cx && ny == cy) {
							continue;
						}
						const n = g.cell(nx, ny);
						const sRoot = n.staticTree.root;
						const dRoot = n.dynamicTree.root;
						if (sRoot == NULL_NODE && dRoot == NULL_NODE) {
							continue;
						}
						const reach = sRoot == NULL_NODE ? nodes[dRoot].aabb
							: dRoot == NULL_NODE ? nodes[sRoot].aabb
							: AABB.merge(nodes[sRoot].aabb, nodes[dRoot].aabb);
						if (nx < cx) { safe.minX = Math.max(safe.minX, reach.maxX); }
						if (nx > cx) { safe.maxX = Math.min(safe.maxX, reach.minX); }
						if (ny < cy) { safe.minY = Math.max(safe.minY, reach.maxY); }
						if (ny > cy) { safe.maxY = Math.min(safe.maxY, reach.minY); }
					}
				}
				cell.safe = safe;
			}
		}
	}
}
```

## The Frame

The step integrates movers, migrates the ones that changed layer or cell, refits the ones whose tight box escaped their fattened leaf, recomputes safe bounds, and reconciles pairs for everything that moved. Projectiles run the segment query from previous to current position, so a bullet crossing a cell in one frame still reports the wall it passed through.

```js
// world.js
export class ProjectileHit {
	projectile: Entity | null;
	target: Entity;
	constructor(projectile: Entity | null, target: Entity) {
		this.projectile = projectile;
		this.target = target;
	}
}

partial class World {
	step() {
		this.#moved.length = 0;

		for (const e of this.entities) {
			if (e.isStatic) {
				continue;
			}
			if (e.isProjectile) {
				e.prevX = e.x;
				e.prevY = e.y;
			}
			e.x += e.vx;
			e.y += e.vy;
			if (e.x < e.hw) { e.x = e.hw; e.vx = -e.vx; }
			if (e.x > WORLD_W - e.hw) { e.x = WORLD_W - e.hw; e.vx = -e.vx; }
			if (e.y < e.hh) { e.y = e.hh; e.vy = -e.vy; }
			if (e.y > WORLD_H - e.hh) { e.y = WORLD_H - e.hh; e.vy = -e.vy; }

			const layer = targetLayer(e);
			if (layer != e.layer) {
				this.#migrateLayer(e, layer);
				if (!e.isProjectile) {
					this.#moved.push(e);
				}
				continue;
			}
			if (e.layer != Layer.Large) {
				const g = e.layer == Layer.Small ? this.#grid0 : this.#grid1;
				const ncx = uint16(g.clampX(e.x));
				const ncy = uint16(g.clampY(e.y));
				if (ncx != e.cellX || ncy != e.cellY) {
					this.#removeFromTree(e);
					e.cellX = ncx;
					e.cellY = ncy;
					this.#insertToTree(e);
					if (!e.isProjectile) {
						this.#moved.push(e);
					}
					continue;
				}
			}
			// In place: refit only when the tight box escapes the fattened leaf
			if (!nodes[e.node].aabb.contains(tightAABB(e))) {
				const cell = this.#trees(e);
				const tree = cell == null ? this.#globalDynamic : cell.dynamicTree;
				tree.remove(e.node);
				nodes[e.node].aabb = expandedAABB(e);
				nodes[e.node].parent = NULL_NODE;
				tree.insert(e.node);
				if (!e.isProjectile) {
					this.#moved.push(e);
				}
			}
		}

		this.#computeSafeBounds(this.#grid0);
		this.#computeSafeBounds(this.#grid1);
		this.#updatePairs();
		this.#updateProjectiles();
	}

	#updatePairs() {
		for (const e of this.#moved) {
			this.#addPairsFor(e);
		}
		for (const e of this.#moved) {
			let w: uint32 = 0;
			for (let j: uint32 = 0; j < e.pairs.length; j++) {
				const pair = e.pairs[j];
				if (!nodes[pair.a.node].aabb.overlaps(nodes[pair.b.node].aabb)) {
					this.#tracked.delete(pairKey(pair.a, pair.b));
					const other = pair.a == e ? pair.b : pair.a;
					const idx = other.pairs.indexOf(pair);
					if (idx != -1) {
						other.pairs[idx] = other.pairs[other.pairs.length - 1];
						other.pairs.pop();
					}
				} else {
					e.pairs[w++] = pair;
				}
			}
			e.pairs.length = w;
		}
	}

	#updateProjectiles() {
		this.projectileHits.length = 0;
		for (const p of this.entities) {
			if (!p.isProjectile) {
				continue;
			}
			this.#currentProjectile = p;
			this.castSegment(p.prevX, p.prevY, p.x, p.y, p.projRadius, p, this.#projHitCB);
		}
		this.#currentProjectile = null;
	}

	#castGrid(g: Grid, box: AABB, x0: float32, y0: float32, x1: float32, y1: float32, radius: float32, excludeId: uint32, cb: HitCallback) {
		const sx = g.clampX(box.minX);
		const sy = g.clampY(box.minY);
		const ex = g.clampX(box.maxX);
		const ey = g.clampY(box.maxY);
		for (let cx = sx; cx <= ex; cx++) {
			for (let cy = sy; cy <= ey; cy++) {
				const cell = g.cell(cx, cy);
				cell.dynamicTree.querySegment(x0, y0, x1, y1, radius, excludeId, cb);
				cell.staticTree.querySegment(x0, y0, x1, y1, radius, excludeId, cb);
			}
		}
	}

	castSegment(x0: float32, y0: float32, x1: float32, y1: float32, radius: float32, exclude: Entity | null, cb: HitCallback) {
		// The exclusion compares entity ids, not node indices; the two are both uint32,
		// which is exactly the confusion the coverage notes describe.
		const excludeId = exclude != null ? exclude.id : NULL_NODE;
		const box = {
			minX: Math.min(x0, x1) - radius, minY: Math.min(y0, y1) - radius,
			maxX: Math.max(x0, x1) + radius, maxY: Math.max(y0, y1) + radius
		} := AABB;
		this.#castGrid(this.#grid0, box, x0, y0, x1, y1, radius, excludeId, cb);
		this.#castGrid(this.#grid1, box, x0, y0, x1, y1, radius, excludeId, cb);
		this.#globalDynamic.querySegment(x0, y0, x1, y1, radius, excludeId, cb);
		this.#globalStatic.querySegment(x0, y0, x1, y1, radius, excludeId, cb);
	}
}
```

## Usage

The module form is what a game or a renderer imports. A renderer draws by walking ```world.entities```, hit-tests by calling ```queryPoint``` on the trees under the cursor's cell, and never needs to know how the broadphase is layered.

```js
// main.js
import { World } from './world.js';

const world = new World();

for (let i: uint32 = 0; i < 500; i++) {
	const isStatic = i % 5 == 0;
	const angle = Math.random() * Math.PI * 2;
	const speed = isStatic ? 0 : 0.5 + Math.random() * 2;
	world.spawn(
		float32(100 + Math.random() * 11800), float32(100 + Math.random() * 11800),
		float32(0.5 + Math.random() * 99.5), float32(0.5 + Math.random() * 99.5),
		float32(Math.cos(angle) * speed), float32(Math.sin(angle) * speed),
		isStatic);
}
world.spawnProjectile(6000, 6000, 40, 12, 15);

for (let frame: uint32 = 0; frame < 600; frame++) {
	world.step();
	for (const hit of world.projectileHits) {
		// Respond to the projectile crossing hit.target this frame
	}
}

let overlapping: uint32 = 0;
for (const pair of world.pairs) {
	overlapping++;
}
```

## Coverage Notes

This example was chosen to stress the proposal from the data-structure side rather than the numeric side, and the frictions it found are as informative as the wins.

- **Index pools trade nullability away, and the type system offers nothing back.** A reference representation with ```Node | null``` links makes an absent child unrepresentable-by-accident; the ```uint32``` plus ```NULL_NODE``` chosen here for contiguity makes it a convention the compiler cannot check, and worse, a node index and an entity id are the same type, so passing one where the other is expected typechecks and corrupts. Rust's answer is a zero-cost newtype index per pool; here ```type NodeIndex = uint32``` is a structural alias with no distinction, and the nominal alternative - a one-field class with cast operators - costs more ceremony than the bug it prevents. A lightweight nominal alias, opted into distinctness, is the missing feature this example wants most.
- **The reference liveness rule reproduced the borrow checker's discipline, friction included.** Holding a ```ref``` into ```nodes``` across ```allocNode``` is a compile-time TypeError because ```push``` can move the storage - which is exactly the dangling-interior-pointer bug a reference representation is structurally immune to, its nodes never moving, and an index-based C++ port would have silently. The cost is the shape of ```#makeParent``` and ```#refit```: repeated ```nodes[i].field``` accesses where a held reference would read better, each one a fresh bounds-checked index. The queries, which never allocate, hold references across whole traversals with no ceremony. The rule drew the line in the right place; the code that restructures pays, the code that reads does not.
- **A pooled slot's two states share one field, invisibly.** A live node's ```parent``` and a free node's next-in-free-list occupy the same ```uint32```, as in Box2D's union. Both states type identically, so the overloading is carried by a comment. A tagged representation would cost bytes per node for a distinction only the allocator cares about; this is a place where the honest answer is that the type system neither helps nor hinders, and C and Rust (via ```union```/```MaybeUninit```) are in the same position unless they spend the tag.
- **Callback-based queries sit outside the ```inline``` guarantee by design.** An ```inline``` function's value cannot be taken, so ```queryPairs```'s callback is inherently an indirect call, and whether it vanishes is the engine's monomorphization bet. The two escapes both appear here: the projectile system writes into a preallocated hit buffer instead of calling out, and the pair callback does so little (a ```Map``` probe) that the call cost is noise. A future direction the proposal could take is a callback-taking function that is itself ```inline```, specializing per call site the way Rust monomorphizes a closure-generic function; nothing in the current semantics forbids an engine doing so, but nothing guarantees it either.
- **Guaranteed elision plus ```inline``` removes the whole allocation-avoidance layer a hand-tuned broadphase carries.** Scratch boxes, ```*Into(out, ...)``` output-parameter functions, and the object a naive surface-area-heuristic search allocates per push become value returns and a fixed ```[4096].<SAHEntry>``` stack. Nothing is traded for it - the value-returning spellings are shorter than the output-parameter ones. Together with ```withCapacity``` on the moved buffer and hit list, a steady-state frame performs zero allocations, which an object-per-push search cannot reach.
- **The ```uint64``` pair key is exact where a float key is probabilistic.** ```(uint64(lo) << 32) | uint64(hi)``` covers the full id range; the ```a.id * 1000000 + b.id``` idiom collides once ids pass a million and loses integer precision past 2^53. A typed map hashing an eight-byte value key also skips the number boxing a plain ```Map``` performs per probe. This is the quiet class of win - not a redesign, just arithmetic that means what it says.
- **```float32``` throughout halves broadphase bandwidth, with one deliberate exception.** Boxes, velocities, and slab-test math are ```float32```; node ```height``` is ```int32``` specifically so the balance subtraction is signed - declared ```uint16``` it would typecheck and wrap, which is the kind of latent bug the strict-conversion rules surface only if the author picks the right type to begin with. Layer thresholds compare in the same precision the boxes carry, so promotion hysteresis cannot flap from a mixed-precision comparison.
