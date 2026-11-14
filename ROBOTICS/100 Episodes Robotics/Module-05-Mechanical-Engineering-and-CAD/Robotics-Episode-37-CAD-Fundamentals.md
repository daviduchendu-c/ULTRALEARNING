# Day 37: CAD Fundamentals

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 4 hours
**Level:** beginner. You need the mechanics from Day 34 and the drivetrain sizes from Day 36, because the first model you build is a part that must fit those numbers. You do not need prior CAD experience. You do need to be comfortable with a mouse, keyboard shortcuts and a file browser.

**By the end of this day, you will have:**

- A fully constrained 2D sketch of a motor bracket, with every dimension driven from a single spreadsheet
- A parametric 3D part built from that sketch, with a pad, a pocket and a hole pattern, that rebuilds correctly when one parameter changes
- An exported STEP file and an STL file of the part, with the bounding box checked against the spreadsheet
- A short written note on the design intent of the part, and on the three tests you used to confirm it

**What you need:**

- FreeCAD 1.0 (the stable 1.0 release line, with the latest 1.0.x patch), free and open source under the LGPL, from freecad.org
- Optional alternative: Onshape, which has a free plan. Its free plan stores documents publicly, which you must accept before you use it for private work. Onshape Education licences are free for students with a school email.
- Optional alternative: Autodesk Fusion. Its Personal licence is free for non-commercial use, and Autodesk offers free educational licences. Check the current terms before you choose it, because they change.
- Trimesh 4 or newer, installed with `pip install trimesh` (for the bounding box check)
- Python 3.12 with pytest 8 or newer
- A digital caliper, reading to 0.01 mm, from Day 34
- Optional: a 3D printer, or access to a makerspace, for printing the bracket in PLA or PETG

**Prerequisites:**

