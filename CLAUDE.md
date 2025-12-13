# CLAUDE.md - Project Context

## Project Overview

This repository documents the reverse-engineering of the Panasonic AC CN-CNT serial protocol. The CN-CNT port is found on Panasonic air conditioners and is typically used by the CZ-TACG1 WiFi adapter module.

## Key Files

- `Panasonic-CN-CNT-Protocol-v1.md` - Main protocol specification document

## Technical Summary

### Serial Connection
- **Baud**: 9600
- **Data bits**: 8
- **Parity**: Even
- **Stop bits**: 1
- **Device**: /dev/ttyAMA0 (Raspberry Pi)

### Protocol Basics
- **Poll command**: 13 bytes, header 0x70
- **Poll response**: 35 bytes, header 0x70
- **Control command**: 13 bytes, header 0xF0
- **Poll interval**: 5 seconds
- **Checksum**: Sum of all bytes = 0x00

### Confirmed Fields (Poll Response)
| Byte | Purpose |
|------|---------|
| 2 | Mode + Power state |
| 3 | Target temperature (×2) |
| 5 | Fan speed |
| 6 | Swing position (V+H) |
| 12 | Operational status (0x40/0x44/0x48/0x4C) |
| 18 | Inlet temperature |
| 19 | Outside temperature |
| 20 | Humidity % |
| 21 | Display temperature (byte 18 + 2) |
| 28-30 | Power consumption |

### Byte 12 State Machine
- 0x40 = Idle/stopped
- 0x44 = Shutdown transition
- 0x48 = Startup (fan warming)
- 0x4C = Running (full operation)

## Known Issues

1. **Power formula inaccurate**: CN-CNT reports ~100-150W less than actual (verified via Shelly meter)
2. **Bytes 31-33**: Multiplexed telemetry, not status flags - cycles during operation

## Unconfirmed / Needs Investigation

- Defrost detection (byte 12 may show 0x50, 0x54, or other patterns)
- Exact meaning of bytes 31-33 mux slots
- Byte 30 correlation with compressor frequency

## Data Collection (on Raspberry Pi)

Scripts in `/home/sebby/debug/`:
- `monitor.py` - CN-CNT polling (5s)
- `shelly.py` - Power meter (5s)
- `sensors.py` - Zigbee temp/humidity (60s)
- `acunit.py` - Home Assistant climate entity (60s)

Status scripts accept `--time` argument (e.g., `--time 5m`, `--time 1h`):
- `status.py`, `shelly-status.py`, `sensors-status.py`, `acunit-status.py`

Database: `/home/sebby/debug/ac_data.db` (SQLite)

## Validation Sources

| Data | Source |
|------|--------|
| Mode, temp, fan, swing | Sensibo/IR control |
| Inlet temp (b18) | Zigbee sensor at AC intake |
| Humidity (b20) | Zigbee sensor at AC intake |
| Outside temp (b19) | Panasonic Comfort Cloud app |
| Power consumption | Shelly power meter |

## Working With This Project

When making changes to the protocol document:
1. Mark confirmed items with checkmark
2. Mark theories/unconfirmed with warning symbol and clear "UNCONFIRMED" or "THEORY" label
3. Update the version table at the top
4. Update the TODO section if completing investigations
