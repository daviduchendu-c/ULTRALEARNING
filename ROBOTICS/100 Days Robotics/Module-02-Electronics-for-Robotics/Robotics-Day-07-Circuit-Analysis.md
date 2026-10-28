# Day 7: Circuit Analysis

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 3 hours
**Level:** beginner. You should have finished Day 6 and be comfortable applying `V = IR` and `P = VI` to a single resistor. Today you apply the same tools to several resistors at once.

**By the end of this day, you will have:**

- Equivalent resistances for series and parallel networks, calculated by hand and checked in code
- Predicted voltages and currents for a voltage divider, a current divider and a three-resistor loop, each confirmed on the bench or in a simulator
- A Python module that computes network values and a SPICE netlist that reproduces them

**What you need:**

- Everything from Day 6 (breadboard, DMM, 9 V battery, resistor assortment, red LEDs)
- Extra resistors: two 10 kΩ, one 4.7 kΩ, one 100 Ω, one 300 Ω, one 330 Ω, one 2.2 kΩ (about $3 extra if you bought the Day 6 assortment)
- Two more red LEDs, for a three-LED series string
- A 5 V DC supply for the bench: a USB charger with a USB-A to jumper-wire breakout, or a regulated bench supply (about $20 for the breakout, or $80 and up for a bench supply)
- Python 3.10 or newer, and `pytest` 8.x (`pip install pytest`)
- Free LTspice from Analog Devices, for the simulation exercise. Search "LTspice" on the analog.com site for the download page. It is free and runs on Windows and macOS.

**Prerequisites:**

