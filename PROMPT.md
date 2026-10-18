# Project prompt: AIO Device Configurator

You're helping me build a small, open-source **Windows desktop app in Rust** that configures my gaming peripherals without the vendor software (Razer Synapse, JBL QuantumENGINE). I want to uninstall those apps and use this instead. Before you start, ask me about anything in this prompt that is unclear or that you think is a bad idea.

## My devices

| Device | Type | Connection | What I want to control |
|---|---|---|---|
| Razer DeathAdder Elite (USB `1532:005C`) | Mouse | Wired USB | Polling rate, DPI, static color for the logo and scroll wheel |
| Razer Huntsman Mini (USB `1532:0257`) | 60% keyboard | Wired USB-C | Polling rate, one static color for the whole keyboard |
| JBL Quantum 610 Wireless *(or 600, I'll confirm)* | Headset | 2.4 GHz USB dongle | Static RGB color, plus audio settings (EQ, sidetone, mic level) and battery level, wherever the headset itself supports them |

Polling rate applies only to the mouse and keyboard; the headset has none.

## Scope for v1

**In scope**
- Detect which supported devices are connected.
- Read the current settings from each device where the protocol allows it, and show them.
- Change polling rate, DPI, static RGB, and headset audio settings.
- Remember my last chosen settings in a config file, with an "Apply saved settings" button.

**Out of scope (don't build these, not even partially)**
- Any background process, tray icon, autostart, or Windows service
- RGB effects (breathing, spectrum, wave), per-key lighting, or synced animations
- Macros or button remapping
- Logitech or any other brand
- Firmware updates
- macOS or Linux builds

## How the app behaves

- **A desktop window only.** I open it, see my devices, change settings, click Apply, and close it. Nothing keeps running after the window closes.
- When writing Razer settings, use the device's persistent-storage flag (`VARSTORE` in OpenRazer) wherever the device supports it, so settings can survive a replug. If some settings are still lost on unplug or reboot, that's acceptable for v1: write in the README which settings persist on each device. Don't add a background process to work around it without asking me first.
- If Razer Synapse, Razer Central, JBL QuantumENGINE, or their Windows services are running, show a warning banner, because they can overwrite settings or hold the device open.
- Errors are shown in the UI in plain language: device not found, access denied, device didn't respond, setting not supported. The app never panics on a device error.

## Tech stack

- Rust stable, MSVC toolchain, as a Cargo workspace
- [`hidapi`](https://crates.io/crates/hidapi) for USB HID access
- [`eframe`/`egui`](https://crates.io/crates/eframe) for the GUI
- `serde` + `toml` for the config at `%APPDATA%\aio-device-configurator\config.toml`
- `tracing` for logging, `thiserror` / `anyhow` for errors
- Ask me before adding any other significant dependency.

### Workspace layout

```
crates/
  core/     Device trait, capability types, errors, Transport trait
  razer/    Razer protocol, plus DeathAdder Elite and Huntsman Mini drivers
  jbl/      JBL Quantum protocol and driver
  gui/      egui app (the main binary I'll run)
  devtool/  developer CLI: list HID interfaces, send and read raw reports, run single commands
```

### Design

- Each device driver reports its **capabilities**: the polling rates it supports, its DPI range and step, its RGB zones, and its audio controls. The GUI builds its controls from these, so the GUI contains no device-specific code.
- All hardware I/O goes through a `Transport` trait, so drivers can be unit-tested with a mock transport that records the exact bytes sent and replays canned responses.

## Protocol references

### Razer

The Razer protocol has been well reverse-engineered by two GPL projects. Treat them as the source of truth:

- **OpenRazer**: https://github.com/openrazer/openrazer, especially `driver/razercommon.*`, `driver/razerchromacommon.*`, `driver/razermouse_driver.c` and `driver/razerkbd_driver.c`
- **OpenRGB**: https://gitlab.com/CalcProgrammer1/OpenRGB, the `Controllers/RazerController` directory

Take these from them:
- The report layout: a 90-byte feature report made of status, transaction ID, remaining packets, protocol type, data size, command class, command ID, arguments and CRC.
- The **per-device** values: transaction ID, LED IDs, supported polling rates, and DPI range (the DeathAdder Elite goes up to 16,000). Look them up for each exact USB PID. Never assume that a value from one device applies to another.
- Which USB interface or HID collection accepts control reports on Windows.

If HID access fails on Windows, check whether Razer's own drivers are still installed and holding the device, even after Synapse has been uninstalled.

### JBL Quantum

I found **no public protocol documentation or open-source driver** for the Quantum 600/610, so this part needs reverse engineering:

1. I'll temporarily install QuantumENGINE, Wireshark and USBPcap.
2. Give me a step-by-step capture procedure: which setting to change in QuantumENGINE, one at a time, and how to save and send you each capture.
3. Analyze the captures I send and document the protocol in `docs/jbl-protocol.md`.
4. Implement only commands that appear in those captures.

Notes:
- Some QuantumENGINE features (QuantumSURROUND, and possibly EQ) may be processed on the PC by an audio driver rather than inside the headset. If the captures show that, tell me, and leave that feature out of v1 instead of reimplementing audio processing.
- The headset's RGB doesn't light up over the 3.5 mm cable, so all testing goes through the 2.4 GHz dongle.

## Safety rules (important: I don't want to brick anything)

- Only send commands confirmed in OpenRazer/OpenRGB source or in my own USB captures. No fuzzing and no guessing command IDs.
- Never send firmware-update, bootloader, or factory-reset commands.
- Validate every value before sending it: DPI within the device's range and step, polling rate from the supported list, colors as valid RGB.
- If a device returns an error or "not supported" status, stop and report it. Don't retry with different payloads.

## Working with my hardware

You can't see my devices. Whenever you need information from the hardware (HID enumeration, test results, captures), give me the exact commands to run on Windows and tell me what to paste back. Build `devtool` early so I have something to run.

## Milestones

Do these in order. At the end of each one, stop, tell me exactly what to test on my PC, and wait for my results before moving on.

1. **Skeleton + enumeration.** Workspace, CI, and `devtool list`, which prints every HID interface (VID, PID, interface number, usage page, usage, path) for Razer and JBL devices.
   *Done when:* I run it and both Razer devices and the JBL dongle show up.
2. **Razer protocol core.** Packet builder and parser, CRC, send plus read response with a timeout, and status-code handling. Unit tests compare the packets you build against byte sequences taken from OpenRazer.
   *Done when:* `cargo test` passes and `devtool` can do a harmless read (firmware version or serial) on both Razer devices.
3. **DeathAdder Elite driver.** Read and set polling rate (125/500/1000 Hz), DPI, and static colors for the logo and scroll wheel.
   *Done when:* each change is visible on my mouse and reading it back matches what was set.
4. **Huntsman Mini driver.** Read and set polling rate (whatever rates OpenRazer lists for it) and a static color for the whole keyboard.
   *Done when:* same as for the mouse.
5. **GUI.** Device list, a settings panel per device built from its capabilities, an Apply button, the vendor-app warning, error display, and config save/load with "Apply saved settings".
   *Done when:* I can do everything from milestones 3 and 4 without the command line.
6. **JBL research.** The capture procedure, my captures, and `docs/jbl-protocol.md`.
   *Done when:* the doc covers RGB and every audio setting the headset handles itself.
7. **JBL driver.** Static RGB first, then each audio feature the protocol doc confirms, and battery level if it can be read.
   *Done when:* each setting works from the GUI.
8. **Release.** A README (install, usage, supported devices, which settings persist after unplug or reboot), the LICENSE file, credits, and a GitHub Actions workflow that builds a Windows `.exe` as a release artifact.

## Code quality

- `cargo fmt`, `cargo clippy -- -D warnings` and `cargo test` run in CI on `windows-latest`.
- No panics in device code paths: every I/O error reaches the UI as a readable message.
- Small, focused commits, at least one per milestone.

## License and credits

- License the project **GPL-2.0**. Reusing protocol details or code from OpenRazer and OpenRGB is fine.
- Credit OpenRazer and OpenRGB in the README, and add a code comment wherever a protocol detail came from them.
- Don't use Razer or JBL logos or branding in the app.
