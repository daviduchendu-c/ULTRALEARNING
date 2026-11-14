# Day 6: Electricity Fundamentals

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 3 hours
**Level:** beginner. No electronics background is needed. You should be comfortable with multiplication, division and unit prefixes (milli, kilo), because every lesson in this module is arithmetic on a circuit.

**By the end of this day, you will have:**

- A working vocabulary for voltage, current, resistance, power and energy, with the units for each
- A bench log of measured resistance, battery voltage and LED current, set beside the calculated values
- A small Python module that applies Ohm's law and the power and energy formulas, with tests

**What you need:**

- A breadboard, 830-point size (about $5)
- A digital multimeter (DMM) with DC voltage, current (mA) and resistance modes (about $15 to $25)
- Jumper wire set, male to male (about $5)
- 9 V battery with a snap connector (about $3)
- Resistor assortment, 1/4 W, including 220 Ω, 470 Ω, 1 kΩ and 10 kΩ (about $8)
- Red 5 mm LEDs, pack of 10 (about $4)
- Python 3.10 or newer, and `pytest` 8.x for the project (`pip install pytest`)
- Optional: a free [Wokwi](https://wokwi.com/) account, to check your answers in a simulator

**Prerequisites:**

- Day 2: Anatomy of a Robot, especially the battery section and the gearbox arithmetic
- Basic algebra: rearranging `a = b × c` to solve for `b` or `c`
- No electronics knowledge yet

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Electric charge](#electric-charge)
- [Voltage](#voltage)
- [Current](#current)
- [Resistance and Ohm's law](#resistance-and-ohms-law)
- [Power and energy](#power-and-energy)
- [DC and AC](#dc-and-ac)
- [Conductors and insulators](#conductors-and-insulators)
- [Ground and reference](#ground-and-reference)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a power and energy calculator with a bench log](#project-a-power-and-energy-calculator-with-a-bench-log)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot you cannot power is a pile of parts. Every motor, sensor and computer on a robot runs on the same few physical quantities: how much energy is stored, how fast charge moves through a wire, how hard it is pushed, and how much of it turns into heat. Day 2 gave you the battery and the regulator as blocks on a diagram. Today you learn what flows between those blocks.

Robot teams argue about these numbers every week: whether the 2S pack is big enough for the drive motors, or why the microcontroller reset when the gripper closed. Each question reduces to voltage, current and resistance, and the answer comes from two relationships, `V = IR` and `P = VI`, that you will use for the rest of this series.

Today is about vocabulary and measurement. You will build one circuit, measure it with a multimeter, and check every reading against a calculation. When a bench reading disagrees with your arithmetic, find the cause before you move on.

---

## Short history

The units were named later than the ideas, and the ideas arrived in a useful order.

| Year | Person or event | What it added |
|---|---|---|
| 1800 | Alessandro Volta builds the voltaic pile | The first battery, and the first steady source of current for experiments. |
| 1827 | Georg Ohm publishes his law of current in a circuit | Current is proportional to voltage and inversely proportional to resistance. |
| 1831 | Michael Faraday demonstrates electromagnetic induction | Changing magnetic fields create voltage. This principle drives motors and generators. |
| 1880s to 1890s | The "war of the currents" | AC won for long-distance power because transformers can change its voltage easily. |

---

## Glossary

- **Charge:** a property of matter that feels a force in an electric field. Measured in coulombs (C).
- **Voltage:** the difference in electric potential energy per unit charge between two points. Measured in volts (V).
- **Current:** the rate at which charge flows past a point. Measured in amperes (A).
- **Resistance:** how strongly a material opposes current for a given voltage. Measured in ohms (Ω).
- **Power:** the rate at which energy is converted, to heat or motion. Measured in watts (W).
- **Energy:** the capacity to do work. Measured in joules (J) or watt-hours (Wh).
- **DC (direct current):** current that flows in one direction at a steady level.
- **AC (alternating current):** current that periodically reverses direction.
- **Conductor:** a material that lets charge move through it easily, such as copper.
- **Insulator:** a material that resists charge movement, such as plastic or dry air.
- **Ground:** the reference point against which voltages are measured, defined as 0 V in a circuit.
- **DMM (digital multimeter):** a meter that measures voltage, current and resistance.
- **Milli (m) and kilo (k):** prefixes meaning 0.001 and 1,000. 15 mA is 0.015 A.

---

## Electric charge

Think of charge as water in a closed pipe. You cannot see one molecule of water, but you can measure how much is in the pipe. Charge is the same kind of quantity: it exists in matter and can be counted. The unit is the coulomb (C). One electron carries about 1.6 × 10⁻¹⁹ C. Like charges repel, and opposite charges attract.

Charge does useful work only when it moves. A flow of charge is current, and the total charge moved is the current multiplied by time. If 1 A flows for 1 second, that is 1 C.

A motor driver that draws 0.1 A for 10 minutes moves `0.1 × 600 = 60 C`. You will rarely write coulombs, but the idea sits behind the battery rating in amp-hours. A 2,000 mAh cell holds `2 × 3600 = 7,200 C` of charge.

The common mistake is mixing up charge and current. Charge is an amount and current is a rate. A battery labeled 2,000 mAh says how much charge it holds, not how fast it can deliver it. A pack can hold plenty of charge and still sag to 5 V under a motor load, because its internal resistance limits the current it can give.

Charge is the physics behind every battery in the robot industry, from a warehouse robot's pack to the coin cell that keeps a real-time clock running.

---

## Voltage

Voltage is the pressure in the water analogy. Water in a tall tank pushes harder than water in a short one. Voltage is the same kind of push, defined as the energy per unit charge between two points. Because it is a difference, a voltage always belongs to a pair of points. You cannot measure "the voltage of a wire" alone.

One volt is one joule per coulomb, `1 V = 1 J/C`. A 9 V battery gives each coulomb 9 J of energy as it leaves the positive terminal, and the energy is spent in whatever load lies between the terminals.

Set the DMM to DC voltage and touch the red probe to the positive terminal and the black probe to the negative terminal. A fresh 9 V alkaline cell reads close to 9 V. Now move the black probe to a point in the middle of a breadboard circuit. The reading changes, because you are measuring between two different points.

Reversing the probes on a DC circuit gives a negative reading. The meter is not broken. The black probe is at a higher potential than the red one, and reading the sign is a free check that the meter is connected the way you think.

Robots have several voltage levels at once: logic chips at 3.3 V or 5 V, motors at 6 V to 24 V, and the battery at whatever the pack provides. Each part's datasheet gives the voltage it must see.

---

## Current

Current is the flow rate: how much charge passes a point each second. In the water analogy, it is liters per second through a pipe. The unit is the ampere (A), one coulomb per second. Current needs a complete loop. Charge leaves the positive terminal, passes through the load, and returns to the negative terminal. Break the loop anywhere and current stops.

Current is the quantity that heats wires and burns parts. A small indicator LED tolerates about 20 mA continuously, and far more destroys it. Most of this module's safety rules come down to limiting current.

To measure current, the meter must sit in series with the load, so all the charge through the load also passes through the meter. You break the loop and insert the meter. A voltage reading works the other way: the meter goes across two points.

A 470 Ω resistor across 9 V carries `9 / 470 ≈ 0.0191 A`, or 19.1 mA. If you put the meter across the battery in current mode instead, the meter's internal resistance is tiny, and the battery pushes a large current through a meter that is not built for it. The fuse blows, or the battery heats. Never measure current by placing the meter across a source.

The common mistake is the unit. Many meters have separate mA and A jacks, and reading 200 mA on the A jack is wrong by a factor of a thousand. Check the jack label before every measurement.

Battery management systems in mobile robots and electric vehicles monitor current continuously, because current is the number that drains a pack and heats it.

---

## Resistance and Ohm's law

Resistance is friction for current. A narrow, rough pipe slows water more than a wide, smooth one. A long, thin wire slows charge more than a short, thick one. The unit is the ohm (Ω).

Ohm's law ties the three quantities together for most ordinary resistors:

```
V = I × R
I = V / R
R = V / I
```

Use the form that solves for your unknown. Take a 9 V battery across a 1 kΩ resistor. The resistance is `1000 Ω`, so the current is `I = 9 / 1000 = 0.009 A`, or 9 mA. Doubling the resistance halves the current.

Ohm's law describes resistors, which have a fixed resistance. An LED's forward voltage stays nearly constant while its current changes, so it does not obey the law. A diode conducts in one direction only. A lamp filament changes resistance as it heats. Treat Ohm's law as an approximation for anything that is not a resistor until a datasheet says otherwise.

The common mistake is connecting an LED straight across a supply with no series resistor. A red LED drops about 2 V. On a 5 V supply, the leftover 3 V has almost nothing to push against, so the current climbs until the LED overheats and fails. Choose the resistor with Ohm's law. For a 15 mA target on 5 V, `R = (5 − 2) / 0.015 = 200 Ω`. Use the next standard value up, 220 Ω, which gives `3 / 220 ≈ 13.6 mA`. Stay below 20 mA for a standard 5 mm LED.

Resistance matters to robots in sensor circuits, pull-up and pull-down networks (Day 9), voltage dividers (Day 7) and motor windings. Motor drivers and cable selection depend on it too.

---

## Power and energy

Power is the rate at which energy is converted. A 100 W bulb turns 100 J of electrical energy into light and heat each second. In a circuit, `P = V × I`, in watts (W), where 1 W = 1 J/s.

Combining power with Ohm's law gives two more forms:

```
P = I² × R
P = V² / R
```

Take the 1 kΩ resistor with 9 mA through it. The power is `9 × 0.009 = 0.081 W`, or 81 mW, which a 1/4 W resistor handles. Check it another way: `0.009² × 1000 = 0.081 W`. The same answer.

Energy is power over time, `E = P × t`. Batteries are rated in watt-hours (Wh), where 1 Wh = 3,600 J. A 3 W motor that runs for 2 hours uses `3 × 2 = 6 Wh`. A 2,000 mAh, 3.7 V lithium-ion cell stores `2 × 3.7 = 7.4 Wh`. At a steady 1.5 W load, the ideal runtime is `7.4 / 1.5 ≈ 4.9 hours`. Real runtime is shorter, because capacity falls at high discharge and as the cell ages, and because regulators (Day 2) lose energy as heat.

The common mistake is unit mixing. Writing `P = 9 V × 9 mA = 81` and reporting 81 W, instead of 0.081 W, is the classic error. Convert to base units (volts, amperes, ohms, seconds) before any calculation, and convert to Wh only at the end for battery work. A second mistake is confusing power with energy. A high-power motor running for a second uses little energy.

Thermal ratings on drivers, regulators and resistors are power ratings, and battery sizing for mobile robots is power multiplied by time.

---

## DC and AC

Direct current (DC) flows one way at a steady level, as from a battery. Alternating current (AC) reverses direction on a regular cycle, as in household mains. Each needs different hardware to switch and convert.

An AC waveform has an amplitude, a frequency in hertz (Hz) and a period `T = 1 / f`. US mains is 120 V at 60 Hz, a period of about 16.7 ms. UK and most of European mains is 230 V at 50 Hz, a period of 20 ms. The quoted voltages are root-mean-square values, the DC equivalents that deliver the same heating.

Robot electronics run on DC almost everywhere. The exceptions are wall chargers and power supplies, which convert mains AC to DC before the robot ever sees it. Check whether a driver datasheet expects DC or AC, because the internal switching differs.

Mains is dangerous at currents you would never feel from a battery. Do not open a mains adapter, do not wire mains into a robot, and do not work on a plugged-in device. Everything in this module runs on batteries or low-voltage USB supplies.

The common mistake is ignoring peak voltage. A capacitor rated for 230 V AC has a peak of about 325 V, so the AC rating and the DC rating are not interchangeable. Use the peak for voltage ratings on AC lines.

---

## Conductors and insulators

Conductors like copper, silver and aluminum let charge move with little resistance. Insulators like plastic, glass, rubber and dry air hold charge in place. Semiconductors such as silicon sit between the two and can be switched. Day 9 covers them.

Copper is the default for robot wiring: cheap, flexible and low in resistance. Thicker wire has a lower gauge number and lower resistance, so choose the thickest wire that fits the current, and keep high-current runs short.

On a breadboard, the metal strips under the holes are conductors and the plastic body is an insulator. The long rails along the sides usually carry power and ground, and each short strip is internally connected, so every part plugged into one strip shares a node.

The failure to watch for is contact resistance. A cheap breadboard, a bent jumper or an oxidized component leg adds ohms or fails when the robot vibrates. When a circuit misbehaves, wiggle the connections first. A flickering LED that steadies when you press the breadboard has a contact problem.

Insulation is rated for voltage too. Bare wires that touch can short a battery, and a harness with damaged insulation is a fault waiting to happen. Industrial robot arms lose more uptime to cable failure than to almost any other part.

---

## Ground and reference

Voltage is always a difference, so every measurement needs a reference point. That point is ground, defined as 0 V. Every other voltage in the circuit is measured relative to it. In a battery circuit, ground is usually the negative terminal.

Consider a 9 V battery feeding an LED through a 470 Ω resistor, with the battery negative as ground. The positive terminal sits at +9 V. The resistor takes about 7 V, so the junction at the LED anode is about +7 V. The LED cathode, on the ground side, reads about 0 V. Each number is relative to the same reference.

The common mistake is connecting two circuits with different references. When a microcontroller and a motor driver share a signal line, their grounds must be joined. Without a shared reference, the signal has no fixed meaning, and readings wander. The rule is to tie all grounds in a system to one point, and never tie a supply's positive terminal to another system's ground.

Reference errors cause some of the loudest failures on robots. Motor current returns to the battery through a path that can share copper with logic signals, and the voltage drop along that path shifts the reference the sensors see. This is called ground bounce, and it is why motor power and sensor wiring are routed apart. You will meet it again on Day 12.

Printed circuit board designers route a solid ground plane for the same reason: one reference that every part sees the same way.

---

## Hands-on exercises

These exercises need a DMM, a breadboard, a 9 V battery, resistors and one red LED. Write every reading in a table, because the table is the bench log for the project.

1. **Check your meter.** Set the DMM to resistance and touch the probes together. Record the reading, which should be close to 0 Ω. Switch to DC voltage and measure the battery terminals. Note the sign and explain it.
2. **Measure three resistors.** Read the color code of the 220 Ω, 1 kΩ and 10 kΩ resistors and write each nominal value. Measure each and calculate the percentage error. A 5% tolerance resistor should read within 5% of nominal.
3. **Measure the battery under load.** Record the open-circuit voltage. Connect a 470 Ω resistor across the battery and measure the voltage across the resistor. The loaded reading should sit a little below the open-circuit value.
4. **Build the LED circuit.** Connect the battery positive to the 470 Ω resistor, the resistor to the LED anode (the longer leg), and the cathode back to the battery negative. Measure the voltage across the resistor and across the LED. The two should sum to about 9 V.
5. **Measure the current two ways.** Calculate `I = V_resistor / R`. Then break the loop at the battery negative, set the meter to mA in series, and compare. A difference under 10% is normal for a fresh battery and a 5% resistor.
6. **Calculate the power.** Multiply the resistor voltage by the measured current. Confirm that a 1/4 W resistor handles it, and repeat with the 220 Ω resistor.

**Safety notes:** never measure current across a battery. Do not short a battery with a wire, because a 9 V cell can heat and vent. Remove the battery when you finish. Keep each LED below 20 mA.

**Done when:** you have at least 10 readings, each with a calculated value beside it, and every discrepancy has a one-sentence explanation.

---

## Project: a power and energy calculator with a bench log

This is a light project. Today's concepts are measurement and arithmetic, so the project is a small Python module with tests and a bench log. Its purpose is to make unit conversion automatic.

**Goal:** write functions for Ohm's law, power, energy and battery runtime, test them against this lesson's worked examples, and commit the bench log with measured and calculated values side by side.

**Repo name:** `100-days-robotics-engineering-day-06-electricity-fundamentals`

**Repo layout:**

```
100-days-robotics-engineering-day-06-electricity-fundamentals/
├── README.md
├── electricity.py
├── test_electricity.py
└── measurements/
    └── day06_bench_log.csv
```

**Starter code** (`electricity.py`):

```python
def current(voltage_v: float, resistance_ohm: float) -> float:
    """Ohm's law: I = V / R, in amperes."""
    if resistance_ohm <= 0:
        raise ValueError("resistance must be positive")
    return voltage_v / resistance_ohm


def power(voltage_v: float, current_a: float) -> float:
    """P = V * I, in watts."""
    return voltage_v * current_a


def energy_wh(power_w: float, hours: float) -> float:
    """Energy in watt-hours: E = P * t."""
    return power_w * hours


def battery_runtime_h(capacity_mah: float, voltage_v: float, load_w: float) -> float:
    """Ideal runtime in hours: stored energy (Wh) divided by load power (W)."""
    stored_wh = capacity_mah / 1000 * voltage_v
    return stored_wh / load_w
```

**Step-by-step tasks:**

1. Create the repo and a virtual environment, and install `pytest`.
2. Implement the four functions. Each docstring states the unit of every argument and of the return value.
3. In `test_electricity.py`, use `pytest.approx` to check `current(9, 1000)`, `power(9, 0.009)`, `energy_wh(3, 2)` and `battery_runtime_h(2000, 3.7, 1.5)`, which should be about 4.93.
4. Add a test that a zero resistance raises `ValueError`.
5. Fill `measurements/day06_bench_log.csv` with the columns `item,nominal,measured,calculated,error_percent`, using every reading from the exercises.
6. Write the README: what each function does, how to run `pytest -q`, a units table, and a one-sentence cause for each error above 5%.

**README checklist:**

- [ ] Purpose and setup commands
- [ ] Units table for each function
- [ ] Bench log summary with the largest error and its cause
- [ ] Safety notes from the exercises

**Done when:**

- `pytest -q` passes with at least five tests
- The bench log has at least 10 rows with measured and calculated values
- Every error above 5% has a written cause
- The repo is pushed to GitHub with a clear commit message

**Evidence to commit:** the source, the tests, the CSV, the README, and a photo of the circuit with the multimeter reading visible.

---

## Check your understanding

1. A motor draws 0.8 A from a 6 V supply. Another engineer says it uses 4.8 W for 2 hours, so 9.6 Wh. Is that right? Explain which step, if any, is wrong.
2. You want 10 mA through a red LED (forward drop about 2 V) on a 9 V supply. Calculate the resistor, pick a standard value, and explain why rounding up is the safer direction.
3. A student measures current by placing the meter across a 5 V supply in current mode. Describe what happens to the meter and supply, and state the correct connection.
4. Two resistors each dissipate 0.1 W. One is 100 Ω and the other is 1 kΩ. Which has the larger voltage, and which has the larger current? Show your working.
5. A motor controller is rated for 12 V and 10 A. A team connects it to a 24 V pack, reasoning that it is safe because the current never reaches 10 A. What is the flaw, and which rating protects against it?
6. Two boards share a signal line, but their grounds are not connected, and the readings wander. Explain the physical reason and the fix.

**Answers:**

1. The reasoning is correct. Power is `6 × 0.8 = 4.8 W`, and energy is `4.8 × 2 = 9.6 Wh`. Both steps are right.
2. `R = (9 − 2) / 0.010 = 700 Ω`. The next standard value up is 750 Ω, giving `7 / 750 ≈ 9.3 mA`. Rounding up lowers the current, which is the safe direction for an LED.
3. In current mode the meter has very low resistance, so it acts as a short across the supply. The fuse can blow or the supply can overheat. Put the meter in series with the load.
4. For the 100 Ω resistor, `V = √(0.1 × 100) ≈ 3.16 V` and `I = 0.1 / 3.16 ≈ 0.0316 A`. For the 1 kΩ resistor, `V = √(0.1 × 1000) = 10 V` and `I = 0.1 / 10 = 0.01 A`. The 1 kΩ resistor has the larger voltage, and the 100 Ω resistor has the larger current.
5. A 24 V supply exceeds the 12 V voltage rating, which protects insulation, capacitors and semiconductors from breakdown. The current was never the only risk, so the reasoning misses the failure mode.
6. Each board measures its signal against its own 0 V, so the line has no shared meaning and noise shifts the readings. Join the grounds at one point before connecting the signal.

---

## Sources to read and watch

**Read**

- Wikipedia: *Electric charge*, *Voltage*, *Electric current*, *Electrical resistance and conductance*, *Ohm's law*, *Electric power*, *Electrical energy*, *Alternating current*, *Direct current*, *Ground (electricity)*
- Arduino documentation at [docs.arduino.cc](https://docs.arduino.cc), the Arduino Uno Rev3 section, for the 5 V and 3.3 V pin ratings

**Tutorials**

- SparkFun Learn, the tutorial on voltage, current, resistance and Ohm's law. Search the SparkFun Learn site for the title.
- Adafruit Learning System, the basics of electricity and Ohm's law guides. Search the Adafruit site for "Ohm's law".
- All About Circuits, the DC lessons on voltage, current and resistance. Search the site for the lesson titles.

**Watch**

- Khan Academy, the physics lessons on circuits, current, voltage and resistance
- MIT OpenCourseWare, 6.002 Circuits and Electronics, the opening lectures on Ohm's law and lumped circuits

**Practice**

- [Wokwi](https://wokwi.com/): build the 9 V, 470 Ω and LED circuit, then compare the simulated voltages with your bench readings

---

## Tomorrow

Tomorrow you combine resistors. Today you analyzed one resistor on one source. Circuit analysis covers resistors in series and parallel, how they divide voltage and current, and Kirchhoff's laws for any loop or junction. You will predict every voltage and current before you wire the circuit, and the bench log from today is the test of those predictions.

**Next:** [Day 7: Circuit Analysis](Robotics-Day-07-Circuit-Analysis.md)
