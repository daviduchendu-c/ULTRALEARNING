# Day 33: Sensor Integration Project

*100 Days of Robotics Engineering, Module 4: Sensors and Actuators*

**Time:** about 6 hours, spread over two sittings if you can
**Level:** intermediate. You need Days 26 to 32 of this module, and the parts from them: the DC gearmotor and driver (Day 26), the servo with its calibration (Day 27), the encoder decoder (Day 29), the distance sensor (Day 30) and the MPU-6050 IMU (Day 31). The camera from Day 32 is optional here. This is the module's capstone, and it combines the sensors into one logged, timestamped and visualized dataset.

**By the end of this day, you will have:**

- A single Arduino sketch that streams encoder, IMU, distance and servo-event data as timestamped CSV lines at 50 Hz
- A Python logger that writes each line with the host's arrival time and the device's own time, and keeps a separate event log
- A measured latency between a servo command and the gyroscope's response, with its spread over at least 20 trials
- A multi-panel plot of one synchronized run, with the event markers drawn on every panel

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3 or newer, about $25 for an official board or $10 to $15 for a clone
- The parts from earlier days: one DC gearmotor with an encoder, the TB6612FNG driver, one SG90 or MG996R servo with its calibration from Day 27, one HC-SR04 ultrasonic module, one MPU-6050 IMU breakout
- A 5 V regulated supply for the servo, rated at 2 A or more for the MG996R, with a 1000 µF capacitor across it, as Day 27 set up
- A 6 V bench supply with a 1.2 A current limit for the DC motor, as Day 13 set up
- A USB cable and a PC or laptop with Python 3.12, NumPy 2.1 or newer, Matplotlib 3.9 or newer and pyserial 3.5, all free and open source
- A rigid plate or bracket to mount the IMU on the servo horn, so that the gyroscope sees the servo's rotation. A piece of 3 mm acrylic or a 3D-printed bracket costs under $5.
- A tape measure and a protractor, to check the servo angle independently

**Prerequisites:**

