# Day 3: Robotics Engineering Workflow

*100 Days of Robotics Engineering, Module 1: Robotics Engineering Foundations*

**Time:** about 4 hours
**You will build:** a one-page requirements and test plan for a robot of your choice, with the margin math, a traceability table and a first engineering notebook entry.
**You should know:** Days 1 and 2 (the sense, decide, act loop and the layers of a mobile robot). Basic use of a terminal helps but is not required.

---

## 1. The big idea

Beginners start a robot by buying parts. Engineers start by writing down the problem. The difference shows up three weeks later, when the beginner's motors are too weak, the battery is too small, and nobody remembers why the sensor is mounted upside down.

A workflow is the order in which you make decisions, so that early choices constrain later ones on purpose. Teams hiring robotics engineers want to see that you can turn a vague request into numbers, test against those numbers, and leave a record someone else can follow. Today you learn that order and the documents that carry it.

## 2. The workflow in order

The chain for today is:

```
problem → requirements → specifications → architecture
        → mechanical design → electronics → firmware → software
        → simulation → prototype → integration → testing
        → iteration → documentation
```

Read as a straight line, this is the waterfall model. Winston Royce described the stages in a 1970 paper on large software projects, and Royce himself warned that a single pass through them was risky. Real projects loop. Barry Boehm's spiral model (1986 to 1988) repeats the cycle and attacks the biggest risk on each pass.

The V-model makes the loops explicit. Each stage on the left has a test on the right:

```
 Problem / user need ────────────────────────────────▶ Validation: does it solve the problem?
    Requirements ─────────────────────────────────▶ System tests: every "shall" checked
       Specifications + architecture ──────────▶ Integration tests: subsystems together
          Mech / electronics / firmware / sw ─▶ Unit and bench tests: each part alone
                         \                    /
                          \__ build + simulate __/
```

Two rules make this work:

1. **Decide in order, revise in any order.** You write requirements before picking motors. If testing shows the motors are too weak, you go back, change the spec or the part, and record why.
2. **Every stage produces a document.** The document lets the next stage, or the next person, trust the decision.

Regulated fields make these rules formal. Automotive robots follow ISO 26262, and medical device software follows IEC 62304. Startups run lighter versions, but interviewers still ask, "How did you verify that?"

## 3. Problem, requirements and specifications

**Intuition.** A customer says, "I want a robot that brings coffee." That is a wish. A requirement is "The robot shall carry a 500 g mug." A specification is "Payload 0.5 kg, tilt under 5 degrees, maximum acceleration 0.5 m/s²." Each step makes the wish harder to misread.

- **Problem statement:** who has the problem, what it is, and why it matters, in one or two sentences with no solution in them.
- **Requirement:** a "The system shall..." statement. Good requirements are specific, measurable, achievable, relevant and testable. Usually they use "shall."
- **Specification (spec):** the numbers, units, tolerances and test conditions behind a requirement.
- **Constraint:** a limit you did not choose, such as budget, size or deadline.

**Worked example: the desk courier.** You will carry this robot through Day 5. In a small office, people walk across the room to pass light items (keys, a phone, a pen box) between two desks about 5 m apart.

| ID | Requirement | Specification | How to verify |
|---|---|---|---|
| R1 | The robot shall carry a light item. | Payload 0.5 kg; total robot mass ≤ 2.0 kg | Load 0.5 kg, drive 5 m, no tipping |
| R2 | The robot shall make the trip quickly. | 5 m in ≤ 30 s, so cruise speed ≥ 0.25 m/s | Stopwatch, 5 runs, report mean and worst |
| R3 | The robot shall not hit obstacles. | Detect objects ≥ 0.30 m ahead; stop before contact | Place a box in the path at 10 speeds |
| R4 | The robot shall run a working day of light use. | ≥ 1 h of driving per charge | Timed drive test from full charge |
| R5 | The robot shall be controlled from a phone. | Wi-Fi, command latency ≤ 200 ms | Log command and response timestamps |
| R6 | The robot shall be affordable. | Parts ≤ $250 USD | BOM total |

