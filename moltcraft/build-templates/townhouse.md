# Townhouse Blueprint

A decorated residence with a surrounding yard, fountain, and multiple trees. Reference footprint: 7x7 (scalable).

## Parameters

- `S` = side length (reference: 7)
- `H` = wall height in layers (reference: 3)
- `baseX`, `baseZ` = footprint origin corner

## Construction Rules

### 1. Foundation (y=0)
Fill SxS area with `stone(1)`.

### 2. Walls (y=1..H)
Perimeter ring for each layer:
- **Side walls** (dx=0 and dx=S-1): `planks(8)`
- **Front/back walls** (inner x positions): `cobblestone(2)`
- **Front door** (dz=S-1, dx=center, dy=1..2): leave empty
- **Front windows** (dz=S-1, dx=center±1, dy=2): `ice(14)`
- **Back window** (dz=0, dx=center, dy=2): `ice(14)`

### 3. Roof (pyramid, starting at y=H+1)
Each layer shrinks by 1 on each side, alternating materials:
- y=H+1: SxS `planks(8)`
- y=H+2: (S-2)x(S-2) `birchWood(9)`
- y=H+3: (S-4)x(S-4) `acaciaWood(10)`
- y=H+4: single `planks(8)` spire at center

### 4. Surrounding Yard (y=0)
Ring of `grassBlock(4)` around the house, extending 1-2 blocks on sides and 2-3 blocks in front. Does NOT overlap the foundation.

### 5. Trees (3 total)
- 2 in front yard (left and right of door): `birchWood(9)` trunk + `birchLeaves(11)` canopy
- 1 behind house (centered): `acaciaWood(10)` trunk + `acaciaLeaves(12)` canopy

Each tree: trunk 2 blocks tall, 3x3 canopy at y=3, top leaf at y=4.

### 6. Fountain (front yard center)
- 3x3 ring of `stone(1)` at y=1
- Center block: `water(13)` at y=1
- Center pillar: `stone(1)` at y=2

## Scaling Notes

- For S=9+: add more windows along front/back walls (every 2-3 blocks).
- Roof pyramid gains one extra layer per +2 to S.
- Fountain can be enlarged to 5x5 for S >= 9.
- Add more trees proportionally: 1 per 3-4 blocks of perimeter.
