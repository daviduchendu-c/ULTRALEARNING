# 100 Days of Robotics Engineering

# Module 1: Robotics Engineering Foundations (Days 1-5)

## Day 1: What Is Robotics?

**Tutorial:** [Day 1: What Is Robotics?](100-Day%20Tracker/Day-01-What-Is-Robotics.md)

Study:

-   Robotics vs automation vs mechatronics
-   Sensing, computation, decision and actuation
-   Open and closed loop
-   Autonomous and teleoperated robots
-   Mobile robots, manipulators, drones, humanoids, industrial and service robots

**Exercise:** Decompose five robots into sensors, controller/compute,
algorithms, actuators, mechanics, power and feedback.

**Output:** General robot architecture block diagram.

## Day 2: Anatomy of a Robot

**Tutorial:** [Day 2: Anatomy of a Robot](100-Day%20Tracker/Day-02-Anatomy-of-a-Robot.md)

Study MCUs, SBCs, CPUs/GPUs, sensors, actuators, motor drivers,
batteries, regulators, buses, frames, joints, transmissions, firmware
and high-level software.

**Output:** Hardware/software architecture for a mobile robot.

## Day 3: Robotics Engineering Workflow

**Tutorial:** [Day 3: Robotics Engineering Workflow](100-Day%20Tracker/Day-03-Robotics-Engineering-Workflow.md)

Problem → requirements → specifications → architecture → mechanical
design → electronics → firmware → software → simulation → prototype →
integration → testing → iteration → documentation.

Learn engineering notebooks, block diagrams, wiring diagrams,
schematics, BOMs and version control.

## Day 4: Measurement and Engineering Tools

**Tutorial:** [Day 4: Measurement and Engineering Tools](100-Day%20Tracker/Day-04-Measurement-and-Engineering-Tools.md)

Study multimeters, bench supplies, oscilloscopes, logic analyzers,
calipers, breadboards, soldering tools, datasheets and systematic
debugging.

## Day 5: First Robot System Design

**Tutorial:** [Day 5: First Robot System Design](100-Day%20Tracker/Day-05-First-Robot-System-Design.md)

Design a robot on paper: requirements, compute, power, sensors,
actuators, communications, mechanics, software architecture and BOM.

------------------------------------------------------------------------

# Module 2: Electronics for Robotics (Days 6-15)

## Day 6: Electricity Fundamentals

Charge, voltage, current, resistance, power, energy, DC/AC concepts,
conductors, ground/reference.

Core relationships: `V = IR`, `P = VI`.

## Day 7: Circuit Analysis

Series/parallel circuits, equivalent resistance, voltage/current
division, Kirchhoff's Voltage and Current Laws.

## Day 8: Components

Resistors, potentiometers, capacitors, inductors, diodes, LEDs,
switches, relays, connectors, fuses and protection components.

## Day 9: Semiconductor Fundamentals

PN junctions, diodes, BJTs, MOSFETs, transistor switching,
pull-ups/pull-downs and logic-level switching.

## Day 10: Digital Electronics

Binary, bits/bytes, number systems, logic levels, logic gates, Boolean
algebra, truth tables, combinational/sequential logic.

## Day 11: Analog Electronics and Signals

Analog/digital signals, dividers, ADC/DAC concepts, resolution,
sampling, noise, filters and signal conditioning.

## Day 12: Robot Power Systems

Batteries, capacity, current requirements, regulators, buck/boost
conversion, rails, grounding, decoupling, power budgets, protection and
brownouts.

## Day 13: Motor Electronics

DC motor electrical behavior, back-EMF concept, H-bridges, drivers,
MOSFET switching, PWM, current limits and electrical noise.

## Day 14: Schematics and Datasheets

Symbols, nets, rails, IC pins, connectors, reference designators,
ratings, operating conditions and timing diagrams.

## Day 15: Electronics Project

Create and document a small **sensor → controller → actuator** system.

------------------------------------------------------------------------

# Module 3: Embedded Systems (Days 16-25)

## Day 16: Microcontrollers

CPU, clock, flash, SRAM, registers, GPIO, peripherals, firmware, boot
process and MCU vs SBC.

## Day 17: GPIO

Digital I/O, logic levels, pull-ups/down, buttons, LEDs and debouncing.

## Day 18: ADC, PWM and Timers

ADC resolution/sampling, PWM duty/frequency, timers and timing-sensitive
control.

## Day 19: Interrupts

Polling, interrupts, ISRs, hardware events, timer interrupts and
blocking vs non-blocking code.

## Day 20: UART

TX/RX, baud rate, framing concepts, serial debugging and device
communication.

