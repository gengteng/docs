# 异步析构函数

> 原文: Asynchronous Destructors
> 地址: https://boats.gitlab.io/blog/post/poll-drop/    
> 作者: Withoutboats(Saoirse Shipwreckt@Github)
> 翻译: 耿腾

The first version of async/await syntax is in the beta release, set to be shipped to stable in 1.39 on November 7, next month. There are a wide variety of additional features we could add to async/await in Rust beyond what we’re shipping in that release, but speaking for myself I know that I’d like to pump the breaks on pushing forward big ticket items in this space. Let’s let the ecosystem develop around what we have now before we start sprinting toward more big additions to the language.

There are a few exceptions, though: smaller features that would make the async/await integration a bit smoother, and which are smaller designs. One of these is adding an `IntoFuture` trait - this would allow applying the await operator to something that isn’t a future, but has an obvious way to transform it into a future. This is similar to how the `IntoIterator` trait interacts with for loops today. [Seanmonstar has already opened a PR to add this feature to nightly experimentally.](https://github.com/rust-lang/rust/pull/65244)

The other smaller feature I’ve been mulling over lately is support for asynchronous destructors. The rest of this blog post contains notes that have occurred to me these past few weeks.

## What are async destructors for?

One way that destructors are used in Rust is to “finish out” some sort of IO operation or handle. Closing files and sockets is a common example, but in some higher level APIs you may also need to send a message over IO or wait for some operation to complete. For blocking IO, this works just like any other kind of destructor. But there is no way to perform these operations in a non-blocking way right now: a destructor cannot yield control, even if it’s being called inside of an async context.

In addition to these higher level APIs, async destructors have a really interesting low level use case: building futures for handling completion based APIs like Linux’s io_uring and Window’s IOCP. As Matthias247 discussed [here](https://gist.github.com/Matthias247/ffc0f189742abf6aa41a226fe07398a8), these futures need to hold on to the buffers they operate on until the IO actually completes, even if the program’s interest in that IO is cancelled. One great way to implement this would be to wait until the IO completes in the future’s destructor, but without async destructors, the future would need to block the entire thread it is running on, rather than blocking this task alone.

The concept of async destructors seems straightforward: these destructors when an object goes out of scope in an async context, but they can yield control. Naively, one might imagine adding a new async version of the drop trait:

```rust
trait AsyncDrop {
    async fn drop(&mut self);
}
```

Now, whenever we drop a type that implements `AsyncDrop` in an async context, we will call this destructor. However, there are a few complications.

## The non-async drop problem

The first problem we encounter is this: what happens when you drop something with an async destructor in a non-async context? There are two options:

1. Call it’s non-async destructor, like every other type.
2. Introduce some kind of executor to the runtime (probably just [block on](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.19/futures/executor/fn.block_on.html)) to call as part of the drop glue.

The first has some pitfalls: if you only implement an async destructor and not a non-async destructor, your type’s destructor will not be called if its dropped outside of an async context. That’s an unfortunate bug we’d like to avoid.

However, the second seems unworkable. We’d be making a lot of decisions for all users that may not be the best choice for their use cases. Maybe there’s a more optimal way to implement the non-async destructor than blocking on the async destructor. Maybe they’d prefer to spawn the async destructor as a new task on some kind of multithreaded executor.

The best situation is to give the user control by calling the normal destructor in non-async contexts. If they want that to block on their async destructor, they can implement it that way themselves. But the API design should make it hard to accidentally forget to implement the normal destructor when you implement the async destructor.

## The destructor state problem

There’s another problem as well, which I don’t think I’ve seen mentioned in any previous discussions. Using the definition of `AsyncDrop` from above, users can introduce new state during the destructor that will exist across yield points. For example:

```rust
impl AsyncDrop for MyType {
    async fn drop(&mut self) {
        let rc = Rc::new(0);
        self.async_method(&rc).await;
    }
}
```

Even if `MyType` implements `Send` and `Sync`, the async destructor of `MyType` now does not. This means that if you have an async function that captures a value of `MyType`, the future that function evaluates to now also isn’t `Send`, because it has to run that non-send async destructor. This is pretty problematic, and it gets worse when we are examining generic code. There’s not a good way to support a bound that says “if this type has an async destructor, that async destructor must be `Send`”.

Send is not the only problem: a similar problem emerges when dropping trait objects. Trait objects virtually dispatch their destructors, because the concrete type is no longer known. If async destructors can introduce new state, that state can be of variable size; we’d need to heap allocate the destructor futures, or some sort of alloca trick, or maybe some other clever trick, to be able to dynamically dispatch the drop call on a trait object.

So, at least in an initial version, it would be ideal to prevent async destructors from introducing new state. Instead, the state needed to asynchronously tear down a type should be entirely contained in the type itself. Fortunately, we already have an API pattern which enforces that restriction: the pattern of using “poll” methods, instead of async methods.

## A sketched proposal

Based on these comments, I have a concrete proposal that resolves these concerns and is both a minimal addition to the language and APIs.

First, we add a new method to `Drop`, which has a default implementation:

```rust
trait Drop {
    fn drop(&mut self);

    // this method is added to Drop:
    fn poll_drop(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        unsafe { ptr::drop_in_place(self.get_unchecked_mut()); }
        Poll::Ready(())
    }
}
```

Second, we change the way that drop glue works in an async context. In an async context, instead of calling `drop`, we call `poll_drop` in a loop until it returns `Ready`. We do this for all types: as you can see, this doesn’t change the behavior for existing types, because the default implementation of `poll_drop` forwards to `drop` and then returns `Ready`.

We should also add this function to the standard library, and probably the prelude:

```rust
async fn drop_async<T>(_: T) { }
```

(Yes, the empty drop fn trick falls out of this design for async destructors too!)

Poll_drop is not an async method, it is a “poll” method, just like the poll method on `Future` and the IO methods on [`AsyncRead` and `AsyncWrite`](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.19/futures/io/index.html). This is very critical: a poll method doesn’t introduce any new state. All of the state for the async destructor must exist in the type getting destroyed, sidestepping the problem of destructor state.

Because this is just an additional method in the drop trait, users implementing an async destructor must implement the drop method as well. This helps avoid the problem of types that don’t properly implement `Drop` when they implement `AsyncDrop` (and giving us plenty of opportunities through docs and lints to emphasize for users that implementing `poll_drop` does not do anything outside of async contexts!).

Obviously, writing an async destructor using a poll method is not as convenient as being able to use an async fn, and this is the biggest downside of this plan. However, I don’t see a short term plan for resolving the problem of additional state in destructors other than using a poll method. I think, also, this represents a successful incremental approach to the problem. First, we make async destructors possible (using poll methods), then we can revisit the problem later to make them easier (using async methods), when more necessary infrastructure exists.