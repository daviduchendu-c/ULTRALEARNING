# Day 4: Measurement and Engineering Tools

*100 Days of Robotics Engineering, Module 1: Robotics Engineering Foundations*

**Time:** about 4 hours (plus shopping time if you buy tools)
**You will build:** a costed bench toolkit plan, a measurement log with real or simulated readings, datasheet notes for a real motor driver, and a debugging template you will use for the rest of the series.
**You should know:** Days 1 to 3. You do not need to own any tools yet. Every exercise has a no-hardware option.

---

## 1. The big idea

A robot that "doesn't work" tells you nothing. A motor driver that reads 7.8 V on its supply pin and 0 V on its standby pin tells you exactly what to fix. Measurement turns guessing into engineering.

On Day 3 you wrote test plans with a "Result" column. Today you learn the instruments that fill it in, how to use them without hurting yourself or your parts, and how to read the datasheet that says what the number should be. Test engineers, technicians and firmware engineers are judged on how quickly they find a fault on a bench.

## 2. Safety first

Most of what you build in this series runs on low voltage (under 30 V DC), which is not a shock hazard for dry skin. The real dangers are heat, fire and eyes.

| Hazard | Where it comes from | What to do |
|---|---|---|
| Burns | Soldering iron tips run at roughly 300 to 400 °C | Return the iron to its stand. Never catch a falling iron. |
| Eye injury | Clipped leads fly; solder can spit | Wear safety glasses when soldering and cutting. |
| Lithium battery fire | Shorted, punctured, overcharged or over-discharged Li-ion and LiPo cells | Fuse every pack. Charge with a balance charger on a non-flammable surface, never unattended. Store LiPo cells at about 3.8 V per cell. Recycle swollen packs. |
| Fumes | Flux smoke from solder | Ventilate, or blow smoke away from your face with a fan or extractor. |
| Lead | Leaded solder | Wash your hands after soldering. No food or drink at the bench. Lead-free solder avoids this but needs a hotter iron. |
| Mains electricity | Wall sockets, power supply internals | Do not open or probe anything plugged into the wall in this series. Mains work needs training. |
| Short circuits | A dropped probe or a loose wire across a battery | Set current limits, insulate exposed wires, and remove rings and watches. |

## 3. Electrical tools

### 3.1 Multimeter

**Intuition.** The multimeter is your thermometer. You would not treat a fever without a reading, and you should not debug a circuit without a voltage reading.

A digital multimeter measures:

- **DC voltage (V⎓):** the probes sit across two points, in parallel with the circuit.
- **Current (A):** the meter goes in series, so the current flows through it. You must break the circuit to add it.
- **Resistance (Ω):** only on unpowered circuits.
- **Continuity:** a beep when resistance is very low. It is the best tool for finding broken wires.
- **Diode test:** shows the forward voltage of a diode or LED.

```
 Voltage (parallel)            Current (series)
   ┌──[ load ]──┐               ┌──[ load ]──[ METER ]──┐
   │            │               │                       │
  (+)  METER   (−)             (+)       battery       (−)
   └──battery───┘               └───────────────────────┘
```

