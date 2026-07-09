# Binary Packet Writer and Reader

A bit-granular packet writer and reader for realtime network protocols. Values are written at arbitrary bit widths - a boolean is one bit, an ASCII character is seven, a quantized position fits in eighteen - so state updates pack densely into a single datagram. This is the hand-tuned end of the serialization spectrum: [Serialization](../serialization.md) covers typed values whose layout follows their declaration, while this document is for wire formats where every bit is chosen. The ```@data``` decorators in [Decorators](../decorators.md) sit on top of exactly these ```write.<T>``` methods to automate field-driven packets.

The code doubles as a tour of the proposal, and several pieces only work *because* of types:

- ```uint.<N>``` arbitrary-width integers are the currency of the whole format, and casting to one range-checks: writing a non-ASCII character through ```uint.<7>``` is a TypeError, not corruption.
- Typed integer division makes ```this.#bitIndex / BufferBits``` a word index and ```(bits + 7) / 8``` a byte count; untyped, both are fractions.
- Value generics with defaults (```Size```, ```HeaderSize```, ```BufferBits```) parameterize the class, and compile-time quantization arguments like ```write.<float32, -1024, 1024, 18>``` specialize per call site with no runtime configuration object.
- Generic method overloads select by their type argument, the family constraint ```LengthType extends uint``` bounds string length prefixes, fixed-length arrays bounds-check overruns, uninitialized typed declarations default to zero, and views reinterpret floats as their bit patterns without a scratch ```DataView```.

## PacketWriter