## Day 21: I²C

SDA/SCL, addressing, bus topology, multiple devices and debugging.

## Day 22: SPI

MOSI/MISO/SCLK/CS, bus speed, multiple devices and SPI vs I²C vs UART.

## Day 23: Embedded Software Architecture

State machines, drivers, modules, hardware abstraction, non-blocking
architecture, telemetry and fault handling.

## Day 24: RTOS Fundamentals

Tasks, scheduling, queues, semaphores, mutexes, timing, concurrency and
race-condition concepts.

## Day 25: Embedded Project

Integrate multiple sensors/actuators, communications, a state machine,
telemetry and structured firmware.

------------------------------------------------------------------------

# Module 4: Sensors and Actuators (Days 26-33)

## Day 26: DC Motors

Construction, torque, speed, current, stall current, gearing, encoders
and drivers.

## Day 27: Servo Motors

Position control, feedback concepts, command interface, torque, range
and calibration.

## Day 28: Stepper Motors

Steps, step angle, drivers, microstepping concepts, torque/speed and
positioning.

## Day 29: Encoders

Incremental/absolute concepts, quadrature, ticks, direction, RPM,
position and odometry foundations.

## Day 30: Distance Sensors

Ultrasonic, IR, ToF and LiDAR principles, range, FOV, resolution, noise
and limitations.

## Day 31: IMUs

Accelerometers, gyroscopes, magnetometers, axes, drift, bias,
calibration and orientation.

## Day 32: Cameras

RGB, stereo/depth concepts, resolution, FPS, exposure, FOV, calibration
and camera frames.

## Day 33: Sensor Integration Project

Collect, timestamp, log and visualize multiple sensor streams.

------------------------------------------------------------------------

# Module 5: Mechanical Engineering and CAD (Days 34-42)

## Day 34: Mechanical Fundamentals

Force, torque, mass, center of mass, friction, stress, strain, stiffness
and equilibrium concepts.

## Day 35: Mechanisms

Gears, ratios, pulleys, belts, chains, lead screws, linkages, bearings,
shafts and couplings.

## Day 36: Robot Drivetrains

Differential, skid, Ackermann, omni, mecanum and tracked systems;
compare maneuverability, complexity, odometry and terrain.

## Day 37: CAD Fundamentals

Sketches, constraints, dimensions, reference geometry, parametric
modeling and design intent.

## Day 38: 3D Part Modeling

Extrusions, revolutions, patterns, holes, fillets, chamfers and
parametric dimensions.

## Day 39: Assemblies

Parts, joints/mates, DOF, interference, clearances and assembly
hierarchy.

## Day 40: Robot Component Design

Motor brackets, sensor mounts, wheel hubs, chassis parts, servo
mechanisms and enclosures.

## Day 41: Design for Manufacturing

3D printing, machining/laser-cutting concepts, fasteners, tolerances,
materials and assembly.

## Day 42: CAD Project

Design a complete small robot chassis with assembly, drawings, exploded
view, BOM and manufacturing plan.

------------------------------------------------------------------------

# Module 6: PCB Design (Days 43-47)

## Day 43: PCB Fundamentals

Layers, traces, pads, vias, footprints, solder mask, silkscreen, ground
planes and power distribution.

## Day 44: Schematic Capture

Symbols, labels, power, connectors, decoupling, ERC and component
selection.

## Day 45: PCB Layout

Placement, routing, trace width, ground/power planes, clearances, DRC
and mechanical constraints.

## Day 46: Manufacturing and Bring-Up

Gerbers, drill files, BOM, fabrication/assembly concepts, inspection,
continuity tests and board bring-up.

## Day 47: Robotics Controller PCB

Design a controller board with MCU, sensor/communication connectors,
actuator interface, power, protection and debug/programming.

------------------------------------------------------------------------

# Module 7: Mathematics for Robotics (Days 48-55)

## Day 48: Algebra and Trigonometry

Functions, equations, radians, sine/cosine/tangent, identities and
vectors.

## Day 49: Linear Algebra I

Scalars, vectors, matrices, multiplication, transpose, inverse concepts
and systems of equations.

## Day 50: Linear Algebra II

Dot/cross products, transformations, basis, determinants and
eigenvalue/eigenvector concepts.

## Day 51: Coordinate Systems

World, base, sensor and end-effector frames; position vectors and frame
changes.

## Day 52: Rotations

2D/3D rotations, rotation matrices, Euler angles, roll/pitch/yaw and
quaternion concepts.

## Day 53: Homogeneous Transformations

