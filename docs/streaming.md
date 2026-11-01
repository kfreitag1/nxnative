# Streaming

This fork turns the Mac app into a fixed-size streaming source: the game still runs as nxnative runs it, but the present draws into a 4K drawable by default, and MetalFX scales the game's picture into it. The intent is a 4K60 capture for a Sunshine host, with per-game work where 4K output is not the same as 4K rendering.

## The Mac present

- The drawable is 3840 × 2160 whatever the window (`DM_PRESENT_SIZE`, the profile's `present_size`, default `3840x2160`). `2560x1440` and `window` are the other values. `front_mac.m` sets the layer's drawable size and aspect-fits it into the view, so a small window shows a letterboxed 4K picture instead of a stretched one.
- The present scales the game's own picture into that drawable. `DM_PRESENT_UPSCALE=metalfx` is the Mac default; `off` scales bilinearly.
- The game's picture is still what it was: at most 1920 × 1080 docked (and less for games with dynamic resolution or a 900p-class internal buffer). The 4K drawable is output size, not render size.

The compositor scales the layer to the window. A capture that reads the screen's pixels gets the window's size, not the drawable's. Capture the layer's surface (or run the app fullscreen on a 4K display or a 4K virtual display) to keep the 4K frames.

`DM_PRESENT_SIZE=window` restores the upstream behaviour for benchmarks, regressions and development runs. The Mac profile default is 4K, so a run of `scripts/bench.py` or `scripts/regress.py` on this fork measures the 4K present unless that variable is set.

## What 4K output costs

The present itself is cheap: one MetalFX spatial pass from 1080p to 4K per frame. The GPU cost is in the game's own rendering, which does not change. The M4 Pro has the encode hardware for 4K60 HEVC; the capture path and the encoder settings belong to the streaming host, not to this fork.

The game's frame rate is unchanged by the drawable. A 30 fps game still produces 30 new pictures a second; a 4K60 stream of it is 30 real frames and 30 duplicates. The frame-step and 60 fps unlocks in `dm/games/*/patches.py` are the per-title work for that, independent of resolution.

## Per-game resolution tiers

True 4K rendering is not a global switch. The game allocates its own render targets and memory pools for a 1080p window (`dm/host/hle_oe.c`, `GetDefaultDisplayResolution`), the UI is composited by the game into that same window texture, and `DM_RENDER_SCALE` is capped at the profile's `render_scale_max` (1.0 by default) because the pools are the game's. Each tier below is a per-game commit.

### Output only (the default)

Nothing in the game changes. The profile keeps `present_size 3840x2160` and the present upscales 1080p-class content. A game whose 4K present is not worth its cost uses `present_size 2560x1440`.

### 1440p 3D with a 4K composite

The game's 3D views render at about 1440p, while its window texture stays 4K, so the UI glyphs and layout rasterize at 4K and the game's own composite scales the 3D into it. This is the same shape as a Switch game rendering 900p into a 1080p output, one step larger.

Requires:

- `render_scale_max` raised in the profile, and the render-scale stub accepting it.
- the game's view-size function and dynamic-resolution controller patched so they do not clamp back to the Switch's sizes;
- pools large enough for the larger views (the reason for the 1.0 ceiling), and memory measured;
- the UI layout's scale following the 4K window, not a fixed 1920 × 1080 layout.

### 4K 3D

The same patch set with the views at 4K. On an M4 Pro this is scene-dependent: Breath of the Wild's outdoors frame is about 6.5 ms of GPU time at 1600 × 900 and carries about 800–900 MB of attachment traffic, so 4K is roughly 5.8 times the pixels and more than the machine's memory bandwidth at 60 fps. Expect 30 fps targets and lighter titles to be the realistic candidates.

### 4K UI

The window textures and the final composite at 4K, with the 3D views wherever the tier above puts them. This is what makes the UI native rather than upscaled, and it is the prerequisite for the 1440p tier: without a 4K window texture there is no 4K surface for the UI to draw into.

## Per-game commit shape

One branch and one squashed commit per game and tier, based on `stream/4k-present`:

```
stream/<title>-1440p
stream/<title>-4k
stream/<title>-4k-ui
```

Each commit carries:

- the profile changes (`present_size`, `render_scale_max`, and any new key the patch needs);
- the rewriter cuts in the game's `patches.py`, with the build ID they were verified on;
- the game's README note: what was enlarged, what was measured, and what is still open;
- the default `DM_PRESENT_SIZE` fallback if the tier does not hold at 4K.

Verify with `DM_SHOW_FPS=1`, `scripts/bench.py` for frame times and GPU time, and a screenshot at 4K for the UI's scale. A tier that cannot hold its frame rate at 60 is still useful at 30, but say so in the profile rather than leaving it to the player's GPU.

## Near-term code

- A profile key for the window texture size, so the 4K UI tier can patch the guest's window allocation through the profile instead of a per-game stub.
- A `present_size`-aware capture hook that hands the 4K surface to the streaming host without a virtual display.
- The render scale row's ceiling in the Mac settings window is already the profile's `render_scale_max` through `settings.py`.
