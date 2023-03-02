---
layout: post
title:  \Device\Afd, or, the Deal with the Devil that makes async Rust work on Windows
categories: [async, rust, windows]
excerpt: The monster in the closet that no one wants to make eye contact.
comments: true
---

I'm feeling awfully Satanic today. I just played a fun game of Dungeons and Dragons, and I'm on my way home to enjoy a nice session of DOOM. But you know, what's more Satanic than a deal with the devil? Specifically, a deal with one of the most famous, gruesome, misanthropic devils in history. That's right, today we're talking about Microsoft.

It's no secret that Windows' networking code is basically just duct taped to the system. It combines the general over-simplification of the Berkeley socket APIs that Unix programmers know and despise, with the weird enterprise-y API decisions that characterize Windows. I remember that my first real attempt at programming was writing an internet game on my Windows PC. I stubbed my toe on that WinSock API more times than I'd care to admit.

However, there are some important features that WinSock just doesn't expose. Sometimes, you just have to blaze your own trail. Nuts to program stability, we're going to dig our hands deep into Windows' guts and see what we can find. Maybe you'll find some treasure... or something so cursed it'd make Indiana Jones hesitate.

Rust's current async ecosystem is built atop a particularly cursed concept. It's an unstable, undocumented Windows feature. It's the lynchpin of not only the Rust ecosystem, but the JavaScript one as well. It's controversial. It's efficient. When you Google it, the first result is asking if it's a virus. Without it, it's unlikely that the async ecosystem would exist in its current form. It's called `\Device\Afd`, and I'm tired of no one talking about it.

## Background

The [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait looks like this:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

The idea is that, if you have a `Future`, you call `poll` on it. If it's ready, great, you get the value that you want. If it isn't, it'll take the `Waker` in the `Context` (which is a [glorified callback](https://docs.rs/waker-fn/latest/waker_fn/)) and says "hey, I'll call you back when I'm ready." Or, alternatively, you just call `.await`, which abstracts over all of this nicely.

However, the question is, *what* calls the `Waker`, when the `Future` is inevitably ready. It depends on the `Future`. Maybe, it's another thread that's actually handling the I/O. This is what happens for [blocking thread pools](https://crates.io/crates/blocking). It takes whatever the operation is, stuffs it in another thread, and then calls the `Waker` when it's done. In some cases, like for files, this is the only option. 

But, when it comes to network sockets, we can do better. If you have millions of sockets (as is expected of many web servers), that means millions of threads. Not only does that incur a lot of overhead, but it ends up being a nightmare of context switching and competition for resources. It scales about as well as a refrigeration-free grocery store.

This is why the `poll` system call exists. The API varies between Windows and Unix, but it looks roughly like this:

```c
typedef struct {
    int fd;
    int events;
    int revents;
} pollfd;

int poll(pollfd *fds, unsigned long nfds, int timeout);
```

For every socket you need to keep track of, you create a `pollfd` struct, where the `events` field contains the events you want to keep track of. Then, when you try to run an operation, it returns one of two things:

1). It returns immediately, which means the operation completed. Great!
2). It tells you that it's not ready yet. This means that you need to wait for the socket to be ready. This is where `poll` comes in; it will block until *any* one of the sockets is ready.

Here's how it would work, in Rust form, using the [`nix`](https://crates.io/crates/nix) crate:

```rust
use std::io;
use std::net::TcpStream;
use nix::poll::{poll, PollFd, PollFlags};

// Test to see if a socket is ready.
fn is_ready(sock: &TcpStream) -> bool {
    let mut data = [0u8; 1]
    match sock.read(&mut data) {
        Ok(_) => {
            // The socket operation went through!
            true
        },

        Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
            // The socket operation returns WouldBlock, which means it's not ready yet.
            false
        },

        Err(e) => {
            // Some other error happened.
            panic!("Error: {:?}", e);
        }
    }
}

// Create two sockets.
let sock1: TcpStream = /* ... */;
let sock2: TcpStream = /* ... */;

// Set them both to non-blocking mode.
sock1.set_nonblocking(true).unwrap();
sock2.set_nonblocking(true).unwrap();

loop {
    // Create a pollfd for each socket.
    // POLLIN means "wait for data to be ready to read."
    let mut fds = [
        PollFd::new(sock1.as_raw_fd(), PollFlags::POLLIN),
        PollFd::new(sock2.as_raw_fd(), PollFlags::POLLIN),
    ];

    // Wait for either socket to be ready.
    poll(&mut fds, -1).unwrap();

    // Check if either socket is ready.
    if is_ready(&sock1) {
        // Do something with sock1.
    }

    if is_ready(&sock2) {
        // Do something with sock2.
    }
}
```

