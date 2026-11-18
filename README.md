# Pitwall

Desktop application for working with the nodes on the car over CAN FD: live monitoring, node status, parameters, faults and firmware flashing.

The UI is in English and the app works fully offline.

## Tech stack

- Python 3.12
- PySide6 (GUI)
- pyqtgraph (plots)
- python-can (PCAN backend `pcan`, CAN FD with up to 64 B per frame)
- SQLite via the standard library `sqlite3` (last known state only)
- pytest

## Design rules

- **No hardcoding.** CAN IDs, message contents, parameters, faults and nodes live in config files. Code only reads them.
- **Easy to extend.** A new message, signal, parameter or fault is a config change. A new screen is a new folder plus one line in the screen list.
- **Testable without the car.** Everything is developed against a simulator that runs as a separate program, and against wrappers around the CAN interface.
- **The GUI never freezes.** CAN reading runs in its own thread, screens refresh from a timer (about 20 Hz).
- **Screens are isolated.** Each screen lives in its own folder under `screens/` and never imports another screen. Shared code lives in `app/`.

## Project structure

```
pitwall-software/
  main.py                 # only starts the app
  config/
    can_messages.toml     # ALL CAN FD messages: ID, length, signals
    nodes.toml            # nodes on the car
    faults.toml           # fault codes and descriptions
    ams_view.toml         # cell/segment counts, colour thresholds for the AMS view
    params/
      vcu.toml            # per-board parameters with limits
      ams-master.toml
      lv-battery.toml
      imu.toml
  pitwall/
    app/                  # framework: window, CAN wrapper, config loader, SQLite
    screens/
      nodes/
      firmware/
      parameters/
      monitor/
      faults/
    sim/                  # simulator (separate program)
  tests/
```

Each screen folder has at least `view.py` (widgets) and `model.py` (logic, no widgets). A screen receives CAN data already decoded and addressed by signal name from the config, never by numeric ID.

## Getting started

```bash
pip install PySide6 pyqtgraph python-can pytest

# start the simulator (acts as the car on the bus)
python -m sim

# start the app against the simulator
python main.py --sim

# run tests
pytest
```

The exact flags and settings may change as the app takes shape. The implementation is chosen at startup (for example `--sim` or a setting in the app).

## Configuration

Config format is TOML (a proposal; it can be replaced as long as there is one file for CAN messages and one file per board for parameters).

| To add...                         | Change...                                              |
|-----------------------------------|--------------------------------------------------------|
| CAN message or signal             | `config/can_messages.toml`                             |
| Node                              | `config/nodes.toml` + new file in `config/params/`     |
| Parameter or change its limits    | `config/params/<board>.toml`                           |
| Fault                             | `config/faults.toml`                                   |
| Screen                            | new folder in `screens/` + one line in the screen list in `app/` |
| Plot or signal display            | chosen in the app from the signals in the config       |

If any of these needs a code change, something is hardcoded and should be moved to config.

### Examples

`can_messages.toml` (all values are placeholders):

```toml
[[message]]
name = "AMS_CellVoltages_A"
id = 0x000              # not yet defined
node = "ams-master"
length = 64
period_ms = 100

[[message.signal]]
name = "cell_v"
start_bit = 0
length = 16
scale = 0.001
offset = 0
unit = "V"
repeat = 32             # cell_v[0..31], 16 bits each
```

`nodes.toml`:

```toml
[[node]]
key = "vcu"
name = "VCU"
heartbeat = "VCU_Heartbeat"   # message name from can_messages.toml
timeout_ms = 1000
params = "params/vcu.toml"
```

`params/<board>.toml`:

```toml
board = "vcu"

[[param]]
name = "example_limit"
id = 0x0000             # not yet defined
type = "u32"            # u32 | i32 | f32 | bool | enum
unit = "%"
min = 0
max = 100
default = 80
group = "Torque"
description = "Example parameter"
read_only = false
```

`faults.toml`:

