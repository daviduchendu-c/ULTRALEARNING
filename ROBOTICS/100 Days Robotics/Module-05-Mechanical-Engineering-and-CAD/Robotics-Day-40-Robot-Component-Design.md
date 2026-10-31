# Day 40: Robot Component Design

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 4 hours
**Level:** intermediate. You need the load and stiffness calculations from Day 34, the fits from Day 39, and the CAD features from Day 37 and Day 38. Today you design parts for a purpose: a mount that holds a sensor, a servo arm that transmits torque, a chassis plate that carries the electronics, and an enclosure that protects them.

**By the end of this day, you will have:**

- A written requirements table for four robot components: a motor bracket, a sensor mount, a servo arm adaptor and an electronics enclosure
- A hand calculation of the load, stiffness and factor of safety for the sensor mount and the servo arm
- A parametric enclosure written as a CadQuery script, with a base, a lid, mounting holes and a test that checks its outside dimensions
- A material choice for each component, with the reason, and a note on how the part will be made

**What you need:**

- Python 3.12 with CadQuery 2.4 or newer, installed through conda-forge or pip (`pip install cadquery`, if your Python version is supported by the release you choose), and pytest 8 or newer
- FreeCAD 1.0.x if you prefer a graphical tool for the bracket and the sensor mount. FreeCAD is free and open source.
- A digital caliper, reading to 0.01 mm
- A hobby servo, such as a standard 9 g or 20 kg servo, with its horn and mounting screws. A servo costs about $3 to $25 depending on its torque.
- A printed test piece in PLA or PETG, or access to a laser cutter for 3 mm acrylic or plywood
- A spring scale or a set of masses, to test a load on a sensor mount, about $10
- Optional: a Raspberry Pi or Arduino board and a sensor, such as an ultrasonic distance sensor, to fit the mount and the enclosure to real parts

**Prerequisites:**

