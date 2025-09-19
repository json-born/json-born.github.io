---
title: "Building the DOOM Engine from Scratch: A Learning Journey"
date: 2025-09-19T18:50:00+01:00
draft: false
tags: ["C++", "Game Development", "Devlog", "DOOM Engine Series"]
cover:
  image: "/images/doom-logo.webp"
  alt: "DOOM Logo"
  caption: "Exploring the legendary DOOM engine"
---

I've been reading Fabien Sanglard's [Game Engine Black Book: DOOM](https://fabiensanglard.net/gebbdoom/), and it's honestly blown my mind. The book breaks down all the clever tricks behind id Software's masterpiece and makes you realize just how insanely talented those early developers were. Reading about BSP trees has me wanting to build something myself.

So that's exactly what I'm going to do.

I'm building an id Tech 1 engine recreation project in C++. This isn't about making a game, it's about learning by doing, understanding how this stuff actually works, and appreciating the engineering genius that Sanglard documents so well in his book. I want to implement the core systems from scratch and really get to grips with how they work together.

The plan is deliberately old-school: pure software rendering using [SDL](https://www.libsdl.org/) just as a display window. No modern GPU tricks, no shortcuts. I'll be calculating every single pixel in a framebuffer and pushing that each frame, just like the original engine did back in 1993.

## The approach

After a couple of evenings chatting with GPT, I've come up with a rough plan of action:

### Foundations

Setting up the C++20 project structure, getting SDL running, and implementing the basic framebuffer system. You can read about this below, aside from some tooling issues, it was pretty straightforward.

### WAD Loader

This is where things get interesting. I need to parse DOOM's WAD files and extract the maps, textures, and sprites. The WAD format is beautifully simple but requires careful handling to get right.

### BSP & Software Rendering

This is the big one. Implementing the Binary Space Partitioning system for proper visibility determination, then building the software renderer that draws walls, floors, and ceilings pixel by pixel. Texture mapping in particular looks like it's going to be a real pain.

### Game Features

Player movement, input handling, sprite rendering, lighting effects, and all the other systems that make DOOM feel like DOOM.

I'm not trying to recreate every feature or optimize for modern hardware. I just want to understand the clever solutions that Carmack, Romero, and the rest of the id team came up with when working with the limitations of early 90s PCs.

This feels like the perfect bridge between my web development background and wanting to make games eventually. It's low-level enough to teach me proper C++ and graphics programming, but focused enough that I won't get completely lost in scope creep.

## Laying the foundations

I began by setting up a new project in JetBrains Rider, thinking I could use CMake. After some confusion, I realized that Rider doesn’t support CMake for native C++ projects out of the box — a frustrating discovery, but it quickly led me to switch to Visual Studio, which has native support for C++ and integrates seamlessly with vcpkg, which should hopefully make dependency management less of a headache.

For those unfamiliar, [SDL](https://www.libsdl.org/) (Simple DirectMedia Layer) is a lightweight, cross-platform library that handles graphics, input, audio, and more. In this project, I'm using it purely as a window and display backend — all the actual rendering will happen in software, which gives me full control over every pixel, just like the original DOOM engine did.

Once the setup was complete, I wrote some boilerplate for initializing SDL creating a window and drawing some stuff into it:

```cpp
#include <SDL3/SDL.h>
#include <iostream>

int main() {
    if (SDL_Init(SDL_INIT_VIDEO) != true) {
        std::cerr << "SDL_Init Error: " << SDL_GetError() << std::endl;
        return 1;
    }

    SDL_Window* window = SDL_CreateWindow(
        "SDL3 Test Window",
        800,
        600,
        0
    );

    if (!window) {
        std::cerr << "SDL_CreateWindow Error: " << SDL_GetError() << std::endl;
        SDL_Quit();
        return 1;
    }

    SDL_Renderer* renderer = SDL_CreateRenderer(window, nullptr);

    if (!renderer) {
        std::cerr << "SDL_CreateRenderer Error: " << SDL_GetError() << std::endl;
        SDL_DestroyWindow(window);
        SDL_Quit();
        return 1;
    }

    bool running = true;
    SDL_Event event;

    while (running) {
        while (SDL_PollEvent(&event)) {
            if (event.type == SDL_EVENT_QUIT) {
                running = false;
            }
        }

        SDL_SetRenderDrawColor(renderer, 0, 0, 0, 255);
        SDL_RenderClear(renderer);

        SDL_SetRenderDrawColor(renderer, 255, 0, 0, 255);
        SDL_RenderPoint(renderer, 400, 300);

        SDL_RenderPresent(renderer);
    }

    SDL_DestroyRenderer(renderer);
    SDL_DestroyWindow(window);
    SDL_Quit();

    return 0;
}
```

Seeing it compile cleanly was a relief after banging my head against google for an hour trying to work out the best tools to use. Heres the end result:

![SDL3 Test Window](/images/test-window.png)

It aint much, but it's a start...
