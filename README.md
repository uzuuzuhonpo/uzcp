# UZCP 1.0 Specification
**Universal Zeta Coffee Protocol — Specification 1.0.0**

---

## 1. Overview

UZCP (Universal Zeta Coffee Protocol) is an open, JSON-based communication protocol designed for full automation of coffee roasting. It enables interoperability between all devices involved in the roasting process — controllers, roasters, thermometers, robot arms, and more.

### Design Principles

- **Simple** — JSON text-based; implementable in any language or environment
- **Bidirectional** — Symmetric communication between controllers and roasting devices
- **Transport-agnostic** — Supports Wi-Fi, USB, Bluetooth, and serial connections
- **Extensible** — Custom fields freely added using the `x_` prefix without breaking compatibility
- **Automation-ready** — One protocol from manual operation to fully autonomous robot roasting

### Naming

The "Zeta" in UZCP is inspired by the **Riemann Zeta Function** (ζ(s)) — one of the most profound functions in mathematics, known for its ability to describe an infinitely complex system in a unified way. Just as the Zeta function brings order to the seemingly chaotic distribution of prime numbers, UZCP aims to bring a single, unified protocol to the fragmented landscape of coffee roasting devices.

---

## 2. Terminology

| Term | Definition |
|------|------------|
| Controller | The host that controls and monitors the roast (PC, smartphone, UZU ROASTER, etc.) |
| Device | An endpoint device such as a roaster, thermometer, or robot |
| Profile | A roast profile defining a target temperature curve over time |
| RoR | Rate of Rise — temperature increase rate in ℃/min |
| Charge | Loading green beans into the drum |
| FC | First Crack |
| SC | Second Crack |
| Drop | Ejecting the roasted beans (end of roast) |

---

## 3. Transport Layer

UZCP is transport-agnostic. Each implementation MUST support at least one of the following transports.

### 3.1 Wi-Fi (HTTP REST)

