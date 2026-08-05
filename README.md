# Nissan Micra K12 – CAN IDs (Body CAN)

## Vehicle

| | |
|---|---|
| Model | Nissan Micra K12 |
| Trim | Visia (base trim) |
| Year | 2009 |
| Engine | 1.2L CR12DE, petrol |
| Transmission | manual, 5-speed |
| Bus | 500 kbit/s, 11-bit identifier |

## How this was measured

Waveshare ESP32 C6 Zero M with a CAN transceiver, two data lines (CAN-H/CAN-L) + ground wired
to the bus, traffic sniffed passively.

## Confirmed signals

| CAN ID | Frame (byte0…byte7) | Signal | Meaning |
|---|---|---|---|
| `0x285` | **xx** xx xx xx xx xx xx xx | Speed | `byte0 * 1.131` = km/h |
| `0x181` | **xx** **xx** xx xx xx xx xx xx | RPM | `(byte0*256 + byte1) / 8` = rpm |
| `0x551` | **6F** 28 00 80 xx xx xx xx | Coolant temperature | Byte 0, degrees Celsius = raw &minus; 40. This sample decodes to 111 &minus; 40 = 71 &deg;C. Confirmed two ways: the factory manual's CAN signal chart (LAN-19) lists "Engine coolant temperature signal" as transmitted by the ECM and received by the combination meter and BCM, and a logged drive showed byte 0 rising 111 -> 133, i.e. 71 -> 93 &deg;C, which matches a thermostat-controlled warm engine. Worth noting because it is easy to assume otherwise: this trim has no temperature gauge at all, only a red warning lamp, yet the value is still broadcast |
| `0x2DE` | 00 00 80 09 5F FF **0D** **1D** | Fuel level | 16-bit, big endian, bytes 6–7. (46 L tank, reserve ≈ 10 L) this worked out to roughly 79–80 raw units per liter — a starting point to verify against, not a fixed constant |
| `0x181` | 00 00 2A **10** 4D 20 00 00 | Accelerator pedal | Byte 3, drive-by-wire pedal position. Two-point calibration from measured samples at idle (`0x10` = 16) and full throttle (`0xE0` = 224): `(byte3-16)*100/208`, clamped 0–100 %. Same message as RPM above |
| `0x284` | **2A** FF **2A** 9B 13 18 81 20 | Wheel speed | Bytes 0 and 2, same scale as `0x285` byte 0 (`byte * 1.131` ≈ km/h). Tracks the known `0x285` speed to within ±1 raw unit across a full 10 km drive (Pearson r = 1.000 for both bytes against `0x285` byte 0) — almost certainly two individual wheel-speed channels from the ABS ECU, with `0x285` being the averaged/cluster value the instrument cluster actually displays. Which physical wheel each byte belongs to isn't determined yet (would need a one-wheel-off-ground test) |
| `0x354` | xx xx xx xx xx xx **xx** xx | Brake pedal | Bit 4 (`0x10`) of byte 6 set = brake pressed |
| `0x5C5` | **44** 04 09 E5 00 0C 00 7F | Handbrake | Bit 2 (`0x04`) of byte 0 set = engaged (this sample: not engaged) |
| `0x5C5` | 44 **04** **09** **E5** 00 0C 00 7F | Odometer | Bytes 1–3, 24-bit big endian, 1 count = 1 km, no offset or scaling. This sample decodes to `0x0409E5` = 264677 km. Verified two ways: a later capture read `0x040C93` = 265363 km against a dashboard reading of exactly 265363 km, and the 715 km between the two captures matches the distance actually driven in between. Same message as the handbrake above |
| `0x625` | xx **xx** xx xx xx xx xx xx | Light switch position | Byte 1: `0x00`=off, `0x40`=parking lights, `0x60`=low beam, `0x50`/`0x10`=high beam (two different raw values, likely high beam held vs. flash-to-pass — not conclusively distinguished) |
| `0x358` | xx **xx** xx xx xx xx xx xx | Cabin blower | Bit 6 (`0x40`) of byte 1 set = on |
| `0x358` | xx xx xx **xx** xx xx xx xx | Central locking status | Bits `0x14` of byte 3 set = locked |
| `0x60D` | **00** 14 08 00 00 70 00 00 | Doors (bitmask) | Byte 0: Bit 3=front left, Bit 4=front right, Bit 5=rear left, Bit 6=rear right (this sample: no bits set, all doors closed) |
| `0x60D` | 00 **14** 08 00 00 70 00 00 | Turn signals | Byte 1: Bit 5 (`0x20`)=left, Bit 6 (`0x40`)=right. Hazards set both at once (this sample: `0x14` = neither bit set, no turn signal active) |
| `0x60D` | 00 14 **08** 00 00 70 00 00 | Rear fog light | Bit 2 (`0x04`) of byte 2 set = on (this sample: `0x08` = bit 2 not set, off) |
| `0x60D` | 00 14 08 00 00 70 **00** 00 | Reverse light | Bit 4 (`0x10`) of byte 6 set = reverse light on (this sample: off). What was actually tested is the light coming on when shifting into reverse, not a transmission gear-position signal — confirmed over four independent engage/disengage cycles |
| `0x35D` | xx xx **xx** xx xx xx xx xx | Wipers | Bit 6 (`0x40`) of byte 2 set = running |
| `0x35D` | xx xx **xx** xx xx xx xx xx | Trunk | Bit 7 (`0x80`) of byte 2 set = open. Byte 0 of `0x625` also changes when the trunk is opened and was used for this at first, but that byte turns out to carry several consumers at once (see rear window heater below) and produced false positives — the indicator lit up whenever the wipers ran. Bit 7 here is a clean single bit, right next to the wiper bit in a byte that is otherwise understood |
| `0x35D` | **xx** xx xx xx xx xx xx xx | Rear window heater | Byte 0, measured `0x88` (off) ↔ `0x8E` (on) over four on/off cycles. Bits 1 and 2 change together (`0x88 ^ 0x8E = 0x06`); which of the two is the heater itself cannot be separated from two states alone, so the pattern is matched as a whole rather than guessing a single bit. `0x625` byte 0 shows `0x02` ↔ `0x03` for the same action — the usual second message, but on the byte that is already known to be shared |

