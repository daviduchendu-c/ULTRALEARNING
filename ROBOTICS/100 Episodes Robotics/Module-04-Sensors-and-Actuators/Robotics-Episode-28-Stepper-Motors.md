# Day 28: Stepper Motors

*100 Days of Robotics Engineering, Module 4: Sensors and Actuators*

**Time:** about 3.5 hours
**Level:** intermediate. You need Day 26 (torque and speed), Day 27 (servo position control) and Day 13 (drivers, current limits and flyback protection). Today you learn how a motor can hold a position with no sensor at all, and where that trick fails.

**By the end of this day, you will have:**

- A NEMA 17 stepper that turns a measured number of steps per revolution, confirmed by counting a full turn
- A stepper driver set to a measured current limit, with the reference voltage calculated from its sense resistor
- An Arduino sketch that runs a trapezoidal acceleration profile and a homing routine using a limit switch
- A step-loss test that shows at what speed and load the motor starts to miss steps

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3 or newer, about $25 for an official board or $10 to $15 for a clone
- One NEMA 17 bipolar hybrid stepper, 200 steps per revolution (1.8 degrees per full step), rated 1.5 A to 1.7 A per phase, about $10 to $15. Check the label on the motor for the rated current before you set the driver.
- One A4988 stepper driver module, about $2 to $4, or one TMC2209 breakout if you want quieter operation and UART configuration. The TMC2209 breakout costs about $5 to $10. This lesson uses the A4988 for the main steps and mentions the TMC2209 where it differs.
- A 12 V regulated power supply rated for at least 2 A, about $10 to $15
- A small screwdriver to adjust the driver's potentiometer, and a multimeter with DC volts mode to measure the reference voltage
- A mechanical limit switch, about $1 to $2, and a short length of wire
- Optional: an 8 mm lead screw with a nut and a carriage, from a 3D printer kit or a linear stage, about $10 to $25. The linear stage makes the step-loss test easy to see.
- Arduino library AccelStepper, free and open source, installed through the Arduino Library Manager

**Prerequisites:**

