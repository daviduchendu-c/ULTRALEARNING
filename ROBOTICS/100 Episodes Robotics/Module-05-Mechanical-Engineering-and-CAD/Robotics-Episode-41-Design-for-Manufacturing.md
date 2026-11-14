# Day 41: Design for Manufacturing

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 4 hours
**Level:** intermediate. You need the parts from Days 37 to 40, the fits from Day 39, and the stress and deflection calculations from Day 34. Today you check whether those parts can be made, with the process you chose and the tolerance you wrote down. A design that is correct on the screen can still be impossible to print, cut or assemble.

**By the end of this day, you will have:**

- A design-for-manufacturing (DFM) rule set for FDM printing, laser cutting, CNC machining and fasteners, with the numbers and the source of each rule
- A worked compensation for a printed hole and a laser-cut slot, based on a test piece you made
- A tolerance stack-up for a three-part joint, computed in worst case and in root-sum-square form
- A Python checker that reads a design description and reports the rules that a part breaks, with tests

**What you need:**

- Python 3.12 with pytest 8 or newer (`pip install pytest`)
- A 3D printer, or access to a makerspace, for a test coupon in PLA or PETG. A test coupon is a small printed plate with holes and slots of several sizes.
- A digital caliper, reading to 0.01 mm
- Optional: access to a laser cutter for a test coupon in 3 mm plywood or acrylic, and a CNC mill or a makerspace with one, to see a machined corner
- Optional: a set of M3 bolts, nuts and washers, about $5, to test the fit of printed holes
- The enclosure and bracket files from Day 37, Day 38 and Day 40

**Prerequisites:**

