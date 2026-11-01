# week-aqua-ble

ESPHome config for controlling Week Aqua Bluetooth aquarium lights from Home Assistant, plus notes
on the BLE protocol.

I have a P1200 Pro and got tired of opening the app every time I wanted to change it. There was no
integration and I couldn't find the protocol written up anywhere, so I worked it out (decompiled
the app, then sniffed what it actually sends). An ESP32 next to the tank now runs it, and it shows
up in Apple Home through HA's HomeKit bridge like any other light.

Protocol details are in [PROTOCOL.md](PROTOCOL.md).

## What works

Tested on my P1200 Pro (shows up as `B3.0-P1200Pro-1210`): brightness for all four channels
(R/G/B/UV), fan, clock and schedule.

The P600 Pro and P900 Pro are the same product line, so they'll probably work as-is. If you have
one, let me know either way. Other Week Aqua models I haven't been able to test, see further down.

In Home Assistant you get:

- an RGBW light. R/G/B are the color, and the UV channel is mapped to "white"
- a Green Mode button that puts the app's default spectrum back without changing brightness
  (HomeKit remembers whatever color you last picked, so it's handy)
- the fan runs at 100% whenever the light is on and stops when it's off. I didn't want to control
  it separately

The ESP only connects to the light when it has something to send, and drops the connection about
15 seconds later. While it's connected the light shows a green LED, which bugged me, and with it
disconnected the official app still works too.

## Setup

1. Find your light's MAC address. nRF Connect on your phone works, and so does the BLE tracker log in ESPHome.
2. Copy `esphome/week-aqua-light.yaml` into your ESPHome setup and set `light_mac`. It uses the usual
   secrets: `wifi_ssid`, `wifi_password`, `ap_password`, `api_key` and `ota_password`.
3. Flash any regular ESP32 and put it within a few meters of the light.
4. Adopt it in HA. For Apple Home, expose the light (and the button if you want it) through the HomeKit bridge.
5. Set your on/off times in HA or Apple Home, not in the Week Aqua app.

## Schedule

The light won't turn on outside its built-in schedule, so the config sets that schedule to all day.
Do your on/off times in HA or HomeKit instead.

## Is it safe to try?

Yes.

- Everything it sends is something the official app sends. I captured these from the app talking
  to the light, I didn't guess them.
- The app has no firmware update code at all, so there's nothing in the light that could get
  "updated" by accident.
- It only changes the levels, the fan and the schedule window, and you can set all of those back
  in the app. If anything ever seems off, set the light up again in the app and that overwrites it.
- Settings are saved in the light, so unplugging it doesn't reset anything (it doesn't need to).
- The light only accepts one connection at a time. If the ESP can't connect, close the app on your phone.

## Other models

Different Week Aqua families seem to differ in small ways: older UUIDs (`FFE0`/`FFE1` instead of
`FFF0`/`FFF2`), or a different brightness opcode. The config has substitutions for those, and
PROTOCOL.md has what I found in the app for the other families. Note that none of that is tested.

The best way to find out what your light wants is `esphome/week-aqua-fake.yaml`. It makes an ESP32
pretend to be your light, then you connect the app to it, move some sliders, and it logs every
command. That's how I figured mine out. It never talks to the real light, so there's no risk.

If you get another model working (or not working), please open an issue. It'd be great to grow the
list.

---

Not affiliated with Week Aqua. MIT license.
