---
layout: post
title: A Proposal for Safe Window Handles
categories: [gui, rust]
excerpt: This year is supposed to be the year of the Rust GUI. So why is it still so unsafe?
comments: true
---

This year is supposed to be the year of the Rust GUI. So why is it still so unsafe?

If we're ever going to make Rust a language that can be used to develop desktop apps, we need to make GUI less unsafe to use. In all fairness, we've done a relatively good job of it so far. We have [`winit`] for setting up windows; while [`winit`] lacks a few features that are necessary for first-class GUI systems, it's [easy enough to implement them ourselves](https://crates.io/crates/menubar). There's [`wgpu`] for GPU-accelerated drawing and [`softbuffer`] for a software-rendered way around that headache. All of these crates provide safe APIs... unless they're used together, that is.

Today, I'd like to change that.

## What's the problem?

Ideally, you would be able to use any windowing crate ([`winit`], [`miniquad`], [`druid-shell`]) with any rendering crate ([`wgpu`], [`softbuffer`], [`glutin`]). That way, [`wgpu`] doesn't have to pull in all of [`winit`] to function properly, and it can also work with [`druid-shell`] if it wanted to.

As of now, the primary crate for this compatability in the ecosystem is [`raw-window-handle`]. It provides a [`RawWindowHandle`] enum that contains a library-independent description of most types of windows. In addition, it also has a [`HasRawWindowHandle`] trait, which says "this object is a valid window". The idea is you can ask the windowing system for a [`RawWindowHandle`], and then tell the GPU API to render to that window. This setup is analagous to the [`RawFd`] and [`AsRawFd`] items in the standard library; if you need to pass something that uses a file descriptor to something else, you can just plug in an [`AsRawFd`] implementor. This is used in the ecosystem, for example, by [`async-io`], where it can drive most types that implement [`AsRawFd`] as [`AsyncRead`]/[`AsyncWrite`].

The issue lies in the fact that the handles returned by [`HasRawWindowHandle`] are not durable. The window handle may be valid for the immediate future (however long that is in a multithreaded world), but after that it isn't valid anymore. There's also another issue; on Android, that window handle can be rendered invalid at any time, and there's (almost) nothing you can do about it. Because of these two reasons, the windowing APIs that take [`HasRawWindowHandle`] need to be `unsafe`.

There are also some minor API quibbles, like how [`RawWindowHandle`] can't implement [`HasRawWindowHandle`], since [`RawWindowHandle`] can be constructed in safe code and isn't guaranteed to be valid. 

## Borrowed Handles

For now, let's assume that window handles can't just be invalidated by some external outside force.

I mentioned [`RawFd`]/[`AsRawFd`] above. The standard library has since realized that the whole "raw FD" system is a bad idea, since anyone can just create a random FD and submit that to be messed around with by any system call they want. This might not seem like an issue until you realize that "random FD" could be "resource that the standard library expects to still be in a valid state". This led to the launching of an "I/O safety" initiative. Rather than [`RawFd`]/[`AsRawFd`], you now have [`BorrowedFd`]/[`AsFd`]. Unlike [`RawFd`], [`BorrowedFd`] is *guaranteed* by the safety contract to be a valid, non-closed file descriptor. It is also impossible to close it/put it into an invalid state without other unsafe code, meaning that you can depend on a [`BorrowedFd`] to still be there, no matter what.

I figured that this was a good system, and my first implementation of my "safe window handle" idea looked like this:

```rust
struct WindowHandle<'a> {
    raw: RawWindowHandle,
    lifetime: PhantomData<'a>
}

impl WindowHandle<'_> {
    // SAFETY: `raw` must be valid for the provided lifetime.
    unsafe fn borrow_raw(raw: RawWindowHandle) -> Self {
        Self {
            raw,
            lifetime: PhantomData
        }
    }
}
```

This structure provides a good summary of what I wanted; it makes sure that the window handle is valid for the specified lifetime by making it a part of the safety contract. This system *seems* to work, at a glance. But, naturally, the devil's in the details.

## Window Handles are not FDs

File descriptors are local to the current process. Even if you send them to another process (through something like the [`sendmsg`] system call), it just adds a reference to the reference counter of the underlying system resource. [`close`]-ing that cloned file descriptor doesn't affect the resource beyond decrementing the counter.

On the other hand, it is expected that window handles can be closed fully from other processes. On X11 on Linux, any process can send a `CloseWindow` request to any window. It doesn't even have to be another process. In fact, since X11 is a network protocol, *another machine* can close your window if it wants to. There's a similar situation on Windows; with the right function, anybody can just walk in, take your `HWND`, and then close it. It's like anarchy over there.

The good news is that these are just window IDs instead of window pointers that we dereference. Yes, even on Windows, the `HWND` is less of a "handle" and more of an index into a thread-local table. This means that, technically, it's not unsound to call a function with an invalid window ID; it's annoying and it'll return an error state, sure, but it's not `unsafe`. So, we just have to say that those window IDs might be invalid in rare cases. There's nothing we can really do about that.

