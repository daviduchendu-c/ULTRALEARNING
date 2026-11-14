# Robotics Engineering 100

# Module 1: Robotics Engineering Foundations (Episodes 1-5)

## Episode 1: What Is Robotics?

**Tutorial:** [Episode 1: What Is Robotics?](100%20Episodes%20Robotics/Module-01-Robotics-Engineering-Foundations/Robotics-Episode-01-What-Is-Robotics.md)

**Study:**

-   Robotics vs automation vs mechatronics
-   Sensing, computation, decision and actuation
-   Open and closed loop
-   Autonomous and teleoperated robots
-   Mobile robots, manipulators, drones, humanoids, industrial and service robots

**Exercise:** Decompose five robots into sensors, controller/compute,
algorithms, actuators, mechanics, power and feedback.

**Output:** General robot architecture block diagram.

## Episode 2: Anatomy of a Robot

**Tutorial:** [Episode 2: Anatomy of a Robot](100%20Episodes%20Robotics/Module-01-Robotics-Engineering-Foundations/Robotics-Episode-02-Anatomy-of-a-Robot.md)

**Study:**

-   MCUs
-   SBCs
-   CPUs/GPUs
-   Sensors
-   Actuators
-   Motor drivers
-   Batteries
-   Regulators
-   Buses
-   Frames
-   Joints
-   Transmissions
-   Firmware
-   High-level software

**Output:** Hardware/software architecture for a mobile robot.

## Episode 3: Robotics Engineering Workflow

**Tutorial:** [Episode 3: Robotics Engineering Workflow](100%20Episodes%20Robotics/Module-01-Robotics-Engineering-Foundations/Robotics-Episode-03-Robotics-Engineering-Workflow.md)

**Workflow:**

1.  Problem
2.  Requirements
3.  Specifications
4.  Architecture
5.  Mechanical design
6.  Electronics
7.  Firmware
8.  Software
9.  Simulation
10. Prototype
11. Integration
12. Testing
13. Iteration
14. Documentation

**Study:**

-   Engineering notebooks
-   Block diagrams
-   Wiring diagrams
-   Schematics
-   BOMs
-   Version control

## Episode 4: Measurement and Engineering Tools

**Tutorial:** [Episode 4: Measurement and Engineering Tools](100%20Episodes%20Robotics/Module-01-Robotics-Engineering-Foundations/Robotics-Episode-04-Measurement-and-Engineering-Tools.md)

**Study:**

-   Multimeters
-   Bench supplies
-   Oscilloscopes
-   Logic analyzers
-   Calipers
-   Breadboards
-   Soldering tools
-   Datasheets
-   Systematic debugging

## Episode 5: First Robot System Design

**Tutorial:** [Episode 5: First Robot System Design](100%20Episodes%20Robotics/Module-01-Robotics-Engineering-Foundations/Robotics-Episode-05-First-Robot-System-Design.md)

**Design a robot on paper:**

-   Requirements
-   Compute
-   Power
-   Sensors
-   Actuators
-   Communications
-   Mechanics
-   Software architecture
-   BOM

------------------------------------------------------------------------

# Module 2: Electronics for Robotics (Episodes 6-15)

## Episode 6: Electricity Fundamentals

**Study:**

-   Charge
-   Voltage
-   Current
-   Resistance
-   Power
-   Energy
-   DC/AC concepts
-   Conductors
-   Ground/reference

**Core relationships:** `V = IR`, `P = VI`.

## Episode 7: Circuit Analysis

**Study:**

-   Series/parallel circuits
-   Equivalent resistance
-   Voltage/current division
-   Kirchhoff's Voltage and Current Laws

## Episode 8: Components

**Study:**

-   Resistors
-   Potentiometers
-   Capacitors
-   Inductors
-   Diodes
-   LEDs
-   Switches
-   Relays
-   Connectors
-   Fuses and protection components

## Episode 9: Semiconductor Fundamentals

**Study:**

-   PN junctions
-   Diodes
-   BJTs
-   MOSFETs
-   Transistor switching
-   Pull-ups/pull-downs
-   Logic-level switching

## Episode 10: Digital Electronics

**Study:**

-   Binary
-   Bits/bytes
-   Number systems
-   Logic levels
-   Logic gates
-   Boolean algebra
-   Truth tables
-   Combinational/sequential logic

## Episode 11: Analog Electronics and Signals

**Study:**

-   Analog/digital signals
-   Dividers
-   ADC/DAC concepts
-   Resolution
-   Sampling
-   Noise
-   Filters
-   Signal conditioning

