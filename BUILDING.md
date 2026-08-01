# Building and installing the fork (PCB v1.2)

Everything below was verified on this machine (Windows 11, VS2022 Community, Python 3.14).
Two artifacts come out of this repo:

| Artifact | What it is | Where it goes |
|---|---|---|
| `firmware.bin` | ESP32-S3 firmware | Flashed to the pedal over USB |
| `DiyFfbPedal.dll` | SimHub plugin | Copied into `C:\Program Files (x86)\SimHub\` |

They talk to each other over USB serial, so **flash the firmware and update the DLL together** —
mixed versions are how you get silent config mismatches.

---

## 1. Firmware (`ESP32/` tree)

### One-time setup

PlatformIO is already installed (`pip install platformio`). If you're on a fresh machine:

```bash
pip install platformio
```

### Build

Your PCB v1.2 maps to the `ControlBoard_PCBA_V1X` environment (`PCB_VERSION=7`, ESP32-S3):

```bash
cd ffb-pedal-fork/ESP32
python -m platformio run -e ControlBoard_PCBA_V1X
```

First build downloads the Xtensa toolchain (~10 min); later builds take ~1–2 min.
Output lands at:

```
ESP32/.pio/build/ControlBoard_PCBA_V1X/firmware.bin
```

### Flash — important quirk

The stock `env:ControlBoard_PCBA_V1X` is configured for **OTA upload** (`upload_protocol = espota`,
`upload_port = pedal_ota.local`). That works if the pedal is on your WiFi and mDNS resolves.
For USB flashing (recommended for a first flash of modified firmware), override the protocol:

```bash
python -m platformio run -e ControlBoard_PCBA_V1X -t upload --upload-protocol esptool --upload-port COM<N>
```

To find `COM<N>`: Device Manager → Ports, or `python -m platformio device list`.
The board resets into the bootloader automatically (`use_1200bps_touch = yes`). If the port
doesn't appear, hold **BOOT**, tap **RESET**, release BOOT — then it enumerates as a download port.

**Before your first flash of a fork build:** note which stock version you're on (SimHub shows the
firmware version string). If you ever want to go back, the stock flasher at the project's
web-flash tool restores upstream `main` in one click.

### Flashing with a four-file flasher (web flasher / esptool GUI)

If your flasher asks for four separate files, they're all in
`ESP32\.pio\build\ControlBoard_PCBA_V1X\` after a build, at these offsets (ESP32-S3 layout —
note the bootloader goes to **0x0**, not 0x1000 like classic ESP32):

| Offset | File |
|---|---|
| `0x0000` | `bootloader.bin` |
| `0x8000` | `partitions.bin` |
| `0xE000` | `boot_app0.bin` |
| `0x10000` | `firmware.bin` |

Only `firmware.bin` changes when you rebuild after a code edit — the other three are boilerplate
(bootloader, partition table, OTA boot selector) and only change if the partition layout or
platform version changes. For repeat flashing you can write just `firmware.bin` at `0x10000`.

Alternative: `firmware.factory.bin` in the same folder is all four merged into one image —
flash it alone at offset `0x0` if your tool supports a single combined file.

### Verify after flashing

Open a serial monitor at **3,000,000 baud**:

```bash
python -m platformio device monitor -b 3000000
```

You should see the normal boot chatter. The fork adds two new messages worth knowing:

- `Rudder init FAILED: pedal never settled near center. Rudder disabled.` — the new 15 s failure
  timeout fired (previously this hung forever, silently).
- `Rudder on` / `Rudder off` — now printed from the shared helper on both serial and wireless paths.

---

## 1b. Wireless Bridge firmware (`ESP32_master/` tree)

If you use the wireless setup, there is a **third device**: the Bridge dongle plugged into the
PC. It has its own firmware and participates in the DAP protocol version handshake — the plugin
refuses a version-mismatched bridge ("Bridge Dap version: X, Plugin DAP version: Y"), and the
pedals **silently drop** ESP-NOW packets whose version byte doesn't match theirs. All three
(pedal firmware, bridge firmware, plugin DLL) must be built from the same tree.

Pick the env for your hardware:

| Bridge hardware | Env |
|---|---|
| ESP32-S3 dev kit / WaveShare ESP32-S3 N8R8 | `Bridge_esp32s3usbotg` |
| Custom USB dongle (Gilphilbert Dongle V1, Lolin S3 Mini) | `Bridge_Dongle_V1` |

Build (PowerShell, same MSYS caveat as the pedal):

```bash
python -m platformio run -e Bridge_esp32s3usbotg
```

from `ffb-pedal-fork\ESP32_master`. Flash over the bridge's own COM port (close SimHub first —
it holds the port):

```bash
python -m platformio run -e Bridge_esp32s3usbotg -t upload --upload-port COM<N>
```

Same four-file layout as the pedal (S3: bootloader @ 0x0, partitions @ 0x8000, boot_app0 @
0xE000, firmware @ 0x10000) if you prefer the manual flasher; output lands in
`ESP32_master\.pio\build\<env>\`. An app-only flash preserves the stored pedal pairing; if the
pedals don't reconnect afterwards, re-run pairing from the SimHub plugin.

## 2. SimHub plugin (`SimHubPlugin/` tree)

### One-time setup (already done on this machine)

- Visual Studio 2022 with .NET desktop workload (MSBuild)
- **.NET Framework 4.8 Developer Pack** — the stock VS install lacked it; installed via:
  ```
  winget install Microsoft.DotNet.Framework.DeveloperPack_4 --version 4.8
  ```
- SimHub installed at `C:\Program Files (x86)\SimHub\` (the project references SimHub's own DLLs)

### Build

The fork fixed the `.csproj` so it builds from any checkout location (upstream hardcoded relative
paths that only worked from one specific folder depth). From a normal shell:

```bash
cd ffb-pedal-fork/SimHubPlugin
"/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe" \
    DIYFFBPedalUI.csproj -p:Configuration=Release -v:minimal
