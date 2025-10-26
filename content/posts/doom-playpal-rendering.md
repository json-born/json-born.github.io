---
title: "Rendering the Color Palette"
date: 2025-10-26
draft: false
tags: ["C++", "Game Development", "Devlog", "DOOM Engine Series"]
---

Last time I stopped after getting the WAD loader up and running. I could open a WAD file, parse the header, read the directory, and extract individual lumps. The next natural step was to look at one of those lumps in detail. I decided to start with `PLAYPAL`, the color palette that defines every color Doom uses.

## Understanding the PLAYPAL Lump

The `PLAYPAL` lump stores 14 palettes. Each palette has 256 colors, and each color is three bytes: red, green, blue. That means each palette is 256 × 3 = 768 bytes, and the whole lump is 14 × 768 = 10 752 bytes.

I only needed the first palette for now, since it’s the one used most of the time in the game.

## Parsing the Lump

The lump comes into the loader as a vector of bytes. Before touching it, I added a quick size check:

```cpp
if (lumpData.size() % 768 != 0) {
    throw std::runtime_error("PLAYPAL lump data is of incorrect size");
}
```

That confirmed the data was a multiple of 768 bytes, so I knew it contained whole palettes. Then I parsed the first one:

```cpp
std::vector<std::array<uint8_t, 3>> parsePlaypalLump(const std::vector<char>& lumpData) {
    std::vector<std::array<uint8_t, 3>> colors;
    colors.reserve(256);

    for (size_t i = 0; i < 256; ++i) {
        size_t offset = i * 3;
        std::array<uint8_t, 3> color = {
            static_cast<uint8_t>(lumpData[offset]),
            static_cast<uint8_t>(lumpData[offset + 1]),
            static_cast<uint8_t>(lumpData[offset + 2])
        };
        colors.push_back(color);
    }
    return colors;
}
```

I printed out the first few entries and saw values like `(0,0,0)`, `(31,23,11)`, `(23,15,7)` all looking good so far.

## Generating Pixel Data

To visualize the palette, I wanted to arrange those 256 colors in a grid. A 16×16 grid made sense: 256 colors, 16 rows, 16 columns. I wrote a small function to build a pixel buffer.

Each color became a 32×32 block of solid pixels. The math was simple: figure out which row and column a color belongs to, calculate the starting X and Y positions, and fill that square.

```cpp
std::vector<uint8_t> generate256ColorPalettePixels(const std::vector<std::array<uint8_t, 3>>& colors) {
    if (colors.size() != 256) {
        throw std::runtime_error("Palette must contain exactly 256 colors");
    }

    constexpr std::size_t gridWidth = 16;
    constexpr std::size_t cellSize = 32; // doubled for clarity

    const std::size_t imageWidth = gridWidth * cellSize;
    const std::size_t imageHeight = gridWidth * cellSize;

    std::vector<uint8_t> pixelBuffer(imageWidth * imageHeight * 3);

    for (std::size_t i = 0; i < 256; ++i) {
        const auto& color = colors[i];

        const std::size_t row = i / gridWidth;
        const std::size_t col = i % gridWidth;

        const std::size_t startX = col * cellSize;
        const std::size_t startY = row * cellSize;

        for (std::size_t y = 0; y < cellSize; ++y) {
            for (std::size_t x = 0; x < cellSize; ++x) {
                const std::size_t pixelIndex =
                    ((startY + y) * imageWidth + (startX + x)) * 3;

                pixelBuffer[pixelIndex + 0] = color[0];
                pixelBuffer[pixelIndex + 1] = color[1];
                pixelBuffer[pixelIndex + 2] = color[2];
            }
        }
    }
    return pixelBuffer;
}
```

When I printed the first few RGB triplets from that buffer, I saw nothing but `(0,0,0)`, which made sense: the first palette color is black, and the first 32×32 block of pixels are all the same.

## Rendering with SDL3

Once the pixel data looked correct, I used SDL3 to turn it into a texture and draw it on screen.

```cpp
SDL_Texture* createTextureFromRGB(SDL_Renderer* renderer,
                                  const std::vector<uint8_t>& pixels,
                                  int width, int height) {
    SDL_Texture* texture = SDL_CreateTexture(
        renderer,
        SDL_PIXELFORMAT_RGB24,
        SDL_TEXTUREACCESS_STATIC,
        width,
        height
    );
    SDL_UpdateTexture(texture, nullptr, pixels.data(), width * 3);
    return texture;
}
```

The render loop was straightforward:

```cpp
void renderFrame(SDL_Renderer* renderer, SDL_Texture* texture) {
    SDL_RenderClear(renderer);
    SDL_RenderTexture(renderer, texture, nullptr, nullptr);
    SDL_RenderPresent(renderer);
}
```

In `main()` I created a 512×512 window, generated the pixel buffer from the parsed colors, created a texture, and drew it.

## The Result

When I ran it, I got a perfect grid: rows of browns, greys, and greens at the top, bright reds and blues further down. LFG!

![Doom PLAYPAL Palette](/images/palette.png)

## Next Steps

Now that I can see the palette, the next logical step is to apply it, probably by loading a sprite, or any other lump that stores indexed color data and translate it through this palette to pixels on screen.

This small piece is the first real link between the data structures inside the WAD and what actually appears on screen.

you can check out the [full commit](https://github.com/json-born/DoomClone/commit/a22d27d79988f5324f3cd009f456a485fc3503a4) here.
