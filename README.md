# A proposal for resource management in JS

*Initially inspired by [this ES Discuss thread](https://esdiscuss.org/topic/resource-management-eg-try-with-resources#content-2).*

JS really needs a resource management system. How often do you find yourself doing something like these?

```js
// Get some rows from a database, using the Promise disposer pattern
await db.connect(async connection => {
    const rows = await connection.query("SELECT * FROM TABLE")
    console.log(rows)
})

// Get some rows from a database, using `try`/`finally`
const connection = await db.connect()

try {
    const rows = await connection.query("SELECT * FROM TABLE")
    console.log(rows)
} finally {
    connection.close()
}

// Use a temp file
const tmp = require("tmp-promise")
const fs = require("fs/promises")

tmp.withFile(o => {
    console.log('Doing work with the temp file', o.path)
    await fs.writeFile(o.path, 'test', 'utf-8')
}).then(() => {
    console.log(`Temp file cleaned up!`)
})

// Multiple parallel connections
const Promise = require("bluebird")
await using([
    db.connect("db"),
    db.connect("db"), // Let's hope this doesn't fail... ;-)
], async ([conn1, conn2]) => {
    // use conn1 and conn 2 here
})
```

If so, it's pretty obvious this is all annoying boilerplate. What if, instead, it was like this?

```js
// Get some rows from a database
{
    with connection = await db.connect()
    const rows = await connection.query("SELECT * FROM TABLE")
    console.log(rows)
}

// Use a temp file
const tmp = require("tmp-promise")
const fs = require("fs/promises")

{
    with {path} = tmp.file()
    console.log('Doing work with the temp file', o.path)
    await fs.writeFile(o.path, 'test', 'utf-8')
}

console.log(`Temp file cleaned up!`)

// Multiple parallel connections
{
    with [conn1, conn2] = await Promise.withAll([
        () => db.connect(),
        () => db.connect(),
    ])
    // use conn1 and conn 2 here
}
```

## Proposal

So here's the basic context:

- A resource is an object with a `Symbol.dispose` method. Anything with that method can be used as a resource.

- `value[Symbol.dispose]()` - This is how you explicitly dispose of a resource. There are several ways a potential resource could implement this:
    - Built-in iterators and async iterators could implement this as an alias for `iter.return()`.
    - Subscriptions could implement this as an alias for `unsubscribe`.
    - Array buffers could implement this to detach the array buffer. (This is useful if the buffer is large.)
    - An object pool might use this to release an object back into the pool once it leaves the scope.
    - Other more complex scenarios like async temp files would need this to destroy the file before returning.
    - It's obviously recommended that this be idempotent, although the spec makes no requirement of this.

- `value[Symbol.asyncDispose]()` - This is how you explicitly dispose of an async resource.
    - Built-in async iterators could implement this as an alias for `iter.return()`.
    - Node streams could use this as an alias for `.destroy()` or `.close()`, as appropriate.
    - Other more complex scenarios like async temp files would need this to destroy the file before returning.

- `with value = resource` - This is how you create a resource. It's closed at the end of the block it's in, so you don't need to care about that.
    - These are created as `const` variables, so they carry all the necessary restrictions with it. (This is to make it clearer what's being closed.)
    - If there's multiple `with` statements, they are closed in the reverse order they were created.
    - This doesn't actually conflict with the legacy `with` statement - patterns can't start with parentheses.
    - Resources are closed regardless of the completion type (it can be "normal", "throw", "return", or "break").
    - If an error occurs during closure, the first error to happen is propagated, but not until all resources within that block are closed (successfully or not).
    - If `value` is `null` or `undefined`, nothing happens. This is to support things like `with res = getResource(); if (res == null) return false` or similar.
    - In sync contexts, it only calls `value[Symbol.dispose]()`.
    - In async contexts, it calls `value[Symbol.asyncDispose]()` if the method exists, but it falls back to `value[Symbol.dispose]()` if it doesn't.
    - In either case, the method is retrieved at the end of the block, so that's where the `TypeError` is thrown.

- `Object.withIterable(iterable) -> Iterable & Iterator & {[Symbol.dispose](): void} | AsyncIterable & AsyncIterator & {[Symbol.asyncDispose](): Promise<void>` - Convert an iterable to a resource that auto-returns when you're done. For iterables implementing `Symbol.iterator`, this returns a sync iterator proxy that also implements `Symbol.dispose` as an alias for `return`. For iterables implementing `Symbol.asyncIterator`, this works similarly, just for async iterators and returning an async iterator proxy that also implements `Symbol.asyncDispose` as an alias for `return`. This is useful if you're building an iterable operator that's more complex than what a `for` loop with `yield` could handle.

- `Promise.dispose(resource) -> Promise<void>` - This disposes of a resource explicitly.
    - For most async resources, if you need to dispose of them, you should call this method instead of calling `resource[Symbol.asyncDispose]()` directly - the resource might instead implement `resource[Symbol.dispose]()`, and it might throw synchronously (which could cause bugs).

- `Promise.withAll(factories: Iterable<() => Promise<asyncResource>>) -> Promise<[...values] & {[Symbol.asyncDispose](): void}>` - This creates and combines multiple async resources into an array of them, an array that also happens to have a `Symbol.asyncDispose` method set on it. If creating any resource fails, all remaining ones are closed as necessary, and the first error propagated. (The rest are swallowed.)
    - The main goal here is to open several resources in parallel and ensure they all close if one fails.
    - When `result[Symbol.asyncDispose]()` is called, this calls `value[Symbol.asyncDispose]()` for each resource if the method exists, but it falls back to `value[Symbol.dispose]()` if it doesn't. Also, all calls to `Symbol.asyncDispose` are awaited in parallel, like via `Promise.all`.
    - If, when closing, a resource lacks both `value[Symbol.dispose]()` and `value[Symbol.asyncDispose]()`, a `TypeError` is thrown. This doesn't block other resources, and it's as if there *was* a disposer method that just happened to throw the relevant `TypeError`.
    - This is analogous to Bluebird's [`Promise.using([...resources], ([...values]) => ...)`](http://bluebirdjs.com/docs/api/promise.using.html), so there's precedent. It's also hard to get right.
    - Maybe, a method to aggregate exceptions might be nice for this + `Promise.all`?

- `Promise.wrap(init: (body: (...values) => Promise<void>) => Promise<any>) -> Promise<[...values] & {[Symbol.asyncDispose](): Promise<void>}>`
    - This exists to adapt a promise disposer to an async resource, much like the `Promise` constructor is used to adapt callbacks to promises.
    - One could wrap an existing resource this way using an async function here with a `with` inside.
    - This is exposed as a built-in, since it's non-trivial to implement (you're literally having to do a restricted form of `call/cc` to do it).

Intentionally, there is no equivalent for any of the above utility methods for sync resources. For `Promise.dispose`, sync resources only have one possible symbol, `Symbol.dispose`, so you should prefer calling that over `Promise.dispose`. For `Promise.withAll`, the sync equivalent is just creating them with multiple `with` declarations - close-on-fail will just take care of itself naturally. For `Promise.wrap`, if you're dealing with a callback, JS doesn't have a "call with current continuation" primitive, so it's not possible to implement. If you're dealing with an object with an explicit disposer method, wrapping it is pretty trivial:

```js
function wrapFoo(...args) {
    const foo = getFoo(...args)
    return {value: foo, [Symbol.dispose]: () => foo.close()}
}

async function wrapFooAsync(...args) {
    const foo = await getFoo(...args)
    return {value: foo, [Symbol.asyncDispose]: async () => foo.close()}
}
```

## Why this way?

I know [this isn't the only proposal out there](https://github.com/rbuckton/proposal-using-statement), and many languages already use something like `using` or blocks:

- [Ruby uses blocks with `yield` + `ensure`](http://jakegoulding.com/blog/2012/10/01/using-ruby-blocks-to-ensure-resources-are-cleaned-up/)
- [Java uses `try`-with-resources](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
- [C# uses `using`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-statement)
- [Python uses `with`](http://preshing.com/20110920/the-python-with-statement-by-example/)
- The promise disposer pattern is basically Ruby's `yield` + `ensure`, just done with promises and a higher-order function instead.

There's also the concept of "deferred" blocks:

- Swift has [`defer` blocks](https://www.hackingwithswift.com/new-syntax-swift-2-defer).
- Go has [`defer` statements](https://gobyexample.com/defer).

But here's why I chose this route:

- Resources should be easy to deal with, hence the simple replacement of a token.

- In the language, iterators are already basically treated [RAII-style ("Resource Acquisition Is Initialization")](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) in `for ... of` because of their `return` method call behavior, even though they aren't *technically* resource-like from a pragmatic perspective. This has been used by [Node.js streams](https://nodejs.org/api/stream.html#stream_readable_symbol_asynciterator) for async iteration, and [I've come up with a related abuse](https://esdiscuss.org/topic/resource-management) in something topically related to this.

- RAII-style initialization with a keyword not implying the RHS is *conceptually* a resource makes it strictly easier and more flexible to work with. For example:
    - Atomic and read/write locks would be easier to use - you just scope the lock normally using a `with` statement, and it gets released when you're done. The moral equivalent of Java's `synchronized` method type would become just a simple top-level mutex.
    - You could make array buffers implement `Symbol.dispose` to detach the array buffer programmatically. This would make it easier to work with large temporary array buffers while making their lifetime explicit, so you're not pushing the assumption to the GC in potentially critial situations.
    - In the DOM, most of IndexedDB stuff would make better sense as methods returning promises to RAII-like resources as appropriate (like for database transactions), rather than what it is now.
    - In Node, if you're doing a lot of complex file system-related stuff with occasional interleaved computation and/or logging, it might be easier to avoid the nesting inherent with a `using` block and instead just use the file itself.
    - One might implement an object pool or unique ID system via returning resources that when it's closed, the object/ID is reclaimed for later. This avoids the necessary explicit use+free situation you'd normally need otherwise.
    - [C++](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization#C++11_example) and [Rust](https://doc.rust-lang.org/rust-by-example/scope/raii.html) both use and *strongly* encourage RAII. Their resource management style is what this proposal was inspired by, and since they have to deal with resources and potential leaks heavily, and so they've had to sugar over them to make them easier to use. I mean, take a look at [their example for files](https://rustbyexample.com/std_misc/file/open.html) - it's almost like magic, but in a good way. C++ works pretty similarly for sockets and other file management - it's basically effortless.

- You don't have to deal with the pyramid of doom when opening resources, even when they depend on other resources.
    - Unlike the [`using` proposal](https://github.com/rbuckton/proposal-using-statement), you can even drop a `console.log` between opening two different resources.
    - And it's *pretty obvious* that you could run into this issue in a hurry with Promises unless you're lucky enough to have a very non-trivial utility that can turn a `Iterable<(body: (value: T) => Promise<U>) => Promise<U>>` into a `(body: (value: [...T]) => Promise<U>) => Promise<U>` while handling all the edge cases that can arise from that (like timings).

- Reusing an existing keyword is always nice, especially when it still remains meaningful. Python's use of `with` was another influence, and it made me feel it was a little more justified.

- Block scopes are nice for restricting scopes already. Why invent yet another concept, when we can just reuse what we have?

## Polyfills?

I don't have any polyfills ready yet, but I would like to eventually. This also includes syntax, so it'd also be pending a Babel parser plugin.
