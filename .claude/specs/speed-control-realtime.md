---
status: active
priority: high
area: showcase
---

# Speed Control: Immediate Effect on Animation

## Problem

When the user changes the speed slider, it only takes effect at the start of the next strike phase. If a strike is playing at very slow speed and the user moves the slider back up, the current strike must fully play out at the slow speed before the change applies. This makes the speed control feel broken and unresponsive.

## Goal

Speed changes must take effect IMMEDIATELY on the currently playing animation. If a bolt is mid-leader-phase at 0.1x speed and the user slides to 1.0x, the animation should instantly speed up.

## Architecture Challenge

The current system pre-computes bolt geometry in a Web Worker, then plays it back via BoltAnimator using a timeline. The speed parameter is used during playback but may only be sampled at phase transitions. The fix likely requires:
- BoltAnimator to read speed every frame, not just at phase start
- The animation timeline to use elapsed-time-scaled-by-speed rather than fixed durations
- LightningController to pass the current speed ref to the active strike on every frame

## Acceptance Criteria

- [ ] Changing speed slider immediately affects the current strike animation
- [ ] Slowing down mid-strike shows the bolt in slow motion
- [ ] Speeding up mid-strike accelerates the remaining animation
- [ ] No visual glitches or jumps when speed changes mid-animation
- [ ] Works for all phases: leader stepping, return stroke, decay
