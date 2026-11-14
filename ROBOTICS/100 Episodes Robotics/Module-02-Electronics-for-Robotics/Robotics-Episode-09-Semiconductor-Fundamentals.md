# Day 9: Semiconductor Fundamentals

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 3 hours
**Level:** beginner to intermediate. You need Day 8's diode and relay material and Day 6's Ohm's law. Today you use transistors as switches, which is the most common role a semiconductor plays on a robot.

**By the end of this day, you will have:**

- A physical picture of the PN junction and why diodes and transistors behave as they do
- A base-resistor calculation for an NPN transistor that saturates fully, and a measured proof on the bench
- A working circuit in which an Arduino Uno switches a 5 V DC fan or motor through a transistor, with a flyback diode
- A logic-level check that shows whether a 5 V signal and a 3.3 V board can talk safely

**What you need:**

- Everything from Days 6 to 8 (breadboard, DMM, resistor assortment, LEDs, 1N4007 diodes, pushbuttons, 100 µF capacitor)
- Arduino Uno R3 (ATmega328P), about $25 official or about $10 for a clone, with Arduino IDE 2.3 or newer
- A Raspberry Pi Pico (RP2040), board version Pico (original) or Pico 2, for the logic-level exercise. Pico runs at 3.3 V logic. Check that you have the 3.3 V version before you wire it to a 5 V source. The Pico is about $4 to $5. If you run MicroPython on it, install the current MicroPython release for the Pico from the Raspberry Pi documentation site.
- 2N2222 NPN transistor, pack of 10 (about $3). Use the TO-92 package.
- 5 V DC brushed motor or small 5 V fan rated under 150 mA, about $3 to $5, for the switching exercise
- Logic-level N-channel MOSFET, such as IRLZ44N (about $2 to $3). Check the datasheet before use.
- Resistors: 470 Ω, 620 Ω, 1 kΩ, 2 kΩ, 10 kΩ, 100 Ω and 10 kΩ again (about $3 extra)
- Python 3.10 or newer, and `pytest` 8.x for the project

**Prerequisites:**

