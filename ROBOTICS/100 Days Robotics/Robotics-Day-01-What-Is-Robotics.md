# Day 1: What Is Robotics?

*100 Days of Robotics Engineering, Module 1: Robotics Engineering Foundations*

**Time:** about 4 hours
**Level:** beginner. No electronics or robotics experience is needed. Some Python helps for the project, and the project explains every idea it uses.

**By the end of this day, you will have:**

- A working definition of a robot, and a block diagram of its sense, compute, decide and act loop
- A simulated differential-drive robot in Python that drives to a goal open loop and closed loop, and saves a plot of both paths
- A table that decomposes five real robots into their parts

**What you need:**

- Python 3.10 or newer. Python 3.12 is recommended. Check with `python --version`.
- `matplotlib` 3.9 or newer, installed in the project's virtual environment. It was tested with 3.11.
- No hardware. The project runs on Windows, macOS or Linux.
- Optional, for later: ROS 2 Jazzy Jalisco (an LTS release from May 2024, supported until May 2029) on Ubuntu 24.04. On Windows 11, use Ubuntu 24.04 inside WSL 2. You do not need it today.
- Optional, for later: Gazebo Harmonic (free, open source, paired with Jazzy) or MuJoCo (free, open source, Apache 2.0 licence). Neither is needed today.

**Prerequisites:**