Check that the numbers are sensible. 5 m in 30 s is only 0.17 m/s on average, but the robot must accelerate, turn and stop, so 0.25 m/s leaves margin. For R3, stopping distance at speed v with deceleration a is v² / (2a). At 0.25 m/s and 0.5 m/s², that is 0.0625 m, about 6 cm. Add 100 ms of reaction time (0.25 × 0.1 = 2.5 cm) and you need about 9 cm. Detecting at 30 cm gives more than three times that.

**Common mistakes.**
- Writing a solution as a requirement: "The robot shall use a Raspberry Pi." That belongs in the architecture, unless the customer truly demands it.
- Untestable words such as "fast," "robust" or "user-friendly." Replace each with a number and a test.
- No margin. If the spec equals exactly what the requirement needs, any real-world loss makes you fail.

**Limitation.** Requirements are guesses about the future. Users change their minds once they see a prototype. Keep the document under version control and expect to revise it.

## 4. Architecture and the four parallel stages

**Intuition.** The architecture is the floor plan before the furniture. It says "there is a kitchen here, and it connects to the dining room." It does not say which fridge to buy.

The architecture names the major blocks (compute, sensing, actuation, power, communication, mechanics and software layers), the interfaces between them, and the reason for each split. Day 2 gave you the typical blocks of a mobile robot. Each block should trace back to a requirement.

**Worked example.** For the desk courier, R3 (avoid obstacles) creates a forward distance-sensing block. R5 (phone control) creates a Wi-Fi block. R1 and R2 together create a drive block with enough torque for 2 kg at 0.25 m/s. If you cannot point to a requirement for a block, either a requirement is missing or the block is not needed.

After the architecture is fixed, four stages run partly in parallel. Each has its own tools and outputs:

| Stage | Main output | Common tools (approximate cost) |
|---|---|---|
| Mechanical design | CAD model, drawings, STL files for printing | FreeCAD (free, open source); Onshape (free plan, documents are public); Autodesk Fusion (free personal-use license with limits); SolidWorks (paid) |
| Electronics | Schematic, wiring diagram, PCB layout | KiCad (free, open source); Fritzing (small payment for the download); diagrams.net (free) |
| Firmware | Code for the MCU | Arduino IDE 2 (free); PlatformIO in VS Code (free core); Pico SDK; ESP-IDF |
| Software | Code for the SBC, usually ROS 2 | ROS 2 Jazzy on Ubuntu 24.04 (free); Python; C++; VS Code |

**Intuition.** Four trades build one house: the framer, the electrician, the plumber and the interior designer. They work at the same time, but they must agree on where the walls are before anyone drills a hole.

The agreement is written down. Before firmware and software are split, both sides agree on the interface:

```
MCU ⇄ SBC link: USB serial, 115200 baud
SBC → MCU:  "V <left_mps> <right_mps>\n"     e.g. "V 0.25 0.25\n", sent at 20 Hz
MCU → SBC:  "E <left_ticks> <right_ticks> <battery_mV>\n", sent at 50 Hz
Timeout: if no "V" line arrives for 500 ms, MCU stops both motors.
```

Firmware and software can then be written and tested separately. When something fails later, you check each side against the agreement.

**Common mistakes.** Designing the frame before the electronics are known, then finding that the battery does not fit. Agree on a volume budget early, using rough boxes in CAD for the battery, boards and motors.

**Limitation.** Parallel work saves time only while the interfaces stay stable. Changing an interface late breaks both sides at once.

## 5. Simulation, prototypes and testing

**Simulation.** A simulator models the robot's body, sensors and environment, so software can be tested before the hardware exists. Gazebo (free, open source, the usual pairing with ROS 2; the Harmonic release pairs with ROS 2 Jazzy) and Webots (free, open source under Apache 2.0) are the main robot simulators. Wokwi runs microcontroller circuits in a browser, with a free tier and paid plans. NVIDIA Isaac Sim is free to use but needs a strong NVIDIA GPU.

For the desk courier, you could simulate a differential-drive robot in Gazebo with a forward distance sensor, drive it at 0.25 m/s toward a wall, and check that it stops more than 6 cm away. The sim-to-real gap is the catch: simulated wheels do not slip on carpet, simulated sensors are cleaner than real ones, and simulated motors never overheat. Use simulation to find logic bugs, then confirm on hardware.

