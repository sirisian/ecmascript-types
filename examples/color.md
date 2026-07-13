# Color

Packed RGBA and color math: the type behind CSS's ```rgb()```/```rgba()```, Skia's ```SkColor```, and a shader's ```vec4```. A color wants two representations at once - a compact four-byte word for storage and framebuffers, and a floating vector for correct blending - and it crosses between hex strings, integer channels, and linear-space math. This example builds it from value type classes (the packed representation), [operator overloading](../operatoroverloading.md) (the hex cast and blending), [primitive metadata](../primitivemetadata.md) (channel bounds), and [SIMD](../simd.md) (linear-space compositing). It's also the answer to a standing request to make ```Color``` a built-in type with ```#fff``` literal syntax: the type system already expresses it as user code, and the one thing it can't reproduce - the bare literal - is the one thing that shouldn't be a language feature.

Features exercised:

- A value type class of four ```uint8``` channels is exactly four bytes, so ```[].<Color>``` is a framebuffer: it aliases ```[].<uint8>``` for upload and ```[].<uint32>``` for per-pixel fills, with no serialization step between the color objects and the bytes.
- A user-defined cast from ```string``` - ```operator Color(hex: string)```, a member of the target that runs with no receiver - makes ```const c: Color = '#3af'``` parse hex at the assignment, which is as close to a bare ```#3af``` literal as the grammar allows.
- ```rgb``` and ```rgba``` are ordinary functions, the literal forms the request asked for, with ```rgba```'s alpha a ```float32.<{ minimum: 0, maximum: 1 }>``` so an out-of-range value is a boundary error, not a silent wrap.
- Correct compositing happens in linear space on a ```float32x4```: premultiplied over-compositing is a vector add and a scalar broadcast, the blend that ```uint8``` sRGB channels get visibly wrong.
- ```LinearColor``` wraps that vector as a nominal class so its ```over``` and ```+``` operators stay its own, the alias-versus-class distinction the [SIMD](../simd.md) extension draws, and the Color-to-linear cast is added with a ```partial class``` that leaves the four-byte layout untouched.

## The Color Type

Four channels, gamma-encoded sRGB, packed. Every field is a value type, so ```Color``` is one: four bytes, alignment 1, no per-object header. The cast from a hex string is declared here too, so a string literal becomes a color wherever one is expected.

```js
class Color {
	r: uint8;
	g: uint8;
	b: uint8;
	a: uint8;

	// Cast from #rgb, #rgba, #rrggbb, or #rrggbbaa, with or without the '#'.
	// Declared as a cast *from* the source, so it takes the string and runs
	// with no receiver, and is consulted wherever a Color is expected.
	operator Color(hex: string) {
		let s = hex[0] == '#' ? hex.slice(1) : hex;
		if (s.length == 3 || s.length == 4) {
			s = [...s].map(c => c + c).join(''); // Expand each nibble to a byte
		}
		const packed = uint32.parse(s.length == 6 ? s + 'ff' : s, 16); // Default alpha to opaque
		return {
			r: uint8(packed >> 24), // >> is logical on uint32, so no sign extension
			g: uint8((packed >> 16) & 0xff),
			b: uint8((packed >> 8) & 0xff),
			a: uint8(packed & 0xff),
		} := Color;
	}
}

Color.byteLength; // 4
Color.alignment;  // 1
```

An array of colors is therefore contiguous, and reinterpreting it costs nothing - the same no-copy view path the [particle system](particlesystem.md) uses for vertices:

```js
const pixels: [1920 * 1080].<Color>; // ~8 MB, zero-filled (transparent black)
const bytes = [].<uint8>(pixels);    // r, g, b, a per pixel: the upload path
const words = [].<uint32>(pixels);   // One packed word per pixel: fast fills and copies
```

## Constructors

```rgb``` and ```rgba``` are the literal forms the request asked for, as ordinary functions. Alpha is a ```0..1``` float, so it is validated at the call:

```js
function rgb(r: uint8, g: uint8, b: uint8): Color {
	return { r, g, b, a: 255 } := Color;
}
function rgba(r: uint8, g: uint8, b: uint8, a: float32.<{ minimum: 0, maximum: 1 }>): Color {
	return { r, g, b, a: uint8(Math.round(a * 255)) } := Color;
}

const white = rgb(255, 255, 255);
const halfRed = rgba(255, 0, 0, 0.5);
// rgba(255, 0, 0, 1.5); // TypeError: 1.5 is outside [0, 1]

const c: Color = '#3af';       // Parsed at the assignment: rgb(51, 170, 255)
const d = Color('#12345678');  // Explicit call of the same operator
const e: Color = '#ff000080';  // Half-opaque red
```

## Gradients

A per-channel interpolation, computed in ```float32``` so the unsigned channels never underflow. This is fine for a UI gradient; physically correct blending is the next section.

```js
function mix(a: Color, b: Color, t: float32.<{ minimum: 0, maximum: 1 }>): Color {
	const lerp = (x: uint8, y: uint8): uint8 =>
		uint8(Math.round(float32(x) + (float32(y) - float32(x)) * t));
	return { r: lerp(a.r, b.r), g: lerp(a.g, b.g), b: lerp(a.b, b.b), a: lerp(a.a, b.a) } := Color;
}

// Fill one row of a framebuffer left-to-right, then let the byte view upload it.
function gradientRow(row: [].<Color>, left: Color, right: Color) {
	const last = float32(row.length - 1);
	for (let x: uint32 = 0; x < row.length; ++x) {
		row[x] = mix(left, right, float32(x) / last);
	}
}
```

## Linear-Space Compositing

