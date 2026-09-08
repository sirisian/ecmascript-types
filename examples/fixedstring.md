# Fixed-length strings

A ```string``` has no layout — its size is a property of the value, not of the type ([memory layout](../memorylayout.md)) — so a record with a name in it is not a value type, and an array of a thousand such records is a thousand heap objects rather than one buffer.

This is the type that closes that gap. It is a **worked example rather than a language feature**: it needs no engine support beyond the [UTF-8 codec](../serialization.md#strings-at-a-binary-boundary), and a program whose format does not match its padding convention writes its own class over the same ```[N].<uint8>``` and keeps the codec. Rust reaches the same shape the same way — ```arrayvec::ArrayString``` and ```heapless::String``` are libraries, not language.

## The type

<!-- run: setup -->
```js
class FixedString<N: uint32> {
  #bytes: [N].<uint8>;

  constructor(value: string = '') {
    if (value.length > 0 && value.charCodeAt(value.length - 1) === 0) {
      throw new TypeError('a FixedString cannot end in U+0000: the padding would swallow it');
    }
    value.toUtf8(this.#bytes);
  }

  static get capacity(): uint32 { return N; }

  // The bytes in use: the slot is zero-padded, so the value ends at the last
  // non-zero byte.
  get byteLength(): uint32 {
    let used: uint32 = N;
    while (used > 0 && this.#bytes[used - 1] === 0) { used = used - 1; }
    return used;
  }

  operator string() { return String.fromUtf8(this.#bytes.slice(0, this.byteLength)); }

  operator ==(other: FixedString.<N>) {
    if (this.byteLength !== other.byteLength) { return false; }
    for (let i: uint32 = 0; i < this.byteLength; i = i + 1) {
      if (this.#bytes[i] !== other.#bytes[i]) { return false; }
    }
    return true;
  }

  toString(): string { return String.fromUtf8(this.#bytes.slice(0, this.byteLength)); }
}
```

## What it buys

A record with a name is a value type again, and a table of them is one allocation:

<!-- run -->
```js
class Employee {
  name: FixedString.<32>;
  id: uint32;
  salary: float64;
}

Employee.byteLength;             // 48, 44 of content padded to float64's alignment
const table: [1000].<Employee>;  // 48,000 contiguous bytes

const ref e = table[0];          // `ref`, or the write below lands on a copy
e.name = new FixedString.<32>('Ada');
let n: string = table[0].name;   // 'Ada', decoded on read
```

Reading and writing:

<!-- run -->
```js
const s = new FixedString.<8>('café');
let t: string = s;               // 'café' — five bytes used, three of padding
s == new FixedString.<8>('café'); // true
String(s) === 'café';             // true
s === 'café';                     // FALSE — see below
Number(s.byteLength);             // 5
Number(FixedString.<8>.capacity); // 8
```

## What it costs

**```===``` does not work against a string.** Strict equality runs no declared conversion, so ```s === 'café'``` is ```false``` and the spellings that work are ```==``` and ```String(s)```. This is the one place where a library type is visibly worse than a primitive would be, and it is the evidence that would justify making one: if real code reaches for ```String(s) === t``` often enough that ```==``` stops being the obvious spelling, the escalation is a ```str.<N>``` primitive whose reads yield ordinary interned strings, entered deliberately with the read cost measured.

**A read decodes.** ```e.name``` is bytes until something asks for a string, and then it allocates and interns. Scanning a table and comparing names is faster through ```==``` on the bytes than through two decodes.

**Nothing truncates.** A value whose encoding does not fit is a TypeError and nothing is written, which is the rule the codec states and the reason this is not ```strncpy``` or ```CHAR(n)```. That includes the case that catches people out — a multibyte value that would split a code point at the boundary is refused, not cut:

<!-- run: throws -->
```js
new FixedString.<2>('Ada');      // TypeError: "3" bytes of UTF-8 do not fit in "2"
new FixedString.<4>('caféé');    // TypeError: "7" bytes of UTF-8 do not fit in "4"
new FixedString.<8>('\uD800');   // TypeError: unpaired surrogate, no UTF-8 encoding
```

## The padding convention, and why this one

Zero-padded; the value ends at the last non-zero byte. Chosen because the point is to overlay an external format byte for byte, and such formats zero-pad. A length prefix would round-trip better and would *not* match the bytes on disk, which is the job.

The cost is stated rather than hidden: an embedded ```U+0000``` survives, and a value **ending** in one cannot be stored and is refused on construction rather than silently altered.

<!-- run -->
```js
let a: string = new FixedString.<8>('a\u0000b');   // 'a\u0000b' — embedded, kept
```

And the value that cannot be stored:

<!-- run: throws -->
```js
new FixedString.<8>('ab\u0000');                   // TypeError — trailing, refused
```

A format with a different convention — length-prefixed, space-padded, NUL-terminated — is a different class over the same ```[N].<uint8>```, and reuses the codec unchanged. That separation is deliberate: padding is a property of a format, not of UTF-8.

## The bytes are private

```#bytes```, and ```operator ==``` is the only member that reaches across to another instance's copy of it. That works because a value type class carries its private fields through a copy â€” a typed parameter boundary copies the operand, and a copy that dropped the private store would leave the operator reading a field that is not there.
