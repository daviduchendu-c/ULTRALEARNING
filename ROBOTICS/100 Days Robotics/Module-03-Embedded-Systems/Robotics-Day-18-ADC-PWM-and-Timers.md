# Day 18: ADC, PWM and Timers

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 4 hours

**Level:** beginner to intermediate. You need Day 16 (the Uno, the Arduino IDE and `millis()`), Day 17 (logic levels and pull-ups) and Day 11 (the ADC as a concept and PWM as an average voltage). Today the numbers matter more than the wiring, so keep a notebook for your measurements.

**By the end of this day, you will have:**

- A potentiometer read with `analogRead()`, with the ADC step size and the reading converted to volts, and checked against a multimeter
- An LED dimmed with `analogWrite()`, with the duty cycle and the measured average voltage matched within a few percent
- A timer-based sampling loop that runs at a fixed rate, with the timing error measured
- A short written note on the trade-offs between sampling rate, resolution and noise

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3.x and the `Arduino AVR Boards` core at version 1.8.6
- One 10 kOhm linear potentiometer, about $1 to $2
- One red LED and one 220 ohm resistor, about $0.20
- One 0.1 uF ceramic capacitor, about $0.10, for the noise exercise
- A breadboard and jumper wires, about $5 to $8
- A digital multimeter with DC volts, about $15 to $25
- Optional: a USB logic analyzer or a two-channel oscilloscope kit, about $10 to $80, for measuring PWM frequency directly. The multimeter is enough for the main project.

**Prerequisites:**