- Day 8: diodes, flyback, pushbuttons and the Arduino sketch structure
- Day 7: voltage dividers
- Basic C-style Arduino code (`pinMode`, `digitalWrite`, `analogWrite`)

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [The PN junction](#the-pn-junction)
- [Diodes](#diodes)
- [Bipolar junction transistors (BJTs)](#bipolar-junction-transistors-bjts)
- [MOSFETs](#mosfets)
- [Transistor switching](#transistor-switching)
- [Pull-up and pull-down resistors](#pull-up-and-pull-down-resistors)
- [Logic-level switching](#logic-level-switching)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a transistor switch calculator](#project-a-transistor-switch-calculator)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Almost every control signal on a robot passes through a semiconductor. A microcontroller pin can switch a LED, but it cannot drive a motor. A transistor closes that gap: a small signal controls a large current, and the transistor carries the load. Motor drivers, battery protection, power regulators and the logic chips themselves are built from the same material and the same junction.

Robot builders reach for transistors when a pin is too weak, when a circuit has to switch quickly or when two voltage levels must be bridged. Choosing a transistor takes a calculation, and most failures on a first robot come from a transistor that was driven halfway, so it heated up and died. This lesson gives you the physics that explains the failure and the arithmetic that prevents it.

---

## Short history

The semiconductor story moved from a laboratory curiosity to the basis of every chip in about 80 years.

| Year | Event | What it changed |
|---|---|---|
| 1874 | Ferdinand Braun notes that certain metal sulfide crystals rectify current | The first observation of a one-way conducting material. |
| 1904 | John Ambrose Fleming patents the vacuum diode | The first electronic valve. It was the main way to rectify signals for decades. |
| 1947 | John Bardeen, Walter Brattain and William Shockley build the point-contact transistor at Bell Labs | The first solid-state amplifier, a replacement for the vacuum tube. |
| 1951 | Shockley describes the junction transistor | The bipolar junction transistor (BJT) that most designs used for many years. |
| 1959 | Jean Hoerni develops the planar process for making transistors | Transistors can be made on flat, protected silicon, which suits mass production. |
| 1961 | Fairchild Semiconductor sells the first commercial silicon integrated circuits | Several transistors share one chip, the start of the modern chip industry. |
| 1960 | Mohamed Atalla and Dawon Kahng demonstrate the MOSFET at Bell Labs | The field-effect transistor that dominates modern digital logic. |

---

## Glossary

- **Semiconductor:** a material with a conductivity between a conductor and an insulator, which can be changed by doping and by voltage.
- **Doping:** adding small amounts of impurity atoms to a semiconductor to create extra electrons (n-type) or extra holes (p-type).
- **PN junction:** the boundary between p-type and n-type material. It forms the basis of the diode and the transistor.
- **Forward bias:** a voltage that lowers the barrier at a PN junction so current flows.
- **Reverse bias:** a voltage that raises the barrier, so almost no current flows.
- **BJT (bipolar junction transistor):** a three-terminal device whose base current controls a larger collector current.
- **MOSFET (metal-oxide-semiconductor field-effect transistor):** a transistor whose gate voltage controls a channel between drain and source, with almost no DC gate current.
- **Saturation:** the state of a BJT when it is fully on, with a small voltage across the collector and emitter.
- **Forced gain:** the ratio of collector current to base current that a design assumes, usually set lower than the transistor's minimum gain to ensure saturation.
- **R_DS(on):** the drain-to-source resistance of a MOSFET when it is fully on.
- **Logic level:** the voltage range that a digital input reads as high or low.

---

## The PN junction

A semiconductor such as silicon has four electrons in its outer shell, and in pure form, it holds them tightly. Adding a small amount of phosphorus, which has five outer electrons, leaves one electron free to move. That is n-type material. Adding boron, which has three, leaves a missing electron, or hole, that acts as a positive carrier. That is p-type material.

Think of two rooms. One room has extra people who can move freely, and the other has empty chairs that people can fill. Where the rooms meet, some people wander across and fill the chairs, and they leave a thin zone with no free people or empty chairs in the middle. That zone is the depletion region, and it forms a barrier.

Apply a forward voltage, positive on the p side and negative on the n side, and the barrier shrinks. Carriers cross easily, and current flows. Apply a reverse voltage, and the barrier grows. Almost nothing crosses, and only a tiny leakage current flows until the reverse breakdown voltage is reached.

For silicon at room temperature, the barrier needs about 0.6 to 0.7 V of forward bias before the current is significant. That number is the forward voltage you measured on the diode in Day 8. Germanium is about 0.3 V, which is why it appeared in early radios. Silicon is used in most robot electronics because it is cheap, stable and handles higher temperature.

The common mistake is treating the junction as a perfect switch. It is not. Above the forward voltage, current rises very steeply with voltage, so a small change in voltage has a big effect on current. That is why diodes need a series resistor when they conduct from a stiff source. The limitation is temperature: the forward voltage falls by about 2 mV per degree Celsius, so a hot diode conducts more easily.

The PN junction is the building block of diodes, BJTs, MOSFETs, LEDs and solar cells, and it is why nearly every robot chip is made from doped silicon.

---

## Diodes

A diode is a single PN junction in a package with two leads, anode and cathode. It conducts from anode to cathode when forward biased, and it blocks in reverse until breakdown. In Day 8 you used diodes as flyback clamps and reverse-polarity protectors. The physics is now clear. The forward voltage is the barrier, and the reverse voltage must stay below the breakdown rating.

A rectifier diode such as the 1N4007 is built for low frequency, with 1 A current rating and 1,000 V reverse rating. A signal diode such as the 1N4148 is built for speed and low capacitance, and it suits logic and small-signal clamps. A Zener diode is designed to break down at a set reverse voltage, usually to clamp a supply at a known level.

Consider a Zener diode rated at 5.1 V in a 9 V circuit with a 1 kΩ series resistor. The Zener clamps its own voltage at about 5.1 V in reverse. The current through the resistor is `(9 − 5.1) / 1000 = 3.9 mA`, and the Zener carries that current if the load draws nothing. The resistor must carry the difference, so if the load changes, the Zener current changes.

The common mistake is ignoring the forward voltage when you compute a supply level. Two diodes in series drop about 1.4 V, not 0 V. The limitation is that a diode always drops voltage, so at low supply voltages (3.3 V and below) it can take a large fraction of the supply. Schottky diodes, with lower forward drop, are used in those cases.

Diodes sit in every motor driver, in every battery input that needs reverse-polarity protection and in the rectifiers inside every robot's wall adapter.

---

## Bipolar junction transistors (BJTs)

A BJT is two PN junctions placed back to back, forming an NPN or PNP device. An NPN has an n-type emitter, a thin p-type base and an n-type collector. It has three terminals: base, collector and emitter. The base current controls the collector current.

A faucet on a water main works the same way. A small turn of the handle lets a large flow through the main. The base is the handle, and the collector-to-emitter path is the main.

In the active region, the relationship is `I_C = β × I_B`, where β (often written h_FE) is the current gain. A 2N2222 has a gain that varies with current and temperature, often 100 or more at low current, and it drops at higher current. The datasheet lists the minimum and typical gain at several currents. Never rely on a single number from memory.

The base-emitter junction behaves like a diode. It needs about 0.6 to 0.7 V to conduct, and when it does, the base-emitter voltage stays close to that value. That is why the base needs a series resistor: the base-emitter path is a stiff diode, so the base current is set by the resistor, not the source.

Build a common-emitter switch. The emitter goes to ground, the load sits between the supply and the collector, and the base connects to the control pin through a resistor. When the base voltage is high enough, the collector current flows through the load. When the base is low, almost nothing flows.

The common mistake is connecting the base directly to a microcontroller pin with no resistor. The base-emitter junction is a diode, so the pin sees almost no resistance once the base is at 0.7 V, and it can source a current far above its limit. Another mistake is confusing the pinout. TO-92 transistors can use different pin orders depending on the part, so check the datasheet before you wire it. The limitation of a BJT as a switch is the voltage drop and heat when it is saturated, plus the base current the control pin must supply.

BJTs remain common in analog amplifiers, signal driving and low-cost switching. Most hobby motor drivers use BJTs for small loads, and industrial power designers still use them for specific high-voltage jobs.

---

## MOSFETs

A MOSFET is a field-effect transistor. The gate sits on an insulating oxide layer over the channel, so the gate draws almost no DC current. A gate voltage above the threshold creates a channel between drain and source, and the device conducts. An N-channel MOSFET is the common switch for low-side loads, with the source on ground.

A sluice gate opened by a lever shows the idea, with the water pressure doing the work. The lever uses almost no effort because it only moves the gate.

The threshold voltage for a standard MOSFET is often 2 to 4 V, which means a 5 V gate may not fully enhance it. A logic-level MOSFET is designed for a gate drive of 3.3 V to 5 V, with a lower threshold and a specified R_DS(on) at that gate voltage. Conduction loss is `P = I² × R_DS(on)`. A hypothetical logic-level device with R_DS(on) of 50 mΩ carrying 2 A dissipates `2² × 0.05 = 0.2 W`. Check the datasheet curve for R_DS(on) at your gate voltage, because the value at 10 V is much lower and does not apply to a 5 V drive.

The gate is a capacitor. Every switch charges and discharges the gate capacitance, which costs energy and limits speed. A gate resistor of 100 Ω to 470 Ω slows the edge and damps ringing. A 10 kΩ pull-down from gate to ground keeps the device off when the pin is floating, as explained later in this lesson.

The common mistake is using a standard MOSFET with a 5 V gate. It is only partly on, its drop is high and it overheats. Choose a logic-level part and check R_DS(on) at your gate drive. The second mistake is leaving the gate floating. An unconnected gate picks up noise and can switch the device on randomly. MOSFETs are also sensitive to static discharge, so handle them with grounded fingers.

MOSFETs are used in every modern motor driver, DC-DC converter and robot battery protection circuit, because their low R_DS(on) keeps losses small at high current.

---

## Transistor switching

A transistor used as a switch should spend its time in one of two states. It is either off, with almost no current, or saturated, with almost no voltage across it. In the middle, the transistor dissipates a lot of power, and that is where things get hot.

For a BJT, the design procedure is short. Find the load current, `I_C`. Pick a forced gain, `β_forced`, usually 10, which is lower than the transistor's minimum gain at that current. The base current is then `I_B = I_C / β_forced`. The base resistor is `R_B = (V_drive − V_BE) / I_B`, with V_BE about 0.7 V.

Worked example. An Arduino Uno pin at 5 V drives a 2N2222 that switches a relay coil at 71 mA, which is `5 / 70 Ω`. The forced gain is 10, so `I_B = 71 / 10 = 7.1 mA`. The base resistor is `R_B = (5 − 0.7) / 0.0071 ≈ 606 Ω`. Choose 620 Ω, which gives `I_B = 4.3 / 620 ≈ 6.9 mA`, a forced gain of `71 / 6.9 ≈ 10.2`. The pin supplies 6.9 mA, under the 20 mA recommended limit.

Check the dissipation. At saturation, V_CE is about 0.2 V, so the transistor dissipates `0.2 × 0.071 ≈ 14 mW`. The relay coil dissipates the rest, and the transistor stays cool.

A flyback diode goes across the relay coil, as explained on Day 8. Without it, the transistor must absorb the coil's voltage spike.

The common mistake is using a base resistor so large that the transistor is only partly on. Suppose you choose 10 kΩ: the base current is `4.3 / 10,000 = 0.43 mA`, which gives a collector current of up to `0.43 × β`. With β = 100, the maximum is 43 mA, not enough for 71 mA. The transistor stays in its linear region and heats up, so the relay chatters. The second mistake is forgetting that gain varies with temperature. A design that works cold can fail when hot. The forced-gain margin is there to absorb that variation.

Transistor switching drives relays, solenoids, LED strips and small motors on hobby and service robots. The same calculation is the first step of a motor driver.

---

## Pull-up and pull-down resistors

A digital input must read high or low, and a floating input reads neither. A pull-up resistor connects the input to the positive supply through a high resistance, so it reads high when nothing else drives it. A pull-down connects the input to ground, so it reads low when undriven.

A spring holding a door open works the same way. When the door is released, the spring pulls it back to a known position. A pull-up is the spring, and the button is a hand that can overcome it.

With a 10 kΩ pull-up to 5 V and a pushbutton to ground, the released input reads 5 V. When the button is pressed, the input is pulled to 0 V through the switch. The current while pressed is `5 / 10,000 = 0.5 mA`, which is small enough that the resistor does not overheat and the pin's input current is negligible.

The value is a trade-off. A lower resistance pulls harder, is less sensitive to noise and draws more current when the input is held low. A higher resistance saves power and is more sensitive to noise and to stray capacitance. Common values are 4.7 kΩ to 10 kΩ. The I²C bus uses a 4.7 kΩ pull-up on each line in standard designs, because the bus capacitance needs a strong pull to meet its rise time.

The common mistake is leaving a button without a pull-up or pull-down. The Arduino's `INPUT_PULLUP` mode fixes it in software, as used in the Day 8 sketch. The limitation of an internal pull-up is its fixed value, which is often around 20 to 50 kΩ on microcontrollers. That is weak for long wires, and an external 4.7 kΩ or 10 kΩ resistor is better.

Pull-ups and pull-downs appear on every digital input in a robot, from the emergency-stop line to the reset pin of a microcontroller.

---

## Logic-level switching

Digital logic families define voltage thresholds for high and low. For a CMOS input, the datasheet gives VIL max (the highest voltage read as low) and VIH min (the lowest voltage read as high). In the ATmega328P used by the Arduino Uno at 5 V, the datasheet gives VIL max as 0.3 times VCC, about 1.5 V, and VIH min as 0.6 times VCC, about 3.0 V. Between 1.5 V and 3.0 V, the input can read either state.

The RP2040 on the Pico runs its I/O at 3.3 V. Its thresholds are fractions of its I/O supply, so a 5 V signal is not safe to feed directly to a Pico GPIO pin. Most RP2040 pins are not 5 V tolerant, and a 5 V signal can damage the pin.

A 5 V output can talk safely to a 3.3 V input only when the input's maximum voltage rating is respected. The simple fix is a voltage divider, which you know from Day 7. To scale a 5 V signal to about 3.3 V, use `R_top / R_bottom = (5 − 3.3) / 3.3 ≈ 0.515`. Choose 1 kΩ on top and 2 kΩ on the bottom, which gives `5 × 2 / 3 ≈ 3.33 V`. The divider works for one-way signals that switch slowly. For fast or bidirectional buses, use a dedicated level shifter, such as a MOSFET-based bidirectional shifter.

A 3.3 V output into a 5 V input is often accepted, because the 5 V part's VIH is lower than 5 V. Check the datasheet for the specific input, because some 5 V parts specify a higher VIH.

The common mistake is connecting a 5 V output straight to a 3.3 V board and assuming it works because the board does not smoke at once. Damage accumulates, and the failure shows up later as a dead pin. The limitation of a divider is that it draws a continuous current from the signal and slows the edge, so it is only for slow signals.

Logic-level problems are a common reason for a robot that runs on the bench with a USB supply and fails when the battery is connected. Check the voltage levels on every signal that crosses a board boundary.

---

## Hands-on exercises

Record every reading in a table. Set the DMM to diode test mode for the first two exercises.

1. **Measure diode and LED drops.** Measure the forward voltage of a 1N4007, a 1N4148 and a red LED with the DMM in diode test mode. Record the readings. The silicon diodes should read about 0.6 to 0.7 V, and the red LED should read about 1.8 to 2.0 V.
2. **Measure the 2N2222.** Identify the emitter, base and collector from the datasheet. Measure the base-emitter diode in diode test mode. It should read about 0.6 to 0.7 V in the forward direction.
3. **Pull-up and button.** Build a 10 kΩ pull-up from 5 V (Uno 5 V pin) to a digital pin, with a pushbutton from that pin to GND. Upload the sketch below. Watch the Serial Monitor at 115200 baud while you press and release the button. The output should read 1 when released and 0 when pressed. Then measure the pin voltage with the DMM while you press and release.

```cpp
const int INPUT_PIN = 2;

void setup() {
  pinMode(INPUT_PIN, INPUT);  // external pull-up is used, not the internal one
  Serial.begin(115200);
}

void loop() {
  Serial.println(digitalRead(INPUT_PIN));
  delay(200);
}
```

4. **Switch a 5 V motor with an NPN transistor.** Build the circuit below. Power the motor from the Uno's 5 V pin, which is limited by the USB supply and should stay under 150 mA for this exercise. Put the 1N4007 across the motor with the stripe to 5 V, so it is reverse-biased in normal operation and conducts during a spike. Connect the 620 Ω base resistor from pin 9 to the 2N2222 base, the emitter to GND, and the collector to the motor's negative terminal. Upload the sketch below, which toggles the motor once per second.

```cpp
const int MOTOR_PIN = 9;

void setup() {
  pinMode(MOTOR_PIN, OUTPUT);
}

void loop() {
  digitalWrite(MOTOR_PIN, HIGH);
  delay(1000);
  digitalWrite(MOTOR_PIN, LOW);
  delay(1000);
}
```

   Measure V_CE across the transistor when the motor is on. It should be a few tenths of a volt. Measure the base current by measuring the voltage across the 620 Ω resistor, and calculate the current. It should be about 7 mA.

5. **Level shifter (optional, with Pico).** Build the 1 kΩ and 2 kΩ divider from a 5 V source. Measure the midpoint. It should be about 3.33 V. Explain whether the midpoint is safe for the Pico's pin.

**Safety notes:** keep the 2N2222 base current under 20 mA from the Uno. Keep the motor under 150 mA from USB. Do not put a battery across a motor without a driver, a fuse and a flyback diode. Remove power before changing the transistor's pinout.

**Done when:** the table has diode, pull-up, transistor and divider measurements, and the motor switches on and off on the Uno with the flyback diode in place.

---

## Project: a transistor switch calculator

This project is a light one. The arithmetic here is small, but it is the same arithmetic that goes into a motor driver, and a tested calculator is the habit that prevents a failed transistor.

**Goal:** write functions for the base resistor, the saturation check, the MOSFET conduction loss and the divider ratio, test them, and commit the Uno switching sketch with a README that records your measured V_CE and base current.

**Repo name:** `100-days-robotics-engineering-day-09-semiconductor-fundamentals`

**Repo layout:**

```
100-days-robotics-engineering-day-09-semiconductor-fundamentals/
├── README.md
├── transistor_calc.py
├── test_transistor_calc.py
└── firmware/
    └── motor_switch/
        └── motor_switch.ino
```

**Starter code** (`transistor_calc.py`):

```python
def base_resistor_ohm(v_drive: float, i_load: float, forced_gain: float, v_be: float = 0.7) -> float:
    """Base resistor for a saturated NPN switch: R_B = (V_drive - V_BE) / (I_load / forced_gain)."""
    if forced_gain <= 0 or i_load <= 0:
        raise ValueError("gain and load current must be positive")
    i_base = i_load / forced_gain
    return (v_drive - v_be) / i_base


def base_current_a(v_drive: float, r_base: float, v_be: float = 0.7) -> float:
    """Base current through a given base resistor, in amperes."""
    return (v_drive - v_be) / r_base


def is_saturated(i_load: float, i_base: float, min_gain: float) -> bool:
    """True when the base current supports the load at the minimum gain: i_base * min_gain >= i_load."""
    return i_base * min_gain >= i_load


def mosfet_conduction_loss_w(i_drain: float, r_ds_on: float) -> float:
    """Conduction loss in a fully on MOSFET: P = I^2 * R_DS(on), in watts."""
    return i_drain ** 2 * r_ds_on


def divider_ratio(v_high: float, v_target: float) -> float:
    """Ratio R_top / R_bottom that scales v_high down to v_target."""
    if v_target <= 0 or v_high <= v_target:
        raise ValueError("v_target must be positive and below v_high")
    return (v_high - v_target) / v_target
```

**Step-by-step tasks:**

1. Create the repo and a virtual environment, and install `pytest`.
2. Implement the five functions, with docstrings that state units.
3. Write tests: `base_resistor_ohm(5, 0.071, 10)` ≈ 606 Ω, `base_current_a(5, 620)` ≈ 0.00694 A, `is_saturated(0.071, 0.00915, 10)` is True (the 470 Ω case from the calculation above), `mosfet_conduction_loss_w(2, 0.05)` = 0.2 W, and `divider_ratio(5, 3.3)` ≈ 0.515.
4. Add a test that `base_resistor_ohm` raises `ValueError` for a zero load current.
5. Save `motor_switch.ino` with the motor sketch from the exercises.
6. In the README, record the measured V_CE and base current from the bench and compare them with the calculator's output for the 620 Ω resistor.

**README checklist:**

- [ ] Setup and run commands
- [ ] Table of functions, with units and formulas
- [ ] Bench measurements against calculated values
- [ ] Safety notes: pin current, USB current, flyback diode

**Done when:**

- `pytest -q` passes with at least five tests
- The motor switches on and off on the Uno with the flyback diode in place
- The README records measured V_CE and base current, with the calculated value beside each one
- The repo is pushed to GitHub with a clear commit message

**Evidence to commit:** the source, the tests, the sketch, the README and a photo of the transistor circuit.

**Why the project is light:** the functions are short, and the bench work carries most of the learning. A heavier project would hide the physics behind code.

---

## Check your understanding

1. A transistor must switch a 100 mA load, and the design uses a forced gain of 10. A student uses a 1 kΩ base resistor from a 5 V pin and says it saturates. Calculate the base current and the forced gain actually achieved. Decide whether the design saturates, using a minimum gain of 30.
2. A MOSFET with R_DS(on) of 0.1 Ω at 5 V drive switches a 3 A load. Calculate the conduction loss, and explain whether a 0.5 W package is enough.
3. A designer connects a 5 V output to a 3.3 V board pin through a 1 kΩ and 2 kΩ divider. Calculate the pin voltage. Explain what the divider does to the edge speed and why it is only for slow signals.
4. A pull-up of 10 kΩ holds a button input high. A second pull-up of 1 kΩ is added in parallel. Calculate the new equivalent pull-up value, and explain what it does to current when the button is pressed.
5. A 2N2222 is driven from a 3.3 V pin with a 470 Ω base resistor to switch a 40 mA load. Calculate the base current, and decide whether the transistor saturates for a forced gain of 10.
6. A Zener diode with a 5.1 V rating and a 1 kΩ series resistor sits across a 9 V supply, with no load. Calculate the Zener current and explain what happens when the load draws 2 mA.

**Answers:**

1. The base current is `(5 − 0.7) / 1000 = 4.3 mA`. The forced gain achieved is `100 / 4.3 ≈ 23`. With a minimum gain of 30, the base current supports `4.3 × 30 = 129 mA`, which is above 100 mA, so the design saturates. The margin is small, so use a lower base resistor for safety.
2. The loss is `3² × 0.1 = 0.9 W`. A 0.5 W package is not enough. Choose a larger package, a lower R_DS(on) part, or a lower-current design.
3. The pin voltage is `5 × 2 / 3 ≈ 3.33 V`. The divider adds an RC-like delay with the pin's capacitance, which slows the edges. It suits slow signals such as a button, not a fast clock or a serial bus.
4. The parallel value is `1 / (1/10,000 + 1/1000) = 909 Ω`. When the button is pressed, the current through the pull-up rises from 0.5 mA to `5 / 909 ≈ 5.5 mA`, which is more current for little gain in noise immunity. It is a poor design.
5. The base current is `(3.3 − 0.7) / 470 ≈ 5.5 mA`. The collector current needed is 40 mA, so the forced gain is `40 / 5.5 ≈ 7.2`, below the design target of 10. The base drive is more than the design needs, and the transistor saturates as long as its minimum gain at 40 mA is above 7.2. Confirm that on the datasheet before you accept the design.
6. The Zener is in reverse breakdown at 5.1 V, so the resistor drops `9 − 5.1 = 3.9 V`, and the current is `3.9 / 1000 = 3.9 mA`. With a 2 mA load, the Zener current falls to about 1.9 mA, and the Zener still holds about 5.1 V as long as it stays above its minimum current.

---

## Sources to read and watch

**Read**

- Wikipedia: *Semiconductor*, *PN junction*, *Diode*, *Zener diode*, *Bipolar junction transistor*, *MOSFET*, *Logic level*, *Pull-up resistor*, *Transistor*
- Datasheets: the 2N2222 family datasheet from a major manufacturer such as ON Semiconductor, and the IRLZ44N datasheet from Infineon or International Rectifier. Search each part number.
- The Raspberry Pi Pico datasheet, from the Raspberry Pi documentation site, for the GPIO voltage limits

**Tutorials**

- SparkFun Learn, the tutorials on transistors and MOSFETs. Search the SparkFun Learn site for the titles.
- Adafruit Learning System, the guides on transistors and MOSFETs and the logic-level guide. Search the Adafruit site for "transistor" and "level shifter".
- All About Circuits, the lessons on bipolar transistors and MOSFETs in the semiconductor section. Search the site for the lesson titles.

**Watch**

- MIT OpenCourseWare, 6.002 Circuits and Electronics, the lectures on the transistor and the common-emitter amplifier
- Khan Academy, the lessons on diodes and transistors in the electrical engineering content

**Practice**

- [Wokwi](https://wokwi.com/): build the Uno, transistor and motor circuit, and run the motor sketch in the simulator before you wire the real board

---

## Tomorrow

Tomorrow you move from analog levels to numbers. Today a transistor switched one signal between on and off, and the logic-level section assumed a high and a low. Digital electronics gives those levels meaning. You will convert between decimal, binary and hexadecimal, read truth tables for gates such as AND, OR and XOR, and use bitwise operators to read several pins as one byte, which is how robot firmware reads switches and sensors.

**Next:** [Day 10: Digital Electronics](Robotics-Day-10-Digital-Electronics.md)
