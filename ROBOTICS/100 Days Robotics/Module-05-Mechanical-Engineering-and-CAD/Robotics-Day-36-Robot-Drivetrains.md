# Day 36: Robot Drivetrains

*100 Days of Robotics Engineering, Module 5: Mechanical Engineering and CAD*

**Time:** about 4 hours
**Level:** intermediate. You need Day 35 for ratios and gear trains, Day 13 for motor drive, and Day 34 for friction and wheel slip. The lesson uses basic 2D geometry and vectors, and it introduces the kinematics you will develop fully in Module 8.

**By the end of this day, you will have:**

- A comparison table of five drivetrains (differential, skid steer, Ackermann, omni and mecanum, and tracked), scored on manoeuvrability, mechanical complexity, odometry and terrain
- The forward kinematics for a differential-drive robot and a four-wheel mecanum robot, written as equations and checked on a worked example
- A Python simulator that predicts the pose of each drivetrain from wheel commands, with tests
- A measured odometry drift for a real differential-drive robot driving a 1 m square, with the cause of the error identified

**What you need:**

- Python 3.12 with numpy 2.1 or newer, matplotlib 3.9 or newer, and pytest 8 or newer
- A two-wheel differential chassis with two geared DC motors and wheel encoders. Kits from Pololu, DFRobot and similar suppliers cost about $30 to $60. If you built a chassis for Day 33, use it.
- Optional: a set of four mecanum wheels, about $60 to $120 for a kit, and three omni wheels, about $10 to $15 each
- A microcontroller board from Day 16 to Day 25, such as an ESP32 or Arduino Uno, to drive the motors
- A 5 m measuring tape, about $5, and masking tape to mark the floor
- A phone with a stopwatch, and a notebook to log the results

**Prerequisites:**

