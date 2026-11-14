# Day 42: CAD Project

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 6 hours, which you can split over two sessions
**Level:** intermediate. This is the capstone of Module 5. You need every earlier day in the module, and the motor and power work from Module 2. The project draws on the design rules, the fits, the assembly checks and the material choices from Days 34 to 41.

**By the end of this day, you will have:**

- A small two-wheel differential chassis, designed as a parametric CAD assembly with a laser-cut top plate, printed brackets and an electronics enclosure
- A two-view technical drawing with dimensions, a title block and a general tolerance note, exported as PDF
- An exploded view, a bill of materials and a manufacturing plan that gives the process for every part
- A stability and acceleration check that shows the maximum acceleration before the castor unloads, with the software limit that follows from it
- A Python check that compares the bill of materials with the assembly counts, with tests

**What you need:**

- FreeCAD 1.0.x with the PartDesign, Assembly and TechDraw workbenches, free and open source from freecad.org. Onshape (free plan) or Autodesk Fusion (Personal licence for non-commercial use) are alternatives.
- Python 3.12 with pytest 8 or newer
- A DXF export from the CAD tool, read by your laser cutter's software. Most laser cutting software reads DXF files directly.
- Access to a 3D printer, and a laser cutter if you can reach one. A makerspace or a fabrication lab usually provides both.
- A two-wheel gearmotor kit with two 66 mm wheels, a castor, two 608 bearings, M3 bolts, nuts and standoffs, about $30 to $60 for the set
- A digital caliper, reading to 0.01 mm, and a kitchen scale, reading to 1 g
- A 2S or 3S battery, or a 6 V battery pack, of about 0.6 kg
- The parts from Days 37 to 41 saved as STEP files, including the enclosure, the motor bracket and the wheel hub

**Prerequisites:**

