# Day 30: Distance Sensors

*100 Days of Robotics Engineering, Module 4: Sensors and Actuators*

**Time:** about 4 hours
**Level:** intermediate. You need Day 29 (encoders and the idea of a reference measurement), Day 13 (logic levels and the voltage differences between boards) and Module 3 (reading a digital pin and a serial port). Today you give the robot a way to see the distance to the things around it.

**By the end of this day, you will have:**

- An ultrasonic range reading from an HC-SR04 converted to centimetres from the echo time, with a median filter over five samples
- A time-of-flight reading from a VL53L1X over I2C, with its ranging mode set and its timing budget stated
- A measured error table for two sensors at five distances and three surfaces (white paper, black cloth and a glass sheet)
- A written field-of-view and beam-width calculation for each sensor, and a short note on which sensor fits which robot task

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3 or newer, about $25 for an official board or $10 to $15 for a clone
- One HC-SR04 ultrasonic module, about $2 to $4. Its echo pin outputs 5 V, so use a level shifter or a voltage divider if you connect it to a 3.3 V board.
- One VL53L1X time-of-flight breakout, about $10 to $20. The breakout is usually sold with a 3.3 V regulator and a 5 V-tolerant I2C interface, so check the breakout's logic level before you wire it to a 5 V board.
- One Sharp GP2Y0A21YK0F infrared distance sensor, about $10 to $15, optional for the IR section and for the comparison table
- A ruler or a tape measure, a 1 m stick and a sheet of white paper, a sheet of black cloth and a clean glass pane or a sheet of clear acrylic, all cheap
- Adafruit VL53L1X library, free and open source, installed through the Arduino Library Manager
- Python 3.12 with NumPy 2.1 or newer and Matplotlib 3.9 or newer, both free
- Optional: an RPLIDAR A1 2D laser scanner from Slamtec, which costs about $100 to $120 at the time of writing. The lesson explains it, but the exercises use the cheap sensors. Check the price and the availability before you buy.

**Prerequisites:**