- Day 6: Electricity Fundamentals, especially Ohm's law and the difference between measuring voltage across a part and current through it
- Algebra with fractions and reciprocals
- Basic Python: functions, tuples and `pytest.approx`

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Series circuits](#series-circuits)
- [Parallel circuits](#parallel-circuits)
- [Equivalent resistance](#equivalent-resistance)
- [Voltage division](#voltage-division)
- [Current division](#current-division)
- [Kirchhoff's current law](#kirchhoffs-current-law)
- [Kirchhoff's voltage law](#kirchhoffs-voltage-law)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a resistor network calculator with a SPICE check](#project-a-resistor-network-calculator-with-a-spice-check)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A real robot has dozens of resistors. Pull-up resistors hold sensor lines high, dividers scale battery voltage down to a level an analog pin can read, and LED strings light status indicators. You will not calculate each one from scratch by intuition. You need a method that works on any network of resistors, and that method is circuit analysis.

Circuit analysis is the habit of predicting every voltage and current before you power the circuit. It has three parts: reduce a network to a single equivalent resistance, split the voltage or current with the divider rules, and check the answer with two conservation laws. Kirchhoff's current law says charge does not pile up at a junction. Kirchhoff's voltage law says the voltages around a loop add to zero.

On the job, a sensor reads 2.9 V on one board and 2.2 V on another, and the engineer who finds the reason is usually the one who can predict the loaded voltage of a divider in their head.

---

## Short history

Network analysis grew out of the telegraph and telephone industries, where engineers needed to predict signals over long lines.

| Year | Person or event | What it added |
|---|---|---|
| 1845 | Gustav Kirchhoff publishes his circuit laws while still a student | The current and voltage laws, stated as conservation principles. |
| 1880s | Oliver Heaviside develops operational methods for telegraph and telephone lines | Circuits solved as equations of networks, which later became standard. |
| 1883 | Léon Charles Thévenin publishes his equivalent-circuit theorem | Any linear network can be replaced by one source and one resistor for analysis. |
| 1970s onward | SPICE simulators such as Berkeley SPICE spread through industry | Engineers simulate networks before building them. |

---

## Glossary

- **Node:** a point where two or more components connect.
- **Branch:** a path between two nodes that holds one component or a chain of components.
- **Loop (mesh):** a closed path through the network that returns to its starting node.
- **Series:** components connected end to end, so the same current flows through each.
- **Parallel:** components connected across the same two nodes, so each has the same voltage across it.
- **Equivalent resistance:** a single resistance that behaves the same as a network, as seen from its two terminals.
- **Voltage divider:** two series resistors that split a source voltage into a smaller fraction.
- **Current divider:** two parallel resistors that split a total current in inverse proportion to their resistances.
- **Kirchhoff's current law (KCL):** the currents entering a node equal the currents leaving it.
- **Kirchhoff's voltage law (KVL):** the sum of voltage rises and drops around any closed loop is zero.
- **Netlist:** a text description of a circuit, with a line for each component and its nodes, used by simulators such as SPICE.
- **Loading:** the reduction in a divider's output voltage caused by a connected load.

---

## Series circuits

Imagine a row of turnstiles along a single path. Everyone who passes the first one must pass every other turnstile in the row. A series circuit works the same way: the same current flows through each component, because charge has no other route.

Because the current is shared, the resistances add. For resistors in series, the total is `R_total = R1 + R2 + R3`. The voltages across the resistors add to the source voltage, which is KVL, covered later in this lesson.

Take a 12 V supply, with a 220 Ω, a 330 Ω and a 1 kΩ resistor in series. The total resistance is `220 + 330 + 1000 = 1550 Ω`. The current is `I = 12 / 1550 ≈ 0.00774 A`, or 7.74 mA. That same 7.74 mA flows through all three. The voltage across the 220 Ω resistor is `7.74 mA × 220 Ω ≈ 1.70 V`, across the 330 Ω resistor `7.74 mA × 330 Ω ≈ 2.55 V`, and across the 1 kΩ resistor about 7.74 V. The three drops add to 12 V.

The common mistake is assuming that a series chain of components shares the supply voltage equally. It shares it in proportion to resistance, and the biggest resistor takes the largest share. A failure to watch for is an open connection in a series string. One broken contact stops current through the whole string, which is why a single bad solder joint can dark an entire LED strip.

Series networks show up in robots as LED strings with one current-limiting resistor, and in sensors that require a fixed resistor to be wired in line with a variable one.

---

## Parallel circuits

A parallel circuit is a set of paths that all start at one node and end at another. Each path sees the same voltage across it, and the total current splits among the paths. Parallel connections are the way to add current capacity, which is why batteries are wired in parallel to increase capacity.

For resistors in parallel, the reciprocals add:

```
1 / R_total = 1 / R1 + 1 / R2 + 1 / R3
```

Two resistors in parallel give the simple product-over-sum form, `R_total = (R1 × R2) / (R1 + R2)`. Take 100 Ω in parallel with 220 Ω. The product is 22,000 and the sum is 320, so `R_total = 22,000 / 320 = 68.75 Ω`. The total is always lower than the smallest branch. Three 1 kΩ resistors in parallel give `1000 / 3 ≈ 333 Ω`.

The common mistake is using the parallel formula for series parts or the series formula for parallel parts. A quick check resolves it: the answer for parallel must be smaller than the smallest resistor, and the answer for series must be larger than the largest.

LEDs in parallel have a specific limitation. Placing three LEDs in parallel with a single resistor looks economical, but LEDs differ in forward voltage. The one with the lowest forward voltage takes most of the current, and it can overheat while the others barely light. Each LED in a parallel group needs its own series resistor, or the group must be avoided.

Parallel networks appear in power distribution, where the total load of several motors is shared across the supply, and in sensor arrays that are wired to a common bus.

---

## Equivalent resistance

Equivalent resistance is the technique of reducing a network step by step to a single resistance. You simplify the innermost pieces first, replacing each series or parallel group with one resistor, and repeat until one value remains.

Consider a network: R1 = 100 Ω in parallel with R2 = 220 Ω, and that combination in series with R3 = 300 Ω. First reduce the parallel pair to `68.75 Ω`. Then add the series resistor: `68.75 + 300 = 368.75 Ω`. Apply 12 V across the whole network, and the total current is `12 / 368.75 ≈ 32.5 mA`.

The method works for any combination of series and parallel parts. It fails for bridge networks, where no pair is purely series or purely parallel. In those cases you need nodal or mesh analysis, which is beyond today's scope.

The common mistake is reducing a pair that is not in parallel. Two resistors share a node only if both of their ends connect to the same two nodes. Trace each path on the schematic with a pencil before you combine anything.

Equivalent resistance is the first tool a motor-driver or sensor-interface engineer uses to estimate the load a circuit presents to its supply. Datasheets quote input impedance for the same reason.

---

## Voltage division

A voltage divider is two resistors in series across a source, with the output taken from the junction between them. The output is a fraction of the input, set by the ratio of the resistors. The rule is:

```
V_out = V_in × R2 / (R1 + R2)
```

Here R1 is the upper resistor, connected to the source, and R2 is the lower resistor, connected to ground. The current through both is the same, and the voltage drop across each is proportional to its resistance.

Take V_in = 9 V, R1 = 10 kΩ and R2 = 4.7 kΩ. Then `V_out = 9 × 4.7 / 14.7 ≈ 2.88 V`. Check: the drop across R1 is `9 − 2.88 = 6.12 V`, and the ratio of drops is `6.12 / 2.88 ≈ 2.13`, which equals `10 / 4.7 ≈ 2.13`. The ratio holds.

Dividers are everywhere in robotics. A battery monitor divides a 7.4 V pack down to about 3 V for a microcontroller pin, and a potentiometer is a divider whose ratio you turn with a knob.

Now add a load. Connect a 10 kΩ resistor across the lower resistor, as a sensor input might. The lower leg becomes `4.7 kΩ ∥ 10 kΩ = 3.20 kΩ`. The output becomes `9 × 3.20 / (10 + 3.20) ≈ 2.18 V`. The load pulled the output from 2.88 V to 2.18 V, a drop of about 24 percent. This is loading, and it is the most common reason a divider reads wrong on real hardware.

The common mistake is forgetting the load. A divider designed for an open-circuit output misreads as soon as a meter or pin draws current from it. The fix is a low-impedance buffer, or resistors high enough that the load is small compared to the lower leg. A limitation: higher resistances save current but make the output more sensitive to noise and to the load.

---

## Current division

A current divider is two resistors in parallel, with a total current flowing into the node. The total splits so that more current goes through the smaller resistor. For a branch with resistance R_a, in parallel with R_b:

```
I_a = I_total × R_b / (R_a + R_b)
```

Note the order: the current through R_a depends on the other resistor, R_b. Take a 12 mA source into 100 Ω in parallel with 300 Ω. The current through the 100 Ω branch is `12 × 300 / 400 = 9 mA`, and through the 300 Ω branch is `12 × 100 / 400 = 3 mA`. Check the sum: `9 + 3 = 12 mA`. Check the voltage: both branches share a voltage, and `9 mA × 100 Ω = 0.9 V`, while `3 mA × 300 Ω = 0.9 V`. The two checks agree.

The most common mistake is the order of terms. Students write `I_a = I_total × R_a / (R_a + R_b)`, which gives the current through the wrong branch. Test it with the check: the smaller resistor should carry the larger current.

Current dividers matter when a sensor or LED is driven by a fixed current source, and when a shunt resistor is placed in parallel with a meter movement to measure larger currents. A limitation: the divider only works while the source delivers the total current, and a real source with a voltage limit can fall out of regulation when the load resistances are too low.

---

## Kirchhoff's current law

Kirchhoff's current law says that the total current entering a node equals the total current leaving it. Charge cannot pile up at a point in a steady circuit, so the inflow and outflow must balance.

Take a node with 5 mA entering from one wire. Two branches leave: 2 mA through one resistor and 3 mA through another. The sum out is `2 + 3 = 5 mA`, which matches the sum in. If your bench meter shows 5.1 mA in and 2 mA plus 3 mA out, the 0.1 mA is measurement error or a leak, and you should find which.

Use KCL to find an unknown current. At a node with 12 mA entering, one branch carrying 4.5 mA and another unknown, the unknown is `12 − 4.5 = 7.5 mA`.

The common mistake is signing errors. Pick a convention, such as currents entering a node are positive, and apply it everywhere. A limitation: KCL assumes steady current, so the law itself still holds for capacitors, but the currents are not steady while a capacitor charges. Day 8 introduces that transient case.

KCL is what every power-distribution board relies on. The total current from a battery equals the sum of the currents to every motor, driver and sensor it feeds, and the sum must match the battery's rating.

---

## Kirchhoff's voltage law

Kirchhoff's voltage law says that going once around any closed loop, the voltage rises and drops sum to zero. Energy is conserved: the energy a charge gains from the source is spent in the resistors, and nothing is left over.

Build a loop with a 9 V source, a 2.0 V drop across one resistor, a 3.5 V drop across a second and a 3.5 V drop across a third. Starting at the source, the rise is +9 V, and the drops are −2.0, −3.5 and −3.5 V. The sum is `9 − 2.0 − 3.5 − 3.5 = 0`. If the sum is not zero, you have missed a component, a meter reading is wrong, or there is a contact resistance you have not counted.

KVL is the basis for predicting the voltage across each part in a series string. Given a 12 V supply and three resistors in series, you know the drops must add to 12 V before you measure anything.

The common mistake is tracing the wrong direction. Choose a direction around the loop, and count every source and drop with its sign relative to that direction. A limitation: KVL holds in a lumped circuit, where wires are short compared to the wavelength of the signals. Long cables and fast signals can carry voltage across distance, and that case belongs to transmission-line theory, which you will see in later robot communication work.

KVL underlies the checks used by design and test engineers: when the measured loop voltages do not sum to zero, a connection is faulty.

---

## Hands-on exercises

Predict every value on paper before you measure it. Record predicted and measured values side by side.

1. **Series string.** Wire the 220 Ω, 330 Ω and 1 kΩ resistors in series across the 9 V battery. Predict the total resistance, the current and the voltage across each resistor. Measure the voltage across each one and the current in the loop with the DMM in mA. Explain any difference above 5%.
2. **Voltage divider.** Build the 10 kΩ and 4.7 kΩ divider from 9 V. Predict `V_out` (about 2.88 V). Measure it. Then add a 10 kΩ load across the 4.7 kΩ resistor and predict the new output (about 2.18 V). Measure again and note the drop.
3. **Current divider.** Build 100 Ω in parallel with 300 Ω. Connect the 9 V battery through a 1 kΩ resistor to make a rough current source. Predict the branch currents. Measure each branch in series with the DMM. Verify that the two branches have the same voltage.
4. **Three-LED string.** Wire three red LEDs in series with a resistor across the 9 V battery. The string drops about 6 V, so the resistor takes about 3 V. Pick 220 Ω and predict a current of about 13.6 mA. Measure the current and the voltage across each LED.
5. **Simulate.** Open LTspice, create a new schematic, and place a 9 V voltage source, a 10 kΩ resistor, a 4.7 kΩ resistor and a ground. Use the following netlist for the same divider and run an operating point analysis (Simulate, then Run, then read the node voltages):

```
* Voltage divider check
V1 in 0 9
R1 in out 10k
R2 out 0 4.7k
.op
.end
```

   The simulated `V(out)` should match your hand calculation of 2.88 V.

**Safety notes:** measure current only in series, and never across a battery. Keep the LED string current below 20 mA. Remove power before you change wiring.

**Done when:** the table shows predicted and measured values for at least four circuits, and each difference above 5% has a written cause.

---

## Project: a resistor network calculator with a SPICE check

This project is light. The calculations are small, but building a tested calculator and checking it against a simulator teaches the habit of verifying a prediction before you trust it.

**Goal:** write functions for series and parallel resistance, the two divider rules and a KCL check, test them, and reproduce one divider in a SPICE netlist.

**Repo name:** `100-days-robotics-engineering-day-07-circuit-analysis`

**Repo layout:**

```
100-days-robotics-engineering-day-07-circuit-analysis/
├── README.md
├── netlists/
│   └── divider.cir
├── resistor_network.py
└── test_resistor_network.py
```

**Starter code** (`resistor_network.py`):

```python
def series(*resistors_ohm: float) -> float:
    """Equivalent resistance of resistors in series."""
    if any(r <= 0 for r in resistors_ohm):
        raise ValueError("resistances must be positive")
    return sum(resistors_ohm)


def parallel(*resistors_ohm: float) -> float:
    """Equivalent resistance of resistors in parallel: 1/R = sum of 1/Ri."""
    if any(r <= 0 for r in resistors_ohm):
        raise ValueError("resistances must be positive")
    return 1 / sum(1 / r for r in resistors_ohm)


def voltage_divider(v_in: float, r_top: float, r_bottom: float) -> float:
    """Unloaded divider output: V_out = V_in * R_bottom / (R_top + R_bottom)."""
    return v_in * r_bottom / (r_top + r_bottom)


def current_divider(i_total: float, r_branch: float, r_other: float) -> float:
    """Current through r_branch: I = I_total * R_other / (R_branch + R_other)."""
    return i_total * r_other / (r_branch + r_other)


def kcl_residual(currents_in: list[float], currents_out: list[float]) -> float:
    """Sum of currents in minus sum of currents out. Zero means KCL holds."""
    return sum(currents_in) - sum(currents_out)
```

**Step-by-step tasks:**

1. Create the repo and a virtual environment, and install `pytest`.
2. Implement the five functions. Keep the units in the docstrings: ohms, volts and amperes.
3. Write tests with `pytest.approx`. Cover `series(220, 330, 1000) == 1550`, `parallel(100, 220) ≈ 68.75`, `voltage_divider(9, 10000, 4700) ≈ 2.878`, `current_divider(0.012, 100, 300) ≈ 0.009`, and `kcl_residual([0.005], [0.002, 0.003]) == 0`.
4. Add a test that a zero resistance raises `ValueError` in `parallel`.
5. Write `netlists/divider.cir` with the netlist from the exercises, and record the `V(out)` value from LTspice in the README beside your Python result.
6. Write the README: how to run the tests, a table of the functions and their units, the predicted and simulated divider result, and one paragraph on loading with the 10 kΩ load case.

**README checklist:**

- [ ] Setup and run commands
- [ ] Units and formula for each function
- [ ] Predicted versus simulated divider output
- [ ] Loading case, with the 2.18 V result
- [ ] One limitation of the divider model

**Done when:**

- `pytest -q` passes with at least six tests
- The LTspice result matches the Python result to within 0.01 V
- The README states the loading effect in numbers
- The repo is pushed to GitHub with a clear commit message

**Evidence to commit:** the source, the tests, the netlist, a screenshot of the LTspice operating-point result and the README. Explain in one line why the LTspice and Python results agree or differ.

---

## Check your understanding

1. A 6 V supply feeds a 470 Ω resistor in series with a 330 Ω resistor. A colleague says each resistor has 3 V across it, since they are equal in number of ohms. Explain why that is wrong and calculate the actual voltage across each.
2. You want a divider that gives 3.0 V from a 9 V supply, using a 10 kΩ top resistor. Find the bottom resistor value. Then explain why a 10 kΩ load on that divider would cause a problem.
3. Three LEDs in parallel share one 220 Ω resistor on a 5 V supply. The LEDs have slightly different forward voltages. Explain what happens to the current in each LED, and what change fixes it.
4. At a node, 15 mA enters from a sensor, 4 mA leaves through one branch, and a second branch carries an unknown current. The meter reads 13 mA on the second branch. Is KCL satisfied? If not, what is the likely explanation?
5. A current divider has 20 mA flowing into 150 Ω in parallel with 50 Ω. Find the current through each branch and confirm the voltage across both is the same.
6. A student says Kirchhoff's voltage law fails for a loop of wires that is 500 m long and carries a fast signal. Is the student right? Explain what assumption KVL depends on.

**Answers:**

1. The voltage divides in proportion to resistance, so the two resistors get unequal shares. The total is `470 + 330 = 800 Ω`, and the current is `6 / 800 = 7.5 mA`. The 470 Ω resistor drops `7.5 mA × 470 = 3.53 V`, and the 330 Ω resistor drops `7.5 mA × 330 = 2.48 V`. The sum is `3.53 + 2.48 = 6.0 V`.
2. Set `V_out = 9 × R_bottom / (10,000 + R_bottom) = 3.0`. Solving gives `R_bottom = 5,000 Ω`, so use 5.0 kΩ (or 4.99 kΩ if you have 1% parts). A 10 kΩ load on that divider would pull the output down strongly, because the lower leg becomes about 3.3 kΩ, which changes the ratio.
3. Each LED takes current that depends on its forward voltage, so the LED with the lowest forward voltage carries the most current and may overheat. Give each LED its own series resistor.
4. KCL is not satisfied: 15 mA in is not equal to 4 mA plus 13 mA, which is 17 mA out. The discrepancy is 2 mA. Check the meter's reading, the wiring at the node, and any unintended leak path.
5. Current through the 150 Ω branch is `20 × 50 / 200 = 5 mA`, and through the 50 Ω branch is `20 × 150 / 200 = 15 mA`. The sum is 20 mA. The voltage across each is `5 mA × 150 = 0.75 V` and `15 mA × 50 = 0.75 V`, so they match.
6. The student is right to question it, and the answer depends on the wavelength of the signal. KVL assumes a lumped circuit, where the wires are short compared to the signal's wavelength. For a 500 m loop carrying a fast signal, distributed effects matter, and simple KVL on the wires is not the right model.

---

## Sources to read and watch

**Read**

- Wikipedia: *Series and parallel circuits*, *Voltage divider*, *Current divider*, *Kirchhoff's circuit laws*, *Electrical network*, *Thévenin's theorem*
- The LTspice help files, available from Analog Devices, for the operating-point analysis and the netlist syntax

**Tutorials**

- SparkFun Learn, the tutorial on series and parallel circuits. Search the SparkFun Learn site for the title.
- All About Circuits, the DC circuits lessons on Kirchhoff's laws and voltage dividers. Search the site for the lesson titles.
- Adafruit Learning System, the guides on resistors and voltage dividers. Search the Adafruit site for "voltage divider".

**Watch**

- Khan Academy, the circuits lessons on resistors in series and parallel, and on Kirchhoff's laws
- MIT OpenCourseWare, 6.002 Circuits and Electronics, the lectures on nodal analysis and the lumped-element model

**Practice**

- [Wokwi](https://wokwi.com/): build the three-resistor series string and the current divider, and compare the simulated currents with your predictions

---

## Tomorrow

Tomorrow you meet the parts that fill these circuits. Today every component was a plain resistor. Tomorrow you cover potentiometers, capacitors, inductors, diodes, LEDs, switches, relays, connectors and fuses. Capacitors and inductors change over time, so the steady-state rules from today will need a time dimension. You will see how a capacitor's charge curve explains a button that bounces, and why a motor coil needs a flyback diode.

**Next:** [Day 8: Components](Robotics-Day-08-Components.md)
