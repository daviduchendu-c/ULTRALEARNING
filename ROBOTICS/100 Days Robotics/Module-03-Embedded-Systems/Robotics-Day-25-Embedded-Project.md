# Day 25: Embedded Project

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 6 hours, split over two sessions if needed

**Level:** intermediate. You need Days 16 to 24. The project combines GPIO, the ADC, PWM, interrupts or debouncing, I2C, serial telemetry, a state machine, FreeRTOS tasks and layered code. If one of those is weak, go back to its day before you start, because the project will expose it.

**By the end of this day, you will have:**

- An ESP32 device that reads a temperature and humidity sensor over I2C, a setpoint potentiometer and a button, and drives a 5 V fan through a MOSFET with PWM
- A state machine with four states, a hysteresis rule for the fan, and a manual override reached from the button or a serial command
- Framed, checksummed telemetry at 2 Hz, and a receiver script or serial log that validates every frame
- A fault test log that shows the device entering a safe state when the sensor is unplugged and recovering afterward
- A written requirements list and a bench test matrix, each item marked as passed or failed with evidence

**What you need:**

- An ESP32 DevKit board, about $6 to $12, and a data-capable USB cable
- PlatformIO Core 6.1.x with the `espressif32` platform, free and open source. The Arduino framework for ESP32 (Arduino-ESP32 core 3.x) is installed by PlatformIO
- An SHT31 temperature and humidity sensor breakout, I2C address 0x44, about $5 to $15 depending on the brand. Check that the breakout has pull-ups or add two 4.7 kOhm resistors
- A 10 kOhm linear potentiometer, about $1, on GPIO 34
- A pushbutton, about $0.50, on GPIO 14 with the internal pull-up
- One LED and a 220 ohm resistor on GPIO 25, about $0.20
- A 5 V DC fan, 40 mm, about $3 to $5, with a current draw you have measured or read from its label
- A logic-level N-channel MOSFET with a datasheet that gives its on-resistance at a 3.3 V gate drive. The IRLZ44N is common, but its datasheet gives Rds(on) at 4 V and 5 V, so check the numbers before you rely on it at 3.3 V. Cost about $1.
- A flyback diode, 1N5819 or equivalent, about $0.20, across the fan
- A 5 V power supply for the fan, rated above the fan's current, about $8. The fan must not draw its current from the ESP32's 3.3 V or 5 V pin.
- A breadboard, jumper wires and a multimeter, about $25 to $40 if you do not already have them
- Total cost for the project, excluding tools you already own: about $35 to $55

**Prerequisites:**

