# CLAUDE.example.md — Analog & Mixed-Signal Design with Virtuoso Bridge

> Copy this file to `CLAUDE.md` in your design project and adapt the **Project facts**
> section. It tells Claude how to drive Cadence Virtuoso / Spectre through
> `virtuoso-bridge` **and**, just as importantly, which mistakes to avoid — every
> "Lesson" below is a real failure that cost round-trips, with the verified fix.

---

## Project facts (EDIT THIS)

- **PDK:** `tsmcN28` (CRN28HPCP). Core devices: `nch_mac` (NMOS), `pch_mac` (PMOS).
  MOS terminals are `G/S/D/B`. 2-terminal `analogLib` devices (`res`, `idc`, `vdc`, `cap`)
  use `PLUS/MINUS`.
- **Scratch library:** `PLAYGROUND` — draw new cells here unless told otherwise.
- **Bridge target:** remote `user@host` (set in `~/.virtuoso-bridge/.env`); Virtuoso IC618, Spectre 19.x.
- **Venv:** `<repo>/.venv` (`uv venv .venv && source .venv/bin/activate && uv pip install -e .`).

---

## 0. Golden rules

1. **Verify before claiming done.** A schematic is finished only when `schCheck` markers == 0
   **and** a netlist read-back matches the intended topology. "It ran" ≠ "it's correct".
2. **Confirm topology before drawing.** "Diff pair", "current mirror", "OTA" each map to several
   real circuits. Ask which variant (loads? tail source? single/diff output?) — a 1-line question
   beats rebuilding.
3. **Prefer the highest-level API that works:** Python `client.schematic.*` / `client.layout.*`
   → inline SKILL → `.il` file. Drop down only when the level above can't do it.
4. **Never SSH in and run SKILL by hand.** All SKILL goes through the bridge
   (`client.execute_skill`, `virtuoso-bridge eval/load`).
5. **Remote files stay remote.** Anything Virtuoso/Spectre writes (netlists, logs, results) lives
   on the remote host — pull it with `client.download_file()`.

---

## 1. Connect & verify (do this first, in order)

```bash
source .venv/bin/activate
virtuoso-bridge status          # if .env already exists, skip `init`
virtuoso-bridge start           # starts tunnel + deploys daemon
virtuoso-bridge status          # MUST be healthy: [daemon] OK, [spectre] OK
virtuoso-bridge windows         # know which window numbers exist
```

- If status is **degraded** → the user must paste the printed `load(".../virtuoso_setup.il")`
  line into the Virtuoso CIW. You cannot do this for them.
- `VB_REMOTE_HOST` must be the **compute host running Virtuoso**, never the jump/bastion host
  (most common misconfiguration).

---

## 2. Standard design workflow

```
topology decision → place + wire (stubs) → schCheck==0 → set params → re-check → (sim)
```

**Draw a cell** (verified pattern):

```python
from virtuoso_bridge import VirtuosoClient
from virtuoso_bridge.virtuoso.schematic import (
    schematic_create_inst_by_master_name as inst,
    schematic_create_pin as pin,
    schematic_label_instance_term as termlabel,   # for 2-terminal devices
)
client = VirtuosoClient.from_env()

# delete-before-recreate — add_instance accumulates on top of an existing cell
client.execute_skill('when(ddGetObj("PLAYGROUND" "MYCELL") ddDeleteObj(ddGetObj("PLAYGROUND" "MYCELL")))')

with client.schematic.edit("PLAYGROUND", "MYCELL") as sch:
    sch.add(inst("tsmcN28", "nch_mac", "symbol", "M1", 3.0, 4.5, "R0"))
    # MOS terminals: auto-routed stubs, same net-name = same net
    sch.add_net_label_to_transistor("M1", drain_net="OUTN", gate_net="INP",
                                    source_net="TAIL", body_net="VSS")
    # 2-terminal device terminals (res/idc/cap/vdc): PLUS/MINUS
    sch.add(termlabel("R1", "PLUS", "VDD"))
    sch.add(termlabel("R1", "MINUS", "OUTN"))
    # pins at the circuit EDGE, never on instance terminals; connect by matching net name
    sch.add(pin("INP", -1.5, 4.5, "R0", direction="input"))
    # schCheck + dbSave run automatically on context exit
```

