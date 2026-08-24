# Dashboard Screenshots

Screenshots of the Node-RED HMI dashboard in various operational states.

## Available Screenshots

| File | State | Description |
|------|-------|-------------|
| `Dashboard Startup.png` | IDLE | Initial state — all bins full, no batch running |
| `Dashboard Dosing.png` | DOSING | Active dosing — weight gauge rising, bins depleting |
| `Dashboard Complete.png` | COMPLETE | Batch finished — counter incremented, weight drained |
| `Dashboard Emergency Stop.png` | FAULT | E-Stop triggered — Safety Lock RED, system halted |
| `Dashboard Error.png` | FAULT | Alarm condition — Empty Bin or Underweight triggered |
| `CODESYS Project Tree.png` | — | CODESYS project structure showing all POUs |
| `CODESYS Symbol Configuration (OPC UA Variables).png` | — | OPC UA Symbol Set showing exposed variables |
| `Node-Red Flow.png` | — | Node-RED flow editor showing dashboard wiring |

## Dashboard layout

```
┌──────────────┬──────────────────┬──────────────────┐
│   Status     │     Process      │    Controls      │
│   Alarm      │   (gauges)       │   (buttons)      │
└──────────────┴──────────────────┴──────────────────┘
```
