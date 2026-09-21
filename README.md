# Screen Zoom for MangoWM

Adds a proper screen zoom to [MangoWM](https://github.com/mangowm/mango) — hold a modifier and scroll to zoom in/out, centered on where your cursor is. Smooth animated, not janky. Resets cleanly back to 1x.

This patches MangoWM directly. It renders the scene to an offscreen buffer, crops a viewport around the cursor, and scales it back up to fill the output. Uses wlroots render passes so there's no extra compositor overhead.

This is a fork of the old version built for the latest release of Mango.

## What it does

- `screen_zoom_in` / `screen_zoom_out` — step zoom level by `zoom_speed`
- `screen_zoom_reset` — snap back to 1x
- `screen_zoom_set` — jump to a specific zoom level
- Viewport follows your cursor and clamps to screen edges
- Smooth ease-out animation between zoom levels
- Swapchain is only allocated while zoomed, freed when you reset
- Supports zooming with multiple monitors.

## Patch info

Built against MangoWM commit `999a546fd22cd7faef665f340d793f8d068d7255` (upstream `main` as of 2026-09-21).

The patch touches these files:

| File | What changed |
|------|-------------|
| `src/main.c` | Adds required wlroots headers (mentioned below) |
| `include/mango/config/parse_config.h` | New config options for: `zoom_max`, `zoom_speed` |
| `src/config/parse_config.c` | Register config functions and options. |
| `include/mango/dispatch/bind.h` | Function declarations for the 4 zoom actions |
| `src/dispatch/bind.c` | Function definitions for zoom in/out/reset/set |
| `include/mango/manage/monitor.h` | Adds monitor specific zoom variables: 'zoom_level', 'zoom_target', 'zoom_animating', 'zoom_x', 'zoom_y' |
| `src/manage/monitor.c` | Main zoom render loop is in here. |

## Dependencies

You need everything mangowc needs, plus you're specifically using some wlroots render APIs that are already linked but worth calling out:

**Build deps** (assuming Arch, adjust package names for your distro):
```
meson ninja gcc pkgconf
wayland wayland-protocols
wlroots (0.19.x) — this is the big one, the render_pass and swapchain APIs come from here
scenefx (>=0.4.1) — mangowc's effects library
libinput libdrm libxkbcommon pixman
libdisplay-info libliftoff hwdata seatd pcre2
xorg-xwayland libxcb xcb-util-wm  (optional, for xwayland support)
```

On Arch most of these pull in automatically if you just grab `wlroots` and `scenefx`. The AUR package `mangowc-git` has the full dep list if you want to cross-reference.

The zoom rendering specifically uses these wlroots headers (already part of wlroots, nothing extra to install):
- `wlr/render/pass.h` — render pass API
- `wlr/render/swapchain.h` — buffer management
- `wlr/render/drm_format_set.h` — format negotiation
- `wlr/render/wlr_texture.h` — texture from buffer

## How to apply

Clone mangowc and apply the patch:

```bash
git clone https://github.com/DreamMaoMao/mangowc.git
cd mangowc
git apply /path/to/screen-zoom.patch
```

If you're on a newer commit and the patch doesn't apply clean, you can try:
```bash
git apply --3way screen-zoom.patch
```
and resolve any conflicts manually. The changes are pretty self-contained so it shouldn't be bad.

## Build & install

```bash
meson build -Dprefix=/usr
sudo ninja -C build install
```

If you already have a build dir from before, just rebuild:
```bash
sudo ninja -C build install
```

## Configuration

Add these to your `config.conf` (usually `~/.config/mango/config.conf`):

**Zoom settings:**
```ini
zoom_max=6.0
zoom_speed=0.3
```

- `zoom_max` — how far you can zoom in (clamped between 1.0 and 20.0). 4x is plenty for most uses.
- `zoom_speed` — how much each scroll step changes the zoom level. 0.2 feels good, go lower for finer control.

**Keybindings (scroll wheel):**
```ini
axisbind=SUPER,UP,screen_zoom_in
axisbind=SUPER,DOWN,screen_zoom_out
```

This gives you Super + scroll-up to zoom in, Super + scroll-down to zoom out.

**You'll probably also want a reset binding:**
```ini
bind=SUPER,0,screen_zoom_reset
```

**Or if you want to jump to a specific level:**
```ini
bind=SUPER,equal,screen_zoom_set,2.0
```

See `config-example.conf` in this repo for a copy-paste ready snippet.

## How it works (brief)

The zoom runs in the compositor's render loop (`handle_output_frame` in 'monitor.c'). Each frame:

1. `screen_zoom_update(m)` interpolates `zoom_level` toward `zoom_target` on the selected monitor with ease-out (`+= diff * 0.15`)
2. If `zoom_level > 1.0`, `render_zoomed(m)` takes over instead of the normal scene commit
3. The scene gets rendered to a buffer normally via `wlr_scene_output_build_state`
4. A texture is created from that buffer
5. A viewport is calculated centered on the cursor position (clamped to screen edges)
6. That viewport region is drawn scaled-up to fill the full output using bilinear filtering
7. The zoomed buffer replaces the normal output buffer and gets committed

The swapchain for the zoom buffer is lazily created and destroyed when zoom returns to 1x, so there's zero cost when you're not zoomed.