- Day 34: stress, deflection of a cantilever, and the factor of safety
- Day 35: torque at a joint and the stall torque of a motor
- Day 37 and Day 38: sketches, features, parameters and holes
- Day 39: clearance and fit, and the bill of materials for a unit

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How robot component design developed](#how-robot-component-design-developed)
- [Glossary](#glossary)
- [Requirements and interfaces](#requirements-and-interfaces)
- [Motor brackets](#motor-brackets)
- [Sensor mounts](#sensor-mounts)
- [Servo mechanisms](#servo-mechanisms)
- [Chassis parts and wheel hubs](#chassis-parts-and-wheel-hubs)
- [Enclosures](#enclosures)
- [Choosing a material and a process](#choosing-a-material-and-a-process)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a parametric electronics enclosure](#project-a-parametric-electronics-enclosure)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot component is a part with a job. A motor bracket holds a motor against torque. A sensor mount holds a camera or a distance sensor at a fixed angle, and any movement of the mount becomes an error in the reading. A servo arm transfers torque from a small splined shaft to a linkage. An enclosure keeps dust, cables and careless fingers away from the electronics. Each part has loads, an interface with its neighbours, and a process by which it will be made. Those three things decide the design.

Today you learn the design method that connects the calculations from Module 5 to real components. You start from a requirements table, choose the interfaces and the fasteners, check the load and the stiffness, and then choose the material and the process. You then build an enclosure in code, so that its dimensions are set by a few parameters and checked by a test. The method is the one a robotics team uses on every part, and the enclosure is the first complete component you will print or cut.

---

## How robot component design developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1900s | Standard components such as bearings, bolts and motor frames are catalogued, so designers can choose parts from a table. | Robot components are chosen from catalogues of servos, motors and sensors. |
| 1950s to 1970s | Industrial robot arms, such as the first hydraulic and electric manipulators, use cast brackets and machined housings. | Heavy cast housings set the expectations for stiffness that robot designers still use. |
| 1980s to 1990s | Hobby servos become cheap and standard, and the 9 g servo and the RC receiver become common in education. | Small servos are the first actuators that most robotics students use. |
| 2000s | Desktop 3D printers make it possible to design, print and test a robot component within one day. | Iteration on a part now takes hours, not weeks. |
| Today | Parametric scripts, such as CadQuery, let a designer generate a family of parts from a few numbers, and print them directly. | The enclosure you build today can be regenerated for a new board in seconds. |

The shift from catalogue parts to printed custom parts is why the design method now starts from requirements and a test.

---

## Glossary

- **Requirement:** a statement of what a component must do, with a number, such as "holds 2 N at the tip with less than 0.5 mm deflection".
- **Interface:** the place where two components meet, including the fasteners, the clearances and the mounting pattern.
- **Load case:** a set of forces and moments that a part must carry, such as the weight of a sensor plus a push from a user.
- **Stall torque:** the maximum torque a servo or motor can produce while held still, usually given in kg·cm or N·m.
- **Spline:** a set of ridges on a shaft that engage with matching grooves in a horn or hub, so torque transfers without slip.
- **Servo horn:** the arm attached to a servo output shaft, which carries the load into a linkage.
- **Enclosure:** a housing that protects electronics from dust, cables, impact and accidental contact.
- **Boss:** a raised cylinder inside a housing, which holds a screw or a standoff.
- **Standoff:** a spacer that lifts a circuit board off a surface and holds it with a screw.
- **Ingress protection (IP) code:** a two-digit rating from IEC 60529 that describes protection against solids and water, such as IP54.
- **Print orientation:** the direction in which a part is printed, which decides the strength of the printed layers.
- **Anisotropy:** the property of a material that is stronger in one direction than another, which applies to FDM printed parts.
- **Parametric script:** a program that builds a part from named inputs, so that a change to one input updates the whole part.

---

### Requirements and interfaces

A requirement is a number that a component must meet. Without one, a design has no finish line. A requirement is more useful when it is measured, such as "the sensor must not tilt more than 1 degree under a 2 N push", than when it is vague, such as "the mount must be strong".

The first principle is that each component has three interfaces: what it holds, what holds it, and what it must not touch. The mount for a distance sensor holds the sensor with two M2 screws, is held to the chassis by an M3 bolt, and must not block the sensor's field of view. Writing the three lines for each part gives the designer the list of dimensions to fix before drawing.

Here is a worked example of a requirements table for a sensor mount:

| Item | Requirement |
|---|---|
| Holds | One ultrasonic distance sensor, 45 mm by 20 mm, 10 g |
| Held by | Two M3 bolts into the chassis, 6 mm apart |
| Load | 2 N at the tip from an accidental push |
| Limit | Tip deflection under 0.5 mm |
| Clearance | No material in front of the transducer within a 30 degree cone |
| Material | PETG, printed with the layers across the load |

The common mistake is to write the requirements after the design, so that the design defines the requirement. Write the table first, and test the design against it. Change the table only with a reason you can state.

Requirements and interfaces are the first step in every component in this lesson, and they decide the rest of the work.

---

### Motor brackets

A motor bracket was designed on Day 34 and Day 37. Its job is to hold the motor against the torque it produces. The torque is reacted at the bolts, which must carry the shear, and the bracket must carry the bending moment from the motor's weight and the reaction.

The first principle is that the bracket resists two loads: the reaction torque, which twists the bracket, and the weight, which bends it. A motor of 0.4 N·m stall torque, held by a bracket with two bolts 16 mm apart, puts a shear force of about 0.4 / 0.016 = 25 N on each bolt if the bolts carry the torque directly. That is a large load for a 3 mm bolt in plastic, and it is why a motor bracket is usually metal or thick plastic with a bolt pattern that carries the torque in shear through the bracket face.

Here is a worked example. A 28 mm square motor with a 6 mm shaft, weighing 35 g, is held by a PETG bracket 3 mm thick. The weight is 0.035 × 9.81 = 0.34 N, which is small. The torque is the larger load, so the designer checks the bolt shear at the shaft and the fatigue of the bracket around the bolt holes. A stress concentration factor of 2 or 3 at the holes, from Day 38, must be included in the check for any cyclic load.

The common mistake is to design the bracket for the weight and forget the torque. The bracket bends a little, and the motor rotates in its seat, which loosens the bolts over time. Include the stall torque in the load case.

Motor brackets are some of the most common parts a robot team designs, and their interfaces are the most likely source of misalignment.

---

### Sensor mounts

A sensor mount holds a sensor at a fixed position and angle. The reading of the sensor depends on that position, so any movement of the mount becomes an error. A camera that shifts by 1 degree points its image 1 degree off, and a distance sensor that tilts by 2 degrees measures a different point on a wall.

The first principle is stiffness. A cantilever mount deflects under a load, and the deflection moves the sensor. For a cantilever of length L, width b, thickness h and modulus E, the deflection under a tip load F is δ = F L³ / (3 E I), with I = b h³ / 12. The angle the sensor turns is the deflection divided by the lever arm from the root to the sensor, for small angles.

Here is a worked example. A PLA sensor mount is 30 mm long, 10 mm wide and 3 mm thick, with a sensor at the tip. A push of 2 N acts at the tip. Take E as about 3.5 GPa, a typical value for PLA, which varies with the print settings. I = 10 × 3³ / 12 = 22.5 mm⁴ = 2.25 × 10⁻¹¹ m⁴. The tip deflection is δ = 2 × (0.03)³ / (3 × 3.5 × 10⁹ × 2.25 × 10⁻¹¹) = 0.23 mm. That meets the 0.5 mm limit from the requirements table. The sensor tilt is small, but the test still matters, because the modulus of printed PLA can be much lower than the value used here, depending on the print settings.

The common mistake is to design the mount from the datasheet modulus of a bulk material and ignore the layer orientation of a printed part. A part printed with its layers across the load can be much weaker than a part with its layers along the load. Print the mount with the layers along the bending direction, and test it with the force you expect.

Sensor mounts are used on every robot that needs to look at or measure something. Their stiffness and alignment set the accuracy of the sensor.

---

### Servo mechanisms

A servo is a motor with a position feedback loop, which holds an output shaft at a commanded angle. A small hobby servo has a splined output shaft that carries a horn, and the horn carries the load into a linkage. The servo's stall torque sets the limit of what it can hold, and the horn and the linkage must carry the same load without slipping or bending.

The first principle is torque transfer through the spline. A horn that is a loose fit on the spline slips under load, and the servo loses its position. A horn that is tight may crack the plastic of the spline. Most hobby servos use a metal horn with a set screw, and the spline fit is specified by the manufacturer.

Here is a worked example. A servo with a stall torque of 1.5 N·m holds a linkage arm of 50 mm, which carries a load of 20 N at the end. The torque at the servo is 20 × 0.05 = 1.0 N·m, which is 67 percent of the stall torque. The margin is 1.5 / 1.0 = 1.5, which is low for a design that must hold for many cycles. The designer either chooses a larger servo, shortens the arm to 35 mm and reduces the torque to 0.7 N·m, or reduces the load. The hand calculation takes two minutes, and it prevents a servo from stalling in the field.

The common mistake is to use the stall torque as the working torque. A servo held at stall for more than a few seconds overheats, and its gears wear quickly. Use a working torque of about half the stall torque for a continuous load, and check the servo's datasheet for the duty cycle.

Servo mechanisms are used in grippers, in pan and tilt heads, in small robot arms and in steering linkages. The servo arm adaptor you design today is the interface between a standard horn and a custom part.

---

### Chassis parts and wheel hubs

The chassis carries the drivetrain, the electronics and the load. Chassis parts are usually plates, rails and brackets that are cut from sheet material or printed. The wheel hubs you designed on Day 38 are chassis parts, and their interfaces are the bore and the bolt pattern.

The first principle for a chassis plate is that it must be stiff enough not to bend under the wheels and the battery. A plate of 3 mm aluminium carrying a 2 kg robot with a 10 cm span between supports deflects by a small amount, but a plate of 3 mm PLA can deflect enough to misalign the gears. Use ribs, a larger thickness or a stiffer material when the span is long.

Here is a worked example. A 150 mm by 100 mm chassis plate, 3 mm thick, is supported at its two short edges, and carries a battery of 0.5 kg at its centre. For a simply supported plate, a rough beam check treats the plate as a beam of width 100 mm over a span of 150 mm. The load is 0.5 × 9.81 = 4.9 N at the centre. In aluminium with E = 69 GPa, I = 100 × 3³ / 12 = 225 mm⁴ = 2.25 × 10⁻¹⁰ m⁴. The deflection at the centre of a simply supported beam is δ = F L³ / (48 E I). Here F L³ = 4.9 × (0.15)³ = 1.65 × 10⁻² N·m³, and 48 E I = 48 × 69 × 10⁹ × 2.25 × 10⁻¹⁰ = 745 N·m². So δ = 2.2 × 10⁻⁵ m, or 0.02 mm. The plate is stiff enough. In PLA, with E of about 3.5 GPa, the same calculation gives 48 E I = 37.8 N·m² and δ of about 0.44 mm. That is too much for a drivetrain with gears, so the designer adds ribs or changes the material.

The common mistake is to design a chassis from the static load alone. The robot also sees impact, acceleration and the torque from the drive motors. Use a factor of safety of 2 or more for the chassis, and check the bolt joints at the wheel mounts.

Chassis parts and wheel hubs are the largest parts in a robot, and their stiffness sets the alignment of every component mounted on them.

---

### Enclosures

An enclosure protects electronics from dust, cables, impact and contact with people. It also holds the circuit board in a known position, and it provides the mounting points for the sensors and the switches that must reach the outside.

The first principle is that the internal dimensions come from the parts inside. A printed circuit board of 50 mm by 40 mm needs room for the components, a clearance around the board for the standoffs and the heat, and walls that carry the lid screws. The outside dimensions are the inside plus twice the wall thickness. Work from the inside out.

Here is a worked example. A board of 50 mm by 40 mm is mounted on four standoffs. The designer adds a clearance of 1.5 mm on each side, which gives an inside width of 50 + 2 × 1.5 = 53 mm and an inside depth of 40 + 2 × 1.5 = 43 mm. A wall of 2 mm on each side gives an outside width of 53 + 4 = 57 mm and an outside depth of 43 + 4 = 47 mm. The floor is 2 mm thick, and the inside height is 20 mm, which leaves room for the standoffs and the connectors. The total height is 22 mm.

Enclosures also need ventilation and an ingress rating if they are used outdoors. An IP54 rating means protection against dust and splashing water, and it requires gaskets and sealed cable entries. Most indoor robot enclosures are not rated, and the designer should state that in the requirements table.

The common mistake is to forget the connectors that must reach the outside, such as a USB port or a power switch. Place every cutout on the wall before you close the design, and check that its position clears the standoffs and the board. The second mistake is to make the lid screws too close to the wall edge, which splits the plastic when the screw is tightened.

Enclosures are the last component most robot designers make, and they are often rushed. A parametric enclosure avoids the rush, because the dimensions update when the board changes.

---

## Choosing a material and a process

The material and the process decide the strength, the cost and the time of a component. A part printed in PLA takes hours and costs a few cents of plastic. A part cut from 3 mm aluminium takes a machine shop days and costs more, but it is stiff and durable. The choice is a trade between these.

The first principle is that the process decides the shape. FDM 3D printing makes parts layer by layer, which allows complex shapes, but the layers are weaker across their boundaries. Laser cutting makes flat parts from sheet, which is fast and cheap for brackets and chassis plates. CNC machining makes precise parts from solid stock, which is strong and stiff but costly for one-off parts. Injection moulding is for large runs and needs a mould.

Here is a short comparison for the parts in this lesson:

| Part | Material | Process | Reason |
|---|---|---|---|
| Motor bracket | PETG or 3 mm aluminium | FDM or laser cut and bend | Holds torque; aluminium is stiffer |
| Sensor mount | PETG with layers along the bend | FDM | Complex shape, low load |
| Servo arm adaptor | Aluminium or nylon | CNC or FDM with infill | Spline wear and torque |
| Chassis plate | 3 mm aluminium or acrylic | Laser cut | Flat, stiff, fast |
| Enclosure | PLA or ASA | FDM | Shape, cost, easy to change |

The common mistake is to choose PLA for every part because it is easy to print. PLA softens at about 60 degrees Celsius, so a part in a hot car or near a motor can deform. ASA or PETG handle heat better.

---

## Hands-on exercises

1. **Write a requirements table.** Choose one component from this lesson, and write its requirements table with at least five rows. Include the interfaces, the load case and the limit.

2. **Test a sensor mount.** Print or cut a 30 mm cantilever of your chosen material, 10 mm wide and 3 mm thick. Clamp it at one end and hang a 200 g mass from the tip. Measure the deflection with a caliper against a fixed reference. Compare the result with the value from the lesson, scaled to your load and modulus.

3. **Check a servo.** Measure the stall torque from the datasheet of your servo, and compute the working torque for a 50 mm arm with a 20 N load. Decide whether the servo is adequate, and explain the margin.

4. **Measure a board.** Measure a circuit board and its components with the caliper. Compute the inside and outside dimensions of an enclosure for it, following the method in this lesson.

5. **Print a fit test.** Print a small test piece with three holes of 3.2, 3.4 and 3.6 mm, and check which one takes an M3 bolt with a light fit. Record the result, and use it for the enclosure bolt holes.

---

## Project: a parametric electronics enclosure

This project builds a parametric enclosure for a small circuit board. It is written as a CadQuery script, so its dimensions come from a few named inputs, and it is tested by a script that checks the outside dimensions. It is a full mini-project, because the enclosure is a complete component you will print, fit and use on the robot.

**Goal:** a base and a lid for a board of 50 mm by 40 mm, with four mounting holes in the floor, a 1.5 mm clearance, a 2 mm wall and a 20 mm internal height. The script must generate both parts, export them as STEP and STL, and pass the outside-dimension test.

**Repository name:** `100-days-robotics-day-40-component-design`

**Repo layout:**

```
100-days-robotics-day-40-component-design/
├── README.md
├── requirements.txt
├── enclosure.py
├── export.py
├── tests/
│   └── test_enclosure.py
├── out/
│   ├── base.step
│   ├── base.stl
│   ├── lid.step
│   └── lid.stl
└── renders/
    └── base-iso.png
```

**Starter code.** Write `enclosure.py` with the functions that build the base and the lid. The dimensions come from a single function, so that the tests and the model use the same numbers.

```python
# enclosure.py
import cadquery as cq


def dimensions(pcb_w=50.0, pcb_d=40.0, clearance=1.5, wall=2.0, floor=2.0, inside_h=20.0):
    """Return the internal and external dimensions of the enclosure, in millimetres."""
    inner_w = pcb_w + 2 * clearance
    inner_d = pcb_d + 2 * clearance
    outer_w = inner_w + 2 * wall
    outer_d = inner_d + 2 * wall
    total_h = floor + inside_h
    return {
        "inner_w": inner_w,
        "inner_d": inner_d,
        "outer_w": outer_w,
        "outer_d": outer_d,
        "total_h": total_h,
        "floor": floor,
        "inside_h": inside_h,
        "wall": wall,
    }


def mount_points(d, inset=4.0):
    """Four mounting hole centres in the floor, in the local frame centred on the base."""
    x = d["outer_w"] / 2 - inset
    y = d["outer_d"] / 2 - inset
    return [(x, y), (-x, y), (x, -y), (-x, -y)]


def make_base(d, hole_d=3.4):
    base = cq.Workplane("XY").box(d["outer_w"], d["outer_d"], d["total_h"], centered=(True, True, False))
    cavity = (
        cq.Workplane("XY")
        .workplane(offset=d["floor"])
        .box(d["inner_w"], d["inner_d"], d["inside_h"] + 1.0, centered=(True, True, False))
    )
    base = base.cut(cavity)
    for x, y in mount_points(d):
        hole = cq.Workplane("XY").center(x, y).circle(hole_d / 2).extrude(d["total_h"])
        base = base.cut(hole)
    return base


def make_lid(d, lid_t=2.0, hole_d=3.4):
    lid = cq.Workplane("XY").box(d["outer_w"], d["outer_d"], lid_t, centered=(True, True, False))
    for x, y in mount_points(d):
        hole = cq.Workplane("XY").center(x, y).circle(hole_d / 2).extrude(lid_t)
        lid = lid.cut(hole)
    return lid
```

**Export script.** Write `export.py` to build both parts and write them as STEP and STL.

```python
# export.py
import os
import cadquery as cq
from enclosure import dimensions, make_base, make_lid

d = dimensions()
os.makedirs("out", exist_ok=True)
base = make_base(d)
lid = make_lid(d)
cq.exporters.export(base, "out/base.step")
cq.exporters.export(base, "out/base.stl")
cq.exporters.export(lid, "out/lid.step")
cq.exporters.export(lid, "out/lid.stl")
print("exported base and lid to out/")
```

**The test.** Write `tests/test_enclosure.py`. It checks the outside dimensions against the bounding box of the base, and it checks that the base is hollow.

```python
# tests/test_enclosure.py
import math
import sys
import os

sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

from enclosure import dimensions, make_base, make_lid


def test_base_outside_dimensions():
    d = dimensions()
    bb = make_base(d).val().BoundingBox()
    assert math.isclose(bb.xlen, d["outer_w"], abs_tol=0.01)
    assert math.isclose(bb.ylen, d["outer_d"], abs_tol=0.01)
    assert math.isclose(bb.zlen, d["total_h"], abs_tol=0.01)


def test_base_is_hollow():
    d = dimensions()
    solid_volume = d["outer_w"] * d["outer_d"] * d["total_h"]
    assert make_base(d).val().Volume() < solid_volume


def test_lid_is_thin_plate():
    d = dimensions()
    bb = make_lid(d).val().BoundingBox()
    assert math.isclose(bb.zlen, 2.0, abs_tol=0.01)
```

**Step-by-step tasks.**

1. Create the repository and the virtual environment. Install CadQuery and pytest. If CadQuery does not install on your Python version, use Python 3.10 or 3.11 through conda, and record the version in the README.
2. Write `enclosure.py`, `export.py` and the test file. Run `pytest`, and fix any failing dimension until all tests pass.
3. Run `python export.py`. Confirm that the `out/` folder contains the four files.
4. Open `out/base.stl` in a slicer, such as PrusaSlicer 2.8 or newer, and check that the base fits on the bed in the orientation you intend to print. Note the orientation in the README.
5. Measure the printed base with the caliper, and compare its outside width and depth with the values from the script. Record the difference.
6. Change `pcb_w` to 60 mm, regenerate the files, and confirm that the outside width becomes 67 mm. Change it back, and note in the README why the enclosure is parametric.
7. Render an isometric view of the base, using any viewer or the CadQuery display, and save it to `renders/base-iso.png`.
8. Write the README with the requirements table, the material and process choice, the dimension method, the test output and the measured difference.

**README checklist:**

- [ ] A requirements table with the interfaces, the load and the limits
- [ ] The dimension method, with a worked example of the inside and outside dimensions
- [ ] The test output from `pytest`
- [ ] The printed dimensions compared with the script values
- [ ] The material, process and print orientation, with the reason for each

**Done when:**

- `pytest` passes all three tests
- `python export.py` writes the four files into `out/`
- The printed base measures within 0.3 mm of the script values, or the README explains the difference
- Changing `pcb_w` updates the outside width without any other edit
- The README states the material and process you chose for each component in this lesson, with a reason

**What to commit to GitHub:** the source files, the tests, the exported STEP and STL files, the render, the printed-dimension table, and the README. Do not commit a cache folder or a local virtual environment. Add a `.gitignore` for these.

**Why this project is full and not light:** the enclosure is a complete component with tests, and the same parametric method extends to the mounts and the brackets in the rest of the robot.

---

## Check your understanding

1. A sensor mount is a 40 mm cantilever, 8 mm wide and 2 mm thick. A 1 N push acts at the tip. Explain how you would check whether the deflection meets a 0.5 mm limit, and which value of E you would use if the part is printed in PETG.

2. A servo with a stall torque of 2 N·m holds a 40 mm arm carrying a load of 30 N. The designer says the servo is adequate because 30 × 0.04 = 1.2 N·m is less than 2 N·m. Explain what is missing from the check.

3. An enclosure must hold a board of 60 mm by 30 mm with a 2 mm clearance on each side, and a 2 mm wall. Compute the outside width and depth.

4. Explain why a motor bracket that is designed only for the motor's weight will fail in service.

5. A team chooses PLA for an enclosure that sits near a motor controller. Explain the risk and propose a material.

6. A chassis plate is designed in aluminium and passes its static check. The team then makes it in PLA with the same thickness and finds it misaligns the gears. Explain why.

**Answers:**

1. Compute I = b h³ / 12 = 8 × 2³ / 12 = 5.33 mm⁴ = 5.33 × 10⁻¹² m⁴. The deflection is δ = F L³ / (3 E I) = 1 × (0.04)³ / (3 × E × 5.33 × 10⁻¹²). For PETG, use a modulus of about 2 GPa, which gives δ = 6.4 × 10⁻⁵ / (3.2 × 10⁻²) = 2.0 mm. That is far above the 0.5 mm limit, so the mount must be thicker, shorter or made of a stiffer material. Check the modulus against the filament datasheet, and print the test piece to confirm it.

2. The check ignores the working torque. A continuous load should use about half the stall torque, so the working limit is 1.0 N·m, and the 1.2 N·m load exceeds it. The servo will overheat or stall under continuous load. The designer must choose a larger servo, shorten the arm or reduce the load.

3. Inside width is 60 + 2 × 2 = 64 mm. Outside width is 64 + 2 × 2 = 68 mm. Inside depth is 30 + 2 × 2 = 34 mm, and outside depth is 34 + 4 = 38 mm.

4. The bracket also carries the reaction torque from the motor, which twists the bracket and shears the bolts. Under stall, the torque can be many times the weight-induced load. The bracket then loosens, rotates or cracks, and the bolts work loose over time.

5. PLA softens at about 60 degrees Celsius, so a part near a motor or in a hot enclosure can deform. Propose ASA or PETG, which handle heat better, and check the datasheet of the filament for the heat deflection temperature.

6. The static check used the stiffness of aluminium, which is about 69 GPa, while PLA has a modulus of a few gigapascals. The same thickness gives a far larger deflection in PLA, which moves the gears out of alignment. The fix is to add ribs, increase the thickness or use a stiffer material, and to recheck the deflection with the correct modulus.

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Servo motor", "Enclosure", "Ingress protection rating" (search for "IP code" if the title differs), "Standoff (electronics)", "Fused filament fabrication", "Anisotropy", "Heat deflection temperature" and "Polylactic acid"
- *IEC*: the standard IEC 60529 on degrees of protection provided by enclosures. Search for its title on iec.ch.
- *CadQuery*: the documentation at cadquery.readthedocs.io, for the Workplane API, the box and extrude functions, and the exporters
- *Prusa Knowledge Base*: the guides at help.prusa3d.com on print orientation, layer lines and the materials PLA, PETG and ASA

**Tutorials**

- *GeeksforGeeks*: search for "Cantilever beam deflection" for the basic formulas
- *W3Schools*: the Python tutorial on w3schools.com for the basics of the scripts and tests
- *Python documentation*: python.org tutorial on modules and functions, for the package structure

**Watch**

- *CadQuery*: the project's examples on its documentation site, and its introductory tutorial videos on the official channel
- *Practical Engineering*: the YouTube channel by Grady Hillhouse has accessible videos on structures, materials and deflection

**Practice**

- *CadQuery online*: the documentation includes runnable examples for the Workplane API. Run the box and hole examples and change the dimensions.

---

## Tomorrow

Tomorrow, on Day 41, you learn design for manufacturing. You look at 3D printing, machining and laser cutting, the tolerances each process can hold, fasteners, materials and assembly. The enclosure and the brackets you designed today only work if they can be made with the tolerance and the process you chose, and Day 41 teaches you the rules that decide that.