**Prototype.** A prototype is a quick build made to reduce the biggest risk first. A motor taped to a board with a weight answers "Can this motor move 2 kg?"

**Integration.** Connect subsystems one at a time. Power first, then the MCU, then one motor, then two, then sensors, then the SBC. Never all at once. "Big bang" integration, where everything is connected together, makes it impossible to tell which part caused the failure.

**Testing levels.** Unit tests check one function. Bench tests check one board on the desk. Integration tests check subsystems together. HIL (hardware-in-the-loop) tests run a real controller against a simulated world. Field tests run in the real environment.

**Iteration.** Each test result feeds back into the spec, the design or the part choice. A test log records conditions and outcomes:

| Test | Spec | Setup | Result | Pass? | Action |
|---|---|---|---|---|---|
| T2.1 | ≥ 0.25 m/s with 0.5 kg | Flat tile, 5 m, 5 runs | Mean 0.21 m/s, worst 0.19 | No | Raise motor voltage or change gear ratio; retest |

A failed test is useful data, as long as it is written down with its conditions.

**Common mistakes.**
- Testing only the happy path. Test low battery, lost Wi-Fi and a blocked wheel too.
- Changing two things between tests, so you cannot tell which change helped.

## 6. Drawings: block, wiring and schematic

**Intuition.** A city has three maps: a subway map (which stations connect), a street map (where the roads physically run) and an engineering plan (exact pipes and valve sizes). Those are the block diagram, the wiring diagram and the schematic.

All three show the same connection: an MCU driving one motor through a driver.

**Block diagram** (what talks to what):

```
[ MCU ] ──PWM, DIR──▶ [ Motor driver ] ──power──▶ [ Motor ]
```

**Wiring diagram** (physical wires, connectors and colors):

```
MCU header J1               Driver board J2
 pin GP2  ── yellow ──────▶  PWMA
 pin GP3  ── green  ──────▶  AIN1
 pin GP4  ── blue   ──────▶  AIN2
 GND      ── black  ──────▶  GND
                             J3: AO1 ── red   (22 AWG) ──▶ Motor +
                                 AO2 ── black (22 AWG) ──▶ Motor −
```

**Schematic** (electrical symbols and values): drawn in KiCad, with the MCU symbol, the driver IC symbol, decoupling capacitors next to the driver's power pins, and net labels such as `PWM_A` and `VMOTOR`. A schematic shows the parts and values that the other two diagrams hide.

**Common mistakes.** Block diagrams cluttered with pin numbers are hard to read. Wiring diagrams without wire colors or connector names are useless when you are debugging with a multimeter (Day 4).

**Limitation.** Diagrams drift from reality as you change things on the bench. Update the drawing in the same commit as the change.

## 7. Records: notebook, BOM and version control

**Engineering notebook.** A dated, append-only record of decisions, experiments and results. Your notebook is a letter to yourself in three months, when you have forgotten everything. A Markdown file per day works well.

```markdown
## 2026-10-24: Motor sizing for the desk courier

Goal: check whether 50:1 gearmotors can reach 0.25 m/s with 2 kg.
Hypothesis: yes, at 7.4 V, with about 30% speed margin.
Did: calculated wheel speed from the datasheet no-load rpm, derated by 25% for load.
Result: about 0.30 m/s expected. Margin is thinner than hoped.
Decision: keep 50:1, but note a 30:1 option in the BOM as a fallback.
Next: bench test one motor with a 1 kg load on day X.
```

Write entries as you go, including what failed. Keep them short and in plain text, so search and `git log` work on them.

**Bill of materials (BOM).** The list of every part, with a reference, quantity, description, manufacturer, manufacturer part number (MPN), supplier, unit cost and extended cost (quantity × unit cost). CSV works well: GitHub renders it as a table, and any spreadsheet opens it.

