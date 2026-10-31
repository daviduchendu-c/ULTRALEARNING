# Day 27: Servo Motors

*100 Days of Robotics Engineering, Module 4: Sensors and Actuators*

**Time:** about 3.5 hours
**Level:** beginner to intermediate. You need Day 26 (the torque-speed curve and gearing) and basic Arduino from Module 3. Today you move from a motor that spins to a joint that holds an angle.

**By the end of this day, you will have:**

- A standard hobby servo that sweeps between two calibrated end stops on an Arduino Uno
- A measured pulse-width-to-angle table for one servo, with its real minimum, center and maximum
- A torque check for a 10 cm lever carrying a 0.5 kg load, with a safety factor
- A pan-tilt head that accepts angle commands over a serial port and logs its own commanded and reported angles

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3 or newer, about $25 for an official board or $10 to $15 for a clone. The Servo library ships with the Arduino AVR core, which the lesson uses at version 1.8.6 or newer.
- Two hobby servos. An SG90 micro servo costs about $2 to $3 each. An MG996R metal-gear servo costs about $6 to $10 each, and its listed torque and current depend on the seller's clone, so read the datasheet of the actual unit.
- A 5 V regulated supply rated for at least 2 A for the MG996R tests. A USB port does not supply enough current for a metal-gear servo at stall.
- A 1000 µF electrolytic capacitor rated at 10 V or more, about $0.50, placed across the servo supply to absorb the start-up surge.
- Jumper wires, a breadboard and a few hundred grams of weight for the torque test, such as a stack of coins or a small bottle.
- A digital multimeter, about $15 to $25, to check the supply voltage under load.
- Optional: a Feetech STS3215 or a ROBOTIS Dynamixel XL330 for the bus-servo section. These cost roughly $20 to $30 and $25 to $35 respectively at the time of writing, so check current prices before you buy.

**Prerequisites:**

