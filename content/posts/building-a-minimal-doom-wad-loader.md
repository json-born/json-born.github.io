---
title: "Building a Minimal Doom WAD Loader in C++"
date: 2025-10-24
tags: ["C++", "Game Development", "Devlog", "DOOM Engine Series"]
draft: false
---

I wanted to learn how Doom’s WAD files work, so I wrote a simple loader in C++.
My goal was just to open a WAD, read its contents, and understand how it’s structured.

---

## WAD File Overview

A WAD file (“Where’s All the Data”) is Doom’s archive format. It stores maps, textures, sprites, palettes, and other data.
The layout is straightforward:

``` python
┌──────────────────────────────────────────┐
│ Header (12 bytes)                        │
│ ├── 4 bytes: "IWAD" or "PWAD"            │
│ ├── 4 bytes: Lump count (little-endian)  │
│ └── 4 bytes: Directory offset (LE32)     │
│                                          │
│ Lump data...                             │
│                                          │
│ Directory (16 bytes per entry):          │
│ ├── 4 bytes: Lump position               │
│ ├── 4 bytes: Lump size                   │
│ └── 8 bytes: Lump name (null-padded)     │
└──────────────────────────────────────────┘
```

Each WAD starts with a header that tells me how many “lumps” there are and where the directory is.
The directory is a list of fixed-size entries that point to each lump’s position and size in the file.

---

## Data Structures

I used three simple structs to represent the data:

```cpp
struct WadHeader {
    std::string id;
    uint32_t lumpCount = 0;
    uint32_t directoryOffset = 0;
};

struct LumpEntry {
    std::string name;
    uint32_t position = 0;
    uint32_t size = 0;
};

using LumpDirectoryMap = std::unordered_map<std::string, LumpEntry>;

struct Wad {
    WadHeader header;
    LumpDirectoryMap directoryMap;
    std::vector<char> data;
};
```

- `WadHeader` represents the 12-byte header at the start of the file.
- `LumpEntry` represents one entry in the directory.
- `directoryMap` lets me look up lumps by name.
- `data` holds the entire WAD file in memory.

Keeping the full file in memory simplifies things, I can slice directly from it when I need lump data, without reopening or seeking through the file again.

---

## Reading the File into Memory

To make everything predictable, I wrote a helper that reads the entire WAD into memory.
It also resets the stream position afterward so subsequent reads start at the beginning.

```cpp
std::vector<char> readData(std::ifstream& fileStream) {
    fileStream.seekg(0, std::ios::end);
    uint32_t size = static_cast<uint32_t>(fileStream.tellg());
    std::vector<char> buffer(size);

    fileStream.seekg(0, std::ios::beg);
    fileStream.read(buffer.data(), size);
    fileStream.seekg(0, std::ios::beg);

    return buffer;
}
```

This function does three things:

1. Gets the total size by seeking to the end and using `tellg()`.
2. Allocates a buffer large enough to hold the whole file and reads it.
3. Seeks back to the beginning for the next read.

Resetting the cursor is important, otherwise the next read (for the header) will start at the end of the file and return garbage.

---

## Reading the Header

The header is always the first 12 bytes. I read it into a small fixed-size buffer and then extract the values.

```cpp
WadHeader readHeader(std::ifstream& fileStream) {
    std::array<char, 12> buffer;
    fileStream.read(buffer.data(), static_cast<std::streamsize>(buffer.size()));

    WadHeader header;
    header.id = std::string(buffer.data(), 4);
    header.lumpCount = readLE32(buffer, 4);
    header.directoryOffset = readLE32(buffer, 8);

    return header;
}
```

The only tricky part is that the integers are stored in **little-endian** order.
That means the least significant byte comes first, so I can’t just `reinterpret_cast` them directly.

---

## Converting Little-Endian Integers

This helper converts four bytes starting at a given offset into a `uint32_t`:

