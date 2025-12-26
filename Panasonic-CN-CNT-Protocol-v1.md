# Panasonic AC CN-CNT Protocol Analysis

| Version | Date       | Author | Changes                                      |
|---------|------------|--------|----------------------------------------------|
| 2.0     | 2025-12-22 | -      | **MAJOR**: Remote testing - Fan modes confirmed (Auto/Quiet/Powerful use b5=0xA0 + b7 flags), B13 offset correlates with outside temp (r=-0.71), Sleep timer not visible |
| 1.9     | 2025-12-22 | -      | B13 offset for RUN expanded to +2 to +6 (same as IDLE), Auto fan = 0xA0 confirmed |
| 1.8     | 2025-12-22 | -      | **DEFROST DETECTED**: Byte 14 = 0x02 during defrost, RUN→START transition |
| 1.7     | 2025-12-21 | -      | B13 offset varies by state: +6 right after RUN→IDLE, settles to +4 |
| 1.6     | 2025-12-16 | -      | **Power OFF command verified**, protocol probing (only 0x70/0xF0 respond) |
| 1.5     | 2025-12-16 | -      | Feature testing: Powerful/Quiet NOT in Slot 2, B13 offset +5 in Powerful mode |
| 1.4     | 2025-12-16 | -      | **MUX Slot 2 discovery**: NanoE-X status in bytes 31-33, not static |
| 1.3     | 2025-12-14 | -      | OFF vs IDLE baseline distinction, power formula refinement (×1.10) |
| 1.2     | 2025-12-14 | -      | **NEW STATE 0x04** (power-down transition), byte 13 behavior when OFF |
| 1.1     | 2025-12-14 | -      | **Control commands tested**: Bidirectional communication confirmed, set-temp working |
| 1.0     | 2025-12-13 | -      | **MAJOR**: Discovered byte 12 = 0x00 (OFF state), complete state machine now 5 states |
| 0.9     | 2025-12-13 | -      | Analyzed unknown bytes: 10 static, byte 13 variable (temp offset) |
| 0.8     | 2025-12-13 | -      | Bytes 31-33 are static identifiers (not telemetry), same values in all states |
| 0.7     | 2025-12-13 | -      | Validated power formulas (R²>0.99), b30=current×5, confirmed outdoor unit measurement |
| 0.6     | 2025-12-13 | -      | Corrected startup timing, byte 12 bit meanings, power formula rework, thermal baselines |
| 0.5     | 2025-12-13 | -      | Byte 12 state machine (4 states), power formula issues, bytes 31-33 mux pattern |
| 0.4     | 2025-12-12 | -      | Confirmed settings bytes via Sensibo/IR, confirmed temps via Panasonic app, added serial params |
| 0.3     | 2025-06-01 | -      | Added ODS documentation, complete byte map, swing matrix, TODO section |
| 0.2     | 2025-06-01 | -      | Initial protocol analysis from ESPHome component |
| 0.1     | -          | -      | Original ESPHome implementation by @DomiStyle |

> **Status**: Working Document - Active Investigation

### Legend
| Symbol | Meaning |
|--------|---------|
| ✅ Confirmed | Verified via external source or multiple observations |
| ⚠️ Theory/Unconfirmed | Hypothesis based on limited data, needs verification |
| ❓ Unknown | Purpose not yet determined |

## Overview

The CN-CNT port is a serial UART interface found on Panasonic air conditioners, typically used by the CZ-TACG1 WiFi adapter module. This document describes the protocol reverse-engineered from the ESPHome component and live serial monitoring.

**Tested Unit:** CS-HZ35XKE (3.5kW heat pump, NanoE-X, no EcoNavi)

## Physical Layer

- **Interface**: UART Serial
- **Baud Rate**: 9600 ✅ Confirmed
- **Data Bits**: 8
- **Parity**: Even ✅ Confirmed
- **Stop Bits**: 1
- **Connection**: CN-CNT port on the AC indoor unit
- **Read Timeout**: 20ms (time to wait before considering a packet complete)

---

## Packet Structure

### General Format

```
┌────────┬────────┬─────────────────────┬──────────┐
│ Header │ Length │      Payload        │ Checksum │
│ 1 byte │ 1 byte │    N bytes          │  1 byte  │
└────────┴────────┴─────────────────────┴──────────┘
```

| Field    | Size   | Description                                    |
|----------|--------|------------------------------------------------|
| Header   | 1 byte | `0x70` = Poll, `0xF0` = Control                |
| Length   | 1 byte | Number of payload bytes (N)                    |
| Payload  | N bytes| Data bytes                                     |
| Checksum | 1 byte | Sum of all bytes (including checksum) = 0x00   |

### Checksum Calculation

The checksum is calculated so that the sum of all bytes in the packet equals zero:

```cpp
uint8_t checksum = 0;
for (uint8_t i : packet_without_checksum)
    checksum -= i;  // Subtract each byte
packet.push_back(checksum);
```

To verify: sum all bytes including checksum; result should be 0x00.

---

## Message Types

### Poll Command (Controller → AC)

- **Header**: `0x70`
- **Payload Length**: 10 bytes (0x0A)
- **Interval**: Every 5000ms (5 seconds)
- **Payload**: All zeros

