---
pcx_content_type: concept
title: Remote-procedure call (RPC)
meta:
  description: The built-in, JavaScript-native RPC system built into Workers and Durable Objects.
---

# Remote-procedure call (RPC)

Workers provide a built-in, JavaScript-native [RPC (Remote Procedure Call)](https://en.wikipedia.org/wiki/Remote_procedure_call) system, allowing you to:

- Define public methods on your Worker that can be called by other Workers on the same Cloudflare account, via [Service Bindings](/workers/runtime-apis/bindings/service-bindings/rpc)
- Define public methods on [Durable Objects](/durable-objects) that can be called by other workers on the same Cloudflare account that declare a binding to it.

The RPC system is designed to feel as similar as possible to calling a JavaScript function in the same Worker. In most cases, you should be able to write code in the same way you would if everything was in a single Worker.

## Example

{{<render file="_service-binding-rpc-example.md" productFolder="workers">}}

The client, in this case Worker A, calls Worker B and tells it to execute a specific procedure using specific arguments that the client provides. This is accomplished with standard JavaScript classes.

## All calls are asynchronous

Whether or not the method you are calling was declared asynchronous on the server side, it will behave as such on the client side. You must `await` the result.

Note that RPC calls do not actually return `Promise`s, but they return a type that behaves like a `Promise`. The type is a "custom thenable", in that it implements the method `then()`. JavaScript supports awaiting any "thenable" type, so, for the most part, you can treat the return value like a Promise.

(We'll see why the type is not actually a Promise a bit later.)

## Structured clonable types, and more

Nearly all types that are [Structured Cloneable](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#supported_types) can be used as a parameter or return value of an RPC method. This includes, most basic "value" types in JavaScript, including objects, arrays, strings and numbers.

As an exception to Structured Clone, application-defined classes (or objects with custom prototypes) cannot be passed over RPC, except as described below.

The RPC system also supports a number of types that are not Structured Cloneable, including:

- Functions, which are replaced by stubs that call back to the sender.
- Application-define classes that extend `RpcTarget`, which are similarly replaced by stubs.
- [ReadableStream](/workers/runtime-apis/streams/readablestream/) and [WriteableStream](/workers/runtime-apis/streams/writablestream/), with automatic streaming flow control.
- [Request](/workers/runtime-apis/request/) and [Response](/workers/runtime-apis/response/), for conveniently representing HTTP messages.
- RPC stubs themselves, even if the stub was received from a third Worker.

## Functions

You can send a function over RPC. When you do so, the function is replaced by a "stub". The recipient can call the stub like a function, but doing so makes a new RPC back to the place where the function originated.

### Return functions from RPC methods

{{<render file="_service-binding-rpc-functions-example.md" productFolder="workers">}}

### Send functions as parameters of RPC methods

You can also send a function in the parameters of an RPC. This enables the "server" to call back to the "client", reversing the direction of the relationship.

Because of this, the words "client" and "server" can be ambiguous when talking about RPC. The "server" is a Durable Object or WorkerEntrypoint, and the "client" is the Worker that invoked the server via a binding. But, RPCs can flow both ways between the two. When talking about an individual RPC, we recommend instead using the words "caller" and "callee".

## Class Instances

To use an instance of a class that you define as a parameter or return value of an RPC method, you must extend the built-in `RpcTarget` class.

Consider the following example:

```toml
---
filename: wrangler.toml
---
name = "counter"
main = "./src/counter.js"
```

```js
---
filename: src/counter.js
---
import { WorkerEntrypoint, RpcTarget } from "cloudflare:workers";

class Counter extends RpcTarget {
  #value = 0;

  increment(amount) {
    this.#value += amount;
    return this.#value;
  }

  get value() {
    return this.#value;
  }
}

export class CounterService extends WorkerEntrypoint {
  async newCounter() {
    return new Counter();
  }
}

export default {
  fetch() {
    return new Response("ok")
  }
}
```

The method `increment` can be called directly by the client, as can the public property `value`:

```toml
---
filename: wrangler.toml
---
name = "client-worker"
main = "./src/clientWorker.js"
services = [
  { binding = "COUNTER_SERVICE", service = "counter", entrypoint = "CounterService" }
]
```

```js
---
filename: clientWorker.js
---
export default {
  async fetch(request, env) {
    using counter = await env.COUNTER_SERVICE.newCounter();

    await counter.increment(2);   // returns 2
    await counter.increment(1);   // returns 3
    await counter.increment(-5);  // returns -2

    const count = await counter.value;  // returns -2

    return new Response(count);
  }
}
```

{{<Aside type="note" description="The `using` declaration">}}
Refer to [Explicit Resource Management](/workers/runtime-apis/rpc/lifecycle) to learn more about the `using` declaration shown in the example above.
{{</Aside>}}

Classes that extend `RpcTarget` work a lot like functions: the object itself is not serialized, but is instead replaced by a stub. In this case, the stub itself is not callable, but its methods are. Calling any method on the stub actually makes an RPC back to the original object, where it was created.

As shown above, you can also access properties of classes. Properties behave like RPC methods that don't take any arguments — you await the property to asynchronously fetch its current value. Note that the act of awaiting the property (which, behind the scenes, calls `.then()` on it) is what causes the property to be fetched. If you do not use `await` when accessing the property, it will not be fetched.

{{<Aside type="note" description="Returning class instances is more efficient than returning objects with many functions">}}
While it's possible to define a similar interface to the caller using an object that contains many functions, this is less efficient. If you return an object that contains five functions, then you are creating five stubs. If you return a class instance, where the class declares five methods, you are only returning a single stub. Returning a single stub is often more efficient and easier to reason about. Moreover, when returning a plain object (not a class), non-function properties of the object will be transmitted at the time the object itself is transmitted; they cannot be fetched asynchronously on-demand.
{{</Aside>}}

{{<Aside type="note" description="Why not use Structured Clone semantics for classes">}}
Classes which do not inherit `RpcTarget` cannot be sent over RPC at all. This differs from Structured Clone, which defines application-defined classes as clonable. Why the difference? By default, the Structured Clone algorithm simply ignores an object's class entirely. So, the recipient receives a plain object, containing the original object's instance properties but entirely missing its original type. This behavior is rarely useful in practice, and could be confusing if the developer had intended the class to be treated as an `RpcTarget`. So, Workers RPC has chosen to disallow classes that are not `RpcTarget`s, to avoid any confusion.
{{</Aside>}}

### Promise pipelining

When you call an RPC method and get back an object, it's common to immediately call a method on the object:

```js
// Two round trips.
using counter = await env.COUNTER_SERVICE.getCounter();
await counter.increment();
```

But consider the case where the Worker service that you are calling may be far away across the network, as in the case of [Smart Placement](/workers/runtime-apis/bindings/service-bindings/#smart-placement) or [Durable Objects](/durable-objects). The code above makes two round trips, once when calling `getCounter()`, and again when calling `.increment()`. We'd like to avoid this.

With most RPC systems, the only way to avoid the problem would be to combine the two calls into a single "batch" call, perhaps called `getCounterAndIncrement()`. However, this makes the interface worse. You wouldn't design a local interface this way.

Workers RPC allows a different approach: You can simply omit the first `await`:

```js
// Only one round trip! Note the missing `await`.
using promiseForCounter = env.COUNTER_SERVICE.getCounter();
await promiseForCounter.increment();
```

In this code, `getCounter()` returns a promise for a counter. Normally, the only thing you would do with a promise is `await` it. However, Workers RPC promises are special: they also allow you to initiate speculative calls on the future result of the promise. These calls are sent to the server immediately, without waiting for the initial call to complete. Thus, multiple chained calls can be completed in a single round trip.

How does this work? The promise returned by an RPC is not a real JavaScript `Promise`. Instead, it is a custom ["Thenable"](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise#thenables). It has a `.then()` method like `Promise`, which allows it to be used in all the places where you'd use a normal `Promise`. For instance, you can `await` it. But, in addition to that, an RPC promise also acts like a stub. Calling any method name on the promise forms a speculative call on the promise's eventual result. This is known as "promise pipelining".

This works when calling properties of objects returned by RPC methods as well. For example:

```js
---
filename: myService.js
---
import { WorkerEntrypoint } from "cloudflare:workers";

export class MyService extends WorkerEntrypoint {
  async foo() {
    return {
      bar: {
        baz: () => "qux"
      }
    }
  }
}
```

```js
---
filename: client.js
---
export default {
  async fetch(request, env) {
    using foo = await env.MY_SERVICE.foo();
    let baz = await foo.bar.baz();
    return new Response(baz);
  }
}
```

If the initial RPC ends up throwing an exception, then any pipelined calls will also fail with the same exception

## ReadableStream, WriteableStream, Request and Response

You can send and receive [`ReadableStream`](/workers/runtime-apis/streams/readablestream/), [`WriteableStream`](/workers/runtime-apis/streams/writablestream/), [`Request`](/workers/runtime-apis/request/), and [`Response`](/workers/runtime-apis/response/) using RPC methods. When doing so, bytes in the body are automatically streamed with appropriate flow control.

Only [byte-oriented streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_byte_streams) (streams with an underlying byte source of `type: "bytes"`) are supported.

In all cases, ownership of the stream is transferred to the recipient. The sender can no longer read/write the stream after sending it. If the sender wishes to keep its own copy, it can use the [`tee()` method of `ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/tee) or the [`clone()` method of `Request` or `Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response/clone). Keep in mind that doing this may force the system to buffer bytes and lose the benefits of flow control.

## Forwarding RPC stubs

A stub received over RPC from one Worker can be forwarded over RPC to another Worker.

```js
---
filename: introducer.js
---
using counter = env.COUNTER_SERVICE.getCounter();
await env.ANOTHER_SERVICE.useCounter(counter);
```

Here, three different workers are involved:

1. The calling Worker (we'll call this the "introducer")
2. `COUNTER_SERVICE`
3. `ANOTHER_SERVICE`

When `ANOTHER_SERVICE` calls a method on the `counter` that is passed to it, this call will automatically be proxied through the introducer and on to the [`RpcTarget`](/workers/runtime-apis/rpc/compatible-types) class implemented by `COUNTER_SERVICE`.

In this way, the introducer Worker can connect two Workers that did not otherwise have any ability to form direct connections to each other.

Currently, this proxying only lasts until the end of the Workers' execution contexts. A proxy connection cannot be persisted for later use.

## More Details

{{<directory-listing showDescriptions="true">}}

## Limitations

- [Smart Placement](/workers/configuration/smart-placement/) is currently ignored when making RPC calls. If Smart Placement is enabled for Worker A, and Worker B declares a [Service Binding](/workers/runtime-apis/bindings) to it, any RPC calls to Worker A will run locally.