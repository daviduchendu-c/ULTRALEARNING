# Day 12: Robot Power Systems

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 4 hours
**Level:** intermediate. You need Days 6 to 11, especially Ohm's law, power (P = VI) and the voltage divider. You also need to be comfortable working with a battery and a bench supply, because today includes measurements on live packs.

**By the end of this day, you will have:**

- A power budget for a small robot, in watts and in amp-hours, with every load listed and every efficiency stated
- A 2S lithium-ion battery with a protection board, an inline fuse and a 5 V buck converter, measured under a known load
- A measured brownout threshold for a microcontroller, and a written plan for the regulator and decoupling choices that avoid it
- A protection checklist that you can apply to any robot you build later

**What you need:**

- One 2S (two-cell) lithium-ion pack, nominal 7.4 V, about 2000 mAh, with a built-in protection board that limits overcharge, over-discharge and short circuit. About $15 to $25 for a pack from a reputable hobby supplier. Do not use an unprotected pack.
- A charger made for 2S lithium-ion (8.4 V constant-current, constant-voltage). About $15 to $25. Never charge a lithium pack with a charger you have not checked for the correct cell count and voltage.
- A fireproof charging bag or a metal container for the pack while charging, about $10.
- An inline fuse holder with a 3 A fast-blow fuse, about $3 in a kit.
- A 5 V buck converter module, based on the LM2596 or MP1584 regulator, rated at least 3 A output, about $2 to $4.
- A 5 W 10 Ω wire-wound resistor, about $1, as a known load.
- Arduino Uno Rev3 with Arduino IDE 2.3 or newer, which you used on Day 11.
- Two electrolytic capacitors, 470 µF at 16 V, and two 100 nF ceramic capacitors, about $2 in total.
- A DC bench power supply with adjustable voltage and current limit, about $60 to $150 for a basic model. This is the safe way to test the power system before you connect the battery.
- A digital multimeter that measures DC volts and amps, about $15 to $25. An inexpensive clamp meter, about $25, is useful but optional.
- Breadboard, jumper wires, and the 10 kΩ and 5.6 kΩ resistors from Day 11.

**Prerequisites:**