```cpp
uint32_t readLE32(std::span<const char> buffer, size_t offset) {
    return static_cast<uint32_t>(
        static_cast<uint8_t>(buffer[offset]) |
        (static_cast<uint8_t>(buffer[offset + 1]) << 8) |
        (static_cast<uint8_t>(buffer[offset + 2]) << 16) |
        (static_cast<uint8_t>(buffer[offset + 3]) << 24)
    );
}
```

Each byte is read, cast to an unsigned value, shifted into the correct position, and then combined with bitwise OR.
For example, if the bytes in the file are:

``` python
34 12 00 00
```

The result is `0x00001234` (4660 in decimal).

The shifting is just a way to move each byte into the right position in the 32-bit number:

- The first byte stays where it is.
- The second byte moves up 8 bits.
- The third moves up 16 bits.
- The fourth moves up 24 bits.

Once I saw it that way, the “bit-shift magic” just became a manual reconstruction of a 32-bit integer from its four component bytes.

---

## Reading the Directory

After reading the header, I know:

- how many lumps the file contains (`lumpCount`), and
- where the directory starts (`directoryOffset`).

Each directory entry is 16 bytes, so the directory’s total size is `lumpCount * 16`.

The code that reads it looks like this:

```cpp
LumpDirectoryMap readDirectory(std::ifstream& fileStream, uint32_t offset, uint32_t size) {
    std::vector<char> buffer(size);
    LumpDirectoryMap lumpMap;
    lumpMap.reserve(size / 16);

    fileStream.seekg(offset, std::ios::beg);
    fileStream.read(buffer.data(), static_cast<std::streamsize>(buffer.size()));

    for (uint32_t i = 0; i + 16 <= size; i += 16) {
        uint32_t lumpPosition = readLE32(buffer, i);
        uint32_t lumpSize = readLE32(buffer, i + 4);
        std::string lumpName(buffer.data() + i + 8, 8);

        // Remove trailing null characters
        size_t endPos = 7;
        while (endPos > 0 && lumpName[endPos] == '\0') {
            --endPos;
        }
        lumpName.resize(endPos + 1);

        lumpMap[lumpName] = { lumpName, lumpPosition, lumpSize };
    }

    return lumpMap;
}
```

This function reads the entire directory block at once, then iterates over it in 16-byte chunks.
For each entry, it extracts:

- the lump’s position in the file,
- the size of that lump in bytes, and
- its name (up to eight bytes).

---

## Trimming Trailing Nulls

WAD lump names are fixed at 8 bytes, and any unused bytes are filled with null characters (`\0`).
If I copy those bytes directly into a string, I end up with names like `"PLAYPAL\0\0"`.
These don’t compare equal to `"PLAYPAL"`, which breaks lookups later.

The first approach I tried was:

```cpp
auto pos = lumpName.find('\0');
if (pos != std::string::npos)
    lumpName.erase(pos);
```

This looks reasonable, but it has a subtle problem.
If a lump name starts with a null (for example, because of a misread offset), `find('\0')` returns 0, and `erase(0)` deletes the whole string.
That means I insert empty strings into the map, which overwrite each other and result in an empty or broken directory.

The safer way is to trim **only trailing nulls**:

```cpp
size_t endPos = 7;
while (endPos > 0 && lumpName[endPos] == '\0') {
    --endPos;
}
lumpName.resize(endPos + 1);
```

This leaves the meaningful characters intact and removes only padding at the end.

For example:

- `"PLAYPAL\0\0"` → `"PLAYPAL"`
- `"E1M1\0\0\0\0"` → `"E1M1"`
- `"\0\0\0\0\0\0\0\0"` → `""` (empty, which is fine)

That small fix made name lookups consistent and reliable.

---

## Loading the WAD

With the helpers written, the main loader is short and clear:

```cpp
Wad loadWad(std::string path) {
    std::ifstream fileStream(path, std::ios::binary);
    Wad wad;

    if (!fileStream.is_open()) {
        throw std::runtime_error("Failed to open WAD file: " + path);
    }

    wad.data = readData(fileStream);
    wad.header = readHeader(fileStream);

    uint32_t directorySize = wad.header.lumpCount * 16;
    wad.directoryMap = readDirectory(fileStream, wad.header.directoryOffset, directorySize);

    return wad;
}
```

