# Timeline Worker System

This document describes the architecture for running the atmosphere simulation ahead of visual time in a Web Worker, enabling smooth animation despite expensive strike computations.

---

## Core Principle

**The simulation is a deterministic function: `seed + parameters → timeline of events`.**

We run this function AHEAD of visual time in a Web Worker, generating a complete timeline that the main thread plays back. The main thread NEVER runs simulation logic - it only renders pre-computed state.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           WEB WORKER (runs ahead)                        │
│                                                                          │
│  SimulationWorker:                                                      │
│    AtmosphereSimulator.update() → evolve charge, wind, moisture         │
│         ↓                                                               │
│    if breakdown threshold reached:                                      │
│         → simulateBolt() with EXACT current state (BLOCKS)              │
│         → apply post-strike effects (ionization, charge dissipation)    │
│         → continue simulation                                           │
│         ↓                                                               │
│    Output: AtmosphereSnapshot + StrikeEvent                             │
│                                                                          │
│    postMessage() streams snapshots + events to main thread              │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         MAIN THREAD (playback only)                      │
│                                                                          │
│  TimelinePlayer:                                                        │
│    - Receives snapshots and events into TimelineBuffer                  │
│    - Advances visualTimeMs at wall-clock rate                           │
│    - Retrieves snapshot at current visual time                          │
│    - Triggers strike rendering when event time is reached               │
│                                                                          │
│  NO simulation logic on main thread - only rendering pre-computed data  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Critical Constraint: Sequential Strike Computation

**Strikes MUST be computed in the same worker, in sequence with atmosphere simulation.**

This is NOT optional. Attempting to compute strikes in a separate thread or asynchronously would break the simulation model.

### Why Sequential?

1. **Strike path depends on exact atmosphere state**
   - The bolt follows charge gradients
   - It avoids high-moisture regions
   - It preferentially follows existing ionization paths
   - Computing with wrong/stale state produces incorrect paths

2. **Strike immediately affects atmosphere**
   - Charge dissipates around the strike path
   - Ionization is created along the main channel
   - These effects MUST be applied before the next simulation step
   - Future breakdown positions depend on this updated state

3. **Deterministic causal chain**
   - `seed + parameters → timeline` must be reproducible
   - Same inputs must produce identical strike sequence
   - Out-of-sequence computation breaks reproducibility

### What This Means

When `simulateBolt()` runs (1-6 seconds), the worker is blocked:
- No atmosphere steps run
- No snapshots are emitted
- Buffer depletes on main thread

The solution is NOT to parallelize strikes, but to CONTROL WHEN they occur.

---

## Adaptive Charge Accumulation

The key mechanism for maintaining buffer health despite blocking strikes.

### The Problem

- Strike cooldown: 500ms simulation time
- Strike compute: 1-6 seconds real time
- If strikes happen every 500ms, worker is blocked most of the time
- Buffer depletes faster than it fills

### The Solution

When buffer (lead time) is low, slow down charge accumulation so strikes happen less frequently.

```typescript
// SimulationWorker.ts
const leadTime = simTimeMs - visualTimeMs;

let chargeRate = BASE_CHARGE_RATE;  // 0.15 normally

if (leadTime < CRITICAL_LEAD_MS) {           // < 5s
  chargeRate = BASE_CHARGE_RATE * 0.1;       // 10% of normal
} else if (leadTime < MIN_LEAD_FOR_NORMAL_CHARGE_MS) {  // < 15s
  // Interpolate between 10% and 100%
  const t = (leadTime - CRITICAL_LEAD_MS) / (MIN_LEAD_FOR_NORMAL_CHARGE_MS - CRITICAL_LEAD_MS);
  chargeRate = BASE_CHARGE_RATE * (0.1 + 0.9 * t);
}

simulator.setChargeAccumulationRate(chargeRate);
```

### How It Works

