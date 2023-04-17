---
layout: post
title: Announcing async-winit
categories: [gui, rust]
excerpt: Event handling finally done right
comments: true
---

Here's something that I've been working on that, as far as I know, is unique in the GUI space. Statistics tells me that it's because my new model is either innovative or wrong. Let's see which one it is.

I've spent the past week or so creating the [`async-winit`] crate. It takes the [`winit`] interface and makes it asynchronous. The Cliff Notes version is:

- You don't call `EventLoop::run`, you call `EventLoop::block_on`.
- You don't wait for an event by waiting for it to show up in the event loop, you just `await` it.
- You can stop thinking about your GUI program as an event loop and more like the sequential program that it should be.

## Background 

For those who haven't spent the past while neck-deep in the GUI ecosystem like I have, here's a little background information.

[`winit`] is a cross-platform windowing library for Rust. What that means it that it handles setting up the window frames and window events on the system. For instance, you would ask for a window, and it sets up a rectangular region on your screen. Then, you provide a callback that gets events for that window (as well as for any other windows you happen to set up). [`winit`] intentionally limits itself in scope to just windowing and events; it doesn't handle things like setting up menu bars or drawing to the window. That's left to other libraries.

[`winit`] has also become the de-facto windowing library for Rust, being used in showstopping GUI frameworks like [`egui`] and [`iced`]. There are alternatives, like [`miniquad`] and [`druid-shell`], but [`winit`]'s number of downloads leaves those crates in the dust.

The core of a [`winit`] program looks like this:

```rust
use winit::event::{Event, WindowEvent};
use winit::event_loop::EventLoop;

fn main2(event_loop: EventLoop<()>) {
    event_loop.run(|event, _, control_flow| {
        match event {
            Event::Resumed => { /* ... */ },
            Event::Suspened => { /* ... */ },
            Event::WindowEvent { event, window_id } => {
                let window = get_window_data_from_the_id(window_id);
                match event {
                    WindowEvent::CloseRequested => { /* ... */ },
                    WindowEvent::Resized(size) => { /* ... */ },
                    /* ... */
                    _ => {}
                }
            }
            _ => {}
        }
    });
}

#[cfg(not(target_os = "android"))]
fn main() {
    let event_loop = EventLoop::new();
    main2(event_loop);
}

#[cfg(target_os = "android")]
use winit::platform::android::{
    EventLoopBuilderExtAndroid,
    activity::AndroidApp
};

#[cfg(target_os = "android")]
#[no_mangle]
fn android_main(app: AndroidApp) {
    use winit::event_loop::EventLoopBuilder;

    main2(EventLoopBuilder::new().with_android_app(app).build());
}
```

This pattern is generally how most GUI programs are set up, not just with this framework. A callback is provided to the event loop that tells it how to handle events. This is usually combined with some kind of persistent state to keep track of windows and other events in the program. A large `match` statement is used to handle the events. The rest of the program is mostly just special casing for Android, where cross-platform GUI tends to be really hard to handle. Buy one of the [`winit`] Android maintainers a drink and I'm sure they'll tell you more.

## The Next Level

The "event loop/event handler" pattern here isn't necessarily bad. In fact, it's not even unique to GUIs. It's a common pattern everywhere. For instance, there are many programs built around the idea of a polling loop, where you register a bunch of sources into [`poll`]/[`epoll`]/[`kqueue`]/[IOCP] and then wait for one of them to be ready. When it is, you handle the event and then go back to waiting for the next one.

```rust
let mut my_sources = vec![/* ... */];
loop {
    poll(&mut my_sources);
    
    for source in my_sources {
        if source.is_ready() {
            match source.key {
                INTERNET_KEY => { /* ... */ },
                FILE_KEY => { /* ... */ },
                /* ... */
                _ => {}
            }
        }
    }
}
```

Just like the [`winit`] pattern, this loop takes in sources, waits on them, and then handles events when they occur. [`winit`] uses a callback to abstract over the many different operating systems and how they prefer to handle events, but the idea is fundamentally the same. It's just that, instead of handling window events, we're handling I/O events.