```js
@doc('A bit-granular packet writer for realtime network protocols.')
export class PacketWriter<Size: uint32 = 1400, HeaderSize: uint32 = 16, BufferBits: uint32 = 64> {
	@doc('The word buffer for the packet. Bits fill each word from the most significant end.')
	#buffer: [].<uint.<BufferBits>>;

	// Typed declarations without initializers default to 0.
	@doc('The current bit index of the writer.')
	#bitIndex: uint32;

	@doc('The bit index recorded by end() and written into the header.')
	#maximumBitIndex: uint32;

	constructor() {
		// 1500 byte MTU minus IP/TCP/WebSocket framing ~= 1400 byte default payload.
		// Typed integer division truncates, so round the word count up first.
		const words: uint32 = (Size + BufferBits / 8 - 1) / (BufferBits / 8);
		this.#buffer = new [words].<uint.<BufferBits>>(); // Zero-filled fixed-length array
	}

	@doc('The written packet as bytes, sized to the bits actually used.')
	get bytes(): [].<uint8> {
		return [].<uint8>(this.#buffer).slice(0, (this.#maximumBitIndex + 7) / 8);
	}

	@doc('Reserves the header. Call before writing values.')
	begin(): PacketWriter {
		this.#bitIndex = HeaderSize;
		return this;
	}

	@doc('Records the final bit length into the header. Call after writing values.')
	end(): PacketWriter {
		this.#maximumBitIndex = this.#bitIndex;
		this.#buffer[0] |= uint.<BufferBits>(this.#maximumBitIndex) << (BufferBits - HeaderSize);
		return this;
	}

	@doc('Reads bits at an arbitrary bit index without moving the cursor.')
	#bitsAt(bitIndex: uint32, bits: uint32): uint.<BufferBits> {
		const offset = bitIndex % BufferBits;
		let value = this.#buffer[bitIndex / BufferBits] << offset >> (BufferBits - bits);
		if (offset > BufferBits - bits) {
			value |= this.#buffer[bitIndex / BufferBits + 1] >> (2 * BufferBits - bits - offset);
		}
		return value;
	}

	@doc('Writes the low `bits` bits of value at the cursor, spilling across a word boundary when needed.')
	#writeBits(value: uint.<BufferBits>, bits: uint32) {
		const offset = this.#bitIndex % BufferBits;
		// Left-align the value in a word, then shift it down to the cursor.
		// The left shift also discards any bits above `bits`.
		this.#buffer[this.#bitIndex / BufferBits] |= value << (BufferBits - bits) >> offset;
		if (offset > BufferBits - bits) {
			// The value crossed into the next word.
			this.#buffer[this.#bitIndex / BufferBits + 1] |= value << (2 * BufferBits - bits - offset);
		}
		this.#bitIndex += bits;
	}

	@doc('Writes a 1-bit boolean.')
	write<boolean>(value: boolean): PacketWriter {
		this.#writeBits(value ? 1 : 0, 1);
		return this;
	}

	@doc('Writes an n-bit unsigned integer, e.g. write.<uint.<12>>(value).')
	write<uint<N: uint32>>(value: uint.<N>): PacketWriter {
		this.#writeBits(value, N);
		return this;
	}

	@doc('Writes an n-bit signed integer as two\'s complement.')
	write<int<N: uint32>>(value: int.<N>): PacketWriter {
		return this.write.<uint.<N>>(uint.<N>(value));
	}

	@doc('Writes an unsigned integer in [0, maximum] using the fewest bits that hold the range.')
	write<uint<N: uint32>, maximum: uint32>(value: uint.<N>): PacketWriter {
		const bits: uint32 = 32 - Math.clz32(maximum);
		this.#writeBits(value, bits);
		return this;
	}

	@doc('Writes an unsigned integer in [minimum, maximum] using the fewest bits that hold the range.')
	write<uint<N: uint32>, minimum: uint32, maximum: uint32>(value: uint.<N>): PacketWriter {
		const bits: uint32 = 32 - Math.clz32(maximum - minimum);
		this.#writeBits(value - minimum, bits);
		return this;
	}

	@doc('Writes a signed integer in [minimum, maximum] using the fewest bits that hold the range.')
	write<int<N: uint32>, minimum: int32, maximum: int32>(value: int.<N>): PacketWriter {
		const bits: uint32 = 32 - Math.clz32(uint32(maximum - minimum));
		this.#writeBits(uint32(value - minimum), bits);
		return this;
	}

	@doc('Writes an exact 32-bit float by reinterpreting its bits through a view.')
	write<float32>(value: float32): PacketWriter {
		const scratch: [1].<float32> = [value];
		return this.write.<uint32>([].<uint32>(scratch)[0]);
	}

	@doc('Writes a float quantized onto [0, maximum] with the given bit budget.')
	write<float32, maximum: float32, bits: uint32>(value: float32): PacketWriter {
		this.#writeBits(Math.round(value / maximum * ((1 << bits) - 1)), bits);
		return this;
	}

	@doc('Writes a float quantized onto [minimum, maximum]. When the range spans zero, code 0 is reserved so 0.0 round-trips exactly.')
	write<float32, minimum: float32, maximum: float32, bits: uint32>(value: float32): PacketWriter {
		if (minimum < 0 && maximum > 0) {
			this.#writeBits(value == 0 ? 0 : Math.round((value - minimum) / (maximum - minimum) * ((1 << bits) - 2)) + 1, bits);
		} else {
			this.#writeBits(Math.round((value - minimum) / (maximum - minimum) * ((1 << bits) - 1)), bits);
		}
		return this;
	}

	@doc('Writes an exact 64-bit float by reinterpreting its bits through a view.')
	write<float64>(value: float64): PacketWriter {
		const scratch: [1].<float64> = [value];
		return this.write.<uint64>([].<uint64>(scratch)[0]);
	}

	@doc('Writes an unsigned integer with a variable-width encoding. Each continuation bit adds `bits` more value bits; size `bits` for the typical value.')
	writeVariableWidthUint<bits: uint32 = 8>(value: uint.<BufferBits>): PacketWriter {
		let width: uint32 = bits;
		while (width < BufferBits && value >> width != 0) {
			this.write.<boolean>(true); // One more interval follows.
			width += bits;
		}
		if (width < BufferBits) {
			this.write.<boolean>(false); // Terminate the sequence.
		}
		this.#writeBits(value, Math.min(width, BufferBits));
		return this;
	}

	@doc('Writes a signed integer with a variable-width encoding using zigzag mapping, so small magnitudes of either sign stay small.')
	writeVariableWidthInt<bits: uint32 = 8>(value: int.<BufferBits>): PacketWriter {
		return this.writeVariableWidthUint.<bits>(uint.<BufferBits>((value << 1) ^ (value >> (BufferBits - 1))));
	}

	@doc('Writes a length-prefixed ASCII string. A non-ASCII character fails the uint.<7> cast with a TypeError.')
	write<string, LengthType extends uint = uint16>(value: string): PacketWriter {
		this.write.<LengthType>(LengthType(value.length));
		for (let index: uint32 = 0; index < value.length; ++index) {
			this.write.<uint.<7>>(value.charCodeAt(index));
		}
		return this;
	}

	@doc('Writes another packet, header included, so the receiver can carve it back out.')
	write<PacketWriter>(value: PacketWriter): PacketWriter {
		for (let cursor: uint32 = 0; cursor < value.#maximumBitIndex; cursor += BufferBits) {
			const bits = Math.min(BufferBits, value.#maximumBitIndex - cursor);
			this.#writeBits(value.#bitsAt(cursor, bits), bits);
		}
		return this;
	}

	@doc('Returns the written bits as a string of 0s and 1s for debugging.')
	trace(): string {
		let s = '';
		const end = Math.max(this.#bitIndex, this.#maximumBitIndex);
		for (let index: uint32 = 0; index < end; ++index) {
			if (index != 0 && index % 8 == 0) {
				s += ' ';
			}
			s += this.#bitsAt(index, 1) == 0 ? '0' : '1';
		}
		return s;
	}
}
```

