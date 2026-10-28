# Day 14: Schematics and Datasheets

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 3 to 4 hours
**Level:** intermediate. You need Days 6 to 13. Today is a reading and drawing day, so the difficulty is in patience and detail, not in new physics. It is the skill that separates a hobby build from a design someone else can build, repair or buy.

**By the end of this day, you will have:**

- A schematic of the Day 13 motor switching stage, drawn in KiCad 9 with reference designators, net names and rail labels
- An electrical rules check (ERC) run on that schematic, with each warning either fixed or explained
- A datasheet review table for two parts, listing the absolute maximum ratings, the recommended operating conditions and the pinout
- A timing analysis of the HC-SR04 ultrasonic sensor's trigger and echo signals, from its datasheet

**What you need:**

- KiCad 9.0 or newer, free and open source, for Windows, macOS and Linux. Download from kicad.org. The installer includes the symbol and footprint libraries.
- A web browser with access to the datasheets for: the TB6612FNG (Toshiba), the IRLZ44N (Infineon or its original manufacturer), the 1N5819 (a major diode maker) and the HC-SR04 ultrasonic module. Each is free to download from the manufacturer's site or from a distributor such as Digi-Key or Mouser.
- A PDF reader that shows bookmarks and lets you search the text.
- A printed copy or a second screen for the datasheets, which is optional but helpful.
- Optional: a physical Day 13 motor board, which you use to compare with the schematic.
- Cost: free if you use the software and datasheets you already have, plus about $0 to $10 if you order a printed datasheet or a KiCad reference card.

**Prerequisites:**

