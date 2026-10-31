# Day 20: UART

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 4 hours

**Level:** beginner to intermediate. You need Day 16 (the Uno and the Arduino IDE), Day 17 (logic levels), Day 18 (timing) and Day 19 (interrupts and why receiving must not lose bytes). You will need a second board or a USB-to-serial adapter for the project.

**By the end of this day, you will have:**

- Two boards that exchange a text frame over a UART at 9600 baud, with the wiring documented
- A sender that writes a checksummed frame once every 200 ms, and a receiver that accepts only frames with a valid checksum
- A baud-rate calculation for the Uno's serial port, with the timing error stated
- A short written note on how to diagnose a serial link that fails, using the logic-level and framing rules from this lesson

**What you need:**

- Two Arduino Uno Rev3 boards, about $25 each for official boards or $10 to $15 each for clones. A cheaper option is one Uno and one USB-to-TTL adapter, described in the project.
- Arduino IDE 2.3.x with the `Arduino AVR Boards` core at version 1.8.6. The `SoftwareSerial` library is included with the core.
- Optional: a USB-to-TTL adapter with a CP2102 or FT232RL chip, about $3 to $6. Set it to 3.3 V or 5 V to match the board you connect it to. The 5 V setting is the right one for the Uno, and check the jumper on the adapter before you connect it.
- A potentiometer, 10 kOhm, for a changing sensor value, about $1 to $2
- A breadboard and jumper wires, about $5 to $8
- A digital multimeter with DC volts, about $15 to $25
- Optional: a logic analyzer, about $10 to $20, to see the frame on the wire

**Prerequisites:**