`0x215` byte 1 (`0x30`↔`0x70`, bit 6) showed the identical transition in the same reverse-light tests — likely a redundant copy of the same signal for a different ECU. Not decoded separately; either byte works.

`0x181` bytes 2 and 4 (same message as RPM and the accelerator pedal above) correlate with the pedal position from a full drive's data — r=0.83 and r=0.78 respectively against the byte 3 formula above — but don't fit a single clean linear formula the way byte 3 does. Likely a second pedal sensor (APPS2, drive-by-wire pedals normally carry two for redundancy) and/or throttle body angle rather than the same signal restated. Not calibrated yet; needs its own two-point measurement across more drives before it goes in the confirmed table.

## Not yet found / still being searched for

| Signal | Status | Notes |
|---|---|---|
| Battery / system voltage | Searching | Expect two regimes: resting (~12.4V) vs. charging (~14V) once running |
| ABS active | Searching | Correlated with brake pedal + hard deceleration, no confirmed byte yet |
| Oil pressure warning lamp | Not started | Only lights briefly during cranking before oil pressure builds |
| Steering angle | Unlikely to exist | This trim has no ESP, which is the main consumer of this sensor |
| Outside temperature | Unlikely to exist | Base trim has no climate display to show it on |
| Seatbelt switch | Unfindable currently | Sensor physically disconnected on this vehicle, so it no longer toggles |
| Airbag / SRS status | Not pursued | Even if present, a passive status flag isn't useful for a dashboard |

---

## The project behind this data

These CAN IDs aren't just a reverse-engineering exercise — they power a full custom dashboard and diagnostics system built for this car, called **Micra-Core**.

**Hardware:**
- Waveshare ESP32-C6 Zero (RISC-V), wired into the OBD-II port via a CAN transceiver, plus GPIO relays for the door locks, windows, and remote engine start
- W5500 Ethernet module for a wired connection alongside its own Wi-Fi access point
- A GPS module for position and trip tracking
- A Raspberry Pi 5 handling the camera/video side

**What it does:**
- A live web-based speedometer dashboard (switchable skins), reachable over Wi-Fi, Ethernet, or the car's own hotspot
- An admin panel for diagnostics, calibration, and live CAN traffic inspection
- Reads and clears real OBD-II fault codes (DTCs), with automatic background checks while parked
- Learns fuel consumption over time and estimates remaining range
- A two-axis driving-style score (smoothness vs. harshness) measured against your own average, not a fixed threshold
- Gear detection calculated from speed/RPM ratios, with reverse overridden by the genuine reverse-light signal instead (ratios can't reliably tell reverse apart at low speed)
- Push-button engine start and phone-based auto lock/unlock (locks itself when your phone's Wi-Fi leaves range)
- A family of automatic signal-discovery tools ("Finders") that passively watch the bus for monotonic counters, warm-up curves, and event-correlated bytes — how most of the signals in this table were actually found
- **CarGuard** on the Raspberry Pi: dashcam while driving, motion-triggered sentry mode while parked and locked, a live camera feed, an automatic "black box" flight recorder that saves footage and raw CAN data around harsh-driving events, GPS-mapped trip replay, and synthesized engine sound

Built entirely through passive observation and manual testing on my own car — no dealer tools, no proprietary diagnostic software, just a transceiver, patience, and physically triggering every switch and pedal to see what shows up on the bus.
