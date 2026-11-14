# Day 23: Embedded Software Architecture

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 4 hours

**Level:** intermediate. You need Days 16 to 22, because today you organize the code from those days into one firmware. You should be able to write a small C++ class or a set of functions in a header and source file, and you should know what a `volatile` variable is from Day 19.

**By the end of this day, you will have:**

- A multi-file Arduino sketch with separate hardware, driver, telemetry and application layers, where no application file touches a register or a pin number directly
- A finite state machine with four states and a documented transition table, implemented in code
- A non-blocking main loop built on `millis()`, with the worst-case loop time measured
- A fault test in which an unplugged sensor moves the system into a degraded state and back, with the logs that show it

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3.x and the `Arduino AVR Boards` core at version 1.8.6
- A DS3231 real-time clock breakout from Day 21, with its pull-ups
- A 10 kOhm potentiometer from Day 18, on A0
- One red LED with a 220 ohm resistor from Day 17, on pin 13 (the onboard LED is also acceptable)
- A breadboard and jumper wires, about $5 to $8
- A digital multimeter, about $15 to $25
- Optional: `arduino-cli` 1.x, for command-line compile checks, free and open source
- Optional: PlatformIO Core 6.1.x, if you want a second build system to compare against. It is free and open source, and it is not required for this lesson.

**Prerequisites:**

