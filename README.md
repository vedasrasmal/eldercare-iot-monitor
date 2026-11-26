# ElderCare — Screenless Chest-Worn Eldercare Monitor

A chest-worn monitoring prototype for an older person living at home. It watches for a
fall, tracks heart-rate and blood-oxygen estimates when the optical signal is good enough
to trust, and gives the wearer a single physical SOS button. When something needs
attention, it messages caregivers directly on Telegram — with the readings captured at
the moment of the event. There is deliberately no display: the wearer does not need to
read a screen, charge a phone app, or learn an interface.

> **Status: engineering prototype — firmware compile-verified, hardware validation
> pending.**
>
> This is **not a validated medical monitor and not a guaranteed fall detector**. It
> should not be the sole safety mechanism for an unattended elder. No physical ESP32 or
> sensor was available for the firmware verification recorded here, so buzzer audibility
> and current, sensor timing under load, delivery over real Wi-Fi loss, and enclosure fit
> are all still to be tested. Chest-mounted MAX30102 accuracy has not been established.
> Missing or invalid data is reported as unavailable — never as a normal health result.

Author: **Vedas Rasmal** — B.Tech, Electronics and Telecommunication Engineering,
G H Raisoni College of Engineering and Management, Wagholi, Pune.

Full project report (6 pages): [docs/eldercare-iot-project-report.pdf](docs/eldercare-iot-project-report.pdf)

## At a glance

| Area | Detail |
| --- | --- |
| Microcontroller | ESP32-C3 SuperMini (single core, Wi-Fi), Arduino core 3.3.0, ESP32C3 Dev Module target |
| Motion sensing | MPU6050 accelerometer / gyroscope — free-fall, impact, stillness and posture-change evidence |
| Optical sensing | MAX30102 pulse-oximeter — heart-rate and SpO2 estimates from a 4-second rolling window |
| Alerts | Telegram Bot API to two private recipients, per-recipient retry and acknowledgement; passive piezo buzzer on device |
| Enclosure | Parametric CadQuery case, 45 × 35 × 15 mm shell (59 mm wide with strap loops), chest / back worn on a 20 mm strap |
| Updates | Over-the-air firmware upload from a phone, inside a guarded 5-minute maintenance window |

Technologies: C / Arduino, FreeRTOS, I2C, ArduinoJson, Telegram Bot API (TLS), CadQuery.

## Hardware

Both sensors share a single I2C bus, so the wiring stays short and everything fits
inside a case roughly the size of a matchbox.

| Connection | Assignment |
| --- | --- |
| MPU6050 + MAX30102 SDA / SCL | GPIO 8 / GPIO 9, shared I2C bus at 400 kHz |
| MPU6050 address / AD0 | 0x68, AD0 tied to GND |
| MAX30102 address | 0x57 |
| Passive piezo buzzer | GPIO 3, 2.5 kHz hardware PWM, non-blocking timed envelope |
| SOS tactile button | GPIO 2 to GND, internal pull-up, 35 ms stable debounce, one event per press |
| Power | 3.3 V to both sensor breakouts, common ground throughout |

Two practical details: GPIO 2, 8 and 9 also take part in ESP32-C3 boot strapping, so
external wiring must not force incompatible levels at reset — and the SOS button must
not be held during reset. Only a low-current passive piezo is driven directly from a
GPIO; a higher-current active buzzer would need a driver stage.

## How it works

### Sensing runs at full rate while the network waits its turn

The ESP32-C3 has one core, so the firmware splits work by priority: a FreeRTOS sensor
task at priority 3 and a network task at priority 1. Telegram requests, retries and
dashboard edits therefore cannot block sampling.

| Operation | Configured behaviour |
| --- | --- |
| MPU6050 sampling | 100 Hz device output, host service roughly every 10 ms |
| MAX30102 acquisition | 100 Hz, averaged by four to 25 FIFO samples per second |
| Optical calculation | 4-second / 100-sample window, new estimate every 25 samples (about once per second) |
| Optical gap handling | A FIFO gap of 100 ms or more invalidates the window and restarts acquisition |
| Telegram polling | Nominally every 2 seconds, subject to the delivery queue |
| Alarm sound | 180 ms bursts at 0, 300 and 600 ms in each second, stopping at 3 seconds |

These are configured schedules, not hard real-time guarantees. Gap and missed-window
counters are exposed in the health command, so timing problems under load become
visible instead of silently corrupting readings.

### A fall claim needs evidence, not a single spike

A single acceleration spike is a dropped device, a door slam, or a hand clap — not
necessarily a fall. The heuristic looks for a sequence: low acceleration or a strong
impact, followed by about one second of stillness, together with prior low acceleration
or a substantial posture change. A candidate triggers a short local pattern immediately,
then allows up to four seconds to assess the evidence before an event is queued.

