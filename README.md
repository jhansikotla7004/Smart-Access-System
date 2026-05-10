# Smart Access Security System
# Embedded Systems | Project 2: Specialist Embedded Systems Project
**Student:** Kotla Jhansi Lakshmi | **Student No:** A00046968


---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Learning Outcomes Addressed](#2-learning-outcomes-addressed)
3. [Hardware Components](#3-hardware-components)
4. [Circuit Schematic and Pin Map](#4-circuit-schematic-and-pin-map)
5. [Software Architecture](#5-software-architecture)
6. [HAL Documentation (LO5)](#6-hal-documentation-lo5)
7. [Keypad Matrix Scanning](#7-keypad-matrix-scanning)
8. [SPI Display Interface](#8-spi-display-interface)
9. [UART Debugging (LO4)](#9-uart-debugging-lo4)
10. [Buzzer Feedback](#10-buzzer-feedback)
11. [GPIO Conflict Resolution](#11-gpio-conflict-resolution)
12. [Debounce Implementation](#12-debounce-implementation)
13. [Testing and Results (LO4)](#13-testing-and-results-lo4)
14. [Ethical Considerations (LO7)](#14-ethical-considerations-lo7)
15. [Source Code Control (LO6)](#15-source-code-control-lo6)
16. [Future Improvements](#16-future-improvements)
17. [Build and Flash Instructions](#17-build-and-flash-instructions)
18. [References](#18-references)

---

## 1. Project Overview

This project implements a PIN-based electronic access control system on the **STM32 NUCLEO-L432KC** development board. The system accepts a digit sequence from a 3×4 matrix keypad, validates it against a stored PIN, and communicates the result to the user through two independent feedback channels: a graphical TFT display and an active buzzer.

The motivation for this project came from studying commercial building access systems and wanting to understand the embedded firmware behind them. At its core, such a system is a constrained real-time application — it needs to be deterministic, handle noisy hardware inputs (key bounce), manage shared resources (GPIO lines that serve multiple peripherals), and give immediate feedback so the user knows the system is responding.

**Key functional behaviour:**
- User enters digits on the keypad; each press is masked on screen as `*`
- After 3 digits, the PIN is evaluated
- **Correct PIN (1-1-1 for demonstration):** green screen, three confirmation beeps, UART log entry
- **Incorrect PIN:** red screen, error beep, attempt counter increments
- **Three wrong attempts:** system locks; only a hardware reset restores operation
- `*` key clears a partially entered PIN at any point

**Development environment:** PlatformIO + VS Code, STM32Cube HAL, arm-none-eabi-gcc, ST-LINK/V2 on-board debugger.

---

## 2. Learning Outcomes Addressed

| LO | Description | Where Addressed |
|----|-------------|-----------------|
| **LO3** | Evaluate suitable hardware components for an embedded system | [Section 3](#3-hardware-components) — component selection justification |
| **LO4** | Debug an embedded system using hardware and software debugging tools | [Section 9](#9-uart-debugging-lo4) (UART), [Section 13](#13-testing-and-results-lo4) (test cases) |
| **LO5** | Develop and document elements of a HAL with good documentation procedures | [Section 6](#6-hal-documentation-lo5) — full HAL function reference |
| **LO6** | Work with a source code control system | [Section 15](#15-source-code-control-lo6) — Git commit history |
| **LO7** | Discuss the ethical considerations of embedded systems | [Section 14](#14-ethical-considerations-lo7) |

---

## 3. Hardware Components

| Component | Part Number | Justification |
|-----------|-------------|---------------|
| Microcontroller | STM32L432KC (NUCLEO board) | ARM Cortex-M4 at 80 MHz; hardware SPI1 and UART2 on Port A; 256 KB Flash; on-board ST-LINK eliminates external programmer |
| TFT Display | ST7735 1.8" (128×160 RGB565) | SPI interface conserves GPIO; 3.3 V compatible; low pin count (5 control lines + SPI bus) |
| Keypad | 3×4 matrix, 7-wire | Passive component; 12 keys using only 7 GPIO lines via pair-scanning |
| Buzzer | Active, 3 V, ~2.3 kHz | Internal oscillator — no PWM or timer required; direct GPIO drive |
| Breadboard + wires | Standard 830-tie | Rapid prototyping; all connections solderless |

**Why not I2C for the display?**
The ST7735 is an SPI-only device. Even if an I2C display were available, SPI at 10 MHz delivers far higher pixel throughput than I2C at 400 kHz — critical for full-screen redraws. With 128×160 pixels at 16 bits each, I2C would take over 1 second per screen update; SPI completes it in approximately 20 ms.

---

## 4. Circuit Schematic and Pin Map

The circuit schematic is located at [`schematic/smart_access_schematic.png`](schematic/smart_access_schematic.png).

### Complete Pin Allocation

| Device | Signal | STM32 Pin | GPIO Config | Notes |
|--------|--------|-----------|-------------|-------|
| ST7735 TFT | VCC | 3.3 V | Power | — |
| ST7735 TFT | GND | GND | Ground | — |
| ST7735 TFT | CS | PA4 | Output PP | SPI chip select, software-controlled |
| ST7735 TFT | DC / A0 | PA6 | Output PP | Low = command, High = data |
| ST7735 TFT | RST | PA0 | Output PP | Pull LOW ≥10 ms to reset |
| ST7735 TFT | SCK | PA5 | AF5 (SPI1) | SPI1 clock |
| ST7735 TFT | SDA / MOSI | PA7 | AF5 (SPI1) | SPI1 master-out |
| ST7735 TFT | LED | 3.3 V | Power | Always-on backlight |
| Active Buzzer | + | PA1 | Output PP | HIGH = tone on |
| Active Buzzer | − | GND | Ground | — |
| Keypad | Wire 0 | PB0 | Dynamic | Pair-scan line |
| Keypad | Wire 1 | PB4 | Dynamic | Pair-scan line |
| Keypad | Wire 2 | PB5 | Dynamic | Pair-scan line |
| Keypad | Wire 3 | PB6 | Dynamic | Pair-scan line |
| Keypad | Wire 4 | PA8 | Dynamic | Pair-scan line |
| Keypad | Wire 5 | PA9 | Dynamic | Pair-scan line |
| Keypad | Wire 6 | PA10 | Dynamic | Pair-scan line |
| UART2 | TX | PA2 | AF7 (UART2) | 115200 baud → ST-LINK VCP |
| UART2 | RX | PA3 | AF7 (UART2) | Configured, not used in demo |

> **Note on keypad routing:** Wires 0–3 are on GPIOB deliberately. This keeps them away from the SPI1/UART2/TFT lines concentrated on GPIOA, which simplifies the GPIO conflict guard (see [Section 11](#11-gpio-conflict-resolution)).

---

## 5. Software Architecture

The firmware is structured as a **flat, modular single-file application** (`src/main.c`). A single infinite polling loop drives the state machine; there are no RTOS threads, no DMA, and no interrupt handlers beyond SysTick (HAL internal). This keeps the execution model simple and fully deterministic — important for a first embedded systems project where tracing bugs must be straightforward.

### State Machine

```
          ┌─────────────────────────────────────────────┐
          │                                             │
          ▼                                             │
   ┌─────────────┐   digit key    ┌──────────────────┐ │
   │    HOME     │──────────────► │   PIN ENTRY      │ │
   └─────────────┘                └──────────────────┘ │
          ▲                              │              │
          │                    3 digits? │              │
          │                    ┌─────────┴────────┐     │
          │                    ▼                  ▼     │
          │             ┌────────────┐    ┌──────────┐  │
          │             │  GRANTED   │    │  DENIED  │  │
          │             └────────────┘    └──────────┘  │
          │                    │                │       │
          └────────────────────┘     attempts<3 │       │
                                                └───────┘
                                     attempts=3
                                          │
                                          ▼
                                   ┌────────────┐
                                   │   LOCKED   │  (reset to exit)
                                   └────────────┘
```

### Module Breakdown

| Function | Responsibility |
|----------|----------------|
| `main()` | Init sequence, polling loop, state transitions |
| `Keypad_Read()` | Pair-scan all 7 wires, debounce, return ASCII key |
| `Set_Pin_Output_Low(idx)` | Dynamic GPIO reconfiguration — output LOW |
| `Set_Pin_InputPullup(idx)` | Dynamic GPIO reconfiguration — input pull-up |
| `Screen_Home/Pin/Granted/Denied/Locked/Cleared()` | TFT screen renderers, one per state |
| `Access_Granted()` | Grant sequence: screen + UART + beep pattern |
| `Access_Denied(attempts)` | Deny sequence: screen + UART + beep + lockout check |
| `Beep(ms)` | Drive PA1 HIGH for specified duration |
| `UART_Print(msg)` | Wrap `HAL_UART_Transmit` for null-terminated strings |
| `ST7735_Init()` | Hardware reset + command sequence for display |
| `ST7735_FillScreen(colour)` | Bulk SPI pixel fill |
| `ST7735_WriteCommand/Data()` | Low-level SPI byte transfer with DC line control |

---

## 6. HAL Documentation (LO5)

This section documents every HAL function used in the firmware, consistent with the documentation procedures described in the STM32Cube HAL user manual (UM1884) [3].

### `HAL_SPI_Transmit`
```c
HAL_StatusTypeDef HAL_SPI_Transmit(SPI_HandleTypeDef *hspi,
                                    uint8_t *pData,
                                    uint16_t Size,
                                    uint32_t Timeout);
```
**Used in:** `ST7735_WriteCommand()`, `ST7735_WriteData()`, `ST7735_FillScreen()`
**Configuration:** SPI1, master mode, CPOL=0/CPHA=0 (Mode 0), 8-bit, MSB first, prescaler /8 → ~10 MHz clock.
**Notes:** Called in polling mode (not DMA). The CPU blocks until the transfer completes, guaranteeing that SPI transactions do not overlap with keypad GPIO operations that share GPIOA.

### `HAL_UART_Transmit`
```c
HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart,
                                     uint8_t *pData,
                                     uint16_t Size,
                                     uint32_t Timeout);
```
**Used in:** `UART_Print()`
**Configuration:** USART2, 115200 baud, 8N1, no hardware flow control. PA2 (TX) routed through ST-LINK VCP to USB.
**Timeout:** 100 ms. Sufficient for strings up to ~1000 bytes at 115200 baud.

### `HAL_GPIO_WritePin`
```c
void HAL_GPIO_WritePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin, GPIO_PinState PinState);
```
**Used in:** TFT CS/DC/RST control, buzzer drive, keypad output drive.
**Notes:** Writes directly to BSRR register — atomic, glitch-free. Does not require disabling interrupts.

### `HAL_GPIO_ReadPin`
```c
GPIO_PinState HAL_GPIO_ReadPin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```
**Used in:** `Keypad_Read()` — checks whether a sense line was pulled LOW.
**Returns:** `GPIO_PIN_RESET` (0) when key is pressed; `GPIO_PIN_SET` (1) when floating high via pull-up.

### `HAL_GPIO_Init`
```c
void HAL_GPIO_Init(GPIO_TypeDef *GPIOx, GPIO_InitTypeDef *GPIO_Init);
```
**Used in:** `Set_Pin_Output_Low()`, `Set_Pin_InputPullup()`, `MX_GPIO_Init()`.
**Notes:** Called during runtime (not just at startup) to dynamically switch keypad pins between output-low and input-pullup modes. This is the core mechanism of the pair-scan algorithm.

### `HAL_Delay`
```c
void HAL_Delay(uint32_t Delay);
```
**Used in:** Debounce timing (50 ms, 30 ms), state hold durations (700 ms, 1200 ms, 1500 ms), TFT init sequence, buzzer patterns.
**Basis:** SysTick at 1 ms resolution, derived from 80 MHz system clock.

### `HAL_SPI_Init` / `HAL_UART_Init`
Standard peripheral initialisation functions called in `MX_SPI1_Init()` and `MX_USART2_UART_Init()`. The `Init` structure fields are documented in the pin map table above.

---

## 7. Keypad Matrix Scanning

A standard 3×4 matrix keypad has 12 keys arranged in 4 rows × 3 columns. The conventional scanning approach drives rows and reads columns (or vice versa), requiring separate knowledge of row/column pinout — something that varies between keypad manufacturers and is not always documented for generic modules.

**Pair-scanning** avoids this problem. With 7 wires, there are 7×6/2 = 21 possible pair combinations, which is more than enough to cover 12 keys. Each wire is driven LOW in turn (all others floating as pull-up inputs). If a key is pressed, it bridges two wires — the driven wire pulls the connected sense wire LOW, which `HAL_GPIO_ReadPin` detects.

```
Wire index:  0(PB0)  1(PB4)  2(PB5)  3(PB6)  4(PA8)  5(PA9)  6(PA10)
             │       │       │       │       │       │       │
             └──K1───┘       │       │       │       │       │
             └──K2───────────┘       │       │       │       │
             └──K3───────────────────┘  ...and so on for all 12 keys
```

The `keyPairs[7][7]` lookup table maps each detected `(drive, sense)` index pair to an ASCII character. This table was calibrated by probing each physical wire against every other wire while pressing each key.

### Why this works better than row/column for this project

- No external pull-up resistors needed (STM32 internal pull-ups used)
- No dependency on documented row/column pinout of the module
- Wires can be connected in any order and the table recalibrated
- Guard clause prevents accidental reconfiguration of SPI/UART pins

---

## 8. SPI Display Interface

The ST7735 controller accepts an 8-bit serial command stream over SPI. The interface has two modes distinguished by the **DC line (PA6)**:
- **DC LOW:** the byte is a command register address
- **DC HIGH:** the byte is data (pixel, parameter, or address value)

The CS line (PA4) is controlled in software rather than using the hardware NSS feature. This gives precise control over the CS/DC transition order, which the ST7735 datasheet requires to be stable before the first SCK edge.

**Screen fill calculation:**
128 × 160 pixels × 2 bytes/pixel (RGB565) = **40,960 bytes** per full screen.
At 10 MHz SPI (with HAL polling overhead), a full fill takes approximately **40–45 ms**.
Partial updates (e.g., only redrawing the asterisk row during PIN entry) complete in under **5 ms**, which is why `Screen_Pin()` redraws only one text row rather than calling `ST7735_FillScreen()` on every keypress.

---

## 9. UART Debugging (LO4)

UART2 transmits all system events to the PlatformIO serial monitor at 115200 baud. The virtual COM port (VCP) is provided by the on-board ST-LINK, so no USB-to-UART adapter is needed.

### Recorded Serial Output (Full Session)

```
Smart Access System Ready
Enter PIN: * * *
>> ACCESS GRANTED

Enter PIN: * * *
>> ACCESS DENIED (Attempt 1 of 3)

Enter PIN: * * *
>> ACCESS DENIED (Attempt 2 of 3)

Enter PIN: * * *
>> ACCESS DENIED (Attempt 3 of 3)
>> SYSTEM LOCKED
```

### Why UART matters for debugging

During development, UART was the primary tool for diagnosing keypad behaviour before the display driver was working. By printing the raw `(drive, sense)` indices on each key detection, it was possible to populate the `keyPairs` lookup table through systematic probing — essentially using the UART terminal as a live calibration tool.

UART also provides **independent verification** of correct operation that does not rely on visual observation of the display. For a test examiner reviewing the submission remotely, the UART log is objective evidence of state transitions.

---

## 10. Buzzer Feedback

The active buzzer on PA1 contains an internal oscillator and emits a continuous tone (~2.3 kHz) whenever the pin is held HIGH. The `Beep(ms)` function wraps `HAL_Delay` to produce timed pulses:

```c
void Beep(uint16_t ms) {
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);
    HAL_Delay(ms);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
}
```

Different auditory patterns are composed from multiple `Beep()` calls:

| Event | Pattern | Rationale |
|-------|---------|-----------|
| System startup | 1 × 200 ms | Long single beep — confirms boot |
| Key accepted | 1 × 80 ms | Brief, non-distracting click |
| Access granted | 3 × 80 ms (50 ms gap) | Positive triple-beep |
| Access denied | 2 × 100 ms (50 ms gap) | Distinct from granted |
| System locked | 5 × 100 ms (80 ms gap) | High-urgency alert |

Using an active buzzer (rather than a passive one driven by PWM) kept the circuit simple and eliminated the need for a hardware timer, which would have added another resource conflict to manage.

---

## 11. GPIO Conflict Resolution

The most significant hardware challenge was that the STM32L432KC has limited GPIO, and Port A is heavily contested:

| Pins | Used for |
|------|----------|
| PA0 | TFT RST |
| PA1 | Buzzer |
| PA2, PA3 | UART2 TX/RX |
| PA4 | TFT CS |
| PA5 | SPI1 SCK |
| PA6 | TFT DC |
| PA7 | SPI1 MOSI |
| PA8, PA9, PA10 | Keypad wires 4–6 |

The keypad pair-scan algorithm dynamically reconfigures GPIO pins between output and input modes. If this reconfiguration accidentally targeted PA5 (SCK) or PA7 (MOSI), it would disconnect the SPI bus mid-transaction, causing display corruption or a freeze.

**Solution — guard clause in `Set_Pin_Output_Low()` and `Set_Pin_InputPullup()`:**

```c
/* Protect SPI1/UART2/TFT-control pins on PA0–PA7 */
if (keyPorts[idx] == GPIOA && keyPins[idx] <= GPIO_PIN_7) return;
```

This check executes before any `HAL_GPIO_Init()` call on a keypad wire. Keypad wires 4–6 (PA8, PA9, PA10) are above PIN_7 and pass through. All GPIOB wires are unaffected. The three lines on PA8/PA9/PA10 are in the safe range — they are above the SPI/UART/TFT-control zone.

Routing wires 0–3 through GPIOB (PB0, PB4, PB5, PB6) was a deliberate wiring decision made precisely to minimise the number of keypad lines on GPIOA, reducing the guard clause burden.

---

## 12. Debounce Implementation

Mechanical keypad contacts bounce for 5–20 ms after a press or release. In early testing, a single key press registered as two or three events, corrupting the PIN entry counter.

**Two-stage debounce strategy:**

1. **Press confirmation:** On first detection of a line going LOW, wait 50 ms, then re-read. If still LOW, the press is confirmed.
2. **Release wait:** After processing the key, poll until the line returns HIGH, then wait an additional 30 ms.

```c
if (HAL_GPIO_ReadPin(keyPorts[r], keyPins[r]) == GPIO_PIN_RESET) {
    HAL_Delay(50);                        /* stage 1: settle */
    if (HAL_GPIO_ReadPin(keyPorts[r], keyPins[r]) == GPIO_PIN_RESET) {
        /* valid press — process key */
        while (HAL_GPIO_ReadPin(...) == GPIO_PIN_RESET) HAL_Delay(10);
        HAL_Delay(30);                    /* stage 2: post-release */
    }
}
```

This approach adds up to 80 ms of latency per keypress, which is imperceptible to a human user but reliably eliminates double-registration. Over 40+ test presses during verification, no double-registration was observed.

---

## 13. Testing and Results (LO4)

Testing followed an incremental integration strategy — each subsystem was verified in isolation before being combined with the others. This made faults much easier to isolate.

**Phase 1 — UART:** Startup message verified in PlatformIO serial monitor before any other peripheral was initialised.
**Phase 2 — Buzzer:** PA1 toggled at 1 Hz via `Beep(500)` in a loop; tone confirmed audibly.
**Phase 3 — Keypad:** Drive/sense indices printed via UART for each physical keypress; used to populate `keyPairs[][]`.
**Phase 4 — Display:** Solid colour fills confirmed correct SPI wiring and ST7735 init sequence.
**Phase 5 — Integration:** Full state machine exercised with all test cases below.

### Test Cases

| ID | Description | Expected | Observed | Method | Result |
|----|-------------|----------|----------|--------|--------|
| T01 | Power-on boot | UART prompt; home screen; startup beep | All three confirmed | Visual + UART | ✅ PASS |
| T02 | Single valid key press | One `*` on UART; one asterisk on TFT; pinCount = 1 | Matched exactly | Visual + UART | ✅ PASS |
| T03 | Correct PIN (1-1-1) | Green screen; three beeps; UART "ACCESS GRANTED" | All confirmed | Visual + UART | ✅ PASS |
| T04 | Incorrect PIN | Red screen; error beep; UART "DENIED (Attempt 1 of 3)" | Matched | Visual + UART | ✅ PASS |
| T05 | Three wrong PINs | LOCKED screen; rapid beep; UART "SYSTEM LOCKED"; input ignored | All confirmed | Visual + UART | ✅ PASS |
| T06 | `*` clear mid-entry | Yellow "PIN RESET" screen; UART "PIN cleared"; return to home | Correct | Visual + UART | ✅ PASS |
| T07 | SPI + keypad concurrency | No display corruption during keypad scan; no SPI bus errors | Guard clause effective; no corruption | Visual | ✅ PASS |
| T08 | Attempt counter accuracy | Counter increments 1→2→3 correctly; lockout triggers at 3 | Correct sequence | UART | ✅ PASS |
| T09 | Keypad debounce | Single physical press = single key event | No double-registration in 40 presses | UART | ✅ PASS |
| T10 | Post-grant recovery | After GRANTED, home screen returns; new PIN cycle accepted | New cycle completed correctly | Visual + UART | ✅ PASS |

---

## 14. Ethical Considerations (LO7)

Deploying an access control system in a real-world setting raises several engineering ethics obligations:

**PIN Security:** The current firmware stores the reference PIN as a plaintext constant in Flash memory. Anyone with a JTAG debugger and physical access to the board can read it. Production deployment would require hashing the PIN (e.g., SHA-256 with a random salt) before storage, so the stored value cannot be reversed.

**Lockout and Fail-Safe:** The three-attempt lockout is a security feature, but in a life-safety application (emergency egress, for example) a hard lock that requires hardware reset could trap occupants. Any production system would need a fail-safe bypass — a physical key override or a remote unlock signal — that operates independently of the firmware state.

**Accessibility:** A keypad-only system excludes users who cannot physically press small buttons due to motor impairment. Inclusive design would offer alternative authentication methods (NFC card, voice, proximity sensor).

**Data Privacy:** Even though this demonstrator has no network connectivity, any extension that logs access events to a cloud service must comply with data protection legislation (GDPR in the EU/Ireland). Access event logs — timestamps, user identifiers — constitute personal data under GDPR Article 4.

**Reliability and Liability:** A bug that grants access incorrectly, or refuses it to an authorised user, carries real-world consequences in a deployed system. This underlines why test coverage, code review, and fault-injection testing are not optional extras but ethical obligations for safety-critical embedded software.

---

## 15. Source Code Control (LO6)

This repository is maintained using Git. All development work was committed incrementally. Key commit milestones:

| Commit | Description |
|--------|-------------|
| `init` | Project scaffold — platformio.ini, empty main.c |
| `feat: uart-init` | UART2 initialised; startup message verified |
| `feat: buzzer` | Buzzer tone confirmed on PA1 |
| `feat: keypad-scan` | Pair-scan algorithm; keyPairs table calibrated |
| `feat: tft-init` | ST7735 hardware reset and init sequence |
| `feat: screen-states` | All 6 screen state functions implemented |
| `feat: state-machine` | Full PIN validation loop integrated |
| `fix: gpio-guard` | Added SPI pin protection to keypad scanner |
| `fix: debounce` | Two-stage debounce; resolved double-registration |
| `docs: readme` | This README completed for submission |

The full commit history is visible in the repository's commit log.

---

## 16. Future Improvements

**RFID / NFC authentication:** Adding an MFRC522 reader over SPI2 (or I2C) would allow contactless card authentication — faster, more hygienic, and scalable to multi-user whitelists stored in EEPROM.

**Fingerprint biometrics:** An AS608 UART fingerprint module would add a biometric second factor. Combined with PIN entry, this creates two-factor authentication aligned with NIST SP 800-63B [8] guidance for medium-assurance applications.

**Non-volatile PIN storage:** Currently the PIN is hardcoded. Using the STM32L432KC's Flash emulation for EEPROM (via `HAL_FLASH_Program()`) would allow the PIN to be changed without reflashing — a basic requirement for any real deployment.

**Interrupt-driven keypad:** Replacing the polling loop with GPIO interrupt (EXTI) on keypad lines would allow the MCU to enter low-power sleep between keypresses, reducing average current from ~10 mA to under 100 µA — essential for battery-powered installations.

**IoT event logging:** A UART-connected ESP8266 module could forward access events to an MQTT broker. Combined with a Node-RED or Grafana dashboard, this would provide a real-time access log accessible remotely — the core of any enterprise access control installation.

**Full keypad calibration:** The current demo is limited to PIN `1-1-1` because the pair-scan table was calibrated for this specific keypad wiring. Using a keypad with a verified, documented pinout (or a dedicated decoder IC such as the MM74C923) would enable arbitrary PINs and use of the `#` key as a confirm button — matching commercial PIN entry device behaviour.

---

## 17. Build and Flash Instructions

**Prerequisites:**
- [VS Code](https://code.visualstudio.com/) with [PlatformIO extension](https://platformio.org/install/ide?install=vscode)
- STM32 NUCLEO-L432KC connected via USB

**Build and upload:**
```bash
# Clone the repository
git clone https://github.com/<your-username>/smart-access-system.git
cd smart-access-system

# Build
pio run

# Flash to board
pio run --target upload

# Open serial monitor (115200 baud)
pio device monitor --baud 115200
```

**Expected serial output on first boot:**
```
Smart Access System Ready
Enter PIN:
```

---

## 18. References

[1] STMicroelectronics, *STM32L432KC Datasheet*, DS11451 Rev. 9, 2022.

[2] STMicroelectronics, *STM32L4 Series Reference Manual*, RM0394 Rev. 4, 2021.

[3] STMicroelectronics, *Description of STM32L4/L4+ HAL and Low-Layer Drivers*, UM1884 Rev. 8, 2022.

[4] PlatformIO Labs, *ST STM32 Platform Documentation*, 2024. [Online]. Available: https://docs.platformio.org/en/latest/platforms/ststm32.html

[5] Sitronix Technology Corporation, *ST7735 262K Colour Single-Chip TFT Controller/Driver Datasheet*, Rev. 1.4, 2010.

[6] ARM Limited, *Cortex-M4 Generic User Guide*, ARM DUI 0553B, 2014.

[7] J. W. Valvano, *Embedded Systems: Introduction to ARM Cortex-M Microcontrollers*, 5th ed., CreateSpace, 2015.

[8] National Institute of Standards and Technology, *Digital Identity Guidelines*, NIST SP 800-63B, 2020.

[9] MISRA Consortium, *MISRA C:2012 Guidelines for the Use of the C Language in Critical Systems*, 2013.

[10] Embedded Artistry, "Debouncing Buttons and Mechanical Switches," 2018. [Online]. Available: https://embeddedartistry.com/blog/2018/02/05/debouncing-buttons-and-mechanical-switches/

---

*Repository maintained as part of EENG1030 Embedded Systems, Project 2 submission.*
*Student: Kotla Jhansi Lakshmi | A00046968 | May 2026*
