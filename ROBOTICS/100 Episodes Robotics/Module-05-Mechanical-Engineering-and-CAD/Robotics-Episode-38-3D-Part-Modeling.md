# Day 38: 3D Part Modeling

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 4 hours
**Level:** beginner to intermediate. You need the sketch and parameter method from Day 37, because every feature today is built from a sketch and driven by a spreadsheet. You also need the stress ideas from Day 34, since fillets and chamfers exist to control stress and manufacturing.

**By the end of this day, you will have:**

- A three-dimensional wheel hub made with a revolution, an extrusion, a hole pattern, a fillet and a chamfer, with every feature driven by a named parameter
- A written feature history that lists each feature, its purpose and the parameter that drives it
- A mass estimate for the part, computed by hand from its volume and checked against the CAD mass property
- An STL export that you slice and print, or a documented reason why you did not print it

**What you need:**

- FreeCAD 1.0.x with the PartDesign and Spreadsheet workbenches, free and open source from freecad.org. Onshape (free plan) or Autodesk Fusion (Personal licence for non-commercial use) work too, and the feature names differ.
- Python 3.12 with trimesh 4 or newer, for an optional check of the exported mesh volume
- A digital caliper, reading to 0.01 mm
- Optional: a 3D printer, or access to a makerspace, for printing the hub in PLA or PETG. A basic desktop printer costs about $200 to $300. A slicer such as PrusaSlicer 2.8 or newer, or UltiMaker Cura 5.x, is free.
- Optional: a small DC gearmotor with a 3 mm or 6 mm shaft, to test the fit

**Prerequisites:**

