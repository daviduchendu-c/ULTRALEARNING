# Day 35: Mechanisms

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 3.5 hours
**Level:** beginner to intermediate. You need Day 34 for torque, force and stress, and you should be comfortable reading a motor datasheet. Today you choose the parts that carry motion from a motor to a wheel or an arm, so the numbers from Day 13 and Day 34 go straight into the calculations.

**By the end of this day, you will have:**

- A gear train, pulley drive or lead screw that you have sized by hand, with the ratio, speed and torque calculated and written down
- A four-bar linkage with its Grashof check done, and a note on whether one link rotates fully
- A bearing and shaft check for a 1 N·m drive shaft, with the shear stress and a bearing choice
- A Python module that computes these quantities, with tests, and a small measured test of backlash on a real gear pair

**What you need:**

- Python 3.12 with pytest 8 or newer (`pip install pytest`)
- A kit of spur gears, or a LEGO Technic set, or two 3D printed gears of different tooth counts. A LEGO Technic set with gears, axles and beams costs about $40 to $100 and is enough for every exercise. Gear kits for makers cost about $10 to $25.
- A small DC gearmotor, the same one you used on Day 13, to drive the gear train
- A digital caliper, reading to 0.01 mm, which you already have from Day 34
- A tachometer app on a phone, or an optical tachometer, about $10 to $20, for speed measurement
- A small bearing kit with 608 ball bearings (8 mm bore, 22 mm outside diameter, 7 mm wide), about $8 for 10
- Optional: a free CAD tool such as FreeCAD 1.0 or Onshape, which you will use from Day 37

**Prerequisites:**