## Episode 12: Robot Power Systems

**Study:**

-   Batteries
-   Capacity
-   Current requirements
-   Regulators
-   Buck/boost conversion
-   Rails
-   Grounding
-   Decoupling
-   Power budgets
-   Protection
-   Brownouts

## Episode 13: Motor Electronics

**Study:**

-   DC motor electrical behavior
-   Back-EMF concept
-   H-bridges
-   Drivers
-   MOSFET switching
-   PWM
-   Current limits
-   Electrical noise

## Episode 14: Schematics and Datasheets

**Study:**

-   Symbols
-   Nets
-   Rails
-   IC pins
-   Connectors
-   Reference designators
-   Ratings
-   Operating conditions
-   Timing diagrams

## Episode 15: Electronics Project

**Build:**

-   A small **sensor → controller → actuator** system
-   Documentation for the system

------------------------------------------------------------------------

# Module 3: Embedded Systems (Episodes 16-25)

## Episode 16: Microcontrollers

**Study:**

-   CPU
-   Clock
-   Flash
-   SRAM
-   Registers
-   GPIO
-   Peripherals
-   Firmware
-   Boot process
-   MCU vs SBC

## Episode 17: GPIO

**Study:**

-   Digital I/O
-   Logic levels
-   Pull-ups/pull-downs
-   Buttons
-   LEDs
-   Debouncing

## Episode 18: ADC, PWM and Timers

**Study:**

-   ADC resolution/sampling
-   PWM duty/frequency
-   Timers
-   Timing-sensitive control

## Episode 19: Interrupts

**Study:**

-   Polling
-   Interrupts
-   ISRs
-   Hardware events
-   Timer interrupts
-   Blocking vs non-blocking code

## Episode 20: UART

**Study:**

-   TX/RX
-   Baud rate
-   Framing concepts
-   Serial debugging
-   Device communication

## Episode 21: I²C

**Study:**

-   SDA/SCL
-   Addressing
-   Bus topology
-   Multiple devices
-   Debugging

## Episode 22: SPI

**Study:**

-   MOSI/MISO/SCLK/CS
-   Bus speed
-   Multiple devices
-   SPI vs I²C vs UART

## Episode 23: Embedded Software Architecture

**Study:**

-   State machines
-   Drivers
-   Modules
-   Hardware abstraction
-   Non-blocking architecture
-   Telemetry
-   Fault handling

## Episode 24: RTOS Fundamentals

**Study:**

-   Tasks
-   Scheduling
-   Queues
-   Semaphores
-   Mutexes
-   Timing
-   Concurrency
-   Race-condition concepts

## Episode 25: Embedded Project

**Build:**

-   Multiple integrated sensors/actuators
-   Communications
-   A state machine
-   Telemetry
-   Structured firmware

------------------------------------------------------------------------

# Module 4: Sensors and Actuators (Episodes 26-33)

## Episode 26: DC Motors

**Study:**

-   Construction
-   Torque
-   Speed
-   Current
-   Stall current
-   Gearing
-   Encoders
-   Drivers

## Episode 27: Servo Motors

**Study:**

-   Position control
-   Feedback concepts
-   Command interface
-   Torque
-   Range
-   Calibration

## Episode 28: Stepper Motors

**Study:**

-   Steps
-   Step angle
-   Drivers
-   Microstepping concepts
-   Torque/speed
-   Positioning

## Episode 29: Encoders

**Study:**

-   Incremental/absolute concepts
-   Quadrature
-   Ticks
-   Direction
-   RPM
-   Position
-   Odometry foundations

## Episode 30: Distance Sensors

**Study:**

-   Ultrasonic, IR, ToF and LiDAR principles
-   Range
-   FOV
-   Resolution
-   Noise
-   Limitations

## Episode 31: IMUs

**Study:**

-   Accelerometers
-   Gyroscopes
-   Magnetometers
-   Axes
-   Drift
-   Bias
-   Calibration
-   Orientation

## Episode 32: Cameras

**Study:**

-   RGB
-   Stereo/depth concepts
-   Resolution
-   FPS
-   Exposure
-   FOV
-   Calibration
-   Camera frames

## Episode 33: Sensor Integration Project

**Build:**

-   Collect multiple sensor streams
-   Timestamp them
-   Log them
-   Visualize them

------------------------------------------------------------------------

# Module 5: Mechanical Engineering and CAD (Episodes 34-42)

## Episode 34: Mechanical Fundamentals

**Study:**

