# Nissan Micra K12 – CAN IDs (Body CAN)

Decoded CAN signals of a 2009 Nissan Micra K12, found by passively sniffing the bus on my own car and
checking every signal against the real thing (switch, pedal, dashboard, fuel receipt, factory manual).

| | |
|---|---|
| Model | Nissan Micra K12, Visia (base trim), 2009 |
| Engine | 1.2 L CR12DE petrol, 5-speed manual |
| Bus | 500 kbit/s, 11-bit identifiers, OBD-II pins 6/14 |
| Hardware | ESP32-C6 + CAN transceiver, listen-only |

**Conventions:** bytes are numbered 0–7 from the start of the frame. Bit masks are given in hex
(`0x10` = bit 4). Multi-byte values are big endian. "Confirmed" means checked against a real reference, not just
correlated once.

## Contents

- [Quick reference by ID](#quick-reference-by-id)
- [Engine](#engine) · [Speed and wheels](#speed-and-wheels) · [Fuel](#fuel) · [Body (BCM)](#body-bcm) · [Lights and warnings](#lights-and-warnings)
- [Diagnostics](#diagnostics) · [Vehicle constants](#vehicle-constants)
- [Candidates](#candidates) · [Not on the bus / not found](#not-on-the-bus--not-found)

## Quick reference by ID

| ID | Byte · bits | Signal | Decode |
|---|---|---|---|
| `0x181` | 0–1 | Engine speed | `(b0*256 + b1) / 8` rpm |
| `0x181` | 3 | Accelerator pedal | `(b3 − 16) * 100 / 208` %, clamp 0–100 |
| `0x181` | 2, 4 | Throttle / engine load | raw, idle ≈ 52, overrun ≈ 47, > 40 % pedal ≈ 96 |
| `0x1F9` | 2–3 | Engine speed (copy) | same as `0x181` bytes 0–1 |
| `0x280` | 4–5 | Speed, cluster group (copy of `0x355`) | `raw * 0.01` km/h |
| `0x284` | 0, 2 | Wheel speed (2 wheels) | `b * 1.13` km/h |
| `0x284` | 4–5 | Speed, high-res (copy of `0x354`) | `raw * 0.01` km/h |
| `0x285` | 0 | Vehicle speed | `b0 * 1.131` km/h |
| `0x285` | 2 | Wheel speed | `b2 * 1.13` km/h |
| `0x2DE` | 6–7 | Fuel level sender | raw, non-linear, `0xFFFF` below range |
| `0x354` | 0–1 | Speed, high-res | `raw * 0.01` km/h |
| `0x354` | 6 · `0x10` | Brake pedal | set = pressed |
| `0x355` | 0–1 | Speed, cluster group | `raw * 0.01` km/h |
| `0x358` | 1 · `0x40` | Cabin blower | set = on |
| `0x35D` | 0 · `0x06` | Rear window heater | both bits set = on |
| `0x35D` | 2 · `0x40` | Wipers | set = running |
| `0x35D` | 2 · `0x80` | Trunk | set = open |
| `0x35D` | 4 · `0x10` | Brake pedal (copy) | set = pressed |
| `0x35D` | 4 · `0x40` | Vehicle moving | set = rolling |
| `0x551` | 0 | Coolant temperature | `b0 − 40` °C |
| `0x551` | 1 | Fuel injection counter | 8-bit counter, ≈ 0.117 ml per count |
| `0x551` | 5 · `0x02` | Engine running | set = running |
| `0x5C5` | 0 · `0x04` | Handbrake | set = engaged |
| `0x5C5` | 1–3 | Odometer | 24-bit, 1 count = 1 km |
| `0x5E4` | 0 · `0x04` | Oil pressure warning (copy) | set = lamp on |
| `0x60D` | 0 · `0x08` `0x10` `0x20` `0x40` | Doors FL, FR, RL, RR | set = open |
| `0x60D` | 1 · `0x20` / `0x40` | Turn signal left / right | both = hazards |
| `0x60D` | 2 · `0x04` | Rear fog light | set = on |
| `0x60D` | 2 · `0x10` | Central locking status | set = locked |
| `0x60D` | 6 · `0x10` | Reverse light | set = reverse engaged |
| `0x625` | 1 | Light switch | `0x00` off, `0x40` parking, `0x60` low, `0x50`/`0x10` high |
| `0x625` | 3 · `0x80` | Oil pressure warning | set = lamp on |

## Engine

**Engine speed – `0x181` bytes 0–1:** `(b0*256 + b1) / 8`. Not the OBD-II `/4` scaling; checked against the
tachometer (700 rpm idle, ~2500 rpm cruising). `0x1F9` bytes 2–3 carry the same value (1732 of 1735 samples
identical, the rest a few ms apart).

**Accelerator pedal – `0x181` byte 3:** drive-by-wire pedal position. `0x10` released, `0x19` slightly pressed,
`0x95` almost floored, `0xE0` floored, monotonic in between. Reads `0x10` both at idle and when coasting.

**Throttle / load – `0x181` bytes 2 and 4:** follow the pedal (r = 0.83 / 0.78) but differ between idle
(≈ 51/54) and coasting (≈ 48/46) with the pedal released – so throttle plate angle or calculated load rather
than a second pedal sensor. Byte 4 ≤ 48 with the pedal released and > 1300 rpm is the overrun state in which
the ECU cuts fuel (see the injection counter below).

**Coolant temperature – `0x551` byte 0:** `b0 − 40` °C. The factory manual (LAN-19 signal chart) lists
"engine coolant temperature signal" from the ECM to the cluster and BCM, and a drive showed 71 → 93 °C
(thermostat regulated). Broadcast even though this trim has no temperature gauge, only a warning lamp.

**Engine running – `0x551` byte 5, bit `0x02`:** the "engine status signal" of the LAN-19 chart. Bit `0x01`
of the same byte is set only while cranking.

## Speed and wheels

**Vehicle speed – `0x285` byte 0:** `b0 * 1.131` km/h, fitted against GPS across 7 independent logs
(R² = 0.97). Coarse (1.13 km/h steps) but always there.

**High-resolution speed – `0x354` bytes 0–1** (copy in `0x284` bytes 4–5): `raw * 0.01` km/h, 640+ distinct
values per drive, r = 0.9998 against `0x285`. Reads 2.73 % above `0x285 * 1.131`. Settles to 0 at standstill.

**Cluster speed – `0x355` bytes 0–1** (copy in `0x280` bytes 4–5): same encoding, 4.77 % above
`0x285 * 1.131`. A speedometer may read high but never low, so this is most likely the value the cluster
displays. Not yet verified against the needle.

**Wheel speeds – `0x285` bytes 0/2 and `0x284` bytes 0/2:** four channels on the `0x285` scale
(≈ 1.117–1.131 km/h per count). Bytes of the *same* frame differ by up to 4 counts, so they are separate wheels,
not a time offset. Which byte is which corner is not determined yet (needs a slow tight circle in one
direction – outer wheels run several km/h faster).

**Vehicle moving – `0x35D` byte 4, bit `0x40`:** sets between 5.7 and 15.8 km/h, clears between 0 and
3.4 km/h, never set at standstill in 1264 samples. A noise-free "car is rolling" flag (GPS reports 0.6–2.4 km/h
while parked).

**Counters:** `0x284`/`0x285` byte 6 advances by 50 per second while moving (20 ms time base), byte 7 looks like
a checksum.

## Fuel

**Fuel injection counter – `0x551` byte 1:** the manual's "fuel consumption monitor signal". 8-bit counter that
wraps roughly every 15 s at idle, so use differences. Its rate rises with rpm and load, and it **stops completely
during overrun fuel cut** (pedal released, in gear, above ~1300 rpm) – nothing else on the engine behaves like
that. 5977 counts per hour at idle; with ~0.7 L/h idle consumption that is ≈ **0.117 ml per count**, which gave
9.4 L/100 km for a drive where the fuel sender independently said 9.2. A cleaner value needs a full tank
measured against a receipt.

**Fuel level – `0x2DE` bytes 6–7:** raw sender value, 16-bit. The sender is **not linear** – calibrate with
several (raw, litres) points across the tank. On this car the reserve lamp came on at raw ≈ 699 (≈ 10 L of 46 L).
Below the sender's range the value becomes `0xFFFF`; near that point it flickers between the real value and
`0xFFFF` every second, so debounce it.

## Body (BCM)

`0x60D` is the BCM's collective message; all bits were confirmed by operating the switch several times with the
ignition on.

| Signal | Location | Notes |
|---|---|---|
| Doors | byte 0: `0x08` FL, `0x10` FR, `0x20` RL, `0x40` RR | |
| Turn signals | byte 1: `0x20` left, `0x40` right | hazards set both |
| Rear fog light | byte 2: `0x04` | |
| **Central locking status** | byte 2: `0x10` | `0x18` locked, `0x08` unlocked. Follows the interior button, the key and the remote, also with the ignition off (the BCM wakes the bus briefly). The locks have no feedback switches – the BCM remembers its last command and lights the button LED from it |
| Reverse light | byte 6: `0x10` | the reverse-light switch, i.e. reverse gear. `0x215` byte 1 (`0x30` ↔ `0x70`) shows the same event |

Other body signals:

| Signal | Location | Notes |
|---|---|---|
| Trunk | `0x35D` byte 2, `0x80` | `0x625` byte 0 also changes but is shared with other consumers and gave false positives |
| Wipers | `0x35D` byte 2, `0x40` | |
| Rear window heater | `0x35D` byte 0, `0x06` | `0x88` off ↔ `0x8E` on; bits 1 and 2 change together, match the pair |
| Brake pedal | `0x354` byte 6, `0x10` | copy in `0x35D` byte 4 `0x10` (1264 of 1264 samples agree) |
| Handbrake | `0x5C5` byte 0, `0x04` | |
| Odometer | `0x5C5` bytes 1–3 | `0x040C93` = 265363 km, exactly the dashboard reading; a capture 715 km earlier matched too |
| Cabin blower | `0x358` byte 1, `0x40` | |

## Lights and warnings

**Light switch – `0x625` byte 1:** `0x00` off, `0x40` parking lights, `0x60` low beam, `0x50` / `0x10` high beam
(probably held vs. flash-to-pass, not separated).

**Oil pressure warning – `0x625` byte 3, bit `0x80`:** set with ignition on and engine off, clears exactly one
second after the engine starts (when oil pressure builds), returns at shutdown. `0x5E4` byte 0 bit `0x04` shows
the identical pattern in all 795 log lines – the usual second copy. (An earlier version of this list called
`0x5E4` "engine running"; it is the oil lamp, which behaves the same way.)

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

## Candidates

Seen, but not yet confirmed by a second independent observation:

| ID | Byte · bits | What was seen |
|---|---|---|
| `0x500` | 0 · `0x04` | Set for about one second while cranking (`0x02` → `0x06` → `0x02`). Probably "starter engaged" |
| `0x511` | 1–6 | Six bytes change together when the ignition is switched on and stay put. Some ECU or immobilizer status |
| `0x215` | 1 · `0x40` | Same transition as the reverse light; likely a copy for another ECU |

## Not on the bus / not found

| Signal | Status |
|---|---|
| Fuel reserve lamp | **Not on CAN.** The fuel sender is wired directly to the cluster, which switches the lamp and chime itself (manual DI-8/34/83). Use the fuel level raw value instead |
| ABS active | Still searching. Brake events are now recorded at full frame rate (`0x284`/`0x285`/`0x354`) to find it on the next ABS stop |
| Battery / system voltage | No byte with the expected resting vs. charging shape |
| Outside temperature | Probably not present (no display on this trim) |
| Steering angle | Unlikely (no ESP on this trim) |
| Seatbelt switch | Sensor disconnected on this car |
| Airbag / SRS status | Not pursued |

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