- Day 13: MOSFET switching, the flyback diode and the current sizing rule
- Day 17: debouncing and active-low buttons
- Day 18: ADC conversion and PWM duty
- Day 21: I2C addresses, pull-ups and the Wire return codes
- Day 20: framed serial telemetry with a checksum
- Day 23: layers, interfaces, the state machine and the watchdog
- Day 24: tasks, queues and mutexes

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Sensors and actuators working together](#sensors-and-actuators-working-together)
- [Two-way communication](#two-way-communication)
- [The state machine in a real device](#the-state-machine-in-a-real-device)
- [Telemetry you can trust](#telemetry-you-can-trust)
- [Structured firmware for a finished device](#structured-firmware-for-a-finished-device)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a fan controller with telemetry and fault recovery](#project-a-fan-controller-with-telemetry-and-fault-recovery)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A product that works on the bench is a prototype. A product that works when the sensor is unplugged, the power dips, the operator presses the wrong button and the serial cable is pulled out is a product. The difference is the set of decisions that the firmware makes when something goes wrong, and the evidence that those decisions were tested.

This project is a small device with a real purpose: a fan that keeps a space near a temperature setpoint. It is simple on purpose. The sensor is an I2C device with a checksum. The setpoint is an analog knob. The fan is a power load that must be switched safely. The button gives the operator an override. The serial link reports what the device is doing and accepts a command. Every part of it has appeared in an earlier day, and the work today is connecting them and proving that the connections hold under failure.

The project is the module's capstone, so the deliverable is more than code. You will write a requirements list, a test matrix, a set of logs and a short report on what failed and what you changed. A hiring manager who reads that package learns more about your engineering than from a demonstration video.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1950s to 1960s | Thermostats and electromechanical controllers use bimetal strips and relays, with hysteresis built in by the mechanical design. | Hysteresis, the gap between the on and off points, is still the simplest way to stop a relay or a fan from chattering. |
| 1970s to 1980s | Microcontroller-based controllers replace many mechanical controllers in appliances and HVAC systems. | Embedded thermostats are now common in homes and in lab equipment. |
| 1990s to 2000s | Digital sensors with I2C or SPI outputs and built-in checksums replace analog thermistors in many products. | The SHT31 in this project is one of those sensors, with a CRC on every reading. |
| 2010s | Low-cost Wi-Fi microcontrollers make connected sensors and fans cheap, and firmware updates over the air become common. | ESP32 projects are often the first connected device a robotics engineer builds. |
| Today | Field devices use state machines, watchdogs, telemetry and fault logs as standard parts of their design. | A product review asks for the fault behavior and the test evidence before the feature list. |

The thermostat is an old idea, and the hysteresis rule that keeps it stable is the one you will use today. The newer parts are the checksum on the sensor, the fault logs and the test evidence.

---

## Glossary

- **Setpoint:** the target value that a controller tries to hold, here a temperature.
- **Hysteresis:** a gap between the switch-on and switch-off thresholds, which prevents rapid switching near the setpoint.
- **Relative humidity (RH):** the amount of water vapor in air, as a percentage of the maximum it can hold at that temperature.
- **CRC (cyclic redundancy check):** a checksum calculated with a polynomial. The SHT31 uses an 8-bit CRC on each two-byte value.
- **Single-shot measurement:** a sensor command that makes one measurement and then waits for a read.
- **Flyback diode:** a diode across an inductive load that absorbs the energy stored when the load is switched off. Day 13 covered it.
- **Manual override:** a mode in which the operator sets the output directly, bypassing the automatic control.
- **Command frame:** a message from the operator to the device, with a fixed format and a checksum.
- **Requirements list:** a numbered list of what the device must do, with a test for each item.
- **Bench test matrix:** a table of the tests, the expected result and the measured result for each requirement.
- **Fault injection:** a deliberate failure, such as an unplugged sensor, used to test the firmware's response.
- **Degraded mode:** a reduced operating state in which the device runs with fewer functions and reports a warning.

---

### Sensors and actuators working together

A thermostat in a house is a sensor, a decision and a switch. The thermometer reads the room, the controller decides whether the room is too hot, and the switch turns the cooling on or off. Each part is simple. The difficulty is that the parts have different speeds, different failure modes and different voltages, and the design must handle all of them together.

From first principles, the SHT31 measures temperature and humidity when asked. The measurement takes about 15 ms with the high-repeatability command, and the sensor returns six bytes: a 16-bit temperature, its CRC, a 16-bit humidity, and its CRC. The conversion to physical units is linear: T in degrees Celsius equals −45 + 175 × raw / 65535, and RH in percent equals 100 × raw / 65535. The CRC check tells you whether the bytes arrived intact. A reading that fails the CRC is discarded, and the failure is counted.

The fan is an actuator with a power requirement. It is driven by PWM through a MOSFET, with the flyback diode across its terminals. The PWM frequency is set to 25 kHz, which is above the audible range and is a common choice for PC-style fans. The duty cycle is set by the control logic and written with `ledcWrite`. The fan's speed is not linear with duty, because the fan has a minimum starting voltage. Below a certain duty, it does not start, and the README should record the duty at which your fan starts reliably.

The worked example is the power budget. A 5 V fan that draws 150 mA at full speed uses 0.75 W. A fan supply of 5 V at 2 A gives a margin of about 2 A / 0.15 A = 13 times the fan's current, which is a generous safety margin for startup surges. The ESP32 supplies 5 V from USB at a limited current, and a fan connected to the ESP32 pins would overload the regulator. The fan goes on its own supply, with a common ground to the ESP32.

The common mistake is to power the fan from the ESP32's 5 V pin. The USB power budget is limited, and the ESP32 can reset when a load pulls the supply down. The second mistake is to drive the MOSFET gate from the ESP32 without checking its gate voltage. A 3.3 V gate drive may not fully turn on some MOSFETs, and the MOSFET then heats up under load. The check is the Rds(on) at 3.3 V in the datasheet.

On the job, sensor and actuator selection is a power and timing budget. Engineers check the supply current, the sensor's measurement time and the actuator's start-up behavior before they write the first line of control code.

---

### Two-way communication

A radio on a boat sends its position and also receives instructions from shore. The two directions have different rules. The position report must arrive on schedule, and an instruction must be checked before the crew acts on it. The device in this project has the same two directions over one USB serial link.

From first principles, the device sends telemetry frames at 2 Hz from the telemetry task. The frames use the format from Day 20 and Day 23: a start character, a body, a star and a two-digit hexadecimal checksum. The operator can send command frames back, and the device parses them in the telemetry task, validates the checksum, and sends a command struct into a queue for the control task. The control task is the only task that changes the fan output, which keeps the rule that one owner controls each output.

Two commands are enough for this project. `A` returns the device to automatic control. `F` followed by a number from 0 to 100 sets the manual fan duty as a percentage. The checksum rule for commands is the same as for telemetry, so the device rejects a command with a bad checksum and reports the rejection in the next telemetry frame.

The worked example is a command that arrives while the device is in the Fault state. The control task receives the manual command, but the fault rule takes priority. The device stays in the Fault state, the fan remains off, and the telemetry shows that the command was received and rejected. The rule is simple, but it must be tested, because a command that turns on a fan during a sensor fault is the kind of bug that causes damage.

The common mistake is to parse a command with `Serial.parseInt()` and no checksum. A noisy cable or a stray byte can turn a `F` command into an unexpected duty, and the device then runs the fan at full speed. Another mistake is to let the telemetry task write the fan output directly. A second writer to the same output is a race condition, and it appears only when the two tasks happen to write at the same time.

On the job, command channels are where most field incidents start. A command with a checksum, a range check and a single owner is a basic safety design.

---

### The state machine in a real device

A vending machine has states for idle, accepting coins, vending and out of stock. A coin inserted in the wrong state is returned, and the machine tells the customer why. The machine works because each state has a clear list of what it accepts. The fan controller uses the same method.

From first principles, the device has four states. AUTO_OFF means the sensor is good and the temperature is below the setpoint minus the hysteresis band, so the fan is off. AUTO_ON means the temperature is above the setpoint plus the band, so the fan runs at full duty. MANUAL means the operator has set the duty. FAULT means the sensor has failed three times in a row, so the fan is forced off, and the device waits for recovery. The hysteresis band is 0.5 degrees Celsius on each side of the setpoint, which stops the fan from switching on and off near the set point.

The transition rules are short. AUTO_OFF moves to AUTO_ON when the temperature exceeds the setpoint plus 0.5 degrees. AUTO_ON moves to AUTO_OFF when the temperature falls below the setpoint minus 0.5 degrees. Any automatic state moves to MANUAL when the button is pressed or an `F` command is accepted. MANUAL moves to AUTO_OFF on an `A` command. Any state except MANUAL moves to FAULT after three consecutive sensor failures, and FAULT moves to AUTO_OFF after five seconds of consecutive good readings.

The worked example is the hysteresis check. With a setpoint of 25.0 degrees, the fan turns on at 25.6 degrees, which is above 25.5, and it turns off at 24.4 degrees, which is below 24.5. A reading that wobbles between 24.8 and 25.2 degrees keeps the fan off, and the fan does not chatter. Without the band, the same wobble would switch the fan on and off every reading, which wears the relay and the fan bearings.

The common mistake is to test only the happy path. A state machine has a state-by-input matrix, and each cell should have a defined result. The matrix in `docs/state_table.md` is the first thing a reviewer reads, and a missing cell is a bug.

On the job, state machines in field devices are reviewed for every state and every input. The test plan covers each transition, and the fault injection covers the transitions that never happen on the bench.

---

### Telemetry you can trust

A flight recorder on an aircraft records the state of the plane many times per second, in a fixed format, with checks that show whether the data is intact. A telemetry stream from a robot works the same way. The device must send its state in a format that a person or a script can read, and the receiver must be able to tell whether a line is corrupted.

From first principles, a telemetry frame carries the time since boot, the state name, the temperature, the humidity, the setpoint, the fan duty and the fault code. The frame is one line, and the checksum is the XOR of the body. The receiver recomputes the XOR and compares it with the transmitted value. A frame that fails the check is counted as bad, and the count is part of the test evidence. The rate of 2 Hz gives a stream of about 120 lines a minute, which a person can read and a script can log.

The worked example is a checksum that catches a single bit flip. If one character changes from `2` to `3`, the XOR changes by one bit, and the checksum no longer matches. The receiver flags the frame. If two bytes change in a way that cancels in the XOR, the checksum misses it. That is why the README should describe the checksum limit, and why a critical system uses a stronger check such as a CRC-16.

The common mistake is to print floats without a fixed format. The output changes width from line to line, and a parser that expects fixed columns breaks. Use `%.1f` for the temperature and humidity, and a fixed field order.

On the job, telemetry is the first tool that a support engineer uses when a field device misbehaves. A stable format and a validation rule turn a pile of logs into a tool for diagnosis.

---

### Structured firmware for a finished device

A house built with a plan has rooms, doors and utilities in known places. A house built without a plan may work, but every repair takes longer. Firmware is the same. A structure that separates the sensor driver, the control logic, the telemetry and the hardware layer makes each change smaller and each test clearer.

From first principles, the project uses the layered structure from Day 23, extended to tasks on the ESP32. The sensor driver owns the I2C bus and returns a status code and a value. The control module owns the state machine and the fan output. The telemetry module owns the frame format and the command parser. The hardware layer owns the pin numbers and the PWM setup. Each module has a header with its interface and a source file with its internals.

The task structure follows the Day 24 pipeline. The sensor task runs every 500 ms, reads the SHT31 and the pot, and puts a snapshot into a queue. The control task receives snapshots and commands, runs the state machine and writes the fan duty. The telemetry task formats a frame every 500 ms and parses incoming commands into the command queue. Only the control task writes the fan output, and only the sensor task uses the I2C bus.

The worked example is the requirements list. The project needs about ten requirements, for example: the fan turns on above the setpoint plus 0.5 degrees, the fan is off in FAULT, the telemetry arrives at 2 Hz with valid checksums, and a bad command is rejected. Each requirement has a test in the matrix, a measured result and a pass or fail mark. A requirement without a test is a wish, and a test without a requirement is a guess.

The common mistake is to build the whole device first and write the tests afterward. The fault behavior is designed in the state table, and the tests check it. Writing the tests first, even as a short list, changes the design.

On the job, a finished device is reviewed against its requirements, its test matrix and its fault log. A team that can show that evidence ships with fewer surprises.

---

## Hands-on exercises

1. **Read the sensor alone.** Build a sketch that reads the SHT31 once per second and prints the temperature and humidity. Check the CRC on each read, and record the number of CRC failures in a minute. Compare the temperature with a reference thermometer.

2. **Test the fan driver.** Drive the MOSFET with PWM at 25 kHz and step the duty from 0 to 100 percent in steps of 10. Record the duty at which the fan starts and the duty at which it reaches full speed. Measure the MOSFET source-drain voltage with the fan running and check that the MOSFET is not hot after a five-minute run.

3. **Test the button.** Press the button 20 times and record the debounced events. Compare the count with the raw edges and write a sentence on the debounce time you used.

4. **Test the hysteresis.** Set the setpoint with the pot and warm or cool the sensor slowly with a hand or a cool pack. Record the temperatures at which the fan switches on and off. Check them against the band.

5. **Send a command.** Send `F50` from the serial monitor and confirm the fan runs at about half duty. Send a frame with a bad checksum and confirm the device rejects it. Save the telemetry lines that show each result.

6. **Inject a fault.** Unplug the SHT31 while the device runs. Record the state changes and the time of each. Reconnect and record the recovery time. Check the recovery against the five-second rule.

---

## Project: a fan controller with telemetry and fault recovery

The goal is a repository that contains the complete fan controller, a requirements list, a bench test matrix, a state table, a serial command set and a fault test with logs. It is the module capstone, so the evidence matters as much as the code.

**Repo name:** `100-days-robotics-day-25-embedded-project`

**Repo layout:**

```
100-days-robotics-day-25-embedded-project/
├── README.md
├── platformio.ini
├── src/
│   ├── main.cpp
│   ├── hal.h
│   ├── hal.cpp
│   ├── sht31.h
│   ├── sht31.cpp
│   ├── fan_control.h
│   ├── fan_control.cpp
│   ├── telemetry.h
│   └── telemetry.cpp
├── tools/
│   └── check_frames.py
├── logs/
│   ├── normal_run_5min.txt
│   ├── command_test.txt
│   ├── fault_test_unplug.txt
│   └── hysteresis_test.csv
├── docs/
│   ├── requirements.md
│   ├── test_matrix.md
│   ├── state_table.md
│   ├── power_budget.md
│   ├── wiring.md
│   └── report.md
└── photos/
    └── bench.jpg
```

Put `platformio.ini` at the project root with the `[env:esp32dev]` section from Day 24. The `src/` folder holds the module files, and PlatformIO compiles them together.

**Pin map (wire it exactly so):**

| Signal | ESP32 pin | Notes |
|---|---|---|
| SHT31 SDA | GPIO 21 | Default I2C data on the ESP32 DevKit |
| SHT31 SCL | GPIO 22 | Default I2C clock |
| Setpoint pot wiper | GPIO 34 | ADC1, input-only pin |
| Button | GPIO 14 | Internal pull-up, pressed reads LOW |
| Status LED | GPIO 25 | Through 220 ohm to the LED cathode at GND |
| Fan PWM | GPIO 26 | Drives the MOSFET gate through a 100 ohm resistor |

**Starter code, `src/hal.h` and `src/hal.cpp`:**

```cpp
// hal.h
#pragma once
#include <stdint.h>

namespace hal {
  void init();
  uint16_t readSetpointRaw();          // 0 to 4095
  bool buttonPressed();                // debounced by the caller
  void setStatusLed(bool on);
  void setFanDuty(uint8_t duty);       // 0 to 255
}
```

```cpp
// hal.cpp
#include <Arduino.h>
#include "hal.h"

namespace hal {
  static const int POT_PIN = 34;
  static const int BUTTON_PIN = 14;
  static const int LED_PIN = 25;
  static const int FAN_PIN = 26;
  static const uint32_t FAN_FREQ_HZ = 25000;
  static const uint8_t FAN_RES_BITS = 8;

  void init() {
    pinMode(BUTTON_PIN, INPUT_PULLUP);
    pinMode(LED_PIN, OUTPUT);
    analogReadResolution(12);
    ledcAttach(FAN_PIN, FAN_FREQ_HZ, FAN_RES_BITS);
    ledcWrite(FAN_PIN, 0);
  }

  uint16_t readSetpointRaw() { return analogRead(POT_PIN); }
  bool buttonPressed() { return digitalRead(BUTTON_PIN) == LOW; }
  void setStatusLed(bool on) { digitalWrite(LED_PIN, on ? HIGH : LOW); }
  void setFanDuty(uint8_t duty) { ledcWrite(FAN_PIN, duty); }
}
```

The `ledcAttach` call is the Arduino-ESP32 core 3.x form. Older cores use `ledcSetup` and `ledcAttachPin`. Check the core version in your build log, and record it in the README, because the call changed between versions.

**Starter code, `src/sht31.h` and `src/sht31.cpp`:**

```cpp
// sht31.h
#pragma once
#include <stdint.h>

enum class ShtStatus : uint8_t { Ok, NoAck, ShortRead, BadCrc };
struct ShtReading { float tempC; float rh; };

namespace sht31 {
  void init();
  bool startMeasure();                          // returns false on NACK
  ShtStatus readResult(ShtReading &out);        // call about 20 ms after startMeasure
}
```

```cpp
// sht31.cpp
#include <Arduino.h>
#include <Wire.h>
#include "sht31.h"

namespace sht31 {
  static const uint8_t ADDR = 0x44;

  static uint8_t crc8(const uint8_t *d, size_t n) {
    uint8_t crc = 0xFF;
    for (size_t i = 0; i < n; i++) {
      crc ^= d[i];
      for (int b = 0; b < 8; b++) {
        crc = (crc & 0x80) ? (uint8_t)((crc << 1) ^ 0x31) : (uint8_t)(crc << 1);
      }
    }
    return crc;
  }

  void init() { Wire.begin(21, 22); }

  bool startMeasure() {
    Wire.beginTransmission(ADDR);
    Wire.write(0x24);                  // single shot, high repeatability
    Wire.write(0x00);                  // clock stretching disabled
    return Wire.endTransmission() == 0;
  }

  ShtStatus readResult(ShtReading &out) {
    if (Wire.requestFrom(ADDR, (uint8_t)6) != 6) return ShtStatus::ShortRead;
    uint8_t d[6];
    for (int i = 0; i < 6; i++) d[i] = Wire.read();
    if (crc8(d, 2) != d[2] || crc8(d + 3, 2) != d[5]) return ShtStatus::BadCrc;
    uint16_t rawT  = ((uint16_t)d[0] << 8) | d[1];
    uint16_t rawRH = ((uint16_t)d[3] << 8) | d[4];
    out.tempC = -45.0f + 175.0f * rawT / 65535.0f;
    out.rh    = 100.0f * rawRH / 65535.0f;
    return ShtStatus::Ok;
  }
}
```

The `startMeasure` function returns false on a NACK, and the caller maps that to `NoAck`. The example above uses `ShortRead` for a missing byte, and a full driver would also return `NoAck` from `startMeasure`. Make that mapping in `main.cpp` or in the driver, and document where it lives.

**Starter code, `src/fan_control.h` and `src/fan_control.cpp`:**

```cpp
// fan_control.h
#pragma once
#include <stdint.h>

enum class Mode : uint8_t { AutoOff, AutoOn, Manual, Fault };

struct Snapshot {
  uint32_t ms;
  float tempC;
  float rh;
  bool sensorOk;
  uint16_t setpointRaw;
  bool buttonEvent;
  uint8_t faultCode;
};

struct Command {
  bool isAuto;          // true for A, false for F
  uint8_t percent;      // 0 to 100 for F
};

namespace fan {
  void init();
  void update(const Snapshot &s, const Command *cmd);
  Mode mode();
  uint8_t duty();
}
```

```cpp
// fan_control.cpp
#include "fan_control.h"
#include "hal.h"

namespace fan {
  static Mode currentMode = Mode::AutoOff;
  static uint8_t currentDuty = 0;
  static uint8_t failures = 0;
  static uint32_t okSince = 0;
  static const float BAND_C = 0.5f;
  static const uint8_t FAILURE_LIMIT = 3;
  static const uint32_t RECOVERY_MS = 5000;

  void init() { currentMode = Mode::AutoOff; currentDuty = 0; hal::setFanDuty(0); }

  Mode mode() { return currentMode; }
  uint8_t duty() { return currentDuty; }

  static float setpointC(uint16_t raw) { return 15.0f + raw * 20.0f / 4095.0f; }

  void update(const Snapshot &s, const Command *cmd) {
    if (!s.sensorOk) {
      if (failures < 255) failures++;
      okSince = s.ms;
      if (failures >= FAILURE_LIMIT) currentMode = Mode::Fault;
    } else {
      failures = 0;
      if (currentMode == Mode::Fault) {
        currentMode = Mode::AutoOff;          // recovery after RECOVERY_MS, see below
        okSince = s.ms;
      }
    }

    if (currentMode != Mode::Fault && currentMode != Mode::Manual) {
      float sp = setpointC(s.setpointRaw);
      if (currentMode == Mode::AutoOff && s.sensorOk && s.tempC > sp + BAND_C) currentMode = Mode::AutoOn;
      if (currentMode == Mode::AutoOn  && s.sensorOk && s.tempC < sp - BAND_C) currentMode = Mode::AutoOff;
    }

    if (s.buttonEvent && currentMode != Mode::Fault) currentMode = Mode::Manual;

    if (cmd != nullptr && currentMode != Mode::Fault) {
      if (cmd->isAuto) currentMode = Mode::AutoOff;
      else { currentMode = Mode::Manual; currentDuty = (uint8_t)(cmd->percent * 255 / 100); }
    }

    switch (currentMode) {
      case Mode::AutoOff:  currentDuty = 0;   break;
      case Mode::AutoOn:   currentDuty = 255; break;
      case Mode::Manual:   break;             // duty set by command
      case Mode::Fault:    currentDuty = 0;   break;
    }
    hal::setFanDuty(currentDuty);
  }
}
```

This draft has a flaw that the lesson asks you to find and fix. The recovery rule requires five seconds of good reads before leaving FAULT, but the code leaves FAULT on the first good read and never uses `RECOVERY_MS`. The `okSince` variable is set but not used to gate the transition. Write the fix as part of the project. The correct approach is to move from FAULT to a recovering state on the first good read, and to leave that state only after `RECOVERY_MS` of good reads, and to reset the timer on any failure. Record the bug and the fix in `docs/report.md`.

**Starter code, `src/telemetry.h` and `src/telemetry.cpp`:**

```cpp
// telemetry.h
#pragma once
#include "fan_control.h"

namespace telemetry {
  void init();
  void sendFrame(const Snapshot &s, Mode mode, uint8_t duty);
  bool parseCommand(const char *line, Command &out);
}
```

```cpp
// telemetry.cpp
#include <Arduino.h>
#include <string.h>
#include "telemetry.h"

namespace telemetry {
  static const char *modeName(Mode m) {
    switch (m) {
      case Mode::AutoOff: return "AUTO_OFF";
      case Mode::AutoOn:  return "AUTO_ON";
      case Mode::Manual:  return "MANUAL";
      case Mode::Fault:   return "FAULT";
    }
    return "UNKNOWN";
  }

  void init() { Serial.begin(115200); }

  void sendFrame(const Snapshot &s, Mode mode, uint8_t duty) {
    char body[120];
    snprintf(body, sizeof(body), "TEL,%lu,%s,%.1f,%.1f,%u,%u,%u",
             (unsigned long)s.ms, modeName(mode), s.tempC, s.rh,
             (unsigned)s.setpointRaw, (unsigned)duty, (unsigned)s.faultCode);
    uint8_t cs = 0;
    for (size_t i = 0; body[i] != '\0'; i++) cs ^= (uint8_t)body[i];
    Serial.print('$'); Serial.print(body); Serial.print('*');
    if (cs < 16) Serial.print('0');
    Serial.println(cs, HEX);
  }

  // Accepts "$A*XX" or "$F<0-100>*XX" with a valid checksum.
  bool parseCommand(const char *line, Command &out) {
    const char *star = strchr(line, '*');
    if (line[0] != '$' || star == nullptr) return false;
    size_t bodyLen = (size_t)(star - (line + 1));
    uint8_t cs = 0;
    for (size_t i = 0; i < bodyLen; i++) cs ^= (uint8_t)line[1 + i];
    if (cs != (uint8_t)strtoul(star + 1, nullptr, 16)) return false;
    if (bodyLen == 1 && line[1] == 'A') { out.isAuto = true; out.percent = 0; return true; }
    if (bodyLen >= 2 && line[1] == 'F') {
      int p = atoi(line + 2);
      if (p < 0 || p > 100) return false;
      out.isAuto = false; out.percent = (uint8_t)p; return true;
    }
    return false;
  }
}
```

The command format is `$A*XX` or `$F50*XX`, where `XX` is the XOR of the body characters in hexadecimal. The operator computes the checksum with the same rule as the telemetry. A small helper in `tools/check_frames.py` can compute it for you.

**Starter code, `src/main.cpp`:**

```cpp
#include <Arduino.h>
#include "hal.h"
#include "sht31.h"
#include "fan_control.h"
#include "telemetry.h"

static QueueHandle_t snapQueue;
static QueueHandle_t cmdQueue;
static SemaphoreHandle_t stateMutex;      // guards the latest snapshot for logging

void sensorTask(void *) {
  TickType_t last = xTaskGetTickCount();
  for (;;) {
    Snapshot s{};
    s.ms = millis();
    s.setpointRaw = hal::readSetpointRaw();
    s.sensorOk = false;
    s.faultCode = 0;
    if (sht31::startMeasure()) {
      vTaskDelay(pdMS_TO_TICKS(20));
      ShtReading r;
      ShtStatus st = sht31::readResult(r);
      if (st == ShtStatus::Ok) {
        s.sensorOk = true; s.tempC = r.tempC; s.rh = r.rh;
      } else {
        s.faultCode = (uint8_t)(10 + (int)st);
      }
    } else {
      s.faultCode = 1;                      // NACK on start
    }
    xQueueOverwrite(snapQueue, &s);         // single-slot latest value
    vTaskDelayUntil(&last, pdMS_TO_TICKS(500));
  }
}

void controlTask(void *) {
  static bool wasPressed = false;
  static bool pendingPress = false;
  static uint32_t lastSnapMs = 0xFFFFFFFFu;
  for (;;) {
    // Edge detector, polled every 20 ms. It records a press between snapshots.
    bool pressed = hal::buttonPressed();
    if (pressed && !wasPressed) pendingPress = true;
    wasPressed = pressed;

    // The fan logic runs once per new sensor snapshot, not once per poll,
    // so the failure counter advances once per sample.
    Snapshot s;
    if (xQueuePeek(snapQueue, &s, 0) == pdTRUE && s.ms != lastSnapMs) {
      lastSnapMs = s.ms;
      s.buttonEvent = pendingPress;
      pendingPress = false;
      Command c; Command *cp = nullptr;
      if (xQueueReceive(cmdQueue, &c, 0) == pdTRUE) cp = &c;
      fan::update(s, cp);
      hal::setStatusLed(fan::mode() != Mode::Fault);
    }
    vTaskDelay(pdMS_TO_TICKS(20));
  }
}

void telemetryTask(void *) {
  static char line[64];
  static size_t n = 0;
  TickType_t last = xTaskGetTickCount();
  for (;;) {
    while (Serial.available()) {
      char ch = Serial.read();
      if (ch == '\n') {
        line[n] = '\0';
        Command c;
        if (telemetry::parseCommand(line, c)) xQueueSend(cmdQueue, &c, 0);
        n = 0;
      } else if (n < sizeof(line) - 1) {
        line[n++] = ch;
      }
    }
    Snapshot s;
    if (xQueuePeek(snapQueue, &s, 0) == pdTRUE) {
      telemetry::sendFrame(s, fan::mode(), fan::duty());
    }
    vTaskDelayUntil(&last, pdMS_TO_TICKS(500));
  }
}

void setup() {
  hal::init();
  sht31::init();
  telemetry::init();
  fan::init();
  snapQueue = xQueueCreate(1, sizeof(Snapshot));
  cmdQueue  = xQueueCreate(4, sizeof(Command));
  stateMutex = xSemaphoreCreateMutex();
  xTaskCreatePinnedToCore(sensorTask,    "sensor", 4096, nullptr, 2, nullptr, 1);
  xTaskCreatePinnedToCore(controlTask,   "control", 4096, nullptr, 3, nullptr, 1);
  xTaskCreatePinnedToCore(telemetryTask, "telemetry", 4096, nullptr, 1, nullptr, 0);
}

void loop() { delay(1000); }
```

This starter code has two known issues for the lesson to find and document. First, the `Snapshot` in `controlTask` is read with `xQueuePeek` and not copied under a lock, which is safe because the queue copies the struct. Second, the `stateMutex` is created but never used, and the README should say whether it should be removed or used for the fan state read by the telemetry task. The button check is an edge detector polled every 20 ms. It is not a full debounce, so the button test in the matrix should show whether a single press can produce more than one event on your switch. If it does, add the stable-time check from Day 17 before you mark the requirement as passed.

**Step-by-step tasks:**

1. Create the repository and folder tree. Initialize git and commit the README skeleton.
2. Write the requirements list in `docs/requirements.md` with at least ten numbered items. Each item must have a number and a test that you can run.
3. Wire the device as in the pin map. Measure the fan's current at full speed and write it in `docs/power_budget.md` with the supply rating. Confirm the MOSFET does not heat under a five-minute run.
4. Build and upload the firmware. Record the platform and core versions from the build log in the README.
5. Fix the recovery bug in `fan_control.cpp` as described above, and write the change in `docs/report.md`.
6. Run the normal test for five minutes and save the output as `logs/normal_run_5min.txt`. Use `tools/check_frames.py` to count the valid and invalid frames.
7. Run the hysteresis test and save the temperatures and the fan transitions to `logs/hysteresis_test.csv`.
8. Run the command test. Send `A`, `F50`, a bad checksum and an out-of-range `F150`. Save the output as `logs/command_test.txt` and record the result of each.
9. Run the fault test. Unplug the SHT31 for ten seconds, reconnect it and wait for recovery. Save the output as `logs/fault_test_unplug.txt` and record the state changes and the recovery time.
10. Complete `docs/test_matrix.md` with each requirement, its test, the result and the evidence file. Mark each as pass or fail.
11. Write `docs/report.md` with the bugs you found, the fixes and the results. Keep it under two pages.
12. Write the README with the checklist below.

**README checklist:**

- [ ] The platform and Arduino-ESP32 core versions are stated from the build log
- [ ] The pin map and the wiring photo match the code
- [ ] The power budget lists the fan current, the supply rating and the MOSFET Rds(on) at 3.3 V
- [ ] The requirements list has at least ten items, each linked to a test
- [ ] The test matrix marks each item as pass or fail, with the evidence file
- [ ] The state table covers every state and every input, and matches the code
- [ ] The hysteresis test shows the on and off thresholds with the band
- [ ] The fault test shows the state changes, the fan output and the recovery time
- [ ] The report lists at least two bugs found during testing and how each was fixed
- [ ] The checksum limit is stated, with a note on when a CRC-16 would be needed

**Done when:**

- Every requirement in the matrix has a passing test with a log file, or a documented failure and a plan
- The normal run shows at least 500 frames over five minutes, with zero invalid frames
- The fault test shows FAULT entered after three failed reads, the fan forced off during the fault, and recovery only after five seconds of good reads
- The command test shows that the device rejects a bad checksum, an out-of-range percent and any command during FAULT
- The hysteresis test shows no fan chatter within the band
- The repository has at least five commits, one for each major change, and every number in the README and the report matches a log file

**Evidence to commit to GitHub:** the source files, `platformio.ini`, the tool script, the four logs, the requirements, the test matrix, the state table, the power budget, the wiring document, the report, the bench photo and the README. Do not commit the `.pio` build folder.

This project is the module capstone. The code matters, but the reviewer will read the requirements, the matrix and the report first. A device that you can test and explain is worth more to an employer than a device with more features that you cannot prove. Keep the report honest about what failed, because that is what an engineering review looks for.

---

## Check your understanding

1. A fan turns on at 25.6 degrees and off at 24.4 degrees with a setpoint of 25.0. Explain why the band is needed, and what would happen with a band of zero.
2. The SHT31 returns a temperature raw value of 0x6666 with a valid CRC. Compute the temperature in degrees Celsius to one decimal place.
3. The fan's startup current is 0.5 A and its running current is 0.15 A. Explain why the power supply rating and the MOSFET choice both need to account for the startup current, not only the running current.
4. A command `$F150*XX` with a valid checksum arrives during FAULT. Describe what the device should do, and name the module that enforces the rule.
5. The telemetry checksum is an XOR of the body. Give one error the checksum catches and one it misses, and state what you would change for a critical deployment.
6. During the fault test, the device returns to AUTO_OFF after one good read. Name the requirement this violates, and describe the change to the state machine that fixes it.

---

## Answers

1. The band prevents the fan from switching on and off every time the temperature wobbles near the setpoint. With a band of zero, a reading of 25.01 degrees turns the fan on and 24.99 turns it off, which chatters the relay and the fan.
2. The temperature is -45 + 175 × 26214 / 65535 = -45 + 70.0 = 25.0 degrees Celsius. Using 0x6666 = 26214 gives 175 × 0.4 = 70.0, which is 25.0 degrees.
3. The supply must deliver the startup current without a voltage drop, or the fan may stall or the ESP32 may reset. The MOSFET must handle the startup current in its drain current and heat rating. Sizing for only the running current leaves no margin for the surge.
4. The device should stay in FAULT, keep the fan off, and report the rejected command in the next telemetry frame. The control module enforces the rule, because it owns the state machine and the fan output.
5. The XOR catches a single-bit error in any character. It misses two errors that cancel out, such as two bit flips in the same position. For a critical deployment, use a CRC-16 or a stronger check, and add a sequence number so that dropped or repeated frames are visible.
6. The violation is the requirement that recovery from FAULT needs five seconds of consecutive good reads. The fix is to move to a recovering state on the first good read, to stay there while a timer counts the good reads, and to reset the timer on any failure. Leave for AUTO_OFF only after the timer reaches 5000 ms.

---

## Sources

*Read:*

- The Wikipedia article titled **Hysteresis**, for the control concept and its uses
- The Wikipedia article titled **Cyclic redundancy check**, for the CRC concept and the limits of simple checksums
- The Wikipedia article titled **Pulse-width modulation**, for the duty cycle and fan control
- The Wikipedia article titled **Finite-state machine**, for the state model
- The SHT31 datasheet by Sensirion, free from the Sensirion website. Read the sections on the single-shot commands, the measurement timing, the CRC and the conversion formulas
- The ESP32 datasheet and the ESP32 technical reference manual by Espressif, free from the Espressif website. Read the sections on the ADC, the LEDC PWM controller and the GPIO matrix
- The IRLZ44N datasheet by Infineon, free from the Infineon website. Read the Rds(on) table and note that it is specified at 4 V and 5 V gate drive. Compare it with the parts you consider for 3.3 V gate drive
- The Arduino-ESP32 core documentation for `ledcAttach` and the Wire library, on the Espressif GitHub repository or the docs.espressif.com site

*Tutorials:*

- The PlatformIO documentation at docs.platformio.org, for the ESP32 platform and the project layout
- The GeeksforGeeks article on finite-state machines. Search the GeeksforGeeks site for "finite state machine"

*Watch:*

- The Espressif YouTube channel, which has ESP32 tutorials on I2C, PWM and FreeRTOS. Search for "Espressif ESP32 tutorial"
- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, for lectures on design and testing of embedded projects. Search for the course title on the MIT OpenCourseWare site

*Practice:*

- Wokwi at wokwi.com, which supports the ESP32 and common sensors. Build the state machine and the telemetry there first, then build the real device. The free tier is enough for the logic, but it does not reproduce the sensor's CRC or the fan's startup behavior, so the bench tests are required.

---

## Tomorrow

Day 26 begins the next module, which moves from the microcontroller to the robot system around it. The fan controller you built today is a single node. A robot is many nodes that share data, timing and faults. The telemetry frame, the state machine and the test matrix from this project are the pieces you will reuse when the nodes need to talk to a larger system, and the report you wrote will show the next reviewer what this device does and how you proved it.