-   Force
-   Torque
-   Mass
-   Center of mass
-   Friction
-   Stress
-   Strain
-   Stiffness
-   Equilibrium concepts

## Episode 35: Mechanisms

**Study:**

-   Gears
-   Ratios
-   Pulleys
-   Belts
-   Chains
-   Lead screws
-   Linkages
-   Bearings
-   Shafts
-   Couplings

## Episode 36: Robot Drivetrains

**Study:**

-   Differential
-   Skid
-   Ackermann
-   Omni
-   Mecanum
-   Tracked

**Compare:** maneuverability, complexity, odometry and terrain.

## Episode 37: CAD Fundamentals

**Study:**

-   Sketches
-   Constraints
-   Dimensions
-   Reference geometry
-   Parametric modeling
-   Design intent

## Episode 38: 3D Part Modeling

**Study:**

-   Extrusions
-   Revolutions
-   Patterns
-   Holes
-   Fillets
-   Chamfers
-   Parametric dimensions

## Episode 39: Assemblies

**Study:**

-   Parts
-   Joints/mates
-   DOF
-   Interference
-   Clearances
-   Assembly hierarchy

## Episode 40: Robot Component Design

**Design:**

-   Motor brackets
-   Sensor mounts
-   Wheel hubs
-   Chassis parts
-   Servo mechanisms
-   Enclosures

## Episode 41: Design for Manufacturing

**Study:**

-   3D printing
-   Machining/laser-cutting concepts
-   Fasteners
-   Tolerances
-   Materials
-   Assembly

## Episode 42: CAD Project

**Design a complete small robot chassis with:**

-   Assembly
-   Drawings
-   Exploded view
-   BOM
-   Manufacturing plan

------------------------------------------------------------------------

# Module 6: PCB Design (Episodes 43-47)

## Episode 43: PCB Fundamentals

**Study:**

-   Layers
-   Traces
-   Pads
-   Vias
-   Footprints
-   Solder mask
-   Silkscreen
-   Ground planes
-   Power distribution

## Episode 44: Schematic Capture

**Study:**

-   Symbols
-   Labels
-   Power
-   Connectors
-   Decoupling
-   ERC
-   Component selection

## Episode 45: PCB Layout

**Study:**

-   Placement
-   Routing
-   Trace width
-   Ground/power planes
-   Clearances
-   DRC
-   Mechanical constraints

## Episode 46: Manufacturing and Bring-Up

**Study:**

-   Gerbers
-   Drill files
-   BOM
-   Fabrication/assembly concepts
-   Inspection
-   Continuity tests
-   Board bring-up

## Episode 47: Robotics Controller PCB

**Design a controller board with:**

-   MCU
-   Sensor/communication connectors
-   Actuator interface
-   Power
-   Protection
-   Debug/programming

------------------------------------------------------------------------

# Module 7: Mathematics for Robotics (Episodes 48-55)

## Episode 48: Algebra and Trigonometry

**Study:**

-   Functions
-   Equations
-   Radians
-   Sine/cosine/tangent
-   Identities
-   Vectors

## Episode 49: Linear Algebra I

**Study:**

-   Scalars
-   Vectors
-   Matrices
-   Multiplication
-   Transpose
-   Inverse concepts
-   Systems of equations

## Episode 50: Linear Algebra II

**Study:**

-   Dot/cross products
-   Transformations
-   Basis
-   Determinants
-   Eigenvalue/eigenvector concepts

## Episode 51: Coordinate Systems

**Study:**

-   World, base, sensor and end-effector frames
-   Position vectors
-   Frame changes

## Episode 52: Rotations

**Study:**

-   2D/3D rotations
-   Rotation matrices
-   Euler angles
-   Roll/pitch/yaw
-   Quaternion concepts

## Episode 53: Homogeneous Transformations

**Study:**

-   Translation + rotation
-   Transform composition
-   Inverses
-   Frame chains

**Exercise:** Implement in Python.

## Episode 54: Calculus for Robotics

**Study:**

-   Derivatives
-   Rates
-   Position/velocity/acceleration
-   Integration
-   Differential-equation concepts

## Episode 55: Probability

**Study:**

-   Random variables
-   Mean
-   Variance
-   Gaussian distributions
-   Conditional probability
-   Bayes
-   Sensor uncertainty

------------------------------------------------------------------------

# Module 8: Robot Kinematics (Episodes 56-62)

## Episode 56: DOF and Configuration Space

**Study:**

-   Joints
-   Degrees of freedom
-   Configuration
-   C-space
-   Workspace
-   Constraints