The problem is that `poll` is inefficient for large numbers of sockets. The `poll` system call needs to do a linear scan over every socket passed to the function, which is `O(n)`. This problem means that, if you have a million sockets, it needs to do a million operations. The scalability of this is terrible.

Of course, operating systems have introduced more scalable ways of polling sockets. Linux and Android use [`epoll`](https://en.wikipedia.org/wiki/Epoll) and BSD uses [`kqueue`](https://en.wikipedia.org/wiki/Kqueue). These are both `O(1)` operations, which means that they scale much better. They're used in almost every Rust async runtime. 

To massively oversimplify async-I/O reactors, the system looks like this:

1). After creating a system resource (say, a [`TcpStream`](https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html)), you add it to the polling system. This would involve [`EPOLL_CTL_ADD`](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html) on Linux.

2). You run your future by calling `poll()` on it. The `Future` hopefully takes that `TcpStream` and tries reading some data from it. If it's not ready, it takes the `Waker` and puts it in a slot that is uniquely associated with that `TcpStream`.

3). Eventually, that `Future` returns `Poll::Pending`, which means it's time to start waiting. The runtime calls [`epoll_wait`](https://man7.org/linux/man-pages/man2/epoll_wait.2.html) and waits for incoming readiness events.

4). The `TcpStream` receives new data, and the `epoll_wait` (or whatever) call indicates that the stream is ready. The runtime then wakes up the `Waker` associated with that `TcpStream`, which in turn wakes up the `Future` that was waiting on it.

5). The `Future` then tries to read the data again, and this time it succeeds. Continue from step 2.

It's a great system that works on Linux, BSD, and [sometimes Solaris](https://illumos.org/man/3C/port_create). So, when it comes to Windows, how does it go so wrong?

## Windows is different, as always