```toml
[[fault]]
code = 0x0000           # not yet defined
node = "vcu"
name = "EXAMPLE_FAULT"
severity = "critical"   # critical | warning | info
description = "Example fault"
action = "What the driver or team should check"
related_params = ["example_limit"]
```

## Wrappers, simulator and tests

The app never talks to PCAN or the node protocol directly. It always goes through a wrapper, so the same code runs against the simulator and, later, on the car.

| Wrapper         | Purpose                                                    | Implementations                                              |
|-----------------|------------------------------------------------------------|--------------------------------------------------------------|
| `CanInterface`  | open the bus, send/receive CAN FD frames, decode by config | `pcan` (car), `udp_multicast` (simulator in another process) |
| `DeviceService` | read/write parameters, read/clear faults                   | `FakeDeviceService` (in memory), later `CanDeviceService`    |
| `FlashBackend`  | check and flash firmware                                   | `FakeFlashBackend` (fake progress), later the real one       |

`python-can`'s `virtual` interface only works inside one process, so the simulator uses `udp_multicast`, which supports CAN FD on Windows and Linux.

### Simulator (`python -m sim`)

- reads the same `can_messages.toml` and `nodes.toml` as the app (no duplicated definitions)
- sends every message at its configured period, with values generated within limits (sine, ramp, random)
- supports scenarios in separate files, e.g. node drops out, cell undervoltage, a fault appears
- will answer app requests once the protocol is defined

### Tests

- **Config tests:** messages at most 64 B, signals fit in the message, unique names, parameter defaults between min and max.
- **Decoder:** for every message, build bytes from known values and check the decoder returns the same values.
- **Screen models:** tested with `FakeDeviceService`, `FakeFlashBackend` and frame lists, without a window or PCAN.
- `pytest` must pass before every PR.

## Storage

SQLite keeps only the last known state, never history. Each change updates the existing row (`INSERT ... ON CONFLICT DO UPDATE`).

- One file, `pitwall.db`, in the user's app data folder (`QStandardPaths.AppDataLocation`).
- Access only through one class in `app/` (e.g. `StateStore`). Screens never write SQL.
- CAN frames and signals are not stored. Live data stays in memory while the app runs.
- On the next start the app shows the previous session's state, marked as "last session" (greyed out) until fresh data arrives.

| Table         | Key                  | Contents                                           | Updated by |
|---------------|----------------------|----------------------------------------------------|------------|
| `node_state`  | node                 | state, firmware version, last heartbeat time       | Nodes      |
| `param_values`| node + param name    | last read value, read time                         | Parameters |
| `fault_state` | node + code          | active, counter, last occurrence time              | Faults     |
| `ui_state`    | key                  | last interface, selected signals, selected tabs    | all screens|

## Screens

### Nodes
Connection to PCAN (Connected/Disconnected, interface choice, Connect/Disconnect), cards (interface, nominal and data bitrate, bus load %, controller state), node table (name, state, firmware version, time since last heartbeat, actions to open Parameters or Firmware), warnings list, and a session log. A node without a heartbeat for longer than `timeout_ms` is shown as `NO RESPONSE`. Table columns are defined in one place as a list of signal names; missing signals show an empty cell and never crash the app.

### Firmware
Target node selection with current version, `.elf` file selection from a local folder, a list of pre-flash checks (Flash is enabled only when all pass), progress bar with current step and log, and an always visible Abort button during flashing. Flashing runs in the background. The screen knows nothing about the bootloader: checks, steps and flashing all come from a `FlashBackend` (`checks`, `steps`, `flash`, `abort`). Only `FakeFlashBackend` exists for now.

### Parameters
One tab per node, group list with search, and a table (Parameter, On device, New value, Unit, Range, Default). Changed values are highlighted, out-of-range values are red with a reason, and a value outside min/max is never sent to a node. Buttons: Read from device, Write, Save to flash, plus a counter of unsent changes. Cell editors are chosen by parameter `type`. Uses only `DeviceService` (`read_param`, `write_param`, `save_params`).