Also, for the Wayland geeks: this is a problem that Wayland solves. You can't randomly drop a window without destroying the entire protocol connection.

## Paindroid

However, there *is* one case where you have a *real life pointer* to a system resource, that can be invalidated at any point in time. And of course it's related to Java.

On Android, there is an [Activity]-global [`ANativeWindow`] object that is used for drawing. This
handle is used [within the `RawWindowHandle` type] for Android NDK, since it is necessary for GFX
APIs to draw to the screen.

However, the [`ANativeWindow`] type can be arbitrarily invalidated by the underlying Android runtime.
The reasoning for this is complicated, but this idea is exposed to native code through the
[`onNativeWindowCreated`] and [`onNativeWindowDestroyed`] callbacks. To save you a click, the
conditions associated with these callbacks are:

- [`onNativeWindowCreated`] provides a valid [`ANativeWindow`] pointer that can be used for drawing.
- [`onNativeWindowDestroyed`] indicates that the previous [`ANativeWindow`] pointer is no longer
  valid. The documentation clarifies that, *once the function returns*, the [`ANativeWindow`] pointer
  can no longer be used for drawing without resulting in undefined behavior.

In [`winit`], these are exposed via the [`Resumed`] and [`Suspended`] events, respectively. Therefore,
between the last [`Suspended`] event and the next [`Resumed`] event, it is undefined behavior to use
the raw window handle. This condition makes it tricky to define an API that safely wraps the raw
window handles, since an existing window handle can be made invalid at any time.

My solution to this problem was to set up an "active handle" type, such that the new definition of `WindowHandle` looks like this:

```rust
struct WindowHandle<'a> {
    raw: RawWindowHandle,
    active: ActiveHandle<'a>
}

struct ActiveHandle<'a> {
    #[cfg(target_os = "android")]
    guard: Option<RwLockReadGuard<'a, bool>>,
}

struct Active {
    #[cfg(target_os = "android")]
    lock: RwLock<bool>
}

trait HasWindowHandle {
    fn window_handle(&self) -> Result<WindowHandle<'_>, InactiveError>;
}
```

The idea here is that an `ActiveHandle` signifies that a window handle is currently being used, so that the windowing system can be aware that there are currently window handles and that it needs to wait for those handles to be dropped before it leaves the [`onNativeWindowDestroyed`] callback.

What does the workflow here look like? First, you get a window handle from your windowing system.

```rust
use raw_window_handle::HasWindowHandle;
use winit::window::Window;

let window: Window = /* ... */;
let handle = window.window_handle().expect("window is inactive");
```

The `window_handle()` function can return an error, since it's possible that the application has been `Suspended` and it is impossible to actually use the window handle in this case. Actually, it might make more sense if we drill into how `HasWindowHandle` is implemented for `Window`.

```rust
use raw_window_handle::{Active, HasWindowHandle, InactiveError, WindowHandle};

pub struct Window {
    // snip: platform specific internals

    active: Active,
}

impl HasWindowHandle for Window {
    fn window_handle(&self) -> Result<WindowHandle<'_>, InactiveError> {
        match self.active.handle() {
            Some(active_handle) => {
                // SAFETY: Our raw handle is valid, and the application isn't suspended.
                Ok(unsafe { WindowHandle::borrow_raw(self.raw_window_handle(), active_handle) })
            }

            None => {
                Err(InactiveError)
            }
        }
    } 
}

// Meanwhile, in the event handling code...
fn handle_event(event: Event, window: &Window) {
    match event {
        Event::Resumed => {
            self.active.set_active();
            handle_resumed_event();
        }

        Event::Suspended => {
            handle_suspended_event();
            self.active.set_inactive();
        }

        other_event => /* ... */
    }
}
```

Some points of note here:

- `self.active.handle()` acquires the read lock on the inner `RwLock` in the `Active`. This tells the application that there is an active window handle. However, the application might be suspended, as told by if the `bool` that the `RwLock` protects is `false`. In this case, `handle()` returns `None`, and the application bubbles that up to the user in the form of `InactiveError`.
- When the windowing system gets `Event::Resumed`, it calls `set_active()`, which sets the boolean inside of the `RwLock` to be `true`. This allows new `WindowHandle`s to be created again.
- When the windowing system gets `Event::Suspended`, it calls `set_inactive()`, which sets that boolean to `false`. Since this requires a write lock on the inner `RwLock`, it will block until all of the `ActiveHandle`s/`WindowHandle`s have been released.

The third point here is the main flaw of this system, in my opinion. It is very possible for a deadlock to occur if there is an outstanding `WindowHandle` on the main thread. Because of this, it's probably better to pass around the object that implements `HasWindowHandle` instead of the `WindowHandle` itself, in order to make sure that you're not stepping on the main loop's toes.

