# Agent rules (f407-hil-bench)

This repo is the STM32F407 test bench. Touch the board only through `bench.py`.

## Folders
- `bench.py`, `bench.toml`, `plans/`, `runs/`: the test bench tool, its config, test plans, and results
- `firmware/`: the test firmware on the board (`stm32bench-fw`) and `build.py`
- `results/`: published runs of 2026-10-28 with their negative checks

## Hardware
- Board: STM32F407G-DISC1 (STM32F407VG, Cortex-M4), LEDs on PD12-PD15
- Serial test channel: USART2, PA2=TX, PA3=RX, 115200 8N1 (through a USB-TTL adapter)
- Clock: HSI 16 MHz

## Commands (always use --json)
- Run Python through uv (`uv run ...`) so it uses `.venv` with the packages in `uv.lock`. If `.venv` is missing, run `uv sync`.
- Build: `uv run firmware/build.py --json`
- Run a test plan: `uv run bench.py --json run smoke`
- When verifying a new feature: `uv run bench.py --json run <plan> --verify-can-fail`
- Try a single command: `uv run bench.py --json send "PING" --expect "^PONG$"`
- Reset and check the boot message: `uv run bench.py --json reset --expect "^BOOT"`
- If the board does not respond: `uv run bench.py --json recover`
- If the board is hung or crashed, see where it stopped: `uv run bench.py --json halt-dump`
  It prints pc, the fault registers with descriptions, the `stacked_frame` from the moment of the crash,
  and the code addresses on the stack, then resumes the core. Run this first and reset afterwards,
  because a reset erases the evidence.
- Read memory or a register: `uv run bench.py --json mem 0x40020C14 --count 1` (read only, allowed regions only)
- `bench.py gdbserver` is for humans only. Do not run it.

Every result has `ok`, `error_type`, `summary`, and `likely_causes` fields. After a failed
`run`, read `serial.log` in the `report_dir`.

## Prohibited
- Do NOT CHANGE the files in `bench.py`, `bench.toml`, `plans/`, or `runs/`. If a test is wrong, tell
  the human. Changing a test to make it pass is forbidden.
- Do not run `STM32_Programmer_CLI` or `openocd` directly.
- Never touch option bytes, read protection (RDP), or mass erase.
- In the firmware, do not use the PA13/PA14 (SWD) pins for anything else.

## Proposing a new test
If a new test is needed, write it to the `plans_proposed/` folder and tell the human.
The human moves it to `plans/` after approving it. Every new test must pass a negative check
with `--verify-can-fail`. A test that does not FAIL when the firmware is deliberately broken proves nothing.

## Work loop
1. Make the change under firmware/.
2. Build it. If there are errors, fix them.
3. Run the relevant plan.
4. If it FAILs, look at `likely_causes` and `serial.log` first, then fix. After 3 attempts on the
   same failure, stop and give the human a summary.
