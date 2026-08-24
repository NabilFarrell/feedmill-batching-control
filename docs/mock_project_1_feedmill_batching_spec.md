# Mock Project 1 — Feedmill Batching & Dosing System (MVP Spec)

> Purpose: Portfolio project for PT Charoen Pokphand Indonesia (CPIN) Automation Engineer application.
> CP match: mirrors real CP projects — minibin dosing (Agus Prayudi), Silo-Dryer automatic weight batching (Oding Panji Syahdana), feedmill weighing/interlocking.
> Handoff doc for a dedicated AI assistant. Follow milestones in order. Verify each milestone before moving on.

---

## 1. Goal & Success Criteria

Build a simulated feedmill batching system in CODESYS that:
1. Runs a 4-bin → weigh hopper → mixer → discharge batch cycle with recipes, interlocks, tolerance checking, and alarms
2. Uses the **two-layer architecture**: real controller logic + separate plant model (fake physics)
3. Exposes live variables over **OPC UA** (verified with UaExpert)
4. Is demoable in the free 2-hour demo mode

**Success =** UaExpert connects to `opc.tcp://localhost:4840`, sees the variables, and the batch cycle runs correctly end-to-end with the weight climbing, stopping at target, mixing, discharging, and counting batches.

**Out of scope for MVP:** Node-RED bridge, SQL logging, VB.NET/Delphi HMI (later milestones/projects).

---

## 2. Environment (already installed)

| Component | Version | Notes |
|---|---|---|
| CODESYS Development System | V3.5 SP22 (64 bit) Patch 3 | Free, installed at D:\Nabil\Software\Codesys\CODESYS |
| CODESYS Control Win V3 (x64) | 3.5.22.30 | Soft-PLC runtime, **already in project** |
| OPC UA Server | Included in standard install (basic version) | "License only" store product = just the license; software is built in |
| UaExpert | Latest | Free OPC UA client for testing |

**Demo mode rules:** runtime + OPC UA run 2 hours per session without license, then shut down. Restart the runtime to reset. No total cap. Program offline freely; only run the soft-PLC when testing/demoing.

**Known gotchas:**
- "OPC UA Server (not available)" in Security settings = **no certificate installed yet** (not a license issue). Generate one, then trust the client cert (move from Quarantined to Trusted).
- Port 4840 conflict with Windows OPC UA Discovery Service: check `netstat -ano | findstr :4840`. Log: `%ProgramData%\CODESYS\CODESYSControlWinV3\<PlcLogic>\Logs\OpcUa.log` (look for `E_BIND_FAILURE`).
- Use **Communication Manager** (SP18+), NOT the old Symbol Configuration.
- Security advisory 2024-03 (OPC UA buffer bug) fixed in 3.5.20.10 — SP22 is safe.

---

## 3. Architecture

```
┌────────────────────────── CODESYS Control Win V3 (soft-PLC) ──────────────────────────┐
│  Task: MainTask (IEC task, cycle time 100 ms)                                          │
│                                                                                        │
│  PLC_PRG (main, calls all three)                                                       │
│   ├── Controller_ST    ← REAL logic (state machine, recipes, alarms)                   │
│   ├── Controller_LD    ← interlocks (one-gate-at-a-time, permissives)                  │
│   └── PlantModel_ST    ← FAKE factory (physics: flow integration, gate delays, clamps) │
│                                                                                        │
│  Communication Manager → OPC UA Server → symbol set (exposed variables)                │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                    │ opc.tcp://localhost:4840
                                    ▼
                              UaExpert (test client)
```

**Rule:** Controller_ST must be written exactly as if running on real hardware. It only *reads* sensor variables and *writes* command variables. Controller_LD combines those commands with physical interlocks and drives the actuator outputs. PlantModel_ST owns the physics — it reads actuator outputs and computes sensor values. Never mix the layers.

---

## 4. Process Description (what we're simulating)

A feedmill batching station:
- **4 ingredient bins** (Bin1–Bin4), each with a discharge gate and a level sensor
- Material flows from an open bin into a **weigh hopper** (load cell)
- When the recipe target weight is reached, the gate closes
- After all 4 bins are dosed, the **mixer** runs for a fixed time
- Then the **discharge gate** opens and the hopper empties into the next process
- Batch counter increments; cycle repeats (auto or manual restart)

---

## 5. Global Variables (GVL)