One of the places you tend to see this pattern is in web servers. You'll have a bunch of network sockets registered in the polling system, often representing clients. Then, [`poll`] gives you an event representing a client that's ready to be written to or read from. You handle the event, and then go back to waiting for the next one.

The thing is, though, for network servers, they've somewhat moved on from the polling model. It turns out keeping all of that state consistent across iterations of the event loop is a bit complicated. There's this cool new thing called `async`/`await`, which lets you write that entire pattern as a sequential program. Instead of a polling loop, with all of its state and cases, you just write a function that `await`s on the operation to finish. This automatically takes care of all of the event switching and handling for you.

```rust
// An example TCP server using the `smol` runtime.
use smol::net::TcpListener;
use smol::prelude::*;
use smol::Executor;

let executor = Executor::new();
let listener = TcpListener::bind("0.0.0.0:8080").await?;
executor.run({
    listener.incoming().for_each(|socket| executor.spawn(async move {
        let mut name = String::new();
        socket.read_to_string(&mut name).await?;
        
        let response = format!("Hello, {}!", name);
        socket.write_all(response.as_bytes()).await?;
    }).detach())
}).await;
```

That's it; no need to handle readability and writability. That's all taken care of for you as part of `read_to_string` and `write_all`. On top of that, the executor automatically handles switching between ongoing connections. You don't have to worry about which connections are read-ready or write-ready; you just write your program as if it were sequential. Just spawn tasks instead of threads, and you're as fresh as a squeezed lime.

There's a handful of other bonuses too; for instance, you can more easily integrate files and other, non-socket event sources into `async`/`await` too. There's also a bunch of concurrency primitives available in the `async` ecosystem that make writing `async`/`await` easier, especially for large programs.

So, I took a look at the [`winit`] model with the event loop, another look at how `async`/`await` builds a layer over event loops, and then got an awful, terrible idea to create a crate about it.

## The Sales Pitch

Here's a quick program in [`async-winit`] that just creates a window.

```rust
use async_winit::event_loop::EventLoop;
use async_winit::window::Window;

// snip: all of the setup code above to get an EventLoop

let target = event_loop.window_target().clone();

event_loop.block_on(async move {
    // Create a window.
    let window = Window::new().await.unwrap();

    // Wait for the window to close.
    window.close_requested().await;

    // Exit the program.
    target.exit().await
});
```

Compare that against the equivalent [`winit`] program:

```rust
use winit::event::{Event, WindowEvent};
use winit::event_loop::EventLoop;
use winit::window::Window;

// snip: all of the setup code above to get an EventLoop

let window = Window::new(&event_loop).unwrap();
event_loop.run(move |event, _, flow| {
    // If the event is a window close event, exit the program.
    match event {
        Event::WindowEvent { event: WindowEvent::CloseRequested, .. } => {
            flow.set_exit();
        },
        _ => {},
    } 
});
```

I understand that there's going to be some personal preference here, but I prefer the first version. The ordering of the program is laid out in front of you; you just create a window, wait for the close request, and then exit the program. There's no need to worry about the event loop, window state, or flow control. The [`winit`] example might have less lines, but it introduces a lot of concepts; you need to handle flow control, you need to handle window state, and you need to handle the event loop.

The two models here are pretty close; [`winit`]'s maintainers have (endearingly) called the event loop model "poor man's `async`". Under the hood, it all acts the same. Some people just find it easier to reason about an event loop instead of `async`/`await`. Personally, I'm not "some people".

## But wait, there's more!

The biggest advantage of [`async-winit`], in my opinion, is how it integrates with the rest of the `async` ecosystem seamlessly. Out of the box, [`async-winit`] works with [future combinators], [executors], and [asynchronous IO]. For instance, take the above program. Do you want to print whatever the user types to the console? Just try this:

