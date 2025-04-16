# Efficient Ebitengine

> [!WARNING]
> UNDER CONSTRUCTION

This article is oriented to programmers familiar with [Ebitengine](https://github.com/hajimehoshi/ebiten) —the 2D game engine written in Golang by Hajime Hoshi—; it explains how to use the engine's API in an efficient way, avoiding common pitfalls and showcasing the most performant ways to get your graphics into the screen.

Internally, Ebitengine uses different techniques to boost rendering performance, but they are not always documented or easy to discover. This article should help you speed up the process of learning about the most important parts:

- What is draw command batching and how to avoid breaking batching.
- How do internal atlases and automatic atlasing operate.
- What are the most common performance pitfalls.
- Performance tips and optimization advice.
- Tips for idle / non-game applications.

Notice that Ebitengine's website already includes one [performance tips](https://ebitengine.org/en/documents/performancetips.html) page which you should check before going deeper into this article. Most points covered there are also covered here with more context and in deeper detail.

----

## Two words of context

Before jumping into Ebitengine details, we need to understand the underlying GPU situation. It can be summarized pretty simply, but there are two key elements you *absolutely* need to understand:

- The commands we are actually sending to the GPU.
- Why sending commands to the GPU is slow.

### Basic GPU commands

Unless you are diving into [shader programming](https://github.com/tinne26/kage-desk), computer graphics boil down to pretty much only two points:
1) Sending image data (textures) to GPU memory.
2) Transfering image data between textures by *drawing triangles*.

This might sound overly simplistic, but it's actually fairly accurate. Ebitengine's model is quite close to this too:
- You can create images.
- You can draw images onto other images (one of which will be the screen or main canvas you are rendering to).

Internally, to "draw an image onto another" means to select a rectangular region from a GPU texture and drawing it onto a rectangular region of another texture. The rectangular regions, or quads, can be created with two triangles (which are the main primitive shape that GPUs understand).

So basically we have two main commands for the GPU:
- Send image data.
- Send orders to transfer or operate with image data already in the GPU.

### Why communication is slow

Of the two commands described (sending image data and sending operation commands), the first is much more expensive than the later. Sending images can lead to very significant data transfers:
- A 256x256 image, in RGBA format, takes 4 bytes per pixel, 65536 pixels, for a total of around 0.25MB.
- A full sized offscreen of 1920x1080 is almost 8MB.

This is a first important lesson to learn: **avoid creating or loading new images often**; transferring/loading/creating multiple images per frame at medium to big sizes can very quickly make your application completely collapse.

But this is not the only reason communication with the GPU is slow: CPUs and GPUs are different components of your computer's hardware, and CPU<->GPU communication needs to happen with the mediation of the operating system using CGO, purego or system calls. All this makes the communication very *high latency* when compared to single process operation.

> [!IMPORTANT]
> Summarizing:
> - We want to minimize the amount of image data we send to GPU.
> - We want to minimize the number of commands we send to GPU, as communication is high latency.

Don't worry if you don't fully get this yet, we will keep coming back to it when needed. Feel free to revisit this section later when you've connected more pieces of the puzzle together. Let's go back to Ebitengine now.

----

## Draw batching, hot loops and reordering draws

The first optimization that Ebitengine does to avoid sending many commands to the GPU is **grouping similar, consecutive draw commands and sending them all together at once**. This is what we call **draw batching**. The [performance tips](https://ebitengine.org/en/documents/performancetips.html#Make_similar_draw_function_calls_successive) document lists the basic conditions that need to be satisfied to prevent "batch breaking".

Now, I'm not gonna lie to you: if you want to understand why something would or wouldn't break batching, you need to be familiar with [`DrawTriangles`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Image.DrawTriangles) and [`DrawTrianglesShader`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Image.DrawTrianglesShader) directly. If you can express multiple image draws as just more triangles on a `DrawTriangles` or `DrawTrianglesShader` call, then it can be batched as a single draw command. Otherwise, just memorizing the conditions is the way to go. If you want to learn more about those functions, feel free to check out the [triangles tutorial](https://github.com/tinne26/kage-desk/blob/main/docs/tutorials/triangles.md) (but you absolutely don't *need* to).

In more practical terms, this means that during a sequence of draws —especially **hot rendering loops** that draw dozens or hundreds of elements—, you want to make sure that you:
- Are rendering only to a single target.
- Don't change the `Filter` nor `Blend` fields of [`DrawImageOptions`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#DrawImageOptions). You can reuse the draw image options and rely on some of its `Reset()` functions, or create new options if needed. Creating new options does not break batching on itself, but if it's natural to reuse the options it's typically more efficient to do so.
- Do *not* use deprecated `DrawImageOptions` fields like `CompositeMode` or `ColorM`. Both can or will break batching.
- Be careful with external functions that might do their own drawing and break batching for you.

When you start looking for these, you will often find yourself **reordering or regrouping draw calls** to put similar calls together. This is one of the most common and effective optimization techniques that you should have in your repertoire.

> [!NOTE]
> If you are curious, the reason `Filter` breaks batching is that different filtering types require different shaders in Ebitengine. `Blend` does not require different shaders, but it requires different draw command "settings". If you tried, you would see that you can't express multiple draws with different filters or blends as a single `DrawTriangles` command *(well... you could create a single shader that uses different filtering methods dynamically, but this would only make sense in very rare cases that are outside of scope)*.

## Internal atlases and automatic atlasing

...

## TODO

- draw triangles and unmanaged atlases when really needed
- do not draw (screen cleared every frame = false, needsRefresh flags, etc)
- profiling example
- PGO
- debug tags and automated analyzer
- unlocked FPS for testing
- different graphical backends (especially useful on Windows)
- disable mipmaps when relevant on FilterLinear