## Episode 57: Forward Kinematics

**Study:**

-   Joint variables → end-effector pose
-   Planar arms
-   Matrix formulations

## Episode 58: Denavit-Hartenberg Parameters

**Study:**

-   Link frames
-   DH parameters
-   Transformation chains

## Episode 59: Inverse Kinematics

**Study:**

-   Desired end-effector pose → joint configuration
-   Geometric methods
-   Analytical/numerical concepts
-   Multiple solutions
-   Constraints

## Episode 60: Jacobians

**Study:**

-   Differential motion
-   Joint velocity
-   End-effector velocity
-   Jacobian mapping

## Episode 61: Singularities

**Study:**

-   Singular configurations
-   Rank concepts
-   Lost DOF
-   Manipulability concepts

## Episode 62: Kinematics Project

**Build a simulated arm with:**

-   FK
-   IK
-   Frame visualization
-   End-effector trajectories

------------------------------------------------------------------------

# Module 9: Dynamics and Control (Episodes 63-71)

## Episode 63: Robot Dynamics

**Study:**

-   Mass
-   Inertia
-   Forces
-   Torque
-   Gravity
-   Friction
-   Equations of motion

## Episode 64: Dynamic System Modeling

**Study:**

-   Differential equations
-   Inputs/outputs
-   State variables
-   State-space concepts

## Episode 65: Feedback Control

**Study:**

-   Plant
-   Controller
-   Setpoint
-   Sensor
-   Feedback
-   Error
-   Disturbance
-   Stability concepts

## Episode 66: PID

**Study:**

-   P/I/D terms
-   Rise time
-   Overshoot
-   Settling
-   Steady-state error
-   Tuning
-   Saturation
-   Windup concepts

## Episode 67: Motor Control

**Study:**

-   Position/velocity control
-   Encoder feedback
-   Setpoints
-   Closed-loop control

## Episode 68: State-Space Control

**Study:**

-   State vectors/equations
-   Controllability
-   Observability
-   State-feedback concepts

## Episode 69: Trajectory Generation

**Study:**

-   Position, velocity and acceleration profiles
-   Smooth motion
-   Polynomial trajectory concepts

## Episode 70: Motion Control

**Study:**

-   Path vs trajectory
-   Trajectory tracking
-   Feedback tracking

## Episode 71: Control Project

**Build:**

-   A closed-loop simulation or suitably safe physical experiment
-   An analysis of its response

------------------------------------------------------------------------

# Module 10: ROS 2 (Episodes 72-81)

## Episode 72: ROS 2 Foundations

**Study:**

-   Middleware
-   Distributed robotic systems
-   DDS concepts
-   ROS graph
-   Workspaces
-   Packages
-   Distributions

## Episode 73: ROS 2 CLI and Turtlesim

**Study:**

-   Environment
-   CLI
-   `ros2 run`
-   Node/topic/service/action inspection
-   `rqt`
-   Graph exploration

## Episode 74: Nodes and Topics

**Study:**

-   Publishers
-   Subscribers
-   Messages
-   Topic types
-   QoS introduction
-   Communication graphs

## Episode 75: Services and Actions

**Study:**

-   Clients/servers
-   Request-response
-   Goals
-   Feedback/results
-   When to choose each communication model

## Episode 76: Parameters, Launch and Logging

**Study:**

-   Parameters
-   YAML
-   Launch files
-   Logging
-   Namespaces
-   Remapping

## Episode 77: ROS 2 Packages

**Study:**

-   Python packages
-   C++ packages
-   Dependencies
-   Build systems
-   `colcon`
-   Workspace sourcing

## Episode 78: TF2

**Study:**

-   Transform trees
-   Static/dynamic transforms
-   Parent-child frames
-   Transform lookup

## Episode 79: URDF and Xacro

**Study:**

-   Links
-   Joints
-   Visual/collision/inertial data
-   Robot trees
-   Macros

## Episode 80: ros2_control

**Study:**

-   Hardware interfaces
-   Controllers
-   Joint states/commands
-   Controller management

## Episode 81: ROS 2 Project

**Build a multi-node system with:**

-   Robot model
-   TF
-   Communication
-   Configuration
-   Launch
-   Controller interfaces

------------------------------------------------------------------------

# Module 11: Robot Simulation (Episodes 82-86)

> Use **modern Gazebo**, not Gazebo Classic, for a new curriculum.

## Episode 82: Gazebo Fundamentals

**Study:**

