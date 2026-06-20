# Noxnet CAN Protocol — Technical Specification

Complete technical reference for the **Noxnet** protocol used by Spline / Innoxel
installations. This document is specification only: layouts, addressing,
commands, and decode formulas.

---

## 1. Physical layer

| Parameter | Value |
| :-- | :-- |
| Bit rate | 100 kbps |
| CAN ID type | Standard, 11-bit (`use_extended_id: false`) |
| Transceiver | TJA1050 (5 V) / SN65HVD230 (3.3 V) |
| Termination | 120 Ω at each bus end |
| Payload encoding | ASCII strings inside the CAN data bytes |

---

## 2. Addressing

Three module families. The CAN ID is derived from the Spline module index. Consecutive
modules differ by 2 (C/A) or 8 (B); the intermediate address is used as the **poll-reply**
address (§5).

| Family | Module type | Index → CAN ID | Step | Range | Channels |
| :-- | :-- | :-- | :-- | :-- | :-- |
| **C** | Inputs (buttons, motion/radar, temp sensor) | `C1 → 0x402` | +2 (even) | `0x402`–`0x494` | 8 (`T1`–`T8`) |
| **A** | Relays / binary outputs, covers | `A1 → 0x203` | +2 (odd) | `0x203`–`0x239` | 8 (`S0`/`C0`–`S7`/`C7`) |
| **B** | Dimmers (phase / 0-10 V / DALI) | `B1 → 0x60C` | +8 | `0x60C`–`0x6FC` | 4 (`W0`–`W3`) |

**Channel numbering is reversed** relative to the physical keypad/output position
(keypad button 8 = channel index 0).

---

## 3. Control commands (write)

ASCII payload sent to the module's CAN ID.

### 3.1 Inputs (received, not sent)

| Payload | Meaning |
| :-- | :-- |
| `T<1-8>` | Press / active |
| `t<1-8>` | Release / inactive |

### 3.2 Relays (binary outputs)

| Payload | Meaning |
| :-- | :-- |
| `S<0-7>` | Channel ON |
| `C<0-7>` | Channel OFF |

### 3.3 Dimmers

Format: **`W<channel><brightness><time>`** (8 ASCII bytes).

| Field | Width | Range | Notes |
| :-- | :-- | :-- | :-- |
| `W` | 1 | — | Command |
| channel | 1 | `0`–`3` | Output channel |
| brightness | 3 | `000`–`100` | Percent, zero-padded |
| time | 3 | `000`–`nnn` | Ramp seconds, zero-padded |

Examples: `W2100000` (ch2 → 100 %, instant); `W2050004` (ch2 → 50 % over 4 s);
`W2000000` (ch2 off). Recommended: keep `time = 000` and ramp in software.

### 3.4 Covers / shades

Directional only; no absolute position.

| Payload | Meaning |
| :-- | :-- |
| `U<0-3>` | Up / open |
| `D<0-3>` | Down / close |
| `H<0-3>` | Halt / stop |

---

## 4. Poll commands (read)

A poll is the 2 ASCII bytes `P?`, sent to the module's **poll address** (§5). Replies
return on the **reply address** (§5). Poll commands are read-only.

| Command | Bytes | Returns | Answered by |
| :-- | :-- | :-- | :-- |
| `PT` | `50 54` | Temperature (§7) | C-series with NTC sensor |
| `PF` | `50 46` | Output status (§6) | all output families |
| `P8` | `50 38` | Dimmer levels (§6.3) | B-series dimmers |

`P8` to a non-dimmer returns the ACK `6B 38` ("k8").

---

## 5. Poll → reply address mapping

| Family | Device `can_id` | Poll target | Reply address |
| :-- | :-- | :-- | :-- |
| C-series | even | `can_id + 1` | `can_id` |
| A-series | odd | `can_id` | `can_id − 1` |
| B-series | `0x60C + 8k` | `can_id` | `can_id − 4` |

| Example poll | Reply on |
| :-- | :-- |
| `PT` → `0x409` | `0x408` |
| `PF` → `0x20B` | `0x20A` |
| `P8` → `0x624` | `0x620` |

Temperature address relation: **`can_id = 0x400 + 2 × node`**.

