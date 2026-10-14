# Nissan Micra K12 – CAN IDs (Body CAN)

Decoded CAN signals of a 2009 Nissan Micra K12 (1.2 L CR12DE, 5-speed manual, ABS, no A/C, no Intelligent Key).
Bus: **500 kbit/s, 11-bit IDs, OBD-II pins 6/14**. Every signal was checked against the real switch, pedal,
dashboard or fuel receipt.

Bytes are numbered 0–7, bit masks in hex (`0x10` = bit 4), multi-byte values big endian.

## Signals

| ID | ECU | Byte · bits | Signal | Decode |
|---|---|---|---|---|
| `0x181` | ECM | 0–1 | Engine speed | `(b0*256 + b1) / 8` rpm |
| `0x181` | ECM | 2 | Throttle plate | ≈ 42 closed, ≈ 55 idle, ≈ 100 wide open |
| `0x181` | ECM | 3 | Accelerator pedal | `(b3 − 16) * 100 / 208` %, `0x10` released, `0xE0` floored |
| `0x181` | ECM | 4 | Engine load | idle ≈ 52, overrun ≤ 46, full load ≈ 96–100 |
| `0x1F9` | ECM | 0 · `0x20` | Engine running | set = running |
| `0x1F9` | ECM | 0 · `0x40` | Radiator fan request | set = fan on (from ≈ 95 °C) |
| `0x1F9` | ECM | 2–3 | Engine speed (copy) | as `0x181` bytes 0–1 |
| `0x215` | IPDM | 1 · `0x03` | Starter engaged | ≈ 1 s while cranking |
| `0x215` | IPDM | 1 · `0x40` | Reverse switch | set = reverse |
| `0x284` | ABS | 0–1 / 2–3 | Wheel speed front right / front left | `raw16 * 1.13 / 256` km/h |
| `0x285` | ABS | 0–1 / 2–3 | Wheel speed rear right / rear left | `raw16 * 1.13 / 256` km/h |
| `0x285` | ABS | 4 | Vehicle speed, whole km/h | |
| `0x2DE` | Meter | 6–7 | Fuel level sender | raw, non-linear, see [Fuel](#fuel) |
| `0x300` | EPS | 0 | Steering assist level | 0 straight, up to ≈ 62 steering at standstill |
| `0x354` | ABS | 0–1 | Vehicle speed, high-res | `raw * 0.01` km/h (copy in `0x284` bytes 4–5) |
| `0x354` | ABS | 2–3 | Distance counter | 0.108 m per count (9240 per km), wraps at 65536 |
| `0x354` | ABS | 4 · `0x0A` | ABS warning lamp | ≈ 1 s at ignition on and while cranking |
| `0x354` | ABS | 5 · `0x80` | ABS control active | set during ABS braking |
| `0x354` | ABS | 6 · `0x10` | Brake pedal | set = pressed (copy in `0x35D` byte 4 `0x10`) |
| `0x354` | ABS | 6 · `0x40` | ABS control active (2nd flag) | set together with byte 5 `0x80` during ABS braking |
| `0x355` | Meter | 0–1 | Speed, cluster | `raw * 0.01` km/h, ≈ 4.8 % above `0x354` |
| `0x358` | BCM | 1 · `0x40` | Heater blower | set = on |
| `0x358` | BCM | 3 | Lock command (report only) | `0x14` lock, `0x0A` unlock, ≈ 1 s pulse |
| `0x35D` | BCM | 0 · `0x06` | Rear window heater switch | both set = on |
| `0x35D` | BCM | 0 · `0x40` | Ignition switch in START | ≈ 1 s while cranking |
| `0x35D` | BCM | 1 | Sleep / wake-up | `0x03` awake, `0x00` = bus sleeps 3 s later |
| `0x35D` | BCM | 2 · `0x40` / `0x80` | Front wiper / washer | set = on |
| `0x35D` | BCM | 4 · `0x40` | Vehicle moving | set from ≈ 6–16 km/h, clears below ≈ 3 km/h |
| `0x551` | ECM | 0 | Coolant temperature | `b0 − 40` °C (copy in `0x60D` byte 5) |
| `0x551` | ECM | 1 | Fuel injection counter | 8-bit counter, **0.1037 ml per count** |
| `0x551` | ECM | 3 · `0x20` | MIL (check engine) | on with ignition until the engine runs |
| `0x551` | ECM | 5 · `0x03` | Engine status | `0` stop, `1` stall, `2` run, `3` crank |
| `0x5C5` | Meter | 0 · `0x04` | Handbrake | set = engaged |
| `0x5C5` | Meter | 0 · `0x20` | Fuel below sender range | while `0x2DE` reads `0xFFFF` |
| `0x5C5` | Meter | 1–3 | Odometer | 24-bit, km |
| `0x5E4` | EPS | 0 · `0x04` | EPS warning lamp | on with ignition until ≈ 1–3 s after start |
| `0x60D` | BCM | 0 · `0x06` | Light switch | `0x04` parking, `0x06` low beam |
| `0x60D` | BCM | 0 · `0x08` `0x10` `0x20` `0x40` | Doors FL, FR, RL, RR | set = open |
| `0x60D` | BCM | 0 · `0x80` | Tailgate | set = open |
| `0x60D` | BCM | 1 · `0x20` / `0x40` | Turn signal left / right | both = hazards |
| `0x60D` | BCM | 2 · `0x04` | Rear fog light | set = on |
| `0x60D` | BCM | 2 · `0x10` | Central locking status | set = locked (copy: `0x358` byte 5 `0x03`) |
| `0x60D` | BCM | 6 · `0x10` | Reverse light | set = reverse |
| `0x625` | IPDM | 0 · `0x01` | Rear window heater relay | set = heating |
| `0x625` | IPDM | 0 · `0x04` | Wiper out of park position | ≈ 1 s per wipe (use for interval wiping) |
| `0x625` | IPDM | 0 · `0x30` | Starter relay | ≈ 1 s while cranking |
| `0x625` | IPDM | 1 | Headlamp status | `0x00` off, `0x40` parking, `0x60` low, `0x50` / `0x10` high |
| `0x625` | IPDM | 3 · `0x80` | Oil pressure warning | set = lamp on |

**Ignition on:** `0x35D` byte 0 `0x80` (also `0x60D` byte 1 `0x04`, `0x625` byte 3 `0x10`, `0x5C5` byte 0 `0x40`).

## Fuel

- **Consumption:** take differences of `0x551` byte 1 (8-bit, wraps) × 0.1037 ml. The counter stops completely
  during overrun fuel cut. Warm idle ≈ 0.75–0.8 L/h.
- **Fuel level `0x2DE` bytes 6–7:** non-linear, calibrate with several (raw, litres) points. Reserve lamp at
  raw ≈ 699 (≈ 10 L), full tank (46 L) ≈ 5760–5790. Updated every 30 s while driving, moves in steps,
  `0xFFFF` below range. The reserve lamp itself is not on CAN.

## Wheels and speed

- `0x284` = front axle, `0x285` = rear axle, right wheel first. Use the rear (non-driven) wheels for speed,
  steering and gear detection.
- `0x284`/`0x285` byte 6 = message counter, byte 7 = checksum `(b0 + … + b6 + (ID >> 8) + (ID & 0xFF)) & 0xFF`.

## Constants

| Gear | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|
| rpm per km/h | 132.7 | 73.3 | 51.3 | 36.7 | 29.2 |

Overrun fuel cut ends below ≈ 1300 rpm.

## Good to know

- **Locking over CAN is not possible.** `0x358` byte 3 only reports what the BCM does; lock it via the
  hard-wired button input.
- **Bus sleep:** ≈ 60 s after locking `0x35D` byte 1 goes `0x00`, 3 s later the bus is silent. `0x682` (1 byte,
  `00`) is the wake-up call. BCM, IPDM and meter wake with the bus; ECM, ABS and EPS only with the ignition.
- **OBD-II:** requests on `0x7DF`, engine answers on `0x7E8`. Two testers sending on `0x7DF` at once collide
  (bus-off) – wait for silence from the other one.
- **Not on the bus:** battery voltage, outside temperature, steering angle, fuel reserve lamp.