- Day 16: the Uno's registers and the difference between a sketch and its build output
- Day 19: `volatile`, interrupts and the rule that shared data needs protection
- Day 20: the frame format and checksum
- Day 21: the DS3231 register map and the Wire return codes
- Day 22: why each driver should hide its bus details from the rest of the code

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Layers and the hardware abstraction layer](#layers-and-the-hardware-abstraction-layer)
- [Modules and clear interfaces](#modules-and-clear-interfaces)
- [State machines](#state-machines)
- [Non-blocking architecture](#non-blocking-architecture)
- [Telemetry as a separate layer](#telemetry-as-a-separate-layer)
- [Fault handling and the watchdog](#fault-handling-and-the-watchdog)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a layered sensor logger with fault recovery](#project-a-layered-sensor-logger-with-fault-recovery)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A workshop where every tool sits in one drawer works until the drawer is full. A new employee cannot find the wrench, the spare bits are mixed with the screws, and a dropped part ends up under the bench. Firmware that grows by adding code to one file has the same problem. The first version is easy to read. The tenth version has pin numbers scattered through the code, a sensor read that blocks the motor update, and an error path that only the author understands.

Architecture is the set of rules that keeps the drawer organized. It decides which file owns each pin, which module knows the wire format of the telemetry, and which part of the code decides what the system does after a fault. The rules matter most when something breaks. A sensor that stops answering should cause one clear behavior, such as an indicator change and a fault code in the log, and the robot keeps running. A layered firmware can be tested one layer at a time, and an unlayered one is tested by hoping.

Today you combine the drivers from earlier days into one sensor logger. The logger reads a potentiometer, reads a real-time clock over I2C, sends a framed telemetry line over the serial port, drives an LED to show its state, and recovers from a sensor fault. Each of those jobs lives in its own module, and each module has a small interface. The project tests that the system recovers when you unplug the clock, and the logs prove it.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1960s to 1970s | Structured programming, led by Edsger Dijkstra and others, argues that code should be organized into small blocks with clear entry and exit points. | Module boundaries in firmware still follow this idea. |
| 1970s to 1980s | Embedded systems move from assembly to C, which allows separate files, header declarations and modules. | C's header files are the origin of the interface files used in this lesson. |
| 1990s | Real-time operating systems and the "superloop" design, a main loop that calls each task in turn, become common in industrial controllers. | The superloop is the design you will use today, and the RTOS on Day 24 is the next step. |
| 2000s | Automotive standards, such as AUTOSAR, define layered architectures with a hardware abstraction layer, a services layer and an application layer. | Large robot and vehicle firmware uses the same layers, in a larger scale. |
| Today | Layered firmware, state machines, watchdog timers and telemetry are standard in robot controllers, medical devices and vehicles. | A reviewer will ask how the firmware behaves on a fault before asking about the features. |

The layers in this lesson are a small, informal version of the same idea. They follow the same rules that the larger standards describe.

---

## Glossary

- **Architecture:** the set of decisions about how a program is divided into parts, and how the parts communicate.
- **Layer:** a group of modules at the same level of abstraction, such as the hardware layer or the application layer.
- **HAL (hardware abstraction layer):** the layer that wraps the chip's pins, timers and peripherals behind functions with clear names.
- **Driver:** a module that controls one device, such as a clock or a sensor, using the HAL or the bus library.
- **Interface:** the declaration of what a module offers to other modules, usually a header file with function prototypes and types.
- **Coupling:** how much one module depends on the internals of another. Low coupling is the goal.
- **FSM (finite state machine):** a model with a fixed set of states, a set of inputs and a table of transitions between them.
- **Transition:** a move from one state to another, triggered by an input or a condition.
- **Superloop:** a main loop that calls each task in turn, without blocking, and repeats forever.
- **Telemetry:** data that a device sends to a monitor, such as readings, states and fault codes.
- **Fault:** an error condition that the system detects and must handle, such as a failed bus transaction.
- **Watchdog timer:** a hardware timer that resets the chip if the program does not reset it within a set time. It recovers the system from a hang.
- **Degraded mode:** a reduced operating state in which the system keeps running with fewer functions, usually with a warning.

---

### Layers and the hardware abstraction layer

A restaurant has a kitchen, a waiter and a dining room. The customer does not walk into the kitchen to cook. The waiter takes the order, and the kitchen follows a recipe. The layers keep each job in its place. Firmware works the same way, with the hardware at the bottom, drivers above it and the application at the top.

From first principles, a layered design has two rules. The upper layer may call the lower layer, but not the other way around, and an upper layer may not reach past its neighbor to touch the hardware directly. The hardware abstraction layer is the lowest level of the firmware. It is the only code that writes registers, sets pin modes or calls `analogRead`. Everything above it uses functions with names such as `hal::readAdc` and `hal::setLed`, and no upper module contains a number like 13 or 0x68.

In the worked example, the sensor logger uses the onboard LED on pin 13. If the application code calls `digitalWrite(13, HIGH)` in twelve places, moving the LED to pin 7 requires twelve edits, and one will be missed. If the application calls `hal::setLed(true)`, the change is one line in the HAL. The cost of the layer is a few extra function calls, which on the Uno take a few microseconds, and the benefit is a single place to change the wiring.

The common mistake is to call the HAL from everywhere, so the HAL becomes the whole program. The fix is to keep the HAL small and to put decisions in the layer above it. A HAL that contains the fault policy has moved the application into the wrong layer.

On the job, the HAL is the first module that a new board bring-up touches, and it is the module that changes most often when the hardware is revised. Keeping it small makes a board revision a small change.

---

### Modules and clear interfaces

A power tool has a handle, a trigger and a battery connector, and each part has a defined shape. You can replace the battery without rebuilding the handle. A firmware module is the same kind of part. Its header file is the shape, and its source file is the mechanism. Other code uses the header, and it does not look inside the source.

From first principles, a module's header declares its types and its functions. The header is what the rest of the program includes. The source file defines the functions and keeps its variables `static`, so no other file can change them by accident. In C++, the same idea uses a namespace or a class, and the rule is identical: the interface is small, and the internals are hidden.

The worked example is the RTC driver from Day 21. Its header declares one enum for the result of a read, one struct for the time and two functions. The header does not mention the register map or the 0x68 address. If the clock is replaced with a different chip, only the source file changes, and the application code stays the same. The status enum is the important part. It returns a value that the application can act on, instead of a number that the application has to decode.

The common mistake is to expose internal variables through a header, such as a global `int lastTemperature` that every file can read and write. The result is that any file can change the state, and a bug in one file breaks another in a way that is hard to trace. The fix is to make the variable `static` in the source and offer a function to read it.

On the job, interfaces are reviewed before implementations. A team agrees on the header, and the drivers are written to match it. This allows two engineers to work at the same time, and it makes the tests for each module clear.

---

### State machines

A traffic light changes between red, green and yellow in a fixed order, and each change depends on a timer or a sensor. A state machine is a way of writing that behavior in code, with each state and each allowed change listed. It replaces a tangle of flags with a single variable that says where the system is.

From first principles, a state machine has a set of states, a set of events or inputs, and a transition function that takes the current state and an input and returns the next state. The output of each state is defined too, such as the LED pattern or the telemetry state name. The advantage is that every behavior can be read from a table. The disadvantage is that a missing transition is a silent bug, so the table should be checked against the design.

The logger in this lesson has four states. Init runs once after reset and moves to Sampling when the first sensor read succeeds. Sampling is normal operation. Degraded is entered after a single failed read, and it is still reading the sensors. Fault is entered after a number of consecutive failures, and the system keeps trying to recover. The transition table is short. A failure in Sampling moves to Degraded. Three failures in a row move to Fault. A success in Fault moves to Degraded. Five seconds of continuous success in Degraded moves back to Sampling.

Here is the core of the transition logic in code:

```cpp
void onRtcResult(rtc::RtcStatus s, uint32_t now) {
  if (s == rtc::RtcStatus::Ok) {
    failures = 0;
    if (state == State::Init) {
      state = State::Sampling;
    } else if (state == State::Fault) {
      state = State::Degraded;
      okSince = now;
    } else if (state == State::Degraded && now - okSince >= RECOVERY_MS) {
      state = State::Sampling;
    }
  } else {
    if (failures < 255) failures++;
    okSince = now;
    if (failures >= FAILURE_LIMIT) {
      state = State::Fault;
    } else if (state == State::Sampling) {
      state = State::Degraded;
    }
  }
}
```

The worked example is a sequence of reads: OK, fail, fail, fail, OK, OK. The states go Sampling, Degraded, Degraded, Fault, Degraded, Degraded, and the system returns to Sampling only after five seconds of continuous success. A reviewer can check each step against the table in a minute.

The common mistake is to put the state inside several flags, such as `isFault` and `isReading` and `hasError`, which can disagree with each other. A single enum prevents that. The second mistake is to forget the default case, which leaves the system in a state it cannot leave. The compiler warns about a missing case in a `switch` on an enum class, and the warning should be treated as an error.

On the job, state machines are used for every protocol, every motor controller and every safety function. Reviewers ask for the state table first, and a test plan covers each transition.

---

### Non-blocking architecture

A chef who waits by the oven for the timer to finish cannot prepare the salad. A chef who checks the oven, prepares the salad and checks again can do both. A non-blocking design works the same way. Each task does a small amount of work and returns, so the loop can run the next task without waiting.

From first principles, a superloop calls each task in order, and each task checks its own schedule with `millis()`. A task that is due runs, and a task that is not due returns immediately. The loop's period is the sum of the tasks' work times, and the worst case is the sum of their longest runs. The design is only as good as the longest task, because one slow task delays all the others.

The worked example measures the loop. The sensor logger calls the RTC read once every 200 ms. The I2C transaction for the three bytes of time takes about 1 ms at 100 kbit/s once the address bytes and acknowledgments are counted. The telemetry line is about 40 characters at 115200 baud, which takes about 3.5 ms, because each character is 10 bits on the wire. The potentiometer read takes about 0.1 ms. The worst-case loop pass is therefore about 5 ms, which is well under the 200 ms sample period. The logger checks that estimate by recording the maximum loop time with `micros()` and reporting it in the telemetry.

The common mistake is to use `delay()` inside a task. A delay in one task stops every other task, and it hides the blocking from the reader. The second mistake is to put a long print in the loop, such as a debug dump, that takes longer than the sample period. The telemetry layer is designed for this: the frame is short and fixed.

On the job, non-blocking architecture is a requirement for any robot loop that controls motion. The control task must run on time, and the rest of the system must not delay it.

---

### Telemetry as a separate layer

A newspaper reporter writes the story, and a printer decides the page layout. The reporter does not need to know the printer's font. Telemetry works the same way. The application decides what happened, and the telemetry layer decides how to format it on the wire. If the format changes, the application does not change.

From first principles, a telemetry module takes plain values, such as the state name, the reading and the fault code, and converts them into a frame. The frame has a start character, the body, a checksum and an end of line. The Day 20 frame format is the example. Keeping the format in one module means that the checksum rule is in one place, and that a second receiver, such as a logging script, reads the same format.

The worked example uses the Day 20 frame: `$LOG,<ms>,<state>,<adc>,<fault>*XX`. The application passes the four values, and the telemetry module builds the string, computes the XOR of the body and prints it. The telemetry module does not know what the states mean. It only knows how to format the values it is given. That separation is what makes the module reusable.

The common mistake is to format the telemetry inside the application logic, so that each state prints its own message in its own style. The output then becomes inconsistent, and a parser breaks when someone adds a new state. A single telemetry function avoids this.

On the job, telemetry is the main way to see inside a deployed robot. Field logs, dashboards and fault reports are all built on a stable frame format, and changing the format breaks the tools that read it.

---

### Fault handling and the watchdog

A car with a warning light does not stop on the highway the moment a sensor fails. It shows the warning, keeps the engine running within safe limits and reports the fault. A firmware fault policy does the same. It detects the problem, chooses a safe behavior, reports it and tries to recover.

From first principles, a fault has three parts: detection, response and reporting. Detection is a check of a return code, a timeout or a range limit. The response is a state change, such as Degraded, and a safe output, such as turning off a motor. Reporting is the telemetry frame with a fault code. Recovery is the path back to normal operation after the fault clears, and it should require evidence, such as five seconds of good readings, before the system trusts the sensor again.

A watchdog is the last line of defense. The hardware timer resets the chip if the program does not reset the timer within its period. On the Uno, the watchdog is set with `wdt_enable(WDTO_2S)` from `avr/wdt.h`, and the main loop calls `wdt_reset()` each pass. If the loop hangs, for example because an I2C transaction never completes, the watchdog resets the board in two seconds. The reset is a recovery that the program cannot perform for itself.

The worked example is the I2C hang. The Arduino AVR Wire library does not give a timeout on every transaction by default, so a bus stuck low can stop the program inside `endTransmission`. Without a watchdog, the logger freezes with its LED in one state, and no telemetry arrives. With the watchdog, the board resets after two seconds, and the telemetry shows the reboot. The fault is still a problem, but the robot recovers.

The common mistake is to rely on the watchdog for ordinary faults. A watchdog reset is a crude recovery that loses state. Use the state machine for faults that can be handled in place, and use the watchdog for hangs. The second mistake is to reset the watchdog inside an interrupt or inside a task that always runs, which hides a loop that has stopped. Reset it only at the top of the main loop.

On the job, the watchdog and the fault policy are reviewed together. A safety review asks what happens when each sensor fails, and what happens when the program hangs.

---

## Hands-on exercises

1. **Set up the layers.** Create a sketch folder with `sensor_logger.ino`, `hal.h`, `hal.cpp`, `rtc_driver.h`, `rtc_driver.cpp`, `telemetry.h`, `telemetry.cpp`, `app.h` and `app.cpp`. Confirm that the Arduino IDE builds the sketch with all files. Check that no file other than `hal.cpp` contains a pin number or a register name.

2. **Test the HAL.** Move the LED from pin 13 to pin 7 by changing one line in `hal.cpp`. Build and run. Record the change and confirm the LED moves.

3. **Read the telemetry.** Run the sketch and capture 60 seconds of output. Confirm each line matches the frame format and the checksum. Write the checksum calculation for one line by hand.

4. **Trace the state machine.** Draw the state diagram from the transition code. Label each arrow with its condition. Compare it with the telemetry output during a fault test.

5. **Measure the loop.** Add a `micros()` timer around the superloop and record the maximum pass time over 60 seconds. Include the number in the README and compare it with the 200 ms sample period.

6. **Inject a fault.** While the logger runs, disconnect the DS3231's SDA wire for ten seconds, then reconnect it. Capture the telemetry and the LED pattern. Record the time of each state change.

---

## Project: a layered sensor logger with fault recovery

The goal is a repository that contains a layered firmware for the Uno with four modules, a four-state machine, framed telemetry, a watchdog and a documented fault test. It brings together the work from Days 16 to 22 and shows how a firmware is organized so that a reviewer can read it.

**Repo name:** `100-days-robotics-day-23-embedded-software-architecture`

**Repo layout:**

```
100-days-robotics-day-23-embedded-software-architecture/
├── README.md
├── sensor_logger/
│   ├── sensor_logger.ino
│   ├── hal.h
│   ├── hal.cpp
│   ├── rtc_driver.h
│   ├── rtc_driver.cpp
│   ├── telemetry.h
│   ├── telemetry.cpp
│   ├── app.h
│   └── app.cpp
├── logs/
│   ├── normal_run_60s.txt
│   ├── fault_test_10s_unplug.txt
│   └── loop_timing.csv
├── docs/
│   ├── architecture.md
│   ├── state_table.md
│   └── wiring.md
└── photos/
    └── bench.jpg
```

**Starter code, `sensor_logger/sensor_logger.ino`:**

```cpp
#include <avr/wdt.h>
#include "hal.h"
#include "rtc_driver.h"
#include "telemetry.h"
#include "app.h"

void setup() {
  wdt_enable(WDTO_2S);
  hal::init();
  rtc::init();
  telemetry::init();
  app::init();
}

void loop() {
  wdt_reset();
  app::step();
}
```

**Starter code, `sensor_logger/hal.h` and `hal.cpp`:**

```cpp
// hal.h
#pragma once
#include <stdint.h>

namespace hal {
  void init();
  uint32_t nowMs();
  void setLed(bool on);
  uint16_t readAdc(uint8_t channel);
}
```

```cpp
// hal.cpp
#include <Arduino.h>
#include "hal.h"

namespace hal {
  static const uint8_t LED_PIN = 13;

  void init() { pinMode(LED_PIN, OUTPUT); }
  uint32_t nowMs() { return millis(); }
  void setLed(bool on) { digitalWrite(LED_PIN, on ? HIGH : LOW); }
  uint16_t readAdc(uint8_t channel) { return analogRead(A0 + channel); }
}
```

**Starter code, `sensor_logger/rtc_driver.h` and `rtc_driver.cpp`:**

```cpp
// rtc_driver.h
#pragma once
#include <stdint.h>

enum class RtcStatus : uint8_t { Ok, NoAck, ShortRead };
struct RtcTime { uint8_t hour, minute, second; };

namespace rtc {
  void init();
  RtcStatus readTime(RtcTime &out);
}
```

```cpp
// rtc_driver.cpp
#include <Arduino.h>
#include <Wire.h>
#include "rtc_driver.h"

namespace rtc {
  static const uint8_t ADDR = 0x68;

  static uint8_t bcd(uint8_t b) { return (b >> 4) * 10 + (b & 0x0F); }

  void init() { Wire.begin(); }

  RtcStatus readTime(RtcTime &out) {
    Wire.beginTransmission(ADDR);
    Wire.write((uint8_t)0x00);
    if (Wire.endTransmission(false) != 0) return RtcStatus::NoAck;
    if (Wire.requestFrom(ADDR, (uint8_t)3) != 3) return RtcStatus::ShortRead;
    out.second = bcd(Wire.read() & 0x7F);
    out.minute = bcd(Wire.read() & 0x7F);
    out.hour   = bcd(Wire.read() & 0x3F);
    return RtcStatus::Ok;
  }
}
```

**Starter code, `sensor_logger/telemetry.h` and `telemetry.cpp`:**

```cpp
// telemetry.h
#pragma once
#include <stdint.h>

namespace telemetry {
  void init();
  void emit(uint32_t ms, const char *state, uint16_t adc, uint8_t faultCode);
}
```

```cpp
// telemetry.cpp
#include <Arduino.h>
#include "telemetry.h"

namespace telemetry {
  void init() { Serial.begin(115200); }

  void emit(uint32_t ms, const char *state, uint16_t adc, uint8_t faultCode) {
    char body[64];
    snprintf(body, sizeof(body), "LOG,%lu,%s,%u,%u",
             (unsigned long)ms, state, adc, faultCode);
    byte cs = 0;
    for (int i = 0; body[i] != '\0'; i++) cs ^= (byte)body[i];
    Serial.print('$');
    Serial.print(body);
    Serial.print('*');
    if (cs < 16) Serial.print('0');
    Serial.println(cs, HEX);
  }
}
```

**Starter code, `sensor_logger/app.h` and `app.cpp`:**

```cpp
// app.h
#pragma once

namespace app {
  void init();
  void step();
}
```

```cpp
// app.cpp
#include "hal.h"
#include "rtc_driver.h"
#include "telemetry.h"
#include "app.h"

namespace app {
  enum class State : uint8_t { Init, Sampling, Degraded, Fault };

  static const uint32_t SAMPLE_PERIOD_MS = 200;
  static const uint8_t  FAILURE_LIMIT    = 3;
  static const uint32_t RECOVERY_MS      = 5000;

  static State state = State::Init;
  static uint8_t failures = 0;
  static uint32_t lastSampleMs = 0;
  static uint32_t okSince = 0;
  static uint8_t lastFault = 0;

  static const char *stateName(State s) {
    switch (s) {
      case State::Init:     return "INIT";
      case State::Sampling: return "SAMPLING";
      case State::Degraded: return "DEGRADED";
      case State::Fault:    return "FAULT";
    }
    return "UNKNOWN";
  }

  static void onRtcResult(rtc::RtcStatus s, uint32_t now) {
    if (s == rtc::RtcStatus::Ok) {
      failures = 0;
      lastFault = 0;
      if (state == State::Init) {
        state = State::Sampling;
      } else if (state == State::Fault) {
        state = State::Degraded;
        okSince = now;
      } else if (state == State::Degraded && now - okSince >= RECOVERY_MS) {
        state = State::Sampling;
      }
    } else {
      lastFault = (s == rtc::RtcStatus::NoAck) ? 1 : 2;
      if (failures < 255) failures++;
      okSince = now;
      if (failures >= FAILURE_LIMIT) {
        state = State::Fault;
      } else if (state == State::Sampling) {
        state = State::Degraded;
      }
    }
  }

  void init() {
    hal::init();
    lastSampleMs = hal::nowMs();
  }

  void step() {
    uint32_t now = hal::nowMs();
    if (now - lastSampleMs < SAMPLE_PERIOD_MS) return;
    lastSampleMs = now;

    rtc::RtcTime t;
    rtc::RtcStatus s = rtc::readTime(t);
    onRtcResult(s, now);

    uint16_t adc = hal::readAdc(0);

    // LED pattern per state: solid when sampling, slow blink when degraded,
    // fast blink when faulted.
    bool led = false;
    if (state == State::Sampling) led = true;
    else if (state == State::Degraded) led = ((now / 500) % 2) == 0;
    else if (state == State::Fault)    led = ((now / 100) % 2) == 0;
    hal::setLed(led);

    telemetry::emit(now, stateName(state), adc, lastFault);
  }
}
```

The `app::init()` function calls `hal::init()` as well, and the sketch calls both. That is a deliberate double call, and the README should explain whether it is needed. The cleaner design is to call `hal::init()` only from `setup()`. Make that change as part of the project and record it in the architecture document.

**Step-by-step tasks:**

1. Create the repository and folder tree. Initialize git and commit the README skeleton.
2. Create the sketch folder with the nine files. Compile with `arduino-cli compile --fqbn arduino:avr:uno sensor_logger` and save the output to a text file in `docs/`.
3. Wire the RTC, the pot and the LED as in Day 21 and Day 18. Upload and run for 60 seconds. Save the output to `logs/normal_run_60s.txt`.
4. Measure the loop time. Add a `micros()` timer around `app::step()` in `loop()`, record the maximum over 60 seconds in `logs/loop_timing.csv`, and report it in the README.
5. Run the fault test. Disconnect the DS3231 SDA wire for ten seconds, then reconnect it. Save the output to `logs/fault_test_10s_unplug.txt`. Note the time of each state change in the README.
6. Write `docs/state_table.md` with the transition table from the lesson, and check each transition in your log.
7. Make the `hal::init()` change so that it is called only from `setup()`, and update `docs/architecture.md` with the reason.
8. Write the README with the checklist below.

**README checklist:**

- [ ] The layer diagram shows which files may call which, and no file outside `hal.cpp` contains a pin number
- [ ] The state table is in `docs/state_table.md` and matches the code
- [ ] The normal run log shows the SAMPLING state with a checksum on each line
- [ ] The fault test log shows the transitions from SAMPLING to DEGRADED to FAULT and back, with the times
- [ ] The maximum loop time is reported and compared with the sample period
- [ ] The watchdog behavior is described, including a note on whether it was triggered during the test
- [ ] The double `hal::init()` call is documented and removed, with the reason stated

**Done when:**

- The sketch compiles with all nine files and no warnings that point to the architecture
- The normal run shows SAMPLING for the whole 60 seconds with valid checksums
- The fault test shows the logger reaching FAULT after three failed reads and returning to SAMPLING after recovery, with the recovery time at least five seconds after the reconnect
- The loop timing log shows a maximum pass time well under 200 ms
- The repository has at least three commits, and the README numbers match the logs

**Evidence to commit to GitHub:** the nine sketch files, the three logs, the architecture document, the state table, the compile output, the wiring document, the photo and the README. Do not commit build folders.

The project looks small, but the layers, the state table and the fault test are the parts a reviewer checks first. A reviewer who sees the state table and the matching log can trust the firmware more than one who sees a long single file.

---

## Check your understanding

1. A student adds `digitalWrite(13, HIGH)` to the application file to light the LED. Explain what rule this breaks, and name the one-line change that avoids it.
2. A state machine has states A, B and C. Describe what happens if the transition table has no entry for state C with an input that never occurs, and why that is still a risk.
3. The RTC read takes 0.5 ms and the telemetry frame takes 3.5 ms. Explain why the non-blocking loop is safe with a 200 ms sample period, and what would make it unsafe.
4. A team wants the telemetry format to change from one line per sample to one line per state change. Name the module that changes, and the modules that do not.
5. The Wire library blocks on a stuck bus. Explain why the state machine alone cannot recover the system, and what hardware feature does.
6. A sensor fails once and recovers 100 ms later. The state machine moves to DEGRADED and back to SAMPLING after five seconds of good reads. Explain why the design requires five seconds, and what would go wrong with a one-read recovery.

---

## Answers

1. The rule is that only the HAL touches pins and registers. The one-line change is to call `hal::setLed(true)` from the application, which keeps the pin number inside the HAL.
2. If an input arrives in state C that has no transition, the system stays in C with no defined response. The risk is that the input is silently ignored, and the system does not recover. The design should define a transition or a default for every state and input, and a test should cover the missing case.
3. The loop's worst case is about 4 ms, which is far below the 200 ms period, so every task runs on time. It would become unsafe if a task blocked for longer than the period, such as a long print or a delay, or if many sensors each took several milliseconds.
4. The telemetry module changes, and its interface may stay the same if the application still passes the same values. The application changes only if it must report different events. The sensor drivers and the HAL do not change.
5. The state machine runs only when the loop runs, and a blocked `endTransmission` never returns, so the state machine never gets a result. The watchdog resets the chip if the loop stops calling `wdt_reset`, which recovers the system from a hang.
6. The five-second wait prevents the system from flapping between DEGRADED and SAMPLING on a noisy bus. With a one-read recovery, an intermittent fault would cause a state change on every good read, which makes the telemetry hard to read and can hide a real fault.

---

## Sources

*Read:*

- The Wikipedia article titled **Software architecture**, for the general concepts of layers and modules
- The Wikipedia article titled **Finite-state machine**, for the state machine model
- The Wikipedia article titled **Watchdog timer**, for the hardware recovery mechanism
- The Wikipedia article titled **Hardware abstraction**, for the HAL concept
- The Wikipedia article titled **Embedded system**, for the overview of firmware design
- The Arduino AVR core documentation on the watchdog, found in the avr-libc manual under the `avr/wdt.h` section. Search for "avr-libc wdt.h" to find it
- The AUTOSAR layered software architecture documentation from the AUTOSAR consortium. Search for "AUTOSAR layered software architecture" and read the overview

*Tutorials:*

- The official Arduino documentation on multi-file sketches, in the Arduino IDE 2 documentation under sketches and libraries
- The GeeksforGeeks article on finite state machines. Search the GeeksforGeeks site for "finite state machine"

*Watch:*

- The Arduino YouTube channel, which has videos on Arduino project structure. Search for "Arduino project structure tutorial"
- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, for lectures on embedded design and project structure. Search for the course title on the MIT OpenCourseWare site

*Practice:*

- Wokwi at wokwi.com, which supports multi-file Arduino projects in its editor. Build the logger there to test the state machine without the hardware, then run the fault test on the bench. The free tier is enough for this lesson, but it does not simulate a stuck I2C bus, so the fault test must be done on real hardware.

---

## Tomorrow

Day 24 moves the same ideas onto a real-time operating system. You will use an ESP32 with FreeRTOS, create tasks, pass data through queues, protect shared variables with mutexes and semaphores, and see how a race condition appears when two tasks share data without protection. The superloop from today is the design that an RTOS replaces, and the state machine and telemetry layers carry over directly into the tasks you will write.