```

If SimHub lives somewhere non-standard, add `-p:SimHubDir="D:\Wherever\SimHub\\"`.

Output:

```
SimHubPlugin/bin/Release/DiyFfbPedal.dll
```

(Upstream wrote the DLL directly into Program Files, which required an admin shell and clobbered
your live install mid-build. The fork builds to `bin\Release\` and you install deliberately.)

### Install

1. **Close SimHub** (it locks the DLL).
2. Copy `bin\Release\DiyFfbPedal.dll` over `C:\Program Files (x86)\SimHub\DiyFfbPedal.dll`
   (needs one UAC prompt; keep a copy of the old DLL if you want a fallback).
3. Start SimHub → Add/remove features → confirm the DIY FFB Pedal plugin is active.

---

## 3. What changed in this fork (flight mode, tiers 1–2)

| Fix | Symptom it addresses |
|---|---|
| FM1 — signed-arithmetic fix + clamp + receive validation on the rudder position ratio | "Push pedal fully down → jams / overflows / freaks out" |
| FM2 — 16-bit sync position saturates instead of wrapping | latent; wire format unchanged |
| FM3 — engage/disengage no longer calls a blocking move from the 4 kHz control task | rough/steppy engage & disengage |
| FM5 — enable/disable transitions mutually exclusive; four duplicated toggle blocks unified | flight mode dead-until-reboot after a mistimed toggle |
| FM6 — init dwell timer resets on leaving center; 15 s hard failure timeout, loud abort | init hanging forever / completing instantly |
| Build — `.csproj` path fix, local `bin\` output | plugin now builds anywhere without admin |

**Not yet done** (Tier 3–4 from FLIGHT-MODE.md): position effects restored in flight mode (FM9),
the centre-crossing notch (FM11), filter lag (FM13), heli-rudder rewrite (FM10), and the
structural change — rudder centring as a force in the admittance model instead of moving soft
endstops (FM7/FM8).

## 4. Bench test checklist for this build

In rough order, pedal secured, feet clear:

1. **Normal brake mode sanity** — nothing in this change set should alter brake feel. Confirm.
2. **FM1:** enable flight mode, floor the pedal to the hard stop, hold 5 s, release. Watch the
   partner pedal — no jump, no thrash. Repeat 10×.
3. **FM3:** toggle flight mode on/off repeatedly — the transition should move smoothly with no
   stutter in the *other* pedal while one is transitioning.
4. **FM5:** enable flight mode and disable it *during* the move-to-center (within ~1 s). Re-enable.
   Pedals must still sync. Repeat 10×. Under stock firmware this bricked sync until reboot.
5. **FM6:** enable flight mode while physically holding the pedal away from center. After 15 s you
   should hear the low failure beep and see the serial message, and the pedal returns to brake mode.
