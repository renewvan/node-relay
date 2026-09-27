# node-relay

ESPHome YAML config for the 8-channel switched-load relay node (lights,
fan, USB outlets, etc.) — no custom firmware. Per
`hub/.scratch/renewvan-hub-v0-build/issues/03-esp32-relay-node-config.md`
and the interface decision in
`hub/.scratch/renewvan-hub-v0/issues/03-esp32-relay-node-driver-interface.md`.

## Hardware

Bare ESP32 dev board + a separate 8-channel relay board wired to ESP32
GPIO (not an integrated relay+MCU board — keeps switching decoupled
from the hub's own lifecycle). Each channel also has a physical
momentary/toggle button wired to its own GPIO input (pulled up
internally, switched to GND when pressed).

| Channel | Default id | Relay GPIO | Button GPIO |
|---|---|---|---|
| 1 | `lights_ceiling` | GPIO4 | GPIO14 |
| 2 | `lights_reading` | GPIO5 | GPIO15 |
| 3 | `lights_kitchen` | GPIO13 | GPIO21 |
| 4 | `lights_bathroom` | GPIO16 | GPIO22 |
| 5 | `lights_awning` | GPIO17 | GPIO25 |
| 6 | `fan_vent` | GPIO18 | GPIO26 |
| 7 | `usb_outlets` | GPIO19 | GPIO27 |
| 8 | `aux_1` | GPIO23 | GPIO32 |

## Switching path — fully local, no MQTT command topic

Each channel pairs a `binary_sensor` (button) with a `switch` (relay
output): `on_press` on the button directly calls `switch.toggle` on the
paired relay, entirely on-device. This automation runs regardless of
WiFi/MQTT connectivity — physical buttons keep working even with the
Pi, dashboard, or broker down. There is no MQTT command topic for
relays in v0 (read-only dashboard, no tap-to-toggle); nothing subscribes
to control a relay remotely.

## Renewvan bus MQTT interface

Every relay is `internal: true` (no Home Assistant/ESPHome-API
frontend, no discovery — `mqtt: { discovery: false }` too). Its only
MQTT presence is an explicit `mqtt.publish` in `on_turn_on`/
`on_turn_off`, which publishes retained `"true"`/`"false"` directly to
`renewvan/relay/<id>/state` on every state change — matching the v0.1
`relay.schema.json` boolean `state` field.

(ESPHome's built-in `payload_on`/`payload_off` switch options — as
described in the ticket 03 decision doc — turned out not to exist on
ESPHome's actual `MQTT Component` base config; only `state_topic`/
`command_topic` do, and those still emit ESPHome's native `"ON"`/`"OFF"`
strings, not `"true"`/`"false"`. Explicit `mqtt.publish` actions produce
the exact same wire behavior the ticket specifies — retained `"true"`/
`"false"` on `renewvan/relay/<id>/state` on every change — via a documented
ESPHome mechanism instead of a nonexistent config key.)

## Customizing channels

The 8 channel labels above (`lights_ceiling`, etc.) are placeholders —
rename them to match your actual van's loads. For each channel, update
all three of: the `switch.id`, its `mqtt.publish` topic string (both
`on_turn_on` and `on_turn_off`), and the `switch.toggle:` target in the
paired button's `on_press`. GPIO pin assignments are physical wiring —
change only if your board is wired differently.

## Flashing

```bash
pip install esphome
cp secrets.yaml.example secrets.yaml   # fill in your WiFi/MQTT/OTA values
esphome run relay-node.yaml            # first flash: connect via USB
```

Subsequent updates can go over WiFi (`esphome run relay-node.yaml`
again, once the node has joined your network) using the OTA password
in `secrets.yaml`.

## Verifying

1. With the ESP32 flashed and powered, physically disconnect it from
   WiFi/the renewvan bus (or just don't configure `wifi_ssid` yet) — confirm
   each physical button still toggles its relay.
2. Reconnect to the renewvan bus, press a button, and confirm
   `renewvan/relay/<id>/state` publishes `"true"`/`"false"` (retained) —
   e.g. `mosquitto_sub -h <broker> -t 'renewvan/relay/#' -v`.

## CI

`.github/workflows/validate.yml` runs `esphome config`/compile
validation (via `esphome/build-action`) against a placeholder
`secrets.yaml` on every push/PR — catches YAML/pin/config errors
without needing real hardware. There's no versioned image to publish
for this repo (it's flashed once per node, not a compose service) —
`hub`'s deployment manifest references this repo's own flash
instructions, not a pinned image.
