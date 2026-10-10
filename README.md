# smart-stackmatch-controller
9A5CW stackmatch controller
# SJ2W StackMatch Controller (Arduino Nano)

A high-performance, contest-grade SJ2W StackMatch antenna controller firmware and hardware design powered by an **Arduino Nano**. It features independent RX/TX profiles, flexible and exclusive receiver modes, real-time visual feedback, and advanced RF safety measures without requiring any external I/O expanders.

---

## 🚀 Key Features

* **Independent RX/TX Profiles:** Listen on one antenna (or a combination) while transmitting on another. Profiles automatically switch the moment PTT is engaged.
* **Dual RX Modes (Toggle via MOD Button):**
  * **Exclusive Mode:** Traditional single-antenna selection (selecting an antenna automatically deselects the others).
  * **Flexible Mode:** Combinatorial antenna selection (freely mix RX A, B, or C).
* **Independent TX Fade (Breathing Effect):** During transmission, active TX LEDs smoothly pulse (fade) independently based on which antennas are actually selected in the TX profile.
* **Smart Visual Mode Confirmation:** Pressing the RX MOD button causes all three red TX LEDs to blink **once** (Exclusive Mode) or **twice quickly** (Flexible Mode).
* **Anti-Hot-Switching (Tail-Delay):** An 80ms post-PTT lockout keeps buttons disabled briefly after releasing PTT, allowing external relays enough time to drop safely.
* **PTT Lockout:** Front-panel buttons are completely locked while PTT is active to prevent dangerous RF switching.
* **Safety Fallback (Anti-Open-Load):** If an operator accidentally deselects all TX antennas, the controller automatically defaults to **Antenna A** to protect the transmitter from an unmatched open load.
* **Hardware Watchdog Timer:** A built-in 2-second WDT automatically resets the Arduino if heavy RF interference causes a lockup in the shack.

---

## 📌 Pin Mapping (Arduino Nano)

All 20 available pins (`D2-D13`, `A0-A6`) are fully allocated, leaving `A7` free.

| Pin | Type | Function / Component | Connection Details |
| :--- | :--- | :--- | :--- |
| **D2** | Digital Output | StackMatch Relay 1 | Via ULN2003 (OUT1) to external Relay 1 |
| **D3** | Digital Output | StackMatch Relay 2 | Via ULN2003 (OUT2) to external Relay 2 |
| **D4** | Digital Output | StackMatch Relay 3 | Via ULN2003 (OUT3) to external Relay 3 |
| **D5** | Digital Input | RX Button A | Pushbutton to **GND** (uses internal pull-up) |
| **D6** | Digital Input | RX Button B | Pushbutton to **GND** (uses internal pull-up) |
| **D7** | Digital Input | RX Button C | Pushbutton to **GND** (uses internal pull-up) |
| **D8** | Digital Input | TX Button A | Pushbutton to **GND** (uses internal pull-up) |
| **D9** | Digital Input | TX Button B | Pushbutton to **GND** (uses internal pull-up) |
| **D10** | Digital Input | TX Button C | Pushbutton to **GND** (uses internal pull-up) |
| **D11** | Digital Input | PTT Line | PC817 Optocoupler output (Active LOW = Transmit) |
| **D12** | Digital Input | RX ALL / STACK Button | Pushbutton to **GND** (Selects all 3 RX antennas) |
| **D13** | Digital Input | TX ALL / STACK Button | Pushbutton to **GND** (Selects all 3 TX antennas) |
| **A0** | Analog/Digital Output | RX LED A (Green) | Via $220\ \Omega$ resistor to LED $\rightarrow$ GND |
| **A1** | Analog/Digital Output | RX LED B (Green) | Via $220\ \Omega$ resistor to LED $\rightarrow$ GND |
| **A2** | Analog/Digital Output | RX LED C (Green) | Via $220\ \Omega$ resistor to LED $\rightarrow$ GND |
| **A3** | PWM Output | TX LED A (Red) | Via $220\ \Omega$ resistor to LED $\rightarrow$ GND (supports PWM fade) |
| **A4** | PWM Output | TX LED B (Red) | Via $220\ \Omega$ resistor to LED $\rightarrow$ GND (supports PWM fade) |
| **A5** | PWM Output | TX LED C (Red) | Via $220\ \Omega$ resistor to LED $\rightarrow$ GND (supports PWM fade) |
| **A6** | Analog Input | RX MOD Button | Toggle pushbutton to **GND** for switching RX modes |
| **A7** | Analog (Free) | *Reserved / Unused* | Left unconnected (Mode feedback is routed to TX LEDs) |

---

## 🛠️ Operation Guide

1. **Power-On Default State:** 
   * **RX:** Antenna A active.
   * **TX:** All antennas active (A + B + C).