Translation + rotation, transform composition, inverses and frame
chains. Implement in Python.

## Day 54: Calculus for Robotics

Derivatives, rates, position/velocity/acceleration, integration and
differential-equation concepts.

## Day 55: Probability

Random variables, mean, variance, Gaussian distributions, conditional
probability, Bayes and sensor uncertainty.

------------------------------------------------------------------------

# Module 8: Robot Kinematics (Days 56-62)

## Day 56: DOF and Configuration Space

Joints, degrees of freedom, configuration, C-space, workspace and
constraints.

## Day 57: Forward Kinematics

Joint variables → end-effector pose. Start with planar arms and progress
to matrix formulations.

## Day 58: Denavit-Hartenberg Parameters

Link frames, DH parameters and transformation chains.

## Day 59: Inverse Kinematics

Desired end-effector pose → joint configuration; geometric,
analytical/numerical concepts, multiple solutions and constraints.

## Day 60: Jacobians

Differential motion, joint velocity, end-effector velocity and Jacobian
mapping.

## Day 61: Singularities

Singular configurations, rank concepts, lost DOF and manipulability
concepts.

## Day 62: Kinematics Project

Create a simulated arm with FK, IK, frame visualization and end-effector
trajectories.

------------------------------------------------------------------------

# Module 9: Dynamics and Control (Days 63-71)

## Day 63: Robot Dynamics

Mass, inertia, forces, torque, gravity, friction and equations of
motion.

## Day 64: Dynamic System Modeling

Differential equations, inputs/outputs, state variables and state-space
concepts.

## Day 65: Feedback Control

Plant, controller, setpoint, sensor, feedback, error, disturbance and
stability concepts.

## Day 66: PID

P/I/D terms, rise time, overshoot, settling, steady-state error, tuning,
saturation and windup concepts.

## Day 67: Motor Control

Position/velocity control, encoder feedback, setpoints and closed-loop
control.

## Day 68: State-Space Control

State vectors/equations, controllability, observability and
state-feedback concepts.

## Day 69: Trajectory Generation

Position, velocity and acceleration profiles, smooth motion and
polynomial trajectory concepts.

## Day 70: Motion Control

Path vs trajectory, trajectory tracking and feedback tracking.

## Day 71: Control Project

Create a closed-loop simulation or suitably safe physical experiment and
analyze its response.

------------------------------------------------------------------------

# Module 10: ROS 2 (Days 72-81)

## Day 72: ROS 2 Foundations

Middleware, distributed robotic systems, DDS concepts, ROS graph,
workspaces, packages and distributions.

## Day 73: ROS 2 CLI and Turtlesim

Environment, CLI, run, node/topic/service/action inspection, `rqt` and
graph exploration.

## Day 74: Nodes and Topics

Publishers, subscribers, messages, topic types, QoS introduction and
communication graphs.

## Day 75: Services and Actions

Clients/servers, request-response, goals, feedback/results and when to
choose each communication model.

## Day 76: Parameters, Launch and Logging

Parameters, YAML, launch files, logging, namespaces and remapping.

## Day 77: ROS 2 Packages

Create Python and C++ packages; understand dependencies, build systems,
`colcon` and workspace sourcing.

## Day 78: TF2

Transform trees, static/dynamic transforms, parent-child frames and
transform lookup.

## Day 79: URDF and Xacro

Links, joints, visual/collision/inertial data, robot trees and macros.

## Day 80: ros2_control

Hardware interfaces, controllers, joint states/commands and controller
management.

## Day 81: ROS 2 Project

Build a multi-node system with robot model, TF, communication,
configuration, launch and controller interfaces.

------------------------------------------------------------------------

# Module 11: Robot Simulation (Days 82-86)

> Use **modern Gazebo**, not Gazebo Classic, for a new curriculum.

## Day 82: Gazebo Fundamentals

Worlds, models, physics, rendering, SDF concepts, simulation time and
spawning.

## Day 83: Simulated Sensors

Cameras, IMUs, LiDAR/range sensors, update rates and sensor noise.

## Day 84: ROS 2 + Gazebo

Integrate robot descriptions, ROS topics, sensors, controllers and TF
with simulation.

## Day 85: Simulated Mobile Robot

Create a differential-drive robot with sensors, controller and ROS
interfaces.

## Day 86: Simulated Manipulator

Create an arm with links, joints, controllers, sensors and ROS
integration.

------------------------------------------------------------------------

# Module 12: Autonomous Robotics (Days 87-94)

## Day 87: Odometry

Encoder measurements, differential-drive kinematics, pose estimation and
accumulated error.

## Day 88: Sensor Fusion

