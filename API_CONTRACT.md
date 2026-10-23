# Solar Monitor Pro - API Contract & Hardware Specification

**Contract Version:** `1.2.0`  
**Target Platforms:** Windows PC Browsers (Google Chrome, Microsoft Edge), Native Android, ESP32 Firmware  
**Default Local Endpoint:** `http://192.168.4.1/api`

---

## 1. Overview & Data Philosophy

This specification defines the unified JSON telemetry and control interface between the **ESP32 Controller**, the **Solar Monitor Pro Web Application**, and the **Native Android Mobile Application**.

### Critical Telemetry Contract Rules
1. **Strict Nullability:** Missing, disconnected, uncalibrated, or out-of-range sensor channels MUST be returned as `null` (Unavailable), **NEVER as zero (`0`)**. Zero is a mathematically valid measurement (e.g. 0.0 W at night or 0.0 A when idle).
2. **Stale Threshold:** Clients must poll every `5000 ms` (5 seconds) while active and mark data as **stale** if the sample `timestamp` is older than `30000 ms` (30 seconds).
3. **CORS Headers:** The ESP32 web server MUST return:
   ```http
   Access-Control-Allow-Origin: *
   Access-Control-Allow-Methods: GET, POST, OPTIONS
   Access-Control-Allow-Headers: Content-Type, Authorization
   ```

---

## 2. Telemetry Ingestion Endpoint: `GET /api/telemetry`

### Response Payload Schema (`application/json`)

```json
{
  "device_id": "ESP32-SOLAR-001",
  "timestamp": 1729671600000,
  
  "solar_voltage_v": 34.25,
  "solar_current_a": 5.85,
  "solar_power_w": 200.36,
  "solar_energy_wh": 845.2,

  "battery_voltage_v": 26.54,
  "battery_current_a": 4.12,
  "battery_power_w": 109.34,
  "battery_soc_percent": 84.5,
  "battery_remaining_ah": 84.5,
  "battery_status": "charging",

  "dc_load_voltage_v": 24.18,
  "dc_load_current_a": 1.15,
  "dc_load_power_w": 27.81,
  "dc_load_energy_wh": 142.6,

  "ac_voltage_v": 230.8,
  "ac_current_a": 2.45,
  "ac_active_power_w": 525.0,
  "ac_energy_kwh": 4.15,
  "ac_frequency_hz": 50.02,
  "ac_power_factor": 0.94,

  "sensor_status": {
    "ina228_ch0_solar": "ok",
    "ina228_ch1_battery": "ok",
    "ina228_ch2_dc_load": "ok",
    "tca9548a_mux": "ok",
    "pzem004t_ac": "ok"
  },

  "ac_contactor_feedback": true,
  "ac_breaker_aux_status": true,
  "ac_control_status": "closed"
}
```

### Field Definitions & Types

| Field Name | Type | Unit | Nullable | Description |
|---|---|---|---|---|
| `device_id` | String | - | No | Unique hardware identifier of the ESP32 node. |
| `timestamp` | Number | ms | No | Epoch millisecond timestamp of measurement sample. |
| `solar_voltage_v` | Number | V | Yes | INA228 Channel 0 bus voltage (0.0 to 85.0 V). |
| `solar_current_a` | Number | A | Yes | INA228 Channel 0 shunt current (0.0 to 30.0 A). |
| `solar_power_w` | Number | W | Yes | INA228 Channel 0 real-time power (V × I). |
| `solar_energy_wh` | Number | Wh | Yes | Daily accumulated solar energy. |
| `battery_voltage_v` | Number | V | Yes | INA228 Channel 1 pack terminal voltage (20.0 to 29.2 V). |
| `battery_current_a` | Number | A | Yes | Signed current: `+` = Charging, `-` = Discharging. |
| `battery_power_w` | Number | W | Yes | Net pack power. |
| `battery_soc_percent`| Number | % | Yes | State of Charge (0.0 to 100.0%). |
| `battery_remaining_ah`| Number | Ah | Yes | Usable remaining Amp-hours based on 100Ah capacity. |
| `battery_status` | String | - | Yes | Enum: `'charging'`, `'discharging'`, `'idle'`, `'full'`, `'depleted'`. |
| `dc_load_voltage_v` | Number | V | Yes | INA228 Channel 2 DC bus voltage. |
| `dc_load_current_a` | Number | A | Yes | INA228 Channel 2 DC load current. |
| `dc_load_power_w` | Number | W | Yes | INA228 Channel 2 instantaneous power. |
| `dc_load_energy_wh` | Number | Wh | Yes | Daily accumulated DC energy consumed. |
| `ac_voltage_v` | Number | V | Yes | PZEM-004T v3.0 AC RMS line voltage (80 to 260 V). |
| `ac_current_a` | Number | A | Yes | PZEM-004T v3.0 100A CT RMS current (0.0 to 100.0 A). |
| `ac_active_power_w` | Number | W | Yes | PZEM-004T v3.0 real active power (0 to 23000 W). |
| `ac_energy_kwh` | Number | kWh | Yes | PZEM-004T non-volatile cumulative energy. |
| `ac_frequency_hz` | Number | Hz | Yes | PZEM-004T AC frequency (45.0 to 65.0 Hz). |
| `ac_power_factor` | Number | - | Yes | PZEM-004T displacement factor (0.00 to 1.00). |
| `sensor_status` | Object | - | No | Health map: `'ok'`, `'fault'`, `'offline'`. |
| `ac_contactor_feedback` | Boolean | - | Yes | Dry-contact auxiliary microswitch feedback (`true` = closed, `false` = open, `null` = UNCONFIRMED). |
| `ac_breaker_aux_status` | Boolean | - | Yes | Physical DIN-rail circuit breaker auxiliary contact. |
| `ac_control_status` | String | - | No | Enum: `'closed'`, `'open'`, `'tripped'`, `'command_pending'`, `'unknown'`. |