2. **Changing RX Modes:** 
   * Press the **MOD** button (connected to pin `A6`). 
   * The red TX LEDs will blink **once** for *Exclusive Mode* or **twice quickly** for *Flexible Mode*.
3. **Transmission & Visual Feedback:**
   * Grounding the **PTT** line (pin `D11`) switches the unit to transmit instantly and locks all front-panel buttons.
   * Active TX LEDs will pulse with a smooth **fade/breathing effect** while transmitting. Inactive TX LEDs remain dark.
4. **Quick Stack Shortcuts:**
   * Pressing **RX ALL** (`D12`) or **TX ALL** (`D13`) instantly selects all three antennas for reception or transmission respectively, regardless of the active mode.

---

## 📄 Source Code

```cpp
// ========================================================
// SJ2W StackMatch Controller - Flexible/Exclusive RX Mode
// Arduino Nano + Watchdog + Tail-Delay + Independent TX Fade
// ========================================================

#include <avr/wdt.h>

const int RELAY_1 = 2;
const int RELAY_2 = 3;
const int RELAY_3 = 4;

const int BTN_RX_A    = 5;
const int BTN_RX_B    = 6;
const int BTN_RX_C    = 7;
const int BTN_RX_ALL  = 12;
const int BTN_RX_MODE = A6; 

const int BTN_TX_A    = 8;
const int BTN_TX_B    = 9;
const int BTN_TX_C    = 10;
const int BTN_TX_ALL  = 13;

const int PTT_PIN     = 11;

const int LED_RX_A    = A0;
const int LED_RX_B    = A1;
const int LED_RX_C    = A2;
const int LED_TX_A    = A3;
const int LED_TX_B    = A4;
const int LED_TX_C    = A5;

bool rxAnt_A = true;
bool rxAnt_B = false;
bool rxAnt_C = false;

bool txAnt_A = true;
bool txAnt_B = true;
bool txAnt_C = true;

bool rxFlexibleMode = false;
bool lastPttState = HIGH;

unsigned long pttReleasedTime = 0;
const unsigned long TAIL_DELAY_MS = 80; 
int fadeBrightness = 0;
int fadeStep = 5;

void setup() {
  pinMode(RELAY_1, OUTPUT);
  pinMode(RELAY_2, OUTPUT);
  pinMode(RELAY_3, OUTPUT);

  pinMode(LED_RX_A, OUTPUT);
  pinMode(LED_RX_B, OUTPUT);
  pinMode(LED_RX_C, OUTPUT);
  pinMode(LED_TX_A, OUTPUT);
  pinMode(LED_TX_B, OUTPUT);
  pinMode(LED_TX_C, OUTPUT);

  pinMode(BTN_RX_A, INPUT_PULLUP);
  pinMode(BTN_RX_B, INPUT_PULLUP);
  pinMode(BTN_RX_C, INPUT_PULLUP);
  pinMode(BTN_RX_ALL, INPUT_PULLUP);
  pinMode(BTN_RX_MODE, INPUT_PULLUP);

  pinMode(BTN_TX_A, INPUT_PULLUP);
  pinMode(BTN_TX_B, INPUT_PULLUP);
  pinMode(BTN_TX_C, INPUT_PULLUP);
  pinMode(BTN_TX_ALL, INPUT_PULLUP);

  pinMode(PTT_PIN, INPUT_PULLUP);

  updateStatusLEDs();
  updateHardwareRelays();

  wdt_enable(WDTO_2S);
}

void loop() {
  wdt_reset();

  bool currentPttState = digitalRead(PTT_PIN);
  bool isPttActive = (currentPttState == LOW);
  bool isTailDelayActive = (millis() - pttReleasedTime < TAIL_DELAY_MS);

  if (lastPttState == LOW && currentPttState == HIGH) {
    pttReleasedTime = millis();
  }
  lastPttState = currentPttState;

  if (currentPttState != lastPttState || currentPttState == HIGH) {
    updateHardwareRelays();
  }

  if (isPttActive) {
    fadeBrightness += fadeStep;
    if (fadeBrightness <= 20 || fadeBrightness >= 255) {
      fadeStep = -fadeStep; 
    }

    analogWrite(LED_TX_A, txAnt_A ? fadeBrightness : 0);
    analogWrite(LED_TX_B, txAnt_B ? fadeBrightness : 0);
    analogWrite(LED_TX_C, txAnt_C ? fadeBrightness : 0);
  } else {
    analogWrite(LED_TX_A, txAnt_A ? 255 : 0);
    analogWrite(LED_TX_B, txAnt_B ? 255 : 0);
    analogWrite(LED_TX_C, txAnt_C ? 255 : 0);
    fadeBrightness = 20; 
  }

  if (isPttActive || isTailDelayActive) {
    delay(10);
    return; 
  }

  if (digitalRead(BTN_RX_MODE) == LOW) {
    rxFlexibleMode = !rxFlexibleMode; 
    visualModeFeedback(rxFlexibleMode ? 2 : 1);
    updateStatusLEDs(); 
    delay(250); 
  }

  if (digitalRead(BTN_RX_ALL) == LOW) {
    rxAnt_A = true;  rxAnt_B = true;  rxAnt_C = true;  
    updateStatusLEDs(); delay(200);
  }
  else if (digitalRead(BTN_RX_A) == LOW) {
    if (rxFlexibleMode) { rxAnt_A = !rxAnt_A; } 
    else { rxAnt_A = true; rxAnt_B = false; rxAnt_C = false; }
    updateStatusLEDs(); delay(200);
  }
  else if (digitalRead(BTN_RX_B) == LOW) {
    if (rxFlexibleMode) { rxAnt_B = !rxAnt_B; } 
    else { rxAnt_A = false; rxAnt_B = true; rxAnt_C = false; }
    updateStatusLEDs(); delay(200);
  }
  else if (digitalRead(BTN_RX_C) == LOW) {
    if (rxFlexibleMode) { rxAnt_C = !rxAnt_C; } 
    else { rxAnt_A = false; rxAnt_B = false; rxAnt_C = true; }
    updateStatusLEDs(); delay(200);
  }

  if (digitalRead(BTN_TX_ALL) == LOW) {
    txAnt_A = true;  txAnt_B = true;  txAnt_C = true;  
    updateStatusLEDs(); delay(200);
  }
  else if (digitalRead(BTN_TX_A) == LOW) { txAnt_A = !txAnt_A; updateStatusLEDs(); delay(200); }
  else if (digitalRead(BTN_TX_B) == LOW) { txAnt_B = !txAnt_B; updateStatusLEDs(); delay(200); }
  else if (digitalRead(BTN_TX_C) == LOW) { txAnt_C = !txAnt_C; updateStatusLEDs(); delay(200); }

  delay(15);
}

void visualModeFeedback(int blinks) {
  for (int i = 0; i < blinks; i++) {
    digitalWrite(LED_TX_A, HIGH);
    digitalWrite(LED_TX_B, HIGH);
    digitalWrite(LED_TX_C, HIGH);
    delay(120);
    digitalWrite(LED_TX_A, LOW);
    digitalWrite(LED_TX_B, LOW);
    digitalWrite(LED_TX_C, LOW);
    delay(120);
  }
}

void updateHardwareRelays() {
  bool isTX = (digitalRead(PTT_PIN) == LOW);

  bool active_A = isTX ? txAnt_A : rxAnt_A;
  bool active_B = isTX ? txAnt_B : rxAnt_B;
  bool active_C = isTX ? txAnt_C : rxAnt_C;

  driveSJ2WRelays(active_A, active_B, active_C);
}

void driveSJ2WRelays(bool antA, bool antB, bool antC) {
  if (antA && !antB && !antC)      { setRelayPins(LOW,  LOW,  LOW);  } 
  else if (!antA && antB && !antC) { setRelayPins(HIGH, LOW,  LOW);  } 
  else if (!antA && !antB && antC) { setRelayPins(LOW,  HIGH, LOW);  } 
  else if (antA && antB && !antC)  { setRelayPins(HIGH, HIGH, LOW);  } 
  else if (antA && !antB && antC)  { setRelayPins(LOW,  LOW,  HIGH); } 
  else if (!antA && antB && antC)  { setRelayPins(HIGH, LOW,  HIGH); } 
  else if (antA && antB && antC)   { setRelayPins(HIGH, HIGH, HIGH); } 
  else {
    setRelayPins(LOW, LOW, LOW); 
  }
}

void setRelayPins(uint8_t r1, uint8_t r2, uint8_t r3) {
  digitalWrite(RELAY_1, r1);
  digitalWrite(RELAY_2, r2);
  digitalWrite(RELAY_3, r3);
}

void updateStatusLEDs() {
  digitalWrite(LED_RX_A, rxAnt_A ? HIGH : LOW);
  digitalWrite(LED_RX_B, rxAnt_B ? HIGH : LOW);
  digitalWrite(LED_RX_C, rxAnt_C ? HIGH : LOW);

  if (digitalRead(PTT_PIN) == HIGH) {
    digitalWrite(LED_TX_A, txAnt_A ? HIGH : LOW);
    digitalWrite(LED_TX_B, txAnt_B ? HIGH : LOW);
    digitalWrite(LED_TX_C, txAnt_C ? HIGH : LOW);
  }
}
