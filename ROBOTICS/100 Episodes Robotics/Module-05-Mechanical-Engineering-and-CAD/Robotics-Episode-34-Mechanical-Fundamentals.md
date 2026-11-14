# Day 34: Mechanical Fundamentals

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 3 hours
**Level:** beginner to intermediate. You need the electronics basics from Module 2 and the motor and power work from Days 12 and 13, because today you size a motor and a bracket against real loads. You do not need calculus. Algebra and right-angle trigonometry are enough.

**By the end of this day, you will have:**

- A free-body diagram of a robot arm link, drawn by hand and checked with a short script
- A Python module that computes torque, centre of mass, bending stress, deflection and factor of safety for a cantilever bracket
- A measured deflection for a plastic ruler acting as a cantilever, compared against your model
- A tipping-stability check for a small mobile robot, with a written conclusion about its centre of mass

**What you need:**

- Python 3.12 (the python.org installer or your system package manager)
- numpy 2.1 or newer and matplotlib 3.9 or newer, installed with `pip install numpy matplotlib`
- pytest 8 or newer, installed with `pip install pytest`
- A digital kitchen scale that reads to 1 g, about $10 to $15
- A digital caliper that reads to 0.01 mm, about $12 to $20. Harbor Freight, Amazon and most hardware stores sell them.
- A spring scale or a bathroom scale for a rough force test, about $10
- A plastic or aluminium ruler, 300 mm long, about $2
- Optional: a free CAD tool, which you will use from Day 37 onward. Today you only need a pencil and paper.

**Prerequisites:**

