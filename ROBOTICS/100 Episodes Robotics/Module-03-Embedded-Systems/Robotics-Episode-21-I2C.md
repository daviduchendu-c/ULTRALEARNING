# Day 21: I²C

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 4 hours

**Level:** intermediate. You need Day 16 (the Uno and Arduino IDE), Day 17 (pull-up resistors), Day 19 (why a bus transaction can be interrupted) and Day 20 (framing, baud rate and the logic-level rules). Today you connect real sensors, so check the voltage of each device before you power it.

**By the end of this day, you will have:**

- A working I2C bus with at least two devices, found with a scanner sketch that prints each address in hexadecimal
- A real-time clock read over I2C, with its time compared with your computer's clock and its temperature register decoded
- A measured pull-up resistance and an idle bus voltage, recorded with and without the pull-ups, with the result explained
- A short bus design note that lists addresses, pull-up values and the total bus capacitance, with the source for each number

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3.x and the `Arduino AVR Boards` core at version 1.8.6. The `Wire` library is included with the core.
- A DS3231 real-time clock breakout, about $3 to $6. Many breakouts include a 4.7 kOhm pull-up pair and a CR2032 battery holder. Check the board's documentation for the pull-up values before you use it.
- Optional: a 128 by 64 SSD1306 OLED display on I2C, about $4 to $6, to use as a second device on the bus. It usually uses address 0x3C or 0x3D.
- Two 4.7 kOhm resistors if your breakout has no pull-ups, about $0.20
- A breadboard and jumper wires, about $5 to $8
- A digital multimeter with DC volts and ohms modes, about $15 to $25
- Optional: a logic analyzer with I2C decoding, about $10 to $20

**Prerequisites:**