```csv
ref,qty,description,manufacturer,mpn,supplier,unit_usd,ext_usd,notes
U1,1,Microcontroller board,Raspberry Pi,Pico W,any authorized reseller,6.00,6.00,Wi-Fi for R5
U2,1,Dual motor driver,Toshiba (chip),TB6612FNG breakout,SparkFun or Adafruit,8.00,8.00,check stall current
M1-M2,2,Gearmotor with encoder,TBD,TBD,TBD,15.00,30.00,verify torque in Day 5
HW1,1,M3 screw and standoff kit,generic,,any,8.00,8.00,easy to forget
```

The prices are placeholders for the format. Day 5 replaces them with checked ranges. Most BOMs forget wires, connectors, screws, heat-shrink, shipping and spares. A common rule of thumb adds 10 to 20 percent for these and for parts you break.

**Version control.** Git tracks changes to files, and GitHub hosts the repository online. Every commit says what changed and why, and you can always go back. A robotics repository mixes text (code, Markdown, CSV) with binaries (CAD, images). Git shows line-by-line differences for text, but for binaries it only stores whole versions. Large binaries can go in Git LFS (Large File Storage), and CAD should also be exported to a neutral format such as STEP.

```bash
git init
git add README.md requirements.md bom.csv
git commit -m "Add desk courier requirements R1-R6 and BOM template"
git tag v0.1-requirements
git log --oneline
```

Keep build output out of the repo with a `.gitignore`:

```
build/
install/
log/
.pio/
__pycache__/
*.bak
```

The first three are folders created by `colcon`, the ROS 2 build tool. `.pio/` is PlatformIO's build folder.

**Common mistakes.**
- Commit messages such as "update" or "fix." Say what changed and why.
- Committing build output, which bloats the repo and causes conflicts.
- Never tagging, so you cannot say which code version ran on the robot during a test.
- Committing Wi-Fi keys or passwords in firmware. Keep them in a file listed in `.gitignore`.

**Limitation.** Git cannot merge two people's edits to the same binary CAD file. Teams assign ownership of each CAD file, or use a CAD tool with built-in versioning such as Onshape.

## 8. Hands-on exercises (about 60 minutes)

1. **Requirements rewrite (15 min).** Rewrite each vague statement as a testable "shall" plus a spec: "The robot should be fast." "The battery should last a long time." "It should be quiet." "It should be safe around people."
2. **Stopping distance (10 min).** For speeds of 0.1, 0.25 and 0.5 m/s, with 0.5 m/s² deceleration and 100 ms reaction time, compute the stopping distance. Check: at 0.5 m/s you get 0.25 m + 0.05 m = 0.30 m, exactly the R3 detection distance. What does that tell you about raising the speed?
3. **Three drawings (20 min).** For a Raspberry Pi Pico reading one HC-SR04 ultrasonic sensor (VCC, TRIG, ECHO, GND), draw a block diagram, a wiring diagram with colors, and a rough hand-drawn schematic. The HC-SR04's echo pin outputs 5 V when powered from 5 V, and the Pico is a 3.3 V device. Write in your notebook how you would handle that: a voltage divider, or a 3.3 V-compatible sensor variant. You will size the divider in Module 2.
4. **Git practice (15 min).** Complete the first levels of Learn Git Branching, or create a local repo, make three commits and one tag, and run `git log --oneline`.

## 9. Practical check: a one-page requirements plan

**Why this is a written check and not a repo:** today's skill is process. A single page that shows the process working is enough evidence. You will apply the same method to your full design on Day 5.

**Task.** On one page (a Markdown file, a document or paper), cover the following for the desk courier or a robot of your own:

1. A two-sentence problem statement with no solution in it.
2. At least six requirements, each with a number, a unit and a verification method. Show the margin math for at least two.
3. A traceability table in this form, with no empty cells:

   | Req | Design block | Test | Status |
   |---|---|---|---|
   | R3 detect ≥ 0.30 m | Forward ToF sensor | T3.1 box-in-path | not run |

4. A block diagram in Mermaid or on paper, and a wiring diagram for one subsystem.
5. A BOM with at least five rows. Placeholders are allowed if they are marked "TBD."
6. One dated notebook entry.

**Done when.** Every requirement has a number, a unit and a test. A stranger could read the page and know what the robot must do and how you will prove it. If you commit it, the page is the file and the commit message says what it contains.