## PacketReader

```js
@doc('A bit-granular packet reader mirroring PacketWriter.')
export class PacketReader<Size: uint32 = 1400, HeaderSize: uint32 = 16, BufferBits: uint32 = 64> {
	#buffer: [].<uint.<BufferBits>>;
	#bitIndex: uint32;
	#maximumBitIndex: uint32;

	@doc('Constructs a reader over received bytes, e.g. a WebSocket message or a WebTransport datagram.')
	constructor(buffer: [].<uint8>) {
		// Copy into whole words so the shift math never runs off the end.
		const words: uint32 = (uint32(buffer.length) + BufferBits / 8 - 1) / (BufferBits / 8);
		this.#buffer = new [words].<uint.<BufferBits>>();
		[].<uint8>(this.#buffer).set(buffer);
		this.readHeader();
	}

	@doc('Reads the packet bit length from the header.')
	readHeader() {
		// Bounds checks compare against the whole buffer until the real length is known.
		this.#maximumBitIndex = uint32(this.#buffer.length) * BufferBits;
		this.#maximumBitIndex = this.#readBits(HeaderSize);
	}

	#bitsAt(bitIndex: uint32, bits: uint32): uint.<BufferBits> {
		const offset = bitIndex % BufferBits;
		let value = this.#buffer[bitIndex / BufferBits] << offset >> (BufferBits - bits);
		if (offset > BufferBits - bits) {
			value |= this.#buffer[bitIndex / BufferBits + 1] >> (2 * BufferBits - bits - offset);
		}
		return value;
	}

	@doc('Reads `bits` bits at the cursor with a bounds check against the header length.')
	#readBits(bits: uint32): uint.<BufferBits> {
		if (this.#bitIndex + bits > this.#maximumBitIndex) {
			throw new Error(`${bits} bits expected`);
		}
		const value = this.#bitsAt(this.#bitIndex, bits);
		this.#bitIndex += bits;
		return value;
	}

	@doc('Reads a 1-bit boolean.')
	read<boolean>(): boolean {
		return this.#readBits(1) == 1;
	}

	@doc('Reads an n-bit unsigned integer, e.g. read.<uint.<12>>().')
	read<uint<N: uint32>>(): uint.<N> {
		return uint.<N>(this.#readBits(N));
	}

	@doc('Reads an n-bit signed integer written as two\'s complement.')
	read<int<N: uint32>>(): int.<N> {
		return int.<N>(this.read.<uint.<N>>());
	}

	@doc('Reads an unsigned integer written with the [0, maximum] range encoding.')
	read<uint<N: uint32>, maximum: uint32>(): uint.<N> {
		const bits: uint32 = 32 - Math.clz32(maximum);
		return uint.<N>(this.#readBits(bits));
	}

	@doc('Reads an unsigned integer written with the [minimum, maximum] range encoding.')
	read<uint<N: uint32>, minimum: uint32, maximum: uint32>(): uint.<N> {
		const bits: uint32 = 32 - Math.clz32(maximum - minimum);
		return uint.<N>(this.#readBits(bits) + minimum);
	}

	@doc('Reads a signed integer written with the [minimum, maximum] range encoding.')
	read<int<N: uint32>, minimum: int32, maximum: int32>(): int.<N> {
		const bits: uint32 = 32 - Math.clz32(uint32(maximum - minimum));
		return int.<N>(int32(this.#readBits(bits)) + minimum);
	}

	@doc('Reads an exact 32-bit float.')
	read<float32>(): float32 {
		let scratch: [1].<uint32>; // Defaults to [0]
		scratch[0] = uint32(this.#readBits(32));
		return [].<float32>(scratch)[0];
	}

	@doc('Reads a float quantized onto [0, maximum].')
	read<float32, maximum: float32, bits: uint32>(): float32 {
		return float32(this.#readBits(bits)) / ((1 << bits) - 1) * maximum;
	}

	@doc('Reads a float quantized onto [minimum, maximum], honoring the reserved exact-zero code for ranges spanning zero.')
	read<float32, minimum: float32, maximum: float32, bits: uint32>(): float32 {
		const value = this.#readBits(bits);
		if (minimum < 0 && maximum > 0) {
			return value == 0 ? 0 : float32(value - 1) / ((1 << bits) - 2) * (maximum - minimum) + minimum;
		}
		return float32(value) / ((1 << bits) - 1) * (maximum - minimum) + minimum;
	}

	@doc('Reads an exact 64-bit float.')
	read<float64>(): float64 {
		let scratch: [1].<uint64>;
		scratch[0] = this.#readBits(64);
		return [].<float64>(scratch)[0];
	}

	@doc('Reads an unsigned integer written with the variable-width encoding.')
	readVariableWidthUint<bits: uint32 = 8>(): uint.<BufferBits> {
		let width: uint32 = bits;
		while (width < BufferBits && this.read.<boolean>()) {
			width += bits;
		}
		return this.#readBits(Math.min(width, BufferBits));
	}

	@doc('Reads a signed integer written with the zigzag variable-width encoding.')
	readVariableWidthInt<bits: uint32 = 8>(): int.<BufferBits> {
		const zigzag = this.readVariableWidthUint.<bits>();
		return int.<BufferBits>((zigzag >> 1) ^ -(zigzag & 1));
	}

	@doc('Reads a length-prefixed ASCII string.')
	read<string, LengthType extends uint = uint16>(): string {
		let value = '';
		const length = this.read.<LengthType>();
		for (let index: uint32 = 0; index < length; ++index) {
			value += String.fromCharCode(this.read.<uint.<7>>());
		}
		return value;
	}

	@doc('Reads a nested packet written with write.<PacketWriter>, positioned after its header.')
	read<PacketReader>(): PacketReader {
		const maximumBitIndex = uint32(this.#readBits(HeaderSize));
		if (this.#bitIndex + maximumBitIndex - HeaderSize > this.#maximumBitIndex) {
			throw new Error('Packet expected');
		}
		// The constructor's own header read is discarded by the explicit cursor below.
		const value = new PacketReader.<Size, HeaderSize, BufferBits>([].<uint8>(this.#buffer));
		value.#bitIndex = this.#bitIndex;
		value.#maximumBitIndex = this.#bitIndex + maximumBitIndex - HeaderSize;
		this.#bitIndex += maximumBitIndex - HeaderSize;
		return value;
	}
}
```

