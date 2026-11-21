# ESP32-CAM Waste Classification & Sorting System

An Arduino sketch for the ESP32-CAM (AI-Thinker / ESP-EYE) that runs a Edge Impulse–trained image classification/object-detection model on live camera frames to identify waste type, then sends a command over UART to a companion Arduino (e.g. controlling a sorting mechanism / servo bins).

## Overview

The device continuously:

1. Captures a JPEG frame from the onboard camera.
2. Converts it to RGB888 and resizes/crops it to the model's expected input dimensions.
3. Runs the Edge Impulse classifier (`run_classifier`) on the frame.
4. Reads the resulting label(s) — either raw classification scores or object-detection bounding boxes, depending on how the model was built.
5. If a label's confidence meets the threshold, sends a short command string over a hardware UART (`Serial2`) to a second microcontroller, which is presumably responsible for physically sorting the item.

## Waste Categories & Commands

|Detected Label|Command Sent|Meaning|
|---|:-:|---|
|`biodegradable`|`CMD-B`|Organic / compostable waste|
|`nonbiodegradable`|`CMD-NB`|Non-biodegradable waste|
|`recyclable`|`CMD-R`|Recyclable material|
|`specialwastes` / `special`|`CMD-P` or `CMD-SW`*|Special/hazardous waste|


## Hardware Requirements

- **ESP32-CAM** board (default configuration: **AI-Thinker** module with PSRAM). ESP-EYE pin definitions are also included but commented out.
- A second Arduino/microcontroller connected via UART to receive sorting commands.
- PSRAM is **required** — the sketch halts at boot if PSRAM is not detected.

### UART Wiring

|ESP32-CAM Pin|Function|
|---|---|
|GPIO 17|TX (to receiving Arduino's RX)|
|GPIO 16|RX (from receiving Arduino's TX)|

Communication uses `HardwareSerial` on UART2 at **115200 baud**.

## Software Requirements

- **Arduino IDE** with the **ESP32 Arduino Core v2.0.4** ([release link](https://github.com/espressif/arduino-esp32/releases/tag/2.0.4))
- An **Edge Impulse** exported Arduino library  ( `idiotbinmodel_inferencing.h`.
- The Edge Impulse SDK's image DSP module (`edge-impulse-sdk/dsp/image/image.hpp`)

## Configuration

Key settings near the top of the sketch:

```cpp
#define CAMERA_MODEL_AI_THINKER   // Select your camera board
#define confidence_threshold 0.90 // Minimum confidence to trigger a command
```

- Change `CAMERA_MODEL_AI_THINKER` to `CAMERA_MODEL_ESP_EYE` (uncomment the appropriate `#define`) if using a different board.
- Adjust `confidence_threshold` (0.0–1.0) to tune sensitivity vs. false positives.
- Camera settings (`frame_size`, `jpeg_quality`, `pixel_format`, etc.) are configured in the `camera_config` struct and default to `FRAMESIZE_QVGA` with JPEG output at quality level 12.

## Building & Uploading

1. Install the ESP32 Arduino Core (v2.0.4) via the Arduino Boards Manager.
2. Export your trained model from Edge Impulse as an **Arduino library** and add it to your Arduino `libraries` folder.
3. Update the `#include` at the top of the sketch to match your exported library's inferencing header.
4. Select the correct board (e.g. **AI Thinker ESP32-CAM**) and port in the Arduino IDE.
5. Upload the sketch. Open the Serial Monitor at **115200 baud** to view initialization messages and live predictions.

## Program Flow

```
setup()
 ├─ Initialize Serial (115200)
 ├─ Initialize camera (ei_camera_init)
 ├─ Verify PSRAM is present
 └─ Delay before starting inference

loop()
 ├─ Allocate snapshot buffer
 ├─ Capture + convert + resize frame (ei_camera_capture)
 ├─ Run classifier (run_classifier)
 ├─ Parse results:
 │   ├─ Object detection mode → iterate bounding boxes
 │   └─ Classification mode   → iterate label scores
 ├─ If confidence ≥ threshold → send_command() over UART
 └─ Free snapshot buffer
```



## License

Based on Edge Impulse's Arduino example code, © 2022 EdgeImpulse Inc., released under the MIT License.
