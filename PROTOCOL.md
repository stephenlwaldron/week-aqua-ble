# Week Aqua BLE protocol

Reverse-engineered notes for Week Aqua aquarium lights that use the **Week Aqua** phone app
(`com.weekled.weekaquas`). Verified end-to-end on a **P1200 Pro** (advertises as `B3.0-P1200Pro-1210`).
The rest of the families are described from the app's code but **untested on real hardware**.
If you have one, please open an issue or PR with what you find.

## How this was worked out

1. **Static analysis.** I decompiled the Android app and read its BLE command builders to learn the
   opcodes and the encoding.
2. **Dynamic analysis.** I flashed an ESP32 to impersonate the light (same name, manufacturer data
   and GATT layout, see [`esphome/week-aqua-fake.yaml`](esphome/week-aqua-fake.yaml)). Then I connected
   the real app to it and logged every byte it wrote. This is how the schedule gotcha below was found.
3. **Replay.** I sent the captured frames from an ESP32 `ble_client` to the real light.

## Transport

| Item | Value |
|---|---|
| Advertised service | `FFF0` (older units: `FFE0`; the app also accepts `FF60`) |
| Write characteristic | `FFF2`, Write Without Response (older units: `FFE1`) |
| Notify characteristic | `FFF1` (read/notify; the P1200 returned 128 zero bytes and never sent anything useful) |
| Frame | 8 raw bytes, unused bytes padded with `0x55` |
| Auth / pairing / encryption | **None** |

The app finds lights by service UUID. It then reads a 2-byte **family code** from the advertisement's
manufacturer data. The P1200's manufacturer data is company ID `0x4657` (on-air bytes `57 46` = ASCII "WF"),
followed by `00 01` and the light's own MAC address.

| Code | ASCII | App family |
|---|---|---|
| `5745` | WE | HJ (also the fallback for unknown codes) |
| `5746` | WF | **WEK**: the P1200 (verified) |
| `5747` | WG | FOUR |
| `5748` | WH | FIVE (uses 12-byte frames) |
| `5749` | WI | SIX |
| `5750` | WP | SEVEN (power strip) |
| `5751` | WQ | EIGHT |
| `5752` | WR | NINE |

## WEK family commands (P1200, verified)

Percentages are encoded as `round(pct / 100 * 235)`, so 100% = `0xEB`. Values stay below `0xF0`,
so they can never be mistaken for an opcode.

| Frame | Meaning |
|---|---|
| `FF HH MM SS 55 55 55 55` | Clock sync. Each value is **BCD** (decimal digits written as hex): 14:05:09 → `FF 14 05 09 …` |
| `F9 55 55 55 55 55 55 55` | Sent by the app right after the clock sync |
| `FD F1 55 55 55 55 55 55` | Select schedule mode (app tab 0). `FD F2` = tab 1 (untested) |
| `FE F9 h1 m1 h2 m2 00 00` | Schedule window from h1:m1 to h2:m2 (BCD), type 0, ramp 0 |
| `FB F9 RR GG BB UV 55 55` | Channel brightness: red, green, blue, UV |
| `FC FF 55 55 55 55 55 55` | Fan speed, where `FF` = pct × 2.35 (`FC EB` = 100%, `FC 00` = off) |

When the app opens it sends: clock → `F9` → `FD F1` → `FB F9 …`. After a schedule edit it sends `FE F9 …` and then `FD F1`.

The app's default spectrum at 100% ("green mode" on the P1200) is `FB F9 6C E3 73 EB 55 55`.

### ⚠️ The gotcha: brightness only shows inside the schedule window

The light runs its **own schedule** from its own clock. Outside the `FE F9` window it stays dark and
silently ignores `FB F9` brightness frames, which makes it look like your commands aren't working.
To control it live at any hour, send the clock, then `FD F1`, then an all-day window
`FE F9 00 00 23 59 00 00`, then `FD F1` again. After that, `FB F9` takes effect immediately.
Do your scheduling elsewhere (Home Assistant, Apple Home, …).

Settings are stored in the light's flash and survive a power cycle. To get the light's own schedule
back, set it again in the app.

### Other notes

- The green LED on the light is on **while a BLE client is connected**. The light keeps its state when
  you disconnect, so you can connect only when you need to send something. The ESPHome config here does that.
- The light accepts one connection at a time. If the phone app is connected, nothing else can connect.

## Other families (from the app code, untested)

- **HJ** live brightness: `FB EF RR GG BB UV 55 55`.
- **Three/Five/Nine** families: `FB FA` / `FB FB` / `FB FC` followed by channel data, scaled ×2.40.
  The FIVE family pads its frames to 12 bytes.
- `FB Fn …` writes spectrum preset *n*. `FE EF` / `FE FD` are further time/schedule frames.
- Some families use an inverted "mode" encoding: `(100 − pct) × 2.46`.
