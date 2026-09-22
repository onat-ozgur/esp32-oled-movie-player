# 🎬 Watching The Odyssey on a 0.96" OLED (As Nolan Imagined It)

> *"Cinema belongs on the grandest screens... or a 0.96-inch monochrome I2C display."*
---

## 📽️ Project Overview

Standard microcontrollers struggle with real-time video decoding. This project circumvents that limitation by pre-processing clips into optimized raw monochrome frames and uncompressed PCM audio:

* **1-Bit Floyd-Steinberg Dithering:** Converts high-res movie frames into crisp 128×64 1-bit bitmaps, preserving cinematic contrast and fine visual textures.
* **Embedded Audio Stream:** Audio is downsampled to 8 kHz 8-bit mono and stored directly in the ESP32’s internal flash memory (`PROGMEM`).
* **Hardware DAC & Timer Playback:** The ESP32's built-in 8-bit DAC (GPIO 25) outputs audio via an 8000 Hz hardware timer interrupt, keeping soundtrack playback strictly in sync with frame rendering.
* **Overclocked I2C:** Runs the SSD1306 I2C bus at 800 kHz (Fast-Mode Plus) to deliver smooth 15–20 FPS video playback without an external SD card.

---

## ⚡ Hardware Requirements

| Component ||
| :--- | :---: |
| **ESP32 Dev Module** |
| **0.96" I2C OLED Display** |
| **Mini Speaker** |
| **Current-Limiting Resistor** | 220Ω|
| **Capacitor** | 100µF (DC-blocking filter) |

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

---

## 📂 Repository Structure

```text
├── demo.gif                   # Showcase preview animation
├── donusturucu.py             # Python Converter Script (MP4 -> video_data.h)
├── esp32_video/
│   ├── esp32_video.ino        # Main Arduino firmware
│   └── video_data.h           # Pre-converted Odyssey clip (Ready to flash)
└── README.md                  # Project documentation
```

---

## 🔄 Video Preprocessing & Asset Generation

Because the ESP32 lacks the compute power to decode MP4 on the fly, I implemented an offline preprocessing pipeline using Python (`OpenCV`, `NumPy`, and `MoviePy`):

* **Audio Extraction & Resampling:** The soundtrack is extracted, mixed down to mono, resampled to 8 kHz, and normalized to 8-bit unsigned values (`0–255`) suitable for direct DAC register writes.
* **Spatial Scaling & Dithering:** Each video frame is extracted at 15 FPS, scaled down to 128×64, and dithered using the Floyd-Steinberg error diffusion algorithm to simulate multi-tone grayscale on a monochrome 1-bit screen.
* **Bit-Packing & Header Generation:** Pixels are bit-packed using `numpy.packbits` (8 pixels per byte, exactly 1024 bytes per frame) and serialized directly into a C header (`video_data.h`) using the `PROGMEM` flash attribute.








https://github.com/user-attachments/assets/f7dcd32d-6e1f-4599-aaf7-add715ea5c9d








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