## Typestate Reads (experimental)

A reader can accumulate the values it reads into a tuple: an extra tuple generic collects the types read so far, each ```read``` returns ```this``` reparameterized with the tuple grown by one, and a cast operator hands the tuple to typed destructuring, which selects it by shape.

```js
export class AccumulatingPacketReader<ReadTypes extends [] = []> extends PacketReader {
	#values: ReadTypes = [];

	@doc('Reads a value and accumulates it, returning this with the tuple type grown.')
	read<T>(): AccumulatingPacketReader.<[...ReadTypes, T]> {
		this.#values = [...this.#values, super.read.<T>()];
		return this;
	}

	@doc('Destructuring an accumulating reader yields the collected tuple.')
	operator ReadTypes() {
		return this.#values;
	}
}

const reader = new AccumulatingPacketReader(bytes);
const [alive: boolean, id: uint.<12>, name: string] =
	reader.read.<boolean>().read.<uint.<12>>().read.<string>();
```

The experimental part is the return type: each call returns the same object viewed at a new parameterization, so this is typestate rather than immutability, and mixing the accumulating and plain styles on one instance would confuse the tuple. It's included because it exercises tuple spread in type arguments and the typed-return destructuring rules in one place.

## Building a Packet

```js
const movement = new PacketWriter()
	.begin()
	.write.<uint16>(sequence)
	.write.<boolean>(firing)
	.write.<float32, -1024, 1024, 18>(position.x) // 18 bits: ~0.03 unit resolution over 2048
	.write.<float32, -1024, 1024, 18>(position.y)
	.write.<uint32, 0, 1000000>(entityId) // 20 bits, from the range
	.write.<string>(callsign)
	.end();

movement.trace(); // '01000000 00101101 1...'
movement.bytes; // [].<uint8>, only the bytes used
```

