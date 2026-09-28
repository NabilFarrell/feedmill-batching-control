# Screenshots

Screenshots are grouped by the layer they document.

| Folder | Contents |
|--------|----------|
| `hmi/` | Ignition Perspective HMI — the current operator interface |
| `codesys/` | CODESYS project structure and OPC UA symbol configuration |
| `v1-node-red/` | Archived Node-RED dashboard from the v1 HMI layer |

## `hmi/` — Ignition Perspective

Two capture sources: `designer-*` from the Ignition Designer (shows the project being built),
`client-*` from a runtime Perspective session (the actual operator experience).

### Client captures — runtime

| File | View | State |
|------|------|-------|
| `client-overview-idle.jpg` | Overview | Idle — bins full, no batch running |
| `client-overview-dosing.jpg` | Overview | Dosing — valve open, hopper weight climbing |
| `client-overview-discharging.jpg` | Overview | Discharging — hopper emptying to mixer/downstream |
| `client-overview-fault.jpg` | Overview | Fault — alarm condition raised |
| `client-controls.jpg` | Controls | Operator controls — recipe select, target batches, start/stop, E-Stop |
| `client-history-1.jpg` | History | Batch ledger view |
| `client-history-2.jpg` | History | Batch ledger view |
| `client-history-alarm-journal.jpg` | History | Alarm journal populated |

### Designer captures — development environment

| File | View |
|------|------|
| `designer-overview.jpg` | Overview |
| `designer-controls.jpg` | Controls |
| `designer-history.jpg` | History |
| `designer-header.jpg` | Docked header / shared navigation region |
| `designer-startup.jpg` | Startup view |

## `codesys/` — PLC

| File | Shows |
|------|-------|
| `CODESYS Project Tree.png` | Project structure with all POUs across the three layers |
| `CODESYS Symbol Configuration (OPC UA Variables).png` | IEC Symbol Set listing the variables exposed to the HMI |

## `v1-node-red/` — archived

| File | State |
|------|-------|
| `Dashboard Startup.png` | IDLE — all bins full, no batch running |
| `Dashboard Dosing.png` | DOSING — weight gauge rising, bins depleting |
| `Dashboard Complete.png` | COMPLETE — counter incremented, weight drained |
| `Dashboard Emergency Stop.png` | FAULT — E-Stop triggered, system halted |
| `Dashboard Error.png` | FAULT — alarm condition on the dashboard |
| `Node-Red Flow.png` | Node-RED flow editor — dashboard wiring |

See `../v1-node-red/README.md` for why this layer was replaced.
