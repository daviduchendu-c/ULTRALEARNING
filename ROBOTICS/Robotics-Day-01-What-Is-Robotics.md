# Day 1: What Is Robotics?

*100 Days of Robotics Engineering, Module 1: Robotics Engineering Foundations*

**Time:** about 3 to 4 hours
**You will build:** a block diagram of a robot you can see every day, plus an architecture sketch of your own robot idea.
**You should know:** nothing about electronics yet. Basic reading and a bit of curiosity are enough.

---

## 1. The big idea

A robot is a machine that senses its environment, decides what to do, and acts on the world. That loop is the whole field in one sentence. Sensing, computation, decision and actuation repeat many times per second.

A washing machine that adjusts its spin when the drum is unbalanced is a robot in this sense. So is a warehouse arm that picks boxes, and so is a delivery drone. The difference between them is how much they sense, how much they decide, and how much they can change in the world.

Robotics engineers combine mechanics, electronics, embedded software, control theory and, increasingly, machine learning. Most jobs sit in one or two of these areas, but you will be expected to talk across all of them.

Start with the [Wikipedia article on robotics](https://en.wikipedia.org/wiki/Robotics) and read the introduction and the section on "Components of robots."

---

## 2. Robotics, automation and mechatronics

These three words are used loosely. Here is a working distinction.

| Term | What it means | Example |
|---|---|---|
| Automation | A fixed process runs without a human. It may not sense anything. | A timer that switches on a water pump every morning. |
| Robotics | A machine that senses, decides and acts, usually with some flexibility. | A robot arm that finds a part on a conveyor even when its position shifts. |
| Mechatronics | The design discipline that combines mechanics, electronics and software into one system. | A sealed motor with an integrated driver and encoder. |

Robotics usually sits inside mechatronics. When people say "robotics engineer," they often mean someone who works across the boundaries.

---

## 3. The sense, compute, decide, act loop

Every robot has four jobs, and most architecture decisions are about how to do them well.

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

- **Sensing:** cameras, encoders, distance sensors, inertial sensors and microphones turn the world into numbers.
- **Computation and decisions:** a microcontroller, single-board computer or GPU processes those numbers and picks an action.
- **Actuation:** motors, servos, valves and grippers change the world.
- **Feedback:** the robot senses the result of its action, and the loop continues.

Notice that the physical world closes the loop. A robot that is not connected to reality through feedback is just a program.

---

## 4. Open loop vs closed loop

This distinction matters for every control day later in the series.

- **Open loop:** the controller sends a command and does not check the result. A stepper motor told to take 200 steps assumes it took 200 steps, even if it stalled.
- **Closed loop:** the controller measures the result and corrects the command. A DC motor with an encoder compares its actual speed with the target speed and adjusts.

Example: you want a wheel to turn exactly 90 degrees.

- Open loop: send a fixed pulse count and hope.
- Closed loop: read the encoder, stop when the reading reaches 90 degrees, and correct for overshoot.

Closed loop costs more hardware and computation. It is almost always worth it when precision or safety matters.

---

## 5. The major types of robots

| Type | Typical sensing | Typical actuation | Where you see it |
|---|---|---|---|
| Mobile robot | Wheel encoders, LiDAR or sonar, IMU, camera | Wheel motors | Warehouse carts, vacuum cleaners, delivery robots |
| Manipulator (robot arm) | Joint encoders, force sensing, camera | Joint motors, gripper | Factory assembly, surgery, pick-and-place |
| Drone (UAV) | IMU, GPS, barometer, camera | Propeller motors | Inspection, mapping, delivery pilots |
| Humanoid | IMU, joint encoders, cameras, force sensors | Many joint motors | Research platforms, early commercial pilots |
| Industrial robot | Joint encoders, safety sensors | Large servo arms | Car manufacturing, welding |
| Service robot | Cameras, microphones, touch sensors | Wheels, arms, speakers | Hotels, hospitals, cleaning |

Pick one robot from this table and ask yourself which of the four loop jobs is the hardest. The answer usually tells you which discipline is most important for that robot.

**Reference images to find:** search "robot types diagram" and compare a manipulator with a mobile robot. Notice how many more joints the arm has, and how the mobile robot needs navigation that the arm does not.

**Video (about 15 minutes):** [Boston Dynamics: a short overview of their robots](https://www.youtube.com/@BostonDynamics) is a good way to see a range of robot types. Watch it for the variety, then pause and identify the sensors and actuators you can see.

---

## 6. Hands-on exercise: decompose five robots (about 60 minutes)

Pick five robots you have seen in videos, products or real life. For each one, fill in this table in your notes.

| Robot | Sensors | Compute / controller | Algorithms (what decides) | Actuators | Mechanics | Power | Feedback loop |
|---|---|---|---|---|---|---|---|
| | | | | | | | |

Rules:

- Write "not visible" when you cannot tell. Do not invent sensors.
- In the algorithms column, write what the robot decides, such as "follows a line," "avoids obstacles," or "balances." Do not write "AI" unless you can say what it learns.
- Check your entries against any product page or paper you can find.

---

## 7. Project: a robot architecture on paper

Sketch the architecture of a robot you could plausibly build in two years. Keep it small: a line-following car, a desk-sized robot arm, a self-balancing wheeled robot, or a plant-watering rover.

Your sketch must include:

1. **One sentence of purpose.** What does the robot do, and for whom?
2. **Inputs.** The sensors and what each one measures.
3. **Compute.** The computing platform and why you chose it.
4. **Outputs.** The actuators and power source.
5. **The loop.** Draw the sense, decide, act arrows and label the feedback.
6. **Open or closed loop.** For each actuator, say which, and why.

**Repository setup (GitHub):**

```
robotics-day-01-architecture/
├── README.md          # purpose, architecture diagram, loop explanation
├── architecture.png   # your block diagram (photo of paper or drawn in any tool)
└── notes/
    └── robot-types.md # your five-robot decomposition table
```

**Done when:**

- The README states the purpose in one sentence.
- Every sensor and actuator in the diagram has a reason to be there.
- You can explain the loop out loud in under a minute, without notes.

---

## 8. Check your understanding

1. What are the four jobs every robot performs?
2. Give one example of automation that is not robotics, and say why.
3. Why does feedback change an open-loop system into a closed-loop one?
4. A robot vacuum uses wheel encoders. Is that open loop or closed loop for driving distance? What could go wrong?
5. Why would a manipulator need more joint encoders than a mobile robot needs wheel encoders?
6. Which sensor would you add to a drone to know its tilt, and why not just use a camera?

**Answers (check after you try):**

1. Sense, decide (compute), act, and get feedback from the result.
2. A timer that runs a pump on a schedule. It senses nothing and adapts to nothing.
3. Feedback lets the controller compare the result with the target and correct the command.
4. Wheel encoders give a closed loop for wheel rotation. Wheels slip on carpet and drift, so distance estimates accumulate error.
5. Each joint must be known to place the end of the arm. The arm has several joints in sequence, and errors compound along the chain.
6. An inertial measurement unit (IMU) measures acceleration and rotation rate directly. A camera can estimate tilt, but it is slower and needs good light and a visible horizon.

---

## 9. Sources to read and watch

**Read:**
- [Wikipedia: Robotics](https://en.wikipedia.org/wiki/Robotics)
- [Wikipedia: Mechatronics](https://en.wikipedia.org/wiki/Mechatronics)
- [Wikipedia: Control theory](https://en.wikipedia.org/wiki/Control_theory) (read the introduction only today)
- [Wikipedia: Closed-loop controller](https://en.wikipedia.org/wiki/Closed-loop_controller)
- [GeeksforGeeks: Robotics topics and introduction articles](https://www.geeksforgeeks.org/): search "robotics basics" and read one introduction
- [W3Schools: no direct robotics course; use it only for Arduino or Python basics later in the series](https://www.w3schools.com/)
- [Modern Robotics course resources](https://modernrobotics.northwestern.edu/) (overview only)

**Watch:**
- [MIT OpenCourseWare: Introduction to robotics lectures](https://ocw.mit.edu/) (search "robotics" and watch the first lecture)
- [The Construct](https://www.youtube.com/@TheConstruct): watch one introductory video on robot software.
- [Articulated Robotics](https://www.youtube.com/@ArticulatedRobotics): watch one introduction to a mobile robot build.

**Practice:**
- [ROS 2 documentation: "Introduction" page](https://docs.ros.org/en/rolling/Tutorials.html) (read, do not install yet)

**Note on links:** open each one once. If a page has moved, search its title.

---

## 10. Tomorrow

Day 2, **Anatomy of a Robot**, opens the box. You will map each block from today's loop to real hardware: microcontrollers, single-board computers, motor drivers, batteries and frames. Bring your architecture sketch from today, because tomorrow you will compare it with real parts.