- Day 11: the ADC's job, the idea of sampling, and PWM as an average voltage
- Day 16: `millis()` and the register-level view of a timer
- Day 17: the debounce pattern and the rule about blocking code

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Analog-to-digital conversion](#analog-to-digital-conversion)
- [Sampling and the Nyquist limit](#sampling-and-the-nyquist-limit)
- [PWM duty cycle and frequency](#pwm-duty-cycle-and-frequency)
- [Hardware timers and prescalers](#hardware-timers-and-prescalers)
- [Timing-sensitive control](#timing-sensitive-control)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a potentiometer dimmer with a timed logger](#project-a-potentiometer-dimmer-with-a-timed-logger)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot lives in a continuous world. Battery voltage drifts, a wheel's position changes smoothly, a temperature rises over minutes, and a joint moves through an angle. The microcontroller lives in discrete steps. It reads numbers at fixed moments and changes outputs at fixed moments. The analog-to-digital converter, pulse-width modulation and the hardware timers are the three tools that connect the two worlds. The ADC turns a voltage into a number. PWM turns a number into an average power. The timers decide when both of them happen.

These three pieces decide whether a robot is responsive or sluggish, smooth or jerky, accurate or drifting. A motor controller that samples current too slowly misses a stall. A dimmer that updates its PWM at the wrong rate flickers in a camera image. A sensor logger that uses a timer with the wrong prescaler records data at a rate it does not expect. Most of the timing bugs on a robot come from one of these three mechanisms, so it pays to understand them before you build on them.

Today you will read a potentiometer, dim an LED with PWM, and sample at a fixed rate using a timer. You will check each result against a multimeter, so the numbers you write in your notebook are measured values and not only calculated ones.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1930s to 1940s | Pulse modulation methods such as pulse-code modulation are developed for communication, with its theory later formalized by Harry Nyquist and others. | The sampling theory used in ADCs and PWM design comes from this work. |
| 1960s | Integrated circuit ADCs appear, but they are large and expensive. | Robots of that era used simple analog control loops. |
| 1970s to 1980s | Microcontrollers gain built-in timers, which generate precise intervals and PWM without the CPU doing the work. | Modern motor and servo control relies on hardware timers rather than software loops. |
| 1990s | Successive-approximation ADCs with 10-bit resolution become common on microcontrollers. | The ATmega328P's 10-bit ADC is in this family. |
| 2000s to today | PWM is the standard way to control motors, LEDs and heaters, and 12-bit and 16-bit ADCs are common on robot controllers. | Most robot power control is PWM, and most sensor reads are ADC. |

The 10-bit ADC on the Uno is old design, and it is slower than a modern 12-bit part. It is still an excellent teaching tool, because every number it produces is easy to check by hand.

---

## Glossary

- **ADC (analog-to-digital converter):** a circuit that measures a voltage and produces a binary number that represents it.
- **Resolution:** the number of distinct output codes an ADC can produce. A 10-bit ADC produces 1024 codes.
- **LSB (least significant bit):** the voltage step represented by one unit of the ADC output. On the Uno, one LSB is VREF / 1024.
- **VREF (reference voltage):** the voltage that the ADC treats as full scale. On the Uno the default is the 5 V supply.
- **Sampling:** measuring a signal at fixed times, so that a continuous voltage becomes a series of numbers.
- **Nyquist limit:** the rule that a signal must be sampled at more than twice its highest frequency to be recovered without distortion.
- **Aliasing:** the distortion that appears when a signal is sampled too slowly, which makes a fast signal look slow.
- **PWM (pulse-width modulation):** a method of setting an average output by switching a signal fully on and off at a fixed frequency.
- **Duty cycle:** the fraction of each PWM period that the output is high, expressed as a percentage or a number from 0 to 1.
- **Timer:** a hardware counter that increases at a set rate and can trigger an action when it reaches a value.
- **Prescaler:** a divider between the clock and the timer, which slows the counting rate.
- **Overflow:** the event when a timer counter reaches its maximum value and wraps back to zero.
- **Compare match:** the event when a timer counter equals a set value, which can trigger an interrupt or change an output.
- **Jitter:** the variation in the timing of an event from one period to the next.

---

### Analog-to-digital conversion

A kitchen scale has a needle that moves over a dial. A person reads the needle and writes the weight on a notepad. An ADC does the reading for the microcontroller. It compares the input voltage with a set of reference steps, finds the closest step, and outputs its number. The number is only as accurate as the steps, and the steps are only as fine as the resolution.

From first principles, a 10-bit ADC divides its reference voltage into 1024 steps. The step size, one LSB, is VREF / 1024. With VREF at 5 V, one LSB is 5 / 1024 = 0.00488 V, or 4.88 mV. A reading of 512 means the input is about 512 × 4.88 mV = 2.5 V. A reading of 1023 means the input is at the top of the range, about 4.995 V. The conversion from reading to voltage is V = reading × VREF / 1024, and that formula is the only calculation you need for today.

In the worked example, a 10 kOhm pot wired across 5 V and ground, with its wiper at the midpoint, gives 2.5 V at the wiper. The ADC reads about 512, and the code converts it back to 2.5 V. If the reading is 511, the voltage is 511 × 5 / 1024 = 2.495 V, and the difference is one LSB. The error reflects the resolution limit of the 10-bit ADC.

The common mistake is to use a high-impedance source on the ADC input. The ADC has a small sampling capacitor that must charge to the input voltage before each conversion. If the source resistance is high, the capacitor does not charge in time, and the reading is low or noisy. The Arduino documentation suggests a source impedance below about 10 kOhm for reliable reads. A potentiometer at 10 kOhm sits right at this limit, which is why the project uses a 0.1 uF capacitor on the wiper for the noise test.

ADCs appear in every robot that reads battery voltage, temperature, current sense amplifiers and force sensors. The rule is the same in each case: the number is only as good as the reference, the source and the sample timing.

---

### Sampling and the Nyquist limit

A camera takes pictures at a fixed rate. If a car wheel spins fast in a movie, the wheel can look like it is spinning backward. The camera's frame rate is too slow for the wheel's rotation, and the picture gives a false impression. An ADC has the same problem. If the input changes faster than the sampling rate allows, the readings show a false, slower signal.

The first principle is the Nyquist rule, which states that a signal with a highest frequency f must be sampled at a rate above 2f. A sine wave at 1 Hz needs a sampling rate above 2 Hz to be recovered, and a faster signal needs a proportionally faster sample rate. Below that rate, the high-frequency content folds back into the sampled data as aliasing.

The Uno's ADC conversion takes about 13 ADC clock cycles. With the Arduino core's default prescaler of 128, the ADC clock is 16 MHz / 128 = 125 kHz, so one conversion takes about 13 / 125,000 = 104 microseconds. That gives a maximum sampling rate of roughly 9,600 samples per second for `analogRead`, and in practice a loop with other work runs much slower. A motor vibration at 200 Hz needs at least 400 samples per second, and a loop that reads once every 1 ms, which is 1,000 samples per second, is comfortably above that.

The worked example is a signal at 60 Hz, such as the ripple from a mains-powered supply. A loop that samples at 50 Hz will alias the 60 Hz ripple into a false 10 Hz pattern, because 60 Hz is above the 25 Hz Nyquist limit for a 50 Hz sample rate. The reading looks like a slow wobble and it has nothing to do with the real signal. A check is to sample at a different rate and see whether the pattern changes.

The common mistake is to think a fast loop always gives a good sample. The loop may be fast, but the sample times may be uneven because the loop does other work. This uneven spacing is jitter, and it is why a timer-driven sample is more reliable than a loop that reads whenever it gets to the line. The section on timing-sensitive control returns to this point.

Sampling theory is the basis of every sensor logger, every audio system and every vibration analysis on a robot. An engineer who understands aliasing avoids wrong conclusions from a clean-looking but wrong graph.

---

### PWM duty cycle and frequency

A water tap with a fast valve delivers the same average flow as a slow tap if you open it for the same total time. PWM uses that trick with voltage. The output switches fully on and off at a fixed frequency, and the average voltage depends on how long it stays high in each period. The motor or LED responds to the average, because its inertia or its thermal and electrical response smooths the switching.

From first principles, the average voltage is V_avg = D × V_high, where D is the duty cycle as a fraction and V_high is the output's high level. The Uno's PWM output is 5 V when high. The `analogWrite` function takes a value from 0 to 255, which sets the duty as value / 255. A value of 128 gives a duty of 128 / 255 = 0.502, so the average is about 2.51 V.

The frequency is set by the timer that drives the pin. The Uno's default PWM frequency is about 490 Hz on pins 3, 9, 10 and 11, and about 980 Hz on pins 5 and 6. The 980 Hz output comes from Timer0, and the 490 Hz outputs come from Timer1 and Timer2, which the Arduino core configures in its default PWM modes. The 490 Hz setting can produce an audible whine in a motor and a visible flicker in a camera, so for some robot designs, the frequency is changed in the timer registers.

The worked example is the flicker check. An LED driven at 490 Hz will look steady to the eye and to most cameras. A camera with a shutter speed near 1/500 second can capture a pattern from a PWM signal, and the banding is a clue that the PWM frequency is close to the frame rate. If you ever see rolling bands in a video of a robot's LEDs, check the PWM frequency first.

The common mistake is to assume the average voltage is the same as the speed. A motor at 50 percent duty does not necessarily run at half speed, because the back-EMF, the friction and the load all change the relation. Duty sets the average voltage, and the speed follows from the motor's behavior. The LED's brightness follows more closely, because the eye's response is roughly linear in the average light for a small range.

PWM is the basic power control of robotics. Motor drivers, servo signals, LED strips, heaters and pump speed controls all use it, and the frequency choice affects noise, efficiency and heating.

---

### Hardware timers and prescalers

A stopwatch counts seconds by ticking, and the speed of the tick decides how precise the measurement is. A microcontroller timer is a counter that ticks from the clock. Every tick increases a number, and when the number reaches a set value, the timer can do something: raise a flag, call an interrupt or change an output pin. Because the counter runs in hardware, it does not need the CPU to count, and it is much more precise than a loop.

From first principles, the timer's tick rate is the clock divided by the prescaler. The ATmega328P's clock is 16 MHz, and the prescaler options are 1, 8, 64, 256 and 1024 for Timer0 and Timer1, with additional options for Timer2. A prescaler of 64 gives a tick rate of 16,000,000 / 64 = 250,000 ticks per second, so each tick is 4 microseconds. Timer1 is 16 bits wide, so it counts from 0 to 65,535, and at that tick rate it overflows every 65,536 × 4 microseconds, about 262 milliseconds.

The period of a compare match timer is T = (OCR + 1) × prescaler / F_CPU, where OCR is the value in the compare register. For a 1 kHz timer with prescaler 1, the OCR value is 16,000,000 / 1000 − 1 = 15,999. That is the number used in the project for an interrupt every millisecond, which you will see again on Day 19.

The worked example uses the PWM frequency formula for Timer0 in fast PWM mode. The frequency is F_CPU / (prescaler × 256). With the prescaler at 64, that is 16,000,000 / (64 × 256) = 976.6 Hz, which matches the Uno's 980 Hz output on pins 5 and 6. The arithmetic is the reason to know the prescaler: the same timer gives a different frequency with a different prescaler.

The common mistake is to change Timer0 without knowing what depends on it. Timer0 drives `millis()`, `micros()` and `delay()`. A change to its prescaler makes the time functions run at the wrong speed, and the robot's loop timings all shift. The way to spot the problem is that `millis()` reports time that is too fast or too slow, so test that before you change any timer register.

Hardware timers are used for every precise timed event on a robot: encoder counting, servo pulses, sensor sampling and the periodic control loop. The timer configuration is one of the first things a firmware reviewer reads.

---

### Timing-sensitive control

A kitchen timer that rings after a set number of minutes is more reliable than a cook who checks the clock at random. The timer keeps the time, and the cook responds when it rings. A control loop works the same way. The loop must run at a fixed rate, and the timing must not drift with the work it does.

From first principles, a control loop has a sample time T_s. Each cycle reads sensors, computes an output and writes that output. If the cycle takes longer than T_s, the loop runs late, and the control action is wrong by the lateness. If the cycle timing varies, the control action has jitter, which shows as a rough motion or a noisy output. A robot's motor controller might run at 1 kHz with T_s = 1 ms. A loop that takes 1.2 ms on one pass and 0.8 ms on the next has a timing error of 0.2 ms on each step, which is 20 percent of the period.

The fix is to base the loop on a timer compare event, not on the loop's natural speed. The timer triggers the sample, and the main code does its work in the time that remains. This pattern appears again on Day 19 with interrupts and on Day 24 with RTOS timing.

The worked example compares two designs. A loop that calls `analogRead` and `digitalWrite` in order, with no fixed timing, runs at a rate that depends on the code's branches. A loop that waits for a timer flag and then runs has a fixed period, so the sample times are evenly spaced. You can measure the difference with `micros()` by recording the time between iterations and computing the standard deviation of the intervals.

The common mistake is to assume `delay()` gives accurate timing. It waits for a specified time, but it does nothing for other work, and its real timing depends on the loop's previous and later code. For a control loop, the timer-based pattern is more reliable.

On the job, fixed-rate control is a basic requirement of any motor controller, flight controller or balance controller. A robot that cannot hold its loop rate will be unstable, and the timing budget is part of every design review.

---

## Hands-on exercises

1. **Read the pot.** Wire a 10 kOhm potentiometer with its ends to 5 V and GND and its wiper to A0. Print the raw `analogRead` value every 200 ms. Turn the knob to both ends and record the minimum and maximum readings. Explain why they are not exactly 0 and 1023.

2. **Calibrate the ADC.** With the multimeter in DC volts mode, measure the wiper voltage at five knob positions. For each one, record the multimeter reading and the Arduino reading converted with V = reading × 5 / 1024. Compute the error at each point.

3. **Test the noise.** Add the 0.1 uF capacitor from the wiper to GND. Print 100 readings with the capacitor disconnected and then connected. Compute the standard deviation of each set and write the result in your notebook.

4. **Dim the LED.** Wire the red LED through a 220 ohm resistor to pin 9. Use `analogWrite` with values 0, 64, 128, 192 and 255. For each value, measure the average voltage on the LED pin with the multimeter and compare it with D × 5 V.

5. **Read the timer frequency.** Use a logic analyzer or an oscilloscope on pin 9 if you have one. Measure the PWM frequency and compare it with the expected 490 Hz. If you do not have a scope, write a sentence on what you would measure.

6. **Sample at a fixed rate.** Use `micros()` in a loop to sample A0 at 1 kHz without a timer. Record 1,000 intervals and compute their mean and standard deviation. Write one sentence on the jitter you see.

---

## Project: a potentiometer dimmer with a timed logger

The goal is a repository that reads a potentiometer, dims an LED, and logs the readings at a fixed rate, all with evidence you can check. The project is practical, and the results connect the theory of this lesson to numbers that a reviewer can recalculate.

**Repo name:** `100-days-robotics-day-18-adc-pwm-timers`

**Repo layout:**

```
100-days-robotics-day-18-adc-pwm-timers/
├── README.md
├── dimmer/
│   └── dimmer.ino
├── timed_logger/
│   └── timed_logger.ino
├── logs/
│   ├── adc_calibration.csv
│   ├── pwm_average_voltage.csv
│   └── timing_intervals.csv
├── docs/
│   ├── calculations.md
│   └── wiring.md
└── photos/
    └── bench.jpg
```

**Starter code, `dimmer/dimmer.ino`:**

```cpp
const int POT_PIN = A0;
const int LED_PIN = 9;               // Timer1 PWM output on the Uno

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(115200);
}

void loop() {
  int raw = analogRead(POT_PIN);     // 0 to 1023
  int duty = raw / 4;                // map 0 to 1023 onto 0 to 255
  analogWrite(LED_PIN, duty);
  float volts = raw * 5.0 / 1024.0;  // ADC conversion from the lesson
  Serial.print(raw);
  Serial.print(",");
  Serial.print(duty);
  Serial.print(",");
  Serial.println(volts, 3);
  delay(100);
}
```

**Starter code, `timed_logger/timed_logger.ino`:**

```cpp
const unsigned long PERIOD_US = 10000;   // 100 Hz sample rate
unsigned long nextSample = 0;
unsigned long lastSample = 0;
const int SAMPLES = 500;
int count = 0;

void setup() {
  Serial.begin(115200);
  nextSample = micros() + PERIOD_US;
  Serial.println("index,interval_us,raw");
}

void loop() {
  if (count >= SAMPLES) return;
  unsigned long now = micros();
  if ((long)(now - nextSample) >= 0) {
    unsigned long interval = now - lastSample;
    lastSample = now;
    int raw = analogRead(A0);
    Serial.print(count);
    Serial.print(",");
    Serial.print(interval);
    Serial.print(",");
    Serial.println(raw);
    nextSample += PERIOD_US;
    count++;
  }
}
```

The timed logger uses a software schedule, not a hardware compare interrupt. That is a teaching choice. It is simpler to read, and it shows why jitter appears. The README should say so, and the report should compare the jitter with the timer-based design from Day 19.

**Step-by-step tasks:**

1. Create the repository and folder tree. Initialize git and commit the README skeleton.
2. Wire the pot and the LED as in the lesson. Upload `dimmer` and confirm the LED changes brightness with the knob.
3. Use the multimeter to record the average voltage on pin 9 at five duty values. Save the results to `logs/pwm_average_voltage.csv` with columns for the duty, the expected voltage and the measured voltage.
4. Record 100 ADC readings at five knob positions with the multimeter also reading the wiper. Save the table as `logs/adc_calibration.csv` and compute the error for each row.
5. Upload `timed_logger`. Save the serial output as `logs/timing_intervals.csv`. Compute the mean and standard deviation of the interval column and write them in the README.
6. Write `docs/calculations.md` with the step size, the expected PWM average, the Nyquist check for a 5 Hz pot wobble and the timing jitter.
7. Write the README with the checklist below and the summary of your measured numbers.

**README checklist:**

- [ ] The ADC step size is calculated and the reading conversion is shown
- [ ] The PWM average voltage is compared with the measured value for each duty
- [ ] The calibration table has a column for the error at each point
- [ ] The interval mean and standard deviation are reported from the timed logger
- [ ] The Timer1 PWM frequency on pin 9 is stated, with the source of the number
- [ ] The README says which timer drives `millis()` and why changing it would break the time functions

**Done when:**

- The dimmer runs, the LED brightness follows the knob, and the serial output shows the raw, duty and volts columns
- The PWM table shows measured average voltages within a reasonable tolerance of D × 5 V, and the README explains any larger gap
- The calibration log has at least five positions with a multimeter reading and an ADC reading each
- The timing log has at least 400 intervals, and the README reports their mean and standard deviation
- The repository has at least three commits, and every number in the README matches a file in `logs/`

**Evidence to commit to GitHub:** both sketches, the three log files, the calculations document, the wiring document, the bench photo and the README. Do not commit the compiled build folders.

The evidence matters more than the code here. A reviewer who opens the calibration log can check the ADC math in a minute, and a reviewer who sees the jitter numbers knows you understand why timing matters.

---

## Check your understanding

1. An ADC reads 800 with a 5 V reference on a 10-bit converter. Compute the voltage. Then compute the voltage represented by one LSB, and explain whether an error of 3 mV is within one LSB.
2. You sample a 40 Hz vibration at 50 samples per second and see a clean 10 Hz pattern in the data. Explain why, and state the minimum sample rate needed to avoid it.
3. A PWM output at 50 percent duty is connected to a 6 V motor. Someone says the motor runs at half speed. Give two reasons the speed will not be exactly half.
4. The `analogWrite` value is 200. Compute the duty cycle as a percentage and the expected average voltage on a 5 V output.
5. You change the Timer0 prescaler to make pin 6 run faster and your robot's `delay(1000)` now lasts about 4 seconds. Explain the cause and name the safer way to get a different PWM frequency.
6. Your control loop runs at an average of 1 kHz but the standard deviation of its interval is 0.3 ms. Explain why the average can look right while the control quality is still poor, and name the design change that fixes it.

---

## Answers

1. V = 800 × 5 / 1024 = 3.906 V. One LSB is 5 / 1024 = 4.88 mV. An error of 3 mV is less than one LSB, so it is within the converter's resolution and cannot be resolved from the reading alone.
2. The 40 Hz signal is above the Nyquist limit for a 50 samples per second rate, which is 25 Hz. It aliases to 50 − 40 = 10 Hz. The sample rate must be above 80 samples per second to avoid it, and a common rule is to sample at least five to ten times the highest frequency of interest.
3. The motor's back-EMF and friction change the relation between voltage and speed, so 50 percent of the supply voltage does not give 50 percent of the no-load speed. The load also changes the speed, and the motor's start-up friction makes low duty values behave differently.
4. The duty cycle is 200 / 255 = 78.4 percent. The expected average voltage is 0.784 × 5 = 3.92 V.
5. Timer0 drives `millis()` and `delay()`, so changing its prescaler changes the time base for those functions. The safer way is to use a different timer for the new PWM frequency, such as Timer1 or Timer2, and leave Timer0 alone.
6. An average of 1 kHz hides the spread of the intervals. A standard deviation of 0.3 ms on a 1 ms period is 30 percent jitter, which adds noise to the control output. The fix is to trigger the loop from a hardware timer compare event so that the interval is set by hardware, not by the code's path.

---

## Sources

*Read:*

- The Wikipedia article titled **Analog-to-digital converter**, for the conversion process and the resolution concept
- The Wikipedia article titled **Nyquist-Shannon sampling theorem**, for the sampling rule and aliasing
- The Wikipedia article titled **Pulse-width modulation**, for the duty cycle and its applications
- The Wikipedia article titled **Aliasing**, for the distortion that a slow sample rate causes
- The ATmega328P datasheet by Microchip Technology, free from the Microchip website. Read the sections on the ADC, the timers and the PWM modes
- The Arduino Language Reference at docs.arduino.cc, for `analogRead`, `analogWrite` and `micros`

*Tutorials:*

- The official Arduino tutorial on analog input, found in the Arduino documentation under Built-in Examples
- The GeeksforGeeks article on analog-to-digital conversion. Search the GeeksforGeeks site for "analog to digital converter"

*Watch:*

- The 3Blue1Brown video on the Fourier transform, which builds the intuition for sampling and frequency. Search for "3Blue1Brown Fourier transform visual introduction"
- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, for lectures on ADC, timers and PWM. Search for the course title on the MIT OpenCourseWare site

*Practice:*

- Wokwi at wokwi.com, which includes an Arduino Uno, a potentiometer and an LED. You can read the ADC and see the PWM output there, which is useful before you measure on the bench. The free tier is enough for this lesson.

---

## Tomorrow

Day 19 is about interrupts. Today you used timers to schedule sampling, and tomorrow you will let the hardware interrupt the program when an event happens. You will write interrupt service routines, handle a pin change with `attachInterrupt`, use a timer interrupt for a fixed-rate loop, and see why shared variables need care. The timer compare value from today is the same register you will use for that timer interrupt.
