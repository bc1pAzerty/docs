# Villa Blueprint

A grand symmetrical mansion with stepped pyramid roof, corner trees, front pond, and rich landscaping. Reference footprint: 9x9 (scalable).

## Parameters

- `S` = side length (reference: 9)
- `H` = wall height in layers (reference: 4)
- `baseX`, `baseZ` = footprint origin corner

## Construction Rules

### 1. Foundation (y=0)
Fill SxS area with `stone(1)`.

### 2. Walls (y=1..H)
Perimeter ring for each layer:
- **Side walls** (dx=0 and dx=S-1): `planks(8)`
- **Front/back walls**: `cobblestone(2)`
- **Front door** (dz=S-1, dx=center, dy=1..2): leave empty
- **Door frame** (dx=center±1, dy=1..2): `planks(8)`, lintel at (dx=center, dy=3): `planks(8)`
- **Front windows** (dz=S-1, dx=center±2, dy=2..3): `ice(14)` — two blocks tall for symmetry
- **Back windows** (dz=0, dx=center±2, dy=2): `ice(14)`

### 3. Roof (stepped pyramid, starting at y=H+1)
Shrink by 1 on each side per layer, alternating materials:
- y=H+1: SxS `planks(8)`
- y=H+2: (S-2)x(S-2) `birchWood(9)`
- y=H+3: (S-4)x(S-4) `acaciaWood(10)`
- y=H+4: (S-6)x(S-6) `planks(8)`
- y=H+5: single `birchWood(9)` spire at center

For larger S, continue the shrink pattern until 1x1 is reached.

### 4. Surrounding Yard (y=0)
Ring of `grassBlock(4)` around the house, extending 2 blocks on all sides. Clamp to regionBounds (exclusive upper bound).

### 5. Corner Trees (4 total)
One tree at each corner of the yard, alternating wood types:
- Left-side trees: `birchWood(9)` + `birchLeaves(11)`
- Right-side trees: `acaciaWood(10)` + `acaciaLeaves(12)`

Each tree: trunk 2 blocks tall, 3x3 canopy at y=3, top leaf at y=4. Keep canopy within regionBounds.

### 6. Front Pond
Positioned in front yard, offset from door:
- 3x3 `stone(1)` base at y=0
- Ring of `stone(1)` at y=1 (perimeter)
- Center: `water(13)` at y=1
- Decorative leaves at ±2 blocks around pond: alternating `birchLeaves(11)` and `acaciaLeaves(12)`

## Scaling Notes

- For S=11+: add a second row of windows on side walls.
- For S=13+: consider an interior dividing wall to create rooms.
- Roof pyramid layers = floor((S-1)/2). Each layer shrinks the footprint by 2.
- Pond can scale to 5x5 for S >= 11, add a central fountain pillar.
- Corner trees can become groves (2-3 trees per corner) for S >= 13.
- Window pattern: place ice windows at dx = center ± floor(S/4), spaced evenly.
