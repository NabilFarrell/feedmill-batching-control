# Hardware Reference

**This is a reference bill of materials for what a real installation of this system would
require. It is not a parts list of hardware that was installed.** This project runs entirely
as a simulation — CODESYS Control Win V3 soft PLC plus an Ignition Gateway, with no physical
process.

The purpose of this document is to show that the control logic maps onto real, purchasable
equipment, and that each simulated variable has a genuine industrial counterpart.

## Variable → hardware mapping

| Simulated variable | Real hardware equivalent | Notes |
|---|---|---|
| `Weight_LoadCell` | 4-point shear beam load cell + weight transmitter | Transmitter outputs 4–20 mA or RS-485; load cell is the only sensor that measures the actual recipe target |
| `BinLevel_1` … `BinLevel_4` | Bin level sensor — rotary paddle, capacitive, or load cell on the bin | Simulated as remaining kg; a real bin would typically use a low-level switch plus a high-level switch |
| `Gate_1` … `Gate_4` | Pneumatic slide gate + 5/2 solenoid valve | Material flow from bin to weigh hopper |
| `Gate1_Closed` … `Gate4_Closed` | Reed or inductive limit switch | Closed-position feedback; the controller waits for this before advancing to the next bin |
| `Gate1_Activate` … `Gate4_Activate` | Solenoid coil output via interposing relay | Controller command, enforced by the ladder interlock layer |
| `Mixer_Run` | Mixer motor + VFD (VFD run/fault contacts) | Fixed-duration mix cycle |
| `Discharge_Gate` | Pneumatic discharge gate + solenoid + limit switch | Hopper to downstream packaging |
| `E_Stop` | 2-channel NC mushroom + safety relay | Hardwired, independent of the PLC in a real installation |
| `SAFETY_Lock` | Safety relay auxiliary contact | GREEN = safe, RED = E-Stop latched |
| `Alarm_*` tags | Alarm annunciator / SCADA alarm system | Priority, acknowledgement, and journal |
| `Batch_State`, `Batch_Counter` | HMI / SCADA status registers | Exposed over OPC UA |
| — | PLC (IEC 61131-3 capable) | WAGO, Beckhoff, Phoenix Contact, or equivalent soft/hard PLC |
| — | Industrial PC or panel PC for the HMI | Running the Ignition Gateway |
| — | Ethernet network | OPC UA between PLC and Gateway |
| — | MCC, breakers, contactors | Power distribution for motors and actuators |

## Why the architecture maps cleanly

The two-layer split in this project is not an academic exercise — it is the standard FAT
(Factory Acceptance Testing) pattern:

- **Controller_ST / Controller_LD** would run unchanged on real hardware. They read sensors
  and write command outputs, and they contain no knowledge of the simulation.
- **PlantModel_ST** is replaced by the actual physical process.

That is what makes the control logic verifiable before commissioning: the same code that
runs in simulation is the code that ships to the plant.

## Safety architecture

In this simulation the E-Stop is a **software latch** in `Controller_LD`, mimicking a
hardwired safety relay. In a real installation:

- The E-Stop mushroom is hardwired directly to the safety relay — independent of the PLC
- The safety relay drops power to all hazardous actuators (motor contactors, VFDs, valves)
- The PLC sees the E-Stop input and forces a safe state in its own logic
- Restart requires three deliberate steps: release the button, reset the safety relay,
  acknowledge in the PLC

The simulation implements the same three-step logic so the operator workflow is faithful,
even though the physical layer is modelled rather than wired.
