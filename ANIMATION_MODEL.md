# Animation Model

## Core idea

The animation holds no mutable state except time `t`. Every frame, each of the 3,600 items is computed independently by a pure function:

```
(col, row, t)  →  (length, offset, color)
```

No object lists, no physics, no history — just one mathematical function evaluated 3,600 times.

---

## Coordinate system

Before the wave calculation, item coordinates are normalized to the [0, 1] range:

```lua
local nx = col / GRID_W   -- 0.0 .. 1.0 horizontal
local ny = row / GRID_H   -- 0.0 .. 1.0 vertical
```

This makes the formula grid-size agnostic — `GRID_W = 60` and `GRID_W = 240` produce the same wave shape.

---

## Snake formula

```lua
local phase    = (nx + ny) * 6.0 - t * 3.5
local wave     = math.sin(phase)
local wave2    = math.sin(phase * 2.3 + 1.5) * 0.4
local combined = wave + wave2
```

**`(nx + ny) * 6.0`** — spatial phase. Summing nx and ny produces diagonal isolines: all points on the same grid diagonal share the same phase value. The multiplier 6.0 sets the spatial frequency — how many full waves fit across the diagonal.

**`- t * 3.5`** — time offset. As `t` grows each frame, the phase decreases and the wave travels in the direction of increasing (nx + ny), i.e. toward the bottom-right.

**`wave2 = sin(phase * 2.3 + 1.5) * 0.4`** — a second harmonic with an irrational frequency ratio (2.3). Because it's not a multiple of the fundamental, the pattern never repeats periodically. The phase offset of 1.5 breaks symmetry around zero.

**`combined = wave + wave2`** — sum of both harmonics. Range ≈ [−1.4, +1.4].

---

## Three output values

### Item length (`len_mult` → `D_LENGTH`)

```lua
local len_mult = 0.1 + 0.9 * math.max(0, combined / 1.4)
```

`math.max(0, ...)` clips negative values: when the wave is in a trough, the item sits at its minimum length rather than disappearing. Dividing by 1.4 normalizes `combined` to [0, 1]. Result: `len_mult` ∈ [0.1, 1.0] — items never collapse entirely.

```lua
local new_len = math.max(0.002, BASE_LENGTH * len_mult)
```

`BASE_LENGTH = 0.12` seconds × `len_mult` gives the final length in REAPER timeline units.

### Item position (`pos_off` → `D_POSITION`)

```lua
local pos_off = combined * CELL_WIDTH * 0.4
```

Offset from the item's base position. The sign of `combined` determines direction: a wave crest pushes the item forward, a trough pulls it back. `CELL_WIDTH * 0.4 = 0.06` seconds is the maximum displacement — less than half the inter-item spacing, so items never overlap.

### Color (`hue` → `I_CUSTOMCOLOR`)

```lua
local hue = (nx + ny) * 0.3 + t * 0.25
```

Hue depends on position and drifts slowly over time (`t * 0.25`). The diagonal color bands move more slowly than the length wave, creating a visual layering between the two animations.

```lua
local brightness = 0.4 + 0.6 * len_mult
```

Brightness is tied to length: long items are bright, short ones are dark. This reinforces the sense of depth.

The final color is assembled via HSV → RGB → native REAPER color in `hsv_to_native`.

---

## Delta optimization

Before each `SetMediaItemInfo_Value` call, the new value is compared to the previous one:

```lua
if math.abs(new_len - prev_length[row][col]) > 0.001 then
    reaper.SetMediaItemInfo_Value(item, "D_LENGTH", new_len)
    prev_length[row][col] = new_len
end
```

On most frames most items change very little — the delta check skips several thousand API calls per frame.

`PreventUIRefresh(1)` / `PreventUIRefresh(-1)` wrap the entire update loop so REAPER batches all redraws to the end of the frame rather than re-rendering after each call.

---

## Data flow for one frame

```
reaper.time_precise()
    │
    ▼
dt = now - last_frame_time
t  = t + dt * SPEED
    │
    ▼  (for each of the 3,600 items)
compute_wave(col, row, t)
    │
    ├─→ len_mult  →  D_LENGTH
    ├─→ pos_off   →  D_POSITION
    └─→ hue       →  I_CUSTOMCOLOR
    │
    ▼
reaper.UpdateArrange()
```

`dt` is taken from real elapsed time, not a fixed step. This means that if FPS drops, `t` jumps by a larger increment and the animation stays at the correct speed rather than slowing down.

---

## Why the diagonals look like a snake

The key is `phase = (nx + ny) * 6.0`. Isolines of equal phase are lines of the form `nx + ny = const` — diagonals at 45°. As `t` grows, these diagonals slide along the (nx + ny) axis, moving from the top-left toward the bottom-right. The second harmonic breaks the monotony: instead of a clean sinusoid the result is an uneven pulse that resembles the movement of a snake's body.
