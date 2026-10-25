# Rainy fork of OpenFL 9.5.2

Upstream: [openfl/openfl](https://github.com/openfl/openfl) 9.5.2, MIT.
The base is imported as a single commit — the whole tree is present, upstream
history is not.

## What differs from upstream

### Stage3D GL state shadowing (`display3D/Context3D`)

`__flushGLTextures` ran on every draw call and re-bound the texture even when the
sampler already held it. A redundant `bindTexture` costs a full driver
validation pass, and this workload submits enough draws for that to matter.

The cache is only trusted when it is actually current:

- `i < bound.length` guards the lookup. An out-of-range read also returns null,
  so a missing entry would look identical to a recorded null binding and the
  first draw after every invalidate would skip an unbind that was still needed.
- `glSamplerStale[i]` marks slots touched by paths that bypass the flush —
  texture upload, video frames, raw binds. Those used to be skipped silently and
  sampled whatever the other caller had left on the unit.
- The AGAL alpha sampler (`sampler + 4`) is bound on the unchanged-texture path
  as well. It used to be skipped there, so swapping the alpha texture while the
  colour texture stayed put kept the old alpha, and turning `textureAlpha` off
  left the "enabled" uniform at 1.

`__bindGLTexture2D` / `__bindGLTextureCubeMap` are the single exit for every GL
texture bind, which is what makes the stale marking complete.

The viewport is shadowed too, and `OpenGLRenderer.setViewport` routes through it.
Writing `gl.viewport` directly left Stage3D believing the older rectangle was
current, and the next draw that wanted that rectangle skipped the write — the
frame landed in the wrong region.

`__invalidateGLCache` resizes its arrays in place; it runs several times per
frame and `new Array()` each time handed the collector a fresh object per call.

### Depth texture attachment (`display3D/textures/TextureBase`)

Optional `depthTextureEnabled` (default `false`) and `getDepthTextureID()`. On
attach the framebuffer is completeness-checked and falls back to a renderbuffer
when the depth attachment is incomplete, so a driver that refuses the format
degrades instead of drawing nothing.

### `BitmapData.__cpuFill`

Fills the CPU buffer instead of doing a scissored framebuffer clear. The
framebuffer path costs two render-target switches per call, which is ruinous for
UI repainting hundreds of small rectangles per frame. `getTexture()` re-uploads
on `image.version` change and `ImageDataUtil.fillRect` bumps it, so the GPU copy
follows on the next bind.

## Install

```bash
haxelib git openfl https://github.com/GengYan0/openfl-rainy-fork
```

Pairs with the Flixel and Lime forks:

```bash
haxelib git flixel https://github.com/GengYan0/flixel-rainy-fork
haxelib git lime    https://github.com/GengYan0/lime-rainy-fork
```
