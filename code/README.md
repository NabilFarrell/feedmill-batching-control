# PLC Code Exports

PDF exports from CODESYS of all PLC program organization units (POUs).

## Files

| File | Pages | Description |
|------|-------|-------------|
| `Controller-ST.pdf` | 4 | Main state machine — 6 states, recipe management, alarm handling, E-Stop, manual discharge |
| `Controller-LD.pdf` | 1 | Safety interlocks — E-Stop latching, gate interlocks, mixer/dischARGE safety |
| `PlantModel-ST.pdf` | 2 | Physics simulation — gate timers, flow integration, bin level tracking |
| `GVL.pdf` | 2 | Global Variable List — all PLC variables, recipes, status, safety |

## How to read

- **Controller-ST**: Start at line 20 (IDLE state), follow the CASE statement through all 6 states
- **Controller-LD**: Rungs 1-4 = gate interlocks, Rung 5 = mixer, Rung 6 = discharge, Rungs 7-8 = E-Stop
- **PlantModel_ST**: Timer logic at top, flow integration in middle, bin depletion at bottom
- **GVL**: Variable declarations grouped by function (Commands, Actuators, Sensors, Status, Safety)
