# Day 31: IMUs

*100 Days of Robotics Engineering, Module 4: Sensors and Actuators*

**Time:** about 4 hours
**Level:** intermediate. You need Day 29 (odometry and its drift), Day 30 (noise and filtering) and Module 3 (I2C and reading registers). Today you learn the sensor that tells a robot how it is tilted and how fast it is turning, independently of its wheels.

**By the end of this day, you will have:**

- A raw six-axis reading from an MPU-6050 over I2C, converted to g and degrees per second with the sensitivity from the datasheet
- A measured gyroscope bias from one minute of stationary data, and a drift estimate in degrees per minute
- A tilt estimate from the accelerometer alone, and a complementary filter that combines it with the gyroscope
- A comparison, on a protractor turntable, of the accelerometer, the gyroscope and the filtered angle at known angles

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3 or newer, about $25 for an official board or $10 to $15 for a clone
- One MPU-6050 six-axis IMU breakout, about $3 to $6. The MPU-6050 is an older part, from InvenSense and now TDK, and it is widely cloned, so the test values in this lesson apply to the genuine chip. Check the breakout's logic level and regulator.
- Optional: a BNO055 or BNO085 nine-axis IMU with on-chip sensor fusion, about $25 to $35 for the BNO055 breakout, for a comparison in the project. The BNO055 is from Bosch Sensortec and the BNO085 is from CEVA (formerly Hillcrest Labs). Check the current price and availability.
- A protractor or a printed angle gauge, and a flat board that can be tilted at known angles, such as a book under one end
- A rotating turntable made from a plastic lid or a cardboard disk with 10 degree marks, to test heading
- A digital inclinometer app on a phone, free, as a reference, although its accuracy is about 0.5 degrees at best
- Python 3.12 with NumPy 2.1 or newer and Matplotlib 3.9 or newer, both free
- Optional: a Raspberry Pi running ROS 2 Jazzy, which gives access to the sensor_msgs/Imu message used by most robot localization stacks

**Prerequisites:**

