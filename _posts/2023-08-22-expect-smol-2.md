---
layout: post
title: What to expect from smol 2.0
categories: [async, rust, smol]
excerpt:
comments: true
---

The tree of robust software must be refreshed from time to time with the blood of breaking changes. [`smol`] is no exception.

[`smol`] version 1.0 was originally released on September 7th, 2020. Since then, we've seen almost thirty new minor version bumps of Rust and a small handful of ecosystem changes. Some of these changes are so important that it necessitates changes to the way that [`smol`] is built.

In addition to hype building, the purpose of this blog post is to outline the changes that will be coming in [`smol`] v2.0. If you are a [`smol`] user, you should be ready for these changes when they come. The `smol-rs` team is planning on releasing [`smol`] v2.0 in the near future, although we don't have a concrete date yet.

## I/O Safety

[Rust v1.63] was released with a new landmark feature: [I/O safety]. Prior to I/O safety, I/O resources were handled by passing around raw file descriptors and *hoping* that they were valid file resources. Here's how a safe interface to the [`read`] syscall would be defined without I/O safety.

```rust
fn read(fd: RawFd, buf: &mut [u8]) -> io::Result<usize>;
```

In theory, this all works out fine. In practice, it is very possible for the file descriptor to be taken out from under you, through other threads or processes. Being ready for this essentially makes it impossible to write safe code that uses I/O resources.

I/O safety fixes this by assigning a lifetime to the file descriptor and guaranteeing that the underlying I/O resource will be valid for that lifetime. This shifts the burden of keeping those resources alive onto the borrow checker. The above function would be rewritten as follows:

```rust
fn read(fd: BorrowedFd<'_>, buf: &mut [u8]) -> io::Result<usize>;
```

I/O safety is very important to [`smol`]. In order to handle async I/O, I/O resources are registered into a [global reactor]. It is possible for the resources to be deallocated while they are still in this global reactor, which can lead to missing or spurious events. Now that we can assign a lifetime to the file descriptor before we register it into the reactor, this isn't a problem anymore. [`smol`] v2.0 will have its [`Async`] type take [`AsFd`] types instead of [`AsRawFd`].

This new advantage comes with a catch. Previously, [`Async`] allowed for easy `&mut` access to the inner I/O resource. However, this isn't allowed now, as it is possible to move out and then drop the underlying I/O resource while it is still registered into the reactor, which voids the entire point. Therefore mutable access through methods like `get_mut`, `readable_with_mut` or `writable_with_mut` are now `unsafe`, with the condition that the user is not allowed to move out or drop the underlying file descriptor. 

Unfortunately this change is very harsh and will likely break a lot of code in the wild. I've seen many real-world use cases use `readable_with_mut` for efficiently polling some I/O resource. Not to mention, the only real fix that can be done while still ensuring I/O safety holds is to use interior mutability, which is a code smell.

In any case, it is recommended for users of [`async-io`], especially ones that implement `async` wrappers around I/O resources like [`inotify`], to expect this change and start thinking about how the architecture should be changed.