- **Confirmed pattern:** queued as a fall event to both recipients, with a sensor
  snapshot and event age.
- **Impact without confirmation:** sent as an informational event, explicitly not
  labelled as a definite fall.
- **Cooldown:** 15 seconds between fall events, so one incident does not flood
  caregivers.
- **Starting thresholds (prototype values):** free-fall below 4.0 m/s², impact above
  20.0 m/s². These are engineering starting points, **not validated fall thresholds**.

Some real falls will not meet this heuristic, some non-falls will, and an 8 g
accelerometer range can clip severe impacts. A demonstration mode (`/demo on`,
120 seconds) labels drop-test events as DEMO, so a test is never mistaken for a real
incident.

### Vital signs are withheld when the signal is not good enough

The firmware uses the MAX30102 library algorithm at its expected 25 Hz rate, then adds
its own gates: optical amplitude and correlation checks, motion rejection, and a
requirement for consecutive consistent pulse estimates before a value is shown.
Accepted estimates are bounded to 35–200 BPM and 70–100% SpO2; outside those bounds
the reading is unavailable, not clamped.

A vital advisory requires persistently abnormal valid estimates for at least
10 seconds — advisory thresholds are 45–120 BPM and SpO2 below 92% in the prototype
configuration. Motion gating rejects readings taken while the wearer is moving, when
optical estimates degrade most. Motion and SOS monitoring continue even when no usable
pulse signal exists. No contact is never interpreted as "not worn", and no alert is
never interpreted as "safe".

Chest-mounted optical accuracy has not been established. A stationary fingertip is the
practical starting point for demonstration; bare-chest operation through an enclosure
needs its own measured signal-quality and reference comparison before any claim can be
made.

## Caregivers control everything from Telegram — no app to install

Both recipients must open the bot and press Start; only the two configured private
chats are accepted, and group chats are intentionally unsupported. Status replies go
only to the person who asked. Events and alarms go to both recipients, each with
separate delivery and retry state, and each event carries an ID so a duplicate caused
by an ambiguous network timeout can be recognised.

| Command | What it does |
| --- | --- |
| `/start`, `/status` | Current estimates, validity, motion state, reading age and any open alert |
| `/health` | Sensor availability, sample gaps, lost-event count, free heap and device IP |
| `/live` / `/stop` | Live dashboard edited every 5 seconds for up to 60 seconds, per chat |
| `/test` | One-second buzzer test and a labelled TEST event to both recipients |
| `/demo on` / `off` | Drop / impact demonstration mode for 120 seconds |
| `/ack NUMBER` | Acknowledge the current alert — logged, without declaring the wearer safe |
| `/update` | Opens the guarded firmware-update flow |
| `/help` | Command guide and alert meanings |

If Wi-Fi drops, events queue with their original snapshot. The incoming queue holds 12
events and network storage holds 16; overflow is counted, and alarms can displace
informational events. The queue is held in RAM, so it is lost on reset or power loss —
a limitation stated plainly, with no claim of guaranteed delivery.

## Security and phone-based updates

Telegram runs over HTTPS with certificate validation against public trust anchors in
firmware — there is no insecure fallback. Wi-Fi, DNS and NTP time sync are required
before Telegram starts; local monitoring continues without them. An invalid bot token
is reported on the Serial Monitor and stops repeated unauthorised requests until reboot.

After the first USB flash, new firmware can be uploaded from a phone: `/update`
explains maintenance, `/update confirm` within 30 seconds reboots into a 5-minute
update window — only when no unacknowledged alert is open. The upload page uses digest
authentication and a random form token, on trusted Wi-Fi only. Monitoring pauses during
maintenance, and a faulty new build may still need USB recovery — there is no
signed-firmware rollback.

## Enclosure

The case was modelled in CadQuery from the electronics layout, not drawn around a
guessed box. The USB ports sit on the 41 mm wall, the optical sensor looks out through
the chest-facing floor, and the button stays on the outward lid where the wearer can
reach it. Strap loops are part of the body, sized for a 20 mm strap worn on the chest
or back with elastic strips.

| Dimension | Value |
| --- | --- |
| Internal cavity | 41 × 31 × 13 mm, no extra padding assumed |
| Outer shell | 45 × 35 × 15 mm — 2 mm walls, 1 mm base, 1 mm lid, 2 mm corner radius |
| With strap loops | 59 × 35 × 15 mm; strap slots 21 × 3 mm |
| Sensor window | Exactly 6 × 3.3 mm, starting corner x = 29 mm, y = 7 mm from the USB edge, no added clearance |
| Fastening | Four M1.6 × 5 mm screws — 1.8 mm lid clearance holes, 1.2 mm body pilot holes |
| Suggested print | PETG for use, PLA for a first fit check; 0.2 mm layers, 0.4 mm nozzle, 4 perimeters, 25% infill |