### Commands (written by Controller_ST, read by Controller_LD)
| Variable | Type | Meaning |
|---|---|---|
| Gate1_Activate .. Gate4_Activate | BOOL | Bin gate open request (interlocked by LD) |
| Mixer_Activate | BOOL | Mixer start request |
| Discharge_Activate | BOOL | Discharge gate open request |

### Actuators (written by Controller_LD, read by PlantModel_ST)
| Variable | Type | Meaning |
|---|---|---|
| Gate_1 .. Gate_4 | BOOL | Bin discharge gates (interlocked outputs — the signal to the plant) |
| Mixer_Run | BOOL | Mixer motor (interlocked output) |
| Discharge_Gate | BOOL | Hopper discharge gate (interlocked output) |

### Sensors (written by PlantModel_ST, read by Controller_ST and Controller_LD)
| Variable | Type | Meaning |
|---|---|---|
| Weight_LoadCell | REAL (kg) | Weigh hopper weight |
| BinLevel_1 .. BinLevel_4 | REAL (kg) | Remaining material in each bin |
| Gate1_Closed .. Gate4_Closed | BOOL | Gate fully closed feedback (after travel delay) |
| Mixer_Done | BOOL | Mix cycle finished |

### Operator inputs (written by HMI/OPC UA, read by Controller_ST)
| Variable | Type | Meaning |
|---|---|---|
| Start | BOOL (rising edge) | Start batch cycle |
| Stop | BOOL (rising edge) | Batch-end stop: finish current batch, then IDLE (no next batch) |
| Recipe_Select | INT (1..3) | Recipe selection |
| Auto_Restart | BOOL | Auto-start next batch when TRUE |
| Target_Batches | INT (0 = unlimited) | Run N batches then stop at COMPLETE |
| Alarm_Reset | BOOL (rising edge) | Clear latched alarms, return to IDLE |

**Note:** `Recipe_Select` is latched at the `Start` rising edge — changes during a running batch apply to the NEXT batch only.

### Safety input (written by HMI/OPC UA, read by Controller_LD)
| Variable | Type | Meaning |
|---|---|---|
| E_Stop | BOOL | Emergency stop — NC contact in every Controller_LD output rung (hardware-style safety chain; drops all outputs instantly, regardless of state machine). NOT a fault — safety overrides process. |

### Status outputs (written by Controller_ST, read by HMI/OPC UA)
| Variable | Type | Meaning |
|---|---|---|
| Batch_State | INT (0..5) | 0=IDLE, 1=DOSING, 2=MIXING, 3=DISCHARGING, 4=COMPLETE, 5=FAULT |
| Batch_Counter | INT | Completed batches |
| Total_Weight | REAL (kg) | Cumulative weight dosed |
| Current_Target | REAL (kg) | Active recipe target for current bin |
| Alarm_Underweight | BOOL | Batch outside tolerance |
| Alarm_EmptyBin | BOOL | Bin ran empty mid-dosing |
| Alarm_Active | BOOL | Any alarm latched |