Reading mirrors it, with every width and range coming from the same compile-time arguments so the two sides can't drift apart silently within one codebase:

```js
const packet = new PacketReader(bytes);
const sequence = packet.read.<uint16>();
const firing = packet.read.<boolean>();
const x = packet.read.<float32, -1024, 1024, 18>();
const y = packet.read.<float32, -1024, 1024, 18>();
const entityId = packet.read.<uint32, 0, 1000000>();
const callsign = packet.read.<string>();
```

## WebSocket

Binary frames carry the bytes directly. The framing is the transport's, so one message is one packet:

```js
const socket = new WebSocket('wss://game.example');
socket.binaryType = 'arraybuffer';

function sendMovement(sequence: uint16, x: float32, y: float32) {
	const packet = new PacketWriter()
		.begin()
		.write.<uint16>(sequence)
		.write.<float32, -1024, 1024, 18>(x)
		.write.<float32, -1024, 1024, 18>(y)
		.end();
	socket.send(packet.bytes);
}

socket.onmessage = (event) => {
	const packet = new PacketReader([].<uint8>(event.data)); // View over the ArrayBuffer
	const sequence = packet.read.<uint16>();
	const x = packet.read.<float32, -1024, 1024, 18>();
	const y = packet.read.<float32, -1024, 1024, 18>();
};
```

## WebTransport

Datagrams are the natural fit - unreliable, unordered delivery is why the format spends bits on a sequence number instead of trusting the transport. Typed ```for await``` infers the datagram type from the stream:

```js
const transport = new WebTransport('https://game.example:4433/session');
await transport.ready;

const writer = transport.datagrams.writable.getWriter();
function sendMovement(sequence: uint16, x: float32, y: float32) {
	const packet = new PacketWriter()
		.begin()
		.write.<uint16>(sequence)
		.write.<float32, -1024, 1024, 18>(x)
		.write.<float32, -1024, 1024, 18>(y)
		.end();
	writer.write(packet.bytes);
}

for await (const datagram of transport.datagrams.readable) {
	const packet = new PacketReader([].<uint8>(datagram));
	const sequence = packet.read.<uint16>();
	// Drop stale packets; datagrams arrive out of order.
	if (sequence <= lastSequence) continue;
	lastSequence = sequence;
	const x = packet.read.<float32, -1024, 1024, 18>();
	const y = packet.read.<float32, -1024, 1024, 18>();
}
```

Over a reliable WebTransport stream the packets self-frame: the header carries each packet's bit length, so a receiver can accumulate stream chunks and carve packets out by the same header ```read.<PacketReader>``` uses for nesting.

## Notes

- Datagrams should fit a single MTU, which is what the ```Size = 1400``` default is for; the fixed-length word buffer turns an overrun into a RangeError at the write instead of silent truncation.
- The byte view of the word buffer uses platform byte order per the array views section. A cross-platform wire deployment pins the word order, either by swapping on little-endian platforms when producing ```bytes``` or by keeping ```BufferBits``` at 8; it's elided here to keep the shift math front and center.
- The string overloads are deliberately ASCII: the ```uint.<7>``` boundary makes that a checked guarantee. Length-prefixed UTF-8 via ```TextEncoder``` composes the same way with ```write.<uint8>``` per byte.