- Day 29: a reference measurement, repeated runs and the difference between accuracy and resolution
- Day 13: logic levels, pull-up resistors and the danger of connecting a 5 V output to a 3.3 V input
- Module 3: reading a digital pin with pulseIn() and reading a serial port
- Basic trigonometry: sine, cosine and tangent of an angle in a right triangle

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How distance sensing developed](#how-distance-sensing-developed)
- [Glossary](#glossary)
- [Measuring distance by time of flight](#measuring-distance-by-time-of-flight)
- [Ultrasonic sensors](#ultrasonic-sensors)
- [Infrared triangulation](#infrared-triangulation)
- [Time-of-flight sensors](#time-of-flight-sensors)
- [LiDAR](#lidar)
- [Range, field of view and resolution](#range-field-of-view-and-resolution)
- [Noise and filtering](#noise-and-filtering)
- [Surfaces and failure modes](#surfaces-and-failure-modes)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a range error study for three sensors](#project-a-range-error-study-for-three-sensors)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot that cannot measure distance walks into walls. Encoders from Day 29 tell the robot how far its wheels have turned, but they say nothing about what is in front of it. A distance sensor fills that gap. It measures the gap between the sensor and the nearest surface, and that measurement is the basis of obstacle avoidance, docking, mapping and most of the localization a mobile robot does.

No single sensor is right for every task. An ultrasonic sensor is cheap and wide, but it is slow and easily fooled by soft surfaces. A time-of-flight sensor is fast and precise over a short range, but its beam is narrow. A LiDAR scans a plane with many points per second, but it costs far more. Today you measure each type under the same conditions, so that the choice in your own robot rests on numbers you collected yourself.

---

## How distance sensing developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1826 to 1880s | Sound and light are measured as waves and their speeds are determined, with the speed of sound measured precisely in the 19th century. | The speed of sound is the constant that sets the ultrasonic formula you use today. |
| 1910s to 1940s | Sonar is developed for ship detection and submarine tracking, using echoes from underwater sound. | The echo timing principle behind ultrasonic sensors dates from this work. |
| 1960s | The laser is invented, and lasers are used for ranging to the Moon in the 1960s with retroreflectors left by Apollo missions. | Laser ranging is the ancestor of the LiDAR used on robots today. |
| 1970s to 1990s | Infrared triangulation sensors become cheap for consumer products, and ultrasonic modules become common in cameras and cars. | Infrared triangulation is still used in many low-cost robot vacuums and toys. |
| 2000s to today | Solid-state time-of-flight sensors, low-cost ultrasonic modules and compact LiDARs enter robots, cars and phones. | A student can now buy a range sensor for a few dollars and a laser scanner for about $100. |

The physics of sound, light and time has not changed. What changed is the size and cost of the electronics that time the echo and convert it into a number.

---

## Glossary

- **Range:** the distance from the sensor to a target, along the sensor's axis or along the beam.
- **Time of flight (ToF):** the time a signal takes to travel to a target and back.
- **Ultrasonic sensor:** a range sensor that sends a sound pulse above the human hearing range, usually at 40 kHz, and times the echo.
- **Triangulation:** a method that finds the distance from the angle at which a reflected spot appears on a detector.
- **LiDAR:** light detection and ranging, a sensor that measures distance with a laser, often scanning it across an angle.
- **Field of view (FOV):** the angular range over which the sensor responds.
- **Beam width:** the width of the sensed area at a given distance, which grows with distance for a cone-shaped beam.
- **Resolution:** the smallest change in distance the sensor can report. Resolution is a property of the electronics and the sampling, and it says nothing about accuracy.
- **Accuracy:** how close a reading is to the true distance on average. A sensor can be precise and inaccurate, if it repeats a wrong number.
- **Precision (repeatability):** how much repeated readings of the same distance vary, measured as a standard deviation or a spread.
- **Median filter:** a filter that replaces each reading with the median of a small window of recent readings, which removes isolated outliers.
- **Multipath:** an effect in which a signal reaches the sensor by a path other than the direct one, which gives a reading that is too long.
- **Retroreflector:** a surface that reflects light back along its incoming direction, used to make ranging strong and reliable.

---

## Measuring distance by time of flight

Most range sensors measure how long a signal takes to go out and come back. The distance is then half the round-trip time multiplied by the signal's speed. The factor of one half is the most common error in the first sensor project, because it is easy to forget.

The first principle is d = v times t / 2, where v is the signal's speed and t is the round-trip time. For sound in air at 20 degrees Celsius, v is about 343 m/s. For light, v is about 3 times 10^8 m/s. An echo that returns after 5.8 ms from a sound pulse has travelled 343 times 0.0058 = 1.99 m, and the target is at 1.99 / 2 = 0.995 m, which is about 1 m.

The worked example compares the two speeds. A laser ranging system that measures a 10 m target has a round trip of 2 times 10 / (3 times 10^8) = 67 nanoseconds. A sound pulse for the same target takes 2 times 10 / 343 = 58 milliseconds. The laser needs a timer that resolves nanoseconds, and the ultrasonic sensor needs a timer that resolves microseconds. That difference is why cheap time-of-flight sensors use phase measurement or other tricks, which the time-of-flight section explains.

The common mistake is to use the speed of sound at the wrong temperature. The speed rises by about 0.6 m/s for every degree Celsius, so a sensor used at 35 degrees, for example in a hot warehouse, reads about 2.6 percent short if it assumes 20 degrees. For an ultrasonic target at 3 m, that is about 8 cm of error, which matters for a docking robot.

In the market, time-of-flight is the common principle in most range sensors, and the differences lie in the signal, the speed of the wave and the electronics that time it.

---

## Ultrasonic sensors

An ultrasonic sensor sends a short burst of sound at about 40 kHz and listens for the echo. The HC-SR04 is the standard hobby module. It has a trigger pin and an echo pin. The Arduino sends a trigger pulse of about 10 microseconds, the module sends eight bursts of sound, and the echo pin goes high for the time the echo takes to return.

The first principle is the timing of the echo pulse. The echo pin's high time is the round-trip time, so the distance in centimetres is pulse_us times 0.0343 divided by 2, which is pulse_us times 0.01715. At 20 degrees Celsius, a 2 ms echo gives 2000 times 0.01715 = 34.3 cm.

The worked example uses the HC-SR04 in code. The `pulseIn()` function returns the echo's high time in microseconds, and a timeout of 30 milliseconds stops the wait when there is no echo. That timeout corresponds to a range of about 5 m. The code below is the core of a reading:

```cpp
const int TRIG = 9;
const int ECHO = 10;

// Returns distance in centimetres, or -1 if no echo arrives within 30 ms.
float readUltrasonicCm() {
  digitalWrite(TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  unsigned long t = pulseIn(ECHO, HIGH, 30000UL);
  if (t == 0) return -1.0f;
  return t * 0.0343f / 2.0f;    // speed of sound in cm per microsecond, halved for the round trip
}
```

The HC-SR04's datasheet gives a range of about 2 cm to 400 cm and a measuring angle of about 15 degrees. The range is for a flat, hard target facing the sensor. Real targets are smaller, softer or angled, and the usable range is shorter.

The common mistake is to read the sensor too often. The echo from one pulse must return before the next trigger, and at 400 cm the round trip takes about 23 ms. A loop that triggers every 10 ms gets echoes from the previous pulse. Wait at least 60 ms between readings if you want the full range, or use a shorter timeout.

Ultrasonic sensors have clear limits. Soft materials such as foam and cloth absorb sound and give no echo. A surface at an angle reflects the sound away from the sensor. Several sensors at the same frequency can interfere, and the wind or a strong noise source can cause false echoes. In the market, ultrasonic sensors are used for parking aids, tank level measurement and low-cost obstacle detection on robots, where the price matters more than the precision.

---

## Infrared triangulation

An infrared distance sensor, such as the Sharp GP2Y0A21YK0F, uses triangulation instead of timing. It emits a small beam of infrared light and a position-sensitive detector sees the reflected spot. The position of the spot on the detector depends on the distance to the target, and the sensor's electronics convert that position into a voltage.

The first principle is similar triangles. The emitter and the detector are separated by a baseline b. The reflected spot is at a distance x on the detector, and the focal length of the lens is f. The target distance is D = b times f / x, which means the distance falls as the spot moves across the detector. Because the spot moves less at long range, the output is nonlinear: a large change in distance at close range gives a large change in voltage, and a small change at far range gives a very small change.

The worked example uses the datasheet's typical output. The GP2Y0A21YK0F is rated for about 10 cm to 80 cm, and its analog output falls from about 3 V at 10 cm to about 0.4 V at 80 cm. Between 20 cm and 40 cm, the voltage changes by roughly 1 V, which gives a steep slope. Between 60 cm and 80 cm, the same 1 V change would cover a much larger distance, so a resolution in centimetres at long range is poor. The best practice is to fit a curve, such as a power law of the form distance = a times voltage to the power of minus b, from your own calibration data, and to use it only within its calibrated range.

The common mistake is to read the output as a linear function of distance. A linear fit gives errors of several centimetres across the range. The second mistake is to ignore the target's reflectivity. A black surface reflects less light, and the output drops, so the sensor reads a longer distance than the truth. The third is to use the sensor in sunlight, where the sunlight's infrared content swamps the signal.

The IR sensor is useful for close obstacle detection and for measuring the reflectance of a surface, which is the basis of many line sensors. It is a short-range sensor, and its reading depends on surface colour, which an ultrasonic reading does not.

In the market, infrared triangulation is common in cheap robot vacuums, cliff detection and the proximity sensors built into consumer devices.

---

## Time-of-flight sensors

A time-of-flight sensor in the modern sense is a chip that emits a short light pulse or a modulated light signal and measures the delay with a fast timer or a phase detector. The VL53L1X from STMicroelectronics is a common breakout. It uses a laser diode at near-infrared wavelength and a single-photon avalanche diode array, and it reports the distance over I2C.

The first principle is the timing of light. Light travels about 0.3 m per nanosecond. A 1 m target is a 6.7 ns round trip, which a chip cannot time directly with a typical clock. Modern ToF chips therefore use a histogram of photon arrival times: they fire many pulses, count the arrival times in bins, and find the peak, which gives the distance with a resolution much finer than the bin width. The chip's timing budget, which sets how long the measurement runs, controls how many photons are collected, and so it sets the noise.

The worked example sets the timing budget. The VL53L1X's datasheet gives longer timing budgets for better accuracy. A budget of 100 ms gives lower noise than a budget of 20 ms, but it gives a slower update rate. For a robot that must update at 10 Hz, a budget of 50 ms or less may be the best compromise. The Adafruit library sets the budget with setTimingBudget(), and the chosen value must be less than the inter-measurement period, which setIntermeasurementPeriod() controls.

```cpp
#include <Wire.h>
#include <Adafruit_VL53L1X.h>

Adafruit_VL53L1X tof = Adafruit_VL53L1X();

void setup() {
  Serial.begin(115200);
  Wire.begin();
  if (!tof.begin(0x29, &Wire)) {
    Serial.println("VL53L1X not found");
    while (1) delay(10);
  }
  tof.startRanging();
  tof.setTimingBudget(50);              // milliseconds
  tof.setIntermeasurementPeriod(60);    // must be >= the timing budget
}

void loop() {
  if (tof.dataReady()) {
    int16_t mm = tof.distance();
    tof.clearInterrupt();
    if (mm < 0) Serial.println("out of range");
    else Serial.println(mm);
  }
}
```

The common mistake is to ignore the field of view. The VL53L1X's beam is narrow, and a target that sits to one side of the beam is not seen, even when it is close. The sensor reports the distance to whatever is inside its cone, which may be a far wall if the nearest object is outside the cone. This is the opposite of the ultrasonic problem: the ultrasonic cone is wide and may see a wall the robot does not want to hit, while the ToF beam may miss an object it should avoid.

A second mistake is to use the sensor behind glass or through a window, where the reflection from the glass may dominate or the light may pass through. Glass is one of the hardest surfaces for every range sensor in this lesson.

In the market, time-of-flight sensors are used in phones for autofocus, in robot vacuums for cliff and obstacle detection, and in gesture sensors. They are the most common short-range sensor in consumer robots.

---

## LiDAR

A LiDAR sends laser pulses and receives the reflections, measuring the distance to each point. A scanning LiDAR rotates the beam, so it measures a plane or a volume with many points each second. A 2D LiDAR, such as the RPLIDAR A1, scans one plane, and it returns the distance and the angle for each point. That output is the input to most mobile-robot mapping systems.

The first principle is the same as time of flight: distance is the speed of light times the round-trip time divided by two. The difference is the scan. A LiDAR points its beam at a known angle at each instant, so the set of distances at all angles forms a map of the room's edges. A common 2D LiDAR has an angular resolution of about 1 degree, and it produces several thousand points per second.

The worked example is the point spacing. At a range of 5 m, a 1-degree angular step spaces adjacent points by 5 times sin(1 degree), which is 5 times 0.01745, or 87 mm. A wall 5 m away is therefore sampled every 87 mm, so a 20 mm post between two samples can be missed. At 2 m, the spacing falls to 35 mm. The point spacing is the resolution of the map, not the accuracy of each point, and it limits the smallest feature the scanner can see.

The common mistake is to use the LiDAR's maximum range without checking its accuracy at that range. Most low-cost 2D LiDARs have a rated range of 6 to 12 m, but their accuracy degrades with distance and with the surface's reflectivity. The datasheet gives the accuracy at a specified range and surface, and the number you need for a map is the accuracy at the distance you use.

LiDAR also has a field-of-view problem. A 360-degree scanner has no blind spot in its plane, but it sees only that plane. A table top at the height of the scanner is visible, and the table legs are not, unless the scanner is mounted at a different height. Robot builders often mount the LiDAR too high or too low, and the map then misses the furniture the robot must avoid.

In the market, 2D LiDAR is the standard sensor for warehouse robots, lawn mowers and many service robots. Its cost has fallen sharply, and the pair of a LiDAR and an encoder-based odometry is the standard starting point for SLAM.

---

## Range, field of view and resolution

Three numbers describe a range sensor in its datasheet and in its use: the range, the field of view and the resolution. Each one answers a different question, and a robot needs all three for each task.

The first principle is beam geometry. A cone-shaped beam with a full angle theta spreads to a width of w = 2 times d times tan(theta / 2) at a distance d. The ultrasonic HC-SR04 has a measuring angle of about 15 degrees, which is a full angle of about 30 degrees if the datasheet gives the half-angle. The width at 1 m is 2 times 1 times tan(15 degrees), which is 0.54 m. A chair leg 5 cm wide at 1 m is inside that cone, but so is the wall behind it, and the sensor returns the nearest echo, which may be from either.

The worked example uses the VL53L1X's narrow cone. If its field of view is about 27 degrees, the width at 1 m is 2 times 1 times tan(13.5 degrees), about 0.48 m, which is similar to the ultrasonic cone at that range. The difference is at close range. At 0.3 m the ToF beam is 0.14 m wide, and the ultrasonic beam is 0.16 m wide. The two cones look similar on paper, but the ultrasonic echo is returned from a wider region of the cone, which is why the two sensors disagree near the edges of a surface.

Resolution is a different number. The VL53L1X reports in whole millimetres, and an ultrasonic reading in centimetres or in tenths of a centimetre is limited by the timer's resolution and the speed of sound. Resolution is the smallest step the output can take. It does not mean the reading is accurate to that step.

The common mistake is to read the resolution as accuracy. A sensor reporting 1 mm steps may be accurate only to 10 mm, and its repeat spread across ten readings of the same target may be 3 mm. The error study in the project measures both, separately, so you can state each one.

In the market, the three numbers are the first filter in a sensor selection document. A robot that must detect a 2 cm post at 2 m needs a sensor whose beam and resolution allow that detection, and a sensor whose datasheet range is 4 m may still fail that test.

---

## Noise and filtering

Every range sensor reads a slightly different value each time it measures the same target. The variation is noise, and it comes from the electronics, the timing, the light and sound levels and the target's surface. Noise is a property that a robot must handle, and the standard tool is a filter.

The first principle is the median filter. A median of five consecutive readings takes the middle value after sorting, and it ignores a single outlier completely. A moving average of five readings would be pulled by the outlier. The median is the better choice for range sensors, because their noise often includes occasional false echoes that are far from the true value.

The worked example applies a median filter to five readings. Suppose the readings are 52, 53, 51, 180 and 52 centimetres. The sorted order is 51, 52, 52, 53, 180, and the median is 52. The average would be 77.6 cm, which is wrong by 25 cm. The median gives 52 cm, which is close to the true value. A filter with a window of five adds about two samples of delay, which is acceptable for a robot moving at walking speed, but it may be too slow for a fast robot.

The common mistake is to filter until the signal looks smooth, and to forget that the filter adds lag. A robot that filters a distance with a long window reacts late to an obstacle, and it may hit it. The fix is to measure the lag and the response time, and to choose the window from the robot's speed and stopping distance.

Averaging is also useful when the noise is random and not outlier-driven. Averaging N independent readings reduces the standard deviation by about the square root of N. Ten readings give a spread about three times smaller, if the noise is truly random and the target does not move.

In the market, the filter is part of the sensor's driver, and the datasheets of the better sensors describe their internal filters. Most robot stacks apply a filter to range data before it reaches the planner.

---

## Surfaces and failure modes

Every range sensor in this lesson works on some surfaces and fails on others, and the failure is not always an error message. A sensor can give a confident wrong number, which is more dangerous than no number. Testing the sensor on the surfaces the robot will meet is the cheapest protection against that.

The first principle is reflection. A sensor needs a return signal from the target, and the return depends on the surface's absorption, its angle and its smoothness. A white matte surface returns sound and light well. A black surface absorbs infrared and visible light, so an IR sensor reads low and a ToF sensor may lose the target. A sound-absorbing surface such as foam returns little sound, so an ultrasonic sensor may read nothing. A smooth surface at an angle reflects the signal away, so the sensor sees no target at all. A glass sheet transmits light and reflects a little, which can produce a false reading of a distance behind the glass or a reading of the glass itself.

The worked example is the three-surface test in the project. For white paper, the expected result is a clean reading with low spread. For black cloth, the IR and ToF sensors may show a longer distance or no reading, while the ultrasonic sensor may still work if the cloth is not sound-absorbing. For glass, the ToF sensor may read the glass correctly at short range, and the ultrasonic sensor may read the glass or the wall behind it. Write the expected result before you test, and then compare.

Multipath is another failure. An ultrasonic echo can bounce off one wall, then another, and return later than the direct echo, which gives a distance that is too long. The fix is to test in the room where the robot will work, and to compare the reading with a ruler.

The common mistake is to test on one surface only and to conclude that the sensor works. A sensor that reads the white wall correctly may read the dark sofa wrongly, and the robot will hit the sofa. Test on the surfaces the robot will meet, and report each failure.

In the market, sensor failures on dark or glossy surfaces are a common cause of field complaints. Vendors publish reflectance and angle curves for this reason, and a robot team reads them before choosing the sensor.

---

## Hands-on exercises

The HC-SR04 and the VL53L1X both run from 5 V or 3.3 V, depending on the breakout. Check the breakout's voltage before you connect it, and connect the grounds of all sensors to the Arduino's ground. Use only a 5 V supply for the HC-SR04, and do not connect its echo pin directly to a 3.3 V board.

1. **Measure the ultrasonic speed.** Place a flat board at a known distance of 50 cm from the HC-SR04. Upload the core function above and print the distance every 100 ms for 10 seconds. Record the average and the spread. Compare the average with 50 cm and state the error.

2. **Add a median filter.** Modify the sketch to keep the last five readings in an array, sort a copy, and print the median. Repeat the test from exercise 1 and compare the spread with the unfiltered spread. State the lag you observe when you move the board suddenly.

3. **Time-of-flight range check.** Set up the VL53L1X with a timing budget of 50 ms. Measure the distance to a flat board at 20 cm, 50 cm, 100 cm, 200 cm and 300 cm, with 30 readings at each distance. Record the mean, the standard deviation and the error from the ruler.

4. **Surface test.** At 50 cm, test the HC-SR04 and the VL53L1X on white paper, black cloth and a glass sheet. Record 20 readings for each sensor and each surface. Note any reading that is missing or far from the expected value, and state what you think caused it.

5. **Field-of-view check.** Place a narrow object, such as a 2 cm dowel, at 50 cm and move it sideways from the sensor's axis in 2 cm steps. Record the sideways distance at which each sensor stops seeing the dowel. Compare with the beam width you calculated from the datasheet's field of view.

6. **Optional IR curve.** If you have the Sharp GP2Y0A21YK0F, measure its output voltage with the Arduino's ADC at 10, 20, 40, 60 and 80 cm against a white target. Fit a power-law curve in Python and state the range over which the fit is within 5 percent.

---

## Project: a range error study for three sensors

This project compares the error, precision and field of view of two or three range sensors under the same conditions. The output is a table and a plot that shows which sensor fits which task, backed by your own measurements. It is a practical study, and an honest result is more useful than a sensor that looks good on paper.

**Goal:** measure the accuracy, precision and usable field of view of the HC-SR04 and the VL53L1X, and of the IR sensor if you have it, on three surfaces at five distances, and write a sensor selection note for a small mobile robot.

**Repo layout:**

```
100-days-robotics-engineering-day-30-distance-sensors/
├── README.md
├── firmware/
│   └── range_logger/
│       └── range_logger.ino
├── analysis/
│   ├── error_study.py
│   └── requirements.txt
├── data/
│   ├── ultrasonic_runs.csv
│   ├── tof_runs.csv
│   └── fov_check.csv
├── calculations/
│   ├── beam_width.md
│   └── sensor_selection.md
├── plots/
│   └── error_by_distance.png
└── images/
    └── test_setup.jpg
```

**Starter code:** `firmware/range_logger/range_logger.ino`

```cpp
#include <Wire.h>
#include <Adafruit_VL53L1X.h>

const int TRIG = 9;
const int ECHO = 10;
Adafruit_VL53L1X tof = Adafruit_VL53L1X();

float readUltrasonicCm() {
  digitalWrite(TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  unsigned long t = pulseIn(ECHO, HIGH, 30000UL);
  if (t == 0) return -1.0f;
  return t * 0.0343f / 2.0f;
}

void setup() {
  Serial.begin(115200);
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  Wire.begin();
  if (!tof.begin(0x29, &Wire)) {
    Serial.println("VL53L1X not found");
    while (1) delay(10);
  }
  tof.startRanging();
  tof.setTimingBudget(50);
  tof.setIntermeasurementPeriod(60);
  Serial.println("ms,ultrasonic_cm,tof_mm");
}

void loop() {
  static unsigned long last = 0;
  if (millis() - last < 100) return;
  last = millis();

  float us = readUltrasonicCm();
  int tofMm = -1;
  if (tof.dataReady()) {
    tofMm = tof.distance();
    tof.clearInterrupt();
  }
  Serial.print(last);
  Serial.print(",");
  Serial.print(us);
  Serial.print(",");
  Serial.println(tofMm);
}
```

The logger prints one line every 100 milliseconds with the time, the ultrasonic distance in centimetres and the ToF distance in millimetres. A value of -1 means no reading. The ToF reading is taken only when a new measurement is ready, so an old value is never repeated. The ultrasonic reading is taken on every cycle, which is fine for a study at a fixed target.

The analysis script `analysis/error_study.py` reads each run's CSV, computes the mean, the standard deviation and the error against the ruler distance, and writes the summary. Write it as part of the project, following the same structure as the odometry script in Day 29.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the sensor models, their datasheet ranges and fields of view, and the wiring.
2. Set up the rig. Mount the sensors on the same bracket, with their axes parallel and their centres 10 mm apart, and mark the true distance from each sensor to the target face.
3. Run the logger at 20, 50, 100, 200 and 300 cm for each of the three surfaces, collecting 50 readings per run. Save the runs in `data/` with the surface and the distance in the file name.
4. Run the field-of-view check with the dowel and record the sideways distance at which each sensor stops seeing it in `data/fov_check.csv`.
5. Complete `analysis/error_study.py` and produce `plots/error_by_distance.png`, with one line for each sensor and error bars showing the standard deviation.
6. Write `calculations/beam_width.md` with the beam width at each distance from the datasheet field of view, and `calculations/sensor_selection.md` with a recommendation for a robot that must avoid a 5 cm obstacle at 1 m on a floor with mixed surfaces.

**README checklist:**

- [ ] Goal, sensor models, datasheet range and field of view for each
- [ ] Wiring diagram with the level shifter or divider for the HC-SR04 if used
- [ ] Error table at each distance, with the mean, the standard deviation and the error
- [ ] Surface table with the missing readings and the observed cause
- [ ] Field-of-view result compared with the calculated beam width
- [ ] Sensor selection note with a recommendation and a reason
- [ ] One limit of each sensor, stated with a number from your data

**Done when:**

- Each sensor has 50 readings at each of the five distances on the white surface, and the error is reported in millimetres.
- The standard deviation at 1 m is reported for each sensor, and the README states which sensor is more precise at that distance.
- The field-of-view result for each sensor is within 20 percent of the calculated beam width at 50 cm, or the README explains the difference.
- The selection note names one sensor for the stated task and gives a number from the study that supports the choice.

**Evidence to commit to GitHub:** the firmware, the analysis script, the raw run files, the field-of-view check, the calculations, the plot, the photo and the README. A reviewer should be able to rerun the analysis on the raw files and reproduce the table.

The rig is light and safe, but the HC-SR04 and the VL53L1X are only sensors. The hazard is the bench supply, so use a USB supply or a regulated 5 V supply with a fuse.

---

## Check your understanding

1. An ultrasonic sensor returns an echo after 11.6 ms at 20 degrees Celsius, and the sensor's code assumes that speed. Calculate the distance. Then calculate the true distance if the air is at 35 degrees Celsius, where the speed of sound is about 352.5 m/s, and state the sign and size of the error.
2. A LiDAR scans at 0.5-degree steps. Calculate the point spacing at 4 m. Decide whether the LiDAR can see a 3 cm post at 4 m, and explain your answer.
3. A time-of-flight sensor reports 1000 mm on 20 readings of a target at 1000 mm, with a standard deviation of 2 mm. Explain whether the sensor is accurate, precise or both, and state what would change if the reading had a bias of 15 mm.
4. A robot uses a median filter with a window of five readings, and its sensor samples at 10 Hz. Calculate the delay the filter adds, and decide whether a robot moving at 0.5 m/s with a stopping distance of 0.2 m can use it safely for obstacle detection.
5. A robot with an IR distance sensor reads 30 cm on a black box that is really 20 cm away. Name the cause, and state one test that confirms it.
6. A robot's ultrasonic sensor sees a wall 40 cm away at one moment, and a chair 30 cm away at the next, although the chair has not moved. Name two causes and one test that would separate them.

---

### Answers

1. The echo is 11.6 ms, which is 11,600 microseconds. The distance is 11,600 times 0.0343 divided by 2, which is about 199 cm, or 1.99 m. At 35 degrees, the true speed is about 352.5 m/s, or 0.03525 cm per microsecond, so the true distance is 11,600 times 0.03525 / 2, about 2.04 m. The sensor reads about 5 to 6 cm short, a 2.6 percent error. The error is proportional to the distance, so at 3 m it would be about 8 cm.
2. The point spacing is 4 times sin(0.5 degrees), which is 4 times 0.008727, or 35 mm. A 3 cm post is narrower than the 35 mm spacing, so the LiDAR may place zero or one point on it. Whether it sees the post depends on the beam's alignment with the post, and the answer is not reliable. The robot should not depend on the LiDAR alone for a 3 cm post at 4 m.
3. The sensor is precise, because its standard deviation is 2 mm. Its mean matches the true distance, so in this test it is also accurate. If the mean instead read 1015 mm or 985 mm, the sensor would still be precise but inaccurate, with a 15 mm bias. The bias must be removed by calibration against a known distance.
4. A window of five readings at 10 Hz has a median delay of about two samples, or 0.2 seconds. At 0.5 m/s the robot travels 0.1 m during the delay. The stopping distance of 0.2 m is larger than the 0.1 m delay, so the filter is usable if the rest of the stopping distance is not used up by the robot's own braking. The margin is only 0.1 m, so the designer should check the braking distance and reduce the window if the margin is too small.
5. The black box reflects less infrared light, so the output falls and the sensor reads a longer distance than the truth. A test is to measure the same distance with a white target of the same shape. If the white target reads 20 cm and the black box reads 30 cm, reflectance is the cause.
6. Two causes are a multipath echo that bounced from another surface, or a second object inside the cone that returned the nearest echo. A test is to measure the chair's position with a ruler and to point the sensor at a plain wall with the chair removed. A repeat with the chair at a different angle also separates the cause, because a multipath echo changes with the angle while a real object does not.

---

## Sources

*Read*

- [Wikipedia: Ultrasonic transducer](https://en.wikipedia.org/wiki/Ultrasonic_transducer) for the transducer principle behind ultrasonic range sensors.
- [Wikipedia: Lidar](https://en.wikipedia.org/wiki/Lidar) for the principle, the scanning types and the applications.
- [Wikipedia: Time-of-flight camera](https://en.wikipedia.org/wiki/Time-of-flight_camera) for the principle used by modern time-of-flight sensors.
- [Wikipedia: Triangulation](https://en.wikipedia.org/wiki/Triangulation) for the geometry behind infrared distance sensors. Read the section on active triangulation.
- [Wikipedia: Median filter](https://en.wikipedia.org/wiki/Median_filter) for the filter used in the noise section.
- The HC-SR04 datasheet. Search "HC-SR04 datasheet" for the measuring angle, the range and the timing diagram.
- The STMicroelectronics VL53L1X datasheet. Search "VL53L1X datasheet STMicroelectronics" for the timing budget, the field of view and the ranging modes.
- The Sharp GP2Y0A21YK0F datasheet. Search "GP2Y0A21YK0F datasheet" for the output curve and the range.

*Tutorials*

- Adafruit Learning System, the guide on the VL53L1X and the library's examples. Search "Adafruit VL53L1X guide".
- SparkFun Learn, the tutorial on ultrasonic distance sensors. Search "SparkFun Learn ultrasonic sensor".
- GeeksforGeeks, the article on ultrasonic sensors and their working principle. Search "GeeksforGeeks ultrasonic sensor working".
- Arduino Language Reference, the page for pulseIn(), at docs.arduino.cc.

*Watch*

- Cyrill Stachniss, the lecture series on range sensors and sensor models for robot mapping on YouTube. Search "Stachniss range sensor models".
- MIT OpenCourseWare, 2.12 Introduction to Robotics, the lectures on sensors and sensing. Search the course title on ocw.mit.edu.

*Practice*

- Wokwi simulator at wokwi.com, which includes an HC-SR04 model and an Arduino Uno. Set the distance in the simulator and check your code's conversion before you test with real hardware.

---

## Tomorrow

Tomorrow you will measure motion of the robot itself, through its acceleration and rotation. Day 31 covers IMUs: accelerometers, gyroscopes and magnetometers, their axes, bias and drift, and how calibration and orientation estimates work. The IMU is the sensor that tells the robot it is tilting or turning when its wheels cannot tell it, and it pairs with the encoder odometry from Day 29 in almost every mobile robot.

**Next:** [Day 31: IMUs](Robotics-Day-31-IMUs.md)
