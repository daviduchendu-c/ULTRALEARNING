# Day 22: SPI

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 4 hours

**Level:** intermediate. You need Day 16 (the Uno and the register view), Day 17 (chip-select logic and active-low signals), Day 18 (ADC resolution and step size), Day 20 (UART framing) and Day 21 (I2C addresses and the bus trade-offs). The project uses an external analog-to-digital converter, so you will also measure voltages with your multimeter.

**By the end of this day, you will have:**

- An MCP3008 analog-to-digital converter read over SPI from the Uno, with its 10-bit result converted to volts
- A measured SPI clock rate, set with `SPISettings`, and a note on the highest rate the wiring and the chip allow
- A calibration table comparing the MCP3008 reading with a multimeter at five voltages, with the error for each point
- A written comparison of SPI, I2C and UART for a specific robot subsystem, with the reasons for each choice

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3.x and the `Arduino AVR Boards` core at version 1.8.6. The `SPI` library is included with the core.
- One MCP3008 8-channel 10-bit ADC in a DIP-16 package, about $3 to $5. Microchip makes the chip, and the datasheet is free.
- A DIP-16 breadboard adapter or a socket, about $1, if you do not want to use a breadboard-friendly chip directly
- Two 10 kOhm potentiometers, about $2 total, for two analog channels
- A 0.1 uF decoupling capacitor, about $0.10, for the MCP3008 supply pin
- A breadboard and jumper wires, about $5 to $8
- A digital multimeter with DC volts, about $15 to $25
- Optional: a second MCP3008, about $4 to $5, for the multi-device exercise
- Optional: a logic analyzer, about $10 to $20, to view the SPI frame

**Prerequisites:**