```
Base URL: http://{device_ip}/uzcp/v1/
Content-Type: application/json
```

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/status` | Get device status |
| GET | `/telemetry` | Get latest telemetry snapshot |
| POST | `/command` | Send a command |
| GET | `/profile` | Retrieve the current profile |
| POST | `/profile` | Write a profile to the device |

### 3.2 Wi-Fi (WebSocket)

```
ws://{device_ip}/uzcp/v1/ws
```

After connection, both sides exchange messages in real time. Recommended for telemetry streaming.

### 3.3 USB / Serial

```
Baud rate : 115200
Line ending: \n (LF)
Encoding  : UTF-8
```

Each message is one JSON object per line, terminated by LF.

### 3.4 Bluetooth (BLE)

- Service UUID: `UZCP-SERVICE-UUID` (defined by implementer)
- Characteristics: TX (Controller → Device) / RX (Device → Controller)
- MTU: up to 512 bytes per packet; split into multiple packets if exceeded

---

## 4. Message Structure

Every UZCP message shares the following common header.

```json
{
  "uzcp": "1.0",
  "type": "<message_type>",
  "id": "<uuid_or_sequential_int>",
  "ts": 1700000000.123,
  "src": "<sender_id>",
  "dst": "<receiver_id>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uzcp` | string | ✅ | Protocol version (e.g. `"1.0"`) |
| `type` | string | ✅ | Message type (see below) |
| `id` | string | ✅ | Message identifier (UUID recommended) |
| `ts` | float | ✅ | UNIX timestamp in seconds (fractional allowed) |
| `src` | string | ✅ | Sender device ID |
| `dst` | string | ✅ | Destination device ID (`"*"` for broadcast) |

---

## 5. Message Types

### 5.1 telemetry (Device → Controller)

Streams real-time roasting state from the device.

```json
{
  "uzcp": "1.0",
  "type": "telemetry",
  "id": "a1b2c3d4",
  "ts": 1700000060.5,
  "src": "roaster-01",
  "dst": "*",
  "data": {
    "bt": 185.3,
    "et": 210.7,
    "ror": 8.2,
    "elapsed": 360,
    "phase": "maillard",
    "drum_rpm": 60,
    "airflow": 75,
    "heater": 80,
    "fc_detected": false,
    "sc_detected": false
  }
}
```

| Field | Type | Unit | Description |
|-------|------|------|-------------|
| `unit` | string | — | Temperature unit: `"celsius"` / `"fahrenheit"` / `"kelvin"`. If omitted, Celsius (℃) is assumed. |
| `bt` | float | — | Bean Temperature |
| `et` | float | — | Environmental (exhaust) Temperature |
| `ror` | float | ℃/min | Rate of Rise |
| `elapsed` | int | sec | Seconds since roast start |
| `phase` | string | — | Current roast phase (see below) |
| `drum_rpm` | int | rpm | Drum rotation speed (if supported) |
| `airflow` | int | % | Airflow level (0–100) |
| `heater` | int | % | Heater power (0–100) |
| `fc_detected` | bool | — | First Crack detection flag |
| `sc_detected` | bool | — | Second Crack detection flag |

**Phase Definitions:**

| phase | Description |
|-------|-------------|
| `idle` | Standby |
| `preheat` | Preheating |
| `charge` | Immediately after green bean charge |
| `drying` | Drying phase |
| `maillard` | Maillard reaction phase |
| `fc` | During First Crack |
| `development` | Development phase (FC to Drop) |
| `sc` | During Second Crack |
| `drop` | Beans ejected |
| `cooling` | Cooling |

---

### 5.2 command (Controller → Device)

Sends a control command to a device.

```json
{
  "uzcp": "1.0",
  "type": "command",
  "id": "b2c3d4e5",
  "ts": 1700000065.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "cmd": "set_heater",
  "params": {
    "value": 85
  }
}
```

**Command Reference:**

| cmd | params | Description |
|-----|--------|-------------|
| `start` | `{ "profile_id": "..." }` | Start roasting |
| `stop` | — | Emergency stop |
| `drop` | — | Trigger bean ejection |
| `set_heater` | `{ "value": 0–100 }` | Set heater power (%) |
| `set_airflow` | `{ "value": 0–100 }` | Set airflow level (%) |
| `set_drum_rpm` | `{ "value": 0–200 }` | Set drum rotation speed |
| `set_profile` | `{ "profile_id": "..." }` | Switch active profile |
| `charge` | `{ "weight_g": 200 }` | Charge green beans (robot use) |
| `preheat` | `{ "target_et": 200 }` | Begin preheat to target ET |
| `custom` | `{ "key": "...", "value": "..." }` | Custom command |

---

### 5.3 ack (Device → Controller)

Acknowledges receipt of a command.

```json
{
  "uzcp": "1.0",
  "type": "ack",
  "id": "c3d4e5f6",
  "ts": 1700000065.1,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "b2c3d4e5",
  "status": "ok",
  "message": ""
}
```

| status | Description |
|--------|-------------|
| `ok` | Command accepted |
| `error` | Error occurred (`message` contains details) |
| `unsupported` | Command not supported by this device |

---

### 5.4 profile (Bidirectional)

Transfers a roast profile between controller and device.

```json
{
  "uzcp": "1.0",
  "type": "profile",
  "id": "d4e5f6a7",
  "ts": 1700000000.0,
  "src": "controller-01",
  "dst": "roaster-01",
  "profile": {
    "id": "ethiopia-natural-light",
    "name": "Ethiopia Natural Light Roast",
    "version": "1.0",
    "charge_temp_et": 200,
    "charge_weight_g": 200,
    "steps": [
      { "elapsed": 0,   "bt_target": 80,  "heater": 100, "airflow": 50, "drum_rpm": 60 },
      { "elapsed": 60,  "bt_target": 110, "heater": 90,  "airflow": 50, "drum_rpm": 60 },
      { "elapsed": 180, "bt_target": 150, "heater": 80,  "airflow": 60, "drum_rpm": 60 },
      { "elapsed": 300, "bt_target": 175, "heater": 70,  "airflow": 70, "drum_rpm": 60 },
      { "elapsed": 420, "bt_target": 195, "heater": 60,  "airflow": 80, "drum_rpm": 60 },
      { "elapsed": 480, "bt_target": 200, "heater": 50,  "airflow": 90, "drum_rpm": 60, "event": "drop" }
    ],
    "fc_auto_detect": true,
    "drop_bt": 200,
    "drop_ror": 3.0
  }
}
```

---

### 5.5 status (Device → Controller)

Returns the current state and capabilities of the device.

```json
{
  "uzcp": "1.0",
  "type": "status",
  "id": "e5f6a7b8",
  "ts": 1700000000.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "device": {
    "id": "roaster-01",
    "name": "UZU ROASTER UZ-01",
    "firmware": "3.1.0",
    "uzcp_version": "1.0",
    "capabilities": ["telemetry", "profile", "set_heater", "set_airflow", "fc_detection"],
    "state": "idle",
    "uptime": 3600
  }
}
```

---

### 5.6 event (Device → Controller)

Notifies the controller of a significant roasting event.

```json
{
  "uzcp": "1.0",
  "type": "event",
  "id": "f6a7b8c9",
  "ts": 1700000370.0,
  "src": "roaster-01",
  "dst": "*",
  "event": "fc",
  "data": {
    "bt": 196.2,
    "et": 218.5,
    "elapsed": 370
  }
}
```

**Event Reference:**

| event | Description |
|-------|-------------|
| `charge` | Green beans charged |
| `fc` | First Crack detected |
| `sc` | Second Crack detected |
| `drop` | Beans ejected |
| `alarm` | Anomaly detected (e.g. over-temperature) |
| `profile_end` | Profile sequence completed |
| `cooling_done` | Cooling cycle complete |

---

## 6. Robot Roasting Extension

Extended specification for full automation and robotic roasting.

### 6.1 robot_status (Robot → Controller)

```json
{
  "uzcp": "1.0",
  "type": "robot_status",
  "id": "a7b8c9d0",
  "ts": 1700000000.0,
  "src": "robot-01",
  "dst": "controller-01",
  "robot": {
    "state": "idle",
    "hopper_weight_g": 1500,
    "last_charge_g": 200,
    "cycles_today": 5
  }
}
```

### 6.2 Robot Commands

| cmd | params | Description |
|-----|--------|-------------|
| `charge` | `{ "weight_g": 200 }` | Charge specified weight of green beans |
| `collect` | `{ "bin": "light" }` | Collect roasted beans into the specified bin |
| `clean` | — | Run drum cleaning sequence |
| `standby` | — | Move robot to standby position |

---

## 7. Error Handling

### 7.1 Error Codes

| code | Description |
|------|-------------|
| `E001` | Unknown message type |
| `E002` | Missing required field |
| `E003` | Value out of range |
| `E004` | Device busy |
| `E005` | Invalid profile |
| `E006` | Over-temperature alarm |
| `E007` | Communication timeout |
| `E999` | Other / unspecified error |

```json
{
  "uzcp": "1.0",
  "type": "ack",
  "id": "...",
  "ts": 1700000000.0,
  "src": "roaster-01",
  "dst": "controller-01",
  "ref_id": "...",
  "status": "error",
  "error_code": "E003",
  "message": "heater value out of range: 150"
}
```

---

## 8. Extension Fields

Fields prefixed with `x_` are reserved for custom use. They MUST NOT conflict with standard fields and MUST NOT break protocol compatibility.

```json
{
  "x_roast_level": "light",
  "x_bean_origin": "Ethiopia Yirgacheffe",
  "x_operator": "robot-arm-01"
}
```

---

## 9. Versioning

| Version | Notes |
|---------|-------|
| 1.0.0 | Initial release. Core telemetry, commands, profiles, events, and robot extension. |

---

## 10. Conformance Requirements

- **MUST** implement: `telemetry`, `command`, `ack`, `status`
- **SHOULD** implement: `event`, `profile`
- **MAY** implement: Robot extension, `x_` custom fields

A device claiming UZCP 1.0 conformance MUST implement all MUST-level message types.

---

## 11. Contributing

UZCP is an open protocol. Proposals for new message types, commands, or extensions are welcome via the GitHub repository.

- Spec Repository: [github.com/uzuuzuhonpo/uzcp](https://github.com/uzuuzuhonpo/uzcp) *(coming soon)*
- Issues & Discussions: open to the community

---

*UZCP 1.0.0 — Designed by Uzuuzu Coffee Roastery*

*UZU ROASTER is the world's first UZCP-compatible device.*