**Set device sizes/values** (handles the CDF two-step for you):

```python
from virtuoso_bridge.virtuoso.schematic.params import set_instance_params
client.open_window("PLAYGROUND", "MYCELL", view="schematic")   # REQUIRED first — see Lesson 3
set_instance_params(client, "M1", w="2u", l="100n", nf="4")    # MOS shorthand
set_instance_params(client, "R1", r="10k")                     # analogLib via kwargs
set_instance_params(client, "IREF", idc="20u")
```

**Verify** (always, before saying it's done):

```python
from virtuoso_bridge.virtuoso.schematic.reader import read_schematic
data = read_schematic(client, "PLAYGROUND", "MYCELL", include_positions=False)
# inspect data["nets"] / data["instances"][i]["terms"] against your intended netlist
markers = client.execute_skill('length(geGetEditCellView()~>markers)').output   # want "0"
```

**Read attributes** the safe way — one slot per eval, or batch with `fetch`:

```python
client.fetch("geGetEditCellView()~>instances", ["name", "cellName"])    # batch, 1 round-trip
client.execute_skill('car(setof(x geGetEditCellView()~>instances x~>name=="M1"))~>w').output
```

---

## 3. Lessons from failures (read before you repeat them)

> Format: **Symptom → Root cause → Correct way.** These are observed failures, not hypotheticals.

### Lesson 1 — Inline shell SKILL with `~>` or multiple forms returns empty / errors at a bogus line

- **Symptom:** `virtuoso-bridge eval --stdin` / `load file.il` returns `"output": ""` or
  `*Error* load: error while loading file ... at line N` where line N is just a closing paren.
- **Root cause:** nested-list return values and `~>` slot traversal don't survive the
  shell → CLI → SKILL quoting/serialization path reliably.
- **Correct way:** for anything non-trivial, write a **`.py` file** and use the Python client.
  Read attributes with `client.fetch(expr, fields)` or **one single-form `~>slot` eval each**.
  Reserve inline `eval` for simple, single-form expressions (`1+1`, `geGetEditCellView()~>cellName`).

### Lesson 2 — Guessing API method names (AttributeError)

- **Symptom:** `'SchematicEditor' object has no attribute 'add_net_label_to_instance_term'`;
  `'SchematicOps' object has no attribute 'open'`.
- **Root cause:** assuming a method exists from memory. The **batched `SchematicEditor`** exposes
  **only** `add()` and `add_net_label_to_transistor()`.
- **Correct way:** for 2-terminal stubs use the builder `schematic_label_instance_term(inst, term, net)`
  with `sch.add(...)`. Before calling any helper you haven't used this session, check the export
  list (`grep __all__ .../schematic/__init__.py`) or `references/schematic-python-api.md`. Don't
  fabricate names.

### Lesson 3 — `set_instance_params` → "no active schematic cellview"

- **Symptom:** `ValueError: no active schematic cellview`.
- **Root cause:** the helper targets `geGetEditCellView()`, which is unset until a schematic edit
  window is open and current.
- **Correct way:** call `client.open_window(lib, cell, view="schematic")` **once** first, then set
  params. Do **not** loop `geCreateEditWindow(...)` per call — that stacks GUI windows and caused a
  90 s **timeout (exit 124)**.

### Lesson 4 — Zoom/fit functions all error; canvas shows the wrong/empty view

- **Symptom:** `schZoomFit` / `hiZoomFit` / `hiZoomAbsoluteScale` / `geZoomFit` / `geFitToWindow`
  return `status: error`.
- **Root cause:** after any window open/close, the "current window" pointer goes **stale** (this is
  documented in `references/troubleshooting.md`), so window-less zoom calls target nothing.
- **Correct way:** target the window **explicitly** by number (from `virtuoso-bridge windows`) and
  zoom to the cellview bbox:
  ```python
  client.execute_skill('hiZoomIn(window(6) geGetEditCellView()~>bBox)')
  ```

### Lesson 5 — Parsing CLI JSON output fails

- **Symptom:** `json.JSONDecodeError: Expecting value: line 1 column 1`.
- **Root cause:** `virtuoso-bridge eval` prints a `using .env: ...` banner line before the JSON.
- **Correct way:** use the **Python client** (returns a structured `VirtuosoResult`) instead of
  parsing CLI stdout. If you must parse the CLI, strip non-JSON lines first
  (`grep -v "using .env"`).

### Lesson 6 — A GUI dialog freezes the whole SKILL channel

- **Symptom:** every `execute_skill` times out; a modal popup is on screen; `maeWaitUntilDone`
  returns nil.
- **Root cause:** a modal dialog blocks the CIW; or `?waitUntilDone t` deadlocked the event loop.
- **Correct way:** **never** use `?waitUntilDone t` — run sims async then `maeWaitUntilDone('All)`.
  Clear blockers with `virtuoso-bridge dismiss-dialog` (X11, bypasses the stuck channel) or
  `list-windows --json` + `dismiss-window <id> --action enter`. In optimization loops add
  `maeSaveSetup` + dialog recovery every iteration.

### Lesson 7 — `maeGetOutputValue` returns nil for computed expressions (gain, bandwidth, dB20)

- **Symptom:** scalar outputs read fine, but expressions over waveforms return nil.
- **Root cause:** Maestro saved only pre-computed scalars; raw waveforms were never written to PSF.
- **Correct way:** `maeSetEnvOption(test ?option "save" ?value "all")` + `maeSaveSetup()` **before**
  running; or parse the `results/maestro/<history>.log` file (tab-separated `expr\t\tvalue`).

### Lesson 8 — Changing `nf` is rejected; param display doesn't update

- **Symptom:** `SCH-1725 ... not editable` on `nf`; or a value changes but derived annotations stay stale.
- **Root cause:** PDK MOS `nf` is read-only; and `schHiReplace` alone doesn't fire CDF callbacks.
- **Correct way:** use **`fingers`** (not `nf`). Prefer `set_instance_params()` (does
  `schHiReplace` + `CCSinvokeCdfCallbacks` for you). If hand-rolling, run callbacks with
  `?order` for only the changed params (running all can fail on PDK vars like `mdlDir`).

### Lesson 9 — Polling on `system()` return codes hangs for the full timeout

- **Symptom:** a batch tool (`strmin`, `ihdl`, sometimes `spectre`) dies in seconds but the wrapper
  sleeps for its whole timeout.
- **Root cause:** `system()` rc is unreliable for fork-and-log tools.
- **Correct way:** (1) `upload_file()` any local file args to the tool's cwd first, and (2) in the
  poll loop `tail -n 200 <tool.log>` for terminal markers (`Translation failed`, `OPEN_FAILED`,
  `ERROR`) and fast-exit on them — don't wait only for the success artifact.

---

## 4. Mixed-signal specifics

- **Two independent services.** The **Virtuoso daemon** (SKILL) and **Spectre** (sim) are
  decoupled — `virtuoso-bridge status` reports both. You can netlist/sim without the GUI bridge and
  edit schematics without Spectre.
- **Analog vs digital handoff.** Routed digital blocks come in via standalone batch tools
  (`strmin` for GDS, `ihdl` for Verilog→schematic) wrapped in SKILL `system()` — see Lesson 9 and
  `examples/01_virtuoso/digital_import/`.
- **PDK paths live in the netlist.** To find model includes/sections, export a netlist
  (**Simulation > Netlist > Create**); the `.scs` header has the `include ... section=...` lines.
- **Body/bulk nets matter in analog.** Default NMOS bulk → VSS, PMOS bulk → VDD; for isolated wells
  use the `*_dnw_mac` deep-nwell devices. State this explicitly rather than assuming.

---

## 5. Definition of done

A task is complete only when **all** hold — state each one plainly, don't hedge:

- [ ] `schCheck` markers == 0 (`length(geGetEditCellView()~>markers)` → `0`).
- [ ] Netlist/connectivity read back matches the intended topology (verified, not assumed).
- [ ] Device params confirmed on the actual instances (read `~>w/~>l/~>r/~>idc`, not just "I set them").
- [ ] For sims: results bound to the exact `history` returned by `maeRunSimulation`; no stale dialog.
- [ ] If anything was skipped or failed, say so with the evidence — never report unverified success.
```
