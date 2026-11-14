# Day 11: Analog Electronics and Signals

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 4 hours
**Level:** beginner to intermediate. You should have finished Days 6 to 10, so you can use Ohm's law, read a resistor's colour code and explain what a binary number is. Today is the first day where the real world enters the circuit, which makes it the day most robotics beginners find surprising.

**By the end of this day, you will have:**

- A working voltage divider that you measured with a multimeter and matched against your calculation
- A sketch on an Arduino Uno that reads an analog sensor, converts the reading to volts and logs it over serial
- A measured noise floor for your sensor, with and without a first-order RC low-pass filter, and a written explanation of the difference
- A worked table that links ADC resolution, reference voltage and step size for two different microcontrollers

**What you need:**

- Arduino Uno Rev3 (ATmega328P) microcontroller board, about $25 new, or a clone at about $10 to $15
- Raspberry Pi Pico (RP2040) or Raspberry Pi Pico 2 board, about $4 to $5, for the second half of the exercises
- Arduino IDE 2.3 or newer (free, Windows, macOS and Linux) with the Arduino AVR board package installed
- Thonny (free) or the current MicroPython firmware for the Pico, downloaded from micropython.org
- One 10 kΩ linear potentiometer, about $1
- Resistors: one 4.7 kΩ and one 10 kΩ, about $0.10 each from a kit
- One 1 µF ceramic or film capacitor, rated at 16 V or more, about $0.20
- Breadboard and jumper wires, about $5 for a kit
- A digital multimeter, about $15 to $25. You need one that measures DC volts to at least two decimal places.
- USB cable for the Arduino and a USB cable for the Pico
- Optional: [Wokwi](https://wokwi.com/), a free browser simulator that supports the Arduino Uno and the Pico, for the calculations that do not need hardware

**Prerequisites:**

- Days 6 to 10: voltage, current, resistance, Ohm's law, series and parallel circuits, and binary numbers
- Day 8 components: you should know what a potentiometer is and how it changes the voltage at its wiper
- A basic idea of a microcontroller pin, which Day 2 covered

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How measurement became digital](#how-measurement-became-digital)
- [Glossary](#glossary)
- [Analog and digital signals](#analog-and-digital-signals)
- [Voltage dividers](#voltage-dividers)
- [Sampling and the ADC](#sampling-and-the-adc)
- [Resolution and the DAC](#resolution-and-the-dac)
- [Noise](#noise)
- [Filters](#filters)
- [Signal conditioning](#signal-conditioning)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a noise-aware analog logger](#project-a-noise-aware-analog-logger)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A robot does not sense the world in numbers. A distance sensor produces a voltage, a battery monitor produces a voltage, and a light sensor produces a current that a resistor turns into a voltage. The processor on the other side of those wires only understands integers. Between the two sits a set of small circuits that scale, filter and convert, and most of the bugs a robotics team spends its time on live in exactly that gap. A sensor reads 4.8 V when the battery is full and the controller thinks the battery is empty. A motor makes the distance reading jump by ten centimetres every time it starts. An engineer who can predict those problems on paper, before the robot moves, saves days of debugging. That is the skill today builds.

You will use three tools. A voltage divider scales a voltage into the range the chip can read. An analog-to-digital converter (ADC) turns the voltage into a number. A low-pass filter removes the fast noise that would otherwise corrupt that number. Every sensor board you buy for a robot, from a $3 ultrasonic module to a $300 force sensor, is some combination of these three parts.

---

## How measurement became digital

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1930s to 1950s | Analog computers solve differential equations with op-amps and potentiometers. | The op-amp is still the workhorse for signal conditioning. |
| 1960s | Integrated circuits make the op-amp cheap and small. | Sensor front ends fit on a board the size of a fingernail. |
| 1970s to 1980s | Single-chip microcontrollers add ADC inputs to the processor. | A robot can read a sensor and run a control loop on one chip. |
| 1990s to 2000s | Sigma-delta and successive-approximation ADCs reach high resolution at low cost. | Accurate battery monitoring and load sensing become routine. |
| Today | Many microcontrollers include a 10-bit to 12-bit ADC and an analog comparator on the same die. | The Arduino Uno and Raspberry Pi Pico in this module both have one. |

The table shows a pattern. Each generation moved more of the signal chain onto the chip, which means you now need to understand the chip's limits as well as the sensor's.

---

## Glossary

- **Analog signal:** a quantity that can take any value within a range, such as voltage.
- **Digital signal:** a quantity that takes only a finite set of values, such as 0 or 1.
- **Voltage divider:** two resistors in series that produce an output voltage between the input voltage and ground.
- **ADC (analog-to-digital converter):** a circuit that turns a voltage into a binary number.
- **DAC (digital-to-analog converter):** a circuit that turns a binary number into a voltage.
- **Resolution:** the number of bits an ADC or DAC uses. More bits mean finer steps.
- **LSB (least significant bit):** the smallest step an ADC can report. Its voltage is the reference voltage divided by 2 to the power of the number of bits.
- **Sampling rate:** how many times per second the ADC measures the signal.
- **Noise:** unwanted variation in a measurement that does not come from the thing you are measuring.
- **Low-pass filter:** a circuit that passes slow changes and attenuates fast ones.
- **Cutoff frequency:** the frequency at which a filter's output power falls to half, which is about 70.7 percent of its input amplitude.
- **Signal conditioning:** the circuits that scale, offset, amplify, buffer and filter a sensor signal before the ADC reads it.
- **Buffer:** a circuit with high input impedance and low output impedance that copies a voltage without loading the source.

---

### Analog and digital signals

Think of a dimmer switch and an ordinary light switch. The dimmer can sit at any brightness between off and full, while the light switch is either on or off. An analog signal is the dimmer: a voltage that can sit at 2.31 V or 2.315 V, with no fixed set of allowed values. A digital signal is the light switch: a wire that is either near 0 V (logic low) or near the supply voltage (logic high), with a defined threshold between them.

Physical quantities such as temperature, distance, light and motor current are analog. The sensor turns that quantity into a voltage or current, and the microcontroller's digital pins only see logic levels. For a 5 V Arduino Uno, a pin reads as high above about 3 V and low below about 1.5 V. The exact thresholds are listed in the ATmega328P datasheet from Microchip, and they differ between chip families, which is why you must check them rather than assume.

Here is the first worked example. A thermistor sits in a voltage divider with a 10 kΩ resistor. Its output voltage changes smoothly as the room warms, which is an analog signal. If you add a comparator that flips when the voltage crosses 2.5 V, the output becomes digital: high or low. You have thrown away the information about how hot the room is and kept only whether it is above a threshold. That trade is often the right one for a bump sensor or an end stop, and the wrong one for a temperature logger.

The common mistake is to assume that a sensor marked "digital" gives you a fine-grained measurement. A switch, a limit switch and a Hall-effect proximity sensor in on-off mode are all digital outputs. They tell you that something happened, not how much. Check the datasheet for whether the output is a voltage that varies or a logic level, before you design the read-out.

On the job, this distinction decides whether a mobile robot's bumper is a switch that stops the drive motors or an analog force sensor that tells the planner how hard the robot hit something, which is used in a collaborative arm that must stop before it pinches a person.

---

### Voltage dividers

A water tank with two outlets tells you the idea. Water flows in at the top and leaks out through two pipes in a row. The level between the two pipes depends on how wide each pipe is. A voltage divider works the same way with resistors: the voltage at the middle node depends on the ratio of the two resistances.

The derivation is short. Two resistors R1 and R2 are in series across an input voltage Vin. The same current flows through both, so the current is I = Vin / (R1 + R2). The voltage across R2, which is the output taken from the middle node, is Vout = I × R2. Substituting gives the divider formula:

Vout = Vin × R2 / (R1 + R2)

The output is always smaller than the input, and it is set only by the ratio of the resistors, as long as nothing draws current from the middle node. That condition is the usual trap.

Take a 5 V supply feeding a sensor whose output may reach 5 V. You want to measure it with a microcontroller pin that is rated for 5 V, so there is no need to divide. Now take a 9 V battery and a 3.3 V pin. R1 is 10 kΩ and R2 is 5.6 kΩ. Vout = 9 × 5.6 / (10 + 5.6) = 50.4 / 15.6 = 3.23 V, which is safe for the pin. The divider draws 9 V / 15.6 kΩ = 0.58 mA from the battery all the time. For a battery monitor that is acceptable. For a battery that must last months in a sleeping robot, it is not, and you would use larger resistors or switch the divider off with a transistor.

Now the example from the exercises. A 4.7 kΩ resistor and a 10 kΩ resistor divide 5 V. Vout = 5 × 10 / (4.7 + 10) = 50 / 14.7 = 3.401 V. You will check that number with a multimeter later.

The common mistake is connecting a load to the middle node. If you attach a 10 kΩ input to the example divider, the lower leg is now 10 kΩ in parallel with 10 kΩ, which is 5 kΩ. The output falls to 5 × 5 / (4.7 + 5) = 25 / 9.7 = 2.58 V, about 0.8 V lower than before. You spot the problem when the measured value depends on what is plugged in. The fix is a buffer, covered under signal conditioning.

Dividers are everywhere in robotics. The battery sense line on many motor controllers and the battery monitor on a typical robot board are both a divider feeding an ADC pin.

---

### Sampling and the ADC

A film camera records a moving scene as still frames. An ADC does the same to a voltage. It takes a snapshot, holds the value steady, converts it to a binary number, and then takes the next snapshot. The time between snapshots sets the sampling rate, and the snapshot count per second sets the highest frequency the reading can represent.

The governing rule is the sampling theorem, which is documented under the Wikipedia article for the Nyquist rate. A signal that changes at frequency f needs a sampling rate greater than 2f to be recovered without distortion. If you sample a 60 Hz hum at 100 Hz, the samples alias into a false, lower frequency. For this reason, an ADC input is usually preceded by a low-pass filter that removes anything above half the sampling rate.

The ADC has its own speed limit. The Arduino Language Reference for analogRead states that a single read takes about 100 microseconds on the Uno. That allows at most about 10,000 reads per second, and the real value is lower once you add loop overhead. For a robot wheel encoder that can run at thousands of pulses per second, the ADC is far too slow, which is why encoders use digital counters, not ADC inputs.

Here is a worked example. A servo's position feedback sits on a potentiometer that moves at up to 60 degrees in 0.15 seconds, which is roughly 400 degrees per second. Its highest useful frequency is about 2 Hz for a slow sweep. With a 100 Hz sampling rate, the Nyquist condition is satisfied with margin. With a 100 Hz motor noise pickup, it is not, and the filter in the signal conditioning section becomes necessary.

The common mistake is to measure a signal once and treat it as the truth. A single read captures one instant, which might be during a switching spike from a nearby motor. The way to spot this is to plot consecutive samples against time. If the values jump around by more than the sensor's specification, the single read is not trustworthy. Averaging several reads, covered under noise, is the usual remedy.

Real systems sample at fixed intervals in a timer interrupt, so the timing is consistent. The Arduino loop with a delay() call does not sample at a fixed rate, and a control loop built that way will drift. Robot firmware for balancing robots and drone flight controllers uses hardware timers for exactly this reason.

---

### Resolution and the DAC

A ruler marked in millimetres measures length more precisely than one marked in centimetres. The ADC's resolution is the number of marks on its ruler. A 10-bit ADC has 2 to the power of 10, which is 1,024 levels. A 12-bit ADC has 4,096. The voltage per level is the reference voltage divided by the number of levels. That step, the LSB, sets the smallest change the ADC can see.

The ideal mapping for a unipolar ADC is code = floor(Vin / Vref × 2^N), where N is the number of bits. The reading ranges from 0 to 2^N − 1. The Arduino Uno's ATmega328P has a 10-bit ADC, with a default reference of the 5 V supply. Its LSB is 5 / 1024 = 4.88 mV. The Raspberry Pi Pico's RP2040 has a 12-bit ADC with a 3.3 V reference, so its LSB is 3.3 / 4096 = 0.806 mV. The Pico is about six times finer, but only if the input signal is not already noisier than that step, and only if you do not feed it more than 3.3 V, which would damage the pin.

A DAC runs the process in reverse. It takes a binary code and produces a voltage. An 8-bit DAC with a 5 V reference has a 5 / 256 = 19.5 mV step. Many microcontrollers have a built-in DAC, but the Uno does not. On an Uno you produce an analog-like voltage with pulse-width modulation (PWM) and a low-pass filter, which Day 13 covers in detail.

Here is the worked example for today's divider. The 4.7 kΩ and 10 kΩ divider outputs 3.401 V. The Uno reads it as code = floor(3.401 / 5 × 1024) = floor(696.6) = 696. Converting back gives 696 × 5 / 1024 = 3.398 V. The 3 mV gap is less than one LSB, which is the best any 10-bit ADC can do. The Pico, fed by a divider scaled so its full range is 3.3 V, has a step about one sixth of the Uno's, so the same measurement is resolved far more finely.

The mistake to watch for is quoting resolution as accuracy. A 12-bit ADC gives 4,096 steps, but its measurement can still be off by tens of LSB because of offset, gain error and noise. Read the datasheet's accuracy specification, which lists both, and do not assume that more bits means a more accurate sensor. On the job, the same question comes up when a team chooses an ADC for a force sensor: the sensor's noise and nonlinearity usually limit the usable bits long before the converter does.

---

### Noise

A radio tuned between two stations gives a hiss. That hiss is noise: a signal that carries no information about the station you want. Every electronic measurement has noise. Some is thermal, from random motion of electrons in resistors. Some is quantization noise, from rounding a continuous voltage to the nearest LSB. Some is interference, from motors, switch-mode converters and radios that pick up signals through the air or the power rail.

The mathematical handle on random noise is the standard deviation. If you take N readings of a signal with uncorrelated random noise and average them, the noise in the average falls by the square root of N. Averaging 16 readings reduces random noise by a factor of 4. Averaging 100 readings reduces it by a factor of 10. This is why averaging is the first tool you reach for, and why it has a cost: more samples take more time.

A worked example makes the trade concrete. A distance sensor's output has a random noise of 20 mV standard deviation. A single read therefore varies by about ±20 mV. Averaging 16 reads brings that to 20 / 4 = 5 mV. Averaging 64 reads brings it to 20 / 8 = 2.5 mV. The ADC's LSB on a 5 V, 10-bit system is 4.88 mV, so 16 reads are enough to see the noise is smaller than one LSB, and 64 reads are overkill for this sensor. Averaging does not remove slow drift or bias, only random variation, which is the limitation to remember.

Interference is different. A 50 Hz or 60 Hz hum from mains wiring does not average out quickly unless the sample count spans whole cycles. The clean fix is to route the signal wire away from motor wires, to add a ground plane or shielded wire, and to filter the signal. The common mistake is to route a sensor cable along a motor cable for 30 cm, then blame the sensor. You spot it by unplugging the motor. If the noise vanishes, the wiring is the problem, not the sensor.

Robots feel noise most strongly during motor starts. A DC motor's brushes make sparks and its current spikes, and both show up on the 5 V rail and on nearby sensor wires. Day 12 covers the power-rail fixes, and Day 13 covers the motor-side fixes. In industry, the same measurement problem appears in every sensor that sits near a motor, from the current sense on a servo drive to the encoder line on a warehouse robot.

---

### Filters

A filter is a circuit that treats slow and fast changes differently. The simplest analog filter is a resistor followed by a capacitor. Picture a bucket with a small hole. Water poured in at a steady rate fills the bucket to a steady level. A sudden splash raises the level only briefly before the water drains away. The capacitor is the bucket, and the resistor sets how fast it drains.

The relation comes from the charging equation of an RC circuit. A resistor R in series with a capacitor C forms a time constant τ = R × C. At frequencies much lower than 1 / (2πRC), the capacitor has high impedance relative to the resistor, so most of the signal reaches the output. At frequencies above that point, the capacitor shorts the high-frequency content to ground. The cutoff frequency is:

fc = 1 / (2π × R × C)

Take R = 10 kΩ and C = 100 nF. RC = 10,000 × 0.0000001 = 0.001 seconds, so fc = 1 / (2π × 0.001) = 159 Hz. Signals above 159 Hz lose about half their power, and a 50 Hz hum passes almost unchanged. For a slow sensor such as a temperature probe, you want a cutoff near 1 Hz, so you pick R = 10 kΩ and C = 1 µF: RC = 0.01 seconds, fc = 1 / (2π × 0.01) = 15.9 Hz. That is still too high for a 1 Hz goal, so for a 1 Hz goal with the same resistor, C must be about 16 µF. A standard 22 µF part gives fc = 1 / (2π × 0.22) = 0.72 Hz. The arithmetic is the same each time: compute RC, then divide into 1 / (2π).

The filter is only as good as its load. The divider problem returns here. If the ADC input draws current, the RC network is loaded and its cutoff shifts. The standard fix is a buffer, which is the next concept. The limit is that a first-order RC filter rolls off slowly, at 20 dB per decade. Above the cutoff, a signal at 10 times the cutoff is attenuated only to about one tenth of its amplitude, which is often not enough. Sharper filters need more components or digital processing.

The common mistake is to pick a capacitor value without checking the voltage rating or the leakage. A 1 µF electrolytic capacitor on a 5 V sensor line can leak enough current to shift a divider, and an undersized ceramic can lose much of its capacitance at its rated voltage. Check the capacitor's voltage rating with a safety margin, and use a film or X7R ceramic for analog work.

In practice, every commercial sensor board has a low-pass filter on its output, usually an RC pair like the one you will build today, and the robot's own controller often adds a digital filter on top.

---

### Signal conditioning

A sensor's output rarely matches the input range of the ADC. A thermistor may swing from 0.8 V to 4.2 V, which is fine for a 5 V Uno. A strain gauge may produce 2 mV, which is far below one LSB. Signal conditioning is the set of circuits that bring the sensor's output into a usable range and form before the ADC reads it. The four basic operations are scaling, offsetting, buffering and filtering.

Scaling is the divider from earlier. Offsetting shifts the signal up or down so the useful part sits inside the ADC range. Buffering copies the voltage without loading it, using an op-amp in voltage-follower mode, where the output is wired to the inverting input and the signal enters the non-inverting input. The output equals the input, but the buffer can drive a load the source could not.

Here is a worked example of why buffering matters. The 4.7 kΩ and 10 kΩ divider from earlier, loaded by a 10 kΩ ADC input, drops the reading from 3.40 V to 2.58 V. If you add a voltage follower between the divider's middle node and the ADC pin, the ADC sees 3.40 V again, because the follower draws almost no current. The op-amp needs a supply that covers the signal range, so a rail-to-rail op-amp such as the MCP6002 from Microchip is a sensible choice when running from 5 V. Check the datasheet for the input and output voltage range before you pick one.

Every conditioning stage adds error. An op-amp has an input offset voltage, a bias current and a limited bandwidth. A signal conditioning chain that is accurate on the bench may drift when the temperature changes. For a robot's distance sensor, a few tenths of a volt of error may be harmless. For a load cell in a gripper that must hold an egg, it is not.

The mistake is to design the conditioning on paper with ideal parts. Always simulate or prototype the full chain with the real part numbers and measure the output at the ADC pin. Signal conditioning lives inside the sensor modules that industrial robots use, such as the current-sense amplifiers in motor drives, and the teams that build those modules spend much of their time on exactly this problem.

---

## Hands-on exercises

Work through these in order. Each one ends with a value you can check.

1. **Build the divider and measure it.** On a breadboard, connect a 4.7 kΩ resistor from the Uno's 5 V pin to a node, and a 10 kΩ resistor from that node to GND. Measure the node voltage with the multimeter in DC volts mode. Your reading should be within about 0.05 V of 3.401 V. If it differs by more than that, measure each resistor with the multimeter's ohms mode and recalculate. Record the measured resistance values, because resistors often differ from their colour-code value by 1 to 5 percent.

2. **Read a potentiometer through the ADC.** Replace the lower resistor with a 10 kΩ potentiometer, wired with its outer legs to 5 V and GND and its wiper to A0. Upload the logger sketch in the project below, which prints the time, the averaged ADC count and the voltage. Turn the knob slowly from one end to the other. Confirm that the count goes from near 0 to near 1023 and that the voltage tracks the multimeter within about 10 mV.

3. **Measure the noise floor.** With the wiper connected and the knob fixed, record 200 readings at 100 ms intervals without averaging. Compute the standard deviation in a spreadsheet or in Python. Then change the sketch to average 16 readings and repeat. The standard deviation should fall by about a factor of 4. If it falls by much less, you probably have interference on the wire, so move the wire away from the USB cable and the power supply.

4. **Add the RC filter.** Insert a 10 kΩ resistor between the wiper and A0, and a 1 µF capacitor from A0 to GND. The cutoff is about 15.9 Hz. Repeat step 3 with no software averaging. The standard deviation should be lower than in step 3 without averaging. Record how much, and whether it matches your expectation from the theory.

5. **Port the measurement to the Pico.** Connect the same divider, but scale it so the maximum voltage is 3.3 V: use 10 kΩ on the top and 20 kΩ on the bottom, which gives 3.33 V for a 5 V input. Feed the divider from the 5 V pin, and check with the multimeter that the Pico's ADC pin never sees more than 3.3 V. Use MicroPython and the machine.ADC class on GP26, which is ADC0 on the Pico. Read the value with read_u16(), which returns a 16-bit scaled number, and convert it to volts using 3.3 / 65535 as the scale. Compare the Pico's reading with the Uno's. The Pico should show less step size, but it may show more noise.

6. **Write the resolution table.** On paper, make a table with three columns: bits, number of levels, and LSB in millivolts for a 5 V reference and a 3.3 V reference, for 8, 10 and 12 bits. Check your values against the figures in the resolution section.

---

## Project: a noise-aware analog logger

The project is light on mechanics and heavy on measurement. It is a small measurement system that reads one analog sensor, averages and filters it, and records the results in a form you can analyse, which is the same pattern a robot uses for its battery monitor or its distance sensor.

**Goal:** build a logger that reads an analog voltage on an Arduino Uno, reports the averaged value over serial in a comma-separated format, and produces a noise report comparing the raw and filtered signal. The goal is to show that you can predict the behavior of a signal chain on paper and then confirm it with a measurement.

**Repo layout:**

```
100-days-robotics-engineering-day-11-analog-signals/
├── README.md
├── firmware/
│   └── analog_logger/
│       └── analog_logger.ino
├── analysis/
│   ├── noise_report.py
│   └── requirements.txt
├── data/
│   ├── raw_no_filter.csv
│   └── raw_with_rc_filter.csv
├── calculations/
│   └── divider_and_resolution.md
└── images/
    ├── divider_schematic.png
    └── breadboard_setup.jpg
```

**Starter code:** `firmware/analog_logger/analog_logger.ino`

```cpp
const int SENSOR_PIN = A0;
const int SAMPLE_COUNT = 16;
const float VREF = 5.0;  // Measure the real 5 V rail with the multimeter and update this value

void setup() {
  Serial.begin(115200);
  Serial.println("millis,average_count,volts");
}

void loop() {
  long sum = 0;
  for (int i = 0; i < SAMPLE_COUNT; i++) {
    sum += analogRead(SENSOR_PIN);
  }
  float averaged = (float)sum / SAMPLE_COUNT;
  float volts = averaged * VREF / 1024.0;

  Serial.print(millis());
  Serial.print(",");
  Serial.print(averaged, 2);
  Serial.print(",");
  Serial.println(volts, 4);

  delay(100);
}
```

**Starter analysis:** `analysis/noise_report.py`

```python
import csv
import statistics

def load_volts(path):
    with open(path, newline="") as f:
        reader = csv.DictReader(f)
        return [float(row["volts"]) for row in reader]

for name in ["data/raw_no_filter.csv", "data/raw_with_rc_filter.csv"]:
    volts = load_volts(name)
    print(f"{name}: n={len(volts)} mean={statistics.mean(volts):.4f} V "
          f"stdev={statistics.stdev(volts) * 1000:.2f} mV")
```

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the wiring, and the list of parts with their measured values.
2. In `calculations/divider_and_resolution.md`, record the divider calculation, the LSB for the Uno and the Pico, and the cutoff frequency of your RC filter. Show the arithmetic.
3. Build the divider and the RC filter on the breadboard. Photograph the setup and save it in `images/`.
4. Upload the sketch and capture 200 readings with the filter removed. Save them to `data/raw_no_filter.csv`, using the serial monitor's logging or a short Python script with the pyserial package.
5. Reconnect the filter. Capture another 200 readings and save them to `data/raw_with_rc_filter.csv`.
6. Run `noise_report.py`. Compare the standard deviations and check them against the prediction for the filter.
7. Add a paragraph to the README explaining one way the measurement could be wrong that the noise numbers do not reveal, such as offset or drift.

**README checklist:**

- [ ] Goal, one paragraph
- [ ] Parts list with measured resistance and capacitance values
- [ ] Wiring diagram or photo
- [ ] Calculations for the divider, LSB and cutoff frequency
- [ ] Noise report table with the two standard deviations
- [ ] One limitation of the measurement

**Done when:**

- The measured divider output is within 0.05 V of your calculation, or you have explained the difference in the README.
- The noise report shows the filtered standard deviation is lower than the unfiltered one, and the ratio is compared with the prediction.
- The repo runs `noise_report.py` without errors on a fresh clone.
- Every number in the README can be traced to a calculation or a file in `data/`.

**Evidence to commit to GitHub:** the firmware, the analysis script, the two CSV files, the calculations file, the photo and the README with the noise table. A reviewer should be able to reproduce your result from the repo alone.

This project is light on new hardware because the lesson's purpose is measurement discipline. Nothing here drives a motor or stores much energy, so the electrical risk is low, but keep the Pico below 3.3 V on every pin.

---

## Check your understanding

1. A 12-bit ADC and a 10-bit ADC both use a 3.3 V reference. A colleague says the 12-bit one is always more accurate. Explain why that may be false, and name the two datasheet specifications you would check before deciding.
2. You average 64 readings of a sensor whose random noise has a standard deviation of 16 mV. What is the new standard deviation, and why would averaging not fix a 30 mV offset error?
3. A 10 kΩ and 10 kΩ divider runs from 9 V. A load of 20 kΩ is connected to the midpoint. Calculate the midpoint voltage with and without the load, and explain which number your robot firmware should trust.
4. You want a first-order RC low-pass filter with a cutoff near 2 Hz. Choose a resistor of 10 kΩ and compute the capacitor value. Explain why the capacitor's voltage rating matters for the choice.
5. A motor starts and the distance readings jump by 80 mV for half a second. Name two likely causes and describe a test that separates them.
6. Explain why an ADC input connected to a microcontroller pin rated for 3.3 V must never receive 5 V, even briefly, and describe the circuit you would add to make a 5 V sensor safe to read.

---

### Answers

1. The 12-bit converter has finer steps, 0.81 mV against 3.2 mV for the 10-bit one, but finer steps do not guarantee accuracy. Check the offset error and the gain error, along with the effective number of bits or the noise figure, because those set the real accuracy. A converter with a large offset gives a precise but wrong number.
2. The standard deviation falls to 16 / √64 = 16 / 8 = 2 mV. Averaging only reduces random variation. A fixed 30 mV offset is the same in every reading, so averaging keeps it. Calibration against a known reference removes an offset.
3. Without the load, the midpoint is 9 × 10 / 20 = 4.5 V. With the 20 kΩ load, the lower leg is 10 kΩ in parallel with 20 kΩ, which is 6.67 kΩ. The output is 9 × 6.67 / (10 + 6.67) = 60 / 16.67 = 3.60 V. The loaded value is the one the ADC sees, so firmware should trust the measured value, and the design should add a buffer if the load is meant to be ignored.
4. C = 1 / (2π × R × fc) = 1 / (2π × 10,000 × 2) = 1 / 125,664 ≈ 7.96 µF. A standard 10 µF capacitor gives fc = 1.59 Hz, which is close. The voltage rating matters because a capacitor used near its rated voltage can lose capacitance, and electrolytic types leak, so a margin of at least double the working voltage is typical.
5. Two likely causes are a shared ground or supply sag from the motor current, which couples into the ADC's reference, and electromagnetic pickup on the sensor wire. A test: power the motor from a separate supply and watch whether the jumps remain. Then move the sensor wire away from the motor wire. If the jumps go away with separation, it is pickup; if they remain with the separate supply, the shared rail is the cause.
6. The pin's internal protection diodes conduct when the input exceeds the supply, which can damage the chip and let current flow into the supply rail. Make a 5 V sensor safe with a voltage divider that scales the output down, plus a clamp or a Zener diode for transient protection, and check that the divider output stays within the pin's rated range.

---

## Sources to read and watch

*Read*

- [Wikipedia: Analog signal](https://en.wikipedia.org/wiki/Analog_signal) for the definition and the contrast with digital signals.
- [Wikipedia: Voltage divider](https://en.wikipedia.org/wiki/Voltage_divider) for the derivation and the loading effect.
- [Wikipedia: Analog-to-digital converter](https://en.wikipedia.org/wiki/Analog-to-digital_converter) for resolution, sampling and the main converter types.
- [Wikipedia: Nyquist rate](https://en.wikipedia.org/wiki/Nyquist_rate) for the sampling condition.
- [Wikipedia: Low-pass filter](https://en.wikipedia.org/wiki/Low-pass_filter) for the RC cutoff derivation.
- [Wikipedia: Signal conditioning](https://en.wikipedia.org/wiki/Signal_conditioning) for the overview of conditioning stages.
- The ATmega328P datasheet from Microchip, which lists the ADC specification and the I/O pin thresholds. Search "ATmega328P datasheet Microchip".
- The Raspberry Pi Pico datasheet, which describes the RP2040's ADC inputs and their 3.3 V limit. Search "Raspberry Pi Pico datasheet" on raspberrypi.com.
- Arduino Language Reference, the pages for analogRead and the sketch structure. Site: docs.arduino.cc.

*Tutorials*

- SparkFun Learn, the tutorials on voltage dividers and analog-to-digital conversion. Search "SparkFun Learn voltage dividers".
- All About Circuits, the lessons on voltage dividers and on RC filters. Search "All About Circuits RC low-pass filter".
- Adafruit Learning System, the guide on analog inputs and the Arduino analogRead function. Search "Adafruit analog inputs Arduino".

*Watch*

- MIT OpenCourseWare, 6.002 Circuits and Electronics. The lectures on the RC circuit and the op-amp are the right depth for today.
- Ben Eater's YouTube channel. Search his videos on analog-to-digital conversion and on building a DAC on breadboard.

*Practice*

- Wokwi, the free browser simulator. It supports the Arduino Uno and the Raspberry Pi Pico. Build the divider and read the ADC before you build it on a breadboard.

---

## Tomorrow

Today's divider and filter assumed a steady supply rail. Tomorrow you build the power system that provides it. You will size a battery pack by capacity and current, choose between a linear regulator and a buck converter, and place decoupling capacitors so the sensor noise you measured today does not come from the power rail. The noise you measured this afternoon will show you where the rail is leaking into your sensors, and Day 12 is where you fix it.

**Next:** [Day 12: Robot Power Systems](Robotics-Day-12-Robot-Power-Systems.md)
