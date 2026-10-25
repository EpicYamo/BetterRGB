# Better RGB

Per-key RGB control for **Tongfang / ITE8291** gaming laptops, written from
scratch in C for Windows.

Built by reverse-engineering the keyboard's USB HID protocol with a sniffer and
a lot of patience — no vendor SDK, no leaked source.

**Current release: [v1.0.0](https://github.com/EpicYamo/BetterRGB/releases/tag/v1.0.0)**
· [English manual](MANUAL_EN.md) · [Türkçe kılavuz](MANUAL_TR.md)

---

## Contents

- [What it is](#what-it-is)
- [History](#history)
- [Why this project exists](#why-this-project-exists)
- [The developer's background](#the-developers-background)
- [Tools used](#tools-used)
- [The reverse-engineering process](#the-reverse-engineering-process)
- [The vanishing lights bug](#the-vanishing-lights-bug)
- [Challenges along the way](#challenges-along-the-way)
- [What the application does](#what-the-application-does)
- [Supported models](#supported-models)
- [Known limitations](#known-limitations)
- [Community projects](#community-projects)
- [What's next](#whats-next)
- [Building from source](#building-from-source)
- [Manual](#manual)
- [AI usage](#ai-usage)
- [Windows security warnings](#a-note-on-windows-security-warnings-smartscreen--smart-app-control)
- [License](#license)
- [Author](#author)
- [Disclaimer](#disclaimer)

## What it is

A native Win32 application that drives the per-key RGB keyboard of Tongfang
barebones with an ITE8291 controller — the chassis sold as Game Garaj Slayer in
Turkey, XMG Apex 16 Max, Schenker, Erazer Major and several other rebrands.

It speaks the keyboard's own HID protocol directly, so lighting is smooth
(about 30 FPS, streamed from the host) and every key can be addressed
individually: nine effects, per-effect settings, static zone layouts, a tray
icon, auto-start at login, and a device lock that keeps the vendor software from
overriding the lighting.

## History

This repository is the **modular rewrite** of the original single-file program
(`better_rgb.c`, about 2500 lines, v2.2). Behaviour is unchanged — every effect
and feature works the same — but the code is split into `device`, `config`,
`effects/`, `ui/`, `vendor_lock` and `autostart` so it can be read and extended.

The original history and the last single-file release live in
[BetterRGB-old](https://github.com/EpicYamo/BetterRGB-old), which is archived.

## Why this project exists

The laptop ships with a vendor RGB application ("GCUBridge" / Control Center)
that supports per-key lighting. The hardware is capable; the software wasn't.
Its built-in animations (wave, fireworks, breathing) felt noticeably choppy,
especially next to a Razer Blade 14 whose Synapse produced buttery-smooth
per-key effects.

The obvious first move was [OpenRGB](https://openrgb.org/), the open-source
universal RGB controller. It detected the keyboard's HID interface and
correctly identified it as an ITE-family controller (reported generically as a
"Clevo keyboard"), but writes had no effect — the lighting simply did not
respond, with or without the vendor service running.

With no OpenRGB support for this exact device and firmware revision, and no
public documentation of the protocol, the only path left was to reverse
engineer it from scratch.

## The developer's background

No prior Windows application development experience. Background in C (through
the 42 school curriculum: minishell, miniRT, ft_irc) and in embedded systems
from hobby projects — FPV drones, RC aircraft, a custom FPV head tracker, DIY
flight controllers and radio links.

That hardware background turned out to be the right toolkit for this kind of
work: reading USB traffic, working with raw byte-level HID reports, and
debugging hardware that occasionally locks up are familiar territory to anyone
who has built their own RC gear.

## Tools used

- **Wireshark + USBPcap** — capturing raw USB traffic between the vendor
  software and the keyboard controller.
- **Python** — custom parsers for the captured `.pcapng` files; some quirks of
  the USBPcap packet header needed byte-level decoding that generic tools did
  not handle.
- **mingw-w64 / gcc** — the application is plain C, compiled with MinGW on
  Windows.
- **hidapi** — cross-platform HID access, bundled as a submodule with a small
  patch (see "The vanishing lights bug").
- **Win32 API directly**, no framework — window creation, owner-drawn controls,
  `Shell_NotifyIcon` for the tray, the Service Control Manager, and Task
  Scheduler.

## The reverse-engineering process

### 1. Identifying the hardware

Using USBDeview and Device Manager, the keyboard was narrowed down to a
composite device: **VID_048D, PID_600B** (ITE Tech. Inc.). It exposes two HID
interfaces:

- **Interface 0 (MI_00)** — standard boot keyboard, handles normal key input.
- **Interface 1 (MI_01)** — vendor-defined (Usage Page `0xFF03`), reporting
  itself as `ITE Device(8291)`; this is the one that controls the RGB.

### 2. Capturing the protocol

With the vendor software running, Wireshark (via the USBPcap extcap interface)
captured USB traffic while changing colors, switching built-in animations and
entering per-key edit mode.

The capture format required manual parsing: USBPcap wraps every USB transfer in
a `USBPCAP_BUFFER_PACKET_HEADER` before the pcapng encapsulation, and the header
layout (bus, device, endpoint, transfer type, data length) had to be decoded by
hand, cross-checked against known values such as the USB device descriptor's
vendor/product IDs in a `GET_DESCRIPTOR` response.

This surfaced two distinct channels:

- **HID feature reports** (`SET_REPORT`, sent as USB control transfers) carrying
  short 8-byte commands.
- **HID output reports** (interrupt transfers on endpoint 2) carrying 64-byte
  chunks — eight of them make up a 512-byte per-key color buffer.

### 3. Decoding the command set

Comparing the exact bytes sent for each user action against the visible result
produced three opcodes:

| Opcode | Purpose |
| --- | --- |
| `0x14` | set a solid/static color (whole keyboard or a palette slot) |
| `0x08` | select the active lighting mode (static, wave, firework, custom, …) |
| `0x12` | prepare the device for an incoming buffer / custom-mode data stream |

### 4. Mapping every physical key

Reset the keyboard to black, light one key at a time, and diff each captured
frame against the previous one to see which 3-byte slot changed. Repeated across
the whole board — about 103 physical positions, including a full numpad, the
arrow cluster and ISO-specific keys — this produced a complete
key-name → byte-offset table.

### 5. Talking to the device from C

With the protocol decoded, hidapi replicated it from a native Windows program:
open the right HID interface by its enumerated path, send the same feature
reports and interrupt writes, and confirm the result matches what the vendor
software produced.

## The vanishing lights bug

Long after the effects worked, one maddening bug remained: the keyboard would
randomly go black, usually within the first 30–60 minutes after boot, while
every `hid_write()` kept returning success. Restarting the app always fixed it.

The investigation, step by step:

1. **The two-instance clue.** With the first instance "dead", launching a second
   one instantly revived the first. The only thing the second instance did that
   the first was not doing continuously was send the mode-initialization feature
   reports. Conclusion: the EC had been kicked out of custom/per-key mode, and
   the frame stream was being silently ignored — no error, no log, just black.
2. **Confirming it in the app.** Re-sending the full init sequence on every mode
   switch proved it: when the lights died, switching modes revived them.
3. **A passive listener.** A read-only monitor thread logged anything the EC
   sent back on the interrupt IN endpoint around the moment of death: total
   silence. The EC was not announcing anything — something external was
   commanding it.
4. **The experiment.** The vendor Control Center had been set to "keyboard
   lighting off". The hypothesis: what if the vendor service periodically
   re-applies its selected lighting state, and "off" simply looks like a random
   blackout? Changing it to a solid color and waiting proved it — the next
   "death" was not black, it was the vendor's color. Case closed: the GCUBridge
   service periodically rewrites its own lighting state to the EC, stomping
   whatever another program set.
5. **Why it could not be blocked politely.** The service also holds a persistent
   handle to the RGB interface from boot, and cannot simply be disabled — it
   also manages fan curves, power limits and performance modes.

### The fix: an exclusive device lock

Windows lets a HID interface be opened with `dwShareMode = 0`. While such a
handle is held, every other process's attempt to open the interface fails with
`ERROR_SHARING_VIOLATION` at driver level. The bundled hidapi's `windows/hid.c`
is patched accordingly: `open_device` first tries an exclusive open and falls
back to shared mode only if someone else already holds the device.

With the lock held, the vendor service physically cannot touch the RGB
interface; its fan/power features are unaffected because they use a different
interface, and its keyboard-lighting page simply does not load until the app
exits and releases the lock.

One problem remains: the service grabs the interface at boot, before the user
can launch anything. The application handles this in layers:

- Started **elevated** and the interface is taken → it restarts the vendor
  service through the Service Control Manager, snatches the exclusive lock in
  the gap, and lets the service come back up locked out of RGB, fully functional
  otherwise. The service is located by name, with a fallback scan of installed
  services, since rebrands may ship it under different names.
- Started **unelevated** and the interface is taken → a bilingual dialog offers
  to relaunch elevated and perform the same fix with one UAC prompt. Declining
  triggers a second explicit warning, because running shared alongside the
  vendor service risks the EC lockup described below.
- The elevated relock instance never runs shared: it retries the exclusive open
  with a progressive backoff (1–5 s), and if the lock still cannot be acquired
  it restores the vendor service, reports the failure and refuses to start.
- An **Auto-Start at Login** button installs a Scheduled Task that launches the
  app elevated at every login (`/SC ONLOGON /RL HIGHEST`), making the whole
  dance invisible: no UAC prompts, no dialogs, the lock is taken silently every
  boot.

## Challenges along the way

- **The device could be locked up.** Malformed or overly aggressive command
  sequences — especially repeating the "enter custom mode" preparation in a
  tight loop — could put the keyboard's embedded controller into a bad state.
  Lighting would stop responding entirely, sometimes to *any* software,
  including the vendor's own. Recovery required a full EC reset: power off,
  disconnect the charger, hold the power button for about 30 seconds to
  discharge residual capacitance, then power on. This happened several times
  during development and shaped a much more cautious, incremental approach to
  testing new timings.
- **Windows fighting back.** The vendor Control Center had a Scheduled Task
  configured to relaunch it every minute for ten minutes after login —
  discovered after wondering why the vendor app kept reappearing mid-test.
  Disabling that task and the underlying service (with recovery actions set to
  "take no action") was necessary for a clean, conflict-free test environment.
- **A silent saboteur.** The hardest bug produced no evidence at all: no failed
  API calls, no log entries, no incoming data — just a keyboard that randomly
  went dark. Cracking it required treating the running system as another
  reverse-engineering target (see above), and the smoking gun was a designed
  experiment rather than a captured packet: making the invisible takeover
  visible by giving the vendor software a bright solid color to take over with.
- **Finding a safe, real frame rate.** `hid_write()` returning success does not
  mean the firmware rendered that frame. Early tests that ignored this produced
  a "three frames flash, then freeze" result at high send rates. Real-world
  testing (timed loops, slow-motion camera review) converged on roughly 30 FPS
  as a reliable ceiling for this firmware, well above the vendor software's own
  animation smoothness.
- **Task Scheduler's hidden defaults.** The Auto-Start task initially never
  fired at logon, reporting only "the task has not yet run" (`0x41303`). Tasks
  created from the `schtasks` command line silently default to "start only if on
  AC power" — a death sentence on a laptop running on battery — plus a 72-hour
  execution time limit that would have killed the app after three days. Neither
  default can be changed through `schtasks` switches, so the task is registered
  from a full XML definition with those settings disabled. Related lesson: a
  process launched by Task Scheduler starts with `System32` as its working
  directory, which is where the app's relative-path log file had been silently
  going; all paths are now resolved absolutely, next to the exe.
- **A missing HID Report ID byte**, the classic hidapi gotcha: both
  `hid_send_feature_report` and `hid_write` expect the buffer's first byte to be
  the Report ID, and forgetting it caused silent data misalignment before being
  tracked down.

## What the application does

A modular Win32 desktop application — no external UI framework, plain GDI,
owner-drawn controls, and a manifest for modern visual styling — offering:

- **Breathing** — solid color pulsing, with adjustable speed and smoothness.
- **Wave** — a moving color gradient across the keyboard with an adjustable
  angle (0–359°), computed from each key's real physical position (accounting
  for the numpad and navigation-cluster gaps).
- **Sparkle** — random per-key flashes, with independently tunable speed (flash
  lifetime) and density (how many keys are lit at once).
- **Reactive** — keys light up on keypress (fixed or random color) and fade out
  over a selectable duration (short/medium/long); uses `GetAsyncKeyState`
  polling with proper key-hold edge detection.
- **Wheel** — a radial rainbow rotating around the keyboard's geometric center,
  direction-reversible.
- **Lightning** — a smooth, falling bolt with adjustable travel speed, strike
  width, glow smoothness, and how many bolts can strike concurrently.
- **Flame** — a drifting heat simulation (not per-frame noise) mapped through a
  black → red → orange → yellow → white palette.
- **Rain** — falling colored drops with a trailing glow, adjustable density and
  fall speed, and a configurable multi-color palette.
- **Matrix** — four selectable sub-styles (diagonal streaks, classic vertical
  "digital rain" columns, laser sweep, expanding ripples), each with adjustable
  speed, density and palette.
- **Static** — a motionless multi-zone layout: split the keyboard into up to 6
  horizontal or up to 10 vertical bands and give each zone its own color.

Beyond the effects:

- **Exclusive device lock** — the RGB interface is opened unshared, making
  vendor-software interference impossible while the app runs, with an automatic
  elevated fix when the vendor service grabbed the interface first.
- **Auto-Start at Login** — one button installs or removes a Scheduled Task
  that launches the app elevated at logon, so the lock is acquired silently on
  every boot; the task runs on battery, has no execution time limit, and starts
  the app hidden in the tray.
- **Settings persistence** — the selected mode (including Stop), brightness and
  every per-effect parameter and color are saved to `better_rgb.cfg` next to
  the exe and restored on the next launch.
- **Self-healing device handling** — sleep/resume and device arrival/removal
  trigger automatic reconnection and mode re-initialization.
- **Tray integration** — minimizing hides the app to the tray, where it keeps
  animating in the background.
- **Bilingual dialogs** — every user-facing message is shown in English and
  Turkish.

All effects run on a background thread, computed entirely on the host and
streamed to the keyboard live; the firmware's own built-in effects are never
used.

## Supported models

The protocol was mapped on this chassis and is shared by its rebrands.

| Chassis / brand | USB ID | Status |
| --- | --- | --- |
| Game Garaj Slayer (Turkey) | `048D:600B` | development target |
| XMG Apex 16 Max / Schenker / Erazer Major | `048D:600B` | same barebone |
| Mechrevo 苍龙 16 X Pro | `048D:600B` | verified by a community fork |

Other ITE8291 boards may work, but the keymap and effect timings were tuned for
this one.

## Known limitations

- The `Fn` key and the key sending the OEM "Copilot" shortcut produce no
  distinguishable input signal on Windows, so they cannot be used in Reactive
  mode.
- The numpad Enter and the main Enter share a Windows virtual-key code, so they
  cannot be told apart in Reactive mode; both light up together.
- The keymap was built for one physical layout (Turkish ISO). Other regional
  layouts need their own mapping pass; the per-key byte offsets are
  hardware-dependent and transfer directly, only the labels change.
- While the app runs, the vendor Control Center's keyboard-lighting page will
  not load. That is the device lock working as designed, not a malfunction.

## Community projects

- [BetterRGB ITE8291 Adaptive](https://github.com/ZavierChen/BetterRGB-ITE8291-Adaptive)
  by [@ZavierChen](https://github.com/ZavierChen) — an independently developed
  derivative with 16 effects, power profiles, transitions, top-row indicator
  bars, white calibration and a Chinese/English UI. Verified on the Mechrevo
  苍龙 16 X Pro (`048D:600B`).

## What's next

A frame-by-frame custom animation editor, letting the user design sequences
directly instead of picking from preset effects.

## Building from source

Requires MSYS2 (mingw64), CMake and Ninja.

```powershell
git clone --recurse-submodules https://github.com/EpicYamo/BetterRGB.git
cd BetterRGB
cmake -B build -G "Ninja"
cmake --build build
```

`hidapi` is a submodule, so `--recurse-submodules` (or
`git submodule update --init --recursive` afterwards) is required — the build
also carries this project's exclusive-open patch to `windows/hid.c`.

The resulting `build/better-rgb.exe` is self-contained (statically linked) and
runs without separate DLLs.

## Manual

- [MANUAL_EN.md](MANUAL_EN.md) — English user manual: modes and their settings,
  the interface, coexistence with the vendor software, troubleshooting.
- [MANUAL_TR.md](MANUAL_TR.md) — Türkçe kullanım kılavuzu.

## AI usage

AI was used only to look up Windows API functions and build details (for
example how `Shell_NotifyIcon`, the Service Control Manager or `schtasks` XML
definitions behave). The protocol, the code and the design are the author's own
work.

## A note on Windows security warnings (SmartScreen / Smart App Control)

The released executable is **unsigned** — this is a small open-source project,
and code-signing certificates from a trusted CA are a paid, identity-verified
service. Every fresh build also has a brand-new file hash with zero reputation.
As a result:

- **Microsoft Defender SmartScreen** may show "Windows protected your PC" on
  first launch. Click *More info → Run anyway*.
- **Smart App Control** (enabled by default on some new Windows 11 installs) may
  block the app entirely with "This app may be unsafe", with no per-app
  exception possible. The app legitimately does things that look suspicious to
  an unsigned-binary classifier: it restarts a Windows service (to take the
  device lock), creates a Scheduled Task (Auto-Start), and relaunches itself
  elevated. SAC can only be turned off globally (Settings → Privacy & security →
  Windows Security → App & browser control → Smart App Control), and once off it
  cannot be re-enabled without reinstalling Windows.

If you prefer not to trust a prebuilt binary, the project is fully open source —
build it yourself.

## License

MIT — see [LICENSE](LICENSE). Use it, modify it, redistribute it, ship it in
your own projects; just keep the copyright notice.

The bundled [hidapi](https://github.com/libusb/hidapi) library (a submodule at
`third_party/hidapi/`, including this project's small exclusive-open patch to
`windows/hid.c`) is used and redistributed under hidapi's **BSD 3-Clause**
option; its original license files are preserved in that folder.

## Author

**Arda Aycan** ([@EpicYamo](https://github.com/EpicYamo)).

## Disclaimer

This project is the result of independent reverse engineering for personal and
interoperability use. It is not affiliated with, endorsed by, or supported by
the laptop's manufacturer or ITE Tech. Inc. Use it at your own risk: direct HID
communication with embedded controllers carries a small risk of triggering
unexpected firmware states (see "Challenges"). Nothing was permanently damaged
during this project, but there are no guarantees.
