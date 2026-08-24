# Documentation

Project specification and design documents.

## Files

| File | Description |
|------|-------------|
| `mock_project_1_feedmill_batching_spec.md` | Full project specification — architecture, state machine design, variable definitions, recipes, alarm logic, OPC UA configuration |

## How to read the spec

The spec is the authoritative design document. It contains:
1. **Architecture** — 3-program structure (Controller_ST → Controller_LD → PlantModel_ST)
2. **State Machine** — 6 states with transition conditions
3. **Recipes** — 4 recipes with ingredient proportions
4. **Variables** — Complete GVL definition
5. **Alarms** — Alarm conditions and handling
6. **Safety** — E-Stop design, manual discharge
7. **OPC UA** — Symbol Set configuration
8. **Troubleshooting** — Bugs found and fixes applied
