---
title: Vinny Panchal Datasheet · TerraGuard
tags:
- terraguard
- phoenix-force
- vinny
- esp32
---

<center>
<font size="6">Vinny Panchal Datasheet</font><br>
as part of<br>
<font size="8">TerraGuard</font><br>
for<br>
<font size="5">Team 315 · Phoenix Force · EGR314</font><br>

**Submission: December 01, 2025**
</center>

## Introduction

TerraGuard is a two-board wildfire-response rover plus handheld screen that creates its own Wi‑Fi access point, streams MJPEG video, and surfaces gas/air-quality and proximity alerts. This datasheet captures Vinny’s individual contributions: screen-side firmware and UI/UX, communication flow, PCB pinout strategy, and integration.

## Project Summary

The rover (ESP32-CAM) hosts the AP, reads BME688, MQ-2, ultrasonic, GPS, and MPU6050, and drives an L298N. The commander screen (CrowPanel ESP32-S3) joins that AP, polls telemetry, displays live MJPEG via LVGL, mirrors alerts through LEDs/audio, and sends drive commands back over HTTP. My work centered on the screen firmware, UI, comms polling, and connector strategy so off-board parts remain serviceable.

Full team context lives in the [team report](https://egr314-2025-f-315.github.io/phoenixforce.github.io/).

## My Contribution

- **HMI firmware (ESP32-S3)**: LVGL UI layout, RGB panel config (800×480), MJPEG sprite pipeline, telemetry polling, error-handling, and HTTP drive commands.
- **UX and enclosure intent**: Tablet-like feel, translucent back so LEDs stay visible when handheld, base geometry that stands on a table for hands-free viewing.
- **Comms shaping**: AP join + retry cadence, metric/coord polling, stream watchdog, and command POSTs back to the rover.
- **Hardware integration**: Screen-side pin mapping, connectors for off-board sensors/LEDs, and power routing awareness to keep 5 V/3.3 V rails clean.

## Hardware Artifacts

- **Schematic (rover)**:  
  `https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/schematic.png`
- **PCB layout and renders (rover)**:  
  `https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/pcb_kicad_pcb_editor.png`  
  `https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/PCB_FRONT.png`  
  `https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/PCB_BACK.png`
- **CAD renders**:  
  Rover: `https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/Rover_model.png`  
  Screen: `https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/SCREEN_RENDER.png`

## Software Approach

Two core codebases run the system.

### Rover firmware (ESP32-CAM · `sketch_nov24a.ino`)

- Hosts Wi‑Fi AP and serves:
  - `/metrics`: BME688 temp/humidity + MQ-2 gas resistance mapped to air-quality labels.
  - `/coords`: GPS lat/lon/satellite count (GGA parse).
  - `/control`: HTTP POST body with comma-separated drive commands.
- Drives L298N motors; preheats/calibrates MQ-2 at boot; streams MJPEG.

```cpp
// Metrics endpoint: package BME + MQ-2 into JSON
void handleMetrics() {
  bme_read(bme);
  float mq2Rs = mq2ReadRs();
  String gasType = classifyGas(mq2Rs, digitalRead(MQ2_D_PIN));
  String airQuality = classifyAir(mq2Rs);
  String json = "{\"temp_c\":" + String(bme.tempC)
              + ",\"humidity\":" + String(bme.humidity)
              + ",\"gas_kohms\":" + String(mq2Rs)
              + ",\"gas_type\":\"" + gasType + "\""
              + ",\"air_quality\":\"" + airQuality + "\"}";
  server.send(200, "application/json", json);
}

// Control endpoint: parse "2 forward, left" into motor actions
void handleControl() {
  if (!server.hasArg("plain")) return server.send(400, "text/plain", "Empty");
  processCommandString(server.arg("plain")); // splits on commas, maps to drive helpers
  server.send(200, "text/plain", "OK");
}
```

```cpp
// Motor helpers (L298N)
void driveForward(int ms) { setMotors(HIGH, LOW, HIGH, LOW); delay(ms); stopMotors(); }
void turnLeft(int ms)     { setMotors(LOW, HIGH, HIGH, LOW); delay(ms); stopMotors(); }
// ...similar for driveBackward/turnRight
```

### Screen firmware (CrowPanel ESP32-S3 · `TerraGuard_HMI.ino`) — Vinny

- Configures RGB panel (800×480) via LovyanGFX; LVGL renders telemetry cards, drive pad, and status bars.
- Polls rover for metrics/coords, ingests MJPEG into an LVGL sprite, shows link/stream health.
- Sends drive POSTs back to `/control`; NeoPixel LEDs mirror link/alert states; audio cues on warnings.
- UI sized for gloves; translucent back lets LEDs show through when handheld or on-table.

```cpp
// Display + LED setup
LGFX lcd; // RGB bus + panel cfg for 800x480
Adafruit_NeoPixel strip(LED_COUNT, LED_PIN, NEO_GRB + NEO_KHZ800);

// Periodic polling loop
fetchMetrics();  // GET /metrics -> update LVGL labels/colors
fetchCoords();   // GET /coords -> GPS text
pollStream();    // MJPEG -> sprite -> LVGL image widget

// Drive commands from UI buttons
void sendDrive(const char* cmd) {
  HTTPClient http;
  http.begin(CONTROL_URL);
  http.POST(cmd); // e.g., "forward", "left"
  http.end();
}
```

```cpp
// Link/LED feedback (simplified)
void setLinkState(bool ok) {
  if (ok) strip.fill(strip.Color(0, 150, 50));
  else    strip.fill(strip.Color(200, 40, 40));
  strip.show();
}
```

## Power and Connectors (Context)

- Input: 7.4–7.8 V pack → L7805CV (5 V) → LM3940 (3.3 V) for logic/sensors.
- Breakouts: camera, GPS, MQ-2, BME688, LEDs, motors exposed on headers to mount off-board while keeping the PCB within 100×100 mm.
- LEDs on both boards mirror alerts so states are visible from the rover and the handheld screen.

## Links

- Full resources (code, schematics, PCB, CAD):  
  `https://egr314-2025-f-315.github.io/phoenixforce.github.io/resources/#code`
- Team report:  
  `https://egr314-2025-f-315.github.io/phoenixforce.github.io/`

## Individual Assignments

### Block Diagram

Vinny mapped rover electronics, power, and HMI into a clear three-block split to keep pins, rails, and comms obvious while staying under 100×100 mm.

![Block Diagram](https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/Block_Diagram.png)

### Component Selection

Component picks emphasize ruggedness and clarity: ESP32-CAM for vision/control, ESP32-S3 screen for LVGL + LEDs, BME688 + MQ-2 for air quality, NEO-6M GPS, L298N for drive, WS2812B for feedback, and 7.4 V pack with L7805CV → LM3940 rails. Full BOM lives in the project resources.

### Schematic Design

Single-page rover schematic with ESP32-S3-U1 at the core, L7805CV → LM3940 rails, L298N motor driver, and clearly labeled headers for camera, GPS, MQ-2, BME688, WS2812B, and battery. Fuses and connectors are placed near the source to protect downstream loads, and UART/GPIO lines are broken out for service/debug. Power notes are called out on the sheet to align with the PCB copper pours and trace widths.

![Schematic](https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/schematic.png)

### PCB Design

Board routed under 100×100 mm with short, wide traces on power, star-like fan-out from regulators, and segregated sensor headers along the edges for easy off-board mounting. Camera and GPS headers are spaced to minimize crosstalk; WS2812B and motor connections sit near their power entry. Copper pours tie the 5 V and 3.3 V planes cleanly, with silkscreen callouts to reduce wiring mistakes. Mounting holes and connector spacing match the mechanical CAD so the board seats cleanly in the chassis.

![PCB Layout](https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/pcb_kicad_pcb_editor.png)

![PCB Front](https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/PCB_FRONT.png)

![PCB Back](https://egr314-2025-f-315.github.io/phoenixforce.github.io/assets/PCB_BACK.png)
