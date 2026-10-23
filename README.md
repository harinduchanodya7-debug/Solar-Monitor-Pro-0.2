# Solar Monitor Pro - ESP32 Solar Energy Monitoring & Control System

A high-precision, responsive web application engineered for **Windows PC browsers (Google Chrome & Microsoft Edge)** and cross-platform devices to monitor and control an **ESP32-based solar energy system**.

---

## 1. System Architecture & Hardware Stack

- **Microcontroller:** ESP32 DevKit v1 (Dual-Core 240MHz, Wi-Fi 802.11 b/g/n)
- **I2C Bus Multiplexer:** TCA9548A 8-channel I2C switch (Address `0x70`)
- **DC Telemetry Monitors (x3):** Texas Instruments INA228 85V, 20-bit precision bidirectional I2C power monitors
  - **Channel 0:** Solar PV Array (100 W to 300 W configurable)
  - **Channel 1:** Battery Bank (24V nominal, 100 Ah LiFePO4 battery)
  - **Channel 2:** DC System Load (24V native auxiliary circuits)
- **AC Inverter Monitor:** PZEM-004T v3.0 AC power meter with 100A split-core CT via UART2 (GPIO16/17)
- **Power Inverter:** 230V AC pure sine wave inverter, 3000 W rated continuous output
- **Safety Interlock:** High-current 230V AC contactor with auxiliary dry-contact feedback microswitch

---

## 2. Windows PC Browser Integration (Chrome & Edge)

### Browser Security Realities: Mixed-Content & Local Network Access
Modern Chromium-based browsers on Windows (Google Chrome and Microsoft Edge) restrict pages loaded over `https://` from fetching unencrypted local LAN addresses such as `http://192.168.4.1/api/telemetry` directly (**Mixed-Content Restriction** and **Private Network Access preflights**).

Solar Monitor Pro solves this through three deployment options:

### Deployment Option 1: Full-Stack Local Server (Recommended for Windows PC)
Run the application locally on your Windows PC over HTTP:
```bash
npm install
npm run dev
```
- Open `http://localhost:3000` in Chrome or Edge.
- Because the origin is local HTTP, the browser can communicate with local LAN IPs without mixed-content restrictions.
- In addition, the included Node.js server provides a built-in proxy route:
  `/api/proxy?endpoint=http://192.168.4.1/api/telemetry`

### Deployment Option 2: Host Static Dashboard on ESP32 Flash
Build the optimized static bundle and flash it directly to the ESP32’s SPIFFS or LittleFS filesystem:
```bash
npm run build
```
Upload the `/dist` directory files to the ESP32 using the Arduino or ESP-IDF LittleFS plugin. Connect your PC to the `SolarMonitor-AP` Wi-Fi network and open `http://192.168.4.1/`. The UI and API run on the exact same origin—zero CORS, zero Mixed Content!

### Deployment Option 3: Remote Cloud Telemetry (Firebase / HTTPS Gateway)
When the ESP32 is connected to home Wi-Fi or cellular IoT, configure the telemetry ingestion endpoint:
`POST /api/telemetry/ingest`
Authenticate requests using Bearer tokens and view telemetry from any PC worldwide.

---

## 3. Connecting to the ESP32 Wi-Fi Access Point on Windows 10/11

1. Click the **Network / Wi-Fi icon** in the Windows Taskbar (bottom right).
2. Select the network: **`SolarMonitor-AP`**.
3. Enter the default security key: **`solar1234`** and click Connect.
4. Windows will automatically receive an IP address via DHCP:
   - Your PC: `192.168.4.2`
   - ESP32 Gateway: `192.168.4.1`
5. Open Chrome or Edge and navigate to `http://192.168.4.1/` (or `http://localhost:3000` with the local proxy).

---

## 4. Application Pages

1. **Dashboard:** High-level system overview, real-time energy flow diagram, summary metric cards, sensor health matrix, quick alerts.
2. **Live Monitoring:** 20-bit INA228 Ch0/1/2 live bus & shunt readings, TCA9548A multiplexer diagnostics, and PZEM-004T AC metrics.
3. **Battery:** 24V 100Ah LiFePO4 bank management, SOC %, cell voltage plateau curve (8S 20.0V–29.2V), runtime estimate, BMS cutoffs.
4. **Solar PV:** Configurable 100W–300W array tracking, daily Wh total, peak power, irradiance utilization bar.
5. **DC Load:** Channel 2 DC load analysis, consumption breakdown, energy Wh accumulator.
6. **AC Load:** PZEM-004T v3.0 measurements (230V, A, W, kWh, 50Hz, power factor cos φ, apparent VA, reactive VAR, inverter % load).
7. **Energy History:** 1-hour, 24-hour, 7-day, 30-day interactive SVG charts, generation vs consumption comparison, CSV export.
8. **Alerts & System Protection:** Active hardware alarms, threshold configurator, event logs, acknowledgment.
9. **Device Connection:** Wi-Fi AP instructions, mixed-content diagnostic tool, latency ping tester, custom IP configurator, local proxy toggle.
10. **AC Disconnect:** Safety contactor control, mandatory confirmation modal, verified contactor feedback vs measured AC voltage, unconfirmed state warnings, trip lockout.
11. **Settings & API Diagnostics:** Hardware ratings, polling settings, API Contract Schema viewer, raw JSON validator, demo physics simulator.

---

## 5. AC Contactor Safety Architecture

- **Auxiliary Contact Feedback:** The application distinguishes between the commanded state (`ac_control_status`), the verified mechanical contactor microswitch (`ac_contactor_feedback`), and the measured AC output voltage (`ac_voltage_v`).
- **Unconfirmed Warning:** If auxiliary dry contacts are not wired or missing, the state is prominently flagged as **`UNCONFIRMED`**.
- **Fault Trip Lockout:** After an electrical trip or overload, software reconnection is locked out.
- **Physical Safety Rule:** Software switching is not a substitute for a mechanical circuit breaker. Always lock out physical breakers before servicing.

---

## 6. Testing & Validation

Run unit tests for telemetry parsing, strict nullability, and calculations:
```bash
npm test
```
Result:
```
--- Starting Telemetry Parser Unit Tests ---
✅ PASSED: Missing solar_voltage_v is null, not 0
✅ PASSED: Missing contactor feedback is null, not false
✅ PASSED: Full payload parses with isValid=true
✅ PASSED: Stale timestamp detected (>30s)
--- All Unit Tests Completed Successfully ---
```