**Note:** `Total_Weight` is the cumulative production total (kg dosed across all completed batches) — NOT the live hopper weight (that's `Weight_LoadCell`). `Current_Target` is the active target for the bin currently being dosed — NOT a batch counter.

### Recipe table (array of 3 recipes × 4 bins, kg)
| Recipe | Bin1 | Bin2 | Bin3 | Bin4 | Total |
|---|---|---|---|---|---|
| 1 | 200 | 150 | 100 | 50 | 500 |
| 2 | 150 | 100 | 75 | 25 | 350 |
| 3 | 250 | 125 | 75 | 50 | 500 |

### Constants (GVL — shared)
| Constant | Value | Used by | Meaning |
|---|---|---|---|
| FLOW_1 .. FLOW_4 | 50 / 40 / 30 / 20 kg/s | PlantModel_ST | Bin flow rates (different sizes = realistic) |
| DISCHARGE_FLOW | 20 kg/s | PlantModel_ST | Discharge rate |
| MIX_TIME | 10 s | PlantModel_ST | Mix duration |
| GATE_TRAVEL | 0.5 s | PlantModel_ST | Gate open/close delay |
| CYCLE_TIME | 0.1 s | PlantModel_ST | Task cycle (100 ms) — used in integration math |
| TOLERANCE | 2% | Controller_ST | Weight tolerance |
| EMPTY_THRESHOLD | 10% of bin start level | Controller_ST | Empty-bin alarm trigger |

---

## 6. Controller_ST — Logic Spec (state machine, recipes, alarms)

State machine in **ST** using `CASE Batch_State OF` (SFC is an acceptable alternative; ST is simpler for MVP).

### States & transitions
| State | Entry condition | Actions | Exit condition |
|---|---|---|---|
| 0 IDLE | Power-on / after COMPLETE | All gates closed, mixer off | Start rising edge AND Weight < 5 kg AND no alarm |
| 1 DOSING | Start accepted | Sequence bins 1→4: open gate, wait until `Weight >= cumulative target_n` (sum of targets 1..n), close gate, wait `GateN_Closed`, move to next bin | All 4 bins dosed |
| 2 MIXING | All bins dosed | Mixer_Activate := TRUE; timer 10 s | Mixer_Done |
| 3 DISCHARGING | Mixer done | Discharge_Activate := TRUE; wait until Weight < 5 kg | Hopper empty |
| 4 COMPLETE | Hopper empty | Batch_Counter += 1; Total_Weight += recipe total; hold 2 s | Auto_Restart → back to 1 (unless Stop_Requested or batch limit `Batch_Counter >= Target_Batches`); else → 0 |
| 5 FAULT | Any alarm | All actuators off (safe state) | Alarm acknowledged + reset |

### Interlocks (must hold in every state)
- Never more than one gate open at a time
- Mixer can only start when all 4 gates confirmed closed
- Discharge only when mixer done
- Start only when hopper empty (Weight < 5 kg) and no active alarm
- Stop (rising edge) → batch-end stop: finish current batch (dose → mix → discharge → COMPLETE), then IDLE instead of next batch (not FAULT)
- E_Stop → instant drop of ALL outputs via Controller_LD rungs (safety chain, overrides every state — not a fault, not a machine state)

### Tolerance check
- **MVP decision (documented):** check on **final total weight** after all 4 bins dosed (before MIXING): if `|Weight_LoadCell − Recipe_Total| > Recipe_Total × 2%` → latch `Alarm_Underweight`, go to FAULT
- (Per-bin check catches compensating errors that break the recipe ratio — future enhancement, not MVP)

### Empty-bin check
- During dosing of bin n: if `BinLevel_n < EMPTY_THRESHOLD` → latch `Alarm_EmptyBin`, go to FAULT

### Alarm handling
- Alarms latch (stay TRUE until acknowledged). `Alarm_Reset` (BOOL rising edge) → clears latches, returns to IDLE.

### Controller_LD — Interlock layer (Ladder Diagram)
Controller_ST issues commands only; Controller_LD enforces physical permissives and drives the actuators:

- `Gate_1 := Gate1_Activate AND NOT E_Stop AND Gate2_Closed AND Gate3_Closed AND Gate4_Closed` (same pattern for gates 2–4 — NO contact of every OTHER gate's closed feedback) — gate n may open only when all other gates are physically closed
- `Mixer_Run := Mixer_Activate AND NOT E_Stop AND Gate1_Closed AND Gate2_Closed AND Gate3_Closed AND Gate4_Closed` — mixer only when all gates confirmed closed
- `Discharge_Gate := Discharge_Activate AND NOT E_Stop AND Mixer_Done` — discharge only after mix finished

---

## 7. PlantModel_ST — Physics Spec

Runs every cycle (100 ms). Pure math, no state machine.

```
// Gate travel delay (per gate): TON with GATE_TRAVEL
//   Gate1_Open_State := TON(Gate_1, 0.5s).Q    — plant-internal: flow only while physically open
//   Gate1_Closed     := TON(NOT Gate_1, 0.5s).Q — closed feedback to controller

// Flow integration (per open bin):
//   IF Gate1_Open_State THEN
//     Weight_LoadCell += FLOW_1 * CYCLE_TIME        // e.g. 50 * 0.1 = +5 kg per cycle
//     BinLevel_1 -= FLOW_1 * CYCLE_TIME
//   END_IF

// Discharge:
//   IF Discharge_Gate THEN
//     Weight_LoadCell -= DISCHARGE_FLOW * CYCLE_TIME
//   END_IF

// Clamps:
//   Weight_LoadCell := MAX(Weight_LoadCell, 0.0)
//   BinLevel_n := MAX(BinLevel_n, 0.0)

// Mixer done:
//   Mixer_Done := TON(Mixer_Run, MIX_TIME).Q
```

**Initialization:** on first cycle (or a `Init` flag), set `BinLevel_n` to 1000 kg each, `Weight_LoadCell := 0`.

**Why this is correct:** the controller sees weight rising at 5 kg per cycle, closes the gate at target, weight stops — exactly like a real plant. The controller never touches the physics.

---

## 8. OPC UA Configuration

1. In the device tree: right-click **Application** → Add Object → **Communication Manager**
2. Inside Communication Manager: Add Object → **OPC UA Server**
3. In OPC UA Server settings: enable the server, note the port (default 4840)
4. **Security settings** → generate a certificate (this fixes "OPC UA Server (not available)")
5. Add a **Symbol Set** (IEC Symbol Set Configuration) listing the variables to expose:
   - All operator inputs (Start, Stop, Recipe_Select, Auto_Restart, Alarm_Reset, Target_Batches, E_Stop)
   - All status outputs (Batch_State, Batch_Counter, Total_Weight, Current_Target, alarms)
   - Weight_LoadCell, BinLevel_1..4 (for the HMI later)
6. Login → Download → Start the PLC
7. **UaExpert**: add server `opc.tcp://localhost:4840` → connect → browse → find your variables
8. If UaExpert shows a certificate prompt: trust it (move from Quarantined to Trusted in CODESYS Security settings)

**Test:** write `Start := TRUE` from UaExpert → watch `Batch_State` and `Weight_LoadCell` change live.

---

## 9. Milestones (build order — do NOT skip)

| # | Milestone | Deliverable | Time est. |
|---|---|---|---|
| M1 | **Smoke test** | Toggle BOOL in PLC_PRG, OPC UA server up, UaExpert sees + writes the variable | 1 evening |
| M2 | **Controller state machine** | Controller_ST: CASE state machine + recipe table + alarms; Controller_LD: interlock rungs (no plant model yet — test with manual weight forcing) | 2–3 evenings |
| M3 | **Plant model** | Physics + gate delays + clamps; full cycle runs end-to-end in CODESYS visualization | 1–2 evenings |
| M4 | **OPC UA exposure** | Communication Manager + symbol set + UaExpert full control (start/stop/recipe/alarms) | 1 evening |
| M5 | **Stretch: Node-RED** | node-red-contrib-opcua subscribes, logs to SQL Server Express | 1–2 evenings |
| M6 | **Next project** | VB.NET HMI (OPC UA client), then Delphi HMI (HTTP→Node-RED or Modbus TCP) | later |

Each milestone must be demoable on its own — if time runs out before applying, M1–M4 alone is already a strong portfolio piece.

---

## 10. Verification Checklist (run before calling it done)

- [ ] Weight climbs at expected rate (50 kg/s → +5 kg per 100 ms cycle)
- [ ] Gate closes at target ±2%; weight stops rising
- [ ] Bins dose in order 1→2→3→4, never two gates open simultaneously
- [ ] Mixer runs 10 s after all bins dosed
- [ ] Discharge empties hopper (weight → 0, clamped, never negative)
- [ ] Batch_Counter increments; Total_Weight accumulates
- [ ] Recipe 2 and 3 produce correct totals (350 / 500 kg)
- [ ] Stop mid-cycle → safe stop, all actuators off
- [ ] Empty-bin fault triggers FAULT state (set BinLevel low to test)
- [ ] UaExpert: read all variables, write Start/Stop/Recipe_Select, see live changes
- [ ] Full cycle completes in under 2 hours (demo mode) — record a demo video

---

## 11. Interview Talking Points (what this proves)

- "I built a two-layer simulation: production-grade controller logic + a plant model that fakes the physics — the same pattern used in FAT testing"
- "IEC 61131-3: ST state machine, interlocks, tolerance checking, alarm handling"
- "OPC UA server configuration, certificates, symbol sets — real industrial communication"
- "The process mirrors CP's actual minibin dosing and silo-dryer batching projects"
- "Next: Node-RED bridge + SQL logging, then native HMIs in VB.NET and Delphi"

---

## 12. Constraints & Rules for the AI Assistant

1. Follow milestones in order; verify each before proceeding
2. Controller logic must stay hardware-realistic (no physics inside Controller_ST / Controller_LD)
3. Keep the project in English (variable names, comments) — it's a portfolio piece
4. Demo mode: 2-hour sessions — save/backup the project before long sessions; restart runtime when it stops
5. If OPC UA Server object is missing from Add Object despite the docs saying it's included: install the "CODESYS OPC UA Server" package via CODESYS Installer (sign in first), then retry
6. When stuck > 30 min on one issue: stop, document the error, ask the user before changing architecture
7. Every milestone ends with a short demo script (what to click, what to show)