- Day 26: torque in newton-metres, gearbox ratio and the idea that stall torque is the limit of a joint
- Day 13: the power budget, the bulk capacitor and the common ground
- Module 3: a sketch that reads serial input and drives a PWM pin
- Basic trigonometry: converting a lever arm and a mass into a torque

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How hobby servos developed](#how-hobby-servos-developed)
- [Glossary](#glossary)
- [What a servo is](#what-a-servo-is)
- [The command signal](#the-command-signal)
- [Feedback and the control loop](#feedback-and-the-control-loop)
- [Torque and the lever arm](#torque-and-the-lever-arm)
- [Speed and response](#speed-and-response)
- [Calibration](#calibration)
- [Power and surge current](#power-and-surge-current)
- [Bus servos](#bus-servos)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a calibrated pan-tilt head](#project-a-calibrated-pan-tilt-head)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot joint needs two things at once: enough torque to hold a load, and a known angle. A DC motor gives the first and needs a sensor for the second. A servo packages the motor, a gearbox, a position sensor and a small control circuit into one part. You send it an angle, and it moves there and holds it, as long as the load stays within its torque rating.

Servos are the easiest actuator to get working, which is why they appear in almost every first robot arm, pan-tilt head and hexapod. They are also easy to misuse. The most common failure is a servo that overheats because it is asked for a position it cannot reach, or a robot that resets because its servos draw a surge current from a shared 5 V rail. Today you learn the signal, the feedback loop, the torque check and the calibration, so that a servo does what you command and does not fail when it is loaded.

---

## How hobby servos developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1960s to 1970s | Radio-control model aircraft and boats use small motors with gearboxes and potentiometers to move control surfaces. | The servo's basic design, a motor with feedback, comes from model radio control. |
| 1970s to 1980s | Standard pulse-width signalling becomes the convention for radio receivers, with a 20 ms frame and a pulse of about 1 to 2 ms. | The same 50 Hz signal still drives most hobby servos. |
| 1990s to 2000s | Low-cost servos reach hobbyists and educators, and microcontroller libraries make them easy to drive. | The Arduino Servo library made servos a standard first project for students. |
| 2010s | Digital servos with faster control loops and metal gears spread, and bus servos with serial networks appear for multi-joint arms. | Multi-joint arms use bus servos so that many joints share one cable. |
| Today | Smart servos with built-in temperature and load reporting are common in research arms, and cheap bus servos power many open-source manipulators. | Reading the servo's own load and temperature is now a standard part of a manipulator's safety check. |

The servo's control signal is an old convention, and its mechanics are unchanged. What has changed is the feedback: an early servo's potentiometer is now a digital encoder in many models, and the control loop runs faster.

---

## Glossary

- **Servo:** a motor, a gearbox, a position sensor and a control circuit in one housing, which moves to and holds a commanded angle.
- **Hobby servo:** a small servo that takes a PWM pulse and moves between about 0 and 180 degrees.
- **Pulse width:** the length of the high part of a PWM pulse, measured in microseconds.
- **Frame period:** the time between successive command pulses, about 20 ms (50 Hz) for a standard hobby servo.
- **Potentiometer:** a three-terminal variable resistor. In a servo, the wiper voltage gives the shaft angle.
- **Dead band:** a small range of commands around the target in which the servo does not move, because the error is too small to overcome friction.
- **Stall torque:** the torque a servo can hold at zero speed, usually given in kg·cm or N·m.
- **kg·cm:** kilogram-centimetre, a torque unit common in hobby servo listings. One kg·cm equals 0.0981 N·m.
- **Lever arm:** the distance from the joint axis to the point where a load acts.
- **Bus servo:** a servo that receives commands and returns status over a shared serial bus, such as TTL or RS-485, using a unique ID for each servo.
- **Surge current:** the brief high current a servo draws when it starts moving or when its load changes suddenly.

---

## What a servo is

A servo looks like a small plastic box with three wires and an output horn. Inside is a DC motor, a set of gears, a potentiometer or encoder connected to the output shaft, and a control board that compares the target angle with the measured angle and drives the motor to remove the difference.

Think of a driver steering a car by turning the wheel until the car points where the map says. The driver is the control board, the map is the command, and the steering wheel is the motor. The driver also feels how far the car has turned and corrects as it goes, which is the feedback.

The first principle is the closed loop. The controller computes the error e = target minus measured angle. It sets the motor voltage proportional to that error, so a large error gives a large voltage, and the motor turns the output toward the target. When the error reaches zero, the voltage goes to zero, and the gears hold the angle against friction and small loads. A servo that is more than a few degrees away from its target is either still moving or fighting a load it cannot overcome.

The worked example is the standard 0 to 180 degree servo. Its potentiometer gives 0 V at one end of travel and the supply voltage at the other. The control board reads that voltage with an ADC and compares it with the target, which the PWM pulse specifies. A pulse of 1.5 ms is the center of travel, so the target is 90 degrees, and the error is the difference between the target and the measured angle.

The common mistake is to assume the servo knows its absolute angle at power-up. Most hobby servos know their angle relative to the potentiometer's range, and that range is set by the mechanical stops. A servo that has been forced past its stop may report an angle that differs from the real one. Calibration, covered below, fixes most of this.

In the market, hobby servos are cheap actuators for prototypes, and industrial servos with encoders and motor drives are used in factory automation. The principle is the same, and the difference is in the precision and the cost of the feedback.

---

## The command signal

The standard hobby servo receives a PWM signal with a period of 20 ms and a pulse width between about 1.0 and 2.0 ms. The pulse width sets the target angle, and the servo's control board reads the width of each pulse and moves toward it. A signal that is not repeated every 20 ms may be ignored, and the servo may hold its last position or go limp, depending on the model.

The first principle is that the pulse width is a number that the servo maps to an angle. The mapping is linear for most servos, and the endpoints are set by the manufacturer. The usual approximation is angle = (pulse_us minus 1000) times 180 divided by 1000, which gives 0 degrees at 1000 microseconds, 90 degrees at 1500 and 180 degrees at 2000. Many servos, especially the metal-gear models, respond to a wider range, such as 500 to 2500 microseconds, and their angular range goes beyond 180 degrees.

The worked example uses the duty cycle of the 20 ms frame. A 1.5 ms pulse is 1.5 divided by 20, which is 7.5 percent duty. An Arduino Uno's timers can produce a 50 Hz signal with the Servo library, which handles the timing for you. The library's writeMicroseconds() function sends the pulse width directly, which is more precise than write(), which takes an angle and maps it with the library's default endpoints.

The common mistake is to use analogWrite() on a servo pin. The default PWM frequency on the Uno is about 490 Hz on most pins, which is far too fast for a hobby servo. The servo then receives pulses that are repeated every 2 ms, and it reacts to some of them and ignores others. Use the Servo library, or a timer set to 50 Hz.

The command signal has no feedback. The servo does not tell the microcontroller that it reached the target, and the microcontroller has to know the servo's state from the commands it sent. A joint that is blocked by an obstacle does not report the blockage through the PWM line. Some bus servos report status, which the bus servo section covers.

In industry, the same pulse-width idea appears in R/C receivers and in many simple actuators, which accept a pulse train as an interface. The pulse train is simple, reliable and easy to test with a logic analyzer.

---

## Feedback and the control loop

Feedback is what makes a servo more than a motor with a gearbox. The potentiometer or encoder reports the output angle, and the control loop uses that report to drive the motor to the target. Without feedback, a motor's position depends on how long it was powered and how much it was loaded.

The first principle is the proportional controller. The motor voltage is V = Kp times e, where e is the angle error in degrees and Kp is a gain in volts per degree. A higher gain makes the servo respond faster but also makes it overshoot and oscillate. A lower gain is slow and may not reach the target under load. Most hobby servos use a proportional-derivative controller with fixed gains set by the manufacturer, and a digital servo uses a faster loop and higher gains.

The worked example is the step response. Suppose the target jumps from 0 to 90 degrees. The error is 90 degrees, so the initial voltage is Kp times 90. The motor accelerates and the error falls. At the target, the error is zero and the voltage drops to zero, but the motor's inertia carries it past the target, which gives overshoot. The derivative term slows the approach and reduces the overshoot. A servo with a 2 percent overshoot on a 90-degree move is well tuned for a light load. A servo with 15 percent overshoot is either underdamped or heavily loaded.

The common mistake is to treat the servo's dead band as a sign of a fault. The dead band is the range of errors that the controller ignores, which prevents the motor from buzzing when it is near the target. A servo with a 1 to 2 degree dead band will hold a position within that band, and it will not reach a target that is closer than the band to its current position. This matters for a robot joint that needs fine control. The fix is a closed loop with a sensor that has better resolution than the dead band, or a servo with a smaller dead band.

Feedback also has limits. A potentiometer wears and gets noisy. A cheap potentiometer may give a reading that jitters by a degree or more, which the servo's control loop turns into a buzzing motor. A digital encoder inside a servo is more precise but has its own resolution, and the output angle is the encoder count divided by the counts per revolution after the gearbox. Day 29 covers encoders in detail.

In the market, servo feedback decides the repeatability of a joint. A hobby servo may repeat within a few degrees, while an industrial servo repeats within a fraction of a degree, and the price follows.

---

## Torque and the lever arm

A servo is rated by its stall torque, which is the torque it can hold at zero speed. A servo holding an arm against gravity is always at or near stall, so the holding torque must exceed the load torque with some margin. The load torque depends on the mass and where the mass sits relative to the joint.

The first principle is torque equals force times lever arm, T = F times d. For a mass m held at a distance d from the joint axis, the force is m times g, where g is 9.81 m/s². So the load torque is T = m times g times d. The torque is largest when the arm is horizontal and the mass sits at the end, and it falls to zero when the arm points straight down past the joint.

The worked example is a 0.5 kg load at a 10 cm lever arm, which is the case from the lesson's torque check. The load torque is 0.5 times 9.81 times 0.10, which is 0.49 N·m. In kg·cm, that is 0.49 divided by 0.0981, about 5.0 kg·cm. A servo rated at 4.8 kg·cm would be at its limit, with no margin. The standard design rule is a safety factor of 1.5 to 2 on the stall torque, which means a servo rated at 8 to 10 kg·cm should be used for this load. An MG996R at 6 V, rated at around 10 to 11 kg·cm in many listings, would meet the target with margin, if the datasheet of the actual unit confirms the number.

The common mistake is to use the servo's rated torque without a safety factor, and to forget the arm's own weight. A 3D-printed arm of 100 grams adds torque at the joint, and so does the bracket and the cable. Add every part's mass times its own lever distance to the total, and compute the torque at the worst position, which is usually horizontal.

Torque is also limited by the servo's speed. A servo's rated torque is at stall. As the speed rises, the available torque falls along the servo's torque-speed curve, which Day 26 introduced. A servo that holds a 5 kg·cm load at rest may stall at a moderate speed if the arm is moving quickly against gravity. The check is the stall torque and the speed together.

In a robot, torque sizing is done for each joint separately, and the joint nearest the base carries the most load. The shoulder of a six-joint arm sees the weight of the whole arm and the payload, so it needs the largest servo.

---

## Speed and response

A servo's speed is given as the time to move 60 degrees with no load at a stated voltage, such as 0.1 seconds at 4.8 V. That number is a no-load figure, and it falls under load. The response time also includes the control loop's settling, which the overshoot and the damping affect.

The first principle is that the speed is set by the motor's no-load speed and the gearbox ratio, as Day 26 described. A servo at 0.1 seconds per 60 degrees turns at 600 degrees per second, which is 10 rad/s at the output. The motor itself runs much faster, and the gearbox reduces the speed and raises the torque.

The worked example is a move of 90 degrees at the servo's rated speed. At 600 degrees per second, the move takes 0.15 seconds without the settling time. With settling and a small overshoot, the total time to settle within 1 degree of the target might be 0.25 seconds. A robot that needs a 90-degree move in 0.1 seconds must choose a faster servo, or it must accept a motion that does not reach the target in time.

The common mistake is to compare servo speeds at different voltages. The same servo may be rated at 0.12 seconds at 4.8 V and 0.10 seconds at 6.0 V, and a listing that gives only the faster figure hides the voltage. Use the figure at the voltage you run, and check the servo's voltage rating before raising the supply.

Response time also depends on the command rate. A servo that receives commands every 20 ms responds to each one, but a command sent every 100 ms causes a stepped motion. For smooth motion, interpolate the commanded angle at the frame rate and send a new pulse every 20 ms.

In the market, speed is often the least important servo spec for a robot joint, and torque and accuracy matter more. The exception is a fast pick-and-place arm, where the speed sets the cycle time.

---

## Calibration

Every servo has its own minimum pulse, center pulse and maximum pulse, and the defaults in a library rarely match the servo you have. Calibration measures those three numbers for one servo and stores them, so that a command of 90 degrees really points the horn at the middle of the joint's range.

The first principle is that calibration is a mapping from the joint's real angle to the pulse width. You measure two or three points, usually the end stops and the center, and fit a linear map. The servo's real range may be narrower than its nominal range, because a horn or a bracket limits the motion, and the calibration must reflect the joint's real limits, not the servo's maximum.

The worked example uses a protractor and a bracket. Set the servo to 1000 microseconds, mark the horn's angle on the bracket, then set 2000 microseconds and mark again. If the two marks are 170 degrees apart, the servo's real range at those pulses is 170 degrees, not 180. A linear fit gives the pulse width for any angle inside the range, using the two measured endpoints. For 85 degrees from the minimum mark, the pulse is 1000 plus 85 divided by 170 times 1000, which is 1500 microseconds, not 1472 as the default 180-degree map would give.

The common mistake is to calibrate with the horn mounted at a random angle and then to remount it. Each change of horn position shifts the mapping, so the calibration must be done with the horn in its final position. Mark the horn and the output shaft before you remove it. Another mistake is to drive a servo past its stops to get a larger range. The servo's gears and potentiometer can be damaged by that, and the position reported by the potentiometer becomes meaningless.

Calibration also needs a repeatability check. Command the same angle five times from the same starting point and measure the spread. A servo that repeats within 1 degree is good for many joints, and a spread of 3 to 5 degrees means the joint needs a finer sensor or a different servo.

In a factory, calibration is a documented step with a record of each joint's limits. Robot builders often skip it, and the robot then fails when it is moved to a new bench or a new mount.

---

## Power and surge current

A hobby servo draws a small current when it holds still and a large current when it starts moving or when it stalls. The surge current can be several times the holding current, and for a metal-gear servo at stall it can reach 1 to 2 A at 6 V. A supply that cannot deliver the surge will sag, and the microcontroller may reset.

The first principle is that the supply's output impedance and the wire resistance set the voltage drop during the surge. A supply with an internal resistance of 0.5 Ω delivering a 2 A surge drops 1 V before the servo sees it. A 5 V supply that drops to 4 V during a surge may still run the servo, but it may reset the microcontroller if the two share a rail.

The worked example uses the surge. An MG996R stalling at 6 V draws about 2 A. With a 0.2 Ω supply and wire resistance combined, the drop is 0.4 V. A 1000 µF capacitor across the servo supply supplies part of the surge for a short time, so the voltage stays higher for the first milliseconds. The capacitor's energy is one half times C times V squared, which is 0.5 times 0.001 times 36, or 0.018 J at 6 V. That is enough for a brief surge, but not for a stall of several seconds.

The common mistake is to power servos from the Arduino's 5 V pin. The Uno's regulator cannot supply the surge of more than one or two servos, and it may overheat or reset the board. Use a separate supply for the servos, and connect its ground to the Arduino's ground so the signal has a common reference. Do not connect the servo's power wire to the Arduino's 5 V pin.

Power sharing is also a design choice. Many servos on one supply need a supply sized for the sum of the stall currents, not the sum of the running currents. A six-servo arm with 2 A stall per servo needs a supply that can deliver 12 A for a brief time, or the design must limit the motion so that not all servos stall at once.

In the market, power distribution is part of the robot's electrical design, and it is a common cause of field failures. A reviewer who sees servos on the Arduino 5 V pin will ask for a separate supply before the robot moves.

---

## Bus servos

A bus servo replaces the PWM line with a serial bus, usually TTL or RS-485. Each servo has a unique ID, and the controller sends a command to one ID at a time or to all of them. The servo returns its position, temperature, voltage and load, which a PWM servo cannot report.

The first principle is addressing. The controller sends a packet with the target servo's ID, a command and a checksum. Each servo listens for its own ID and ignores the rest. The bus carries many servos on one pair of wires, which reduces the wiring of a multi-joint arm to a single cable.

The worked example is a three-servo bus. The controller sends a position command to ID 1, and the servo replies with its position in the same packet format. A status request to ID 2 returns its temperature, which the controller can compare with a limit. A servo that reports a temperature above its rated value can be stopped before it fails. This is the feature that makes bus servos attractive for research arms, and it is the reason they cost more.

The common mistake is to mix bus servos with different protocols on one bus. The Feetech and Dynamixel servos use different packet formats, and their ID ranges and baud rates differ. Check the protocol and the default baud rate for each model, and set the IDs one at a time before connecting them to a shared bus.

Bus servos also have a limit on bus length and load. A long cable adds reflections and voltage drop, and a bus with many servos can overload the controller's transmitter. Keep the bus short, use the recommended connectors, and terminate it as the manufacturer describes.

In the market, bus servos are the usual choice for research arms and small humanoid robots, and their cost is higher than a PWM servo of the same torque. Their feedback is what makes them useful in safety-critical work.

---

## Hands-on exercises

Keep the servo's supply separate from the Arduino's 5 V pin, and keep the common ground connected. Never hold a servo at stall for more than two seconds during these tests, because a stalled servo heats quickly.

1. **Sweep a servo with the Servo library.** Wire an SG90 to the Uno with its signal wire on pin 9, its power on the 5 V supply and its ground to the common ground. Upload a sketch that attaches the servo and sweeps from 0 to 180 degrees with a 20 ms delay per degree. Watch the motion and note whether the servo buzzes at the ends.

2. **Measure the endpoints.** Replace the sweep with a sketch that sends writeMicroseconds(1000), waits two seconds, then sends 1500 and then 2000, holding each for two seconds. Mark the horn's angle on a bracket at each pulse. Record the real angle at each pulse and compare it with the 0 to 180 default.

3. **Calibrate and save the map.** Using the two endpoint marks and the center mark, write a calibration table in a header file with the minimum, center and maximum pulse widths. Replace the write() calls with a function that maps an angle to a pulse using the table, and test it with five angles across the range.

4. **Repeatability test.** Command the servo to 45 degrees from 0 degrees five times, pausing for two seconds between each move. Measure the angle at the horn each time with a protractor or a phone angle app. Compute the spread, the largest minus the smallest reading.

5. **Torque check.** Mount a 10 cm lever arm on the MG996R's horn. Hang 0.5 kg from the end of the arm with the arm horizontal, and command the servo to hold that position. Watch the servo for two seconds only, then remove the load. Record whether it held, and compute the load torque in kg·cm. Compare with the rated stall torque.

6. **Surge check.** With the MG996R on its own 5 V supply and the 1000 µF capacitor in place, measure the supply voltage at the servo with the multimeter while the servo starts a 90-degree move. Repeat without the capacitor. Record both minimum voltages and explain the difference in one sentence.

---

## Project: a calibrated pan-tilt head

This project builds a two-axis pan-tilt head with calibrated servos. It accepts angle commands over the serial port, moves to the commanded angle within the calibrated range, and logs the commanded and measured angles. The head is a useful test rig for later sensor work, because the camera and the IMU in Days 31 and 32 need a stable mount that can be moved to known angles.

**Goal:** build a two-axis pan-tilt head with calibration data for both servos, a serial command interface and a logged test run that shows the repeatability of each axis.

**Repo layout:**

```
100-days-robotics-engineering-day-27-servo-motors/
├── README.md
├── firmware/
│   └── pan_tilt/
│       ├── pan_tilt.ino
│       └── calibration.h
├── tests/
│   ├── repeatability_log.csv
│   └── surge_measurements.md
├── calculations/
│   └── torque_check.md
├── docs/
│   └── wiring_and_power.md
└── images/
    └── pan_tilt_assembly.jpg
```

**Starter code:** `firmware/pan_tilt/calibration.h`

```cpp
// Calibration for one servo. Fill these in from your own measurements.
// minUs, centerUs and maxUs are pulse widths in microseconds.
// minDeg and maxDeg are the real mechanical angles at minUs and maxUs.
struct ServoCal {
  int minUs;
  int centerUs;
  int maxUs;
  float minDeg;
  float maxDeg;
};

// Example values only. Replace them with your measurements.
const ServoCal PAN_CAL  = {1000, 1500, 2000, 0.0, 170.0};
const ServoCal TILT_CAL = {1050, 1480, 1950, 0.0, 160.0};

// Convert a target angle to a pulse width, clamped to the calibrated range.
int angleToPulse(const ServoCal &cal, float angle) {
  if (angle < cal.minDeg) angle = cal.minDeg;
  if (angle > cal.maxDeg) angle = cal.maxDeg;
  float frac = (angle - cal.minDeg) / (cal.maxDeg - cal.minDeg);
  return cal.minUs + (int)(frac * (cal.maxUs - cal.minUs) + 0.5f);
}
```

**Starter code:** `firmware/pan_tilt/pan_tilt.ino`

```cpp
#include <Servo.h>
#include "calibration.h"

const int PAN_PIN  = 9;
const int TILT_PIN = 10;

Servo pan;
Servo tilt;

void setup() {
  Serial.begin(115200);
  pan.attach(PAN_PIN, 500, 2500);     // attach with wide limits, then clamp in software
  tilt.attach(TILT_PIN, 500, 2500);
  pan.writeMicroseconds(PAN_CAL.centerUs);
  tilt.writeMicroseconds(TILT_CAL.centerUs);
  Serial.println("ready: send 'P<angle>' or 'T<angle>', e.g. P45");
}

void loop() {
  if (!Serial.available()) return;
  char axis = Serial.read();
  float angle = Serial.parseFloat();
  if (axis == 'P') {
    int us = angleToPulse(PAN_CAL, angle);
    pan.writeMicroseconds(us);
    Serial.print("pan cmd=");
    Serial.print(angle);
    Serial.print(" us=");
    Serial.println(us);
  } else if (axis == 'T') {
    int us = angleToPulse(TILT_CAL, angle);
    tilt.writeMicroseconds(us);
    Serial.print("tilt cmd=");
    Serial.print(angle);
    Serial.print(" us=");
    Serial.println(us);
  }
}
```

The starter code accepts a serial command such as P45 to pan to 45 degrees, clamps the angle to the calibrated range and reports the pulse it sent. The clamp matters: without it, a command outside the range sends a pulse that drives the horn into its stop.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the servo models and their datasheet torque, the wiring diagram and the power arrangement.
2. Calibrate both servos using the method from the exercises. Fill in `calibration.h` with your measured values and record the marks in `docs/wiring_and_power.md`.
3. Run a repeatability test. Send the same angle, for example P45, five times from the same start, and record the commanded and measured angles in `tests/repeatability_log.csv`. Repeat for the tilt axis.
4. Compute the torque check for the tilt axis with a camera-sized mass at the lever distance you choose, and write the result in `calculations/torque_check.md` with a safety factor of at least 1.5.
5. Record the surge measurement in `tests/surge_measurements.md`, with the minimum supply voltage with and without the capacitor.
6. Write the README with the limits of the head, such as the repeatability spread, the dead band and the maximum load, and attach the photo.

**README checklist:**

- [ ] Goal, both servo models and their datasheet torque and voltage
- [ ] Wiring diagram and a power arrangement that separates the servos from the Arduino 5 V pin
- [ ] Calibration table for each axis, with the marks and the date
- [ ] Repeatability log and the spread for each axis
- [ ] Torque check with the safety factor stated
- [ ] Surge measurement with and without the capacitor
- [ ] One limit of the head, stated with a number from a test

**Done when:**

- The head responds to P and T commands over serial and never drives a servo past its calibrated range.
- The repeatability spread for each axis is measured and written in the README, and the spread is within 3 degrees for the pan axis and within 5 degrees for the tilt axis, or the README explains why not.
- The torque check uses the measured mass and lever arm and shows a safety factor of at least 1.5 for the tilt axis.
- The repository includes the calibration header with your real values, not the example values from the starter code.

**Evidence to commit to GitHub:** the firmware, the calibration header, the repeatability log, the surge measurement, the torque calculation, the wiring diagram, the photo and the README. A reviewer should be able to reproduce a repeatability test from the README steps.

This project has no moving parts beyond the servos, but the servos can pinch fingers. Keep your hands clear of the horns during every move, and keep the head's mass within the torque check you wrote.

---

## Check your understanding

1. A servo's pulse width is 1.8 ms. Calculate the target angle using the standard 1000 to 2000 microsecond map, and then recalculate it for a servo whose calibrated range is 0 to 160 degrees over the same pulse range. Explain why a calibration is needed when the two results differ.
2. A 0.8 kg load hangs at 12 cm from a servo axis with the arm horizontal. Calculate the load torque in N·m and in kg·cm. Decide whether a 9 kg·cm servo is acceptable, and state the safety factor you used.
3. A servo repeats a 45-degree command with a spread of 4 degrees across five trials. Explain whether the spread is caused by the dead band, the feedback resolution or the mechanical play, and name one test that would separate them.
4. A robot with four MG996R servos on the Arduino's 5 V pin resets every time two of them start at the same time. Explain the cause using the surge current, and propose a change to the power design that does not change the software.
5. A bus servo reports a temperature that rises by 2 degrees per second while it holds a load. Decide whether the controller should reduce the load, stop the servo or ignore the report, and state the data you would need to make that decision.
6. A servo's control loop oscillates around the target after every move. List two causes in the servo and two causes in the command signal, and state which one you would test first.

---

### Answers

1. Using the standard map, the angle is (1800 minus 1000) times 180 / 1000 = 144 degrees. For a servo calibrated to 0 to 160 degrees over the same pulse range, the angle is 1800 minus 1000 over 1000 times 160, which is 128 degrees. The two results differ by 16 degrees, so a command of 144 degrees sent to the second servo would point it 16 degrees beyond the target. Calibration makes the command match the real joint angle.
2. The load torque is 0.8 times 9.81 times 0.12 = 0.942 N·m. In kg·cm that is 0.942 divided by 0.0981, about 9.6 kg·cm. A 9 kg·cm servo is below the load, so it is not acceptable. A servo rated at 18 kg·cm would give a safety factor of about 1.9, which meets the rule of 1.5 to 2.
3. A spread of 4 degrees could come from any of the three. A dead band of about 2 degrees would stop the servo short of the target by up to that amount, so the test is to command small steps from both directions and see whether the stop position changes. Feedback resolution appears as a fixed step size in the readings, while mechanical play shows as a difference that depends on the direction of the last move.
4. Four servos at stall draw a combined surge that the Uno's regulator cannot supply, so the 5 V rail sags and the Uno resets. The fix is a separate supply sized for the total stall current, with its ground connected to the Arduino's ground and a bulk capacitor on the servo rail. The software can also stagger the start times, but that does not change the design's power limit.
5. The controller should first look at the trend and the absolute temperature against the rated limit. A rise of 2 degrees per second may stop at a safe level or may climb toward the limit, so the controller needs the rated temperature, the current temperature and the load duration. A reasonable rule is to reduce the load or stop the servo before the rated limit, and to log the event for review.
6. Causes inside the servo include a high gain in the control loop and a worn potentiometer or encoder with jitter. Causes in the command signal include a command rate that changes too quickly and a step size that is larger than the joint can follow. The first test should be the command: send the same step with a slower ramp and watch whether the oscillation stops. If it does, the command is the cause, and the servo's gain is not.

---

## Sources

*Read*

- [Wikipedia: Servomotor](https://en.wikipedia.org/wiki/Servomotor) for the closed-loop principle and the types of servo.
- [Wikipedia: Pulse-width modulation](https://en.wikipedia.org/wiki/Pulse-width_modulation) for the duty-cycle relation used in the command signal.
- [Wikipedia: Potentiometer](https://en.wikipedia.org/wiki/Potentiometer) for the position sensor inside a hobby servo.
- [Wikipedia: Proportional-integral-derivative controller](https://en.wikipedia.org/wiki/Proportional%E2%80%93integral%E2%80%93derivative_controller) for the control loop used in the feedback section.
- The datasheets of your servos. Search the part number with "datasheet". Feetech and ROBOTIS publish protocol documents for their bus servos, and Feetech's protocol document is the reference for the bus packet format.

*Tutorials*

- Arduino Reference, the Servo library page, which lists attach(), write() and writeMicroseconds(). Site: docs.arduino.cc.
- SparkFun Learn, the tutorial on servo motors and PWM. Search "SparkFun Learn servo motor".
- GeeksforGeeks, the articles on servo motors and their working principle. Search "GeeksforGeeks servo motor working".

*Watch*

- Official Arduino channel, the videos on servo motors and the Servo library. Search "Arduino servo motor tutorial" on YouTube and check the channel name.
- MIT OpenCourseWare, the control systems course lectures on feedback and proportional control. Search the course title on ocw.mit.edu.

*Practice*

- Wokwi simulator at wokwi.com, which includes an Arduino Uno and a servo model. Build the sweep sketch and watch the output angle change with the pulse width.

---

## Tomorrow

Tomorrow you will use the stepper motor, which holds position in a different way. A stepper moves in fixed steps with no feedback, so its position is known only while it does not miss a step. Day 28 covers steps, the step angle, the driver and microstepping, and it compares the stepper's torque at speed with the servo's torque-speed curve from today. The calibration habits from this day carry over: measure the real behavior of the motor before you trust its datasheet.

**Next:** [Day 28: Stepper Motors](Robotics-Day-28-Stepper-Motors.md)