---

## 3. AC Contactor Control Endpoint: `POST /api/control/ac-disconnect`

Dispatches safety-critical open/close commands to the high-voltage contactor.

### Request Headers
```http
Content-Type: application/json
Authorization: Bearer <SECURE_AUTH_TOKEN>
```

### Request Body
```json
{
  "action": "open",
  "target_device": "ESP32-SOLAR-001",
  "auth_token": "SOLAR_SECURE_TOKEN_2026",
  "timestamp": 1729671600000
}
```

### Safety Requirements
1. The ESP32 MUST verify the authorization token before driving the relay GPIO.
2. The controller MUST verify that no fault trip condition is active prior to closing.
3. The response MUST confirm receipt, while verified contactor feedback is updated on subsequent telemetry polls via GPIO auxiliary dry contacts.

### Success Response (`200 OK`)
```json
{
  "success": true,
  "action": "open",
  "message": "AC Contactor command 'OPEN' accepted by controller.",
  "timestamp": 1729671600050
}
```

---

## 4. Hardware I2C / Serial Bus Topology

```
+-------------------------------------------------------------------+
|                        ESP32 DevKit v1                            |
|                                                                   |
|   GPIO21 (SDA) ---+               GPIO16 (RX2) <--- PZEM-004T TX   |
|   GPIO22 (SCL) ---|               GPIO17 (TX2) ---> PZEM-004T RX   |
|                   |                                               |
|                   v                                               |
|            +--------------+                                       |
|            |   TCA9548A   |                                       |
|            |   I2C MUX    |                                       |
|            | (Addr 0x70)  |                                       |
|            +-------+------+                                       |
|                    |                                              |
|      +-------------+-------------+                                |
|      | Port 0      | Port 1      | Port 2                         |
|      v             v             v                                |
|  [INA228 #0]   [INA228 #1]   [INA228 #2]                          |
|  Solar PV      Battery Bank  DC System Load                       |
|  (0x40)        (0x41)        (0x44)                               |
+-------------------------------------------------------------------+
```

---

## 5. Historical Query & Logging Endpoints

To support long-term trend analysis across Web and Native Android platforms without combining AC and DC data streams, the backend and firmware provide timestamped history logging endpoints.

### Query History: `GET /api/telemetry/history`
- **Query Parameters:**
  - `start` (number, ms timestamp): Start boundary of query window
  - `end` (number, ms timestamp): End boundary of query window
  - `limit` (number, optional, max 2000): Maximum number of chronological samples to return
- **Response (`application/json`):**
```json
{
  "success": true,
  "count": 96,
  "start": 1729600000000,
  "end": 1729686400000,
  "samples": [
    {
      "timestamp": 1729600000000,
      "ac_voltage_v": 230.4,
      "ac_current_a": 1.85,
      "ac_active_power_w": 395.0,
      "ac_energy_kwh": 3.45,
      "ac_frequency_hz": 50.02,
      "ac_power_factor": 0.93,
      "solar_voltage_v": 33.8,
      "solar_current_a": 4.5,
      "solar_power_w": 152.1,
      "solar_energy_wh": 620.0,
      "battery_voltage_v": 26.5,
      "battery_current_a": 1.2,
      "battery_power_w": 31.8,
      "battery_soc_percent": 82.0,
      "battery_remaining_ah": 82.0,
      "dc_load_voltage_v": 24.2,
      "dc_load_current_a": 1.1,
      "dc_load_power_w": 26.6,
      "dc_load_energy_wh": 115.0
    }
  ]
}
```

### Critical AC/DC Electrical Separation Rule
- **No Joint Currents:** PZEM-004T AC Current (A) and INA228 DC Current (A) are measured at different reference points. They MUST NEVER be combined or summed into a single series.
- **No Joint Energy:** AC energy (kWh) and DC energy (Wh/kWh) MUST NOT be added together into a single total because solar energy passing through the inverter to AC loads would be double-counted.

