# How the animation works

## The core idea

Every frame, each of the 3,600 items on the timeline gets three things recalculated: its length, its position, and its color. There's no physics, no object list, no history. Just one math function called 3,600 times per frame:

```
(column, row, time)  →  (length, position, color)
```

---

## Step 1 — normalize the grid coordinates

Before any math happens, column and row numbers get converted to a 0–1 range:

```lua
nx = col / 120   -- 0.0 on the left,  1.0 on the right
ny = row / 30    -- 0.0 on the top,   1.0 on the bottom
```

This makes the formula work the same regardless of grid size.

---

## Step 2 — compute the wave

```lua
phase    = (nx + ny) * 6.0 - t * 3.5
wave     = sin(phase)
wave2    = sin(phase * 2.3 + 1.5) * 0.4
combined = wave + wave2
```

**`nx + ny`** — adding the two coordinates together means that items sitting on the same diagonal share the same phase value. This is what makes the wave travel diagonally rather than horizontally or vertically.

**`* 6.0`** — controls how many wave crests fit across the screen at once. Higher = more crests, shorter wavelength.

**`- t * 3.5`** — subtracting time from the phase makes the wave move forward each frame. The bigger this number, the faster it travels.

**`wave2`** — a second sine wave layered on top of the first. Its frequency (2.3) isn't a clean multiple of the first wave's frequency, so the two never perfectly repeat together. This is what makes the motion look organic and snake-like rather than a simple smooth pulse.

**`combined`** — the sum of both waves. Ranges roughly from −1.4 to +1.4.

---

## Step 3 — turn the wave into item properties

### Length

```lua
len_mult = 0.1 + 0.9 * max(0, combined / 1.4)
```

When the wave is at a crest, the item is near its maximum length. When it's in a trough, the item shrinks to 10% of its base size. The `max(0, ...)` clamps negative values so items never fully disappear.

### Position

```lua
pos_off = combined * 0.06
```

Items shift slightly forward or backward from their resting position depending on whether the wave is pushing or pulling at that moment. The maximum shift is 0.06 seconds on the timeline — small enough that items never overlap each other.

### Color

```lua
hue        = (nx + ny) * 0.3 + t * 0.25
brightness = 0.4 + 0.6 * len_mult
```

The hue slowly drifts over time and varies by diagonal position, so colored bands travel across the grid independently of the length wave. Brightness is tied to length — long items are bright, short items are dim — which makes the wave feel three-dimensional.

---

## Step 4 — write only what changed

Before updating an item, the script checks whether the new value actually differs from the last frame:

```lua
if math.abs(new_len - prev_len) > 0.001 then
    SetMediaItemInfo_Value(item, "D_LENGTH", new_len)
end
```

On most frames, most items barely move, so the majority of API calls get skipped. This is the main performance optimization.

The entire update loop is also wrapped in `PreventUIRefresh(1/−1)` so REAPER redraws the timeline only once per frame, not after every single item change.

---

## Why it looks like a snake

The diagonal wave direction comes from using `nx + ny` as the spatial coordinate. Lines of equal phase run diagonally at 45°, and as time advances those lines slide down-right across the grid. The second harmonic breaks the smooth regularity — instead of a clean rolling wave you get an uneven, wriggling motion that reads as a snake moving through the grid.
