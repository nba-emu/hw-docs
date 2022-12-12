# Layer compositing

Compositing of the background, sprite and backdrop layers begins after a wait period of 46 cycles.
The PPU then composites an output pixel every four cycles. The last pixel will be completed on cycle #1006 (`46 + 4 * 240 = 1008`).
It does this during all visible scanlines (0 to 159) and always composites pixels for the current scanline.

For each output pixel the top two opaque layers for that pixel are extracted. Then up to two Palette RAM (PRAM) accesses are performed, to resolve palette indices from the background or sprite layers to 16-bit colors or to get the backdrop colour. The first PRAM access  resolves the colour for the top-most layer and the second PRAM access the colour for the second top-most layer.

For both layers, there are distinctive conditions that decide whether the respective PRAM access will be performed or not.

## Top-most layer condition

- not (PPU is in Mode 3 and the layer is BG2)

## Second top-most layer condition

- not (PPU is in Mode 3 and the layer is BG2)
- Alpha-blending is enabled
- The top-most layer is selected as the first blend target
- The second top-most layer selected as the second blend target

Notice that for both layers the access is not performed when the PPU is in Mode 3 and the layer is BG2. This simply is because the BG2 bitmap in Mode 3 does not use palettes and fetches 16-bit colors directly from VRAM.

## PRAM access pattern

```
A-B- A-B- A-B- A-B- ...

- = no fetch
A = top-most layer PRAM fetch
B = second top-most layer PRAM fetch
```