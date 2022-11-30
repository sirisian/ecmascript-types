# Math.random Extension

The goal of this extension is to describe how a random proposal would change with types. This will cover the basic generation tasks like random numbers and seeded randoms with various methods.

Refer to this as I'll base these notes off of it: https://github.com/tc39/proposal-seeded-random

**Note**: This is a pseudorandom library identical to Math.random and not cryptographically random. It's designed around speed for things like games. Extending the Crypto library later into ECMAScript would probably be ideal to have a wide range of Crypto RNG generation.

### Math.random<T, M=???>()

The first addition is a generic version of ```Math.random``` for the float types: 

```float16```, ```float32```, ```float64```

The second generic argument is a method set to the browser default PRNG method. Not sure how to namespace that. Like ```Math.PRNG.Default```.

Rapidly generating arrays of random numbers in the range \[0, 1):

```js
const a:[100]<float32>;
Math.random<float32>(a);
```

### Math.random<T, M=???>(min:T, max:T)
  
For generating between a min and max inclusive the following data types are allowed:
  
```int8```, ```int16```, ```int32```, ```int64```  
```uint8```, ```uint16```, ```uint32```, ```uint64```  
```bigint```  
```float16```, ```float32```, ```float64```
  
So to generate an int32 between -5 and 5 you only need to do:
  
```js
Math.random<int32>(-5, 5);
```

Rapidly generating arrays of random numbers in the range \[-1, 1]:

```js
const prng = Math.seededRandom<float32>({seed:0});
const a:[100]<float32>;
prng.random(a, -1, 1);
```

Could also define ```Math.random<T, M=???>(max)``` since function overloading exists.
  
### Math.seededRandom<T, M=???>(config)

```js
const prng = Math.seededRandom<float32>({ seed: 0 });
const i = prng.random(); // [0, 1)
const j = prng.random(-5, 5); // [-5, 5]
```

Rapidly generating arrays of random numbers:

```js
const prng = Math.seededRandom<float32>({ seed: 0 });
const a:[100]<float32>;
prng.random(a, -1, 1);
```

### WIP

Extracting state to save is still up for debate. I'd think a standardized ```[]<uint8>``` array of binary data would be sufficient for each PRNG method. Basically just need a way that works as nobody is inspecting it probably.
