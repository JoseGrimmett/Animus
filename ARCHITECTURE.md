# Architecture

## Layers

```
┌──────────────────────────────────────────────┐
│  Game (pong, platformer)                     │  game rules, entities, levels
├──────────────────────────────────────────────┤
│  Engine API  (engine/include/raster/*.h)     │  the only thing games see
├──────────────────────────────────────────────┤
│  Engine modules                              │  render · input · assets · audio
│                                              │  physics · scene · memory · log
├──────────────────────────────────────────────┤
│  Platform layer  (platform/sdl_platform.c)   │  the ONLY place SDL is touched
├──────────────────────────────────────────────┤
│  SDL3 / OS / GPU                             │
└──────────────────────────────────────────────┘
```

Dependencies only point downward. If you ever want a different platform layer (raw Win32, or a C64-style emulator target just for fun), you replace one file.

## Full folder layout

```
raster/
├── CMakeLists.txt
├── README.md
├── docs/
│   ├── ARCHITECTURE.md
│   ├── ROADMAP.md
│   ├── SETUP.md
│   └── devlog.md
├── engine/
│   ├── CMakeLists.txt
│   ├── include/raster/
│   │   ├── raster.h          # umbrella include + app callback table
│   │   ├── types.h           # u8/u32/f32, v2, rect, color
│   │   ├── memory.h          # arena allocator
│   │   ├── log.h             # logging + assert macros
│   │   ├── input.h           # button state queries
│   │   ├── render.h          # framebuffer + draw calls
│   │   ├── assets.h          # textures, fonts, sounds (Phase 3)
│   │   ├── audio.h           # (Phase 3)
│   │   ├── collision.h       # AABB, tile collision (Phase 2)
│   │   └── scene.h           # entities + serialization (Phase 3)
│   └── src/
│       ├── core/
│       │   ├── engine.c      # main loop, fixed timestep
│       │   ├── memory.c
│       │   └── log.c
│       ├── platform/
│       │   └── sdl_platform.c
│       ├── input/
│       │   └── input.c
│       ├── render/
│       │   ├── render.c      # dispatches to the active backend
│       │   ├── software/
│       │   │   ├── sw_raster.c   # pixel, line, rect, blit
│       │   │   └── sw_text.c     # bitmap font
│       │   └── opengl/           # Phase 4
│       │       └── gl_renderer.c
│       ├── assets/
│       ├── audio/
│       ├── collision/
│       └── scene/
├── games/
│   ├── pong/
│   │   ├── CMakeLists.txt
│   │   ├── main.c            # fills in the app table, calls raster_run()
│   │   └── pong.c
│   └── platformer/
│       ├── CMakeLists.txt
│       ├── main.c
│       ├── player.c
│       ├── level.c
│       └── assets/           # game-specific sprites, tilesets, levels
├── tools/
│   └── editor/               # Phase 5
├── assets/                   # shared assets (default font, test sprites)
├── third_party/
└── scratch/                  # throwaway experiments, not built by default
```

## How a game plugs in

Games implement a small callback table. The engine owns `main` and the loop. This mirrors the C64 split between the IRQ handler and the main program.

```c
// engine/include/raster/raster.h
typedef struct RasterConfig {
    const char *title;
    int width, height;     // logical framebuffer size, e.g. 320x200
    int scale;             // window scale, e.g. 3
    int tick_hz;           // simulation rate, e.g. 60
    size_t memory_bytes;   // size of the permanent arena given to the game
} RasterConfig;

typedef struct RasterApp {
    RasterConfig config;
    void (*init)   (void *game_memory);
    void (*update) (void *game_memory, const Input *in, float dt);  // fixed dt
    void (*render) (void *game_memory, Framebuffer *fb, float alpha); // alpha = interp
    void (*shutdown)(void *game_memory);
} RasterApp;

int raster_run(const RasterApp *app);
```

```c
// games/pong/main.c
#include <raster/raster.h>
#include "pong.h"

int main(int argc, char **argv) {
    RasterApp app = {
        .config = { "Pong", 320, 200, 3, 60, 1 << 20 },
        .init = pong_init, .update = pong_update,
        .render = pong_render, .shutdown = pong_shutdown,
    };
    return raster_run(&app);
}
```

The game gets one block of memory from the engine and puts all its state there. This makes hot-reloading the game DLL possible later, Handmade Hero style.

