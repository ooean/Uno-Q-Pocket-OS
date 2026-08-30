# Pocket OS

A small touchscreen operating system/shell built around the **Arduino UNO Q**.

Pocket OS uses the UNO Q's STM32U585 microcontroller as the UI processor and its Qualcomm QRB2210 MPU for higher-level services such as the local LLM, filesystem access, networking, and web retrieval.

The UI is rendered directly on a **480×320 ST7796S SPI TFT** with an **XPT2046 resistive touchscreen**. The MCU and MPU communicate through Arduino's Router Bridge.

> **Status:** Experimental / Work in progress
> This project is primarily a personal embedded-systems project and is not intended to be production-ready.

## Features

* Touch-driven graphical UI
* 480×320 ST7796S SPI display support
* XPT2046 resistive touchscreen
* On-device launcher
* Local LLM assistant
* Streaming LLM responses
* Conversation archive
* Text-mode web browser
* Calculator
* Notes
* File browser
* Games
* Music player
* Clock / timer / alarm
* Settings and device status
* Wi-Fi/network status reporting from the MPU
* MCU ↔ MPU communication through Router Bridge
* Boot-time hardware diagnostics
* Configurable touchscreen deadzone
* ASCII bitmap font renderer
* Custom lightweight graphics primitives

## Hardware

### Main board

**Arduino UNO Q**

The MCU handles:

* Display rendering
* Touch input
* UI state
* Application navigation
* Hardware interaction
* Input gestures

The MPU handles:

* Local LLM inference
* Persistent storage
* Network access
* Browser fetching/parsing
* Conversation storage
* Notes storage

The UNO Q combines a Qualcomm QRB2210 MPU with an STM32U585 MCU.

### Display / touch

**SeengGreat SG-L4INCH-B shield**

* ST7796S
* 480×320
* SPI
* RGB565
* XPT2046 resistive touchscreen

Current display configuration:

```text
SPI clock: 20 MHz
Color: RGB565
Rotation: 3
Resolution: 480×320
```

### UNO Q connections

| Function      | Pin |
| ------------- | --: |
| LCD CS        | D10 |
| LCD DC        |  D8 |
| LCD RST       |  D7 |
| LCD Backlight |  D9 |
| Touch CS      |  D5 |
| Touch IRQ     |  D4 |
| Touch BUSY    |  D3 |
| SPI SCK       | D13 |
| SPI CIPO      | D12 |
| SPI COPI      | D11 |
| SD CS         |  D6 |

The LCD and XPT2046 share the SPI bus.

## Architecture

