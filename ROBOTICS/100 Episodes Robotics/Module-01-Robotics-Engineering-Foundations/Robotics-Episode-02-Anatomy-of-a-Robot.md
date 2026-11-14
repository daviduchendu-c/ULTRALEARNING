# Day 2: Anatomy of a Robot

*100 Days of Robotics Engineering, Module 1: Robotics Engineering Foundations*

**Time:** about 4 hours
**Level:** beginner. No electronics knowledge is needed. You should be comfortable with Day 1's loop and with basic arithmetic, because most of today's work is unit-by-unit calculation.

**By the end of this day, you will have:**

- A layered architecture for a small wheeled mobile robot, with every block mapped to a real class of part
- Worked calculations for gear ratio, wheel speed, encoder resolution, regulator losses, battery runtime and motor driver sizing
- A failure plan that says what the robot does when the link, the battery or a sensor fails

**What you need:**

- Pen and paper, a drawing tool, or a text editor for a Markdown note
- A calculator or Python's interactive prompt (Python 3.10 or newer) for the arithmetic
- A viewer that renders Mermaid diagrams: GitHub and VS Code do it automatically, or use the [Mermaid Live Editor](https://mermaid.live)
- No hardware. Optional: a free [Wokwi](https://wokwi.com/) browser account for the practice item in the sources
- For later, not today: ROS 2. The long-term support releases named in this lesson were current when it was written, so check the current release before you install anything.

**Prerequisites:**

- Day 1: What Is Robotics? (the sense, compute, decide and act loop, and open loop versus closed loop)
- Your five-robot table from Day 1, Exercise 1, if you made it
- No electronics knowledge yet

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How compute moved onto the robot](#how-compute-moved-onto-the-robot)
- [The map of a mobile robot](#the-map-of-a-mobile-robot)
- [Frames, joints and transmissions](#frames-joints-and-transmissions)
- [Batteries and regulators](#batteries-and-regulators)
- [Actuators and motor drivers](#actuators-and-motor-drivers)
- [Sensors](#sensors)
- [Buses: how the parts talk](#buses-how-the-parts-talk)
- [Compute: MCUs, SBCs, CPUs and GPUs](#compute-mcus-sbcs-cpus-and-gpus)
- [Firmware and high-level software](#firmware-and-high-level-software)
- [Hands-on exercises](#hands-on-exercises)
- [Project: draw the architecture of a small wheeled robot](#project-draw-the-architecture-of-a-small-wheeled-robot)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Yesterday you drew a robot as four boxes: sensors, compute, actuators and feedback. Today you open the boxes. A real robot is a stack of layers: a mechanical body, a power system, electronics that switch current into motors, a microcontroller running firmware, and usually a larger computer running high-level software.

Most failures happen at the boundaries between layers. A motor browns out the computer. A sensor sits on the wrong bus. Firmware cannot keep up with its control loop. When an interviewer asks you to "walk me through the architecture," they want to hear that you know the layers and how they connect, so today's sections go from the bottom of the stack to the top.

[Wikipedia: Microcontroller](https://en.wikipedia.org/wiki/Microcontroller) is the reference for today. Read the introduction and the section on embedded design.

---

## How compute moved onto the robot

| Period | What happened | Why it matters today |
|---|---|---|
| 1961 | Unimate, the first industrial arm, goes to a General Motors plant and stores its program on a magnetic drum. | Early robots were fixed machines with the program inside the machine. |
| 1970s to 1980s | The computer lives in a cabinet beside the robot. | Control logic and moving parts are separate boxes connected by cables. |
| 1990s to 2000s | Cheap microcontrollers put real-time control on the robot itself. | Fast loops run next to the motors. |
| 2005 | Arduino makes microcontrollers easy for students. | The entry point for most hobbyists and many engineers. |
| 2007 | ROS is first released. | Robot software becomes shareable. |
| 2012 | The Raspberry Pi arrives, at about $35. | A full Linux computer fits on a small robot. |
| 2017 | ROS 2 is first released, as the successor to ROS 1. NVIDIA Jetson boards then bring GPUs to small robots. | Perception models run on the robot, not on a remote server. |

**Takeaway:** the result is a split. A small real-time chip runs the fast loops, and a larger general-purpose computer runs planning and perception. Nearly every design in this course uses some version of that split.

---

## The map of a mobile robot

This is Day 1's loop, expanded into the layers you will meet in this series.

```
 ┌─────────────────────────── HIGH-LEVEL SOFTWARE (SBC, Linux, ROS 2) ───────────────────────────┐
 │  mapping / localization   path planning   perception (camera, LiDAR)   teleop / UI   logging  │
 └───────────────────────────────────────────────▲───────┬───────────────────────────────────────┘
                                 odometry, IMU    │       │  velocity commands
                                                  │       ▼
 ┌──────────────────────────────── FIRMWARE (MCU, real time) ─────────────────────────────────────┐
 │  read encoders + IMU   wheel speed PID   motor PWM output   safety watchdog   battery monitor   │
 └──────▲─────────────▲──────────────────────────┬────────────────────────────────────────────────┘
        │ I2C         │ GPIO pulses               │ PWM + direction
 ┌──────┴─────┐ ┌─────┴──────┐            ┌───────▼───────┐                  ┌─────────────┐
 │ IMU,       │ │ Wheel      │            │ MOTOR DRIVER  │── high current ─▶│ DC MOTORS   │
 │ distance   │ │ encoders   │            │ (H-bridge)    │                  │ + gearbox   │
 └────────────┘ └────────────┘            └───────────────┘                  └─────────────┘

 POWER:      battery ─▶ fuse ─▶ switch ─┬─▶ motor driver (raw battery voltage)
                                        ├─▶ 5 V regulator ─▶ SBC
                                        ├─▶ 3.3 V regulator ─▶ MCU and sensors
                                        └─▶ voltage divider ─▶ MCU ADC pin (battery monitor)

 MECHANICS:  frame, motor mounts, wheels, bearings, caster (holds and carries everything above)
```

Read it from the bottom up. The frame holds everything. Power feeds everything. Electronics move current. Firmware closes the fast loops, and high-level software makes slower, smarter decisions.

The TurtleBot3 Burger from ROBOTIS follows this split. A Raspberry Pi runs ROS 2, an OpenCR board (an STM32 microcontroller) runs firmware, two Dynamixel smart servos drive the wheels, and a small 360-degree LiDAR scans the room.

**Reference image to find:** search "TurtleBot3 Burger hardware components" on the ROBOTIS e-Manual and match each labeled part to a box above.

---

### Frames, joints and transmissions

A motor is a sprinter: fast, but weak. A transmission trades that speed for force, the way a weightlifter trades distance for load. The frame is the skeleton that keeps everything in place while forces push and pull on it.

The frame (chassis) is the structure. Common materials are laser-cut acrylic (cheap, brittle), 3D-printed PLA or PETG (fast to iterate), aluminum extrusion (strong, modular) and sheet metal. Joints are the places where parts move relative to each other. A revolute joint rotates, like an elbow or a wheel axle. A prismatic joint slides, like a linear rail. Each joint usually adds one degree of freedom (DOF), one independent way to move. Transmissions include spur gear trains, planetary gearboxes, belts and pulleys, lead screws, and harmonic drives (strain-wave gears, used in precise arms). Bearings carry the load so the motor shaft does not have to.

A gearbox with ratio `N:1` divides speed by `N` and multiplies torque by `N`, minus friction losses. Say a motor spins at 10,000 rpm with no load and gives 0.01 N·m of torque. You add a 50:1 gearbox at 80 percent efficiency.

- Output speed: 10,000 / 50 = 200 rpm.
- Output torque: 0.01 × 50 × 0.8 = 0.4 N·m.
- With a 65 mm wheel, ground speed: π × 0.065 × 200 / 60 ≈ 0.68 m/s.

A 100:1 gearbox gives about 0.34 m/s with twice the torque. Choosing a gear ratio means choosing a point on the speed-versus-strength curve, and the heavier the robot or the steeper the slope, the further toward torque you go.

The mistake to watch for is hanging a wheel directly on the motor shaft with no bearing. The gearbox then carries the robot's weight and wears out, and you spot it as play or wobble in the wheel after a few hours of running. Two related slips: choosing a frame for looks instead of stiffness, since a floppy frame makes encoders and sensors disagree, and ignoring backlash, the small free play in gears, which hurts precise positioning.

In practice, hobby and education platforms are built from exactly these pieces (3D-printed brackets, extrusion and gearmotors), and industrial arms use harmonic drives where joints must position precisely.

---

### Batteries and regulators

The battery is the fuel tank. The regulator is the pump that holds pressure steady while the tank drains.

| Battery type | Cell voltage | Notes |
|---|---|---|
| Alkaline AA | 1.5 V nominal | Cheap, weak under high current, not rechargeable |
| NiMH AA | 1.2 V nominal | Rechargeable and safe, heavy for the energy it holds |
| Li-ion (18650 cell) | 3.6 to 3.7 V nominal, 4.2 V full | High energy, needs protection and proper charging |
| LiPo pack | 3.7 V nominal per cell, 4.2 V full | Very high current and light, fire risk if punctured or overcharged |

Cells in series add voltage. Cells in parallel add capacity. A 2S Li-ion pack, two cells in series, is 7.4 V nominal, 8.4 V full and about 6 V nearly empty. Battery energy in watt-hours (Wh) is voltage times capacity in amp-hours.

A regulator turns the changing battery voltage into a fixed rail. A linear regulator (such as the LM7805 or AMS1117-3.3) burns the extra voltage as heat. It is simple and quiet, but wasteful. A switching (buck) regulator chops and smooths the voltage. It is usually 85 to 95 percent efficient and a little noisier.

Say your Raspberry Pi needs 5 V at 1 A from a 7.4 V battery.

- Linear: the regulator drops 2.4 V at 1 A, so it turns 2.4 W into heat. Efficiency is 5 / 7.4 ≈ 68 percent, and it needs a heatsink.
- Buck at 90 percent efficiency: the input is 5 / 0.9 ≈ 5.56 W, so only about 0.56 W is lost.

For the battery, a 2S pack of 2,500 mAh cells holds 7.4 V × 2.5 Ah ≈ 18.5 Wh. At a 10 W average load, it lasts under two hours before losses. Capacity printed on cheap cells is often exaggerated, so buy cells from known brands and measure runtime yourself.

The classic mistake is powering the SBC and the motors from the same regulator. When the motors start, the voltage dips and the computer reboots. That is a brownout, and you spot it as a reboot that happens exactly when the robot begins to move. Two related slips: no fuse, since a shorted wire on a LiPo can melt its insulation in seconds, and underpowering the Raspberry Pi 5, which is designed for a 5 V, 5 A supply and limits USB current on a weaker one.

Drone and hobby robot builders buy LiPo packs for their current and weight, and each pack carries the fire warnings in the table for a reason.

---

### Actuators and motor drivers

A microcontroller pin is a light switch. It can signal, but it cannot carry much current. A motor driver is the relay that lets a small signal control a large current. An MCU pin can safely supply about 20 mA. A small gearmotor can draw over 1 A when it stalls, which is 50 times more.

Common robot actuators:

- **Brushed DC gearmotor:** cheap and simple. It needs an H-bridge driver, and an encoder for closed-loop control.
- **Hobby servo:** motor, gearbox, position sensor and controller in one box. You send a pulse and it holds an angle.
- **Stepper motor:** moves in fixed steps. Often run open loop, so Day 1's point about missed steps applies.
- **Brushless DC (BLDC) motor:** efficient and powerful. It needs an ESC or FOC driver. Drones and many modern robot joints use them.
- **Smart servo:** a servo with a digital bus and feedback, such as the ROBOTIS Dynamixel.

A driver for a brushed DC motor is usually an H-bridge: four switches around the motor. Close one diagonal pair and current flows one way. Close the other pair and it flows the other way. Switching them with PWM (pulse-width modulation: a signal switched on and off quickly, where the fraction of on-time sets the average power) controls speed.

```
        V+                     V+
        │                      │
     [S1]                    [S3]
        ├────── (MOTOR) ───────┤
     [S2]                    [S4]
        │                      │
       GND                    GND

   S1 + S4 closed: forward      S3 + S2 closed: reverse
   S1 + S2 closed: short circuit ("shoot-through"), never do this
```

Compare two common driver boards running a motor at 1 A from a 7.4 V battery:

- **L298N** (older bipolar design): drops roughly 2 V or more at 1 A. The motor sees about 5.4 V, and the driver turns about 2 W into heat per channel.
- **TB6612FNG** (MOSFET design, about 0.5 Ω per channel): drops about 0.5 V at 1 A. The motor sees about 6.9 V, and the driver wastes about 0.5 W.

The TB6612FNG is rated for 1.2 A continuous and 3.2 A peak per channel, with motor voltage up to 15 V (check the datasheet for your exact board). If your motor stalls at 3 A for long, you need a bigger driver.

The common mistake is sizing the driver to the motor's running current. Compare the motor's stall current with the driver's continuous rating instead. You spot it when the driver overheats or shuts down whenever the robot pushes against a wall or a wheel jams. Related slips: leaving out a common ground between the MCU and the driver, so the control signals have no reference, and driving a motor directly from an MCU pin, which can destroy the pin or the whole chip.

Brushes also wear out, and the motor makes electrical noise that can reset nearby electronics. Plan for capacitors across the motor terminals and separate wiring for motor power. Module 2 explains why.

Dual H-bridge boards like the L298N and TB6612FNG are standard in hobby and education wheeled robots, while drones and many modern robot joints use BLDC motors with their own drivers.

---

### Sensors

Each sensor is narrow. A distance sensor is one finger poking forward. It knows one thing, in one direction, with some error.

Robot sensors fall into two groups. Proprioceptive sensors measure the robot itself: wheel encoders, IMUs, motor current, battery voltage. Exteroceptive sensors measure the world: ultrasonic rangers, time-of-flight (ToF) distance sensors, LiDAR, cameras and bump switches. Every sensor has a typical limit:

| Sensor | Real example | Measures | Typical limit |
|---|---|---|---|
| Quadrature encoder | Magnetic encoder on a gearmotor | Wheel rotation | Cannot see wheel slip |
| IMU | MPU-6050, BNO055 | Acceleration, rotation rate | Drifts over time when integrated |
| Ultrasonic | HC-SR04 | Distance, about 2 cm to 4 m | Wide cone, soft surfaces absorb sound |
| ToF laser | VL53L0X | Distance, up to about 2 m | Sunlight and dark surfaces reduce range |
| 2D LiDAR | Slamtec RPLIDAR A1 | 360-degree distance scan | Sees one plane only; glass is invisible |
| Camera | Raspberry Pi Camera Module 3 | Images | Needs light and heavy compute |

A wheel encoder gives 12 counts per motor revolution, the motor has a 50:1 gearbox, and the wheel is 65 mm in diameter.

- Counts per wheel revolution: 12 × 50 = 600.
- Wheel circumference: π × 0.065 m ≈ 0.204 m.
- Distance per count: 0.204 / 600 ≈ 0.34 mm.

Each tick means the wheel moved about a third of a millimeter, if it did not slip. The encoder cannot report slip, which is one reason to add an IMU.

The most common mistake is trusting one sensor. Every sensor has a blind spot, so real robots fuse several. You spot it when the robot's belief about its position slowly diverges from reality while every reading looks fine. Related slips: ignoring update rate, since an HC-SR04 needs tens of milliseconds per reading and at 1 m/s the robot moves several centimeters between readings, and buying a 5 V sensor module for a 3.3 V board without checking logic levels.

The parts in this table, such as the HC-SR04, VL53L0X and MPU-6050, are staples of hobby and education robots, and the same sensor families scale up into industrial products.

---

### Buses: how the parts talk

A bus is a shared road with traffic rules. Some roads have one lane and a traffic officer (I2C). Some are private two-way links (UART). Some are rugged routes built for noise (CAN).

| Bus | Wires | Typical speed | Typical use | Watch out for |
|---|---|---|---|---|
| UART (serial) | TX, RX, GND | 9,600 to 1,000,000+ baud | MCU to SBC link, GPS | Only two devices; both must agree on the baud rate |
| I2C | SDA, SCL, GND | 100 kHz or 400 kHz typical | IMUs, small sensors, displays | Address conflicts; short wires only |
| SPI | MOSI, MISO, SCK, one CS per device | 1 to tens of MHz | Fast sensors, displays, SD cards | Many wires for many devices |
| CAN | CAN-H, CAN-L | 125 kbit/s to 1 Mbit/s (classic) | Motor controllers, cars, industrial | Needs transceivers and 120 Ω termination |
| USB | D+, D−, power, GND | Up to gigabits | Cameras, LiDAR, MCU to SBC | Cable length; hub power |
| Ethernet | Twisted pairs | 100 Mbit/s to 1 Gbit/s+ | Industrial arms, 3D LiDAR | More power and bigger connectors |

Say you want two VL53L0X distance sensors on one I2C bus, facing left and right. Both ship with the same default address, 0x29. Connect both, and they answer at once, so the data is garbage. The fix is to wire each sensor's shutdown pin (XSHUT) to an MCU pin. Power up one sensor, change its address in software (for example to 0x30), then power up the second. "What happens when two devices share an address?" is a common interview question, and that is the answer to give.

The mistake to watch for is running long I2C wires through a noisy robot body. I2C was designed for short traces on one board, so you spot the problem as readings that work on the bench and fail once the robot is assembled or the motors run. Related slips: crossing TX and RX on UART (one device's TX goes to the other device's RX), and forgetting the ground wire, since every bus needs a shared reference.

CAN shows up in cars and industrial equipment, and USB or Ethernet link the cameras and LiDARs on commercial mobile robots.

---

### Compute: MCUs, SBCs, CPUs and GPUs

A restaurant has a line cook and a head chef. The line cook flips the steak at exactly 90 seconds, every time. The head chef plans the menu and handles surprises, but can be interrupted. The MCU is the line cook. The SBC is the head chef.

| Class | Typical examples | Strength | Weakness |
|---|---|---|---|
| MCU (microcontroller unit: one chip with processor, memory and I/O pins) | Arduino Uno R3 (ATmega328P), Raspberry Pi Pico (RP2040), ESP32, STM32 | Deterministic timing, microsecond response, low power, cheap | Little memory, no operating system, limited math |
| SBC with CPU (single-board computer: a full computer on one board) | Raspberry Pi 4 or 5 | Runs Linux, ROS 2, Python, networking, cameras | Not real time by default, slower to boot, more power |
| SBC with GPU (a processor with many simple cores, good at the same math on lots of data) | NVIDIA Jetson Orin Nano | Runs neural networks on camera images | Costs more, draws more power, needs cooling |

The numbers give a feel for the gap. The Arduino Uno R3's ATmega328P runs at 16 MHz with 2 KB of RAM and 32 KB of flash. The Raspberry Pi Pico's RP2040 has two Arm Cortex-M0+ cores at 133 MHz and 264 KB of RAM. A Raspberry Pi 5 has a quad-core Arm Cortex-A76 at 2.4 GHz and 2 GB to 16 GB of RAM, depending on the model (check the current specifications). The Pico alone has more than 100 times the Uno's RAM.

Say you want a wheel-speed controller to run at 1 kHz on a 16 MHz MCU. Each clock cycle takes 62.5 ns, so one 1 ms loop gets 16,000 cycles. Reading an encoder and computing a PID update (a proportional-integral-derivative controller, covered in Module 9) takes a few hundred cycles. It fits easily, and with no operating system it runs on time every time.

On a Raspberry Pi running standard Linux, the same loop usually works. But the operating system can pause your program for several milliseconds to handle Wi-Fi or disk activity. For a wheel that is tolerable. For a balancing robot, it can mean falling over. So most mobile robots run the fast loop on an MCU and send it speed commands from the SBC.

The MCU and SBC split adds a communication link that can fail. If the serial cable drops, the MCU needs a rule, usually "stop the motors." You design that rule. It does not come for free.

The common mistake is picking a Jetson for a robot that only follows a line. Match compute to the hardest task, and you spot over-sizing when the board idles at low load while the battery drains. Related slips: running motor control from Raspberry Pi pins in a Python loop with `time.sleep()` and then wondering why the speed jitters, and forgetting that the Pi's GPIO pins are 3.3 V and not 5 V tolerant, so a 5 V sensor output can damage the pin.

Many commercial mobile robots pair a microcontroller for motor control with a larger computer for perception and planning, which is the split in the table.

---

### Firmware and high-level software

Firmware is a reflex: you pull your hand off a hot stove before you think. High-level software is planning: choosing a route across town. Both are needed, and they run at very different speeds.

Firmware runs on the MCU, usually in C or C++ (Arduino framework, Pico SDK, ESP-IDF, STM32Cube) or MicroPython. It reads sensors, runs fast control loops (100 Hz to 10 kHz), and enforces safety, such as stopping the motors when commands stop arriving. High-level software runs on the SBC under Linux, usually as ROS 2 nodes written in Python or C++. It handles mapping, localization, planning, perception and user interfaces at 1 to 30 Hz. The long-term support releases when this lesson was written were ROS 2 Humble (Ubuntu 22.04, supported to 2027) and ROS 2 Jazzy (Ubuntu 24.04, supported to 2029). Check the current release before you install. The bridge between the two layers is usually a serial link with a small custom protocol, or micro-ROS, which lets an MCU appear as a ROS 2 node.

Here is a firmware skeleton for a wheel controller, in Arduino-style C++:

```cpp
const unsigned long LOOP_MS = 10;      // 100 Hz control loop
const unsigned long TIMEOUT_MS = 500;  // stop if no command for 0.5 s
unsigned long lastCommand = 0;
float targetSpeed = 0.0;               // metres per second

void loop() {
  if (Serial.available()) {            // command from the SBC, e.g. "0.25\n"
    targetSpeed = Serial.parseFloat();
    lastCommand = millis();
  }
  if (millis() - lastCommand > TIMEOUT_MS) {
    targetSpeed = 0.0;                 // watchdog: lost link means stop
  }
  float measured = readWheelSpeed();   // from encoder counts
  setMotorPWM(pid(targetSpeed, measured));
  delay(LOOP_MS);                      // fine for a sketch; real firmware uses timers
}
```

This is a sketch, not a complete program: `readWheelSpeed`, `setMotorPWM` and `pid` are placeholders you would write for your own hardware. The `delay()` loop drifts as the work inside it grows, so real firmware uses hardware timers or an RTOS (real-time operating system) such as FreeRTOS, which Module 3 covers.

On the SBC side, a ROS 2 system for the same robot would publish velocity commands on `/cmd_vel` and receive wheel odometry on `/odom`. You will build that in Module 10.

The usual mistake is putting everything in firmware because it works, then being unable to add a map or a camera. The reverse is just as common: putting everything on the SBC and then fighting timing jitter in motor control. You also spot a missing watchdog in testing: unplug the link while the robot moves, and if it keeps going, the last command is still running.

Job postings for embedded robotics roles often ask for STM32 or ESP32 firmware next to Linux and ROS 2 experience, and the engineers hired fastest understand the interface between those two layers.

---

## Hands-on exercises

**Exercise 1: Parts hunt.** Open the ROBOTIS TurtleBot3 Burger specifications page (search "TurtleBot3 specifications" on emanual.robotis.com). List its SBC, MCU board, motors, LiDAR, battery and frame material. Put each one into a layer of the map of a mobile robot.

**Exercise 2: Compute sizing.** For each task, choose an MCU, an SBC or an SBC with GPU, and write one sentence on why: blink an LED at exactly 1 Hz; detect people in a camera image at 10 frames per second; build a 2D map of a room; balance a two-wheeled robot. Check: the LED and the balancing robot need an MCU, the camera task needs an SBC with GPU, and the mapping task runs on an SBC.

**Exercise 3: Encoder math.** A motor encoder gives 7 counts per motor revolution (counting one edge). The gearbox is 30:1 and the wheel is 80 mm in diameter. Compute the distance per count. Check: 7 × 30 = 210 counts per wheel revolution, and π × 0.08 / 210 ≈ 1.2 mm.

**Exercise 4: Power math.** Your robot draws an average of 6 W. Your pack is 3S LiPo, 2,200 mAh (11.1 V nominal). Compute the watt-hours and the theoretical runtime. Cut the runtime by 20 percent for losses, and say why the real runtime is still lower. Check: 11.1 × 2.2 ≈ 24.4 Wh, which is about 4.1 hours, or about 3.3 hours after the 20 percent cut.

**Exercise 5: Bus assignment.** You have an IMU (I2C), a GPS (UART), a small display (SPI) and a link to the SBC (USB serial). Draw the wires from each to an ESP32 or Pico. Mark which devices share a bus and which need their own pins.

---

## Project: draw the architecture of a small wheeled robot

The skill today is reasoning about layers. A diagram and one page of calculations show it, so this project is a single note instead of a code repository. You will reuse the diagram on Day 5: First Robot System Design.

**Goal:** the architecture of a small indoor differential-drive robot (two driven wheels plus a caster) that drives by remote command and avoids obstacles in front of it.

### Repository layout

```
day-02-anatomy-of-a-robot/
├── README.md             # optional: one paragraph on what the note contains
├── architecture.md       # the diagram, the layer table, the messages, the calculations and the failure plan
└── architecture.png      # optional: an export or a photo of a hand drawing
```

A single Markdown file is enough. If you commit it to GitHub, name the repository `100-days-robotics-day-02-anatomy-of-a-robot`.

### Step 1: start from the diagram

Paste this Mermaid diagram into `architecture.md` inside a fenced code block, and fix anything you disagree with.

```mermaid
flowchart LR
  BAT[2S Li-ion pack] --> FUSE[Fuse + switch]
  FUSE --> BUCK5[5 V buck regulator]
  FUSE --> DRV[Dual motor driver]
  BUCK5 --> SBC[SBC: Raspberry Pi]
  SBC <-->|USB serial| MCU[MCU: Pico or ESP32]
  MCU -->|PWM + DIR| DRV
  DRV --> ML[Left gearmotor]
  DRV --> MR[Right gearmotor]
  ENC[Wheel encoders] -->|GPIO| MCU
  IMU[IMU] -->|I2C| MCU
  TOF[ToF distance sensor] -->|I2C| MCU
```

### Step 2: add the battery voltage monitor

Add the monitor to the diagram. Hint: an ADC pin behind a voltage divider.

### Step 3: split the software

Write which code runs on the MCU and which on the SBC, and the messages between them: target wheel speeds going down, and encoder counts, IMU data and battery voltage coming up.

### Step 4: name real parts

Name one real example part per layer, taken from parts in this lesson or from datasheets you have checked.

### Step 5: run the calculations

Run the gear, encoder, runtime and motor driver checks from the sections on frames, batteries and motor drivers with your chosen parts. Show units on every line.

### Step 6: write the failure plan

Write what happens if the USB link drops, the battery runs low, or the front sensor reads nothing.

### Done when

- You can point at any arrow in the diagram and say what flows on it and what breaks if it stops.
- Every calculation has units and matches the parts you named.
- The note contains the diagram, the software split, the named parts, the calculations and the failure plan.
- The battery monitor is on the diagram, with the divider and the ADC pin.

---

## Check your understanding

1. A robot must stop within 5 ms when a bump switch is pressed. Should that logic live on the MCU or the SBC? Why?
2. Your robot reboots its Raspberry Pi every time it starts driving uphill. Name the most likely cause and two fixes.
3. Why might a designer choose a 100:1 gearbox over a 30:1 for a heavy robot, and what do they give up?
4. Two I2C sensors return nonsense only when both are connected. What do you check first?
5. An L298N and a TB6612FNG can both drive your motors. Give two reasons to prefer the TB6612FNG, and one case where neither is enough.
6. Your wheel encoders report that the robot drove 3 m in a straight line, but it actually drove 2.7 m. Which sensor could help detect this, and why can't the encoder alone?
7. Why does the firmware need a command timeout even if the SBC software is well tested?

**Answers (check after you try):**

1. The MCU. It responds in microseconds with no operating system to pause it. The SBC can be delayed by Linux scheduling, and the USB link adds latency.
2. A voltage dip (brownout) when the motors draw high current. Fixes: power the SBC from its own regulator, use a battery with higher current capacity, add bulk capacitance, and limit motor acceleration in firmware.
3. More torque at the wheel, to move the weight and climb. The cost is top speed, and gear friction rises.
4. Their addresses. Two devices with the same address conflict. Also check the pull-up resistors and the wire length.
5. Lower voltage drop, so more voltage reaches the motor, less heat, and a smaller board. Neither is enough when the motor's stall current exceeds about 3 A or the motor voltage exceeds their ratings.
6. An IMU's gyroscope detects rotation the wheels did not report, and a LiDAR or camera can measure position against walls. Encoders measure wheel rotation, not ground motion, so they cannot see slip.
7. Cables come loose, processes crash, and batteries sag. Firmware is the last line of defense, so it must assume the link can vanish at any moment.

---

## Sources to read and watch

**Read:**
- [Wikipedia: Microcontroller](https://en.wikipedia.org/wiki/Microcontroller)
- [Wikipedia: Single-board computer](https://en.wikipedia.org/wiki/Single-board_computer)
- [Wikipedia: Graphics processing unit](https://en.wikipedia.org/wiki/Graphics_processing_unit)
- [Wikipedia: Rotary encoder](https://en.wikipedia.org/wiki/Rotary_encoder)
- [Wikipedia: Inertial measurement unit](https://en.wikipedia.org/wiki/Inertial_measurement_unit)
- [Wikipedia: H-bridge](https://en.wikipedia.org/wiki/H-bridge)
- [Wikipedia: Pulse-width modulation](https://en.wikipedia.org/wiki/Pulse-width_modulation)
- [Wikipedia: Lithium polymer battery](https://en.wikipedia.org/wiki/Lithium_polymer_battery)
- [Wikipedia: Buck converter](https://en.wikipedia.org/wiki/Buck_converter)
- [Wikipedia: I²C](https://en.wikipedia.org/wiki/I%C2%B2C), [Serial Peripheral Interface](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface), [CAN bus](https://en.wikipedia.org/wiki/CAN_bus)
- [Wikipedia: Firmware](https://en.wikipedia.org/wiki/Firmware)
- [Wikipedia: Robot Operating System](https://en.wikipedia.org/wiki/Robot_Operating_System)
- [ROS 2 Jazzy documentation](https://docs.ros.org/en/jazzy/) (read the "Concepts" overview only)
- ROBOTIS e-Manual, TurtleBot3 section: search "TurtleBot3 e-Manual" on emanual.robotis.com
- [Raspberry Pi documentation](https://www.raspberrypi.com/documentation/): the Pico and Raspberry Pi 5 overview pages

**Tutorials:**
- GeeksforGeeks: search "Microcontroller vs Microprocessor" and "I2C communication protocol"
- SparkFun Learn: "I2C", "Serial Peripheral Interface (SPI)" and "Serial Communication" (search the titles on learn.sparkfun.com)
- [Arduino documentation](https://docs.arduino.cc/): the Arduino Uno R3 hardware page

**Watch:**
- [Articulated Robotics](https://www.youtube.com/@ArticulatedRobotics): the first few videos of the mobile robot build series, to see how a real hobby robot is split into layers
- [GreatScott!](https://www.youtube.com/@greatscottlab): search the channel for "motor driver" or "H-bridge"
- MIT OpenCourseWare: search ocw.mit.edu for "2.12 Introduction to Robotics" and skim the first lecture

**Practice:**
- [Wokwi](https://wokwi.com/): start a free Raspberry Pi Pico or ESP32 project in the browser, wire a virtual MPU-6050 and read it. No hardware needed.
- [Mermaid Live Editor](https://mermaid.live): paste the starter diagram from the project and edit it into your own version.

**Note on links:** open each one once. If a page has moved, search its title on the site named in the entry.

---

## Tomorrow

Day 3, **Robotics Engineering Workflow**, covers process. Today you learned what a robot is made of. Tomorrow you learn the order in which engineers make those decisions: problem, requirements, specifications, architecture, then mechanics, electronics, firmware and software, followed by simulation, prototyping, testing and documentation. Keep today's architecture note open. You will write the requirements it should have started from, and check which of your part choices survive them.

**Next:** [Day 3: Robotics Engineering Workflow](Robotics-Day-03-Robotics-Engineering-Workflow.md)
