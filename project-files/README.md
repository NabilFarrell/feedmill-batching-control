# Project Files

Raw project files that can be imported into the respective tools.

## Files

| File | Tool | How to use |
|------|------|------------|
| `flows.json` | Node-RED | Import via Node-RED menu → Import → Clipboard → paste JSON |

## CODESYS Project

The CODESYS project (`Feedmill Control.project`) is not included in this folder because:
1. CODESYS project folders contain many internal files (compiled, backup, etc.)
2. The full project is too large for a GitHub repo
3. The code exports in `/code/` contain the readable source

If you need the CODESYS project files, contact me directly.

## Node-RED Import

To import the flow:
1. Open Node-RED (http://127.0.0.1:1880)
2. Menu → Import → select "import nodes"
3. Paste the contents of `flows.json`
4. Click Deploy

Note: You will need `node-red-contrib-opcua` and `node-red-dashboard` packages installed.
