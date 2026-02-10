# Return Stroke Physics

The return stroke is the brilliant flash that makes lightning visible to the eye. It occurs only after the stepped leader connects to ground.

---

## Overview

The return stroke is a luminous wave that travels UP from ground along the ionized channel created by the stepped leader. It's what we see as "the lightning bolt."

---

## Physical Mechanism

### Trigger: Ground Connection

When the stepped leader approaches ground (within ~100m):
1. **Upward connecting leaders** may initiate from ground objects
2. Upward leader speed: 1-5 x 10^5 m/s
3. Connection occurs at the **striking distance** (30-200m depending on charge)
4. Connection creates a conducting path

### The Wave

The return stroke is NOT a current pulse traveling up. It's a **potential wave**:

1. Ground is at zero potential
2. Leader channel is at high potential (~50-100 MV)
3. Connection collapses this potential difference
4. The collapse propagates up the channel
5. Current flows down as potential wave travels up

---

## Measured Values

From Rakov & Uman (2003), Dwyer & Uman (2014):

| Parameter | Typical Value | Range |
|-----------|---------------|-------|
| Propagation speed | c/3 (1 x 10^8 m/s) | (0.1-0.5)c |
| Peak current (first stroke) | 30 kA | 10-200 kA |
| Peak current (subsequent) | 12 kA | 5-50 kA |
| Current rise time (first) | 5 microseconds | 1-10 microseconds |
| Current rise time (subsequent) | 0.5 microseconds | 0.2-2 microseconds |
| Total duration | 100-200 microseconds | - |
| Channel temperature | 30,000 K | 25,000-35,000 K |
| Peak channel radius | 1-3 cm | - |
| Energy per meter | 10^3-10^4 J/m | - |
| Peak power | 10^12 W | (briefly) |

---

## Visual Characteristics

### Brightness

The return stroke is **orders of magnitude brighter** than the stepped leader:

| Phase | Relative Brightness |
|-------|---------------------|
| Stepped leader | 1-10% |
| Return stroke | 100% |

This extreme contrast is why:
- Leader phase shows branching structure (dim)
- Return stroke shows single bright channel (overwhelming)
- In photographs, leader branches are visible; return stroke is a single line

### Appearance

During return stroke:
- Single bright channel (the main channel)
- Branches connected to main channel briefly illuminate
- Branches NOT connected to main channel remain dark
- No branching structure visible (too bright, too fast)

### Color

- Temperature: ~30,000 K
- Emission dominated by: N II, O II, H alpha spectral lines
- Perceived color: White with slight blue tint
- Color temperature: ~25,000-30,000 K

---

## Luminosity Profile

### Spatial Distribution

Luminosity follows the current:
```
L(z, t) = L0 * I(t - z/v_rs) * exp(-z / lambda_L)
```

Where:
- z = height above ground
- v_rs = return stroke speed
- lambda_L = luminosity decay length (2000-5000 m)

### Temporal Behavior

1. Peak luminosity at base
2. Wave propagates upward at v_rs
3. Upper portions remain bright for ~100 microseconds after passage
4. Entire channel dims together in decay phase

---

## Current Waveform

The Heidler function is commonly used:

```
I(t) = I0 * (t/tau1)^2 * exp(-t/tau2) / ((t/tau1)^2 + 1)
```

Where:
- I0 = peak current scaling
- tau1 = rise time constant
- tau2 = decay time constant

**First stroke:**
- I0 = 30,000 A (30 kA peak)
- tau1 = 1-2 microseconds
- tau2 = 50-100 microseconds

**Subsequent strokes:**
- I0 = 10,000-15,000 A
- tau1 = 0.5 microseconds (faster rise)
- tau2 = 30-50 microseconds

---

## Channel Radius

The Braginskii model describes channel expansion:

```
r(t) ~ (I^2 * t / rho)^(1/4)
```

Where rho = air density.

Simplified:
```
r ~ r0 * (I/I0)^0.5 * (t/t0)^0.25
```

| State | Radius |
|-------|--------|
| Stepped leader channel | 1-5 mm |
| Peak return stroke | 1-3 cm |
| Visible glow (bloom) | 1-10 m |

---

## Subsequent Strokes

Most flashes contain 3-5 strokes (range: 1-26).

### Between Strokes

1. **J-process**: Current continues flowing in cloud (40-100 ms)
2. **K-changes**: Recoil streamers in cloud
3. **Dart leader**: Continuous (NOT stepped) leader along existing channel
   - Speed: 1-2 x 10^7 m/s (10x faster than stepped leader)
   - Duration: 1-2 ms
   - Current: 1-2 kA

### Subsequent Stroke Characteristics

- Peak current: 10-15 kA (lower than first)
- Rise time: 0.5-1 microseconds (faster)
- Same channel as first stroke

### Interstroke Interval

- Duration: 40-100 ms
- Creates characteristic "flickering" appearance
- Perceptible to human eye

---

## Animation Implications

### Timing (Scaled for Visibility)

Real timing is too fast to perceive:
- Return stroke: ~100 microseconds
- Human perception threshold: ~10 milliseconds

Our animation scales:
- Return stroke duration: 80-100 ms (visible upward travel)
- Stroke hold: 100-150 ms (peak brightness)
- Fade: 200-400 ms (gradual dim)

### Brightness Model

During return stroke phase:
```javascript
// Main channel lights up from ground
for (segment in mainChannel.reversed()) {
  if (segment.index < wavePosition) {
    brightness = 1.0;  // Full illumination
  } else if (segment.index === wavePosition) {
    brightness = 1.0;  // Wavefront
  } else {
    brightness = 0.2;  // Leader remnant, not yet reached
  }
}

// Connected branches briefly flash
for (segment in branches) {
  const parentBrightness = getBrightness(segment.parent);
  brightness = parentBrightness * 0.3 * exp(-segment.depth * 0.5);
}
```

### Subsequent Strokes

For flickering effect:
- Each subsequent stroke dimmer: `brightness *= 0.8^strokeIndex`
- Interstroke period: low brightness (0.05-0.1)
- 3-4 strokes typical for showcase

---

## Attachment Process

Before return stroke, there may be upward connecting leaders:

### When They Occur

- When stepped leader is within ~100m of ground
- From tall structures, trees, or ground points
- Speed: 1-5 x 10^5 m/s

### Striking Distance

```
d_s = 10 * I_p^0.65  (in meters, I_p in kA)
```

For 30 kA first stroke: d_s ~ 150 meters

### Implementation Note

Our current simulation doesn't model upward leaders. The stepped leader simply terminates when reaching ground (within connectionThreshold).

Future enhancement could add:
- Ground-initiated upward leaders
- Meeting point in mid-air
- More authentic final connection behavior

---

## References

- Rakov & Uman (2003) - Return stroke measurements
- Dwyer & Uman (2014) - Optical observations
- Heidler (1985) - Current waveform model
- Braginskii (1958) - Channel radius model
