# Panasonic AC CN-CNT Protocol Analysis

| Version | Date       | Author | Changes                                      |
|---------|------------|--------|----------------------------------------------|
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
- **Interval**: Commands sent with 250ms minimum spacing

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
| 8    | 0x00    | **?**                            | ❓ Unknown  |
| 9    | 0x40    | **?**                            | ❓ Unknown  |
| 10   | 0x00    | Eco mode                         | ✅ Known    |
| 11   | 0x00    | **?**                            | ❓ Unknown  |
| 12   | 0x40-4C | Operational status (state machine)| ✅ Confirmed |
| 13   | 0x2C    | **?**                            | ❓ Unknown  |
| 14   | 0x00    | **?**                            | ❓ Unknown  |
| 15   | 0x00    | **?**                            | ❓ Unknown  |
| 16   | 0x00    | **?**                            | ❓ Unknown  |
| 17   | 0x00    | **?**                            | ❓ Unknown  |
| 18   | 0x17    | Current/room temperature         | ✅ Confirmed (via Zigbee sensor at inlet) |
| 19   | 0x03    | Outside temperature              | ✅ Confirmed (via Panasonic app) |
| 20   | 0x1F    | Humidity %                       | ✅ Confirmed (via Zigbee sensor) |
| 21   | 0x16    | Room/inlet temperature           | ✅ Confirmed (via Panasonic app) |
| 22   | 0x03    | Outside temperature (alt)        | ✅ Confirmed |
| 23   | 0xFF    | Marker                           | ✅ Known    |
| 24   | 0x80    | **?**                            | ❓ Unknown  |
| 25   | 0x80    | **?**                            | ❓ Unknown  |
| 26   | 0xFF    | Marker                           | ✅ Known    |
| 27   | 0x80    | **?**                            | ❓ Unknown  |
| 28   | 0x22    | Power consumption (low byte)     | ✅ Known    |
| 29   | 0x00    | Power consumption (high byte)    | ✅ Known    |
| 30   | 0x01    | Power consumption offset         | ✅ Known    |
| 31   | 0x80/C0 | Telemetry mux (cycles during operation) | ⚠️ See below |
| 32   | 0x00/19 | Telemetry mux data               | ⚠️ See below |
| 33   | 0x00/83 | Telemetry mux data               | ⚠️ See below |
| 34   | 0x??    | Checksum                         | ✅ Confirmed |

### Bytes 31-33: Multiplexed Telemetry

> ⚠️ **THEORY - Multiplexed Data**: These bytes cycle through different patterns during operation. They do NOT appear to be simple status flags.

**Observed Patterns:**
| b31  | b32  | b33  | Interpretation |
|------|------|------|----------------|
| 0x80 | 0x00 | 0x00 | Idle/baseline |
| 0xC0 | 0xXX | 0xXX | Active operation (values vary) |

**Observations:**
- Values cycle through patterns even during steady-state operation
- Pattern suggests time-multiplexed telemetry (different data on each poll)
- May contain: compressor frequency, fan RPM, refrigerant pressure, or other sensor data
- Not suitable for status detection (too dynamic)

> ⚠️ **UNCONFIRMED**: The exact meaning of each mux slot is unknown. Further investigation with longer capture periods needed.

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

| Byte | Purpose                          |
|------|----------------------------------|
| 0    | Mode + Power                     |
| 1    | Target temperature × 2           |
| 2    | Mild dry                         |
| 3    | Fan speed                        |
| 4    | Swing (V + H)                    |
| 5    | Preset / NanoE-X / EcoNavi       |
| 6    | (unused in control)              |
| 7    | (unused in control)              |
| 8    | Eco mode                         |
| 9    | (unused in control)              |

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
| 0x40  | 0100 0000   | Idle/stopped             | ✅ Confirmed |
| 0x44  | 0100 0100   | Shutdown transition      | ✅ Confirmed (35 occurrences observed) |
| 0x48  | 0100 1000   | Startup (fan warming)    | ✅ Confirmed |
| 0x4C  | 0100 1100   | Running (full operation) | ✅ Confirmed |

**Bit Layout:**
```
0x4C = 0100 1100 - Running (fan + compressor)
0x48 = 0100 1000 - Startup (fan only, pre-compressor)
0x44 = 0100 0100 - Shutdown (compressor off, fan winding down)
0x40 = 0100 0000 - Idle (stopped)
       ││││ ││││
       ││││ │└┴┴─ Bits 0-2: (always 0 in observed states)
       ││││ └──── Bit 3 (0x08): Fan running
       │││└────── Bit 2 (0x04): System transitioning (startup/shutdown)
       │└┴─────── Bits 4-6: (always 010 in observed states)
       └───────── Bit 7: (always 0)
```