- All of Module 5, Days 34 to 41
- Day 13: motor drive, stall current and the battery
- Day 36: the differential drive equations and odometry
- Day 41: the DFM rules, the kerf and the hole compensation, and the checker

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How robot chassis design developed](#how-robot-chassis-design-developed)
- [Glossary](#glossary)
- [Requirements and the design brief](#requirements-and-the-design-brief)
- [Layout and the mass budget](#layout-and-the-mass-budget)
- [Stability and acceleration](#stability-and-acceleration)
- [The assembly](#the-assembly)
- [Drawings](#drawings)
- [Bill of materials](#bill-of-materials)
- [Manufacturing plan](#manufacturing-plan)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a chassis design package](#project-a-chassis-design-package)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A chassis is the first complete mechanical system in a robot. It carries the drive, the battery and the electronics, it holds the sensors at a fixed position, and it is the first thing the software must know about. In a real team, the chassis is designed as a package: the CAD model, the drawings, the bill of materials, the manufacturing plan and the checks that show it will work. A model alone is not enough, because a manufacturer or a teammate who receives only the model has to guess the material, the tolerance and the process.

Today you design that package for a small differential-drive robot. You apply the requirements, the layout and the mass budget, the stability and acceleration check, the assembly, the drawings and the process rules from this module. You finish with a bill of materials and a manufacturing plan, and a Python check that keeps the bill of materials honest. This is the capstone of Module 5, and the same package format carries into the electronics and software modules that follow.

---

## How robot chassis design developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1950s to 1960s | Early mobile robots, such as Shakey at the Stanford Research Institute (1966 to 1972), carry their electronics on a wheeled platform built from standard parts. | Platform design is one of the oldest problems in mobile robotics. |
| 1970s to 1980s | Hobby robots and educational platforms such as the Turtle robots from the MIT Logo project use simple wheeled chassis and standard motors. | Small educational chassis still use the same two-wheel differential layout. |
| 1990s | The LEGO Mindstorms system (1998) brings modular chassis design to schools. | Modular platforms teach the same design habits as a custom chassis. |
| 2000s | Open-source platforms and 3D printing make custom chassis parts cheap and fast to make. | A chassis can now be designed, printed and tested in a week. |
| Today | Commercial robots use aluminium chassis, carbon fibre panels and injection-moulded parts, and makers use laser-cut acrylic and printed brackets. | The process choice is the same decision you make today, at a different scale. |

The history shows that the chassis is where the mechanical, electrical and software teams first meet. Its package is the document that brings them together.

---

## Glossary

- **Chassis:** the structural frame of a robot that carries the drive, the electronics and the load.
- **Track width:** the distance between the centres of the two driven wheels.
- **Wheelbase:** the distance between the front and rear axles, or, for a differential robot with a castor, the distance from the drive axle to the castor axle.
- **Castor:** a free-rotating support wheel, which carries weight without driving the robot.
- **Centre of mass (CoM):** the point through which the weight of the robot acts, as defined on Day 34.
- **Load transfer:** the change in the normal force on each wheel when the robot accelerates or brakes.
- **Mass budget:** a list of the mass of each subsystem, with a total that must stay below a target.
- **Bill of materials (BOM):** the list of parts, quantities, processes and files that a robot is built from.
- **Manufacturing plan:** the document that states the process, the material, the tolerance and the order of assembly for each part.
- **Title block:** the part of a drawing that gives the name, the material, the scale, the revision and the author.
- **General tolerance:** a default tolerance for dimensions without an explicit tolerance, such as ISO 2768 medium.
- **Exploded view:** a view of an assembly with its parts separated along their assembly path, used in the manual and the BOM.
- **Revision:** a letter or number that marks a version of a drawing, so that changes can be tracked.

---

## Requirements and the design brief

A design package starts with a brief. The brief states what the robot must do, and the requirements turn that into numbers. For this project the brief is a small indoor robot that drives on a flat floor, carries a sensor board and a battery, and can be repaired with a screwdriver. The requirements are:

| Item | Requirement | Reason |
|---|---|---|
| Drive | Differential drive, two driven wheels of 66 mm diameter | Matches the Day 36 drivetrain and odometry |
| Total mass | 1.8 kg or less, including battery | Motor size and battery life |
| Top speed | 0.5 m/s | Safe speed for an indoor lab |
| Acceleration limit | At least 1.0 m/s² with the castor in contact | Keeps the robot stable under software control |
| Plate deflection | Under 0.5 mm at the battery load | Keeps the gears aligned |
| Process | Laser-cut top plate, printed brackets and enclosure | Matches the process rules from Day 41 |
| Repair | Every part removable with M3 tools | Serviceability |

The brief also has a requirement it does not state outright: the robot must not tip or lift its castor under normal use. The stability check below tests that requirement, and it is the reason the software limit is set.

The common mistake is to write the brief after the design, so that the requirements match the design by accident. Write the table first, and change it only with a stated reason.

---

## Layout and the mass budget

The layout places the parts on the plate. A differential robot has the two driven wheels on one axle, a castor at one end and the battery and the enclosure above the plate. The centre of mass should sit close to the drive axle, so that the castor carries a small share of the weight and the driven wheels carry most of it.

The first principle is that the layout sets the load distribution. The centre of mass must lie between the two supports, which are the drive axle and the castor. If it lies behind the axle, the castor lifts even when the robot is at rest. If it lies far toward the castor, the castor carries too much weight and the drive wheels lose traction. A small offset toward the castor is the usual target.

Here is the layout used in this project. The top plate is 160 mm long and 120 mm wide, and it is laser cut from 3 mm acrylic. The two driven wheels sit on an axle at the middle of the plate, with 66 mm wheels and a track of 150 mm between the wheel centres. A castor sits 90 mm ahead of the drive axle. The battery sits just ahead of the axle, 10 mm toward the castor, and the enclosure sits above the plate between the wheels.

The mass budget lists each subsystem, with the target of 1.8 kg:

| Subsystem | Mass (kg) |
|---|---|
| Battery | 0.60 |
| Motors (two gearmotors) | 0.14 |
| Wheels (two) | 0.12 |
| Castor | 0.02 |
| Chassis plate (acrylic, with holes) | 0.09 |
| Electronics enclosure with board | 0.12 |
| Sensors | 0.10 |
| Fasteners | 0.03 |
| Cabling and margin | 0.58 |
| Total | 1.80 |

The cabling and margin line is large on purpose. A design that uses all of its budget on the first build leaves no room for the cable, the wire and the change that the first test will ask for.

The common mistake is to take the mass of a part from the CAD model without a density check. The CAD mass uses the density you set for the material, and an acrylic plate with a default density is wrong by 10 to 20 percent. Use the density of the actual sheet, which is about 1.19 g/cm³ for cast acrylic.

---

## Stability and acceleration

A robot that accelerates hard can lift its castor, and a robot that lifts its castor can tip or lose control. The check is a load-transfer calculation, which uses the weight, the height of the centre of mass and the wheelbase.

The first principle is the load transfer. When the robot accelerates at a, the inertial force m a acts at the centre of mass, at height h above the ground. That force creates a moment about the contact points, and it shifts a load ΔN from the front to the rear. For a castor ahead of the drive axle, the load transfer that unloads the castor is:

ΔN = m a h / L

where L is the distance from the drive axle to the castor, and h is the height of the centre of mass.

Here is the worked example for this chassis. The total mass is m = 1.8 kg, the centre of mass height is h = 0.08 m, and the castor is L = 0.09 m ahead of the drive axle. The centre of mass is 10 mm ahead of the axle, toward the castor. Taking moments about the drive axle, the static load on the castor is:

N_castor = m g × (0.010 / 0.090) = 0.111 m g = 0.111 × 1.8 × 9.81 = 1.96 N

The castor unloads completely when the load transfer equals this static load:

m a h / L = 0.111 m g

which simplifies to a = g x / h, where x is the offset of the centre of mass ahead of the axle. So a = 9.81 × 0.010 / 0.08 = 1.23 m/s².

The robot can accelerate at up to about 1.2 m/s² before the castor lifts. The requirement from the brief is 1.0 m/s², so the design passes. The software must limit the acceleration to 1.0 m/s² or less, and the margin of 0.23 m/s² is the safety factor for the castor. The formula shows the sensitivity: moving the battery 20 mm toward the castor would raise the limit, while moving it back toward the axle would lower it sharply, and at the axle itself the castor lifts under any acceleration. This is why the layout and the stability check belong together.

The friction check is separate. The driven wheels carry the rest of the weight, which is 0.889 m g, and the maximum traction at μ = 0.6 is 0.6 × 0.889 × 1.8 × 9.81 = 9.4 N. A requirement of 1.0 m/s² needs a force of m a = 1.8 N. The friction limit is far above it, so the castor is the limit, not the traction.

The common mistake is to check friction alone and to assume a robot that does not slip is stable. A robot can have plenty of traction and still tip, or lift a wheel, when the load transfer is too large. Check both.

---

## The assembly

The assembly places the parts as instances, as you learned on Day 39. The chassis plate is the ground. The two motor brackets from Day 40 are fixed to the plate with M3 bolts. The motors are fixed to the brackets. The wheel hubs from Day 38 are placed on the motor shafts with revolute joints, and the castor is fixed to the plate with its own mount. The enclosure sits on four standoffs, which are fixed to the plate with M3 bolts from below.

The first principle is the order of the assembly. A part must be reachable with a screwdriver when it is added, and a fastener must be reachable after the part above it is placed. The motors must go on before the wheels, and the standoffs must go on before the board. Check the order with the exploded view.

The degrees-of-freedom check from Day 39 gives the robot's motion. The chassis is grounded. Each wheel has a revolute joint, so each adds one degree of freedom. The castor adds a free rotation, which the model represents as a revolute joint with a vertical axis and a free swivel. The assembly count should show that the robot's two wheels are the only moving parts of the drive, and that the castor is the only other free motion.

The common mistake is to build the assembly by placing parts by eye. Use joints and mates, and then check the interference report. A clean assembly report is the first sign that the parts fit.

---

## Drawings

A drawing is the document the manufacturer reads. It shows the views of a part, its dimensions, its tolerances, its material and its process. A technical drawing for a laser-cut plate has a top view with all holes and slots dimensioned, and a note that states the material, the thickness and the kerf.

The first principle is that every dimension on the drawing must be needed, and that every needed dimension must appear. A dimension that appears twice invites a conflict, and a missing dimension forces the manufacturer to guess. Use the FreeCAD TechDraw workbench to place the views, and add the dimensions from the model.

Here is the drawing note for the top plate:

"Chassis plate, cast acrylic, 3.0 mm thick, laser cut. Material density 1.19 g/cm³. Laser kerf 0.15 mm, measured on test coupon. Holes drawn at 3.25 mm to finish at 3.4 mm (M3 clearance). Slots drawn at 2.85 mm for 3.0 mm tabs. General tolerance ISO 2768-m. Revision A."

The note tells the cutter the material, the thickness and the kerf. It tells the manufacturer that the holes and slots are compensated, so that a reader does not "correct" them back to the nominal size. It also gives the general tolerance and the revision.

The common mistake is to put the printed part's hole value on a laser-cut drawing, or to omit the kerf. A hole drawn for a printer is 3.58 mm, and the same hole cut by a laser is 3.25 mm drawn. Use the process rules from Day 41 for each part.

---

## Bill of materials

The bill of materials is the list of every part, with its quantity, its process, its file and its source. It must agree with the assembly. A BOM that lists four bearings when the assembly has two is a purchasing error, and a BOM that lists a part that the assembly does not use is a waste of money.

The first principle is one row per unique part. Two identical brackets are one row with a quantity of two. The BOM is the document that the purchasing team and the workshop both read, so every row must have a part number and a source.

Here is the BOM for one chassis, with the counts for the assembly:

| Part | Quantity | Process | File |
|---|---|---|---|
| chassis_plate | 1 | Laser cut, 3 mm acrylic | dxf/chassis_plate.dxf |
| motor_bracket | 2 | FDM, PETG | cad/parts/motor_bracket.step |
| tt_motor | 2 | Purchased gearmotor | purchased |
| wheel_hub | 2 | FDM, PETG | cad/parts/wheel_hub.step |
| castor | 1 | Purchased | purchased |
| bearing_608 | 4 | Purchased | purchased |
| enclosure_base | 1 | FDM, PLA or ASA | cad/parts/enclosure_base.step |
| enclosure_lid | 1 | FDM, PLA or ASA | cad/parts/enclosure_lid.step |
| standoff_m3_15mm | 4 | Purchased | purchased |
| m3_bolt | 12 | Purchased | purchased |
| m3_nut | 12 | Purchased | purchased |

The bearing count is four because each wheel uses two bearings in its bracket, and the motor shaft carries the load through them. If your design uses a different number, change the count and the assembly file together.

The common mistake is to count the bolts from the drawing and forget the nuts and the washers. Count every fastener in the assembly, and check the count with the exploded view.

---

## Manufacturing plan

The manufacturing plan states, for each part, the process, the material, the tolerance and the order of assembly. It turns the design into a list of actions that a person can carry out, and it names the checks that confirm each part before it goes on the robot.

The first principle is that a manufacturing plan follows the DFM rules. A printed bracket needs a minimum wall of 1.2 mm, a print orientation with its layers along the bending direction, a hole compensation from a test coupon and a visual check of the overhangs. A laser-cut plate needs a kerf check, a coupon of the tab and slot, and a check of the flatness after cutting. Each step is a check that a reviewer can confirm.

Here is the plan for the chassis plate. Cut the plate from 3 mm acrylic, with the holes and slots from the DXF file, on a laser cutter with a measured kerf of 0.15 mm. Test-fit one tab and one slot before cutting the full plate. Measure three holes with the caliper, and accept the plate if each is within 0.05 mm of 3.40 mm. Remove the protective film only after the plate is assembled, because the film protects the surface from scratches during assembly.

Here is the plan for a motor bracket. Print the bracket in PETG, 0.2 mm layer height, 4 walls, 30 percent infill, with its layers along the bend. Check the bolt holes with an M3 bolt before fitting. Measure the motor seat with the motor, and accept the bracket if the motor slides in with a light fit.

The order of assembly is: plate, standoffs, motor brackets, motors, wheels and hubs, castor, battery, enclosure, board, cables. Each step ends with a check. The motor step ends with a check that the shaft turns freely, and the wheel step ends with a check that the wheels roll straight with no load.

The common mistake is to write a plan that lists the process but not the checks. A plan without checks gives no way to confirm that a step was done. Add a check to every step.

---

## Hands-on exercises

1. **Measure the mass budget.** Weigh each part on the kitchen scale, and compare it with the budget table. Record the difference for each row, and find the row with the largest error.

2. **Test the castor limit.** Place the assembled chassis on a smooth floor with the battery in place. Push it gently forward until the castor lifts, and measure the force with a spring scale. Compare the result with the value you computed from the stability check.

3. **Check the wheel alignment.** Drive the robot 1 m forward on a taped line. Measure the lateral error, and compare it with the expected error from the drivetrain's odometry from Day 36.

4. **Measure a laser-cut hole.** Cut a coupon with the hole and slot from the DXF file, and measure the finished holes and slots. Use the measured kerf to check the drawn values, and update the drawing note if the kerf differs from 0.15 mm.

5. **Exploded view check.** Use the exploded view to count the fasteners in the built robot. Compare the count with the BOM, and correct either one until they agree.

---

## Project: a chassis design package

This project delivers a design package for the chassis: the CAD assembly, the drawings, the exploded view, the bill of materials, the manufacturing plan, and a Python check that compares the BOM with the assembly. It is the full project for Module 5, and it is the document a team would hand to a manufacturer or a teammate.

**Goal:** a complete package for the two-wheel differential chassis described in this lesson. The package must pass the BOM check, the stability check and the DFM checker from Day 41, and it must be complete enough for another person to build the robot without asking you a question.

**Repository name:** `100-days-robotics-day-42-chassis-cad-project`

**Repo layout:**

```
100-days-robotics-day-42-chassis-cad-project/
├── README.md
├── requirements.txt
├── cad/
│   ├── chassis.FCStd
│   ├── chassis.step
│   └── parts/
│       ├── motor_bracket.step
│       ├── wheel_hub.step
│       ├── enclosure_base.step
│       └── enclosure_lid.step
├── dxf/
│   └── chassis_plate.dxf
├── drawings/
│   └── chassis_drawing.pdf
├── exploded/
│   └── exploded.png
├── bom.csv
├── assembly.json
├── manufacturing_plan.md
├── stability.py
├── check_bom.py
├── dfm/
│   └── rules.py
└── tests/
    ├── test_check_bom.py
    └── test_stability.py
```

**Starter code.** Write `stability.py`, which computes the castor-lift acceleration from the layout and the centre of mass height.

```python
# stability.py
G = 9.81


def castor_lift_acceleration(h_cm, castor_lever, cm_offset_ahead_of_axle):
    """Acceleration (m/s^2) at which the castor unloads.

    h_cm: height of the centre of mass above the ground (m)
    castor_lever: distance from the drive axle to the castor (m)
    cm_offset_ahead_of_axle: distance from the drive axle to the centre of mass,
        toward the castor (m). Must be positive, or the castor is unloaded at rest.
    """
    if cm_offset_ahead_of_axle <= 0:
        raise ValueError("centre of mass must lie ahead of the drive axle")
    static_fraction = cm_offset_ahead_of_axle / castor_lever
    return static_fraction * G * castor_lever / h_cm


if __name__ == "__main__":
    a = castor_lift_acceleration(h_cm=0.08, castor_lever=0.09, cm_offset_ahead_of_axle=0.010)
    print(f"castor lifts at {a:.2f} m/s^2")
```

The function simplifies to a = g x / h, where x is the offset of the centre of mass ahead of the axle. For the worked example that is 9.81 × 0.010 / 0.08 = 1.23 m/s², and the script must print that value.

**Starter code.** Write `check_bom.py`. It compares the BOM with the assembly counts and checks that each file exists.

```python
# check_bom.py
import csv
import json
import os
import sys


def load_bom(path):
    with open(path, newline="") as f:
        return {
            row["part"]: {"qty": int(row["qty"]), "file": row["file"]}
            for row in csv.DictReader(f)
        }


def check(bom, counts, root="."):
    problems = []
    for part, need in counts.items():
        if part not in bom:
            problems.append(f"{part}: missing from BOM")
            continue
        if bom[part]["qty"] != need:
            problems.append(f"{part}: BOM says {bom[part]['qty']}, assembly needs {need}")
        f = bom[part]["file"]
        if f != "purchased" and not os.path.exists(os.path.join(root, f)):
            problems.append(f"{part}: file {f} not found")
    for part in bom:
        if part not in counts:
            problems.append(f"{part}: in BOM but not in assembly")
    return problems


def main(bom_path, assembly_path, root="."):
    with open(assembly_path) as f:
        counts = json.load(f)["counts"]
    problems = check(load_bom(bom_path), counts, root)
    for p in problems:
        print("FAIL", p)
    if not problems:
        print("PASS: BOM matches the assembly")
    return 1 if problems else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1], sys.argv[2]))
```

Write `assembly.json` with the same counts as the BOM table, and write `bom.csv` with the columns `part,qty,process,file`.

**Step-by-step tasks.**

1. Create the repository and the virtual environment. Install pytest. Copy `dfm/rules.py` from Day 41 into the repo.
2. Build the chassis in FreeCAD 1.0. Use the parts from Days 38, 40 and 41 as STEP files, placed as instances in the assembly. Use joints for the wheels and the castor, and fixed joints for the brackets and the standoffs.
3. Run the interference check in the assembly tool. Resolve every overlap, or record it as an intended fit with a reason.
4. Write `stability.py` and run it. Confirm that it gives 1.23 m/s² for the worked example. Record the result in the README.
5. Export the top plate as a DXF file for the laser cutter, with the holes and slots drawn at their compensated values.
6. Create the TechDraw page with a top view, a front view and a title block. Dimension every hole, slot and outside edge, add the drawing note, and export the page as `drawings/chassis_drawing.pdf`.
7. Create the exploded view and save it as `exploded/exploded.png`.
8. Write `bom.csv` and `assembly.json`. Run `python check_bom.py bom.csv assembly.json` and fix every failure.
9. Write `tests/test_check_bom.py` with a passing case, a case where the quantity is wrong, and a case where a file is missing. Use a temporary folder for the file check.
10. Write `tests/test_stability.py` with a test for the worked example and a test that a battery moved forward reduces the castor lift acceleration.
11. Run the DFM checker from Day 41 on the printed parts, using their design files, and record any violation.
12. Write `manufacturing_plan.md` with the process, the material, the tolerance, the checks and the order of assembly for every part.
13. Write the README. It is the front page of the package, so it must state the brief, the requirements, the stability result, the mass budget, and where each file is.

**README checklist:**

- [ ] The requirements table, with the reason for each requirement
- [ ] The mass budget, with the measured mass of the built robot
- [ ] The stability result from `stability.py`, and the software acceleration limit
- [ ] The links to the drawing, the exploded view, the BOM and the manufacturing plan
- [ ] The output of `check_bom.py` and of the tests
- [ ] A short list of what you would change in revision B, with the reason

**Done when:**

- `pytest` passes all tests
- `check_bom.py` prints PASS for the BOM and the assembly
- `stability.py` prints 1.23 m/s² for the worked example, and the software limit in the README is 1.0 m/s² or less
- The drawing has every hole and slot dimensioned, with the drawing note, and it exports to PDF
- A person who has not seen the model can build the robot from the package, which you can test by asking a classmate to read the manufacturing plan and to list the first three steps
- The README names one change you would make in revision B, with a reason from your measurements

**What to commit to GitHub:** the CAD files, the STEP exports, the DXF file, the PDF drawing, the exploded view, the BOM, the assembly file, the manufacturing plan, the scripts, the tests, the DFM checker output and the README. Add a `.gitignore` for FreeCAD backup files and cache folders.

**Why this project is full and not light:** it is the complete design package, with every check from Module 5 applied to one real robot.

---

## Check your understanding

1. A chassis has a total mass of 2.0 kg, a centre of mass height of 0.10 m, and a castor 0.12 m ahead of the drive axle with the centre of mass 15 mm ahead of the axle, toward the castor. Compute the acceleration at which the castor unloads.

2. A designer moves the battery 5 mm back toward the axle to make room for a sensor at the front, so the centre of mass is 5 mm ahead of the axle instead of 10 mm. Explain the effect on the castor-lift acceleration and on the software limit, and say whether the brief still passes.

3. A BOM lists two motor brackets, but the assembly file has four. Explain how the check catches the error, and what the consequence would be if the BOM were used to order parts.

4. A drawing gives the hole diameter as 3.4 mm, with no kerf note, for a laser-cut plate. Explain what goes wrong, and write the correct drawing note.

5. The mass budget is 1.8 kg, but the built robot weighs 2.1 kg. Explain three places the extra mass could come from, and state which one you would check first.

6. A manufacturing plan says "print the bracket" with no orientation, no walls and no checks. Explain why this plan fails, and write the minimum of the plan that a second person could follow.

**Answers:**

1. The static castor fraction is 0.015 / 0.12 = 0.125. The acceleration is a = 0.125 × g × L / h = 0.125 × 9.81 × 0.12 / 0.10 = 1.47 m/s², which equals g x / h = 9.81 × 0.015 / 0.10. The mass does not change the result, because it cancels out.

2. Moving the centre of mass back toward the axle reduces the static castor load, so the castor unloads at a lower acceleration. The new value is a = 9.81 × 0.005 / 0.08 = 0.61 m/s². That is below the 1.0 m/s² requirement, so the software limit must drop to about 0.6 m/s², and the brief no longer passes. The designer must either move the battery back to its 10 mm position, or accept a lower acceleration limit and record that change in the requirements.

3. The check finds that the BOM says two and the assembly needs four, and it prints a FAIL line. If the BOM were used to order parts, the workshop would receive two brackets, and the robot would be built with two missing parts. The check prevents that order.

4. The laser kerf widens each hole by about 0.15 mm, so a hole drawn at 3.4 mm will be cut at about 3.55 mm, which is loose for an M3 bolt. The drawn value must be 3.4 − 0.15 = 3.25 mm, and the note must state the kerf and the drawn value, so that the cutter and the manufacturer know the hole is compensated.

5. The extra mass could come from the cabling and margin, from the battery being heavier than the datasheet, or from a part printed with more infill or walls than the design assumed. Check the largest uncertain item first, which is usually the battery, and then the infill of the printed parts.

6. The plan does not say how to make the part, so a second person may print it in the wrong material, orientation or wall count, and the part may fail. The minimum plan states: material PETG, 0.2 mm layers, 4 walls, 30 percent infill, layers along the bend, a bolt check of the holes, and a visual check before assembly.

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Differential wheeled robot", "Caster", "Centre of mass", "Load transfer" (search if the title differs), "Bill of materials", "Technical drawing", "Design for manufacturability" and "Shakey the robot"
- *ISO*: ISO 2768 for general tolerances and ISO 286 for limits and fits, on iso.org
- *ASME*: ASME Y14.5 for geometric dimensioning and tolerancing, on asme.org
- *FreeCAD*: the TechDraw and Assembly workbench documentation at wiki.freecad.org
- *Prusa Knowledge Base*: the guides at help.prusa3d.com on print orientation, walls and tolerance, for the printed parts

**Tutorials**

- *GeeksforGeeks*: search for "Free body diagram" and "Stability of a body" for the stability calculation
- *W3Schools*: the Python tutorial on w3schools.com for the check scripts
- *Python documentation*: python.org tutorial on modules, files and tests

**Watch**

- *MIT OpenCourseWare*: course 2.007 Design and Manufacturing I, which covers the design package and the drawing process. Search for the course on ocw.mit.edu.
- *FreeCAD*: the official FreeCAD YouTube channel has TechDraw tutorials. Search for "TechDraw" on the channel.

**Practice**

- *Onshape Learn*: the drawing and assembly tutorials at learn.onshape.com. Build the same chassis in Onshape and compare the BOM counts with your FreeCAD model.

---

## Tomorrow

Tomorrow, on Day 43, you move from mechanical design to electronics layout. You learn PCB fundamentals: layers, traces, pads, vias, footprints, solder mask, silkscreen, ground planes and power distribution. The enclosure you designed today holds a board, and the board's size, its mounting holes and its connectors come from the design package. The first board you design in Module 6 has to fit inside the enclosure that you just checked.