On the other hand, `DisplayHandle` can't be arbitrarily invalidated, so it doesn't need an `ActiveHandle`. It just uses a raw pointer and a lifetime, like the original system.

Another item of note here is that all of this complexity is `target_os = "android"` only. Therefore, it all compiles down to no-ops on systems where this silliness isn't necessary. In fact, I have an additional method:

```rust
#[cfg(not(target_os = "android"))]
impl ActiveHandle<'_> {
    pub fn new() -> Self { /* ... */ }
}
```

...that allows users to just create `ActiveHandle`s out of thin air if they don't care about supporting Android. Yay!

## Integration

Now that we have a safe API, the next step is to implement it for all of the popular crates. Thus far, I have draft PRs for:

- [`raw_window_handle`](rwhpr)
- [`winit`](winitpr)
- [`softbuffer`](sbpr)
- [`glutin`](glutinpr) (my first experience with GATs 0_0)

I'd also like to write a PR for [`wgpu`], but that codebase is huge, so I need to mentally prepare myself first. Between all of the GPU backends it supports, there's probably a lot of different use cases to keep in mind.

If you own a crate that uses [`raw-window-handle`], please comment on [this issue for safer window handles](refining). My goal here is to get a good idea of what the ecosystem will look like after borrowed window handles are available. I want this to be the kind of thing we don't need to break in the future for semver.

## What's Next?

After I finish modeling what the ecosystem will look like, I'll need to test everything. Testing will need to happen especially on Android, but it's best to test all of the other platforms as well, just to make sure that there's not some invisible line we're crossing. After that, I'll start trying to get these PRs merged. Afterwards, we'll have a safer, sounder Rust GUI ecosystem.

That being said, before I start wading into that mess, I'd like some comments on this proposal. Does this all seem not only safe and sound, but also usable? I'd also like to make sure to keep all platforms in mind; if you are the maintainer for a platform that you'd like to eventually worm your way into [`raw-window-handle`], please comment on [this issue](refining) if your platform would have any special handle requirements.

Together, we can make ~~2015~~ ~~2020~~ 2023 the year that Rust is GUI yet.

[Activity]: https://developer.android.com/reference/android/app/Activity
[`ANativeWindow`]: https://developer.android.com/ndk/reference/group/a-native-window
[within the `RawWindowHandle` type]: struct.AndroidNdkWindowHandle.html#structfield.a_native_window
[`onNativeWindowCreated`]: https://developer.android.com/ndk/reference/struct/a-native-activity-callbacks#onnativewindowcreated
[`onNativeWindowDestroyed`]: https://developer.android.com/ndk/reference/struct/a-native-activity-callbacks#onnativewindowdestroyed
[`Resumed`]: https://docs.rs/winit/latest/winit/event/enum.Event.html#variant.Resumed
[`Suspended`]: https://docs.rs/winit/latest/winit/event/enum.Event.html#variant.Suspended
[`sdl2`]: https://crates.io/crates/sdl2
[`RawWindowHandle`]: https://docs.rs/raw-window-handle/latest/raw_window_handle/enum.RawWindowHandle.html
[`HasRawWindowHandle`]: https://docs.rs/raw-window-handle/latest/raw_window_handle/trait.HasRawWindowHandle.html
[`winit`]: https://crates.io/crates/winit
[`wgpu`]: https://crates.io/crates/wgpu
[`softbuffer`]: https://crates.io/crates/softbuffer
[`miniquad`]: https://crates.io/crates/miniquad
[`druid-shell`]: https://crates.io/crates/druid-shell
[`raw-window-handle`]: https://crates.io/crates/raw-window-handle
[`async-io`]: https://crates.io/crates/async-io
[`glutin`]: https://crates.io/crates/glutin
[`RawFd`]: https://doc.rust-lang.org/std/os/fd/type.RawFd.html
[`AsRawFd`]: https://doc.rust-lang.org/std/os/fd/trait.AsRawFd.html
[`AsyncRead`]: https://docs.rs/futures-io/latest/futures_io/trait.AsyncRead.html
[`AsyncWrite`]: https://docs.rs/futures-io/latest/futures_io/trait.AsyncWrite.html
[`BorrowedFd`]: https://doc.rust-lang.org/stable/std/os/fd/struct.BorrowedFd.html
[`AsFd`]: https://doc.rust-lang.org/stable/std/os/fd/trait.AsFd.html
[`sendmsg`]: https://linux.die.net/man/2/sendmsg
[`close`]: https://linux.die.net/man/2/close
[rwhpr]: https://github.com/rust-windowing/raw-window-handle/pull/116
[winitpr]: https://github.com/rust-windowing/winit/pull/2744
[sbpr]: https://github.com/rust-windowing/softbuffer/pull/82
[glutinpr]: https://github.com/rust-windowing/glutin/pull/1582
[refining]: https://github.com/rust-windowing/raw-window-handle/issues/111