- Day 16: the Uno's serial monitor and `Serial.begin`
- Day 17: logic levels and the rule about matching voltages between two devices
- Day 19: interrupts, and why a receive buffer must be read before it fills

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Serial frames and framing](#serial-frames-and-framing)
- [Baud rate and timing error](#baud-rate-and-timing-error)
- [TX, RX and the common ground](#tx-rx-and-the-common-ground)
- [Logic levels between boards](#logic-levels-between-boards)
- [Serial debugging](#serial-debugging)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a checksummed sensor link](#project-a-checksummed-sensor-link)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Two people in different rooms can talk by passing notes under a door, one letter at a time. They agree in advance on how long a letter takes and where each word starts and ends. If they agree, the message gets through. If one person writes faster than the other reads, letters are lost. The UART, the universal asynchronous receiver-transmitter, works the same way. It sends one byte at a time over one wire, and the two ends agree on the speed and the frame layout before any data is sent.

The UART is the most common way that robot parts talk to each other. A GPS module sends its position on a UART. A motor driver accepts commands on one. A Bluetooth module, a single-board computer's console and many debug probes use one. Most of the time, the UART works without any attention. When it fails, the failure is usually a mismatched baud rate, reversed TX and RX, a missing ground or a voltage level that the receiver does not accept. Each of these problems is easy to diagnose if you know the rules.

Today you will build a small text protocol between two boards. The sender writes a frame with a sequence number, a sensor reading and a checksum, and the receiver checks the checksum before it accepts the frame. The project is short, but it contains the pieces of every serial link you will meet in a robot.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1960 | The Electronic Industries Association publishes the RS-232 standard for serial communication between computers and terminals. | RS-232 framing and naming (TX, RX, RTS, CTS) are still used in robot consoles and debug ports. |
| 1960s to 1970s | Teletype machines and early terminals use asynchronous serial frames with start and stop bits. | The frame format used on the Uno is the same idea. |
| 1970s to 1980s | Integrated UART chips, such as the 8250 family used in PC serial ports, replace discrete logic. | The name "UART" comes from this era of chip design. |
| 1990s to 2000s | Microcontrollers add built-in UART peripherals, and USB serial adapters replace the old RS-232 port on most computers. | Most robot boards now expose a UART that connects to a USB adapter. |
| Today | Most sensors and modules use 3.3 V TTL-level UART, and a USB-to-TTL adapter connects them to a laptop. | The 5 V and 3.3 V level question is one of the most common wiring problems in robotics. |

The frame format is decades old, and it is the simplest reliable link between two chips. The voltage levels changed, and that change causes most of the modern problems.

---

## Glossary

- **UART (universal asynchronous receiver-transmitter):** a hardware block that sends and receives bytes over two wires, without a shared clock.
- **Baud rate:** the number of signal changes per second on the wire. For simple UART links, one bit is sent per baud, so 9600 baud is 9600 bits per second.
- **Frame:** the group of bits that makes up one byte on the wire, including the start bit, data bits, optional parity and stop bit.
- **Start bit:** a low bit that tells the receiver a new byte is starting.
- **Stop bit:** a high bit at the end of a frame, which gives the receiver time to finish reading before the next start.
- **Parity:** an optional extra bit that makes the number of 1s in the frame even or odd, so a single bit error can be detected.
- **8N1:** a common frame format with 8 data bits, no parity and 1 stop bit.
- **TX and RX:** transmit and receive. The TX pin of one device connects to the RX pin of the other.
- **Common ground:** a shared 0 V connection between two devices, so their logic levels mean the same thing.
- **TTL (transistor-transistor logic):** a family of logic levels, typically 5 V or 3.3 V in modern boards.
- **Level shifter:** a circuit that converts logic voltages between two levels, such as 5 V and 3.3 V.
- **Checksum:** a value calculated from the data in a message and sent with it, so the receiver can check for corruption.
- **Baud error:** the difference between the baud rate that a device actually produces and the rate it was asked to produce.
- **Serial monitor:** a program on the computer that shows the bytes arriving on a serial port.

---

### Serial frames and framing

A telegraph operator sends each letter as a short pattern of clicks, with a pause between letters. The receiver knows where each letter starts because the pause marks the boundary. A UART frame uses a similar idea. The line sits high when idle. A byte starts with a low start bit, followed by eight data bits from least significant to most significant, and ends with a high stop bit.

From first principles, the receiver listens for the falling edge of the start bit, waits half a bit time to sample in the middle of the bit, and then samples each following bit at one-bit intervals. If the receiver is set to the same baud rate as the sender, the middle samples are reliable. The frame layout is the agreement that makes the sampling work.

Here is the worked example for the letter A, which is 0x41 in ASCII, or 0100 0001 in binary. The bits are sent from least significant to most significant, so the wire sees 1, 0, 0, 0, 0, 0, 1, 0 for the data. With the start bit low and the stop bit high, the full frame is 0 (start), then 1 0 0 0 0 0 1 0 (data), then 1 (stop). That is ten bits in total. At 9600 baud, one bit lasts 1 / 9600 = 104 microseconds, so the whole frame lasts about 1.04 ms. A string of 100 characters takes about 104 ms at 9600 baud.

The common mistake is to forget that the frame has overhead. A UART at 8N1 sends 10 bits for every 8 bits of data, so the useful data rate is 80 percent of the baud rate. A link at 9600 baud carries about 960 bytes per second of payload. A design that needs 1,000 bytes per second will fail at that speed, and the failure looks like lost data, not an obvious error.

On the job, the frame format is part of the protocol design. Every serial protocol, from the NMEA GPS format to the binary packets of a motor driver, is built on top of these frames. The frame tells you how fast the bytes can move, and the protocol decides what they mean.

---

### Baud rate and timing error

A conductor who sets the tempo for an orchestra must agree with the musicians. If the conductor's beat is 120 and the players count at 118, the music drifts. A UART has the same issue. Both ends must sample the wire at close to the same rate, and any difference adds up across the ten bits of a frame.

From first principles, the Uno's serial peripheral derives its baud rate from the CPU clock using a register called UBRR. The formula for the normal mode is UBRR = F_CPU / (16 × baud) − 1. For 9600 baud with a 16 MHz clock, UBRR = 16,000,000 / (16 × 9600) − 1 = 103.17, which rounds to 103. The actual baud rate is then 16,000,000 / (16 × 104) = 9615, which is 0.2 percent above the target. That is a small error, and UARTs tolerate errors of a few percent.

The worked example is 115200 baud. The double-speed mode of the USART halves the divisor, and the Arduino core enables it. The datasheet's baud table lists about 2.1 percent error for 115200 with a 16 MHz clock in that mode, and that is within the tolerance of most receivers. A device with a different clock, such as a ceramic resonator with a 1 percent error, can push the total error past the limit, and the link then fails at high speed but works at 9600. This is one of the reasons a link that works at a low speed can fail at a high one.

The common mistake is to set the baud rate on one device and forget the other. A serial monitor set to 115200 with a sketch that calls `Serial.begin(9600)` shows garbage characters, and the garbage is correct data at the wrong speed. The check is to compare the two numbers in the code and in the serial monitor before you look for a hardware fault.

Baud error is a planning issue for every robot with a GPS or a motor controller on a UART. A good rule is to choose the lower baud rate that meets the data needs, and to verify the link at the highest temperature and voltage the robot will see.

---

### TX, RX and the common ground

A phone call needs two people, one who speaks and one who listens, and the wires must connect the speaker's microphone to the listener's earpiece. A UART link has the same structure. The TX pin of one device drives the RX pin of the other, and the two devices must share a ground so that a low on one side means low on the other.

From first principles, UART is point to point, with one wire for each direction. A full-duplex link uses two wires, TX and RX, and each device has its own transmitter and receiver. The wire from TX on device A goes to RX on device B, and the wire from TX on device B goes to RX on device A. The ground wire connects the two 0 V references. Without a shared ground, the logic levels have no common zero, and the receiver may read noise or nothing at all.

The worked example is the cross-over. If you connect TX to TX and RX to RX, neither device receives anything, and the serial monitor stays silent. The fix is to swap the two wires. This is the single most common serial mistake, and it causes more lost time than any other.

The common mistake is to omit the ground because the two devices are powered from separate supplies. Each supply has its own 0 V, and the signal wire cannot provide a return path for the ground current. A missing ground can produce a link that works on some days and fails on others, which is the sign of a ground loop or a floating reference.

On the job, crossing TX and RX and sharing ground are the first two checks on any new serial device. Datasheets and wiring diagrams for modules such as GPS receivers show these connections clearly.

---

### Logic levels between boards

A plug and socket that look the same but carry different voltages can fry the device on one side. A UART is the same. A 5 V output on an Uno can exceed the input limit of a 3.3 V device, and the device may be damaged without any sign of failure at first.

From first principles, the Uno's TX pin drives 5 V when high. The ESP32 and many modern sensors run at 3.3 V, and their pins are not 5 V tolerant. A 5 V signal into a 3.3 V input can exceed the absolute maximum rating, which is a fault that does not show up at first and then kills the input later. The reverse is usually safe, because a 3.3 V output is above the logic-high threshold of a 5 V input. The Uno's VIH minimum is about 3.0 V at 5 V VCC, so 3.3 V clears it.

The fix for the 5 V to 3.3 V direction is a voltage divider or a level shifter. A simple divider uses two resistors, for example 2.2 kOhm from the Uno's TX to the receiver's RX, and 3.3 kOhm from RX to ground. The output is 5 × 3.3 / (2.2 + 3.3) = 3.0 V, which is high enough for a 3.3 V input to read as high and low enough not to damage it. A dedicated level shifter is more reliable at high speed, and a divider is fine at 9600 baud.

The common mistake is to connect a 5 V device to a 3.3 V board with no conversion and assume that the board will tolerate it. Check the absolute maximum ratings in the datasheet, not the typical operating values, and add a level shifter or divider before the first test.

On the job, a robot with a 5 V motor controller and a 3.3 V computer will have at least one level shifter on the UART. Getting this wrong is one of the most expensive mistakes in early prototyping, because a damaged chip costs more than the shifter.

---

### Serial debugging

A doctor who listens to a patient's heartbeat with a stethoscope checks the sounds in a fixed order before trying treatment. A serial link is easier to fix with the same method. Check the power, then the ground, then the wires, then the baud rate, and only then the protocol.

From first principles, serial debugging uses three tools: a serial monitor, a multimeter and, if you have one, a logic analyzer. The monitor shows the bytes that arrive. The multimeter shows the idle voltage on each line, which should be near the high level when no data is sent. A logic analyzer shows the frame timing and makes baud errors visible as stretched or shortened bits.

The worked example is a line that reads 0 V when idle. That means the sender's TX pin is not driving the line high. It may be unconnected, it may be the wrong pin, or the sender may not have started. A line that reads about 2.5 V when idle suggests a floating or contended line, which can be caused by two drivers on one wire or a missing pull-up.

The common mistake is to change the protocol before checking the wiring. Students often rewrite a checksum routine when the real problem is that RX and TX are swapped. The second mistake is to read the serial monitor as if it shows what the sender meant. It shows what the receiver heard, and a baud mismatch changes that.

On the job, the serial link is the first thing that a field technician checks when a robot stops reporting. A clear debugging order, and a log that records the raw bytes, saves hours of guessing.

---

## Hands-on exercises

1. **Check the idle line.** Connect one Uno's TX pin 1 to a multimeter, with the black lead on GND, and upload an empty sketch. Measure the idle voltage. It should be close to 5 V when no data is sent, and close to 0 V during data bursts when measured with a slow multimeter reading.

2. **Print a frame.** Send the letter A at 9600 baud with `Serial.write('A')` in a loop. Use a logic analyzer if you have one, or use a receiver sketch that prints the byte value. Confirm that 0x41 arrives.

3. **Test the baud mismatch.** Set the serial monitor to 115200 and the sketch to 9600. Record what you see. Then set both to 9600 and record the difference. Write one sentence on why the wrong baud rate produces data that looks random.

4. **Calculate the payload rate.** For 9600 baud with 8N1, compute the maximum payload bytes per second. For 115200 baud, compute the same. Compare with the data rate of a GPS module that sends 80 bytes of NMEA sentences once per second.

5. **Swap the wires.** Connect two boards and confirm the link works. Swap the TX and RX wires and observe the result. Record the change in your notes.

6. **Add a divider.** With the sender at 5 V and the receiver at 3.3 V, insert a voltage divider with 2.2 kOhm and 3.3 kOhm on the signal line, and measure the voltage at the receiver. Confirm it is near 3.0 V when the sender is high.

---

## Project: a checksummed sensor link

The goal is a working link between two boards with a simple frame format and a checksum. The sender reads a potentiometer, builds a frame with a sequence number and the reading, computes a checksum and sends it. The receiver accepts only frames with a valid checksum and rejects the rest. This is the same pattern used by NMEA GPS sentences and many motor protocols.

**Repo name:** `100-days-robotics-day-20-uart`

**Repo layout:**

```
100-days-robotics-day-20-uart/
├── README.md
├── sender/
│   └── sender.ino
├── receiver/
│   └── receiver.ino
├── logs/
│   ├── clean_run_60s.txt
│   └── corrupted_run.txt
├── docs/
│   ├── wiring.md
│   └── frame_format.md
└── photos/
    └── link.jpg
```

**Wiring:** the sender's TX pin 1 connects to the receiver's RX pin on SoftwareSerial pin 10. The sender's GND connects to the receiver's GND. If you use the 3.3 V board, add the divider from the lesson. The Uno's TX pin is 5 V and the receiver is also an Uno, so no divider is needed. The potentiometer's wiper goes to A0 on the sender.

**Frame format:** each frame is one line of ASCII text. It starts with `$`, contains the body `SENS,<sequence>,<raw>`, then `*`, then the checksum as two hexadecimal digits. The checksum is the XOR of every byte in the body. The frame ends with a newline.

**Starter code, `sender/sender.ino`:**

```cpp
unsigned long seq = 0;

void setup() {
  Serial.begin(9600);
}

void loop() {
  int raw = analogRead(A0);
  char body[32];
  snprintf(body, sizeof(body), "SENS,%lu,%d", seq, raw);

  byte cs = 0;
  for (int i = 0; body[i] != '\0'; i++) {
    cs ^= (byte)body[i];
  }

  Serial.print('$');
  Serial.print(body);
  Serial.print('*');
  if (cs < 16) Serial.print('0');
  Serial.println(cs, HEX);

  seq++;
  delay(200);
}
```

**Starter code, `receiver/receiver.ino`:**

```cpp
#include <SoftwareSerial.h>

SoftwareSerial link(10, 11);           // RX on pin 10, TX unused on pin 11

unsigned long goodFrames = 0;
unsigned long badFrames = 0;

void setup() {
  Serial.begin(115200);
  link.begin(9600);
}

void loop() {
  if (!link.available()) return;

  String line = link.readStringUntil('\n');
  line.trim();

  int star = line.indexOf('*');
  if (line.charAt(0) != '$' || star < 0) {
    badFrames++;
    Serial.print("bad frame, total bad = ");
    Serial.println(badFrames);
    return;
  }

  String body = line.substring(1, star);
  byte cs = 0;
  for (unsigned int i = 0; i < body.length(); i++) {
    cs ^= (byte)body.charAt(i);
  }
  unsigned int want = strtoul(line.substring(star + 1).c_str(), NULL, 16);

  if (cs == want) {
    goodFrames++;
    Serial.print("ok ");
    Serial.print(body);
    Serial.print(" good = ");
    Serial.println(goodFrames);
  } else {
    badFrames++;
    Serial.print("checksum error, total bad = ");
    Serial.println(badFrames);
  }
}
```

The receiver uses `SoftwareSerial` on pins 10 and 11 because the Uno's hardware serial port is shared with the USB connection on pins 0 and 1. SoftwareSerial is reliable at 9600 baud, and it is less reliable at higher speeds, which is a lesson worth recording in the README.

**Step-by-step tasks:**

1. Create the repository and folder tree. Initialize git and commit the README skeleton.
2. Wire the two boards as described, and check the wiring against the diagram in `docs/wiring.md`. Use a multimeter to confirm that the grounds are common.
3. Upload `sender` to the first board and `receiver` to the second. Open the receiver's serial monitor at 115200 baud and check that frames arrive.
4. Let the link run for 60 seconds with a steady potentiometer. Copy the output to `logs/clean_run_60s.txt`. Count the good and bad frames.
5. Turn the potentiometer during the run and confirm the raw value changes. Note any bad frames.
6. Create a corrupted run. Disconnect the ground for ten seconds, then reconnect it, and save the output to `logs/corrupted_run.txt`. Write one sentence on what you observed.
7. Write `docs/frame_format.md` with the frame layout, the checksum and a worked example for one frame, including the checksum arithmetic.
8. Write the README with the checklist below.

**README checklist:**

- [ ] The wiring is shown with TX and RX crossed, and the common ground is labeled
- [ ] The frame format is documented, with a worked example and the checksum calculation
- [ ] The baud rate at each end is stated, and the UBRR or baud error is noted for the Uno
- [ ] The clean run log shows the good and bad counts for 60 seconds
- [ ] The corrupted run is described, with the result and the cause
- [ ] The README states why SoftwareSerial was used and what its limit is
- [ ] A short paragraph explains one debugging step that found a fault

**Done when:**

- The receiver accepts frames from the sender, and the clean run log shows at least 200 good frames in 60 seconds with no checksum errors
- The corrupted run shows bad frames or dropped frames during the fault, and the README explains the cause
- The frame format document includes a worked checksum that you can recompute by hand
- The wiring diagram matches the physical wiring on the bench, with a photo in `photos/`
- The repository has at least three commits, and all counts in the README match the logs

**Evidence to commit to GitHub:** the two sketches, both logs, the frame format document, the wiring document, the photo and the README. Do not commit build folders.

The evidence is the log. A reviewer who sees the clean run and the corrupted run understands both the protocol and the debugging process.

---

## Check your understanding

1. Two devices communicate at 9600 baud, 8N1. Compute the time to send a 40-byte frame, and the maximum payload rate in bytes per second.
2. A link works between a laptop and a sensor, but when you connect the sensor to a motor controller, the controller shows no data. List three checks in the right order.
3. The ESP32's UART receives 5 V from an Uno's TX pin. Explain the risk, and propose a fix that costs under $1.
4. A student sets 115200 in the serial monitor and 9600 in the sketch and sees data that looks random. Explain why the data is not random, and what the correct fix is.
5. The receiver loses about 5 percent of frames when the sender runs at 115200 baud on a SoftwareSerial link. Give two causes and one design change to reduce the loss.
6. A checksum is the XOR of the body bytes. Explain one kind of corruption it catches and one kind it misses.

---

## Answers

1. A frame of 40 bytes has 400 bits at 10 bits per byte under 8N1. At 9600 baud, that takes 400 / 9600 = 0.0417 s, or about 41.7 ms. The maximum payload rate is 9600 / 10 = 960 bytes per second.
2. First, check the ground between the two devices. Second, check the TX and RX crossover. Third, check the baud rate and the voltage level on the line. Each check is a few minutes, and the order goes from the simplest fault to the most complex.
3. The ESP32 pins are not 5 V tolerant, so a 5 V signal can exceed the absolute maximum rating and damage the input. A voltage divider using a 2.2 kOhm resistor in series and a 3.3 kOhm resistor to ground on the signal line costs well under $1 and brings the signal to about 3.0 V.
4. The data is correct for 115200 baud, but the receiver samples at 9600 baud, so the bits are read at the wrong times and the result looks random. The fix is to set both ends to the same baud rate, which is usually done in the code.
5. Possible causes include the software sampling jitter of SoftwareSerial at high speed and the receive loop being slower than the byte rate, so bytes arrive while the code is busy. A design change is to lower the baud rate, use the hardware UART on a board with a second serial port, or move the receive into an interrupt-driven buffer.
6. The XOR catches single-bit errors and many multi-bit errors in the body, because a flipped bit changes the XOR result. It misses errors that cancel out, such as two identical bit flips in the same position of two bytes, and it does not detect a swapped or reordered sequence of bytes unless the sequence number is checked separately.

---

## Sources

*Read:*

- The Wikipedia article titled **Universal asynchronous receiver-transmitter**, for the frame format, the baud rate and the history of the chip
- The Wikipedia article titled **RS-232**, for the legacy standard and its voltage levels, which explain why it differs from TTL
- The Wikipedia article titled **Baud**, for the definition of the unit
- The Wikipedia article titled **Checksum**, for the general idea of error detection
- The ATmega328P datasheet by Microchip Technology, free from the Microchip website. Read the USART chapter, including the baud rate table
- The NMEA 0183 standard. The standard itself is paid, but the format is described in many free references. Search for "NMEA 0183 sentence structure" and read the descriptions from the GPS manufacturers' documentation

*Tutorials:*

- The official Arduino tutorial on serial communication, found in the Arduino documentation under Built-in Examples
- The GeeksforGeeks article on UART communication. Search the GeeksforGeeks site for "UART communication protocol"

*Watch:*

- The Arduino YouTube channel, which has videos on serial communication and SoftwareSerial. Search for "Arduino serial communication tutorial"
- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, for lectures on serial interfaces. Search for the course title on the MIT OpenCourseWare site

*Practice:*

- Wokwi at wokwi.com, which includes UART-connected boards and a virtual serial monitor. Build the sender and receiver there first. The free tier is enough for this lesson, and it does not show real voltage levels, so repeat the test on the bench.

---

## Tomorrow

Day 21 covers I2C, a two-wire bus that connects many devices to one pair of pins. Where the UART links two devices, I2C shares one data line and one clock line among many, and each device has an address. You will use the Wire library, scan the bus for devices, read a real sensor and handle the pull-up resistors that make the bus work. The framing and checksum habits from today carry over to I2C transactions, and the logic-level lessons carry over to the 3.3 V sensors you will use there.