```rust
use async_winit::event_loop::EventLoop;
use async_winit::window::Window;

use futures_lite::prelude::*;

// snip: all of the setup code above to get an EventLoop

let target = event_loop.window_target().clone();
event_loop.block_on(async {
    let window = Window::new().await.unwrap();

    // Future to wait for new keypresses.
    // The `wait_many()` function gets a `Stream` over key-presses.
    let keypresses = window.received_character().wait_many().for_each(|key| println!("{key}"));

    // Run in parallel with waiting for the window to close.
    window.close_requested().or(keypresses).await;
    target.exit().await
});
```

Have you ever wanted to do networking in a GUI program? Since [`async-winit`] is a GUI reactor and not an I/O reactor, it doesn't handle networking for you. Thankfully, [`async-net`] just spawns a thread with an I/O reactor if it can't find one.

```rust
use async_winit::event_loop::EventLoop;
use async_winit::window::Window;

use async_net::TcpStream;
use futures_lite::prelude::*;

// snip: all of the setup code above to get an EventLoop

const HTTP_REQUEST: &[u8] = b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n";

let target = event_loop.window_target().clone();
event_loop.block_on(async {
    let window = Window::new().await.unwrap();

    // A future to do some networking.
    let networking = async {
        let mut stream = TcpStream::connect("example.com:80").await.unwrap();
        stream.write_all(HTTP_REQUEST).await.unwrap();

        let mut response = vec![];
        stream.read_to_end(&mut response).await.unwrap();

        println!("Got {} bytes", response.len());
    };

    // Run in parallel with waiting for the window to close.
    window.close_requested().zip(networking).await;
    target.exit().await
});
```

I'd also like to point out, `Window::new()` takes no parameters here. It might seem like a little thing, but in [`winit`] you would need to pass around a window target object from the event loop to create a window. This means that windows can be created and polled from anywhere. It might be a small thing, but I think it'll make big difference from a GUI perspective in how widgets are created and managed.

[`async-winit`] also has a [`Timer`] type that you can use to create a future that resolves after a certain period of time. This is useful for creating animations, or just having timeouts.

```rust
use async_winit::event_loop::EventLoop;
use async_winit::window::Window;
use async_winit::Timer;

use futures_lite::prelude::*;

// snip: all of the setup code above to get an EventLoop

let target = event_loop.window_target().clone();
event_loop.block_on(async {
    let window = Window::new().await.unwrap();

    // A future to wait for 5 seconds.
    let timer = Timer::after(std::time::Duration::from_secs(5));

    // Either wait for 5 seconds or close the window after.
    window.close_requested().or(timer).await;
    target.exit().await
});
```

If you have a lot of futures, you need something to coordinate them. While [`async-winit`] doesn't come with an executor, it's easy enough to integrate one into it. For instance, [`async-executor`] is a simple executor that can be used with [`async-winit`]. It can be used to handle many events at once, or to handle events in parallel.

```rust
use async_winit::event_loop::EventLoop;
use async_winit::window::Window;

use async_executor::Executor;
use futures_lite::prelude::*;

use std::sync::Arc;
use std::thread;

// snip: all of the setup code above to get an EventLoop

let target = event_loop.window_target().clone();
let executor = Executor::new();
let (signal, shutdown) = async_channel::bounded::<()>(1);

// Spawn a few threads to poll tasks on.
for _ in 0..4 {
    let executor = executor.clone();
    let shutdown = shutdown.clone();
    thread::spawn(move || {
        async_io::block_on(executor.run(shutdown.recv())).ok();
    });
}

// Run the event loop.
event_loop.block_on(async move {
    let window = Window::new().await.unwrap();

    // Spawn a task to handle key presses.
    executor.spawn({
        let window = window.clone();
        async move {
            window.received_character().for_each(|key| println!("{key}")).await;
        }
    }).detach();

    // Spawn a task to handle mouse clicks.
    executor.spawn({
        let window = window.clone();
        async move {
            window.mouse_button_pressed().for_each(|input| {
                println!("{:?}", input.button);
            }).await;
        }
    }).detach();

    // Spawn a few tasks for networking.
    for _ in 0..5 {
        executor.spawn(some_networking_function).detach();
    }

    // Run all of this in parallel with waiting for the window to close.
    executor.run(window.close_requested()).await;
    drop(signal);
    target.exit().await
});
```

