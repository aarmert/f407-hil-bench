# f407-hil-bench

A hardware-in-the-loop test bench for the STM32F407 Discovery that a person or a coding agent can drive
from the command line. `bench.py` flashes and resets the board, talks to it over a serial port, watches
its LEDs through an Android phone camera, and runs test plans written in TOML. Every command takes
`--json` and returns a machine-readable verdict with the likely causes of a failure, so an agent can act
on the result without reading a log. When a test leaves the board hung or crashed, the tool recovers it
by itself.

Every test also has to show that it can fail: `--verify-can-fail` runs a deliberately broken copy of
each plan, and that copy must FAIL.

This repo is part of [tezgah-lab](https://github.com/aarmert/tezgah-lab), together with
[c031-ab-bootloader-stress](https://github.com/aarmert/c031-ab-bootloader-stress), which uses a copy of this `bench.py`.

**Contents:** [Results](#results) · [Try it without hardware](#try-it-without-hardware) ·
[Requirements](#requirements) · [Wiring](#wiring) · [Commands](#commands) ·
[Writing a test plan](#writing-a-test-plan) · [The camera tool](#the-camera-tool) ·
[Production station](#production-station-station) · [Using it with an agent](#using-it-with-an-agent-claude-code-opencode) ·
[Repo layout](#repo-layout)

## Results

The `smoke`, `crash_recovery` and `measure` plans pass on the real board (2026-10-28), and each one FAILS when its expectation is broken on
purpose. The reports are in [`results/runs/`](results/runs/); each folder has `report.md`,
`result.json` and `serial.log`.

| Plan | What it checks | Pass | Negative check (must fail) |
|---|---|---|---|
| `smoke` | boot banner, `PING`, `VER`, `TICK` range, `UID`, an LED, an unknown command | `20261028-184415_smoke`, `20261028-184434_smoke` | `20261028-184417_smoke_negatif` |
| `crash_recovery` | a deliberate HardFault is reported, the board then stays silent, and it answers again after a reset | `20261028-184419_crash_recovery`, `20261028-184434_crash_recovery` | `20261028-184422_crash_recovery_negatif` |
| `measure` | supply voltage, chip temperature, board clock against the PC clock | `20261028-184425_measure`, `20261028-184436_measure` | `20261028-184431_measure_negatif` |

Redactions: the host name, home directory, ST-Link serial number and board UID in these reports were
replaced with placeholders, so the `audit_hash` values no longer match the files byte for byte.

## Try it without hardware

You need [uv](https://docs.astral.sh/uv/) and `arm-none-eabi-gcc` (a plan flashes the firmware, so it
has to be built). `bench.py --sim` runs against a simulated board and needs no `bench.toml`.

```bash
git clone https://github.com/aarmert/f407-hil-bench.git && cd f407-hil-bench
uv run firmware/build.py
uv run bench.py --sim run smoke --verify-can-fail          # 10/10, and the broken copy FAILS
uv run bench.py --sim run crash_recovery
uv run bench.py --json --sim send PING --expect '^PANG$'    # a wrong expectation, to see a failure
```

The last command fails on purpose. This is what a failed result looks like:

```json
{
  "ok": false,
  "command": "send",
  "error_type": "expect_timeout",
  "summary": "No /^PANG$/ reply to 'PING' within 2 s.",
  "likely_causes": [
    "The board may have crashed or hung earlier: try 'bench.py reset --expect \"^BOOT\"'",
    "TX/RX wires swapped, or GND not connected"
  ],
  "details": "Recent serial traffic:\n> PING\n< PONG"
}
```

## Requirements

| For | You need |
|---|---|
| Everything | [uv](https://docs.astral.sh/uv/) (installs Python and `pyserial`) |
| Flashing and debugging | OpenOCD, or STM32CubeProgrammer |
| Building firmware | `arm-none-eabi-gcc` |
| Camera steps | `adb`, scrcpy 2.5+, `ffmpeg`, [Bun](https://bun.sh), the `camera` group (opencv, numpy; `uv sync` installs it), the camera tool (see [The camera tool](#the-camera-tool)), and an Android 12+ phone with USB debugging |

Config: copy `bench.example.toml` to `bench.toml` (gitignored) and set the serial port.

## Wiring

```
Discovery mini-USB (ST-Link)  ->  computer USB   (flashing and reset)

USB-TTL (3.3V)                    Discovery
  RX   <----------------------    PA2  (USART2 TX)
  TX   ---------------------->    PA3  (USART2 RX)
  GND  -----------------------    GND
  VCC  do not connect! (the board is powered over USB)
```

Make sure the USB-TTL adapter is in 3.3V mode. If it has a jumper, set it to 3V3.
The CN3 jumpers on the Discovery must be fitted (ST-Link connected to the board).

## Ubuntu setup

```bash
bash setup_ubuntu.sh
# log out and back in, then:
uv run bench.py ports        # find the USB-TTL's name and write it to bench.toml
uv run bench.py doctor
uv run bench.py run smoke
uv run bench.py run crash_recovery
```

The script installs [uv](https://docs.astral.sh/uv/) (unless it's already installed) and runs
`uv sync --locked`. uv uses the Python version in `.python-version`, downloading it if the system
doesn't have it, and installs the packages at the versions in `uv.lock` into `.venv`.

Read `setup_ubuntu.sh` before you run it. Besides installing packages with `sudo` and adding you to the
`dialout` and `plugdev` groups, it stops the machine from suspending (it masks the sleep targets and
ignores the lid switch). That suits a dedicated bench computer and probably not your laptop. It does
not install the camera tools (`adb`, scrcpy 2.5+, `ffmpeg`, Bun). Install those yourself, then set up the camera tool as described in [The camera tool](#the-camera-tool).

Use the `/dev/serial/by-id/...` name from the `bench.py ports` output. After a reboot, `/dev/ttyUSB0`
can point to a different device. The by-id name stays the same.

## Windows setup

1. Install [uv](https://docs.astral.sh/uv/) and run `uv sync` in the repo.
2. Install STM32CubeProgrammer (it includes the ST-Link driver). For the compiler, install STM32CubeCLT or the Arm GNU Toolchain.
3. Copy `bench.example.toml` to `bench.toml`. Set `backend = "cubeprog"` and `port = "COM5"` (find the port in Device Manager or with `uv run bench.py ports`).
4. `uv run firmware\build.py`, `uv run bench.py doctor`, `uv run bench.py run smoke`

`halt-dump`, `mem`, and `gdbserver` are not available with the `cubeprog` backend.

## Commands

| Command | What it does |
|---|---|
| `uv run bench.py doctor` | Checks the tools, the ST-Link, and the serial port |
| `uv run bench.py ports` | Lists serial ports |
| `uv run bench.py flash file.elf` | Flashes and verifies |
| `uv run bench.py reset --expect "^BOOT"` | Resets and waits for the boot message |
| `uv run bench.py send PING --expect "^PONG$"` | Sends a command and checks the reply |
| `uv run bench.py send TICK --expect "^TICK (\d+)$" --min 1 --max 100000` | Checks a numeric range |
| `uv run bench.py read --seconds 3` | Reads from the serial port |
| `uv run bench.py monitor` | Live serial monitor (Ctrl+C to quit) |
| `uv run bench.py run smoke` | Runs the `plans/smoke.toml` plan |
| `uv run bench.py run smoke --verify-can-fail` | Also runs a deliberately broken copy of the plan, which must FAIL |
| `uv run bench.py recover` | Recovers the board by flashing the known-good firmware under reset |
| `uv run bench.py audit` | Checks that nobody has altered the audit log |
| `uv run bench.py station` | Production station: asks for the operator name, runs smoke+measure on each board, shows a large PASS/FAIL, `q` to quit |
| `uv run bench.py halt-dump` | Halts the core, shows the registers, the fault registers with descriptions, and the pc and stack at the crash, then resumes |
| `uv run bench.py mem 0x40020C14 --count 4` | Reads memory or peripheral registers (read only, the option byte region is blocked) |
| `uv run bench.py gdbserver` | Starts a GDB server (for humans, Ctrl+C to stop; `sudo apt install gdb-multiarch`) |

`halt-dump`, `mem`, and `gdbserver` work only with `backend = "openocd"`, and `halt-dump` and `mem`
have F407 constants, so they give wrong results on the C031.
Reading some peripheral registers has side effects. For example, reading the USART `DR` consumes the received character.

To get the result as JSON (`ok`, `error_type`, `summary`, `likely_causes`), put `--json` before the
subcommand: `uv run bench.py --json run smoke`. `--sim` goes in the same place and runs the command
without hardware. After the subcommand, `bench.py` rejects both.

Each `run` creates a `runs/<YYYYmmdd-HHMMSS>_<plan>/` folder with `report.md`, `result.json`,
and a timestamped `serial.log`. If a test fails, the tool tries to recover the board itself. It resets
the board first, and if that does not work, it flashes the known-good firmware again.
If the board crashes, the tool maps the address in the HARDFAULT line to a function and line in the source code.

## Writing a test plan

The plans in `plans/` are working examples; [`plans/smoke.toml`](plans/smoke.toml) is the shortest.
The steps below and `cam_blink` in the next section are all eight actions.

```toml
name = "example"
firmware = "firmware/build/stm32bench-fw.elf"

[[step]]
action = "flash"            # flash the plan's firmware

[[step]]
action = "reset"

[[step]]
action = "expect"           # wait for a line that matches this pattern
pattern = '^BOOT '
timeout = 3

[[step]]
action = "send"             # send and check the reply
text = "TICK"
expect = '^TICK (\d+)$'
range = [1, 600000]         # the number in the capture group must be in this range

[[step]]
action = "silence"          # no output may arrive during this time
seconds = 1

[[step]]
action = "delay"
seconds = 0.5

[[step]]
action = "clock_check"      # TICK, wait 5 s on the PC, TICK again
seconds = 5
range = [0.97, 1.03]        # board time / PC time (time.monotonic) must be in this range
```

Patterns are regular expressions. Use single quotes in TOML so you don't need to escape backslashes.

`clock_check` writes the ratio, both durations, and the uncertainty of the PC measurement (half the TICK
round trip). If the last expectation step is a `clock_check`, `--verify-can-fail` breaks its range.

Every new plan should pass `--verify-can-fail`. A test that does not fail when the expectation is
deliberately broken proves nothing.

## Measuring an LED with the camera (`cam_blink`)

```toml
[[step]]
action = "cam_blink"        # record a video with the phone and measure the LED's blink frequency
seconds = 4                 # recording length
range = [1.8, 2.2]          # Hz
# duty_range = [0.4, 0.6]   # optional duty cycle range
# skip_s = 0.5              # skip the first frames (let exposure and focus settle)

[[step]]
action = "cam_blink"        # is the LED steady? The region and threshold come from the previous cam_blink step
seconds = 3
steady = "on"               # or "off"
```

- The `camera/` tool (`camera/scripts/record-video.ts`, scrcpy) records the video. Frame times come from
  the video's own timestamps (ffprobe).
- The LED region is the 30×30 pixel spot whose brightness changes most across the frames. If the LED's
  reflection on a shiny surface changes more, the tool picks the reflection. The frequency comes out the same,
  but check the marked frame. To fix the region, add `[camera] roi = [x, y, width, height]` to `bench.toml`.
- The verdict is numeric. The threshold is the midpoint between the 5th and 95th brightness percentiles, transitions
  use hysteresis, and the tool finds each transition time by linear interpolation between two frames.
  It computes the frequency from both rising and falling edges.
- If the measured frequency is above a quarter of the frame rate, the message starts with `WARNING:`.
  Frequencies above half the frame rate alias, so they look lower than they are.
- Each step writes `cam_stepNN.mp4`, `cam_stepNN_roi.jpg` (region marked, other candidates as yellow
  circles), and `cam_stepNN.csv` (time, brightness) to the report folder.
- With `--sim`, the tool uses a synthetic ffmpeg video, so no phone or camera tool is needed.
- If Bun is not at `~/.bun/bin/bun`, or more than one phone is attached, set `bun` or `serial` under
  `[camera]` in `bench.toml`. `bench.example.toml` lists the keys.

## The camera tool

The `cam_blink` step runs the phone tool's `scripts/record-video.ts` (Bun + TypeScript, scrcpy). The
tool is not part of this repo and is not published. `bench.py` looks for it in `camera/` next to
`bench.toml` by default (`camera/` is gitignored); set `tool_dir` under `[camera]` in `bench.toml` to
use another folder (see `bench.example.toml`). With `--sim`, no tool is needed.

## Production station (`station`)

```bash
uv run bench.py station                              # asks for the operator, waits for Enter on each board, q to quit
uv run bench.py station --plans smoke,measure,camera_blink
uv run bench.py station --rounds 3 --no-prompt --operator Ayse   # for testing and automation
```

- For each board, the plans run in order. The station stops at the first failing plan and shows the result as a large PASS/FAIL.
- For each board, the station writes a `station-card` line to the audit log (`runs/audit.jsonl`). The hash
  of that line goes into the header of every report for that board. The header also has the operator, UID,
  firmware SHA-256, bench.py and OpenOCD versions, host name, and start and end times.
- `runs/station_<date>.csv` has one row per board. The columns are `time`, `board_no`, `operator`, `uid`,
  `firmware_sha256`, `result`, `failed_step`, `report_dirs`, `repeat`, and `audit_hash`. Files written before
  the columns were renamed have Turkish headers; they are still read for the repeat check.
- If the same UID has been tested before, the screen shows a `!!!` warning that the board was tested before,
  and the CSV's `repeat` column is filled in. The check covers all `station_*.csv` files.
- Runs with `--sim` go to `station_<date>_sim.csv`, so they never mix with the real log.
- While the station is open, it holds the bench lock. Other bench.py commands wait until it closes.

## Test firmware commands

`PING`, `VER`, `TICK`, `UID`, `VDD` (supply voltage in mV), `TEMP` (chip temperature in °C, 0.1 resolution),
`LED <G|O|R|B> <ON|OFF>`, `BLINK <1-20>`
(the green LED's blink rate, for the camera test), `CRASH` (a deliberate HardFault),
`RESET`. At boot it prints `BOOT stm32bench-fw v1.2.0 reset=<cause>`. The green LED
blinks at 2 Hz while the main loop runs, and stops if the board hangs.

The firmware targets the STM32F407G-DISC1 and STM32F411E-DISCO (LEDs on PD12-PD15). `TEMP` is correct
only on the F407, because the F411's temperature sensor is on ADC channel 18, not 16.
The clock is the internal 16 MHz HSI, with no PLL setup.

## Common errors

| Error | Fix |
|---|---|
| `probe_not_found` | Check the USB cable and the `plugdev` group (log out and back in), or another program is holding the ST-Link |
| `target_not_responding` | `uv run bench.py recover` |
| `serial_open_failed` | Wrong port name, or you are not in the `dialout` group |
| `expect_timeout` with no lines at all | TX/RX swapped, GND not connected, or the USB-TTL is in 5V mode |
| Garbled characters | Baud rate mismatch (it must be 115200) |

## Using it with an agent (Claude Code, OpenCode)

Start `claude` or `opencode` in this folder. The rules are in `AGENTS.md`, and Claude reads it through
`CLAUDE.md`. The agent touches the board only through `bench.py`, cannot change the
test files, and does not touch option bytes. `AGENTS.md` has an `<ADB_SERIAL>` placeholder for the
phone; replace it with your phone's serial from `adb devices`.

`AGENTS.md` only asks the agent to leave the tests alone. Two settings enforce part of it:

```bash
chmod -R a-w plans bench.toml bench.py      # Linux and macOS; the agent can no longer edit them
```

and `plans_dir_only = true` under `[permissions]` in `bench.toml`, which makes `run` refuse any plan
file outside `plans/`. Neither is a security boundary against an agent that runs as your user and
decides to undo them.

## Repo layout

| Path | Purpose |
|---|---|
| `bench.py` | The bench tool: flash, reset, send and read over serial, run test plans, recover, station mode |
| `bench.example.toml` | Config template; copy it to `bench.toml` (gitignored) |
| `plans/` | Test plans: `smoke`, `crash_recovery`, `measure`, `camera_blink` |
| `firmware/` | The F407 test firmware (`main.c`) and `build.py`. Bare metal, no HAL, no make. |
| `results/runs/` | Published runs of 2026-10-28 with their negative checks |
| `runs/` | Where `bench.py run` writes its reports (gitignored) |
| `setup_ubuntu.sh` | Ubuntu setup for a dedicated bench computer (read it first) |
| `AGENTS.md`, `CLAUDE.md` | Rules for a coding agent driving the bench |

## License

Apache License 2.0, see [LICENSE](LICENSE).