The problem lies in the fact that Windows doesn't have an equivalent polling API like `epoll` or `kqueue`. Yes, it has [`WSAPoll`](https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsapoll), which *could* work, if you're fine with `O(n)` time. However, there's no way to wake up `WSAPoll` without a socket doing an I/O operation. While sockets are a big part of Rust's async story, there are also threadpools and [event listeners](https://crates.io/crates/event-listener) and other things that use `Future`s. It's important, for this reason, to be able to wake up from a polling operation without any I/O. Unix solves this by having [self-pipes](https://www.man7.org/linux/man-pages/man2/pipe.2.html) that can be used to wake up operations, but Windows doesn't have a socket-based equivalent.

Windows, however, does have [input/output completion ports](https://docs.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports), or IOCP. Now, on its own, IOCP's are actually really great! I'm not going to complain, they're actually a really well designed API. They not only support sockets, but also files, processes, and many other assorted Windows entities. However, its notification system is different than `epoll`, `kqueue` or anything else. It's *completion*-based, rather not *polling*-based. Instead of waiting for the socket to be ready, it waits for operations on the socket to complete.

There's been a lot of ink spilled about this already, but the overarching idea is that IOCP doesn't gel well with not only the `Future` model, but also with Rust's borrow checker as a whole. To make a long story that I don't fully understand short, you need to have buffers that are pinned in memory and aren't touched as long as the operation is in progress. The other issue comes down to needing to cancel the operation on `Drop` to ensure soundnss. This gets about a thousand time more complicated once `std::mem::forget` and memory leakage is involved. The rabbit hole is deep, twisted, and there's probably a spike pit at the bottom. For more information, Linux has a secondary polling API named [`io_uring`](https://en.wikipedia.org/wiki/Io_uring) that's also completed based. The difference between it and the regular polling API is so large that you need an [entire new async API](https://docs.rs/tokio-uring/latest/tokio_uring/) to properly support it.

## The Solution

The overarching idea is, what if that the operation that the IOCP is waiting on *is* waiting for the socket to become ready? This would handily solve our problem here, since we could just register that waiting operation to adapt it to the polling model. Sadly, the WinSock API doesn't expose a way of doing this, aside from `WSAPoll`, which we've already established doesn't work.

However, Windows is like an onion with foliar disease; it has a dozen layers, and every layer only gets more disgusting. You see, Windows *has* a polling API. You just have to work for it.

You see, WinSock is a just a layer on top of another API; the **Auxillary Function Driver**, or AFD. AFD is the underlying system that handles the work of the TCP/IP stack. If you're dissatisfied with the middleman that is WinSock, you can talk to AFD directly. It's just a bit convoluted, and I'll save the technical details for another article.

However, the process for talking to AFD goes as follows:

1). Gain access to the barely-documented Windows internal functions. You know, the ones that start with `Nt*` and `Rtl*`. If regular Windows APIs are like the nice, well-kempt students, these guys are the bad boys that your crush always liked better. Microsoft has [this ominous warning](https://learn.microsoft.com/en-us/windows/win32/devnotes/calling-internal-apis) about using them. You need to dynamically link to `ntdll.dll` just to get access to them.

2). Using the arcane runes you've just unlocked, it's now time to begin the summoning ritual. You create a new handle in the `\Device\Afd` namespace to get access to the AFD subsystem. `OpenFile` is too fragile and pure for this, so you need to use [`NtCreateFile`](https://learn.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntcreatefile) to bind the demon to our plane of existence.

3). Now, you need to sign a contract with the demon. You use an [I/O Control](https://learn.microsoft.com/en-us/windows/win32/devio/device-input-and-output-control-ioctl-) to tell the AFD device what to do. First, you need to get the ~~true name~~ [base handle](https://learn.microsoft.com/en-us/windows/win32/winsock/winsock-ioctls#sio_base_handle-opcode-setting-o-t1) of the socket. Then, you combine that with an `IOCTL_AFD_POLL` operation to get the polling operation.

Once you have this, the rest is relatively simple. You can register the AFD handle in an I/O completion port so that, when the waiting operation completes, it's received by the end user. When you want to listen for readiness events on a socket, you send an `IOCTL_AFD_POLL` operation. During your "wait" operation, you wait on the completion port and deal with the results as they come in. It's simple to implement a polling interface on top of this, and from there you can build a `Future`-based system and run wild to your heart's content.

## Demons under the bed

I intended for this to be a high-level overview of why we use `\Device\Afd`, mostly so people are more aware of it. That being said, one may be rightfully worried about having unstable APIs like this in the stack. Theoretically, there's a danger that Microsoft could yank the metaphorical rug out from under us and leave the `async` community stranded.

I doubt this will happen; enough libraries use `\Device\Afd` that I don't think that Microsoft would just up and get rid of it. In Rust terms, it's used by [`mio`](https://crates.io/crates/mio), which is the backing async I/O library for [`tokio`](https://crates.io/crates/tokio), among [other](https://github.com/alacritty/alacritty/blob/4ddb608563d985060d69594d1004550a680ae3bd/alacritty_terminal/Cargo.toml#L27-L28) [things](https://crates.io/crates/winit). It's also used by [`wepoll`](https://github.com/piscisaureus/wepoll), which is used as a replacement for `epoll` in libraries like `zeromq`. `wepoll` also backs [`polling`](https://crates.io/crates/polling), which is what's used by [`smol`](https://crates.io/crates/smol) and [`async-std`](https://crates.io/crates/async-std). Although, I'm currently working on porting `polling`'s Windows backend to use [a homegrown implementation](https://github.com/smol-rs/polling/pull/88) of this AFD trick.

Oh yeah, `\Device\Afd` is also used by [`libuv`](https://github.com/libuv/libuv), which powers [Node.js](https://github.com/nodejs/node). I doubt that Microsoft is really rushing to make it so that you can't run Javascript on Windows anymore.

There *are* other issues with using unstable, undocumented APIs like this, however. The premier issue is that, for a long time, `\Device\Afd` wasn't implemented on Wine. To this day, `mio` doesn't support Wine because of it (although [that may change soon](https://github.com/tokio-rs/mio/pull/1637)). In addition, it may be problematic for UWP apps to use undocumented APIs like this. I haven't *read* anything about this being a problem *yet*, but it's a skeleton in the closet to keep in mind. In fact, it may be a good idea to have an alternate implementation of polling on hand, using threadpooling or something, just in case.

## Et al

There's still so much more to talk about when it comes to `\Device\Afd`, but I'm worried that this article would go on forever if I tried to cover even the high-level bits. I just figured that this is something that users of the Rust ecosystem should be aware of, since it's a part of the async stack that's not really talked about, and also one that *could* break your project if you're not careful. Stay frosty out there.

