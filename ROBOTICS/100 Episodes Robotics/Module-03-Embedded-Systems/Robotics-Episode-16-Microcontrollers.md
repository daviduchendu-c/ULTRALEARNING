# Day 16: Microcontrollers

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 3 hours

**Level:** beginner to intermediate. You need Day 6 (voltage, current and Ohm's law) and Day 10 (binary and logic levels). If you have not written firmware before, that is fine. This is the first day that puts code on a chip, and every later day in this module builds on it.

**By the end of this day, you will have:**

- An Arduino Uno flashed with a blink program that uses `millis()` for timing instead of `delay()`
- A second version of the same blink that writes the AVR hardware registers directly, and a note on what changed in the compiled size
- A memory report from the compiler that shows how much flash and SRAM your sketch uses
- A one-page comparison of a microcontroller and a single-board computer, based on datasheet numbers you looked up yourself

**What you need:**

- Arduino Uno Rev3, based on the ATmega328P, about $25 for an official board or $10 to $15 for a clone
- A USB-B cable for the Uno, about $3 to $5
- Arduino IDE 2.3.x (any 2.3 release), free and open source from arduino.cc
- The Arduino AVR core `arduino:avr` version 1.8.6, installed from the Boards Manager inside the IDE
- Optional: arduino-cli 1.x, a free command-line version of the same toolchain, for the compile and upload steps in the project
- One red LED and one 220 ohm resistor, about $0.20 total, for the external LED exercise
- Datasheets: the ATmega328P datasheet from Microchip and the Arduino Uno Rev3 page in the Arduino documentation. Both are free to download.

**Prerequisites:**

- Day 6: voltage, current and Ohm's law
- Day 10: binary numbers, bits and the two logic levels
- Basic programming in any language, including variables, loops and functions

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [CPU and clock](#cpu-and-clock)
- [Flash, SRAM and EEPROM](#flash-sram-and-eeprom)
- [Registers](#registers)
- [Peripherals and GPIO](#peripherals-and-gpio)
- [Firmware and the boot process](#firmware-and-the-boot-process)
- [Microcontroller vs single-board computer](#microcontroller-vs-single-board-computer)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a board survey and a register-level blink](#project-a-board-survey-and-a-register-level-blink)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Almost every robot has a small computer hidden inside its electronics that does one job, does it forever, and does it at a fixed speed. That computer reads a sensor, drives a motor, checks a battery, or blinks an LED. Unlike a laptop, it has no operating system that you install, and it has no screen or hard drive. It is a microcontroller: a processor, memory and input and output pins on a single chip, designed to run one program from start to finish.

Knowing how a microcontroller is built tells you what it can and cannot do. If you know that its memory is small, you will not try to store a camera image in it. If you know that its clock is fixed, you will understand why a timing loop runs at a predictable rate. If you know that it boots from flash memory, you will know what happens when the power comes on and why a bad program can make a board look dead. These facts decide how you design firmware, and they decide what goes wrong on the bench.

Today you will look inside a real chip. You will flash a program onto an Arduino Uno, read the memory usage that the compiler reports, write to hardware registers directly, and compare the chip with a single-board computer such as a Raspberry Pi. Everything in the rest of this module, from GPIO on Day 17 to the RTOS on Day 24, is built on the ideas in this lesson.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1971 | Intel releases the 4004, the first commercial microprocessor. | It made a programmable processor cheap enough to build into other devices. |
| 1974 | Texas Instruments releases the TMS1000, which is often cited as the first single-chip microcontroller. | Processor, memory and I/O on one chip became the model for embedded design. |
| 1977 | Intel's 8048 family is released, with a CPU, RAM and ROM on one chip. | Hobby and industrial controllers adopted single-chip designs widely. |
| 1996 | Atmel introduces the AVR family, which includes the ATmega328P in the Arduino Uno. | The AVR architecture made cheap, easy-to-program boards possible. |
| 2016 | Espressif releases the ESP32, which adds Wi-Fi and Bluetooth to a dual-core microcontroller. | Connected robots and sensor nodes became cheap to build. |
| 2021 | The Raspberry Pi Foundation releases the RP2040, its first in-house microcontroller chip. | Two cores and a large SRAM at a low price changed what hobby boards could do. |

The core idea has stayed the same since the 1970s. A microcontroller runs a fixed program with a small amount of memory and a set of pins. The chips got faster and cheaper, and they gained more memory and peripherals, but the structure did not change.

---

## Glossary

- **Microcontroller (MCU):** a small computer on one chip with a CPU, memory and programmable input and output pins.
- **CPU (central processing unit):** the part that fetches instructions from memory and executes them.
- **Clock:** an oscillator that produces a regular square wave, which sets the rate at which the CPU executes instructions.
- **Flash memory:** non-volatile memory that keeps the program when the power is off. Programs are written into it.
- **SRAM (static random-access memory):** fast volatile memory used for variables and the stack. Its contents are lost when power is removed.
- **EEPROM (electrically erasable programmable read-only memory):** small non-volatile memory used for settings that must survive a power cycle.
- **Register:** a memory location inside a chip that controls a hardware function, such as the direction of a pin or the state of a timer.
- **Peripheral:** a hardware block inside the chip, such as a timer, a serial port or an analog-to-digital converter.
- **GPIO (general-purpose input/output):** a pin that the program can set as an input or an output.
- **Firmware:** the program stored on a microcontroller, written to run on that exact hardware.
- **Bootloader:** a small program that runs first after reset and either loads new firmware or starts the main program.
- **SBC (single-board computer):** a complete computer on one board, usually running Linux, with a larger processor and external memory.
- **Compiler:** a program that turns source code into machine instructions for a specific processor.

---

### CPU and clock

Think of a cook in a small kitchen with a recipe card and a kitchen timer that beeps once a second. The cook reads one step, does it, and moves to the next when the timer beeps. The timer does not make the cook faster or slower. It sets the rhythm. A CPU works the same way with its clock. Each tick of the clock lets the CPU move one step forward, and a faster clock means more steps per second.

From first principles, the CPU repeats three actions: fetch an instruction from memory, decode it to find what it means, and execute it. An instruction might add two numbers, copy a value to a register, or jump to another place in the program. The clock frequency is the number of cycles per second. The Arduino Uno's ATmega328P runs at 16 MHz, which is 16 million clock cycles per second. A simple instruction takes one or two cycles, so the chip executes roughly eight to sixteen million simple instructions per second. Many instructions take more than one cycle, so the real figure is lower.

A loop that takes 40 instructions at two cycles each uses 80 cycles. At 16 MHz, the time for one pass is 80 / 16,000,000 = 0.000005 seconds, or 5 microseconds. This is why a microcontroller can be very precise in timing. The same calculation tells you how much work fits between two sensor readings. If you need to read a sensor every millisecond, you have 16,000 cycles to do the reading and everything else.

The common mistake is to assume that more clock speed always helps. A faster clock draws more current and produces more heat, and many chips run only up to a rated maximum. The Uno runs its clock from a 16 MHz ceramic resonator, and a bad resonator or a wrong board setting can make the timing wrong without any error message. If your serial output looks like garbage, check the board selection in the IDE first.

On the job, clock choice and power budget are part of every embedded design review. A battery-powered robot sensor often runs at a low clock and sleeps most of the time, because power matters more than speed.

---

### Flash, SRAM and EEPROM

A microcontroller has three kinds of memory, and each one has a different job. Think of a workshop. The flash memory is the shelf where the instructions are stored. It keeps them even when the lights go off. The SRAM is the workbench, where the current work is held. It is fast and convenient, but when the power goes out, everything on the bench is lost. EEPROM is a small filing cabinet for records that must survive a power cut, such as a calibration value.

From first principles, the ATmega328P has 32 KB of flash, 2 KB of SRAM and 1 KB of EEPROM. The flash holds your compiled program and the bootloader. The SRAM holds your global variables, the local variables of each function call, and the stack, which is the memory that grows and shrinks as functions call each other. The EEPROM holds whatever you write to it with the EEPROM library, and it is rated for a limited number of write cycles, typically around 100,000 per location.

Here is the numerical check. Suppose your sketch declares an array of 500 integers, each two bytes, to store sensor readings. That takes 500 × 2 = 1,000 bytes of SRAM. The Uno has 2,048 bytes in total, so about half the memory is gone before the program has done anything else. Add a few strings printed with `Serial.println` and a library with its own buffers, and the chip can run out of SRAM. When SRAM runs out, the program does not give a helpful error. It may corrupt a variable or crash in a way that looks random.

The common mistake is to store large constant data in SRAM. Strings and lookup tables that never change belong in flash, and on AVR the `F()` macro and `PROGMEM` keep them there. The way to spot the problem is the compiler report, which you will read in the project. The report shows flash usage and SRAM usage separately.

Flash and SRAM are the memory types that appear in every robot controller. A drone flight controller stores its firmware in flash and keeps its filter state in SRAM, and it stores calibration offsets in a small non-volatile memory.

---

### Registers

A thermostat on a wall has a dial. Turn the dial and the furnace changes its behavior. Registers are the dials inside a microcontroller. Each register is a memory location with a fixed address, and each bit in it controls a specific hardware feature. Writing a value to a register changes the hardware directly, without a library in between.

The first principle is memory-mapped input and output. The CPU reads and writes memory, and the chip places some addresses on the peripherals instead of on RAM. On the ATmega328P, the port B data register, `PORTB`, controls the output level of pins 8 to 13 on the Uno. The data direction register, `DDRB`, decides whether each pin is an input or an output. Setting bit 5 of `DDRB` makes pin 13 an output, because bit 5 corresponds to pin PB5, which is the pin connected to the onboard LED.

Here is the code that does that, using the names from the AVR header file:

```c
#include <avr/io.h>

DDRB |= (1 << DDB5);     // set PB5 as an output
PORTB |= (1 << PORTB5);  // drive PB5 high, LED on
PORTB &= ~(1 << PORTB5); // drive PB5 low, LED off
PORTB ^= (1 << PORTB5);  // toggle PB5
```

The shift `(1 << DDB5)` builds a number with only bit 5 set. The OR, AND and XOR operators change only that bit and leave the rest of the register alone. This read-modify-write pattern appears in every register code base, and it is the reason that a careless assignment such as `PORTB = 0x20` will switch off the other pins on the same port.

The common mistake is to write to a register without reading the datasheet's bit definitions, which can turn on a peripheral that conflicts with another one. The check is to open the datasheet section for the register and confirm each bit name. On the job, register-level work appears in drivers and bootloaders, and most application code uses a library or a hardware abstraction layer instead.

---

### Peripherals and GPIO

A kitchen has a stove, an oven, a sink and a fridge. Each one does a specific task, and each one is connected to the house through a pipe or a wire. Peripherals are the stove and the sink of a microcontroller. They are hardware blocks, such as timers, serial ports, analog-to-digital converters, PWM generators and communication buses, that do work on their own while the CPU does something else. GPIO pins are the doors of the house. Each pin can be an input or an output, and the program decides which.

From first principles, a GPIO pin has a direction, an output level when it is an output, and a way to read the level when it is an input. The ATmega328P pins run at 5 V on an Uno, and an output can source or sink a limited current. The datasheet lists 40 mA as the absolute maximum per pin, and the recommended operating limit is much lower, around 20 mA. A pin driving an LED through a 220 ohm resistor from 5 V sees about (5 − 2) / 220 = 13.6 mA, which is within the limit. A pin that tries to drive a motor directly will overload.

The Uno's peripherals are reached through the Arduino API or through registers. The API function `analogRead` uses the ADC peripheral, and `analogWrite` uses the PWM timers. Those topics are covered in Day 17 and Day 18. Today it is enough to know that peripherals run in parallel with the CPU. A timer keeps counting while your code does something else, and an ADC conversion finishes in the background after you start it.

The common mistake is to treat every pin as interchangeable. On the Uno, some pins are shared with the USB serial connection (pins 0 and 1), some support PWM, and some are used for SPI and I2C (for example, A4 and A5 for I2C). Using a shared pin for an output can break serial debugging. The way to spot it is the pin map in the Uno's documentation, which you should print or keep open whenever you wire a board.

Peripherals are where a microcontroller earns its place in a robot. Motor drivers, encoders, IMUs and radio modules each connect through a peripheral and a few pins, which lets one small chip manage many devices.

---

### Firmware and the boot process

A new car engine has a computer that decides when to inject fuel and when to spark. The code that runs it is written at the factory and stored in a memory that the engine cannot change while it runs. That code is firmware. It is the program that lives in the chip and runs on that exact hardware. Your Arduino sketch becomes firmware when the compiler turns it into machine code and the upload tool writes it into flash.

The boot process is the sequence that starts after power is applied or the reset button is pressed. On the ATmega328P, the chip begins executing at address zero, which holds a reset vector. The Uno has a bootloader, called Optiboot, in a reserved section of flash. At reset, the bootloader runs for a short time and listens on the USB serial port for a new program. If it hears one, it writes the program into flash. If it hears nothing, it jumps to the main program. Your sketch then runs its `setup()` function once and its `loop()` function forever.

The worked example is the timing of a reset. The Uno's bootloader waits a short time, typically well under a second, before it starts the sketch. If you open the serial monitor during that window, the upload will work, but if you power the board without a valid sketch, the bootloader keeps waiting. On the Uno, the IDE pulses the DTR line of the USB serial chip when it starts an upload, and that pulse resets the ATmega328P so the bootloader is listening at the right moment. This is why an upload works without you touching the reset button.

The common mistake is to think that a board that does nothing has a broken chip. The first check is whether the code compiled and uploaded, whether the right board and port are selected, and whether the power LED is on. A sketch with an infinite loop that never toggles a pin looks identical to a dead board.

Firmware updates are a core robotics skill. Field robots ship with a bootloader that accepts updates, and the update process is one of the most failure-prone parts of a product. A power cut during an update can leave a robot unable to boot, which is why production bootloaders keep a second copy of the firmware.

---

### Microcontroller vs single-board computer

A bicycle and a car both move people, but they differ in weight, power, fuel and what they cost to keep running. Microcontrollers and single-board computers both run programs, but they differ in the same way. A microcontroller is a light, low-power machine that runs one program directly on the hardware with fixed timing. A single-board computer is a heavier machine with a full operating system, a larger memory and more software support.

From first principles, the key differences are memory, timing, power and software. A microcontroller such as the ATmega328P has 2 KB of SRAM and runs at 16 MHz. A Raspberry Pi Zero 2 W has a quad-core ARM Cortex-A53 processor, 512 MB of RAM and runs Linux. A Raspberry Pi Pico uses the RP2040, which is a microcontroller with 264 KB of SRAM, dual cores at up to 133 MHz and no operating system. The Pico is the bridge between the two worlds, and it is a useful part to compare.

Here is the calculation that makes the choice. A microcontroller running at 16 MHz with a 5 V supply might draw 20 mA in active mode, which is 0.1 W. A single-board computer under load draws several watts, often 2 W to 5 W, and it needs a longer time to boot. If the robot has a battery of 2,000 mAh at 5 V, which is about 10 Wh, the microcontroller can run for many hours, and the single-board computer runs for a few. That is why a robot often uses a microcontroller for motors and sensors and a single-board computer for vision and planning.

The common mistake is to use a single-board computer to read sensors directly. Linux is not a real-time system by default, so a loop that must run every millisecond can miss its deadline when the operating system does other work. The fix is to put the time-critical work on a microcontroller and send results to the single-board computer over a serial link, which is the architecture you will build in later days.

On the job, the typical robot has both. A ROS 2 robot uses a single-board computer for planning and a microcontroller for the motor control loop, and the two talk over UART, I2C or CAN. Knowing which work belongs on each side is a core skill in robotics engineering.

---

## Hands-on exercises

1. **Install the toolchain.** Install Arduino IDE 2.3.x, open Boards Manager and install `Arduino AVR Boards` at version 1.8.6. Connect the Uno, select `Arduino Uno` as the board and the correct COM port. On Windows, find the port number in Device Manager under Ports (COM and LPT). Upload the built-in Blink example and confirm the onboard LED flashes once per second.

2. **Read the compiler report.** After you compile Blink, read the output at the bottom of the IDE. It should say something like `Sketch uses 932 bytes (2%) of program storage space` and `Global variables use 9 bytes (0%) of dynamic memory`. Your numbers will differ. Write both numbers in your notes. These are the flash and SRAM usage of your program.

3. **Replace delay with millis.** Rewrite Blink so the LED toggles every 500 ms using `millis()` and an `if` check, with no `delay()` calls. Confirm that a second counter in `loop()` keeps running while the LED toggles. This shows why `delay()` blocks the whole program.

4. **Write the register blink.** Write the register version from the examples above, toggling PB5 with `PORTB ^=`. Compile it and compare the flash usage with the Blink example. Explain in one sentence why the register version is smaller or larger.

5. **Show the memory cost.** Declare a global `char` array of 1,500 bytes and compile. Read the SRAM usage. Then declare it as `const` with `PROGMEM` and compile again. Explain the change in the two reports.

6. **Wire an external LED.** Connect a red LED with its anode to pin 8 through a 220 ohm resistor and its cathode to GND. Blink it at 2 Hz using the register method on PB0. Measure the voltage across the resistor with a multimeter in DC volts mode and compute the current with Ohm's law.

7. **Compare two boards.** Find the datasheet or product page for the Arduino Uno Rev3 and the Raspberry Pi Pico. Fill in a table with clock speed, flash, SRAM, operating voltage, number of GPIO pins and whether the board runs an operating system.

---

## Project: a board survey and a register-level blink

The goal is a small repository that you can show to an employer. It contains a working blink in two forms, a memory report and a written comparison of two boards. The project is light on new concepts because the lesson is about understanding, but the evidence it produces is real and reviewable.

**Repo name:** `100-days-robotics-day-16-microcontrollers`

**Repo layout:**

```
100-days-robotics-day-16-microcontrollers/
├── README.md
├── blink_millis/
│   └── blink_millis.ino
├── blink_registers/
│   ├── platformio.ini
│   └── src/
│       └── main.c
├── reports/
│   ├── compile_blink_millis.txt
│   ├── compile_blink_registers.txt
│   └── compile_memory_test.txt
└── docs/
    └── board_comparison.md
```

**Starter code, `blink_millis/blink_millis.ino`:**

```cpp
const int LED_PIN = LED_BUILTIN;   // pin 13 on the Uno
const unsigned long INTERVAL_MS = 500;

unsigned long lastToggle = 0;
unsigned long ticks = 0;
bool ledOn = false;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(115200);
}

void loop() {
  ticks++;                                    // runs while the LED waits
  unsigned long now = millis();
  if (now - lastToggle >= INTERVAL_MS) {
    lastToggle = now;
    ledOn = !ledOn;
    digitalWrite(LED_PIN, ledOn ? HIGH : LOW);
    Serial.print("toggle, loop count = ");
    Serial.println(ticks);
  }
}
```

**Starter code, `blink_registers/src/main.c`:**

```c
#include <avr/io.h>
#include <util/delay.h>

int main(void) {
  DDRB |= (1 << DDB5);
  while (1) {
    PORTB ^= (1 << PORTB5);
    _delay_ms(500);
  }
  return 0;
}
```

The register version is plain C for the AVR library, so it cannot be built as an Arduino sketch. Build it as a PlatformIO project with `platform = atmelavr` and `board = uno` in `platformio.ini`, and leave out any `framework` line, so PlatformIO uses the bare AVR toolchain. Run `pio run` to build and `pio run -t upload` to flash. If that setup is too much for today, keep the code in `docs/` and explain the difference in the README. An honest partial result is better than a broken build.

**Step-by-step tasks:**

1. Create the repository and folder tree above. Initialize git in the root folder with `git init` and make the first commit with your README.
2. Compile the millis sketch. Install the core with `arduino-cli core install arduino:avr@1.8.6`, then run `arduino-cli compile --fqbn arduino:avr:uno blink_millis`. Save the terminal output to `reports/compile_blink_millis.txt`.
3. Upload the sketch with `arduino-cli upload -p COM3 --fqbn arduino:avr:uno blink_millis`, using your own COM port. Open the serial monitor at 115200 baud and confirm that the loop count grows much faster than the toggle count.
4. Write `compile_memory_test.txt` by adding a 1,500-byte global array to a copy of the sketch, compiling it, and saving the report. Compare the SRAM line with the first report.
5. Write `docs/board_comparison.md` with a table of the Uno and the Raspberry Pi Pico, using numbers from the official datasheets. Cite each number to the datasheet page where you found it.
6. Write the README with the checklist below.

**README checklist:**

- [ ] The repo describes what each folder contains
- [ ] The Arduino AVR core version and the `arduino-cli` version are stated
- [ ] Each compile report is linked from the README
- [ ] The register blink explains which bits it changes and why
- [ ] The comparison table lists the datasheet source for each number
- [ ] A short paragraph explains one thing you learned that you did not expect

**Done when:**

- The millis blink compiles and runs on a real Uno, and the serial output shows the loop counter increasing between toggles
- The compile reports for the blink and the memory test are in the `reports/` folder, and the two SRAM figures are different
- The register version either compiles and runs, or its README section explains the build problem you hit and what you tried
- The comparison table has at least six rows and cites the datasheet for each number
- The repository has at least three commits, each with a clear message

**Evidence to commit to GitHub:** the sketches, the three compile reports, the comparison document, a README with the checklist filled in, and at least one photo or serial log that shows the Uno running. Do not commit the compiled `.hex` files or the Arduino build cache.

This project is light for a robotics day because the concept is foundational. The evidence you commit shows that you can install a toolchain, read the output and write a comparison from primary sources. Those are the skills that interviewers check first.

---

## Check your understanding

1. A sketch works on your bench but fails after you add a library and a 1 KB lookup table. The compiler says flash usage is 40 percent and SRAM usage is 97 percent. Which change would you make first, and why?
2. You write `PORTB = 0x20;` to turn on pin 13 and the other LEDs on pins 8 to 12 turn off. Explain why this happens and how you would fix it.
3. Your timing measurement shows a loop running about 1 percent slower than expected on a sensor board. List three causes that are about the clock or the power supply, not about your code.
4. A student says a microcontroller is just a small computer, so it can run Linux and do everything a Raspberry Pi can do. Give two specific technical reasons why that is not right for a motor control loop.
5. You open the serial monitor and see garbled characters after uploading a new sketch. Name two things to check before you change any code.
6. Explain why the Arduino bootloader waits briefly after reset before running the sketch, and what would happen if it did not.

---

## Answers

1. Reduce SRAM use first. Flash at 40 percent is fine, but SRAM at 97 percent leaves almost no room for the stack, so the program is at risk of memory corruption. Move the constant lookup table to flash with `PROGMEM` or the `F()` macro, which costs flash but frees SRAM.
2. The assignment `PORTB = 0x20` writes every bit of the port, so it sets the other pins to low. Use `PORTB |= (1 << PORTB5)` to change only bit 5 and leave the others alone.
3. The clock source may be off, such as a resonator with a different frequency or a wrong board setting. The supply voltage may be low, which slows some parts of the chip. Temperature changes also shift the frequency of a crystal or resonator slightly.
4. A microcontroller has no memory management unit and typically has too little RAM for a full Linux kernel. Linux is also not a real-time system by default, so a motor loop with a strict deadline can miss its timing when the kernel runs other tasks.
5. Check that the baud rate in the serial monitor matches the `Serial.begin` value in the sketch, and check that the correct board and port are selected. A wrong baud rate is the most common cause.
6. The bootloader waits so that a programmer can start an upload, which requires the chip to listen on the serial port during reset. If it skipped the wait, the upload would fail or require a manual reset.

---

## Sources

*Read:*

- The Wikipedia article titled **Microcontroller**, for the definition and the history of single-chip designs
- The Wikipedia article titled **Processor register**, for the register concept
- The Wikipedia article titled **Static random-access memory**, for SRAM and its trade-offs
- The Wikipedia article titled **Flash memory**, for non-volatile storage
- The Wikipedia article titled **Bootloader**, for the boot process
- The ATmega328P datasheet by Microchip Technology, free from the Microchip website. Read the sections on the CPU, memory and the port B registers
- The Arduino Uno Rev3 documentation page at docs.arduino.cc, for the pinout and the electrical limits

*Tutorials:*

- The GeeksforGeeks tutorial on embedded systems, for an overview of the terms in this lesson. Search the GeeksforGeeks site for "embedded systems tutorial"
- The official Arduino Language Reference at docs.arduino.cc, for `millis()`, `digitalWrite` and `Serial`

*Watch:*

- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, which covers microcontroller architecture and programming. Search for "MIT 6.115 Microcomputer Project Laboratory OpenCourseWare"
- The Arduino YouTube channel, which publishes official beginner tutorials for the Uno. Search for "Arduino Uno blink tutorial"

*Practice:*

- Wokwi at wokwi.com, a free browser simulator for the Arduino Uno and other boards. Run the millis blink there before you wire a real board. The free tier is enough for this lesson.

---

## Tomorrow

Day 17 uses the GPIO pins you met today. You will wire buttons and LEDs, learn logic levels and pull-up resistors, and handle the bounce that makes a cheap button look like it was pressed many times. The register knowledge from this lesson is the reason you will understand what a pull-up resistor does inside the chip, and the millis timing you wrote today is the tool you will use to debounce the button.