**Worked example.** An LED with a 2.0 V forward voltage and a 330 Ω resistor runs from 5 V. The expected current is (5.0 − 2.0) / 330 = 9.1 mA. Put the meter in series on the mA range and read about 9 mA. If you read 0 mA, check each joint with continuity. If you read about 15 mA, measure the resistor: it may be 200 Ω. (Day 6 covers Ohm's law properly. For now, compare measured with expected.)

**Common mistakes.**
- Leaving the red lead in the current (A) jack and then measuring a battery's voltage. In current mode the meter is nearly a short circuit. You blow the meter's fuse, or with a LiPo you create a dangerous short. Move the lead back to V/Ω after every current measurement.
- Measuring resistance on a powered circuit. The reading is wrong, and you can damage the meter.
- Trusting a cheap meter's printed "CAT III 1000 V" label. Many cheap meters do not meet the ratings they print. They are fine for robots, not for mains.

**Limitation.** A multimeter shows an average. A 5 V PWM signal at 25 percent duty reads about 1.25 V, which hides the switching. You need a scope for that.

### 3.2 Bench power supply

**Intuition.** A battery gives whatever current the circuit demands, even if that melts a wire. A bench supply is a battery with a safety valve. You set the maximum current, and it will not give more.

You set two knobs: voltage and current limit. If the circuit draws less than the limit, the supply holds the voltage (**CV mode**). If the circuit tries to draw more, the supply drops the voltage to hold the current at the limit (**CC mode**), and an indicator lights.

**Worked example: a safe gearmotor test.**

1. With nothing connected, set 7.4 V (the battery's nominal voltage) and a 0.5 A limit.
2. Connect the motor. It spins freely and draws 0.15 A. The supply stays in CV mode.
3. Grip the wheel to stall it. The supply jumps to CC mode at 0.5 A, and the voltage drops to, say, 2.1 V.
4. The motor's winding resistance is about 2.1 / 0.5 = 4.2 Ω. At the full 7.4 V, the stall current would be about 7.4 / 4.2 ≈ 1.8 A. That number goes straight into the motor driver choice from Day 2.

**Common mistakes.**
- Turning the supply on with the current limit at maximum. The first short circuit destroys the part you are testing.
- Charging lithium batteries from a bench supply. Use a charger with balancing and cutoff.
- Forgetting that motors push energy back when they slow down. Large motors can push the supply's output above its set voltage.

### 3.3 Oscilloscope

**Intuition.** A multimeter is a photo of a moving car: one blurred number. A scope is the video, showing the car move.

A scope plots voltage (vertical, volts per division) against time (horizontal, seconds per division). The specs that matter are:

- **Bandwidth:** the highest frequency measured accurately. For robot work, 50 to 100 MHz is plenty.
- **Sample rate:** at least several times the bandwidth.
- **Channels:** two is the minimum, four is comfortable.
- **Trigger:** tells the scope when to start drawing, such as "when channel 1 rises through 2.5 V," so a repeating signal stays still on screen.

**Worked example.** Your firmware outputs 1 kHz PWM at 25 percent duty on a 3.3 V pin.

- Period: 1 / 1,000 Hz = 1 ms. High time: 0.25 × 1 ms = 0.25 ms.
- Set the time base to 200 µs per division. A typical screen has 10 horizontal divisions, so you see 2 ms: two full periods.
- Set 1 V per division and trigger on the rising edge at about 1.6 V.
- You should see a square wave: 0.25 ms high at 3.3 V, 0.75 ms low. If the edges show big spikes, or the low level wobbles when the motor runs, you have found noise that a multimeter would never show.

**Common mistakes.**
- A probe set to 10× while the scope is set to 1× (or the reverse), so every reading is off by a factor of ten.
- Mains safety: the scope's ground clip connects to the earth pin of the wall socket through the power cord. On a battery-powered robot this is not a hazard. On anything plugged into the wall, clip the ground only to ground, or the clip creates a short through the scope.
- Long ground leads on fast signals, which add ringing that is not in the circuit.

**Limitation.** A 2- or 4-channel scope sees only a few signals. Debugging an 8-wire bus needs a logic analyzer.

### 3.4 Logic analyzer

**Intuition.** A scope is a detailed video of one or two people. A logic analyzer is a grid of security cameras: less detail (only high or low), but many signals at once, with subtitles (decoded protocol data).

A logic analyzer samples digital lines (usually 8 or 16) and records each one as 0 or 1. Software decodes the protocols: I2C addresses and bytes, UART characters, SPI transfers. **Sample rate rule:** sample at least 4 times faster than the fastest signal, and 10 times is safer.

**Worked example.** Your IMU on I2C at 400 kHz returns nothing.

- A cheap 24 MHz analyzer gives 24,000,000 / 400,000 = 60 samples per clock period. That is plenty.
- Connect GND, SDA and SCL, capture, and add the I2C decoder in PulseView.
- The decode shows the MCU sending address 0x68 and getting a NACK (no acknowledgment). The sensor is not answering. Now ask: is it powered? Is its address pin pulling it to 0x69 instead? The analyzer turned "nothing works" into one specific question.

For UART at 115,200 baud, one bit lasts 1 / 115,200 ≈ 8.68 µs, so 24 MHz sampling gives about 208 samples per bit.

**Common mistakes.**
- Forgetting to connect the analyzer's ground to the circuit ground. The capture is garbage.
- Exceeding the input voltage. Many cheap analyzers accept 0 to 5 V only. Never connect one to motor voltage.
- Decoding at the wrong baud rate or with the wrong channel assignment, then blaming the device.

**Limitation.** A logic analyzer cannot show voltage levels or noise. A line at 2.1 V on a 3.3 V system may read as valid to the analyzer and invalid to your chip. Use a scope for that.

## 4. Physical tools

### 4.1 Calipers

**Intuition.** A ruler tells you "about 3 mm." A caliper tells you "3.02 mm," which is the difference between a wheel that fits the motor shaft and one that wobbles.

Digital calipers measure outside dimensions (jaws), inside dimensions (the small top jaws), depth (the rod at the end) and steps. Typical resolution is 0.01 mm, with accuracy around ±0.02 to ±0.03 mm on decent units.

**Worked example.** You need the center distance between two mounting holes on a motor bracket, each 3.2 mm in diameter.

- Measure outside edge to outside edge of the two holes: 28.4 mm.
- Center distance = 28.4 − 3.2 = 25.2 mm. You subtract one diameter, since half a diameter comes off each side.
- Check by measuring inside edge to inside edge. It should read 25.2 − 3.2 = 22.0 mm.

**Common mistakes.** Not zeroing the jaws before measuring. Squeezing too hard on soft plastic. Measuring a 3D-printed part once instead of three times at different spots.

**Limitation.** Cheap calipers drift, and their batteries die. Check them against a known part, such as a drill bit shank.

### 4.2 Breadboards

**Intuition.** A breadboard is LEGO for circuits. Push parts in, test, pull them out, rearrange.

Inside a solderless breadboard are metal spring clips. Each short row of five holes is connected. The long rails along the edges carry power and ground.

```
  + rail  ● ● ● ● ●   ● ● ● ● ●     (all connected along the length; some boards
  − rail  ● ● ● ● ●   ● ● ● ● ●      split the rail in the middle, so check)

          a b c d e     f g h i j
     1    ●─●─●─●─●     ●─●─●─●─●    ← a to e connected, f to j connected
     2    ●─●─●─●─●     ●─●─●─●─●       the centre gap separates them
```

**Worked example.** Map a new breadboard with continuity mode. Touch one probe to the top of the + rail and the other to the bottom. If there is no beep, the rail is split in the middle, and you need a jumper. This one check saves hours.

**Common mistakes.** Running motor current through a breadboard (the clips are meant for well under 1 A). Loose, cheap boards that cause intermittent faults. Long tangled jumpers that are impossible to check.

**Limitation.** Breadboards add stray capacitance and poor contacts. They are bad for fast signals and high current. Move a working circuit to soldered perfboard or a PCB (Module 6).

### 4.3 Soldering tools

**Intuition.** Solder is not glue. It is a metal that melts below the temperature of copper and wets both surfaces, which makes an electrical and mechanical bond.

The kit: a temperature-controlled iron, rosin-core solder (leaded 63/37 or 60/40, or lead-free SAC305), flux, a brass wool tip cleaner, solder wick, flush cutters, wire strippers, heat-shrink tubing and a holder.

**Worked example: a through-hole joint.**

1. Clean the tip and tin it with a little solder.
2. Touch the tip to both the pad and the lead for 1 to 2 seconds.
3. Feed solder into the joint, not onto the tip. Remove the solder, then the iron.
4. A good joint is shiny (leaded) or slightly matte (lead-free), shaped like a small volcano, and covers the pad fully.

Typical tip temperatures are 320 to 350 °C for leaded solder and 350 to 380 °C for lead-free.

**Common mistakes.** A cold joint (dull, lumpy, balled up) from too little heat. Bridging two pads. Heating a pad so long that it lifts off the board. Forgetting to slide heat-shrink onto a wire before soldering the second end.

**Limitation.** Bad joints often pass a quick test and then fail when the robot vibrates. Tug-test every wire joint and inspect pads with a magnifier.

## 5. Reading datasheets

**Intuition.** A datasheet is the part's manual and its contract. If the datasheet does not promise it, do not rely on it.

A typical datasheet has these parts, in roughly this order:

1. **Features and summary** on page 1: the headline numbers.
2. **Pinout:** which pin does what.
3. **Absolute maximum ratings:** limits that cause damage. Never design to these.
4. **Recommended operating conditions:** the range where the part works as specified. Design to these.
5. **Electrical characteristics:** typical, minimum and maximum values under stated conditions.
6. **Timing diagrams, application circuit and package drawing.**

**Worked example: the TB6612FNG motor driver from Day 2.** Open Toshiba's datasheet (search "TB6612FNG datasheet" on toshiba.semicon-storage.com) and find:

- Motor supply absolute maximum: 15 V. A 3S LiPo at 12.6 V full charge is close to that, so check the recommended operating range before trusting it.
- Output current: 1.2 A average and 3.2 A peak per channel. The stall estimate from Section 3.2 (about 1.8 A) exceeds the 1.2 A average. You either limit stall time in firmware or choose a bigger driver.
- Logic supply: 2.7 to 5.5 V, so it works with both 3.3 V and 5 V MCUs.
- A **STBY** (standby) pin that must be driven high before the outputs work. Beginners miss this often.

**Common mistakes.** Reading only page 1. Designing to absolute maximums. Trusting a breakout seller's summary instead of the manufacturer's datasheet. Ignoring the test conditions printed above each table, such as "at 25 °C."

**Limitation.** "Typical" values are not guarantees. Design with the minimum or maximum value that hurts most.

## 6. Systematic debugging

**Intuition.** A doctor does not guess. The doctor checks vital signs, forms a hypothesis, orders one test, and narrows it down. Debugging works the same way.

The loop:

```
 1. Reproduce  →  2. Observe and measure  →  3. Hypothesize one cause
        ▲                                               │
        │                                               ▼
 6. Record in notebook  ←  5. Compare result  ←  4. Test one change
```

Rules: check power first, then ground, then clocks and resets, then signals, then code. Split the system in half and test the middle. Change one thing at a time. Write everything down in the notebook from Day 3.

**Worked example: "the left motor does nothing."**

| Step | Measurement | Result | Conclusion |
|---|---|---|---|
| 1 | Battery voltage | 7.8 V | Battery fine |
| 2 | Driver VM pin to GND | 7.8 V | Power reaches the driver |
| 3 | Driver VCC (logic) to GND | 3.3 V | Logic powered |
| 4 | PWMA pin on a scope or meter | Switching, about 1.6 V average | MCU is sending PWM |
| 5 | STBY pin | 0 V | Found it: standby is active |
| 6 | Wire STBY to 3.3 V and retest | Motor spins | Fixed. Update the wiring diagram and notebook. |

Six measurements, no guessing, and the wiring diagram is updated in the same commit.

**Common mistakes.** Rewriting code when the fault is a loose wire. Changing three things at once. Not writing down what you already ruled out, then checking it again.

**Limitation.** Intermittent faults, such as loose connectors or brownouts under load, may not appear while you measure. Log data over time, and wiggle connectors on purpose while you watch the readings.

## 7. What to buy

Prices are approximate USD ranges from 2025 to 2026 retail listings. They vary by country and seller and change often, so check before you buy.

| Tool | Starter | Step up | Notes |
|---|---|---|---|
| Multimeter | AstroAI or Kaiweets auto-ranging, $15 to $40 | UNI-T UT61E+ or Brymen BM235, $50 to $150 | Fluke 117 or 87V, $200 to $500+, for trusted safety ratings |
| Bench supply | Generic 30 V / 5 A or 10 A switching, $50 to $90 | Korad KA3005D or KA3005P, Riden RD6006, $80 to $160 | Siglent SPD3303X-E or Rigol DP832, $300 to $600. Check for a real current limit |
| Oscilloscope | FNIRSI or Hantek handheld/USB, $50 to $250 | Rigol DHO800, Siglent SDS1104X-E, Rigol DS1054Z, $300 to $550 | Keysight, Tektronix or R&S entry models, $1,000+ |
| Logic analyzer | Generic 24 MHz, 8-channel USB, $10 to $20, used with the free sigrok PulseView | Saleae Logic 8 or Logic Pro 8, about $500 to $1,000 list | Saleae has offered student and hobbyist discounts |
| Calipers | 150 mm digital, $15 to $35 | Mitutoyo 500-196-30 (150 mm Absolute Digimatic), $120 to $180 | Check against a known part |
| Breadboard | 830-point board, $5 to $10, plus jumper kit, $5 to $10 | Branded boards with better clips | Fine for low current only |
| Soldering | PINE64 Pinecil V2, $26 to $40, plus a USB-C PD adapter, $15 to $30 | Hakko FX-888DX, $110 to $150 | Solder, flux, wick and tip cleaner, $20 to $40 total |
| Fume and eye protection | Fan with carbon filter, $30 to $100; safety glasses, $5 to $15 | | |

A USB-C PD trigger board or a fused battery pack can stand in for a bench supply at first, but neither has an adjustable current limit. If a scope is not in your budget yet, wait. A multimeter and a cheap logic analyzer carry you through Module 3.

## 8. Hands-on exercises (about 60 minutes)

**With a multimeter (any starter meter):**

1. Measure a fresh AA battery and a used one. Record both. A fresh alkaline cell reads about 1.5 to 1.6 V with no load.
2. Read the color bands on three resistors, then measure them. Calculate the percentage difference from the marked value.
3. Use continuity mode to test ten jumper wires. Set aside any that fail.
4. Map your breadboard's power rails, as in Section 4.2.

**Without hardware:**

1. In the [Falstad circuit simulator](https://www.falstad.com/circuit/), build a 5 V source, a 330 Ω resistor and an LED. Read the current and compare it with 9.1 mA.
2. In [Wokwi](https://wokwi.com/), make a Raspberry Pi Pico project that outputs PWM, add Wokwi's virtual logic analyzer, and open the exported capture in PulseView. Measure the period and duty cycle.

**Everyone:**

1. Download the TB6612FNG datasheet and find the four values from Section 5. Note the page number of each.
2. Write a debugging table like Section 6 for this fault: "The robot works on the bench supply but resets on the battery."

## 9. Project: bench toolkit and measurement log

**Why this project fits:** the topic is practical, and buying tools is optional. The project produces evidence of two real skills: making a costed equipment decision, and logging measurements in a way another engineer can trust.

**Repo name:** `100-days-robotics-day-04-measurement-tools`

```
100-days-robotics-day-04-measurement-tools/
├── README.md
├── safety.md                    # your bench rules, adapted from Section 2
├── toolkit.csv                  # tools: owned / planned, price, why
├── measurements/
│   ├── 2026-10-24-multimeter.md # readings with expected vs measured
│   └── photos/                  # meter screen photos or simulator screenshots
├── datasheets/
│   └── tb6612fng-notes.md       # key ratings with page numbers
└── debugging/
    ├── template.md              # the six-step table, blank
    └── case-01-reset-on-battery.md
```

**Starter `toolkit.csv`:**

```csv
tool,example,tier,status,price_low_usd,price_high_usd,why_i_need_it
multimeter,AstroAI or Kaiweets auto-ranging,starter,planned,15,40,voltage and continuity on every build
soldering iron,PINE64 Pinecil V2 + USB-C PD adapter,starter,planned,41,70,headers and motor wires
logic analyzer,24 MHz 8-channel + PulseView,starter,planned,10,20,debug I2C and UART in Module 3
calipers,150 mm digital,starter,planned,15,35,motor shafts and mounting holes
breadboard,830-point + jumpers,starter,owned,10,20,prototyping
```

**Steps.**

1. Write `safety.md` as rules you will actually follow, including where you charge batteries.
2. Fill `toolkit.csv` with every tool from today, marking each one owned, planned or skipped, and total the starter kit. A reasonable starter bench (meter, iron with adapter and consumables, glasses, calipers, breadboard kit, logic analyzer) comes to roughly $120 to $250.
3. Record at least ten measurements, each with the expected value, the measured value, the percentage error and the instrument used.
4. Write the datasheet notes with page numbers.
5. Complete the debugging case from the exercises, following the six-step loop, and end with a recorded conclusion.

**README checklist.**

- [ ] Total starter kit cost, with low and high estimates
- [ ] Which tools you are buying first, and why
- [ ] Link to the measurement log and at least one photo or screenshot
- [ ] One paragraph on the safety rule that matters most to you

**Done when.**

- Every measurement has units, an expected value and an instrument.
- The datasheet notes would let someone choose a battery voltage for the TB6612FNG without opening the datasheet.
- The debugging case follows the six-step loop and ends with a recorded conclusion.

**Evidence to commit:** all files above, including photos or simulator screenshots of real readings.

## 10. Check your understanding

1. Your multimeter reads 1.65 V on a 3.3 V PWM pin. What is the likely duty cycle, and why would a scope be more convincing?
2. You set a bench supply to 12 V with a 3 A limit to test a small sensor board. What is wrong with that, and what should the limit be?
3. Why must you never measure a battery's voltage with the red lead in the current jack?
4. An I2C capture shows the address byte followed by a NACK. List two hardware causes.
5. A datasheet lists 15 V as the absolute maximum motor voltage. Why is a 3S LiPo (12.6 V full) a risky choice without checking further?
6. A motor works on the bench supply, but the robot resets on the battery. Which measurement would you take first, and with which tool?
7. Why is a breadboard a poor place for your motor power wiring?

**Answers.**

1. About 50 percent (1.65 / 3.3). The meter shows only the average, so a steady 1.65 V would read the same. The scope shows the actual switching.
2. The limit is far too high. A fault would let up to 3 A flow and burn something. Set it slightly above the expected draw, for example 100 to 200 mA for a small board.
3. In current mode the meter has almost no resistance, so it short-circuits the battery. At best it blows the fuse. At worst it starts a battery fire.
4. The sensor is unpowered or miswired, its address pin sets a different address, pull-up resistors are missing, or SDA and SCL are swapped.
5. 12.6 V leaves only 2.4 V of headroom below the damage limit, and motor switching spikes can exceed that. The recommended operating range may be lower than 12.6 V.
6. The logic supply voltage during motor start, captured on a scope or with a meter's min/max mode. It is probably dipping below the MCU's minimum when the motors draw current.
7. Breadboard clips are rated well under 1 A, the contacts are loose, and heat or a poor contact causes intermittent faults.

## 11. Sources

**Read:**
- [Wikipedia: Multimeter](https://en.wikipedia.org/wiki/Multimeter)
- [Wikipedia: Oscilloscope](https://en.wikipedia.org/wiki/Oscilloscope)
- [Wikipedia: Logic analyzer](https://en.wikipedia.org/wiki/Logic_analyzer)
- [Wikipedia: Calipers](https://en.wikipedia.org/wiki/Calipers)
- [Wikipedia: Breadboard](https://en.wikipedia.org/wiki/Breadboard)
- [Wikipedia: Soldering](https://en.wikipedia.org/wiki/Soldering)
- [Wikipedia: Datasheet](https://en.wikipedia.org/wiki/Datasheet)
- [Wikipedia: Debugging](https://en.wikipedia.org/wiki/Debugging)
- [Wikipedia: Lithium-ion battery](https://en.wikipedia.org/wiki/Lithium-ion_battery) (read the "Safety" section)
- [sigrok PulseView](https://sigrok.org/wiki/PulseView) documentation
- Toshiba TB6612FNG datasheet: search "TB6612FNG datasheet" on toshiba.semicon-storage.com
- Fluke application note on multimeter safety categories: search "ABCs of multimeter safety" on fluke.com

**Tutorials:**
- SparkFun Learn: "How to Use a Multimeter," "How to Use a Breadboard," "How to Solder: Through-Hole Soldering" and "How to Use an Oscilloscope" (search the titles on learn.sparkfun.com)
- Adafruit Learn: "Adafruit Guide To Excellent Soldering" (search the title on learn.adafruit.com)
- GeeksforGeeks: search "Multimeter" and "Oscilloscope" for short explainers

**Watch:**
- [EEVblog](https://www.youtube.com/@EEVblog): search the channel for "multimeter tutorial" and "oscilloscope tutorial"
- [GreatScott!](https://www.youtube.com/@greatscottlab): search "Electronic Basics" for multimeter and oscilloscope episodes
- Rigol and Siglent official YouTube channels: getting-started videos for their entry scopes

**Practice:**
- [Falstad circuit simulator](https://www.falstad.com/circuit/) and [Wokwi](https://wokwi.com/) for measurements without hardware

**Note on links:** open each one once. If a page has moved, search its title. Prices are approximate and change often, so check before you buy.

## 12. Tomorrow

Day 5, **First Robot System Design**, brings Module 1 together. You will design a complete robot on paper: the requirements from Day 3, the hardware and software layers from Day 2, and the tools and datasheet habits from today. You will size the motors, battery and regulators with real calculations, and produce a BOM with honest prices. The stall test and datasheet check you learned today are how you will verify those choices once you build.