```
TX: 70 0A 00 00 00 00 00 00 00 00 00 00 86
    │  │  └──────────────────────────┘  │
    │  │         10 zero bytes          │
    │  └─ Length (10)                   └─ Checksum (0x86)
    └─ Header (Poll)
```

### Poll Response (AC → Controller)

- **Header**: `0x70`
- **Payload Length**: 32 bytes (0x20)
- **Total Packet Size**: 35 bytes (header + length + 32 payload + checksum)

### Control Command (Controller → AC)

- **Header**: `0xF0`
- **Payload Length**: 10 bytes (0x0A)
- **Total Packet Size**: 13 bytes (header + length + 10 payload + checksum)
- **Interval**: Commands sent with 250ms minimum spacing

> ✅ **CONFIRMED (v1.1)**: Bidirectional control tested and working via Raspberry Pi on 2025-12-14.

**Example - Set Temperature to 20.5°C:**
```
TX: F0 0A 44 29 80 30 5C 00 00 00 00 00 8D
    │  │  │  │  │  │  │              │
    │  │  │  │  │  │  │              └─ Checksum
    │  │  │  │  │  │  └─ Swing (V+H)
    │  │  │  │  │  └─ Fan speed (0x30 = level 1)
    │  │  │  │  └─ Mild dry
    │  │  │  └─ Target temp (0x29 = 41 → 20.5°C)
    │  │  └─ Mode + Power (0x44 = Heat + ON)
    │  └─ Length (10)
    └─ Header (Control)

RX: 70 20 44 29 80 30 5C 00 00 40 00 00 4C 2C ... (35 bytes)
    │     │  │                       │  │
    │     │  └─ Target confirmed     │  └─ Byte 13 = target + 3
    │     └─ Mode confirmed          └─ State (Running)
    └─ Poll response header
```

**Tested Commands (2025-12-14/16):**
| Command | Hex Sent | Result |
|---------|----------|--------|
| Set temp 20.5°C | `F00A442980305C00000000008D` | ✅ Verified |
| Set temp 20.0°C | `F00A442880305C00000000008E` | ✅ Verified |
| Power OFF | `F00A0000000000000000000006` | ✅ Verified (2025-12-16) |

> **Note**: Power OFF command sends all zeros in payload. Mode+Power byte = 0x00 triggers shutdown.

---

## Poll Response - Complete Byte Map (35 bytes)

| Byte | Example | Purpose                          | Status      |
|------|---------|----------------------------------|-------------|
| 0    | 0x70    | Header                           | ✅ Confirmed |
| 1    | 0x20    | Payload length (32)              | ✅ Confirmed |
| 2    | 0x44    | Mode + Power                     | ✅ Confirmed (via Sensibo/IR) |
| 3    | 0x28    | Target temperature × 2           | ✅ Confirmed (via Sensibo/IR) |
| 4    | 0x80    | Mild dry                         | ✅ Known    |
| 5    | 0x40    | Fan speed                        | ✅ Confirmed (via Sensibo/IR) |
| 6    | 0x5C    | Swing (V + H)                    | ✅ Confirmed (via Sensibo/IR) |
| 7    | 0x00    | Preset / NanoE-X / EcoNavi       | ✅ Known    |
| 8    | 0x00    | Reserved (always 0x00)           | ✅ Static   |
| 9    | 0x40    | Unknown flag (always 0x40)       | ✅ Static   |
| 10   | 0x00    | Eco mode                         | ✅ Known    |
| 11   | 0x00    | Reserved (always 0x00)           | ✅ Static   |
| 12   | 0x00-4C | Operational status (state machine)| ✅ Confirmed |
| 13   | 0x2A-2C | Temp offset (target +2 to +4 when ON, =target when OFF) | ✅ Confirmed |
| 14   | 0x00/02 | **Defrost flag** (0x00=normal, 0x02=defrost) | ✅ Confirmed |
| 15   | 0x00    | Reserved (always 0x00)           | ✅ Static   |
| 16   | 0x00/30 | Model-specific (XKE=0x00, ZKE=0x30) | ❓ Unknown |
| 17   | 0x00/93 | Model-specific (XKE=0x00, ZKE=0x93) | ❓ Unknown |
| 18   | 0x17    | Current/room temperature         | ✅ Confirmed (via Zigbee sensor at inlet) |
| 19   | 0x03    | Outside temperature              | ✅ Confirmed (via Panasonic app) |
| 20   | 0x1F    | Humidity %                       | ✅ Confirmed (via Zigbee sensor) |
| 21   | 0x16    | Room temp display (≈ b18, ±1°C)  | ✅ Confirmed |
| 22   | 0x03    | Outside temperature (alt)        | ✅ Confirmed |
| 23   | 0xFF    | Marker                           | ✅ Known    |
| 24   | 0x80    | Reserved/unsupported (always 0x80)| ✅ Static   |
| 25   | 0x80    | Reserved/unsupported (always 0x80)| ✅ Static   |
| 26   | 0xFF    | Marker                           | ✅ Known    |
| 27   | 0x80    | Reserved/unsupported (always 0x80)| ✅ Static   |
| 28   | 0x22    | Outdoor unit power (low byte)    | ✅ Validated |
| 29   | 0x00    | Outdoor unit power (high byte)   | ✅ Validated |
| 30   | 0x01    | Compressor current × 5           | ✅ Validated (R²=0.9948) |
| 31   | 0x80/C0/C1 | Static identifier slot selector | ✅ See below |
| 32   | 0x19/00/44 | Static identifier data          | ✅ See below |
| 33   | 0x83/00/15 | Static identifier data          | ✅ See below |
| 34   | 0x??    | Checksum                         | ✅ Confirmed |