## 10. Check your understanding

1. Why is "The robot shall use an ESP32" usually a poor requirement?
2. Your robot passes every unit test but fails when assembled. Which workflow stage was weak, and what would you change?
3. Give one bug a simulator would catch and one it would likely miss.
4. Why write the MCU ⇄ SBC interface agreement before writing either side's code?
5. A teammate updates the wiring on the robot but not the diagram. What goes wrong a month later?
6. You test a speed change and a new battery at the same time, and speed improves. What can you conclude, and what should you have done?
7. Why is a CSV BOM in Git better than a spreadsheet emailed around?

**Answers.**

1. It names a solution, not a need. Unless the customer demands that chip, the real need (for example "phone control over Wi-Fi") should be the requirement, leaving the design free to choose.
2. Integration. Bring subsystems together one at a time, with integration tests at each step, instead of all at once.
3. Caught: a logic bug, such as the robot turning the wrong way when an obstacle appears. Likely missed: wheel slip on carpet, sensor noise from sunlight, or a motor overheating.
4. It lets both sides be built and tested independently against the same contract, and it gives you a reference to check when integration fails.
5. Someone debugs from the diagram, probes the wrong pin, and either wastes hours or damages a part.
6. Nothing, about which change caused it. Change one variable per test.
7. Every change is recorded with who, when and why. Older versions can be restored, and the diff shows exactly which rows changed.

## 11. Sources

**Read:**
- [Wikipedia: Systems engineering](https://en.wikipedia.org/wiki/Systems_engineering)
- [Wikipedia: V-model](https://en.wikipedia.org/wiki/V-model)
- [Wikipedia: Requirements engineering](https://en.wikipedia.org/wiki/Requirements_engineering)
- [Wikipedia: Engineering design process](https://en.wikipedia.org/wiki/Engineering_design_process)
- [Wikipedia: Bill of materials](https://en.wikipedia.org/wiki/Bill_of_materials)
- [Wikipedia: Schematic](https://en.wikipedia.org/wiki/Schematic) and [Wiring diagram](https://en.wikipedia.org/wiki/Wiring_diagram)
- [Wikipedia: Hardware-in-the-loop simulation](https://en.wikipedia.org/wiki/Hardware-in-the-loop_simulation)
- [Wikipedia: Version control](https://en.wikipedia.org/wiki/Version_control)
- NASA Systems Engineering Handbook (NASA/SP-2016-6105 Rev2): search the title on nasa.gov
- [Pro Git book](https://git-scm.com/book/en/v2), chapters 1 and 2

**Tutorials:**
- GeeksforGeeks: search "Software Engineering V-Model" and "Requirements Engineering Process"
- SparkFun Learn: "How to Read a Schematic" (search the title on learn.sparkfun.com)
- [KiCad documentation](https://docs.kicad.org/): the "Getting Started in KiCad" guide
- [Mermaid flowchart syntax](https://mermaid.js.org/syntax/flowchart.html)
- [GitHub Docs](https://docs.github.com/): search "About READMEs" and "Working with non-code files"

**Watch:**
- MIT OpenCourseWare: search ocw.mit.edu for "Fundamentals of Systems Engineering" (course 16.842) and watch the first lecture
- [Articulated Robotics](https://www.youtube.com/@ArticulatedRobotics): any video on planning a robot build, for a real example of requirements turning into parts
- [GitHub's YouTube channel](https://www.youtube.com/@GitHub): search "Git and GitHub for beginners"

**Practice:**
- [Learn Git Branching](https://learngitbranching.js.org/): interactive Git exercises in the browser

**Note on links:** open each one once. If a page has moved, search its title.

## 12. Tomorrow

Day 4, **Measurement and Engineering Tools**, moves from the desk to the bench. Today you wrote specifications and test plans. Tomorrow you learn the instruments that produce the numbers in those plans: multimeters, bench power supplies, oscilloscopes, logic analyzers, calipers, breadboards and soldering tools. You will also learn to read a datasheet and debug in a fixed order. Your test log from today gets its first real column of measurements.
