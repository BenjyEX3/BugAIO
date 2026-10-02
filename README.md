# BUG AIO

High-performance peripheral controller, Elgato Wave XLR hardware bridge, and low-latency system tools for Windows.

[![Release](https://img.shields.io/badge/Release-v1.0.8-blue.svg?style=flat-square)](https://github.com/BenjyEX3/BugAIO/releases)
[![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%2F%2011%20(x64)-0078D6.svg?style=flat-square)](https://github.com/BenjyEX3/BugAIO)
[![Binary Size](https://img.shields.io/badge/Size-270%20KB-success.svg?style=flat-square)](https://github.com/BenjyEX3/BugAIO)
[![Dependencies](https://img.shields.io/badge/Dependencies-0%20(Standalone)-brightgreen.svg?style=flat-square)](https://github.com/BenjyEX3/BugAIO)
[![Anti-Cheat Compliance](https://img.shields.io/badge/Anti--Cheat-100%25%20Passive%20Safe-blueviolet.svg?style=flat-square)](https://github.com/BenjyEX3/BugAIO)
[![License](https://img.shields.io/badge/License-Freeware%20%2F%20Proprietary-red.svg?style=flat-square)](#why-is-bug-aio-closed-source)

---

## Overview

Bug AIO is an all-in-one hardware and system utility built for competitive gaming setups.

Most gaming rigs end up running four or five separate background programs just to manage hardware: Razer Synapse, Elgato Wave Link, Logitech G HUB, Wootility, hardware monitors, and audio mixers. These tools are mostly built on Electron, Chromium, or bloated .NET runtimes. They eat hundreds of megabytes of RAM, create DPC latency spikes, and introduce background stutter.

Bug AIO replaces all of them with a single standalone 270 KB executable. It talks directly to your hardware over raw USB HID, WinUSB, and Windows CoreAudio APIs. It idles at 0% CPU, uses under 15 MB of RAM, and runs at below-normal process priority so it never steals cycles from game frames or mouse interrupts.

![BUG AIO Interface](bug_aio_ui.png)

---

## Design Goals

- Single binary, zero bloat: Written in C# targeting standard Windows .NET. No installers, no Electron wrappers, and no external DLLs needed.
- Latency first: Sets its own priority to `BelowNormal` so background polling never competes with games or USB mouse polling threads.
- Direct hardware calls: Talks straight to devices via Win32 `winusb.dll`, `hid.dll`, and CoreAudio COM interfaces. No vendor background daemons required.
- Anti-cheat safe: Uses passive `PROCESS_QUERY_LIMITED_INFORMATION` calls to check the active game window. No memory scanning, no code injection, and no API hooks.

---

## Protocols & Hardware Architecture

Bug AIO handles communication through direct Win32 APIs and native protocol packets rather than relying on vendor runtimes.

```
+-----------------------------------------------------------------------------+
|                                  BUG AIO                                    |
+-----------------------------------------------------------------------------+
       |                |                 |                |            |
       | WinUSB         | Raw USB HID     | WASAPI COM     | Named Pipe | Win32
       | Control        | Reports         | Interop        | IPC        | APIs
       v                v                 v                v            v
+-------------+  +-------------+  +---------------+  +-----------+  +---------+
| Elgato Wave |  | 8K Mice &   |  | Per-App Audio |  | Discord   |  | Memory  |
| XLR MK.2    |  | Magnetic    |  | Sessions &    |  | Client    |  | & Power |
| Hardware    |  | Keyboards   |  | Peak VU Meter |  | RPC       |  | Tweaks  |
+-------------+  +-------------+  +---------------+  +-----------+  +---------+
```

### 1. WinUSB (Elgato Wave XLR MK.2)
Communicates with the Wave XLR through `winusb.dll` and `setupapi.dll` using device interface GUID `{94DC254F-6BFB-44D1-AECE-54CFF9FA57A5}` on Interface `MI_03`.

The app sends synchronous USB vendor setup packets (`WINUSB_SETUP_PACKET`) to control the internal DSP hardware:
- `RequestType`: `0xC1` (Device-to-Host) / `0x41` (Host-to-Device)
- `Request`: `0x01`
- `Block 0x0004` (Input Channel & DSP): 38-byte payload controlling preamp gain (0 to 75 dB), 48V phantom power, 80 Hz high-pass filter, noise expander, ClipGuard anti-distortion limiter, Voice Tune, and compressor modes.
- `Block 0x0005` (Headphone Output): 2-byte register setting headphone volume attenuation and analog impedance switching (LowZ <50 ohm vs HighZ >50 ohm).
- `Block 0x0001` (Monitor Crossfade): Hardware balance slider between live zero-latency mic monitor and PC playback audio.

### 2. USB HID Feature & Output Reports (Mice & Keyboards)
- 8K Gaming Mice (STMicroelectronics MCU):
  - Enumerates via `hid.dll` on VID `0x0483` (PID `0xA462` / `0xA3CF`).
  - Writes packed 16-bit register bitmasks through raw HID Feature Reports (`HidD_SetFeature`).
  - Configures High-Speed USB mode, polling rate (8000 Hz, 4000 Hz, 2000 Hz, 1000 Hz), lift-off distance (1mm, 2mm, 3mm), angle snapping, button swapping, and raw CPI values (50 to 20,000 in 50 CPI steps).
- Hall Effect Keyboards (Venom60HE & Mad68HE):
  - Vendor Usage Page `0xFF00` through `0xFF53`, Usages `0x61`, `0x62`, `0x01`.
  - Sends 64-byte raw HID output reports directly to controller endpoints to query firmware version (`0xE3`), read active profile (`0x40`), and toggle hardware SOCD (Simultaneous Opposing Cardinal Directions / Snap Tap) on the fly.

### 3. Windows CoreAudio & WASAPI COM
Uses direct unmanaged COM interfaces without third-party audio wrappers:
- `IMMDeviceEnumerator`, `IMMDevice`, and `IPropertyStore` for audio device discovery.
- `IAudioEndpointVolume` for master hardware level control and register callbacks (`IAudioEndpointVolumeCallback`).
- `IAudioMeterInformation` for live dBFS peak metering.
- `IAudioSessionManager2`, `IAudioSessionEnumerator`, `IAudioSessionControl2`, and `ISimpleAudioVolume` to list running audio sessions, pull application icons, and set per-app volume sliders.

### 4. Inter-Process Communication (IPC)
- Discord Rich Presence:
  - Talks directly to the local Discord client over Windows Named Pipes (`\\.\pipe\discord-ipc-0` through `\\.\pipe\discord-ipc-9`).
  - Uses the native framed wire format: an 8-byte header `[Opcode: uint32][Length: uint32]` followed by UTF-8 JSON (`OP_HANDSHAKE`, `OP_FRAME`). No external Discord GameSDK binaries needed.
- Hardware Telemetry:
  - Reads RivaTuner / MSI Afterburner kernel sensor data through the `MAHMSharedMemory` memory-mapped file (`MAHM_SHARED_MEMORY_HEADER` and `MAHM_SHARED_MEMORY_ENTRY`).
  - Fallback readings use `GlobalMemoryStatusEx` and WMI queries.

---

## Features

### Peripherals
- 8000 Hz mouse polling: Instant switching between 8K, 4K, 2K, and 1K polling rates.
- Sensor adjustments: CPI slider from 50 to 20,000 CPI (50 CPI increments), LOD selection (1mm, 2mm, 3mm), and angle snapping toggle.
- Hall Effect keyboards: Native support for Venom60HE and Mad68HE magnetic switch keyboards.
- Hardware SOCD: Toggle SOCD (Snap Tap / Rappy Snappy) directly on the keyboard hardware, with automatic profile switching based on what game is running.

![Peripherals Tab](snap_peripherals_mad68.png)

### Elgato Wave XLR MK.2 Control
- Wave Link alternative: Full access to the Wave XLR hardware without having Elgato Wave Link installed.
- Physical Dial Gain Lock: When enabled, Bug AIO monitors dial input. If you bump the physical gain knob during a match, it catches the volume change event and immediately pulls the gain back to your locked setting.
- Hardware DSP controls:
  - Preamp gain from 0 dB to 75 dB with live bidirectional sync.
  - ClipGuard analog limiter toggle.
  - 48V phantom power toggle.
  - 80 Hz low-cut filter toggle.
  - Headphone volume, mute status, and headphone impedance mode (<50 ohm vs >50 ohm).
  - Crossfade balance between microphone direct monitoring and PC audio.
  - 4-band hardware digital EQ (100 Hz, 500 Hz, 2.5 kHz, 8 kHz).
  - Noise expander / clean voice modes and Voice Tune intensity slider.
- Live peak meter and sound check: Hardware dBFS VU meter and a built-in test recording/playback tool using Windows MCI.

![Wave XLR Tab](snap_wave_xlr_updated.png)

### Per-App Audio Mixer
- Clean replacement for the Windows 11 Volume Mixer.
- Lists active WASAPI audio sessions with application icons and process IDs.
- Volume sliders and mute buttons for each application.
- Remembers volume levels across restarts in `config.json`.

### Game Profile Automation
- Passive foreground watcher: Checks the active window every 400ms using `PROCESS_QUERY_LIMITED_INFORMATION`.
- Automatic game rules:
  - Drops polling rate to 1000 Hz for games that stutter on high report rates (like Rust).
  - Turns on 8000 Hz polling and SOCD for supported titles (Apex Legends, Overwatch).
  - Turns off SOCD for games that ban it (Counter-Strike 2, VALORANT) so you stay tournament legal.
- Reverts to your default desktop profile when you alt-tab or close the game.

### Hardware Monitoring & System Tweaks
- Hardware stats: Live CPU/GPU clock speeds, temperatures, RAM and VRAM usage. Includes a pause button to freeze monitoring during matches for zero background cycles.
- Latency tweaks:
  - Power plan: Activates the Windows Ultimate or High Performance power scheme.
  - Memory trim: Calls `EmptyWorkingSet` and forces garbage collection.
  - Shader cache cleaner: Wipes NVIDIA (`DXCache`, `GLCache`, `ComputeCache`), AMD (`DxCache`, `GLCache`), and DirectX shader caches to fix stutter.
  - Network flush: Clears the Windows DNS resolver cache via `DnsFlushResolverCache`.
  - Service stopper: Stops non-essential background services like telemetry, spooler, SysMain, and error reporting.
  - Drive TRIM: Runs background SSD TRIM and optimization (`defrag /C /O`).

![Monitoring & Optimization Tab](snap_monitoring.png)

### Discord Rich Presence
- Shows current game profile or system status on Discord using pure named pipe IPC.
- Custom Client ID support.

---

## Hardware & Protocol Matrix

| Device / Subsystem | Identifier (VID / PID) | Protocol / Transport | Endpoints / Commands |
|:---|:---|:---|:---|
| Elgato Wave XLR MK.2 | VID: 0x0FD9<br>PID: 0x00B6 | WinUSB Direct Control<br>CoreAudio WASAPI | Interface MI_03<br>Vendor Blocks 0x0004, 0x0005, 0x0001<br>IAudioEndpointVolume Callback Sink |
| STMicroelectronics 8K Mouse | VID: 0x0483<br>PID: 0xA462 / 0xA3CF | USB HID Feature Reports | Packed 16-bit register payload<br>Feature Report (Length >= 5 bytes) |
| Venom60HE Keyboard | VID: 0x534B<br>PID: 0x3104 | USB HID Output / Input Reports | UsagePage 0xFF53, Usage 0x61/0x62<br>Commands: 0xE3 (FW), 0x40 (Profile) |
| Mad68HE Keyboard | VID: 0x373B / 0x3710 | USB HID Output / Input Reports | UsagePage 0xFF00, Usage 0x61/0x01<br>64-byte structured packets |
| WASAPI Audio Engine | System Audio Endpoint | Windows COM / CoreAudio | IAudioSessionManager2, IAudioMeterInformation |
| Hardware Telemetry | RivaTuner / MSI Afterburner | Win32 Memory-Mapped File | File: MAHMSharedMemory (0x4D48414D) |
| Discord RPC | Local Named Pipe | Local IPC Named Pipe | \\\\.\\pipe\\discord-ipc-[0-9] (Framed JSON) |

---

## Comparison vs Vendor Bloatware

| Metric / Capability | BUG AIO | Vendor Suites (Wave Link, Synapse, G HUB) |
|:---|:---|:---|
| Distribution Footprint | 270 KB (Single Standalone Executable) | 350 MB to 1.2 GB (Multiple Installers) |
| Runtime Architecture | Native C# / Pure Win32 P/Invoke | Electron / Chromium / Node.js / Heavy .NET |
| Idle Memory Consumption | Under 15 MB | 400 MB to 1.5 GB combined |
| Background CPU Usage | 0.0% (BelowNormal priority) | 1.5% to 6.0% (Periodic spikes) |
| External Background Services | 0 | 4 to 12 background services |
| Anti-Cheat Interference Risk | None (Passive PROCESS_QUERY_LIMITED_INFORMATION) | Occasional hook collisions & overlay flags |
| Hardware Dial Lock | Yes (Active volume callback enforcement) | No |
| Cross-Device Profile Sync | Unified (Mouse + Keyboard + Audio in one trigger) | Fragmented across individual apps |

---

## Why is Bug AIO Closed Source?

Bug AIO is distributed as a closed-source freeware binary. That decision comes down to four practical reasons:

### 1. Anti-Cheat Safety and Tournament Integrity
Bug AIO automatically adjusts hardware profiles, polling rates, and keyboard SOCD based on which game window is active. Anti-cheat engines (Vanguard, EAC, BattlEye, VAC) monitor low-level input manipulation and process querying closely.

If this code were public, it would inevitably be forked and repurposed by cheat developers into injector bases or rapid-trigger macro exploits. Once an open-source codebase gets abused in competitive games, anti-cheat vendors blacklist its driver signatures, window hooks, and binary patterns. That would result in false positives or blanket bans for legitimate users. Keeping the binary closed ensures consistent cryptographic signatures and prevents bad actors from weaponizing the code.

### 2. Hardware Safety (No Bricked Controllers)
The program writes directly to hardware microcontrollers and onboard EEPROM over raw USB HID feature reports and WinUSB vendor control transfers. This includes low-level register writes to the Elgato Wave XLR, custom 8K mouse MCUs, and Hall Effect keyboards.

Sending an invalid bitmask or malformed byte packet to these chips can corrupt calibration data or brick the controller permanently. Keeping the codebase centralized ensures that every register write is tested and verified against actual hardware before it ships.

### 3. Untampered, Zero-Bloat Binaries
Bug AIO is a single 270 KB executable with no installer, no ads, no telemetry, and no bundled junk. Releasing closed-source binaries directly from this GitHub repo ensures users know the executable is clean, authentic, and untampered with.

### 4. Proprietary Reverse Engineering
A lot of time went into sniffing USB packets and mapping out undocumented hardware protocols, specifically the Wave XLR MK.2 WinUSB interface, Hall Effect keyboard command structures, and RivaTuner shared memory structs. Keeping the implementation proprietary protects that work.

---

## Installation & Usage

Bug AIO needs no installation and writes no files outside its own folder:

1. Download `Bug AIO.exe` from the official [Releases](https://github.com/BenjyEX3/BugAIO/releases) page.
2. Put `Bug AIO.exe` anywhere you like (for example, `C:\Tools\Bug AIO\`).
3. Run the executable. On first launch, it generates a `config.json` file in the same directory.
4. (Optional) Check **Start with Windows** and **Start Minimized** in the app to keep it running in the system tray.

### Command-Line Flags

```bash
# Launch minimized to the system tray
"Bug AIO.exe" --minimized

# Open straight to a specific tab (0: Peripherals, 1: Wave XLR, 2: App Mixer, 3: Games, 4: Monitoring)
"Bug AIO.exe" --tab 1

# Check GitHub for updates and exit
"Bug AIO.exe" --test-update
```

---

## Configuration (config.json)

Settings, app volumes, and game profiles are stored in `config.json`:

```json
{
  "perGameEnabled": true,
  "startWithWindows": false,
  "startMinimized": false,
  "closeToTray": true,
  "suppressAudioPopups": true,
  "defaultPollingRate": "8k",
  "defaultSocd": "Off",
  "discordRpcEnabled": true,
  "discordClientId": "383226320970055681",
  "monitoringPaused": false,
  "mouseUsbSpeed": 1,
  "mousePollingRate": 0,
  "mouseLod": 1,
  "mouseAngleSnapping": 0,
  "mousePrimaryButton": 0,
  "mouseCpi": 1600,
  "waveXlrGain": 47,
  "waveXlrOutputVol": 100,
  "waveXlrCrossfade": 8,
  "waveXlrClipguard": true,
  "waveXlrInlineBoost": "Off (Standard)",
  "waveXlrPhantom48V": false,
  "waveXlrHeadphoneMode": "Standard (<50Ω)",
  "waveXlrDirectMonitor": true,
  "waveXlrDialGainLock": true,
  "waveXlrMuted": false,
  "waveXlrLowcut": "80 Hz High-Pass",
  "waveXlrNoiseCancelMode": "Studio Clean (Voice Focus)",
  "waveXlrNoiseCancelIntensity": 75,
  "waveXlrVoiceTuneEnabled": true,
  "waveXlrVoiceTuneIntensity": 32,
  "waveXlrCompressorMode": "Off",
  "appVolumes": {
    "spotify.exe": 100,
    "discord.exe": 90,
    "steam.exe": 100
  },
  "games": [
    {
      "exeName": "RustClient.exe",
      "path": "C:\\Program Files (x86)\\Steam\\steamapps\\common\\Rust\\RustClient.exe",
      "pollingRate": "1k",
      "socd": "On"
    },
    {
      "exeName": "cs2.exe",
      "path": "C:\\Program Files (x86)\\Steam\\steamapps\\common\\Counter-Strike Global Offensive\\game\\bin\\win64\\cs2.exe",
      "pollingRate": "8k",
      "socd": "Off"
    }
  ]
}
```

---

## Requirements

- Windows 10 (64-bit) or Windows 11 (64-bit)
- Microsoft .NET Framework 4.5+ (already included in Windows 10 and 11)
- Hardware:
  - Supported STMicroelectronics 8K mouse MCU for mouse controls
  - Venom60HE or Mad68HE keyboard for magnetic switch controls
  - Elgato Wave XLR MK.2 for studio audio features
  - (Optional) RivaTuner Statistics Server (RTSS) / MSI Afterburner for live GPU/CPU sensor readings

---

## Verification & Releases

- Releases: [github.com/BenjyEX3/BugAIO/releases](https://github.com/BenjyEX3/BugAIO/releases)
- Author: BenjyEX3
- Verify the SHA256 checksum posted in the release notes before running.
