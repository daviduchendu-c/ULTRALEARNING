# Day 39: Assemblies

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 4 hours
**Level:** intermediate. You need the parts from Days 37 and 38, the drive and linkage ideas from Days 35 and 36, and the fits and clearances that Day 35 introduced for bearings. An assembly is the first place where the parts you have made must work together, so the checks you learn today catch the errors that single parts hide.

**By the end of this day, you will have:**

- An assembly of the motor bracket and the wheel hub, with a fixed joint, a revolute joint and a bearing seat, built in FreeCAD 1.0 or another CAD tool
- A degree-of-freedom count for the assembly, computed by hand with the Kutzbach formula and checked in the CAD tool
- A fit table for the bearing seat and the shaft, classified as clearance, transition or interference, with the clearance computed from the tolerances
- A bill of materials and an assembly hierarchy written in the README

**What you need:**

- FreeCAD 1.0.x, with the Assembly workbench (in FreeCAD 1.0 the assembly tools are available in the Workbench menu; if they are missing, check the Addon Manager). Onshape (free plan) and Autodesk Fusion (Personal licence for non-commercial use) also support joints and mates.
- The parts from Day 37 and Day 38, saved as separate files, plus a simplified motor model. A cylinder 28 mm square by 30 mm long with a 5 mm shaft is enough.
- Python 3.12 with pytest 8 or newer (`pip install pytest`)
- A digital caliper, reading to 0.01 mm, for measuring the hub bore and the shaft
- Optional: a 608 ball bearing and an 8 mm shaft, to test a press fit with a printed housing

**Prerequisites:**

