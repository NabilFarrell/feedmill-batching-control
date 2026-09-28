# Ignition Perspective HMI Project

The Ignition 8.3 Perspective project for the feedmill batching HMI.

## File

| File | Description |
|------|-------------|
| `designer project export.zip` | Ignition project archive — all views, bindings, styles, page configuration, and the docked header |

## What it contains

| Element | Detail |
|---------|--------|
| **Views** | `homr` (process mimic), `Input` (operator controls), `Error` (history + alarm journal), `header` (shared navigation region) |
| **Page config** | 3 pages with mounted primary views — `/`, `/Input`, `/History` |
| **Shared region** | `header` docked to the page layout, providing navigation and system status on every view |
| **Alarms** | Native Ignition alarm records for `Alarm_EmptyBin` and `Alarm_Underweight`, with High/Medium priority and acknowledgement lifecycle |
| **Data** | SQLite batch-history database, named queries, and historian-backed load cell trending |

## How to import

1. Open Ignition 8.3 Gateway (Designer → Tools → Import Project, or drag the archive onto the Project Browser)
2. Restart the client session — Perspective resolves page configuration at session load
3. The OPC UA device connection must be re-pointed at your own gateway endpoint; credentials are intentionally **not** included

## Implementation notes

- **State-driven colour** — valve, pipe, and bin fills are bound to tag state, not fixed values. Idle renders grey, active renders green or red per ISA-101
- **Per-bin fault identification** — the PLC exposes one global `Alarm_EmptyBin` flag, so per-bin indication is derived in the HMI from `BinLevel_n` against `EMPTY_THRESHOLD`
- **Status priority chain** — the system status box evaluates EMERGENCY STOP before process alarms, so the most critical condition always wins the display
- **Tag Groups** — mimic tags are assigned to a fast tag group; the default 1000 ms rate is a visible latency on a live weight display