---

## 6. `PF` / `P8` reply decode (state)

### 6.1 `PF` frame — relays / switches / covers

```
BB  b1  05  b3  byte4  00
```

| Byte | Value | Meaning |
| :-- | :-- | :-- |
| 0 | `0xBB` | Marker |
| 1 | `0x00` / `0x01` | Device class: `0x00` = relay/switch, `0x01` = cover |
| 2 | `0x05` | Constant (length) |
| 3 | `0x52` / `0x20` | Module-class tag (secondary; not needed for state) |
| 4 | — | **STATE** (see below) |
| 5 | `0x00` | Constant |

### 6.2 `byte4` state decode

**Relay / switch module (`byte1 = 0x00`)** — per-channel ON bitmask; one frame covers all
channels:

```
channel_C_on = (byte4 >> C) & 1
```

**Cover module (`byte1 = 0x01`)** — direction-relay bitmask, 2 bits per cover:

```
cover_C_opening = (byte4 >> (2*C))     & 1
cover_C_closing = (byte4 >> (2*C + 1)) & 1
both 0          = halted / idle
```

Covers are open-loop (no position reported). The direction bit stays latched after the
motor reaches its end-stop until a Halt clears it.

### 6.3 `P8` frame — dimmer levels

```
99  ST  L0  L1  L2  L3  t  t
```

| Byte | Meaning |
| :-- | :-- |
| 0 | `0x99` marker |
| 1 | Module sub-type (`0x0C`/`0x11`/`0x0D`/`0x12` …) |
| 2 + C | **Level of channel C** (`0x00` = off, `0x64` = 100 = full) |
| 6–7 | Per-module config constant |

```
level(channel C) = byte[2 + C]
```

The level byte is the **physical output**, not the logical percent. Gamma relation:

```
output ≈ 100 · (percent / 100) ^ 2.85      (floor ≈ 10; output = 0 below ~15 %)
```

| logical % | 100 | 90 | 80 | 75 | 70 | 60 | 50 | 40 | 30 | 25 | ≤15 |
| :-- | --: | --: | --: | --: | --: | --: | --: | --: | --: | --: | --: |
| byte value | 100 | 74 | 53 | 44 | 36 | 23 | 14 | 10 | 10 | 10 | 0 |

### 6.4 Note
C-series modules answer `PF` with an `AA 01 …` frame; input/sensor state decode is not
specified here.

---

## 7. `PT` reply decode (temperature)

C-series modules carry an in-wall **NTC** sensor read via `PT`. Reply is two 16-bit
big-endian words:

```
AA  00  <wordA hi> <wordA lo>  <wordB hi> <wordB lo>
```

| Field | Meaning |
| :-- | :-- |
| byte 0 | `0xAA` marker |
| byte 1 | `0x00` (temperature reply; `0x01` = status, §6.4) |
| wordA | sensor 1 raw |
| wordB | sensor 2 raw |

- A module wires sensor 1 **or** sensor 2; the unused word reads **`0x03FF` (1023)** and
  must be ignored.
- If **both** words are `0x03FF`, no sensor is fitted — produce no reading.

**Conversion (inverted NTC — higher raw = cooler):**

```
°C = 64.44 − 0.0791 × raw
```

Valid across the ~22–29 °C band sampled; accuracy ≈ ±0.66 °C without per-sensor offset.

**Examples:**

```
AA 00 01 EA 03 FF  → wordA = 0x01EA = 490, wordB unused → 64.44 − 0.0791·490 = 25.7 °C
AA 00 03 FF 01 F0  → wordB = 0x01F0 = 496, wordA unused → 64.44 − 0.0791·496 = 25.2 °C
```

---

## 8. Behavioural constraints

- **No autonomous broadcast.** State and temperature appear only in response to a poll.
  A reader must poll; a passive sniffer (no controller, no poller) sees no state/temp.
- **Read-only polls.** `PF`/`PT`/`P8` never actuate.
- **Open-loop control.** Control commands return no confirmation; use polling for state.
- **Bus load.** The original controller sweeps continuously (~hundreds of polls at
  startup, then ~15/min). Use hardware acceptance filters; avoid verbose CAN logging.
