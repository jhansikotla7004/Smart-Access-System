# 🔐 Smart Access Security System

**EENG1030 — Embedded Systems | Project 2: Specialist Embedded Systems Project**

| Field | Detail |
|---|---|
| **Student Name** | Kotla Jhansi Lakshmi |
| **Student Number** | A00046968 |
| **Microcontroller** | STM32 NUCLEO-L432KC |
| **Development Environment** | VS Code + PlatformIO + STM32Cube HAL |
| **Submission Date** | 10 May 2026 |
| **Module** | EENG1030 Embedded Systems |

---

## 📋 Project Overview

A fully functional **PIN-based door access control system** built on the STM32 NUCLEO-L432KC (ARM Cortex-M4). The system integrates four embedded subsystems:

- 🔢 **Matrix Keypad** — 7-pin pair-scanning algorithm for PIN entry
- 🖥️ **ST7735 TFT Display** — SPI-driven 1.8" graphical interface with 5 screen states
- 🔔 **Active Buzzer** — GPIO-controlled audio feedback for all system events
- 📡 **UART2 Serial Monitor** — 115200 baud debug and audit logging

---

## 🎯 Learning Outcomes Addressed

| LO | Description | How Met |
|---|---|---|
| **LO3** | Evaluate suitable hardware components | MCU, display, keypad, and buzzer selection justified in report |
| **LO4** | Debug using hardware/software tools | UART serial output + register-level debugging documented in test cases |
| **LO5** | Develop and document HAL elements | HAL-based drivers for SPI, UART, GPIO with inline documentation |
| **LO6** | Use source code control | This Git repository with commit history |
| **LO7** | Discuss ethical considerations | Section 16 of the technical report |

---

## 🔧 Hardware

### Pin Allocation Table

| Device | Signal | STM32 Pin | Description |
|---|---|---|---|
| ST7735 TFT | SCK | PA5 | SPI1 clock |
| ST7735 TFT | SDA/MOSI | PA7 | SPI1 data |
| ST7735 TFT | CS | PA4 | Chip select (active low) |
| ST7735 TFT | DC | PA6 | Data/Command select |
| ST7735 TFT | RST | PA0 | Hardware reset |
| Active Buzzer | + | PA1 | GPIO push-pull output |
| Keypad | Pin 1 | PB0 | Pair-scan line 1 |
| Keypad | Pin 2 | PB4 | Pair-scan line 2 |
| Keypad | Pin 3 | PB5 | Pair-scan line 3 |
| Keypad | Pin 4 | PB6 | Pair-scan line 4 |
| Keypad | Pin 5 | PA8 | Pair-scan line 5 |
| Keypad | Pin 6 | PA9 | Pair-scan line 6 |
| Keypad | Pin 7 | PA10 | Pair-scan line 7 |
| UART2 | TX | PA2 | Serial debug output (115200 baud) |
| UART2 | RX | PA3 | Serial receive (configured) |

> **Schematic:** See [`schematics/`](./schematics/) folder.

---

## 💻 Software Architecture

The firmware uses a **polling-based finite state machine** with five states:

```
IDLE → PIN_ENTRY → ACCESS_GRANTED
                 → ACCESS_DENIED → (counter < 3) → IDLE
                                 → (counter ≥ 3) → SYSTEM_LOCKED
```

### Module Breakdown

| File | Responsibility |
|---|---|
| `src/main.c` | Main loop, state machine, access logic |
| `src/keypad.c` | Pair-scan algorithm, debounce, GPIO dynamic reconfiguration |
| `src/display.c` | ST7735 SPI driver, screen state renderers |
| `src/buzzer.c` | GPIO toggle buzzer tone generation |
| `src/uart.c` | UART2 transmit wrapper |

---

## 🖥️ TFT Display States

| State | Screen Content | Trigger |
|---|---|---|
| `SCREEN_HOME` | "WELCOME / ENTER PIN" | Startup, after granted/cleared |
| `SCREEN_PIN` | "VERIFYING / \*\*\*" (masked) | Each valid digit pressed |
| `SCREEN_GRANTED` | "ACCESS GRANTED" (green) | Correct 3-digit PIN |
| `SCREEN_DENIED` | "ACCESS DENIED / TRY X OF 3" (red) | Wrong PIN |
| `SCREEN_LOCKED` | "SYSTEM LOCKED / RESET BOARD" (red) | 3 failed attempts |
| `SCREEN_CLEARED` | "CLEARED" (yellow) | `*` key pressed |