```text
                         Arduino UNO Q
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  STM32U585 MCU                         QRB2210 MPU          │
│  ─────────────                         ──────────           │
│                                                             │
│  ┌───────────────┐                    ┌─────────────────┐   │
│  │ UI / Router   │                    │ Python Runtime  │   │
│  │               │                    │                 │   │
│  │ Home          │                    │ Local LLM       │   │
│  │ Apps          │◄── Router Bridge ─►│ Network         │   │
│  │ Touch         │                    │ Browser         │   │
│  │ Display       │                    │ Files           │   │
│  │ Input         │                    │ Conversations   │   │
│  └───────┬───────┘                    └─────────────────┘   │
│          │                                                  │
│          │ SPI                                              │
│          ▼                                                  │
│  ┌───────────────────┐                                      │
│  │ SG-L4INCH-B       │                                      │
│  │                   │                                      │
│  │ ST7796S           │                                      │
│  │ XPT2046           │                                      │
│  └───────────────────┘                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

The MCU deliberately owns the display.

The MPU does **not** render a desktop or stream screenshots to the TFT. Instead, it sends structured data to the MCU, which renders the final interface locally.

This keeps touch interaction responsive and avoids sending full framebuffer data across the MCU/MPU bridge.

## Applications

### Assistant

A local LLM interface running through the UNO Q's MPU.

Features include:

* Streaming responses
* Conversation history
* New conversation
* Conversation drawer
* On-screen keyboard
* Persistent conversation archive

The current App Lab configuration uses:

```text
Qwen3.5-0.8B-Q4_0
```

The model is preloaded by the Python side so that opening the Assistant does not have to initiate the initial model load.

### Browser

Pocket OS uses a text-oriented browser rather than attempting to stream Chromium-rendered pixels.

The MPU:

1. Fetches the webpage
2. Parses the HTML
3. Extracts useful text and links
4. Wraps the content
5. Sends the resulting information to the MCU
6. The MCU renders it

This keeps the display protocol lightweight and allows the browser to remain interactive.

### Calculator

A standalone calculator running entirely on the MCU.

It does not require the MPU.

### Notes

A touchscreen text editor with persistent storage on the MPU.

### Files

Provides access to files exposed by the Linux side.

### Games

Small MCU-side games designed for the touchscreen interface.

### Music

The music application obtains its file information from the MPU and presents playback controls on the MCU UI.

### Clock

Provides:

* Clock
* Timer
* Alarm

The MCU receives wall-clock synchronization from the MPU.

### Settings

Provides device controls and diagnostics including:

* Backlight
* Network status
* LLM status
* Router Bridge status
* Touch debugging
* Touch deadzone visualization

## Navigation

Pocket OS does not use a conventional back button.

### Back gesture

Swipe from the left side of the screen toward the right.

The gesture is deliberately separated from normal application controls.

### Control Centre

Swipe down from the upper-right region of the screen.

The Control Centre provides quick access to system-level controls and status.

## Touchscreen

The XPT2046 touchscreen is polled rather than relying on PENIRQ.

This is intentional because the particular shield configuration used by the project has shown unreliable PENIRQ behavior.

Touch pressure is measured during boot to establish an idle baseline:

```text
idle Z + margin = press threshold
```

This avoids relying on a universal fixed pressure threshold.

### Touch calibration

Touch calibration values are stored in:

```text
sketch/touch.h
```

The following parameters define the mapping:

```cpp
TS_SWAP_XY
TS_RAW_X0
TS_RAW_X1
TS_RAW_Y0
TS_RAW_Y1
```

These values are hardware-specific and should be recalibrated if the display/touch hardware is changed.

## Display Driver

The ST7796S driver is intentionally lightweight and self-contained.

Located at:

```text
sketch/st7796s.h
```

It implements:

* Display initialization
* Rotation
* RGB565 colors
* Rectangle fills
* Screen clearing
* Circles
* Rectangles
* Character rendering
* Text rendering
* SPI window addressing

The graphics layer in:

```text
sketch/gfx.h
```

adds:

* Text alignment
* UI shapes
* Icons
* Application graphics
* Higher-level drawing helpers

The panel is treated as **write-only**. The renderer therefore avoids read-modify-write operations and keeps enough background information to render UI elements correctly.

## Graphics Architecture

The renderer is intentionally immediate-mode.

Applications call primitives such as:

```cpp
fillRect(...)
fillCircle(...)
drawText(...)
fillRoundRect(...)
```

There is no full framebuffer.

This significantly reduces RAM requirements and makes the renderer simple enough to run directly on the MCU.

The downside is that the application must carefully repaint regions when something visually destructive occurs.

## Router Bridge

The MCU and MPU communicate using Router Bridge events.

### MCU → MPU

Examples:

```text
sys_ack
llm_load
llm_prompt
llm_stop
chat_new
chat_list
chat_open
note_load
note_save
web_go
web_link
web_back
```

### MPU → MCU

Examples:

```text
sys_hello
llm_state
llm_token
llm_end
llm_note
note_chunk
net_state
chat_item
chat_line
web_line
web_end
```

Bridge callbacks do not directly render UI.

Instead, events are placed into an MCU-side event queue and processed from `loop()`.

This prevents asynchronous bridge callbacks from accessing the shared display/touch SPI bus while it is being used.

## Project Structure

```text
pocket-os/
├── app.yaml
├── README.md
│
├── python/
│   ├── main.py
│   ├── netctl.py
│   ├── prompts.py
│   └── system_prompt.txt
│
└── sketch/
    ├── sketch.ino
    ├── sketch.yaml
    │
    ├── board_pins.h
    ├── st7796s.h
    ├── touch.h
    ├── gfx.h
    ├── font5x7.h
    ├── theme.h
    ├── ui.h
    ├── home.h
    ├── apps.h
    ├── clock.h
    ├── control.h
    ├── bridge_link.h
    │
    ├── app_browser.h
    ├── app_calc.h
    ├── app_clock.h
    ├── app_files.h
    ├── app_games.h
    ├── app_llm.h
    ├── app_music.h
    ├── app_notes.h
    └── app_settings.h