- Day 34: stress, the second moment of area, and the factor of safety, for choosing a bracket thickness
- Day 36: the wheel and motor sizes of the drivetrain you chose
- Basic understanding of the three orthographic views of a part
- A working text editor, and Git installed with a GitHub account

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How CAD developed](#how-cad-developed)
- [Glossary](#glossary)
- [Sketches and profiles](#sketches-and-profiles)
- [Constraints](#constraints)
- [Dimensions and parameters](#dimensions-and-parameters)
- [Reference geometry](#reference-geometry)
- [Parametric modeling](#parametric-modeling)
- [Design intent](#design-intent)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a parametric motor bracket](#project-a-parametric-motor-bracket)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Computer-aided design, or CAD, is the tool that turns a sketch of a robot part into a shape that a machine can cut, print or cast. A beginner usually opens a CAD tool, draws a box and pulls it into a solid, and that works until the first change. The first time the motor is moved 5 mm, or the bracket must be thicker, the box has to be redrawn by hand. A competent CAD user builds the model so that one change propagates correctly through the whole part.

Today you learn the core of that method. You will draw a sketch that is fully constrained, meaning every line and point is pinned by a rule rather than by where your mouse happened to land. You will drive every dimension from a spreadsheet, build a three-dimensional part from the sketch, and export it. The motor bracket you make here is the first real part of the robot, and the same method carries through the assemblies on Day 39 and the chassis on Day 42.

---

## How CAD developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1950s | The Whirlwind computer at MIT is used for early interactive graphics, and the Sketchpad program by Ivan Sutherland (1963) introduces interactive constraints. | Sketchpad is the origin of the constraint-based sketch that FreeCAD and Onshape still use. |
| 1960s to 1970s | Aircraft and car makers develop their own in-house CAD systems, and the first solid modelling methods are developed. | Solid modelling is the base of every CAD tool now. |
| 1980s | Parametric modelling appears in commercial systems, where a change to one dimension updates the whole part. | Parametric design is the feature that makes a bracket easy to change. |
| 1990s | STEP (ISO 10303) becomes the main neutral format for exchanging 3D models between CAD systems. | You will export STEP files from FreeCAD and send them to a manufacturer. |
| 2000s to today | Free and low-cost tools such as FreeCAD, Onshape and Fusion put parametric CAD on laptops and phones. | A beginner can now produce manufacturing-grade parts without a licence fee. |

The thread through this history is constraints. A sketch that remembers its rules can be changed, and a sketch that does not remember its rules must be rebuilt.

---

## Glossary

- **CAD (computer-aided design):** software that creates two- and three-dimensional models of physical objects.
- **Sketch:** a two-dimensional drawing of lines, arcs and circles on a plane, used as the basis for a three-dimensional feature.
- **Profile:** a closed loop in a sketch that defines a region, which can be padded, pocketed or revolved.
- **Constraint:** a geometric rule between sketch elements, such as coincident, horizontal, parallel or equal.
- **Dimension:** a numerical value that fixes the size of a sketch element, such as a length, an angle or a radius.
- **Fully constrained:** a sketch in which every point and line is fixed by constraints, so it has zero remaining degrees of freedom.
- **Degree of freedom (DoF):** an independent way a sketch element can move while it still satisfies its constraints.
- **Parameter:** a named value, such as a bracket thickness, that drives one or more dimensions.
- **Parametric model:** a model in which the features are defined by rules and parameters, so that changing a parameter rebuilds the geometry.
- **Feature:** a single operation in a CAD history, such as a pad, a pocket, a hole or a fillet.
- **Pad:** a feature that extrudes a sketch profile into a solid.
- **Pocket:** a feature that removes material by extruding a sketch profile into the existing solid.
- **Reference geometry:** planes, axes and points that are not part of the final shape, but that position features.
- **Design intent:** the reason a feature is built the way it is, such as "this hole must stay centred on the motor shaft".
- **STEP file:** a neutral, exchangeable CAD format defined by ISO 10303, which stores the exact geometry of a part.
- **STL file:** a mesh format that describes a surface as a set of triangles, used for 3D printing.

---

### Sketches and profiles

A sketch is the drawing that a feature is built from. Think of the sketch as a stencil. The stencil has a shape, and the pad or pocket pushes that shape through the material. Every profile must be a closed loop, which means its lines meet at their ends. A profile with a gap cannot become a solid, and this is the first error that beginners meet.

The sketch lives on a plane. In FreeCAD, the XY, XZ and YZ planes are the default planes, and you can attach a sketch to any face of a solid. Choose the plane first, because the plane sets the direction of the pad. A sketch on the XY plane pads upward along Z, and a sketch on the XZ plane pads along Y.

For the bracket in this lesson, you will draw a rectangle for the plate, with a rounded or chamfered corner, and a circle for each hole. The first principle is that the sketch is a two-dimensional model, and every line in it must be placed deliberately.

The common mistake is to draw a profile by eye and to leave small gaps or overlaps. FreeCAD reports an open wire, and the pad fails. The fix is to close the loop with coincident constraints, which are described in the next section, rather than to draw longer lines.

On the job, sketches are the first thing a CAD reviewer looks at. A sketch that is clean, fully constrained and easy to read saves time for everyone who changes the part later.

---

### Constraints

A sketch of a rectangle drawn by hand has four lines, but its corners and sides are free to move. A rectangle with its sides horizontal and vertical, and its corners coincident with the sides, is a constrained rectangle. Constraints are the rules that make the sketch stable.

The common constraints are:

- **Coincident:** two points are at the same location.
- **Horizontal and vertical:** a line is parallel to an axis.
- **Parallel and perpendicular:** two lines have a fixed angle between them.
- **Equal:** two lines or circles have the same length or radius.
- **Tangent:** a line and an arc meet smoothly.
- **Symmetric:** two points are mirror images about a line.
- **Distance and angle:** a fixed size between two points or two lines.

Every sketch element has a fixed number of degrees of freedom. A point in two dimensions has two, and a line has four. Each constraint removes one or more. A line with two points, four degrees of freedom, is fixed when you add a horizontal constraint (one), a length dimension (one), and positions for both ends (two). A sketch that is fully constrained has zero degrees of freedom left.

Here is a worked example for the bracket plate. Four separate lines have 16 degrees of freedom in total, four per line. Four coincident constraints join the corners and remove 8, leaving 8. Two horizontal and two vertical constraints remove 4 more, leaving 4. The width and the height are two dimensions, which leave 2. The last two degrees of freedom are the position of the rectangle in the plane, and two dimensions from the origin to one corner, one in X and one in Y, remove them. The rectangle is fully constrained with 12 constraints, and the status bar shows zero remaining degrees of freedom.

The common mistake is to add constraints that conflict. The sketch then turns red or reports a redundant constraint. Read the message carefully, because the conflicting constraint is usually one you added by habit, such as a horizontal line that is already horizontal from another constraint.

Constraints are the reason a sketch can be changed without breaking. A fully constrained sketch moves as a whole when you change a dimension, and an under-constrained sketch lets a stray drag move a corner by accident.

---

### Dimensions and parameters

A dimension fixes the size of a sketch element with a number. A parameter is a name for that number, which lets you change many dimensions from one place. In FreeCAD, the spreadsheet workbench stores named parameters, and every dimension can refer to a cell with an expression. The same idea exists in Onshape as variables, and in Fusion as user parameters.

Use parameters for the values that you might change: the bracket thickness, the motor hole spacing, the plate width and the hole diameter. Do not parameterize every line, because that makes the model hard to read. The rule is that a parameter should correspond to a decision you might revisit, not to a number that happens to be on the drawing.

Here is a worked example. The motor is a 28 mm square motor with four M3 mounting holes on a 22 mm square pattern. Put the hole spacing in a spreadsheet cell named `hole_spacing` with the value 22 mm. The hole circle centres are at ±hole_spacing/2 from the centre. The bracket thickness, `t`, is 3 mm from the Day 34 calculation. The plate width is `w` = 40 mm. When you change the motor to a different model with a 25 mm hole pattern, you change one cell, and the holes move with it.

The common mistake is to type the same number in twenty dimensions. When the value changes, only some of them are updated, and the part becomes inconsistent. A second mistake is to use a parameter with no unit, which makes the expression ambiguous. Always write the unit in the spreadsheet, such as "22 mm".

Parameters are also how a robot team shares a design. The spreadsheet tells the next student which numbers to change for a different motor, without asking them to redraw the bracket.

---

### Reference geometry

A reference plane is a plane that is not part of the final part, but that you use to place features. A reference axis is a line that you use to align holes or revolved features. Reference geometry is invisible in the finished part, and it is essential when a feature must sit at a location that is not on a face of the existing solid.

Reference geometry is useful when a bracket must sit at 90 degrees to the plate, or when a hole must line up with the centre of a motor shaft. You create a plane offset from the XY plane by the bracket height, or an axis through the centre of the motor shaft, and you attach the sketch to it.

Here is the worked example. The motor's shaft axis must sit 15 mm above the plate surface and run parallel to it. A reference plane offset 15 mm from the plate, with the offset set by a spreadsheet alias, gives a place for a sketch that centres the shaft hole. Changing the alias moves the reference plane, and the shaft hole stays on the shaft axis.

The common mistake is to draw the shaft hole on a face of the plate and then to move the plate. The hole stays where the face was, which is not where the shaft is. Attach the sketch to reference geometry that is defined by the design intent, not to a face that may move.

Reference geometry shows up in every assembly and every robot mount, because the mounts must be aligned to a shaft, a sensor or a wheel axis.

---

### Parametric modeling

Parametric modelling is the method that records how a part is made as well as what it looks like. The feature history is a list of operations, such as a sketch, a pad, a pocket and a hole pattern, and each operation depends on the ones before it. When you change a parameter, the history rebuilds the part from the top.

The common workflow is short. Sketch a profile, constrain it, pad it to the thickness, sketch the holes on the face, pocket or hole them, and pattern the holes if they repeat. Each step is a feature in the tree, and each feature can be edited later.

Here is the worked example in order. The plate is a 40 mm by 30 mm rectangle, padded 3 mm. The motor hole is a 6 mm diameter circle at the centre of the reference plane, pocketed through the plate. The four M3 clearance holes, each 3.4 mm in diameter, are placed at ±11 mm in X and Y and pocketed through the plate. A pattern of the four holes uses the same sketch, so changing the hole spacing updates all four holes at once.

The common mistake is to build a part by applying features in the wrong order. A fillet applied before a hole can change the position of the hole's edge, and the rebuild may fail. Apply the simple features first, and the detail features last, and check the history after each change.

Parametric features make CAD useful for design changes, which are the usual change in robotics. A part that cannot be changed is a part that must be redrawn for every new motor, and that costs days in a project.

---

### Design intent

Design intent is the reason behind a choice in the model. A hole is 3.4 mm because an M3 bolt must pass through it with a clearance. A plate is 3 mm thick because the Day 34 calculation gives a factor of safety of about 8 for the expected load. A design intent is a sentence that explains a number.

Write the intent in the README, and where possible in the feature names. A feature called "M3 clearance holes, 3.4 mm, for M3 bolt" tells the next reader more than "Pocket 001". The intent also tells you which parameters can change and which must not.

Here is a worked example. Suppose the bracket is to be made from 3D printed PETG instead of aluminium. The intent of the thickness was a strength and stiffness margin for aluminium. PETG has a much lower elastic modulus than aluminium, roughly a few gigapascals against about 69 GPa, so the thickness must be rechecked with the Day 34 formulas, and the holes must be checked for the printed tolerance. The intent tells you which numbers to re-examine, and which, such as the M3 clearance, remain the same.

The common mistake is to change a dimension without checking whether its intent still holds. A student who lengthens the bracket arm to reach a further sensor may find that the bending stress has doubled, because for a fixed load the bending moment grows in proportion to the length. The intent sentence would have prompted the check.

Design intent is what a reviewer reads first, and it is what a manufacturer asks about when a part arrives with a question. Writing it down takes minutes and saves days of rework.

---

## Hands-on exercises

1. **Install and open a sample.** Install FreeCAD 1.0. Open the sketcher, draw a rectangle, and check the constraint count in the status bar. Add constraints until the sketch is fully constrained, and note the number of constraints you needed.

2. **Build a spreadsheet.** In a new FreeCAD document, add a Spreadsheet object. Create the aliases `t`, `w`, `h`, `hole_spacing` and `hole_d` with their values and units. Attach a dimension in the sketch to one of the aliases, and change the alias to confirm that the sketch updates.

3. **Break a constraint on purpose.** Remove one constraint from your fully constrained sketch and drag a corner. Observe what moves. Restore the constraint and confirm that the sketch returns to its original shape.

4. **Measure a real motor.** Measure the hole pattern on a motor you own, using the caliper. Record the spacing and the diameter of the shaft. Compare it with the datasheet value. The two numbers should agree to within 0.2 mm.

5. **Export a STEP file.** Export one part as STEP from FreeCAD and reopen it in a second tool, such as Onshape or Fusion, to check that the geometry is the same.

---

## Project: a parametric motor bracket

This project builds a motor bracket from a fully constrained sketch with a spreadsheet of parameters. You then export the part as STEP and STL, check its bounding box with a Python script, and write the design intent. It is a full mini-project, because the bracket is a real part that you will print or machine, and it is the first part you will check against the mechanics from Day 34.

**Goal:** a parametric bracket that holds a 28 mm square DC motor with four M3 holes on a 22 mm pattern, with a 6 mm shaft hole and a 3 mm plate. The bracket must rebuild correctly for a 25 mm pattern, and its bounding box must match its parameters.

**Repository name:** `100-days-robotics-day-37-cad-fundamentals`

**Repo layout:**

```
100-days-robotics-day-37-cad-fundamentals/
├── README.md
├── requirements.txt
├── cad/
│   ├── motor_bracket.FCStd
│   ├── motor_bracket.step
│   └── motor_bracket.stl
├── params.csv
├── check_bbox.py
└── renders/
    └── bracket-front.png
```

**Parameters.** Create `params.csv` with the design values. The check script reads this file, so keep the values in one place.

```
name,value,unit,intent
plate_w,40,mm,width of the bracket plate
plate_h,30,mm,height of the bracket plate
t,3,mm,plate thickness from Day 34 strength check
hole_spacing,22,mm,M3 hole pattern of the motor
hole_d,3.4,mm,M3 clearance holes
shaft_d,6,mm,motor shaft hole
```

**Starter code.** Write the check script that compares the STL bounding box with the parameters. Install trimesh first.

```python
# check_bbox.py
import csv
import sys
import trimesh

TOLERANCE_MM = 0.05


def load_params(path):
    params = {}
    with open(path, newline="") as f:
        for row in csv.DictReader(f):
            params[row["name"]] = float(row["value"])
    return params


def main(stl_path, params_path):
    params = load_params(params_path)
    mesh = trimesh.load(stl_path)
    extents = mesh.bounding_box.extents  # in the units of the STL, assumed mm
    expected = (params["plate_w"], params["plate_h"], params["t"])
    ok = True
    for axis, (got, want) in zip("XYZ", zip(extents, expected)):
        diff = abs(got - want)
        status = "OK" if diff <= TOLERANCE_MM else "FAIL"
        print(f"{axis}: expected {want:.3f} mm, got {got:.3f} mm, diff {diff:.3f} mm, {status}")
        ok = ok and diff <= TOLERANCE_MM
    return 0 if ok else 1


if __name__ == "__main__":
    sys.exit(main(sys.argv[1], sys.argv[2]))
```

The script assumes the plate lies flat, with the 40 mm side on X, the 30 mm side on Y and the thickness on Z. If you export the part in another orientation, change the expected tuple to match, and record the orientation in the README. The point of the check is that the overall size matches the parameters.

**Step-by-step tasks.**

1. Create the repository and the virtual environment. Install trimesh and pytest. Save `params.csv` in the repo.
2. In FreeCAD 1.0, create a new document. Add a Spreadsheet and enter the parameters from `params.csv` with their aliases.
3. Draw the plate sketch on the XY plane as a rectangle, and constrain it with horizontal and vertical lines, equal opposite sides, and dimensions linked to `plate_w` and `plate_h`. Make the sketch fully constrained, and confirm the zero degrees of freedom in the status bar.
4. Pad the plate sketch to the thickness `t`.
5. Draw the motor hole sketch on the top face. Place a circle of diameter `shaft_d` at the centre, and four circles of diameter `hole_d` at ±`hole_spacing`/2 in X and Y. Use the spreadsheet aliases in the dimensions.
6. Pocket or hole the motor hole sketch through the plate. Use the hole feature if your version offers it, or a pocket through all if not.
7. Create a rectangular pattern or a second sketch of the four clearance holes, so that the pattern follows the spreadsheet.
8. Test the parametric behaviour. Change `hole_spacing` from 22 mm to 25 mm in the spreadsheet, confirm that all four holes move, and change it back.
9. Export the part as `cad/motor_bracket.step` and `cad/motor_bracket.stl`. Save the FreeCAD document as `cad/motor_bracket.FCStd`.
10. Run `python check_bbox.py cad/motor_bracket.stl params.csv` and record the output. Fix any mismatch, and note the orientation you used.
11. Render a front view from FreeCAD or from a screenshot, and save it to `renders/bracket-front.png`.
12. Write the README. Include the design intent for each feature, the constraint count of the sketch, the check output, and a short note on what would change if the bracket were printed in PETG.

**README checklist:**

- [ ] FreeCAD version and the licence you used
- [ ] A table of the parameters, their values and their design intent
- [ ] The degrees-of-freedom result for each sketch
- [ ] The check output from `check_bbox.py`
- [ ] A note on the 25 mm test, with the screenshot or the result

**Done when:**

- The plate sketch and the hole sketches are fully constrained, with zero degrees of freedom in FreeCAD
- Changing `hole_spacing` in the spreadsheet moves all four holes without redrawing any feature
- `check_bbox.py` prints OK for all three axes
- The STEP file opens in a second CAD tool with the same geometry
- The README explains the design intent of every parameter and names one parameter you would change for a different motor

**What to commit to GitHub:** the FreeCAD file, the STEP and STL exports, `params.csv`, `check_bbox.py`, the check output as a text file, the render, and the README. Do not commit the FreeCAD backup files (`.FCBak`) or the generated cache folders. Add a `.gitignore` that excludes them.

**Why this project is full and not light:** the bracket is a real part, and the same parametric method is used for every part in the robot.

---

## Check your understanding

1. A line in a sketch is constrained to be horizontal and given a length dimension, but it can still slide. Count the remaining degrees of freedom and state the constraints that remove them.

2. You change the bracket thickness from 3 mm to 2 mm in the spreadsheet, and the pocketed holes now fail to rebuild. Explain two possible causes and how you would find which one applies.

3. A student builds the motor hole on the top face of a plate, then changes the plate height from 30 mm to 40 mm. The motor shaft is now off-centre. Explain the design-intent error and the change that prevents it.

4. Explain why a STEP file is more useful than an STL file for sending a part to a machine shop.

5. A design review asks for the design intent of the 3.4 mm holes. Write a one-sentence answer that gives the reason for the number and the consequence of changing it.

6. Explain why a fully constrained sketch can still produce a bad part.

**Answers:**

1. A line has four degrees of freedom. The horizontal constraint removes one, and the length dimension removes one. Two remain: the line can slide up and down, and one endpoint can slide along the line direction. A distance from the origin to one endpoint in X and in Y removes both, which fixes the line completely.

2. A thinner plate may have a pocket that now cuts through the plate at a point where the depth is zero or negative, which makes the feature fail. Alternatively, a fillet or a chamfer on the edge may now be larger than the thickness of the plate, which also fails. Check the feature history, identify the first feature that fails, and compare its depth or radius with the new thickness.

3. The intent of the shaft hole is that it stays centred on the motor shaft, but the student attached the sketch to a face whose position depends on the plate height. The fix is to attach the sketch to a reference plane that is defined by the shaft height, for example an offset plane whose value comes from a spreadsheet alias.

4. A STEP file stores the exact geometry, with surfaces and curves, so the machine shop can read the true shape. An STL file is a triangle mesh, which approximates the curved surfaces and is used for printing, not for machining.

5. The 3.4 mm diameter gives a clearance for an M3 bolt, so the bolt passes through the plate without binding. Changing the diameter to a value near 3 mm would make the bolt bind, and changing it to a much larger value would weaken the joint and allow the bolt to move.

6. A fully constrained sketch can still have the wrong dimensions, wrong constraints or the wrong plane. Constraints guarantee that the sketch is stable, not that it is correct. A reviewer must check the dimensions against the design intent, and the feature order and the plane of each sketch.

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Computer-aided design", "Parametric design", "Solid modeling", "STEP (ISO 10303)", "STL (file format)", "Degrees of freedom (mechanics)" and "Sketchpad"
- *FreeCAD*: the official documentation at wiki.freecad.org, especially the pages for the Sketcher, PartDesign and Spreadsheet workbenches
- *Onshape*: the Onshape Learn pages at learn.onshape.com, for the constraint and sketch tutorials

**Tutorials**

- *GeeksforGeeks*: search for "CAD" and "Computer aided design" for an introduction to the terms
- *W3Schools*: the Python tutorial on w3schools.com for the check script

**Watch**

- *FreeCAD*: the official FreeCAD YouTube channel has beginner tutorials for the Sketcher and PartDesign workbenches. Search for "Sketcher basics" on the channel.
- *Autodesk Fusion*: the Autodesk YouTube channel has beginner tutorials on parametric modelling. Search for "parametric modeling basics".

**Practice**

- *Onshape Learn*: the free sketch tutorials at learn.onshape.com are a good second tool to check your FreeCAD sketches. Build the same bracket in both and compare the constraint counts.

---

## Tomorrow

Tomorrow, on Day 38, you move from sketches to three-dimensional features. You learn extrusions, revolutions, patterns, holes, fillets and chamfers, and you build a robot part with them. The bracket you made today is a pad and a pocket, and Day 38 adds the features that make a part strong and easy to manufacture, such as fillets at the root of a bend and a chamfer on a hole.