## The main loop (fixed timestep)

```c
// engine/src/core/engine.c  (sketch)
double accumulator = 0.0;
const double dt = 1.0 / app->config.tick_hz;
u64 prev = platform_now();

while (!platform_should_quit()) {
    u64 now = platform_now();
    double frame_time = platform_seconds(now - prev);
    prev = now;
    if (frame_time > 0.25) frame_time = 0.25;     // avoid spiral of death
    accumulator += frame_time;

    platform_poll_events(&input);                 // fills Input struct

    while (accumulator >= dt) {
        app->update(game_mem, &input, (float)dt);
        input_end_tick(&input);                   // clear pressed/released edges
        accumulator -= dt;
    }

    float alpha = (float)(accumulator / dt);
    render_begin(&fb);
    app->render(game_mem, &fb, alpha);
    render_end(&fb);                              // present (vsync = wait for the beam)
}
```

## Key data structures

```c
// types.h
typedef uint8_t  u8;  typedef uint32_t u32; typedef uint64_t u64;
typedef int32_t  i32; typedef float    f32;
typedef struct { f32 x, y; } V2;
typedef struct { f32 x, y, w, h; } Rect;
typedef u32 Color;                       // 0xAARRGGBB

// render.h — software renderer
typedef struct Framebuffer {
    u32 *pixels;
    int  width, height, pitch;           // pitch in pixels
} Framebuffer;

void fb_clear(Framebuffer *fb, Color c);
void fb_pixel(Framebuffer *fb, int x, int y, Color c);
void fb_rect (Framebuffer *fb, int x, int y, int w, int h, Color c);
void fb_line (Framebuffer *fb, int x0, int y0, int x1, int y1, Color c);
void fb_blit (Framebuffer *fb, const Texture *t, Rect src, int dx, int dy);
void fb_text (Framebuffer *fb, const Font *f, int x, int y, const char *s, Color c);

// input.h
typedef enum { BTN_UP, BTN_DOWN, BTN_LEFT, BTN_RIGHT,
               BTN_A, BTN_B, BTN_START, BTN_COUNT } Button;
typedef struct { u8 down, pressed, released; } ButtonState;
typedef struct Input { ButtonState buttons[BTN_COUNT]; V2 mouse; } Input;

static inline int btn_down   (const Input *in, Button b) { return in->buttons[b].down; }
static inline int btn_pressed(const Input *in, Button b) { return in->buttons[b].pressed; }

// memory.h — linear arena, no per-object free
typedef struct Arena { u8 *base; size_t size, used; } Arena;
void *arena_push(Arena *a, size_t bytes, size_t align);
void  arena_reset(Arena *a);
```

Games use logical buttons (`BTN_A`), not keys. The platform layer maps keys and controller buttons to them.

## Module responsibilities

| Module | Owns | Doesn't own |
|---|---|---|
| platform | Window, events, high-resolution timer, presenting pixels, file I/O | Any game concept |
| core/engine | Loop, timestep, wiring modules together | Game rules |
| memory | Arenas, scratch allocators | malloc calls in game code |
| input | Button state and edges, key mapping | Raw SDL events (it reads them through platform) |
| render | Drawing to the framebuffer (software) or GPU (OpenGL) | Deciding *what* to draw |
| assets | Loading and caching textures, fonts, sounds by handle | Gameplay meaning of assets |
| collision | AABB tests, swept AABB, tilemap queries | Response logic (the game decides) |
| scene | Entity storage, JSON save/load | Per-game entity behavior |

## Renderer backends (Phase 4)

Keep the draw API identical and swap the implementation behind a function table:

```c
typedef struct RenderBackend {
    void (*begin)(void);
    void (*clear)(Color c);
    void (*rect)(Rect r, Color c);
    void (*sprite)(TextureHandle t, Rect src, Rect dst);
    void (*text)(FontHandle f, V2 pos, const char *s, Color c);
    void (*end)(void);
} RenderBackend;
```

Phase 4 is done when the platformer runs unchanged on either backend, switched by a command-line flag.

## Conventions

- Prefix public engine functions by module: `fb_`, `input_`, `arena_`, `asset_`.
- Origin is top-left with +Y down in screen space. Pick a world-space convention once and write it down here.
- No globals in game code. All state lives in the game memory block.
- Asserts stay on in debug builds. `RASTER_ASSERT(x)` should break into the debugger.