```

## Adding an Application

Applications are implemented as headers in:

```text
sketch/app_*.h
```

An application normally provides:

```cpp
icon()
paint()
enter()
tick()
touch()
event()
```

It may additionally provide:

```cpp
leave()
ready()
```

The application is then registered in:

```text
sketch/apps.h
```

The launcher automatically incorporates registered applications into its grid.

## Building

This project is designed for the Arduino UNO Q and Arduino's Zephyr-based UNO Q platform.

The current sketch configuration uses:

```yaml
platforms:
  - platform: arduino:zephyr
```

and:

```yaml
libraries:
  - XPT2046_Touchscreen (1.4.0)
```

Arduino's current Zephyr core documentation lists the UNO Q board as `arduino:zephyr:unoq` and documents App Lab integration with the Zephyr core.

Install the required UNO Q / Arduino Zephyr support, install the XPT2046 library, then upload the sketch to the UNO Q.

The Linux-side Python application is deployed as part of the Pocket OS App Lab application.

## Current Limitations

Pocket OS is still experimental.

Known limitations include:

* ASCII-only built-in font
* No full framebuffer
* Limited bridge payload sizes
* Text-oriented browser
* Browser page size limits
* Limited LLM context
* Touchscreen deadzone caused by the current hardware
* No hardware-accelerated graphics
* Display updates are relatively expensive
* Some applications are still basic implementations
* Rendering artifacts can occur during some display updates

## Known Display Rendering Issue

Some UI elements can occasionally appear with uneven **red/blue stripe artifacts**.

The current display driver uses the Arduino bulk SPI API:

```cpp
SPI.transfer(buf, n);
```

for RGB565 pixel transfers.

The first diagnostic step is to disable the bulk transfer path:

```cpp
#define USE_BLOCK_TRANSFER 0
```

The driver then falls back to individual byte transfers.

If this eliminates the artifacts, the likely problem is in the bulk SPI transfer path rather than the graphics primitives.

Additional tests should include:

* 10 MHz SPI instead of 20 MHz
* Different `LCD_CHUNK_PIXELS` values
* Solid-color full-screen fills
* Solid-color rectangles
* Text-only rendering
* Logic-analyzer inspection of MOSI / SCK / DC / CS

The UNO Q's Arduino Zephyr core has had multiple SPI-related changes and fixes, so the exact core version should also be recorded when reproducing display issues.

## Development Priorities

Planned / recommended improvements:

1. Stabilize ST7796S SPI transfers
2. Investigate bulk SPI rendering artifacts
3. Add a more efficient transmit-only display path
4. Improve rendering performance
5. Add more robust clipping
6. Improve text rendering
7. Add additional applications
8. Improve persistent storage handling
9. Improve browser parsing
10. Add automated hardware/display diagnostics

## License

License information should be added before publishing the repository publicly.

---

**Pocket OS** is an experimental embedded UI project exploring what can be built by combining the Arduino UNO Q's MCU and MPU into a single small touchscreen computer.