---

## 🧪 Testing

Ten structured test cases were defined and executed. Full results in [`docs/`](./docs/).

| Test | Description | Expected | Result |
|---|---|---|---|
| TC01 | Correct PIN entry | ACCESS GRANTED + 3 beeps | ✅ Pass |
| TC02 | Wrong PIN entry | ACCESS DENIED + 2 long beeps | ✅ Pass |
| TC03 | PIN clear with `*` | CLEARED screen + HOME | ✅ Pass |
| TC04 | 3 consecutive wrong entries | SYSTEM LOCKED | ✅ Pass |
| TC05 | Masked display during entry | Only `*` shown | ✅ Pass |
| TC06 | UART log on correct PIN | `>>> ACCESS GRANTED <<<` | ✅ Pass |
| TC07 | UART log on wrong PIN | `>>> ACCESS DENIED <<<` + count | ✅ Pass |
| TC08 | Startup beep sequence | Double beep on boot | ✅ Pass |
| TC09 | Locked state prevents input | No response to keypad | ✅ Pass |
| TC10 | Attempt counter resets on success | Counter back to 0 | ✅ Pass |

---

## 📡 UART Debug Output

Connect at **115200 baud, 8N1** via the NUCLEO USB Virtual COM Port.

```
================================
     SMART ACCESS SYSTEM
================================
Enter PIN: * * *

>>> ACCESS GRANTED <<<
Enter PIN: * *

>>> ACCESS DENIED <<<
Wrong attempt: 1/3
Enter PIN: ...

!!! SYSTEM LOCKED !!!
Reset board to restart.
```

---

## 🔊 Buzzer Feedback Summary

| Event | Pattern |
|---|---|
| System startup | Double beep (120ms each) |
| Key pressed | Short beep (80ms) |
| Access granted | Triple beep (140ms × 3) |
| Access denied | Double long beep (400ms × 2) |
| System locked | Continuous 300ms beep loop |

---

## ⚙️ Build & Flash

### Requirements
- [PlatformIO](https://platformio.org/) (VS Code extension)
- ST-LINK USB driver
- `arm-none-eabi-gcc` toolchain (managed by PlatformIO)

### Steps
```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/smart-access-system.git
cd smart-access-system

# Build
pio run

# Upload to NUCLEO board
pio run --target upload

# Open serial monitor (115200 baud)
pio device monitor --baud 115200
```

---

## ⚖️ Ethical Considerations

| Concern | Mitigation |
|---|---|
| **Data Privacy** | PIN is never stored in flash or transmitted in plaintext; masked on display |
| **Fail-Safe** | System locks after 3 failed attempts, preventing brute-force attacks |
| **Accessibility** | Audio feedback provided for users with visual impairment |
| **Reliability** | Debounce implementation prevents false key detections |
| **Liability** | System is a demonstrator prototype; not recommended for production without EEPROM PIN storage and physical tamper protection |

---

## 🚀 Future Improvements

- RFID / NFC card authentication
- Fingerprint biometric sensor integration
- EEPROM non-volatile PIN storage
- IoT monitoring and remote management via MQTT
- Mobile application integration over Bluetooth
- Multi-user PIN support

---

## 📁 Repository Structure

```
smart-access-system/
├── README.md                   ← This file (main documentation)
├── platformio.ini              ← PlatformIO build configuration
├── src/
│   └── main.c                  ← Full firmware source

├── schematics/
│   └── schematic_smart access png   ← Circuit schematic description & pin map
         
```

---

## 📚 References

- STMicroelectronics, *STM32L432KC Datasheet*, DS11451, Rev 5, 2021.
- STMicroelectronics, *STM32Cube HAL and LL Drivers Reference Manual*, UM1884, Rev 8, 2022.
- Sitronix, *ST7735S TFT LCD Controller Datasheet*, Ver 1.1, 2010.
- Mazidi, M.A. & Naimi, S., *STM32 ARM Programming for Embedded Systems*, MicroDigitalEd, 2017.

---

*EENG1030 Embedded Systems — Technological University Dublin*


  