### Live Monitor
Bound exclusively to real CAN FD traffic, decoded from `can_messages.toml`. Nothing is stored.

- **Signals:** all signals grouped by node with search and checkboxes, plots (5/10/30/60 s window), current values with units, Pause (freezes the display, reception continues).
- **AMS:** battery monitoring view carried over from last year's app. It receives already decoded signals by name (e.g. `cell_v[0..n]`, `cell_t[0..n]`); cell/segment/sensor counts and colour thresholds are in `config/ams_view.toml`.
- **Raw trace:** the last few thousand frames (time, ID, length, hex data, decoded message name), filter by ID or name, frames per second per message.

Decoding runs on receive, drawing runs from the ~20 Hz timer, and plots use in-memory ring buffers. Tabs are registered in one place, so a new view is a new tab.

### Faults
Filters (All, Active, by node, by severity, with counters), a table (state active/stored, code, node, description, first seen, last seen, count), details of the selected fault (description, action, values at the time, related parameters with an Open parameters button), an active-fault badge next to "Faults" in the sidebar, and Clear stored (with confirmation) and Export report. An unknown code is shown as `UNKNOWN` with its raw data. Live faults come from signals named in `faults.toml`; stored faults are read and cleared through `DeviceService` (`read_faults`, `clear_faults`). Opening Parameters goes through a shared signal in `app/`, not by importing another screen.

## What is defined and what is not

| Topic                                            | Status        | Handling for now                                |
|--------------------------------------------------|---------------|-------------------------------------------------|
| CAN FD, 64 B, PCAN                               | defined       | use as is                                       |
| Screens and look                                 | defined (mockups) | build from the mockups, improvements allowed |
| Folder per screen, config files, simulator       | defined       | as in this document                             |
| SQLite, last state only                          | defined       | see Storage                                     |
| CAN IDs                                          | not defined   | made-up values in `can_messages.toml`           |
| Message contents (signals, bytes, factors)       | not defined   | made-up values in `can_messages.toml`           |
| Node list and heartbeat                          | not defined   | `nodes.toml` with the nodes from the mockups    |
| Parameters and their limits                      | not defined   | one file per board in `config/params/`          |
| Fault codes                                      | not defined   | `faults.toml`                                   |
| CAN bootloader and flashing procedure            | not defined   | display only, open interface (`FlashBackend`)   |
| Protocol for reading/writing parameters, faults  | not defined   | `DeviceService`, answered by the simulator      |
| MCUs on the boards                               | not defined   | does not affect the app                         |

Anything in the "not defined" rows must not be written into code. It goes into config files with placeholder values that will be replaced later.

## Team and workflow

| Area       | Žan                                                              | Nino                                                       |
|------------|------------------------------------------------------------------|------------------------------------------------------------|
| Screens    | Live Monitor (Signals, AMS, Raw trace), Firmware (+ `FakeFlashBackend`), Nodes | Parameters, Faults                       |
| Framework  | `CanInterface` (pcan + udp_multicast), config reader and message decoder, simulator in `sim/` | main window and navigation, `StateStore` (SQLite), `DeviceService` + `FakeDeviceService` |
| Tests      | own screens and own part of the framework                        | own screens and own part of the framework                  |

Before starting, agree on the interfaces between the two parts:

- how a screen gets decoded signals by name (e.g. `subscribe(["cell_v"], callback)`)
- the screen interface (how it registers in the main window and what it receives on start)
- `DeviceService` and `FlashBackend` (function names and return values)
- config file formats

Until the decoder exists, Faults can be tested with fake signals sent directly to the callback.

Workflow: short branches and one PR per screen or framework part, reviewed by the other team member. Each person writes their own tests and test wrappers. Anything unclear, undefined or badly designed (including in the spec) should be reported and discussed. Architecture suggestions are welcome.