- Basic Python: variables, functions, loops, lists and importing a module
- Comfort with a terminal: changing folders and running a script
- The idea of an angle, and the sine and cosine functions. The project explains how they are used.

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [A short history, so the vocabulary makes sense](#a-short-history-so-the-vocabulary-makes-sense)
- [Robots, automation and mechatronics](#robots-automation-and-mechatronics)
- [The sense, compute, decide and act loop](#the-sense-compute-decide-and-act-loop)
- [Open loop and closed loop](#open-loop-and-closed-loop)
- [Differential drive: how two wheels steer a body](#differential-drive-how-two-wheels-steer-a-body)
- [Sensors](#sensors)
- [Actuators](#actuators)
- [Compute](#compute)
- [Types of robots](#types-of-robots)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: a simulated differential-drive robot on your laptop](#project-a-simulated-differential-drive-robot-on-your-laptop)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot is a machine that senses its environment, decides what to do, and acts on the world. That loop is the whole field in one sentence. Sensing, computation, decision and action repeat many times per second.

A washing machine that adjusts its spin when the drum is unbalanced is a robot in this sense. So is a warehouse arm that picks boxes, and so is a delivery drone. They differ in how much they sense, how much they decide, and how much they can change in the world.

Robotics engineers combine mechanics, electronics, embedded software, control theory and, more and more, machine learning. Today gives you the vocabulary and one working model you can run on a laptop. Day 2 opens the hardware.

[Wikipedia: Robotics](https://en.wikipedia.org/wiki/Robotics) is the reference for today. Read the introduction and the section on "Components of robots."

---

## A short history, so the vocabulary makes sense

| Period | What happened | Why it matters today |
|---|---|---|
| 1921 | Karel Čapek's play *R.U.R.* introduces the word "robot," from the Czech *robota* (forced labour). | The word arrives before the machines do. |
| 1961 | The [Unimate](https://en.wikipedia.org/wiki/Unimate) arm, built from a patent by George Devol, starts work on a General Motors assembly line. | The first industrial robot in production. It repeated a fixed sequence with almost no sensing. |
| 1966 to 1972 | SRI's Shakey robot combines a camera, a planner and wheels. | Shows the sense, plan, act loop in one machine, and how slow it was. |
| 1997 | NASA's Sojourner rover drives on Mars during the Pathfinder mission. | A mobile robot working far from any human, with command delays of many minutes. |
| 2002 | iRobot ships the Roomba vacuum cleaner. | Cheap mobile robots reach homes. A narrow task is enough for a useful product. |
| 2007 | The Robot Operating System (ROS) is first released. ROS 2 is its current successor. | The shared software layer that most research robots and many commercial ones use. |

**Takeaway:** robotics has moved from fixed motions on a factory floor, to machines that sense and plan, to shared software frameworks. Each step added sensing, and sensing is what separates a robot from a machine that only repeats.

---

## Robots, automation and mechatronics

A dishwasher runs the same program whatever is inside it. A cleaning robot that stops when it bumps a chair and turns left reacts to what it senses. That reaction is the difference between automation and robotics.

Automation is a fixed process that runs without a human, and it may sense nothing. A timer that switches on a water pump every morning is automation. Robotics adds sensing and decisions, so the machine can cope with variation, such as a robot arm that finds a part on a conveyor even when the part's position shifts. Mechatronics is the design discipline that combines mechanics, electronics and software into one system. A sealed motor with a built-in driver and encoder is mechatronics, and most robots are mechatronic systems.

A common mistake is calling any machine with a motor a robot. To spot a robot, ask whether the machine senses something and changes its action because of it. If not, you are looking at automation.

In the market, warehouse systems such as those from Amazon Robotics mix fixed conveyors (automation) with mobile robots that sense their route (robotics).

## The sense, compute, decide and act loop

A thermostat reads the room temperature, compares it with the setting, and switches the heater on or off. Every robot runs the same kind of loop, with more sensors and harder decisions.

Sensors turn the world into numbers. A computer processes those numbers and picks an action. Actuators such as motors, servos, valves and grippers change the world. Feedback is the robot sensing the result of its action, which starts the next pass through the loop.

```
   ┌──────────┐    raw data    ┌──────────────┐   commands   ┌───────────┐
   │ SENSORS  │ ─────────────▶ │ COMPUTE +    │ ───────────▶ │ ACTUATORS │
   │ (eyes,   │                │ DECISIONS    │              │ (motors,  │
   │  touch,  │ ◀───────────── │ (controller, │              │  pumps,   │
   │  motion) │   feedback     │  planner)    │              │  grippers)│
   └──────────┘                └──────────────┘              └─────┬─────┘
        ▲                                                         │
        └──────────────── the physical world changes ─────────────┘
```

The physical world closes the loop. A program that never checks its effect against reality is only a program.

A common mistake is drawing the loop as a one-way pipeline from sensors to planner to motors. That drops the feedback arrow, and the design drifts as soon as the world does something unexpected. To spot it, check that every actuator has a path back to a sensor that reports its effect.

In the market, a Roomba and a Tesla Optimus prototype both run this loop. What differs is the sensor set and how fast the decision step must run.

## Open loop and closed loop

A toaster with a timer is open loop. You set three minutes and the toaster assumes three minutes have passed, whatever the bread looks like. A toaster that stops when a sensor sees the crust turn brown is closed loop.

An open-loop controller sends a command and never checks the result. A stepper motor told to take 200 steps assumes it took 200 steps, even if it stalled. A closed-loop controller measures the result and corrects the command. A DC motor with an encoder (a sensor that counts shaft rotations) compares its actual speed with the target and adjusts.

The test is whether the reading changes the command. A sensor that only writes to a log file does not make a loop closed:

```python
motor.run(200)                    # open loop: trust the step count

while encoder.degrees() < 90:     # closed loop: the reading decides when to stop
    motor.run(1)
```

A common mistake is calling a system closed loop because it has a sensor. Closed loop costs more hardware and computation, and it is worth that cost whenever precision or safety matters. In the market, 3D printers often use open-loop stepper motors, while industrial arms use closed-loop servo motors in every joint.

## Differential drive: how two wheels steer a body

Pushing a shopping cart with two hands shows the idea. Push both handles equally and the cart goes straight. Push harder on one side and it curves. A differential-drive robot has two independently driven wheels on a common axle, and the speed difference between them sets the turn.

The body's forward speed is the average of the two wheel speeds, and its turn rate depends on the difference. If the left wheel moves at `v_L` metres per second and the right at `v_R`, then the forward speed is `v = (v_R + v_L) / 2`, and the turn rate is `omega = (v_R - v_L) / L` radians per second, where `L` is the distance between the wheel centres. Equal and opposite wheel speeds make the body spin in place.

```
          front
   +--------------------+
   |                    |
 [L]  <-- L apart -->  [R]     v_L = left wheel speed
   |         (x, y)     |       v_R = right wheel speed
   +--------------------+
          back
```

The robot's pose is its position `(x, y)` and its heading `theta`. Each time step, `x` changes by `v cos(theta)` times the step, `y` changes by `v sin(theta)` times the step, and `theta` changes by `omega` times the step. Estimating pose this way from wheel motion is called dead reckoning, and the project uses exactly these three updates.

A common mistake is assuming the wheels move the robot exactly as commanded. Wheels slip and tyres wear unevenly, so errors accumulate. To spot it, command a straight two-metre drive and measure the endpoint. An offset of 10 cm means the odometry is drifting.

In the market, Roomba-style vacuums, most warehouse carts and the TurtleBot3 education robot use differential drive, because it is cheap, compact and can turn in place.

## Sensors

Every robot you will meet is built from the same three families of parts (sensors, actuators and compute), plus a frame and a power source. The loop tells you what each family is for.

A sensor turns a quantity in the physical world into a signal a computer can read. Start from the question the robot must answer: where am I, what is in front of me, or is my arm level?

Common sensors include wheel encoders, which count rotations; an inertial measurement unit (IMU), which measures acceleration and rotation rate; LiDAR, which measures distance with laser pulses; cameras; and touch or force sensors. Each has errors. An encoder drifts when a wheel slips, and an IMU drifts when integrated over time, so good designs combine sensors whose errors differ.

A common mistake is assuming a reading is the truth. To spot a bad sensor, record it while the robot stays still. If the value wanders, the sensor has noise or bias.

In the market, the IMUs in smartphones and quadcopter flight controllers, such as those from DJI, use the same family of sensor.

## Actuators

An actuator converts a command into motion or force. A DC motor is fast and simple but needs an encoder to be precise. A servo is a motor with built-in position feedback. A stepper moves in fixed increments and is open loop unless it has an encoder. Pumps, valves and grippers are actuators too.

The actuator sets the limits. A motor with too little torque cannot lift the arm it is mounted on, however good the controller is. Check speed, torque and power before choosing a computer.

A common mistake is choosing the controller first and fitting the motor later. To spot it, compare the motor's rated torque with the load it must move, including the robot's own weight.

In the market, industrial arms from manufacturers such as FANUC and ABB use high-torque servo motors with encoders in every joint.

## Compute

The compute layer takes sensor data and decides what the actuators do. It can be a microcontroller such as the ATmega328P on an Arduino Uno, a single-board computer such as a Raspberry Pi, a GPU, or a mix. A microcontroller runs tight control loops with predictable timing. A Raspberry Pi runs full Linux and can host ROS 2. A GPU runs neural networks for vision.

The main question is how fast each decision must happen. A balancing robot needs its control loop at a few hundred hertz, which a microcontroller handles well. A vision model may run only a few frames per second on a bigger board.

A common mistake is putting everything on one board because it looks simpler. To spot it, check whether a slow vision task delays motor control. If it does, split the work.

In the market, many commercial mobile robots pair a microcontroller for motor control with a larger computer for perception and planning.

---

## Types of robots

Robots come in families that share sensors and actuators. The table lists six common families.

| Type | Typical sensing | Typical actuation | Where you see it |
|---|---|---|---|
| Mobile robot | Wheel encoders, LiDAR or sonar, IMU, camera | Wheel motors | Warehouse carts, vacuum cleaners, delivery robots |
| Manipulator (robot arm) | Joint encoders, force sensing, camera | Joint motors, gripper | Factory assembly, surgery, pick-and-place |
| Drone (UAV) | IMU, GPS, barometer, camera | Propeller motors | Inspection, mapping, delivery pilots |
| Humanoid | IMU, joint encoders, cameras, force sensors | Many joint motors | Research platforms, early commercial pilots |
| Industrial robot | Joint encoders, safety sensors | Large servo arms | Car manufacturing, welding |
| Service robot | Cameras, microphones, touch sensors | Wheels, arms, speakers | Hotels, hospitals, cleaning |

Pick one robot and ask which of the four loop jobs is hardest. For a mobile robot it is often state estimation. For a manipulator it is control and kinematics. For a humanoid it is balance.

**Reference images to find:** search "robot types diagram" and compare a manipulator with a mobile robot. Notice how many more joints the arm has, and how the mobile robot needs navigation the arm does not.

**Video (about 15 minutes):** [Boston Dynamics on YouTube](https://www.youtube.com/@BostonDynamics). Watch a range of their robots, then pause and identify the sensors and actuators you can see.

---

## Hands-on exercises (about 60 minutes)

**Exercise 1: Decompose five robots (about 40 minutes).**

1. Pick five robots from videos, products or real life. Include at least one mobile robot and one arm.
2. Create `notes/robot-types.md` with this table header and one row per robot:

   | Robot | Sensors | Compute / controller | Algorithms (what decides) | Actuators | Mechanics | Power | Feedback loop |
   |---|---|---|---|---|---|---|---|

3. Fill each cell from what you can see or read. Write "not visible" when you cannot tell. Do not invent sensors.
4. In the algorithms column, write what the robot decides, such as "follows a line" or "avoids obstacles." Do not write "AI" unless you can say what it learns.
5. Check each row against a product page or manufacturer video. Mark unconfirmed cells with a question mark.

**Exercise 2: Open or closed loop (about 20 minutes).**

1. List five everyday machines: a kettle, a washing machine, a car's cruise control, a microwave and a dishwasher.
2. For each, write `open loop` or `closed loop`, and name the sensor that makes it closed loop, if any.
3. Compare with the open-loop and closed-loop section. If you wrote closed loop for a machine whose reading never changes its command, reread the test in that section.

---

## Project: a simulated differential-drive robot on your laptop

This project needs no robot. You will simulate a two-wheeled robot, drive it toward a goal in two ways, and measure how far each run ends from the target. Both runs face the same wheel errors, so the difference comes only from feedback.

**Goal:** a Python program that moves a simulated differential-drive robot to a goal point, first open loop (plan once, then trust the wheels) and then closed loop (correct the course every step).

### Repository layout

```
100-days-robotics-day-01-what-is-robotics/
├── README.md          # results, and answers to the questions in Step 6
├── diff_drive.py      # the robot's kinematics: wheel speeds in, pose out
├── sim.py             # open-loop and closed-loop runs, and the plot
├── requirements.txt   # the one library the project needs
└── paths.png          # the plot produced by sim.py, committed as evidence
```

### Step 1: create the repository and a virtual environment

Run these from the folder where you keep projects:

```bash
mkdir 100-days-robotics-day-01-what-is-robotics
cd 100-days-robotics-day-01-what-is-robotics
git init
python -m venv .venv
```

Activate it:

- macOS or Linux: `source .venv/bin/activate`
- Windows (PowerShell): `.venv\Scripts\Activate.ps1`
- Windows (Command Prompt): `.venv\Scripts\activate.bat`

Create `requirements.txt` with this one line, then install it:

```text
matplotlib>=3.9
```

```bash
pip install -r requirements.txt
```

### Step 2: the robot's kinematics (`diff_drive.py`)

Create `diff_drive.py`. The class holds the pose and updates it from two wheel speeds, using the formulas from the differential-drive section.

```python
"""Differential-drive kinematics: two wheel speeds decide how the body moves."""

from __future__ import annotations

import math
from dataclasses import dataclass


@dataclass
class Pose:
    x: float = 0.0       # metres
    y: float = 0.0       # metres
    theta: float = 0.0   # heading in radians; 0 means facing the +x axis


class DiffDriveRobot:
    def __init__(self, wheel_base: float = 0.3):
        self.wheel_base = wheel_base  # distance between the two wheel centres, metres
        self.pose = Pose()

    def step(self, v_left: float, v_right: float, dt: float) -> None:
        """Advance the pose by dt seconds, given wheel speeds in metres per second."""
        v = (v_right + v_left) / 2.0                  # forward speed of the body
        omega = (v_right - v_left) / self.wheel_base  # turn rate in radians per second
        p = self.pose
        p.x += v * math.cos(p.theta) * dt
        p.y += v * math.sin(p.theta) * dt
        p.theta = (p.theta + omega * dt) % (2 * math.pi)


if __name__ == "__main__":
    # Sanity check: equal wheel speeds of 0.5 m/s for 1 second should move the robot 0.5 m along +x.
    robot = DiffDriveRobot()
    for _ in range(100):
        robot.step(0.5, 0.5, 0.01)
    print(f"straight: x = {robot.pose.x:.3f} m, y = {robot.pose.y:.3f} m")
```

### Step 3: check the kinematics

```bash
python diff_drive.py
```

Expected output:

```text
straight: x = 0.500 m, y = 0.000 m
```

If `x` is not 0.5, check that `x` uses `cos` and `y` uses `sin`. Swapping them is the most common mistake here, and the robot then drives sideways.

### Step 4: the open-loop and closed-loop runs (`sim.py`)

Create `sim.py`. The open-loop function plans once: turn in place to face the goal, then drive straight for a time set by the distance. The closed-loop function re-plans every step from the current pose. Each wheel has a fixed speed error for the whole run, drawn from the same seed in both runs.

```python
"""Drive a simulated differential-drive robot to a goal, open loop and closed loop."""

import math
import random

import matplotlib

matplotlib.use("Agg")  # draw to a file only, so no display window is needed
import matplotlib.pyplot as plt  # noqa: E402

from diff_drive import DiffDriveRobot

WHEEL_BASE = 0.3      # metres between the wheel centres
SPEED = 0.2           # cruise speed, metres per second
TURN_RATE = 0.5       # rate used to turn in place, radians per second
DT = 0.02             # simulation step, seconds
GOAL = (2.0, 1.0)     # target point, metres
TOLERANCE = 0.03      # the robot counts as arrived inside this radius, metres
MAX_TIME = 60.0       # stop after this many simulated seconds
WHEEL_BIAS = 0.10     # each wheel's real speed is off by up to 10 percent, fixed for a run


def wrap(angle):
    """Map an angle to the range -pi to pi."""
    return math.atan2(math.sin(angle), math.cos(angle))


def draw_wheel_bias(rng):
    """Return the real-to-commanded speed factor for the left and right wheels."""
    return 1 + rng.uniform(-WHEEL_BIAS, WHEEL_BIAS), 1 + rng.uniform(-WHEEL_BIAS, WHEEL_BIAS)


def run_open_loop(rng):
    """Plan once from the start: turn to face the goal, then drive straight for a fixed time."""
    robot = DiffDriveRobot(WHEEL_BASE)
    left, right = draw_wheel_bias(rng)
    goal_heading = math.atan2(GOAL[1], GOAL[0])
    turn = TURN_RATE if goal_heading >= 0 else -TURN_RATE
    plan = [
        (-turn * WHEEL_BASE / 2, turn * WHEEL_BASE / 2, abs(goal_heading) / TURN_RATE),
        (SPEED, SPEED, math.hypot(*GOAL) / SPEED),
    ]
    path = [(robot.pose.x, robot.pose.y)]
    for v_left, v_right, duration in plan:
        for _ in range(round(duration / DT)):
            robot.step(v_left * left, v_right * right, DT)
            path.append((robot.pose.x, robot.pose.y))
    return path


def run_closed_loop(rng):
    """Re-plan every step from the current pose, so feedback corrects the wheel bias."""
    robot = DiffDriveRobot(WHEEL_BASE)
    left, right = draw_wheel_bias(rng)
    path = [(robot.pose.x, robot.pose.y)]
    for _ in range(round(MAX_TIME / DT)):
        p = robot.pose
        dx, dy = GOAL[0] - p.x, GOAL[1] - p.y
        if math.hypot(dx, dy) < TOLERANCE:
            break
        error = wrap(math.atan2(dy, dx) - p.theta)
        turn = max(-1.0, min(1.0, 2.0 * error))     # proportional steering, capped
        speed = SPEED if abs(error) < 0.5 else 0.0  # stop to turn when far off heading
        v_left = speed - turn * WHEEL_BASE / 2
        v_right = speed + turn * WHEEL_BASE / 2
        robot.step(v_left * left, v_right * right, DT)
        path.append((robot.pose.x, robot.pose.y))
    return path


def distance_to_goal(path):
    x, y = path[-1]
    return math.hypot(GOAL[0] - x, GOAL[1] - y)


def main():
    # Both runs use the same seed, so they face the same wheel bias.
    open_path = run_open_loop(random.Random(7))
    closed_path = run_closed_loop(random.Random(7))

    print(f"open loop:   ends {distance_to_goal(open_path):.3f} m from the goal")
    print(f"closed loop: ends {distance_to_goal(closed_path):.3f} m from the goal")

    fig, ax = plt.subplots(figsize=(6, 5))
    for path, label in [(open_path, "open loop"), (closed_path, "closed loop")]:
        xs, ys = zip(*path)
        ax.plot(xs, ys, label=label)
    ax.plot([GOAL[0]], [GOAL[1]], "k*", markersize=12, label="goal")
    ax.set_aspect("equal")
    ax.set_xlabel("x (m)")
    ax.set_ylabel("y (m)")
    ax.set_title("Differential-drive paths with wheel bias")
    ax.legend()
    fig.savefig("paths.png", dpi=120)
    print("saved paths.png")


if __name__ == "__main__":
    main()
```

### Step 5: run the simulation

```bash
python sim.py
```

Expected output:

```text
open loop:   ends 0.362 m from the goal
closed loop: ends 0.027 m from the goal
saved paths.png
```

The open-loop robot misses the goal by more than 30 cm because its wheels never match their commands. The closed-loop robot ends inside the 3 cm tolerance. If your numbers differ, check that `WHEEL_BIAS` is 0.10 and both runs use `random.Random(7)`. Open `paths.png`: the open-loop path should bend away from the straight line, and the closed-loop path should curve back to the star.

### Step 6: write the README

Create `README.md`. Paste the two printed lines at the top, then answer these questions in full sentences.

1. How far did each run end from the goal? Which one is closer, and by how much?
2. Why do both runs face the same wheel errors? What would change with a different seed?
3. The closed loop reads the robot's true pose directly. Why is that unrealistic, and what would a real robot read instead?
4. Set `WHEEL_BIAS` to 0.0 and rerun. Does the open loop still miss? What does that say about where its error comes from?
5. Name one sensor on a real robot that would let the open-loop plan correct itself, and what it would measure.

### Done when

- `python diff_drive.py` prints `straight: x = 0.500 m, y = 0.000 m`.
- `python sim.py` prints both distances, and the closed-loop distance is under the 0.03 m tolerance.
- `paths.png` is in the repository and shows both paths and the goal.
- The README includes the printed output and answers all five questions.
- You can explain, without notes, why the closed-loop run reaches the goal and the open-loop run does not.
- The repository is pushed to GitHub with the four files above, under the name `100-days-robotics-day-01-what-is-robotics`.

---

## Check your understanding

1. What are the four jobs every robot performs in its loop?
2. Give one example of automation that is not robotics, and say why.
3. A robot has a wheel encoder, but its controller only writes the reading to a log file. Is the loop open or closed, and why?
4. In the differential-drive formulas, what is the body's forward speed when the wheels turn at equal speed in opposite directions?
5. Why would a robot with a fast balance loop and a slow vision task usually split its computing across two boards?
6. Why does the open-loop robot in the project end off target even though its code is correct?

**Answers (check after you try):**

1. Sense, decide (compute), act, and receive feedback from the result of the action.
2. A timer that switches on a water pump every morning. It senses nothing and adapts to nothing, so it is automation.
3. Open loop. The reading never changes a command, so nothing is corrected. Logging is not feedback.
4. Zero. The average of equal and opposite speeds is zero, so the body spins in place.
5. A slow vision model on the same board can delay the timing-critical control loop. A microcontroller handles the control, and a larger computer handles perception.
6. Its plan assumes each wheel moves exactly as commanded. The project gives each wheel a speed error of up to 10 percent, so the robot drifts, and without feedback the error is never corrected.

---

## Sources to read and watch

**Read:**
- [Wikipedia: Robotics](https://en.wikipedia.org/wiki/Robotics)
- [Wikipedia: Mechatronics](https://en.wikipedia.org/wiki/Mechatronics)
- [Wikipedia: Closed-loop controller](https://en.wikipedia.org/wiki/Closed-loop_controller)
- [Wikipedia: Dead reckoning](https://en.wikipedia.org/wiki/Dead_reckoning)
- [Wikipedia: Unimate](https://en.wikipedia.org/wiki/Unimate), for the history table

**Tutorials:**
- [Matplotlib documentation](https://matplotlib.org/stable/): the "Quick start" page, for plotting and saving figures.
- [Python venv documentation](https://docs.python.org/3/library/venv.html): the first section, for Step 1.
- Search "differential drive kinematics" and read one university or course page that shows the same pose equations used in the project.

**Watch:**
- [Boston Dynamics on YouTube](https://www.youtube.com/@BostonDynamics): a range of robot types and the sensors visible in each.
- [Articulated Robotics on YouTube](https://www.youtube.com/@ArticulatedRobotics): search the channel for a short introduction to a wheeled mobile robot.
- [The Construct on YouTube](https://www.youtube.com/@TheConstruct): search the channel for one introductory ROS 2 video. Watch it for the vocabulary only.

**Practice:**
- [ROS 2 Jazzy documentation: Tutorials](https://docs.ros.org/en/jazzy/Tutorials.html): read the list of beginner tutorials. Do not install ROS 2 yet.
- [Gazebo documentation](https://gazebosim.org/docs): find the Harmonic release, which pairs with Jazzy. Read the introduction only.
- [MuJoCo documentation](https://mujoco.org): read the overview, which describes it as a free, open-source physics simulator.

**Note on links:** open each one once. If a page has moved, search its title on the site named in the entry.

---

## Tomorrow

Day 2, **Anatomy of a Robot**, opens the box. You will map each block from today's loop to real hardware: microcontrollers such as the Arduino Uno, single-board computers such as the Raspberry Pi, motor drivers, batteries and frames. Bring your five-robot table, because tomorrow you will compare each robot's parts with parts you can buy.
