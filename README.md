# Industrial Telemetry & Environmental Safety System (ITESS)

## Overview
The Industrial Telemetry & Environmental Safety System (ITESS) is an embedded microcontroller project designed to perform real-time environmental monitoring, dynamic user-defined threshold control, and multi-tier hazard mitigation. Built around the ATmega328P (Arduino Uno) platform, ITESS demonstrates a closed-loop control system that prioritizes physical proximity safety over secondary thermal management routines.

I built this system to practice integrating analog thermal sensors, dynamic user controls, and safety-override logic into a single Arduino station.


https://github.com/user-attachments/assets/4508a875-b354-4d10-89e3-7e41c52be0ea



## System Features
* Real-Time Thermal Monitoring: Continuous analog temperature acquisition via a TMP36 sensor.
* Dynamic User Threshold Control: Manual alarm point configuration (20°C to 80°C) via a potentiometer mapped using analog input scaling.
* Priority Safety Overrides: Ultrasonic distance sensing (HC-SR04) detects operator proximity within 50 cm, overriding standard operations to trigger immediate visual and audible hazard alerts.
* On-Board Telemetry Display: Live system metrics rendered on a 16x2 Character LCD driven via an I2C serial interface.
* High-Current Actuation & Multi-Tone Audio: Low-side NPN transistor switching to drive a DC cooling fan alongside dynamic Piezo buzzer alarm frequencies (1000 Hz for proximity hazards, 500 Hz for thermal over-temperature conditions).


## Hardware Architecture & Pinout Map

| Component | Interface / Signal | Arduino Uno Pin | Function Description |
| :--- | :--- | :--- | :--- |
| TMP36 Sensor | Analog Input | A0 | Temperature Voltage Signal |
| Potentiometer | Analog Input | A1 | Dynamic Threshold Dial |
| I2C LCD (16x2) | Data (SDA) | A4 | Serial Data Line |
| I2C LCD (16x2) | Clock (SCL) | A5 | Serial Clock Line |
| HC-SR04 Sensor | Digital Output | Pin 7 | Ultrasonic Trigger Pulse (TRIG) |
| HC-SR04 Sensor | Digital Input | Pin 6 | Echo Return Signal (ECHO) |
| Piezo Buzzer | Digital Output | Pin 8 | Audible Hazard Frequency Generation |
| DC Motor / LED | Digital Output | Pin 13 | NPN Transistor Switch Base Gate |

---

## Control Logic & System Hierarchy

The system operates on a dual-priority decision hierarchy inside the main execution loop:

                  +--------------------------------+
                  |  Read Sensors & Serial Inputs  |
                  +---------------+----------------+
                                  |
                                  v
                   /--------------+--------------\
                  /  Is Distance < 50 cm?         \
                  \  (Proximity Violation)        /
                   \--------------+--------------/
                                 / \
                        YES     /   \     NO
                       +-------+     +-------+
                       |                     |
                       v                     v
            +---------------------+   /--------------+--------------\
            | EMERGENCY OVERRIDE  |  /  Is Temp >= Set Threshold?    \
            | - LCD: PROXIMITY    |  \  (Thermal Overheat)           /
            | - Buzzer: 1000 Hz   |   \--------------+--------------/
            | - LED: Fast Flash   |                 / \
            +---------------------+        YES     /   \     NO
                                          +-------+     +-------+
                                          |                     |
                                          v                     v
                               +---------------------+ +-----------------+
                               | THERMAL HAZARD      | | NORMAL MODE     |
                               | - Fan/LED: ON       | | - Telemetry ON  |
                               | - Buzzer: 500 Hz    | | - Outputs: OFF  |
                               +---------------------+ +-----------------+

---
## Personal Implementation & Hardware Notes
* Project Purpose: Designed as a multi-sensor closed-loop control system integrating analog temperature sensing, dynamic user thresholds, and emergency override logic.
* Key Demonstration: Evaluated via Tinkercad Circuits to confirm priority safety hierarchy—ensuring proximity warnings ($<50\text{ cm}$) immediately override standard cooling routines.

---
Developer: Afolabi Adedokun  
Department: Mechatronics Engineering

## Source Code

```cpp
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

// Hardware Pins
const int tempSensorPin = A0; 
const int dialPin       = A1; 
const int hazardPin     = 13; 
const int trigPin       = 7;  
const int echoPin       = 6;  
const int buzzerPin     = 8;  

void setup() {
  pinMode(hazardPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  lcd.begin(16, 2); 
  lcd.setBacklight(HIGH);
  lcd.print("SYSTEM STARTING");
  delay(1000);
  lcd.clear();
}

void loop() {
  // 1. Distance Measurement via Ultrasonic Sensor
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH);
  float distanceCm = duration * 0.034 / 2.0;

  // 2. Read Analog Inputs
  int rawTemp = analogRead(tempSensorPin);
  float voltage = rawTemp * (5.0 / 1023.0);
  float currentTempC = (voltage - 0.5) * 100.0;

  int rawDial = analogRead(dialPin);
  float setThresholdC = map(rawDial, 0, 1023, 20, 80);

  // 3. PRIORITY 1: Proximity Hazard Safety Override
  if (distanceCm < 50.0) {
    lcd.setCursor(0, 0);
    lcd.print("PROXIMITY ALERT!");
    lcd.setCursor(0, 1);
    lcd.print("Dist: ");
    lcd.print(distanceCm, 0);
    lcd.print("cm HAZARD ");

    tone(buzzerPin, 1000); 
    digitalWrite(hazardPin, HIGH);
    delay(150);
    noTone(buzzerPin);
    digitalWrite(hazardPin, LOW);
    delay(150);
  } 
  // 4. PRIORITY 2: Thermal Overheat Mitigation
  else if (currentTempC >= setThresholdC) {
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    lcd.print(currentTempC, 1);
    lcd.print(" C OVER");

    lcd.setCursor(0, 1);
    lcd.print("FAN & ALARM ON ");

    tone(buzzerPin, 500); 
    digitalWrite(hazardPin, HIGH);
    delay(200);
    noTone(buzzerPin);
    digitalWrite(hazardPin, LOW);
    delay(200);
  } 
  // 5. NORMAL OPERATION: Telemetry Standby
  else {
    noTone(buzzerPin);
    digitalWrite(hazardPin, LOW);

    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    lcd.print(currentTempC, 1);
    lcd.print(" C   ");

    lcd.setCursor(0, 1);
    lcd.print("Set: ");
    lcd.print(setThresholdC, 1);
    lcd.print("C ");
    lcd.print((int)distanceCm);
    lcd.print("cm ");
  }
}