- Day 26: torque, speed and the torque-speed curve
- Day 27: why position control needs a reference and why calibration matters
- Day 13: current limits, the driver's supply and the flyback diode inside a driver
- Day 12: fuses and the power budget for a 12 V rail

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How steppers developed](#how-steppers-developed)
- [Glossary](#glossary)
- [Stepper types](#stepper-types)
- [Steps and step angle](#steps-and-step-angle)
- [Driver circuits and current control](#driver-circuits-and-current-control)
- [Microstepping](#microstepping)
- [Torque and speed](#torque-and-speed)
- [Acceleration profiles](#acceleration-profiles)
- [Open-loop position and missed steps](#open-loop-position-and-missed-steps)
- [Homing](#homing)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a homed linear stage with step-loss tests](#project-a-homed-linear-stage-with-step-loss-tests)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A stepper motor is a motor that moves in fixed angular steps. If you send it 200 step pulses, a common stepper turns exactly one revolution, provided it does not miss any. That property lets a robot know its position without a sensor, which is why steppers are common in 3D printers, CNC machines, camera sliders and many low-cost robot arms.

The trade-off is the main risk of steppers. A stepper has no feedback, so the controller believes the position it has commanded, not the position the motor actually reached. A stepper that is pushed too fast, too hard or with too little current misses steps silently. The robot then ends up in the wrong place and the controller does not know. The skills you practice today are the ones that prevent this: setting the current correctly, choosing an acceleration profile the motor can follow, homing against a physical reference, and measuring how often steps are lost.

---

## How steppers developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| Late 1800s to early 1900s | Early electric motors with rotating magnetic fields are developed, and step-by-step rotation is used in telegraph and clock mechanisms. | The idea of advancing a shaft one tooth at a time is much older than the electronics. |
| 1930s to 1950s | Variable-reluctance steppers are used in some telephone and instrument mechanisms that need small, repeatable steps. | The reluctance principle is still used in some specialized motors. |
| 1960s to 1970s | Permanent-magnet and hybrid steppers appear for industrial positioning, and transistor drivers replace mechanical switches. | Hybrid steppers became the standard for precision positioning. |
| 1980s to 1990s | Stepper drivers with microstepping reach office and industrial equipment, and the NEMA 17 frame becomes a common size. | The NEMA 17 size is the default for hobby 3D printers and small machines today. |
| 2000s onward | Low-cost driver modules such as the A4988 and later the TMC2209 make steppers cheap for hobby robots, with quieter and more capable drive electronics. | A student can now drive a NEMA 17 for under $20 in parts. |

The motor's electromagnetic design has changed less than its driver. The modern driver chooses the current waveform, the step pattern and the protection, and the motor follows.

---

## Glossary

- **Stepper motor:** a motor that moves in discrete steps when its coils are energized in sequence.
- **Step angle:** the angle the rotor moves for one full step, often 1.8 degrees, which gives 200 steps per revolution.
- **Hybrid stepper:** a stepper with a toothed rotor and a permanent magnet, which is the most common type for precision work.
- **Phase:** a set of windings in a stepper that are energized together. A bipolar stepper has two phases, each with two wires.
- **NEMA 17:** a standard flange size of 42 mm by 42 mm for stepper motors. It does not define the step angle or the torque.
- **Holding torque:** the torque a stepper holds at standstill with its coils energized.
- **Pull-out torque:** the torque at which the stepper stops following the step pulses at a given speed.
- **Microstep:** a fraction of a full step, made by driving the two phases with sine and cosine currents.
- **Sense resistor:** a small, precise resistor on a driver board that the driver uses to measure the coil current.
- **Reference voltage (Vref):** the voltage a driver uses to set its current limit, usually adjusted with a potentiometer on the board.
- **Step loss:** a step pulse that does not turn the rotor by the expected angle, so the motor's real position falls behind the commanded position.
- **Homing:** moving a machine to a known reference position, such as a limit switch, to set the zero point of the controller's count.
- **Trapezoidal profile:** a speed plan with a constant acceleration, a constant speed and a constant deceleration, which looks like a trapezoid when speed is plotted against time.

---

## Stepper types

A stepper motor has a rotor and a stator, and the stator's coils are energized in a sequence that pulls the rotor from one position to the next. The types differ in how the rotor is built and how the poles are arranged.

The first principle is the magnetic detent. A permanent-magnet rotor is pulled into the positions where its magnetic poles line up with the energized stator poles. A variable-reluctance rotor is a soft iron piece with teeth, and it moves to the position that minimizes the magnetic reluctance of the path, which is the position with the least air gap. A hybrid stepper combines the two: a permanent-magnet rotor with two toothed halves, which gives a finer step and more holding torque than either pure type.

The worked example is a NEMA 17 hybrid stepper with 50 teeth on each half of the rotor and a 1.8-degree step. The step angle is 360 divided by (2 times 50 times 2), which is 1.8 degrees. The stepper gives 200 full steps per revolution, which matches the number printed on most NEMA 17 datasheets.

The common mistake is to buy a stepper by its frame size alone. A NEMA 17 can be 20 mm, 34 mm or 60 mm deep, and the depth sets the torque. Two NEMA 17 motors of the same flange can differ in holding torque by a factor of two or more. Check the holding torque, the rated current and the resistance per phase on the datasheet.

Unipolar and bipolar steppers are also different. A unipolar stepper has a center tap on each coil, which lets a simple driver energize half of each coil at a time, at the cost of torque. A bipolar stepper uses the whole coil with current reversal, and it gives more torque for the same size. Most NEMA 17 steppers sold for robots are bipolar, and the lesson assumes that type.

In the market, the hybrid bipolar NEMA 17 is the default precision stepper for small machines and mobile robots that need a simple linear axis. Industrial positioning uses larger hybrids and closed-loop drives with encoders, which combine a stepper's simplicity with feedback.

---

## Steps and step angle

A stepper's position is a count of steps, and its speed is the step rate in steps per second. A controller that knows the step angle and the step count knows the position without a sensor. That is the entire advantage of a stepper, and it is also the source of its risk.

The first principle is the relation between steps and angle. The rotor angle is theta = steps times step angle. For a 1.8-degree stepper, 200 steps give 360 degrees, and 1 step gives 1.8 degrees. For linear motion through a lead screw with a lead of L millimetres per revolution, the travel is steps times L divided by the steps per revolution. An 8 mm lead screw on a 200-step stepper gives 8 / 200 = 0.04 mm per full step.

The worked example compares a stepper and a servo for a 100 mm linear move. With the 8 mm lead screw and a full-step drive, the stepper needs 100 / 0.04 = 2500 steps. At 1/16 microstepping, which the lesson covers below, it needs 2500 times 16 = 40,000 microsteps, and each microstep moves 0.0025 mm. The stepper's resolution is thus 2.5 micrometres in the best case, and a servo with an encoder on the same screw may have a different resolution depending on its encoder count and the gearing.

The common mistake is to assume that the step count is the position. The step count is the commanded position. If a step is lost, the real position is different, and the controller does not know. A stepper that moves 100 mm when the controller counted 2500 steps may have moved 98 mm because of a missed step somewhere in the move. Measure the real position at the end of each test, and use a limit switch or a sensor to check the count.

The stepper's step rate is also the controller's clock. The Arduino Uno's timer generates pulses at a rate set by the code, and a delay() call in a loop limits the rate and the timing jitter. The AccelStepper library handles the timing for you, and it runs a non-blocking step loop, so the code can do other things between steps.

In the market, the step count is used in every CNC and 3D printer to command position, and the quality of the mechanical parts sets how accurate the step count is in practice.

---

## Driver circuits and current control

A stepper driver takes three logic signals from the microcontroller: a step pulse, a direction pin and an enable pin. It uses them to switch current through the stepper's two phases in a sequence, and it regulates the current in each phase to a set level. The driver is therefore a current controller, and the current setting is the most important one to get right.

The first principle is that a coil's current is set by the voltage across it divided by its resistance and its inductance. A stepper's coil resistance is small, often a few ohms, so a fixed supply voltage would drive a large current and burn the coil. The driver chops the supply, switching it on and off, so the average current in the coil stays at the set level. This is the same switching principle as the PWM in Day 13, applied to the coil's current rather than its voltage.

The A4988 sets its current limit from the reference voltage Vref and the sense resistor on the board. The usual formula is I_max = Vref / (8 times R_sense), where R_sense is commonly 0.1 Ω on many carrier boards. Check the value printed on the board or in its documentation, because some boards use different sense resistors. With R_sense = 0.1 Ω, a motor rated at 1.0 A needs Vref = 1.0 times 8 times 0.1 = 0.8 V. The Vref is measured between the potentiometer's wiper and the GND pin, with the motor disconnected and the driver powered.

The worked example is a motor rated at 1.2 A per phase. The target current is 1.2 A, so Vref = 1.2 times 0.8 = 0.96 V. Setting the reference to 0.96 V gives the driver's current limit at 1.2 A. Running the motor above its rated current overheats it, and running it below the rated current reduces its torque, so the setting should match the motor's rating, or be reduced a little for cooling.

The common mistake is to set the current limit by turning the potentiometer until the motor feels strong. A motor set too high overheats in minutes, and a driver set too low misses steps under load. Measure Vref with a multimeter and compute the current from the formula. The TMC2209 uses a different formula and can be set over UART, which is easier to repeat.

The A4988 also needs its logic supply to be stable, and its motor supply to be decoupled with a bulk capacitor, typically 100 µF or more, close to the driver. A driver that is powered through a long, thin wire can reset when the motor starts, which looks like random step loss.

In the market, the driver's current accuracy and its thermal design decide whether a stepper runs cool and quiet. Industrial drivers use the same control principle with better sensing and protection.

---

## Microstepping

A full step moves the rotor from one stable position to the next, and the motion between full steps is a jump. Microstepping drives the two phases with sine and cosine currents, so the rotor moves through intermediate positions. A 1/16 microstep gives 16 positions per full step, which reduces the step size and smooths the motion.

The first principle is that the two phase currents are sine and cosine functions of the position. Phase A carries I times cos(theta), and phase B carries I times sin(theta), with theta stepping through the microstep angles. The rotor follows the resultant magnetic field, which points at angle theta, so it moves in small increments between the full-step positions. The increments are not perfectly even, because the rotor's magnetic detent is not a perfect sine, and that imperfection causes some of the resonance and noise that microstepping cannot remove.

The worked example uses the NEMA 17 with 1/16 microstepping. The steps per revolution are 200 times 16, which is 3200. The angle per microstep is 1.8 / 16 = 0.1125 degrees. With the 8 mm lead screw, one microstep moves 8 / 3200 = 0.0025 mm, as Steps and step angle showed.

The common mistake is to treat the microstep count as the accuracy. The rotor's real position between full steps depends on the motor's magnetic detent and the current accuracy of the driver. Real microstepping accuracy is often a few percent of a full step, which means that a 1/16 microstep does not give 1/16 of a full step's accuracy. A stepper that moves 0.0025 mm per microstep may still be accurate only to within 0.01 to 0.02 mm over a full move, and the error depends on the mechanism.

Microstepping also reduces torque at high microstep counts. At a 1/16 step the holding torque is about the same as at the full step, but the torque at a given microstep angle depends on the phase currents, and the peak torque at intermediate positions is lower. Microstepping does not make a stepper stronger, and it cannot replace a larger motor.

On the job, microstepping is used to hide the resonance of a stepper and to make slow motion smooth. Printers and CNC machines use it, and the accuracy requirement sets the microstep setting, not the other way round.

---

## Torque and speed

A stepper's torque falls as its speed rises, and this is the most important limit to know before you pick a step rate. A stepper that holds 0.5 N·m at standstill may give only 0.1 N·m at 500 steps per second, and it may stop following the pulses altogether at a higher speed.

The first principle comes from the coil's electrical behavior. The coil has inductance L and resistance R, and at high step rates the current cannot rise to its set level in the time available. The back-EMF generated by the rotor also grows with speed, and it reduces the voltage across the coil. The result is a torque-speed curve that falls steeply above a certain speed, which the manufacturer gives as the pull-out curve.

The worked example uses the L over R time constant. Suppose a coil has L = 3 mH and R = 1.5 Ω. The time constant is L / R = 0.003 / 1.5 = 2 ms. At the step rate, the coil has less time to reach the full current, so the current at high speed is lower than the set level. A driver with a higher supply voltage, such as 24 V instead of 12 V, shortens the current rise and raises the speed at which torque falls, which is why motor supply voltage matters for steppers.

The common mistake is to assume the stepper's holding torque is its available torque. Holding torque is the torque at zero speed, and it is much larger than the running torque. A move that needs 0.3 N·m at 300 steps per second may fail on a stepper whose holding torque is 0.5 N·m, because the running torque is much lower at that speed.

In a robot, the torque-speed curve sets the top speed for a stepper axis. Choose the motor from its curve at the speed you need, add a safety factor, and test the axis with the real load.

In the market, steppers are used where the speed is moderate and the load is steady. Where high speed and variable load are needed, servos are the better choice, and that is the trade-off Day 27 introduced.

---

## Acceleration profiles

A stepper that starts at full speed will miss steps, because the rotor cannot accelerate fast enough to follow the pulses. The solution is a speed ramp: start slowly, accelerate to the target speed, run at that speed, then decelerate to a stop. The most common ramp is the trapezoidal profile.

The first principle is kinematics. The speed rises at a constant acceleration a, so v equals a times t, and the distance covered in the ramp is s = 0.5 times a times t². A move of N steps has a trapezoid in its speed-time plot, where the area under the curve equals the number of steps. The acceleration must stay below the level at which the rotor can follow the pulses under the load.

The worked example uses a move of 2000 steps, a maximum speed of 800 steps per second and an acceleration of 1000 steps per second squared. The time to reach full speed is 800 / 1000 = 0.8 seconds. The distance in the ramp is 0.5 times 1000 times 0.8² = 320 steps. The ramp uses 320 steps up and 320 steps down, so the constant-speed part is 2000 minus 640, which is 1360 steps, at 800 steps per second, taking 1.7 seconds. The total move takes 0.8 + 1.7 + 0.8 = 3.3 seconds. The AccelStepper library computes these numbers for you, but you should check them by hand once.

The common mistake is to set the acceleration to the maximum the datasheet allows. The datasheet's acceleration is for an unloaded motor, and a loaded axis needs much less. Find the maximum acceleration by testing: increase the acceleration in steps until the motor misses steps, and then use a value 20 to 30 percent below that.

Acceleration also affects heat and vibration. A fast ramp draws a large current, and a stepper that accelerates hard and often runs hotter. A slower ramp gives a longer move and less stress on the mechanism.

On the job, acceleration profiles are part of motion planning. A 3D printer's firmware computes a ramp for each move, and a robot's joint planner uses the same idea with the joint's torque limit.

---

## Open-loop position and missed steps

An open-loop system sends commands and assumes they are carried out. A stepper is open-loop by default, so its controller counts the pulses it sent, not the rotations the rotor made. The count is accurate only if no step is missed.

The first principle is that a missed step is an error that persists. If the rotor misses one step of 1.8 degrees, the controller's position is off by that step for the rest of the move, and every later move starts from the wrong place. A stepper that misses steps repeatedly drifts over time, and the drift accumulates, which is why open-loop steppers need a reference to correct the count.

The worked example uses the stage from the project. A 100 mm move with 8 mm lead and 1/16 microstepping is 40,000 microsteps. A single missed full step, equal to 16 microsteps, moves the carriage 0.04 mm short. Ten missed full steps in a long run of moves give a 0.4 mm error, which is significant for a precision axis. The step-loss test measures how often this happens under a given load and speed.

The common mistake is to treat a silent stepper as a correct stepper. The motor does not make a sound when it misses steps, and the carriage may look like it moved correctly. The only reliable check is to compare the count with a sensor, such as a limit switch at the end of travel, or an encoder on the shaft. A stepper that reaches the limit switch at the wrong count has lost steps.

Missed steps have three common causes: a current too low for the load, an acceleration too high for the rotor's inertia, and a speed above the pull-out curve. Each cause has a different fix, and the step-loss test can tell them apart by varying one thing at a time.

In the market, open-loop steppers are used where missed steps are either impossible, because the load is light, or detectable and correctable, because a sensor checks the count. The 3D printer's endstop switches are an example of the second case.

---

## Homing

Homing moves an axis to a known reference and sets the controller's count to zero at that point. It is the step that gives the open-loop count a meaning in the physical world. Without homing, the controller's zero is wherever the motor was when it was powered on, which is arbitrary.

The first principle is that a reference sensor gives a known position, and the count is then relative to that position. A limit switch is the simplest reference: the axis moves toward it at a slow speed, stops when the switch triggers, backs off a small distance, and sets the count to zero. The slow approach limits the error from the switch's position and the stepper's inertia.

The worked example uses the stage with a limit switch at one end. The homing speed is 200 steps per second, slower than the move speed. The carriage moves toward the switch until it triggers, which the code reads on a digital input. The code then backs off 200 microsteps, which is 0.5 mm at 1/16 microstepping with an 8 mm lead, and it approaches again at 50 steps per second for the final precision. The second approach eliminates most of the variation from the switch's trigger point, which has a hysteresis of about 0.1 mm for a typical mechanical switch.

The common mistake is to home at the move speed. The switch's trigger point depends on the speed, and a fast approach may overshoot the switch, or it may trigger the switch while the carriage is still moving, which changes the zero point by an amount that varies with speed. Use a slow approach for the final contact.

Homing also needs a safe direction. The homing move must go toward the switch that is at the end of travel, so a wrong direction can drive the carriage into its mechanical stop. Test the homing with the carriage far from the stop, and check the direction before you trust it.

In a robot, homing is part of every startup routine. A joint that homes against a sensor starts each session with a known position, and a joint that does not home must be positioned by hand before the robot is allowed to move.

---

## Hands-on exercises

Use a current limit that matches your motor, and check the Vref before you connect the motor. Keep the 12 V supply fused, and never hot-plug the motor while the driver is powered. Stop the test if the motor or the driver gets too hot to hold comfortably.

1. **Count the steps per revolution.** Mount a pointer on the stepper shaft. Drive the stepper with the Arduino at full-step mode and send 200 step pulses with a 2 ms interval. Check that the shaft completes one revolution. If it does not, check the step angle on the motor's label and the driver's MS pins. Record the actual steps per revolution.

2. **Set the current limit.** With the motor disconnected from the driver, power the driver and measure Vref between the potentiometer and GND. Compute the current limit from I_max = Vref / (8 times R_sense), with R_sense as printed on the board. Adjust the potentiometer to the value that gives the motor's rated current, and record the Vref and the computed current.

3. **Microstep test.** Set the MS1, MS2 and MS3 pins to 1/16 microstepping, according to the driver's truth table, and send 3200 steps. Confirm the shaft turns once. Then run 3200 steps at 1/8 microstepping and confirm the shaft turns once with a different number of steps. Record the steps per revolution at each setting.

4. **Find the maximum speed.** Run a constant-speed test at 100, 200, 400 and 800 steps per second, with the load you intend to use. Count the steps the carriage moves against a ruler or a fixed mark for 1000 commanded steps. Record the error at each speed, and identify the speed at which the error becomes significant.

5. **Run a trapezoidal profile.** Use the AccelStepper library, set the maximum speed to 800 steps per second and the acceleration to 1000 steps per second squared. Move 2000 steps forward and back. Measure the move's total time with a stopwatch and compare with the 3.3 seconds computed in the lesson.

6. **Homing test.** Wire a limit switch to a digital input with a pull-up resistor, set up the homing routine from the project, and home the axis five times from different starting positions. Record the position reading after each homing move, and compute the spread.

---

## Project: a homed linear stage with step-loss tests

This project builds a single-axis linear stage with a NEMA 17 stepper, an 8 mm lead screw, a limit switch and a homing routine. It measures how often steps are lost at different speeds and accelerations, which is the question every stepper user must answer before trusting an open-loop axis.

**Goal:** build a homed linear axis that moves a fixed carriage a known distance, and produce a step-loss table for three speeds and two accelerations with the load you plan to use.

**Repo layout:**

```
100-days-robotics-engineering-day-28-stepper-motors/
├── README.md
├── firmware/
│   └── linear_stage/
│       └── linear_stage.ino
├── tests/
│   ├── step_loss_log.csv
│   └── homing_repeatability.csv
├── calculations/
│   ├── current_limit.md
│   └── profile_and_resolution.md
├── docs/
│   └── wiring_and_power.md
├── plots/
│   └── step_loss_by_speed.png
└── images/
    └── stage_setup.jpg
```

**Starter code:** `firmware/linear_stage/linear_stage.ino`

```cpp
#include <AccelStepper.h>

// A4988 wiring: STEP on D3, DIR on D4, EN on D5.
const int STEP_PIN = 3;
const int DIR_PIN  = 4;
const int EN_PIN   = 5;
const int LIMIT_PIN = 2;   // limit switch to GND, read with INPUT_PULLUP

// Microstepping: 16 steps per full step, 200 full steps per revolution.
const long STEPS_PER_REV = 3200;
const float LEAD_MM = 8.0;
const long STEPS_PER_MM = STEPS_PER_REV / LEAD_MM;   // 400 steps per mm

AccelStepper stepper(AccelStepper::DRIVER, STEP_PIN, DIR_PIN);

void setup() {
  Serial.begin(115200);
  pinMode(EN_PIN, OUTPUT);
  digitalWrite(EN_PIN, LOW);               // A4988 enable is active low
  pinMode(LIMIT_PIN, INPUT_PULLUP);
  stepper.setMaxSpeed(400);                // steps per second
  stepper.setAcceleration(800);            // steps per second squared
  Serial.println("ready: send H to home, or M<mm> to move");
}

void home() {
  // Approach the switch slowly in the negative direction.
  stepper.setMaxSpeed(200);
  stepper.setSpeed(-200);
  while (digitalRead(LIMIT_PIN) == HIGH) {
    stepper.runSpeed();
  }
  // Back off 0.5 mm, then approach again at a slower speed.
  stepper.move(STEPS_PER_MM / 2);
  stepper.setMaxSpeed(100);
  stepper.runToPosition();
  stepper.setSpeed(-50);
  while (digitalRead(LIMIT_PIN) == HIGH) {
    stepper.runSpeed();
  }
  stepper.setCurrentPosition(0);           // this point is now zero
  stepper.setMaxSpeed(400);
  Serial.println("homed");
}

void loop() {
  if (!Serial.available()) {
    stepper.run();
    return;
  }
  char cmd = Serial.read();
  if (cmd == 'H') {
    home();
  } else if (cmd == 'M') {
    float mm = Serial.parseFloat();
    stepper.moveTo((long)(mm * STEPS_PER_MM));
    while (stepper.distanceToGo() != 0) stepper.run();
    Serial.print("at position steps=");
    Serial.println(stepper.currentPosition());
  }
}
```

The starter code homes the stage against the limit switch and moves it to a commanded position in millimetres. The `home()` function uses a fast approach, a 0.5 mm backoff and a slow second approach, which the lesson described. The code uses the STEPS_PER_MM constant, so the count matches the physical distance only if the lead, the microstep setting and the step angle are correct. Check each of these before you run the stage.

Note that `stepper.runToPosition()` blocks until the move is done, and the code relies on that during the backoff. The `loop()` function calls `run()` only when no command is waiting, so a long move is driven by the `while` loop in the `M` command. This is simple, and it is enough for step-loss tests where each move runs to completion before the next command.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the motor's label values, the driver model and its R_sense, the lead screw and the wiring.
2. Calculate the current limit and the Vref, and record them in `calculations/current_limit.md`, with the measured Vref.
3. Home the stage five times from different positions, and record the position after each home in `tests/homing_repeatability.csv`. Compute the spread.
4. Run the step-loss test. For each of three speeds, 100, 200 and 400 steps per second, and two accelerations, 500 and 800 steps per second squared, move the carriage 100 mm forward and back five times. Measure the position at the end of each move from the limit switch or a fixed mark, and record the error in `tests/step_loss_log.csv`.
5. Plot the step-loss error against speed in `plots/step_loss_by_speed.png`, with one line for each acceleration.
6. Write `calculations/profile_and_resolution.md` with the trapezoidal profile for your fastest test, the resolution in millimetres per microstep, and the speed at which the error became significant.

**README checklist:**

- [ ] Goal, the motor's rated current and the driver model with its R_sense
- [ ] Wiring diagram and the power supply with its fuse
- [ ] Current limit calculation and the measured Vref
- [ ] Homing repeatability spread
- [ ] Step-loss table for three speeds and two accelerations
- [ ] Plot of the error against speed
- [ ] One limit of the stage, stated with a number from the test

**Done when:**

- The stage homes from any start position within the travel, and the homing spread across five trials is under 0.1 mm.
- The step-loss table is complete, and the README states the highest speed and acceleration at which the error stays below 0.2 mm over 100 mm.
- The current limit is set from the measured Vref and the formula, and the README shows the calculation.
- The repository contains the firmware with the values you used, not the placeholder values, and the README explains any change from the starter code.

**Evidence to commit to GitHub:** the firmware, the homing and step-loss CSV files, the plot, the calculation files, the wiring diagram, the photo and the README. A reviewer should be able to see the speed at which steps began to be lost and the number the README uses for the safe limit.

This project moves a carriage that can pinch fingers and that can drive into a stop. Keep your hands clear during homing, and set the speed to the slow value for the first run.

---

## Check your understanding

1. A stepper with a 1.8-degree step drives an 8 mm lead screw at 1/8 microstepping. Calculate the steps per millimetre and the linear distance of one microstep. Explain why the microstep distance is not the same as the accuracy of the axis.
2. A driver's sense resistor is 0.1 Ω and the motor is rated at 1.5 A per phase. Calculate the reference voltage to set. Explain what happens if the reference is set to half of that value and the motor is run at its rated load.
3. A stepper's holding torque is 0.6 N·m, but a move at 500 steps per second with the same load misses steps. Explain why the holding torque does not predict the running torque, and name one change to the driver supply or the move profile that would help.
4. An axis moves 1000 steps forward and 1000 steps back, and the carriage returns 0.2 mm short of the starting mark. Calculate the number of full steps lost if the lead is 8 mm and the axis uses 1/16 microstepping, and explain why the controller does not show the error.
5. A trapezoidal move has a maximum speed of 600 steps per second and an acceleration of 1200 steps per second squared. Calculate the time and the steps used in the ramp up, and explain why a higher acceleration might cause missed steps even if the maximum speed is safe.
6. Explain why an open-loop stepper with a limit switch can be accurate over a long run of moves, but still accumulate error between homing events. Name the measurement that would detect the accumulation.

---

### Answers

1. One revolution is 200 full steps and moves the screw 8 mm, so the step count is 200 / 8 = 25 full steps per millimetre. At 1/8 microstepping that is 200 microsteps per millimetre. One microstep moves 1 / 200 = 0.005 mm. The microstep distance is the resolution of the commanded position. The accuracy depends on the motor's detent, the driver's current accuracy and the mechanism, so a full-scale move may be accurate only to a few hundredths of a millimetre.
2. The reference voltage is 1.5 times 8 times 0.1 = 1.2 V. If it is set to 0.6 V, the current limit is 0.75 A. The motor then produces about half its rated torque, which may not hold the load, and it may miss steps. The motor itself is not overheated, but the load is above its available torque, so the axis fails in a different way from an overheated motor.
3. Holding torque is measured at zero speed, and the running torque falls as speed rises because of the coil's inductance and the back-EMF. A higher supply voltage or a lower acceleration can help. A higher supply voltage shortens the current rise, and a gentler ramp reduces the speed at which the rotor must follow the pulses.
4. The 0.2 mm error is 0.2 divided by 0.0025, which is 80 microsteps, or 5 full steps at 16 microsteps per full step. At 1/16 microstepping, 0.0025 mm is one microstep, so 80 microsteps is 5 full steps. The controller shows no error because it counts only the pulses it sent, and the carriage's real position is not measured.
5. The time to reach 600 steps per second at 1200 steps per second squared is 0.5 seconds. The ramp covers 0.5 times 1200 times 0.5 squared, which is 150 steps. The ramp uses 150 steps up and 150 down. A higher acceleration may cause missed steps because the rotor's inertia and the load may not follow the pulses fast enough, even at a maximum speed that is safe on its own.
6. Open-loop steppers accumulate error when a step is missed, and the count is not corrected until the next reference. A limit switch or a shaft encoder checks the position at the reference. The measurement that detects accumulation is the difference between the controller's count and the reference position at the end of a series of moves, which is the step-loss error the project records.

---

## Sources

*Read*

- [Wikipedia: Stepper motor](https://en.wikipedia.org/wiki/Stepper_motor) for the types, the step angle and the drive principles.
- [Wikipedia: NEMA Standards](https://en.wikipedia.org/wiki/NEMA_Standards) for the frame size of NEMA 17 motors. The article covers several standards, so read the stepper section.
- [Wikipedia: Lead screw](https://en.wikipedia.org/wiki/Lead_screw) for the relation between lead and linear travel.
- The Allegro A4988 datasheet. Search "A4988 datasheet Allegro" for the current limit formula, the microstep table and the logic thresholds.
- The Trinamic (now part of Analog Devices) TMC2209 datasheet. Search "TMC2209 datasheet" for its UART configuration and its current setting.

*Tutorials*

- SparkFun Learn, the tutorials on stepper motors and stepper drivers. Search "SparkFun Learn stepper motor driver".
- Adafruit Learning System, the guides on stepper motors and the A4988 driver. Search "Adafruit stepper motor guide".
- AccelStepper library documentation by Mike McCauley, installed with the library. Search "AccelStepper Arduino library" to find the project page.

*Watch*

- MIT OpenCourseWare, the courses on electromechanical systems and motors, which cover the torque-speed behavior of steppers. Search the course on ocw.mit.edu.
- Official Arduino channel, the videos on stepper motors and the Arduino AccelStepper library. Search "Arduino stepper motor" on YouTube and check the channel name.

*Practice*

- Wokwi simulator at wokwi.com, which includes a stepper motor and an Arduino Uno. Build the step-count test and watch the shaft count against the step pulses.

---

## Tomorrow

Tomorrow you will learn how to measure motion directly. Day 29 covers encoders: incremental and absolute types, the quadrature signal that gives direction, the counts per revolution, and how to convert counts into RPM, position and the wheel's odometry. The step-loss test from this day is the reason encoders matter. An encoder on the stage would have shown you the missed steps as they happened.

**Next:** [Day 29: Encoders](Robotics-Day-29-Encoders.md)
