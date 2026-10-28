# WiFi Sensing on a Laptop: Testing RuView Without Special Hardware

Can an ordinary laptop tell whether someone is moving in a room, using nothing but the home WiFi signal?

I built and ran [RuView](https://github.com/ruvnet/RuView), an open-source WiFi sensing platform, on a Windows laptop with **no extra hardware**. The answer was yes: motion was detectable.

## Setup

| Item | Detail |
|---|---|
| Machine | Lenovo ThinkPad, Windows |
| Sensing mode | `--source wifi`: RSSI (signal strength) read through `netsh wlan` |
| Extra hardware | None. No ESP32 or CSI-capable research NIC |
| Toolchain | Rust (rustup), project-pinned 1.89; MSVC Build Tools |
| Date | 28 Oct 2026 |

RuView's full features (pose, breathing, heart rate, people counting) need **Channel State Information (CSI)** from an ESP32-S3 or a research NIC. A normal laptop only exposes RSSI, which is a single signal-strength number. This test explores what that much more limited signal can still reveal.

## Building on Windows: problems hit and fixed

```powershell
git clone https://github.com/ruvnet/RuView.git
cd RuView
git submodule update --init --recursive   # fix 1
cd v2
cargo build --release -p wifi-densepose-sensing-server
.\target\release\sensing-server.exe --source wifi --tick-ms 500 --bind-addr 127.0.0.1 --http-port 3000 --ws-port 3001 --ui-path ..\ui
```

1. **`failed to read ...\vendor\rufield\...\Cargo.toml`**: the repo depends on git submodules, which a plain `git clone` does not fetch. Fixed with `git submodule update --init --recursive`.
2. **`linker 'link.exe' not found`**: Rust's MSVC target needs Microsoft's C++ linker. Fixed by installing Visual Studio Build Tools with the "Desktop development with C++" workload.

The server was bound to `127.0.0.1` so the dashboard was reachable only from the local machine. The dashboard is at `http://localhost:3000/ui/index.html`, and `/health` confirmed `"source":"wifi:<SSID>"` and `"state":"live"`.

## Results

| State | Signal variance | RSSI (dBm) |
|---|---:|---:|
| Startup noise (weak signal, no baseline yet) | ~3,588 | -90 |
| Quiet room (baseline) | ~33 | -52 |
| Someone moving | ~90 | -52 |
| Quiet again | ~16 | -52 |

- **Movement raised signal variance about 3-5x** over a quiet room (2.7x against the first baseline, 5.6x against the second).
- Startup readings on a weak signal (-90 dBm) were dominated by noise. Results became meaningful once the signal was strong (-52 dBm) and the server had settled on a baseline.

Raw readings and a repeat-test log are in [`data/ruview_wifi_test.xlsx`](data/ruview_wifi_test.xlsx).

## What did *not* work (and why)

| Dashboard element | Trustworthy in RSSI mode? | Reason |
|---|---|---|
| Presence / motion | Yes, coarsely | Movement visibly changes signal variance |
| Person count | No | Showed 2 at startup, then settled at 1 with two people at home. In RSSI mode the count comes from an activity score that saturates with one moving person (noted in RuView's own source comments). Real counting needs per-node ESP32 data. |
| 3D figures / pose | No | The visualisation always renders figures; RSSI carries no spatial information |
| Heart rate / breathing | No | Showed `--`; the signal was too noisy to estimate |
| "Baseline calibration..." caption | No | Left over from the demo scenarios, not a description of the real room |

## Security and privacy angle

The interesting finding isn't the dashboard. It's that **WiFi signals leak information about physical activity in a space**, even from a commodity laptop with no special hardware.

- **Threat:** anyone with a device on, or near, the same wireless environment can infer occupancy and movement patterns, such as whether someone is home, when a room is in use, or daily routines. With CSI-capable hardware (an ESP32 costs about US$9), RuView reports finer-grained inference, including breathing and through-wall presence.
- **Why it matters:** this is passive sensing. It needs no camera, no microphone, and no access to the victim's devices, so traditional privacy controls (covering cameras, app permissions) don't address it.
- **Mitigations to explore:** limiting which devices can join or sit near the network, monitoring for unknown devices on the network, randomising transmit power and channels (which degrades sensing), and physical RF shielding for sensitive rooms. RuView's own repo includes **WiFi Veil**, a "privacy firewall" that shapes outgoing WiFi feedback so passive sniffers can't infer activity, without jamming. Its published results are synthetic (from tests, not measured on real radios), so testing it on hardware would be a worthwhile next step.

## Limitations

- One room, one laptop, one session, and a handful of manual readings. This is a demonstration, not a controlled study.
- Variance values were read manually off the dashboard.
- RSSI is affected by other devices, people and neighbouring networks, so readings are noisy.

## Next steps

- Repeat the still -> move -> still test several times and fill in the log in the spreadsheet.
- Add an ESP32-S3 node for real CSI and compare against these RSSI results.
- Test countermeasures (for example `wifi-veil`) and measure how much they reduce detectability.

## Credits

[RuView](https://github.com/ruvnet/RuView) by ruvnet (MIT licence). This repo contains only my own test notes and data.