1. Worker starts, buffer is empty (low lead time)
2. Charge accumulation slowed to 10%
3. No breakdown for a while, snapshots flood in
4. Buffer fills rapidly
5. Once lead time exceeds 15s, charge rate returns to normal
6. Breakdown occurs, strike computed (blocks 1-6s)
7. Buffer depletes during strike but was large enough
8. After strike, if lead is low, charge slows again

### Benefits

- Single-threaded sequential simulation preserved
- Causal consistency maintained
- Adapts automatically to device performance
- No architectural changes to BoltSimulator required

---

## Timing System

### Worker-Side Timing

```typescript
// SimulationWorker.ts
let simTimeMs = 0;           // Simulation clock (runs ahead)
let visualTimeMs = 0;        // Received from main thread via 'pace' message

const SIM_STEP_MS = 16;      // Each step advances sim time by 16ms
const MAX_LEAD_MS = 45000;   // Don't compute more than 45s ahead

function simulationStep() {
  const leadTime = simTimeMs - visualTimeMs;

  // Pause if too far ahead
  if (leadTime > MAX_LEAD_MS) {
    setTimeout(simulationStep, 100);
    return;
  }

  // Adjust charge rate based on buffer health
  // ... (see adaptive charge section)

  // Run one simulation step
  simulator.update(dtSec);
  simTimeMs += SIM_STEP_MS * config.speed;

  // Emit snapshot if interval reached
  // Handle breakdown if detected (BLOCKS)

  // Schedule next step immediately
  setTimeout(simulationStep, 0);
}
```

### Main-Thread Timing

```typescript
// TimelinePlayer.ts
update() {
  const now = performance.now();
  const dtReal = Math.min(now - lastUpdateTime, 100);  // Cap to prevent huge jumps
  lastUpdateTime = now;

  // Don't start until buffer has minimum lead time
  if (!isPlaying && buffer.leadTimeMs >= MIN_BUFFER_MS) {
    isPlaying = true;
  }

  if (!isPlaying) {
    return buffer.getSnapshot(0);  // Show first snapshot while buffering
  }

  // Binary play/pause based on buffer state
  const effectiveSpeed = leadTimeManager.suggestPlaybackSpeed(buffer.leadTimeMs);
  // Returns 1.0 (play) or 0.0 (pause) - NEVER variable speeds

  visualTimeMs += dtReal * config.speed * effectiveSpeed;

  // Retrieve snapshot and events at current visual time
  return {
    snapshot: buffer.getSnapshot(visualTimeMs),
    strikes: buffer.consumeEvents(prevTime, visualTimeMs)
  };
}
```

### Why Binary Play/Pause?

Variable playback speeds (0.25x, 0.5x) cause slow-motion physics which looks wrong. The correct behavior is:
- Play at full speed (1.0x) when buffer is healthy
- Pause completely (0.0x) when buffer is depleted
- Never intermediate speeds

---

## Data Flow

### Snapshot Flow

```
Worker: simulator.update()
    ↓
Worker: createSnapshot() - serializes all field data
    ↓
Worker: postMessage({ type: 'snapshot', snapshot })
    ↓
Main: TimelineBuffer.addSnapshot(snapshot)
    ↓
Main: TimelinePlayer.update() retrieves snapshot at visualTimeMs
    ↓
Main: ChargeFieldRenderer.updateFromSnapshot(snapshot)
    ↓
GPU: Render charge field visualization
```

### Strike Flow

```
Worker: simulator.update() detects breakdown
    ↓
Worker: computeStrike(position) - BLOCKS for 1-6 seconds
    ↓
Worker: applyPostStrikeEffects() - dissipate charge, create ionization
    ↓
Worker: postMessage({ type: 'strike', event })
    ↓
Main: TimelineBuffer.addEvent(event)
    ↓
Main: TimelinePlayer.consumeEvents() when visualTimeMs reaches event time
    ↓
Main: LightningController creates LightningBoltEffect with pre-computed geometry
    ↓
GPU: Render lightning bolt
```

---

## Key Files