- Day 37: sketches, constraints, spreadsheet parameters and the pad and pocket features
- Day 34: volume, mass, stress, and the idea of a stress concentration
- Day 35: the bearing and shaft sizes that the hub must fit
- Basic understanding of what an STL file is

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How 3D modeling developed](#how-3d-modeling-developed)
- [Glossary](#glossary)
- [Extrusions](#extrusions)
- [Revolutions](#revolutions)
- [Patterns](#patterns)
- [Holes](#holes)
- [Fillets](#fillets)
- [Chamfers](#chamfers)
- [Mass properties](#mass-properties)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a parametric wheel hub](#project-a-parametric-wheel-hub)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Almost every robot part is built from a small set of features applied in a sensible order. A bracket is an extrusion with holes. A wheel is a revolution with a bore and a pattern of lightening holes. A housing is an extrusion with fillets at the corners and chamfers at the edges that a hand must touch. The skill is not knowing hundreds of commands. It is knowing which feature builds which shape, and in which order to apply them so that the model rebuilds cleanly when you change it.

Today you build a wheel hub from five feature types: a revolution for the body, an extrusion for the flange, a polar pattern of holes, a fillet on the outside edge and a chamfer on the bore. You will also compute the volume and mass by hand, so you can check the CAD result against your own arithmetic. The hub is a useful test part, because it is round, it has a bore that must fit a shaft, and it must be light enough for a small motor to spin.

---

## How 3D modeling developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1960s to 1970s | Solid modelling research at universities, including the Build system developed at the University of Cambridge, and the first constructive solid geometry methods. | Constructive solid geometry is the idea behind the boolean operations that CAD still uses. |
| 1980s | Commercial CAD systems add parametric features, so that a dimension can drive a whole part. | The feature history you write today works because of this development. |
| 1990s | Boundary representation and feature-based modelling become standard in commercial systems. | Holes, pockets and fillets are first-class features, not hand-drawn shapes. |
| 2000s | Fused filament fabrication (FFF) 3D printers become cheap enough for makers, and STL becomes the everyday format for printing. | A CAD model is now often printed the same day it is designed. |
| Today | Free CAD tools support full parametric modelling, and simulation and mass properties are available in the same window. | The same hub can be designed, checked and printed on one laptop. |

The lesson is that the features did not change much. The tools now make them cheap to repeat and easy to change.

---

## Glossary

- **Feature:** a single operation in a CAD history, such as a revolution, a pad, a hole or a fillet.
- **Revolution:** a feature that rotates a closed sketch profile about an axis to make a solid of revolution.
- **Extrusion:** a feature that pushes a closed profile along a straight line to make a solid. A pad is an extrusion that adds material.
- **Pattern:** a feature that copies another feature in a line, a circle or a grid.
- **Polar pattern:** a pattern that copies a feature around a central axis at equal angles.
- **Hole feature:** a CAD feature that creates a drilled or tapped hole with standard dimensions, such as a clearance hole for an M3 bolt.
- **Fillet:** a rounded edge, which replaces a sharp edge with a circular arc of a given radius.
- **Chamfer:** a flat, angled edge, which replaces a sharp edge with a straight bevel.
- **Stress concentration:** a local increase in stress caused by a sharp corner, a hole or a change of section.
- **Stress concentration factor (Kt):** the ratio of the peak stress at a stress concentration to the nominal stress, which is often 2 or more for sharp corners.
- **Bore:** the hole through the centre of a wheel, hub or gear, which holds the shaft.
- **Flange:** a flat rim of a part that extends outward to hold bolts or other parts.
- **Solid of revolution:** a solid made by rotating a plane figure about an axis, such as a cylinder, a cone or a ring.
- **Pappus's centroid theorem:** a theorem that gives the volume of a solid of revolution as the area of the profile times the distance its centroid travels.
- **Mass properties:** the calculated volume, mass, centre of mass and inertia of a solid, which the CAD tool computes from its geometry and a material density.

---

### Extrusions

An extrusion is the simplest three-dimensional feature. You draw a closed profile, and the tool pushes it along a line to make a solid. Think of a cookie cutter pressed through dough: the shape of the cutter is the profile, and the depth of the dough is the extrusion length.

The first principle is that every point of the profile moves the same distance along the extrusion direction. The volume is the area of the profile times the length. For a rectangle of width w and height h extruded by a length t, the volume is w × h × t. This simple relation is the first check on any extruded part, and you should do it on paper before you rely on the CAD result.

Here is a worked example. The flange of the hub is a ring with an outer diameter of 40 mm and an inner diameter of 6 mm, extruded 3 mm. The area of the ring is π(20² − 3²) = π × 391 = 1228 mm². The volume is 1228 × 3 = 3685 mm³, or 3.69 cm³.

The common mistake is to extrude a profile that is not closed, or one that overlaps itself. The CAD tool rejects the profile or creates a solid with an error. Check the sketch for gaps and overlaps before you extrude, and use the constraint solver to close loops.

Extrusions are how most robot brackets, plates and housings are made. They are also how a 3D printed part gets its first layer of material.

---

### Revolutions

A revolution is an extrusion around an axis. Draw a half-section of the part in a plane that contains the axis, and the tool rotates it through an angle, usually 360 degrees, to make a solid. A wheel, a pulley, a shaft or a bushing is a revolution.

The first principle is that every point of the profile moves in a circle about the axis. The volume follows from Pappus's centroid theorem: V = A × 2π × r̄, where A is the area of the profile and r̄ is the distance from the axis to the centroid of the profile. This formula is the fastest check for a revolved part.

Here is the worked example for the hub body. The profile is a rectangle from radius 3 mm to radius 20 mm, and it is 8 mm tall. Its area is (20 − 3) × 8 = 136 mm². Its centroid is at radius (3 + 20) / 2 = 11.5 mm. The volume is 136 × 2π × 11.5 = 9827 mm³. The same result from the ring formula is π(20² − 3²) × 8 = 9827 mm³, which matches.

The common mistake is to sketch the profile on the wrong side of the axis, so that the revolution creates a shape with a hole through the wrong place, or fails completely. Keep the profile on one side of the axis, touching the axis only where the bore is, and check the result in an isometric view.

Revolutions appear in every wheel, pulley, bearing housing and motor shaft, and any part whose shape is round about an axis is a better candidate for a revolution than for an extrusion.

---

### Patterns

A pattern copies a feature. A hole drilled once and then copied four times is a pattern, and it is the reason a parametric model stays small. When the hole spacing changes, the pattern follows, and the four holes stay equally spaced.

The main types are the linear pattern, which copies along a line, the polar pattern, which copies around an axis, and the mirror, which copies across a plane. For a wheel, the polar pattern is the right choice, because the holes are at equal angles around the centre.

Here is a worked example. Four holes of diameter 3.4 mm sit on a pitch circle of diameter 16 mm, equally spaced around the centre. The pitch circle radius is 8 mm, and the holes are at 0, 90, 180 and 270 degrees. The centres are at (8, 0), (0, 8), (−8, 0) and (0, −8) mm. A polar pattern of one hole, with four instances at 360 / 4 = 90 degrees, gives the same result, and changing the pitch circle radius changes all four holes at once.

The common mistake is to pattern a feature that intersects another feature in a way that is not symmetric, such as a hole that cuts into a fillet. The pattern then fails on some copies. Build the simple pattern first, and add fillets after the pattern, so that the fillet follows the pattern cleanly.

Patterns are used everywhere that holes repeat: motor mounts, gear faces, wheel lightening holes and the vent holes of an enclosure.

---

### Holes

A hole is a cylindrical removal of material. In a CAD tool, a hole feature is a pocket with a standard diameter and a standard depth. It is usually more convenient than a sketch, because the tool knows the standard dimensions, such as the clearance hole for an M3 bolt or the tapped hole for an M3 thread.

The first principle is the clearance. A bolt of nominal diameter 3 mm passes through a clearance hole of about 3.2 to 3.4 mm, depending on the fit class. A tapped hole for an M3 bolt is drilled to about 2.5 mm, and the thread is cut by a tap or formed by the bolt. For a 3D printed part, the printed hole is usually smaller than the designed hole, so you must print a test and adjust.

Here is the worked example for the hub's set screw. The set screw is M3, and it presses on the shaft to hold the hub. A tapped hole for M3 in the hub wall is 2.5 mm in diameter, and it is placed at the centre of the flange thickness, 1.5 mm from the top face. A 2.5 mm hole in a 3 mm wall is thin, so the designer should add a boss or increase the wall. The design constraint here is the wall thickness, which matters more than the hole diameter.

The common mistake is to place a hole too close to an edge, so that the material between the hole and the edge is too thin to hold a bolt. A rule of thumb is to keep the edge distance at least equal to the hole diameter, and for a load-bearing bolt, to keep it larger.

Holes are the most common feature in mechanical parts, and their placement is one of the first details a manufacturer checks.

---

### Fillets

A fillet is a rounded edge that replaces a sharp corner. Fillets make a part easier to machine and easier to print, and they reduce stress at the corner, because a sharp inside corner acts as a stress concentration.

The first principle is that a fillet of radius r removes material from the corner and replaces it with an arc tangent to both faces. The stress at a sharp corner is theoretically infinite in an ideal elastic model, and the stress concentration factor depends on the radius. A larger radius lowers the peak stress. For a shoulder with a sharp inside corner, the factor can be 2 or 3, and a generous fillet can bring it down toward 1.5.

Here is a worked example for the outside edge of the hub flange. A fillet of radius 1 mm on the 8 mm tall outside edge removes a small amount of material. The removed area is (1 − π/4) × 1² = 0.215 mm² per unit length, and the total removed volume is 0.215 × 2π × 20 = 27 mm³, about 0.3 percent of the hub volume. The change in mass is negligible, and the edge is much easier to handle and print.

The common mistake is to choose a fillet radius that is larger than the face it rounds. The CAD tool fails, or it creates a self-intersecting surface. Another mistake is to apply a fillet after a pattern of holes that the fillet must avoid. Apply the fillet early or late, but check the rebuild after each change.

Fillets appear in every machined part, every moulded part and every printed part where a sharp corner would break or crack.

---

### Chamfers

A chamfer is a flat bevel that replaces a sharp edge. It works like a fillet, but the cut is straight rather than curved. Chamfers are easy to check on a drawing and easy to machine with a single cut. They also help a bolt or a shaft slide into a hole, because the chamfer guides the part.

The first principle is the size of the bevel. A chamfer of distance d on both faces removes a triangle with legs of length d, which has area d²/2 in the section. A 0.5 mm chamfer removes 0.125 mm² of section per unit length, which is tiny, but the guiding effect is large.

Here is a worked example for the bore of the hub. The bore is 6 mm in diameter, and a 0.5 mm chamfer on both ends lets a shaft enter without catching the edge. The chamfer is much smaller than the fillet on the outside, and it is placed where the shaft meets the hub, so its job is to guide assembly.

The common mistake is to chamfer an edge that is part of a thin feature, so that the chamfer removes most of the material. A chamfer larger than the wall thickness leaves no wall. Check each chamfer against the thickness of the adjacent face.

Chamfers appear on every shaft, bore and bolt hole that must be assembled by hand, and on the edges of printed parts that will be handled often.

---

### Mass properties

Mass properties are the volume, mass, centre of mass and moments of inertia of a solid. The CAD tool computes them from the geometry and a material density. The values matter for the robot, because the mass of each wheel affects the acceleration and the energy of the drive, and the centre of mass affects stability, as you saw on Day 34.

The first principle is mass equals volume times density. For PLA at about 1.24 g/cm³ and aluminium at 2.70 g/cm³, a 10 cm³ part weighs about 12.4 g in PLA and 27 g in aluminium. The density you use in the CAD tool must match the material, not a default value.

Here is the worked example for the hub. The body is 9.83 cm³ and the flange is 3.69 cm³. Both are rings with the same 6 mm bore, so the bore is already excluded from each. Together they give 13.52 cm³. The four 3.4 mm holes remove 4 × π × 1.7² × 3 = 109 mm³, or 0.11 cm³, which leaves about 13.4 cm³. In PLA, the mass is 13.4 × 1.24 = 16.6 g if the part is printed solid. Printed with 20 percent infill, the mass is closer to 8 to 10 g, but the strength falls, and the stress calculation from Day 34 must use the reduced stiffness of the infilled part.

The common mistake is to take the CAD mass as the printed mass. The printed mass depends on infill, wall count and support material, and the CAD mass is only for a solid part. Check the slicer's estimate for the printed mass, and state which value you use in the design review.

Mass properties are used in every robot design review, because the total mass drives the motor size, the battery size and the stability margin.

---

## Hands-on exercises

1. **Build the body.** Sketch the half-section of the hub body on the XZ plane, with the axis on the Z axis. Revolve the profile 360 degrees, and check its volume in the CAD tool. Compare it with 9827 mm³.

2. **Build the flange and the bore.** Add the flange as a pad, and cut the bore with a pocket or a hole. Check the volume of the flange and the bore against the values in this lesson.

3. **Pattern the holes.** Place one 3.4 mm hole on a 16 mm pitch circle and pattern it four times. Change the pitch circle to 20 mm in the spreadsheet and confirm that all four holes move.

4. **Test a fillet.** Add a 1 mm fillet to the outside edge. Then try a 3 mm fillet and note the point at which the rebuild fails. Explain why it fails.

5. **Measure a printed hole.** If you print the hub, measure the printed 3.4 mm hole with the caliper. Record the actual diameter, and compute the difference from the design value. Use the difference to set the hole size for the next print.

---

## Project: a parametric wheel hub

This project builds a wheel hub from five features, with every dimension driven by a spreadsheet. You export the hub as STL, check its volume with a Python script, and print it if you have a printer. It is a full mini-project, because the hub is a real part that must fit a shaft and carry a load.

**Goal:** a parametric wheel hub with an outer diameter of 40 mm, a 6 mm bore, a flange with four 3.4 mm holes on a 16 mm pitch circle, a 1 mm outside fillet and a 0.5 mm bore chamfer. The model must rebuild correctly when any parameter changes within a stated range.

**Repository name:** `100-days-robotics-day-38-3d-part-modeling`

**Repo layout:**

```
100-days-robotics-day-38-3d-part-modeling/
├── README.md
├── requirements.txt
├── cad/
│   ├── wheel_hub.FCStd
│   ├── wheel_hub.step
│   └── wheel_hub.stl
├── params.csv
├── check_volume.py
└── print/
    └── wheel_hub_print_notes.md
```

**Parameters.** Create `params.csv` with the design values. The check script reads it.

```
name,value,unit,intent
od,40,mm,outer diameter of the hub
bore_d,6,mm,bore to fit a 6 mm shaft
body_h,8,mm,height of the hub body
flange_t,3,mm,flange thickness
pcd,16,mm,pitch circle diameter for the four holes
hole_d,3.4,mm,M3 clearance holes
fillet_r,1,mm,outside edge fillet
chamfer_d,0.5,mm,bore chamfer
```

**Starter code.** Write `check_volume.py`. It computes the expected volume from the parameters and compares it with the STL volume, with a tolerance that allows for the fillet, the chamfer and the holes.

```python
# check_volume.py
import csv
import math
import sys
import trimesh


def load_params(path):
    with open(path, newline="") as f:
        return {row["name"]: float(row["value"]) for row in csv.DictReader(f)}


def expected_volume_mm3(p):
    # Body plus flange: one ring from bore to outer radius, total height body_h + flange_t.
    r_out = p["od"] / 2.0
    r_in = p["bore_d"] / 2.0
    total_h = p["body_h"] + p["flange_t"]
    body = math.pi * (r_out**2 - r_in**2) * total_h
    # Four clearance holes through the flange (each a cylinder of hole_d).
    holes = 4 * math.pi * (p["hole_d"] / 2.0) ** 2 * p["flange_t"]
    return body - holes


def main(stl_path, params_path, tolerance_pct=3.0):
    p = load_params(params_path)
    mesh = trimesh.load(stl_path)
    got = abs(mesh.volume)
    want = expected_volume_mm3(p)
    diff_pct = 100.0 * (got - want) / want
    status = "OK" if abs(diff_pct) <= tolerance_pct else "FAIL"
    print(f"expected {want:.1f} mm^3, got {got:.1f} mm^3, diff {diff_pct:+.2f} %, {status}")
    return 0 if status == "OK" else 1


if __name__ == "__main__":
    sys.exit(main(sys.argv[1], sys.argv[2]))
```

The starter formula is a first approximation. It treats the body and the flange as one ring of height `body_h` + `flange_t`, and subtracts the four holes through the flange, but it ignores the fillet and the chamfer, which remove small volumes. The tolerance of 3 percent covers them. Work out the fillet and chamfer volumes by hand, and state whether they are inside the tolerance. If they are not, improve the formula and record the change in the README.

**Step-by-step tasks.**

1. Create the repository and the virtual environment. Install trimesh and pytest. Save `params.csv`.
2. In FreeCAD 1.0, create a spreadsheet with the aliases from `params.csv`.
3. Sketch the half-section of the hub body in the XZ plane, with the axis on the Z axis. Use the aliases for the outer radius and the height. Revolve it 360 degrees.
4. Sketch the flange as a ring, and pad it at the top of the body by `flange_t`. Keep the flange sketch fully constrained.
5. Cut the bore with a pocket through all, or a hole feature, with the diameter `bore_d`.
6. Place one hole of diameter `hole_d` at a radius of `pcd / 2` on the flange, and pattern it four times with a polar pattern.
7. Add the outside fillet of radius `fillet_r` on the outside top edge of the flange. Check the rebuild.
8. Add the chamfer of `chamfer_d` on the bore ends.
9. Test the parametric behaviour. Change `pcd` from 16 mm to 20 mm and confirm that the holes move. Change `fillet_r` to 2 mm, and check the rebuild. Change them back.
10. Export `cad/wheel_hub.step` and `cad/wheel_hub.stl`, and save `cad/wheel_hub.FCStd`.
11. Run `python check_volume.py cad/wheel_hub.stl params.csv` and record the output. Improve the formula if the tolerance is exceeded for a reason you can explain.
12. Compute the printed mass estimate for PLA with 20 percent infill, using the density of 1.24 g/cm³ and the mass fraction you assume, and write it in `print/wheel_hub_print_notes.md`. If you print the part, record the slicer's estimate and the measured mass on a kitchen scale.
13. Write the README with the design intent for each feature, the feature history, the check output, and the measurement from the hands-on exercises.

**README checklist:**

- [ ] FreeCAD version, and the licence you used
- [ ] The feature history, with each feature, its purpose and its driving parameter
- [ ] The volume check output and the hand calculation for the body and flange
- [ ] A note on the fillet radius at which the rebuild fails, with the reason
- [ ] The printed mass, measured or estimated, with the infill setting

**Done when:**

- Every dimension of the hub is driven by a spreadsheet alias, with no typed numbers left in the sketches
- Changing `pcd` moves the four holes, and changing `fillet_r` updates the outside edge without a rebuild error in the stated range
- `check_volume.py` prints OK within the stated tolerance
- The README gives the feature history and the design intent for each feature
- You have stated the maximum fillet radius you would allow before the flange becomes too thin to print reliably, with a reason

**What to commit to GitHub:** the FreeCAD document, the STEP and STL files, `params.csv`, `check_volume.py`, the check output, the print notes, and a photo of the printed hub if you printed it. Add a `.gitignore` that excludes the FreeCAD backup files and cache folders.

**Why this project is full and not light:** the hub is a complete parametric part that you can print and test, and the checks you write are the first use of the volume reasoning that the robot design depends on.

---

## Check your understanding

1. A revolved profile has an area of 200 mm² and its centroid is 15 mm from the axis. Find the volume of the solid, and state the formula you used.

2. A designer places the centre of a 3.4 mm hole 3 mm from the edge of a plate and says it is strong enough for an M3 bolt. Explain the problem and give a rule of thumb for the minimum edge distance.

3. A fillet of 1 mm on a sharp inside corner of a bracket reduces the stress concentration factor from about 3 to about 2. A second designer says a 0.1 mm fillet gives the same benefit. Decide whether the second designer is right, and explain what the stress concentration depends on.

4. A polar pattern of four holes is placed on a pitch circle of 20 mm. The designer changes the pitch circle to 24 mm and the holes do not move. Explain the two most likely causes.

5. A chamfer of 2 mm is applied to a bore in a plate that is 3 mm thick. Explain what goes wrong and what chamfer size you would choose.

6. A student prints a hub at 20 percent infill and reports its mass as 16.6 g, which is the CAD solid mass for PLA. Explain why the value is wrong, and estimate a more realistic printed mass.

**Answers:**

1. By Pappus's theorem, V = A × 2π × r̄ = 200 × 2π × 15 = 18,850 mm³, or about 18.9 cm³.

2. The hole centre is 3 mm from the edge, and the hole radius is 1.7 mm, so only 1.3 mm of material remains between the hole and the edge. That is too thin to carry the bolt load, and the edge can tear or crack. A rule of thumb is to keep the edge distance at least equal to the hole diameter, and to make it larger for a loaded joint.

3. The second designer is wrong. The stress concentration depends on the radius relative to the size of the feature, and a 0.1 mm fillet is so small that the stress peak remains almost unchanged. The benefit comes from a radius that is a meaningful fraction of the part dimensions.

4. The most likely cause is that the hole feature was not patterned from the pitch circle sketch but was placed with its own constraint, or that the pitch circle dimension is not linked to the spreadsheet alias. A second possible cause is that the pattern refers to a stale feature, so it has not rebuilt. Check the dimension expression and the feature history.

5. A chamfer of 2 mm on a 3 mm plate removes almost the whole edge of the bore, leaving a thin ring of material that may crack. The chamfer must be a fraction of the wall thickness. A value of 0.5 mm or 0.3 mm is a sensible choice for a 3 mm plate.

6. The CAD mass is the mass of a solid part at 100 percent density. The printed part with 20 percent infill has less material, and the walls and the top and bottom layers hold most of the strength. A realistic printed mass for this hub is about 8 to 10 g, depending on the wall count and the solid layers, and the slicer estimate is the best value for the printed mass.

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Solid modeling", "Constructive solid geometry", "Solid of revolution", "Pappus's centroid theorem", "Fillet (mechanics)", "Chamfer", "Stress concentration", "Fused filament fabrication" and "STL (file format)"
- *FreeCAD*: the PartDesign documentation at wiki.freecad.org, for the Revolution, Pocket, Hole, Fillet, Chamfer and Polar Pattern tools
- *Onshape Learn*: the feature tutorials at learn.onshape.com, for revolutions and patterns

**Tutorials**

- *GeeksforGeeks*: search for "Solid of revolution" and "Stress concentration" for the basic formulas
- *W3Schools*: the Python tutorial on w3schools.com for the check script
- *PrusaSlicer documentation*: the knowledge base at help.prusa3d.com has guides on infill, walls and printing tolerances

**Watch**

- *FreeCAD*: the official FreeCAD YouTube channel has PartDesign tutorials. Search for "PartDesign revolution" on the channel.
- *Autodesk Fusion*: the Autodesk YouTube channel has short videos on fillets and chamfers. Search for "fillet and chamfer" on the channel.

**Practice**

- *Thingiverse*: search for "wheel hub" to compare your design with others, and to check your approach against published parts. Only download and print files whose licence allows it.

---

## Tomorrow

Tomorrow, on Day 39, you combine parts into an assembly. You learn to place parts with joints or mates, to count the degrees of freedom of the assembly, to check interference and clearances, and to organise the assembly hierarchy. The hub you built today is the first part you will mount on a motor shaft, and the bore chamfer and the fit you chose will decide whether the hub slides on or binds.
