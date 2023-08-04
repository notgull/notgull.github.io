---
layout: post
title: The Quest for Piet 2, or, the Revenge of the Gradients
categories: [gui, rust, piet, drawing, questforpiet]
excerpt: More samples! More problems!
comments: true
---

It's high time for some vector graphics.

[Last time](/quest-for-piet/), we set up our harnesses for [`piet`] rendering using our hardware-based backends and adjusted it for multisampling and scaling. Today, let's focus on getting some more samples rendered. The next few samples exhibit line styling and gradient rendering, obviously subjects that are easy and unopinionated. Let's get knee deep.

# Number 3: Dashed-Line Duel

This sample demonstrates some of the properties of strokes. When drawing a line, there are a handful of properties you can assign to that line, like width, line cap and the dash pattern. This sample covers those cases, and (as I found out) then some.

The reference image looks like this:

![Reference](/images/cairo-test-03-2.00.png)

Our renderers, [`piet-glow`] and [`piet-wgpu`] both, spit this out:

![Glow](/images/piet-glow-03-2.00.png)

Let's ignore the dashed lines for now. The transforms on the little line-cap chevrons are messed up. I think it's a matter of the order of operations, since the unscaled version of the image isn't affected:

![Glow 1](/images/piet-glow-03-1.00.png)

Since I apply the scale using the `transform()` function, I think I'm doing matrix operations in the wrong order. Right now, I'm doing this:

```rust
// Somewhere in piet-hardware
fn transform(&mut self, transform: Affine) {
    // Get the slot representing the
    let slot = &mut self.state.last_mut().unwrap().transform;
    *slot = transform * *slot;
}
```

So, I should be doing this:

```rust
*slot = *slot * transform;
```

...or, more succinctly:

```rust
*slot *= transform;
```

![Glow v2.0](/images/piet-glow-03-improvement.png)

I forget why I did it the first way. I originally dogfooded this API by using it in a school project of mine. So, either I was using the API wrong or I was doing something weird in my school project. Either way, it's fixed now.

Now, let's talk about dashing. Dashed lines are when you have your big line be made up of a bunch of little lines. Right now, I'm using the [`tiny-skia`] crate to add support for dashing. I'm already using [`tiny-skia`] in a handful of other places, such as gradient generation (foreshadowing). Therefore, just using its facilities makes sense. I pretty naively just convert a path from `piet::Shape` to `tiny_skia::Path`, use its built-in `dash()` function, and just convert it back.

```rust
if !style.dash_pattern.is_empty() {
    let dash = tiny_skia::StrokeDash::new(
        style.dash_pattern.iter().map(|&x| x as f32).collect(),
        style.dash_offset as f32,
    );

    let dashed = dash.and_then(|dash| {
        // We cache the path builder to save on allocations.
        let mut inner = self.dash_builder.take().unwrap_or_default();

        // Convert our shape to a path.
        super::shape_to_skia_path(&mut inner, &shape, tolerance);
        let shape = inner.finish()?;

        // Actually dash the path.
        shape.dash(&dash, 1.0)
    });

    // <snip>: Actually drawing the dash pattern. 
}
```

*Futuregull Edit: Apparently `tiny-skia` just doesn't support odd-numbered dashes. Whoops!*

My first thought was that the resolution scale is too low, so I bumped it up a bit to no noticeable differences. My second thought was that maybe [`tiny-skia`]'s dashing system wasn't as advanced as I thought it was. I figured it was worth it to try a more advanced rasterizer, so I found myself reaching for [`zeno`]. Let's just try translating the path to a [`zeno`] path and see what happens.

![Glow v2.1](/images/piet-glow-03-unimproved.png)

Okay, that's even worse.

*Futuregull Edit: After some discussion in the `kurbo` chatroom, apparently I just did it wrong. Whoops again! This is not indicative of the `zeno` crate, just my inability to write code that works in prod.*