| File | Purpose |
|------|---------|
| `SimulationWorker.ts` | Runs simulation in worker, adaptive charge pacing |
| `AtmosphereSimulator.ts` | Atmosphere physics: charge, wind, moisture, ionization |
| `BoltSimulator.ts` | Strike path computation (expensive, blocks worker) |
| `TimelinePlayer.ts` | Main thread playback, buffer management |
| `TimelineBuffer.ts` | Ring buffer for snapshots and events |
| `LeadTimeManager.ts` | Suggests playback speed based on buffer state |
| `SimulationTimeline.ts` | Type definitions for snapshots, events, config |

---

## Configuration

### Timeline Config

```typescript
interface TimelineConfig {
  snapshotIntervalMs: number;     // How often to emit snapshots (33ms = 30fps)
  speed: number;                   // Playback speed multiplier
  detail: number;                  // Strike detail level
  chargeAccumulationRate: number;  // Base rate (adjusted dynamically)
  breakdownThreshold: number;      // Charge level that triggers strike
  baseWindSpeed: number;           // Wind speed in simulation units
  // ... wind direction, etc.
}
```

### Timing Constants

```typescript
const SNAPSHOT_INTERVAL_MS = 33;        // ~30fps snapshots
const SIM_STEP_MS = 16;                 // 16ms per simulation step
const MAX_LEAD_MS = 45000;              // Max buffer ahead
const MIN_BUFFER_BEFORE_PLAY = 12000;   // Wait for 12s buffer before starting
const STRIKE_COOLDOWN_MS = 500;         // Min time between strikes (sim time)
```

### Adaptive Charge Thresholds

```typescript
const CRITICAL_LEAD_MS = 5000;           // Below this: 10% charge rate
const MIN_LEAD_FOR_NORMAL_CHARGE_MS = 15000;  // Above this: 100% charge rate
const BASE_CHARGE_RATE = 0.15;           // Normal accumulation rate
```

---

## Debugging

### Console Logs

Worker logs (in worker console):
```
[Worker] Strike computed in 4523ms
```

Main thread logs:
```
[TimelinePlayer] lead=12038ms, speed=1.00x, visual=32ms
[TimelinePlayer] Buffer ready, starting playback
[TimelinePlayer] Strike received: simTime=7270, playhead=0, bufferEnd=7782
```

### Health Indicators

- **Healthy**: lead > 15000ms, speed = 1.00x
- **Recovering**: lead 5000-15000ms, charge rate reduced
- **Critical**: lead < 5000ms, charge at 10%, may pause playback
- **Underrun**: lead < 0, speed = 0.00x (paused)

---

## Common Issues

### "Nothing happens for a long time"

**Cause**: Buffer is building, or charge rate is low due to low lead time.

**Check**: Console logs for lead time. If lead is low, adaptive charge is working correctly - wait for buffer to fill.

### "Animation jerks/freezes"

**Cause**: Buffer underrun during strike computation.

**Fix**: Ensure adaptive charge rate is working. Increase `MIN_LEAD_FOR_NORMAL_CHARGE_MS` if needed.

### "Visual time jumps massively"

**Cause**: Tab was backgrounded, then `dtReal` was huge on return.

**Fix**: `dtReal` is capped to 100ms in TimelinePlayer.update().

---

## Historical Context

### Why Not Separate Workers?

Early designs considered computing strikes in a separate worker. This was rejected because:

1. Strike path depends on exact atmosphere state at breakdown time
2. Post-strike effects must be applied before next simulation step
3. Saving/restoring atmosphere state adds complexity and potential desync
4. The adaptive charge rate solution is simpler and maintains causal consistency

### Why Pre-computed Timeline?

The alternative - running simulation in real-time on main thread - was rejected because:

1. Strikes take 1-6 seconds, would freeze UI completely
2. No way to smooth out variable compute times
3. Can't buffer ahead to absorb spikes

The worker-based timeline allows:
- UI stays responsive during strike computation
- Buffer absorbs compute time variation
- Playback is smooth regardless of device speed

---

*Created: 2026-02-18*