- Day 29: odometry, the difference between an estimated pose and a measured one, and drift
- Day 30: noise, filtering and the difference between accuracy and precision
- Module 3: reading registers over I2C with Wire.beginTransmission() and Wire.requestFrom()
- Basic trigonometry: atan2 and the relation between a tilt angle and the gravity vector

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How inertial sensing developed](#how-inertial-sensing-developed)
- [Glossary](#glossary)
- [Axes and frames](#axes-and-frames)
- [Accelerometers](#accelerometers)
- [Gyroscopes](#gyroscopes)
- [Magnetometers](#magnetometers)
- [Bias, scale and drift](#bias-scale-and-drift)
- [Calibration](#calibration)
- [Combining sensors with a complementary filter](#combining-sensors-with-a-complementary-filter)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a tilt and heading estimator with a known reference](#project-a-tilt-and-heading-estimator-with-a-known-reference)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A wheel encoder tells a robot how far its wheels have turned. It does not tell the robot that it has tipped onto a ramp, that it has slid sideways on ice, or that it has rotated while the wheels were locked. An inertial measurement unit, or IMU, reports the robot's own acceleration and rotation, independent of the wheels. Combined with the encoders from Day 29, it gives the robot a much better estimate of its orientation than either sensor alone.

The IMU has a serious weakness of its own. Its gyroscope drifts, its accelerometer is confused by motion, and its magnetometer is confused by steel and by the robot's own motors. The skill of today's lesson is knowing which sensor to trust at which moment, and checking each one against a known angle, so that you can tell a real tilt from a sensor error.

---

## How inertial sensing developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1850s to 1900s | The gyroscope is demonstrated by Léon Foucault, who uses a spinning wheel to show the Earth's rotation, and gyrocompasses for ships follow. | The gyroscope's principle of holding a fixed direction against rotation is the basis of inertial navigation. |
| 1940s to 1960s | Inertial navigation systems for aircraft and missiles use large mechanical gyroscopes and accelerometers, housed in gimbals. | Inertial navigation is the original application of the IMU, and it is still used in aircraft. |
| 1970s to 1990s | Vibrating-structure gyroscopes and silicon accelerometers are developed, and microelectromechanical systems (MEMS) make the sensors small and cheap. | The MEMS sensor is the reason an IMU costs a few dollars today. |
| 2000s | Consumer electronics adopt MEMS accelerometers and gyroscopes for phones, games and cameras. | Most hobby IMUs come from this consumer market. |
| Today | Six-axis and nine-axis IMUs with on-chip fusion are standard in drones, phones, cars and mobile robots. | A robot can get a fused orientation estimate from one chip on one bus. |

The principles are old. The MEMS process made them small, and the consumer market made them cheap, which is why an IMU on a robot is now routine.

---

## Glossary

- **IMU (inertial measurement unit):** a sensor package that measures acceleration and angular rate, usually along three axes each.
- **Accelerometer:** a sensor that measures specific force, which is acceleration relative to free fall. At rest on a table it reads gravity.
- **Gyroscope (gyro):** a sensor that measures angular rate, the rate of rotation about each axis, in degrees or radians per second.
- **Magnetometer:** a sensor that measures the direction and strength of the magnetic field, which gives a heading relative to the Earth's field.
- **Six-axis IMU:** an accelerometer and gyroscope in one package, with three axes each.
- **Nine-axis IMU:** a six-axis IMU with a magnetometer added, with three axes each.
- **g:** the acceleration of gravity at the Earth's surface, about 9.81 m/s².
- **Bias:** a constant offset in a sensor's reading. A gyro with a bias of 0.5 degrees per second reports that rate even when it is still.
- **Drift:** a slow change in a sensor's error over time, which for a gyro comes from integrating the bias and noise.
- **Scale factor:** the relation between the sensor's raw output and the physical value, such as LSB per g.
- **Sensitivity:** the change in output for a unit change in the input, which is the reciprocal of the scale factor.
- **Complementary filter:** a filter that combines a high-pass-filtered gyro estimate with a low-pass-filtered accelerometer estimate.
- **Hard iron and soft iron:** magnetic distortions. Hard iron is a constant offset from a permanent magnet or a steel part. Soft iron is a distortion of the field's shape from a material that bends it.
- **Heading:** the robot's direction of travel, measured as an angle from a reference direction, usually north in a magnetometer system.

---

## Axes and frames

An IMU measures in its own frame, which is attached to the chip. The chip's axes, usually called X, Y and Z, are fixed to the board. The robot's frame is fixed to the robot, and the two frames differ if the board is mounted at an angle. Every calculation in this lesson assumes that the two frames agree, and a mounting error is a common source of wrong tilt.

The first principle is the right-hand rule. X points forward on the board, Y points to the left, and Z points up, in the common robotics convention known as ROS REP-103. The rotation about each axis is positive by the right-hand rule: curl the fingers of the right hand in the direction of the rotation, and the thumb points along the positive axis. Many breakouts use a different convention, so check the board's silkscreen and the datasheet before you trust a sign.

The worked example is a board mounted flat on a table with its Z axis up. At rest, the accelerometer reads about +1 g on Z, because the table pushes up on the chip with a force that balances gravity. It reads about 0 g on X and Y. If the board is turned upside down, the Z reading is about -1 g. The sign of the reading tells you which way the board faces, and the magnitude tells you that it is at rest.

The common mistake is to assume that a sensor's Z axis always points up. A board mounted on a robot's side or upside down reads gravity on a different axis. Mark each axis on the board before you mount it, and write the orientation in the README.

On the job, the frame convention is part of the sensor's calibration document, and it is the first thing a robot software team checks when a robot turns the wrong way.

---

## Accelerometers

An accelerometer measures the force on a small proof mass suspended inside the chip. When the chip accelerates, the mass lags and deflects, and the deflection is measured with a capacitor or piezoresistive element. At rest, the mass is pulled by gravity, so the sensor reads gravity as an acceleration upward.

The first principle is that an accelerometer measures specific force, not the acceleration of the chip. A chip in free fall reads zero, even though it is accelerating at 9.81 m/s² toward the ground, because the proof mass falls with it. A chip on a table reads 1 g upward, because the table holds it up. The reading is therefore the sum of the chip's acceleration and the gravity reaction, and in the static case it gives the direction of gravity, which is the tilt.

The worked example uses the tilt from a static reading. Suppose the board is tilted so that its X axis points up at an angle theta above the horizontal. The gravity vector projects onto the X axis as -sin(theta) in g, and onto the Z axis as cos(theta). A reading of ax = 0.5 g and az = 0.866 g gives an angle theta = atan2(ax, az) = atan2(0.5, 0.866) = 30 degrees. In the common formula, pitch = atan2(-ax, sqrt(ay^2 + az^2)) and roll = atan2(ay, az), which give the tilt around each axis in the static case.

The common mistake is to use an accelerometer's tilt while the robot is moving. A robot that accelerates or turns sees its own acceleration added to gravity, and the tilt estimate is wrong by the amount of that acceleration. A robot that drives at 1 m/s² forward reads an extra 0.1 g, which is an error of about 6 degrees in the pitch estimate. The accelerometer's tilt is trustworthy only when the robot is still or moving slowly.

Another mistake is to read the accelerometer's output directly as acceleration without removing gravity and the bias. The raw output is a signal in LSB, and it must be converted with the scale factor and corrected for bias before it means anything physical.

In the market, accelerometers are used for tilt detection in phones, tilt compensation in cameras, crash detection in cars and step counting in wearables. In robots, they give a gravity reference that corrects the drift of the gyroscope.

---

## Gyroscopes

A MEMS gyroscope measures rotation through the Coriolis effect. A vibrating structure oscillates in one direction, and when the chip rotates, the Coriolis force deflects the vibration in a perpendicular direction. The sensor measures that deflection, which is proportional to the angular rate.

The first principle is that the gyroscope measures angular rate, and the angle is the integral of the rate. If the rate is omega in degrees per second and the sample time is dt, the change in angle is omega times dt, and the total angle is the sum over all samples. An integral is sensitive to any constant error in the rate, which is the bias, because the error accumulates over time.

The worked example uses the bias. Suppose a gyro reads 0.5 degrees per second while the board is perfectly still, which is a bias of 0.5 degrees per second. After one minute, the integrated heading error is 0.5 times 60 = 30 degrees. After ten minutes, it is 300 degrees. A gyro with a bias of this size is useless for long-term heading without correction, and a gyro with a bias of 0.01 degrees per second drifts only 0.6 degrees per minute. The bias is therefore the first number to measure.

The common mistake is to start the gyro's integration while the sensor is still moving, or while the bias is still settling after power-up. A gyro often needs a few seconds after power-up before its bias is stable, and a robot that starts moving at once integrates the bias plus the motion. The fix is to average the gyro for 5 to 10 seconds at rest, subtract the average as the bias, and start the integration after that.

Temperature changes the bias. A gyro that warms up over an hour drifts more than the same gyro at a steady temperature. The datasheet gives the temperature coefficient, and a robot that works outdoors must account for it, or recalibrate periodically.

In the market, gyroscopes are used for stabilization in cameras and drones, for heading in phones and for rate control in aircraft autopilots. A robot's gyro is usually fused with another sensor, because its drift is too large for long-term use alone.

---

## Magnetometers

A magnetometer measures the Earth's magnetic field along each axis. Its readings give a heading, the angle of the board's X axis relative to magnetic north, when the board is level. When the board is tilted, the readings must be corrected with the accelerometer's tilt before the heading is computed.

The first principle is that the horizontal components of the magnetic field give the direction to magnetic north. For a level board with the X axis forward, the heading is atan2(-my, mx), with the sign depending on the frame convention. The magnetic field of the Earth is also inclined toward the ground, with the dip angle varying by place, so the vertical component is large and the horizontal component is smaller. A tilted board therefore needs the tilt correction before its horizontal field is read.

The worked example is a board that reads mx = 0.20 and my = 0.10 in arbitrary units while level. The heading is atan2(-0.10, 0.20), which is about minus 26.6 degrees. A correction of the magnetic declination, which is the difference between magnetic north and true north at the location, is then added. The declination depends on where you are, and it changes slowly over the years, so some locations differ from north by a few degrees while others differ by more than 10 degrees. Look up the declination for your location with the free calculator on the NOAA National Centers for Environmental Information website, and state the value in the README.

The common mistake is to ignore the robot's own magnetic fields. Motors, batteries, speakers and steel screws all distort the field, and the distortion appears as a constant offset (hard iron) or a stretched circle (soft iron). A magnetometer on a robot that runs its motors reads a heading that changes with the motor current. The fix is to mount the magnetometer far from the motors and steel, to calibrate with the robot in its operating state, and to check the heading with a turntable.

In the market, the magnetometer is used in phones for the compass, in drones for heading and in some vehicles for navigation. It is the sensor that gives an absolute heading, which the gyro cannot provide over time.

---

## Bias, scale and drift

Every IMU sensor has three kinds of error: a bias, a scale error and noise. The bias is a constant offset. The scale error is a multiplicative error, so the reading is 1.02 times the true value, for example. The noise is random variation. Each one shows up differently, and each one needs its own measurement.

The first principle is that bias and scale are separate errors with separate tests. A bias shows up at rest, as a nonzero reading when the true value is zero. A scale error shows up at a known nonzero value, as a reading that is off by a percentage. A test that uses only one reference, such as the board at rest, measures the bias but cannot find the scale error, and a test with a turntable at several angles measures both.

The worked example uses the accelerometer's scale. The MPU-6050's default accelerometer range is plus or minus 2 g, with a sensitivity of 16,384 LSB per g, according to its datasheet. If the board reads a raw value of 16,500 on Z while lying flat, the reading is 16,500 / 16,384 = 1.007 g, which is 0.7 percent high. That error is a scale error, and it is small. The same board at a 90-degree tilt should read 0 g on the tilted axis, and any nonzero reading there is a bias.

The gyroscope's drift is the biggest practical problem. A gyro with a noise density of 0.005 degrees per second per root hertz and a bias of 0.1 degrees per second may drift by several degrees over five minutes. The integrated noise is a random walk, which grows with the square root of time, and the bias grows linearly. The bias is removable by calibration. The random walk is not, and it sets a limit on how long a gyro-only estimate is useful.

The common mistake is to believe that a sensor's datasheet figure is the performance of your board. The datasheet gives typical values at a stated temperature and supply voltage, and the board's mounting, the supply noise and the temperature can make the real performance worse. Measure the bias and drift on your board, and use those numbers in the design.

On the job, the bias and the scale factor are measured at the factory and stored in the sensor or in the robot's calibration file. A robot team that skips the calibration will see its heading drift and its tilt misread.

---

## Calibration

Calibration is the step that finds the bias, the scale and the frame orientation for one physical board. It turns a sensor's typical performance into the performance of your sensor, which is the only performance that matters for the robot.

The first principle is that calibration is a fit between the sensor's raw output and a known reference. For the accelerometer, the reference is gravity, and the six-position method rotates the board so each axis points up and down in turn. For each position, the reading on the vertical axis should be +1 g or -1 g, and the averages give the offset and the scale for each axis. For the gyro, the reference is zero rate, and the average of a still reading gives the bias. For the magnetometer, the reference is a sphere: the readings from many orientations should lie on a sphere, and the centre and the stretch of the cloud give the hard and soft iron corrections.

The worked example is the gyro bias. Record 6000 samples at 100 Hz with the board still, which takes one minute. The average of the Z-axis reading is, say, 0.48 degrees per second. That is the bias, and it is subtracted from every later reading. The standard deviation of the same data is the noise, which might be 0.05 degrees per second. The gyro's bias after calibration is the residual, which should be well below the noise, if the calibration is done at the same temperature as the test.

The common mistake is to calibrate the gyro while the board is warming up. A gyro that is still warming changes its bias during the minute of calibration, and the average is wrong. Power the board for 5 minutes before the calibration, and keep the board still during it.

Calibration also needs a check. A calibration that is not checked against a known angle is only a guess. The turntable test in the project gives that check, and the difference between the calibrated angle and the protractor angle is the figure to report.

In the market, the factory calibration of an IMU is part of the product's quality test, and robot teams repeat a simpler version for each robot because the mounting changes the offsets.

---

## Combining sensors with a complementary filter

Each IMU sensor fails in a different way. The accelerometer gives a stable tilt at rest but is confused by motion. The gyroscope gives a smooth rate and a precise change in angle but drifts. A complementary filter combines them, trusting the gyroscope for fast changes and the accelerometer for the long-term average.

The first principle is the high-pass and low-pass split. The gyroscope integral is a high-pass estimate: it is accurate for fast changes and drifts slowly. The accelerometer tilt is a low-pass estimate: it is accurate on average and noisy in the short term. A filter that adds the two with weights that sum to one gives an estimate that uses the good part of each.

The filter is: angle = alpha times (angle + omega times dt) + (1 minus alpha) times accel_angle, where alpha is close to 1, such as 0.98. The term angle + omega times dt is the gyro's prediction. The term accel_angle is the accelerometer's correction, applied with a small weight. The filter's time constant is tau = alpha times dt / (1 minus alpha). For a 100 Hz loop with dt = 0.01 s and alpha = 0.98, tau = 0.98 times 0.01 / 0.02 = 0.49 s. The accelerometer corrects the gyro's drift on that time scale.

The worked example uses the numbers above. Suppose the gyro bias is calibrated to 0 and the board is held at a steady 20 degrees. The gyro reads 0 degrees per second, and the accelerometer reads 20 degrees with a noise of plus or minus 2 degrees. After many steps, the filter's output converges to the accelerometer's average, which is 20 degrees, with a smoothed noise much smaller than 2 degrees. When the board is then rotated quickly by 30 degrees in 0.3 seconds, the gyro responds immediately, and the filter follows the motion. The accelerometer's error during the motion is given only a small weight, which avoids the error from the robot's own acceleration.

The common mistake is to choose alpha without measuring the sensors. A very high alpha trusts the gyro too much, and a drift returns. A low alpha trusts the accelerometer too much, and the estimate jitters with motion. The right alpha depends on the gyro's drift and the accelerometer's noise, and the project measures both, so you can set alpha from the data.

A Kalman filter, which the Day 31 project does not require, is a more general version of the same idea, and it uses a model of the sensors' noise to set the weights. The complementary filter is simpler and is enough for a level robot. In the market, a robot with a nine-axis IMU and a fused orientation output often uses a Kalman-type filter inside the sensor or in the robot's software, and the complementary filter is a good first step.

The complementary filter is also the first place to check the frame convention. A sign error in the gyro or the accelerometer gives a filter that drifts in the wrong direction, and the test on the turntable shows it at once.

---

## Hands-on exercises

Keep the IMU on a stable surface, away from the motors and any steel. Power the board from the Arduino's 5 V pin if the breakout has a regulator, and check the breakout's voltage before you connect it. The IMU has no safety hazard, but a loose wire can short the 3.3 V supply, so check each connection before you power the board.

1. **Read the raw values.** Wire the MPU-6050 to the Arduino's SDA and SCL pins, with its AD0 pin low so the address is 0x68. Write a sketch that wakes the sensor by writing 0 to register 0x6B, then reads 14 bytes from register 0x3B. Convert the first six bytes into accelerometer values in g with the scale of 16,384 LSB per g, and the next six into gyro values in degrees per second with 131 LSB per degree per second. Print the values for ten seconds with the board flat.

2. **Gyro bias.** Keep the board still for 60 seconds after a 5-minute warm-up. Average the gyro readings on each axis and record the bias. Compute the standard deviation as the noise. Record both numbers and the temperature from the MPU-6050's temperature register, which is 0x41 and 0x42.

3. **Gyro drift.** After the calibration, read the integrated Z angle for 5 minutes with the board still. Compare the integrated angle with the expected zero. Compute the drift in degrees per minute and compare it with the bias you measured in exercise 2.

4. **Accelerometer tilt.** Tilt the board at 0, 15, 30 and 45 degrees using the book and the protractor. At each angle, compute the pitch from the accelerometer with the atan2 formula. Record the reading, the protractor angle and the error.

5. **Complementary filter.** Implement the filter with alpha = 0.98 and a loop rate of 100 Hz. Tilt the board to 30 degrees and hold it still for 20 seconds. Record the accelerometer-only angle and the filter angle. Then rotate the board quickly to 60 degrees and back, and record how the filter follows the motion compared with the accelerometer.

6. **Magnetometer heading.** If your breakout includes a magnetometer, mount the board on the turntable and rotate it in 10 degree steps. Record the heading at each step. If the breakout has no magnetometer, use the BNO055 for this exercise, or skip it and state that in the README.

---

## Project: a tilt and heading estimator with a known reference

This project builds a tilt and heading estimator, calibrates it and tests it against a known reference: a protractor for tilt and a marked turntable for heading. The output is a set of measured errors for the accelerometer, the gyroscope and the filtered estimate, which is the honest comparison a robot team needs before choosing a filter.

**Goal:** calibrate one IMU, implement a complementary filter for pitch and roll, and measure its error against a protractor and a turntable at several known angles.

**Repo layout:**

```
100-days-robotics-engineering-day-31-imus/
├── README.md
├── firmware/
│   └── imu_filter/
│       └── imu_filter.ino
├── analysis/
│   ├── tilt_error.py
│   └── requirements.txt
├── data/
│   ├── gyro_bias_still.csv
│   ├── tilt_protractor.csv
│   └── filter_step_response.csv
├── calculations/
│   ├── calibration.md
│   └── filter_parameters.md
├── plots/
│   └── tilt_error_by_angle.png
└── images/
    └── imu_mount.jpg
```

**Starter code:** `firmware/imu_filter/imu_filter.ino`

```cpp
#include <Wire.h>

const uint8_t MPU = 0x68;
const float ALPHA = 0.98f;      // complementary filter weight, tune from your data
const float DT = 0.01f;         // 100 Hz loop, seconds

// Replace these with the values from your own calibration.
float gyroBiasX = 0.0f, gyroBiasY = 0.0f;

float pitch = 0.0f, roll = 0.0f;

// Read two bytes, high byte first. Separate statements keep the order defined.
int16_t readWord() {
  uint8_t hi = Wire.read();
  uint8_t lo = Wire.read();
  return (int16_t)((hi << 8) | lo);
}

void readRaw(int16_t &ax, int16_t &ay, int16_t &az, int16_t &gx, int16_t &gy, int16_t &gz) {
  Wire.beginTransmission(MPU);
  Wire.write(0x3B);
  Wire.endTransmission(false);
  Wire.requestFrom(MPU, (uint8_t)14, (uint8_t)true);
  ax = readWord();
  ay = readWord();
  az = readWord();
  readWord();                               // skip temperature
  gx = readWord();
  gy = readWord();
  gz = readWord();
}

void setup() {
  Serial.begin(115200);
  Wire.begin();
  Wire.beginTransmission(MPU);
  Wire.write(0x6B);                         // PWR_MGMT_1
  Wire.write(0);                            // wake the sensor
  Wire.endTransmission(true);
  delay(100);
  Serial.println("ms,accel_pitch,accel_roll,gyro_pitch,filt_pitch,filt_roll");
}

void loop() {
  static unsigned long last = 0;
  if (millis() - last < (unsigned long)(DT * 1000)) return;
  last = millis();

  int16_t ax, ay, az, gx, gy, gz;
  readRaw(ax, ay, az, gx, gy, gz);

  float axg = ax / 16384.0f, ayg = ay / 16384.0f, azg = az / 16384.0f;
  float gxd = gx / 131.0f - gyroBiasX;      // deg/s, bias removed
  float gyd = gy / 131.0f - gyroBiasY;

  // Tilt from gravity (valid when the board is not accelerating).
  float accelPitch = atan2f(-axg, sqrtf(ayg * ayg + azg * azg)) * 57.2958f;
  float accelRoll  = atan2f(ayg, azg) * 57.2958f;

  // Gyro integration and complementary filter.
  float gyroPitch = pitch + gyd * DT;
  float gyroRoll  = roll + gxd * DT;
  pitch = ALPHA * gyroPitch + (1.0f - ALPHA) * accelPitch;
  roll  = ALPHA * gyroRoll  + (1.0f - ALPHA) * accelRoll;

  Serial.print(last);  Serial.print(",");
  Serial.print(accelPitch); Serial.print(",");
  Serial.print(accelRoll);  Serial.print(",");
  Serial.print(gyroPitch);  Serial.print(",");
  Serial.print(pitch);      Serial.print(",");
  Serial.println(roll);
}
```

The starter code wakes the MPU-6050, reads the accelerometer and gyro registers, converts them to physical units and fuses them into pitch and roll with the complementary filter. It prints the accelerometer-only angle, the gyro-only angle and the filtered angle, so that you can compare the three on the same run. The gyro bias starts at zero, and you must replace it with your measured value from the calibration.

Note the conversion from radians to degrees, 57.2958, and the register addresses 0x6B and 0x3B. The gyro-only value is the integral of the bias-corrected rate, and it will drift. The accelerometer value is accurate only when the board is still or moving slowly. The filtered value should follow the protractor at rest and track the motion when the board moves.

The analysis script `analysis/tilt_error.py` reads the protractor data, computes the error for each sensor at each angle, and produces the plot. Write the script as part of the project, using the same structure as the earlier analysis scripts.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the breakout and its voltage, the mounting orientation of each axis, and the wiring diagram.
2. Calibrate the gyro bias using the 60-second still test after the warm-up. Record the bias in `data/gyro_bias_still.csv` and the calibration result in `calculations/calibration.md`.
3. Calibrate the accelerometer with the six-position method, or at least with the flat and the upside-down positions, and record the scale and offset for Z. State which positions you used.
4. Take the tilt readings at 0, 15, 30, 45 and 60 degrees against the protractor, and record the accelerometer, gyro-only and filtered values in `data/tilt_protractor.csv`.
5. Run the step response test by rotating the board quickly from 0 to 60 degrees and back, and log the filter's response in `data/filter_step_response.csv`. Use the response to justify your alpha in `calculations/filter_parameters.md`.
6. Complete `analysis/tilt_error.py` and produce `plots/tilt_error_by_angle.png`, with one line for each of the three estimates.

**README checklist:**

- [ ] Goal, breakout model, its datasheet sensitivity and the voltage used
- [ ] Mounting orientation of each axis and the frame convention
- [ ] Gyro bias and noise from the still test
- [ ] Accelerometer calibration positions and the resulting offsets
- [ ] Tilt error table for the three estimates at each protractor angle
- [ ] Filter parameters and the step response, with the reason for alpha
- [ ] One limit of the estimate, such as the drift over five minutes or the accelerometer error during motion

**Done when:**

- The gyro bias is measured with 60 seconds of data, and the bias is subtracted in the firmware.
- The filtered tilt is within 2 degrees of the protractor at 0, 15, 30, 45 and 60 degrees when the board is still.
- The step response shows the filter following the 60-degree move within a 0.5-second lag, or the README explains why not.
- The README states the drift of the gyro-only estimate in degrees per minute, and shows that the filter removes it.

**Evidence to commit to GitHub:** the firmware, the analysis script, the raw data files, the calibration notes, the plot, the photo and the README. A reviewer should be able to recompute the errors from the raw files and see the same numbers.

The IMU is a small, cheap board with no hazard of its own. The hazard is the robot it may be mounted on, so keep any motors disconnected while you calibrate, and keep the board away from the motors' magnetic field during the magnetometer exercise.

---

## Check your understanding

1. An accelerometer on a table reads ax = 0, ay = 0, az = 0.5 g. Calculate the pitch and roll, and decide whether the board is tilted or accelerating. Name the test that tells the two apart.
2. A gyro has a bias of 0.2 degrees per second and no other error. Calculate the heading error after 10 minutes of integration. Then explain why calibrating the bias at rest does not remove the random noise.
3. A robot accelerates forward at 0.8 m/s² while its accelerometer is read for pitch. Calculate the extra pitch error, assuming the board is level and the acceleration is along the X axis. Explain why the error does not appear in the gyro-only estimate.
4. A complementary filter uses alpha = 0.95 at a loop rate of 50 Hz. Calculate the time constant. Decide whether the filter responds quickly enough for a robot that tips over in half a second, and explain your answer.
5. A magnetometer on a robot reads a heading that changes by 20 degrees each time the drive motor starts. Name the most likely cause and the fix that is most likely to work, and state one test that would confirm the cause.
6. A board is mounted with its X axis pointing down. Explain what the accelerometer reads on each axis at rest, and state the change to the pitch and roll formulas, if any, needed to get the correct tilt from this mounting.

---

### Answers

1. Pitch is atan2(0, 0.5) = 0 degrees and roll is atan2(0, 0.5) = 0 degrees, so the tilt formulas say the board is level. That answer is not trustworthy here. A board at rest measures a total magnitude of 1 g, because gravity is the only force on it. The vector (0, 0, 0.5 g) has a magnitude of 0.5 g, so the board is not at rest. It is accelerating (a net specific force of half of gravity can come from a downward acceleration of about 0.5 g), or the sensor is badly miscalibrated or faulty. The test is to hold the board still on a table and check that the magnitude is 1 g. A board that reads 1 g at rest with a level Z axis and then shows 0.5 g during a motion has an acceleration problem, not a tilt problem.
2. The heading error after 10 minutes is 0.2 times 600 = 120 degrees. A bias is a constant offset, and calibration at rest removes it. Random noise is a varying error with zero mean, which calibration cannot remove because it changes from sample to sample. Its effect on the integrated angle is a random walk, which grows with the square root of time, so the noise is limited by the sensor's noise density and by filtering the accelerometer, not by calibration.
3. The accelerometer reads gravity plus the robot's acceleration along the X axis. The extra reading is 0.8 / 9.81 = 0.082 g. The pitch error is atan2(0.082, 1) ≈ 4.7 degrees. The gyro-only estimate does not see the acceleration, because it measures rotation, not linear acceleration, so it is not affected by the forward push.
4. The time constant is tau = 0.95 times 0.02 / 0.05 = 0.38 s. The filter's response to a 0.5 s tip is partly within the time constant, so the estimate lags by a significant fraction of the event. For an emergency tip-over detection, the gyro's rate should be used directly, because the filter's accelerometer correction is too slow for that event.
5. The most likely cause is the magnetic field of the motor, which changes when the current starts, and it adds an offset to the magnetometer. The fix is to move the magnetometer away from the motor and its wires, and to calibrate the hard iron offset with the motor running. A test is to run the motor without moving the robot, and see whether the heading changes with the current, which would confirm the field from the motor.
6. With the X axis pointing down, the accelerometer reads about -1 g on X at rest, and about 0 g on Y and Z. The pitch formula atan2(-ax, sqrt(ay^2 + az^2)) gives -90 or 90 degrees, which is the tilt of the X axis, not the robot's pitch. The formulas must be remapped to the mounted frame. The simplest fix is to re-mount the board with the Z axis up or to apply a fixed rotation matrix in the firmware before computing the tilt.

---

## Sources

*Read*

- [Wikipedia: Inertial measurement unit](https://en.wikipedia.org/wiki/Inertial_measurement_unit) for the sensor package and the applications.
- [Wikipedia: Accelerometer](https://en.wikipedia.org/wiki/Accelerometer) for the principle and the specific force.
- [Wikipedia: Gyroscope](https://en.wikipedia.org/wiki/Gyroscope) for the principle of angular rate sensing, the MEMS section and the history.
- [Wikipedia: Magnetometer](https://en.wikipedia.org/wiki/Magnetometer) for the magnetic sensing principle.
- [Wikipedia: Complementary filter](https://en.wikipedia.org/wiki/Complementary_filter) for the filter structure used in the project.
- [Wikipedia: Kalman filter](https://en.wikipedia.org/wiki/Kalman_filter) for the more general estimator that follows from the complementary filter.
- The MPU-6050 product specification and register map from InvenSense, now TDK. Search "MPU-6050 register map datasheet" for the register addresses, the sensitivity and the temperature formula.
- The ROS REP-103 standard on the ROS documentation site, which defines the axis convention. Search "REP-103 ROS coordinate conventions".

*Tutorials*

- Adafruit Learning System, the guides on the MPU-6050 and on the IMU breakouts. Search "Adafruit MPU6050 guide".
- SparkFun Learn, the tutorial on accelerometers and gyroscopes. Search "SparkFun Learn accelerometer gyroscope".
- GeeksforGeeks, the article on the working principle of gyroscopes and accelerometers. Search "GeeksforGeeks gyroscope working principle".
- NOAA National Centers for Environmental Information, the magnetic declination calculator for your location. Search "NCEI magnetic field calculator".

*Watch*

- Cyrill Stachniss, the lecture series on state estimation and sensor fusion for robots on YouTube. Search "Stachniss sensor fusion".
- MIT OpenCourseWare, 2.12 Introduction to Robotics, the lectures on inertial sensing and orientation. Search the course title on ocw.mit.edu.

*Practice*

- Wokwi simulator at wokwi.com, which includes an MPU-6050 model on the I2C bus. Simulate the register reads and check the conversion formulas before you test on hardware.

---

## Tomorrow

Tomorrow you will look at the sensor that sees the world in the widest detail: the camera. Day 32 covers RGB cameras, stereo and depth concepts, resolution, frame rate, exposure, field of view, calibration and the camera frame. A camera can correct the drift that you measured today, if it can see a known feature, and that is the reason robots that use cameras for localization still keep the IMU and the encoder.

**Next:** [Day 32: Cameras](Robotics-Day-32-Cameras.md)