Noise, bias, drift, complementary filters, Kalman-filter intuition and
multi-sensor estimates.

## Day 89: Localization

Pose, uncertainty, probabilistic localization, particle-filter intuition
and map/odom frames.

## Day 90: Mapping and SLAM

Occupancy grids, SLAM, loop-closure concepts, LiDAR SLAM and visual-SLAM
concepts.

## Day 91: Path Planning

Configuration spaces, graph search, Dijkstra, A\*, heuristics, collision
checking and sampling-based concepts.

## Day 92: Navigation

Global/local planning, costmaps, obstacle avoidance, goal navigation and
Nav2 concepts.

## Day 93: Manipulation and Motion Planning

Grasping, collision checking, joint constraints, trajectory execution
and MoveIt 2 concepts.

## Day 94: Autonomous Robot Project

Build a simulated system that can:

**Sense → Estimate → Localize → Plan → Move → Avoid obstacles → Reach a
goal**

------------------------------------------------------------------------

# Module 13: Modern Robotics (Days 95-98)

## Day 95: NVIDIA Isaac Sim

USD concepts, assets, physics, articulations, sensors, URDF import,
OmniGraph, Python and ROS 2 bridge.

## Day 96: Robotics Perception

Camera calibration, image processing, detection/segmentation concepts,
depth, point clouds and pose estimation.

## Day 97: Robot Learning

ML for robotics, reinforcement learning, imitation learning, policies,
observations/actions/rewards, domain randomization and sim-to-real
concepts.

## Day 98: Modern Robot Architectures

Study humanoids, quadrupeds, autonomous vehicles, cobots, mobile
manipulators, warehouse robots and physical AI as integrated systems.

------------------------------------------------------------------------

# Module 14: Complete Capstone (Days 99-100)

## Day 99: Engineer the Robot

Produce:

1.  Problem statement
2.  User requirements
3.  Engineering specifications
4.  System architecture
5.  Mechanical architecture
6.  CAD
7.  Electronics architecture
8.  PCB/schematic where appropriate
9.  Power budget
10.  Sensors
11.  Actuators
12.  Embedded architecture
13.  ROS 2 architecture
14.  Control architecture
15.  Simulation model
16.  Autonomy architecture
17.  BOM
18.  Test plan
19.  Risk/failure analysis
20.  Build schedule

## Day 100: Integrate, Test and Teach

Present the complete system: requirements, mechanics, electronics,
power, firmware, sensors, actuators, mathematics, kinematics, control,
ROS graph, simulation, autonomy, software architecture, testing,
failures, tradeoffs and improvements.

**Suggested episode:**\
`Day 100/100: I Built an Autonomous Robot From Scratch | 100 Days of Robotics Engineering`

------------------------------------------------------------------------

# Primary Resources

## ROS 2

-   https://docs.ros.org/
-   https://docs.ros.org/en/rolling/Tutorials.html
-   https://docs.ros.org/en/rolling/Tutorials/Advanced/Simulators/Gazebo/Gazebo.html

## Gazebo

-   https://gazebosim.org/docs/

## NVIDIA Isaac Sim

-   https://docs.isaacsim.omniverse.nvidia.com/latest/
-   https://docs.nvidia.com/learning/physical-ai/getting-started-with-isaac-sim/latest/index.html
-   https://docs.isaacsim.omniverse.nvidia.com/latest/ros2_tutorials/index.html

## CAD

-   https://www.freecad.org/
-   https://www.solidworks.com/

## PCB

-   https://www.kicad.org/
-   https://docs.kicad.org/

------------------------------------------------------------------------

# Recommended Documentation and Video Learning

## Primary documentation

-   ROS 2: https://docs.ros.org/
-   ROS 2 Tutorials: https://docs.ros.org/en/rolling/Tutorials.html
-   Gazebo: https://gazebosim.org/docs/
-   ros2_control: https://control.ros.org/
-   MoveIt 2: https://moveit.picknik.ai/
-   Nav2: https://docs.nav2.org/
-   FreeCAD: https://www.freecad.org/
-   KiCad: https://docs.kicad.org/
-   NVIDIA Isaac Sim: https://docs.isaacsim.omniverse.nvidia.com/latest/

## University / theory resources

-   MIT OpenCourseWare: https://ocw.mit.edu/
-   MIT Underactuated Robotics: https://underactuated.mit.edu/
-   Modern Robotics course/book resources:
    https://modernrobotics.northwestern.edu/

## YouTube / video sources

-   **MIT OpenCourseWare:** mechanics, control, algorithms and
    robotics: https://www.youtube.com/@mitocw
