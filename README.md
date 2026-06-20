# Innoxel / Spline "Noxnet" CAN bus — protocol notes

Community notes documenting **how the Noxnet CAN bus protocol works** on **Spline /
Innoxel** home-automation installations, so that owners of these systems can understand
and interoperate with their own bus (for example, from [ESPHome](https://esphome.io/) /
Home Assistant on a cheap ESP32 + CAN transceiver).

This is a description of an observed wire protocol, written for **interoperability and
educational purposes**. It is unofficial and not affiliated with, endorsed by, or derived
from any vendor's source code or documentation — everything here was learned by observing
CAN traffic on a working installation. All product and company names belong to their
respective owners.

> 💬 Discussion thread on the Home Assistant community forum:
> **[Canbus / Spline / Innoxel interface with HA](https://community.home-assistant.io/t/canbus-spline-innoxel-interface-with-ha/900008)**

---

## Credits — where this started

This effort was kicked off by **Vincèn PUJOL of [Domedia](https://www.domedia.net)**,
whose April 2025 write-up documented the first decode of the Noxnet protocol: the
button / relay / dimmer / shade message formats, the CAN-address-to-module mapping, and a
working ESPHome integration approach. If you are starting from scratch, **read his article
first** — it is the clearest introduction to the system and the inspiration for everything
here:

> 📖 **["Use case: freeing a fully locked home-automation system"](https://www.domedia.net/?p=1761&lang=en)** — Domedia

These notes **build on that foundation** and add the part Vincèn's article left open: the
**poll protocol** the bus uses for *reading* state and temperature back — the request/reply
exchange that lets a client recover current state, confirm a command's effect, and read the
in-wall temperature sensors.

---

## The system in one paragraph

Spline/Innoxel installs put **every input and output on a single CAN bus** (100 kbps,
standard 11-bit IDs), similar in concept to KNX. Wall buttons, motion sensors and
temperature sensors are *inputs*; relays, phase/0-10V/DALI dimmers and shade motors are
*outputs*. Each physical module has a fixed CAN address, and the controller maps inputs to
outputs over the bus.

The protocol is **ASCII commands inside CAN data frames** (e.g. `T3` = button 3 pressed,
`S6` = relay channel 6 on, `W2050000` = dimmer channel 2 to 50 %). Understanding these
messages is enough for a CAN client to observe and participate in the bus.

---

## What's in here

| Document | Contents |
| :-- | :-- |
| **[docs/specification.md](docs/specification.md)** | The complete technical reference: physical layer, CAN-ID addressing, all control commands (buttons, relays, dimmers, shades), the poll/reply protocol for reading state, and the byte-level state and temperature decode. |
| **[docs/esphome-examples.md](docs/esphome-examples.md)** | Copy-pasteable, standalone ESPHome snippets: a CAN sniffer, button → `binary_sensor`, relay → `switch`, shade → `cover`, dimmer → `light`, and a read-only state/temperature poller. |

---

## Hardware

The same minimal rig Vincèn used works perfectly:

- An **ESP32** board (CAN is only supported on ESP32 — **not** ESP8266/8285). A
  Seeed Studio XIAO ESP32-C3 or any ESP32 dev board is fine.
- A **CAN transceiver** based on the **TJA1050** (or SN65HVD230 for 3.3 V) chip.
- Two wires onto the CAN bus (mind the polarity) and a 120 Ω termination if you're at a
  bus end.

ESPHome's `esp32_can` platform speaks the bus natively at `bit_rate: 100kbps`,
`use_extended_id: false`.

---

## Status & scope

✅ **Documented and confirmed:** the control protocol, the poll/reply protocol for
reading state, temperature decode, and state read-back decode (lights, switches, covers,
dimmers).

If you have an Innoxel/Spline system, **contributions and corrections are very welcome** —
especially confirmations from other installs (channel counts, module variants, DALI
behaviour). Open an issue or pull request, or reply on the forum thread linked above.

---

## A word of caution

These commands actuate real relays, dimmers and shade motors in your home. The **poll
commands are read-only** and safe, but the control commands are not —
test on a known-safe circuit first, and never blindly sweep addresses you haven't mapped.
Everything here is provided as-is, with no warranty; you are responsible for your own
installation.

---

## Need professional help?

If you'd rather have this done for you — or want a tailored integration, a turnkey
build, or commercial support — **[Domedia](https://www.domedia.net)** (Vincèn PUJOL),
who pioneered this work, offers it as a professional service. Reach out to them directly.

---

## License

Released under the [MIT License](LICENSE) — free to use, adapt and redistribute, including
the configuration examples. Attribution to the original protocol work (Vincèn PUJOL /
Domedia) is appreciated.