- Days 26 to 31: the calibrated servo, the decoder, the distance reading and the IMU bias calibration
- Day 13: the power budget and the separate supplies for motors and logic
- Module 3: serial communication, interrupts and reading registers over I2C
- Comfort with CSV files and a basic Python script

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How data logging and time sync developed](#how-data-logging-and-time-sync-developed)
- [Glossary](#glossary)
- [Sensor streams and sampling rates](#sensor-streams-and-sampling-rates)
- [Timestamps: device time and host time](#timestamps-device-time-and-host-time)
- [Clock drift, latency and jitter](#clock-drift-latency-and-jitter)
- [Logging a stream](#logging-a-stream)
- [Synchronizing streams](#synchronizing-streams)
- [Measuring latency with an event](#measuring-latency-with-an-event)
- [Visualizing a run](#visualizing-a-run)
- [Data quality checks](#data-quality-checks)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a logged, synchronized and visualized sensor dataset](#project-a-logged-synchronized-and-visualized-sensor-dataset)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot uses many sensors, and each one reports at its own rate, with its own delay and its own clock. A decision that uses the encoder from one moment and the IMU from another, without knowing the gap between them, is a decision based on the wrong state of the world. The encoder may say the wheel has turned, the IMU may say the robot has tilted, and the servo may have been commanded to move, but if the three readings are not lined up in time, the robot cannot tell cause from effect.

Today you build the data path that makes the module's sensors usable together. You stream every sensor from one Arduino at a known rate, stamp each line, log it on the PC, align the streams and measure the delay between a command and a response. The result is a dataset you can trust and a method you can apply to any sensor on any robot. This work is what turns a demo that runs once into a robot that runs every time.

---

## How data logging and time sync developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1960s to 1970s | Computer systems start to log events with time stamps, and the first distributed systems need a common clock to order events. | The problem of ordering events across machines is the same one a robot faces across sensors. |
| 1985 | The Network Time Protocol (NTP) is specified in RFC 1059 to synchronize computer clocks over networks, and later versions refine it. | NTP is still the standard for keeping a robot computer's clock close to real time. |
| 2002 | IEEE 1588, the Precision Time Protocol (PTP), is published for sub-microsecond synchronization in industrial networks. | Industrial robots and automation use PTP when microsecond alignment matters. |
| 2010s | The Robot Operating System (ROS) message filters provide approximate time synchronization across topics, and rosbag records streams with timestamps. | Robot software now expects each message to carry a time stamp and a frame. |
| Today | ROS 2 and its rosbag2 format record timestamped streams, and many sensors offer hardware trigger inputs for exact alignment. | A robot team that logs with timestamps can replay and debug a failure long after it happened. |

The protocols are standard, but the practical lesson is the same at every scale: record the time of each measurement at the source, and keep the clock's errors in mind.

---

## Glossary

- **Stream:** a sequence of timestamped readings from one sensor, arriving at a rate.
- **Sampling rate:** the number of readings per second that a sensor produces, in hertz (Hz).
- **Device time:** the time as the microcontroller counts it, usually from millis() or micros() since power-up.
- **Host time:** the time as the PC records it when a line arrives over the serial port.
- **Latency:** the delay between an event and its record, or between a command and its effect.
- **Jitter:** the variation in latency or in the interval between samples.
- **Clock drift:** the slow change in the difference between two clocks, measured in parts per million (ppm).
- **Resampling:** computing values on a new, regular time grid from irregular or differently timed samples.
- **Interpolation:** estimating a value between two samples, usually linearly.
- **Hardware trigger:** an electrical signal that makes several sensors sample at the same instant.
- **Event marker:** a logged record of a command or an action, with its own timestamp, used to align streams.
- **Cross-correlation:** a measure of similarity between two signals as one is shifted in time, used to find a delay.
- **Serial framing:** the format of each line sent over the serial port, which lets the receiver find the start and end of each message.

---

## Sensor streams and sampling rates

A robot's sensors run at different rates, and the first design decision is to choose a rate for each one and to accept the rate the sensor allows. The encoder can produce many counts per second, but the logger reads the total at a fixed rate. The IMU can run at a few hundred hertz, but the Uno's I2C bus and the serial link limit how often the data is sent. The distance sensor is the slowest: the HC-SR04 needs about 60 ms between readings to avoid echoes from the previous pulse.

The first principle is that the logging rate sets the time resolution of the dataset. A stream sampled at 50 Hz has a time step of 20 ms. A motion faster than 25 Hz, which is the Nyquist frequency for that sampling, cannot be represented correctly, and the IMU's high-frequency signal is aliased into the slower samples. Sample at least twice the highest frequency you need to see, and filter the input if the sensor produces more.

The worked example sets the rates for this project. The servo event happens at most a few times per second, and the gyro's response to a servo move lasts about 0.3 to 0.5 seconds. A logging rate of 50 Hz gives 15 to 25 samples in each response, which is enough to measure the latency to about 20 ms. The distance sensor runs at 10 Hz by reading it on every fifth loop, which keeps its echo time from blocking the loop for long. The encoder is read by interrupts, so its counts are exact regardless of the logging rate, and only the sample of the count is taken at 50 Hz.

The common mistake is to run every sensor at the fastest possible rate and to find, on the first log, that the serial link cannot carry the data. At 115200 baud the link carries about 11.5 kilobytes per second, since each byte takes 10 bits on the wire. A 50 Hz stream of lines about 60 characters long needs about 3 kilobytes per second, which fits with room to spare. A 1 kHz stream with the same lines needs about 60 kilobytes per second, which the link cannot carry, and the Uno's loop cannot read the sensors that fast anyway. Choose the lowest rate that meets the requirement.

In the market, the sampling plan is part of the sensor specification for every robot. Each stream has a stated rate, a maximum latency and a maximum dropout rate.

---

## Timestamps: device time and host time

A line of sensor data carries two possible times: the device's time, which is when the sensor was read, and the host's time, which is when the line reached the PC. The two differ by the serial transfer delay, the processing time and any queueing on the PC. Using the host time alone hides the delays that the robot needs to measure, and using the device time alone requires that the device's clock be trusted.

The first principle is that the timestamp should be made as close as possible to the measurement. On the Uno, millis() returns the milliseconds since power-up, with a resolution of 1 ms, and micros() gives microseconds with a resolution of about 4 µs. Read the timer at the start of the sensor read, and record the value with the reading. Do not stamp the line after the serial print, because the print can take several milliseconds on a busy bus.

The worked example uses the timing of the serial line. At 115200 baud, a 60-character line takes about 60 times 10 bits divided by 115200 bits per second, which is 5.2 ms to send. The device stamp, taken before the line is sent, is then about 5 ms earlier than the host's arrival stamp, plus any buffering. The host's stamp records the arrival time, which includes the line transfer. The difference between the device and host timestamps for the same line is therefore a measurement of the serial delay, and it should be roughly constant.

The common mistake is to assume the host clock is the true time and to compare the device stamps with it directly. The device clock starts at power-up, not at the Unix epoch, so the two times differ by an unknown offset. Compute the offset from the first few lines, or record a sync event that both clocks see, and subtract the offset before comparing.

Use both stamps. The device time gives the spacing between samples, which is set by the Arduino's own clock. The host time gives the absolute time the PC can use to align the data with other PCs, cameras or logs.

On the job, timestamps are a requirement in every data specification. A logged robot run without device timestamps is hard to use for diagnosis, and a run with inconsistent timestamps is worse.

---

## Clock drift, latency and jitter

Three timing errors matter in a logged dataset: clock drift, latency and jitter. Each one looks different, and each one needs a different fix.

The first principle is that drift is a slow change in the offset between two clocks. A crystal oscillator is accurate to tens of parts per million, which means two clocks can differ by 50 µs every second, or 180 ms after an hour. The Arduino Uno uses a ceramic resonator or a crystal, and its accuracy is typically in the range of 0.5 to 1 percent for a resonator, which is far worse than a crystal. The effect of drift is that a long run drifts out of alignment with the host clock, and an event at the start of the run is not aligned with an event at the end.

The worked example is the drift of a 0.5 percent resonator. Over a 10-minute run, the error is 0.005 times 600 seconds, which is 3 seconds. A servo event at the start of the run and another at the end would be misaligned by 3 seconds, which is large enough to destroy any latency measurement. A crystal at 50 ppm drifts 0.05 percent, or 30 ms over 10 minutes, which is much smaller. The fix for the resonator is to resynchronize the device clock with the host clock during the run, using a sync event at intervals, and to fit a linear clock model to the sync points.

Latency is the fixed delay between an event and its record. A servo command takes time to be sent, received, decoded and acted on, and the gyro takes time to show the motion. The latency of the whole chain is the sum of these delays, and it is measured by the cross-correlation method in the next section.

Jitter is the variation in the delay or in the sample interval. A serial line with a busy PC has jitter of a few milliseconds. A loop that stops for an ultrasonic echo has jitter of up to 30 ms, which is the worst case for the timeout. Jitter cannot be removed, but it can be measured, and the spread of the latency is the jitter of the chain.

The common mistake is to report the mean latency and to ignore the spread. A mean of 40 ms with a spread of 25 ms is a different system from a mean of 40 ms with a spread of 3 ms, and the robot's control loop must be designed for the spread.

In the market, the timing budget of a sensor chain is part of the robot's safety case. A robot that brakes after a sensor reading 200 ms old may have moved 0.1 m past the point it should have stopped.

---

## Logging a stream

A logger receives each line, stamps it with the host time, parses the fields and writes a CSV row. The logger must not drop lines, and it must not block the serial port while it writes to disk. In this project, the logger is a Python script with a simple loop that reads a line, parses it and writes it, and that is enough for 50 Hz.

The first principle is that the file format should carry both timestamps and the kind of record, so that each line can be read on its own. A line that starts with the kind, D for data and E for an event, followed by the device time, the fields and the host time, can be parsed without reading the lines around it. Keep the fields in a fixed order and write a header row once.

The worked example is the record format. The Arduino prints lines such as D,1234,57,0.02,-0.01,1.00,0.5,0.1,-0.2,18 for a data record with the device time 1234 ms, the encoder count 57, the three accelerometer values in g, the three gyro values in degrees per second and the distance in centimetres. The event line is E,1500,45, which means a servo command to 45 degrees was sent at device time 1500 ms. The logger adds the host time to each line as the first column.

The common mistake is to write the CSV file with a buffered write and to lose the last lines when the logger is stopped by Ctrl-C. Flush the file on each write or on every few hundred lines, and handle the keyboard interrupt so the file closes cleanly.

Another mistake is to parse the lines with a regular expression that fails silently on a malformed line. Count the malformed lines and print the count at the end of the run, because a count that is not zero tells you the serial link has a fault.

In the market, a logger of this kind is often the first piece of a data pipeline, and the same format is later converted into a robotics bag or a database table.

---

## Synchronizing streams

Synchronization puts the streams on a common time axis. In this project there is one device, so every line shares one device clock, and the streams are already synchronized in device time. The synchronization problem arises when the host time is compared with the device time, or when a second device joins the run.

The first principle is that synchronization is a mapping between clocks. A line-by-line match of device and host time is not a mapping, because the delays vary. A linear mapping fits the host time as a function of the device time, using the sync points, and its slope is the drift and its intercept is the offset. The fit is done once per run and checked against the sync points.

The worked example is the resampling step. The encoder and the IMU lines arrive every 20 ms, but the distance sensor reading is produced every 100 ms, and the servo events are at random times. To compare them, resample every stream onto a 20 ms grid. For the distance sensor, the value at each grid time is the last reading before that time, which is a zero-order hold. For the IMU, the value is linearly interpolated between the two nearest samples, because the IMU is sampled regularly. The choice of hold or interpolation depends on the signal: a sensor value that changes in steps should be held, and a continuous signal should be interpolated.

The common mistake is to interpolate a stream that has gaps. An interpolation across a 300 ms gap in the IMU creates a smooth line that the sensor never measured, and any latency measured from it is wrong. Mark the gaps, and do not interpolate across them.

In the market, the synchronization method is chosen by the sensor's trigger and clock. Hardware-triggered cameras and IMUs sync to within microseconds, and software-stamped sensors sync to a few milliseconds.

---

## Measuring latency with an event

The most useful measurement a robot logger can make is the latency of a chain from command to response. A servo that moves on command is a good test, because the command is an exact event with a known time, and the IMU mounted on the servo horn sees the motion as a change in gyro rate.

The first principle is that a delay between two signals appears as a shift in their cross-correlation. Let the command signal be a step at time t0, and let the response be the gyro rate, which rises after a delay L. The cross-correlation of the two signals peaks at a lag of L. The peak is found by computing the correlation at many lags, and the lag at the peak is the latency. A smoother alternative is to find the time when the gyro rate first crosses a threshold, such as 10 percent of its peak, which gives the latency of the first movement.

The worked example uses the threshold method. The servo command is logged at device time 1500 ms. The gyro rate, filtered to remove the noise, crosses 5 degrees per second at device time 1545 ms, and the peak rate is 40 degrees per second. The first-movement latency is 45 ms. The servo's own response takes time to build up torque and overcome friction, so the latency includes the servo's mechanical delay as well as the serial and processing delays. The spread of the latency over 20 trials is the jitter, and the mean is the chain's latency.

The common mistake is to measure the latency with the host time of the command and the device time of the response, or to mix two clocks without an offset. Use the device time for both, because they share one clock. If the command is sent from the PC, log the send time on the PC and the arrival time on the device, and compute the transfer delay separately.

Another mistake is to trust a single trial. The servo's response varies with its load, its position and its temperature, so a single trial is a sample of one. Twenty trials at the same position give a mean and a spread that a robot team can use.

In the market, latency tests are the basis of a robot's control budget. A team that measures a 60 ms chain latency designs a control loop that tolerates it, and a team that does not measure the latency designs a loop that fails under load.

---

## Visualizing a run

A plot is the fastest way to see whether a run is good. A multi-panel plot with a shared time axis, one panel per sensor family and the event markers on every panel, shows the timing relations at a glance. A panel that is flat when it should move, or a gap in a stream, shows a fault that the numbers may hide.

The first principle is that every panel must share the time axis and use the same event markers. Plot the device time on the x-axis after subtracting the first sample's time, so the axis starts at zero. Draw a vertical line at each servo event across all panels, and label the event with its command angle. Use a separate panel for each sensor family, and do not overlay units that differ by orders of magnitude, such as encoder counts and g values, on one axis.

The worked example is a four-panel figure. The top panel shows the servo's commanded angle as a step at each event. The second panel shows the gyro rate about the axis of the servo. The third panel shows the encoder count rate, and the bottom panel shows the distance reading with its gaps marked. A reviewer can see that the gyro responds about 45 ms after each command, that the encoder is flat because the motor is off, and that the distance sensor reads the bench edge with a small number of gaps.

The common mistake is to plot a stream with its gaps interpolated across, which hides the gaps. Plot the raw points with markers, or draw the gaps as breaks in the line. Another mistake is to choose a time axis in hours for a 60-second run, which makes the detail invisible. Choose the axis to match the run's length.

In the market, dashboard and log tools present the same kind of multi-stream view, and the team that can read a plot like this can diagnose a field fault in minutes.

---

## Data quality checks

Before a run is used, a few checks confirm that the data is complete and consistent. The checks are simple and quick to write, and they catch most of the faults that would otherwise appear as a wrong result later.

The first principle is that a good dataset has the expected rate, no unexpected gaps and no impossible values. The check for rate counts the samples per second and compares the count with the planned rate. The check for gaps finds any interval longer than twice the planned sample period. The check for values flags readings outside the sensor's physical range, such as an accelerometer reading over 2 g on a board that is at rest, or a distance of zero on a sensor that cannot read it.

The worked example is a run planned at 50 Hz. A 60-second run should have about 3000 data lines. If the logger records 2750 lines, the rate is 45.8 Hz, and the missing 250 lines are either a serial dropout or a loop that stalled, which the gap check locates. A gap of 300 ms at a particular time, plotted on the timeline, may match an ultrasonic echo timeout, which blocks the loop. The fix is to shorten the ultrasonic timeout or to read it less often.

The common mistake is to accept a run without looking at it. A run with the right number of lines may still have a stuck sensor reading the same value for 30 seconds, or a sign reversed on one axis. A plot and a summary table, both checked before analysis, are the minimum.

Keep a record of each run's checks in the README or in a log file, so that a later reviewer can see which runs were accepted and why.

On the job, data quality checks are the first step of any test report, and a dataset that fails them is re-run before any result is reported.

---

## Hands-on exercises

Keep the servo's supply separate from the Arduino's 5 V pin, and keep the DC motor on its own bench supply with the current limit set. Keep all grounds connected. Never hold the servo at stall for more than two seconds, and stop the test if any component becomes hot.

1. **Stream one sensor.** Connect the MPU-6050 and print its readings at 50 Hz with a device timestamp from millis(). Run for 60 seconds and count the lines. Compare the count with 3000, and compute the rate.

2. **Add the encoder.** Attach the encoder from Day 29 with its interrupt pins, and add its count to the printed line. Turn the motor by hand and check that the count changes with the direction. Record the count at the start and the end of a 60-second run with the motor off.

3. **Add the distance sensor.** Add the HC-SR04 on pins 7 and 8, reading it on every fifth loop. Check the distance against a ruler at 30 cm, and note how often the reading is missing.

4. **Add the servo and the event line.** Mount the IMU on the servo horn, attach the servo to pin 9, and add the M command. Send M45 from the serial monitor and check that the event line appears with the device time. Confirm that the gyro's Z rate responds to the move.

5. **Log and check.** Run the Python logger from the project for 60 seconds with three servo moves. Run the data quality checks on the file: the rate, the largest gap and the number of malformed lines.

6. **Measure the latency.** Repeat the servo move 20 times at the same two angles, with a 3-second pause between moves. Use the threshold method to find the first-movement latency for each move, and compute the mean and the spread.

---

## Project: a logged, synchronized and visualized sensor dataset

This project is the module's capstone. It combines the encoder, the IMU, the distance sensor and the servo event into one logged dataset, measures the chain's latency and produces a multi-panel plot of a run. The output is a dataset and a method that a robot team would use on any new robot.

**Goal:** stream four sensor families at 50 Hz from one Arduino, log them with device and host timestamps, measure the servo-to-gyro latency over 20 trials and visualize one run with event markers.

**Repo layout:**

```
100-days-robotics-engineering-day-33-sensor-integration/
├── README.md
├── firmware/
│   └── sensor_stream/
│       └── sensor_stream.ino
├── logger/
│   ├── log_streams.py
│   └── requirements.txt
├── analysis/
│   ├── quality_checks.py
│   ├── latency.py
│   └── plot_run.py
├── data/
│   ├── runs/
│   │   └── run_01.csv
│   └── latency_trials.csv
├── calculations/
│   ├── rates_and_budget.md
│   └── timing_model.md
├── plots/
│   ├── run_01_panels.png
│   └── latency_histogram.png
└── images/
    └── rig_setup.jpg
```

**Starter code:** `firmware/sensor_stream/sensor_stream.ino`

```cpp
#include <Wire.h>
#include <Servo.h>

const int ENC_A = 2;          // interrupt pin
const int ENC_B = 3;          // interrupt pin
const int TRIG  = 7;
const int ECHO  = 8;
const int SERVO_PIN = 9;
const uint8_t MPU = 0x68;

const int8_t TRANS[16] = {0, -1, 1, 0,  1, 0, 0, -1,  -1, 0, 0, 1,  0, 1, -1, 0};
volatile long encCount = 0;
volatile uint8_t encState = 0;

Servo pan;
int loopCount = 0;
float lastDistCm = -1;
const unsigned long PERIOD_MS = 20;      // 50 Hz

void encISR() {
  uint8_t s = (digitalRead(ENC_A) << 1) | digitalRead(ENC_B);
  encCount += TRANS[(encState << 2) | s];
  encState = s;
}

int16_t readWord() {
  uint8_t hi = Wire.read();
  uint8_t lo = Wire.read();
  return (int16_t)((hi << 8) | lo);
}

void readMpu(int16_t v[7]) {
  Wire.beginTransmission(MPU);
  Wire.write(0x3B);
  Wire.endTransmission(false);
  Wire.requestFrom(MPU, (uint8_t)14, (uint8_t)true);
  for (int i = 0; i < 7; i++) v[i] = readWord();   // ax ay az temp gx gy gz
}

float readDistCm() {
  digitalWrite(TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  unsigned long t = pulseIn(ECHO, HIGH, 10000UL);   // 10 ms, about 1.7 m
  if (t == 0) return -1.0f;
  return t * 0.0343f / 2.0f;
}

void setup() {
  Serial.begin(115200);
  pinMode(ENC_A, INPUT_PULLUP);
  pinMode(ENC_B, INPUT_PULLUP);
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  encState = (digitalRead(ENC_A) << 1) | digitalRead(ENC_B);
  attachInterrupt(digitalPinToInterrupt(ENC_A), encISR, CHANGE);
  attachInterrupt(digitalPinToInterrupt(ENC_B), encISR, CHANGE);
  Wire.begin();
  Wire.beginTransmission(MPU);
  Wire.write(0x6B);
  Wire.write(0);
  Wire.endTransmission(true);
  pan.attach(SERVO_PIN, 500, 2500);
  pan.write(90);
}

void loop() {
  // Serial command: M<angle> moves the servo and writes an event line.
  while (Serial.available()) {
    char c = Serial.read();
    if (c == 'M') {
      float angle = Serial.parseFloat();
      if (angle < 0) angle = 0;
      if (angle > 170) angle = 170;          // calibrated range from Day 27
      unsigned long tEvent = millis();
      pan.write((int)angle);
      Serial.print("E,");
      Serial.print(tEvent);
      Serial.print(",");
      Serial.println(angle, 1);
    }
  }

  static unsigned long next = 0;
  unsigned long now = millis();
  if (now < next) return;
  next = now + PERIOD_MS;

  int16_t v[7];
  readMpu(v);
  if (loopCount % 5 == 0) lastDistCm = readDistCm();   // 10 Hz distance
  loopCount++;

  noInterrupts();
  long enc = encCount;
  interrupts();

  Serial.print("D,");
  Serial.print(now);     Serial.print(",");
  Serial.print(enc);     Serial.print(",");
  for (int i = 0; i < 3; i++) { Serial.print(v[i] / 16384.0f, 3); Serial.print(","); }
  for (int i = 4; i < 7; i++) { Serial.print(v[i] / 131.0f, 2); Serial.print(","); }
  Serial.println(lastDistCm, 1);
}
```

The starter code sends a data line every 20 milliseconds with the device time, the encoder count, the accelerometer in g, the gyro in degrees per second and the distance in centimetres. It sends an event line when it receives an M command, with the device time at which the servo was commanded. It uses the Day 27 servo range and the Day 30 distance timing.

Three points need care. The ultrasonic read blocks the loop for up to 10 ms, which is why it runs on every fifth loop and why the timeout is short. The gyro values are raw, with the bias not yet removed; subtract the bias from your Day 31 calibration in the analysis, not in the firmware, so the raw data stays available. The encoder count is read with interrupts disabled for a moment, which keeps the value consistent.

The Python logger `logger/log_streams.py` opens the serial port, sends the M commands on a schedule, writes each line with the host time as the first column, and counts malformed lines. The analysis scripts read the CSV and run the quality checks, compute the latency with the threshold method, and plot the run. Write them as part of the project.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the wiring diagram, the power arrangement and the mounting of the IMU on the servo horn.
2. Flash the firmware and confirm the data and event lines in the serial monitor. Record the line rate in `calculations/rates_and_budget.md`.
3. Write the logger, run it for 60 seconds with three servo moves, and save the file as `data/runs/run_01.csv`.
4. Run `analysis/quality_checks.py` and record the rate, the largest gap, the malformed-line count and any out-of-range value in the README.
5. Repeat the servo move 20 times at two angles with a 3-second pause, and compute the latency for each trial with `analysis/latency.py`. Save the trials to `data/latency_trials.csv`, and plot the histogram in `plots/latency_histogram.png`.
6. Write `calculations/timing_model.md` with the offset between device and host time, the serial transfer delay you measured, and the total chain latency with its spread. Produce `plots/run_01_panels.png` with the four panels from the lesson.

**README checklist:**

- [ ] Goal, the four sensor families and the rate of each
- [ ] Wiring diagram, the power arrangement and the IMU mounting
- [ ] Line rate and the quality check results for run 01
- [ ] Timing model with the device-host offset and the serial delay
- [ ] Latency mean and spread over 20 trials, with the histogram
- [ ] Four-panel plot of run 01 with the event markers
- [ ] One limit of the dataset, such as the 10 Hz distance rate or the clock drift over a long run

**Done when:**

- The data stream runs at 50 Hz with fewer than 2 percent missing lines in a 60-second run, or the README explains the cause of the missing lines.
- The quality checks show no gap longer than 100 ms in run 01, or the README identifies the gap and its cause.
- The latency is measured over 20 trials with its mean and standard deviation, and the README states the chain's jitter.
- The plot shows the servo events on every panel, and the gyro's response follows each event by the latency you measured.

**Evidence to commit to GitHub:** the firmware, the logger, the analysis scripts, the raw run file, the latency trials, the calculations, the plots, the rig photo and the README. A reviewer should be able to run the analysis on the raw files and reproduce the latency and the quality numbers.

This project moves a servo, a DC motor and a bench supply. Keep the servo's horn clear of fingers, keep the DC motor's wheel off the bench during the stream tests, and set the bench supply's current limit before you connect the motor.

---

## Check your understanding

1. A sensor is sampled at 100 Hz, and a motion at 40 Hz is present in its input. Decide whether the sensor can represent the motion correctly, and explain what happens if the sampling is reduced to 50 Hz. State one step that would prevent the error.
2. A device clock drifts by 0.2 percent. Calculate the offset after 15 minutes, and explain why a single sync at the start of the run is not enough.
3. A serial line at 115200 baud carries 80-character lines. Calculate the transfer time for one line. Explain why the device timestamp and the host timestamp for the same line differ by roughly that amount.
4. A servo event at device time 2000 ms is followed by a gyro rate that crosses the threshold at 2052 ms. The cross-correlation of the same data peaks at a lag of 48 ms. Explain why the two methods differ, and state which value to report.
5. A logger records 2950 lines in a 60-second run planned at 50 Hz. Calculate the rate and the missing line count. Name two causes that would produce this result, and one check that separates them.
6. A robot's control loop uses a sensor reading that is 150 ms old. The robot moves at 0.4 m/s. Calculate the distance the robot moves between the true state and the state the controller sees. Decide whether the controller should stop or slow down for an obstacle 0.5 m ahead, and explain your reasoning.

---

### Answers

1. The sensor can represent a motion at 40 Hz at 100 Hz sampling, because the Nyquist frequency is 50 Hz, which is above 40 Hz. At 50 Hz sampling, the Nyquist frequency is 25 Hz, so the 40 Hz motion is aliased to 50 minus 40, which is 10 Hz, and it appears as a false, slower motion. The step that prevents the error is a low-pass filter before sampling, with its cutoff below 25 Hz, or a higher sampling rate.
2. The offset after 15 minutes is 0.002 times 900 seconds, which is 1.8 seconds. A single sync at the start leaves 1.8 seconds of error at the end, which is far larger than a latency of tens of milliseconds. The clock must be re-synced at intervals during the run, and a linear clock model must be fitted to the sync points.
3. The transfer time is 80 characters times 10 bits per character divided by 115200 bits per second, which is 6.9 ms. The device timestamp is taken before the line starts transmitting, and the host timestamp is taken when the line arrives, so the difference is roughly the transfer time plus buffering, which is about 7 ms.
4. The threshold method finds the first movement, which is the time the gyro first rises above a small value. The cross-correlation finds the lag that best aligns the two signals across the whole response, which is influenced by the shape of the response and the filtering. The difference of 4 ms is within the expected spread of the two methods. Report the threshold value with its mean, and state the cross-correlation value as a check, since the first-movement time is the one that matters for the robot's response.
5. The rate is 2950 / 60, which is 49.2 Hz, and the missing count is 3000 minus 2950, which is 50 lines. Two causes are a short serial dropout, where the PC's buffer lost data, and a loop stall, where the Arduino's loop blocked for a short time, such as an ultrasonic timeout. The check that separates them is the gap analysis: a dropout at the serial level shows as a few missing lines at a single time, while a stall in the loop shows a gap that matches the blocking code's timeout.
6. The robot moves 0.4 m/s times 0.15 s, which is 0.06 m, between the true state and the state the controller sees. The obstacle is 0.5 m ahead, so the robot is about 0.44 m from it at the moment the controller acts on the old reading. The stop decision is safe only if the robot's braking distance from its speed is less than 0.44 m. The controller should stop or slow down early enough that its own braking distance plus the 0.06 m delay stays within the 0.5 m, and the design should use the reading's age in its margin.

---

## Sources

*Read*

- [Wikipedia: Network Time Protocol](https://en.wikipedia.org/wiki/Network_Time_Protocol) for the protocol and its history, including RFC 1059.
- [Wikipedia: Precision Time Protocol](https://en.wikipedia.org/wiki/Precision_Time_Protocol) for IEEE 1588 and its use in industrial systems.
- [Wikipedia: Nyquist-Shannon sampling theorem](https://en.wikipedia.org/wiki/Nyquist%E2%80%93Shannon_sampling_theorem) for the sampling rate and aliasing.
- [Wikipedia: Cross-correlation](https://en.wikipedia.org/wiki/Cross-correlation) for the lag estimate used in the latency section.
- [Wikipedia: Jitter](https://en.wikipedia.org/wiki/Jitter) for the definition and the sources of timing variation.
- [Wikipedia: Serial communication](https://en.wikipedia.org/wiki/Serial_communication) for the serial framing and the baud rate.
- The Arduino Language Reference pages for millis(), micros() and pulseIn(), at docs.arduino.cc.
- The pyserial documentation at pyserial.readthedocs.io, for reading and writing the serial port in Python.
- The ROS 2 Jazzy documentation at docs.ros.org, the pages on rosbag2 and on message filters for approximate time synchronization. Search "rosbag2 ROS 2 Jazzy" and "message_filters ApproximateTime".

*Tutorials*

- SparkFun Learn, the tutorial on serial communication and on Arduino timing. Search "SparkFun Learn serial communication".
- Adafruit Learning System, the guides on logging sensor data from an Arduino to a computer. Search "Adafruit Arduino data logging guide".
- GeeksforGeeks, the articles on Nyquist sampling and on the serial communication protocol. Search "GeeksforGeeks Nyquist sampling theorem".

*Watch*

- MIT OpenCourseWare, the course on signals and systems, the lectures on sampling and aliasing. Search "MIT OpenCourseWare signals and systems sampling" on ocw.mit.edu.
- Cyrill Stachniss, the lecture series on sensor data and state estimation on YouTube. Search "Stachniss sensor data synchronization".

*Practice*

- Wokwi simulator at wokwi.com, which includes an Arduino Uno, an MPU-6050 model and a serial monitor. Run the streaming firmware in the simulator and check the line format before you flash the real board.

---

## Tomorrow

Tomorrow the module moves on to Module 5: Mechanical Engineering and CAD, which begins with Day 34, Mechanical Fundamentals. There you will learn the forces, torques, friction and stiffness that decide whether the motors, servos and mounts you tested in this module hold up under the loads a robot puts on them. The latency and the torque numbers from today will reappear in the mechanical design.
