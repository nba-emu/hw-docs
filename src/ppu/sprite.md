# Sprite rendering

The PPU renders sprites during all visible scanlines (0 to 159) and also during the last scanline (227) of the vertical blanking period.
Sprites are rendered one scanline ahead.
This means sprite rendering for line 0 starts during line 227 and rendering for line 1 in line 0 and so on.

Sprite rendering for the current scanline starts at cycle #42 of the previous scanline and continues either until the horizontal blanking period of that previous scanline (if **DISPCNT**.bit5 = 1)
or until cycle #42 of the current scanline (if **DISPCNT**.bit5 = 0).

The sprites are rendered in order from the lowest OAM entry (OAM #0) to the highest entry (OAM #127).
The rendering is done with two pipeline stages: an OAM attribute/matrix fetch stage and a VRAM pixel fetch stage.
Both stages only access OAM/VRAM every two cycles, every odd cycle does not access any memory.

## OAM fetch stage

For every OAM entry the OAM fetch stage first fetches sprite attributes #0 and #1 (single 32-bit read) in cycle #0.
It then decides based on the attributes if the sprite is enabled and vertically intersects the scanline that is rendered.
If this is the case then it fetches attribute #2 in cycle #2 and, if the sprite is affine, the four 2x2 matrix components in cycles #4, #6, #8 and #10 [^1].

Once this pipeline stage has identified an OAM entry that should be rendered, it activates the VRAM fetch stage.
The OAM fetch stage is stalled for all active VRAM fetch stage cycles except for the first and the next-to-last cycles.
That is enough to prefetch OAM attributes #0 and #1 for the next OAM entry and either attribute #2 for the next OAM entry or attributes #0 and #1 for the OAM entry after.

## VRAM fetch stage

The VRAM fetch stage fetches tile data for every sprite pixel in the rendered scanline.

For regular sprites `width` 16-bit VRAM accesses are performed (one access every two cycles).
With each access two pixels are rendered (even for 4BPP tile data).

For affine sprites this stage does not perform a VRAM access during the first two render cycles.
It then performs `2 * width` VRAM accesses (one access every two cycles).
With each access a single pixel is rendered.
Notice that using the 'double area' feature doubles the number of VRAM accesses.

## Examples

### Legend 

```
- = no fetch
A01 = fetch OBJ Attribute 0 and Attribute 1 (single 32-bit OAM read)
A2 = fetch OBJ Attribute 2
PA = fetch matrix entry #0
PB = fetch matrix entry #1
PC = fetch matrix entry #2
PD = fetch matrix entry #3
V = fetch VRAM tile data
```

### OAM #0 - OAM #3 rendered, non-affine and 8 pixels wide, OAM #4 - OAM #127 disabled/culled

| Cycle | OAM #0 | OAM #1 | OAM #2 | OAM #3 | OAM #4 | OAM #5 |
|-------|:------:|:------:|:------:|:------:|:------:|:------:|
| 0     |  A01   |   -    |   -    |   -    |   -    |   -    |
| 2     |   A2   |   -    |   -    |   -    |   -    |   -    |
| 4     |   V    |  A01   |   -    |   -    |   -    |   -    |
| 6     |   V    |   -    |   -    |   -    |   -    |   -    |
| 8     |   V    |   -    |   -    |   -    |   -    |   -    |
| 10    |   V    |   A2   |   -    |   -    |   -    |   -    |
| 12    |   -    |   V    |  A01   |   -    |   -    |   -    |
| 14    |   -    |   V    |   -    |   -    |   -    |   -    |
| 16    |   -    |   V    |   -    |   -    |   -    |   -    |
| 18    |   -    |   V    |   A2   |   -    |   -    |   -    |
| 20    |   -    |   -    |   V    |  A01   |   -    |   -    |
| 22    |   -    |   -    |   V    |   -    |   -    |   -    |
| 24    |   -    |   -    |   V    |   -    |   -    |   -    |
| 26    |   -    |   -    |   V    |   A2   |   -    |   -    |
| 28    |   -    |   -    |   -    |   V    |  A01   |   -    |
| 30    |   -    |   -    |   -    |   V    |   -    |   -    |
| 32    |   -    |   -    |   -    |   V    |   -    |   -    |
| 34    |   -    |   -    |   -    |   V    |   -    |  A01   |

### OAM #0 - OAM #3 enabled, affine and 8 pixels wide, OAM #4 - OAM #128 disabled/culled

| Cycle | OAM #0 | OAM #1 | OAM #2 | OAM #3 | OAM #4 | OAM #5 |
|-------|:------:|:------:|:------:|:------:|:------:|:------:|
| 0     |  A01   |   -    |   -    |   -    |   -    |   -    |
| 2     |   A2   |   -    |   -    |   -    |   -    |   -    |
| 4     |   PA   |   -    |   -    |   -    |   -    |   -    |
| 8     |   PB   |   -    |   -    |   -    |   -    |   -    |
| 10    |   PC   |   -    |   -    |   -    |   -    |   -    |
| 12    |   PD   |   -    |   -    |   -    |   -    |   -    |
| 14    |   -    |  A01   |   -    |   -    |   -    |   -    |
| 16    |   V    |   -    |   -    |   -    |   -    |   -    |
| 18    |   V    |   -    |   -    |   -    |   -    |   -    |
| 20    |   V    |   -    |   -    |   -    |   -    |   -    |
| 22    |   V    |   -    |   -    |   -    |   -    |   -    |
| 24    |   V    |   -    |   -    |   -    |   -    |   -    |
| 26    |   V    |   -    |   -    |   -    |   -    |   -    |
| 28    |   V    |   -    |   -    |   -    |   -    |   -    |
| 30    |   V    |   A2   |   -    |   -    |   -    |   -    |
| 32    |   -    |   PA   |   -    |   -    |   -    |   -    |
| 34    |   -    |   PB   |   -    |   -    |   -    |   -    |
| 36    |   -    |   PC   |   -    |   -    |   -    |   -    |
| 38    |   -    |   PD   |   -    |   -    |   -    |   -    |
| 40    |   -    |   -    |  A01   |   -    |   -    |   -    |
| 42    |   -    |   V    |   -    |   -    |   -    |   -    |
| 44    |   -    |   V    |   -    |   -    |   -    |   -    |
| 46    |   -    |   V    |   -    |   -    |   -    |   -    |
| 48    |   -    |   V    |   -    |   -    |   -    |   -    |
| 50    |   -    |   V    |   -    |   -    |   -    |   -    |
| 52    |   -    |   V    |   -    |   -    |   -    |   -    |
| 54    |   -    |   V    |   -    |   -    |   -    |   -    |
| 56    |   -    |   V    |   A2   |   -    |   -    |   -    |
| 58    |   -    |   -    |   PA   |   -    |   -    |   -    |
| 60    |   -    |   -    |   PB   |   -    |   -    |   -    |
| 62    |   -    |   -    |   PC   |   -    |   -    |   -    |
| 64    |   -    |   -    |   PD   |   -    |   -    |   -    |
| 66    |   -    |   -    |   -    |  A01   |   -    |   -    |
| 68    |   -    |   -    |   V    |   -    |   -    |   -    |
| 70    |   -    |   -    |   V    |   -    |   -    |   -    |
| 72    |   -    |   -    |   V    |   -    |   -    |   -    |
| 74    |   -    |   -    |   V    |   -    |   -    |   -    |
| 76    |   -    |   -    |   V    |   -    |   -    |   -    |
| 78    |   -    |   -    |   V    |   -    |   -    |   -    |
| 80    |   -    |   -    |   V    |   -    |   -    |   -    |
| 82    |   -    |   -    |   V    |   A2   |   -    |   -    |
| 84    |   -    |   -    |   -    |   PA   |   -    |   -    |
| 86    |   -    |   -    |   -    |   PB   |   -    |   -    |
| 88    |   -    |   -    |   -    |   PC   |   -    |   -    |
| 90    |   -    |   -    |   -    |   PD   |   -    |   -    |
| 92    |   -    |   -    |   -    |   -    |  A01   |   -    |
| 94    |   -    |   -    |   -    |   V    |   -    |   -    |
| 96    |   -    |   -    |   -    |   V    |   -    |   -    |
| 98    |   -    |   -    |   -    |   V    |   -    |   -    |
| 100   |   -    |   -    |   -    |   V    |   -    |   -    |
| 102   |   -    |   -    |   -    |   V    |   -    |   -    |
| 104   |   -    |   -    |   -    |   V    |   -    |   -    |
| 106   |   -    |   -    |   -    |   V    |   -    |   -    |
| 108   |   -    |   -    |   -    |   V    |   -    |  A01   |

[^1]: the order in which the matrix components are fetched is not known yet.
