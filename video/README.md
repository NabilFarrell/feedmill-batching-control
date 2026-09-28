# Demo Videos

Screen recordings of the feedmill batching system in operation.

## Current — Ignition Perspective HMI

| File | Duration | What it shows |
|------|----------|---------------|
| `feedmill-batching-demo.mp4` | 154s | Full demo of the Ignition HMI: every batch state, emergency stop, and fault/alarm handling |

- 1920×1080, H.264 + AAC
- Recorded from a runtime client session

## Archived — v1 Node-RED HMI layer (`v1-node-red/`)

| File | Duration | What it shows |
|------|----------|---------------|
| `Idle-Control-Start-Emergency-Release Emergency.mp4` | 158s | Full batch cycle (5 batches) + Emergency Stop trigger and release + COMPLETE state |
| `Idle-Control-Start-Empty Bin-Release Error Status.mp4` | 92s | Full batch cycle (3 batches) + Empty Bin alarm trigger and reset |

### Emergency Stop demo (v1)

- 0:00 — IDLE state, all bins at 5000 kg
- 0:15 — Recipe changed to Poultry, Target Batch set to 10
- 0:30 — First batch in DISCHARGING phase
- 1:30 — Emergency Stop triggered (Safety Lock turns RED, FAULT state)
- 1:45 — Emergency Stop released (Safety Lock GREEN, state stays FAULT)
- 2:00 — System recovered, batch 4 in MIXING
- 2:30 — Target Batch reduced to 5, COMPLETE state reached

### Alarm demo (v1)

- 0:00 — IDLE state, bins pre-loaded with lower levels
- 0:15 — START pressed, DOSING begins
- 0:30 — Batch 1 COMPLETE
- 1:05 — Bin 1 (Corn) runs low, Empty Bin alarm triggered
- 1:20 — System in FAULT, ERROR + Empty Bin LEDs RED
- 1:30 — Reset to IDLE

## Playback

MP4 files — play with any media player (VLC, Windows Media Player, browser).