- Day 16: `Serial.begin`, the serial monitor and the register view of pins
- Day 17: pull-up resistors and the active-low convention
- Day 19: why a transaction must not be interrupted by unrelated code, and the blocking-versus-non-blocking pattern
- Day 20: logic levels between boards and the rule that devices must share a ground

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Two wires and open-drain signaling](#two-wires-and-open-drain-signaling)
- [Addresses and transactions](#addresses-and-transactions)
- [Pull-up resistors and bus capacitance](#pull-up-resistors-and-bus-capacitance)
- [Bus topology and multiple devices](#bus-topology-and-multiple-devices)
- [Debugging an I2C bus](#debugging-an-i2c-bus)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an I2C bus scanner and real-time clock monitor](#project-an-i2c-bus-scanner-and-real-time-clock-monitor)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A house with one telephone line for every room needs a lot of wire. A house with one shared line and an address for each phone needs much less. I2C is the shared-line design for chips. Two wires connect the microcontroller to every sensor, display and clock on the bus, and each device answers only to its own address. A robot may have an inertial sensor, a battery monitor, a display and a clock on the same two wires, and the wiring takes a few centimeters of board space instead of dozens of separate lines.

The bus is convenient, and it is also easy to break. Two devices with the same address will fight for the bus. A missing pull-up resistor leaves the lines floating. A long cable adds capacitance that slows the edges until the data is corrupted. These problems look like software bugs, and engineers spend hours on code before they check the wiring. The rules that prevent them are short, and today you will test each rule on a real bus.

The project is a bus scanner and a real-time clock reader. The scanner finds the devices, and the clock reader shows what a working device returns. The measurements you make on the pull-ups and the bus voltage are the evidence that the wiring follows the specification.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1982 | Philips (now NXP) introduces the I2C bus to connect chips within a TV set, so that a few wires could control many devices. | The two-wire bus idea is still the way most sensors and clocks connect to a microcontroller. |
| 1990s | The 7-bit addressing scheme and the 100 kbit/s standard mode become common in embedded designs. | The Uno's default I2C speed is still 100 kbit/s. |
| 1998 | The fast mode at 400 kbit/s is added, and the specification is revised in later years. | Most sensors support 400 kbit/s, which the Uno can run with `Wire.setClock`. |
| 2000s to 2010s | Fast-mode plus at 1 Mbit/s and high-speed mode at 3.4 Mbit/s are added for faster devices. | High-speed sensors on modern robots use these modes. |
| Today | I2C remains the default low-speed bus for sensors, clocks, power monitors and small displays on robot boards. | Almost every robot board has one or more I2C buses. |

The specification has been updated several times, but the core design, two lines with open-drain drivers, has not changed. The official document is the NXP I2C-bus specification, which is cited in the sources.

---

## Glossary

- **I2C (Inter-Integrated Circuit):** a two-wire bus that connects one or more controllers to many devices, each with an address.
- **SDA:** the serial data line, which carries the address and data bits in both directions.
- **SCL:** the serial clock line, which the controller drives to pace the bits.
- **Controller (master):** the device that starts and ends transactions and drives the clock. The Uno is the controller in this lesson.
- **Target (slave):** a device that responds to a transaction addressed to it.
- **Open-drain:** an output that can pull a line low but cannot drive it high. A pull-up resistor pulls the line high.
- **Pull-up resistor:** the resistor that pulls SDA and SCL high when no device is pulling them low.
- **Start condition:** SDA falling while SCL is high, which begins a transaction.
- **Stop condition:** SDA rising while SCL is high, which ends a transaction.
- **ACK and NACK:** the acknowledge bit that a target sends after each byte (ACK, low) or fails to send (NACK, high).
- **Repeated start:** a new start condition sent without a stop, which lets the controller read after writing a register address.
- **Bus capacitance:** the total capacitance on the bus, from the wires, the devices and the pins. It slows the rising edges of the lines.
- **BCD (binary-coded decimal):** a way to store decimal digits in four bits each, used by many real-time clock chips.

---

### Two wires and open-drain signaling

A pair of traffic lights that can each show only red or green needs a clear rule about who can change the light. I2C uses a similar rule for the wires. Each line is open-drain: a device can pull it to ground, but it cannot push it up. The high level comes from the pull-up resistor. The rule means no two devices can fight over a line in a way that burns out a driver, because a device only pulls low, and when everyone releases the line, it goes high.

From first principles, when a device wants to send a 0, it pulls the line low through its transistor. When it wants to send a 1, it releases the line, and the pull-up raises it. If one device pulls low while another releases, the line reads low. This wired-AND behavior is why clock stretching works: a target that is busy can hold SCL low, and the controller waits until the target releases it.

The worked example is the idle state. With 4.7 kOhm pull-ups on a 5 V bus and no device pulling low, each line sits at 5 V. If one device pulls SDA to ground, the pull-up carries a current of 5 / 4,700 = 1.06 mA through the device's transistor. A device that can sink 3 mA is safe with this pull-up, and a weaker pull-up of 1 kOhm would require 5 mA and might exceed the sink rating of a small sensor.

The common mistake is to think I2C lines can be driven by a push-pull output from a microcontroller that also has an I2C peripheral. On the Uno, the hardware is designed for open-drain operation, and the wires must have pull-ups. A line driven high by a push-pull output from a device on a 3.3 V bus, and pulled low by a 5 V device, can damage the lower-voltage side. The check is to confirm the voltage on the datasheet of each device before you join it to a shared bus.

On the job, the open-drain rule is what lets many devices share two wires without a central arbiter. Engineers rely on it for sensors in cars, phones and industrial controllers.

---

### Addresses and transactions

A postal system delivers letters by reading the address on the envelope. Each house sees the envelope, and only the one with the matching address opens it. An I2C transaction starts with an address byte, and each device compares that byte with its own address. The match is the only thing that makes it respond.

From first principles, the standard addresses are seven bits, so there are 128 possible values, and a few are reserved. The eighth bit of the address byte is the read/write bit: 0 for a write from the controller, and 1 for a read. A transaction begins with a start condition, the address byte, an ACK from the target, one or more data bytes each with an ACK, and a stop condition. A read after a register write uses a repeated start instead of a stop, so the bus stays under the controller's control.

The worked example uses the DS3231. Its 7-bit address is 0x68. To write to it, the controller sends the byte (0x68 << 1) | 0 = 0xD0. To read from it, the controller sends (0x68 << 1) | 1 = 0xD1. The Arduino `Wire` library hides this shift, so the code uses 0x68 and the library builds the byte. Knowing the shift matters when you read a logic analyzer trace, because the trace shows 0xD0 and 0xD1, not 0x68.

The common mistake is to use the 8-bit address from a datasheet in the Arduino code. Many datasheets list addresses such as 0xD0 for the write byte, and the library expects the 7-bit value 0x68. Using 0xD0 makes the device appear to be missing, and the scanner will show nothing. The check is to look at the datasheet's address table and find the 7-bit value.

On the job, address conflicts are a common reason a new board fails. Two sensors that share the default address, such as the MPU-6050 and the DS3231 both at 0x68, cannot coexist on one bus without a change to an address pin.

---

### Pull-up resistors and bus capacitance

A bridge that has to carry heavy traffic needs a strong road surface, and a bridge that is too long needs more support. The pull-up resistor and the bus capacitance are the same kind of trade-off on an I2C bus. The pull-up must be strong enough to raise the line quickly, and the bus must not hold so much capacitance that the rise is too slow.

From first principles, the line is a resistor and a capacitor in series with the pull-up. The rise time is roughly t_r = 0.8473 × R × C_b, where C_b is the bus capacitance. The NXP specification sets a maximum rise time of 1,000 ns for standard mode and 300 ns for fast mode. A smaller pull-up gives a faster rise, but it also increases the current a device must sink when pulling low. The specification gives the largest allowed pull-up as R_max = t_r / (0.8473 × C_b) and the smallest as R_min = (V_DD − V_OL) / I_OL.

The worked example uses a bus with 200 pF of total capacitance, which is typical for a few devices and a short cable. For standard mode, R_max = 1,000 × 10^-9 / (0.8473 × 200 × 10^-12) = 5,900 ohm. A 4.7 kOhm pull-up is below that limit, so the rise time is within the standard. The bus capacitance limit in the specification is 400 pF for standard and fast modes, and a longer cable or more devices can exceed it.

The common mistake is to add a second pair of pull-ups when a breakout board already has them. Two 4.7 kOhm resistors in parallel give 2.35 kOhm, which is still valid, but the pull-ups are now heavier than the design intended and the sink current doubles. More serious, a pull-up added to a device with a built-in pair can cause a problem on a bus with an unusual voltage. The check is to measure the resistance from SDA to VCC with the bus unpowered. Two pairs in parallel show a lower value than expected.

On the job, pull-up choices are part of the schematic review. An engineer chooses the value from the bus speed, the cable length and the number of devices, and records the calculation in the design document.

---

### Bus topology and multiple devices

A town with a single main road and side streets is easy to plan. The traffic on each street is small, and the main road carries all the traffic. An I2C bus is usually a single main road with short branches to each device. The design works best when the devices are close to the controller and the wires are short.

From first principles, the bus is a shared line with many devices attached, and each device adds a small input capacitance and a leakage current. The total capacitance is the sum of the wiring, the connectors and the device pins. A bus with ten devices on 30 cm of ribbon cable may reach 400 pF, which is the specification limit. A bus with a long cable or a connector at each device can exceed it, and then the rise times fail.

The worked example is adding a device with a conflicting address. If two devices share 0x68 and both respond to the same transaction, they both pull SDA low for the ACK, and the controller sees a valid ACK. The read returns a mixed byte, and the data looks corrupted. The error repeats every time, which points to a conflict. It can be found by scanning the bus and noticing the same address appearing twice in a list.

The common mistake is to run I2C on a long cable without testing the bus capacitance and rise time. A cable of two meters can work on the bench and fail in a robot with motors nearby, because the motor noise adds to the edges. The check is to measure the rise time with a logic analyzer or an oscilloscope on the bus under load.

On the job, a bus topology review checks the address map, the pull-up values, the capacitance estimate and the cable routing. Those four items prevent most of the field failures on I2C designs.

---

### Debugging an I2C bus

A mechanic who checks a car's engine begins with the battery, the fuel and the spark, before the expensive parts. An I2C bus has a similar order of checks: power, wiring, pull-ups, addresses and then software. The Wire library's return codes give a quick first answer, because they tell you which step failed.

From first principles, `Wire.endTransmission()` returns a code that tells you what happened. Zero means success. One means the data was too long for the library's buffer. Two means the target did not acknowledge its address, which usually means no device at that address. Three means the target did not acknowledge a data byte, which points to a register that the device does not accept. Four means another error, and in practice that is often a stuck bus. A stuck bus is one where SDA or SCL is held low by a device that stopped in the middle of a transaction, and the fix is often a power cycle of that device.

The worked example uses the scanner. A scanner that prints nothing on a bus with a DS3231 usually has one of three causes: the wires are swapped, the pull-ups are missing, or the device is not powered. A scanner that prints 0x57 when you expected 0x68 suggests a device with an address that you did not check against the datasheet.

The common mistake is to change the software before measuring the bus. A multimeter on the SDA and SCL lines with the bus idle should read near VCC. A reading near 0 V means a device is holding the line low, and a reading near half of VCC suggests a missing pull-up or a line that is not driven.

I2C debugging is one of the most common tasks in robot integration. A reviewer who sees the scanner log and the bus measurements can judge a design quickly, and that is why the project asks for both.

---

## Hands-on exercises

1. **Wire the bus.** Connect the DS3231 to the Uno: VCC to 5 V (or 3.3 V if the breakout is 3.3 V only), GND to GND, SDA to A4 and SCL to A5. Check the breakout's pull-up value with the multimeter in ohms mode, with the board unpowered.

2. **Measure the idle bus.** Power the board and measure SDA and SCL against GND with the bus idle. Record both values. They should be close to the supply voltage.

3. **Run the scanner.** Upload the scanner sketch from the project. Record the addresses found. Look up each one in the datasheet and confirm the match.

4. **Read the clock.** Upload the clock reader and compare the time it prints with your computer's clock. Record the difference after ten minutes. Note any drift.

5. **Remove a pull-up.** Disconnect the pull-up on SCL and run the scanner. Record what happens. Reconnect and confirm the scanner works again.

6. **Add a second device.** If you have an OLED display, connect it to the same bus and run the scanner again. Confirm that both addresses appear. Many OLED boards have a solder jumper that selects 0x3C or 0x3D. If yours does, change it and confirm the scanner shows the new address. Do not try to force a conflict on a real device, because two devices answering to one address can leave the bus stuck, and you would then need a power cycle to recover.

---

## Project: an I2C bus scanner and real-time clock monitor

The goal is a repository that scans the bus, reads a real-time clock and records bus measurements. The project proves that you can work with a two-wire bus, read device registers, decode BCD values and check your wiring against the specification.

**Repo name:** `100-days-robotics-day-21-i2c`

**Repo layout:**

```
100-days-robotics-day-21-i2c/
├── README.md
├── scanner/
│   └── scanner.ino
├── rtc_monitor/
│   └── rtc_monitor.ino
├── logs/
│   ├── scan_output.txt
│   ├── rtc_vs_computer_10min.csv
│   └── idle_bus_measurements.csv
├── docs/
│   ├── bus_design.md
│   └── wiring.md
└── photos/
    └── bus.jpg
```

**Starter code, `scanner/scanner.ino`:**

```cpp
#include <Wire.h>

void setup() {
  Wire.begin();
  Serial.begin(115200);
}

void loop() {
  Serial.println("scanning 0x08 to 0x77");
  int found = 0;
  for (uint8_t addr = 8; addr < 120; addr++) {
    Wire.beginTransmission(addr);
    uint8_t err = Wire.endTransmission();
    if (err == 0) {
      Serial.print("found 0x");
      if (addr < 16) Serial.print('0');
      Serial.println(addr, HEX);
      found++;
    }
  }
  Serial.print(found);
  Serial.println(" device(s)");
  delay(5000);
}
```

Writing to an address with no data can wake some devices or trigger a write to a register. For a DS3231 and an OLED, the scanner's empty write is safe, but check the datasheet for any device you add to the bus before you use the scanner on it.

**Starter code, `rtc_monitor/rtc_monitor.ino`:**

```cpp
#include <Wire.h>

const uint8_t RTC_ADDR = 0x68;

uint8_t bcdToDec(uint8_t b) {
  return (b >> 4) * 10 + (b & 0x0F);
}

bool readTime(uint8_t &h, uint8_t &m, uint8_t &s) {
  Wire.beginTransmission(RTC_ADDR);
  Wire.write(0x00);                          // seconds register
  if (Wire.endTransmission(false) != 0) return false;
  if (Wire.requestFrom(RTC_ADDR, (uint8_t)7) != 7) return false;
  s = bcdToDec(Wire.read() & 0x7F);
  m = bcdToDec(Wire.read() & 0x7F);
  h = bcdToDec(Wire.read() & 0x3F);
  for (int i = 0; i < 4; i++) Wire.read();  // day, date, month, year
  return true;
}

bool readTemp(float &tempC) {
  Wire.beginTransmission(RTC_ADDR);
  Wire.write(0x11);                          // temperature MSB register
  if (Wire.endTransmission(false) != 0) return false;
  if (Wire.requestFrom(RTC_ADDR, (uint8_t)2) != 2) return false;
  int8_t msb = (int8_t)Wire.read();
  uint8_t lsb = Wire.read();
  tempC = msb + ((lsb >> 6) * 0.25);
  return true;
}

void setup() {
  Wire.begin();
  Serial.begin(115200);
  Serial.println("seconds_since_start,rtc_h,rtc_m,rtc_s,temp_c");
}

void loop() {
  uint8_t h, m, s;
  float t;
  if (readTime(h, m, s) && readTemp(t)) {
    Serial.print(millis() / 1000);
    Serial.print(",");
    Serial.print(h); Serial.print(",");
    Serial.print(m); Serial.print(",");
    Serial.print(s); Serial.print(",");
    Serial.println(t, 2);
  } else {
    Serial.println("i2c error");
  }
  delay(1000);
}
```

The `requestFrom` call returns the number of bytes received, and the code checks it. A missing byte is a common bug that goes unnoticed if you only check the return value of `endTransmission`.

**Step-by-step tasks:**

1. Create the repository and folder tree. Initialize git and commit the README skeleton.
2. Wire the DS3231 and measure the pull-up resistance and the idle bus voltage. Save the values to `logs/idle_bus_measurements.csv`.
3. Upload `scanner`. Run it for one minute and save the output to `logs/scan_output.txt`. Look up each address in the DS3231 datasheet and confirm the match. If you added an OLED, confirm its address too.
4. Upload `rtc_monitor` and run it for ten minutes. Set the DS3231 to your computer's time before the run, using a one-time sketch that writes the BCD registers, or the set-time method from the breakout's documentation. Save the output and compare it with the computer's clock every minute. Record the difference.
5. Remove one pull-up and run the scanner. Record the result in the README. Reconnect and confirm the result is restored.
6. Write `docs/bus_design.md` with the address map, the pull-up value, the calculated R_max for a 200 pF bus, and a note on the total capacitance of your wiring.
7. Write the README with the checklist below.

**README checklist:**

- [ ] The address map lists each device with its 7-bit address and its datasheet source
- [ ] The measured pull-up resistance and the idle bus voltage are reported
- [ ] The R_max calculation is shown with the numbers and the specification constant
- [ ] The clock drift over ten minutes is reported, with the logged values
- [ ] The pull-up removal test is described, with the scanner result
- [ ] The README states which of the three scanner outcomes you observed, and why

**Done when:**

- The scanner finds the DS3231 at 0x68, and any other device you added appears with its datasheet address
- The idle bus measurements are close to the supply voltage with the pull-ups connected, and the README explains any unexpected value
- The clock log covers at least ten minutes, with a computed drift in seconds
- The bus design document includes an R_max calculation and an estimate of the total bus capacitance
- The repository has at least three commits, and every number in the README matches a log file

**Evidence to commit to GitHub:** both sketches, the three logs, the bus design document, the wiring document, the bus photo and the README. Do not commit build folders.

The project is practical and it uses a real device. The evidence shows that the wiring follows the specification, that the software reads the right registers and that the clock can be checked against a reference. Those are the checks a robot integration engineer runs on every new bus.

---

## Check your understanding

1. A sensor datasheet lists its address as 0xD0 for writes. What is its 7-bit address, and what value does the Arduino `Wire` library expect?
2. A bus with four devices scans correctly, but a read from one device returns bytes that change in an odd pattern on each run. List two causes and the test that would separate them.
3. A pull-up of 1 kOhm is used on a 5 V bus with a device that can sink 3 mA. Compute the current when the device pulls the line low, and say whether the design is acceptable.
4. Explain why a repeated start is used when reading a register from an I2C device, and what would change if the controller sent a stop instead.
5. You measure SDA at 0.1 V with the bus idle and the board powered. Give two likely causes and the next step you would take.
6. A student says that adding a 100 nF capacitor to the bus will make it faster. Explain whether that is correct, and what the capacitance does to rise time.

---

## Answers

1. The 8-bit write address 0xD0 corresponds to the 7-bit address 0x68, because the eighth bit is the read/write bit. The Arduino `Wire` library expects 0x68 and adds the bit itself.
2. A device conflict, where two devices share the same address, produces a mixed response. Another cause is a bus that is too noisy or too long, which corrupts some bits. The test is to scan the bus with only one device connected, then add the others one at a time to find the conflict or the noise.
3. The current is 5 / 1,000 = 5 mA, which is more than the 3 mA sink rating. The design is not acceptable. A pull-up of 4.7 kOhm gives about 1.06 mA and is within the sink rating.
4. A repeated start keeps the bus under the controller's control, so the register address and the read happen in one transaction. A stop would release the bus between the two, and another controller could change the register pointer in between.
5. Two likely causes are a device holding SDA low, such as a stuck device, or a short to ground on the wiring. The next step is to disconnect devices one at a time and measure the line after each, to find the one that holds it low.
6. The statement is not correct for speed. A capacitor adds to the bus capacitance, which slows the rising edge, and it does not speed the bus. A larger capacitance increases rise time in proportion to the pull-up value and the capacitance.

---

## Sources

*Read:*

- The NXP I2C-bus specification and user manual, UM10204, free from the NXP website. Read the sections on the bus signals, the timing, the pull-up calculation and the bus capacitance limit
- The Wikipedia article titled **I2C**, for the overview and the history
- The Wikipedia article titled **Open collector** for the open-drain concept. If the exact title redirects, search for "open drain" on Wikipedia
- The Wikipedia article titled **Binary-coded decimal**, for the BCD format used by the clock chip
- The DS3231 datasheet by Analog Devices, free from the Analog Devices website. Read the sections on the register map and the temperature registers
- The Arduino Wire library reference at docs.arduino.cc, for `beginTransmission`, `endTransmission`, `requestFrom` and the return codes

*Tutorials:*

- The official Arduino tutorial on the Wire library, found in the Arduino documentation under Libraries
- The GeeksforGeeks article on the I2C protocol. Search the GeeksforGeeks site for "I2C protocol"

*Watch:*

- The Arduino YouTube channel, which has videos on I2C and the Wire library. Search for "Arduino I2C tutorial"
- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, for lectures on serial buses and device interfacing. Search for the course title on the MIT OpenCourseWare site

*Practice:*

- Wokwi at wokwi.com, which includes an Arduino Uno, an I2C device and a logic analyzer view. Use it to run the scanner before you wire the real bus. The free tier is enough for this lesson, but it does not show real pull-up values, so repeat the measurements on the bench.

---

## Tomorrow

Day 22 covers SPI, a faster bus that uses four wires and a separate select line for each device. You will compare SPI with I2C and UART, read an external analog-to-digital converter over SPI and learn why the chip-select line matters. The bus-design habits from today, including the address map and the capacitance estimate, carry over directly, because SPI has its own capacitance and cable limits.