See [this pull request for `polling`](https://github.com/smol-rs/polling/pull/123) for more information on why this was done and how it was considered.

## `async-channel` and `async-lock` !Unpin Futures

Previously, [`async-channel`] and [`async-lock`] (aka `smol::channel` and `smol::lock`) had several methods that returned `Unpin` futures. These futures can be polled without pinning them to the stack or heap. Once [`smol`] v2.0 releases, these futures will now be `!Unpin`.

I expect this to have a low impact on your average bear, as most of the time these futures are immediately `.await`ed, which doesn't care about `Unpin` at all. However I am aware of some use cases that rely on the `Unpin`-ness of these futures. Fortunately this breakage can be ameliorated in the short term by just `Box::pin`ning the future and polling it from there.

The reason for this change is that it allows the `async` primitives underlying these libraries to be implemented in a much more efficient way. These libraries work by storing waiters- tasks waiting for the channel to receive a value or the [`Mutex`] to unlock- in a linked list. Previously, this linked list used a strategy where every node in the list was heap allocated. Now, these nodes use stack storage when possible, which avoids allocations on the heap.

This optimization is a massive win for performance, doubling it in some cases. However, this strategy requires that the futures be `!Unpin`, as the stack storage cannot be moved without invalidating every other node in the linked list.

## `async-channel` and `async-lock` on `no_std`

Previously, [`async-channel`] and [`async-lock`] were only available for platforms with `std` enabled. This is because they internally made use of [`std::sync::Mutex`] for synchronization. Once [`smol`] v2.0 released, these crates can now be used on `no_std` platforms.

All features from these crates should still be available on `no_std` platform, except for blocking methods (like [`send_blocking`](https://docs.rs/async-channel/latest/async_channel/struct.Sender.html#method.send_blocking)). The main drawback is that both of these crates still require a global allocator, which probably can't be removed until [`allocator_api`] is stabilized. So unfortunately it probably won't be used on embedded systems any time soon.

Once again this win comes from the underlying event notification primitive. The previous implementation used an [`std::sync::Mutex`] to synchronize the state of the inner linked list of wakers. Although the `std` implementation still uses this mutex, `no_std` uses another strategy. The state is protected by a spinlock, but put away your [blogpost links](https://matklad.github.io/2020/01/02/spinlocks-considered-harmful.html) for now. Only a limited amount of spinning is allowed, and it falls back to a fully atomic queue after the spinlock fails too many times. This strategy was based on analysis that the spinlock is the most efficient strategy for low contention, and the atomic queue can act as a fallback for when the contention is high.

However, the `no_std` strategy is significantly slower than the `std` strategy. Therefore users should use the `std` strategy whenever possible. Users of these crates should consider exposing an `std` default feature in turn that allows for users to opt-out of using `std` for `no_std` applications.

Although this win and the previous one were developed over a long series of pull requests, [this one](https://github.com/smol-rs/event-listener/pull/51) is the most relevant.

## `futures-lite` now re-exports `ready` and `pending` from libstd

When [`futures-lite`] was originally written, [`ready`] and [`pending`] were not available on `libstd` yet. However, as of Rust v1.63, they can be imported from `core::future`. Therefore, [`futures-lite`] will now re-export these functions from `core::future` instead of defining them itself.

This is a small but significant win, as it represents less functionality needed on the part of [`futures-lite`]. However, our v1.63 MSRV stops us from re-exporting [`poll_fn`] as well, which was introduced in v1.64.

See [this PR](https://github.com/smol-rs/futures-lite/pull/73) for more information.

## ESP-IDF Support

Thanks to [Ivan Markov](https://github.com/ivmarkov), the underlying machinery of [`smol`] now has better support for ESP-IDF. This is a big win for the embedded community, as it allows for [`smol`] to be used on embedded systems that use ESP-IDF.

## New Logo

We now have a new logo! The old [`smol`] logo was a smol, cute orange tabby cat. To represent the new capabilities that [`smol`] has for version 2.0, we've decided to change the logo into a cyborg cat!

![new logo](https://github.com/smol-rs/smol/blob/master/assets/images/logo_fullsize.png)

Thanks to [NebulousStar](https://twitter.com/star_nebulous) on X for drawing this!

## ...and much more!

There are a handful of other minor, non-breaking changes as well. This includes the task system now having a metadata slot available, and the [`async-executor`] crate using thread-local queue optimizations. These changes are not as important as the ones listed above, but they are still worth mentioning.

I'll have a more complete changelog once the release actually takes out. Until then, you would be well advised to start thinking about how these changes will affect your code.

[`smol`]: https://crates.io/crates/smol
[Rust v1.63]: https://blog.rust-lang.org/2022/08/11/Rust-1.63.0.html
[I/O safety]: https://blog.rust-lang.org/2022/08/11/Rust-1.63.0.html#rust-ownership-for-raw-file-descriptorshandles-io-safety
[`read`]: https://man7.org/linux/man-pages/man2/read.2.html
[global reactor]: https://crates.io/crates/async-io
[`Async`]: https://docs.rs/async-io/latest/async_io/struct.Async.html
[`AsFd`]: https://doc.rust-lang.org/std/os/fd/trait.AsFd.html
[`AsRawFd`]: https://doc.rust-lang.org/std/os/fd/trait.AsRawFd.html
[`async-io`]: https://crates.io/crates/async-io
[`inotify`]: https://en.wikipedia.org/wiki/Inotify
[`async-channel`]: https://crates.io/crates/async-channel
[`async-lock`]: https://crates.io/crates/async-lock
[`Mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[`std::sync::Mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[`allocator_api`]: https://doc.rust-lang.org/unstable-book/library-features/allocator-api.html
[`futures-lite`]: https://crates.io/crates/futures-lite
[`ready`]: https://doc.rust-lang.org/std/future/fn.ready.html
[`pending`]: https://doc.rust-lang.org/std/future/fn.pending.html
[`poll_fn`]: https://doc.rust-lang.org/std/future/fn.poll_fn.html
[`async-executor`]: https://crates.io/crates/async-executor
