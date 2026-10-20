# Better RGB by TheYamo

A custom per-key RGB control application for Tongfang/ITE8291 based gaming laptops (sold under various brand names, including "Game Garaj Slayer" in Turkey, and similar chassis sold as XMG Apex 16 Max, Schenker, Erazer Major, and other rebrands of the same Tongfang barebone).

Built from scratch in C, using a fully reverse-engineered USB HID protocol, no vendor SDK, no leaked source code, just a USB sniffer, patience, and a lot of trial and error.

Current version: **2.2**

---

> **Note:** This repository is a from-scratch, behavior-preserving rewrite of the original single-file `better_rgb.c` (v2.2, ~2500 lines). Every effect, feature, and code path here works identically to the original nothing about *what* the app does or *how* it behaves has changed. The code has simply been reorganized into separate modules (`device`, `config`, `effects/`, `ui/`, `vendor_lock`, `autostart`, …) purely to make the codebase easier to read, understand, and extend with future additions. The Original Old Repository is(which wont have updates anymore): https://github.com/EpicYamo/Better_RGB_Old-Repo

---

## Why this project exists

The laptop in question ships with a vendor RGB control application (referred to here as "GCUBridge" / Control Center) that supports per-key RGB lighting. The hardware is capable but the software wasn't. Built-in animations (wave, fireworks, breathing) felt noticeably choppy, especially compared to a Razer Blade 14 also owned by the developer, whose Synapse software produced buttery smooth per-key effects.