- Day 2 (batteries and regulators at a high level) and Day 6 (power as P = VI)
- Day 7: series and parallel circuits and voltage division
- Day 11: the voltage divider and the ADC, which you use for the battery monitor
- Comfort with the multimeter's current and voltage modes, including putting the meter in series with a load

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How robot power changed](#how-robot-power-changed)
- [Glossary](#glossary)
- [Batteries and capacity](#batteries-and-capacity)
- [Current requirements and power budgets](#current-requirements-and-power-budgets)
- [Linear and switching regulators](#linear-and-switching-regulators)
- [Rails and grounding](#rails-and-grounding)
- [Decoupling](#decoupling)
- [Protection and brownouts](#protection-and-brownouts)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a power budget and protected supply for a small robot](#project-a-power-budget-and-protected-supply-for-a-small-robot)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

Most robots that fail in the field fail because of their power, not their brains. The motor drivers stall when the battery sags under a hard start, the single-board computer reboots when the drive motors start, and the ultrasonic sensor returns garbage whenever the wheels spin up. The control code is blamed, but the root cause is a battery with too little current capacity, a regulator too close to its limit, or a ground wire that carries motor current through the sensor's return path.

Power is the most physical part of the robot. It has to store energy, deliver current in bursts, turn one voltage into another without wasting it, and survive a short circuit without fire. A robot's power system is the only part of the design that every other part depends on. Getting it right is a set of calculations you can do on paper, followed by measurements that confirm the calculation.

---

## How robot power changed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1800s | Volta's voltaic pile and later lead-acid cells supply the first practical current. | Lead-acid batteries still power heavy industrial robots and forklifts. |
| 1899 to 1950s | Nickel-cadmium and lead-acid cells become rechargeable at scale. | Early mobile robots ran on lead-acid batteries that weighed more than the robot body. |
| 1970s to 1990s | Linear regulators (such as the 78xx family) and early switching regulators reach hobby prices. | A stable 5 V rail becomes cheap enough for a student project. |
| 1991 onward | Lithium-ion cells reach the market and bring much higher energy per kilogram. | Most small robots now run on 18650 cells or lithium-ion packs. |
| 2000s onward | Buck, boost and buck-boost converters in small modules replace most linear regulators in mobile robots. | Efficiency rises, and battery runtime rises with it. |

The pattern is a steady trade: more energy in less mass, with more electronics needed to control how that energy is released. Lithium-ion gives the energy, and protection and conversion circuits make it safe and usable.

---

## Glossary

- **Capacity:** the charge a battery can deliver, usually in amp-hours (Ah) or milliamp-hours (mAh).
- **Energy:** the total work a battery can do, in watt-hours (Wh). It equals nominal voltage times capacity.
- **C-rate:** the current expressed as a multiple of capacity. A 2 Ah cell at 1C delivers 2 A, and at 10C it delivers 20 A.
- **Power budget:** a table of every load's voltage, current and duty cycle, which tells you the total power the battery must supply.
- **Rail:** a wire or plane that carries one fixed voltage to several loads, such as the 5 V rail or the 3.3 V rail.
- **Regulator:** a circuit that holds an output voltage steady while the input changes or the load changes.
- **Linear regulator:** a regulator that drops the extra voltage across a transistor and turns it into heat.
- **Buck converter:** a switching regulator that lowers voltage efficiently by chopping the input and filtering the result.
- **Boost converter:** a switching regulator that raises voltage by storing energy in an inductor and releasing it at a higher voltage.
- **Duty cycle (D):** the fraction of each switching period that the switch is on.
- **Ground (GND):** the reference point that all voltages are measured against. In robots it is also the return path for current.
- **Decoupling capacitor:** a capacitor placed close to a chip's supply pin to supply short current bursts and to filter noise on the rail.
- **Brownout:** a supply voltage that dips below the level a chip needs to run reliably, causing resets or erratic behavior.
- **BMS (battery management system):** the protection board inside a battery pack that limits overcharge, over-discharge and short circuit.

---

### Batteries and capacity

A water tank is a good model for a battery. Its capacity is how much water it holds, and its flow limit is how fast the tap can open. A large tank with a small tap can last a long time at a trickle, but it cannot deliver a fire hose's flow. A battery has both limits: capacity (Ah) and maximum discharge current (C-rate).

Start with the first principles. Energy in watt-hours is nominal voltage times capacity: E = V × Ah. A 2S lithium-ion pack has two cells in series, each nominally 3.6 to 3.7 V, so the pack is 7.4 V nominal. With 2.0 Ah of capacity, it stores E = 7.4 × 2.0 = 14.8 Wh. That number is the best way to compare packs of different voltage.

Runtime comes from capacity divided by average current. At an average 1.5 A draw, the runtime is 2.0 / 1.5 = 1.33 hours, which is about 80 minutes. That is optimistic. Real packs deliver only about 80 percent of their rated capacity before the protection board cuts out, because the BMS trips at a low-voltage threshold rather than the chemical end of charge. Use 1.6 Ah as the usable capacity, and the runtime becomes 1.6 / 1.5 = 1.07 hours, which is about 64 minutes.

The C-rate tells you whether the pack can supply a motor's start current. A 2 Ah pack rated at 20C can deliver 2 × 20 = 40 A continuously, which is far more than most robots need. A cheap pack rated at 1C delivers only 2 A, and a motor stall could drop its voltage below the BMS cutout in a fraction of a second, shutting the robot down mid-move. Read the continuous and burst ratings separately, and check the datasheet of the pack, not the marketing text.

The usual mistake is to size the battery by voltage alone. The 2S pack and a 12 V lead-acid battery have different voltages and very different energy, and a robot designed for 12 V will not run on a 2S pack without a converter. The way to spot it is the power budget below: if the total energy and peak current do not fit the chosen pack, the design is wrong before anything is built.

Pack selection is the first decision in any mobile robot, from hobby rovers that use 2S packs to delivery robots that use 48 V packs with much larger BMS units. The chemistry and the protection logic change with the size, but the arithmetic does not.

---

### Current requirements and power budgets

A household budget lists every bill and its monthly cost, then adds them to see whether the income covers them. A power budget does the same with watts and amps. Each load is listed with its voltage, its current when running, its duty cycle (the fraction of time it runs), and its efficiency if it sits behind a converter.

The first principles are P = V × I for each load, and E = P × t for energy over time. Most loads spend time in more than one state. A motor draws near zero at rest, a few hundred milliamps while cruising, and several amps at stall. The average current is the sum of each state's current weighted by its fraction of time. A budget that uses only the peak current is too pessimistic, and one that uses only the cruising current hides the stall problem, so you need both numbers.

Here is a budget for a small two-wheel robot with a single-board computer, a microcontroller, a sensor and two 6 V gearmotors. The output power of each load is listed, and efficiency is stated where a converter sits between the battery and the load.

| Load | Rail | Voltage | Current when running | Output power |
|---|---|---|---|---|
| Single-board computer | 5 V | 5 V | 1.2 A | 6.0 W |
| Microcontroller and sensors | 3.3 V | 3.3 V | 0.15 A | 0.50 W |
| Two drive motors, cruising | 6 V (direct from battery) | 6 V | 0.4 A each, 0.8 A total | 4.8 W |

The 5 V buck converter is 90 percent efficient, so its input power is 6.0 / 0.90 = 6.67 W. The 3.3 V buck is 85 percent efficient, so its input is 0.50 / 0.85 = 0.59 W. The motors take their power directly from the battery through the motor driver, so their output power is their input power, with the driver losses ignored for this estimate. The total battery power is 6.67 + 0.59 + 4.8 = 12.06 W. The battery current is 12.06 / 7.4 = 1.63 A.

With 1.6 Ah usable, the runtime at this cruising draw is 1.6 / 1.63 = 0.98 hours, which is about 59 minutes. That is the answer a team needs before ordering parts: the robot runs for about an hour, and the design can be changed if that is not enough.

The peak budget matters too. If both motors stall at 2.0 A each on the 6 V rail, the battery must supply about 2 × 2.0 × 6 / 7.4 = 3.2 A in addition to the computer's load, which is about 4.2 A at the battery terminals (3.24 A for the motors plus 6.67 / 7.4 = 0.9 A for the computer's converter input). That must be below the BMS's over-current trip and the fuse's blow curve, or the robot shuts down during a hard turn. The budget therefore has two rows: running and stall.

The usual mistake is to forget the efficiency of the converter. Writing down 6 W for the single-board computer and leaving out the converter's loss makes the budget about 10 percent too optimistic on that rail, and the gap grows with the converter's inefficiency. Always state the efficiency next to each converter and carry it into the battery figure.

A robotics team uses this exact budget to decide whether a battery upgrade is needed before a product reaches a customer. Warehouse robots and cleaning robots are sold with a runtime figure, and that figure comes from a power budget like this one.

---

### Linear and switching regulators

A regulator is a tap on a water tank that delivers a fixed pressure no matter how full the tank is. The question is what happens to the pressure the tap gives up. A linear regulator is a valve that throttles the flow and wastes the pressure as friction. A switching regulator is a pump that moves fluid in packets and recovers almost all the energy.

A linear regulator works by dropping the input-output difference across a transistor. The current is the same on both sides, so the power dropped is (V_in − V_out) × I. The efficiency is roughly V_out / V_in. A 78xx or AMS1117 regulator from 8.4 V to 5 V delivering 1.2 A dissipates (8.4 − 5.0) × 1.2 = 4.08 W as heat. That is two-thirds of the 6 W the computer actually uses, and it needs a large heatsink. A linear regulator is quiet and has very low noise, which makes it right for a sensitive analog rail, but it is wrong for the main supply of a battery-powered robot.

A buck converter works by switching the input on and off at a frequency such as 150 kHz, storing energy in an inductor, and filtering the output. The output voltage is set by the duty cycle: V_out = D × V_in in an ideal converter. From 12 V to 5 V the duty cycle is 5 / 12 = 0.417, or about 42 percent. A real buck is 85 to 95 percent efficient, so for the same 5 V at 1.2 A, the input is 6.0 / 0.90 = 6.67 W and the loss is 0.67 W, about one sixth of the linear regulator's loss at 8.4 V.

A boost converter raises the voltage, and a buck-boost converter does either. The duty cycle for a boost is D = 1 − V_in / V_out. A single 3.7 V lithium-ion cell boosted to 5 V has D = 1 − 3.7 / 5 = 0.26, or 26 percent. Boost converters are more sensitive to load changes than bucks, and their input current is higher than their output current, which matters when you size the fuse.

A switching regulator's drawback is noise. The switching frequency and its harmonics appear on the rail, and they can couple into a sensor's analog line, as you measured on Day 11. A common approach is to use a buck for the main 5 V rail, and to use a linear regulator or an LC filter for the quiet 3.3 V rail that feeds an analog sensor.

The mistake is to choose a regulator by its maximum output current alone. The regulator must also stay within its input range at the lowest battery voltage and its thermal limit at the highest current. For the 2S pack, a buck with a 40 V input is fine, but a 3.3 V linear regulator fed from 8.4 V dissipates 5.1 × 0.15 = 0.77 W at a modest load, which is already a warm part. Check the input voltage range and the thermal derating on the datasheet before you order.

Buck regulators are in every drone flight controller and every warehouse robot. The choice of topology is one of the first decisions an electronics engineer makes on a new robot.

---

### Rails and grounding

A river has one main channel that splits into tributaries. A power rail is the main channel: one wire or copper plane at one voltage, feeding several loads. The ground is the river's return path, the channel that carries the water back to the source. If the return path is narrow or shared with a sensitive channel, the water backs up and the sensitive channel floods.

The first principles are simple. Current flows in a loop: from the source through the load and back to the source. The loop's resistance times the current gives a voltage drop along the wire. A 22 AWG copper wire has a resistance of about 0.053 Ω per metre. A half-metre run to a motor driver and back is a 1-metre loop, so 0.053 Ω. At 2 A the drop is 2 × 0.053 = 0.106 V. That is small, but it is not zero, and it appears on the return path as a voltage between the robot's ground points. If the sensor's ground is at the end of a long thin wire that carries motor current, the sensor sees a shifted ground, and its readings shift with the motor.

The standard fix is a star ground, where every subsystem returns to one common point, and the high-current returns (motors, battery, drivers) do not share wire with the low-current returns (sensors, microcontroller). In a robot the star point is usually the battery's negative terminal or the power input terminal. Keep the motor return short and thick, and run the sensor return separately to the star point.

A rail is named after its voltage, such as 5 V, 3.3 V and VBAT. The name tells you what should be on that wire. When you draw a schematic, a rail is labelled once and each load is connected to it. Day 14 covers how rails appear in a schematic.

The common mistake is to run the sensor ground through the motor's return path. The symptom is a sensor reading that changes when the motor starts, even though the sensor is not touched. Check it by measuring the voltage between the sensor's ground pin and the battery's negative terminal while the motor runs. If that voltage moves by tens of millivolts, the ground path is shared.

In practice, this is the rule behind every robot control board: separate the power ground from the signal ground, join them at one point, and keep the high-current wires short. Motor controller manufacturers print this layout in their application notes, and industrial drives use the same rule.

---

### Decoupling

A bathtub with a drain that fills slowly still delivers a sudden spray if you open a large tap next to it, because the water already in the pipe is close. A decoupling capacitor is that nearby water. A chip draws current in short bursts, and the wire from the regulator to the chip has inductance and resistance, so the voltage dips at the chip faster than the regulator can respond. A capacitor placed next to the chip's supply pin supplies the burst and holds the rail steady.

The starting point is the relation I = C × dV/dt. A capacitor supplies current when its voltage falls, and it holds the rail up for the short time the regulator needs to catch up. A 100 nF ceramic capacitor has a low equivalent series resistance and responds in nanoseconds, so it handles high-frequency bursts. A 470 µF electrolytic capacitor holds more charge and handles slower, larger demands, such as a motor's stall current, but it responds more slowly and has more series resistance.

Here is a worked example. A microcontroller pin switches a load that draws a 50 mA burst in 1 µs. The rail voltage must not drop by more than 50 mV. The capacitor must supply the burst's charge, Q = I × t = 0.05 × 0.000001 = 5 × 10^-8 C. The voltage drop across the capacitor is ΔV = Q / C. For ΔV = 0.05 V, C = 5 × 10^-8 / 0.05 = 1 × 10^-6 F, which is 1 µF. A 100 nF ceramic would drop 0.5 V on the same burst, which is why you place several in parallel and add the bulk capacitor. Real designs add margin, which is why most datasheets recommend 100 nF per supply pin plus a bulk capacitor on the rail.

The catch is that a capacitor's value is not its effective value. Ceramic capacitors lose much of their capacitance at their rated voltage, and electrolytic capacitors lose capacitance and gain resistance as they age or get warm. A 470 µF electrolytic may be 300 µF at 60 degrees Celsius and two years old. Check the datasheet for the derating, and use a voltage rating of at least twice the rail voltage.

The mistake is to place the decoupling capacitor far from the chip. A capacitor on the far side of a 20 cm jumper wire is almost useless, because the wire's inductance blocks the fast current. The way to spot it is the layout: the capacitor's leads should be as short as possible and the path to the chip's supply and ground pins should be direct.

On the job, decoupling is the first thing a board reviewer checks. A board with no capacitor near a motor driver or a radio module will reset under load, and the fix is a few cents of capacitors placed correctly.

---

### Protection and brownouts

A car's fuse box has a fuse for each circuit, and a fuse blows before the wiring overheats. A robot's power system needs the same layers: a fuse, a battery protection board, a regulator with current limit, and a supply that does not dip below what the chips need. Without them, a short circuit becomes a fire, and a sag becomes a reset.

Fuses are sized by the maximum continuous current and the surge the circuit draws. A common rule is to choose a fuse rated at 125 to 150 percent of the maximum continuous current. A robot with a 2 A maximum load needs a fuse rated at about 2.5 to 3 A. The fuse must also tolerate the motor's start surge, which is why a fast-blow fuse is used for electronics and a slow-blow fuse for motors. The fuse is placed on the battery's positive lead, close to the battery, so a short anywhere on the rail is protected.

The battery protection board (BMS) does three jobs. It stops charging when a cell reaches its upper voltage, typically 4.2 V per cell. It stops discharge when a cell reaches its lower voltage, which is usually set between 2.5 V and 3.0 V per cell by the board's design, so check the board's datasheet. It also cuts the output in a short circuit. A 2S board therefore cuts discharge at about 5.0 to 6.0 V across the pack, and the Day 2 figure of about 6 V nearly empty is the same threshold seen from the other side.

A brownout is a supply that dips below the voltage a chip needs. Each chip has a minimum operating voltage. When the rail sags below it, the chip can reset, lock up or behave unpredictably. Brownouts happen when a motor starts and draws a current spike that pulls the battery voltage down, which drops the regulator's input and then its output. The Raspberry Pi displays a warning icon when its 5 V supply falls below the level it needs, which is a built-in brownout indicator. The Arduino Uno can be configured to reset at a set threshold through the ATmega328P's brown-out detector, documented in the Microchip datasheet.

The design problem fits in one sentence: when a 6 V motor stalls at 2 A and the battery sags by 0.5 V, does the 5 V buck converter still have enough input headroom to hold 5 V? A buck with a 7 V input minimum at 2 A would brown out from a 2S pack at 6.5 V, which is the sag of a motor start. You fix that with a larger battery, a larger bulk capacitor on the 5 V rail, or a separate regulator for the logic that is isolated from the motor spike.

Protection trips after a delay, and the fuse and BMS are slow compared with a chip's reset. A brief short may damage a MOSFET before the fuse opens. Protection reduces the risk; it does not remove the need for correct wiring and component ratings.

The mistake is to treat a reset as a software bug. When the robot resets during a hard turn, check the battery voltage on a multimeter during the turn, or log it with the divider from Day 11. If the voltage dips below the regulator's minimum, the cause is power, and no software change will fix it.

Every serious mobile robot has a BMS, a fused battery input and a power-good indicator. Product safety standards for battery-powered equipment are written around the same failure modes, so a reviewer will ask for the fuse, the BMS and the cutoff figures in the datasheet.

---

## Hands-on exercises

Work with the battery only after you have read all the safety notes in this section. Keep the pack in a fireproof bag while you test, never leave it charging unattended, and never short the battery terminals or connect it to anything without the fuse in place.

1. **Measure the pack at rest.** With nothing connected to the pack, measure its open-circuit voltage with the multimeter and record it. A healthy 2S pack should be between about 6.5 V and 8.4 V. Then set the bench supply to 5 V with a current limit of 0.5 A, turn its output on with nothing connected, and confirm the supply reads 5 V with zero current. Record both readings. The battery is not connected to the bench supply in this step.

2. **Measure the 10 Ω load on the bench supply.** Set the bench supply to 5 V with a current limit of 1 A. Connect the 10 Ω, 5 W resistor across the supply. The current should be about 0.5 A, and the power in the resistor should be P = V² / R = 25 / 10 = 2.5 W. Confirm the resistor does not get hot to the touch in the first minute.

3. **Test the buck converter.** Connect the LM2596 or MP1584 buck module to the bench supply at 8.4 V with a 1 A current limit. Adjust its output potentiometer to 5.0 V with no load. Then connect the 10 Ω resistor as the load and measure the output voltage and the input current. Record the input power and the output power. Compute the efficiency as output power divided by input power. A buck at this load should be between 80 and 90 percent.

4. **Add the fuse and decoupling.** Place the 3 A fast-blow fuse on the positive lead before the buck converter. Add a 470 µF electrolytic capacitor and a 100 nF ceramic capacitor across the buck output, with the electrolytic's negative lead to GND and the positive lead to 5 V. Repeat step 3 and check that the output ripple, measured with the multimeter's AC volts mode, is smaller with the capacitors than without.

5. **Find the Uno's brownout threshold.** With the Uno powered from the bench supply through the buck output, slowly lower the supply voltage from 7.4 V toward 6 V. Watch for the Uno's power LED or for the sketch's serial output to stop. A simple sketch that prints a counter every 100 ms helps you see a reset, because the counter restarts. Record the voltage at which the reset occurs. Your threshold will depend on the board and its regulator, and the number is the input you need for the design.

6. **Record the power budget row.** Using the power budget formulas from this lesson, compute the battery draw for a running state and a stall state of your own robot's loads, and record them in the project's `budget.csv`.

Stop immediately if the battery pack becomes hot, swells or smells. Disconnect it, move it to a non-flammable surface outdoors, and dispose of it through a battery recycling point.

---

## Project: a power budget and protected supply for a small robot

This project turns the lesson into a documented power system. It is a practical project, so the reader builds a real measurement setup and a budget with every number traced to a measurement or a datasheet.

**Goal:** design and document a protected 2S battery supply for a small wheeled robot, with a measured power budget, a measured regulator efficiency, a fuse rating, and a voltage monitor on an Arduino Uno. The output is a repo that someone can use to size the battery for their own robot.

**Repo layout:**

```
100-days-robotics-engineering-day-12-robot-power-systems/
├── README.md
├── power/
│   ├── budget.csv
│   └── power_budget.py
├── measurements/
│   ├── buck_efficiency.csv
│   └── brownout_test.md
├── firmware/
│   └── battery_monitor/
│       └── battery_monitor.ino
├── docs/
│   └── protection_checklist.md
└── images/
    └── wiring_photo.jpg
```

**Starter code:** `firmware/battery_monitor/battery_monitor.ino`

```cpp
const int BATT_PIN = A0;
const float VREF = 5.0;            // Measure the real 5 V rail and update this value
const float R_TOP = 10000.0;       // battery positive to A0
const float R_BOTTOM = 5600.0;     // A0 to GND
const float SCALE = (R_TOP + R_BOTTOM) / R_BOTTOM;

void setup() {
  Serial.begin(115200);
  Serial.println("millis,battery_volts");
}

void loop() {
  long sum = 0;
  for (int i = 0; i < 16; i++) {
    sum += analogRead(BATT_PIN);
  }
  float pinVolts = (sum / 16.0) * VREF / 1024.0;
  float batteryVolts = pinVolts * SCALE;

  Serial.print(millis());
  Serial.print(",");
  Serial.println(batteryVolts, 3);
  delay(500);
}
```

Check the scale factor by hand before you trust it. At 8.4 V, the divider gives 8.4 × 5.6 / 15.6 = 3.015 V on A0, which is below the Uno's 5 V limit. The scale factor is 15.6 / 5.6 = 2.786, and 3.015 × 2.786 = 8.40 V.

**Starter power script:** `power/power_budget.py`

```python
BATT_V_NOMINAL = 7.4
USABLE_FRACTION = 0.8

# (name, output power in W, efficiency of the converter feeding it)
loads = [
    ("single-board computer, 5 V", 6.0, 0.90),
    ("microcontroller and sensors, 3.3 V", 0.50, 0.85),
    ("drive motors, cruising, direct", 4.8, 1.00),
]

def battery_power(loads):
    return sum(p / eff for _, p, eff in loads)

def runtime_hours(capacity_ah, battery_power_w):
    battery_current = battery_power_w / BATT_V_NOMINAL
    return capacity_ah * USABLE_FRACTION / battery_current

total = battery_power(loads)
print(f"battery power: {total:.2f} W")
print(f"runtime on 2.0 Ah pack: {runtime_hours(2.0, total) * 60:.0f} minutes")
```

The script returns about 12.06 W and about 59 minutes, which matches the hand calculation in the lesson. Your own budget should reproduce this check.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the parts list, the battery specification and the safety notes.
2. List every load on your robot in `power/budget.csv`, with its rail, voltage, running current, stall current, duty cycle and converter efficiency. Use the measured current where you can, and the datasheet where you cannot, and mark which is which.
3. Run `power_budget.py` with your numbers. Record the running battery power, the stall battery current and the runtime.
4. Build the buck test from the exercises. Record the input and output voltage and current in `measurements/buck_efficiency.csv`, and compute the efficiency at three load points.
5. Run the brownout test with the Uno and write the threshold you measured in `measurements/brownout_test.md`.
6. Flash the battery monitor sketch and check the reading against the multimeter. Record any difference.
7. Complete `docs/protection_checklist.md` with the fuse rating, the BMS cutoff voltages from the board's datasheet, the decoupling placement and the star-ground layout for your robot.

**README checklist:**

- [ ] Goal and scope
- [ ] Battery specification, with voltage, capacity, continuous and burst C-rating
- [ ] Power budget table and runtime
- [ ] Fuse rating and the reason for it
- [ ] Measured regulator efficiency at three loads
- [ ] Measured brownout threshold
- [ ] Protection checklist with the star-ground layout described

**Done when:**

- The power budget reproduces the runtime in the lesson's worked example when given the same inputs.
- The measured buck efficiency is within 5 percentage points of the datasheet figure at the load you tested.
- The fuse rating is at least 125 percent of the continuous load in the budget.
- Every number in the README traces to a measurement, a datasheet, or a stated assumption.

**Evidence to commit to GitHub:** the budget CSV, the script, the measurement files, the firmware, the checklist and the README with the tables. The reviewer should be able to change one load in the CSV and see the runtime change.

This project is light on fabrication, but the electrical risk is real. The battery must be protected and the test must run from a current-limited supply, as the exercises require. Do not build the project with an unprotected pack, and do not connect a lithium battery directly to the Uno, the buck module or any motor without the fuse.

---

## Check your understanding

1. A 2S pack is 7.4 V nominal, 2.5 Ah and 20C. A robot draws 1.8 A on average. Compute the energy in watt-hours, the usable runtime with 80 percent usable capacity, and the maximum continuous current the pack can supply.
2. A microcontroller board draws 0.3 A at 3.3 V from an 8.4 V linear regulator. Compute the heat dissipated in the regulator, and compare it with a buck converter at 90 percent efficiency for the same load.
3. A 3.7 V cell feeds a 5 V boost converter. Compute the duty cycle, and explain why the input current is higher than the output current at the same power.
4. Your Arduino resets only when the drive motors start. Describe two measurements that would tell you whether the cause is brownout or a software bug, and state which result would point to each.
5. A sensor's readings shift by 40 mV whenever the motors run, and the sensor's wires are short. Explain the most likely cause, and describe the wiring change that would fix it.
6. A motor driver must handle a 2.2 A stall. Choose a fuse rating, explain why a fast-blow fuse may not be the right type for the motor branch, and say where the fuse must sit.

---

### Answers

1. Energy is 7.4 × 2.5 = 18.5 Wh. Usable capacity is 2.5 × 0.8 = 2.0 Ah, so runtime is 2.0 / 1.8 = 1.11 hours, about 67 minutes. The maximum continuous current is 2.5 × 20 = 50 A at 20C, which is far above the 1.8 A average. Check the burst rating separately from the continuous rating.
2. The linear regulator dissipates (8.4 − 3.3) × 0.3 = 1.53 W. A buck at 90 percent efficiency has an output of 3.3 × 0.3 = 0.99 W, so its input is 0.99 / 0.90 = 1.10 W and its loss is 0.11 W. The buck loses about one fourteenth of the linear regulator's heat.
3. D = 1 − 3.7 / 5 = 0.26. The input current is higher because the input voltage is lower for the same power: P = V × I, so the input current is P / V_in, which is larger than P / V_out.
4. Measure the battery or 5 V rail with a multimeter or the Day 11 divider on the same logger while the motors start. If the rail drops below the regulator or the chip's minimum during the reset, the cause is brownout. If the rail holds steady and the reset still happens, look at the code, the watchdog, or the stack.
5. The most likely cause is shared ground: the motor return current flows through the same wire as the sensor's return, so the sensor sees a shifted ground. The fix is a star ground, with the motor return running separately to the battery's negative terminal and the sensor return running separately to the same point.
6. Rate the fuse at 125 to 150 percent of the continuous current. For a branch with a 2.2 A stall that is only brief, a 3 A slow-blow fuse may be correct, because a fast-blow fuse may open during the start surge. The fuse must sit on the battery's positive lead close to the battery, so the whole supply is protected.

---

## Sources to read and watch

*Read*

- [Wikipedia: Electric battery](https://en.wikipedia.org/wiki/Electric_battery) for the basics of cells, capacity and chemistry.
- [Wikipedia: Lithium-ion battery](https://en.wikipedia.org/wiki/Lithium-ion_battery) for the chemistry, the voltage and the safety risks.
- [Wikipedia: Linear regulator](https://en.wikipedia.org/wiki/Linear_regulator) for the dropout and dissipation model.
- [Wikipedia: Switched-mode power supply](https://en.wikipedia.org/wiki/Switched-mode_power_supply) for the principle behind buck and boost converters.
- [Wikipedia: Buck converter](https://en.wikipedia.org/wiki/Buck_converter) and [Wikipedia: Boost converter](https://en.wikipedia.org/wiki/Boost_converter) for the duty-cycle relations.
- [Wikipedia: Decoupling capacitor](https://en.wikipedia.org/wiki/Decoupling_capacitor) for placement and its purpose.
- [Wikipedia: Fuse (electrical)](https://en.wikipedia.org/wiki/Fuse_(electrical)) for fuse types and time-current behavior.
- The Raspberry Pi documentation on power supplies, on raspberrypi.com. Search "Raspberry Pi power supply requirements".
- The ATmega328P datasheet from Microchip, for the brown-out detector settings.

*Tutorials*

- SparkFun Learn, the tutorials on battery basics and on LiPo and lithium-ion safety. Search "SparkFun Learn LiPo battery".
- Adafruit Learning System, the guides on power regulation and on the buck and boost modules. Search "Adafruit power regulation guide".
- Texas Instruments, the application notes on buck converter design. Search "Texas Instruments buck converter application note" on ti.com.

*Watch*

- MIT OpenCourseWare, 6.002 Circuits and Electronics. The lectures on the RC circuit and the power electronics unit give the theory behind decoupling.
- Ben Eater's YouTube channel. Search his videos on power supplies and on building a regulator, which cover the same measurement habits used today.

*Practice*

- A bench power supply, used in the exercises, is the best practice tool. If you do not have one, Wokwi supports the Uno and the voltage monitor sketch, but it does not model the battery, so do the battery tests on the bench.

---

## Tomorrow

Today you sized the supply, protected it and measured how it behaves under load. The next step is the part of the robot that draws most of that power: the motors. Day 13 covers how a DC motor behaves electrically, why its back-EMF limits its speed, how an H-bridge reverses it, and how PWM controls its average voltage. The stall currents you budgeted today are the numbers the motor driver and its flyback diode must survive, so keep the budget file open while you work.

**Next:** [Day 13: Motor Electronics](Robotics-Day-13-Motor-Electronics.md)
