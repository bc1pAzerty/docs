# Cottage Blueprint

A cozy rural cottage with a small front yard and trees. Reference footprint: 5x5 (scalable).

## Parameters

- `S` = side length (reference: 5, minimum: 4)
- `H` = wall height in layers (reference: 2)
- `baseX`, `baseZ` = footprint origin corner

## Construction Rules

### 1. Foundation (y=0)
Fill SxS area with `stone(1)`.

### 2. Walls (y=1..H)
Build a perimeter ring (outer edge of the SxS footprint) for each layer:
- **Side walls** (dx=0 and dx=S-1, full z span): `planks(8)`
- **Front and back walls** (dz=0 and dz=S-1, inner x): `cobblestone(2)`
- **Front door** (dz=S-1, dx=center, dy=1..H): leave empty (doorway)
- **Front windows** (dz=S-1, dx=center±1, dy=H): `ice(14)`
- **Back window** (dz=0, dx=center, dy=H): `ice(14)`

### 3. Roof (pyramid, starting at y=H+1)
Shrink each layer by 1 block on each side:
- y=H+1: fill SxS with `planks(8)`
- y=H+2: fill (S-2)x(S-2) centered with `birchWood(9)`
- y=H+3: single block at center with `acaciaWood(10)`

For larger S, add more intermediate layers following the same pattern.

### 4. Front Yard (y=0)
Extend 3 rows of `grassBlock(4)` from the front wall (dz=S to dz=S+2), same x span as foundation.

### 5. Trees
Place 2 trees symmetrically in the yard (dx=1 and dx=S-2, dz=S+4 from baseZ):
- Trunk: 2 blocks tall, `birchWood(9)`
- Canopy: 3x3 at y=3, `birchLeaves(11)`
- Top: single leaf block at y=4

### 6. Shrubs
One `birchLeaves(11)` and one `acaciaLeaves(12)` at yard corners (y=1).

## Scaling Notes

- Increasing S adds more wall sections; center door and windows shift accordingly.
- Roof layers increase proportionally: for S=7 add one more shrink layer; for S=9 add two.
- Yard width scales with S; tree spacing = S-3 apart, centered.