-   Worlds
-   Models
-   Physics
-   Rendering
-   SDF concepts
-   Simulation time
-   Spawning

## Episode 83: Simulated Sensors

**Study:**

-   Cameras
-   IMUs
-   LiDAR/range sensors
-   Update rates
-   Sensor noise

## Episode 84: ROS 2 + Gazebo

**Integrate with simulation:**

-   Robot descriptions
-   ROS topics
-   Sensors
-   Controllers
-   TF

## Episode 85: Simulated Mobile Robot

**Build a differential-drive robot with:**

-   Sensors
-   Controller
-   ROS interfaces

## Episode 86: Simulated Manipulator

**Build an arm with:**

-   Links
-   Joints
-   Controllers
-   Sensors
-   ROS integration

------------------------------------------------------------------------

# Module 12: Autonomous Robotics (Episodes 87-94)

## Episode 87: Odometry

**Study:**

-   Encoder measurements
-   Differential-drive kinematics
-   Pose estimation
-   Accumulated error

## Episode 88: Sensor Fusion

**Study:**

-   Noise
-   Bias
-   Drift
-   Complementary filters
-   Kalman-filter intuition
-   Multi-sensor estimates

## Episode 89: Localization

**Study:**

-   Pose
-   Uncertainty
-   Probabilistic localization
-   Particle-filter intuition
-   Map/odom frames

## Episode 90: Mapping and SLAM

**Study:**

-   Occupancy grids
-   SLAM
-   Loop-closure concepts
-   LiDAR SLAM
-   Visual-SLAM concepts

## Episode 91: Path Planning

**Study:**

-   Configuration spaces
-   Graph search
-   Dijkstra
-   A\*
-   Heuristics
-   Collision checking
-   Sampling-based concepts

## Episode 92: Navigation

**Study:**

-   Global/local planning
-   Costmaps
-   Obstacle avoidance
-   Goal navigation
-   Nav2 concepts

## Episode 93: Manipulation and Motion Planning

**Study:**

-   Grasping
-   Collision checking
-   Joint constraints
-   Trajectory execution
-   MoveIt 2 concepts

## Episode 94: Autonomous Robot Project

**Build a simulated system that can:**

1.  Sense
2.  Estimate
3.  Localize
4.  Plan
5.  Move
6.  Avoid obstacles
7.  Reach a goal

------------------------------------------------------------------------

# Module 13: Modern Robotics (Episodes 95-98)

## Episode 95: NVIDIA Isaac Sim

**Study:**

-   USD concepts
-   Assets
-   Physics
-   Articulations
-   Sensors
-   URDF import
-   OmniGraph
-   Python
-   ROS 2 bridge

## Episode 96: Robotics Perception

**Study:**

-   Camera calibration
-   Image processing
-   Detection/segmentation concepts
-   Depth
-   Point clouds
-   Pose estimation

## Episode 97: Robot Learning

**Study:**

-   ML for robotics
-   Reinforcement learning
-   Imitation learning
-   Policies
-   Observations/actions/rewards
-   Domain randomization
-   Sim-to-real concepts

## Episode 98: Modern Robot Architectures

**Study as integrated systems:**

-   Humanoids
-   Quadrupeds
-   Autonomous vehicles
-   Cobots
-   Mobile manipulators
-   Warehouse robots
-   Physical AI

------------------------------------------------------------------------

# Module 14: Complete Capstone (Episodes 99-100)

## Episode 99: Engineer the Robot

**Produce:**

1.  Problem statement
2.  User requirements
3.  Engineering specifications
4.  System architecture
5.  Mechanical architecture
6.  CAD
7.  Electronics architecture
8.  PCB/schematic where appropriate
9.  Power budget
10. Sensors
11. Actuators
12. Embedded architecture
13. ROS 2 architecture
14. Control architecture
15. Simulation model
16. Autonomy architecture
17. BOM
18. Test plan
19. Risk/failure analysis
20. Build schedule

## Episode 100: Integrate, Test and Teach

**Present the complete system:**

-   Requirements
-   Mechanics
-   Electronics
-   Power
-   Firmware
-   Sensors
-   Actuators
-   Mathematics
-   Kinematics
-   Control
-   ROS graph
-   Simulation
-   Autonomy
-   Software architecture
-   Testing
-   Failures
-   Tradeoffs
-   Improvements

**Suggested episode:**\
`Episode 100/100: I Built an Autonomous Robot From Scratch | 100 Episodes of Robotics Engineering`

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

# Episode 101: Career, Hiring and Monetization

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
