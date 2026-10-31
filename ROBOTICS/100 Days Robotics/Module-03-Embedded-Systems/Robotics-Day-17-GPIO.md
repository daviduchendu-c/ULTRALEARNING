# Day 17: GPIO

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 3 hours

**Level:** beginner. You need Day 16 (the Arduino IDE, the Uno and the `millis()` timing pattern) and Day 6 (voltage and Ohm's law). Today you wire real buttons and LEDs, so check every connection against the schematic before you apply power.

**By the end of this day, you will have:**

- A working circuit with three buttons and three LEDs on an Arduino Uno, wired with internal pull-up resistors
- A bounce test that records raw button edges and shows the contact bounce in a serial log
- A debounced three-button menu that switches LEDs using a small state machine
- A measured voltage at a button pin, in both pressed and released states, that matches the logic-level calculation in this lesson

**What you need:**

- Arduino Uno Rev3 and Arduino IDE 2.3.x with the `Arduino AVR Boards` core at version 1.8.6, as in Day 16
- Three tactile pushbuttons, 6 mm, about $3 for a pack of 10
- Three 5 mm LEDs in different colors, about $3 for a pack
- Three 220 ohm resistors for the LEDs, about $0.10 each
- A breadboard and jumper wires, about $5 to $8
- A digital multimeter with DC volts, about $15 to $25
- Optional: a 10 kOhm resistor for the external pull-up exercise, about $0.10

**Prerequisites:**

- Day 16: pin modes, `digitalWrite`, `millis()` and the Uno's pin map
- Day 10: logic levels as voltage ranges, not exact values
- Comfort reading a breadboard layout and a simple schematic

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Digital signals and logic levels](#digital-signals-and-logic-levels)
- [Driving LEDs safely](#driving-leds-safely)
- [Pull-up and pull-down resistors](#pull-up-and-pull-down-resistors)
- [Buttons and switch wiring](#buttons-and-switch-wiring)
- [Debouncing](#debouncing)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a debounced three-button LED menu](#project-a-debounced-three-button-led-menu)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Every robot needs to know whether something has happened. A bumper switch has been pressed, a limit switch has reached the end of its travel, a start button has been pushed, or a status line from another board has gone high. Each of these is a digital signal: a wire that is either high or low. The microcontroller reads that wire through a GPIO pin, and it also drives other wires high or low to turn on an LED, a relay or a motor driver input.

GPIO looks trivial. Connect a button, read a pin, and you are done. In practice, most of the bugs on a new robot are GPIO bugs. A pin floats and reads random values. A button bounces and triggers three times. An LED is driven hard enough to overheat a pin. A pin is connected to a signal that is 12 V, and the chip is destroyed. These failures are predictable once you know the rules, and this lesson gives you those rules.

Today you will build a three-button menu with LEDs. The circuit is small, but it covers every concept a robot uses for discrete inputs and outputs. You will also measure the voltages with a multimeter, so the numbers in this lesson become something you have seen on your own bench.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1930s to 1950s | Relays and mechanical switches control machines, and the first digital logic is built from electromagnetic relays. | Relays still appear in robots for high-current switching, and they teach the same idea of a discrete on or off signal. |
| 1960s | Transistor-transistor logic (TTL) defines standard logic voltages, with a high level near 5 V. | The 5 V logic of the Uno comes from this era. |
| 1970s | CMOS logic becomes common, with its high-impedance inputs and low static power. | Modern microcontroller pins are CMOS, which is why floating inputs are a problem. |
| 1980s to 1990s | Microcontrollers ship with built-in pull-up resistors on their pins. | You can wire a button with no external resistor, which is why the Uno's `INPUT_PULLUP` mode exists. |
| 2000s to today | Low-voltage pins at 3.3 V and 1.8 V become standard on sensors and single-board computers. | Level shifting is now part of most robot wiring, as later days in this module show. |

The logic levels and the pull-up idea have not changed much in sixty years. The voltages have dropped and the pin counts have grown, but the wiring rules are the same.

---

## Glossary

- **GPIO (general-purpose input/output):** a pin that the program can configure as an input or an output.
- **Logic level:** the voltage range that the chip reads as high or low. It is a range, not a single value.
- **VIH and VIL:** the minimum input voltage that counts as high (VIH) and the maximum input voltage that counts as low (VIL).
- **Pull-up resistor:** a resistor that connects a pin to the positive supply, so the pin reads high when nothing else drives it.
- **Pull-down resistor:** a resistor that connects a pin to ground, so the pin reads low when nothing else drives it.
- **Floating pin:** an input that is not connected to a defined voltage, which can read high or low at random.
- **Contact bounce:** rapid on and off transitions that a mechanical switch produces for a few milliseconds when it closes or opens.
- **Debouncing:** a method, in hardware or software, that ignores the bounce and reports one clean transition.
- **Active low:** a signal that is considered active when it is low. A button wired to a pull-up is active low.
- **Current limiting resistor:** a resistor in series with an LED that limits its current to a safe value.
- **Forward voltage (Vf):** the voltage an LED drops when it conducts. A red LED is about 1.8 V to 2.2 V, and a blue LED is about 3.0 V to 3.4 V.
- **State machine:** a program structure that moves between a fixed set of states based on inputs.

---

### Digital signals and logic levels

A light switch in a house has two positions. The light is on or off, and nothing in between counts. A digital signal works the same way. The chip does not read the exact voltage on a wire. It decides whether that voltage is in the high range or the low range. The boundary between the ranges is set by the chip's datasheet, and there is a gap between them where the value is not guaranteed.

From first principles, the ATmega328P datasheet defines the input thresholds as fractions of the supply voltage VCC. The input is low when it is at or below VIL, which is 0.3 times VCC at most. The input is high when it is at or above VIH, which is 0.6 times VCC at least. On the Uno, VCC is 5 V, so VIL is at most 1.5 V and VIH is at least 3.0 V. Between 1.5 V and 3.0 V, the input is undefined, and you should not design a circuit that sits in that range.

The worked example is a button wired to a pin. If the button is open and the pin is pulled to 4.9 V by a pull-up, the pin reads high. If the button is closed and the pin is pulled to 0.1 V, the pin reads low. The gap between 1.5 V and 3.0 V is never reached in either state, so the reading is reliable. A fault that drops the pin to 2.2 V would be unreliable and should be found with a multimeter.

The common mistake is to assume a 3.3 V sensor and a 5 V Uno can share a wire without a conversion. A 3.3 V output driving a 5 V input is usually safe, because 3.3 V exceeds the 3.0 V threshold. The reverse is not safe. A 5 V output driving a 3.3 V input can exceed that chip's absolute maximum rating and damage it. Check both datasheets before you connect any signal between boards.

Logic levels are the common language of robot wiring. Limit switches, encoders, Hall sensors and most digital sensors use them, and the voltage rules in this section apply to all of them.

---

### Driving LEDs safely

A water tap with a fixed pressure has a valve that limits how much water flows. A resistor in series with an LED does the same job for current. Without the resistor, the LED's current rises very quickly once its forward voltage is reached, and the LED or the pin is destroyed.

The first principle is Ohm's law applied to the part of the circuit that is not the LED. The LED has a roughly fixed forward voltage, so the resistor takes the rest of the supply voltage. The current is I = (VCC − Vf) / R. For a red LED with Vf of 2.0 V, a 5 V supply and a 220 ohm resistor, the current is (5 − 2.0) / 220 = 0.0136 A, or about 13.6 mA. That is a bright, safe level for a small LED and within the Uno pin's recommended limit of about 20 mA.

The power in the resistor is P = I² × R, which gives 0.0136² × 220 = 0.041 W. A standard quarter-watt resistor handles that easily. A 22 ohm resistor on the same circuit would carry 0.136 A, which is above the pin's safe limit, and the pin would stress over time even if it did not fail at once.

The common mistake is to assume that a pin's voltage sets the LED's brightness on its own. The LED's current sets its brightness, and a missing resistor allows the current to be set by the pin's internal resistance, which varies from chip to chip. The sign of a missing resistor is an LED that is very bright for a moment and then dim, or a pin that gets warm. Check the resistor in the circuit with the multimeter in ohms mode, with the board unplugged.

LED driving is the simplest form of actuator control. Status lights on motor drivers, robot indicators and warning beacons all use the same calculation, and high-power LEDs add a constant-current driver instead of a resistor.

---

### Pull-up and pull-down resistors

A door with a spring closes by itself when nobody pushes it. A pull-up resistor is a spring for an electrical signal. It pulls the pin to a known level when no other device is connecting it to something else. Without it, a button that is open leaves the pin floating, and the pin reads whatever random noise is nearby.

From first principles, a pull-up connects the pin to VCC through a resistor R. When the switch is open, no current flows through R, so the pin sits at VCC. When the switch closes the pin to ground, current flows from VCC through R to ground, and the pin is pulled to ground. The resistor's value sets the trade-off. A small value gives a strong pull that resists noise but draws more current when the button is pressed. A large value draws little current but lets noise in more easily.

The ATmega328P has internal pull-up resistors of roughly 20 kOhm to 50 kOhm on each pin, and the Arduino function `pinMode(pin, INPUT_PULLUP)` turns them on. The worked example uses 35 kOhm as the middle of the range. At 5 V, the current through the pull-up when the button is pressed is 5 / 35,000 = 0.00014 A, or 0.14 mA. That is small, so the internal pull-up is enough for a button. An external 10 kOhm resistor would draw 0.5 mA, which is also fine, and it gives a stronger pull on a long wire.

Pull-down resistors work the opposite way, connecting the pin to ground and making the pressed switch connect it to VCC. Pull-downs are common on boards where the default level must be low. On the Uno, internal pull-downs are not available on the ATmega328P, so an external resistor is needed if you want one.

The common mistake is to forget the pull-up and see a button that reads randomly. A floating input in CMOS can draw extra current when its voltage sits in the undefined range, which heats the chip and wastes power. The fix is one line of code, `pinMode(pin, INPUT_PULLUP)`, and the way to spot the problem is a pin that changes state when you touch the wire with a finger.

On the job, pull-ups appear on every I2C bus, every open-drain output and every reset line. Hardware teams check them in the schematic review before anything else, because a missing pull-up is one of the most common board faults.

---

### Buttons and switch wiring

A doorbell button is simple: press it and the circuit closes. The electrical design around it decides whether the signal is clean. A switch has two terminals, and it connects them when pressed. Wired to a microcontroller, one terminal goes to the pin and the other goes to ground, and the pull-up holds the pin high until the button is pressed.

The first principle is active-low logic. With `INPUT_PULLUP`, the pin reads HIGH when the button is released and LOW when it is pressed. Many beginners reverse this in code and then wonder why the LED turns on when they release the button. The rule is simple: in the Arduino sketch, check `digitalRead(pin) == LOW` for a press when the pin uses `INPUT_PULLUP`.

Tactile switches have four legs, and two pairs are connected internally. Before you wire one, check which pair connects when pressed. Pressing the switch on a breadboard should show continuity between the two terminals on the same side. The worked example is the reading with a multimeter in continuity mode. The released button shows open, and the pressed button shows a short of about 0.1 ohm or less.

The common mistake is to wire the button to 5 V without a resistor, which shorts the supply when the button is pressed. This is the same as the shoot-through problem in the motor lesson, and at the Uno's current limits the result is a hot switch and a damaged supply. The safe wiring is always pin to switch, switch to ground, with the internal or external pull-up on the pin.

Buttons are the simplest human interface on a robot, and they are the default way to start, stop or select a mode in a field test. A bumper on a mobile robot is a button in a different housing.

---

### Debouncing

A ball dropped on a hard floor bounces several times before it settles. A mechanical switch behaves the same way when its contacts close. The metal contacts strike each other and separate for a few milliseconds, so the electrical signal jumps between high and low several times before it stays low. The microcontroller is fast enough to see every jump, and a single press looks like several presses.

From first principles, the bounce lasts from a fraction of a millisecond to around 20 ms, depending on the switch. The fix is to read the pin, wait for the signal to become stable, and only then accept the change. The simplest software method checks whether the pin has been in the same state for a fixed time, such as 20 ms, using `millis()`. Each time the pin changes, the program records the time, and it accepts the new state only when the pin has held that state for the full interval.

Here is the logic in pseudocode. Store the last raw reading and the time of the last raw change. If the raw reading changes, record the new time. If the reading has stayed the same for 20 ms, and it differs from the stable state, update the stable state and report one event. This approach does not block the loop, which is important for a robot that has to read sensors and drive motors at the same time.

The worked example is the bounce test. If a switch bounces for 8 ms and the program reads it every 1 ms, the raw signal may show around 8 changes for one press. With a 20 ms debounce window, the program sees the raw signal change at the start of the bounce, waits 20 ms, and reports one event after the contacts have settled. The test should show exactly one event per press.

The common mistake is to use `delay(20)` for debouncing. That works for one button, but it blocks the rest of the loop for 20 ms every time, which is harmful in a robot. The second mistake is to debounce only on the press and not the release, which gives a false release on a bouncing contact. The check is to run the bounce test with both press and release.

On the job, debouncing is done in hardware with a capacitor and a Schmitt-trigger input, or in firmware with a state machine. Hardware debouncing is more reliable in noisy environments, and firmware debouncing is cheaper and easier to change.

---

## Hands-on exercises

1. **Verify the voltage levels.** Wire one button to pin 2 with `INPUT_PULLUP`. Use the multimeter in DC volts mode with the black lead on GND. Record the pin voltage with the button released and with the button pressed. Expect a value close to 5 V released and close to 0 V pressed.

2. **Check the LED current.** Wire a red LED through a 220 ohm resistor to pin 8. Turn it on with `digitalWrite(8, HIGH)`. Measure the voltage across the resistor. Compute the current with Ohm's law and compare it with the calculation in the LED section.

3. **Run the raw button log.** Upload a sketch that prints the button state and the time of each change, with no debounce. Press the button slowly and then quickly. Save the serial output to a file and count the changes for each press.

4. **Add the debounce.** Write a debounced reader using `millis()`. Run the same test and confirm one event per press. Record the minimum stable time that works for your switch.

5. **Test the floating pin.** Remove the `INPUT_PULLUP` line and leave the button wired. Touch the wire with a finger and observe the serial output. Write one sentence on what you saw and why.

6. **Build the menu.** Connect three buttons to pins 2, 3 and 4, and three LEDs through resistors to pins 8, 9 and 10. Program the menu from the project and demonstrate each mode.

---

## Project: a debounced three-button LED menu

The goal is a repository that shows a working debounced input, a state machine and the bounce evidence that proves the debounce works. The circuit is small, but the repository will show that you understand inputs and outputs well enough to handle a real robot panel.

**Repo name:** `100-days-robotics-day-17-gpio`

**Repo layout:**

```
100-days-robotics-day-17-gpio/
├── README.md
├── bounce_test/
│   └── bounce_test.ino
├── menu/
│   └── menu.ino
├── logs/
│   ├── raw_bounce_log.txt
│   └── debounced_log.txt
├── docs/
│   ├── schematic.png
│   └── wiring_notes.md
└── photos/
    └── breadboard.jpg
```

**Starter code, `bounce_test/bounce_test.ino`:**

```cpp
const int BUTTON_PIN = 2;
int lastState = HIGH;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  Serial.begin(115200);
  Serial.println("ready, press the button");
}

void loop() {
  int state = digitalRead(BUTTON_PIN);
  if (state != lastState) {
    Serial.print(millis());
    Serial.print(" ms  ");
    Serial.println(state == LOW ? "PRESSED" : "RELEASED");
    lastState = state;
  }
}
```

**Starter code, `menu/menu.ino`:**

```cpp
const int BUTTON_PINS[3] = {2, 3, 4};
const int LED_PINS[3] = {8, 9, 10};
const unsigned long DEBOUNCE_MS = 20;

int stableState[3];
int rawState[3];
unsigned long changedAt[3];
int activeLed = 0;

void setup() {
  for (int i = 0; i < 3; i++) {
    pinMode(BUTTON_PINS[i], INPUT_PULLUP);
    pinMode(LED_PINS[i], OUTPUT);
    stableState[i] = HIGH;
    rawState[i] = HIGH;
    changedAt[i] = 0;
  }
  Serial.begin(115200);
}

void loop() {
  unsigned long now = millis();
  for (int i = 0; i < 3; i++) {
    int reading = digitalRead(BUTTON_PINS[i]);
    if (reading != rawState[i]) {
      rawState[i] = reading;
      changedAt[i] = now;
    }
    if (now - changedAt[i] >= DEBOUNCE_MS && rawState[i] != stableState[i]) {
      stableState[i] = rawState[i];
      if (stableState[i] == LOW) {       // press event, active low
        activeLed = i;
        for (int j = 0; j < 3; j++) {
          digitalWrite(LED_PINS[j], j == activeLed ? HIGH : LOW);
        }
        Serial.print("button ");
        Serial.print(i);
        Serial.println(" pressed");
      }
    }
  }
}
```

**Step-by-step tasks:**

1. Create the repository and the folder tree. Initialize git and commit the README skeleton.
2. Wire the circuit on a breadboard. Use the schematic in `docs/` and draw it by hand or with any free tool. Take a photo of the breadboard for `photos/`.
3. Upload `bounce_test` and press the button fifty times, both quickly and slowly. Save the serial output as `logs/raw_bounce_log.txt`. Count the number of changes that appear for each press, and write the average in the README.
4. Upload `menu` and run it. Confirm that each press lights one LED and turns off the others.
5. Modify `menu` to print the debounced events to a log. Press each button ten times and save the output as `logs/debounced_log.txt`. Count the events and confirm that each press produces one event.
6. Test the debounce window. Change `DEBOUNCE_MS` to 2, then 5, then 20. Note the number of false events at each value and write a sentence on the trade-off.
7. Write the wiring notes in `docs/wiring_notes.md`, including the pull-up choice, the resistor values and the pin numbers.

**README checklist:**

- [ ] The circuit and pin map are listed
- [ ] The LED resistor calculation is written out with the forward voltage used
- [ ] The raw bounce count and the debounced event count are both reported
- [ ] The debounce window chosen, and why, is stated
- [ ] The repository contains the photo and the schematic
- [ ] One paragraph describes a fault you found and how you fixed it

**Done when:**

- The bounce test log shows more than one change for at least one press, and the debounced log shows exactly one event per press across the ten-press test
- The three buttons and three LEDs behave as described in the menu sketch
- The README contains the pull-up and resistor calculations with the numbers
- The repository has at least three commits, and the logs are committed as text files

**Evidence to commit to GitHub:** both sketches, both log files, the schematic image, the breadboard photo, the wiring notes and the README. Do not commit the build folders.

This project is practical, and it is light on new tools. The evidence is the point. A raw log shows bounce, and a debounced log shows one event per press. A reviewer can check both quickly, and those two files tell them more about your skill than a description would.

---

## Check your understanding

1. A pin is wired to a button with no pull-up and no pull-down. The button is released, and the serial monitor shows random HIGH and LOW values. Explain the cause in terms of the chip's input behavior, and name the two fixes.
2. You measure 2.4 V on an input pin of a 5 V Uno and the chip reads it inconsistently. Explain why, using the logic-level thresholds.
3. A student uses `digitalWrite` on a pin with a 10 ohm resistor to drive a red LED from 5 V, and the LED is very bright and the pin gets warm. Compute the current and say what changes you would make.
4. A button press produces four events in the serial log. Describe what causes the extra events and how you would confirm the cause with a log.
5. You want a debounce that works while the robot drives its motors and reads an encoder at 1 kHz. Explain why `delay(20)` is the wrong choice, and describe what the code does instead.
6. Your debounced button works on the bench but fails on a robot with a long wire and a motor nearby. Name two things you would check first.

---

## Answers

1. The chip's input is high impedance, so with nothing connected, the pin takes whatever voltage the surrounding noise and leakage produce. Fix it with an internal pull-up with `INPUT_PULLUP`, or with an external pull-up or pull-down resistor.
2. 2.4 V is between VIL, which is at most 1.5 V on a 5 V Uno, and VIH, which is at least 3.0 V. The input is in the undefined range, so the reading is not guaranteed and can change from one read to the next.
3. The current is (5 − 2.0) / 10 = 0.3 A, which is far above the pin's recommended limit of about 20 mA and can damage the pin. Use a 220 ohm resistor, which gives about 13.6 mA.
4. Contact bounce is the most likely cause. Log the raw changes with timestamps; a bounce shows several changes within a few milliseconds of each other. A press that bounces produces a cluster of changes, and a clean switch produces one.
5. `delay(20)` stops the whole loop for 20 ms, so the encoder reads and motor updates are missed. The debounce code instead records the time of each change with `millis()` and checks the stable time on every pass of the loop, which never blocks.
6. Check the grounding of the button and the sensor, which is a common source of noise pickup on long wires. Then check the pull-up value, since a long wire and a motor can add noise that a weak pull-up cannot overcome. A twisted pair, a shorter wire or an added RC filter are the usual fixes.

---

## Sources

*Read:*

- The Wikipedia article titled **General-purpose input/output**, for the GPIO concept and its uses
- The Wikipedia article titled **Pull-up resistor**, for the pull-up and pull-down theory
- The Wikipedia article titled **Logic level**, for the voltage ranges that digital logic uses
- The Wikipedia article titled **Switch**, which has a section on contact bounce
- The ATmega328P datasheet by Microchip Technology, free from the Microchip website. Read the sections on the I/O ports and the electrical characteristics table for VIH and VIL
- The Arduino Language Reference at docs.arduino.cc, for `pinMode`, `digitalRead` and `digitalWrite`

*Tutorials:*

- The official Arduino tutorial on Digital Input Pullup, found in the Arduino documentation under Built-in Examples
- The GeeksforGeeks article on digital electronics basics, for an overview of logic levels. Search the GeeksforGeeks site for "logic levels digital electronics"

*Watch:*

- The Arduino YouTube channel, which has short videos on button inputs and LEDs. Search for "Arduino button input tutorial"
- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, for lectures on digital inputs and interfacing. Search for the course title on the MIT OpenCourseWare site

*Practice:*

- Wokwi at wokwi.com, which includes an Arduino Uno, buttons and LEDs. Build the three-button menu there first, then build it on the bench. The free tier is enough for this lesson.

---

## Tomorrow

Day 18 moves from digital signals to analog and timed signals. You will read a potentiometer with the analog-to-digital converter, set LED brightness with PWM, and use the hardware timers that generate both. The debounce timing from today is a simple form of the timer logic you will meet there, and the logic-level thinking you practiced here applies directly to the ADC input range.
