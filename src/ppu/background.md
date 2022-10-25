# Background rendering

The PPU renders background pixels during all visible scanlines (0 to 159). In each scanline it renders all pixels for the current scanline, meaning that unlike sprite rendering all background rendering happens in the same scanline.

For every background (BG0 - BG3) there is a wait/idle period for a number of clock cycles before the PPU begins rendering that background according to the VRAM access patterns laid out below. Usually the length of the wait period is 33 clock cycles, however it may be shorter for text-mode backgrounds which use horizontal scrolling, as described in more detail in the [Mode 0](#mode-0-bg0-3-text-mode) section.

Once the wait period for a given background has passed, the PPU renders pixels until it enters the horizontal blanking period (h-blank) (cycle #1007 appears to be the last cycle that may fetch data).
If all visible 240 pixels have been rendered, before the PPU reaches the h-blank period, it does not stop rendering pixels early. It renders more pixels (which are discarded) until the h-blank  period.

The hardware fetches **on average** one background pixel (for every enabled background) every four cycles. For affine tilemap and bitmap modes this is the true rate at which pixels are fetched, however for text-mode backgrounds the access pattern is slightly more complicated.

## Legend

```
- = no fetch
M = fetch map entry (8-bit for affine maps or 16-bit for text-mode maps)
T = fetch tile data
B = fetch bitmap (8-bit palette index or 16-bit color)
```

## Mode 0 (BG0 - BG3 text mode)

Text-mode backgrounds are rendered tile-by-tile. Within 32 cycles a single tilemap-entry is rendered for each enabled background, starting with the left-most visible tilemap-entry. Normally 30[^1] tilemap-entries are rendered per scanline. However when sub-tile[^2] horizontal scrolling is used then an additional tilemap-entry needs to be rendered. Notice that while in this case the first and last tile are partially off-screen, hardware still fetches the full data for all eight pixels. The off-screen pixels simply are discarded.

Normally rendering for text-mode backgrounds starts at 33 cycles into the scanline. When a background has sub-tile scrolling[^2], then its start time is shifted closer to the start of the scanline by four cycles per sub-tile scrolled pixel. That is to say, the start time can be calculated from the **BG**[x]**HOFS** register: `33 - 4 * (BG[x]HOFS mod 8)`. This is presumably done to ensure that the *actually* visible pixels are always available in time to be consumed by the composite step.

To render a tilemap-entry the relevant 16-bit descriptor is first fetched from the tilemap. The hardware then performs another two (4BPP) or four (8BPP) 16-bit VRAM fetches to read a full eight pixel line from the tile data.

### 4BPP access patterns

```
BG0: M--- T--- ---- ---- ---- T--- ---- ----
BG1: -M-- -T-- ---- ---- ---- -T-- ---- ----
BG2: --M- --T- ---- ---- ---- --T- ---- ----
BG3: ---M ---T ---- ---- ---- ---T ---- ----
```

### 8BPP access patterns

```
BG0: M--- T--- ---- T--- ---- T--- ---- T---
BG1: -M-- -T-- ---- -T-- ---- -T-- ---- -T--
BG2: --M- --T- ---- --T- ---- --T- ---- --T-
BG3: ---M ---T ---- ---T ---- ---T ---- ---T
```

## Mode 1 (BG0 - BG1 text mode; BG2 affine tilemap)

Presumably just works like Mode 0 for BG0 and BG1 and like Mode 2 for BG2.

## Mode 2 (BG2 - BG3 affine tilemap)

Affine tilemap backgrounds are rendered pixel-by-pixel, starting with the left-most screen pixel.

Every four cycles one BG2 and one BG3 pixel is fetched. In the first cycle the respective BG3 tilemap-entry is fetched. In the second cycle the corresponding tile data is fetched.
The third and fourth cycles repeat the same process for BG2.

Even if the current tilemap pixel coordinate is out-of-bounds and wraparound is disabled in **BG**[x]**CNT**, the pixel will still be fetched (the fetched data will be discarded).

```
BG2: --MT --MT --MT --MT --MT --MT --MT --MT
BG3: MT-- MT-- MT-- MT-- MT-- MT-- MT-- MT--
```

## Mode 3 - 5 (BG2 affine bitmap modes)

The bitmap background (BG2) is rendered pixel-by-pixel, starting with the left-most screen pixel.

Every four cycles a single pixel is fetched from the BG2 bitmap.
This is the case even if the current bitmap pixel coordinate is out-of-bounds of the bitmap (the fetched data will be discarded).

```
BG2: ---B ---B ---B ---B ---B ---B ---B ---B
```

[^1]: 240 pixels / 8 pixels per tile

[^2]: this means that **BG**[x]**HOFS** is not divisible by eight.