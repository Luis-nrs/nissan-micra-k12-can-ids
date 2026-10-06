# Nissan Micra K12 – CAN IDs (Body CAN)

Decoded CAN signals of a 2009 Nissan Micra K12, found by passively sniffing the bus on my own car and
checking every signal against the real thing (switch, pedal, dashboard, fuel receipt, factory manual).

| | |
|---|---|
| Model | Nissan Micra K12, Visia (base trim), 2009, no A/C |
| Engine | 1.2 L CR12DE petrol, 5-speed manual |
| Bus | 500 kbit/s, 11-bit identifiers, OBD-II pins 6/14 |
| Hardware | ESP32-C6 + CAN transceiver, listen-only |
| Data | ~730,000 one-second samples of every ID (late Sept – early Oct 2026), aligned with a GPS/flight recorder |

**Conventions:** bytes are numbered 0–7 from the start of the frame. Bit masks are given in hex
(`0x10` = bit 4). Multi-byte values are big endian. "Confirmed" means checked against a real reference, not just
correlated once. The **Source** column names the ECU that sends the frame (see [Which ECU sends what](#which-ecu-sends-what)).

## Contents

- [Quick reference by ID](#quick-reference-by-id)
- [Which ECU sends what](#which-ecu-sends-what)
- [Engine](#engine) · [Speed and wheels](#speed-and-wheels) · [Fuel](#fuel) · [Body (BCM)](#body-bcm) · [Lights and warnings](#lights-and-warnings)
- [Starter](#starter) · [Network management](#network-management) · [Immobilizer (NATS)](#immobilizer-nats)
- [Diagnostics](#diagnostics) · [Vehicle constants](#vehicle-constants)
- [Probable](#probable) · [Unresolved](#unresolved) · [Not on the bus / not found](#not-on-the-bus--not-found)
- [Corrections to earlier versions](#corrections-to-earlier-versions)

## Quick reference by ID

| ID | Source | Byte · bits | Signal | Decode |
|---|---|---|---|---|
| `0x181` | ECM | 0–1 | Engine speed | `(b0*256 + b1) / 8` rpm |
| `0x181` | ECM | 2 | Throttle plate | raw, ≈ 42 closed (overrun), ≈ 55 idle, ≈ 100 wide open |
| `0x181` | ECM | 3 | Accelerator pedal | `(b3 − 16) * 100 / 208` %, clamp 0–100 |
| `0x181` | ECM | 4 | Engine load | raw, idle ≈ 52, overrun ≈ 47, full load ≈ 96–100 |
| `0x1F9` | ECM | 0 · `0x20` | Engine running | set = running |
| `0x1F9` | ECM | 0 · `0x40` | Radiator fan request | set = fan on (from ≈ 95 °C) |
| `0x1F9` | ECM | 2–3 | Engine speed (copy) | same as `0x181` bytes 0–1 |
| `0x215` | IPDM | 1 · `0x03` | Starter engaged | set ≈ 1 s while cranking |
| `0x215` | IPDM | 1 · `0x04` | Radiator fan (copy) | identical to `0x1F9` byte 0 `0x40` |
| `0x215` | IPDM | 1 · `0x40` | Reverse switch | set = reverse engaged |
| `0x280` | Meter | 4–5 | Speed, cluster group (copy of `0x355`) | `raw * 0.01` km/h |
| `0x284` | ABS | 0–1, 2–3 | Wheel speed **front right, front left** | `raw16 * 1.13 / 256` km/h |
| `0x284` | ABS | 4–5 | Speed, high-res (copy of `0x354`) | `raw * 0.01` km/h |
| `0x285` | ABS | 0–1, 2–3 | Wheel speed **rear right, rear left** | `raw16 * 1.13 / 256` km/h |
| `0x285` | ABS | 4 | Vehicle speed, whole km/h | ≈ `0x354` speed / 1.013 |
| `0x2DE` | Meter | 6–7 | Fuel level sender | raw, non-linear, `0xFFFF` below range, updated every 30 s while driving |
| `0x354` | ABS | 0–1 | Speed, high-res | `raw * 0.01` km/h |
| `0x354` | ABS | 2–3 | Distance counter | 16-bit, ≈ 0.10 m per count, wraps |
| `0x354` | ABS | 4 · `0x0A` | ABS / brake warning lamp | set ≈ 1 s at ignition on (bulb check) and while cranking |
| `0x354` | ABS | 6 · `0x10` | Brake pedal | set = pressed |
| `0x355` | Meter | 0–1 | Speed, cluster group | `raw * 0.01` km/h |
| `0x358` | BCM | 1 · `0x40` | Heater blower switch | set = blower on (**not** the radiator fan) |
| `0x358` | BCM | 3 | Central locking command | `0x14` lock, `0x0A` unlock – short pulse ≈ 1 s before the status changes |
| `0x358` | BCM | 5 · `0x03` | Central locking status (copy) | `0x03` locked, `0x00` unlocked |
| `0x35D` | BCM | 0 · `0x06` | Rear window heater switch | both bits set = on |
| `0x35D` | BCM | 0 · `0x40` | Ignition switch in START | set ≈ 1 s while cranking |
| `0x35D` | BCM | 1 | Sleep / wake-up | `0x03` awake, `0x00` sleep request (bus goes silent 3 s later) |
| `0x35D` | BCM | 2 · `0x40` | Front wiper request | set = wiper on (continuous / low / high) |
| `0x35D` | BCM | 2 · `0x80` | Washer | set = washer pulled |
| `0x35D` | BCM | 4 · `0x10` | Brake pedal (copy) | set = pressed |
| `0x35D` | BCM | 4 · `0x40` | Vehicle moving | set = rolling |
| `0x551` | ECM | 0 | Coolant temperature | `b0 − 40` °C |
| `0x551` | ECM | 1 | Fuel injection counter | 8-bit counter, ≈ 0.117 ml per count |
| `0x551` | ECM | 5 · `0x03` | Engine status | `0` stop, `1` stall, `2` run, `3` crank |
| `0x5C5` | Meter | 0 · `0x04` | Handbrake | set = engaged |
| `0x5C5` | Meter | 1–3 | Odometer | 24-bit, 1 count = 1 km |
| `0x5E4` | EPS? | 0 · `0x04` | Warning lamp, on until the engine runs | see [Lights and warnings](#lights-and-warnings) |
| `0x60D` | BCM | 0 · `0x06` | Light switch | `0x04` position 1 (parking), `0x06` position 2 (low beam) |
| `0x60D` | BCM | 0 · `0x08` `0x10` `0x20` `0x40` | Doors FL, FR, RL, RR | set = open |
| `0x60D` | BCM | 0 · `0x80` | **Tailgate** | set = open (byte 2 `0x08` is the inverse) |
| `0x60D` | BCM | 1 · `0x20` / `0x40` | Turn signal left / right | both = hazards |
| `0x60D` | BCM | 2 · `0x04` | Rear fog light | set = on |
| `0x60D` | BCM | 2 · `0x10` | Central locking status | set = locked |
| `0x60D` | BCM | 5 | Coolant temperature (copy) | `b5 − 40` °C, `0xFF` = no value yet |
| `0x60D` | BCM | 6 · `0x10` | Reverse light | set = reverse engaged |
| `0x625` | IPDM | 0 · `0x01` | Rear window heater relay | set = heating |
| `0x625` | IPDM | 0 · `0x04` | Front wiper out of park position | ≈ 1 s per wipe ("auto stop" signal) |
| `0x625` | IPDM | 0 · `0x30` | Starter relay | set ≈ 1 s while cranking |
| `0x625` | IPDM | 1 | Headlamp status | `0x00` off, `0x40` parking, `0x60` low, `0x50` / `0x10` high |
| `0x625` | IPDM | 3 · `0x80` | Oil pressure warning | set = lamp on |

## Which ECU sends what

The factory manual (LAN section, "CAN communication unit" table) puts this car in **Type 4**: CR12DE, manual
gearbox, ABS, no Intelligent Key. Six units are on the bus: **ECM, combination meter, EPS, BCM, ABS, IPDM E/R**.
The manual has no IDs or byte layouts, only which unit sends which signal. The IDs were matched to units by
the order in which they appear when the bus wakes up (five independent wake-ups, same result every time):

- **Wake with the bus, before the ignition** (units on battery power): `0x35D`, `0x358`, `0x60D` (BCM),
  `0x625`, `0x215` (IPDM), `0x280`, `0x355`, `0x5C5`, `0x2DE` (meter), `0x682`.
- **Wake at ignition on** (units on ignition power): `0x181`, `0x1F9`, `0x551`, `0x500`, `0x511` (ECM / NATS),
  `0x284`, `0x285`, `0x354` (ABS), `0x5E4`, `0x300` (EPS).

This mattered more than expected: knowing that `0x625` is the IPDM (which the manual says reports the
*wiper park position*) and that the tailgate switch is wired to the BCM is what finally sorted out the tailgate
bit (see [Corrections](#corrections-to-earlier-versions)).

## Engine

**Engine speed – `0x181` bytes 0–1:** `(b0*256 + b1) / 8`. Not the OBD-II `/4` scaling; checked against the
tachometer (700 rpm idle, ~2500 rpm cruising). `0x1F9` bytes 2–3 carry the same value.

**Accelerator pedal – `0x181` byte 3:** drive-by-wire pedal position. `0x10` released, `0xE0` floored, monotonic
in between. Reads `0x10` both at idle and when coasting.

**Throttle plate – `0x181` byte 2:** ≈ 42 closed (overrun), ≈ 55 at idle, ≈ 100 wide open. It also opens with
the pedal when the ignition is on and the engine is off, which a load value would not do.

**Engine load – `0x181` byte 4:** the model `pulses/s = K · rpm · (b4 − 44) + C` predicts the injection counter
with R² = 0.87 (rpm alone 0.48, pedal alone 0.69). ≤ 46 with the pedal released above 1300 rpm is overrun.

**Coolant temperature – `0x551` byte 0:** `b0 − 40` °C. The manual lists the "engine coolant temperature signal"
from the ECM to the meter. Broadcast even though this trim only has a warning lamp. The BCM gets a copy in
`0x60D` byte 5 (identical in 98 % of samples, the rest ±1; `0xFF` until the ECM has sent a value).

**Engine status – `0x551` byte 5, bits 0–1:** the manual's "engine status signal" (ECM → EPS), whose
diagnostic monitor knows exactly four states: stop, stall, run, crank. In the logs: `0` = ignition on, engine not
started yet; `1` = after the engine stopped (stall); `2` = running (43,067 of 43,616 samples with the engine
running); `3` = cranking. Use `== 2` for "running" – bit `0x02` alone is also set while cranking.

**Engine running – `0x1F9` byte 0, bit `0x20`:** set while the engine runs.

**Radiator fan request – `0x1F9` byte 0, bit `0x40`:** the manual's "cooling fan speed request" (ECM → IPDM).
Never set below 85 °C coolant, set in 99 % of samples from 95 °C – the switch point the manual gives for cars
without A/C (fan on/off only). `0x215` byte 1 bit `0x04` (IPDM) is an exact copy.

## Speed and wheels

**High-resolution speed – `0x354` bytes 0–1** (copy in `0x284` bytes 4–5): `raw * 0.01` km/h. Settles to exactly
0 at standstill. Reads ≈ 2.7 % above the GPS-fitted coarse value (rolling circumference).

**Whole km/h – `0x285` byte 4:** the same speed rounded to km/h (≈ `0x354` value / 1.013).

**Cluster speed – `0x355` bytes 0–1** (copy in `0x280` bytes 4–5): same encoding, ≈ 4.8 % above `0x354`. The
needle shows another few percent more on top.

**Wheel speeds – `0x284` and `0x285`, bytes 0–1 and 2–3:** four **16-bit** values, `raw16 * 1.13 / 256` km/h
(bytes 1 and 3 are the fine part; earlier versions of this list used only bytes 0 and 2). Position, determined
from corners (outer wheels faster) and wheelspin when pulling away (only the front wheels spin):

| | Right | Left |
|---|---|---|
| Front | `0x284` bytes 0–1 | `0x284` bytes 2–3 |
| Rear | `0x285` bytes 0–1 | `0x285` bytes 2–3 |

**Distance counter – `0x354` bytes 2–3:** 16-bit counter that only advances while the car moves, ≈ 0.10 m per
count against the speed integral, wraps at 65536.

**Vehicle moving – `0x35D` byte 4, bit `0x40`:** sets between 5.7 and 15.8 km/h, clears between 0 and 3.4 km/h,
never set at standstill. A noise-free "car is rolling" flag (GPS reports 0.6–2.4 km/h while parked). It is in a
BCM frame, so it is derived from the meter's speed and lags a little.

**Counters:** `0x284`/`0x285` byte 6 is a shared message counter, byte 7 a checksum. `0x280` bytes 2–3 and
`0x5E4` bytes 1–2 are counters/checksums as well (no relation to any physical signal).

## Fuel

**Fuel injection counter – `0x551` byte 1:** the manual's "fuel consumption monitor signal". 8-bit counter that
wraps roughly every 15 s at idle, so use differences. Its rate rises with rpm and load, and it **stops completely
during overrun fuel cut** – nothing else on the engine behaves like that. **≈ 0.117 ml per count**, confirmed
over several fills against receipts (warm idle then reads 0.85–0.9 L/h).

**Fuel level – `0x2DE` bytes 6–7:** raw sender value, 16-bit, **not linear** – calibrate with several (raw,
litres) points across the tank. On this car the reserve lamp came on at raw ≈ 699 (≈ 10 L of 46 L). While
driving, the meter updates the value **every 30 s** (1055 of 1160 intervals exactly 30 s), so don't expect it
to react faster. Below the sender's range it becomes `0xFFFF`; near that point it flickers, so debounce it.

## Body (BCM)

`0x60D` is the BCM's collective message; the door, light, lock, fog and reverse bits were confirmed by operating
the switch several times with the ignition on.

| Signal | Location | Notes |
|---|---|---|
| Doors | `0x60D` byte 0: `0x08` FL, `0x10` FR, `0x20` RL, `0x40` RR | |
| **Tailgate** | `0x60D` byte 0: `0x80` | Byte 2 `0x08` is always the inverse. 7 openings in the logs, all at standstill, three of them with all four doors shut, never while driving. The manual wires the tailgate switch to the BCM |
| Turn signals | `0x60D` byte 1: `0x20` left, `0x40` right | hazards set both |
| Light switch | `0x60D` byte 0: `0x04` position 1, `0x06` position 2 | the switch as the BCM reads it; the IPDM's lamp status is in `0x625` byte 1 |
| Rear fog light | `0x60D` byte 2: `0x04` | |
| **Central locking status** | `0x60D` byte 2: `0x10` | `0x18` locked, `0x08` unlocked. Follows the interior button, the key and the remote, also with the ignition off. The locks have no feedback switches – the BCM remembers its last command. `0x358` byte 5 = `0x03` is a second copy (identical in 99.996 % of samples, both edges in the same second) |
| **Central locking command** | `0x358` byte 3 | short pulse ≈ 1 s *before* the status changes: `0x14` lock, `0x0A` unlock. Seen again when pressing lock on an already locked car. This is the BCM *reporting* what it does – it drives the lock motors over wires, so sending this frame does not lock the car |
| Reverse light | `0x60D` byte 6: `0x10` | the reverse switch; `0x215` byte 1 `0x40` (IPDM) shows the same |
| Coolant (copy) | `0x60D` byte 5 | `b5 − 40` °C |

Other body signals:

| Signal | Location | Notes |
|---|---|---|
| Front wiper request | `0x35D` byte 2, `0x40` | continuous wiping. In interval mode the bit mostly stays 0 – use the IPDM park signal below |
| Washer | `0x35D` byte 2, `0x80` | **not** the tailgate (an earlier version of this list said so) |
| Wiper park position | `0x625` byte 0, `0x04` | IPDM "auto stop" signal: ≈ 1 s per wipe, every 6–13 s in interval mode. All 58 episodes in the logs were with the ignition on while wiping, never together with the tailgate bit |
| Rear window heater | `0x35D` byte 0, `0x06` (switch, BCM) / `0x625` byte 0, `0x01` (relay, IPDM) | `0x88` off ↔ `0x8E` on |
| Brake pedal | `0x354` byte 6, `0x10` | copy in `0x35D` byte 4 `0x10` |
| Handbrake | `0x5C5` byte 0, `0x04` | |
| Odometer | `0x5C5` bytes 1–3 | exactly the dashboard reading, checked twice 715 km apart |
| Heater blower | `0x358` byte 1, `0x40` | the manual's "heater fan switch signal" (BCM → ECM). Not the radiator fan |

## Lights and warnings

**Headlamp status – `0x625` byte 1:** `0x00` off, `0x40` parking lights, `0x60` low beam, `0x50` / `0x10` high
beam (probably held vs. flash-to-pass).

**Oil pressure warning – `0x625` byte 3, bit `0x80`:** set with ignition on and engine off, clears one second
after the engine starts, returns at shutdown. The oil pressure switch is wired to the IPDM.

**`0x5E4` byte 0, bit `0x04`:** behaves exactly like the oil lamp (on with ignition, off ≈ 1–2 s after the engine
starts). But `0x5E4` wakes with the ignition-powered units, not with the IPDM, so it is more likely the **EPS
warning lamp** (the EPS also keeps its lamp on until the engine runs) than a copy of the oil signal.

**ABS / brake warning lamp – `0x354` byte 4, bits `0x0A`:** set for about one second after ignition on (bulb
check, also when the engine is not started afterwards) and again while cranking (voltage dip).

## Starter

Four independent signals show the starter turning. Each is set for only about a second, so a 1 Hz log catches
roughly half the starts; at the full frame rate all of them are usable. Together they make remote-start
supervision robust (relay pulled but starter not turning → abort).

| Location | Unit | Evidence |
|---|---|---|
| `0x551` byte 5 = `3` | ECM | the manual's "crank" engine state |
| `0x625` byte 0, `0x30` | IPDM | 17 samples, all 0–1 s before the engine caught, **zero** false positives |
| `0x35D` byte 0, `0x40` | BCM | 20 of 21 samples right before the engine caught (ignition switch START) |
| `0x215` byte 1, `0x03` | IPDM | 19 of 20 samples right before the engine caught |

`0x500` byte 0 bit `0x04`, listed here earlier as a starter candidate, sits in the immobilizer frame and was set
at only 3 of 109 starts.

## Network management

**Sleep / wake-up – `0x35D` byte 1:** the manual's "sleep/wake up signal" from the BCM. `0x03` while awake. About
60 s after locking it drops to `0x00`, and the whole bus goes silent **exactly 3 s later** (55 of 55 times).
After a wake-up it reads `0x00` for about 2 s, then `0x03` again. Useful to tell a normal bus sleep from a
wiring dropout.

**`0x600`, `0x602`, `0x682`:** appear on the bus but carry no data – every byte was 0 in every sample.
`0x682` is the first frame after a wake-up; `0x600`/`0x602` only showed up together with OBD traffic.

## Immobilizer (NATS)

`0x500` bytes 1–4 take a new random 32-bit value exactly once per ignition-on, about 5 s after the key is
turned, and stay constant through stalls and restarts in the same ignition cycle. `0x511` bytes 1–6 run a short
exchange at the same moment. That is the challenge/response between the BCM and the ECM (the manual says the
NATS ID check runs over CAN). Not physical signals – in particular, `0x511` byte 1 bit `0x20` is **not** a fuel
reserve lamp, even though it once looked like one. `0x500` byte 0 is a constant `0x02`.

## Diagnostics

- **OBD-II:** functional requests on `0x7DF`, the engine ECU answers on `0x7E8`. A Bluetooth ELM327 on the same
  port also sends on `0x7DF` and, while searching protocols (`ATSP0`), on 29-bit `0x18DB33F1`. **Two testers
  sending on `0x7DF` at the same time collide** – my own sender went bus-off every time until it waited for
  60 s of silence from the other tester.
- **Other ECUs:** a scan of `0x700–0x7F7` with harmless read requests (KWP `1A 80`, `21 80`, UDS `22 F1 90`,
  tester present) got answers from `0x740 → 0x760`, `0x745 → 0x765` and `0x74D → 0x76D`, all `7F 1A 11`
  (service not supported, but the ECU is there). The electric power steering did not answer.

## Vehicle constants

Measured on this car (original tyre size), useful for gear detection and shift hints:

| Gear | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|
| rpm per km/h | 132.7 | 73.3 | 51.3 | 36.7 | 29.2 |

Pedal calibration: `0x10` released, `0xE0` floored. Overrun fuel cut ends below about 1300 rpm.

## Probable

Consistent in the data, but not yet backed by a second kind of evidence:

| ID | Byte · bits | What was seen |
|---|---|---|
| `0x354` | 5 · `0x80` | Set only during the three hardest stops in the logs; other stops just as hard did not set it, so it is not a plain deceleration threshold. Most likely **ABS control active** (depends on grip) |
| `0x300` | 0 | Small integer: 0 at standstill, highest at 1–10 km/h, falling with speed, higher when steering. Looks like the **EPS assist** level |
| `0x5C5` | 0 · `0x20` | Set only while the fuel sender reads `0xFFFF` – fuel below the sender's range |
| `0x2DE` | 4 low nibble + 5 | `0xFFF` below ≈ 20 km/h, `0x000` above (switches at 19–21 km/h). A field the meter of this trim does not fill, marked invalid at low speed |
| `0x5C5` | 0 · `0x40`/`0x80` | `0x40` with ignition on, `0x80` off; a short `0x40` pulse when locking/unlocking with the hazard flash – the meter waking up briefly |

## Unresolved

| ID | Byte · bits | What is known |
|---|---|---|
| `0x2DE` | 4 high nibble (0/1/2) | Changes only on the 30-s fuel-level update and survives ignition cycles. Unrelated to speed (moving averages 30 s – 20 min), fuel level, fuel change or lights |
| `0x354` | 6 · `0x40` | One single event: start of braking at 34 km/h in a tight turn |
| `0x60D` | 3 · `0x02` | One single event: ignition off, tailgate and two doors open, light switch turned off at the same moment |

## Not on the bus / not found

| Signal | Status |
|---|---|
| Fuel reserve lamp | **Not on CAN.** The fuel sender is wired directly to the cluster, which switches the lamp and chime itself. Use the fuel level raw value instead |
| Battery / system voltage | No byte with the expected resting vs. charging shape |
| Outside temperature | Not present (no display on this trim) |
| Steering angle | Not present (no ESP on this trim) |
| Seatbelt switch | Sensor disconnected on this car |
| Airbag / SRS status | Not pursued |
| Remote-control lock | **Not possible over CAN.** The BCM only *reports* its lock commands (`0x358` byte 3); it does not accept them from the bus on this variant (only the Intelligent Key variant exchanges lock data over CAN). Locking needs the hard-wired button input |

## Corrections to earlier versions

- **Tailgate** is `0x60D` byte 0 `0x80`. Earlier versions said `0x35D` byte 2 `0x80` (that is the washer) and
  firmware once used `0x625` byte 0 `0x04` (that is the wiper park position from the IPDM – it caused false
  "tailgate open" warnings in interval wiping).
- **Wheel speeds** are 16-bit (bytes 0–1 and 2–3), and `0x284` is the **front** axle, `0x285` the **rear**.
- **`0x358` byte 1 `0x40`** is the heater blower, not the radiator fan. The radiator fan request is `0x1F9` byte 0 `0x40`.
- **`0x551` byte 5** is a 4-state engine status, not a single "running" bit plus a "cranking" bit.
- **`0x358` byte 3** is the lock *command* pulse, not the lock status (that is why it looked "never set").

---

## The project behind this data

These IDs drive **Micra-Core**, a custom dashboard and diagnostics system for this car:

- ESP32-C6 on the OBD-II port (CAN transceiver), relays for locks, windows and remote start, GPS
- Raspberry Pi 5 with the dashboard display, dashcam/sentry cameras, a flight recorder and a local assistant
- Live speedometer with gear display, shift hints tuned for fuel economy, instant and trip consumption from the
  injection counter, range, trip log, fuel log and cost tracking
- Reads and clears OBD-II fault codes, keyless lock/unlock via the phone

Everything here came from passive observation on my own car – no dealer tools, just a transceiver, the factory
manual and physically triggering every switch and pedal.