- Day 37: sketches, spreadsheet parameters and reference geometry
- Day 38: revolutions, patterns and fillets, and the hub part file
- Day 35: bearing sizes and the basic idea of a fit between a shaft and a bore
- Day 36: the drivetrain you chose, so you know which joints the wheels need

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How assemblies developed](#how-assemblies-developed)
- [Glossary](#glossary)
- [Parts and instances](#parts-and-instances)
- [Joints and mates](#joints-and-mates)
- [Degrees of freedom](#degrees-of-freedom)
- [Interference checks](#interference-checks)
- [Clearances and fits](#clearances-and-fits)
- [Assembly hierarchy and the bill of materials](#assembly-hierarchy-and-the-bill-of-materials)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a motor and wheel assembly with checks](#project-a-motor-and-wheel-assembly-with-checks)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot is an assembly. Each part is a shape on its own, but a robot only works when the parts fit, move freely where they should, and stay still where they must. An assembly is the model that puts parts together and tests whether they do. It catches the errors that single parts hide: a shaft that is too large for a bore, a wheel that rubs the chassis, a joint that is missing a degree of freedom, or a bolt that goes through the motor and not the bracket.

Today you learn to build assemblies with joints, to count the degrees of freedom and check that they match the motion the robot needs, to test for interference, and to choose clearances and fits from standard tables. You also learn the hierarchy that keeps a large assembly readable, and the bill of materials that the workshop and the purchasing team use. The assembly you build here is the motor and wheel unit of the robot, and the checks carry through to the chassis on Day 42.

---

## How assemblies developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1800s | Engineering drawings start to show assemblies with part numbers and exploded views, which lets the workshop build the same machine twice. | The bill of materials and the exploded view you make today follow the same logic. |
| 1900s | Interchangeable parts and standard fits, such as the ISO system of limits and fits, become established. | Standard fits let you buy a bearing and design its seat without measuring every part. |
| 1960s to 1980s | CAD systems add assembly modelling, and the first constraint solvers for mates appear. | Mates and joints in today's tools use the same solver ideas. |
| 1990s to 2000s | Assembly kinematics and interference checking become standard features in commercial CAD. | The DOF check you do by hand is the same one a CAD solver does in the background. |
| Today | Free tools support joints, mass properties and exploded views, and simulation can run the motion of an assembly before any part is cut. | The model you check today can be tested in motion before you order parts. |

The history shows that assemblies moved from paper to model. The method, which is to check the fit and the motion before the parts are built, has not changed.

---

## Glossary

- **Assembly:** a model made from several parts or sub-assemblies, placed in their correct positions relative to each other.
- **Instance:** a copy of a part placed in an assembly, which refers to the same part file. Changing the part changes every instance.
- **Joint (or mate):** a rule that places one part relative to another and removes degrees of freedom. Common types are fixed, revolute, cylindrical, slider and planar.
- **Revolute joint:** a joint that allows rotation about one axis, such as a wheel on an axle.
- **Fixed joint:** a joint that removes all relative motion, so the two parts move together.
- **Degree of freedom (DoF):** an independent motion of a body in an assembly. A rigid body in space has six, three translations and three rotations.
- **Kutzbach formula:** a formula that counts the degrees of freedom of a mechanism from the number of links and joints.
- **Interference:** a situation where two parts occupy the same space, which means they cannot be built as drawn.
- **Clearance:** the gap between a shaft and a bore when the shaft is smaller than the bore.
- **Interference fit:** a fit where the shaft is larger than the bore, so the parts must be pressed together.
- **Tolerance:** the permitted variation in a dimension, given as a maximum and a minimum.
- **Limits and fits (ISO 286):** the international system of tolerance grades and fit classes for cylindrical features.
- **Bill of materials (BOM):** a list of the parts in an assembly, with their quantities, part numbers and sources.
- **Exploded view:** a drawing or model in which the parts of an assembly are shown separated along their assembly path.
- **Sub-assembly:** an assembly that is itself part of a larger assembly, such as the motor unit inside the robot.

---

### Parts and instances

A part is a single file that describes one shape, such as the bracket or the hub. An instance is a placed copy of that part in an assembly. Think of a part as a cookie cutter and an instance as a cookie. One cutter can make many cookies, and changing the cutter changes every cookie made from it afterwards.

The first principle is that the instance keeps a link to its part file, so the assembly does not duplicate the geometry. This matters for a robot with four identical wheels. You make one hub and place four instances, and a change to the hub bore updates all four.

Here is a worked example. The motor unit has one bracket, one motor and one hub. The robot has two motor units, so the assembly contains two brackets, two motors and two hubs, which are six instances from three part files. The bill of materials for the robot lists three parts with quantities of two, two and two, not six separate drawings.

The common mistake is to copy a part file and edit the copy, so that two parts that should be identical drift apart. Use instances for identical parts, and copy the file only when the design differs. The second mistake is to build the assembly inside a part file, which makes the part and the assembly impossible to edit separately.

Instances are the reason a large robot can be designed by a team. Each person edits their own part, and the assembly updates when the part is saved.

---

### Joints and mates

A joint places one part relative to another. Each joint removes some degrees of freedom and leaves others. A fixed joint removes all the relative motion between two parts, so they behave as one body. A revolute joint allows rotation about one axis and removes the other five. A slider allows translation along one axis, and a cylindrical joint allows both rotation and translation about the same axis.

The first principle is that every joint is a rule about contact or alignment. A revolute joint between a wheel and an axle says that the wheel's bore axis coincides with the axle's axis, and that the faces of the wheel and the axle touch at one point along the axis. The solver checks that the rule holds in every position.

Here is a worked example for the motor unit. The bracket is fixed to the chassis, which is the ground. The motor is fixed to the bracket with a fixed joint, because its bolts hold it rigidly. The hub is placed on the motor shaft with a revolute joint about the shaft axis, which allows the hub to rotate. The hub's rotation is the only motion the unit needs, so the revolute joint is the correct choice.

The common mistake is to use a fixed joint where the part must move, or a revolute joint where the part must be held. The result is either a blocked wheel or a wheel that wobbles. Check the joint type against the motion the robot needs, using the drivetrain from Day 36.

Joints are the link between the CAD model and the motion of the real robot. A joint that is wrong in the model will be wrong in the machine, and the model is the cheapest place to find that error.

---

### Degrees of freedom

Degrees of freedom are the independent motions of an assembly. A free body in space has six degrees of freedom, three translations along the axes and three rotations about them. Each joint removes some of these. The count tells you whether the assembly moves as intended.

The Kutzbach formula gives the number of degrees of freedom M of a spatial mechanism:

M = 6 (N − 1 − J) + Σ f_i

where N is the number of links, including the ground, J is the number of joints, and f_i is the number of degrees of freedom of joint i. A fixed joint has f = 0, a revolute has f = 1, and a slider has f = 1.

Here is a worked example for the wheel on the motor unit. There are three links: the ground (chassis), the bracket-motor body and the hub. The chassis to bracket joint is fixed, and the bracket to motor body is fixed. For the simplest count, take the motor body as the second link and the hub as the third. The joints are: chassis to motor body, fixed, f = 0; motor body to hub, revolute, f = 1. So N = 3, J = 2 and Σf = 1. Then M = 6 (3 − 1 − 2) + 1 = 0 + 1 = 1. The assembly has one degree of freedom, which is the rotation of the hub. That is the intended motion.

If you forget the revolute joint and make the hub fixed, the count gives M = 6 (3 − 1 − 2) + 0 = 0, which means the wheel cannot rotate. If you make the hub free, with no joint at all, the count gives N = 3, J = 1, Σf = 0, and M = 6 (3 − 1 − 1) = 6, which means the hub can move freely in all six directions. Those three counts show what a DOF check catches.

The common mistake is to count the joints but forget to count the links, or to forget the ground. Write the list of links and the list of joints on paper before you use the formula. The CAD tool shows the result as a number too, and if the number disagrees with your count, one of the joints is probably misplaced.

Degree-of-freedom checks are used on every mechanism, from a door hinge to a robot arm with six joints. For a 6-joint arm, the count tells you how many motors you need and whether the arm can reach the pose it must reach.

---

### Interference checks

An interference check finds parts that occupy the same space in the assembly. The CAD tool compares the volumes of two parts and reports any overlap. Interference is the most common error in a first assembly, and it is always a real error, because two solid parts cannot occupy the same place in the physical world.

The first principle is that every part must have clearance from every part it does not touch. A shaft inside a bore needs a clearance on its diameter, and a motor body needs a clearance from the chassis. The check does not know your intent, so you must read the result and decide whether the interference is intended, as in a press fit, or a mistake.

Here is a worked example. The hub bore is 6.0 mm in diameter, and the shaft is 6.0 mm too. The interference check reports zero clearance, which is a transition, not a clear fit. If the hub bore is 6.1 mm, the clearance is 0.1 mm, and the check passes. If the shaft is 6.2 mm, the check reports an interference of 0.1 mm on the diameter, which means the hub must be pressed on, or it will not fit at all.

The common mistake is to rely on the nominal dimensions and to forget the printing tolerance. A 3D printed bore can come out smaller than its design value by a few tenths of a millimetre, depending on the printer, the material and the layer height, so a bore that clears in CAD can bind on the printer. Print a test piece for each fit, and measure it with the caliper.

Interference checks are part of every design review, and a manufacturer runs one first on any model that arrives from outside.

---

### Clearances and fits

A fit is the relation between a shaft and a bore that are to be assembled. The ISO 286 system defines the fit by tolerance grades and fit classes, so that a designer can choose a fit from a table instead of guessing. The letter and number codes, such as H7 for the hole and g6 for the shaft, set the upper and lower limits of each dimension.

The first principle is the tolerance band. A hole labelled H7 has a lower limit equal to its nominal size and an upper limit above it, by an amount that depends on the size and the grade. A shaft labelled g6 has an upper limit just below the nominal size. Where the bands overlap, the fit is a transition or an interference, and where they do not, the fit is a clearance.

Here is a worked example for the 608 bearing seat, which is an outside diameter of 22 mm. For a housing that the bearing sits in, a common clearance choice is H7, and the ISO 286 limits for a 22 mm hole in the 18 to 30 mm size range are about 0 to +0.021 mm. A bearing outer ring has a tolerance of about −0.008 to 0 mm, and it is usually specified by the manufacturer. A housing bore of 22.000 to 22.021 mm and a bearing outside of 21.992 to 22.000 mm gives a clearance of 0 to 0.029 mm, which is close to a transition fit. Bearing manufacturers and standards bodies give the exact values, so check the tables for your bearing before you choose.

The common mistake is to choose a fit without reading the tolerance table, or to apply a fit for a metal part to a printed plastic part. Plastic moves and swells with humidity and temperature, so a tight metal fit can crack a printed housing. Use a looser fit for printed parts, and test it.

Fits are used in every bearing seat, every shaft and every bushing in a robot. They are the place where a design either works on the first build or needs rework.

---

### Assembly hierarchy and the bill of materials

An assembly with fifty parts is hard to read as one flat list. A hierarchy groups the parts into sub-assemblies, such as the motor unit, the drivetrain and the electronics bay, and each sub-assembly is a model in its own right. The top-level assembly is the robot, and it is made from sub-assemblies, which are made from parts.

The first principle is that each level should be one function. The motor unit has the bracket, the motor, the hub and the fasteners. The drivetrain has four motor units and the chassis rails. Each level can be checked on its own, and the top-level assembly only needs to check the interfaces between the levels.

The bill of materials is the list of every part, with its quantity, part number, description and source. A good BOM has one row per unique part and a quantity column. It is the document that the workshop uses to cut and print, and the document that purchasing uses to order screws, bearings and motors.

Here is a worked example. The motor unit has one motor, one bracket, one hub, four M3 bolts with nuts, and two 608 bearings. The BOM for the robot with two motor units lists two motors, two brackets, two hubs, eight M3 bolts, eight M3 nuts and four bearings, with a single row for each part and its quantity.

The common mistake is to build a flat assembly with every part at the top level. A reviewer cannot find a part, and a change to one sub-assembly forces a review of the whole robot. Group the parts early, and name each sub-assembly by its function.

Hierarchies and BOMs are how robot teams stay organised as the model grows. A manufacturer reads the BOM early, and a clean one saves weeks of questions.

---

## Hands-on exercises

1. **Build the motor unit.** In FreeCAD 1.0, open the Assembly workbench and insert the bracket from Day 37 as the grounded part. Insert the simplified motor and fix it to the bracket with a fixed joint. Insert the hub from Day 38 and place it on the shaft with a revolute joint about the shaft axis.

2. **Count the degrees of freedom.** Write the Kutzbach count for your unit on paper, with N, J and Σf. Compare it with the number of degrees of freedom the CAD tool reports. Change one joint to fixed and count again. Record the results.

3. **Test the interference.** Run the interference check in the CAD tool. Note any overlap, and decide whether it is intended. Change the hub bore to 6.1 mm and run the check again.

4. **Measure a real fit.** Measure the bore of a printed part and the diameter of a shaft with the caliper. Compute the clearance, and compare it with the design value. Record the difference, which will show the printer's error for that material.

5. **Write the BOM.** Create a table of the unit's parts, with quantities, part numbers and sources. Check that each fastener is counted, including the nuts.

---

## Project: a motor and wheel assembly with checks

This project builds the motor unit as an assembly and checks it with two Python scripts. The first script counts the degrees of freedom from a list of joints. The second checks the clearance of the bearing and shaft fits from their tolerance limits. It is a full mini-project, because the unit is a real sub-assembly of the robot, and the checks are the first design review you will run.

**Goal:** an assembly of the bracket, motor and hub, with a revolute joint on the hub, a degree-of-freedom check that gives one degree of freedom, a fit table with at least three fits classified, and a bill of materials.

**Repository name:** `100-days-robotics-day-39-assemblies`

**Repo layout:**

```
100-days-robotics-day-39-assemblies/
├── README.md
├── requirements.txt
├── cad/
│   ├── motor_unit.FCStd
│   ├── motor_unit.step
│   └── parts/
│       ├── bracket.step
│       ├── motor_simplified.step
│       └── wheel_hub.step
├── assembly.json
├── fits.csv
├── dof.py
├── fit_check.py
├── bom.md
├── tests/
│   ├── test_dof.py
│   └── test_fit_check.py
└── renders/
    └── exploded.png
```

**The joint list.** Write `assembly.json` to describe the motor unit. The DOF script reads it.

```json
{
  "links": ["chassis", "motor_body", "hub"],
  "joints": [
    {"from": "chassis", "to": "motor_body", "type": "fixed"},
    {"from": "motor_body", "to": "hub", "type": "revolute"}
  ]
}
```

**Starter code.** Write `dof.py`, which implements the Kutzbach formula for spatial mechanisms.

```python
# dof.py
import json
import sys

DOF_PER_JOINT = {
    "fixed": 0,
    "revolute": 1,
    "slider": 1,
    "cylindrical": 2,
    "planar": 3,
    "ball": 3,
}


def degrees_of_freedom(links, joints):
    """Kutzbach count for a spatial mechanism. links includes the ground."""
    n = len(links)
    j = len(joints)
    sum_f = 0
    for joint in joints:
        if joint["type"] not in DOF_PER_JOINT:
            raise ValueError(f"unknown joint type: {joint['type']}")
        sum_f += DOF_PER_JOINT[joint["type"]]
    return 6 * (n - 1 - j) + sum_f


def main(path):
    with open(path) as f:
        data = json.load(f)
    m = degrees_of_freedom(data["links"], data["joints"])
    print(f"degrees of freedom: {m}")
    return m


if __name__ == "__main__":
    main(sys.argv[1])
```

Note that the ball joint and the planar joint counts in this table are for spatial joints. For the motor unit you need only the fixed and revolute entries, and the table is there so the script can handle a robot arm later.

**The fits table.** Write `fits.csv` with the nominal size and the limits of each fit, in millimetres. Use values from the ISO 286 tables or from the datasheet of the bearing, and cite the source in the README.

```
name,nominal,hole_min,hole_max,shaft_min,shaft_max
hub_bore_on_motor_shaft,6,6.000,6.030,5.970,6.000
bearing_seat_608,22,22.000,22.021,21.992,22.000
```

**Starter code.** Write `fit_check.py`. It reads each row, computes the smallest and largest clearance, and classifies the fit.

```python
# fit_check.py
import csv
import sys


def classify(min_clear, max_clear):
    if min_clear > 0:
        return "clearance"
    if max_clear < 0:
        return "interference"
    return "transition"


def check(path):
    results = []
    with open(path, newline="") as f:
        for row in csv.DictReader(f):
            hole_min, hole_max = float(row["hole_min"]), float(row["hole_max"])
            shaft_min, shaft_max = float(row["shaft_min"]), float(row["shaft_max"])
            min_clear = hole_min - shaft_max
            max_clear = hole_max - shaft_min
            results.append((row["name"], min_clear, max_clear, classify(min_clear, max_clear)))
    return results


if __name__ == "__main__":
    for name, lo, hi, kind in check(sys.argv[1]):
        print(f"{name}: clearance {lo:+.3f} to {hi:+.3f} mm, {kind}")
```

**Step-by-step tasks.**

1. Create the repository and the virtual environment. Install pytest. Copy the part STEP files from Day 37 and Day 38 into `cad/parts/`.
2. Build the motor unit in FreeCAD 1.0. Insert the bracket as the grounded part, the motor with a fixed joint, and the hub with a revolute joint. Save the document as `cad/motor_unit.FCStd` and export `cad/motor_unit.step`.
3. Write `assembly.json` for the unit, and run `python dof.py assembly.json`. The result must be 1. If it is not, find the joint that is wrong, and fix the model or the file.
4. Write `tests/test_dof.py` with three tests: the motor unit gives 1, the same unit with a fixed hub gives 0, and an unconnected hub gives 6.
5. Measure the bore of a printed hub and the shaft of a motor with the caliper, and compare them with the values in `fits.csv`. Update the file if the real values are different, and note the change in the README.
6. Run `python fit_check.py fits.csv`. With the values in the file, both fits classify as transitions, because their minimum clearance is exactly zero. Change the hub bore to 6.030 to 6.060 mm, and the shaft to 5.950 to 5.980 mm, and the hub fit becomes a clearance. Write `tests/test_fit_check.py` with a test for each classification.
7. Write `bom.md` with one row per unique part, its quantity, its part number and its source. Include the fasteners and the bearings.
8. Export an exploded view from the assembly tool, or draw one, and save it as `renders/exploded.png`.
9. Write the README. It must explain the Kutzbach count for the unit, the fit table with its sources, the BOM and the exploded view. It must also state the assumption that the motor is simplified, and say what that assumption changes.

**README checklist:**

- [ ] FreeCAD version and the licence you used
- [ ] The Kutzbach count for the unit, with N, J and Σf written out
- [ ] The fit table, with the source of each tolerance value
- [ ] The BOM, with the quantities for one motor unit and for the full robot
- [ ] The exploded view and a note on the assumptions

**Done when:**

- `dof.py assembly.json` prints 1, and `pytest` passes
- `fit_check.py fits.csv` prints the classification of each fit, and each classification matches your hand calculation
- The FreeCAD assembly has no unresolved interference, or each interference is explained as intended
- The BOM lists every part, including the bolts and nuts, with its quantity
- The README states one fit you would change for a printed housing, with the reason

**What to commit to GitHub:** the assembly file and the exported STEP files, `assembly.json`, `fits.csv`, the two scripts, the tests, the BOM, the exploded view and the README. Do not commit FreeCAD backup files or cache folders.

**Why this project is full and not light:** the unit is a sub-assembly of the robot, and the checks are the same design reviews that a production team runs before cutting metal.

---

## Check your understanding

1. An assembly has four links, including the ground, and three joints: two revolute and one fixed. Compute the number of degrees of freedom, and state whether the mechanism moves as intended if the two revolute joints are on the same axis.

2. A hub bore is designed as 6.00 mm and the shaft as 6.05 mm. Classify the fit, and say what would happen if the hub were printed 0.2 mm smaller than the design.

3. A designer uses the Kutzbach formula and gets M = 0 for a wheel that must rotate. Explain two likely mistakes in the joint list.

4. Explain why an interference check that passes in CAD does not guarantee that the parts will assemble on the printer.

5. A team has forty parts in one flat assembly, and a reviewer cannot find the bearing seat. Propose a hierarchy for the robot and explain how it helps the review.

6. A printed housing for a bearing is designed with a tight metal fit, and it cracks after a week in a warm room. Explain the likely cause and state the change to the fit.

**Answers:**

1. With N = 4, J = 3, and Σf = 1 + 1 + 0 = 2, M = 6 (4 − 1 − 3) + 2 = 2. The mechanism has two degrees of freedom, which are the two rotations. If the two revolute joints are on the same axis, the two rotations are the same motion, and the mechanism moves with one effective degree of freedom. The count does not show this, so you must check the axes by hand or in the CAD tool.

2. The shaft is 0.05 mm larger than the bore, so the fit is an interference of 0.05 mm, which means the hub must be pressed on. If the hub is printed 0.2 mm smaller, the interference grows to 0.25 mm, and the hub will crack or refuse to fit. The printed part must be designed with a larger bore, or the shaft must be smaller.

3. The most likely mistakes are that the wheel's revolute joint is missing, so the hub is fixed to the chassis, or that the count omits a link. A third possible mistake is that the joint type is fixed instead of revolute. Write the list of links and joints and check each against the motion the wheel needs.

4. The CAD model uses nominal dimensions and does not include the printing error, the shrink of the material or the tolerance of the printer. A bore that clears in CAD can bind on the printer. A test print and a caliper measurement are needed to check the real fit.

5. A hierarchy could group the robot into the drivetrain, the motor units, the electronics bay, the sensor mounts and the chassis, with each sub-assembly containing its own parts. The reviewer then looks for the bearing seat inside the motor unit, and checks only the interface between the unit and the chassis. The review is faster and each change is contained.

6. The likely cause is that the printed plastic expanded with the warm temperature, and the tight fit pressed the bearing outer ring against the housing, which put a hoop stress on the plastic. The change is to use a looser clearance fit, such as H7 or a larger bore, and to test the fit with a printed sample before printing the full housing.

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Engineering drawing", "Bill of materials", "Exploded-view drawing", "Interference fit", "Clearance (engineering)", "Tolerance (engineering)", "Degrees of freedom (mechanics)", "Kutzbach criterion" and "Mechanical joint"
- *ISO*: the standard ISO 286-1 and ISO 286-2, on the ISO code system for tolerances on linear sizes. Search for its title on iso.org.
- *ASME*: the standard ASME Y14.5 on geometric dimensioning and tolerancing, which you use in Day 41 for the drawings. Search for its title on asme.org.
- *FreeCAD*: the Assembly workbench documentation at wiki.freecad.org, for joints and the exploded view

**Tutorials**

- *GeeksforGeeks*: search for "Degree of freedom" in mechanics for worked examples
- *Python documentation*: the tutorial on python.org, for the scripts and the tests

**Watch**

- *MIT OpenCourseWare*: the lectures for course 2.007 Design and Manufacturing I cover fits, tolerances and assemblies. Search for the course on ocw.mit.edu.
- *Autodesk Fusion*: the Autodesk YouTube channel has short videos on joints and the assembly workflow. Search for "joints" on the channel.

**Practice**

- *Onshape Learn*: the free tutorials at learn.onshape.com include lessons on mates and assembly checks. Build the same unit in Onshape and compare the DOF count.

---

## Tomorrow

Tomorrow, on Day 40, you design the parts that make the robot work: motor brackets, sensor mounts, wheel hubs, chassis parts, servo mechanisms and enclosures. You apply the fits from today to the parts that carry the load, and the factor of safety from Day 34 to the parts that hold the electronics. The unit you checked today is the first of these components, and its interfaces set the dimensions of the rest.
