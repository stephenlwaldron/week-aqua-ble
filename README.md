# Week Aqua BLE: local control for Week Aqua aquarium lights

Control a **Week Aqua** Bluetooth aquarium light from **Home Assistant** (and Apple Home, Google, …)
with a ~$5 ESP32 running ESPHome. No phone app, no cloud.

The Bluetooth protocol was reverse-engineered from scratch. As far as we know, this is the first
public write-up of it. See **[PROTOCOL.md](PROTOCOL.md)**.

| Model | Family | Status |
|---|---|---|
| P1200 Pro (`B3.0-P1200Pro-1210`) | WEK | ✅ Verified: brightness (R/G/B/UV), fan, clock, schedule |
| Other Week Aqua lights | various | ❓ Untested. Probably close; see "Other models" below |

## What you get

- **An RGBW light entity**: red, green and blue as color, the **UV channel as white**, plus brightness.
  It shows up in Apple Home as a normal color light through the HomeKit Bridge.
- **A "Green Mode" button** that resets the light to the app's default spectrum while keeping
  brightness. Handy because HomeKit remembers the last color you picked.
- **An automatic fan**: 100% while the light is on, off when it's off.
- **No green "connected" LED.** The ESP connects only to send a change, then disconnects. The official
  app still works whenever the ESP is idle.

## Setup

1. Find your light's **MAC address**: scan with nRF Connect on your phone, or look in an ESPHome
   BLE tracker log. P1200 units advertise service `FFF0`.
2. Copy [`esphome/week-aqua-light.yaml`](esphome/week-aqua-light.yaml) into your ESPHome config. Set `light_mac`, and
   make sure your `secrets.yaml` has `wifi_ssid`, `wifi_password`, `ap_password`, `api_key` and `ota_password`.
3. Flash any ESP32 (classic ESP32, BLE required) and place it within a few meters of the light.
4. Adopt it in Home Assistant. To use it in Apple Home, expose `light.*` (and `button.*` if you want
   Green Mode) through the HomeKit Bridge integration.
5. **Schedule it in Home Assistant or Apple Home**, not in the Week Aqua app (see the gotcha below).

## ⚠️ The gotcha that will waste your evening

These lights run a **built-in schedule**, and outside its time window they **silently ignore
brightness commands**. The config sets the built-in window to 00:00–23:59 when it first connects, so
live control works at any hour. That also means the light's own schedule is off. Do your on/off
timing in your home automation instead. To go back to the built-in schedule, set it again in the app.

## Other models

Different Week Aqua families may use different UUIDs or opcodes. The config has knobs for the known
differences (`service_uuid`, `write_uuid`, `live_opcode`), described in [PROTOCOL.md](PROTOCOL.md).

To find out for certain what *your* light expects, use the **fake light**:
[`esphome/week-aqua-fake.yaml`](esphome/week-aqua-fake.yaml) makes an ESP32 impersonate your light. Connect
the app to it, move the sliders, and the ESP logs every command the app sends. Then send the same frames
from the main config (it has a "Hex Command" debug entity for experiments).

**Please open an issue or PR with your model and findings.** Every model someone adds helps the next owner.

## Disclaimer

An independent, unofficial project with no affiliation to Week Aqua. Only commands the official
app itself sends are used, but this comes with no warranty: use it at your own risk.

## License

MIT