sRGB channels are gamma-encoded, so adding or averaging them directly is wrong - a half-covered red over green is muddy rather than the color the coverage implies. Correct compositing decodes to linear light, works there, and encodes back. The linear color is one ```float32x4```, premultiplied so the Porter-Duff *over* operator is a single add. It is a nominal class rather than an alias of ```float32x4```, so ```over``` and ```+``` belong to it and not to every vector of four floats:

```js
function srgbToLinear(c: float32): float32 {
	return Math.pow(c, 2.2); // Approximation of the sRGB transfer curve
}
function linearToSrgb(c: float32): float32 {
	return Math.pow(c, 1 / 2.2);
}

class LinearColor {
	rgba: float32x4; // (r, g, b, a), premultiplied, linear

	// src over dst, premultiplied: result = src + dst * (1 - src.a).
	over(dst: LinearColor): LinearColor {
		return { rgba: this.rgba + dst.rgba * (1 - this.rgba.w) } := LinearColor;
	}
	operator+(rhs: LinearColor): LinearColor { // Additive blending, for light
		return { rgba: this.rgba + rhs.rgba } := LinearColor;
	}

	// Encode back to gamma-space sRGB. Un-premultiply first.
	operator Color() {
		const alpha = this.rgba.w;
		const inv: float32 = alpha > 0 ? 1 / alpha : 0;
		const encode = (c: float32): uint8 => uint8(Math.round(linearToSrgb(c * inv) * 255));
		return {
			r: encode(this.rgba.x),
			g: encode(this.rgba.y),
			b: encode(this.rgba.z),
			a: uint8(Math.round(alpha * 255)),
		} := Color;
	}
}

// The decode direction is added to Color without touching its layout.
partial class Color {
	operator LinearColor() {
		const inv: float32 = 1 / 255;
		const alpha = float32(this.a) * inv;
		// Decode RGB to linear, keep the 4th lane at 1, then multiply the whole
		// vector by alpha: the RGB lanes come out premultiplied and the 4th lane
		// becomes the straight alpha.
		const linear = float32x4(
			srgbToLinear(float32(this.r) * inv),
			srgbToLinear(float32(this.g) * inv),
			srgbToLinear(float32(this.b) * inv),
			1
		);
		return { rgba: linear * alpha } := LinearColor;
	}
}
```

Flattening a stack of layers is then a fold in linear space, encoding once at the end. The casts make the boundaries invisible: a ```Color``` becomes a ```LinearColor``` at the ```over``` argument, and the accumulator becomes a ```Color``` at the ```return```:

```js
// Composite bottom-to-top: each layer goes over the accumulated result.
function flatten(layers: [].<Color>): Color {
	let acc: LinearColor; // Zero: transparent black
	for (const layer of layers) {
		acc = LinearColor(layer).over(acc); // Color -> LinearColor at the argument
	}
	return acc; // LinearColor -> Color at the return
}
```

## Coverage Notes

Writing this surfaced one deliberate gap and confirmed the rest:

- **The bare ```#3af``` literal.** Not provided, and the example is the argument for leaving it that way. A ```#```-prefixed token begins a private field, and the proposal types literals by propagation rather than by prefixes or suffixes, so there is no lexer hook a color literal could use without colliding. The user-defined cast from ```string``` is the spelling - ```const c: Color = '#3af'``` parses at the boundary, and a tagged template would carry a computed one - so the only thing a built-in ```Color``` primitive would add is the unquoted token, and everything above is the case for not adding it. This is the resolution of the "types might change" question that motivated the type: they don't need to.
- **Casts in both directions.** Resolved: the conversions section provides the cast *from* a source (```operator Color(hex: string)```, a member of the target with no receiver) for hex parsing, and the parameterless cast *to* a target (```operator LinearColor()``` converting the receiver) for the gamma boundary, so ```Color``` and ```LinearColor``` cross assignment, argument, and return boundaries with no named conversion call - which is what makes ```flatten``` read as if the two were one type.
- **Packed layout and views.** Resolved: four ```uint8``` fields make ```Color``` a four-byte value type, so ```[].<Color>``` aliases ```[].<uint8>``` for a framebuffer upload and ```[].<uint32>``` for per-pixel fills, the same path the particle system uses for vertices, with ```Color.byteLength``` the stride a copy or upload uses.
- **A vector with its own operators.** Resolved: ```LinearColor``` wraps a ```float32x4``` as a nominal class rather than aliasing it, so ```over``` and ```+``` stay on ```LinearColor``` instead of the shared lane type where every ```float32x4``` would inherit them - the alias-versus-class rule the [SIMD](../simd.md) extension states. Premultiplied over-compositing is then a vector add and a scalar broadcast, and a hardware SIMD add lowers it directly.
- **Channel bounds.** Resolved: ```rgba```'s alpha is a ```float32.<{ minimum: 0, maximum: 1 }>``` from [primitive metadata](../primitivemetadata.md), so a caller passing ```1.5``` is a boundary error rather than a silent wrap, while the integer channels are ```uint8``` and need no annotation to reject an out-of-range value.
- **Extending a laid-out class.** Resolved: the Color-to-linear cast is added with a ```partial class Color```, which the class extension section allows for methods and operators precisely because it adds no positional fields - so the four-byte layout the framebuffer views depend on is fixed by the primary declaration and the extension can't disturb it.
- **The sRGB transfer curve.** Left approximate. ```Math.pow(c, 2.2)``` stands in for the exact piecewise sRGB curve, which a production implementation would write out; nothing in the type system blocks the exact form, and the ```float32``` math is the same either way. Dimensioned color spaces - tagging a value as sRGB versus Display-P3 the way the [primitive metadata](../primitivemetadata.md) document tags units - would make mixing color spaces a compile error, and are the natural next step if this grew past two representations.
