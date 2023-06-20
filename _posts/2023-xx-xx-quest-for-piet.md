---
layout: post
title: The Quest for Piet
categories: [gui, rust, piet, drawing]
excerpt: How do you create a drawing toolkit?
comments: true
---

Lately I've been working on a drawing framework called [`theo`]. It's based on the [`piet`] API, which has famously been used in the [`druid`] GUI framework. The goal is to have a vector drawing framework that can be used out-of-the-box with something like [`winit`]. Another goal is to take advantage of the GPU in order to draw. [`wgpu`] is used in the common case for rendering.

I've noticed that [`piet`] comes with a "samples" framework. It has a few examples of using the drawing API, which aims to consistently draw a set of "sample" images. These images showcase some features of the API, like text, transforms and clipping. The CI for [`piet-common`], the reference implementation of [`piet`], runs these samples, saves them to images and compares the output to a set of reference images. This is a great way to use CI to tests for regressions in the drawing API.

I decided to try using these samples for myself. I wasn't ready for what happened next.

## Get Going

Well, I would think that I've done a good job of implementing the API so far, right? Let's hook up the samples and see what happens.

[`theo`]'s GPU implementations are based on two other crates: [`piet-wgpu`] and [`piet-glow`]. I am a firm believer in compartmentalized software. If you were to develop some kind of user interface that runs on top of OpenGL, you could take [`piet-glow`] and use it for drawing. If I feel that a part of the code can be used for some other purpose, I like to split it off into another crate.

I started with [`piet-glow`] first, because I'm a 21-year-old Boomer who only knows GL. This turned out to be a bit of a mistake; [`wgpu`] is way better at rendering images without a window present than OpenGL. But, as it turns out, [`wgpu`] is its own dance. More on that later.

So! We need a display and a window to render to, even though we're just drawing to images that we're saving to files, because... OpenGL! Don't worry, with [`winit`] and [`glutin`] at arms reach, creating a window and a GL context... isn't trivial, but also isn't too hard. In fact, I can even reuse parts of the existing harness I use to run OpenGL examples for [`piet-glow`].

I like to think that I have a pretty good setup. Just from the `util` module, I can instantiate a `GlutinSetup` type that contains a GL display, a GL context, a window (for Windows, where you need to create the window before you create the GL display) and a context that can be made "current". It also provides functionality to create the [`glow`] context.

```rust
use winit::event_loop::EventLoop;

#[path = "util/setup_context.rs"]
mod util;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    util::init();
    let event_loop = EventLoop::new();
    let mut glutin_setup = util::glutin_impl::GlutinSetup::new(&event_loop)?;

    // TODO: Actually do things.
}
```

If I were writing a cross-platform program, I would actually have to start the [`EventLoop`], wait for the [`Resumed`] event, and then create the `GlutinSetup`. This is because some platforms don't like it when you create a window before that event is delivered. But it seems to work well enough on X11 (like I said, 21-year-old boomer, I know I should be using Wayland), so I'll keep it this way until someone complains.

Of course, I fear God enough to wait for `Resumed` before I do any actual graphics work, so setting up the base context for [`piet-glow`] looks like this:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    util::init();
    let event_loop = EventLoop::new();
    let mut glutin_setup = util::glutin_impl::GlutinSetup::new(&event_loop)?;

    event_loop.run(move |ev, elwt, flow| {
        flow.set_wait();

        if let winit::event::Event::<()>::Resumed = ev {
            // Create a rendering context.
            let glow_context = glutin_setup.make_current(elwt)();

            // Create the piet-glow context.
            let context = unsafe { piet_glow::GlContext::new(glow_context).unwrap() };

            // TODO: Use this context to draw.
        }
    })
}
```

Note that [`EventLoop::run`] return `!`, the never type. This can be cast to our result type here pretty easily, in case you're wondering what's happening.

Now that we have a [`glow`] context, we can now start running the samples. [`piet`] has a [`samples_main`] function that handles running, generating and comparing all of the samples. We just put the OpenGL code to create a texture, draw to it, and save it to file in a closure for that function.

Unless you have a high level of interest in OpenGL code bloat, feel free to scroll through this.

```rust
// snip: inside of the "Resumed" block
// Create a rendering context.
let glow_context = glutin_setup.make_current(elwt)();

// Create the piet-glow context.
let mut context = unsafe { piet_glow::GlContext::new(glow_context).unwrap() };

