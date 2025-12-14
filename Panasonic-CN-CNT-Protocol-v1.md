# Panasonic AC CN-CNT Protocol Analysis

| Version | Date       | Author | Changes                                      |
|---------|------------|--------|----------------------------------------------|
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

The CN-CNT port is a serial UART interface found on Panasonic air conditioners, typically used by the CZ-TACG1 WiFi adapter module. This document describes the protocol reverse-engineered from the ESPHome component and official protocol documentation.

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

**Tested Commands (2025-12-14):**
| Command | Hex Sent | Result |
|---------|----------|--------|
| Set temp 20.5°C | `F00A442980305C00000000008D` | ✅ Verified |
| Set temp 20.0°C | `F00A442880305C00000000008E` | ✅ Verified |

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
| 14   | 0x00    | Reserved (always 0x00)           | ✅ Static   |
| 15   | 0x00    | Reserved (always 0x00)           | ✅ Static   |
| 16   | 0x00    | Reserved (always 0x00)           | ✅ Static   |
| 17   | 0x00    | Reserved (always 0x00)           | ✅ Static   |
| 18   | 0x17    | Current/room temperature         | ✅ Confirmed (via Zigbee sensor at inlet) |
| 19   | 0x03    | Outside temperature              | ✅ Confirmed (via Panasonic app) |
| 20   | 0x1F    | Humidity %                       | ✅ Confirmed (via Zigbee sensor) |
| 21   | 0x16    | Room/inlet temperature           | ✅ Confirmed (via Panasonic app) |
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

### Bytes 31-33: Static Identifiers (Not Telemetry)

> ✅ **CONFIRMED - Static Data**: These bytes appear to cycle but values are **constant** across all operational states. They are NOT dynamic telemetry.

**Observed Patterns (static, cycling display only):**
| b31  | b32  | b33  | Slot | Interpretation |
|------|------|------|------|----------------|
| 0x80 | 0x19 | 0x83 | 1    | Model/unit identifier (6531 or 33561) |
| 0xC0 | 0x00 | 0x00 | 2    | Empty/reserved |
| 0xC1 | 0x44 | 0x15 | 3    | Version/config identifier (17429 or 5444) |

**Analysis (24h data, all states):**
- Values are **identical** in IDLE, START, RUN, and SHUTDOWN states
- No correlation with power, current, or any operational metric (R²=0)
- The "cycling" is just polling sampling different slots, not values changing
- Slot 2 is always zeros - likely unused/reserved

**Likely Purpose:**
- Unit identification or serial number fragments
- Firmware/hardware version information
- Model capability flags
- NOT: compressor Hz, fan RPM, or any dynamic sensor data

✅ Confirmed static via 24h analysis across all operational states (2025-12-13)

---

### 🎯 Investigation Priority for Defrost Status

**UPDATED PRIORITY - Focus on Byte 12**
Byte 12 state machine is the most promising for defrost detection:
- Observed states: 0x40, 0x44, 0x48, 0x4C
- Other bit patterns (0x50, 0x54, etc.) may indicate defrost
- Need to capture data during active defrost cycle

**LOWER PRIORITY - Bytes 31-33**
These are multiplexed telemetry, not status flags:
- Cycle through patterns during operation
- May contain useful data but not for simple status detection

**MEDIUM PRIORITY - Other unknown bytes**
- Bytes 8, 9, 11-17: Unknown static values
- Bytes 24, 25, 27: Show 0x80 (could be status or "unsupported")

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

| Value | Speed      |
|-------|------------|
| 0xA0  | Automatic  |
| 0x30  | Level 1    |
| 0x40  | Level 2    |
| 0x50  | Level 3    |
| 0x60  | Level 4    |
| 0x70  | Level 5    |

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
| 21   | Room temperature (display) | Byte 18 + 2°C (calculated)| ✅ Confirmed |
| 22   | Outside temp (alternate)   | Same as byte 19           | ✅ Confirmed |

- Value `0x80` = Unsupported/unavailable
- Values > 100 considered out of range

**Note**: Byte 18 confirmed via external Zigbee sensor at AC intake - values match within 0.5°C. Panasonic calls this "room temperature" as it's the only internal air sensor. Byte 20 (humidity) also confirmed via Zigbee sensor - matches within 0.5%. Byte 21 is always byte 18 + 2°C (calculated, not a separate sensor).

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

### Defrost/Deicing Status (HIGH PRIORITY)
- [x] ~~Monitor bytes 31-33 during defrost cycle~~ → Determined to be multiplexed telemetry, not status
- [ ] Monitor byte 12 during defrost cycle (look for 0x50, 0x54, or 0x44 patterns)
- [ ] Capture thermal data (outflow < 28°C anomaly) during defrost
- [ ] Correlate byte 12 changes with thermal lag patterns

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
- [ ] Power on/off command - untested
- [ ] Swing command - untested

### Completed Investigations
- [x] Byte 12 state machine (**6 states**: 0x00, 0x04, 0x40, 0x44, 0x48, 0x4C)
- [x] **0x00 = OFF state** (distinct from 0x40 idle) - confirmed via live testing
- [x] **0x04 = Power-down transition** (RUN → 0x04 → OFF) - discovered via anomaly watchdog
- [x] **Byte 13 behavior**: target +2 to +4 when ON (varies), equals target when OFF
- [x] Byte 12 bit meanings (bit 6 = ON, bit 3 = fan, bit 2 = compressor) - needs defrost verification
- [x] Startup timing corrected (15-20 sec, not 3 min)
- [x] 0x44 state is intermittent (~50% capture rate at 5s polling)
- [x] Byte 18 = inlet temperature (confirmed via Zigbee)
- [x] Byte 20 = humidity (confirmed via Zigbee)
- [x] Byte 21 = byte 18 + 2°C (calculated display value)
- [x] **Bytes 31-33 = static identifiers** (NOT telemetry - same values in all states)
- [x] Thermal baselines documented (running: 35-37°C outflow, stopped: ~30°C)
- [x] **Power formula validated** (R²=0.9943): `total_watts = (b28 + b29×256) × 1.10`
- [x] **Current formula validated** (R²=0.9948): `amps = b30 / 5.0`
- [x] **CN-CNT measures outdoor unit only** (~91% of total power)
- [x] **Stopped state baselines**: OFF (b28=15, b30=0, ~2W) vs IDLE (b28=34, b30=1, ~8W)

### Open Questions
1. ~~Why does CN-CNT report 34 when actual power is 7W?~~ → **ANSWERED**: Baseline constant, not real measurement
2. ~~What's in the multiplexed bytes (31-33)?~~ → **ANSWERED**: Static identifiers, not dynamic data
3. ~~Byte 30 exact meaning?~~ → **ANSWERED**: Compressor current × 5 (validated R²=0.9948)
4. What byte 12 value during defrost? Prediction: 0x44 (compressor on, fan off) or 0x50/0x54
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

- ESPHome Panasonic AC Component
- CZ-TACG1 WiFi Adapter Module
- `protocol_description_query.ods` - Protocol documentation
- Original implementation by [@DomiStyle](https://github.com/DomiStyle/esphome-panasonic-ac)
- Live testing: Raspberry Pi connected to CN-CNT port (2025-12-12/13)