- Day 39: clearance and fits, the ISO 286 tolerance system, and interference checks
- Day 40: the enclosure, the bracket and the material choices
- Day 34: deflection and stress, for judging whether a thin wall is strong enough

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How manufacturing rules developed](#how-manufacturing-rules-developed)
- [Glossary](#glossary)
- [Tolerances and fits in practice](#tolerances-and-fits-in-practice)
- [3D printing rules](#3d-printing-rules)
- [Laser cutting and kerf](#laser-cutting-and-kerf)
- [CNC machining rules](#cnc-machining-rules)
- [Fasteners](#fasteners)
- [Materials and drawings](#materials-and-drawings)
- [Assembly and the stack-up of tolerances](#assembly-and-the-stack-up-of-tolerances)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a design-for-manufacturing checker](#project-a-design-for-manufacturing-checker)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A part is only as good as the process that makes it. A wall that is 0.4 mm thick looks fine in CAD and collapses on the printer. A slot cut to exactly the width of a tab is too tight after the laser has removed its kerf. A corner drawn sharp cannot be milled, because the cutter is round. These are common causes of a failed first prototype, and they are why professional teams review every part against its process before release.

Today you learn the rules for the four processes that a robot team uses most: FDM 3D printing, laser cutting, CNC machining and fastened assembly. You learn the numbers behind each rule and the test that confirms them on your own machine. You then write a small checker that applies the rules to a design file, and you use it on the enclosure and bracket from earlier in the module.

---

## How manufacturing rules developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| Late 1800s | Interchangeable parts and standard gauges become essential for mass production, and tolerance standards start to appear. | The idea of a part that fits any other part of the same size is the origin of every fit and tolerance you use. |
| 1900s | Tolerance and drawing standards such as ASME Y14.5 and the ISO limits and fits system are developed. | GD&T and ISO 286 are the languages that a manufacturer reads from your drawing. |
| 1950s to 1970s | Numerical control (NC) machines are developed, and a part must now be expressed as a program that a machine can follow. | A CNC part must be designed with the cutter in mind, not only the shape. |
| 1980s | Stereolithography, the first commercial 3D printing process, is developed by Chuck Hull, and FDM follows in the late 1980s. | Layer-based processes have their own rules for walls, overhangs and holes. |
| 2000s to today | Desktop FDM printers, laser cutters and small CNC machines reach makers, and open-source design rules are published by printer and service providers. | A beginner can now check a design against published rules before ordering. |

The history explains why the rules exist. Each process has its own limits, set by the tool, the material or the physics, and a design that ignores those limits is rejected or expensive to make.

---

## Glossary

- **Design for manufacturing (DFM):** the practice of designing parts so that they can be made efficiently and reliably with a chosen process.
- **Tolerance:** the permitted variation of a dimension from its nominal value.
- **Tolerance stack-up:** the accumulated variation of several dimensions in a chain, which determines the variation of the final gap or fit.
- **Root-sum-square (RSS):** a stack-up method that adds the squares of the tolerances and takes the square root, which gives a smaller result than the worst case when the errors are random.
- **FDM (fused deposition modeling):** a 3D printing process that extrudes melted thermoplastic layer by layer. Also called FFF (fused filament fabrication).
- **Perimeter (wall line):** a printed loop that forms the outside of a layer. A wall is made from one or more perimeters.
- **Overhang:** a part of a print that extends beyond the layer below it without support.
- **Kerf:** the width of material removed by a laser or saw cut.
- **Bend allowance:** the length of sheet along the neutral axis of a bend, used to compute the flat length of a sheet part.
- **K-factor:** the ratio that locates the neutral axis in a sheet bend, usually between 0.3 and 0.5.
- **Internal corner radius:** the radius of an inside corner in a machined pocket, which equals the radius of the cutter.
- **Tool radius:** the radius of a milling cutter, which limits the smallest internal corner.
- **Clearance hole:** a hole that is slightly larger than the bolt so that the bolt passes through freely.
- **Tapped hole:** a hole with an internal thread, cut by a tap or formed by the bolt.
- **General tolerance:** a default tolerance written on a drawing for dimensions that have no specific tolerance, such as ISO 2768.
- **Datum:** a reference feature that a dimension or tolerance is measured from, in GD&T.

---

### Tolerances and fits in practice

A tolerance is the permitted variation of a dimension. A hole written as 3.40 ± 0.05 mm may measure anywhere from 3.35 to 3.45 mm and still be accepted. The tolerance you write is a promise to the manufacturer, and it sets the cost. A tight tolerance on every dimension makes the part expensive, and a loose tolerance on a critical fit makes the assembly fail.

The first principle is to tolerance only what matters. A fit between a bearing and its seat matters, and the ISO 286 fit from Day 39 is the right tool. The outside size of a bracket does not usually matter, and a general tolerance such as ISO 2768 medium is enough. Writing a tight tolerance on a dimension that does not affect the fit is wasted money.

Here is a worked example. A hub bore must fit an 8 mm shaft with a light running clearance. The designer chooses a hole tolerance of +0.03 to 0 mm around 8.00 mm, which is a close clearance for a machined part, and a shaft tolerance of −0.01 to −0.03 mm. The clearance ranges from 0.01 to 0.06 mm. For a printed part, the designer knows the print will be undersized by about 0.1 to 0.2 mm, so the bore is designed larger and checked on the printer, as the rest of this lesson shows.

The common mistake is to copy a tolerance from a metal drawing onto a printed part. A printed hole can vary by a few tenths of a millimetre, far more than a machined hole, so the printed fit needs a looser allowance and a test.

Tolerances connect the CAD model to the parts a manufacturer makes. A drawing with tolerances on the right features tells the shop which dimensions matter.

---

### 3D printing rules

FDM printing builds a part from layers of melted plastic, and each layer is a set of perimeters and infill. The process has three rules that cover most failures: the minimum wall, the overhang angle and the hole size.

The first rule is the minimum wall. A wall must be at least as thick as the perimeters that fill it. A 0.4 mm nozzle lays perimeters about 0.4 mm wide, so a wall of 0.8 mm is two perimeters, and a wall of 1.2 mm is three. For a load-bearing wall, use at least 1.2 mm, and for a wall that holds a screw, use at least 2 mm around the hole. Those are rules of thumb from common practice, and your printer's settings may differ.

The second rule is the overhang. A printed layer needs material under it. A horizontal surface can print without support if it is supported by the layer below, and a surface that angles outward more than about 45 degrees from vertical usually needs support. Design the part so that the overhangs are small or lie on the print bed, or accept the support material and its cleanup.

The third rule is the hole. An FDM hole is round only if the printer is calibrated, and the hole is often undersized by a few tenths of a millimetre because the extruded plastic shrinks as it cools and the perimeters push inward. Design a clearance hole larger than the bolt by the amount your test piece shows.

Here is a worked example for the hole compensation. The design calls for a 3.4 mm clearance hole for an M3 bolt. The test coupon prints a 3.4 mm hole that measures 3.22 mm with the caliper. The compensation is 3.4 − 3.22 = +0.18 mm, so the designer changes the design value to 3.58 mm, which prints at about 3.40 mm. The new design value is checked with a bolt, which should pass freely.

The common mistake is to design a hole at exactly the bolt size, which prints too small, and then to force the bolt in and crack the plastic. The second mistake is to ignore the layer height and print orientation, which change the hole's roundness. A hole printed with its axis along Z is round, and a hole printed horizontally may be oval.

FDM rules apply to every printed part in a robot, including the brackets, the mounts and the enclosure from Day 40.

---

### Laser cutting and kerf

A laser cutter makes flat parts from sheet, such as acrylic, plywood or thin metal. The beam melts or burns the material along a path, and the path has a width called the kerf. The kerf is typically 0.1 to 0.3 mm, depending on the material, the lens and the power. The kerf matters because every cut removes material, and the cut edges are larger or smaller than the drawn path by half the kerf on each side.

The first principle is that an outer contour is cut half a kerf outside the drawn line, and an inner contour, such as a hole or a slot, is cut half a kerf inside. A hole drawn at 3.0 mm is cut at about 3.0 + kerf. A slot drawn at 3.0 mm is cut wider than 3.0 mm by the kerf. For a tab that must fit into a slot, the slot is drawn narrower than the tab by the kerf, to make a snug fit.

Here is a worked example. A 3 mm plywood tab must fit into a 3 mm slot with a press fit. The laser kerf measured on the test coupon is 0.15 mm. The slot is cut larger than the drawn path by the kerf, so the drawn slot must be 3.0 − 0.15 = 2.85 mm to produce a 3.0 mm slot after cutting. The tab is cut at its drawn width, reduced by the kerf, so its drawn width must be 3.0 + 0.15 = 3.15 mm to produce a 3.0 mm tab. The two cuts produce a tab and slot that fit together. The test coupon confirms the numbers before the full part is cut.

Laser cutters also burn, so the material near the cut is darker and may be weaker. Acrylic can crack at a sharp corner, and plywood needs the grain direction checked for strength.

The common mistake is to draw a tab and slot at the same nominal width, which gives a slot that is too loose when the kerf is applied. The second mistake is to forget that the material thickness changes the kerf and the focus, so a test coupon must match the sheet you cut.

Laser cutting is the usual process for chassis plates, panels and the flat brackets that Day 40 suggested.

---

### CNC machining rules

A CNC mill removes material with a rotating cutter. The cutter is round, so an inside corner in a machined pocket has a radius at least as large as the cutter. A sharp inside corner cannot be made by a rotating cutter, and a designer who draws one forces the shop to use a smaller cutter, a slower feed or an extra operation, all of which add cost.

The first principle is that the cutter sets the geometry. The internal corner radius must be at least the tool radius, and the depth of a pocket is limited by the length of the cutter, which is often about three to four times its diameter for a stable cut. For a hole, a standard drill has a maximum depth of about four to five times its diameter in a typical shop, and deeper holes need special tools.

Here is a worked example. A pocket for a motor is 30 mm wide and 20 mm deep, with inside corners. A 6 mm end mill has a radius of 3 mm, so the minimum internal corner radius is 3 mm. A corner drawn sharp must be redesigned with a 3 mm radius, or it must be cut by a wire EDM machine, which costs much more. A 6 mm mill can reach a depth of about 18 mm to 24 mm, so the 20 mm pocket is within the typical range.

Machining also has a tolerance limit. A typical shop holds about ±0.05 mm on a machined dimension, and tighter tolerances need careful setup and inspection. Your design should specify tight tolerances only on features that need them.

The common mistake is to design a pocket with sharp corners and deep narrow slots, which are easy in CAD and expensive in a shop. The second mistake is to forget the orientation of the part. A feature that can be reached from only one side needs an extra setup, which adds cost.

CNC machining is used for the aluminium parts of a robot, such as the bearing housings, the servo adaptors and the high-load brackets.

---

### Fasteners

A fastener joins two parts. In robotics, the most common fasteners are ISO metric bolts, usually M2, M2.5, M3, M4 and M5, with matching nuts and washers. The design rules for fasteners cover the clearance hole, the thread engagement and the edge distance, and they decide whether a joint holds.

The first principle is the clearance hole. For an M3 bolt, the ISO 273 table gives a fine clearance of 3.2 mm, a medium clearance of 3.4 mm and a coarse clearance of 3.6 mm. The medium clearance is the usual choice for makers, and the printed hole compensation from earlier in this lesson applies on top of it.

The second principle is the thread engagement. A bolt that screws into a tapped hole needs enough thread to carry the load. A common guideline for a steel bolt in aluminium is an engagement of about 1.5 times the bolt diameter, and for a bolt in plastic it is much longer, often 2 to 3 times the diameter, because the plastic thread strips more easily. The actual engagement depends on the material, and the designer should check a published table for the material in use.

The third principle is the edge distance, which is the distance from the hole centre to the edge of the part. The rule from Day 38 applies: keep the edge distance at least equal to the hole diameter, and make it larger for a loaded joint.

Here is a worked example. An M3 bolt joins a 3 mm PETG bracket to a 3 mm aluminium chassis. The bolt uses a medium clearance hole of 3.4 mm in the bracket and a tapped hole of 2.5 mm in the aluminium. The aluminium takes a full thread, which is good, but the PETG bracket needs a hole of 3.4 mm with a nut on the other side, or a heat-set insert for a thread in plastic. The design decision is whether the bracket uses a nut, which adds a part, or an insert, which adds a step and a cost.

The common mistake is to tap a thread into printed plastic and expect it to hold under repeated assembly. A printed thread strips after a few cycles. Use a nut, a heat-set insert or a larger boss.

Fasteners decide whether a robot can be taken apart for repair, and they must be chosen with the same care as the parts they join.

---

### Materials and drawings

A drawing tells the manufacturer what to make. It includes the views, the dimensions, the tolerances, the material and the finish. The material is part of the design, and the drawing must state it, because a bracket in PLA and the same bracket in aluminium are different parts.

The first principle is that the drawing is the contract. The model is a model, but the drawing is what the shop reads when it cuts the part. The standard for a drawing with geometric tolerances is ASME Y14.5, which defines the datums, the form tolerances and the position tolerances that you will see on engineering drawings.

Here is a worked example of a drawing note. "Bracket, PETG, 3 mm, FDM with 0.2 mm layer, 4 walls, 20 percent infill, layers along the bend. Holes Ø3.58 for M3 clearance (print compensated). General tolerance ISO 2768-m." That single line gives the material, the process, the orientation, the hole value, the tolerance class and the reason for the hole size.

The common mistake is to omit the material or the process from the drawing, so that the manufacturer makes a different part. Always state both, and state the orientation for printed parts.

A manufacturer reads the drawing first, and a clear material note saves a round of questions.

---

### Assembly and the stack-up of tolerances

An assembly is only as accurate as the chain of parts that locate each other. Each part has a tolerance, and each joint adds its tolerance to the chain. The total variation of a gap or a position is the stack-up of these tolerances.

The first principle is the worst case. If every dimension in the chain is at its tolerance limit in the same direction, the stack is the sum of the tolerances. For a chain of three parts, each with ±0.1 mm, the worst-case stack is ±0.3 mm. This is the guaranteed bound, and it is the value to use when a fit must never fail.

The second principle is the root-sum-square. When the errors are random and independent, the probability that all of them reach their limits together is small, and the expected stack is smaller. The RSS stack is the square root of the sum of the squares of the tolerances. For the same three parts, the RSS stack is √(0.1² + 0.1² + 0.1²) = √0.03 = ±0.17 mm.

Here is the worked example. A motor is held to a chassis by a bracket, and the bracket is held to the motor shaft by a bearing. The chain has three dimensions, each ±0.1 mm. The worst case is ±0.3 mm, and the RSS value is ±0.17 mm. If the wheel must clear the chassis by at least 0.5 mm, the worst case leaves 0.2 mm, and the RSS case leaves 0.33 mm. The designer chooses the RSS value for a production run with many parts and the worst case for a critical joint.

The common mistake is to add the tolerances of a chain without checking the direction of each dimension. Some dimensions help the fit and some hurt it, and the stack must use the signed sum. A second mistake is to forget the printing or the machining error in the chain, which can be larger than the drawn tolerance.

Stack-up is the check that tells you whether an assembly will fit when the parts come from different machines, or different prints, on different days.

---

## Hands-on exercises

1. **Print a hole coupon.** Print a plate with holes of 3.0, 3.2, 3.4, 3.6 and 3.8 mm. Measure each hole with the caliper, and test each with an M3 bolt. Record the smallest size that takes the bolt freely, and compute the compensation for your printer.

2. **Cut a tab and slot coupon.** If you have a laser cutter, cut a plate with slots of 2.8, 2.9, 3.0 and 3.1 mm, and a set of tabs of 2.9, 3.0 and 3.1 mm. Find the pair that gives a press fit. Compute the kerf from the difference between the drawn and the cut widths.

3. **Check the wall thickness.** Print a wall test with thicknesses of 0.8, 1.2 and 2.0 mm. Test each wall by pressing it with a finger and by measuring it with the caliper. Note the thinnest wall that does not flex.

4. **Measure an overhang.** Print a test piece with overhangs at 30, 45 and 60 degrees from vertical, without support. Note the angle at which the print fails, and compare it with the 45-degree rule.

5. **Compute a stack-up.** Choose three parts from your robot, each with a tolerance you measure or choose. Compute the worst-case and RSS stacks, and state which one you would use for the joint.

---

## Project: a design-for-manufacturing checker

This project is a Python checker that reads a design description and reports the DFM rules it breaks. The design description lists the features of a part and the process it will be made with. The checker applies the rules from this lesson, reports each violation and its rule, and has tests. It is a full mini-project, because it encodes the rules you will use in every part from now on.

**Goal:** a checker that reads a JSON description of a part, applies the FDM, laser and CNC rules to its features, and prints a table of pass and fail results. It must flag the enclosure from Day 40 for any rule it breaks.

**Repository name:** `100-days-robotics-day-41-design-for-manufacturing`

**Repo layout:**

```
100-days-robotics-day-41-design-for-manufacturing/
├── README.md
├── requirements.txt
├── dfm/
│   ├── __init__.py
│   └── rules.py
├── check_design.py
├── designs/
│   ├── enclosure_base.json
│   └── chassis_plate.json
├── tests/
│   └── test_rules.py
└── coupons/
    └── hole_coupon_results.csv
```

**The design files.** Each design file lists the process and the features. Write `designs/enclosure_base.json` for the Day 40 base, with its wall thickness and its mounting holes.

```json
{
  "name": "enclosure_base",
  "process": "fdm",
  "material": "PLA",
  "features": [
    {"type": "wall", "thickness_mm": 2.0},
    {"type": "hole", "diameter_mm": 3.58, "purpose": "M3 clearance"},
    {"type": "overhang", "angle_deg": 30}
  ]
}
```

**Starter code.** Write `dfm/rules.py`. It holds one function for each process and returns a list of violations.

```python
# dfm/rules.py
FDM_MIN_WALL_MM = 1.2
FDM_MAX_OVERHANG_DEG = 45.0
FDM_MIN_HOLE_MM = 2.0
CNC_MAX_DRILL_DEPTH_RATIO = 5.0


def check_fdm(features):
    violations = []
    for f in features:
        if f["type"] == "wall" and f["thickness_mm"] < FDM_MIN_WALL_MM:
            violations.append(f"wall {f['thickness_mm']} mm is below {FDM_MIN_WALL_MM} mm")
        if f["type"] == "overhang" and f["angle_deg"] > FDM_MAX_OVERHANG_DEG:
            violations.append(f"overhang {f['angle_deg']} deg needs support (limit {FDM_MAX_OVERHANG_DEG})")
        if f["type"] == "hole" and f["diameter_mm"] < FDM_MIN_HOLE_MM:
            violations.append(f"hole {f['diameter_mm']} mm is below {FDM_MIN_HOLE_MM} mm")
    return violations


def check_laser(features, kerf_mm):
    violations = []
    for f in features:
        if f["type"] == "slot" and f["width_mm"] <= kerf_mm:
            violations.append(f"slot {f['width_mm']} mm is not wider than the kerf {kerf_mm} mm")
    return violations


def check_cnc(features, tool_radius_mm):
    violations = []
    for f in features:
        if f["type"] == "inside_corner" and f["radius_mm"] < tool_radius_mm:
            violations.append(f"inside corner radius {f['radius_mm']} mm is below tool radius {tool_radius_mm}")
        if f["type"] == "deep_hole" and f["depth_mm"] / f["diameter_mm"] > CNC_MAX_DRILL_DEPTH_RATIO:
            violations.append("hole depth exceeds the drill depth ratio")
    return violations
```

Write `check_design.py` to load a design file, dispatch to the right rule function, and print the result. Use the process name in the file to choose the function.

**Step-by-step tasks.**

1. Create the repository and the virtual environment. Install pytest.
2. Implement `dfm/rules.py` as above. Check each rule by hand against one example.
3. Write `check_design.py`. It reads a JSON file from the command line, calls the function for the process in the file, and prints each violation, or the word PASS if there are none. For laser parts, read the kerf from the file. For CNC parts, read the tool radius.
4. Write `designs/enclosure_base.json` and `designs/chassis_plate.json`. The chassis plate is a laser part with a kerf of 0.15 mm and a set of tab slots of 2.9 mm.
5. Write `tests/test_rules.py` with at least one passing and one failing case for each rule. The enclosure wall of 2.0 mm must pass, and a wall of 0.8 mm must fail.
6. Run the checker on both designs. Record the output.
7. Fill in `coupons/hole_coupon_results.csv` with the results from the hands-on hole test. Use the compensation you computed to update the hole value in `enclosure_base.json`, and rerun the checker.
8. Write the README. Include the rules, their sources, the coupon results, the stack-up calculation and the checker output for both designs.

**README checklist:**

- [ ] Each rule, its numeric limit, and the source or the reason for the limit
- [ ] The coupon results for the hole and the tab-and-slot tests
- [ ] The checker output for both designs, before and after the compensation
- [ ] The worst-case and RSS stack-up for one joint
- [ ] A statement of the process you chose for each part, and why

**Done when:**

- `pytest` passes all tests
- `check_design.py` reports PASS or FAIL with a reason for each feature of both designs
- The hole value in `enclosure_base.json` is based on your measured coupon, and the checker passes it
- The README states at least one rule that you changed because of your own test, with the numbers
- You can explain, in writing, why a hole designed at exactly the bolt size fails on a printer

**What to commit to GitHub:** the source files, the design files, the tests, the coupon results as CSV, the checker output, photos of the coupons, and the README. Do not commit a virtual environment.

**Why this project is full and not light:** the checker encodes the rules you will apply to every part, and the coupon tests replace published numbers with your own printer's numbers.

---

## Check your understanding

1. A printed hole is designed at 3.4 mm, and the coupon shows it prints at 3.20 mm. A student says the design should be changed to 3.2 mm so that it matches the printed result. Explain why this is wrong.

2. A laser cutter has a kerf of 0.2 mm. A designer draws a 3 mm slot and a 3 mm tab, and expects them to fit with a press fit. Explain why they may not fit, and give the drawn slot and tab sizes that would fit.

3. A milling cutter has a diameter of 8 mm. A designer draws a pocket with 90-degree inside corners. Explain what the machine can make, and give the minimum corner radius.

4. Three parts in a chain each have a tolerance of ±0.05 mm. Compute the worst-case stack and the RSS stack, and state when the RSS value is safe to use.

5. A designer prints an M3 thread into a PETG bracket for a bolt that must be removed and refitted every week. Explain the risk and propose two alternatives.

6. A bracket drawing lists "PETG" with no orientation and no process. Explain what a manufacturer might make, and write the missing note.

**Answers:**

1. The printed result is smaller than the design value because the printer shrinks and undersizes the hole. Changing the design to 3.2 mm would make the next print even smaller, around 3.0 mm, which is too tight for an M3 bolt. The correct approach is to increase the design value by the compensation, here 3.4 + 0.2 = 3.6 mm, so the printed hole measures about 3.4 mm.

2. A kerf of 0.2 mm removes material from each side of a cut. The slot is cut wider than drawn by 0.2 mm, so a 3 mm drawn slot becomes about 3.2 mm, while the tab is cut to its drawn width, reduced by the kerf, so a 3 mm drawn tab becomes about 2.8 mm. The result is a loose fit. To make a 3.0 mm slot and a 3.0 mm tab, draw the slot at 2.8 mm and the tab at 3.2 mm.

3. The rotating cutter has a radius of 4 mm, so an inside corner cannot be sharper than 4 mm. The machine can produce a corner with a radius of 4 mm or more. A 90-degree sharp inside corner needs a different process, such as wire EDM, or a redesign with a corner radius of at least 4 mm.

4. The worst-case stack is 3 × 0.05 = ±0.15 mm. The RSS stack is √(0.05² + 0.05² + 0.05²) = √0.0075 = ±0.087 mm. The RSS value is safe to use for a production run with many parts where the errors are independent and random, and the worst case is required for a joint that must never fail.

5. A printed thread in PETG strips after a few assembly cycles, and the bracket may crack when the bolt is tightened. Two alternatives are a heat-set insert, which gives a metal thread in the plastic, and a through-bolt with a nut on the back side of the bracket, which carries the load without a printed thread.

6. A manufacturer might make the bracket in any material and any process, which gives a part that does not match the design. The missing note should say: "Bracket, PETG, 3 mm, FDM, 0.2 mm layer height, 4 walls, 20 percent infill, layers oriented along the bend, general tolerance ISO 2768-m."

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Design for manufacturability", "Tolerance (engineering)", "Stack-up analysis" (search if the title differs), "Kerf", "Fused filament fabrication", "Overhang (3D printing)" (search if the title differs), "Bend allowance" (search if the title differs) and "ISO metric screw thread"
- *ISO*: ISO 2768 on general tolerances, ISO 273 on clearance holes for fasteners, and ISO 286 on limits and fits. Search for their titles on iso.org.
- *ASME*: the standard ASME Y14.5 on geometric dimensioning and tolerancing. Search for its title on asme.org.
- *Prusa Knowledge Base*: the guides at help.prusa3d.com on overhangs, walls, holes and tolerances
- *Ultimaker*: the design guidelines for FDM printing on its support site, for comparison with the Prusa rules

**Tutorials**

- *GeeksforGeeks*: search for "Limits fits and tolerances" for a worked introduction
- *W3Schools*: the Python tutorial on w3schools.com for the checker code
- *Python documentation*: python.org tutorial on modules, dictionaries and tests

**Watch**

- *MIT OpenCourseWare*: course 2.008 Design and Manufacturing II covers manufacturing processes, tolerances and process selection. Search for the course on ocw.mit.edu.
- *Autodesk Fusion*: the Autodesk YouTube channel has short videos on design for manufacturing and on CAM basics. Search for "design for manufacturing" on the channel.

**Practice**

- *Printables and Thingiverse*: search for "tolerance test" or "hole test" to find published coupons. Compare a published coupon with your own printer's results, and note the difference.

---

## Tomorrow

Tomorrow, on Day 42, you bring everything from Module 5 together in a CAD project. You design a complete small robot chassis, with an assembly, drawings, an exploded view, a bill of materials and a manufacturing plan. The rules you checked today decide the process for each part, and the fits and the stack-up decide whether the chassis will go together on the first try.