**State Transitions Observed:**
- **Startup**: 0x40 → 0x48 → 0x4C (idle → fan warming → full operation)
- **Shutdown**: 0x4C → 0x44 → 0x40 (running → fan winding down → idle)

**Power Correlation:**
| State | Typical Power | Description |
|-------|---------------|-------------|
| 0x40  | ~33W          | Standby/idle |
| 0x44  | ~33W          | Fan winding down (compressor off) |
| 0x48  | ~92W          | Fan only (pre-compressor) |
| 0x4C  | ~750W+        | Full operation (varies with load) |

**Startup Sequence Timing (observed):**
- 0x40 → 0x48: Immediate on mode change
- 0x48 → 0x4C: ~3 minutes (compressor startup delay)

✅ All 4 states confirmed via live testing (2025-12-12/13)

> ⚠️ **UNCONFIRMED - Defrost Theory**: Other bit patterns (e.g., 0x50, 0x54) may indicate defrost/deicing mode. Defrost cycles have not been observed yet. This remains speculation pending winter testing with active defrost cycles.

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

> ⚠️ **UNCONFIRMED - Baseline for Defrost Detection**: The following thermal data may help identify defrost cycles when they occur.

**Normal Heating Operation:**
| Measurement | Value | Source |
|-------------|-------|--------|
| Inflow temp (b18) | ~23°C | CN-CNT / Zigbee |
| Outflow temp | ~30°C | Zigbee sensor |
| Delta T | ~7°C | Calculated |
| Inflow humidity | ~30% | Zigbee |
| Outflow humidity | ~18% | Zigbee |

**Theory for Defrost Detection:**
- During defrost, outdoor unit reverses cycle (cooling coil)
- Indoor fan typically stops or slows
- Delta T should drop significantly (outflow ≈ inflow)
- Byte 12 may show different state (0x50, 0x54?)
- Monitor for these conditions during winter operation

## Power Consumption

| Byte | Description      |
|------|------------------|
| 28   | Low byte         |
| 29   | High byte        |
| 30   | Offset/scaling   |

```cpp
power_watts = (byte_28 + (byte_29 * 256)) - byte_30;
```

> ⚠️ **KNOWN ISSUE - Formula Inaccurate**: The above formula produces values ~100-150W lower than actual measured power (verified via Shelly power meter). The discrepancy exists across all operating states.

**Observed Discrepancy (Shelly meter vs CN-CNT):**
| CN-CNT Power | Shelly Actual | Difference |
|--------------|---------------|------------|
| ~600W        | ~750W         | ~150W      |
| ~500W        | ~620W         | ~120W      |
| Idle (~0W)   | ~33W          | ~33W       |

> ⚠️ **THEORY - Byte 30 Correlation**: Byte 30 appears to correlate with compressor frequency/load. When compressor ramps up, b30 increases. The formula may need revision to account for this relationship. More investigation needed.

> ⚠️ **UNCONFIRMED**: The baseline floor (~33W standby) is not reflected in CN-CNT readings. This may be intentional (measuring only variable load) or a formula issue.

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
- [ ] Monitor byte 12 during defrost cycle (look for 0x50, 0x54, or other patterns)
- [ ] Capture thermal data (delta T drop) during defrost
- [ ] Look for bit-level changes in byte 12

### Power Formula (MEDIUM PRIORITY)
- [ ] Investigate byte 30 correlation with compressor frequency
- [ ] Determine if ~100-150W offset is constant or variable
- [ ] Consider if baseline floor (33W) is intentionally excluded

### Unknown Bytes
- [ ] Byte 8: Always 0x00?
- [ ] Byte 9: Seen as 0x40 - purpose?
- [ ] Bytes 11-17: Static or changing?
- [ ] Bytes 24, 25, 27: Status or unsupported markers?

### Completed Investigations
- [x] Byte 12 state machine (4 states: 0x40, 0x44, 0x48, 0x4C)
- [x] Byte 18 = inlet temperature (confirmed via Zigbee)
- [x] Byte 20 = humidity (confirmed via Zigbee)
- [x] Byte 21 = byte 18 + 2°C (calculated display value)
- [x] Bytes 31-33 = multiplexed telemetry (not simple status)
- [x] Power formula discrepancy identified (~100-150W low)

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
