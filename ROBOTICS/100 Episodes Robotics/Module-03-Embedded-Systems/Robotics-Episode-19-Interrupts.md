# Day 19: Interrupts

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 4 hours

**Level:** intermediate. You need Day 16 (the Uno, registers and `millis()`), Day 17 (debouncing) and Day 18 (timer compare values and prescalers). Interrupts change how code runs, so the debugging in this lesson is harder than in earlier days. Keep the serial logs.

**By the end of this day, you will have:**

- A pin-change interrupt on an Uno external interrupt pin that counts pulses, with the count checked against a known frequency
- A timer-compare interrupt that runs at 1 kHz, with the tick count checked against `millis()` over ten seconds
- A button handled by an interrupt flag and a debounced loop, with a note on why the ISR must stay short
- A demonstration of a race condition on a 16-bit shared counter, and the fix that removes it

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3.x and the `Arduino AVR Boards` core at version 1.8.6
- Two jumper wires, about $1
- One pushbutton and a 10 kOhm pull-down or internal pull-up, as in Day 17, about $1
- A breadboard, about $5
- Optional: a function generator or a second Uno for a more realistic test signal
- Optional: a logic analyzer, about $10 to $20, to see the interrupt timing on a second channel

**Prerequisites:**

- Day 16: registers and bit operations, such as `|=` and `&=`
- Day 17: pull-ups and contact bounce
- Day 18: timer prescalers and the compare value formula

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Polling and blocking code](#polling-and-blocking-code)
- [Interrupts and the interrupt service routine](#interrupts-and-the-interrupt-service-routine)
- [External and pin-change interrupts](#external-and-pin-change-interrupts)
- [Timer interrupts](#timer-interrupts)
- [Shared variables and volatile](#shared-variables-and-volatile)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a pulse counter and a 1 kHz tick](#project-a-pulse-counter-and-a-1-khz-tick)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A cook who stirs a pot and checks the oven at the same time is doing two jobs. The cook does not stop stirring to check the oven, because the oven has a bell that rings when the timer is up. A microcontroller can do the same. It runs its main program, and when an event happens, a hardware signal stops the program, runs a short piece of code, and then returns to where it was. That signal is an interrupt, and the short piece of code is an interrupt service routine (ISR).

Interrupts let a robot react to events instead of waiting for them. A wheel encoder sends a pulse for every few degrees of rotation. A bumper switch must stop the motors within milliseconds. A radio packet must be received before the next one arrives. If the program checks each event in a loop, it has to check fast enough, and it cannot do anything else in the meantime. If the hardware raises an interrupt, the event is handled at the moment it happens, and the loop can do its other work.

Interrupts are also the source of the subtlest bugs in embedded code. A variable changes under the program's feet, a counter reads a half-updated value, and a timing measurement disagrees with the expected rate. Today you will write interrupts that work, and you will reproduce the most common failure so you know what it looks like.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1950s | Early computers use interrupts to handle input from tape readers and card punches, so the processor does not wait on slow devices. | The same reason holds today: a slow sensor should not stall a fast controller. |
| 1960s to 1970s | Minicomputers such as the PDP series use priority interrupt systems, where some events can interrupt others. | Priority levels are still used in robot controllers. |
| 1970s to 1980s | Microcontrollers add built-in interrupt controllers, with external pins and timers that can trigger an interrupt. | The ATmega328P's external and timer interrupts are in this family. |
| 1990s to 2000s | Nested interrupts, interrupt latency analysis and interrupt-driven operating systems become standard topics in embedded design. | Real-time systems rely on bounding how long an interrupt can take. |
| Today | Microcontrollers have dozens of interrupt sources, and ISRs are written in C, with hardware flags and atomic access. | Most robot firmware has at least one interrupt for a sensor, one for a timer and one for communication. |

The mechanism has been stable for decades. The number of sources and the speed have grown, and the rule that interrupt code must be short has not changed.

---

## Glossary

- **Interrupt:** a hardware or software signal that makes the CPU pause its current code and run a handler.
- **ISR (interrupt service routine):** the function that runs when an interrupt fires. On the Uno it is called with the `ISR()` macro in AVR code.
- **Interrupt vector:** the fixed address in memory where the CPU jumps when a particular interrupt fires.
- **Interrupt latency:** the time between the event and the first instruction of the ISR.
- **External interrupt:** an interrupt raised by a pin, such as the Uno's INT0 on pin 2 and INT1 on pin 3.
- **Pin-change interrupt:** an interrupt raised when any pin in a group changes state. The ATmega328P has three groups.
- **Timer interrupt:** an interrupt raised when a timer reaches its compare value or overflows.
- **Polling:** repeatedly checking whether an event has happened, usually in a loop.
- **Blocking code:** code that waits for something and does nothing else during the wait, such as `delay()`.
- **Volatile:** a C keyword that tells the compiler a variable can change outside the normal program flow, such as in an ISR.
- **Race condition:** a bug where the result depends on the timing of two or more actors that share data.
- **Atomic operation:** an operation that completes without interruption, so no other code sees a partial result.
- **Critical section:** a block of code run with interrupts disabled, to protect shared data.

---

### Polling and blocking code

A person waiting for a bus can stand at the stop and look up the road every few seconds. That is polling. The person does nothing else while waiting, and the bus can arrive between two looks, so the person may miss it by a few seconds. Blocking is the case where the person sits on a bench and does not look up at all until someone calls their name. Both approaches have a cost.

From first principles, polling checks a flag in a loop, such as `if (digitalRead(2) == LOW)`. The response time is at most the length of one loop pass, which may be milliseconds on a busy loop. Blocking code, such as `delay(500)`, stops the loop completely, so no other event can be seen during the delay. A motor driver that uses `delay` to pulse a stepper will miss every encoder pulse that arrives during the pause.

The worked example measures the cost. Suppose a loop takes 2 ms per pass because it reads four sensors and writes a display. An event that lasts 1 ms can be missed completely if it falls between two checks. Polling at 500 Hz therefore misses any pulse shorter than 2 ms. An interrupt sees the same pulse, because the hardware catches the edge regardless of what the loop is doing.

The common mistake is to use `delay()` inside an event handler, which blocks every other event. The second mistake is to poll a sensor in a tight loop that also does slow work, so the sensor reading is late and the control action is late with it. The check is to time the loop with `micros()`, and if a pass takes longer than the fastest event you expect, interrupts are the answer.

Blocking is still appropriate for simple programs, such as the Blink sketch. The rule is that a robot's main loop should never wait for something that can happen at any time.

---

### Interrupts and the interrupt service routine

A fire alarm in a building interrupts the people inside. They stop what they are doing, leave by the nearest exit and then go back to their desks. An ISR works the same way. The CPU stops the main code, saves its place, runs the ISR, and restores the place to resume the main code. The ISR should be short, because everything else waits while it runs.

From first principles, the CPU has a global interrupt enable bit. When it is set, interrupts can fire. When an interrupt fires, the hardware clears the global enable bit, so no other interrupt fires inside the ISR. This is why a long ISR is dangerous. For the ATmega328P, the minimum latency is about four clock cycles, which is 0.25 microseconds at 16 MHz, and the total time the main code is paused is the latency plus the ISR's runtime. An ISR that takes 100 microseconds at 1 kHz uses 10 percent of the CPU and blocks the main code for that time on every tick.

The ISR rules are the same on most platforms. Keep the ISR short. Do not call `Serial.print`, `delay` or other functions that depend on interrupts being enabled. Do not allocate memory. Set a flag or update a counter, and let the main loop do the slow work. Declare any variable shared with the main code as `volatile`, so the compiler reloads it each time the main code reads it.

Here is a minimal ISR example on the Uno:

```cpp
volatile unsigned long pulses = 0;

void countPulse() {
  pulses++;               // short and safe
}

void setup() {
  pinMode(2, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(2), countPulse, FALLING);
  Serial.begin(115200);
}
```

The worked example is the cost of an ISR at a high pulse rate. If an encoder gives 1,000 pulses per second and the ISR takes 4 microseconds each, the ISR uses 0.4 percent of the CPU. If the same encoder gives 50,000 pulses per second, the ISR uses 20 percent, which is still acceptable. The ISR's cost is the product of its runtime and its rate, and that product is the number to check before you design.

The common mistake is to put `Serial.println` in an ISR. It may appear to work for one or two events, but serial output needs interrupts, and the program can hang or drop data. Another mistake is to forget `volatile`, which leads to a loop that never sees the counter change because the compiler kept a copy in a register.

On the job, ISRs appear in every motor controller, every radio driver and every communication peripheral. The first code review question for an ISR is how long it takes in the worst case.

---

### External and pin-change interrupts

A doorbell rings only when someone presses it. The house does not need to check the door every second. An external interrupt works the same way. The pin is wired to the event, and the hardware signals the CPU when the pin changes in the chosen direction.

From first principles, the ATmega328P has two external interrupt pins, INT0 on pin 2 and INT1 on pin 3 of the Uno. The Arduino function `attachInterrupt` maps the pin to its interrupt number and attaches the ISR. The trigger can be LOW, CHANGE, RISING or FALLING. A rising edge is a transition from low to high, and a falling edge is a transition from high to low. For a button with a pull-up, a falling edge is the press.

The ATmega328P also has pin-change interrupts for the other pins. The chip raises an interrupt when any pin in a group changes, and the code must find which pin changed. The Arduino core does not provide an API for pin-change interrupts, so you use the registers PCICR, PCMSK0, PCMSK1 and PCMSK2. The register approach is more work, and it is needed when you run out of external interrupt pins. For this lesson, the two external pins are enough.

The worked example is the bounce problem from Day 17. A button wired to pin 2 with a falling-edge interrupt can trigger several times for one press, because the contact bounces. An ISR that counts edges reports three or four presses for each real press. The fix is to record the time of the edge in the ISR and ignore edges within 20 ms in the main loop, or to set a flag in the ISR and debounce in the loop. The ISR itself stays short.

The common mistake is to attach an interrupt to a pin that is also used for a serial port or a pin that changes too fast for the ISR. Pin 0 and pin 1 are shared with the USB serial connection, and an interrupt on them can conflict with uploads. The check is the pin map in the Uno's documentation before you wire anything.

External interrupts are how a robot knows that a bumper hit, an end-stop was reached or an encoder edge occurred. Most motor controllers have a dedicated pin for an emergency stop that is wired to an external interrupt.

---

### Timer interrupts

A metronome clicks at a fixed rate, and a musician plays in time with it. A timer interrupt is a hardware metronome for a microcontroller. The timer counts clock ticks, and when it reaches a set value, the interrupt fires. The program uses the interrupt as the time base for its control loop, so the loop runs at a fixed rate with no jitter from the rest of the code.

From first principles, the compare-match mode of Timer1 raises an interrupt when the counter equals the value in the output compare register OCR1A. The counter then resets to zero, which is the clear-timer-on-compare, or CTC, mode. Using the formula from Day 18, a 1 kHz interrupt needs OCR1A = F_CPU / (prescaler × f) − 1. With a prescaler of 1 and F_CPU of 16 MHz, that is 16,000,000 / (1 × 1000) − 1 = 15,999.

Here is the register code for that timer, as it would appear in a sketch. It uses the names from the AVR header:

```cpp
#include <avr/interrupt.h>

volatile unsigned long ticks = 0;

void setupTimer1() {
  cli();                              // stop interrupts while configuring
  TCCR1A = 0;                         // normal port operation
  TCCR1B = (1 << WGM12) | (1 << CS10);// CTC mode, prescaler 1
  OCR1A = 15999;                      // 1 kHz compare value
  TIMSK1 |= (1 << OCIE1A);            // enable compare A interrupt
  sei();                              // re-enable interrupts
}

ISR(TIMER1_COMPA_vect) {
  ticks++;                            // runs once per millisecond
}
```

The worked example is the check. If the timer is correct, ticks should increase by 10,000 over ten seconds. If the ticks are 9,600, the interrupt is either losing events or the clock is not 16 MHz. A tick count that matches `millis()` over the same window is the evidence that the timer is right.

The common mistake is to use a timer that the Arduino core already uses. Timer0 drives `millis()`, and Timer1 drives the Servo library and some PWM outputs on pins 9 and 10. If you use Timer1 for an interrupt and also call `analogWrite` on pin 9, the PWM output may change. Check which timer each library uses before you claim a timer for yourself, and document the choice in the README.

Timer interrupts are the standard way to build a fixed-rate control loop on a robot. A motor controller might run its current loop at 10 kHz in a timer interrupt and its speed loop at 1 kHz in the main code.

---

### Shared variables and volatile

Two people editing the same shopping list at the same time can each erase the other's change. A variable shared between an ISR and the main loop has the same risk. The ISR can change the variable at any moment, and the main loop may read it halfway through the change.

From first principles, the Uno's CPU is 8-bit, so a 16-bit or 32-bit variable takes several instructions to read or write. If an interrupt fires between the instructions, the main code reads a mixture of the old and new bytes. This is a torn read. For example, a 16-bit counter that goes from 0x00FF to 0x0100 can be read as 0x0000 or 0x01FF if the interrupt hits between the two byte reads. The result is a value that never existed.

The fix is to disable interrupts around the read. The Arduino core provides `noInterrupts()` and `interrupts()`, and AVR code provides the `ATOMIC_BLOCK` macro in `util/atomic.h`. Here is the safe read:

```cpp
unsigned long readTicks() {
  unsigned long copy;
  noInterrupts();
  copy = ticks;
  interrupts();
  return copy;
}
```

The worked example is the race condition in the project. A loop that prints `ticks` without protection may show jumps or occasional values that are too large or too small. The fix removes the problem, and the difference between the two logs is the evidence that the race condition exists.

The common mistake is to think `volatile` makes an access atomic. It does not. `volatile` only stops the compiler from caching the variable in a register. The atomic protection comes from the critical section. On a single-core chip, a critical section of a few microseconds is cheap, and on a multi-core chip, the protection must be more careful. The ESP32 on Day 24 has two cores, and its rules are different.

On the job, shared-variable bugs are among the hardest firmware bugs to find, because they appear only under load and disappear when you add a debug print. Keep shared data small, keep critical sections short and document the protection.

---

## Hands-on exercises

1. **Count an edge.** Wire pin 2 with a pull-up and a button. Attach a falling-edge interrupt that increments a `volatile` counter. Print the counter once per second. Press the button ten times and record the count. If the count is more than ten, note the bounce and write one sentence on the cause.

2. **Compare polling with an interrupt.** Write two sketches that count the same button presses. One polls in a loop, and one uses an interrupt. Add a 5 ms `delay` in the loop of the polling sketch. Press the button quickly and compare the counts.

3. **Set up the timer.** Use the Timer1 register code above to create a 1 kHz tick. Print `ticks` and `millis()` every ten seconds. Compute the ratio of the two counts. The ratio should be close to 1.0.

4. **Measure the latency.** Use `micros()` in the ISR to record the time of each tick. Compute the difference between successive ticks and find the minimum and maximum. Write the spread in your notebook.

5. **Demonstrate a torn read.** Write a 32-bit counter in the ISR and read it in the main loop without protection. Print the value a hundred times and look for values that are not plausible. Then add `noInterrupts` around the read and repeat.

6. **Test a long ISR.** Add a 200 microsecond busy-wait inside the timer ISR. Print the tick count over ten seconds and compare it with the expected 10,000. Explain what you see in one sentence.

---

## Project: a pulse counter and a 1 kHz tick

The goal is a repository that counts pulses with an external interrupt and runs a 1 kHz timer interrupt, with the count checked against a known reference. The project uses only the Uno and a jumper, so the cost is zero. It produces the kind of evidence a motor or encoder project needs.

**Repo name:** `100-days-robotics-day-19-interrupts`

**Repo layout:**

```
100-days-robotics-day-19-interrupts/
├── README.md
├── pulse_counter/
│   └── pulse_counter.ino
├── tick_timer/
│   └── tick_timer.ino
├── race_demo/
│   └── race_demo.ino
├── logs/
│   ├── pulse_count_10s.csv
│   ├── tick_vs_millis.csv
│   ├── race_plain.csv
│   └── race_protected.csv
└── docs/
    └── wiring.md
```

**Test wiring:** connect pin 9 to pin 2 with a jumper. Pin 9 will be the test signal from `analogWrite`, and pin 2 is the counting input. Pin 9 is on Timer1 and runs at about 490 Hz in the Arduino core. The count over ten seconds should be close to 4,900. Measure the actual frequency with a logic analyzer if you have one, and use that number in the README.

**Starter code, `pulse_counter/pulse_counter.ino`:**

```cpp
volatile unsigned long pulses = 0;

void countPulse() {
  pulses++;
}

void setup() {
  pinMode(9, OUTPUT);
  pinMode(2, INPUT);
  analogWrite(9, 128);                         // 50 percent duty test signal
  attachInterrupt(digitalPinToInterrupt(2), countPulse, RISING);
  Serial.begin(115200);
}

void loop() {
  static unsigned long lastPrint = 0;
  if (millis() - lastPrint >= 1000) {
    lastPrint = millis();
    noInterrupts();
    unsigned long count = pulses;
    interrupts();
    Serial.println(count);
  }
}
```

The `pulse_counter` sketch uses the PWM output as a stand-in for an encoder. Some students prefer to use a function generator, and that is allowed. The README should state which source you used.

**Starter code, `tick_timer/tick_timer.ino`:**

Use the `setupTimer1` and `ISR(TIMER1_COMPA_vect)` code from the lesson. Print `ticks` and `millis()` every ten seconds, and write the two numbers as a CSV line for `logs/tick_vs_millis.csv`.

**Starter code, `race_demo/race_demo.ino`:**

Write a 32-bit counter incremented by the 1 kHz ISR. In the main loop, read the counter two ways. In the first, read it directly. In the second, read it with `noInterrupts` and `interrupts` around the read. Print both values every 10 ms for 60 seconds and save each stream as a file in `logs/`. At a 1 kHz tick, each 10 ms print should differ from the previous one by about 10. Count the plain reads whose change from the previous print is far from 10, such as a jump of about 256 or more, and compare that count with the protected reads.

**Step-by-step tasks:**

1. Create the repository and folder tree. Initialize git and commit the README skeleton.
2. Wire pin 9 to pin 2 and upload `pulse_counter`. Record ten seconds of output to `logs/pulse_count_10s.csv`. Compute the count per second and compare it with the expected PWM frequency.
3. Upload `tick_timer` and record ten seconds of ticks and milliseconds. Save the CSV and compute the ratio of ticks to milliseconds.
4. Upload `race_demo` and collect the two streams for 60 seconds. Save both logs and report the number of implausible plain reads.
5. Repeat the `race_demo` test with a 200 microsecond busy-wait in the timer ISR. Note the change in the tick ratio and write one sentence on why it happens.
6. Write the README with the checklist below.

**README checklist:**

- [ ] The wiring diagram shows pin 9 connected to pin 2
- [ ] The expected pulse count is computed from the measured PWM frequency
- [ ] The tick-to-millisecond ratio is reported with the logged numbers
- [ ] The race demo results are reported for both the plain and the protected reads
- [ ] The timer register settings are explained, including why Timer1 was chosen
- [ ] The README explains one way a button bounce would affect the pulse count

**Done when:**

- The pulse counter records a count per second that matches the expected PWM frequency within a few percent, and the README states the measured value
- The tick ratio over ten seconds is within 1 percent of 1.0, and the logs are committed
- The race demo logs show at least one implausible plain read or the README explains why none appeared, and the protected logs show none
- The busy-wait experiment shows a measurable change in the tick ratio
- The repository has at least three commits and every number in the README matches a log file

**Evidence to commit to GitHub:** the three sketches, the logs, the wiring document and the README with the calculations. Include a short paragraph that names one thing you did not expect. Do not commit build folders.

The project uses little new hardware, but it produces a lot of evidence. A reviewer can check the pulse count against a PWM frequency, the tick ratio against `millis()`, and the race logs against each other.

---

## Check your understanding

1. A bumper switch must stop the motors within 2 ms of a press. The main loop takes up to 5 ms per pass. Explain why polling in the loop cannot meet the requirement, and name the mechanism that can.
2. An ISR takes 50 microseconds and fires at 2 kHz. Compute the fraction of CPU time the ISR uses, and state whether the design is safe if the main code needs 20 percent of the CPU.
3. A student writes `Serial.println` inside an ISR and sees the program hang after a few seconds. Give the most likely cause and the change that fixes it.
4. A 16-bit counter is incremented in an ISR, and the main loop reads it and sometimes sees a value that jumps by 256. Explain the cause and the correct fix.
5. You want to use Timer1 for a 10 kHz control interrupt on the Uno and also use `analogWrite` on pin 10. Identify the conflict, and suggest a different timer or a different pin for the PWM.
6. Your pulse counter gives twice the expected count on a button. Explain what might cause this and how you would test for it.

---

## Answers

1. The loop takes up to 5 ms, so a press can be missed for the full pass length and the response time can exceed 2 ms. An external interrupt on a pin captures the edge in hardware, with latency measured in microseconds, so it meets the 2 ms requirement.
2. The fraction is 50 microseconds × 2,000 per second = 100,000 microseconds per second, which is 10 percent. The ISR uses 10 percent of the CPU, and 10 percent plus 20 percent is 30 percent, which leaves plenty of margin. The design is safe.
3. Serial output needs interrupts to be enabled, and the ISR runs with interrupts disabled, so the serial code blocks or loses data. Fix it by setting a flag in the ISR and printing from the main loop.
4. The 16-bit value is read in two byte operations, and an ISR that fires between the reads leaves a torn value that combines the old high byte and the new low byte. Read the counter inside a critical section with `noInterrupts` and `interrupts`, or use `ATOMIC_BLOCK`.
5. Timer1 drives pins 9 and 10 for PWM. Using it for a 10 kHz interrupt changes the PWM on those pins, and pin 10 may stop working as PWM. Use Timer2 for the control interrupt, or move the PWM to pins 3 or 11, which use Timer2, and keep Timer1 for the interrupt.
6. The button bounces, and each bounce creates a rising edge. Test by logging the raw edges with timestamps, and look for clusters of edges within a few milliseconds of each press. Add a debounce in the main loop or a short lockout in the ISR.

---

## Sources

*Read:*

- The Wikipedia article titled **Interrupt**, for the general concept and the types of interrupts
- The Wikipedia article titled **Interrupt handler**, for the ISR concept. If that exact title redirects, search for "interrupt handler" on Wikipedia
- The Wikipedia article titled **Race condition**, for the shared-data problem
- The Wikipedia article titled **Critical section**, for the protection technique
- The ATmega328P datasheet by Microchip Technology, free from the Microchip website. Read the sections on the external interrupts, the pin-change interrupts and the 16-bit timer
- The Arduino Language Reference at docs.arduino.cc, for `attachInterrupt`, `noInterrupts` and `volatile`

*Tutorials:*

- The official Arduino tutorial on attachInterrupt, found in the Arduino documentation under Language Reference and Built-in Examples
- The GeeksforGeeks article on interrupts in operating systems, for the general concept. Search the GeeksforGeeks site for "interrupts" and read the overview

*Watch:*

- MIT OpenCourseWare course 6.115, Microcomputer Project Laboratory, for lectures on interrupts and timers. Search for the course title on the MIT OpenCourseWare site
- The Arduino YouTube channel, which has videos on attachInterrupt and timer setup. Search for "Arduino attachInterrupt tutorial"

*Practice:*

- Wokwi at wokwi.com, which supports the Arduino Uno and `attachInterrupt`. You can run the pulse counter and the tick timer there. The free tier is enough for this lesson, and it does not show real latency, so run the final test on the bench.

---

## Tomorrow

Day 20 moves from pins to communication. The UART is the simplest serial protocol, and it sends one byte at a time over two wires. You will use the Uno's hardware serial port, learn the framing and the baud rate, and parse a simple text protocol from a second board or a USB adapter. The interrupt skills from today are how a UART receives bytes without losing them, so the ISR rules from this lesson will apply again.