- Day 8 components: resistors, capacitors, diodes, MOSFETs and connectors, with their symbols
- Day 9: the MOSFET as a switch
- Day 13: the H-bridge, the flyback diode and the switching stage you built on the breadboard
- A basic ability to read a table of numbers with units

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How circuit drawings became a standard](#how-circuit-drawings-became-a-standard)
- [Glossary](#glossary)
- [Symbols and reference designators](#symbols-and-reference-designators)
- [Nets and rails](#nets-and-rails)
- [IC pins](#ic-pins)
- [Connectors](#connectors)
- [Ratings and operating conditions](#ratings-and-operating-conditions)
- [Timing diagrams](#timing-diagrams)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: a schematic and datasheet pack for the motor switching stage](#project-a-schematic-and-datasheet-pack-for-the-motor-switching-stage)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A circuit that lives only on a breadboard is a conversation between you and the wires. A schematic is the written version of that conversation, and a datasheet is the manufacturer's statement of what the part will and will not do. Engineers who cannot read both make mistakes that are invisible until a board is built: a pin is wired to the wrong rail, a part is run above its absolute maximum rating, or a timing requirement is ignored because the datasheet was never opened.

The skill is practical. A new team member has to understand a board in an afternoon. A supplier has to know which part to order. A repair technician has to replace a failed MOSFET with one that has the same ratings. Each of those tasks depends on reading the symbols, the net names, the pin tables and the timing diagrams correctly. Today you learn to read them and to draw them yourself.

---

## How circuit drawings became a standard

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1800s to early 1900s | Electrical drawings use pictures of components, which differ from one company to another. | Early wiring diagrams were hard to share between manufacturers. |
| 1910s to 1930s | Standard symbols for resistors, capacitors and switches appear in engineering texts. | The symbols you use today descend from these drawings. |
| 1940s to 1960s | Printed circuit boards and integrated circuits make schematics more detailed and more layered. | Net names and reference designators become necessary to keep track of hundreds of parts. |
| 1970s onward | Electronic design automation (EDA) software draws, checks and netlists schematics. | Tools such as KiCad and Altium check the wiring before a board is made. |
| Today | Manufacturers publish datasheets as standard, with absolute maximum ratings, electrical characteristics and timing diagrams. | Reading a datasheet is the first step in any part selection. |

The lesson from this history is that the drawing standard exists so that a person who did not design the board can still build it, test it and fix it.

---

## Glossary

- **Schematic:** a diagram that shows components and how they are connected, using standard symbols rather than pictures of the parts.
- **Symbol:** the standard drawing of a component, such as the zigzag for a resistor.
- **Reference designator:** a short code that identifies a component on a schematic, such as R1 for the first resistor, C3 for the third capacitor, U1 for the first integrated circuit and Q2 for the second transistor.
- **Net:** a set of pins and wires that are electrically connected and carry the same voltage.
- **Net name:** a label on a net, such as GND, +5V or MOTOR_A, which lets you read the schematic without tracing every wire.
- **Rail:** a net that carries a fixed supply voltage to several loads, such as +5V or VBAT.
- **Pinout:** the map from a chip's pin numbers to their functions.
- **Connector:** a socket or header that joins a board to external wires or another board, with its own pin numbering.
- **Absolute maximum rating:** a limit beyond which a part may be permanently damaged, even for a moment.
- **Recommended operating condition:** the range within which the part is guaranteed to meet its electrical characteristics.
- **Electrical rules check (ERC):** a software check that finds unconnected pins, conflicting outputs and missing power connections in a schematic.
- **Netlist:** a text list of every net and the pins it connects, which a layout tool uses to place parts and copper.
- **Timing diagram:** a graph of digital signals against time, showing when each signal changes and how long each state lasts.
- **Setup and hold time:** the minimum time a signal must be stable before and after a clock edge.

---

## Symbols and reference designators

A map uses standard symbols for roads, rivers and rail lines, so a reader from any city knows what a line means. Electronic symbols work the same way. A resistor is a zigzag or a rectangle, a capacitor is two parallel lines, an inductor is a row of arcs, and a diode is a triangle with a bar. A MOSFET has a gate, a drain and a source, drawn with a line for the channel and an arrow for the body diode direction.

A symbol represents a function, not a physical shape. A 10 kΩ resistor through-hole and a 10 kΩ surface-mount resistor have the same symbol, because they do the same job in the circuit. The symbol tells you the function and the terminals, and the value, package and rating are written next to it.

Each symbol is labelled with a reference designator and a value. Here is a worked example. A schematic shows R1 with a value of 100 Ω, R2 with 10 kΩ, C1 with 100 nF, D1 as a 1N5819 and Q1 as an IRLZ44N. Reading that list tells you the component count, the function of each part and the part you must order. A bill of materials (BOM) is the list made from these labels, with quantity, value, package and manufacturer part number. It is the document a buyer or a production line uses.

The common mistake is to reuse a designator, such as having two parts called R3 on the same sheet. Every tool will flag it, and a builder who follows the wrong one will solder the wrong part. The fix is to number sequentially within each sheet, and to run the ERC, which catches duplicate designators.

On the job, a clean schematic with a complete BOM is the first thing a contract manufacturer asks for. A company that cannot provide one cannot get a board made at volume.

---

## Nets and rails

A subway map shows which stations share a line, not the exact path of each track. A net is the same kind of abstraction. Two pins are on the same net if they are connected by wire, and the schematic shows the connection by a name rather than by drawing every wire.

Electrically, everything on one net is at the same voltage, at every instant, if the wire resistance is negligible. That is why a net name is enough: a reader sees GND and knows that every GND pin in the drawing is at zero volts relative to the reference. A net named +5V is at 5 V with respect to GND. The name is a claim about voltage, and the board must make it true.

Rails are nets with a fixed supply voltage, and they are named after the voltage, such as +5V, +3V3 and VBAT. A rail is drawn once with a symbol for a power port, and each load connects to it by name. The advantage is that the schematic stays readable when a board has forty loads on one rail.

Here is the worked example from Day 13. The H-bridge switching stage has the supply net VBAT, the ground net GND and the motor terminal nets MOTOR_A and MOTOR_B. The MOSFET drain connects to MOTOR_B, the flyback diode cathode connects to VBAT and its anode connects to MOTOR_B. Each of those net names appears where it is used, so the reader can follow the current path without tracing wires across the sheet.

A net name can lie. Two nets with different names may be joined on the board by a short that is not on the schematic, and a net with the same name on two sheets may not be connected at all unless the design tool joins them. The ERC checks some of this, and the rest is a careful review.

The mistake is to give a net a name that says what you want rather than what it is. A net named 5V_SAFE that is actually connected to the battery is a fault waiting to happen. The rule is to name nets for their function and to check the name against the voltage you measure.

Net naming is the common language between designers, board houses and firmware writers. A firmware developer who sees a pin named MOTOR_PWM knows which signal to set without opening the hardware manual.

---

## IC pins

A hotel key card has a number on the front that tells the staff which room it opens. An integrated circuit's pin number is the same kind of address. The datasheet pinout maps each pin number on the package to its function, such as VM, GND or AIN1, and the schematic symbol shows the same functions around its outline.

Reading a pin means identifying its direction, its type and its limits. Direction says whether the pin is an input, an output or both. Type says whether it is a digital signal, an analog signal, a power pin or a ground pin. Limits are the voltage and current ratings for that pin. A microcontroller pin marked as an output is driven by the chip and can source or sink a limited current. An input pin with a pull-up resistor is held high until something pulls it down.

The TB6612FNG datasheet shows the pinout. Pin names such as AIN1, AIN2 and PWMA are the direction inputs and the PWM input for channel A, and AO1 and AO2 are the motor outputs. VM is the motor supply and VCC is the logic supply. The schematic for your breakout should show each of these pins connected to the net that carries the signal. Check the connections against the datasheet table, not against your memory of the breakout's silkscreen.

A worked example of checking is short. The Day 13 exercise connected AIN1 to D7. In the datasheet, AIN1 is a logic input, and a 5 V logic high on a 5 V logic supply is within the input thresholds. The pin is correct. If you had connected AIN1 to the motor supply VM, you would have driven a logic input at 6 V, above its rating, and the chip would be at risk.

The common mistake is to read the pin numbers from the drawing rather than from the package. Surface-mount packages often number their pins counterclockwise from a dot or notch, and a breakout board may renumber them. Always check the pin one marking and the package drawing in the datasheet before you solder.

On the job, a pinout error is one of the most common causes of a board respin. The cost of a respin is weeks, and the cause is a line in a table that was not read.

---

## Connectors

A power strip with its outlets labelled by their function makes it safe to plug in the right appliance. A connector on a board does the same job for wires: it names the signal and its order, so the wires go in the right place. Connectors are drawn on the schematic as a block with numbered pins, and their pinout is documented in the same way as a chip's.

Connectors come down to the pin count and the mating orientation. A two-pin connector for a battery must be keyed so that it cannot be plugged in backward, and a motor connector should have distinct pins for the two motor terminals. The schematic must show the pin number, the net on each pin, and the orientation of the keying, so that the cable maker builds the right harness.

A worked example for the HC-SR04 module and the servo from Day 15. The HC-SR04 has four pins: VCC, Trig, Echo and GND. The servo has three: signal, V+ and ground. A 4-pin header and a 3-pin header on the board make both connections clear. If the connector pin order does not match the module, the sensor is wired backward. Check the pin order against the module's silkscreen, not against a picture on a website.

A connector's limits are contact resistance and mechanical wear. A jumper-wire connection on a breadboard can be 0.1 Ω or more, and it changes as the wires are moved. A connector carrying current should be sized for the current, with its contact rating checked in its datasheet, and a connector that carries motor current should be rated for it.

The mistake is to reuse a connector with a different pinout. A 3-pin servo header and a 3-pin sensor header look the same, and a servo plugged into a sensor header can apply 5 V to a signal pin that expects a logic level. The check is the label on the schematic and the colour of the wire. Label both ends and check before power is applied.

Connectors are standardized. JST and Molex sell connectors with named pitches, and robotics kits use keyed headers so that a wrong connection cannot be made by accident.

---

## Ratings and operating conditions

Every datasheet has two numbers for each limit. The absolute maximum rating is the value beyond which the part may be damaged, even for a moment. The recommended operating condition is the value within which the part meets its specification. The difference is a margin, and designing without that margin is the most common cause of field failures.

The rule is simple. A part rated for an absolute maximum of 5.5 V on its supply pin and a recommended operating range of 2.7 V to 5.5 V is safe at 5 V with no margin for a supply spike. A part with a recommended maximum of 4.5 V running on a 5 V rail is out of specification even though it may work on the bench. The safe design uses the recommended range with a margin of at least 10 to 20 percent for transients.

Here is the worked example from the microcontroller. The ATmega328P datasheet from Microchip lists an absolute maximum current of 40 mA per I/O pin. A designer who drives an LED directly from a pin should keep the current well below that figure. With a 5 V supply, a forward LED drop of 2 V and a 220 Ω resistor, the current is I = (5 − 2) / 220 = 13.6 mA. The power in the resistor is I² × R = 0.0136² × 220 = 0.041 W, so a 1/4 W resistor is sufficient. The pin's 13.6 mA is well below 40 mA, and it sits near the level many designers use as a comfortable limit of 20 mA per pin.

A second worked example is the MOSFET from Day 13. The IRLZ44N's datasheet lists a drain-source voltage rating of 55 V. The switching stage runs from a 6 V supply, and the flyback diode clamps the spike to about 6.3 V. The margin is 55 / 6.3 = 8.7 times, which is far above the requirement. A designer who used a 30 V MOSFET on the same stage would have a margin of 30 / 6.3 = 4.8 times, which is still acceptable, but the cost of the spike events in a real motor would reduce that margin.

A datasheet's ratings are tested under specific conditions, often at 25 degrees Celsius with a specific load. A part at 85 degrees Celsius inside a sealed robot may run at a lower current rating, which the datasheet's derating curve shows. Read the notes under the table and the curves, not only the headline number.

The common mistake is to design to the absolute maximum rating. A chip rated for 5.5 V absolute maximum is not a 5.5 V chip for design purposes. The way to spot it is to ask whether the design survives a 10 percent rise in the supply, a transient on the motor line, and a warm enclosure. If not, the design has no margin.

Rating margins are the difference between a product that lasts and one that returns. Automotive and industrial robot parts carry tighter margins and wider temperature ratings for that reason.

---

## Timing diagrams

A railway timetable is a timing diagram for trains. It shows when each train leaves, how long it takes and when it must be at a station. A timing diagram in a datasheet shows the same thing for digital signals: which signal changes, when, and how long it must hold its value.

A timing diagram has three parts: the time axis, the signal lines and the timing parameters. The horizontal axis is time, usually with labels such as t1, t2 and t3. Each signal is a line that is high or low, with transitions at specified times. A parameter such as t1 = 10 µs is the minimum or maximum duration of a state, and the datasheet gives it as a number with a unit.

The HC-SR04 ultrasonic module is the worked example. Its trigger input needs a high pulse of at least 10 µs. After the trigger, the module emits eight ultrasonic pulses and sets its echo output high. The echo stays high for a time proportional to the distance to the object. For a round trip, the sound travels twice the distance at about 343 m/s, so the echo width is t = 2d / 343. At 1 m, t = 2 / 343 = 0.00583 s, or 5.83 ms. At the module's 4 m maximum range, t = 8 / 343 = 0.0233 s, or 23.3 ms. A timing diagram that shows a 10 µs trigger and a variable echo tells you what the code must measure, and the arithmetic tells you the timeout the code needs.

Timing diagrams also show setup and hold times for synchronous parts. A data line must be stable for a set time before a clock edge, which is the setup time, and for a set time after it, which is the hold time. Violating these times causes intermittent faults that vanish when the signal is probed.

A timing diagram shows the typical case, while the guaranteed limit is in the table. A diagram that shows 10 µs may be drawn to scale for one example. Read the parameter table for the minimum or maximum value, and design to that value.

The common mistake is to assume the timing diagram's default conditions apply to your board. A signal with a long wire, a high capacitance or a slow driver may not meet the rise time in the diagram. The test is to measure the edge with an oscilloscope or with a logic analyser, if you have one, and compare it with the datasheet.

On the job, timing diagrams are the basis of the firmware for every sensor interface. The firmware that reads a sensor without timing checks is the firmware that fails in the field when the temperature changes.

---

## Hands-on exercises (about 60 minutes)

1. **Draw the switching stage.** In KiCad 9, create a new project called `motor_switch_stage`. Draw the Day 13 switching stage: the IRLZ44N, its gate resistor and pull-down, the flyback diode, the motor and the input connector. Use reference designators (Q1, R1, R2, D1, M1, J1) and net names (VBAT, GND, MOTOR_B, GATE_DRIVE, PWM_IN). Add a power port symbol for each rail.

2. **Run the electrical rules check.** In the schematic editor, run the ERC from the Inspect menu. Read every warning. For each warning, either fix the schematic or write a line in the review notes that explains why the warning is expected, such as an intentionally unconnected pin.

3. **Assign the footprints and generate the BOM.** Assign a footprint to each symbol, using the library footprints for the through-hole resistor, the TO-220 package for the IRLZ44N, the DO-41 package for the diode and a two-pin screw terminal for the motor. Generate a bill of materials with reference, value, footprint and manufacturer part number. Save it as a CSV.

4. **Read the TB6612FNG datasheet.** Open the datasheet and find the absolute maximum ratings table, the recommended operating conditions and the pin functions table. Copy the VM, VCC and output current ratings into a table with three columns: parameter, absolute maximum and recommended. Note where the two differ and what that means for your choice of battery.

5. **Read the IRLZ44N datasheet.** Find the drain-source voltage, the continuous drain current at 25 degrees Celsius, the gate threshold voltage and the on-resistance at a gate voltage of 5 V. Compare them with the calculation from Day 13, and write one sentence on whether the transistor's margin is adequate.

6. **Analyse the HC-SR04 timing.** Read the HC-SR04 datasheet timing diagram and its table of timing parameters. Write the trigger pulse width, the echo width formula, and the recommended measurement cycle. Calculate the echo width for 0.5 m, 2 m and 4 m, and write the timeout value you would set in code for a 4 m maximum range.

7. **Check the net list against the schematic.** Export the netlist from KiCad. For five random nets, trace the connections on the drawing and confirm each one matches the netlist. Record any mismatch in the review notes.

Stop if a datasheet's values contradict the schematic. Do not build a circuit whose values you have not checked against the datasheet.

---

## Project: a schematic and datasheet pack for the motor switching stage

This project documents the Day 13 switching stage as a reviewed design. A reviewer should be able to read the pack, check the ratings and rebuild the circuit without asking you a question. The project is light on fabrication and heavy on documentation, which is why it is a documentation project.

**Goal:** produce a complete, reviewed schematic and a datasheet review for the motor switching stage, with a BOM, an ERC report, a timing and rating table, and a list of the margins that the design relies on.

**Repo layout:**

```
100-days-robotics-engineering-day-14-schematics-and-datasheets/
├── README.md
├── kicad/
│   ├── motor_switch_stage.kicad_pro
│   ├── motor_switch_stage.kicad_sch
│   └── motor_switch_stage.kicad_pcb
├── bom/
│   └── bom.csv
├── review/
│   ├── erc_report.md
│   ├── ratings_table.md
│   └── timing_analysis.md
├── datasheets/
│   └── README.md
└── images/
    └── schematic_export.png
```

The `datasheets/` folder holds a README with the links or titles of the PDFs you used. Do not commit the datasheet PDFs themselves unless their licence permits it. The README should name each document and the page you used.

**Starter review table:** `review/ratings_table.md`

```markdown
| Part | Parameter | Absolute max | Recommended | Design value | Margin |
|---|---|---|---|---|---|
| IRLZ44N | Drain-source voltage | 55 V | (fill in) | 6.3 V (clamped spike) | (fill in) |
| 1N5819 | Reverse voltage | 40 V | (fill in) | 6 V | (fill in) |
| ATmega328P | I/O pin current | 40 mA | 20 mA (design guideline) | 13.6 mA | (fill in) |
```

Fill the blank cells from the datasheets, and compute the margin as the absolute maximum divided by the design value, or the recommended maximum divided by the design value, and say which one you used.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the scope of the review and the list of datasheets used.
2. Draw the switching stage in KiCad, as in the exercises, and save the project files in `kicad/`.
3. Run the ERC and save the report in `review/erc_report.md`. Write the reason for each remaining warning.
4. Generate the BOM and save it as `bom/bom.csv`. Check each manufacturer part number against its distributor page.
5. Complete `review/ratings_table.md` with the datasheet values and the margins.
6. Write `review/timing_analysis.md` with the HC-SR04 timing calculations for 0.5 m, 2 m and 4 m.
7. Export a PNG of the schematic into `images/` and add it to the README.

**README checklist:**

- [ ] Goal and scope of the review
- [ ] Datasheet list with the page numbers used
- [ ] Schematic export image
- [ ] BOM link
- [ ] ERC report with every warning explained
- [ ] Ratings table with margins
- [ ] Timing analysis with the timeout value

**Done when:**

- The ERC report has no unexplained warnings.
- Every rating in the ratings table has a page reference in the datasheet.
- The BOM lists a manufacturer part number and a footprint for every symbol.
- A reader who has not seen the design can find the flyback diode, the gate resistor and the motor connector from the schematic alone.

**Evidence to commit to GitHub:** the KiCad files, the BOM, the ERC report, the ratings table, the timing analysis, the datasheet README and the schematic image. A reviewer should be able to open the project in KiCad and reproduce the ERC report.

This project carries no hazard beyond the reading, but it defines the stage that Day 15 will use with a real motor and sensor. Mark any value you could not verify as unverified, and do not build a board from an unverified value.

---

## Check your understanding

1. A schematic shows two resistors, both labelled R3, on the same sheet. Explain why this is a problem, and name the check that catches it.
2. A net is labelled +3V3 on one sheet and +3.3V on another, and the two are not connected by a power port. Explain the risk, and describe how you would confirm the connection on the board.
3. A part has an absolute maximum supply voltage of 5.5 V and a recommended range of 2.7 V to 5.5 V. You plan to run it from a 5 V regulator with 0.4 V of ripple. Decide whether the design is acceptable, and explain your reasoning with the margin figure.
4. An ATmega328P pin drives an LED through a 330 Ω resistor from a 5 V supply, with a 2 V forward drop. Calculate the current, the power in the resistor, and the pin's margin against the 40 mA absolute maximum.
5. An HC-SR04 is measured at an echo width of 11.7 ms. Calculate the distance, and explain whether a timeout of 10 ms in the code would correctly reject this reading.
6. A datasheet shows a timing diagram with a typical 10 µs pulse and a table that gives a minimum of 10 µs. Explain why the table value is the one the code must meet, and describe the measurement you would make to confirm the margin on your own board.

---

### Answers

1. Two parts with the same designator make the BOM and the build instructions ambiguous, because a builder cannot tell which resistor is meant. The ERC flags duplicate reference designators, and renumbering the parts in the schematic editor fixes it.
2. The two names describe the same voltage, but the tools treat them as different nets unless they are joined. The risk is that a board built from the schematic has an unconnected rail, or that a net named +3.3V is not connected to the regulator. Confirm by checking the continuity of the rail on the board with a multimeter in continuity mode, and by checking the voltage at each load with the supply on.
3. The 5 V supply with 0.4 V ripple peaks at 5.2 V. That is below the 5.5 V absolute maximum, but it is inside the recommended range with a margin of 5.5 / 5.2 = 1.06, or about 6 percent, from the maximum. That margin is thin. The design is marginal and should use a regulator with a lower output tolerance, or filter the ripple, before it is accepted.
4. The current is I = (5 − 2) / 330 = 3 / 330 = 9.1 mA. The power in the resistor is I² × R = 0.0091² × 330 = 0.027 W, which a 1/8 W resistor handles. The pin's margin against the 40 mA absolute maximum is 40 / 9.1 = 4.4 times.
5. The distance is d = v × t / 2 = 343 × 0.0117 / 2 = 2.01 m. A timeout of 10 ms corresponds to a maximum distance of 343 × 0.010 / 2 = 1.72 m, so a 2.01 m reading would be rejected as no echo. The timeout must be set to at least 23.3 ms to cover the 4 m maximum range.
6. The table value is the guaranteed limit under the stated conditions, so the code must meet it with margin. The typical diagram shows what the chip usually does. The measurement is to generate the trigger pulse on your board, measure its width with a scope or a logic analyser, and confirm it is at or above the table's minimum with the wiring and supply you will use.

---

## Sources to read and watch

*Read*

- [Wikipedia: Circuit diagram](https://en.wikipedia.org/wiki/Circuit_diagram) for the purpose and conventions of schematics.
- [Wikipedia: Datasheet](https://en.wikipedia.org/wiki/Datasheet) for what a datasheet contains and how it is used.
- [Wikipedia: Timing diagram](https://en.wikipedia.org/wiki/Timing_diagram) for the general notation.
- [Wikipedia: Electronic design automation](https://en.wikipedia.org/wiki/Electronic_design_automation) for the software that draws and checks schematics.
- [Wikipedia: Bill of materials](https://en.wikipedia.org/wiki/Bill_of_materials) for the BOM's role in manufacturing.
- The TB6612FNG datasheet from Toshiba, the IRLZ44N datasheet and the ATmega328P datasheet from Microchip, which you use in the exercises.
- The KiCad documentation at docs.kicad.org, for the schematic editor, the ERC and the BOM export.

*Tutorials*

- The KiCad getting started guide, on docs.kicad.org, for the first project.
- SparkFun Learn, the tutorials on reading a schematic and on reading a datasheet. Search "SparkFun Learn reading a schematic".
- Adafruit Learning System, the guides on reading a datasheet and on using KiCad. Search "Adafruit reading a datasheet".

*Watch*

- MIT OpenCourseWare, 6.002 Circuits and Electronics. The lectures on circuit analysis give the context for reading a schematic.
- A current KiCad schematic tutorial. Search YouTube for "KiCad 9 schematic tutorial" and choose a video that matches the KiCad 9 interface, because the menus changed between versions.

*Practice*

- KiCad 9 itself. Draw the switching stage and run the ERC, which gives immediate feedback on every connection.

---

## Tomorrow

Today you read and drew the documents that describe a circuit. Tomorrow you build a complete small system from them. Day 15 is the project day: a sensor, a controller and an actuator, wired on a breadboard from a schematic you draw first, tested with the ratings and timing you read today, and documented on GitHub. The HC-SR04 timing and the servo's supply limits from this lesson are the first numbers you will check on your own hardware.