### Bytes 31-33: Multiplexed Status/Identifiers

> ✅ **UPDATED (v1.4)**: Slot 2 is NOT static - it reflects NanoE-X status!

**Observed Patterns:**
| b31  | b32  | b33  | Slot | Interpretation |
|------|------|------|------|----------------|
| 0x80 | 0x19 | 0x83 | 1    | Model/unit identifier (static) - CS-HZ35XKE |
| 0x80 | 0x52 | 0x01 | 1    | Model/unit identifier (static) - CS-HZ35ZKE |
| 0xC0 | 0x00 | 0x00 | 2    | Status: NanoE-X **OFF** |
| 0xC0 | 0x04 | 0x00 | 2    | Status: NanoE-X **ON** |
| 0xC1 | 0x44 | 0x15 | 3    | Version/config identifier (static) - CS-HZ35XKE |
| 0xC1 | 0x47 | 0x17 | 3    | Version/config identifier (static) - CS-HZ35ZKE |

**Slot 2 Discovery (2025-12-16):**
- Middle byte (b32) changes based on NanoE-X setting
- 0x00 = NanoE-X disabled
- 0x04 = NanoE-X enabled
- ✅ **Tested:** Powerful and Quiet modes do NOT appear in Slot 2 (only in byte 7)

**Slot Purpose:**
- Slot 1 (0x80): Unit identification (static)
- Slot 2 (0xC0): Feature status (dynamic)
- Slot 3 (0xC1): Version/config (static)
- NOT: compressor Hz, fan RPM, or any dynamic sensor data

✅ Confirmed static via 24h analysis across all operational states (2025-12-13)

---

### ✅ Defrost Detection (CONFIRMED 2025-12-22)

**Byte 14 = Defrost Flag:**
| Value | Meaning |
|-------|---------|
| 0x00  | Normal operation |
| 0x02  | **Defrost cycle active** |

**Defrost Behavior Observed:**
- Byte 14 changes from 0x00 → 0x02 when defrost starts
- Byte 12 shows unusual **RUN → START** transition (normally goes through IDLE)
- Defrost duration: ~7-10 minutes per cycle
- Cycle frequency: ~1 hour at -2°C outside temperature
- At end of defrost: b14 returns to 0x00, then b12 transitions START → RUN

**Example Timeline (2025-12-22 03:36-03:41):**
```
03:36:00  b12=0x4C (RUN), b14=0x02 (DEFROST) - defrosting while "running"
03:39:05  b12=0x48 (START), b14=0x02 (DEFROST) - transition phase
03:40:35  b14=0x00 (defrost ends)
03:40:45  b12=0x4C (RUN), b14=0x00 (normal operation resumes)
```

---

### ✅ Byte 13 Offset Analysis (CONFIRMED 2025-12-22)

**Discovery**: Byte 13 offset strongly correlates with outside temperature (r = **-0.71**)

Byte 13 = Byte 3 (target) + offset. The offset represents an internal "heating effort level" that compensates for heat loss in cold weather.

**Correlation Analysis (104k samples over 10 days):**

| Outside Temp | Avg Offset | Internal Boost | Interpretation |
|--------------|------------|----------------|----------------|
| -4°C to -2°C | +4.0 to +4.2 | +2.0°C | Cold - max effort |
| 0°C to 2°C | +3.6 to +3.9 | +1.8°C | Cold - high effort |
| 4°C to 6°C | +3.0 to +3.5 | +1.5°C | Mild - normal |
| 8°C to 10°C | +2.0 to +2.7 | +1.0°C | Warm - low effort |

> **Note**: Offset is in raw units (same as byte 3). Divide by 2 for °C boost.

**Practical Meaning**:
- If setpoint is 20°C and offset is +4 (cold day), internal target = 22°C
- The AC overshoots slightly to ensure room reaches setpoint despite heat loss
- This is automatic - no need to manually raise setpoint in cold weather

**Control Strategy Implication**:
- Adjusting setpoint compounds with offset (25°C setpoint + 2°C offset = 27°C internal target)
- Fan level control is more efficient for modulating output without changing thermal target

**Service Manual Confirmation**:
> "When the powerful mode is selected, the internal setting temperature will shift higher up to 1°C (for Heating) than remote control setting temperature to achieve the setting temperature quickly."

This confirms byte 13 is the "internal setting temperature" used by the control system. Powerful mode adds +1°C (+2 raw units) to the base offset.

### Control Loop (from Service Manual)

> "The compressor at outdoor unit is operating following the frequency instructed by the microcomputer at indoor unit that judging the condition according to internal setting temperature and intake air temperature."

```
┌─────────────────────────────────────────────────────────┐
│                    Indoor Unit MCU                       │
│                                                         │
│   Internal Setting Temp (b13) ──┐                       │
│                                 ├──► Calculate gap      │
│   Intake Air Temp (b18) ────────┘         │             │
│                                           ▼             │
│                                   Compressor frequency  │
│                                   command to outdoor    │
└─────────────────────────────────────────────────────────┘
```

