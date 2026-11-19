# Raster Engine

A small 2D game engine written in C on top of SDL3. It starts with a software renderer and moves to OpenGL later.

The project builds the engine out of real games. Each game is built on the engine, and whatever survives across games stays in the engine.

> "Raster" is a placeholder name. Rename it to whatever you like.

## Goals

- Learn engine architecture from the bottom up: framebuffer, loop, input, assets, rendering.
- Keep the engine and game code separate, so the same engine runs several different games.
- Keep a C64 mindset: know where every byte lives, and avoid hidden allocations.
- End with a small level editor built with Dear ImGui.

## Docs

| File | What's in it |
|---|---|
| [SETUP.md](SETUP.md) | Toolchain, dependencies, first build, and a minimal "pixels on screen" program |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Folder layout, module responsibilities, key interfaces and data structures |
| [ROADMAP.md](ROADMAP.md) | Phased milestones with checklists and "done when" criteria |

## Folder layout (summary)

```
raster/
├── CMakeLists.txt
├── README.md
├── docs/                  # ARCHITECTURE.md, ROADMAP.md, SETUP.md, devlog.md
├── engine/
│   ├── include/raster/    # public engine headers (what games are allowed to see)
│   └── src/               # engine implementation, split by module
├── games/
│   ├── pong/              # Milestone 1
│   └── platformer/        # Milestone 2
├── tools/
│   └── editor/            # Milestone 4 (Dear ImGui level editor)
├── assets/                # shared test assets (fonts, sprites, sounds)
└── third_party/           # stb_image, miniaudio, cJSON, etc. (vendored)
```

See ARCHITECTURE.md for the full tree.

## Ground rules

1. **Game code includes only `engine/include/raster/*.h`.** It never includes SDL directly.
2. **The engine never knows which game is running.** Games plug in through a small callback table.
3. **Build a feature in a game first.** Move it into the engine only once a second game needs it.
4. **Use a fixed timestep for simulation** and a variable rate for rendering.
5. **Keep a devlog** (`docs/devlog.md`). Write a few lines per session: what you did, what broke, what's next.
