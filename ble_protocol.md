# TrueGear Bluetooth Protocol Documentation

## Overview

This document describes the TrueGear SDK Bluetooth communication protocol, covering the complete flow from device connection to effect data transmission. The protocol is based on BLE, using GATT services for data transmission.


### Protocol Characteristics
- **Transmission**: BLE Write Without Response
- **Frame Structure**: Fixed header and tail, data fragmentation
- **Flow Control**: Queue management to prevent overload
- **Effects**: Vibration (Shake) and Electrical stimulation

---

## Connection Flow

### State Machine
```
Scanning → Connecting → Connected → Service Discovery → Ready
```

### Key Steps
1. **Scanning**: Filter devices with `Truegear_C` prefix
2. **Connection**: Establish GATT connection with 2-second auto-reconnect
3. **Service Discovery**: Find target UUIDs
4. **Initialization**: Send handshake frame, start queue

### Important Constants
| Item | Value | Description |
|---|----|------|
| Reconnect Interval | 2000ms | Auto-reconnect after disconnection |
| Send Check Interval | 70ms | Queue loop interval |
| Write Interval | 30ms | BLE write interval |

### UUIDs
| Type | UUID | Description |
|------|------|------|
| Service | `6e400001-b5a3-f393-e0a9-e50e24dcca9e` | Central service |
| Characteristic | `6e400002-b5a3-f393-e0a9-e50e24dcca9e` | Read/write characteristic |
| CCCD | `6e400003-b5a3-f393-e0a9-e50e24dcca9e` | Notification descriptor |

---

## Data Transmission

### Transmission Architecture
```
Effect Request → Byte Construction → Queue Aggregation → Fragmentation (≤128B) → BLE Write
```

### Queue Mechanism
- **Concurrent safe, aggregates raw bytes**
- **After fragmentation, ensures order**
- **Flow Control**: 70ms check + 30ms write to avoid overload

---

## Protocol Frame Format

### Frame Structure
```
┌─────────────┬──────────────┬─────────────┐
│ Header (2B) │ Data Area    │ Tail (1B)   │
│ 0x68 0x68   │ ...          │ 0x16        │
└─────────────┴──────────────┴─────────────┘
```

### Shake Frame (16B Data)
| Offset | Size | Field | Description |
|------|------|------|------|
| 0 | 1 | Type | 1=Const, 2=Fade, 3=Keep Const, 4=Keep Fade |
| 1 | 1 | AssetID | Keep mode ID |
| 2 | 2 | StartTime | Start time (ms, BE) |
| 4 | 2 | EndTime | End time (ms, BE) |
| 6 | 1 | StartI | Start intensity (0-255) |
| 7 | 1 | EndI | End intensity (0-255) |
| 8 | 8 | Bitmap | Front/back activation bitmap (4 groups × 2B) |

### Electrical Frame (16B Data)
| Offset | Size | Field | Description |
|------|------|------|------|
| 0 | 1 | Type | 0x10=Once, 0x11=Const, 0x12=Fade |
| 1 | 1 | - | 0x00 |
| 2 | 2 | StartTime | Start time (ms, BE) |
| 4 | 2 | EndTime | End time (ms, BE) |
| 6 | 1 | Interval | Pulse interval |
| 7 | 1 | - | 0x00 |
| 8 | 2 | StartI | Start intensity (× percentage) |
| 10 | 2 | EndI | End intensity (× percentage) |
| 12 | 4 | Bitmap | Front/back activation bitmap (2 groups × 2B) |

### Intensity Modes
| Mode | Shake Type | Electrical Type | Behavior |
|------|------------|-----------------|------|
| Const | 1/3 | 0x11 | Constant |
| Fade | 2/4 | 0x12 | Gradient |
| FadeInAndOut | 2+4 (Dual Frame) | 0x12 (Dual Segment) | Fade in and out |

---

## Device Index Mapping

### Layout Overview
- **Front**: Index 0-99 (5 rows × 2 columns)
- **Back**: Index 100-199 (5 rows × 2 columns)
- **Bitmap**: Each group maps to 16-bit bitmap

### Bitmap Calculation
Index → Group (1-4) → Bitmap Position → OR Combination

Example: Index 0 (a1) → Group 1 → Bit 0x8000 → Front Group 1

---

## Summary

| Aspect | Details |
|------|------|
| Connection | BLE GATT, auto-reconnect, service discovery |
| Transmission | Queue fragmentation, interval flow control, Write Without Response |
| Frame Format | Header/tail markers, fixed fields, bitmap encoding |
| Effects | Shake/Electrical, intensity adjustment, index mapping |
| Performance | MTU fragmentation, concurrent safety |

### Key Methods
- `connect(String address)`
- `PlayEffect_UUID(String uuid)`
- `Build_EffectObject(EffectObject obj)`
- `writeCharacteristicsData(byte[] bytes)`
