# Day 29: Encoders

*100 Days of Robotics Engineering, Module 4: Sensors and Actuators*

**Time:** about 4 hours
**Level:** intermediate. You need Day 26 (gearing and the speed of a DC motor), Day 28 (why open-loop counting fails when a step is lost) and basic Arduino interrupts from Module 3. Today you measure motion directly, which is the fix for the problems you met in the last two days.

**By the end of this day, you will have:**

- A quadrature decoder that counts both edges on two channels with the correct direction, tested on a real encoder
- An absolute angle reading from a magnetic encoder over I2C, converted to degrees with its resolution stated
- A conversion table from counts to millimetres, RPM and output revolutions that uses the gearbox ratio and the encoder's counts per revolution
- A differential-drive odometry script that estimates the position of a two-wheeled test rig and checks it against a tape measure

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3 or newer, about $25 for an official board or $10 to $15 for a clone. Its external interrupt pins are 2 and 3.
- Two DC gearmotors with built-in quadrature encoders on the motor shaft, such as the common N20 gearmotors with a Hall-effect encoder, about $6 to $12 each. Check the listed counts per revolution (CPR) and the gear ratio. Many of these motors list 12 CPR at the motor shaft, which gives 48 counts per motor revolution with 4x decoding.
- The TB6612FNG motor driver from Day 13 to drive the motors, plus the 6 V bench supply with the 1.2 A current limit
- One AS5600 magnetic absolute encoder breakout, about $4 to $8, with a diametrically magnetized magnet of 6 mm diameter that matches the breakout's spacing. The AS5600 works at 3.3 V and 5 V logic on most breakouts. Check the breakout's logic level before you wire it.
- A pull-up resistor set, 4.7 kΩ or 10 kΩ, for any open-collector encoder output, about $1
- A 3D-printed or cardboard wheel of known radius, measured with calipers, about $5 in material, and a steel tape measure of at least 2 m, about $5
- Python 3.12 with NumPy 2.1 or newer and Matplotlib 3.9 or newer, both free and open source
- Optional: a Raspberry Pi 4 or 5 running ROS 2 Jazzy, which gives a path to the standard odometry message. Ubuntu 24.04 is the supported host for Jazzy. The Pi hardware costs roughly $35 to $80 depending on the model and memory, so check the current price.

**Prerequisites:**

