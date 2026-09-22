# 🎬 Watching The Odyssey on a 0.96" OLED (The Way Sir Nolan Intended)

> *"Cinema belongs on the grandest screens... or a 0.96-inch monochrome I2C display."*

<p align="center">
  <img src="demo.gif" alt="The Odyssey on ESP32 OLED Demo" width="420">
</p>

<p align="center">
  <b>128×64 1-Bit Dithered Video</b> • <b>8 kHz Synchronized DAC Audio</b> • <b>Pure ESP32 Hardware</b>
</p>

---

## 📽️ Project Overview

Standard microcontrollers struggle with real-time video decoding. This project circumvents that limitation by pre-processing clips into optimized raw monochrome frames and uncompressed PCM audio:

* **1-Bit Floyd-Steinberg Dithering:** Converts high-res movie frames into crisp 128×64 1-bit bitmaps, preserving cinematic contrast and fine visual textures.
* **Embedded Audio Stream:** Audio is downsampled to 8 kHz 8-bit mono and stored directly in the ESP32’s internal flash memory (`PROGMEM`).
* **Hardware DAC & Timer Playback:** The ESP32's built-in 8-bit DAC (GPIO 25) outputs audio via an 8000 Hz hardware timer interrupt, keeping soundtrack playback strictly in sync with frame rendering.
* **Overclocked I2C:** Runs the SSD1306 I2C bus at 800 kHz (Fast-Mode Plus) to deliver smooth 15–20 FPS video playback without an external SD card.

---

## ⚡ Hardware Requirements

| Component | Quantity | Notes |
| :--- | :---: | :--- |
| **ESP32 Dev Module** | 1 | Standard 30/38-pin board (ESP-WROOM-32 / ESP32-U) |
| **0.96" I2C OLED Display** | 1 | 128×64 SSD1306 driver (I2C address `0x3C`) |
| **Mini Speaker** | 1 | 8Ω (0.5W – 1W) |
| **Current-Limiting Resistor** | 1 | 100Ω – 220Ω (Series protection for DAC pin) |
| **Capacitor (Optional)** | 1 | 100µF electrolytic (DC-blocking filter) |

---

## 🔌 Circuit & Wiring

```text
          ESP32                     0.96" OLED (SSD1306)
     +---------------+             +--------------------+
     |       GPIO 21 |------------>| SDA                |
     |       GPIO 22 |------------>| SCL                |
     |          3.3V |------------>| VCC                |
     |           GND |------------>| GND                |
     +---------------+             +--------------------+

          ESP32                         SPEAKER
     +---------------+             +--------------------+
     |       GPIO 25 |----[ 100Ω ]-| (+)                |
     |           GND |-------------| (-)                |
     +---------------+             +--------------------+
```

> ⚠️ **Important:** Never connect an 8Ω speaker directly to ESP32 GPIO 25 without a current-limiting resistor (100Ω–220Ω). Excessive current draw can permanently damage the internal DAC driver.

---

## 📂 Repository Structure

```text
├── demo.gif                   # Showcase preview animation
├── donusturucu.py             # Python converter script (MP4 -> video_data.h)
├── esp32_video_player/
│   ├── esp32_video_player.ino # Main Arduino firmware
│   └── video_data.h           # Pre-converted Odyssey clip (Ready to flash)
└── README.md                  # Project documentation
```

---

## 🚀 Quick Start Guide

### Option A: Flash the Pre-Converted Clip (No Python Needed)

This repository includes a ready-to-test clip in `esp32_video_player/video_data.h`.

1. Open `esp32_video_player/esp32_video_player.ino` in the Arduino IDE.
2. Install the required libraries via the Library Manager:
   * **Adafruit SSD1306**
   * **Adafruit GFX Library**
3. Configure the flash partition scheme to accommodate video assets:
   * Navigate to: **Tools** > **Partition Scheme** > **Huge APP (3MB No OTA/1MB SPIFFS)**.
4. Select your ESP32 board and COM port, then click **Upload**.

---

### Option B: Convert Your Own Clip

To convert any custom video into embedded C headers:

1. Install Python dependencies:
   ```bash
   pip install opencv-python numpy moviepy
   ```
2. Place your video in the project root directory and name it `video.mp4` *(5–8 seconds recommended due to flash limits)*.
3. Run the conversion script:
   ```bash
   python donusturucu.py
   ```
4. Move the newly generated `video_data.h` into the `esp32_video_player/` directory (replacing the existing file).
5. Compile and flash the sketch to the ESP32.

---

## ⚙️ How It Works

```text
[ Source Video (MP4) ]
          │
          ├──> [ MoviePy ] ──> 8 kHz 8-Bit Audio ─────> PROGMEM audio_data[] ──> Timer ISR -> DAC (GPIO 25)
          │
          └──> [ OpenCV ]  ──> Dithering (128x64) ────> PROGMEM video_frames[] ─> I2C (800 kHz) -> SSD1306
```

* **Floyd-Steinberg Dithering:** Error diffusion spreads quantization artifacts across neighboring pixels, creating continuous perceived tonal values on a pure 1-bit monochrome display.
* **Interrupt-Driven DAC Audio:** An ESP32 hardware timer triggers at 8 kHz (every 125 µs) to push subsequent samples directly into the DAC register via `dacWrite(25, sample)`, preventing audio stutter.
* **Frame Pacing:** The main loop renders frames using `display.drawBitmap()` and enforces a strict microsecond pace, locking video frames to audio playback without clock drift.
