# Day 8: Components

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 3 hours
**Level:** beginner. You should be comfortable with Day 6's Ohm's law and Day 7's divider rules. Today you learn what each common part does, how to read its datasheet and how to choose it for a robot.

**By the end of this day, you will have:**

- A working understanding of resistors, potentiometers, capacitors, inductors, diodes, LEDs, switches, relays, connectors and fuses
- A decoded set of resistor color bands, a measured RC charging curve and a debounced pushbutton running on an Arduino
- A Python helper module for component calculations, and a short parts-selection note for a robot's power and sensor wiring

**What you need:**

- Everything from Days 6 and 7 (breadboard, DMM, 9 V battery, resistor assortment, red LEDs, jumper wires)
- Arduino Uno R3 (ATmega328P) microcontroller board, about $25 for an official board or about $10 for a compatible clone
- USB-A to USB-B cable for the Uno (often included)
- Arduino IDE 2.3 or newer, free from [arduino.cc](https://www.arduino.cc/en/software), installed on Windows, macOS or Linux
- 100 µF, 16 V electrolytic capacitor (about $1)
- 1N4007 rectifier diode, pack of 10 (about $2)
- Two momentary pushbuttons, 6 mm tactile type (about $3)
- 10 kΩ potentiometer, linear taper (about $2)
- A 5 V single-channel relay module, which is simpler than a bare relay and includes its own driver (about $4)
- A 1 A fuse with a holder, or a resettable PTC fuse (about $3)
- Python 3.10 or newer, and `pytest` 8.x for the project

**Prerequisites:**

- Day 6: Ohm's law and power
- Day 7: voltage dividers and KCL
- Basic Arduino sketch structure: `setup()` runs once, `loop()` runs repeatedly. Your code uses `pinMode`, `digitalRead` and `digitalWrite`.

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Background and history](#background-and-history)
- [Glossary](#glossary)
- [Resistors](#resistors)
- [Potentiometers](#potentiometers)
- [Capacitors](#capacitors)
- [Inductors](#inductors)
- [Diodes](#diodes)
- [LEDs](#leds)
- [Switches](#switches)
- [Relays](#relays)
- [Connectors](#connectors)
- [Fuses and protection components](#fuses-and-protection-components)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: a component calculator and a parts-selection note](#project-a-component-calculator-and-a-parts-selection-note)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot's schematic is a list of components connected by wires, and most of the design decisions are about choosing the right part for each job. A resistor limits current. A capacitor stores charge and smooths voltage. A diode allows current in one direction. A relay lets a small signal switch a large load. Pick the wrong part and the robot works on the bench and fails on the field: a button that registers two presses, a motor that kicks back high voltage, or a fuse that never blows.

Today you walk through the common parts one at a time. For each, you learn the physical idea, the number that matters on a datasheet, and the failure you will see if you ignore it. You will finish with a working Arduino sketch that uses a debounced button, because switch bounce is the first real-world signal problem most robot builders meet.

---

## Background and history

Components have changed more than the principles behind them. The table shows how the parts above reached the workbench.

| Year | Component or event | What it changed |
|---|---|---|
| 1745 | The Leyden jar, an early capacitor | Stored static charge in glass, a first step toward capacitors. |
| 1831 | Faraday's induction experiments | Coils with changing current store energy in magnetic fields, the physics of inductors. |
| 1835 | Joseph Henry's work on electromagnets and relays | Small currents can switch large currents at a distance, the principle of the relay. |
| 1904 | John Ambrose Fleming's vacuum diode | The first electronic valve that conducts in one direction. |
| 1950s | Silicon diodes and transistors replace vacuum tubes | Components become small, cheap and reliable. |
| 1960s onward | Surface-mount packages and standard connector pitches such as 2.54 mm | Parts become automated to assemble and standard to wire. |

---

## Glossary

- **Tolerance:** the allowed deviation of a part's value from its nominal value, such as ±5%.
- **Power rating:** the maximum continuous power a part can dissipate without overheating, such as 1/4 W for a resistor.
- **Time constant (τ):** the time for an RC or RL circuit to reach about 63 percent of its final change. For RC, τ = R × C.
- **Forward voltage:** the voltage a diode or LED needs across it before it conducts strongly.
- **Flyback:** the high voltage spike produced when current in an inductor is interrupted.
- **Debounce:** the process of ignoring the rapid on-off chatter a mechanical switch produces when it closes or opens.
- **Pull-up:** a resistor connecting a signal to a positive supply, so it reads high when nothing else drives it. Covered in Day 9.
- **Pitch:** the center-to-center distance between connector pins, such as 2.54 mm.
- **Fuse:** a part that melts or opens at a set current to protect the wiring.
- **PTC (positive temperature coefficient) fuse:** a resettable fuse whose resistance rises sharply when it overheats, then falls when it cools.

---

## Resistors

A resistor is the friction of a circuit, as covered on Day 6. It is the simplest and most common part, and most of its datasheet entries are about power, tolerance and temperature behavior.

A resistor works like a narrow section of a water pipe. It slows the flow and turns some of the energy into heat. Its job is to set a current or a voltage ratio, and its limit is how much heat it can shed. The power rating matters because `P = I²R` heats the part whether or not the circuit is doing useful work.

Take a 470 Ω resistor across 12 V. The current is `12 / 470 ≈ 25.5 mA`, and the power is `12 × 0.0255 ≈ 0.31 W`. A 1/4 W resistor overheats here, so choose a 1/2 W part.

Most resistors use color bands to show their value. The standard reading uses three digits and a multiplier for 5% parts, and a fourth band for tolerance. The table shows the three values you will use most today:

| Value | Bands (first to fourth) |
|---|---|
| 220 Ω | red, red, brown, gold |
| 4.7 kΩ | yellow, violet, red, gold |
| 10 kΩ | brown, black, orange, gold |

The common mistake is reading the bands from the wrong end. Start from the band nearest the end without a gap, or the gold or silver tolerance band. The second mistake is trusting the color under poor lighting. Measure the resistor with the DMM to confirm.

Resistors fail in a few ways. They drift when they overheat, a surface-mount part can crack when a board flexes, and a carbon resistor's value can change with age and humidity. Metal-film resistors hold tolerance better and cost slightly more. Choose tolerance and power by the circuit: a voltage divider for a sensor needs 1% parts if the reading must be accurate.

---

## Potentiometers

A potentiometer is a variable resistor with three terminals. Two ends connect across a fixed resistance, and a wiper taps off the resistance in between. Turning the knob moves the wiper, so the potentiometer is a divider with an adjustable ratio, the same idea as Day 7.

Imagine a garden hose with a movable tap. Moving the tap changes how far along the hose the water is drawn from. The resistance from one end to the wiper changes, and so does the voltage.

With a 10 kΩ linear potentiometer across 5 V, a centered wiper gives `5 × 5 / 10 = 2.5 V` at the wiper. Turn the wiper to one end and the output moves to 0 V or 5 V.

The common mistake is wiring a potentiometer as a variable resistor with the wiper and one end connected, but with the other end floating. That gives a resistance but not a stable voltage. The limitation is wear: the wiper leaves carbon or metal dust on the track, and a cheap potentiometer gets noisy after a few thousand turns. Robots use potentiometers for setpoints, calibration and joysticks, and joysticks often use two per axis.

---

## Capacitors

A capacitor stores charge by holding two conductors close together with an insulator between them. It is like a bucket in a water line: it fills and empties, and while it fills, the flow changes. The stored charge is `Q = C × V`, where C is capacitance in farads (F). Practical values are microfarads (µF), nanofarads (nF) and picofarads (pF).

When a capacitor charges through a resistor, the voltage rises along a curve. The time constant is `τ = R × C`. After one τ, the voltage reaches 63.2 percent of the supply, and after five τ it is within 1 percent of the full value.

Take a 10 kΩ resistor and a 100 µF capacitor. The time constant is `10,000 × 0.0001 = 1 s`. Across a 9 V supply, the capacitor reaches `0.632 × 9 ≈ 5.7 V` after one second, and it is near 9 V after five seconds.

The same arithmetic explains switch debounce. A 10 kΩ resistor with a 100 nF capacitor has `τ = 10,000 × 0.0000001 = 1 ms`, which is a common hardware filter for a switch whose contacts bounce for a few milliseconds. The filter helps, but it is not a complete fix, and firmware debounce is usually the better choice. You will see that in the hands-on work.

Stored energy is `E = ½CV²`. A 1,000 µF capacitor at 12 V stores `0.5 × 0.001 × 12² = 0.072 J`, enough to create a noticeable spark or to glitch a microcontroller when a battery is connected.

The common mistake is connecting an electrolytic capacitor with reversed polarity or above its voltage rating. It can vent or burst. Electrolytics are polarized, with a marked negative stripe, and need a voltage rating above the working voltage with margin. A limitation: real capacitors have leakage and equivalent series resistance (ESR). Leakage drains the charge over time, and ESR limits how much ripple current a supply capacitor can smooth.

Capacitors in robotics filter the noise on a motor supply, store energy for a short burst of current and smooth the output of a regulator.

---

## Inductors

An inductor is a coil of wire, and it stores energy in its magnetic field. A changing current makes a voltage across the coil: `V = L × di/dt`, where L is inductance in henries (H). Current in an inductor cannot change instantly, because an instant change would need an infinite voltage.

Imagine a heavy flywheel. Pushing it up to speed takes effort, and it resists slowing down. A coil resists changes in current in the same way.

Take a 10 mH coil that ramps from 0 A to 1 A in 10 ms. The voltage across it is `0.01 × (1 / 0.01) = 1 V`. A faster change needs a bigger voltage. Now suppose a relay coil with an inductance of 0.1 H, carrying 70 mA, is switched off in one microsecond. The voltage spike is `0.1 × 0.07 / 0.000001 = 7,000 V`. The real spike is limited by the switch, but it is large enough to destroy a transistor. This spike is the reason motors and relays need protection, which is covered in the relay section.

The common mistake is thinking that an inductor is a short circuit at all times. It is a short circuit only for steady DC, where di/dt is zero. The second mistake is ignoring saturation. A coil with a magnetic core stops storing energy when the core saturates, and its inductance drops sharply. Datasheets list the saturation current, and it matters for power converters on Day 12.

Inductors and motor windings show up in every DC motor on a robot. Motor coils are inductors, and the flyback problem they cause is why motor drivers include protection diodes.

---

## Diodes

A diode lets current flow in one direction. Its physics belongs to Day 9, where you meet the PN junction. Today you need the behavior: current flows freely when the anode is at a higher voltage than the cathode by the forward voltage, and almost none flows in reverse until the reverse breakdown voltage.

A one-way valve on a pipe works the same way. Water flows one way and is blocked the other way, and the valve needs a small pressure to open.

A silicon rectifier diode, such as the 1N4007, has a forward drop of about 0.6 to 0.7 V at low current. Its rating is 1 A continuous and 1,000 V reverse. A Schottky diode, with a metal-semiconductor junction, has a forward drop of about 0.2 to 0.4 V, which is why it is common in reverse-polarity protection, where a large voltage drop would waste energy.

In a typical use, a relay coil is connected across a supply, with a diode placed in reverse across the coil. When the supply switches off, the flyback spike reverses the diode, which then clamps the voltage to about 0.7 V above the supply. The energy is dissipated in the coil's resistance instead of across the switch.

The common mistake is installing the diode backwards. A reverse-biased flyback diode does nothing, and the switch takes the spike. The stripe marks the cathode. The limitation is speed: a standard rectifier diode such as the 1N4007 recovers slowly, and high-frequency circuits need a faster part, such as a fast-recovery or Schottky diode.

Diodes protect robot electronics every day, from flyback clamps on motor drivers to reverse-polarity protection on battery inputs.

---

## LEDs

A light-emitting diode is a diode that emits light when it conducts. Its forward voltage depends on the color: about 1.8 to 2.0 V for red, about 2.0 to 2.2 V for yellow and green, and about 3.0 to 3.4 V for white and blue. Its current rating is usually 20 mA continuous for a standard indicator LED.

LEDs are current-driven parts. Their brightness follows the current, and a small change in voltage causes a large change in current. That is why an LED must always have a series resistor, as covered on Day 6.

For a red LED on a 5 V supply at 15 mA, use `R = (5 − 2) / 0.015 = 200 Ω`, and pick 220 Ω. On a 3.3 V microcontroller, the same LED at 10 mA needs `R = (3.3 − 1.9) / 0.010 = 140 Ω`, so use 150 Ω.

The common mistake is connecting an LED with its polarity reversed. Reverse voltage is usually small, and an LED has a reverse rating often around 5 V, so a reversed LED will not light. It is only damaged when the reverse voltage is high. The limitation is heat. An LED's brightness and forward voltage shift with temperature, so a bright LED on a hot board can change color or current.

LEDs are the status indicators on every robot controller, and high-power LEDs light work areas and cameras on mobile robots.

---

## Switches

A switch connects or disconnects a path. A pushbutton is a momentary switch that closes while pressed. A toggle switch latches. A slide switch selects between two modes. For a microcontroller, the important property is what happens when the contacts close.

Mechanical contacts do not close cleanly. They bounce, making and breaking contact several times over a few milliseconds before settling. A microcontroller that samples the pin fast enough sees each bounce as a separate press.

A dropped ball bounces before it lies still. Your eye sees one drop, but a sensor that times each bounce sees many.

Debounce removes the bounces. The simplest software method waits until the reading has been stable for a fixed time, typically 10 to 20 ms for a tactile button. The sketch later in this lesson does that. The hardware method is the RC filter from the capacitor section, a 10 kΩ and 100 nF network. Both work, but the software method is easier to tune and cheaper.

The common mistake is assuming a button wired to a pin without a pull-up or pull-down resistor has a defined state. Its pin floats, and the reading is random. Day 9 covers pull-ups, and the Arduino `INPUT_PULLUP` mode provides one internally.

The limitation of mechanical switches is wear and bounce, and the rating is usually given as a number of operations, often tens of thousands for a tactile button. For a robot's emergency stop, use a switch rated for the current, with a clear mechanical latch and a normally closed contact, so a broken wire stops the machine.

Switches appear on every robot as operator inputs, limit switches at the end of travel and emergency-stop chains.

---

## Relays

A relay is an electrically operated switch. A small coil, energized by a modest current, pulls a set of contacts closed or open. The coil is electrically isolated from the contacts, so a small control signal can switch a larger load. That isolation is the main reason relays are used.

A light switch wired to a remote is a fair picture. The small signal travels to the mechanism, and the mechanism moves the heavy contact that carries the power.

A common 5 V relay has a coil resistance of about 70 Ω. Across 5 V, the coil current is `5 / 70 ≈ 71 mA`, and the power is `5 × 0.071 ≈ 0.36 W`. The Arduino pin cannot supply that current safely. The Uno's pins are rated for 20 mA recommended and 40 mA absolute maximum, so the relay must be driven through a transistor or a driver, which Day 9 covers. A relay module handles this for you, and it usually includes an optocoupler and a flyback diode.

The common mistake is leaving out the flyback diode. The inductance of the coil produces the spike described in the inductor section, and it can damage the driving transistor or the microcontroller. The limitation is mechanical: relay contacts bounce, have a finite electrical and mechanical life, and need an arc-suppression plan when they switch inductive loads. A relay also makes a click, which is a useful indicator but a problem in quiet spaces.

Relays remain common in robot power switching, in industrial control panels and in battery-disconnect circuits, where a clean, isolated, mechanically positive cut is the goal.

---

## Connectors

A connector joins one board or cable to another. Connectors matter because most robot failures show up at the connections: a loose pin, a reversed battery plug or a cable that works on the bench and breaks on the first vibration.

Connectors are defined by their pitch, their pin count and their current rating. The 2.54 mm pitch of a standard breadboard and header is common on development boards. JST-XH and similar 2.5 mm connectors are common for small battery packs and sensors. For battery power, connectors like XT60 and Anderson PowerPole handle high current, and each has a rating in amps for a given wire gauge.

A connector is the part of a pipe that you can unscrew for repair. It must hold pressure and must not leak.

The common mistake is a mismatched connector that fits but is polarized the wrong way. Many battery connectors are not keyed, so a reversed plug reaches the wrong pins, and the result is a short. Use keyed connectors, label each wire, and check polarity with the DMM before the first power-on. The limitation is contact resistance and wear, which is why power connectors are rated for mating cycles.

Connectors appear in every robot harness and separate subsystems, from the motor cable to the battery plug.

---

## Fuses and protection components

A fuse is a sacrificial part. It carries current normally and opens when the current exceeds its rating for long enough. Fuses protect wiring and components from sustained overcurrent and short circuits.

Think of a dam with a spillway. Normal flow passes through, and the spillway fails in a controlled way before the dam does.

A fuse rating should sit above the normal current and below the current the wiring can carry. A 1 A fuse suits a 0.7 A load, and it blows before a short overheats a 22 AWG wire. Fast-blow and slow-blow fuses differ in response time, and a motor's starting surge needs a slow-blow part.

A PTC resettable fuse heats when current flows, and its resistance rises sharply above a trip current. It resets when it cools. Two other parts protect against different faults. A reverse-polarity diode, covered in the diode section, prevents damage when a battery is plugged in backwards. A transient-voltage-suppression diode clamps voltage spikes from a relay or an electrostatic discharge.

The common mistake is replacing a blown fuse with a higher rating to stop the nuisance. The fuse blew for a reason. Find the cause, then fix it. A limitation: a PTC fuse trips more slowly than a conventional fuse and has a higher on-resistance, which costs voltage at high current.

Protection is the reason every robot battery has a fuse or breaker, and every motor driver has current limits. It is also why the safety rules in this module always start with the question "what happens if this part fails short?"

---

## Hands-on exercises (about 60 minutes)

Work through these in order. Record every reading in a table.

1. **Read and check resistors.** Decode the color bands of 220 Ω, 4.7 kΩ and 10 kΩ resistors. Measure each with the DMM and record the reading and the percentage error.
2. **Measure the diode.** Set the DMM to diode test mode. Connect the probes to the 1N4007 in the forward direction. The reading should be about 0.6 to 0.7 V. Reverse the probes. The meter should show overrange or OL. Record both readings.
3. **Charge the capacitor.** Build a 10 kΩ resistor in series with the 100 µF electrolytic capacitor, with the capacitor's negative leg to ground and the positive leg to the resistor. Connect the 9 V battery at the resistor's free end, with the positive terminal going to the resistor. Measure the capacitor voltage every second for 10 seconds. The reading should reach about 5.7 V at 1 second, and it approaches 9 V after about 5 seconds. Discharge the capacitor through a 1 kΩ resistor after the test.
4. **Program the debounced button.** Connect a pushbutton from pin 2 to GND. Upload the sketch below to the Uno with the Arduino IDE. Press the button once and observe the built-in LED on pin 13. It should toggle once per press. Then remove the debounce delay, press again, and observe the extra toggles.

```cpp
const int BUTTON_PIN = 2;
const int LED_PIN = 13;
const unsigned long DEBOUNCE_MS = 20;

bool stableState = HIGH;
bool lastReading = HIGH;
unsigned long lastChangeMs = 0;
bool ledOn = false;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  bool reading = digitalRead(BUTTON_PIN);
  if (reading != lastReading) {
    lastChangeMs = millis();
    lastReading = reading;
  }
  if (millis() - lastChangeMs > DEBOUNCE_MS && reading != stableState) {
    stableState = reading;
    if (stableState == LOW) {  // pressed, because of the pull-up
      ledOn = !ledOn;
      digitalWrite(LED_PIN, ledOn);
    }
  }
}
```

5. **Read the potentiometer.** Connect the 10 kΩ potentiometer across 5 V from the Uno's 5 V pin and GND. Measure the wiper voltage with the DMM at three positions: both ends and the center. Record the values.

**Safety notes:** never connect a relay coil directly to an Arduino pin. Use the relay module. Check electrolytic polarity before you connect the capacitor. Do not exceed 20 mA from any Uno pin. Disconnect power before you change a wire.

**Done when:** your table has the resistor, diode, capacitor and potentiometer readings, and the sketch shows one toggle per press with the debounce on.

---

## Project: a component calculator and a parts-selection note

This project is a light one, with a small Python module for the calculations and a note that records how you chose each part. The aim is to make the component arithmetic repeatable and to practice writing a justification a reviewer can check.

**Goal:** write functions for resistor power, the RC time constant and capacitor voltage, and the coil voltage of an inductor. Add the debounced firmware. Write a one-page note that justifies the part choices for a small sensor-and-button board.

**Repo name:** `100-days-robotics-engineering-day-08-components`

**Repo layout:**

```
100-days-robotics-engineering-day-08-components/
├── README.md
├── PARTS.md
├── component_calc.py
├── test_component_calc.py
└── firmware/
    └── debounce/
        └── debounce.ino
```

**Starter code** (`component_calc.py`):

```python
import math


def resistor_power_w(voltage_v: float, resistance_ohm: float) -> float:
    """Power in a resistor: P = V^2 / R, in watts."""
    if resistance_ohm <= 0:
        raise ValueError("resistance must be positive")
    return voltage_v ** 2 / resistance_ohm


def rc_time_constant_s(resistance_ohm: float, capacitance_f: float) -> float:
    """Time constant tau = R * C, in seconds."""
    return resistance_ohm * capacitance_f


def rc_charge_voltage_v(v_supply: float, t_s: float, tau_s: float) -> float:
    """Capacitor voltage while charging from zero: V(t) = Vs * (1 - e^(-t/tau))."""
    return v_supply * (1 - math.exp(-t_s / tau_s))


def inductor_voltage_v(inductance_h: float, di_a: float, dt_s: float) -> float:
    """Voltage across an inductor: V = L * di/dt."""
    if dt_s <= 0:
        raise ValueError("time step must be positive")
    return inductance_h * di_a / dt_s
```

**Step-by-step tasks:**

1. Create the repo and a virtual environment, and install `pytest`.
2. Implement the four functions and add units to each docstring.
3. Write tests. Check `resistor_power_w(12, 470)` ≈ 0.306 W, `rc_time_constant_s(10000, 100e-6)` = 1.0 s, `rc_charge_voltage_v(9, 1, 1)` ≈ 5.69 V, and `inductor_voltage_v(0.01, 1, 0.01)` = 1.0 V.
4. Put the debounce sketch from the exercises in `firmware/debounce/debounce.ino`.
5. Write `PARTS.md`. For a board with one button, one 10 kΩ potentiometer, one status LED and one relay driven through a module, list each part, its key rating and why you chose it. Include the LED resistor value and the fuse rating for the 5 V rail.
6. Write the README with the test command and a summary of what the calculator checks.

**README checklist:**

- [ ] Setup and run commands
- [ ] Table of functions and units
- [ ] Firmware upload steps for the Uno
- [ ] Link to `PARTS.md`

**Done when:**

- `pytest -q` passes with at least four tests
- The debounce sketch shows one toggle per press
- `PARTS.md` justifies every part with a number from a rating or a calculation
- The repo is pushed to GitHub with a clear commit message

**Evidence to commit:** the source, the tests, the sketch, `PARTS.md`, and one photo of the breadboard with the button and LED.

---

## Check your understanding

1. A 10 kΩ resistor and a 22 µF capacitor form an RC network. Calculate the time constant and the time to reach 90 percent of a 5 V supply. Explain why the answer is not one time constant.
2. A relay coil is 80 Ω and is driven from 12 V. Calculate the coil current and power. Explain why you would not drive it from a microcontroller pin, and what you would add to the circuit.
3. A student says a 2 A fuse protects a 22 AWG wire that carries 0.5 A, so the fuse is always safe. What failure does the student miss?
4. An inductor's current is changed from 0.5 A to 0 A in 2 ms. The inductance is 50 mH. Calculate the average voltage across the coil during the change, and explain why a diode across the coil is needed.
5. A button is wired between a pin and GND, but the pin is not given a pull-up. The sketch reports random presses. Explain the cause, then describe two fixes.
6. A designer uses a 1N4007 to protect a 7.4 V battery input from reverse polarity. The circuit draws 1 A. Explain whether the choice is appropriate, using the forward voltage and the power dissipated.

**Answers:**

1. `τ = 10,000 × 0.000022 = 0.22 s`. Reaching 90 percent requires `t = −τ × ln(1 − 0.9) = 0.22 × 2.30 ≈ 0.51 s`. It is not one time constant, because the voltage rises exponentially, and 63 percent is only reached after one τ.
2. The coil current is `12 / 80 = 0.15 A`, or 150 mA, and the power is `12 × 0.15 = 1.8 W`. A microcontroller pin cannot supply 150 mA. Add a transistor or a relay module, plus a flyback diode across the coil.
3. The fuse does not protect the wire unless its rating is below what the wire can carry continuously. A 2 A fuse on a wire rated for 1 A would allow the wire to overheat before the fuse blows. The student has to check the wire's ampacity as well as the load current.
4. The average voltage is `0.05 × 0.5 / 0.002 = 12.5 V`, with the polarity that opposes the change. Without a flyback diode, the switch or driver sees that spike. A diode across the coil clamps it to about one diode drop above the supply.
5. The pin floats when the button is open, so its reading depends on stray charge and noise. Fix it with an external pull-up resistor (such as 10 kΩ to 5 V) or with the internal `INPUT_PULLUP` mode in the sketch. A debounce delay also helps but does not fix a floating pin.
6. The 1N4007 has a forward drop of about 0.7 V, so at 1 A it dissipates about `0.7 × 1 = 0.7 W`, and the input voltage at the regulator drops by 0.7 V. The 1N4007 is rated for 1 A, but it runs hot at 0.7 W and wastes battery energy as heat. A Schottky diode, with a drop of 0.3 to 0.5 V at that current, is the better choice for a battery input.

---

## Sources to read and watch

**Read**

- Wikipedia: *Resistor*, *Electronic color code*, *Potentiometer*, *Capacitor*, *RC circuit*, *Inductor*, *Diode*, *Light-emitting diode*, *Switch*, *Relay*, *Fuse (electrical)*, *Electrical connector*
- Arduino documentation at [docs.arduino.cc](https://docs.arduino.cc), the Arduino Uno Rev3 page, for the pin current limits and the `INPUT_PULLUP` mode

**Tutorials**

- SparkFun Learn, the tutorials on resistors, capacitors and pushbuttons. Search the SparkFun Learn site for the titles.
- Adafruit Learning System, the guide on the Arduino Uno and the debouncing guide. Search the Adafruit site for "debounce".
- Arduino Documentation, the Tutorials section, for `digitalRead` and `millis`

**Watch**

- MIT OpenCourseWare, 6.002 Circuits and Electronics, the lectures on RC and RL circuits
- Khan Academy, the lessons on capacitors and inductors in the electrical circuits content

**Practice**

- [Wokwi](https://wokwi.com/): build the Uno, pushbutton and LED circuit, and run the debounce sketch in the simulator before you wire the real board

---

## Tomorrow

Tomorrow you open the diode and look at why it conducts in one direction. The PN junction explains the forward drop you measured today, and the same physics gives you the transistor. You will use a transistor to drive the relay coil that a pin cannot supply, and you will use a MOSFET as a switch for a motor, with pull-up and pull-down resistors to keep logic signals defined.