- Day 26: the output speed of a geared motor, counts per revolution and gear ratio
- Day 28: why step counts drift and why a sensor is needed for true position
- Day 13: a logic-level signal, pull-up resistors and the H-bridge driver
- Module 3: attaching an interrupt handler to a pin, and reading a serial port
- Basic trigonometry for converting an angle and a distance into x and y coordinates

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How encoders developed](#how-encoders-developed)
- [Glossary](#glossary)
- [Incremental and absolute encoders](#incremental-and-absolute-encoders)
- [Counting pulses](#counting-pulses)
- [Quadrature and direction](#quadrature-and-direction)
- [Absolute encoders and Gray code](#absolute-encoders-and-gray-code)
- [From counts to speed and position](#from-counts-to-speed-and-position)
- [Odometry for a two-wheeled robot](#odometry-for-a-two-wheeled-robot)
- [Errors and limits](#errors-and-limits)
- [Hands-on exercises](#hands-on-exercises)
- [Project: wheel odometry on a two-wheeled test rig](#project-wheel-odometry-on-a-two-wheeled-test-rig)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

An encoder turns the rotation of a shaft into a stream of electrical signals that a microcontroller can count. It is the sensor that makes a motor's position and speed known, and it is the foundation of almost every closed-loop motion system, from the wheels of a warehouse robot to the joints of a surgical arm. Without an encoder, the robot knows only what it commanded. With one, the robot can measure what happened.

Encoders also fail in ways that are easy to miss. A count that is off by a factor of four because the decoder ignores one channel looks like a plausible speed. A wheel radius that is wrong by two millimetres gives a distance error that grows with every metre of travel. Today you build the decoding, the conversion and the odometry, and you check each step against a measurement, so that you know which number is wrong when a result looks off.

---

## How encoders developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| Early 1800s | Mechanical counters and dividing engines, such as those used in the machine shop, count the turns of a shaft with gears and dials. | The basic idea of counting turns of a shaft predates electronics. |
| 1880s to 1930s | Telegraph and instrument makers develop shaft-driven pulse generators and mechanical counters for positioning. | Early position sensing used the same counting principle. |
| 1950s to 1960s | Optical encoders with a patterned disc and a light source appear for machine tools and instrumentation. | Optical incremental encoders became the standard for precise motor feedback. |
| 1970s to 1990s | Gray code absolute encoders and Hall-effect sensors come into wider use for industrial and automotive applications. | Absolute encoders now report position at power-up without a homing move. |
| Today | Magnetic encoders such as the AS5600 are cheap, contactless and accurate to a fraction of a degree, and they are built into many hobby gearmotors. | A student can add an encoder to a robot for a few dollars. |

The principle has changed little, but the sensing has moved from light and gears to magnetic fields, which are cheaper and tolerant of dust.

---

## Glossary

- **Encoder:** a sensor that converts shaft rotation or linear motion into an electrical signal.
- **Incremental encoder:** an encoder that reports changes in position, not an absolute position. It needs a reference to know where it started.
- **Absolute encoder:** an encoder that reports its position within one turn, or across many turns, directly from its code pattern or magnetic field.
- **Counts per revolution (CPR):** the number of pulses on one channel in one shaft turn.
- **Quadrature:** a signal with two channels, A and B, offset by a quarter cycle, so the order of the edges gives the direction of rotation.
- **Edge:** a change of a signal from low to high or from high to low.
- **Decoding multiplier (x1, x2, x4):** the number of edges counted per cycle. x4 counts every edge on both channels, which gives four counts per cycle.
- **Index pulse (Z):** a pulse once per turn that gives a home reference on an incremental encoder.
- **Gray code:** a binary code in which adjacent values differ by one bit, which prevents large errors when a multi-bit position changes at once.
- **Odometry:** an estimate of a robot's position from its wheel motion over time.
- **Wheel radius:** the radius of the wheel's rolling contact, which converts wheel rotation into distance.
- **Track width:** the distance between the two drive wheels of a differential-drive robot, measured between their contact points.
- **Dead reckoning:** position estimation from motion measurements alone, without an external reference.

---

## Incremental and absolute encoders

An incremental encoder reports how far the shaft has moved from some starting point. Its output is a train of pulses, and each pulse means the shaft has moved by a fixed angle. Its position is the count of pulses since the controller last reset it. Power off the controller and the count is gone, unless it is stored.

An absolute encoder reports where the shaft is, within one turn or within many turns. Its output is a code, and the code for each angle is unique. Power off and on again, and it still knows the position. The cost is complexity: an absolute encoder with 12 bits of resolution has 4096 codes per turn, and each code needs its own sensor line or serial value.

The analogy is a counting clicker and a clock face. The clicker counts how many times a button is pressed, and it needs to be set to zero to mean anything. The clock face shows the time directly, but it does not tell you how many full days have passed unless you add that separately. Incremental encoders are the clicker, absolute encoders are the clock face.

The first principle is resolution. An encoder with N counts per turn divides the turn into N parts, and its angular resolution is 360 divided by N degrees. A 12-bit absolute encoder gives 4096 counts per turn, which is 360 / 4096 = 0.088 degrees per count. A 48-count-per-turn incremental encoder after 4x decoding gives 360 / 48 = 7.5 degrees per count. The same motor, measured with the two types, shows very different detail.

The worked example uses a wheel of radius 32.5 mm on a motor with a 30:1 gearbox and a magnetic incremental encoder of 12 CPR on the motor shaft. The encoder gives 12 times 4 = 48 counts per motor turn after 4x decoding. The output turn has 48 times 30 = 1440 counts. The wheel moves 2 times pi times 32.5 mm, which is 204.2 mm per turn, so each count is 204.2 / 1440 = 0.142 mm. The resolution of the robot's distance estimate is therefore about 0.14 mm per count, which is much better than the wheel's slip or the surface's variation.

The common mistake is to take the encoder's CPR as the number of counts per shaft turn after decoding, or to forget the gearbox. The CPR on the datasheet is often per channel and per motor turn, and the counts seen by the decoder depend on the multiplier and on whether the gearbox sits before or after the encoder. Check the datasheet, and confirm the count with a one-turn test by hand, using a mark on the shaft.

In the market, incremental encoders are the default feedback for motor drives, and absolute encoders are the default for joints that must know their position at power-up, such as robot arms and steering axes.

---

## Counting pulses

A counter increments on every pulse. On a microcontroller, the pulses arrive on a pin and the code increments a variable inside an interrupt handler, which runs when the pin changes. The interrupt is necessary because a fast encoder can produce edges more often than a loop can check them.

The first principle is that an edge-triggered interrupt is the only reliable way to count a fast signal in a simple program. A loop that reads a pin and counts changes will miss edges when the loop is busy. An interrupt runs immediately on the edge, and it stays short so the main program is not delayed.

The worked example is the maximum edge rate. A 12 CPR encoder on the motor shaft, read at 4x decoding, produces 48 edges per motor turn. At 6000 rpm, a common free-running speed for a small gearmotor, the motor turns 100 times per second, so the edge rate is 48 times 100 = 4800 edges per second. An Arduino Uno's interrupt handler can process that rate if the handler is short. At 20,000 rpm, the edge rate reaches 16,000 per second, and the handler must be very short. That is one reason to choose a microcontroller with faster interrupts for high-speed encoders.

The common mistake is to do the work of decoding inside the handler, such as printing to the serial port or computing floating-point values. Those operations take far longer than a pin change, and the handler then misses edges. Keep the handler to updating a counter and a state, and do all other work in the main loop.

The count is an integer that can overflow. A 16-bit counter wraps after 65,536 counts. The code must handle the wrap by computing the difference between two readings in a signed type, so that a change of minus one, after a wrap from zero to 65,535, is still correctly negative.

On the job, counting pulses in hardware is preferred. Many microcontrollers have a hardware quadrature decoder peripheral that counts the edges without an interrupt, which removes the missed-edge risk altogether.

---

## Quadrature and direction

A single channel of pulses tells you how far the shaft has moved, but not which way. Two channels, A and B, offset by a quarter of a cycle, tell you both. The order in which the A and B edges arrive gives the direction.

The first principle is the Gray-code sequence of two bits. The pair (A, B) moves through the states 00, 10, 11, 01 and back to 00 when the shaft turns one way, and through the same states in reverse when it turns the other way. Each change of one bit is one edge, so counting every change on both channels gives four counts per cycle, which is the x4 decoding.

The worked example uses a state table. The decoder keeps the last state and the new state, and it looks up the change in a table. The table in the starter code gives +1 for a forward step, -1 for a reverse step, and zero for no change or an impossible change, such as a jump from 00 to 11 that means a missed edge. The table is indexed by the old state times four plus the new state. Forward counts go 00 to 10, 10 to 11, 11 to 01 and 01 to 00, and they add 1 each time. Reverse counts subtract 1.

The common mistake is to decode only one channel, or to decode both channels but count only the rising edges of channel A. That gives x1 decoding, which has a quarter of the resolution, and it still gives the right direction. The result looks fine at a low speed and falls behind at a high speed. Decode all four transitions, and check the count after one turn.

Direction depends on the wiring. If the encoder counts up when the wheel moves backward, swap channel A and channel B in the code, not in the motor. The test is simple: turn the wheel forward by hand and watch the count. A positive direction must match the robot's forward motion.

In the market, quadrature signals are standard for motor drives. The drive's firmware reads A and B, and the robot's control code assumes the sign convention documented for the motor.

---

## Absolute encoders and Gray code

An absolute encoder gives the position in one reading, so it does not need homing. Its code pattern is designed so that a single step changes only one bit, which is the Gray code property. That property prevents a large error when the reading is taken during a transition, because a bit that changes late can give at most one step of error, not a wrong value of many steps.

The first principle is the binary code and its problem. In ordinary binary, the value 0111 changes to 1000 when moving one step, and four bits change at once. If the reader samples between the changes, it may read 0000 or 1111, which is far from the correct value. A Gray code changes one bit per step, so the worst case is one step of error, which is much smaller.

The worked example uses the AS5600 magnetic encoder, which measures the direction of a magnetic field with a Hall-effect sensor array. The chip gives a 12-bit reading, so each count is 360 / 4096 = 0.088 degrees. The reading is available over I2C at address 0x36, in a register called ANGLE, which holds the 12-bit angle. The conversion to degrees is angle = raw times 360 / 4096. A wheel with a 32.5 mm radius and a 12-bit encoder on the wheel shaft gives a distance resolution of 2 times pi times 32.5 times (1 / 4096) = 0.050 mm per count.

The common mistake is to mount the magnet off-centre or too far from the chip. The AS5600 needs the magnet's axis within a small tolerance of the chip's centre, and the field strength must be within a range set by the magnet's distance. A magnet that is too far gives no reading, and a magnet that is off-centre gives a reading that varies with angle, which looks like a wobble. Check the field strength in the chip's status register, and mount the magnet at the distance the datasheet gives.

Multi-turn absolute encoders add a counter for the number of turns, which is stored in a battery or a non-volatile memory. A single-turn encoder resets its angle at each full turn. Multi-turn is needed for joints that turn more than once, such as a rotating base, and the battery's life is then a design limit.

In the market, absolute encoders are the feedback of choice for robot joints, steering axes and industrial positioning, where a homing move at each power-up is undesirable.

---

## From counts to speed and position

Counts are not useful until they are converted into the quantities the robot needs: speed, distance and angle. The conversion uses the counts per revolution, the gear ratio and the wheel radius, and it must be written as a function with the constants named, so that a change of gearbox does not require a rewrite.

The first principle is the rate-to-speed conversion. If the count changes by N counts in time T, the shaft has turned N divided by the total counts per revolution, and its speed in revolutions per second is N / (counts per rev times T). Multiply by 60 for rpm, and by 2 times pi for radians per second. The output speed of a geared motor uses the output counts per revolution, which is the encoder CPR times the decoding multiplier times the gear ratio.

The worked example uses the output counts of 1440 from the earlier example. If the count rate is 1440 counts per second, the output turns at 1440 / 1440 = 1 rev/s, which is 60 rpm, or 2 pi rad/s, about 6.28 rad/s. The linear speed of the wheel is 6.28 times 0.0325 = 0.204 m/s, which is 204 mm per second, the circumference times one turn per second.

The common mistake is to use the motor's rated speed instead of the output speed. A motor at 6000 rpm drives a 30:1 gearbox to 200 rpm at the output, so a wheel speed computed from the motor's rpm is 30 times too fast. The second common mistake is to forget the sign of the count, which makes the robot's reverse motion look like forward motion in the odometry.

Position from the encoder is the total count times the distance per count. The distance per count is the wheel circumference divided by the output counts per revolution. For the example wheel, 204.2 mm divided by 1440 gives 0.142 mm per count, as before. A robot that counts 10,000 counts has travelled 1.42 m, provided the wheel has not slipped.

On the job, the speed and position from encoders are published to the control system at a fixed rate. In ROS 2, the wheel encoder values go to the odometry node, which is the next section.

---

## Odometry for a two-wheeled robot

Odometry is the estimate of the robot's position from its wheels. A differential-drive robot has two drive wheels on a common axle, and each wheel has its own encoder. The distance each wheel travels, and the difference between them, gives the robot's motion in the plane.

The first principle is the kinematics of two wheels. Let d_L and d_R be the distances travelled by the left and right wheels in one update. The distance the robot's centre moves is d = (d_L + d_R) / 2. The change in heading is dtheta = (d_R minus d_L) divided by the track width b, which is the distance between the two wheels' contact points. The robot's position updates as x = x + d times cos(theta + dtheta / 2), and y = y + d times sin(theta + dtheta / 2), and theta = theta + dtheta. The half-angle in the position update gives a better estimate for a curved path than using the start angle alone.

The worked example uses the wheel from the earlier example, with a radius of 32.5 mm and a track width of 150 mm. In one update, the left encoder counts 500 and the right encoder counts 480. The left distance is 500 times 0.142 = 71.0 mm, and the right distance is 480 times 0.142 = 68.2 mm. The robot's centre moves 69.6 mm, and the heading changes by (68.2 minus 71.0) / 150 = minus 0.0187 radians, about minus 1.07 degrees. The robot turns slightly to the right, toward the slower right wheel, which matches the count difference. Keep the heading in a known range, such as minus pi to pi, so that the arithmetic stays correct over a long run.

The common mistake is to use the wheel radius from the wheel's label or from memory. A wheel's effective radius under load is smaller than its free radius, because the tire compresses, and the effective radius is what matters for the distance. Measure it by rolling the wheel one full turn on a flat surface, measuring the distance, and dividing by 2 times pi. The track width has the same problem. Measure it between the contact points under the robot's weight, not between the wheel hubs.

Odometry drifts, because each estimate depends on the last one. A small error in the radius or the track width grows with distance, and wheel slip adds error that the encoder cannot see. The error is therefore a function of distance travelled and of the surface, and a good odometry estimate on a smooth floor may drift by several percent on carpet. The fix is to fuse odometry with another sensor, such as an IMU from Day 31 or a camera from Day 32, and to measure the drift in the project.

In ROS 2, the odometry is published as a nav_msgs/Odometry message, which carries the position, the orientation and the velocity, with a covariance that describes the estimate's uncertainty. Robot software uses this message as the input to localization, so the odometry must be correct before the rest of the stack can be trusted.

---

## Errors and limits

An encoder system has a few failure modes, and each one looks different in the data. Recognizing the pattern tells you which part to check. The three most common are missed edges, noisy or bouncing signals and wrong constants.

The first principle is that every error changes the count or the conversion. A missed edge drops a count, which makes the position short by one step. A noisy edge adds an extra count, which makes it long by one step. A wrong constant scales every count by the same factor, which makes the distance wrong by the same percentage at every distance. A test over several distances separates the constant error from the counting error, because a constant error grows in proportion to distance, while a counting error grows only with the number of edges lost.

The worked example is a test with a missed edge. The robot is driven 1 m forward and back. The odometry reports 1.000 m forward and 0.996 m back. The 4 mm difference on the return trip is not a constant scale error, because the forward trip matched. A missed edge on the return, about 28 counts at 0.142 mm each, would explain it, or the wheel might have slipped. A repeat of the test that shows the same error on the same segment points to the encoder, and a different segment points to slip or to a bump in the floor.

The common mistake is to treat the encoder as perfect. Magnetic encoders are affected by nearby motor fields, and optical encoders can be affected by dust. Mechanical switches used as encoders bounce, which gives false edges, and a signal that is not debounced in hardware or software gives a count that rises too fast. Route the encoder wires away from the motor wires, and keep the encoder's supply filtered.

Encoders also have a mechanical limit. The resolution of an encoder does not set the accuracy of the mechanism. A gearbox with backlash or a wheel with slip has error larger than one count, and the count will be precise and wrong. The accuracy of the robot's position is set by the weakest part of the chain, which is often not the encoder.

In the market, encoder failures are one of the most common causes of unexplained robot drift. A careful test with a tape measure is cheaper than a long debugging session, and it is the test every robot team runs before trusting its odometry.

---

## Hands-on exercises

Keep the motors on the 6 V bench supply with the 1.2 A current limit, and keep the fuse in the circuit, as Day 13 set up. Never probe an encoder's output with the motor powered and the wheel held by hand, because the motor can drive the wheel into your fingers.

1. **Confirm the counts per revolution.** Mark the motor shaft with a piece of tape. Turn the shaft by hand exactly one revolution while counting the pulses on channel A with a count loop in the Arduino. Compare the count with the datasheet's CPR. If it differs, check whether the datasheet gives counts before or after decoding.

2. **Decode the quadrature signal.** Wire channels A and B to pins 2 and 3 with pull-up resistors if the outputs are open-collector. Upload the decoder from the project, turn the shaft forward and backward by hand, and check the count sign. Then turn it through one full turn and confirm the count is 48 times the CPR divided by 12, or the value your datasheet gives.

3. **Measure the wheel radius.** Mark the wheel's tread with a piece of tape, roll it on the tape measure for exactly one revolution, and measure the distance from start to finish. Compute the radius as distance divided by 2 times pi. Repeat three times and average. Compare with the radius from the calipers and state the difference.

4. **Check the speed conversion.** Drive the motor at a fixed PWM duty of 60 percent through the TB6612FNG. Read the count rate over five seconds and convert it to output rpm using the output counts per revolution. Compare with a phone optical tachometer reading on the output shaft. The two values should agree within about 5 percent.

5. **Test the absolute encoder.** Mount the AS5600 breakout and the magnet on a shaft, with the magnet centred and about 2 mm from the chip. Read the raw angle over I2C and rotate the shaft slowly by hand through one turn. Plot the angle against the shaft's marked position. Record the readings' spread at a fixed position.

6. **Odometry square.** Drive the robot in a square of 0.5 m sides, starting and ending at the same mark. Record the position from the odometry at each corner, and measure the final position error with the tape measure. Compute the error as a percentage of the total distance.

---

## Project: wheel odometry on a two-wheeled test rig

This project builds a small two-wheeled test rig with an encoder on each drive wheel, a decoder, an odometry calculation and a validation test against a tape measure. It is the same odometry that a ROS 2 robot publishes, reduced to an Arduino and a Python script you can read in one sitting.

**Goal:** build a differential-drive rig that estimates its own position from two wheel encoders, record a set of test runs with the tape measure as ground truth, and show how the error depends on distance.

**Repo layout:**

```
100-days-robotics-engineering-day-29-encoders/
├── README.md
├── firmware/
│   └── wheel_encoders/
│       └── wheel_encoders.ino
├── analysis/
│   ├── odometry.py
│   └── requirements.txt
├── data/
│   ├── raw_counts_square.csv
│   └── tape_measure_check.csv
├── calculations/
│   ├── counts_per_revolution.md
│   └── wheel_and_track_measurement.md
├── plots/
│   └── odometry_path.png
└── images/
    └── rig_setup.jpg
```

**Starter code:** `firmware/wheel_encoders/wheel_encoders.ino`

```cpp
// The Uno has only two interrupt pins (2 and 3). This starter uses them for the
// left encoder and polls the right encoder on pins 4 and 5 in loop(). Polling can
// miss edges, so use a board with more interrupt pins or a hardware decoder for a
// full build. The constants in the analysis script must also be your own measurements.

const int L_A = 2;   // interrupt pin
const int L_B = 3;   // interrupt pin
const int R_A = 4;   // polled
const int R_B = 5;   // polled

// Quadrature transition table, indexed by (old_state << 2) | new_state.
// Forward is 00 -> 10 -> 11 -> 01 -> 00 with state = (A << 1) | B.
const int8_t TRANS[16] = {0, -1, 1, 0,  1, 0, 0, -1,  -1, 0, 0, 1,  0, 1, -1, 0};

volatile long leftCount = 0;
volatile uint8_t leftState = 0;

long rightCount = 0;
uint8_t rightState = 0;

void leftISR() {
  uint8_t s = (digitalRead(L_A) << 1) | digitalRead(L_B);
  leftCount += TRANS[(leftState << 2) | s];
  leftState = s;
}

void setup() {
  Serial.begin(115200);
  pinMode(L_A, INPUT_PULLUP);
  pinMode(L_B, INPUT_PULLUP);
  pinMode(R_A, INPUT_PULLUP);
  pinMode(R_B, INPUT_PULLUP);
  leftState = (digitalRead(L_A) << 1) | digitalRead(L_B);
  rightState = (digitalRead(R_A) << 1) | digitalRead(R_B);
  attachInterrupt(digitalPinToInterrupt(L_A), leftISR, CHANGE);
  attachInterrupt(digitalPinToInterrupt(L_B), leftISR, CHANGE);
  Serial.println("ms,left,right");
}

void loop() {
  // Poll the right encoder. Polling can miss edges at high speed, so keep the rig slow.
  uint8_t s = (digitalRead(R_A) << 1) | digitalRead(R_B);
  rightCount += TRANS[(rightState << 2) | s];
  rightState = s;

  static unsigned long last = 0;
  if (millis() - last >= 50) {
    last = millis();
    noInterrupts();
    long l = leftCount;
    interrupts();
    Serial.print(last);
    Serial.print(",");
    Serial.print(l);
    Serial.print(",");
    Serial.println(rightCount);
  }
}
```

The starter code logs the left and right counts every 50 milliseconds. The left encoder uses interrupts, and the right encoder is polled, which is a compromise forced by the Uno's two interrupt pins. The polled encoder can miss edges at high speed, which the README must state as a limit. A full build uses a board with more interrupt pins, a hardware decoder or two boards.

The analysis script `analysis/odometry.py` reads the CSV, converts counts to distance with the constants you measured, and integrates the position. You write this script as part of the project, using the odometry equations from the lesson. The outline below shows the structure.

```python
import csv
import math
from pathlib import Path

WHEEL_RADIUS_M = 0.0325   # measured, not the label value
TRACK_WIDTH_M  = 0.150    # measured between contact points
COUNTS_PER_REV = 1440     # output counts after decoding and gearing

def count_to_m(count):
    return count * (2 * math.pi * WHEEL_RADIUS_M) / COUNTS_PER_REV

# Read rows, keep the previous counts, compute d and dtheta per row,
# update x, y and theta, and write the path to a new CSV.
```

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the encoder's datasheet values, the wiring diagram and the power arrangement.
2. Write `calculations/counts_per_revolution.md` with the CPR from the datasheet, the decoding multiplier and the gear ratio, and the one-turn check from the exercises.
3. Write `calculations/wheel_and_track_measurement.md` with the measured wheel radius from three rolls and the track width measured between the contact points.
4. Drive the rig in a 1 m straight line and back, and in a 0.5 m square, and log the counts to `data/raw_counts_square.csv`. Mark the start and end points on the floor, and measure the end error with the tape measure into `data/tape_measure_check.csv`.
5. Complete `analysis/odometry.py` with the odometry update and the path output. Plot the estimated path in `plots/odometry_path.png`.
6. Compute the error as a percentage of distance for each run and state in the README whether the error is a scale error or a counting error.

**README checklist:**

- [ ] Goal, encoder model and its CPR, and the motors and the gear ratio
- [ ] Wiring diagram, with the pull-up resistors and the power supply
- [ ] Measured wheel radius and track width, with the method
- [ ] Count-to-distance conversion and the one-turn check
- [ ] Table of runs with the odometry and tape-measure values
- [ ] Plot of the estimated path
- [ ] One limit of the rig, such as the polled encoder, the wheel slip or the drift over distance

**Done when:**

- The one-turn count matches the expected CPR within 1 count, and the sign of the count is positive for forward motion.
- The wheel radius from the rolls agrees with the calipers within 2 mm, or the README explains the difference.
- The 1 m straight run is within 3 percent of the tape measure, and the README reports the percentage for each run.
- The odometry script runs from the repo root and produces the path plot from the raw CSV.

**Evidence to commit to GitHub:** the firmware, the analysis script, the raw data, the tape-measure check, the calculations, the plot, the photo and the README. A reviewer should be able to rerun the script on the raw data and see the same path.

This rig moves on the floor, and the encoders sit near the motor. Keep the rig on a clear floor, keep the current limit set, and stop the test if a wheel jams or a driver gets hot.

---

## Check your understanding

1. An encoder reads 1024 CPR on its motor shaft and is decoded at x4. The motor drives a 50:1 gearbox. Calculate the output counts per revolution. Explain why the answer changes if the encoder is mounted on the output shaft instead.
2. A wheel of radius 40 mm has an output of 1200 counts per revolution. The odometry reports 3000 counts for a run that a tape measure shows as 0.600 m. Calculate the distance the odometry reports, the difference from the tape, and the number of extra counts that would explain it. Decide whether the difference is a constant error or a counting error, and state the test that would separate them.
3. A quadrature decoder counts only the rising edges of channel A. Explain the effect on resolution, and explain why the robot still appears to move in the right direction at low speed.
4. Two encoders on a differential drive read 1200 and 1150 counts in one update. The track width is 160 mm, the output counts per revolution is 1440 and the wheel radius is 32.5 mm. Calculate the distance and the heading change for the update.
5. An absolute magnetic encoder reads the same angle every time the shaft is at rest, but the reading wobbles by 3 degrees as the shaft turns slowly through one revolution. Name two causes related to the magnet and one related to the mounting, and state the test you would run first.
6. A robot's odometry is correct on a smooth floor and drifts by 5 percent on carpet. Explain why the encoder can be correct while the odometry is wrong, and name one sensor from later in this module that would reduce the drift.

---

### Answers

1. The output counts per revolution are 1024 times 4 times 50 = 204,800 counts. If the encoder is on the output shaft after the gearbox, the counts per output revolution are 1024 times 4 = 4096, which is the output CPR without the gear ratio. The gear ratio applies only when the encoder sits on the motor side, so the datasheet and the mounting position must be checked together.
2. The odometry turns 3000 / 1200 = 2.5 output revolutions. Each revolution is 2 times pi times 0.040 = 0.2513 m, so the odometry reports 2.5 times 0.2513, which is 0.628 m. The tape shows 0.600 m, so the odometry is 28 mm, or 4.7 percent, long. At 0.2513 / 1200 = 0.2094 mm per count, the 28 mm is equivalent to about 134 extra counts. A constant error, such as a radius that is 4.7 percent too large, scales with distance, so the error on a 2 m run would be about 94 mm. A counting error, such as noise adding edges, adds a roughly fixed number of counts per event, so its size does not scale the same way. Running the same distances at two lengths separates the two cases.
3. Counting only the rising edges of channel A gives one count per cycle of A, which is one quarter of the x4 resolution. The direction still appears right at low speed because the decoder's edge order is unchanged, but the count is coarse and the position error is four times larger per count. At high speed, the decoder may miss the edges it needs, and the count rate may be too low to track the motion.
4. The left distance is 1200 times 2 times pi times 0.0325 divided by 1440, which is 1200 times 0.1418 mm, or 170.2 mm. The right distance is 1150 times 0.1418 mm, or 163.1 mm. The centre moves (170.2 plus 163.1) / 2, which is 166.6 mm. The heading change is (163.1 minus 170.2) / 160, which is minus 0.0444 radians, about minus 2.5 degrees. The robot turns slightly to the right, which is the side of the slower wheel.
5. Two magnet causes are an off-centre magnet, which gives a reading that varies sinusoidally with angle, and a magnet with a field strength outside the chip's range, which gives a nonlinear reading. A mounting cause is a shaft with runout, which moves the magnet off the chip's axis as the shaft turns. The first test is to check the field strength register and to measure the runout with a dial indicator or by watching the magnet's distance as the shaft turns.
6. The encoder counts the wheel's rotation, and the wheel turns the same amount on carpet as on a floor. The error is in the assumption that the wheel's turns become robot motion, because the carpet causes the wheel to slip or to deform, so the distance on the floor is less than the wheel's rolling distance. An IMU from Day 31 reports the robot's heading and acceleration independently of the wheels, and the camera in Day 32 can give a visual reference for the position.

---

## Sources

*Read*

- [Wikipedia: Rotary encoder](https://en.wikipedia.org/wiki/Rotary_encoder) for the types, the resolution and the quadrature signal.
- [Wikipedia: Gray code](https://en.wikipedia.org/wiki/Gray_code) for the single-bit-change property used by absolute encoders.
- [Wikipedia: Odometry](https://en.wikipedia.org/wiki/Odometry) for the principle of wheel-based position estimation.
- [Wikipedia: Dead reckoning](https://en.wikipedia.org/wiki/Dead_reckoning) for the drift that comes with odometry alone.
- The AS5600 datasheet from ams OSRAM. Search "AS5600 datasheet ams" for the register map, the I2C address and the magnet specification.
- The Arduino Language Reference page for attachInterrupt(), at docs.arduino.cc. It explains which pins support interrupts on the Uno.

*Tutorials*

- SparkFun Learn, the tutorial on rotary encoders and their hookup. Search "SparkFun Learn rotary encoder".
- Adafruit Learning System, the guides on quadrature encoders and motor feedback. Search "Adafruit quadrature encoder guide".
- GeeksforGeeks, the article on rotary encoders and their working principle. Search "GeeksforGeeks rotary encoder".
- ROS 2 Jazzy documentation at docs.ros.org, the tutorials on writing a publisher and the nav_msgs/Odometry message definition. Search "nav_msgs Odometry ROS 2 Jazzy".

*Watch*

- Cyrill Stachniss, the lecture series on robot localization and odometry on his YouTube channel. Search "Stachniss robot localization odometry".
- MIT OpenCourseWare, 2.12 Introduction to Robotics, the lectures on sensors and on mobile robot kinematics. Search the course title on ocw.mit.edu.

*Practice*

- Wokwi simulator at wokwi.com, which includes an Arduino Uno and a rotary encoder model. Build the quadrature decoder and watch the count change as you turn the simulated knob in each direction.

---

## Tomorrow

Tomorrow you will measure something the wheels cannot see: how the robot is rotating and accelerating in space. Day 30 covers distance sensors, including ultrasonic, infrared, time-of-flight and LiDAR, and how each one measures range, field of view, resolution, noise and limits. A distance sensor is the second kind of reference, after the encoder, that a robot uses to correct its own map of the world.

**Next:** [Day 30: Distance Sensors](Robotics-Day-30-Distance-Sensors.md)