samples::samples_main(
    |number, _scale, path| {
        let picture = samples::get(number)?;
        let size = picture.size();

        // Create a texture to render into.
        let ctx = context.context();
        let texture = unsafe {
            let texture = ctx.create_texture().unwrap();
            ctx.bind_texture(glow::TEXTURE_2D, Some(texture));
            ctx.tex_image_2d(
                glow::TEXTURE_2D,
                0,
                glow::RGBA as i32,
                size.width as i32,
                size.height as i32,
                0,
                glow::RGBA,
                glow::UNSIGNED_BYTE,
                None,
            );

            // Set up the texture parameters.
            ctx.tex_parameter_i32(
                glow::TEXTURE_2D,
                glow::TEXTURE_MIN_FILTER,
                glow::LINEAR as i32,
            );
            ctx.tex_parameter_i32(
                glow::TEXTURE_2D,
                glow::TEXTURE_MAG_FILTER,
                glow::LINEAR as i32,
            );

            texture
        };

        // Use a framebuffer to render into the texture and make it current.
        let framebuffer = unsafe {
            let framebuffer = ctx.create_framebuffer().unwrap();
            ctx.bind_framebuffer(glow::FRAMEBUFFER, Some(framebuffer));
            ctx.framebuffer_texture_2d(
                glow::FRAMEBUFFER,
                glow::COLOR_ATTACHMENT0,
                glow::TEXTURE_2D,
                Some(texture),
                0,
            );

            // Check that the framebuffer is complete.
            assert_eq!(
                ctx.check_framebuffer_status(glow::FRAMEBUFFER),
                glow::FRAMEBUFFER_COMPLETE
            );

            framebuffer
        };

        // Use a renderbuffer to render into the texture and make it current.
        let renderbuffer = unsafe {
            let renderbuffer = ctx.create_renderbuffer().unwrap();
            ctx.bind_renderbuffer(glow::RENDERBUFFER, Some(renderbuffer));
            ctx.renderbuffer_storage(
                glow::RENDERBUFFER,
                glow::DEPTH_COMPONENT16,
                size.width as i32,
                size.height as i32,
            );
            ctx.framebuffer_renderbuffer(
                glow::FRAMEBUFFER,
                glow::DEPTH_ATTACHMENT,
                glow::RENDERBUFFER,
                Some(renderbuffer),
            );

            // Check that the framebuffer is complete.
            assert_eq!(
                ctx.check_framebuffer_status(glow::FRAMEBUFFER),
                glow::FRAMEBUFFER_COMPLETE
            );

            renderbuffer
        };

        // TODO: Use the context to draw.
    },
    "piet-glow", // Prefix to save to.
    None,
);
```

All this does is created a texture and registers it at the primary framebuffer. In OpenGL speak, it's a global variable that drawing calls will render into.

Actually, we can't do this. For some reason, [`samples_main`] takes a `fn(...)` instead of an `impl Fn(..)` like everything else does. I've [written a PR](https://github.com/linebender/piet/pull/558) to fix this, but let's take an alternate route for now.

```rust
// snip: inside of the "Resumed" block
// Create the piet-glow context.
let context = unsafe { piet_glow::GlContext::new(glow_context).unwrap() };

// piet takes a raw function pointer, making this workaround necessary.
std::thread_local! {
    static CONTEXT: RefCell<Option<GlContext<Context>>> = RefCell::new(None);
}

CONTEXT.with(move |slot| *slot.borrow_mut() = Some(context));

samples::samples_main(
    |number, _scale, path| {
        CONTEXT.with(|slot| {
            let mut guard = slot.borrow_mut();
            let context = guard.as_mut().unwrap();

            // snip: the rest of the block
        })
    },
    "piet-glow",
    None
);
```

Ick, I hate this. It'll do for now, in the same way as duct tape will do on a car engine.

The [`piet_glow`] `GlContext` type has a `render_context` method that lets you instantiate a new render context that implements `piet::RenderContext`. This is what we'll use to draw to the texture.

```rust
// snip: all of the GL initialization.

// Create a piet-glow render context.
let mut rc = unsafe {
    context.render_context(size.width as u32, size.height as u32)
};

// Draw with the context.
picture.draw(&mut rc)?;
```

The `draw` function runs all of the involved [`piet`] API calls on the rendering context. All we have to do it get the texture data and save it to a file.

```rust

```

[`EventLoop`]: https://docs.rs/winit/0.28.6/winit/event_loop/struct.EventLoop.html
[`Resumed`]: https://docs.rs/winit/0.28.6/winit/event/enum.Event.html#variant.Resumed
[`EventLoop::run`]: https://docs.rs/winit/0.28.6/winit/event_loop/struct.EventLoop.html#method.run
[`samples_main`]: https://github.com/linebender/piet/blob/master/piet/src/samples/mod.rs#L85-L99