The obvious first move was [OpenRGB](https://openrgb.org/), the open source universal RGB controller. It detected the keyboard's USB HID interface and correctly identified it as an ITE-family controller (misreported generically as a "Clevo keyboard"), but writes to it had no effect the keyboard lighting simply didn't respond, regardless of whether the vendor service was running or stopped.

With no existing OpenRGB support for this exact device/firmware revision and no public documentation of the protocol, the only path forward was to reverse engineer it from scratch.

## The developer's background

No prior Windows application development experience. Background in:
- C (through 42 School's project curriculum minishell, minirt, and currently irc)
- Embedded systems and hardware, from hobby projects: FPV drones, RC aircraft, a custom FPV head tracker, DIY Flight Contoller(basic) DIY radio transmitters and receivers

That hardware/embedded background turned out to be exactly the right toolkit for this kind of protocol reverse engineering reading USB traffic, working with raw byte level HID reports, and debugging hardware that occasionally locks up are all familiar territory to anyone who's built their own RC gear.

## Tools used

- **Wireshark + USBPcap** capturing raw USB traffic between the vendor software and the keyboard controller
- **Python** (for analysis) writing custom parsers for the captured `.pcapng` files, since some quirks of the USBPcap packet header format required byte level decoding not handled cleanly by generic tools
- **mingw-w64 / gcc** all application code is plain C, compiled with MinGW on Windows
- **hidapi** the cross platform HID library used to talk to the device from C, once the protocol was known (bundled in this repo with a small patch see "The vanishing lights bug" below)
- **Win32 API directly** (no framework) window creation, owner drawn controls, `Shell_NotifyIcon` for the system tray, the Service Control Manager API and Task Scheduler for the v2.2 device lock features

## The reverse-engineering process

### 1. Identifying the hardware

Using `USBDeview` and Windows Device Manager, the keyboard's USB identity was narrowed down to a composite device: **VID_048D, PID_600B** (ITE Tech. Inc.). The composite device exposes two HID interfaces:

- **Interface 0 (MI_00)** standard boot-keyboard interface, handles normal key input
- **Interface 1 (MI_01)** a vendor-defined interface (Usage Page `0xFF03`), reporting itself as `ITE Device(8291)` this is the one that controls the RGB

### 2. Capturing the protocol

With the vendor RGB software running, Wireshark (via the USBPcap extcap interface) was used to capture USB traffic while manually changing colors, switching between built-in animations, and entering per-key edit mode.

The capture format required manual parsing: USBPcap wraps each USB transfer in a `USBPCAP_BUFFER_PACKET_HEADER` before the pcapng encapsulation, and the header layout (bus, device, endpoint, transfer type, data length) had to be decoded by hand, byte-offset by byte-offset, cross-checking against known values (like the USB device descriptor's vendor/product ID appearing in a `GET_DESCRIPTOR` response) to validate the parsing was correct.

This surfaced two distinct communication channels:

- **HID Feature Reports** (`SET_REPORT`, sent as USB Control Transfers) carrying short 8-byte commands
- **HID Output Reports** (sent as USB Interrupt Transfers on Endpoint 2) carrying 64-byte chunks eight of these per "frame" make up a 512-byte per-key color buffer

### 3. Decoding the command set

By comparing the exact bytes sent for each user action (setting a solid color, switching to wave, switching to fireworks, entering custom per-key mode) against the visible result, three opcodes emerged:

| Opcode | Purpose |
|---|---|
| `0x14` | Set a solid/static color (whole keyboard or a palette slot) |
| `0x08` | Select active lighting mode (static, wave, firework, custom/per-key, …) |
| `0x12` | Prepare the device for an incoming buffer / custom-mode data stream |

### 4. Mapping every physical key

The method: reset the whole keyboard to black, then light up one key at a time. While capturing USB traffic, and diff each captured frame against the previous one to see which 3-byte slot in the buffer changed. Repeated across the entire keyboard (~103 physical positions, including a full numpad, arrow cluster, and several ISO-layout-specific keys), this produced a complete key-name → byte-offset table.

### 5. Talking to the device from C

With the protocol decoded, [hidapi](https://github.com/libusb/hidapi) was used to replicate it from a native Windows C program: opening the correct HID interface by its enumerated path, sending the same Feature Reports and Interrupt writes, and confirming the results matched what the vendor software produced.

## The vanishing lights bug fix (v2.2)

Long after the effects were working, a maddening bug remained: **the keyboard would randomly go black**, usually within the first 30–60 minutes after boot, with nothing in the application's own logs every `hid_write()` kept returning success. Restarting the app always fixed it.

The investigation, step by step:

1. **The two instance clue.** While the first instance was "dead," launching a *second* instance of the app instantly brought the first one back to life. The only thing the second instance did at startup that the first wasn't doing continuously was sending the mode initialization Feature Reports. Conclusion: the EC had been kicked out of custom/per-key mode, and the per-frame data stream was being silently ignored no error, no log, just black.
2. **Confirming it in app.** Re sending the full init sequence on every mode switch proved it: when the lights died, simply switching modes revived them. No second instance needed. So the EC was losing its mode state but *why*?
3. **A passive listener.** A read-only monitor thread was added to log anything the EC might send back on the interrupt IN endpoint around the moment of death. Result: total silence. The EC wasn't announcing anything something *external* was commanding it.
4. **The experiment.** The vendor Control Center had been set to "keyboard lighting off" (the recommended coexistence setup at the time). The hypothesis: what if the vendor service periodically *re-applies* its selected lighting state, and "off" just looks like a random blackout? Test: change the Control Center's selection from "off" to a **solid color**, then wait. Sure enough the next "random death" wasn't black anymore. The keyboard switched to the vendor's solid color. Case closed: **the GCUBridge service periodically rewrites its own lighting state to the EC**, stomping whatever any other program has set.
5. **Why it couldn't be blocked politely.** The service also turned out to hold a persistent open handle to the RGB interface from boot, and it can't simply be disabled it also manages fan curves, power limits and performance modes.

### The fix: an exclusive device lock

Windows allows a HID interface to be opened with `dwShareMode = 0` no sharing. While such a handle is held, *every other process's* attempt to open the interface fails with `ERROR_SHARING_VIOLATION` at the driver level. The bundled hidapi's `windows/hid.c` is patched accordingly: `open_device` first tries an exclusive open and falls back to the stock shared mode only if the device is already held by someone else.

With the lock held, the vendor service physically cannot touch the RGB interface its fan/power features are unaffected (they don't use this interface), and its keyboard lighting page simply doesn't load until this app exits and releases the lock.

One problem remains: the service grabs the interface at boot, before the user can launch anything. v2.2 handles this in layers:

- If the app starts **elevated** and finds the interface taken, it silently restarts the vendor service via the Service Control Manager, snatches the exclusive lock in the gap, and lets the service come back up locked out of RGB, fully functional otherwise. The vendor service is located by name (with a fallback scan of installed services), since rebrands of this chassis may ship it under different names.
- If the app starts **unelevated** and finds the interface taken, it explains the situation in a bilingual (English/Turkish) dialog and offers to relaunch itself elevated to perform the same fix one UAC prompt. Declining triggers a second explicit warning, because running shared alongside the vendor service risks the EC lockup described in "Challenges"; shared mode is only entered after that second confirmation.
- The elevated relock instance itself never runs shared: it retries the exclusive open with a progressive backoff (1–5 s), and if the lock still cannot be acquired it restores the vendor service, reports the failure, and refuses to start.
- An **Auto-Start at Login** button installs a Scheduled Task that launches the app elevated at every login (`/SC ONLOGON /RL HIGHEST`), making the whole dance invisible: no UAC prompts, no dialogs, the lock is taken silently every boot.

## Challenges along the way

- **The device could be locked up.** Sending malformed or too aggressive sequences of commands (particularly repeating the "enter custom mode" preparation commands in a tight loop) could put the keyboard's embedded controller into a bad state lighting would stop responding entirely, sometimes to *any* software, including the vendor's own. Recovery required a full EC reset (power off, disconnect charger, hold the power button for ~30 seconds to discharge residual capacitance, then power back on). This happened multiple times during development and shaped a much more cautious, incremental approach to testing new timing.
- **Windows fighting back.** The vendor's control center software turned out to have a Scheduled Task configured to relaunch it every minute for ten minutes after login discovered after wondering why the vendor app kept reappearing mid test. Disabling the scheduled task and the underlying Windows service (with recovery actions set to take no action) was necessary to get a clean, conflict free testing environment.
- **A silent saboteur.** The hardest bug of the project produced *no evidence at all*: no failed API calls, no log entries, no incoming data just a keyboard that randomly went dark. Cracking it required treating the running system as another reverse engineering target (see "The vanishing lights bug" above), and the eventual smoking gun was a designed experiment rather than a captured packet: making the invisible takeover visible by giving the vendor software a bright solid color to takeover *with*.
- **Finding a safe, real frame rate.** `hid_write()` returning success doesn't mean the firmware actually rendered that frame early tests that ignored this produced a "3 frames flash, then freeze" result at high send rates. Real world testing (timed loops, camera slow motion review) converged on roughly 30 FPS as a reliable ceiling for this specific firmware, well above the vendor software's own animation smoothness.
- **Task Scheduler's hidden defaults.** The Auto Start task initially never fired at logon, reporting only "the task has not yet run" (0x41303). The cause: tasks created via the `schtasks` command line silently default to "start only if on AC power" a death sentence on a laptop running on battery plus a 72-hour execution time limit that would have killed the app after three days. Neither default can be changed through `schtasks` switches, so the task is now registered from a full XML definition with those settings disabled. A related lesson: a process launched by Task Scheduler starts with `System32` as its working directory, which is where the app's relative path log file had been silently going all file paths are now resolved absolutely, next to the exe.
- **A missing HID Report ID byte**, the classic hidapi gotcha both `hid_send_feature_report` and `hid_write` expect the buffer's first byte to be the Report ID, and forgetting it caused silent data misalignment before being tracked down.

## What the application does

A modular Win32 desktop application (no external UI framework plain GDI, owner-drawn controls, and a manifest for modern visual styling) offering:

- **Breathing** solid color pulsing, with adjustable speed and smoothness
- **Wave** a moving color gradient across the keyboard with a user-adjustable angle (0–359°), computed from each key's real physical position (accounting for the numpad and navigation-cluster gaps)
- **Sparkle** random per-key flashes, with independently tunable speed (flash lifetime) and density (how many keys are lit at once)
- **Reactive** keys light up on keypress (fixed or random color) and fade out over a selectable duration (short/medium/long); uses `GetAsyncKeyState` polling with proper key-hold edge detection
- **Wheel** a radial rainbow rotating around the keyboard's geometric center, direction-reversible
- **Lightning** a smooth, falling "bolt" with adjustable travel speed, strike width, glow smoothness, and how many bolts can strike concurrently
- **Flame** a persistent, drifting heat simulation (not simple per-frame noise) mapped through a black→red→orange→yellow→white palette
- **Rain** falling colored drops with a trailing glow, adjustable density and fall speed, and a configurable multi-color palette
- **Matrix** four selectable sub-styles (diagonal streaks, classic vertical "digital rain" columns, laser sweep, expanding ripples), each with adjustable speed, density, and color palette
- **Static** a motionless multi zone layout: split the keyboard into up to 6 horizontal or up to 10 vertical bands and assign each zone its own color

Beyond the effects themselves:

- **Exclusive device lock** the RGB interface is opened unshared, making vendor-software interference impossible while the app runs (see the debugging story above), with an automatic elevated fix when the vendor service grabbed the interface first
- **Auto-Start at Login** one button installs/removes a Scheduled Task that launches the app elevated at logon, so the lock is acquired silently on every boot; the task is registered via a full XML definition (runs on battery, no execution time limit) and starts the app hidden in the system tray
- **Settings persistence** the selected mode (including Stop), brightness, and every per effect parameter and color are saved to a plain text `better_rgb.cfg` next to the exe and restored on the next launch, so the last-used effect resumes automatically seconds after login
- **Self-healing device handling** sleep/resume and device arrival/removal events trigger automatic reconnection and mode re-initialization
- **System tray integration** minimizing hides the app to the tray, where it keeps animating in the background
- **Bilingual dialogs** every user facing message is shown in both English and Turkish

All effects run on a background thread, computed entirely on the host and streamed to the keyboard live the firmware's own built in effects are never used.

## Known limitations

- The `Fn` key and the physical key sending the OEM "Copilot" shortcut produce no distinguishable input signal on Windows, so they cannot be used in Reactive mode.
- The numpad Enter key and the main Enter key share the same Windows virtual-key code, so they can't currently be told apart in Reactive mode (both light up together).
- The keymap was built for one specific physical keyboard layout (Turkish ISO). Other regional layouts would need their own key-mapping pass, though the underlying per-key offsets are hardware (not layout) dependent and should transfer directly only the *labels* would need adjusting.
- While the app is running, the vendor Control Center's keyboard-lighting page will not load this is the device lock working as designed, not a malfunction. It becomes available again the moment the app exits.

## Community Projects

- [BetterRGB ITE8291 Adaptive](https://github.com/ZavierChen/BetterRGB-ITE8291-Adaptive) by [@ZavierChen](https://github.com/ZavierChen): an independently developed derivative with 16 effects, power profiles, transitions, top-row indicator bars, white calibration and a Chinese/English UI. Verified on the Mechrevo 苍龙 16 X Pro (`048D:600B`).

## What's next

A frame-by-frame custom animation editor is planned, letting the user design their own sequences directly rather than picking from preset effects.

## Building from source

Requires MSYS2 (mingw64), CMake, and Ninja.

```powershell
git clone --recurse-submodules <this repository's URL>
cd "Better RGB for Tongfang ITE8291"
cmake -B build -G "Ninja"
cmake --build build
```

The resulting `build/better-rgb.exe` is self-contained (statically linked) and can be run directly no separate DLLs required.

## A note on Windows security warnings (SmartScreen / Smart App Control)

The released executable is **unsigned** this is a small open-source project, and code signing certificates from a trusted CA are a paid, identity verified service. Every fresh build also has a brand new file hash with zero reputation. As a result:

- **Microsoft Defender SmartScreen** may show "Windows protected your PC" on first launch. Click *More info → Run anyway*.
- **Smart App Control** (enabled by default on some new Windows 11 installations) may block the app entirely with "This app may be unsafe", with no per app exception possible. The app legitimately does things that look suspicious to an unsigned binary classifier: it restarts a Windows service (to take the device lock), creates a Scheduled Task (the Auto-Start feature), and relaunches itself elevated. SAC can only be turned off globally (Settings → Privacy & security → Windows Security → App & browser control → Smart App Control), and note that once turned off it cannot be re enabled without reinstalling Windows.

If you prefer not to trust a prebuilt binary at all, the project is fully open source see "Building from source" above.

## License

This project is released under the **MIT License** see the [LICENSE](LICENSE) file. Use it, modify it, redistribute it, ship it in your own projects; just keep the copyright notice.

The bundled [hidapi](https://github.com/libusb/hidapi) library (submoduled at `third_party/hidapi/`, including this project's small exclusive-open patch to `windows/hid.c`) is used and redistributed under hidapi's **BSD 3-Clause** license option; its original license files are preserved inside that folder.

## Disclaimer

This project is the result of independent reverse-engineering for personal/interoperability use. It is not affiliated with, endorsed by, or supported by the laptop's manufacturer or ITE Tech. Inc. Use at your own risk direct HID communication with embedded controllers carries a small risk of triggering unexpected firmware states (see "Challenges" above); this hasn't caused any permanent damage in the course of this project, but there are no guarantees.