- Day 16: the Uno's pins and the register idea
- Day 17: active-low signals
- Day 18: ADC step size and the conversion from reading to volts
- Day 20 and Day 21: framing, the cost of a bus, and the comparison that makes choosing a bus a design task

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Four wires and chip select](#four-wires-and-chip-select)
- [Clock polarity and phase](#clock-polarity-and-phase)
- [Speed and the SPISettings call](#speed-and-the-spisettings-call)
- [Multiple devices on one bus](#multiple-devices-on-one-bus)
- [SPI vs I2C vs UART](#spi-vs-i2c-vs-uart)
- [Reading an external ADC](#reading-an-external-adc)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an MCP3008 voltage logger](#project-an-mcp3008-voltage-logger)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A hallway with a single corridor and a door for each room is a simple design. Everyone in the building can hear the same messages, and a door opens only when a person asks for it. SPI uses that design. A controller sends a clock and a data line to every device, and a separate select line for each device opens that device's door. The result is faster than I2C and simpler than the addressed bus, and it is the most common way to connect fast sensors, memory chips, displays and motor encoders on a robot.

The price of speed is wires and care. SPI needs one chip-select line for each device, and a device that does not get its select line low will ignore the traffic. SPI has no standard for acknowledgments, so the controller cannot know whether a device exists unless it reads a known value. SPI is also sensitive to clock timing, and the clock polarity and phase must match on both ends. Each of these details is a common reason a new SPI link appears to work with one chip and fails with another.

Today you will read an external ADC over SPI. The MCP3008 is a small chip that turns eight analog channels into 10-bit numbers. The reading is simple to check, which makes it a good first SPI device. The project then compares the SPI result with a multimeter reading, so the link is tested against a reference.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| Mid-1980s | Motorola develops and promotes the Serial Peripheral Interface as a simple, synchronous bus between a processor and peripheral chips. | SPI became the standard for fast peripherals, which is why it appears on so many modern boards. |
| 1990s | Microcontrollers from many makers add hardware SPI controllers, which make the protocol cheap to implement. | The Uno's SPI peripheral is in this family. |
| 2000s | SPI flash memory and SD cards use the bus for storage, and many displays connect over it. | Robots use SPI for logging to an SD card and for driving color displays. |
| 2010s to today | SPI speeds rise into the tens of megahertz on sensor and memory chips, and the protocol is used for inertial sensors, encoders and radio modules. | High-rate inertial sensors on modern robots often connect over SPI. |

SPI has no formal specification the way I2C does. The electrical behavior is set by each chip's datasheet, and the protocol is a convention that devices follow. That makes the datasheet the authoritative source for every SPI link you build.

---

## Glossary

- **SPI (Serial Peripheral Interface):** a synchronous bus with four signals: clock, data out, data in and chip select. The controller drives the clock.
- **SCLK (SCK):** the serial clock line, driven by the controller.
- **MOSI:** the controller's data output, which goes to the peripheral's data input. Some datasheets call it SDI on the device side.
- **MISO:** the peripheral's data output, which goes to the controller's data input. Some datasheets call it SDO on the device side.
- **CS (chip select) or SS (slave select):** an active-low line that enables one peripheral at a time.
- **Full-duplex:** a link where data moves in both directions at the same time, because MOSI and MISO are separate lines.
- **CPOL (clock polarity):** the idle level of the clock. Low means idle at 0, and high means idle at 1.
- **CPHA (clock phase):** whether data is sampled on the first or second clock edge after the select line goes low.
- **SPI mode:** one of four combinations of CPOL and CPHA, numbered 0 to 3.
- **MSB first:** the bit order in which data is sent, with the most significant bit first. Most SPI devices use this order.
- **SPISettings:** the Arduino object that sets the clock speed, bit order and mode for a transaction.
- **ADC (analog-to-digital converter):** a circuit that converts a voltage to a number, covered on Day 18.
- **DIP (dual in-line package):** a chip package with two rows of pins, which fits a breadboard.

---

### Four wires and chip select

A class with a teacher, a student and a chalkboard can share a lesson in a clear way. The teacher writes on the board and the student reads it, while a bell sets the pace. SPI has the same parts. The controller sends the clock, the data to the device on MOSI, and the device answers on MISO, while the chip-select line tells the device that the lesson is for it.

From first principles, each bit is moved on one clock edge and sampled on the other. The controller shifts out a bit on MOSI and the device shifts out a bit on MISO at the same time, so one byte written by the controller comes with one byte read back. The chip-select line goes low to start a transaction and goes high to end it. Between those points, the device shifts its registers. Without chip select, two devices on the same MISO line would drive the same wire, which would fight in the same way as two open-drain devices on a broken bus.

The worked example is an eight-bit transfer. The controller sends the byte 0x01 on MOSI, and while it does, the device returns a byte on MISO. The return byte is the device's response to the previous command, not to this one. This is why SPI reads often need one transfer to send the request and a second transfer to read the answer. The MCP3008 reads in exactly this way, as shown below.

The common mistake is to forget the chip select or to leave it floating. A floating select line can enable the device at random, or the device can be selected at the same time as another one. Pull the select line high with a resistor or drive it from a pin that is set high before the bus is used, and bring it low only for the transaction.

On the job, SPI bus debugging starts with the chip-select timing. A logic analyzer shows the select line and the clock together, and the timing diagram in the datasheet tells you what each edge should do.

---

### Clock polarity and phase

A rowing crew needs everyone to pull on the same beat. If they pull on the second beat instead of the first, the boat moves but the stroke is wrong. SPI has the same issue. The controller and the device must agree on which clock edge is used to change the data and which edge is used to read it. That agreement is the SPI mode.

From first principles, CPOL sets the idle level of the clock, and CPHA sets which edge samples the data. There are four combinations, numbered 0 to 3. Mode 0 has the clock idle low and samples on the rising edge. Mode 3 has the clock idle high and samples on the rising edge as well, with the data changing on the falling edge. For the MCP3008, the datasheet specifies mode 0,0, which is the common default. Using mode 3 with a device that expects mode 0 produces data that looks almost right but has bits shifted in time, and the errors change with the clock speed.

The worked example is a mismatch. An MCP3008 read that returns 0x3FF on every channel, even when the pot is at the bottom, can mean that the mode is wrong, or that the chip-select line is not toggling, or that the clock is not reaching the chip. A mode check is the first step, because the mode is the most common cause of a result that is constant and wrong.

The common mistake is to copy an SPI example from a different device without checking its mode. Two devices can share a bus only if they use the same mode, or if the controller changes the mode between transactions. The Arduino `SPISettings` call lets you set the mode for each transaction, which is how multi-device buses are handled.

On the job, the datasheet mode is a required field in every SPI configuration table. A firmware review that does not check the mode against the datasheet will miss the first class of SPI bugs.

---

### Speed and the SPISettings call

A conveyor belt moves boxes at a set speed. If the belt is too fast, the robot arm misses the boxes, and if it is too slow, the line runs below capacity. SPI speed works the same way. The controller sets a clock rate, and the device must be able to keep up with it.

From first principles, the SPI clock on the Uno is derived from the CPU clock by a prescaler. The highest clock is F_CPU / 2, which is 8 MHz for a 16 MHz Uno. The `SPISettings` object takes a rate in hertz, the bit order and the mode, and the call `SPI.beginTransaction` applies them. The rate you choose must not exceed the device's rated maximum, and it must be lower than the rate at which the wiring can carry clean edges.

The worked example is a choice of speed for the MCP3008 at 5 V. The datasheet lists a maximum clock frequency that depends on the supply voltage, and at 5 V it allows a clock of up to about 3.6 MHz. A setting of 1 MHz is safe and easy to measure. A setting of 8 MHz on the Uno would exceed the MCP3008's limit, and the readings would become unreliable. The choice should come from the datasheet.

The common mistake is to set the speed to the highest value the controller allows, without checking the device. This gives readings that are sometimes right, and the fault looks random. The check is to find the maximum clock in the datasheet's electrical characteristics table, and to measure the readings at two speeds. If the readings differ, the faster speed is too fast.

On the job, SPI speed is one of the first numbers in a sensor's integration checklist. Engineers choose a speed that meets the sampling rate, stays within the device limit and leaves margin for the wiring.

---

### Multiple devices on one bus

A building with a shared corridor and a separate key for each office is a good comparison. The corridor carries the traffic, and each key opens one door. SPI devices share the clock, MOSI and MISO lines, and each one has its own chip-select line. Only the selected device responds, and the others ignore the traffic.

From first principles, the shared lines have a cost. Each device adds capacitance and a leakage path to MISO. A device that is not selected must put its MISO output into a high-impedance state, so it does not drive the line. Most SPI devices do this automatically when their chip select is high. A device that does not release MISO can corrupt the reply of another device, and a symptom is a read that returns the wrong channel or a mixed value.

The worked example is two MCP3008 chips on one bus. Each has its own chip select, on pins 10 and 9 of the Uno. Only one is low at a time, and the code sets the other high before the transfer. A reading from the wrong chip returns the channel of the other chip, a select-line error that looks like a wiring fault.

The common mistake is to leave the chip-select line of an unused device low, which enables it on every transfer. The fix is to set every chip-select pin to output and high in `setup()` before any transfer.

On the job, a multi-device SPI bus is designed with a chip-select map that names each pin. Errors in that map are a common source of intermittent faults in robots with several sensors on one bus.

---

### SPI vs I2C vs UART

A carpenter chooses a screwdriver, a hammer or a drill for each task. The tool that fits the job is faster and safer than the one that is merely available. Choosing a bus is the same kind of decision. SPI, I2C and UART each solve a different problem, and the right answer depends on the speed, the number of devices, the distance and the wiring you can afford.

From first principles, the three buses trade off four things. SPI is fast and full-duplex, and it uses one chip-select line per device, so the wire count grows with the device count. I2C uses two wires for any number of devices, but it is slower and it needs pull-ups and addresses. UART is point to point with two wires, and it is simple for one device, but it does not share. The table below gives typical values for the Uno.

| Property | UART | I2C | SPI |
|---|---|---|---|
| Wires | 2 (TX, RX) plus ground | 2 (SDA, SCL) plus ground | 3 plus one chip select per device, plus ground |
| Devices | 1 per port | Many, with addresses | Many, with one chip select each |
| Typical speed on the Uno | 9600 to 115200 baud | 100 to 400 kbit/s | Up to 8 MHz |
| Direction | Full-duplex | Half-duplex | Full-duplex |
| Acknowledgment | None in the protocol | Yes, ACK and NACK | None in the protocol |
| Pull-ups needed | No | Yes | No |

The worked example is a robot subsystem: a motor encoder that sends 200,000 pulses per second and needs fast updates, an IMU that needs a few hundred samples per second, and a GPS that sends a sentence every second. The encoder is best handled by an interrupt or a dedicated counter, not by any bus. The IMU fits SPI or I2C depending on its speed. The GPS fits UART. The design choice is clear once the rates are known.

The common mistake is to choose the bus because it is familiar. The rate should decide. A device that needs 1 MHz on I2C will not work at 100 kbit/s, and a device that needs only a few readings a second does not need SPI at all.

On the job, the bus choice is written in a design document with the rates, the device count and the wire budget. A reviewer who sees the table can check the reasoning quickly.

---

### Reading an external ADC

A hand-held calculator takes a number and returns a result. The MCP3008 takes a command and returns a number. The command selects a channel, and the reply gives the conversion. Reading it is a two-transfer process, because the chip returns its result one transfer after the command.

From first principles, the MCP3008's transaction begins with the chip-select line low. The controller sends three bytes. The first byte is 0x01, which is a start bit. The second byte is 0x80 OR'd with the channel number shifted left by four bits, which sets the single-ended mode and the channel. The third byte is 0x00, which only clocks out the result. The reply comes back on MISO at the same time. The 10-bit result is in the lower two bits of the second reply byte and all eight bits of the third reply byte. The result is (second byte AND 0x03) shifted left by eight, OR the third byte.

The worked example is channel 0 with the pot at mid-scale. The reply gives a result of 512, which is 512 × 5 / 1024 = 2.5 V with VREF at 5 V. A result of 1023 means the input is at the full-scale voltage. The conversion from count to volts is the same as the Uno's internal ADC on Day 18, but the reference and the sample timing come from the external chip.

The common mistake is to read only one byte of the reply, which gives a value that is always near 0 or 255 and looks like a broken chip. The result needs three transfers and a mask. The second common mistake is to use a VREF different from the supply without changing the conversion. The MCP3008 uses VDD as its reference by default, so the calculation depends on the measured supply.

On the job, external ADCs are used where the microcontroller's internal ADC is too slow, too noisy or too few channels. Battery monitors, current sensors and force sensors often use an external part, and the same frame structure appears in many of them.

---

## Hands-on exercises

1. **Check the datasheet.** Open the MCP3008 datasheet and find the table of maximum clock frequency against supply voltage. Write the maximum clock at 5 V in your notes. Find the SPI mode and the bit order the device uses.

2. **Wire the chip.** Connect VDD and VREF to 5 V, AGND and DGND to GND, CLK to pin 13, DOUT to pin 12, DIN to pin 11, and CS to pin 10. Place the 0.1 uF capacitor between VDD and GND, as close to the chip as you can. Connect the pot to CH0.

3. **Read one channel.** Use the sketch from the project to read CH0 once per second. Record the value at the bottom and the top of the pot range.

4. **Test the speed.** Set the SPI clock to 100 kHz, then 1 MHz, then 3.6 MHz. At each speed, read CH0 a hundred times and record the spread. Write one sentence on the result.

5. **Test chip select.** Remove the chip-select line and observe the readings. Then reconnect and record the difference. Explain why the readings change.

6. **Check the mode.** Change the SPI mode from 0 to 3 and read the pot. Record the result and explain why the value is wrong.

---

## Project: an MCP3008 voltage logger

The goal is a repository that reads the MCP3008 over SPI, converts the result to volts, and checks the readings against a multimeter across the pot's range. It produces a calibration table with the error at each point, which is the evidence that the link is correct.

**Repo name:** `100-days-robotics-day-22-spi`

**Repo layout:**

```
100-days-robotics-day-22-spi/
├── README.md
├── mcp3008_read/
│   └── mcp3008_read.ino
├── speed_test/
│   └── speed_test.ino
├── logs/
│   ├── calibration.csv
│   └── speed_spread.csv
├── docs/
│   ├── wiring.md
│   └── calculations.md
└── photos/
    └── bench.jpg
```

**Starter code, `mcp3008_read/mcp3008_read.ino`:**

```cpp
#include <SPI.h>

const int CS_PIN = 10;
const float VREF = 5.0;            // measured supply, record your value

int readMcp3008(uint8_t channel) {
  uint8_t cmd1 = 0x01;             // start bit
  uint8_t cmd2 = (0x80 | (channel << 4));
  SPI.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));
  digitalWrite(CS_PIN, LOW);
  SPI.transfer(cmd1);
  uint8_t r1 = SPI.transfer(cmd2);
  uint8_t r2 = SPI.transfer(0x00);
  digitalWrite(CS_PIN, HIGH);
  SPI.endTransaction();
  return ((r1 & 0x03) << 8) | r2;
}

void setup() {
  pinMode(CS_PIN, OUTPUT);
  digitalWrite(CS_PIN, HIGH);
  SPI.begin();
  Serial.begin(115200);
  Serial.println("raw,volts");
}

void loop() {
  int raw = readMcp3008(0);
  float volts = raw * VREF / 1024.0;
  Serial.print(raw);
  Serial.print(",");
  Serial.println(volts, 3);
  delay(500);
}
```

The conversion divides by 1024, which is the number of steps in a 10-bit result. The MCP3008 uses VDD as its reference by default, so the `VREF` constant must match the measured supply, not the nominal 5 V.

**Starter code, `speed_test/speed_test.ino`:**

Copy the `readMcp3008` function. Run it at 100 kHz, 1 MHz and 3.6 MHz, with a hundred readings at each speed. Print the minimum, the maximum and the standard deviation for each speed. Save the output as `logs/speed_spread.csv`.

**Step-by-step tasks:**

1. Create the repository and folder tree. Initialize git and commit the README skeleton.
2. Wire the MCP3008 as in the lesson. Check the wiring with a multimeter, especially VDD, the chip-select line and the MISO line.
3. Measure the supply voltage at the chip's VDD pin with the multimeter. Record the value. This is your `VREF`.
4. Upload `mcp3008_read`. For the calibration, set the pot at five positions. At each position, record the multimeter voltage at the wiper and the MCP3008 reading. Save the table as `logs/calibration.csv`, with a column for the error in volts and the error in LSBs.
5. Upload `speed_test` and run it. Save the output as `logs/speed_spread.csv`. Compare the spread at each speed.
6. Write `docs/calculations.md` with the step size, the maximum SPI clock from the datasheet, the chosen speed and the reason for the choice.
7. Write the README with the checklist below.

**README checklist:**

- [ ] The MCP3008 mode, bit order and maximum clock are cited to the datasheet page
- [ ] The measured VDD is used as the reference in the conversion
- [ ] The calibration table has five positions with the multimeter and the MCP3008 readings
- [ ] The error column is in LSBs and in volts
- [ ] The speed test results are reported, with the spread at each speed
- [ ] The README explains why the chosen speed is within the datasheet limit
- [ ] One paragraph describes a fault found during wiring or testing

**Done when:**

- The MCP3008 reading tracks the pot across its range, and the calibration errors are within 3 LSB at each point
- The speed test shows a spread that is stable at the chosen speed, and the README explains any higher spread at the top speed
- The repository has at least three commits, and every number in the README matches a log file
- The wiring document and the photo show the chip select on pin 10 and the decoupling capacitor

**Evidence to commit to GitHub:** both sketches, the two logs, the calculations document, the wiring document, the bench photo and the README. Do not commit build folders.

The calibration table is the strongest evidence. A reviewer who sees the readings and the multimeter values together can check the link, the conversion and the error in one pass.

---

## Check your understanding

1. An SPI device works at 1 MHz but returns wrong values at 8 MHz. Name two causes that fit this behavior, and the datasheet check that would tell you which one applies.
2. A reading returns 0x3FF on every channel. Give three causes in the order you would check them.
3. An MCP3008 is read with a single byte transfer. Explain what the reply contains and why the result is wrong.
4. Two SPI devices share the bus, and reading one returns data from the other. Explain the likely cause and the change that fixes it.
5. A robot has an encoder with 200,000 pulses per second, an IMU that needs 500 samples per second and a GPS that sends a sentence each second. Choose a bus or method for each, and give the reason.
6. The MCP3008 at 5 V reads 2.5 V on the multimeter, and the code prints 512. Compute the expected reading, and say whether the result is correct.

---

## Answers

1. The 8 MHz speed may exceed the device's maximum clock, which the datasheet lists for each supply voltage. Another cause is wiring capacitance or long leads that distort the clock at high speed. The datasheet table tells you the maximum clock, and a scope or logic analyzer tells you about the edges.
2. The three checks in order: the chip-select line is not going low, the clock or data lines are not connected, and the SPI mode is wrong. A constant 0x3FF often means the input reads high or the chip is not selected, so checking the select line and the wiring first is the fastest path.
3. A single byte returns the reply to the start bit and the channel command, which contains only the top two bits of the result and not the full value. The result needs three transfers, and the 10-bit value is the top two bits of the second reply byte shifted left, plus the third reply byte.
4. The likely cause is that the second chip-select line is not held high when not used, so both devices respond to the same transaction. The fix is to set every chip-select pin to output and high in `setup()`, and to bring each one low only for its own transaction.
5. The encoder at 200,000 pulses per second needs an interrupt or a hardware counter, not a bus, because a bus would miss pulses. The IMU at 500 samples per second fits SPI or I2C at 100 kbit/s or higher, and SPI is the usual choice for higher rates. The GPS fits UART, because it sends one sentence per second over a point-to-point link.
6. The expected reading is 2.5 × 1024 / 5 = 512. The reading is correct within one LSB.

---

## Sources

*Read:*

- The Wikipedia article titled **Serial Peripheral Interface**, for the signals, the modes and the history
- The Wikipedia article titled **Chip select**. If that exact title redirects, search for "chip select" on Wikipedia
- The Wikipedia article titled **Analog-to-digital converter**, for the conversion concepts used in the MCP3008
- The MCP3008 datasheet by Microchip Technology, free from the Microchip website. Read the sections on the SPI timing, the conversion frame, the maximum clock table and the electrical characteristics
- The Arduino SPI library reference at docs.arduino.cc, for `SPISettings`, `SPI.transfer` and `SPI.beginTransaction`
- The ATmega328P datasheet by Microchip Technology, for the SPI peripheral chapter

*Tutorials:*

- The official Arduino tutorial on SPI, found in the Arduino documentation under Libraries and Built-in Examples
- The GeeksforGeeks article on the SPI protocol. Search the GeeksforGeeks site for "SPI protocol"

*Watch:*

- The Arduino YouTube channel, which has videos on SPI. Search for "Arduino SPI tutorial"
- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, for lectures on serial buses and analog interfacing. Search for the course title on the MIT OpenCourseWare site

*Practice:*

- Wokwi at wokwi.com, which includes an MCP3008 model in its component list. Build the logger there first to check the frame, then build the real circuit. Check the current component list on Wokwi, because the exact components change over time, and confirm the MCP3008 is available before you plan around it.

---

## Tomorrow

Day 23 turns the drivers you have written across this module into software architecture. You will organize the code for the ADC, timers, UART, I2C and SPI into layers, use a state machine to control the behavior of a sensor logger, and handle faults in a way that keeps the system running. The bus knowledge from today tells you what each driver has to hide, and the interrupt and timing rules from earlier days decide how the layers talk to each other.
