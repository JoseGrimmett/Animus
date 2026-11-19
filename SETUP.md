# Setup

## Toolchain

| Tool | Notes |
|---|---|
| C compiler | MSVC (Visual Studio 2022 Community), clang, or gcc. Target **C11**. |
| CMake ≥ 3.20 | Build system generator |
| Git | For FetchContent and version control |
| Editor | VS Code (with the C/C++ and CMake Tools extensions) or Visual Studio |
| Debugger | Visual Studio's debugger on Windows, or gdb/lldb. Use it early and often. |

## Dependencies

| Library | Purpose | How it's pulled in |
|---|---|---|
| SDL3 | Window, input, timing, presenting the framebuffer | CMake `FetchContent` |
| stb_image.h | PNG loading | Vendored single header in `third_party/` |
| stb_truetype.h | Font rasterizing (later) | Vendored |
| miniaudio.h | Audio (Phase 3) | Vendored |
| cJSON | Level and scene files (Phase 3) | Vendored |
| Dear ImGui (cimgui) | Editor UI (Phase 5) | Vendored or FetchContent |
| glad | OpenGL loader (Phase 4) | Generated and vendored |

Only add a dependency in the phase that needs it.

## Root `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)
project(raster C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

include(FetchContent)
FetchContent_Declare(
    SDL3
    GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
    GIT_TAG        release-3.2.0   # bump to the latest 3.x release tag
)
FetchContent_MakeAvailable(SDL3)

add_subdirectory(engine)
add_subdirectory(games/pong)
# add_subdirectory(games/platformer)
# add_subdirectory(tools/editor)
```

## `engine/CMakeLists.txt`

```cmake
file(GLOB_RECURSE ENGINE_SRC CONFIGURE_DEPENDS src/*.c)
add_library(raster STATIC ${ENGINE_SRC})
target_include_directories(raster
    PUBLIC  include
    PRIVATE src ${CMAKE_SOURCE_DIR}/third_party)
target_link_libraries(raster PRIVATE SDL3::SDL3)

if (MSVC)
    target_compile_options(raster PRIVATE /W4)
else()
    target_compile_options(raster PRIVATE -Wall -Wextra -Wpedantic)
endif()
```

## `games/pong/CMakeLists.txt`

```cmake
add_executable(pong main.c pong.c)
target_link_libraries(pong PRIVATE raster SDL3::SDL3)
```

SDL is linked here only so SDL3's `main` wrapper works on Windows. The game code still never includes SDL headers.

## Build

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
./build/games/pong/pong        # Windows: build\games\pong\Debug\pong.exe
```

On Windows, copy `SDL3.dll` next to the executable, or add a post-build copy step.

## Smoke test: pixels on screen

Before building any engine structure, get this throwaway program running. It's your "POKE to screen RAM" moment. Put it in `scratch/hello_pixels.c`.

```c
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>
#include <stdint.h>
#include <stdlib.h>

#define W 320
#define H 200

int main(int argc, char **argv) {
    (void)argc; (void)argv;
    if (!SDL_Init(SDL_INIT_VIDEO)) return 1;

    SDL_Window   *win = SDL_CreateWindow("hello pixels", W * 3, H * 3, 0);
    SDL_Renderer *ren = SDL_CreateRenderer(win, NULL);
    SDL_Texture  *tex = SDL_CreateTexture(ren, SDL_PIXELFORMAT_ARGB8888,
                                          SDL_TEXTUREACCESS_STREAMING, W, H);
    SDL_SetTextureScaleMode(tex, SDL_SCALEMODE_NEAREST);   // crisp pixels

    uint32_t *pixels = malloc(W * H * sizeof(uint32_t));
    int running = 1, frame = 0;

    while (running) {
        SDL_Event e;
        while (SDL_PollEvent(&e))
            if (e.type == SDL_EVENT_QUIT) running = 0;

        for (int y = 0; y < H; y++)
            for (int x = 0; x < W; x++)
                pixels[y * W + x] = 0xFF000000 | ((x + frame) & 0xFF) << 16
                                               | ((y + frame) & 0xFF) << 8;

        SDL_UpdateTexture(tex, NULL, pixels, W * sizeof(uint32_t));
        SDL_RenderTexture(ren, tex, NULL, NULL);
        SDL_RenderPresent(ren);
        frame++;
    }

    free(pixels);
    SDL_Quit();
    return 0;
}
```

**Done when:** a scrolling color gradient appears at 320×200, scaled 3× with no blur.
