# Week Aqua BLE protocol notes

These are notes on how the Week Aqua app (`com.weekled.weekaquas`) talks to the lights.
I confirmed everything in the "WEK family" section on my P1200 Pro (`B3.0-P1200Pro-1210`).
The rest comes from reading the app's code and is untested.

How I got there: I decompiled the Android app to find the command builders, then flashed an ESP32
to pretend to be the light (`esphome/week-aqua-fake.yaml`). I connected the real app to it and
logged everything it wrote, then replayed those bytes to the real light from another ESP32.

## Connection

- Service `FFF0`. Write commands to `FFF2` (write without response). `FFF1` is read/notify but
  the P1200 never sent anything useful on it, just 128 zero bytes on read.
- Older units apparently use `FFE0` / `FFE1` instead, and the app also accepts `FF60` when scanning.
- No pairing, no auth, no encryption.
- Every command is 8 bytes. Unused bytes are padded with `0x55`.

The app works out which family a light belongs to from 2 bytes in the advertisement's manufacturer
data. On my light the manufacturer data is `<0x4657> 0001` + the light's MAC, so the bytes on air
are `57 46` = "WF", which is the WEK family.

| Code | ASCII | Family in the app |
|---|---|---|
| 5745 | WE | HJ (the app also falls back to this for unknown codes) |
| 5746 | WF | WEK. This is the P1200 |
| 5747 | WG | FOUR |
| 5748 | WH | FIVE (12-byte commands) |
| 5749 | WI | SIX |
| 5750 | WP | SEVEN (power strip) |
| 5751 | WQ | EIGHT |
| 5752 | WR | NINE |

## WEK family (tested)

Levels are percent × 2.35, rounded, so 100% = `EB`. The app keeps values under `F0`, so they never
clash with an opcode.

```
FF hh mm ss 55 55 55 55   set clock. Time digits as BCD: 14:05:09 -> FF 14 05 09 ...
F9 55 55 55 55 55 55 55   app sends this right after the clock
FD F1 55 55 55 55 55 55   schedule mode (app tab 0). FD F2 is tab 1, haven't tried it
FE F9 h1 m1 h2 m2 00 00   schedule window from h1:m1 to h2:m2 (BCD)
FB F9 RR GG BB UV 55 55   channel levels: red, green, blue, UV
FC xx 55 55 55 55 55 55   fan. FC EB = 100%, FC 00 = off
```

When you open a light in the app it sends the clock, `F9`, `FD F1`, then `FB F9 ...`.
Editing the schedule sends `FE F9 ...` followed by `FD F1`.

The default spectrum at 100% ("green mode" on mine) is `FB F9 6C E3 73 EB 55 55`.

### Brightness is ignored outside the schedule

The light runs its own schedule off its own clock. Outside the `FE F9` window it stays dark and
ignores `FB F9`, with no error. To control it whenever you want, send:

```
FF <current time>  ->  F9  ->  FD F1  ->  FE F9 00 00 23 59 00 00  ->  FD F1
```

`FB F9` works straight away after that.

Settings are kept in the light's memory, so they survive a power cut. You can get the original
schedule back by setting it in the app.

### Other things I noticed

- The light shows a green LED while something is connected. It keeps its state after you
  disconnect, so you only need to connect to send changes.
- Only one connection at a time. If the phone app is connected, nothing else can connect.

## Other families (from the app code, not tested)

- HJ: live levels are `FB EF RR GG BB UV 55 55`
- Three/Five/Nine: `FB FA` / `FB FB` / `FB FC` followed by the channel data, scaled ×2.40. Five pads its commands to 12 bytes.
- `FB Fn ...` looks like it saves spectrum preset n. `FE EF` and `FE FD` are more time/schedule commands.
- Some families use an inverted scale for their "mode" commands: (100 − pct) × 2.46.

If you test any of these, please open an issue.