I just noticed that [`kurbo`], the geometry library underlying [`piet`], has an [open PR](https://github.com/linebender/kurbo/pull/286) that adds stroke rasterization. This includes dashing support. Maybe it's worth a shot?

```toml
[patch.crates-io]
kurbo = { git = "https://github.com/linebender/kurbo.git", branch = "stroke" }
```

![Glow v2.2](/images/piet-glow-03-almost.png)

I'm *almost* there. It looks like the first dash gets skipped in most of the dashed lines. Oh well, that's close enough for now. What's next?

# Number 4: Gradient Gamble

This sample is designed to show off gradients. Here's what the reference image looks like:

![Reference](/images/cairo-test-04-2.00.png)

...and there is no [`piet-glow`] image, because the renderer panicked! Whoop dee doo!

Apparently it's because a gradient with a size of zero is being passed in to some method somewhere along the way, which I forgot to handle. For straight line gradients, the width or height will be zero, which [`tiny-skia`] doesn't like.

*I can't believe we're only on sample number four, please free me from this mortal coil...*

I did some investigation, and it turns out that this stems from the fact that I did gradients wrong. You see, I was thinking mostly about radial gradients when I was writing the implementation for gradients. I defined a "bounding box" for which the gradient wouldn't repeat inside of.

Gradients are actually very difficult to do right inside of shaders. It's possible if you know how many steps it will have ahead of time (albeit somewhat inefficiently), but in this case we don't. Properly rendering a gradient would require the equivalent of a "for" loop, and GPU's don't do branching well. They're meant to render a large multitude of parallel operations and branching isn't parallel. You'd end up with a bunch of GPU cores blocked on other GPU core doing branching, which is frankly wasteful.

So I "cheat", somewhat. I use [`tiny-skia`] to render the gradient into a GPU texture, then render that texture whenever I need a gradient. But these gradients are theoretically infinite, so how do I know how much of the gradient to render?

For radial gradients, my strategy was to take the circle formed by the gradient (center then radius), figure out the bounding box of that circle, and then only render that sized rectangle. From there, I can "clamp" the edges of the texture so that it keeps rendering with the last color indefinitely. This successfully creates the illusion of a gradient that goes on forever.

For linear gradients, I must've not eaten breakfast on the day that I wrote that code. The gradient has a fixed "start" and "stop" point, so I just did the same thing but with a rectangle made up of those two points. This worked for the diagonal lines that I tested on, but I forgot that:

- Many gradients are just a straight horizontal line.
- The height of a horizontal line is zero.

So, that's where that panic comes from. It's a simple enough fix:

```rust
// snip: somewhere in the gradient rendering code

let mut bounds = Rect::from_points(gradient.start, gradient.end);

if approx_eq(bounds.width(), 0.0) {
    bounds.x1 += 1.0;
}

if approx_eq(bounds.height(), 0.0) {
    bounds.y1 += 1.0;
}
```

With that error taken care of, we can finally render the example.

![Glow](/images/piet-glow-04-2.00.png)

The linear gradient looks alright, but that radial gradient looks like the offset is broken. I just misinterpreted the "origin offset" code that they had. The origin offset is used to offset the origin of the gradient by some amount, which can give the illusion of a light falling on the sphere. It's an easy enough fix.

The current code looks like this:

```rust
let shader = tiny_skia::RadialGradient::new(
    convert_to_ts_point(gradient.center),
    convert_to_ts_point(gradient.center - gradient.origin_offset),
    /* ... */
);
```

Once I change it to this:

```rust
let shader = tiny_skia::RadialGradient::new(
    convert_to_ts_point(gradient.center + gradient.origin_offset),
    convert_to_ts_point(gradient.center),
    /* ... */
);
```

...it seems to produce the output that I want.

![Glow v2.0](/images/piet-glow-4-improved.png)

Gradients are fixed, forever!

## Number 6: Line Liquidation

I'm skipping number five because it involves text rendering, which is going to be a whole deal that I'll cover in another post. In addition, the comedic timing is also better.

The reference image:

![Reference](/images/cairo-test-06-2.00.png)

Our image:

![Glow](/images/piet-glow-06-2.00.png)

The dashing is a little bit off, I guess? But like I said, close enough for government work. Some of the gradients are also blended in a different way, but I can just chalk that up to [`tiny-skia`] using a different gradient engine than `cairo`. 

The shapes on the right are too different to ignore, unfortunately. The shapes on the right are rotated in the reference image, but not ours. Looking at the source code for the sample, I'm not sure why this is. Let's just disable the mask on the gradient, and I'm sure we'll figure out what the problem is.

![Expanded](/images/gradient-expand.png)

Ach, there's the problem. Turns out that padding out the gradient doesn't actually solve all of our problems. In fact, it makes more of them. The gradient just gets squeezed horizontally like this.

Here's a better visualization of what's going on:

![Rotation](/images/rotation.gif)

What are the possible solutions to this? One of the possible solutions would be to render the gradient on demand on every frame, but that would be a disaster for performance on debug builds. Not to mention, it means drawing a linear gradient will allocate memory on every frame. One of my goals is to avoid per-frame memory allocations if possible.

I've just realized that, since perfectly horizontal or perfectly vertical gradients work well, what if I rotate linear gradients so that they're always horizontal or vertical. This can be done by feeding the shader a uniform rotation to apply to coordinates indexing into the gradient.

It'll require a rewrite of some of our infrastructure. Let's give it a shot.

Okay, so after some rewiring and rewriting a few shaders, here's what I made:

![Compressed](/images/piet-glow-6-compressed.png)

I'm not sure why it's decided to compress the gradient into a single line, nor why it's barely even rotated at all. But you know what, it's 11:00 PM here, so I'll get some sleep and see what the problem looks like in the morning... Actually, before I go to bed, maybe [RenderDoc] can tell us something I don't know? 

> ### Grotesque, Deformed Bear's Unhinged Ramblings
> 
> If you ever find yourself in a position where you need to debug any GPU code, [RenderDoc] is an incredibly useful swiss-army-knife to have. It lets you take apart the GPU pipeline and see what's going on inside. Inspecting the render pipeline has already saved my bacon more times that I can count.

![Doc](/images/gradient-isthere.png)

Huh, weird, it looks like it's just a shader issue? Maybe if I... aw, man, it's already midnight. Time flies when you're having Fun, I suppose.

Okay, after a night's sleep I'm feeling a bit more refreshed. Now that I think about it, I take a look at the RenderDoc output and see that the Y UV coordinate is too high. That seems weird. What if I just map it to the original gradient bounds?

![Doc](/images/piet-glow-6-improved.png)

Okay, that's better, but not perfect. Let's check the scale one equivalent to see if it works.

![Doc](/images/piet-glow-6-uh-oh.png)

Oh, bother.

## Gradient Grind

At this point, I've given up on the whole "doing it right" thing and have now moved on to "get it done". Which means it's time for some software math!

In general, you want all math to be done on the GPU, where it's nice and parallelized. But, failing that, there's always the CPU. I'm already doing a little bit of processing here for proper scaling. Come on, what's one more?

Nope, tried that. I got the same problems... oh wait, I see what the problem is now.

I made a silly mistake. Here's what my code for calculating the rotation looked like:

```rust
let old_bounds = Rect::from(old_gradient.start, old_gradient.end);

// Use that to create a new gradient.
let new_gradient = FixedLinearGradient {
    start: gradient.start,
    end: horizontal_end,
    stops: gradient.stops,
};

// A transform that maps UV coordinates into this plane.
let offset = gradient.start.to_vec2();
let new_bounds = Rect::from_points(new_gradient.start, new_gradient.end);
let transform = scale_and_offset(old_bounds.size(), old_bounds.origin())
    * Affine::translate(offset)
    * Affine::rotate(-angle)
    * Affine::translate(-offset);
```

Do you see the problem yet? Computer! Enhance!

```rust
let transform = scale_and_offset(old_bounds.size(), old_bounds.origin())
    * /* ... */;
```

That's a bonehead move on my part. I accidentally used the gradient bounds of the un-straightened gradient to squash the straightened gradient. No wonder it was squashed! I can't believe it took me that long to spot it.

Changing it to this:

```rust
let transform = scale_and_offset(old_bounds.size(), old_bounds.origin())
    * /* ... */;
```

That seems to fix the gradient rotation...

![Fixed rotation](/images/fixed_rotation2.gif)

...and now the example puts out the right picture!

![Glow v3.0](/images/piet-glow-06-finished.png)

All it took was just a little rewrite of my application's entire dashing engine and entire gradient engine. Yog Soggoth take me now!

I think that's enough for today. [Next time](/quest-for-piet-part-3/) we'll take a look at text rendering and how it can be improved.

# Addendum: Giving Zeno Another Shot

I talked to the developer of [`zeno`], and it turns out that I was just using it wrong.

You see, [`zeno`] has a `PathBuilder` interface that can be used to build up a path in whatever format you want. I tried to set it up so that it rendered straight into a `kurbo::BezPath`, but apparently that's hard to do unless you're using `zeno::Command`s.

After some brief rewriting of the stroking code, I get a much better result after rendering:

![Glow v99.0](/images/piet-glow-03-attempt2.png)

That seems to work out better than [`kurbo`] does.

[`piet`]: https://crates.io/crates/piet
[`piet-glow`]: https://crates.io/crates/piet-glow
[`piet-wgpu`]: https://crates.io/crates/piet-wgpu
[`tiny-skia`]: https://crates.io/crates/tiny-skia
[`zeno`]: https://crates.io/crates/zeno
[`kurbo`]: https://crates.io/crates/kurbo
[RenderDoc]: https://renderdoc.org/