What if you need to run something inside of the event handler? For instance, when it comes to drawing, sometimes you can really only do it when you're inside of the event handler. Using the `wait_guard` function on the event handler, you can force part of a future to be ran inside of the event handler. Just keep in mind that holding this guard prevents further events from being dispatched.

```rust
use async_winit::event_loop::EventLoop;
use async_winit::window::Window;

use futures_lite::prelude::*;
use softbuffer::GraphicsContext;

// snip: all of the setup code above to get an EventLoop

let target = event_loop.window_target().clone();
event_loop.block_on(async move {
    let window = Window::new().await.unwrap();

    // Future that waits for redraw to be requested, and then fills the window with
    // the color red.
    let redraw = async {
        let mut waiter = window.redraw_requested().wait_guard();
        let mut sb = None;
        let mut buf = vec![];

        loop {
            // Wait for the window to be redrawn.
            let _guard = waiter.wait().await;

            // Async functions can still be called here; it's just that no new GUI events
            // will be processed.
            let size = window.inner_size().await;

            // Get the drawing context.
            let graphics = match &mut sb {
                Some(graphics) => graphics,
                sb @ None => {
                    let graphics = unsafe {
                        GraphicsContext::new(&window, &window)
                    };

                    sb.insert(graphics)
                }
            };

            // Fill the window with red.
            let red = 0xFF0000FF;
            buf.resize(size.width as usize * size.height as usize * 4, red);
            graphics.set_buffer(&buf, size.width as u16, size.height as u16);
        }
    };

    // Run in parallel with the window being closed.
    window.close_requested().or(redraw).await;
    target.exit().await
});
```

There's definitely more that's possible here.

## The Future

You may be looking at [`async-winit`], which is an asynchronous version of [`winit`]. You may also be looking at another recently released crate of mine, [`piet-glow`], which is a GPU-accelerated drawing framework. This may lead you to say "okay, notgull, you silly goose. What are you up to?"

One of my overarching goals in the Rust open source world has been to make a GUI framework. I'm still working out the exact details of how it would work, but the main idea is to represent widgets and their events as asynchronous tasks. Sort of like [`async-ui`], but with events representable as futures rather than callbacks. The idea is to allow flexibility in how events are handled and how they are propagated across the widget tree.

[`async-winit`] would be a cornerstone of this framework. The idea is that [`async-winit`] would be used as the underlying layer for handling events. There are still a few more crates in between there and now, but I'm excited for the journey.

## Thanks

Like any other open-source project, [`async-winit`] is built on top of the work and community of others. I'd like to thank the following people:

- [Pierre Kreiger] and many other contributors for creating [`winit`], which, naturally, is the base of the project.
- Stjepan Glavina for creating [`async-io`](asynchronous IO), which served as inspiration for [`async-winit`]. In addition, several other of his crates ([`async-channel`], [`futures-lite`]) are used as the bread and butter of this project, making `async` possible.
- Many other people who contributed to the above projects, among others. Thank you for creating such a great community.

[`async-winit`]: https://crates.io/crates/async-winit
[`winit`]: https://crates.io/crates/winit
[`egui`]: https://crates.io/crates/egui
[`iced`]: https://crates.io/crates/iced
[`miniquad`]: https://crates.io/crates/miniquad
[`druid-shell`]: https://crates.io/crates/druid-shell
[future combinators]: https://crates.io/crates/futures-lite
[executors]: https://crates.io/crates/async-executor
[asynchronous IO]: https://crates.io/crates/async-io
[`async-net`]: https://crates.io/crates/async-net
[`async-channel`]: https://crates.io/crates/async-channel
[`futures-lite`]: https://crates.io/crates/futures-lite
[`async-ui`]: https://crates.io/crates/async-ui
[`piet-glow`]: https://crates.io/crates/piet-glow
[Pierre Kreiger]: https://github.com/tomaka
