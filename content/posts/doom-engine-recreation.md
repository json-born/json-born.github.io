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

I've been reading Fabien Sanglard's [*Game Engine Black Book: DOOM*](https://fabiensanglard.net/gebbdoom/), and honestly, it has blown my mind. The book breaks down all the clever tricks behind id Software's masterpiece and makes you appreciate just how insanely talented those early developers were. Reading about BSP trees, sub-pixel accuracy and perspective-correct texture mapping has me itching to build something myself.

So that is exactly what I am going to do.

I am starting an **id Tech 1 engine recreation project in C++**. This is not about making a game. It is about learning by doing, understanding how everything works under the hood, and gaining a deeper appreciation for the engineering genius Sanglard documents so well. I want to implement the core systems from scratch and see firsthand how they all fit together.

The plan is deliberately old school: **pure software rendering using [SDL](https://www.libsdl.org/)** as a display window. No modern GPU tricks, no shortcuts. Every single pixel will be calculated in a framebuffer and pushed to the screen each frame, just like the original engine did back in 1993.

---

## The Approach

After a couple of hours thinking the problem through, and with a little help from GPT-5, I sketched out a rough plan:

### Foundations

Set up a C++20 project, get SDL running, and draw something. Aside from some tooling quirks, this part was surprisingly straightforward.

### WAD Loader

This is where things start to get interesting. DOOM stores all of its game data, including maps, textures, sprites, sounds, and more, inside WAD files. WAD stands for 'Where’s All the Data' and is essentially a simple archive format with a structured directory of all the resources the game needs. To bring the engine to life, I will need to parse these files and extract the maps, textures, and sprites.

### BSP and Software Rendering

This is the big challenge. DOOM’s maps in the WAD file already provide all the geometry in terms of sectors, lines, and vertices, but to render the world efficiently I will need to build a Binary Space Partitioning system to determine what is visible at any moment. On top of that, the software renderer will draw walls, floors, and ceilings pixel by pixel, and getting the texture mapping right promises to be a real headache.

### Game Features

Finally, player movement, input handling, sprite rendering, lighting, and the other systems that make DOOM feel like DOOM. I'm keeping my goals for this section open-ended for now.

I am not trying to recreate every feature or optimize for modern hardware. I just want to understand the clever solutions Carmack, Romero, and the id team came up with given the limitations of early 1990s PCs.

This project feels like the perfect bridge between my web development background and my eventual goal of making games. It is low level enough to teach proper C++ and graphics programming but focused enough that I will not get lost in scope creep.

---

## Laying the Foundations

At the start, I set up a new C++ project in JetBrains Rider. At this point, I was unfamiliar with most IDEs for C++ development, so I figured it did not matter much which one I used, and I defaulted to Rider after reading that it was becoming quite popular for game development. I planned to use CMake for build management due to it's portability, but after wrestling with the configuration for a while, I realized that Rider does not fully support CMake for native C++ projects out of the box.

I quickly realized that none of this mattered at the moment and that I was wasting time. I switched to Visual Studio and MSBuild instead, which integrates cleanly with vcpkg for dependency management. This made it much easier to get a working project structure up and running, and I can always port the project to CMake later if I want to.

With the environment ready, I moved on to the first rendering test. For this, I used [SDL](https://www.libsdl.org/).

For those unfamiliar, [SDL](https://www.libsdl.org/) (Simple DirectMedia Layer) is a lightweight, cross platform library for graphics, input, audio, and more. In this project, I am using it purely as a window and display backend. All the actual rendering will happen in software, giving me full control over every pixel just like the original DOOM engine.

Once the setup was complete, I wrote some boilerplate to initialize SDL, create a window, and draw a simple pixel:

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

Seeing it compile cleanly was a relief after an hour of wrestling with tools. Here is the result:

![SDL3 Test Window](/images/test-window.png)

It's not much, but it is a start.