- Day 34: torque as force times lever arm, shear and bending stress, and the factor of safety
- Day 13: stall torque and the link between motor torque and current
- Basic trigonometry, and the ability to compute a ratio and its inverse

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How mechanisms developed](#how-mechanisms-developed)
- [Glossary](#glossary)
- [Gears and gear ratios](#gears-and-gear-ratios)
- [Gear trains](#gear-trains)
- [Pulleys belts and chains](#pulleys-belts-and-chains)
- [Lead screws](#lead-screws)
- [Linkages](#linkages)
- [Bearings](#bearings)
- [Shafts and couplings](#shafts-and-couplings)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a transmission calculator](#project-a-transmission-calculator)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A motor is fast and weak. A robot wheel is slow and strong. The gap between those two numbers is closed by a mechanism, which is a set of parts that transfers and changes motion. Gears trade speed for torque, pulleys and belts move power across distance, lead screws turn rotation into straight-line motion, and linkages turn one motion into a different one. Almost every robot joint, lift and gripper uses at least one of these parts, and most of the failures in a first robot happen at the mechanism, not the motor.

Today you learn to size these parts. You learn the ratio that sets speed and torque, the efficiency that reduces the ideal output, the rule that tells you whether a linkage can turn fully, and the bearing and shaft checks that keep the mechanism alive. You will calculate these quantities by hand, check them in Python, and then measure one of them on a real gear pair.

---

## How mechanisms developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| Antiquity | Gears appear in the Antikythera mechanism, a Greek device from around the 2nd or 1st century BCE that predicted astronomical positions. | The mechanism shows that gear trains were used for precise, repeatable motion long before electricity. |
| 1500s to 1700s | Clockmakers develop precision gears, escapements and bearings. | Modern gear cutting and bearing design still follow the clockmakers' tolerance thinking. |
| Late 1700s to 1800s | James Watt and others use linkages to convert the reciprocating motion of steam engines into rotation. | Four-bar linkages are still the basis of many robot arms and walking mechanisms. |
| 1900s | Standards for gears and bearings are published, and the ball bearing becomes a mass-produced part. | Standard bearings such as the 608 are cheap, interchangeable and well documented. |
| 1960s to today | Harmonic drives, cycloidal gears and planetary gearboxes are developed for high-precision and high-torque joints. | Robot arms in factories use these reducers in the joints. |
| Today | Makers 3D print gears and use kits, and 3D printed gears show the limits of plastic under load. | Today's skill is knowing when a printed gear is enough and when a metal gear is needed. |

The lesson from the history is that mechanisms rarely fail because the idea is wrong. They fail because the load, the speed, the wear or the backlash was not checked on paper before the parts were built.

---

## Glossary

- **Gear:** a toothed wheel that meshes with another to transmit rotation.
- **Gear ratio:** the number of teeth on the driven gear divided by the number of teeth on the driver. A ratio of 3:1 means the driver turns three times for each turn of the driven gear.
- **Pinion:** the smaller gear in a pair.
- **Gear train:** a chain of meshing gears that transmits motion from one shaft to another.
- **Backlash:** the small gap between meshing teeth, which allows the gears to move slightly before the next gear moves.
- **Efficiency:** the ratio of output power to input power. A real gear mesh is never 100 percent efficient, because of friction and sliding between teeth.
- **Pulley:** a wheel with a groove, used with a belt to transmit rotation.
- **Timing belt:** a toothed belt that meshes with toothed pulleys, so it does not slip.
- **Roller chain:** a chain of links joined by pins and rollers, which drives a toothed sprocket.
- **Lead screw:** a threaded rod with a nut that converts rotation into linear motion.
- **Lead:** the distance a lead screw nut moves in one full turn of the screw.
- **Linkage:** a set of rigid links joined by pin joints, which transfers motion between points.
- **Four-bar linkage:** a linkage with four links, one of which is fixed to the frame.
- **Grashof condition:** a rule for four-bar linkages that tells you whether one link can rotate a full turn.
- **Bearing:** a part that supports a rotating shaft and reduces friction.
- **Shaft:** a rotating rod that carries torque from one part to another.
- **Coupling:** a part that joins two shafts, which may allow small misalignment.
- **Shear stress:** the stress produced by a force or torque that acts parallel to a surface, such as the twisting of a shaft.

---

### Gears and gear ratios

A pair of bicycle chainrings works like a gear pair. A small front ring and a large rear wheel give you more distance per pedal stroke but less torque at the wheel, and the reverse setting gives torque for hills. A gear pair lets you pick that trade on purpose, by choosing the tooth counts.

The first principle is that two meshing gears move the same number of teeth past the contact point in the same time. If the driver has N₁ teeth and the driven gear has N₂ teeth, the ratio of their speeds is the inverse of the ratio of their teeth:

ω₂ / ω₁ = N₁ / N₂

and the ideal torque is multiplied by the same factor in the other direction, so T₂ = T₁ × N₂ / N₁ when the gears are lossless. In practice you multiply by the efficiency η as well, which is typically about 95 to 98 percent for a clean, well-lubricated spur gear mesh.

Here is a worked example. A 20-tooth pinion on a motor shaft drives a 60-tooth gear on a wheel shaft. The ratio is 60 / 20 = 3, so the wheel turns at one third of the motor speed. If the motor runs at 300 rpm, the wheel runs at 100 rpm. If the motor delivers 0.4 N·m, the ideal output torque is 0.4 × 3 = 1.2 N·m, and with an efficiency of 0.95 the real output is about 1.14 N·m. The output is slower and stronger, and the input and output power are approximately equal, which is the point of a gear train.

The common mistake is to set the ratio from the diameters of the gears and not from the tooth counts. Diameter and tooth count are proportional only when the module is the same. The second mistake is to treat the efficiency as a fixed number. Efficiency falls with load, with dirt and with wear, so the datasheet value is a best case.

Gears in robots appear in every geared motor, in drive trains and in the wrist of many arms. The ratio is usually the first number a robot engineer writes down.

---

### Gear trains

A single pair of gears is usually limited to a ratio of about 6:1 in a compact design, because the pinion becomes too small to be strong beyond that. To reach higher ratios, you connect several pairs in series, which is called a gear train. The overall ratio is the product of the individual ratios.

A two-stage train with ratios 3:1 and 4:1 gives an overall ratio of 12:1. The output speed of a motor at 300 rpm would be 25 rpm, and the ideal torque would be 12 times the input torque. With an efficiency of 0.95 per stage, the two-stage efficiency is 0.95 × 0.95 = 0.90, so the output torque is about 10.8 times the input torque. Efficiencies multiply, so every stage costs you some of the output.

Backlash is the other effect that matters. Each gear mesh has a small clearance, typically a few tenths of a millimetre for a plastic gear and less for a precision steel gear. Backlash adds up in a train, so a two-stage train may have a total backlash that equals the sum of both gaps, scaled by the ratio of the first stage. A joint with 0.2 degrees of backlash at the output may be too loose for a precise positioning task.

The common mistake is to chain gears of different modules together. Two gears only mesh correctly when their module is the same, and a mismatch causes the teeth to jam or skip. Check the module, which is the pitch diameter divided by the tooth count, before you buy or print a set.

Gear trains show up in every geared motor, in the reducer of a robot joint and in the power transmission of a mobile robot. A planetary gearbox is a gear train that puts several planets around a central sun gear, which gives high ratios in a small, coaxial package.

---

### Pulleys belts and chains

A pulley and belt work like a gear train, with one difference. The belt does not transmit force by meshing teeth unless it is a timing belt. A flat or V-belt transmits torque by friction, and it can slip when the load is high. A timing belt has teeth that mesh with the pulley and does not slip.

The ratio is the ratio of the pulley diameters or the tooth counts, which is the same idea as for gears. A 16-tooth GT2 pulley driving a 48-tooth GT2 pulley gives a ratio of 3:1. GT2 is a common timing belt profile with a 2 mm tooth pitch. Timing belts are widely used in 3D printers and in linear robot axes, because they are lightweight and quiet.

Roller chains are the third option. A chain of links with rollers drives a sprocket, and it is strong and can carry large loads. The common standard is ISO 606, which defines the pitch and the tooth profile, and the chain pitch of 12.7 mm, called 08B or ANSI 40, is common in small robot drives. A chain can stretch over time and needs tension adjustment, and it is noisier than a timing belt.

Here is a worked example for a timing belt in a linear axis. A motor pulley with 20 teeth on a GT2 belt has a pitch of 2 mm. One revolution of the pulley moves the belt 20 × 2 = 40 mm. The linear speed of the carriage at 300 rpm is therefore 300 × 40 mm / 60 s = 200 mm/s.

The common mistake is to tension a belt too tightly, which overloads the bearings, or too loosely, which lets the belt skip teeth. A timing belt that jumps a tooth shifts the position by one pitch, and the error is permanent until you reset the axis. The way to spot a loose belt is to push on it by hand; a belt that moves more than a few millimetres at the midpoint of a long span is too loose.

Belts and chains are on almost every desktop 3D printer and on many mobile robots, because they move power over a distance and are cheap to replace.

---

### Lead screws

A lead screw turns rotation into straight-line motion. The nut moves along the screw by the lead for each full turn, so a screw with a 2 mm lead moves the nut 2 mm per turn. Lead screws are common in 3D printers, in linear actuators and in robot grippers, because they give a large force from a small motor.

The ideal force from torque on the screw is found from the work balance: the input work per turn, 2π T, equals the output work per turn, F × lead. So F = 2π T / lead. Real screws are less efficient, typically 20 to 50 percent for a trapezoidal screw and above 80 percent for a ball screw, so the real force is:

F = 2π × T × η / lead

Here is a worked example. A stepper motor delivers 0.2 N·m to a trapezoidal lead screw with a 2 mm lead and an efficiency of 0.4. The axial force is F = 2π × 0.2 × 0.4 / 0.002 = 251 N. That force is large enough to press a clamp, but the same screw moves a carriage only 2 mm per turn, so the resolution depends on the step size and microstepping. A 200-step motor with 16 microsteps per step gives 3200 steps per turn, which is 2 mm / 3200 = 0.000625 mm per step.

The common mistake is to compute the linear speed from the ideal lead and forget the backlash of the nut. A plastic or brass nut can have a dead zone of 0.1 mm or more when the direction reverses, which breaks any precise position controller unless it is compensated.

Lead screws are used in 3D printer axes, laboratory stages, press brakes and robot linear axes. Ball screws use recirculating balls to cut friction and are used where efficiency matters.

---

### Linkages

Open a folding gate and you see a linkage. The gate's hinges, arms and stops work together to move one end of the gate along a path. A linkage is a set of rigid bars joined at pivots, and it can transform rotation into a different motion, such as a swing, a straight line or a rocking motion.

The simplest linkage with a useful job is the four-bar linkage: four rigid links, one of which is fixed to the frame, joined by four pivots. The links are the ground (the fixed link), the crank (driven by a motor), the coupler (the floating link), and the rocker (the output link). Changing the link lengths changes the motion of the coupler and the output.

Whether the crank can turn fully is decided by the Grashof condition. Let s be the shortest link length, l the longest, and p and q the two others. If s + l ≤ p + q, then at least one link can rotate a full turn relative to the frame, and the linkage is called a Grashof linkage. If the inequality is not met, no link can rotate fully, and the motion oscillates.

Here is a worked example. The ground is 100 mm, the crank is 30 mm, the coupler is 80 mm and the rocker is 90 mm. The shortest link is the crank at s = 30 mm, and the longest is the ground at l = 100 mm. The other two are p = 80 and q = 90. Then s + l = 130 and p + q = 170. Since 130 ≤ 170, the linkage is Grashof, and the crank can rotate fully. A wrist joint built from this linkage will sweep through a full turn without the motor reversing.

The common mistake is to assume a linkage can rotate fully because one of its links looks long enough. The Grashof test is the only reliable check, and it must be done before the linkage is built, because the dead positions where the linkage locks are hard to see in a drawing.

Four-bar linkages drive the walking legs of some hexapods, the grippers of some pick-and-place machines and the suspensions of some mobile robots.

---

### Bearings

A bearing is a support that holds a rotating shaft in place while letting it turn. A plain bushing slides on the shaft, and a rolling-element bearing uses balls or rollers. The rolling bearing is much lower in friction, and it is the default for most robot shafts.

The most common small bearing in makers' robots is the 608 ball bearing, with an 8 mm bore, a 22 mm outside diameter and a width of 7 mm. It is cheap, standard and easy to replace. Its load rating depends on the manufacturer, and a typical 608 has a dynamic radial load rating of a few hundred newtons, but you should always check the datasheet for the exact figure for the brand you buy.

Bearings have two failure modes that you can check on paper. The first is overload, where the load on the bearing exceeds its rating and the balls or races fatigue. The second is misalignment, where the shaft is not parallel to the housing and the bearing is forced to carry a load it was not built for. A bearing in a misaligned housing will run hot and wear quickly even when the load is low.

The common mistake is to press a bearing into a 3D printed housing with a hole that is slightly too small, which crushes the outer race and gives it a rough feel within a few hours. Design a press fit with a clearance tolerance, test it with a bearing sample, and only then print the full part. Day 41 covers fits in more detail.

Bearings are in every motor, every wheel hub and every robot joint. Choosing the right bearing is as important as choosing the right gear, because the bearing decides how long the mechanism lasts.

---

### Shafts and couplings

A shaft carries torque from one part to another, and it must not twist too far or break. The shear stress in a solid circular shaft under torque T is τ = 16 T / (π d³), where d is the diameter. The formula shows that the stress falls rapidly as the diameter grows, because it varies with the cube of the diameter.

Here is a worked example. A shaft carries 1 N·m. For a diameter of 8 mm, τ = 16 × 1 / (π × (0.008)³) = 16 / (π × 5.12 × 10⁻⁷) = 9.95 MPa. For 6061-T6 aluminium, the shear yield strength is roughly 0.6 times the tensile yield, which is about 165 MPa. The factor of safety is then about 16. For a steel shaft, the margin would be even larger. The small 8 mm shaft is more than strong enough for 1 N·m, but the bearings and the set screws are more likely to fail first, which is the usual pattern in small robots.

Couplings join two shafts. A rigid coupling holds the shafts in exact alignment, which is fine if the shafts are perfectly aligned, and a flexible coupling allows small misalignment by bending or sliding. The flexible jaw coupling, which is a three-part design with a spider insert, is a common choice for small robots, because it absorbs a little misalignment and keeps the shafts from loading the motor bearing.

The common mistake is to join two shafts with a rigid coupling and then mount them in housings that are not exactly parallel. The misalignment loads the bearings every turn, and the motor shaft wears or bends. A flexible coupling with a rated misalignment and a set of careful measurements at assembly prevents most of these failures.

Shafts and couplings appear in every drive train of every robot. They are the simplest parts of the mechanism, and they are often the first to show a problem when a machine is built too quickly.

---

## Hands-on exercises

1. **Measure a gear ratio.** Mount two gears of different tooth counts on shafts, one driven by your DC gearmotor from Day 13. Measure the speed of each shaft with a tachometer app or an optical tachometer. Compute the ratio from the speeds and compare it with the ratio from the tooth counts. The two should agree within about 2 percent.

2. **Measure backlash.** Hold the output gear still with one hand. With a fingertip, rotate the input gear back and forth until the output gear begins to move. Measure the arc at the pitch circle with a ruler or with a marked disc. Record the angle at the output in degrees, and compare it with the value from the gear geometry.

3. **Compute a lead screw.** Using a lead screw from a 3D printer or a leftover kit, measure the lead by turning the screw one full turn while marking the nut position. Compute the force for a torque of 0.2 N·m at efficiency 0.4 and compare the result with your hand calculation.

4. **Check a linkage.** Build a four-bar linkage from cardboard strips and paper fasteners, using link lengths from the worked example. Turn the crank by hand and see whether it rotates fully. Then change the coupler length to 110 mm and test the Grashof condition again.

5. **Load a bearing.** Press a 608 bearing into a hole in a plastic plate using a clearance test piece. Measure the fit with a caliper, and note whether the bearing turns freely or binds. Repeat with a hole 0.1 mm smaller and compare.

---

## Project: a transmission calculator

This project is a Python package that sizes a robot's transmission. It computes gear and pulley ratios, lead screw forces, linkage Grashof checks and shaft stress. It is a full mini-project, because the results feed the drive design you will make in Day 36 and the robot chassis in Day 42.

**Goal:** write a command-line tool that reads a transmission description from a JSON file, computes speed, torque, force and stress at each stage, and prints a table with a pass or fail against your limits.

**Repository name:** `100-days-robotics-day-35-mechanisms`

**Repo layout:**

```
100-days-robotics-day-35-mechanisms/
├── README.md
├── requirements.txt
├── transmission/
│   ├── __init__.py
│   ├── gears.py
│   ├── belts.py
│   ├── leadscrew.py
│   ├── linkage.py
│   └── shafts.py
├── check.py
├── tests/
│   ├── test_gears.py
│   ├── test_leadscrew.py
│   ├── test_linkage.py
│   └── test_shafts.py
└── data/
    └── example_drive.json
```

**Starter code.** Write `transmission/gears.py`:

```python
# transmission/gears.py
def gear_ratio(driver_teeth, driven_teeth):
    """Ratio of driven to driver teeth (output turns per input turn is the inverse)."""
    if driver_teeth <= 0 or driven_teeth <= 0:
        raise ValueError("tooth counts must be positive")
    return driven_teeth / driver_teeth


def stage_output(speed_rpm, torque_nm, ratio, efficiency):
    """Output speed and torque for one stage. Speed divides by the ratio, torque multiplies."""
    if not 0 < efficiency <= 1:
        raise ValueError("efficiency must be in (0, 1]")
    return speed_rpm / ratio, torque_nm * ratio * efficiency


def train_efficiency(efficiencies):
    """Overall efficiency of stages in series is the product of their efficiencies."""
    result = 1.0
    for e in efficiencies:
        result *= e
    return result
```

Write `transmission/leadscrew.py` for the axial force:

```python
# transmission/leadscrew.py
import math


def axial_force(torque_nm, lead_m, efficiency):
    """Axial force in newtons from motor torque, lead in metres, and screw efficiency."""
    return 2.0 * math.pi * torque_nm * efficiency / lead_m


def linear_resolution(lead_m, steps_per_rev, microsteps):
    """Linear distance per microstep, in metres."""
    return lead_m / (steps_per_rev * microsteps)
```

Write `transmission/linkage.py` for the Grashof test:

```python
# transmission/linkage.py
def is_grashof(ground, crank, coupler, rocker):
    """True if at least one link can rotate a full turn relative to the frame."""
    links = sorted([ground, crank, coupler, rocker])
    s, p, q, l = links
    return s + l <= p + q
```

Write `transmission/shafts.py` for the shear stress in a solid shaft:

```python
# transmission/shafts.py
import math


def shear_stress_solid_shaft(torque_nm, diameter_m):
    """Maximum shear stress in a solid circular shaft under torque, in pascals."""
    return 16.0 * torque_nm / (math.pi * diameter_m**3)
```

**Step-by-step tasks.**

1. Create the repository, the virtual environment and the package folder. Install pytest.
2. Implement the functions above. Check each one by hand with the worked examples: the 3:1 ratio gives 100 rpm from 300 rpm, the lead screw gives 251 N, the linkage with a crank of 30 mm and a ground of 100 mm is Grashof, and the 8 mm shaft at 1 N·m gives 9.95 MPa.
3. Write `tests/test_gears.py`. Test the ratio, the speed and torque at one stage, and the efficiency of a two-stage train, which is 0.90 for two 0.95 stages.
4. Write `tests/test_leadscrew.py` for the axial force and the linear resolution. Include the 2 mm lead, 3200 steps per turn example, which gives 0.000625 mm per step.
5. Write `tests/test_linkage.py`. Test the Grashof example, which should return True, and a non-Grashof example that you choose. Check the coupler length of 110 mm from the exercise, and say in a comment whether it is Grashof.
6. Write `tests/test_shafts.py` for the shear stress example.
7. Write `check.py`. It loads `data/example_drive.json`, which describes a motor, a gear train, a lead screw and a shaft with their limits, and prints a table with the speed, torque, force and stress at each stage. Mark each row as PASS or FAIL against the limit in the file.
8. Write the README. It must explain each formula, list its units, and state the limits of the model: an ideal lead screw, a constant efficiency and a solid shaft.

**README checklist:**

- [ ] Installation steps and the Python version used
- [ ] How to run `check.py` and the table it prints
- [ ] The formula for each function, its units and its source
- [ ] A table of the worked examples, with the hand value and the script value side by side
- [ ] Your measured gear ratio and backlash, from the hands-on exercises, with the difference from the calculated value

**Done when:**

- `pytest` passes all tests
- `check.py` prints a table with PASS or FAIL for each stage of `example_drive.json`
- The hand values and the script values agree to within 1 percent for every worked example
- The README includes your measured ratio and backlash, and explains any difference from the calculation
- You can state, in writing, which stage in your drive is the weakest and why

**What to commit to GitHub:** the source tree, the test output as a pasted block in the README, the `check.py` output saved as `check-output.txt`, a photo or table of your gear and backlash measurements, and a short note on the one stage that failed or that you expect to fail first in the real robot.

**Why this project is full and not light:** the calculator is the same reasoning every drive design uses, and a design review asks for that table first.

---

## Check your understanding

1. A motor runs at 2400 rpm and delivers 0.05 N·m. It drives a 15-tooth pinion into a 45-tooth gear, and then a 12-tooth pinion into a 36-tooth gear. Each stage has an efficiency of 0.96. Find the output speed and the output torque.

2. A four-bar linkage has links of 60 mm, 20 mm, 50 mm and 70 mm. Decide whether the crank can rotate fully. Then explain what changes if the 70 mm link is replaced by a 90 mm link.

3. A lead screw with a 4 mm lead and an efficiency of 0.3 is driven by a motor that delivers 0.1 N·m. A designer says the axial force is 2π × 0.1 / 0.004 = 157 N. Explain whether that is the right answer, and give the corrected value.

4. A shaft of 10 mm diameter carries 2 N·m. The designer says the shear stress is 16 × 2 / (π × 0.01²) = 0.10 MPa. Explain the error and give the correct value.

5. A timing belt drive moves a carriage. The motor pulley has 20 teeth on a 2 mm pitch belt, and the carriage moves 40 mm per turn. The belt is measured to have stretched by 0.5 percent after a month. Estimate the position error after 500 mm of travel, and explain whether a closed-loop encoder would detect it.

6. Explain why a gear train with a high total ratio can still have poor output precision even when its gears are machined to a tight tolerance.

**Answers:**

1. The first stage has a ratio of 45 / 15 = 3, so the speed is 2400 / 3 = 800 rpm and the torque is 0.05 × 3 × 0.96 = 0.144 N·m. The second stage has a ratio of 36 / 12 = 3, so the speed is 800 / 3 = 266.7 rpm and the torque is 0.144 × 3 × 0.96 = 0.415 N·m. The overall efficiency is 0.96 × 0.96 = 0.92, so the torque after two stages is 0.05 × 9 × 0.92 = 0.414 N·m, which agrees with the stage-by-stage calculation.

2. The shortest link is 20 mm, and the longest is 70 mm. Then s + l = 90, and p + q = 60 + 50 = 110. Since 90 ≤ 110, the linkage is Grashof and the crank can rotate fully. If the 70 mm link becomes 90 mm, the longest link is now 90 mm, so s + l = 110, and p + q = 60 + 50 = 110. The condition still holds, but only just. A slightly longer link would break it, so the design has no margin.

3. The designer forgot the efficiency. The correct value is 2π × 0.1 × 0.3 / 0.004 = 47 N.

4. The designer squared the diameter instead of cubing it. The formula has d³ in the denominator, so the correct value is 16 × 2 / (π × 0.01³) = 32 / (π × 1 × 10⁻⁶) = 10.2 MPa. Always check the exponent on the diameter, because a wrong power changes the answer by a factor of the diameter in metres.

5. The stretch of 0.5 percent over 500 mm gives an error of 2.5 mm. An encoder on the motor shaft does not see belt stretch, because it measures the pulley rotation, not the carriage. A linear encoder on the carriage would detect it. This is why many linear axes use a linear scale or a belt position sensor.

6. Each mesh has a fixed angular gap at its own shaft. The effect of that gap on the output is divided by the ratios of the stages that follow it, so the gap in the last stage counts in full, and the total output backlash is the sum of these contributions. A tight tolerance shrinks each gap but does not remove it, so a precise output angle needs a sensor on the output shaft as well as on the motor.

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Gear", "Gear train", "Backlash (engineering)", "Pulley", "Timing belt", "Roller chain", "Lead screw", "Four-bar linkage", "Bearing (mechanical)" and "Shaft (mechanical engineering)"
- *Wikipedia*: "Antikythera mechanism" for the historical example of gear trains
- *ISO*: the standard ISO 606 for roller chains, and the standard ISO 6336 for calculation of gear load capacity. Search for their titles on iso.org.
- *MIT OpenCourseWare*: course 2.007 Design and Manufacturing I, which covers gears, linkages and power transmission. Search for it on ocw.mit.edu.

**Tutorials**

- *GeeksforGeeks*: search for "Gear trains" and "Four bar mechanism" for step-by-step examples
- *Python documentation*: python.org tutorial, for functions, modules and tests
- *W3Schools*: the Python tutorial on w3schools.com, for the basics of the project code

**Watch**

- *MIT OpenCourseWare*: lecture videos for course 2.007 on the MIT OCW YouTube channel
- *Autodesk Fusion*: the Autodesk YouTube channel has short videos on joints and motion in Fusion. Search for "joints" on the channel.

**Practice**

- *Onshape Learn*: the free tutorials at learn.onshape.com include lessons on mates and mechanism motion. Use them in Day 39 and check your linkage motion there first.

---

## Tomorrow

Tomorrow, on Day 36, you use these mechanisms to build a drivetrain. You compare the differential, skid steer, Ackermann, omni, mecanum and tracked systems on maneuverability, complexity, odometry and terrain. The ratio and efficiency from today decide the speed and torque each wheel gets, and the linkage check decides whether a steering arm can turn through the angles you need.