The order here matters:

1. I read the full file into memory (`wad.data`), so I can slice from it later.
2. I read the header to get the lump count and directory location.
3. I read the directory and build a map of lump names to their positions and sizes.

The loader returns a `Wad` struct that contains everything I need to explore the file.

---

## Reading a Lump

Once the file is loaded, reading a lump is just copying a slice from the in-memory data buffer:

```cpp
std::vector<char> readLump(const Wad& wad, uint32_t position, uint32_t size) {
    const char* start = wad.data.data() + position;
    const char* end = start + size;
    return std::vector<char>(start, end);
}
```

Usage is simple:

```cpp
auto entry = wad.directoryMap.at("PLAYPAL");
auto bytes = readLump(wad, entry.position, entry.size);
```

This gives me the raw bytes for any lump.
For example, I can use this to read the PLAYPAL color palette or map geometry data later on.

---

## Testing the Loader

The `main()` file doesn’t need to be complex. It just needs to open a WAD, print some information, and confirm that lump extraction works.
Here’s how mine ended up:

```cpp
int main() {
    try {
        Wad wad = loadWad("doom.wad");  // replace with your WAD file path

        // Print header
        std::cout << "WAD ID: " << wad.header.id << "\n";
        std::cout << "Lump count: " << wad.header.lumpCount << "\n";
        std::cout << "Directory offset: " << wad.header.directoryOffset << "\n\n";

        // Print all lumps
        std::cout << "Directory entries:\n";
        for (const auto& pair : wad.directoryMap) {
            const LumpDirectoryEntry& entry = pair.second;

            std::cout << "Name: " << entry.name
                      << ", Position: " << entry.position
                      << ", Size: " << entry.size
                      << "\n";
        }

        // Example: read a known lump, e.g., PLAYPAL
        const std::string lumpName = "PLAYPAL";

        auto it = wad.directoryMap.find(lumpName);
        if (it == wad.directoryMap.end()) {
            std::cerr << "Lump not found: " << lumpName << "\n";
            return 1;
        }

        const auto& entry = it->second;
        std::vector<char> lumpData = readLump(wad, entry.position, entry.size);

        std::cout << "Lump: " << lumpName
                  << ", Size: " << lumpData.size() << "\n";

        // Dump first 32 bytes in hex for inspection
        std::cout << "First 32 bytes:\n";
        for (size_t i = 0; i < lumpData.size() && i < 32; ++i) {
            std::cout << std::hex << std::setw(2) << std::setfill('0')
                      << static_cast<unsigned int>(static_cast<unsigned char>(lumpData[i])) << " ";
            if ((i + 1) % 16 == 0) std::cout << "\n";
        }
        std::cout << std::dec << "\n"; // reset to decimal

    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    }

    return 0;
}
```

This version does a few useful things:

- Uses a `try/catch` block to handle I/O errors cleanly.
- Prints the WAD header information for quick verification.
- Lists all lumps and their positions and sizes.
- Looks up a specific lump (in this case, `PLAYPAL`) and reads it into a vector.
- Prints the first 32 bytes in hexadecimal: a quick sanity check that I’m reading real data and not zeroes or garbage.

It’s not polished, but it’s practical and also very temporary.

---

## Summary

The loader now:

- Reads the entire WAD file safely into memory.
- Parses the header and directory correctly using manual little-endian reads.
- Handles padded 8-character names without breaking map keys.
- Provides direct access to lump data by name.

From here I can start building on it: parsing palettes, maps, textures, or even writing tools that inspect and modify WAD contents.
The code is intentionally simple so it’s easy to extend.

---

For a deeper dive into the code, you can check out the GitHub [commit](https://github.com/json-born/DoomClone/commit/f78579ed64b2d056565701f00c970036f25d2483).