- Day 1 to Day 5: the engineering workflow and basic command-line work from Module 1
- Day 12: power budgets, and how a load on a motor changes the current it draws
- Day 13: stall current and torque, which you need again here
- Comfort with solving two linear equations and with sine and cosine of an angle

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How mechanics developed](#how-mechanics-developed)
- [Glossary](#glossary)
- [Force and free-body diagrams](#force-and-free-body-diagrams)
- [Torque and moments](#torque-and-moments)
- [Mass and centre of mass](#mass-and-centre-of-mass)
- [Equilibrium](#equilibrium)
- [Friction](#friction)
- [Stress and strain](#stress-and-strain)
- [Stiffness and deflection](#stiffness-and-deflection)
- [Factor of safety](#factor-of-safety)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a bracket and stability calculator](#project-a-bracket-and-stability-calculator)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Every robot is a set of rigid parts carrying loads. A motor pushes a link, the link pushes a load, the load pushes back, and the bracket that holds the motor either carries that force or bends under it. Mechanical engineers call this statics and strength of materials, and robotics teams use the same ideas when they decide whether a wheel hub will slip, whether a shoulder servo will stall, or whether a plastic mount will crack after a week of use.

Today you learn the small set of quantities that describe those loads: force, torque, mass, centre of mass, friction, stress, strain and stiffness. You learn the one rule that ties them together, which is that a body at rest has no net force and no net moment. You then apply those rules to two real problems. The first is a bracket that carries a motor. The second is a mobile robot that must not tip over. Both problems show up on the job in every robotics company, and both are easy to get wrong with a wrong unit or a missing lever arm.

---

## How mechanics developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| Ancient times | Archimedes described the lever and the principle of moments, which is still the basis of torque calculations. | Every joint in a robot arm is a lever. |
| 1600s | Galileo studied beam strength and Newton stated the laws of motion. | Forces and accelerations are the same in a robot as in a bridge. |
| 1678 | Robert Hooke published the law that stress is proportional to strain for small deformations. | Hooke's law is still the first model for any bracket or spring. |
| 1800s | Engineers developed beam theory and the first systematic tables of material properties. | The datasheets for aluminium and steel come from this work. |
| 1900s | Finite element analysis was developed for complex shapes, and was made practical by computers in the following decades. | CAD tools now run simple stress checks in the design window. |
| Today | Students and makers use free CAD tools, online calculators and Python to check designs before cutting metal or printing plastic. | Today's skill is fast, careful hand calculation followed by a software check. |

The history has one lesson for beginners. The math did not change much in three hundred years. What changed is how cheaply you can check a design, so the habit that matters is checking every part before it leaves the desk.

---

## Glossary

- **Force:** a push or a pull, measured in newtons (N). One newton is the force that accelerates one kilogram by one metre per second squared.
- **Moment (or torque):** the turning effect of a force about a point, equal to force times the perpendicular distance from the point to the line of the force. Measured in newton-metres (N·m).
- **Lever arm:** the perpendicular distance from a pivot to the line of action of a force.
- **Mass:** the amount of matter in an object, measured in kilograms (kg). Weight is the force of gravity on that mass, equal to mass times 9.81 m/s² on Earth.
- **Centre of mass:** the point where the mass of a body can be considered to act. For a uniform gravity field it is also the centre of gravity.
- **Equilibrium:** the state of a body with zero net force and zero net moment about every point.
- **Free-body diagram:** a sketch of one body with every external force drawn on it and nothing else.
- **Friction:** a force that resists sliding between two surfaces. Static friction holds a body still until a threshold is reached.
- **Coefficient of friction (μ):** the ratio of the friction force to the normal force between two surfaces. It has no unit.
- **Stress:** force divided by the area it acts on, measured in pascals (Pa). One megapascal (MPa) is one million pascals.
- **Strain:** the change in length divided by the original length. It is a ratio and has no unit.
- **Elastic modulus (Young's modulus, E):** the ratio of stress to strain in the linear range of a material, in pascals.
- **Stiffness:** the force needed to produce a unit deflection, in newtons per metre.
- **Yield strength:** the stress at which a material starts to deform permanently.
- **Factor of safety:** the ratio of the strength of a part to the load it actually carries. A value of 2 or more is common in hobby robotics, and the right value depends on the consequences of failure.

---

### Force and free-body diagrams

Think of a shopping bag hanging from one hand. The bag pulls down on your hand, your hand pulls up on the bag, and the handle of the bag pulls on your fingers. Each of these pushes and pulls is a force, and every force has a size, a direction and a point where it acts. A free-body diagram is the sketch that lists them.

The method has three steps. Choose one body, draw it as a simple outline, and then draw every external force on it as an arrow from the point where the force acts. Gravity acts at the centre of mass. A joint reaction acts at the pivot. A contact force acts at the surface where two parts touch. Do not draw forces that the body applies to other bodies, because those belong to the other body's diagram.

The first principle is Newton's second law in component form. For a body at rest or moving at constant velocity, the sum of forces in each direction is zero:

ΣFx = 0 and ΣFy = 0

Here is a worked example. A 0.5 kg robot forearm is held horizontal by a servo at its base. Gravity acts at its centre, 0.15 m from the servo. The servo must provide a vertical reaction of 0.5 × 9.81 = 4.9 N to hold the weight up, and it must also supply the turning effect discussed in the next section. The free-body diagram shows the weight at the centre, the reaction at the pivot, and the servo torque as a curved arrow.

The common mistake is to draw the forces on the wrong body, or to draw a force that the body does not experience. Check each arrow by asking which object pushes or pulls on the body at that point. If you cannot name the object, the arrow does not belong.

On the job, free-body diagrams open every design review for a robot arm, a lift or a suspension. Engineers write them on a whiteboard before they open CAD, because the diagram tells them which parts need strength and which need stiffness.

---

### Torque and moments

A child on a seesaw uses the same idea that a robot joint uses. Sitting farther from the pivot makes the seesaw easier to tip, even with the same weight. The turning effect depends on the force and on the distance from the pivot to the line of the force. That turning effect is a moment, which engineers also call torque when it comes from a motor shaft.

The first-principles formula is:

M = F × d

where F is the force perpendicular to the lever and d is the perpendicular distance from the pivot. If the force is at an angle θ to the lever, only its perpendicular component counts, so M = F × d × sin(θ) when d is measured along the lever to the point of application. This is why a wrench works best when you pull at right angles to the handle.

For the forearm in the previous section, the weight of 4.9 N acts 0.15 m from the servo horn. The required holding torque is 4.9 × 0.15 = 0.735 N·m, or about 7.5 kg·cm in the units that hobby servo datasheets use. A servo rated at 10 kg·cm at 6 V gives a margin of about 33 percent at that rating. A motor rated for 0.3 N·m would fail this test, even if its no-load speed looks fast.

The common mistake is to use the distance from the pivot to the centre of the robot instead of the distance to the line of action of the force. A force directed along the lever has zero moment, even if it is large. The way to spot it is to draw the force arrow and the lever on the same diagram, and measure the perpendicular distance between them.

Torque is the number that connects mechanics to motors. Every motor datasheet from a robotics supplier gives a stall torque, and the design question is always whether that stall torque exceeds the moment the load produces at the joint.

---

### Mass and centre of mass

A loaded backpack is easy to carry when its weight sits close to your back, and hard to carry when it hangs from your hand. The reason is the centre of mass: the single point through which the weight of the whole object acts. Where that point sits decides how the object balances and tips.

For a set of point masses, the centre of mass is the mass-weighted average of their positions:

x̄ = (Σ mᵢ xᵢ) / (Σ mᵢ)

and similarly for y and z. For a solid object, the sums become integrals, but for robot design you usually use a table of components, each with a mass and a position.

Here is a worked example for a small differential-drive robot. The chassis is 1.2 kg and its centre is at the origin. The two drive motors are 0.2 kg each, placed at x = 0 and y = ±0.08 m. The battery is 0.5 kg and sits at x = −0.05 m. The total mass is 2.1 kg. The x coordinate of the centre of mass is (0 + 0 + 0 + 0.5 × −0.05) / 2.1 = −0.0119 m, or about 12 mm behind the origin. That result matters for the stability check in the project.

The common mistake is to take the geometric centre of a shape as its centre of mass. A battery placed at one end of a chassis moves the centre of mass toward that end, even though the chassis itself is symmetric. Weigh the parts, or use CAD mass properties, before you assume the centre is at the middle.

Centre of mass shows up in practice in every mobile robot. Warehouse robots, delivery robots and balancing robots are all designed around their centre of mass, and their control systems often estimate it from an inertial measurement unit (IMU).

---

### Equilibrium

A static bridge is in equilibrium: its supports push back exactly as hard as the loads push down, and it does not spin. Equilibrium is the state that engineers use to find unknown forces, such as the reaction at a bearing or the pull in a tendon. It rests on two equations for a planar body:

ΣFx = 0, ΣFy = 0, and ΣM = 0 about any point

The three equations are enough to solve for three unknowns in a planar problem. If a problem has more unknowns than equations, the structure is statically indeterminate, and you need more information, such as the stiffness of each support.

Take the forearm again. Its free-body diagram has the weight W at the centre, the servo pivot reaction R at the base, and the servo torque T. Taking moments about the pivot removes R from the equation, so T = W × d, which gives the same 0.735 N·m as before. Taking moments about the centre instead would remove W, and the result would show the reaction force at the pivot. Choose the point that removes the unknown you do not want.

The common mistake is to write a moment equation with mixed signs. Pick a positive direction for rotation, such as counter-clockwise, and apply it to every term. Write each term with its sign on the page before you add them.

Equilibrium is the first check on every robot mechanism, and CAD tools now compute it directly in their simulation modes. Understanding the hand calculation first lets you read those results without trusting them blindly.

---

### Friction

Push a book across a table and it resists, then suddenly it moves. Friction is the force that resists sliding, and it has two regimes. Static friction holds a body still until a threshold is reached. Kinetic friction acts once the body slides, and it is usually a little lower than the static threshold.

The model is the Coulomb model, named after Charles-Augustin de Coulomb, who studied friction in the 1780s:

F_friction ≤ μ × N

where N is the normal force, the component of contact force perpendicular to the surface, and μ is the coefficient of friction. The inequality holds while the body is still. Once the body moves, the friction force equals μ_k × N.

For the drive wheels of a 2 kg robot on a flat floor with rubber tyres, N is about 2 × 9.81 = 19.6 N if all weight sits on the drive wheels. A typical rubber on painted floor coefficient might be around 0.6, so the maximum traction force is about 0.6 × 19.6 = 11.8 N. If the drive wheels push harder than that, they slip, and the encoder reading no longer matches the robot's position. Engineers call this wheel slip, and it is the single most common source of odometry error in beginner robots.

The common mistake is to use the coefficient for a clean surface when the floor is dusty or wet, or to forget that the normal force on the drive wheels is only part of the robot's weight when the centre of mass sits toward the castor. Measure the coefficient with a ramp and a block if the robot will work on a specific floor, because the table values are only rough guides.

Friction appears in every joint, belt and wheel. It is also the reason a robot can drive up a ramp at all, and it sets the limit of the drivetrain that Day 36 covers.

---

### Stress and strain

Pull on a rubber band and it stretches. Pull harder and it may snap. The same thing happens inside a metal bracket, but you cannot see it. Stress and strain are the two numbers that describe what happens inside a material when a load is applied.

Stress is the force per unit area on a cross-section:

σ = F / A

Strain is the fractional change in length:

ε = ΔL / L

For a linear, elastic material, the two are related by Hooke's law:

σ = E × ε

where E is the elastic modulus. Steel has E of about 200 GPa, aluminium alloys such as 6061-T6 have about 69 GPa, and 3D printed PLA has a few gigapascals, depending on its orientation and infill. Those numbers differ by factors of tens, which is why a plastic bracket can flex under the same load that leaves a metal one unchanged.

Here is a worked example for a tension member. A 6 mm diameter aluminium rod carries a 500 N pull. Its area is π × (0.003)² = 2.83 × 10⁻⁵ m², so the stress is 500 / 2.83 × 10⁻⁵ = 17.7 MPa. The strain is 17.7 × 10⁶ / 69 × 10⁹ = 2.6 × 10⁻⁴, which is a change of about 0.026 percent. For a 100 mm rod the elongation would be 0.026 mm, which is small but measurable with the caliper.

The common mistake is to use the diameter where the radius belongs, which makes the area four times too large or too small. Check the formula: area of a circle is π r², not π d². The second mistake is to ignore stress concentrations at holes and sharp corners, where the local stress can be several times the average stress.

Stress and strain are the language that every material datasheet uses. The yield strength of a material, which marks the start of permanent deformation, is read from a stress-strain curve, and it is the value that sets the working limit of every structural part in a robot.

---

### Stiffness and deflection

A diving board bends under your weight. The bend is the deflection, and the board's resistance to bending is its stiffness. For a robot, stiffness matters because a bent bracket moves the motor, changes the alignment of a gear mesh and can make a sensor read the wrong position.

For a linear spring, stiffness k is the force divided by the deflection, k = F / δ. For a beam, the stiffness depends on the material and the shape. A cantilever beam, which is fixed at one end and loaded at the other, has deflection:

δ = F L³ / (3 E I)

where L is the length and I is the second moment of area of the cross-section, which measures how well the shape resists bending. For a rectangle of width b and height h, I = b h³ / 12. The h³ term means that doubling the height of a bracket makes it eight times stiffer, while doubling its width makes it only twice as stiff.

Here is a worked example. A 20 mm wide, 3 mm thick aluminium strip is 100 mm long and carries a 10 N load at its tip. The second moment of area is I = 20 × 3³ / 12 = 45 mm⁴ = 45 × 10⁻¹² m⁴. The deflection is δ = 10 × (0.1)³ / (3 × 69 × 10⁹ × 45 × 10⁻¹²) = 1.07 mm. The stiffness of the strip is therefore 10 / 0.00107 = 9.3 kN/m.

The common mistake is to treat a bracket as rigid because it looks rigid. A 1 mm deflection at the end of a 100 mm arm is a 1 percent error in position, which may not matter for a bumper, but it matters for a camera mount that must hold a line for a visual odometry system.

Deflection is a design limit as much as a strength limit. Many robot parts fail to meet their stiffness requirement long before they reach their yield strength, and stiffness is often the reason to change a bracket from plastic to aluminium or to add a rib.

---

### Factor of safety

A bridge engineer designs the bridge to carry more than the expected load. The margin between what the bridge can carry and what it must carry is the factor of safety. For a robot part, the factor is the strength of the material divided by the stress the load creates:

n = σ_yield / σ_max

Using the aluminium strip from the previous section, the maximum bending stress is σ = M c / I, where the bending moment M = F × L = 10 × 0.1 = 1 N·m and c = h / 2 = 1.5 mm is the distance from the neutral axis to the outer fibre. Then σ = 1 × 0.0015 / (45 × 10⁻¹²) = 33.3 MPa. With a yield strength of about 276 MPa for 6061-T6, the factor of safety is 276 / 33.3 = 8.3.

That number is high, which is correct for a bracket that holds a motor on a robot, because a failure can break a gear or injure a student. A low-consequence part such as a cable clip can use a lower factor, and a safety-critical part can use a higher one. The choice is a judgement that you must write down.

The common mistake is to use the ultimate tensile strength instead of the yield strength. The ultimate strength is the maximum before a part breaks, and yield is the point where it starts to bend permanently. A bracket that yields is ruined even if it does not fracture. The second mistake is to ignore impact. A dropped robot can see many times its static weight for a few milliseconds, so the design load must include impact or the bracket will fail the first time it hits the floor.

On the job, factor-of-safety calculations appear in design reviews, in CAD simulation reports and in the test plans that a manufacturer must complete before shipping a product.

---

## Hands-on exercises

1. **Measure a ruler cantilever.** Clamp a 300 mm plastic ruler at one end so that 100 mm of its length sticks out over a table edge. Hang a 50 g mass from a loop of thread at the 100 mm mark, and measure the vertical deflection of the free end with a caliper or a ruler against a fixed reference. Record the value to 0.5 mm. Then repeat with 100 g and 150 g. Your results should scale roughly linearly with load. Write your measured stiffness as k = F / δ.

2. **Measure a mass and a centre of mass.** Build a simple robot chassis from a sheet of cardboard or a 3D printed base, weigh each component on the kitchen scale, and record its position from a chosen origin. Compute the centre of mass with the formula from this lesson. Balance the chassis on a thin ruler edge, and compare the balance point with your calculation. The two should agree within about 5 mm.

3. **Check a torque.** Hang a known mass from a 100 mm lever arm attached to a shaft or a dowel. Measure the force needed to hold the lever at a horizontal angle with a spring scale. Compute the torque from the force and the distance, and compare it with the hand value.

4. **Write the free-body diagram for a robot arm.** Use a 2-link arm of your choice. Draw the free-body diagram for each link, label every force, and write the equilibrium equations. Solve for the holding torque at the shoulder.

5. **Test your friction model.** Place a 500 g book on a board, and tilt the board slowly until the book slides. The tangent of the angle at which sliding begins equals the static coefficient of friction. Repeat with a rubber pad under the book and compare the two values.

---

## Project: a bracket and stability calculator

This project is a small Python package. It computes the things you learned today for a motor bracket and for a mobile robot. It is a full mini-project, because the same calculations appear in the design review for every robot part you will build in Module 5.

**Goal:** write a command-line script that takes the geometry and loads of a cantilever bracket and a mobile robot, computes the key quantities, checks them against a factor-of-safety limit, and writes a short text report.

**Repository name:** `100-days-robotics-day-34-mechanical-fundamentals`

**Repo layout:**

```
100-days-robotics-day-34-mechanical-fundamentals/
├── README.md
├── requirements.txt
├── mechanics/
│   ├── __init__.py
│   ├── statics.py
│   ├── beams.py
│   └── stability.py
├── report.py
├── tests/
│   ├── test_beams.py
│   ├── test_statics.py
│   └── test_stability.py
└── data/
    └── example_bracket.json
```

**Starter code.** Begin with `mechanics/beams.py`. It contains the formulas for a rectangular cantilever:

```python
# mechanics/beams.py
def second_moment_rect(b, h):
    """Second moment of area for a rectangle, width b and height h, in m^4."""
    return b * h**3 / 12.0


def cantilever_deflection(F, L, E, I):
    """Tip deflection of a cantilever under a point load at its end, in metres."""
    return F * L**3 / (3.0 * E * I)


def cantilever_bending_stress(F, L, h, I):
    """Maximum bending stress at the root, in pascals."""
    M = F * L
    c = h / 2.0
    return M * c / I


def factor_of_safety(sigma_yield, sigma_max):
    """Ratio of yield strength to maximum stress."""
    if sigma_max <= 0:
        raise ValueError("maximum stress must be positive")
    return sigma_yield / sigma_max
```

Write `mechanics/statics.py` to compute the moment from a force and a distance, and the centre of mass from a list of components:

```python
# mechanics/statics.py
def moment(force, lever_arm):
    """Moment in N*m for a force perpendicular to a lever arm in metres."""
    return force * lever_arm


def centre_of_mass(parts):
    """parts is a list of (mass_kg, x_m) tuples. Returns the x coordinate of the centre of mass."""
    total = sum(m for m, _ in parts)
    if total <= 0:
        raise ValueError("total mass must be positive")
    return sum(m * x for m, x in parts) / total
```

Write `mechanics/stability.py` with a tipping check. A robot tips when the centre of mass lies outside the line between the two wheel contact points. For a differential-drive robot, the tipping margin along the x axis is the distance from the centre of mass to the nearest edge of the support.

```python
# mechanics/stability.py
def tipping_margin(com_x, support_front_x, support_rear_x):
    """Distance from the centre of mass to the nearest support edge, in metres.
    A positive value means the robot is stable. Negative means it tips."""
    return min(com_x - support_rear_x, support_front_x - com_x)
```

**Step-by-step tasks.**

1. Create the repository and a virtual environment. Install numpy, matplotlib and pytest from `requirements.txt`.
2. Implement the functions above, and run each one by hand with the numbers from this lesson. The deflection of the 20 mm by 3 mm aluminium strip should be 1.07 mm, and the bending stress should be 33.3 MPa.
3. Write `tests/test_beams.py` with at least three tests. One test checks the deflection for the worked example. One test checks that doubling the height reduces the deflection by a factor of eight. One test checks that a zero or negative stress raises an error in `factor_of_safety`.
4. Write `tests/test_statics.py` for the moment and centre-of-mass functions. Include the forearm example, which gives 0.735 N·m, and the chassis example, which gives a centre of mass of about −0.0119 m.
5. Write `tests/test_stability.py`. Test a robot whose centre of mass is in the middle (margin positive) and one whose centre of mass is outside the support (margin negative).
6. Write `report.py`. It loads `data/example_bracket.json`, which contains the bracket geometry, the load, the material yield strength and the modulus, and it prints the deflection, stress, factor of safety and tipping margin. Save the output to `report.txt`.
7. Write the README. Include a short section that explains each formula, the units you used, and the assumptions: a point load at the tip, a rectangular section, linear elastic behaviour and a flat floor.

**README checklist:**

- [ ] Installation steps, with the Python version used
- [ ] How to run `report.py` and where the output is written
- [ ] Each formula, its units and its source
- [ ] A table of the worked examples, with your computed values next to the hand values
- [ ] A list of assumptions and their limits

**Done when:**

- `pytest` passes with all tests green
- `report.py` prints the deflection, stress, factor of safety and tipping margin for the example bracket, and writes `report.txt`
- Your hand value and your script value agree to within 1 percent for the worked examples
- The README states at least two assumptions and explains when each one fails
- You can explain, in writing, why the factor of safety you chose is appropriate for a motor bracket

**What to commit to GitHub:** the full source tree, the test output as a pasted block in the README, the generated `report.txt`, and a photo or table of your ruler-cantilever measurements from the hands-on exercises. Commit the measurement results even if they do not match the model, and add a sentence explaining the difference.

**Why this project is full and not light:** the calculations are the basis of every structural check in the rest of Module 5, and the test suite forces you to get the units right before you rely on them in CAD.

---

## Check your understanding

1. A motor produces a torque of 0.4 N·m at its output, and it drives a 0.1 m lever carrying a 5 N load that acts at 60 degrees to the lever. Will the motor hold the load? Show the calculation with the perpendicular component.

2. A robot chassis has a centre of mass 20 mm behind its geometric centre. Its wheels are 180 mm apart. Explain whether the robot is more likely to tip backward or forward when it accelerates forward, and why the answer is not given by the centre of mass alone.

3. Two brackets have the same material and the same load. One is 5 mm thick and the other is 10 mm thick, with the same width. Without calculating the exact numbers, state which one deflects less and by roughly what factor, and explain your reasoning from the formula for I.

4. A student computes the factor of safety of a bracket using the ultimate strength of the material. Explain why this is an error and what the correct value to use is.

5. A 1.5 kg robot sits on a 30 degree ramp with rubber tyres. The coefficient of static friction is 0.7. Decide whether the robot will slide, and state which assumption in your answer would change the result the most.

6. Explain why a CAD simulation that reports a factor of safety of 6 can still be wrong. Name two assumptions in the simulation that you would check before trusting the result.

**Answers:**

1. The perpendicular component is 5 × sin(60°) = 4.33 N. The moment is 4.33 × 0.1 = 0.433 N·m, which is higher than the 0.4 N·m the motor provides. The motor does not hold the load. A motor with a higher stall torque, a shorter lever or a lower load is needed.

2. The centre of mass alone does not decide the tipping direction. Acceleration adds an inertial force at the centre of mass, acting backward, which creates a tipping moment about the wheel contacts. The tipping direction depends on the height of the centre of mass, the acceleration and the horizontal position. Engineers use the full free-body diagram to check both.

3. The 10 mm thick bracket deflects less. Because I = b h³ / 12, doubling the thickness multiplies I by eight, so the deflection falls by about a factor of eight, provided all other values are equal.

4. The ultimate strength is the maximum before fracture, not the stress at which the part starts to deform permanently. The correct value for a bracket is the yield strength, because yielding ruins a part even if it does not break.

5. The normal force is 1.5 × 9.81 × cos(30°) = 12.7 N. The gravitational component along the ramp is 1.5 × 9.81 × sin(30°) = 7.36 N. The maximum static friction is 0.7 × 12.7 = 8.9 N, which is greater than 7.36 N, so the robot holds. The assumption that matters most is the coefficient, because a wet or dusty ramp can drop it below 0.58, which is where the robot would start to slide.
6. A simulation depends on the mesh, the material model, the boundary conditions and the loads. A poor boundary condition, such as a fixed support where the real part is bolted and can slip, or a load applied to the wrong face, gives a wrong result with a confident number. You should check the mesh convergence and compare the simulation with a hand calculation for a simple case.

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Force", "Torque", "Center of mass", "Mechanical equilibrium", "Friction", "Stress (mechanics)", "Strain (mechanics)", "Hooke's law", "Stiffness" and "Factor of safety"
- *Wikipedia*: "Euler-Bernoulli beam theory" for the beam formulas, and "Second moment of area"
- *MIT OpenCourseWare*: course 2.001 Mechanics and Materials I, which covers statics and stress in detail. Search for the course number on ocw.mit.edu.
- *ASME*: the standard ASME Y14.5, which defines geometric dimensioning and tolerancing. You will use it from Day 41 onward. Search for its title on asme.org.

**Tutorials**

- *GeeksforGeeks*: search for "Free body diagram" and "Equilibrium of rigid bodies" for worked examples
- *W3Schools*: W3Schools does not cover mechanics. Use the Python tutorial on w3schools.com for the Python parts of the project.
- *Python documentation*: the tutorial on python.org, for functions, tests and modules

**Watch**

- *MIT OpenCourseWare*: the lecture videos for course 2.001 on the MIT OCW YouTube channel
- *Autodesk Fusion*: the Autodesk YouTube channel has short videos on simulation and static analysis. Search for "static stress analysis" on the channel.

**Practice**

- *Onshape Learn*: the free tutorials at learn.onshape.com include a short lesson on mass properties. Use them as a check on your hand calculation of centre of mass.

---

## Tomorrow

Tomorrow, on Day 35, you move from single parts to the parts that transmit motion between them: gears, pulleys, belts, chains, lead screws, linkages, bearings and shafts. Today's torque and stress calculations are the inputs to those designs. A gear that carries a torque of 0.4 N·m at a 20:1 reduction sees a different load on its teeth than the motor does, and the ratio you compute tomorrow decides the speed of the robot joint you build later.