- Day 35: gear ratios, the efficiency of a gear train, and the role of the gearbox in a wheel drive
- Day 34: friction, stress and factor of safety, and the friction limit of a tyre on the floor
- Day 13: motor drive through an H-bridge
- Day 29: encoders and how they count rotation
- Comfort with sine, cosine and vectors in two dimensions

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How robot drivetrains developed](#how-robot-drivetrains-developed)
- [Glossary](#glossary)
- [Differential drive](#differential-drive)
- [Skid steer](#skid-steer)
- [Ackermann steering](#ackermann-steering)
- [Omni wheels and mecanum wheels](#omni-wheels-and-mecanum-wheels)
- [Tracked drives](#tracked-drives)
- [Odometry and where it drifts](#odometry-and-where-it-drifts)
- [Comparing the drivetrains](#comparing-the-drivetrains)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a drivetrain simulator and drift study](#project-a-drivetrain-simulator-and-drift-study)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot's drivetrain decides where it can go. The motor is the engine, but the wheels, the steering geometry and the contact with the floor decide whether a robot can turn in place, climb a ramp, carry a load across gravel or move sideways into a shelf. Two robots with the same motors can have completely different capabilities because of this choice, and the choice also sets how hard the robot is to program, how well it knows its own position and how much it costs to repair.

Today you compare the five drivetrains that appear in most small and medium robots. You will write the kinematic equations for two of them, check those equations with a simulator, and measure the error a real robot accumulates when it tries to follow a square. The skill is practical. When a team designs a robot, the drivetrain is one of the first decisions, and the odometry error it causes is one of the first problems the team meets.

---

## How robot drivetrains developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1800s | Steam carriages and early carts used a fixed front axle, with the wheels turning on a single axle. | The Ackermann principle appears in carriage design long before robots. |
| Early 1800s | A carriage steering arrangement is patented in England and later becomes known as Ackermann steering, after Rudolph Ackermann, whose London firm was associated with the design. | Cars still use an approximation of this geometry. |
| Early 1900s | Tracked vehicles are developed for rough terrain, and skid steering becomes common in tractors and military vehicles. | Skid steer is still the standard in construction machines. |
| 1960s to 1980s | Industrial automated guided vehicles use differential drives and wire guides. | The differential drive is the most common choice for indoor mobile robots. |
| 1970s | Swedish inventor Bengt Ilon designs the mecanum wheel at Mecanum AB, while working on a drive that moves sideways. | Mecanum wheels are used in warehouse robots and some hobby robots. |
| Today | Differential, omni and mecanum drives appear in warehouse, lab and education robots, and tracks are common in outdoor robots. | The drivetrain is still chosen from the same tradeoffs: manoeuvre, complexity, odometry and terrain. |

The history shows that drivetrains are chosen by the task, not by fashion. A wheel that is ideal on a factory floor is poor on gravel, and a track that climbs stairs is slow on a flat floor.

---

## Glossary

- **Drivetrain:** the set of wheels, tracks or legs that carry the robot and the parts that transmit power to them.
- **Differential drive:** a drive with two independently driven wheels on the same axle, plus a passive support such as a castor.
- **Skid steer:** a drive in which the wheels or tracks on each side turn at different speeds, and the robot turns by scrubbing the surface.
- **Ackermann steering:** a steering geometry in which the inner and outer front wheels turn to different angles, so that all wheels roll without sliding.
- **Wheelbase:** the distance between the front and rear axles.
- **Track width:** the distance between the left and right wheels.
- **Omni wheel:** a wheel with small rollers around its rim, set at 90 degrees to the wheel's rolling direction, so that it can slide sideways.
- **Mecanum wheel:** a wheel with rollers set at 45 degrees to the wheel axis, which together move the robot in a combined direction.
- **Holonomic:** a robot that can move in any direction in the plane regardless of its heading.
- **Non-holonomic:** a robot that cannot move sideways directly, such as a car or a differential-drive robot.
- **Odometry:** an estimate of the robot's position and heading from wheel rotation.
- **Kinematics:** the relation between wheel speeds and body motion, without considering the forces that cause the motion.
- **Pose:** the position and heading of the robot, written as (x, y, θ).
- **Slip:** a difference between the speed of the wheel surface and the speed of the robot over the ground.
- **Scrub:** the sliding of a wheel or track sideways across the surface during a skid steer turn.

---

### Differential drive

Think of pushing a shopping trolley with one hand on each side. Push harder on the left and the trolley turns right, and push both equally and it goes straight. A differential drive works the same way. Two wheels on one axle turn independently, and the speed difference sets the turn.

The first-principles model is kinematic. Let the wheel radius be r, the track width be b, and the angular velocities of the right and left wheels be ω_R and ω_L. The speeds of the two wheel contacts are v_R = r ω_R and v_L = r ω_L. The robot's forward speed v and its turning rate ω are:

v = (v_R + v_L) / 2

ω = (v_R − v_L) / b

With the pose (x, y, θ), the motion over a small time step dt is:

ẋ = v cos θ, ẏ = v sin θ, θ̇ = ω

Here is a worked example. A robot has wheels of radius 0.033 m and a track of 0.2 m. The right wheel turns at 15.2 rad/s and the left at 9.1 rad/s. Then v_R = 0.033 × 15.2 = 0.50 m/s and v_L = 0.033 × 9.1 = 0.30 m/s. The forward speed is 0.40 m/s, and the turning rate is ω = (0.50 − 0.30) / 0.2 = 1.0 rad/s. The robot turns left at about 57 degrees per second while it moves forward. A full circle takes about 6.3 seconds, and the turning radius is v / ω = 0.4 m.

The common mistake is to read the sign of the turn from the wiring. The motors may be mounted mirror-image, so a positive command on the left and right wheels can move the robot in opposite directions. Test each motor alone and check which way the robot turns before you write any control code. The second mistake is to assume the turn is pure rotation. A differential robot turns about the midpoint of its axle only when v = 0 and ω ≠ 0.

Differential drives appear in most educational robots, in warehouse robots that carry shelves and in robot vacuum cleaners. Their simplicity and their ability to turn in place are the reasons.

---

### Skid steer

A bulldozer turns by driving its left track forward and its right track backward. The tracks slide across the ground, and the machine rotates about its centre. A skid steer drive is the same idea with wheels or tracks on each side, and it is used where a robot must turn in a small space.

The model is the same as the differential drive, with one important change. Because the wheels scrub the surface during a turn, the effective track width is larger than the physical track width, and the actual turn is less than the ideal. The kinematic equation still holds, but the track width b is replaced by an effective value that you measure from a test turn. For a robot whose wheels scrub on carpet, the effective track can be 20 to 50 percent wider than the physical track, so the odometry from the ideal model is poor.

Here is a worked example. You command a full turn on the spot. With a physical track of 0.2 m, the ideal rotation for one turn of the wheels is a full turn of the robot after the wheels have covered the circumference of a circle with diameter 0.2 m, about 0.63 m per side. If the robot actually turns only 300 degrees for one command of 360 degrees, the effective track is about 0.2 × 360 / 300 = 0.24 m, and you use 0.24 m in your odometry.

The common mistake is to use the physical track width in the odometry of a skid steer robot. The result is an estimated heading that grows steadily wrong. The fix is to calibrate the effective track on the surface the robot will use.

Skid steer is the standard drivetrain for construction equipment, tractors with tracks and many outdoor robots, because it is simple and gives a zero turn radius.

---

### Ackermann steering

A car does not turn its two front wheels by the same angle. The inner wheel turns more sharply, because it travels a smaller circle. Ackermann steering is the geometry that makes all four wheels roll without sliding, and it is the basis of almost every car steering system.

The geometry is simple. The rear axle sits on a line through the centre of the turn, and each front wheel's axle points at the same centre. For a wheelbase L and a track width W, the angles of the inner and outer front wheels satisfy:

cot δ_inner − cot δ_outer = W / L

The turning radius to the centre of the rear axle is R = L / tan δ_outer, approximately, for small angles and a centre-of-rear-axle reference.

Here is a worked example. A small Ackermann robot has L = 0.25 m and W = 0.20 m. The outer wheel turns to δ_outer = 20 degrees. Then cot(20°) = 2.747, and cot δ_inner = 2.747 − 0.20 / 0.25 = 2.747 − 0.80 = 1.947. So δ_inner = arctan(1 / 1.947) = 27.2 degrees. The inner wheel turns about 7 degrees more than the outer wheel. The turning radius of the rear axle centre is R = 0.25 / tan(20°) = 0.69 m.

The common mistake is to steer the front wheels together with a single linkage that does not implement the geometry. Those robots drag their tyres during every turn, and the tyres wear quickly and add friction that the motor must overcome. The second mistake is to forget the minimum turning radius when planning a path. An Ackermann robot cannot turn in place, and its path planner must respect the radius.

Ackermann steering is used on RC cars, on delivery robots and on automated guided vehicles that drive on roads or in wide aisles. It is the choice when a robot must drive like a car, with a steering wheel that does not scrub.

---

### Omni wheels and mecanum wheels

A shopping trolley with four castors can move sideways if you push it hard enough, but it has to drag its castors around. A robot with omni wheels or mecanum wheels can move sideways directly, because the wheels are built to allow it.

An omni wheel has a hub with a set of small rollers around its rim. The rollers are at 90 degrees to the wheel's rolling direction, so the wheel rolls forward normally and slides sideways freely. Three omni wheels set 120 degrees apart form a holonomic base: the robot can move in any direction and turn at the same time.

A mecanum wheel has rollers set at 45 degrees to the wheel axis, usually in an X-pattern across four wheels. Each wheel contributes a force in a direction that depends on its own rotation, and the four wheels together produce a net motion in any direction. A common convention for a four-wheel mecanum base with radius r, half-length l_x and half-width l_y gives:

v_x = (r / 4) (ω_fl + ω_fr + ω_rl + ω_rr)

v_y = (r / 4) (−ω_fl + ω_fr + ω_rl − ω_rr)

ω = (r / (4(l_x + l_y))) (−ω_fl + ω_fr − ω_rl + ω_rr)

The sign convention depends on the way you number the wheels and the direction of their rollers, so check it against the physical robot.

Here is a worked example. With r = 0.05 m and all four wheels turning at 10 rad/s, the forward speed is v_x = 0.05 / 4 × 40 = 0.5 m/s, and v_y = 0 and ω = 0. The robot drives straight forward. If the front-left and rear-right wheels turn at 10 rad/s and the front-right and rear-left wheels turn at −10 rad/s, the forward speed and turn rate are both zero, and the sideways speed is v_y = 0.05 / 4 × (−10 − 10 − 10 − 10) = −0.5 m/s. Note that a wheel-speed pattern that looks like a turn can produce a sideways motion, which is the surprise that trips up most beginners.

The common mistake is to mount the mecanum wheels with the wrong roller direction, which makes the robot drift sideways when it should drive straight. Check the layout of the rollers before you assemble the base. The second mistake is odometry on a mecanum base: the wheels slip more than a differential base, and the slip is larger on smooth floors and when the robot accelerates sideways.

Omni and mecanum drives are used in warehouse robots, in robot soccer, in lab equipment that must position an object precisely and in education kits that teach holonomic motion.

---

### Tracked drives

A tank has tracks, and so does a snowmobile. A track is a continuous belt of links wrapped around wheels, and it spreads the robot's weight over a larger contact area. The track is the choice for soft ground, steep slopes and rough terrain, where wheels sink or lose grip.

The kinematics of a tracked robot is the same as a skid steer. The turn is produced by the speed difference between the two tracks, and the scrubbing of the tracks makes the odometry poor. The contact patch is long, so the effective track width changes with the load and the surface. A tracked robot is usually the hardest to calibrate.

The worked example is the ground pressure. A 10 kg robot with two tracks, each with a contact patch of 0.25 m by 0.06 m, has a total contact area of 2 × 0.25 × 0.06 = 0.03 m² on the surface. The ground pressure is 10 × 9.81 / 0.03 = 3270 Pa, or about 3.3 kPa. A wheeled robot with four wheels of 0.01 m² each in contact has a pressure of 10 × 9.81 / 0.04 = 2450 Pa. The two numbers are close, but the track spreads its load across soft ground, which matters for sinkage.

The common mistake is to fit a track to a robot that works on a hard floor, where the friction loss and the added mass make the robot slower and harder to steer. Tracks are for rough terrain. Use them only when the terrain requires them.

Tracks are common in field robots, search and rescue robots, military robots, outdoor inspection robots and some agricultural robots.

---

## Odometry and where it drifts

Odometry is the estimate of a robot's pose from its wheel rotation. For a differential drive, each encoder reading gives the distance each wheel travelled, and the pose is updated from the equations above. Odometry is cheap, fast and always available. It is also the most common cause of a robot that ends up in the wrong place.

The error has two main sources. The first is systematic error, which is caused by the robot's geometry. A wheel diameter that is 1 percent larger than the value in the code gives a distance error of 1 percent, and a track width that is 2 percent too small gives a heading error that grows with every turn. These errors are repeatable, so you can measure and correct them. The second source is non-systematic error, caused by slip, bumps, floor texture and a sudden change in friction. These errors are random, and they cannot be corrected by calibration alone.

Here is a worked example. A differential robot is given a command to drive a 1 m square, four sides with four 90-degree turns. Suppose the wheel diameter in the code is 66 mm, but the real diameter is 64 mm. The distance measured by the odometry is 66 / 64 = 1.031 times the real distance, so the robot thinks it travelled 1.031 m on each side and ends about 3 percent off. Add a turning error from a track width that is 5 percent too small, and the square closes with a gap of tens of centimetres. The two errors combine, and that is why a robot that looks accurate on a short run fails on a long one.

The common mistake is to trust odometry for position without an absolute reference such as a landmark, a beacon or a camera. Odometry is accurate over short distances and degrades with time. Module 12 covers how robots fuse odometry with other sensors to correct their pose.

Odometry is in every mobile robot, from vacuum cleaners to warehouse fleets. The first job of a robotics engineer is often to measure its error and to reduce it with calibration.

---

## Comparing the drivetrains

The table below compares the five drivetrains on the four criteria that matter in a design review. The ratings are qualitative and depend on the robot's size, mass and surface.

| Drivetrain | Manoeuvrability | Complexity | Odometry | Terrain |
|---|---|---|---|---|
| Differential | Turns in place. Cannot move sideways. | Low: two motors and a castor | Good on hard floors with calibration | Good on indoor floors, poor on gravel |
| Skid steer | Turns in place, with heavy scrub | Low to medium: two drives on each side | Poor without calibration | Good on mixed terrain |
| Ackermann | Needs a turning radius. Cannot turn in place. | Medium: steering linkage and geometry | Good, with known steering angle | Good on roads and flat floors |
| Omni (three wheels) | Holonomic. Moves in any direction. | Medium: three motors and three omni wheels | Moderate, with slip in sideways motion | Poor on rough or uneven floors |
| Mecanum (four wheels) | Holonomic. Moves in any direction. | Medium to high: four motors and roller wheels | Poor, with the most slip | Poor on rough floors, good on smooth floors |
| Tracked | Turns in place, with heavy scrub | High: track tension, drive sprockets and idlers | Poor, with large effective-track changes | Excellent on soft ground and slopes |

The table is a starting point. A team that needs a robot to carry a shelf across a warehouse will choose differential or mecanum. A team that needs a robot to climb a slope through mud will choose tracks. The choice always reflects the task, and the tradeoffs must be written down.

---

## Hands-on exercises

1. **Calibrate the wheel radius.** Mark a 2 m straight line on the floor. Drive your robot forward for exactly 2 m, measured by the tape, using only the encoder count to decide when to stop. Compare the distance you measured by tape with the distance the encoder reported. Compute the correction factor and apply it to the wheel radius in your code.

2. **Calibrate the track width.** Command the robot to turn 360 degrees in place. Count the actual number of turns, using a mark on the floor or a tape measure on the robot's rim. Compute the effective track width, as in the skid steer example, and compare it with the physical value.

3. **Measure the drift.** Drive the robot around a 1 m square taped on the floor. Record the gap between the start and the end point, in the x and y directions, and the final heading error. Repeat three times on the same surface and on a second surface, such as carpet, to see how the surface changes the result.

4. **Check the kinematics.** Command the differential robot to drive at a constant forward speed and a constant turn rate. Film it from above with a phone and compare the path with the path predicted by your Python simulator.

5. **Test a mecanum base (optional).** If you have a mecanum kit, drive each wheel alone and record the direction of the robot's motion. Compare the four directions with the equations above. Then command a sideways motion and check that the robot moves sideways without rotating.

---

## Project: a drivetrain simulator and drift study

This project is a Python simulator for three drivetrains: differential, Ackermann and four-wheel mecanum. It predicts the pose of each drivetrain from wheel commands, and it includes a simple noise model that shows how odometry drifts. It is a full mini-project, because the simulator is the tool you will use in Module 8 to test control code before you run it on hardware.

**Goal:** write a simulator that takes a sequence of wheel commands for each drivetrain, integrates the pose over time, and writes a plot of the path and a table of the final pose error under calibration errors.

**Repository name:** `100-days-robotics-day-36-robot-drivetrains`

**Repo layout:**

```
100-days-robotics-day-36-robot-drivetrains/
├── README.md
├── requirements.txt
├── drivetrains/
│   ├── __init__.py
│   ├── differential.py
│   ├── ackermann.py
│   ├── mecanum.py
│   └── integrate.py
├── drift_study.py
├── tests/
│   ├── test_differential.py
│   ├── test_ackermann.py
│   └── test_mecanum.py
└── data/
    └── square_path.json
```

**Starter code.** Write `drivetrains/differential.py`:

```python
# drivetrains/differential.py
import math


def body_velocity(v_right, v_left, track):
    """Forward speed (m/s) and turn rate (rad/s) from wheel contact speeds and track width."""
    v = (v_right + v_left) / 2.0
    omega = (v_right - v_left) / track
    return v, omega


def step_pose(pose, v, omega, dt):
    """Advance pose (x, y, theta) by one time step using a simple Euler update."""
    x, y, theta = pose
    x += v * math.cos(theta) * dt
    y += v * math.sin(theta) * dt
    theta += omega * dt
    return (x, y, theta)
```

Write `drivetrains/ackermann.py` for the inner wheel angle:

```python
# drivetrains/ackermann.py
import math


def inner_angle(outer_angle, wheelbase, track):
    """Inner front wheel angle (radians) from the outer angle (radians), wheelbase and track."""
    cot_inner = 1.0 / math.tan(outer_angle) - track / wheelbase
    return math.atan(1.0 / cot_inner)


def turning_radius(outer_angle, wheelbase):
    """Turning radius of the rear axle centre (metres) for the outer wheel angle (radians)."""
    return wheelbase / math.tan(outer_angle)
```

Write `drivetrains/mecanum.py` for the forward kinematics of a four-wheel base, using the convention in this lesson:

```python
# drivetrains/mecanum.py
def body_velocity(w_fl, w_fr, w_rl, w_rr, r, lx, ly):
    """Body velocity (vx, vy, omega) from four wheel angular speeds in rad/s."""
    vx = r / 4.0 * (w_fl + w_fr + w_rl + w_rr)
    vy = r / 4.0 * (-w_fl + w_fr + w_rl - w_rr)
    omega = r / (4.0 * (lx + ly)) * (-w_fl + w_fr - w_rl + w_rr)
    return vx, vy, omega
```

Write `drivetrains/integrate.py` with a function that runs a list of commands and returns the path. Use it for all three drives.

**Step-by-step tasks.**

1. Create the repository and the virtual environment. Install numpy, matplotlib and pytest.
2. Implement the four files above and the integrator. Check each function by hand with the worked examples. The differential example should give v = 0.40 m/s and ω = 1.0 rad/s. The Ackermann example with L = 0.25 m, W = 0.20 m and δ_outer = 20 degrees should give δ_inner = 27.2 degrees and R = 0.69 m. The mecanum example with all wheels at 10 rad/s should give v_x = 0.5 m/s.
3. Write `tests/test_differential.py`. Test a straight drive, a pure rotation with v_R = −v_L, and the worked example.
4. Write `tests/test_ackermann.py`. Test the inner angle for the worked example, and test that the inner angle is larger than the outer angle for any positive steering angle.
5. Write `tests/test_mecanum.py`. Test a straight drive, a pure sideways drive, and a pure rotation, and check each against the equations.
6. Write `drift_study.py`. It reads `data/square_path.json`, which lists wheel commands for a 1 m square. It runs the simulator twice, once with the true wheel radius and once with a radius that is 3 percent too large, and it reports the final gap. Add a second run with a track width 5 percent too small. Print a table and save a plot to `drift.png`.
7. Write the README. It must describe the models, the assumptions and their limits, including the fact that the simulator has no slip and no friction.

**README checklist:**

- [ ] Installation steps and the Python version used
- [ ] How to run `drift_study.py` and where the outputs are saved
- [ ] The equations for each drivetrain, with the source for each
- [ ] A table of the worked examples with the hand values and the script values
- [ ] The measured drift from the hands-on exercises, next to the simulated drift

**Done when:**

- `pytest` passes all tests
- `drift_study.py` prints the final gap for the three error cases and saves `drift.png`
- The script values agree with the hand values for every worked example to within 1 percent
- The README states the measured drift from your real robot and compares it with the simulated drift
- You can explain, in writing, which drivetrain you would choose for a warehouse robot and why

**What to commit to GitHub:** the source tree, the test output as a pasted block in the README, `drift.png`, the printed drift table, and your measurements from the square test with the surface you used. If you did not have a physical robot, say so in the README and commit only the simulator results.

**Why this project is full and not light:** the simulator is the tool you will reuse for control and navigation in Modules 8 and 12, and the drift study is the first measurement of odometry error that you will make.

---

## Check your understanding

1. A differential robot has a track of 0.25 m. Its right wheel moves at 0.4 m/s and its left wheel at 0.6 m/s. Find the forward speed and the turn rate, and state which way the robot turns.

2. An Ackermann robot has a wheelbase of 0.3 m and a track of 0.22 m. The outer wheel is at 25 degrees. Estimate the inner wheel angle, and explain why the inner wheel must turn more than the outer.

3. A mecanum base with r = 0.05 m and l_x + l_y = 0.2 m has its front-left and rear-right wheels at +5 rad/s and its front-right and rear-left wheels at −5 rad/s. Compute the motion of the robot, and explain why a beginner might expect it to turn.

4. A differential robot drives a 2 m straight line and reports 2.04 m from its encoders. Explain whether the error is systematic or random, and describe how you would correct it.

5. A skid steer robot on carpet turns 300 degrees for a command of 360 degrees. Explain what causes the difference, and state the change you would make to the odometry.

6. Explain why a tracked robot is not the best choice for an indoor office, even though it has the best terrain performance.

**Answers:**

1. The forward speed is (0.4 + 0.6) / 2 = 0.5 m/s. The turn rate is (0.4 − 0.6) / 0.25 = −0.8 rad/s. The negative sign means the robot turns clockwise, toward the right, because the left wheel is faster. The sign convention depends on how you define the axes, so state it in your code.

2. cot(25°) = 2.145. cot δ_inner = 2.145 − 0.22 / 0.30 = 2.145 − 0.733 = 1.412. So δ_inner = arctan(1 / 1.412) = 35.3 degrees. The inner wheel travels a smaller circle, so it must turn more sharply to roll without sliding.

3. With the wheel speeds (fl, fr, rl, rr) = (5, −5, −5, 5), the forward speed is v_x = 0.05 / 4 × (5 − 5 − 5 + 5) = 0. The sideways speed is v_y = 0.05 / 4 × (−5 − 5 − 5 − 5) = −0.25 m/s. The turn rate is ω = 0.05 / (4 × 0.2) × (−5 − 5 + 5 + 5) = 0. The robot slides sideways at 0.25 m/s without turning. A beginner expects a turn, because diagonal wheels in opposite directions look like the pattern of a differential robot, but the mecanum rollers turn that pattern into a sideways slide.

4. The error is systematic, because the same error appears on every run with the same wheels and the same surface. The error is 2 percent, which suggests the wheel radius in the code is about 2 percent too small, or the encoder count per revolution is wrong. Measure the real wheel diameter, or run a calibration move and compute the correction factor from the ratio of the measured and reported distances.

5. The difference is scrubbing. The wheels slide sideways during the turn, so the real turn is smaller than the ideal. The effective track width is larger than the physical track, and the calibrated effective track of 0.2 × 360 / 300 = 0.24 m should be used in the odometry for that surface.

6. Tracks have high rolling resistance and scrub on hard floors, which wears the floor and makes the robot hard to steer accurately. Their odometry is poor, and their noise and weight are high for an indoor environment. A differential or omni drive on an indoor floor gives better precision, lower cost and lower noise.

---

## Sources to read and watch

**Read**

- *Wikipedia*: "Differential wheeled robot", "Ackermann steering geometry", "Omni wheel", "Mecanum wheel", "Skid-steering" (search for "Skid steer" if the title differs), "Odometry" and "Holonomic"
- *Wikipedia*: "Rudolph Ackermann" for the history of the steering geometry
- *Carnegie Mellon University*: the course notes on mobile robots from the Robotics Institute. Search for "CMU 16-311 Introduction to Robotics" for lecture notes on mobile kinematics.
- *Pololu*: the product documentation for its wheel and motor kits explains the wheel radius and encoder counts for their hardware. Search for "Pololu Romi" or "Pololu 3pi+" for the documentation pages.

**Tutorials**

- *GeeksforGeeks*: search for "Differential drive robot" and "Mecanum wheel" for worked examples
- *W3Schools*: the Python tutorial on w3schools.com for the basics of the code

**Watch**

- *MIT OpenCourseWare*: the lecture series for course 2.12 Introduction to Robotics, which covers mobile robot kinematics. Search for the course on ocw.mit.edu.
- *Stanford University*: the Stanford Artificial Intelligence Laboratory has public lectures on mobile robots on its YouTube channel. Search for "Stanford mobile robot kinematics".

**Practice**

- *Google Colab*: create a notebook that runs your differential drive simulator and plots the path. The free tier is enough for this project.

---

## Tomorrow

Tomorrow, on Day 37, you open a CAD tool for the first time. You learn sketches, constraints, dimensions, reference geometry, parametric modelling and design intent. The drivetrain you chose today sets the wheel size, the track width and the wheel mounting that you must draw. A sketch that does not match the kinematics of the robot is the first error that Day 37 teaches you to avoid.
