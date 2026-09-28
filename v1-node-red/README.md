# v1 — Node-RED HMI Layer (Archived)

The original HMI for this project, retained as the prototyping layer that preceded the
Ignition Perspective rebuild.

## Files

| File | Description |
|------|-------------|
| `flows_codesys_opcua.json` | Node-RED flow — OPC UA subscription and write clients, dashboard widgets, alert filtering, CSV logging |

Related archived media: `../screenshots/v1-node-red/` and `../video/v1-node-red/`.

## Why it was replaced

Node-RED served as a fast prototyping layer, and the PLC half of this project was validated
end to end against it. It was replaced with Ignition Perspective for one substantive reason:

**Node-RED has no alarm management.** The dashboard rendered alarm conditions as coloured
text and a hand-built table. Ignition provides a real alarm system — priority,
acknowledgement lifecycle, shelve and cleared states, and a persistent journal — which is
what industrial operators and ISA-18.2 actually expect.

Everything else about the HMI also moved to Ignition: ISA-101 state colour coding, per-bin
fault identification, page-based navigation with a docked header, a persistent alarm
banner, a historian-backed trend, and a SQLite batch ledger with named queries.

## Importing the flow

1. Open Node-RED (typically `http://127.0.0.1:1880`)
2. Menu → **Import** → select **import nodes**
3. Paste the contents of `flows_codesys_opcua.json`
4. Click **Deploy**

Requires the `node-red-contrib-opcua` and `node-red-dashboard` packages. The OPC UA endpoint
in the flow points at the local CODESYS PLC and must be adjusted for your machine.

## Note on recipe naming

This archived flow uses an earlier recipe set — Poultry / Pig Growler / Cattle. Recipes
were later revised to Poultry / Cattle / Supplement with an updated bin matrix. The current
names are authoritative; see the README recipe table.

The PLC code in `../code/` is unchanged between the two HMI layers.