-   **Modern Robotics:** use the Northwestern Modern Robotics
    video/course sequence alongside the textbook/site.
-   **The Construct:** ROS/ROS 2 demonstrations and robotics software
    tutorials: https://www.youtube.com/@TheConstruct
-   **Articulated Robotics:** practical ROS 2 and mobile-robot build
    explanations: https://www.youtube.com/@ArticulatedRobotics
-   **NVIDIA Developer:** Isaac Sim / robotics / accelerated
    computing talks: https://www.youtube.com/@NVIDIADeveloper

### Suggested viewing by phase

-   Electronics/embedded: watch concept demonstrations only after
    learning circuit and datasheet fundamentals.
-   Mechanics/CAD: follow modeling demonstrations, then recreate a
    different mechanism without copying.
-   Math/kinematics/control: prioritize university lectures and solve
    equations yourself.
-   ROS 2/Gazebo: official docs first, then implementation videos.
-   Autonomy: pair Nav2/MoveIt documentation with simulation
    experiments.
-   Isaac/robot learning: use current NVIDIA documentation because APIs
    and workflows change.

# Day 101: Career, Hiring and Monetization

## 101.1 Choose a Marketable Direction

-   Robotics software engineer
-   Embedded/firmware engineer
-   Robotics integration engineer
-   Mechatronics/robotics technician or junior engineer
-   Controls/automation-oriented engineer
-   ROS 2 developer
-   Robotics simulation engineer
-   Computer-vision/perception-oriented robotics engineer after the AI
    track

## 101.2 What "Job Ready" Means

Show evidence that you can:

-   integrate sensors, actuators and compute
-   read schematics and datasheets
-   write structured embedded/robotics software
-   model basic mechanics and robot geometry
-   use measurement/debugging tools safely
-   build ROS 2 packages and inspect a ROS graph
-   model and simulate a robot
-   explain kinematics/control fundamentals
-   test hardware/software interfaces
-   document wiring, CAD, code, BOMs and test results

A stronger proof package is:

1.  **2-4 serious projects** relevant to the role.
2.  Clean source repositories with useful READMEs.
3.  A short technical case study for each major project.
4.  A live demo, video demo, simulation, or deployed application where
    appropriate.
5.  Tests, measurements, evaluation results, or engineering evidence.
6.  Clear explanation of tradeoffs and failures.
7.  Consistent Git history showing that you actually built the work.
8.  The ability to explain your own code/design without relying on
    notes.

## 101.3 Build a Proof-of-Work Portfolio

Every flagship project should answer:

-   What problem did I solve?
-   Who is it for?
-   What constraints did I have?
-   Why did I choose this architecture?
-   What did I personally implement?
-   What failed?
-   How did I debug it?
-   How did I test/evaluate it?
-   What are the measurable results?
-   What would I change in version 2?

Publish a portfolio page containing a concise bio, skills, projects,
technical write-ups, Git repositories, demos and a contact method.

## 101.4 Getting Hired

### Resume, applications, networking, interviews

Prepare for:

-   explaining projects deeply
-   debugging
-   fundamentals
-   practical coding/design tasks
-   architecture/tradeoff discussions
-   behavioral questions about teamwork and failure
-   C++/Python and Linux fundamentals
-   electronics/embedded debugging
-   coordinate frames, kinematics and control basics
-   ROS 2 concepts
-   discussion of a robot you personally integrated

## 101.5 Ways to Make Money

A technical skill can produce income in several ways:

-   Employment and internships
-   Contract projects
-   Productized services
-   Products
-   Education and technical content
-   Consulting, later

## 101.6 The First-Income Ladder

**Stage 1, Proof:** Complete small projects and publish them.

**Stage 2, Reliability:** Complete larger projects with
documentation, tests and measurable results.

**Stage 3, Small paid problem:** Solve a bounded problem for a real
person, school, community, organization or business where legally and
practically appropriate.

**Stage 4, Repeatability:** Turn repeated work into a reusable
service/product.

**Stage 5, Specialization:** Become known for solving a narrower,
valuable problem extremely well.

**Stage 6, Scale:** Employment growth, higher-value contracts,
products, content, or eventually a company.

## 101.7 Your Career Capstone

Create a **Career Launch Pack**:

-   one-page role target
-   skills-gap checklist
-   resume
-   portfolio website
-   3 flagship case studies
-   GitHub/repository cleanup
-   90-second introduction
-   project demo reel
-   20 practice interview questions
-   list of target organizations/roles
-   application tracker
-   90-day continued-learning plan
-   one realistic monetization experiment