Digital checks passed: both parts are single valid solids with no lid / body overlap,
and both STL meshes are closed and manifold (body 5,860 triangles, lid 2,576 triangles,
zero non-manifold edges). USB opening sizes, button fit and sensor projection are
marked provisional in the design notes and require physical checking — CAD validity is
not fit validation. See `enclosure/README.txt` for the full notes.

## Build result

| Item | Verified result |
| --- | --- |
| Build status | **Passed — compile and link only, no hardware run** |
| Target | ESP32-C3, USB CDC on boot enabled, default 4 MB partition with OTA slots |
| Toolchain / libraries | ESP32 Arduino core 3.3.0; ArduinoJson 7.4.2; Adafruit MPU6050 2.2.9 (BusIO 1.17.4, Unified Sensor 1.1.15); SparkFun MAX3010x 1.1.2 |
| Program storage | 1,227,406 bytes — 93% of the 1,310,720-byte application slot |
| Static RAM use | 42,904 bytes — 13% of dynamic memory, 284,776 bytes remaining for local variables |

Flash and RAM figures are static build figures, not measured runtime heap. Full
details in `firmware/BUILD_RESULT.txt`. Note: the SHA-256 hashes in that file refer to
the author's original local sources; `firmware/config.h` in this repo is a sanitized
template (see below), so its hash differs by design.

## Setup

Full step-by-step instructions are in [firmware/README.md](firmware/README.md) —
Arduino IDE board setup, wiring table, Telegram setup, demonstration sequence, and the
phone-based update flow.

1. Open `firmware/ElderCare_C3.ino` in Arduino IDE, keeping every companion `.h`
   file in the same folder. Install **esp32 by Espressif Systems 3.3.0**, select
   **ESP32C3 Dev Module**, USB CDC On Boot: Enabled, 4 MB flash, Default 4MB with
   spiffs partition scheme.
2. Install the libraries: Adafruit MPU6050, SparkFun MAX3010x, ArduinoJson **7.4.2**.
3. **Fill in your own credentials in `firmware/config.h`** — Wi-Fi SSID and
   password, Telegram bot token, your two recipient chat IDs, and a unique OTA
   password (at least 12 characters). The file ships with placeholders only;
   **never commit your real credentials**.
4. Click **Verify** first. After compilation succeeds, upload by USB. Serial Monitor:
   115200 baud.
5. In @BotFather, set the bot commands from `firmware/BOT_COMMANDS.txt`, then have
   both recipients open the bot and press Start.

## Repository layout

```
eldercare-iot-monitor/
├── firmware/
│   ├── ElderCare_C3.ino   # Arduino entry points
│   ├── firmware.h         # Sensor, buzzer, event, Telegram and maintenance code
│   ├── config.h           # SANITIZED TEMPLATE — pins, thresholds, credential placeholders
│   ├── certs.h            # Public TLS roots (no private keys)
│   ├── README.md          # Detailed setup, wiring, Telegram and demo guide
│   ├── BOT_COMMANDS.txt   # Paste into BotFather /setcommands
│   └── BUILD_RESULT.txt   # Compilation result and exact library versions
├── enclosure/
│   ├── build_case.py      # Parameterized CadQuery source for the case
│   ├── case_body.stl / case_lid.stl
│   ├── case_body.step / case_lid.step
│   ├── README.txt         # Dimensions, provisional openings, print and assembly notes
│   └── validation.json / mesh_validation.json
└── docs/
    └── eldercare-iot-project-report.pdf   # Full 6-page project report
```

## What this prototype does not claim

- It is not a validated medical monitor and not a guaranteed fall detector.
- No physical ESP32 or sensor was available for the firmware verification recorded
  here, so buzzer audibility and current, sensor timing under load, delivery over
  real Wi-Fi loss, and enclosure fit are all still to be tested.
- There is no battery-voltage measurement in the supplied wiring, no cellular
  fallback, no GPS and no automatic escalation to emergency services.
- False-positive and false-negative rates for falls, and long-duration reliability,
  remain unmeasured.

## Sensible next steps

Print the fit-check case in PLA, verify USB, button and sensor-window fit with the
real boards, then revise the CadQuery parameters and print in PETG. Run the documented
demonstration sequence on hardware: stationary calibration check, `/test`, fingertip
optical check, padded drop in demo mode, and a Wi-Fi-loss queue test. Compare optical
estimates against a reference pulse-oximeter at rest before any chest-accuracy claim.
Add battery monitoring hardware if runtime and low-battery alerts are needed.

## License

MIT — see [LICENSE](LICENSE).
