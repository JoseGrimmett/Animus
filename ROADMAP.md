# Roadmap

Each phase ends with something playable. Don't start a phase until the previous one's "done when" passes.

---

## Phase 0: Pixels on screen

- [ ] Toolchain installed, CMake and SDL3 building
- [ ] `scratch/hello_pixels.c` runs (see SETUP.md)
- [ ] Debugger works: breakpoint inside the pixel loop, inspect `pixels[0]`
- [ ] Git repo initialized, `docs/devlog.md` started

**Done when:** a 320×200 framebuffer you write by hand shows on screen, scaled up with crisp pixels.

---

## Phase 1: Engine skeleton and Pong

**Engine**
- [ ] `types.h`, `log.h` with `RASTER_ASSERT`
- [ ] Arena allocator. The game gets one memory block from the engine.
- [ ] Platform layer: window, event pump, high-resolution timer, present framebuffer
- [ ] `RasterApp` callback table and `raster_run()`
- [ ] Fixed-timestep loop with accumulator and render interpolation
- [ ] Input: logical buttons with `down` / `pressed` / `released` edges, keyboard mapping
- [ ] Software raster: `fb_clear`, `fb_pixel`, `fb_rect`, `fb_line` (Bresenham)
- [ ] Bitmap font: an 8×8 font baked into a C array, `fb_text` (bring back the C64 charset)
- [ ] On-screen FPS and frame-time counter

**Game: Pong**
- [ ] Paddles, ball, wall bounces, scoring, serve
- [ ] Paddle hit angle depends on where the ball hits the paddle
- [ ] Simple AI opponent
- [ ] Title / play / game-over states (a basic state machine)

**Done when:** Pong plays the same at 30, 60 and 144 Hz monitor refresh, and `games/pong` contains no SDL includes.

---

## Phase 2: Sprites, tiles and a platformer

**Engine**
- [ ] Load PNGs with stb_image into a `Texture` (ARGB8888)
- [ ] `fb_blit` with source rects (sprite sheets), clipping and color-key or alpha transparency
- [ ] Horizontal flip for sprites
- [ ] Sprite animation helper: frame list, frame duration, looping
- [ ] Camera: world-to-screen offset, follow target, clamp to level bounds
- [ ] Tilemap: grid of tile indices, draw only visible tiles
- [ ] Collision: AABB overlap, tile-vs-AABB resolution split into X and Y passes
- [ ] Gamepad support through the platform layer

**Game: Platformer**
- [ ] Player run, jump, gravity, variable jump height
- [ ] Coyote time and jump buffering (small things that make it feel good)
- [ ] One hand-authored level (a hardcoded array is fine here)
- [ ] Collectibles and one enemy type that patrols
- [ ] Death and respawn

**Done when:** the platformer feels good to play, and any code shared with Pong has been moved into the engine.

---

## Phase 3: Data-driven content

**Engine**
- [ ] Asset manager: load by path, cache by handle, free on scene unload
- [ ] Audio with miniaudio: one-shot SFX and looping music, volume control
- [ ] Entity storage: start with a fixed-size array of structs and a type tag. Reach for ECS only if this hurts.
- [ ] Scene and level files in JSON (cJSON): tilemap, entity spawns, properties
- [ ] Debug overlay: draw collision boxes, entity IDs, toggled with F1
- [ ] Hot reload of level files (watch the modified time, then reload)

**Game**
- [ ] Platformer levels load from `assets/levels/*.json`
- [ ] Level transitions, sound effects, music

**Done when:** you can add a new level without recompiling.

---

## Phase 4: GPU renderer

- [ ] Work through LearnOpenGL "Getting Started" in a scratch project first
- [ ] glad loader, OpenGL 3.3 core context through the platform layer
- [ ] `RenderBackend` function table (see ARCHITECTURE.md)
- [ ] GL backend: textured quads, orthographic projection, sprite batching (one draw call per texture)
- [ ] Texture atlas support
- [ ] Render at the logical resolution into an FBO, then upscale with nearest filtering to keep the pixel-art look
- [ ] Command-line flag `--renderer=software|gl`

**Done when:** the platformer runs unchanged on both backends, and the GL backend handles 10,000 sprites at 60 FPS.

---

## Phase 5: Tools

- [ ] Dear ImGui (cimgui) integrated in a separate `tools/editor` executable
- [ ] Tile painting, entity placement, and a property inspector
- [ ] Save and load the same JSON format the game reads
- [ ] Play-in-editor: launch the level from the editor
- [ ] Optional: hot-reload the game code as a DLL (Handmade Hero day 21–24)

**Done when:** you build a full level in the editor and play it without touching a text file.

---

## Stretch ideas

- Third game in a different genre (top-down shooter, puzzle game) to stress-test the engine
- Particle system
- Tweening and easing library
- Simple profiler: scoped timers drawn as bars in the debug overlay
- Fixed-point math mode, just for C64 nostalgia
- Port the renderer to a 6502-style palette and resolution as a "retro mode"

---

## Reference

- *Game Programming Patterns*, Robert Nystrom (free online): game loop, update method, component, state
- "Fix Your Timestep!", Glenn Fiedler
- *LearnOpenGL.com*: Phase 4
- Handmade Hero: platform layer, software renderer, hot reload
- *Game Engine Architecture*, Jason Gregory: reference when you need depth
- Mike Acton, "Data-Oriented Design and C++" (CppCon 2014)
