# canbus-scripts

A small collection of standalone **bash + [can-utils](https://github.com/linux-can/can-utils)** scripts for CAN bus / OBD-II diagnostics on Linux (Raspberry Pi in-vehicle rigs, bench SocketCAN setups, etc.).

Each script is self-contained — no shared library, no framework, just SocketCAN (`cansend` / `candump`) and standard OBD-II services. Drop in whichever ones you want.

> ⚠️ **Working on a live vehicle's CAN bus carries real risk.** Writing the wrong bytes can set faults, disable systems, or in rare cases require a dealer reflash. Read each script's header, prefer `--dry-run` first, and start listen-only. These are provided as-is, no warranty — see [Disclaimer](#disclaimer).

---

## Scripts

### `dtc-clear.sh`
Clears OBD-II Diagnostic Trouble Codes — the destructive-write counterpart to a read-only `dtc.sh`. Issues **OBD-II Service 0x04 ClearDiagnosticInformation** against the ECM on `$7E0` / `$7E8`.

Clears stored DTCs (Mode 0x03), pending DTCs (Mode 0x07), freeze-frame data (Mode 0x02), and resets the emissions readiness monitors; the MIL (check-engine light) goes out immediately. Does **not** clear permanent DTCs (Mode 0x0A — regulatory, enforced inside the ECM) or non-OBD-II module codes (those need UDS Service 0x14).

Safety-first by design:
- **Refuses to write without `--yes`** — no interactive y/n to fat-finger
- **`--dry-run`** prints the exact `cansend` frame without sending
- **Previews current DTCs first** via `dtc.sh` (if on `PATH`); `--skip-preview` to skip
- **Decodes NRC bytes** on negative response (e.g. NRC 0x33 securityAccessDenied → SGW hint)

```
./dtc-clear.sh                 # preview only — refuses to clear without --yes
./dtc-clear.sh --dry-run       # show the cansend that would fire
./dtc-clear.sh --yes           # actually clear codes
./dtc-clear.sh --skip-preview --yes
```

Flags: `[--yes|-y] [--dry-run] [--skip-preview] [--wait SEC] [--verbose] [--help]`

### `engine-dash.sh`
A simple terminal dashboard that polls a curated set of standard **OBD-II Mode 01** PIDs in a loop and redraws a fixed-size text frame each cycle. Tuned/labelled for the JEEP Gladiator (JT), but every PID is SAE J1979-standard, so it runs unchanged on **any 2007+ OBD-II vehicle**.

Polls 11 PIDs (~2.2s refresh): engine RPM, vehicle speed, throttle, absolute load, coolant / oil / intake / ambient temps, fuel level, run time, and control-module voltage (≈ battery V). A missing response shows `--` for that field without crashing the dashboard.

```
./engine-dash.sh               # live dashboard, Ctrl+C to exit
./engine-dash.sh --once        # print one frame and exit (scripting / logs)
```

Standalone — issues Service 0x01 requests directly via `cansend` / `candump` with inline single-frame and first-frame PCI parsing. Only runtime dependency is `can-utils`.

---

## Requirements

- **Linux with SocketCAN** (mainline kernel; Raspberry Pi + a CAN HAT is a common rig)
- **can-utils** — `apt install can-utils`
- A CAN interface up at the bus bitrate, e.g.:
  ```
  ip link set can1 up type can bitrate 500000
  ```

Most scripts default to `CAN_C=can1` (the OBD-II ECM bus); override via the environment variable if your wiring differs.

## A note on `.txt` extensions

The scripts are committed as `.txt` so they render and download cleanly in a browser. To run them, save without the extension (or rename) and `chmod +x`:

```
mv dtc-clear.txt dtc-clear.sh && chmod +x dtc-clear.sh
```

## Background & reference

These scripts pair with the CAN bus / OBD-II reference material at
[magikh0e.pl/pubCarHacking](https://magikh0e.pl/pubCarHacking/) — message maps,
UDS read/write guides, the Secure Gateway Module write-up, and a
bus-and-message reference with the byte-level detail the script headers point at.

## Disclaimer

Informational / educational use only. Working on a vehicle's CAN bus can void
warranties and, with write operations, cause real damage. You are responsible
for what you send on your own bus. No warranty, express or implied.

## License

MIT — see [`LICENSE`](LICENSE) if present, otherwise treat as MIT.
