# Day 10: Digital Electronics

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 3 hours
**Level:** beginner. You should have finished Days 6 to 9, especially the logic-level section of Day 9. Today every signal has two states, and the work is in combining them correctly.

**By the end of this day, you will have:**

- Conversions between decimal, binary and hexadecimal, with two's complement for negative numbers
- Truth tables for the basic gates, a half adder and a simplified Boolean expression, each checked in a simulator
- A running Arduino Uno sketch that reads four switches as one byte with bitwise operators, and a 4-bit counter built from flip-flops

**What you need:**

- Breadboard, jumper wires and a DMM (from Day 6)
- Arduino Uno R3 (ATmega328P), about $25 official or about $10 for a clone, with Arduino IDE 2.3 or newer
- A 4-position DIP switch, or four 6 mm tactile pushbuttons with a latching option (about $3). Pushbuttons work if you hold them, and a DIP switch is better for this exercise.
- Two 74HC00 quad 2-input NAND gate chips, DIP-14 package (about $1 each)
- One 74HC74 dual D flip-flop, DIP-14 package (about $1)
- Logisim Evolution, version 3.8 or newer, a free and open-source logic simulator for Windows, macOS and Linux (from the project's GitHub releases page)
- Python 3.10 or newer, and `pytest` 8.x for the project

**Prerequisites:**

- Day 9: the logic-level thresholds (VIL and VIH) and pull-up resistors
- Day 8: the Arduino sketch structure and the debounce idea
- Basic arithmetic with powers of two

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Background and history](#background-and-history)
- [Glossary](#glossary)
- [Binary and number systems](#binary-and-number-systems)
- [Bits and bytes](#bits-and-bytes)
- [Logic levels](#logic-levels)
- [Logic gates](#logic-gates)
- [Boolean algebra](#boolean-algebra)
- [Truth tables](#truth-tables)
- [Combinational logic](#combinational-logic)
- [Sequential logic](#sequential-logic)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: a bit toolkit with a truth table generator](#project-a-bit-toolkit-with-a-truth-table-generator)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Everything in a robot's computer reduces to two states: a wire is high or low, a bit is 1 or 0. Digital electronics is the discipline of building useful behavior from those two states. A microcontroller reads a bank of switches as a number, a gate decides whether a motor enable line is allowed to go high, and a counter counts encoder pulses until a wheel reaches a target.

Robot code is full of bit operations. A status register packs a dozen flags into one byte. A sensor returns a 12-bit reading, and the driver has to shift and mask it. A safety interlock is a logic expression, and if it is wrong, the robot moves when it should not. Learning to read binary, to write truth tables and to simplify expressions is how you avoid those mistakes.

Today's work is partly on paper and partly on a simulator and a breadboard. Paper and simulator come first, because a truth table you can check by hand is a truth table you can trust on a real board.

---

## Background and history

Digital logic grew from a 19th-century algebra of logic and a 20th-century insight about switches.

| Year | Person or event | What it added |
|---|---|---|
| 1854 | George Boole publishes *An Investigation of the Laws of Thought* | An algebra with two values, true and false, which became Boolean algebra. |
| 1937 | Claude Shannon's master's thesis at MIT shows that relay circuits can implement Boolean algebra | The link between logic and electrical switches. |
| 1945 | ENIAC, a large electronic computer, is completed at the University of Pennsylvania | Electronic switching at scale, using vacuum tubes. |
| 1966 | Texas Instruments introduces the 7400 series of logic chips | Standard logic gates on small chips that engineers could wire together. |
| 1971 | Intel releases the 4004 microprocessor | The whole processor on one chip, built from the same gates. |

---

## Glossary

- **Binary:** a number system with base 2, using digits 0 and 1.
- **Hexadecimal:** a number system with base 16, using digits 0 to 9 and A to F. Written with a 0x prefix in C and Arduino code.
- **Bit:** a single binary digit, 0 or 1.
- **Byte:** a group of eight bits, holding values from 0 to 255 when unsigned.
- **Two's complement:** a method for storing negative integers in binary, where the top bit carries a negative weight.
- **Logic gate:** a circuit that computes a Boolean function of one or more inputs, such as AND or NOT.
- **Boolean algebra:** the algebra of 0 and 1 with operations AND, OR and NOT, used to design and simplify logic.
- **Truth table:** a table that lists every combination of inputs and the resulting output.
- **Combinational logic:** logic whose output depends only on the current inputs.
- **Sequential logic:** logic whose output also depends on stored state, usually from flip-flops.
- **Flip-flop:** a circuit that stores one bit and changes it on a clock edge.
- **Clock:** a regular signal that tells sequential circuits when to update.
- **Propagation delay:** the time a gate takes to respond to a change at its input.

---

## Binary and number systems

A decimal number is a sum of powers of ten. The number 352 is `3 × 100 + 5 × 10 + 2 × 1`. Binary works the same way with powers of two. Each position has a weight of 1, 2, 4, 8, 16 and so on, moving left.

Think of a row of light switches, each doubling the brightness of the one before it. A switch turned on adds its weight to the total.

Convert 37 to binary. The weights are 32, 16, 8, 4, 2 and 1. Thirty-seven is `32 + 4 + 1`, so the bits are `1 0 0 1 0 1`, which is `100101`. To check, add the weights where the bits are 1: `32 + 4 + 1 = 37`.

Hexadecimal groups four bits into one digit, which keeps long binary numbers short. The value 37 is `0x25`, because `2 × 16 + 5 = 37`, and `0010 0101` in binary is `0x25` in hex. Arduino and C code often writes masks in hex or with the `0b` prefix for binary, as in `0b00100101`.

The common mistake is reading bits in the wrong order. The least significant bit is the rightmost, with weight 1. Another mistake is mixing bases in one expression. Write the base next to every constant until you are sure. The limitation of hex and binary is length: a 32-bit value is 32 digits in binary and 8 digits in hex, which is why engineers prefer hex for register dumps.

Number systems matter to robots because sensors, buses and registers all deliver numbers as bit patterns. The datasheet of an IMU gives its register map in hex, and the driver code writes those values.

---

## Bits and bytes

A bit is the smallest unit of information in digital electronics. A byte is eight bits, and a 4-bit group is called a nibble. Microcontrollers process data in words of 8, 16 or 32 bits. The Arduino Uno's ATmega328P is an 8-bit processor, and the RP2040 on the Pico is a 32-bit processor.

An 8-bit unsigned byte holds values from 0 to 255. Adding 1 to 255 wraps around to 0, because the eight bits cannot hold 256. This is called overflow, and in C it is silent. A timer or encoder count stored in a byte overflows after 256 counts and reports a sudden jump.

For negative numbers, an 8-bit two's complement value covers −128 to 127. To negate a number, invert every bit and add 1. The number 5 is `00000101`. Inverting gives `11111010`, and adding 1 gives `11111011`. The top bit is 1, so the value is negative, and it equals `−128 + 64 + 32 + 16 + 8 + 2 + 1 = −5`.

The common mistake is storing a signed value in an unsigned type, or the reverse. A signed-to-unsigned conversion of −5 gives 251 in an 8-bit variable. The second mistake is assuming an integer type is large enough. Check the range before storing a sensor reading or a count.

Bits and bytes are how the robot stores its state, from a status flag to the 16-bit value of a motor's encoder count.

---

## Logic levels

Digital signals have two states, but in a real circuit they are voltages. Logic level is the mapping from voltage to logical value. In a 5 V CMOS system, a voltage near 0 V is read as low, and one near 5 V is read as high. The forbidden zone between them, which Day 9 described for the ATmega328P, can read either way.

A traffic light must be clearly red or clearly green. An amber zone that could be read as either would cause collisions, so the design keeps a margin on each side.

The thresholds differ by family. The ATmega328P at 5 V has a high threshold near 3.0 V and a low threshold near 1.5 V, as Day 9 showed. A 3.3 V system has a different pair of thresholds, and a 3.3 V output driving a 5 V input often works but must be checked on the datasheet.

The common mistake is assuming a floating input is low or high. It is neither, and it can be read as a random value. Tie every unused input of a gate to a defined level. The second mistake is mixing logic families without checking the thresholds, as Day 9 warned.

The limitation of logic levels is noise. A long wire near a motor can couple voltage spikes into a line and push it across a threshold. Design with pull-ups, short wires, decoupling capacitors and debounced inputs, as covered in earlier days.

---

## Logic gates

A logic gate computes a Boolean function of its inputs. The basic gates are AND, OR and NOT. Other gates are built by combining them, and two of them, NAND and NOR, are complete: every logic function can be built from NAND gates alone.

Think of a gate as a rule for a door. An AND gate opens only when both keys are turned. An OR gate opens when either key is turned. A NOT gate reverses the position of a single key.

The symbols are standard. AND outputs 1 only when both inputs are 1. OR outputs 1 when at least one input is 1. NOT outputs the opposite of its input. XOR outputs 1 when the inputs differ. NAND is AND followed by NOT, and NOR is OR followed by NOT.

On a chip, the 74HC00 has four independent NAND gates, each with two inputs. Its datasheet gives a propagation delay, the time from an input change to the output change. For the 74HC series, this is in the tens of nanoseconds at 5 V, which is fast enough for most robot logic, but not for a high-speed bus.

The common mistake is wiring an unused NAND input to nothing. Tie it to a defined level, either high or low depending on the gate, or the unused input causes random behavior and extra power draw. The limitation of a chip gate is fan-out, the number of inputs a single output can drive. Each 74HC output can drive several inputs, but a long chain of gates adds delay that accumulates.

Gates are the building blocks of every processor and every interlock on a robot, from a simple AND between a safety switch and a motor enable to the instruction decoder inside the microcontroller.

---

## Boolean algebra

Boolean algebra is the arithmetic of logic. It uses AND as multiplication, OR as addition and NOT as a bar over the variable. It has rules that let you simplify an expression into a cheaper circuit with fewer gates.

Three laws do most of the work. The first is the identity `A·B + A·B' = A`, which you can show by factoring: `A·(B + B') = A·1 = A`. The second is De Morgan's law, `(A·B)' = A' + B'`, which says that a NAND is an OR of inverted inputs. The third is the absorption law, `A + A·B = A`.

A simple safety interlock shows the value. A motor may run when the enable switch `E` is on and either the sensor `S1` or the sensor `S2` is clear. Write it as `Run = E · (S1 + S2)`. If you also have a backup term `E · S1 · S2`, the expression simplifies to `E · (S1 + S2)`, because `E · S1 · S2` is already inside the bracket. Fewer terms means fewer gates and fewer things to wire wrong.

The common mistake is applying De Morgan's law without inverting every term. The correct form is `(A + B)' = A' · B'`, with the operator changed as well. Check each simplification with a truth table, which is the next section.

The limitation of algebra is size. For five or six inputs, simplification by hand is tedious, and a Karnaugh map or a software tool is faster. Logic synthesis in hardware description languages automates the same steps.

Boolean algebra is how interlock logic and PLC programs are written in industrial robots, where a simplified expression means fewer parts that can fail.

---

## Truth tables

A truth table lists every input combination and the output for each. For `n` inputs, there are `2^n` rows. Two inputs give four rows, three give eight, and four give sixteen.

A checklist that covers every case before a safety sign-off is the right model. Missing a row means missing a case.

The half adder is the simplest useful circuit. It adds two bits, `A` and `B`, and produces a sum bit `S` and a carry bit `C`. Its table is:

| A | B | S (sum) | C (carry) |
|---|---|---|---|
| 0 | 0 | 0 | 0 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 1 |

The sum is `A XOR B`, and the carry is `A AND B`. Check the row `1 + 1`: the sum is 0 and the carry is 1, which is `10` in binary, equal to 2 in decimal.

The common mistake is building a truth table with missing rows, which happens when inputs are not enumerated in binary order. Generate the rows with a counter, and check that the count is `2^n`. The limitation of truth tables is that they scale exponentially, so a 20-input table is impossible to write by hand. For large systems, use hierarchical design and simulation.

Truth tables are the test suite of digital design. A test engineer writes the expected table first, then checks the circuit against it.

---

## Combinational logic

Combinational logic is a network of gates whose output depends only on the present inputs. It has no memory, and its outputs change as soon as the inputs change, after the propagation delay. Adders, multiplexers, decoders and parity checkers are all combinational.

A vending machine gives the right change from the coins you insert right now. It does not remember the coins you used yesterday.

A half adder is combinational. A full adder adds three bits and is built from two half adders and an OR gate. Chained together, full adders make a binary adder, the arithmetic core of a processor.

A decoder selects one output line from a binary code. A 2-to-4 decoder takes two inputs and drives one of four outputs high. Robot controllers use decoders to select a register, a motor channel or a sensor chip from an address.

The common mistake is ignoring the propagation delay in a long chain. The output settles only after the slowest path finishes, and a glitch can appear briefly on the output, which triggers a false action when it feeds a clock or an enable line. The limitation is that combinational logic cannot remember anything. For memory, you need sequential logic.

Combinational logic sits between sensors and actuators in a robot, deciding which motor, which channel or which status flag a set of inputs selects.

---

## Sequential logic

Sequential logic includes memory. Its output depends on the present inputs and on a stored state. The basic memory element is the flip-flop, which holds one bit until a clock edge tells it to update.

A notepad is a useful picture. Writing on a page changes the note, and the note stays until you write again. A clock is the moment you decide to write.

A D flip-flop captures its D input on the rising edge of the clock. Between edges, its output Q holds the last captured value. The 74HC74 contains two D flip-flops, each with a clock, a D input, an output Q and an inverted output. Connecting the inverted output back to D makes a toggle, which changes state on each clock edge. The output frequency is then half the clock frequency.

Take a clock at 1 kHz driving a toggle flip-flop. The output toggles every edge, so it completes one full cycle every two clock periods. Its frequency is `1000 / 2 = 500 Hz`. Cascade four toggles, each feeding the clock of the next, and you have a 4-bit binary counter, which counts from 0 to 15 and then wraps.

The common mistake is changing the D input at the same time as the clock edge, which the flip-flop may capture either way. The fix is to change inputs only between edges, which is why microcontroller code writes outputs at known moments. The limitation of flip-flops is timing: every flip-flop has setup and hold times, and a signal that changes too close to the clock edge produces a metastable state. Asynchronous inputs, such as a button, must be synchronized through a flip-flop before the logic uses them. Day 8's debounce handles the bounce, and this two-flip-flop synchronizer handles the timing.

Sequential logic sits behind every counter that reads an encoder, every state machine that sequences a robot's startup, and every register in a microcontroller.

---

## Hands-on exercises (about 60 minutes)

Work through these in order. Check each result against a truth table or a hand calculation.

1. **Convert numbers.** Convert 37 to binary and hex by hand. Then convert 200 and 255. Check each answer with Python's `bin()` and `hex()` in the interactive prompt. Record the results in a table.
2. **Build gates in Logisim Evolution.** Open Logisim Evolution. Place two input pins and one output pin, and build an AND gate, then an OR gate, then an XOR gate. For each, use the input pins to step through all four combinations, and record the output in a truth table.
3. **Build a half adder.** In Logisim Evolution, connect an XOR gate for the sum and an AND gate for the carry from the same two inputs. Enter all four combinations and check your truth table against the one in this lesson.
4. **Read four switches as a byte.** Wire the DIP switch to pins 2 to 5 of the Uno, with each switch closed to GND. Upload the sketch below and open the Serial Monitor at 115200 baud. Set each switch, and check the printed binary value.

```cpp
const int pins[4] = {2, 3, 4, 5};

void setup() {
  Serial.begin(115200);
  for (int i = 0; i < 4; i++) {
    pinMode(pins[i], INPUT_PULLUP);
  }
}

void loop() {
  uint8_t value = 0;
  for (int i = 0; i < 4; i++) {
    if (digitalRead(pins[i]) == LOW) {   // closed switch pulls the pin low
      value |= (1 << i);
    }
  }
  Serial.println(value, BIN);
  delay(250);
}
```

5. **Build a NAND gate from a 74HC00 chip (optional).** Connect the 74HC00 to 5 V and GND, with its power pins on the chip's corners (pins 14 and 7). Wire one NAND gate with two inputs from switches and the output to an LED through a 220 Ω resistor. Tie the unused inputs of the other gates to GND. Check the LED against the NAND truth table.
6. **Simulate a counter.** In Logisim Evolution, build a 4-bit counter from four toggle flip-flops with a clock. Run it and confirm the output counts 0 to 15 and wraps.

**Safety notes:** keep each LED under 20 mA. Keep 74HC chips at 5 V or 3.3 V, and do not exceed the supply rating on the datasheet. Remove power before you change wiring on the breadboard.

**Done when:** you have truth tables for AND, OR, XOR and NAND, the half adder, and the DIP switch reading, and the counter counts 0 to 15 in simulation.

---

## Project: a bit toolkit with a truth table generator

This project is a light one. The toolkit is a small Python module that handles the conversions and the truth table generation you practiced today. Its purpose is to make the checks repeatable, so that a simplified expression is verified by a program and not by memory.

**Goal:** write functions to convert numbers to binary strings, to compute two's complement, to generate a truth table for any Boolean function of `n` inputs and to check De Morgan's law by brute force. Add the DIP switch sketch as firmware.

**Repo name:** `100-days-robotics-engineering-day-10-digital-electronics`

**Repo layout:**

```
100-days-robotics-engineering-day-10-digital-electronics/
├── README.md
├── bits.py
├── test_bits.py
└── firmware/
    └── dip_reader/
        └── dip_reader.ino
```

**Starter code** (`bits.py`):

```python
from itertools import product
from typing import Callable


def to_binary(value: int, width: int = 8) -> str:
    """Unsigned binary string of a given width. Raises if the value does not fit."""
    if value < 0 or value >= 2 ** width:
        raise ValueError(f"{value} does not fit in {width} unsigned bits")
    return format(value, f"0{width}b")


def twos_complement(value: int, width: int = 8) -> str:
    """Binary string of a signed integer in two's complement."""
    if not -(2 ** (width - 1)) <= value < 2 ** (width - 1):
        raise ValueError(f"{value} does not fit in {width} signed bits")
    return format(value & (2 ** width - 1), f"0{width}b")


def truth_table(func: Callable[..., bool], n_inputs: int) -> list[tuple[tuple[int, ...], int]]:
    """Every input combination in binary order, with the output of func."""
    rows = []
    for bits in product((0, 1), repeat=n_inputs):
        rows.append((bits, int(bool(func(*bits)))))
    return rows


def demorgan_holds() -> bool:
    """Check (A AND B)' == A' OR B' for all inputs."""
    lhs = truth_table(lambda a, b: not (a and b), 2)
    rhs = truth_table(lambda a, b: (not a) or (not b), 2)
    return lhs == rhs
```

**Step-by-step tasks:**

1. Create the repo and a virtual environment, and install `pytest`.
2. Implement the four functions, with docstrings that state the width and the signed range.
3. Write tests: `to_binary(37)` returns `"00100101"`, `twos_complement(-5)` returns `"11111011"`, the truth table of XOR has four rows with outputs `0, 1, 1, 0`, `demorgan_holds()` is True, and `to_binary(256)` raises `ValueError`.
4. Save `dip_reader.ino` with the sketch from the exercises.
5. Add a test that generates a three-input truth table and checks it has eight rows.
6. Write the README: the function list, the test command, the Logisim truth tables you built, and a note on the byte you read from the DIP switch at each setting you tried.

**README checklist:**

- [ ] Setup and run commands
- [ ] Function table with width and range notes
- [ ] Truth tables for AND, OR, XOR and NAND
- [ ] DIP switch readings at four settings
- [ ] One limitation of the truth table approach

**Done when:**

- `pytest -q` passes with at least five tests
- The DIP switch reading matches the switch positions you set
- The Logisim half adder matches the table in the README
- The repo is pushed to GitHub with a clear commit message

**Evidence to commit:** the source, the tests, the sketch, a Logisim file for the half adder and the README. A screenshot of the Serial Monitor showing the byte readings helps a reviewer check the firmware.

---

## Check your understanding

1. A sensor returns the 8-bit signed value `11110110` from a register. Decide its decimal value, then explain what the value would be if the driver mistakenly read the register as unsigned.
2. Simplify `Y = A·B + A·B'·C + A·B·C`. Show each step, and state the number of gates the simplified form needs compared with the original.
3. A safety interlock must allow a motor to run only if the enable switch is closed and neither the left nor the right limit switch is triggered. Write the expression, and check it against the truth table for the enable and the two limit switches.
4. A 3-bit binary counter is built from three toggle flip-flops, each divided by 2. The input clock is 8 kHz. Calculate the frequency of each flip-flop output and of the most significant bit.
5. A student wires a NAND gate with one unused input left floating and reports random output behavior. Explain the cause, and give two valid fixes.
6. Explain why a button must go through a synchronizer before the logic uses it, and what the synchronizer does to the timing of a single press.

**Answers:**

1. The value is negative because the top bit is 1. Its value is `−128 + 64 + 32 + 16 + 4 + 2 = −10`. Reading it unsigned gives `128 + 64 + 32 + 16 + 4 + 2 = 246`, so the sign error changes the result by 256.
2. `A·B + A·B'·C + A·B·C`. Factor the first and third terms: `A·B + A·B·C = A·B·(1 + C) = A·B`. Then `Y = A·B + A·B'·C`. Factor again: `A·(B + B'·C) = A·(B + C)`, because `B + B'·C = B + C`. The simplified form is `Y = A·(B + C)`, which needs one OR gate and one AND gate, against three AND gates and an OR gate for the original.
3. `Run = E · L' · R'`, where `L` and `R` are the limit switches triggered. Check the truth table: the motor runs only when `E = 1`, `L = 0` and `R = 0`. Any triggered limit or an open enable stops the motor, which is the fail-safe behavior.
4. Each toggle halves the frequency. The first output is `8000 / 2 = 4 kHz`. The second is `2 kHz`. The third, the most significant bit, is `1 kHz`.
5. A floating input reads a random level, and noise can switch the output. The fix is to tie the unused input to a defined level. Tying it high makes the NAND act as an inverter on the other input. Tying it low holds the output high. Either choice is deliberate, and a floating pin is never one of them.
6. A button press is asynchronous to the clock, so it can change at any time, including inside a flip-flop's setup or hold window. A synchronizer uses two flip-flops in a row to settle the signal onto the clock, which can add up to two clock periods of delay before the logic sees the press. The press itself still registers once, and the delay is the cost of a stable input.

---

## Sources to read and watch

**Read**

- Wikipedia: *Binary number*, *Hexadecimal*, *Bit*, *Byte*, *Two's complement*, *Logic level*, *Logic gate*, *Boolean algebra*, *Truth table*, *Combinational logic*, *Sequential logic*, *Flip-flop (electronics)*, *De Morgan's laws*, *Bitwise operation*
- Texas Instruments datasheet for the SN74HC00 and SN74HC74, which you can find by searching the part number on the TI website
- Arduino Language Reference, [docs.arduino.cc](https://docs.arduino.cc), for `pinMode`, `digitalRead`, `Serial.println` and bitwise operators

**Tutorials**

- GeeksforGeeks, the tutorials on number systems, logic gates and Boolean algebra. Search the site for the titles.
- W3Schools, the C tutorial section on bitwise operators, for `&`, `|`, `^`, `~`, `<<` and `>>`
- Adafruit Learning System, the guide on the 74HC series and logic chips. Search the Adafruit site for "74HC".

**Watch**

- MIT OpenCourseWare, 6.004 Computation Structures, the lectures on combinational and sequential logic
- Ben Eater's YouTube channel, his series on building an 8-bit computer on a breadboard, which uses 74 series logic chips. Search the channel for the series title.

**Practice**

- [Logisim Evolution](https://github.com/logisim-evolution/logisim-evolution), free and open source, for building the gates, the half adder and the counter in this lesson

---

## Tomorrow

Tomorrow you turn to analog signals and the ways digital and analog meet. Today every signal was 0 or 1. The real world gives a voltage that varies smoothly, such as a light sensor or a battery monitor. You will learn analog-to-digital conversion, how resolution and sampling set the accuracy of a reading, and how filters clean up noise before the Arduino's ADC pins read the signal. The 8-bit and 10-bit numbers you practiced today are exactly what the ADC returns.