- **b13** = Internal target (user setpoint + weather-based offset + mode boost)
- **b18** = Actual intake air temperature
- **Gap (b13 - b18)** = Determines compressor effort
- Compressor Hz is NOT reported in CN-CNT status (it's a command to outdoor unit)

---

## Control Command - Byte Map (10 bytes payload)

| Byte | Purpose                          | Status |
|------|----------------------------------|--------|
| 0    | Mode + Power                     | ✅ Tested |
| 1    | Target temperature × 2           | ✅ Tested |
| 2    | Mild dry                         | ⚠️ Untested |
| 3    | Fan speed                        | ⚠️ Untested |
| 4    | Swing (V + H)                    | ⚠️ Untested |
| 5    | Preset / NanoE-X / EcoNavi       | ⚠️ Untested |
| 6    | (unused - send 0x00)             | ✅ Confirmed |
| 7    | (unused - send 0x00)             | ✅ Confirmed |
| 8    | Eco mode                         | ⚠️ Untested |
| 9    | (unused - send 0x00)             | ✅ Confirmed |

**Building a Control Command:**
1. Read current state via poll command (0x70)
2. Copy bytes 2-7 and 10 from response to payload bytes 0-5 and 8
3. Modify desired values
4. Set unused bytes (6, 7, 9) to 0x00
5. Calculate checksum: `(256 - sum(all_bytes)) % 256`
6. Send: `[0xF0, 0x0A, payload[0-9], checksum]`

---

## Field Definitions

### Byte 2: Mode + Power State

```
┌───────────────┬───────────────┐
│  High Nibble  │  Low Nibble   │
│    (Mode)     │   (Power)     │
└───────────────┴───────────────┘
```

| Value | Mode         | Notes           |
|-------|--------------|-----------------|
| 0x04  | Auto + ON    | Heat/Cool       |
| 0x34  | Cool + ON    |                 |
| 0x44  | Heat + ON    |                 |
| 0x24  | Dry + ON     |                 |
| 0x64  | Fan Only + ON|                 |
| 0x30  | Cool + OFF   | (or any x0)     |

- High nibble: 0=Auto, 2=Dry, 3=Cool, 4=Heat, 6=Fan
- Low nibble: 0=OFF, 4=ON

### Byte 3: Target Temperature

- **Format**: Raw value × 0.5 = Temperature in °C
- **Range**: 16°C to 30°C (values 32 to 60)
- **Example**: 0x2B (43) = 21.5°C

### Byte 4: Mild Dry

| Value | State    |
|-------|----------|
| 0x7F  | ON       |
| 0x80  | OFF      |

### Byte 5: Fan Speed

| Value | Speed      | Status |
|-------|------------|--------|
| 0x30  | Level 1 (Low) | ✅ Confirmed (remote 2025-12-22) |
| 0x40  | Level 2    | ✅ Confirmed (remote 2025-12-22) |
| 0x50  | Level 3 (Medium) | ✅ Confirmed (remote 2025-12-22) |
| 0x60  | Level 4    | ✅ Confirmed (remote 2025-12-22) |
| 0x70  | Level 5 (High) | ✅ Confirmed (remote 2025-12-22) |
| 0xA0  | Auto/Quiet/Powerful | ✅ Confirmed (remote 2025-12-22) |

> ✅ **IMPORTANT (v2.0)**: Auto, Quiet, and Powerful modes all set b5=0xA0. They are differentiated by byte 7:

| b5 | b7 | Mode | Notes |
|----|-----|------|-------|
| 0x30-0x70 | 0x00 | Fixed L1-L5 | Byte 5 stays locked to set level |
| 0xA0 | 0x00 | **Auto** | AC modulates fan internally |
| 0xA0 | 0x02 | **Powerful** | Max effort heating/cooling |
| 0xA0 | 0x04 | **Quiet** | Low noise, AC modulates fan |

> **Note**: Sensibo/IR cannot send Quiet mode - only the physical remote can. When fixed levels (L1-L5) are set, byte 5 shows that exact value. When Auto/Quiet/Powerful are set, byte 5 shows 0xA0.

> **Sleep Timer**: Not visible in CN-CNT status. The sleep button on remote cycles through 0.5h-9h shutdown timers but no byte changes in the poll response.

### Byte 6: Swing Position (Combined)

```
┌───────────────┬───────────────┐
│  High Nibble  │  Low Nibble   │
│  (Vertical)   │ (Horizontal)  │
└───────────────┴───────────────┘
```

#### Complete Swing Matrix

| V \ H        | auto | left | left_center | center | right_center | right |
|--------------|------|------|-------------|--------|--------------|-------|
| **auto**     | 0xFD | 0xF9 | 0xFA        | 0xF6   | 0xFB         | 0xFC  |
| **up**       | 0x1D | 0x19 | 0x1A        | 0x16   | 0x1B         | 0x1C  |
| **up_center**| 0x2D | 0x29 | 0x2A        | 0x26   | 0x2B         | 0x2C  |
| **center**   | 0x3D | 0x39 | 0x3A        | 0x36   | 0x3B         | 0x3C  |
| **down_center**| 0x4D | 0x49 | 0x4A      | 0x46   | 0x4B         | 0x4C  |
| **down**     | 0x5D | 0x59 | 0x5A        | 0x56   | 0x5B         | 0x5C  |

**Vertical (High Nibble):**
- 0xF_ = Auto, 0xE_ = Swing
- 0x1_ = Up, 0x2_ = Up-Center, 0x3_ = Center
- 0x4_ = Down-Center, 0x5_ = Down
- 0x0_ = Unsupported

**Horizontal (Low Nibble):**
- 0x_D = Auto
- 0x_9 = Left, 0x_A = Left-Center, 0x_6 = Center
- 0x_B = Right-Center, 0x_C = Right
- 0x_0 = Unsupported

### Byte 7: Preset / NanoE-X / EcoNavi

| Value | Setting          |
|-------|------------------|
| 0x00  | Normal           |
| 0x02  | Powerful         |
| 0x04  | Quiet            |
| 0x40  | Normal + NanoE-X |
| 0x42  | Powerful + NanoE-X|
| 0x44  | Quiet + NanoE-X  |
| 0x10  | EcoNavi          |

**Bit Layout:**
- Bit 6 (0x40): NanoE-X
- Bit 4 (0x10): EcoNavi
- Bits 0-2: Preset (0=Normal, 2=Powerful, 4=Quiet)

### Byte 10: Eco Mode

| Value | State |
|-------|-------|
| 0x00  | OFF   |
| 0x40  | ON    |

### Byte 12: Operational Status (State Machine)

| Value | Binary      | State                    | Status |
|-------|-------------|--------------------------|--------|
| 0x00  | 0000 0000   | **OFF** (unit powered off via mode) | ✅ Confirmed |
| 0x04  | 0000 0100   | **Power-down transition** | ✅ Confirmed (v1.2) |
| 0x40  | 0100 0000   | Idle (ON but waiting)    | ✅ Confirmed |
| 0x44  | 0100 0100   | Shutdown transition      | ⚠️ Intermittent (see note) |
| 0x48  | 0100 1000   | Startup (fan pre-heat)   | ✅ Confirmed |
| 0x4C  | 0100 1100   | Running (full operation) | ✅ Confirmed |

> ✅ **NEW DISCOVERY (v1.2)**: State 0x04 is a brief power-down transition when turning OFF from RUN. Sequence: RUN (0x4C) → 0x04 → OFF (0x00). This is distinct from 0x44 (which transitions to IDLE, not OFF).

> ⚠️ **Note (v1.3)**: State 0x04 only appears when an explicit OFF command is sent while running. Normal thermostat modulation (RUN → IDLE) uses 0x44 transition, not 0x04. If the unit goes to IDLE naturally, you won't see 0x04.

**Bit Layout:**
```
0x4C = 0100 1100 - Running (fan + compressor)
0x48 = 0100 1000 - Startup (fan only, pre-compressor)
0x44 = 0100 0100 - Shutdown transition (to IDLE)
0x40 = 0100 0000 - Idle (ON but waiting)
0x04 = 0000 0100 - Power-down transition (to OFF)
0x00 = 0000 0000 - OFF (unit disabled via mode)
       ││││ ││││
       ││││ │└┴┴─ Bits 0-2: (always 0 in observed states)
       ││││ └──── Bit 3 (0x08): Fan active
       │││└────── Bit 2 (0x04): Compressor active
       │└┴─────── Bits 4-6: State active flag (010 when ON)
       └───────── Bit 7: (always 0)
```

**Bit Interpretation:**
| State | Bit 6 | Bit 3 | Bit 2 | Observed Context |
|-------|-------|-------|-------|------------------|
| 0x00  | 0     | 0     | 0     | Unit OFF (via mode change) |
| 0x40  | 1     | 0     | 0     | Idle (ON but waiting) |
| 0x48  | 1     | 1     | 0     | Startup (fan pre-heat before compressor) |
| 0x4C  | 1     | 1     | 1     | Running (full operation) |
| 0x44  | 1     | 0     | 1     | Shutdown transition (brief, ~5 sec) |

> ⚠️ **BIT MEANINGS UNCERTAIN**: The 0x44 state shows bit 3=0, bit 2=1. If bit 3=fan and bit 2=compressor, this would mean "fan off, compressor on" — but physically during normal shutdown, compressor stops first while fan runs to dissipate heat. Possible explanations:
> - Bit meanings may be swapped (bit 3=compressor, bit 2=fan)
> - 0x44 may represent a different condition than normal shutdown
> - 0x44 pattern may match defrost mode (compressor on, fan off to prevent cold air)
>
> **Needs verification during defrost cycle** when we'd expect compressor on + fan off.

**State Transitions Observed:**
- **Power ON**: 0x00 → 0x40 (OFF → idle, ~30 seconds delay)
- **Startup**: 0x40 → 0x48 → 0x4C (idle → fan pre-heat → full operation)
- **Shutdown to idle**: 0x4C → 0x44 → 0x40 (running → transition → idle)
- **Direct shutdown**: 0x4C → 0x40 (occurs ~50% of time, see note below)
- **Power OFF from RUN**: 0x4C → 0x04 → 0x00 (running → power-down → OFF)
- **Power OFF from IDLE**: 0x40 → 0x00 (idle → OFF, immediate)

**Power Correlation:**
| State | Typical Power | Description |
|-------|---------------|-------------|
| 0x00  | ~7W (Shelly)  | Unit OFF (same standby as idle) |
| 0x04  | ~7W (Shelly)  | Power-down transition (~5 sec) |
| 0x40  | ~7W (Shelly)  | Standby/idle |
| 0x44  | ~7W (Shelly)  | Shutdown transition (~5 sec) |
| 0x48  | ~92W          | Fan only (pre-compressor) |
| 0x4C  | ~750W+        | Full operation (varies with load) |

**Startup Sequence Timing (observed):**
- 0x40 → 0x48: Immediate on mode change
- 0x48 → 0x4C: **15-20 seconds** (3-4 poll cycles)

> ⚠️ **Note**: Original ESPHome documentation stated ~3 minutes for startup. This appears to confuse the compressor protection delay (minimum time between stop/start cycles) with actual startup timing. Live testing shows 15-20 seconds consistently.

**0x44 Detection Note:**
> ✅ **CONFIRMED (v1.3)**: The 0x44 state is real and consistent - captured on multiple shutdown cycles. It's a brief transition (~5 seconds) so may be missed with 5-second polling. Both observed shutdowns showed: 0x4C → 0x44 (~480W) → 0x40 (~27W). Code should still detect 0x4C → 0x40 transitions as backup.

✅ States 0x00, 0x04, 0x40, 0x44, 0x48, 0x4C confirmed via live testing (2025-12-12/13/14)
✅ State 0x00 (OFF) discovered via live testing with mode change (2025-12-13)
✅ State 0x04 (power-down) discovered via anomaly watchdog (2025-12-14)
⚠️ States 0x04 and 0x44 are brief transitions - may be missed at 5s polling

> ⚠️ **UNCONFIRMED - Defrost Theory**: Other bit patterns (e.g., 0x50, 0x54) may indicate defrost/deicing mode. Prediction: defrost may show 0x44 (compressor on, fan off) or a new value. Defrost cycles have not been observed yet - needs cold weather testing.

---

## Temperature & Humidity Fields

| Byte | Description                | Notes                     | Status |
|------|----------------------------|---------------------------|--------|
| 18   | Current/room temperature   | Sensor at AC air intake   | ✅ Confirmed (via Zigbee sensor) |
| 19   | Outside temperature        | Signed int8, °C           | ✅ Confirmed (via Panasonic app) |
| 20   | Humidity %                 | Sensor at AC air intake   | ✅ Confirmed (via Zigbee sensor) |
| 21   | Room temperature (display) | ≈ Byte 18 (same sensor, ±1°C variance) | ✅ Confirmed |
| 22   | Outside temp (alternate)   | Same as byte 19           | ✅ Confirmed |

- Value `0x80` = Unsupported/unavailable
- Values > 100 considered out of range

**Note**: Byte 18 confirmed via external Zigbee sensor at AC intake - values match within 0.5°C. Panasonic calls this "room temperature" as it's the only internal air sensor. Byte 20 (humidity) also confirmed via Zigbee sensor - matches within 0.5%. Byte 21 appears to be the same sensor as byte 18 with minor variance (±1°C) - likely due to timing or rounding differences, not a fixed offset.

### Thermal Baseline Data (Heating Mode)

> ⚠️ **Baseline for Defrost Detection**: The following thermal data may help identify defrost cycles when they occur.

**Running vs Stopped Temperatures (via external Zigbee sensors):**

| Condition | Inflow (b18) | Outflow | Delta T | Notes |
|-----------|--------------|---------|---------|-------|
| **Running** | ~23°C | **35-37°C** | ~12-14°C | Normal heating operation |
| **Stopped** | ~23°C | ~30°C | ~7°C | Coil cooling down |

> ⚠️ **Correction**: Previous documentation stated outflow ~30°C during normal operation. This is incorrect - 30°C only occurs when stopped (coil cooling down). Running outflow is 35-37°C.

**Thermal Dynamics (observed):**
| Transition | Rate | Thermal Lag |
|------------|------|-------------|
| Stop → cooldown | ~2°C/min | 45 seconds before temp drops |
| Start → warmup | ~1.5°C/min | 2 minutes to reach full temp |

**Humidity (via Zigbee sensors):**
| Location | Running | Notes |
|----------|---------|-------|
| Inflow | ~30% | Ambient room humidity |
| Outflow | ~18% | Heated air, lower relative humidity |

**Theory for Defrost Detection:**
- During defrost, outdoor unit reverses cycle (becomes cooling coil)
- Indoor fan typically stops or slows
- **Defrost signature**: Outflow dropping below 28°C during heating mode would be anomalous
- This indicates either: fan stopped, or refrigerant reversed
- Byte 12 may show different state (0x44 with fan off? or new value 0x50, 0x54?)
- Monitor for these conditions during winter operation

## Power Consumption

| Byte | Description      |
|------|------------------|
| 28   | Outdoor unit power (low byte) |
| 29   | Outdoor unit power (high byte) |
| 30   | Compressor current × 5 |

### Validated Formulas (R² > 0.99)

Based on correlation analysis with Shelly power meter (470 running samples):

**Power Formula:**
```
raw_power = b28 + (b29 × 256)
total_watts = raw_power × 1.10
```
- R² = 0.9943 (excellent correlation)
- CN-CNT measures **~91% of total power** (outdoor unit only)
- Difference (~9%) is indoor unit overhead (fan, electronics)

> ✅ **REFINED (v1.3)**: Original multiplier 1.14 was ~5% high at peak power. Testing shows ×1.10 is more accurate across the range. Error typically <2% at mid-range, ~5% at extremes.

**Current Formula:**
```
amps = b30 / 5.0
```
- R² = 0.9948 (excellent correlation)
- Average error: 0.15A

### Stopped State Baseline

> ✅ **UPDATED (v1.3)**: OFF and IDLE have distinct baseline patterns.

| State | b12  | b28       | b29  | b30  | Shelly Actual |
|-------|------|-----------|------|------|---------------|
| OFF   | 0x00 | 0x0F (15) | 0x00 | 0x00 | **~2-3W**     |
| IDLE  | 0x40 | 0x22 (34) | 0x00 | 0x01 | **~7-8W**     |

> ⚠️ **IMPORTANT**: When stopped, bytes 28-30 are **baseline constants**, not real measurements. The values differ between OFF and IDLE states, providing a way to distinguish them even without checking byte 12.

### Running State Validation

| CN-CNT Raw | Shelly Actual | Ratio | b30 | b30/5 | Shelly Amps |
|------------|---------------|-------|-----|-------|-------------|
| 95         | 76W           | 0.80  | 3   | 0.6A  | 0.61A       |
| 500        | 570W          | 1.14  | 14  | 2.8A  | 2.9A        |
| 932        | 1024W         | 1.10  | 22  | 4.4A  | 4.5A        |

### Summary

| Metric | Formula | R² | Notes |
|--------|---------|-----|-------|
| Total Power | `(b28 + b29×256) × 1.10` | 0.9943 | Only valid when running (b12=0x4C) |
| Current | `b30 / 5.0` | 0.9948 | Amps, ~0.15A accuracy |
| OFF baseline | b28=15, b30=0 | - | ~2-3W actual |
| IDLE baseline | b28=34, b30=1 | - | ~7-8W actual |

✅ Formulas validated via Shelly power meter correlation (2025-12-13)

---

## Timing Parameters

| Parameter        | Value   | Description                              |
|------------------|---------|------------------------------------------|
| POLL_INTERVAL    | 5000ms  | Interval between poll commands           |
| CMD_INTERVAL     | 250ms   | Minimum delay between control commands   |
| READ_TIMEOUT     | 20ms    | Time to wait before packet is complete   |

---

## Communication Flow

### Normal Operation

```
Controller                               AC Unit
    │                                       │
    │──── Poll (0x70, 13 bytes) ───────────>│
    │                                       │
    │<──── Response (0x70, 35 bytes) ───────│
    │                                       │
    │           ... 5 seconds ...           │
    │                                       │
```

### Control Command

```
Controller                               AC Unit
    │                                       │
    │──── Control (0xF0, 13 bytes) ────────>│
    │      (new settings)                   │
    │                                       │
    │<──── Response ────────────────────────│
    │                                       │
```

---

## TODO - Investigation Needed

### Defrost/Deicing Status (COMPLETED 2025-12-22)
- [x] ~~Monitor bytes 31-33 during defrost cycle~~ → Determined to be multiplexed telemetry, not status
- [x] **Byte 14 = 0x02 = DEFROST flag** (confirmed 2025-12-22)
- [x] Byte 12 shows RUN → START transition during defrost (unusual, normally goes through IDLE)
- [x] Defrost duration: ~7-10 minutes per cycle
- [x] Cycle frequency: ~1 hour at -2°C outside temperature

### Power Formula (COMPLETED)
- [x] Determine why CN-CNT reports 34 when actual power is 7W → Baseline constant, not real measurement
- [x] Investigate byte 30 → Compressor current × 5 (R²=0.9948)
- [x] Develop corrected formula → `total_watts = raw × 1.14` (R²=0.9943)
- [x] Determine if CN-CNT measures outdoor unit only → Yes, ~88% of total power

### Bytes 31-33 Identifiers (COMPLETED)
- [x] ~~Decode mux slot meanings~~ → Static identifiers, not telemetry
- [x] ~~Determine if compressor Hz is in multiplexed data~~ → No, values are constant
- [x] ~~Check if defrost status appears in any mux slot~~ → No, same values in all states

### Unknown Bytes (COMPLETED)
- [x] Byte 8: Always 0x00 (reserved)
- [x] Byte 9: Always 0x40 (unknown flag, static)
- [x] Bytes 11, 14-17: Always 0x00 (reserved)
- [x] Byte 13: Variable 0x2B/0x2C - related to target temp (target+3 or +4)
- [x] Bytes 24, 25, 27: Always 0x80 (reserved/unsupported marker)

### Control Commands (TESTED 2025-12-14)
- [x] **Bidirectional communication confirmed** - commands sent and acknowledged
- [x] **Set temperature working** - 20.0°C ↔ 20.5°C tested successfully
- [x] **Byte 13 confirmed**: Changes to target + 3 (raw units) after command
- [ ] Fan speed command - untested
- [ ] Mode change command - untested
- [x] **Power OFF command tested** - payload all zeros triggers shutdown (2025-12-16)
- [ ] Swing command - untested

### Completed Investigations
- [x] Byte 12 state machine (**6 states**: 0x00, 0x04, 0x40, 0x44, 0x48, 0x4C)
- [x] **0x00 = OFF state** (distinct from 0x40 idle) - confirmed via live testing
- [x] **0x04 = Power-down transition** (RUN → 0x04 → OFF) - discovered via anomaly watchdog
- [x] **Byte 13 behavior**: varies by mode and state:
  - OFF: equals target (b13 = b3)
  - RUN (Normal/Quiet): +2 to +6 (can be elevated, settles to +4)
  - IDLE (Normal/Quiet): +2 to +6 (elevated after RUN, settles in ~1 min)
  - Powerful: +4 to +8
- [x] **Byte 13 offset correlates with outside temp** (r=-0.71) - colder = higher offset (confirmed 2025-12-22)
- [x] Byte 12 bit meanings (bit 6 = ON, bit 3 = fan, bit 2 = compressor)
- [x] **Byte 14 = DEFROST flag** (0x00=normal, 0x02=defrost) - confirmed 2025-12-22
- [x] Startup timing corrected (15-20 sec, not 3 min)
- [x] 0x44 state is intermittent (~50% capture rate at 5s polling)
- [x] Byte 18 = inlet temperature (confirmed via Zigbee)
- [x] Byte 20 = humidity (confirmed via Zigbee)
- [x] Byte 21 ≈ byte 18 (same sensor, ±1°C variance)
- [x] **Bytes 31-33 = static identifiers** (NOT telemetry - same values in all states)
- [x] Thermal baselines documented (running: 35-37°C outflow, stopped: ~30°C)
- [x] **Power formula validated** (R²=0.9943): `total_watts = (b28 + b29×256) × 1.10`
- [x] **Current formula validated** (R²=0.9948): `amps = b30 / 5.0`
- [x] **CN-CNT measures outdoor unit only** (~91% of total power)
- [x] **Stopped state baselines**: OFF (b28=15, b30=0, ~2W) vs IDLE (b28=34, b30=1, ~8W)

### Fan Mode Testing (COMPLETED 2025-12-22 via physical remote)
- [x] **L1-L5 (0x30-0x70)**: Fixed fan levels, byte 5 shows exact value
- [x] **Auto (b5=0xA0, b7=0x00)**: AC modulates fan internally
- [x] **Powerful (b5=0xA0, b7=0x02)**: Max effort mode
- [x] **Quiet (b5=0xA0, b7=0x04)**: Low noise mode (Sensibo cannot send this!)
- [x] **Sleep timer**: NOT visible in CN-CNT status (internal timer only)
- [x] **+8/15C preset**: Just sets target=13°C + fan=L5, no special mode byte

### Open Questions
1. ~~Why does CN-CNT report 34 when actual power is 7W?~~ → **ANSWERED**: Baseline constant, not real measurement
2. ~~What's in the multiplexed bytes (31-33)?~~ → **ANSWERED**: Static identifiers, not dynamic data
3. ~~Byte 30 exact meaning?~~ → **ANSWERED**: Compressor current × 5 (validated R²=0.9948)
4. ~~What byte 12 value during defrost?~~ → **ANSWERED**: Byte 14 = 0x02 is defrost flag, byte 12 stays 0x4C/0x48
5. What do the static identifiers (31-33) represent? Model code? Serial? Firmware version?

---

## Data Collection Infrastructure

The following services collect data for protocol analysis:

| Service | Script | Interval | Data Collected |
|---------|--------|----------|----------------|
| monitor | monitor.py | 5s | CN-CNT raw bytes |
| shelly-monitor | shelly.py | 5s | Power (W), voltage, current |
| sensors-monitor | sensors.py | 60s | Zigbee inflow/outflow temp & humidity |
| acunit-monitor | acunit.py | 60s | HA climate entity state |

**Database**: `/home/sebby/debug/ac_data.db` (SQLite)

**Status Scripts** (with `--time 5m` or `--time 1h` option):
- `status.py` - CN-CNT readings
- `shelly-status.py` - Power meter readings
- `sensors-status.py` - Zigbee sensor readings
- `acunit-status.py` - HA climate entity readings

---

## Validation Sources

| Field | Validation Source | Method |
|-------|-------------------|--------|
| Mode, target temp, fan, swing | Sensibo/IR | Compare CN-CNT with IR control state |
| Current temp (b18) | Zigbee sensor at AC inlet | Direct comparison |
| Humidity (b20) | Zigbee sensor at AC inlet | Direct comparison |
| Outside temp (b19) | Panasonic Comfort Cloud app | Direct comparison |
| Display temp (b21) | Panasonic app "room temperature" | Confirmed = b18 + 2°C |
| Power consumption | Shelly power meter | Compare actual vs formula |
| Operational status (b12) | Power meter + observation | Correlate state with power |

---

## References

- Original ESPHome implementation by [@DomiStyle](https://github.com/DomiStyle/esphome-panasonic-ac)
- CZ-TACG1 WiFi Adapter Module (Panasonic's official WiFi adapter for CN-CNT)
- **Panasonic Service Manual** - CS-HZ35XKE (confirms internal temp shift, control loop)
- Live testing: Raspberry Pi connected to CN-CNT port (2025-12-12 to 2025-12-22)
- Physical remote testing: Fan modes, presets, sleep timer (2025-12-22)
