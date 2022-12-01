# Threading Extension and Notes

ECMAScript needs real threading where any function can be spawned as a thread to run asynchronously. Most value type data should be accessible across threads with some operations being atomic automatically. The syntax for creating and managing threads should be very minimal and effortless to use.

For example, you should be able to define a global ```a:uint32``` and in a thread ```Atomics.add(a, 5)``` it without shuffling it into a typed array.

```js
let a:uint32 = 0; 
function A() {
    Atomics.add(a, 5);
}
async function B() {
    A();
    Atomics.add(a, 5);
}
// Naive call syntax to show these execute on their own thread and callThread returns a promise.
await Promise.all([A.callThread(), B.callThread()]); // Join
a; // 15
```

The ```callThread``` method would return a Promise. Internally it spawns a thread that automatically closes over all the state referenced. In fully typed code this operation can be relatively optimizable; however, it is possible to use this with dynamic code with higher costs, but this usage would be rare.

It was my hope that a Cancelable Promise proposal would have been finalized by now. Assume one exists.

Some value type operations would be atomic automatically. Addition on integers for instance.
```js
let a:uint32 = 0;
function A() {
    while (true) {
        a += 5;
    }
}
A.callThread();
await new Promise(resolve => setTimeout(resolve, 100));
a; //
```

## Applications

* Game algorithms like pathfinding
* Parsing large binary data formats when using binary Web Sockets.

## Future Applications

* Building DOM nodes in a separate thread then appending in the main thread. This is intuitive for programmers, but currently is not possible. In an ideal web environment this would just work where you could document.createElement in a function and as long as you didn't try to reference the active DOM you'd be fine.
  * In a large single page application multiple threads could be spun up creating different sections of the DOM that are then joined and appended to the document.
* In cases where you're waiting for data from a REST call and get a JSON object back you then need to process the data. A thread could do the rest call, perform the JSON.parse, processing, then return back the data without having to postMessage.

## Concurrent Data Structures

Concurrent data structures would be nice to have with native implementations.

## Pipelines

WIP How do pipelines fit into this? Intuitively piping data to a threaded function should just work and create a thread. Is that a realistic scenario though?

# Node.JS

It should be assumed that Node.JS would use this as well where offloading parsing and expensive operations to threads is very beneficial.
