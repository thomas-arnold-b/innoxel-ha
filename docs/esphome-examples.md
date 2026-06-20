# ESPHome Examples

Standalone, copy-pasteable ESPHome snippets for an Innoxel/Spline CAN bus. Replace the
CAN IDs, channels and pins with your own. See [specification.md](specification.md) for the
protocol details.

The control examples follow the approach first published by Vincèn PUJOL (Domedia).

---

## Bus setup + raw sniffer

Logs every frame as an ASCII string — use it to discover addresses and payloads.

```yaml
canbus:
  - platform: esp32_can
    tx_pin: GPIO3
    rx_pin: GPIO33
    can_id: 4              # this node's address; arbitrary for listening
    bit_rate: 100kbps
    use_extended_id: false
    on_frame:
      - can_id: 0x000
        can_id_mask: 0x000  # match everything
        then:
          - lambda: |-
              std::string s(x.begin(), x.end());
              ESP_LOGD("can", "id=0x%03X len=%d data=%s", can_id, (int)x.size(), s.c_str());
```

> Keep the logger at `INFO`/`WARN` on a busy bus; `DEBUG`-level CAN logging can stall the
> main loop.

---

## Button / motion input → `binary_sensor`

```yaml
binary_sensor:
  - platform: template
    name: "Hall button 3"
    id: hall_button_3

canbus:
  - platform: esp32_can
    # ...bus config as above...
    on_frame:
      - can_id: 0x450          # the C-series module address
        then:
          - lambda: |-
              std::string m(x.begin(), x.end());
              if (m == "T3")      id(hall_button_3).publish_state(true);
              else if (m == "t3") id(hall_button_3).publish_state(false);
```

---

## Relay → `switch`

```yaml
switch:
  - platform: template
    name: "Pool pump"
    optimistic: true
    turn_on_action:
      - canbus.send:
          can_id: 0x21D       # A-series module
          data: "S6"          # channel 6 ON
    turn_off_action:
      - canbus.send:
          can_id: 0x21D
          data: "C6"          # channel 6 OFF
```

---

## Shade → `cover` (time-based)

Measure your real open/close durations so HA can offer intermediate positions.

```yaml
cover:
  - platform: time_based
    name: "Living room shade"
    device_class: shutter
    has_built_in_endstop: true
    open_action:
      - canbus.send: { can_id: 0x211, data: "U0" }
    open_duration: 39s
    close_action:
      - canbus.send: { can_id: 0x211, data: "D0" }
    close_duration: 36s
    stop_action:
      - canbus.send: { can_id: 0x211, data: "H0" }
```

---

## Dimmer → `light` (ramp in software)

Send `time = 000` and let HA do the fade.

```yaml
light:
  - platform: monochromatic
    name: "Kitchen downlights"
    id: kitchen_light
    output: kitchen_output
    default_transition_length: 3s

output:
  - platform: template
    id: kitchen_output
    type: float
    write_action:
      - if:
          condition: { light.is_off: kitchen_light }
          then:
            - canbus.send:
                can_id: 0x67C        # B-series module
                data: "W2000000"     # channel 2 off
          else:
            - canbus.send:
                can_id: 0x67C
                data: !lambda |-
                  int b = (int)(id(kitchen_light).current_values.get_brightness() * 100);
                  if (b > 100) b = 100;
                  char buf[16];
                  sprintf(buf, "W2%03d000", b);   // channel 2, brightness, time=000
                  std::string s = buf;
                  return std::vector<uint8_t>(s.begin(), s.end());
```

---

## Reading state back (polling)

State and temperature are **not broadcast** — you must poll (see
[specification.md §4–7](specification.md#4-poll-commands-read)). Sketch:

1. Send a poll on the module's **poll address** (e.g. `PF` to `0x20B`, or `PT` to `0x409`).
2. Listen on the corresponding **reply address** (`0x20A`, `0x408`).
3. Decode the reply per §6 (state) or §7 (temperature).

```yaml
# Send a status poll to a relay module (reply will arrive on poll_id - 1)
button:
  - platform: template
    name: "Poll 0x20B status"
    on_press:
      - canbus.send: { can_id: 0x20B, data: "PF" }

# Decode the PF reply (relay/switch module) into a per-channel bitmask
canbus:
  - platform: esp32_can
    # ...bus config...
    on_frame:
      - can_id: 0x20A          # reply address = 0x20B - 1
        then:
          - lambda: |-
              if (x.size() >= 5 && x[0] == 0xBB && x[1] == 0x00) {
                uint8_t mask = x[4];
                for (int c = 0; c < 8; c++)
                  ESP_LOGI("pf", "ch%d = %s", c, (mask >> c) & 1 ? "ON" : "off");
              }
```

```yaml
# Temperature: poll PT, decode the AA reply (skip the 0x03FF / unused word)
sensor:
  - platform: template
    name: "Room temperature"
    id: room_temp
    unit_of_measurement: "°C"
    accuracy_decimals: 1

canbus:
  - platform: esp32_can
    # ...bus config...
    on_frame:
      - can_id: 0x408          # reply for PT sent to 0x409
        then:
          - lambda: |-
              if (x.size() >= 6 && x[0] == 0xAA && x[1] == 0x00) {
                uint16_t a = (x[2] << 8) | x[3];
                uint16_t b = (x[4] << 8) | x[5];
                uint16_t raw = (a == 0x03FF) ? b : a;
                if (raw != 0x03FF)
                  id(room_temp).publish_state(64.44f - 0.0791f * raw);
              }
```
