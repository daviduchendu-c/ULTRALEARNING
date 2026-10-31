# Day 13: Motor Electronics

*100 Days of Robotics Engineering, Module 2: Electronics for Robotics*

**Time:** about 4 hours
**Level:** intermediate. You need Days 6 to 12, especially Ohm's law, power, the switch-like behavior of a MOSFET from Day 9 and the power budget from Day 12. Today you drive a real motor, so the safety rules in this lesson apply to every exercise.

**By the end of this day, you will have:**

- A measured resistance and a no-load current for a small DC gearmotor, with a back-EMF estimate that matches the measurement
- A working H-bridge test on an Arduino Uno, using a TB6612FNG motor driver breakout, with forward, reverse, brake and PWM speed control
- A switching circuit with a logic-level MOSFET and a flyback diode, and a calculation that shows why the diode is required
- A written current-limit and fusing plan for the motor branch, using the stall current you measured

**What you need:**

- Arduino Uno Rev3 with Arduino IDE 2.3 or newer, about $25 for an official board or $10 to $15 for a clone
- A TB6612FNG motor driver breakout board, from SparkFun or Pololu, about $5 to $10. The chip is made by Toshiba and its datasheet gives the current and voltage ratings used in this lesson.
- One small 6 V brushed DC gearmotor with a stall current below 1.0 A, such as the common TT-style yellow gearmotor, about $2 to $4 each. Check the listed stall current before you buy.
- One logic-level N-channel MOSFET, IRLZ44N (55 V, 47 A in the datasheet), about $1
- One 1N5819 Schottky diode (40 V, 1 A), about $0.20, or a 1N4007 general-purpose diode as an alternative
- Resistors: one 100 Ω, one 10 kΩ, about $0.10 each
- One 6 V regulated bench power supply with a current limit that you can set between 0.1 A and 1.5 A, about $60 to $150. A battery pack with an inline fuse is the alternative for the mobile exercises, covered in Day 12.
- An inline fuse holder with a 2 A fast-blow fuse, about $3
- Digital multimeter with DC volts and DC amps modes, and an ohms mode, about $15 to $25
- Breadboard and jumper wires, about $5
- Optional: an oscilloscope with two channels, for the switching waveform. A basic USB scope kit costs about $40 to $80.

**Prerequisites:**

- Day 9: the N-channel MOSFET as a switch, the gate threshold and the pull-down resistor
- Day 11: PWM as a way to set an average voltage, and the ADC for reading a signal
- Day 12: the power budget, the fuse rating and the star ground
- Comfort with setting a bench supply's current limit before connecting a load

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How motors and their drivers developed](#how-motors-and-their-drivers-developed)
- [Glossary](#glossary)
- [DC motor electrical behavior](#dc-motor-electrical-behavior)
- [Back-EMF](#back-emf)
- [H-bridges](#h-bridges)
- [Motor drivers](#motor-drivers)
- [MOSFET switching and flyback diodes](#mosfet-switching-and-flyback-diodes)
- [PWM](#pwm)
- [Current limits](#current-limits)
- [Electrical noise](#electrical-noise)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a motor driver characterization rig](#project-a-motor-driver-characterization-rig)
- [Check your understanding](#check-your-understanding)
- [Sources to read and watch](#sources-to-read-and-watch)
- [Tomorrow](#tomorrow)

---

## Introduction

A motor is the only part of a robot that turns electrical energy into motion, and it is also the part that most often damages the electronics around it. The coil in a motor is an inductor, the brushes are a switch that sparks, and the stall current can be ten times the running current. A robot team that treats the motor as a simple resistor will meet the failures in the field: a driver chip that overheats, a microcontroller that resets when the wheels start, and a MOSFET that dies on the first reversal.

Today's skill is the one that stops those failures. You will model a DC motor as a resistor and an inductor with a voltage source that opposes the current. You will drive it through an H-bridge so it can run forward and backward, you will control its speed with PWM, and you will protect the switching circuit against the voltage spike the motor produces when it is switched off. Every robot that moves uses these circuits, so the calculations in this lesson transfer to a wheel, an arm joint or a pump.

---

## How motors and their drivers developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1821 | Michael Faraday demonstrates electromagnetic rotation, the principle behind every electric motor. | The motor's physics has not changed in two centuries. |
| 1830s to 1880s | The first practical DC motors appear, built with electromagnets and mechanical brushes. | Brushed DC motors are still the cheapest motor for a small robot. |
| Late 1800s to 1950s | Motor control moves to rheostats and contactors, which are big, slow and wasteful. | Early robot drives were heavy, and efficiency was poor. |
| 1960s to 1980s | Electronic commutation and transistor drivers replace contactors in many applications. | Small H-bridge chips become cheap enough for hobby robots. |
| 1980s onward | Power MOSFETs and integrated motor drivers make high-current switching compact. | Modern drivers such as the TB6612FNG fit on a board the size of a postage stamp. |
| Today | Brushless DC motors with electronic commutation dominate high-performance robots and drones. | Brushed motors remain common in education and low-cost mobile robots. |

Brushed motors are simple but noisy. Brushless motors are efficient and quiet, but they need a more complex controller. This lesson works with brushed DC motors because the electrical physics is the same and the hardware is cheap.

---

## Glossary

- **Armature:** the rotating coil of a DC motor, which carries current through the brushes.
- **Brushes:** sliding contacts that connect the armature to the supply through the commutator.
- **Stall current:** the current a motor draws when its shaft is held still, equal to the supply voltage divided by the winding resistance.
- **Back-EMF (counter-electromotive force):** the voltage a spinning motor generates that opposes the supply voltage.
- **H-bridge:** four switches arranged in an H shape around a motor, which let current flow in either direction.
- **Motor driver:** an integrated circuit or board that contains an H-bridge, its gate drivers and protection.
- **Flyback diode:** a diode connected across an inductive load in reverse, which gives the stored energy a safe path when the current is switched off.
- **Freewheeling diode:** another name for a flyback diode.
- **PWM (pulse-width modulation):** switching a signal on and off quickly, so its average voltage follows the duty cycle.
- **Duty cycle:** the fraction of each PWM period that the signal is high.
- **Shunt resistor:** a low-value resistor placed in series with a load, which gives a voltage proportional to the current through it.
- **Commutation:** switching the current through the motor's coils so that torque stays in one direction.
- **Ripple:** a small periodic variation in a supply or a current, here caused by PWM switching.

---

### DC motor electrical behavior

A brushed DC motor is a coil of wire inside a magnet, with brushes that reverse the current direction as the coil turns. Electrically, it is an inductor in series with a resistor, and the rotation adds a voltage that acts like a battery in the opposite direction. The coil's resistance R and inductance L are the two numbers a datasheet gives, though many hobby motors list only the no-load speed and current, and the stall current.

The analogy is a rowing boat with a current pushing against it. The oars push the boat forward, and the current opposes the boat's motion. The faster the boat moves, the less the net push is. A motor works the same way: the supply pushes current through the winding, and the spinning winding pushes back.

The first principles are Kirchhoff's voltage law applied around the motor loop, with the back-EMF E as the opposing voltage:

V = I × R + E

When the motor is held still, E = 0, so the current is I = V / R. This is the stall current, and it is the largest current the motor draws from a fixed supply. When the motor spins freely with no load, the current is small and E is close to V. Torque is proportional to current, so the stall condition is also the maximum torque condition.

Here is a worked example for a 6 V gearmotor with winding resistance R = 6 Ω. The stall current is 6 / 6 = 1.0 A. At stall, the motor's torque is at its peak but its speed is zero, and the power dissipated in the winding is I²R = 1.0² × 6 = 6 W, which heats the motor quickly. A motor held at stall for more than a few seconds can overheat and damage its insulation. That is why stall current is a rating for brief peaks, not a steady condition.

The common mistake is to size the driver from the no-load current alone. A driver rated for 0.5 A at a 0.1 A no-load draw looks adequate on paper, but it fails the first time the wheel is stuck against a wall. Use the stall current for the driver and the fuse, and the running current for the power budget.

On the job, every motor datasheet from a robotics supplier lists stall current, no-load current and resistance, and the driver is chosen against the stall figure.

---

### Back-EMF

Imagine pushing a swing. The faster the swing moves, the harder it is to push it faster. A spinning motor behaves the same way: the faster it turns, the more voltage it generates opposite to the supply, and that voltage cuts the current. Back-EMF is the reason a motor draws little current when it is free and a lot when it is loaded.

The relation comes from Faraday's law. A coil moving through a magnetic field has a voltage across it proportional to its speed. For a DC motor, E = K × ω, where K is the motor constant in volts per radian per second and ω is the shaft speed. Torque is T = K × I in the same units, so a motor's speed-torque behavior is determined by its voltage and resistance.

Here is the worked example with the same motor. Its no-load current is 0.1 A at 6 V. The voltage across the winding is I × R = 0.1 × 6 = 0.6 V, so the back-EMF is E = 6 − 0.6 = 5.4 V. The motor spins at the speed that gives 5.4 V of back-EMF. Now load the shaft lightly, so the speed falls and the back-EMF falls to 4.5 V. The current becomes (6 − 4.5) / 6 = 0.25 A. Loading a motor raises its current because the back-EMF falls, and the current goes up with the load.

Back-EMF is only an estimate of speed, not a speed sensor. It depends on the motor's constant, which changes with temperature and brush wear, and it is hard to measure cleanly with PWM running. Robots that need accurate speed use encoders, which Day 29 introduces in a simple form and later modules cover in depth.

The mistake is to read a low current as a healthy motor. A motor with a worn brush, a damaged bearing or a jammed gear can draw less current than expected because the back-EMF is higher or the winding is open. The way to spot it is to measure the no-load current and the speed together. If the speed is low and the current is also low, check the winding resistance with the ohms mode of the multimeter, with the motor disconnected from the driver.

Back-EMF is how sensorless motor controllers in fans, pumps and robot joints estimate speed. The motor generates its own speed signal while it runs, and the controller reads it between PWM pulses.

---

### H-bridges

A door that opens in both directions has a hinge on each side. An H-bridge is the electrical version. Four switches sit in an H, with the motor across the middle bar. Close the top left and bottom right, and current flows through the motor one way. Close the top right and bottom left, and it flows the other way. Open all four, and the motor coasts. Close the two switches on one side, and the motor brakes by shorting its own terminals.

The logic is simple. Each switch is a MOSFET or a bipolar transistor, and the motor sees the voltage difference between its two terminals. If the left terminal is at the supply and the right terminal is at ground, current flows left to right. If the polarity is swapped, the motor runs backward. Never close both switches on the same side of the supply, because that connects the supply directly to ground through two transistors, which is a short circuit.

A typical H-bridge truth table has four rows. Input A high and input B low drives the motor forward. Input A low and input B high drives it in reverse. Both low makes the motor coast. Both high brakes it. A driver chip implements this table with logic inputs and enforces the rule that the two transistors on one side never conduct together, using a dead time between switching edges.

The worked example here is the safety requirement. A 6 V motor supply with a 1 A stall current and a short-circuit condition across a shoot-through path: if the driver's dead time is zero and both high-side and low-side switches on one leg conduct, the current is limited only by the supply's internal resistance and the transistor's on-resistance. For a 0.1 Ω on-resistance per switch, the path resistance is 0.2 Ω and the shoot-through current is 6 / 0.2 = 30 A. That is far above any fuse on a small robot, which is why drivers have dead-time protection and why you never build an H-bridge from discrete transistors without it.

An H-bridge has no speed control by itself. Speed comes from PWM on the switches, and the bridge's losses rise with switching frequency. The mistake to watch for is an inverted logic table, where the forward input is driven by a pin that the code assumes is reverse. The symptom is a robot that turns the wrong way, and the test is to drive each input alone and watch the motor direction.

H-bridges are inside every brushed motor driver on a robot, from the cheap TB6612FNG breakout to the large drives in forklifts.

---

### Motor drivers

A motor driver is an H-bridge with the parts a real motor needs. It has gate drivers that turn the power switches on fast and cleanly, protection against over-temperature and over-current, and input logic that the microcontroller can drive directly. A microcontroller pin cannot supply the current to switch a motor, so the driver is the amplifier between the control logic and the power.

The TB6612FNG from Toshiba is a dual-channel driver with two H-bridges in one package. Its logic supply, VCC, runs at about 2.7 V to 5.5 V, and its motor supply, VM, has a maximum rating in the datasheet that you must read before you pick a battery. A 2S pack at 8.4 V fully charged is the case to check against it. Each channel has two direction inputs (AIN1 and AIN2 for channel A) and a PWM input (PWMA). A standby pin, STBY, must be high for the driver to run. Check the datasheet for the current rating: the TB6612FNG is rated for about 1.2 A average per channel, with a higher peak for short pulses, so it is suitable for the 1.0 A stall motor in this lesson but not for a 3 A motor.

The worked example is the sizing check. A motor with a stall current of 1.0 A and a running current of 0.25 A needs a driver whose average rating exceeds the average current, and whose peak rating exceeds the stall current for the duration of a stall. The TB6612FNG passes both checks. A motor with 2.5 A stall would fail the average check under sustained stall, and a larger driver is needed.

The common mistake is to use the logic supply for the motor. The motor supply, VM, is the battery or bench supply, and the logic supply, VCC, is the microcontroller's 5 V. If the motor is connected to VCC instead, the logic regulator carries the motor current and browns out. Check the labels before you connect the power.

A breakout board's limit is its thermal design. It has no heatsink, and a driver that runs at its rated current for a long time gets hot. Keep the average current under about 60 percent of the rating, and watch the chip's temperature during a long test.

On the job, drivers are chosen by the average current, the peak current, the voltage range and the thermal path. A team that ships a robot with an undersized driver will see returns for overheating.

---

### MOSFET switching and flyback diodes

A light switch on a circuit with a large inductor creates a spark, because the inductor's current keeps flowing and pushes its voltage up until the air gap breaks down. A motor is an inductor, so switching it off with a transistor creates the same problem in electrical form. The flyback diode is the safety valve.

The starting point is the inductor's voltage relation, V = L × di/dt. When the switch opens, the current through the motor's inductance cannot change instantly, so di/dt becomes very large and the voltage across the inductor rises until it finds another path. Without a path, the voltage spikes until the MOSFET breaks down or the insulation fails. A flyback diode, placed across the motor in reverse, gives the current a path that clamps the voltage to about the supply voltage plus the diode's forward drop.

The worked example shows the scale. A motor's effective inductance is about 1 mH. It carries 1 A and the switch opens in 1 microsecond. The voltage spike is V = 0.001 × 1 / 0.000001 = 1000 V. That is well above the 55 V rating of the IRLZ44N. With a 1N5819 diode across the motor, the spike is clamped to about 6 V plus the 0.3 V forward drop of the diode, so the MOSFET sees only about 6.3 V above the supply. The calculation is the reason the diode is not optional.

The design of the switching stage has three parts. The gate of the MOSFET is pulled to ground through a 10 kΩ resistor, so the transistor stays off when the microcontroller pin floats, which Day 9 covered. A 100 Ω resistor in series with the gate limits the current into the microcontroller pin when the gate switches, and it slows the edges to reduce noise. The flyback diode is placed across the motor with its cathode to the supply and its anode to the MOSFET drain.

The catch is that the diode is slow to recover on a fast-recovery diode and not fast enough on a general-purpose diode, and the Schottky diode recovers quickly but has a limited reverse voltage. Check the diode's current and reverse voltage rating against the motor, and check that the MOSFET's drain-source voltage rating is well above the supply plus the spike.

The mistake to watch for is a missing or reversed diode. A reversed diode shorts the supply through the motor and the diode, which can blow the fuse or destroy the diode. The check is the band marking: the cathode of the diode is the end with the stripe, and it goes to the positive supply.

Every relay driver and every solenoid driver in a robot uses a flyback diode. Motor drivers include internal protection diodes, but discrete designs must add them, and the datasheet tells you whether the driver includes them.

---

### PWM

A tap that you open for a fraction of each second delivers an average flow equal to that fraction times the full flow. PWM does the same with voltage. The microcontroller switches the output fully on and off at a fixed frequency, and the motor responds to the average voltage because its inductance and inertia smooth out the switching.

The average voltage is V_avg = D × V_supply, where D is the duty cycle. A 6 V motor at 75 percent duty sees an average of 4.5 V. The switching frequency must be high enough that the motor's current ripple stays small, and low enough that the driver's switching losses stay small. Motor drivers commonly switch between about 10 kHz and 25 kHz, which is above the audible range and avoids the whine of lower frequencies.

The Arduino Uno's default PWM frequency is about 490 Hz on pins 3, 9, 10 and 11, and about 980 Hz on pins 5 and 6. Both are low enough to hear as a faint buzz from the motor. The analogWrite function selects the duty cycle from 0 to 255, so a value of 191 gives a duty of 191 / 255 = 75 percent. For a quieter motor, the timer frequency can be changed in the timer registers, but that is an advanced step and it affects other pins. Check the timer's effect on your other code before you change it.

PWM's limit is torque at low duty. A motor at 10 percent duty receives a small average voltage and may not overcome its static friction, so it stalls and draws a large current in pulses, which heats the motor. Minimum speed is a property of the motor and the load, and you find it by testing, not by calculation.

The common mistake is to think a PWM value sets a fixed speed. The speed depends on the load as well as the duty cycle. A robot that drives straight at 75 percent duty on a flat floor will slow on a carpet, because the back-EMF falls and the current rises. Closing the loop with an encoder is the fix, and that comes later in the series.

PWM is the control method for nearly every brushed motor, LED dimmer and heater. The robot's motor controller and the lighting controller in a building use the same principle.

---

### Current limits

A cap on a water tap limits the flow, and a current limit does the same for electricity. The limit protects the motor, the driver and the wiring from a stall that would otherwise draw enough current to burn them. A robot needs two kinds of current limit: one in the power supply and one in the driver or the wiring.

The starting point is the power the current dissipates. In a conductor or a MOSFET, P = I² × R, so doubling the current quadruples the heat. A shunt resistor measures current by placing a known small resistance in series with the load and reading the voltage across it. Ohm's law gives I = V_shunt / R_shunt.

Here is the worked example. A 0.1 Ω shunt carries a 2 A stall current. Its voltage is 0.1 × 2 = 0.2 V, which an op-amp or ADC can read. Its dissipation is 2² × 0.1 = 0.4 W, so a 1 W shunt is the right rating, because a 0.25 W part would run hot and drift. The same arithmetic tells you that a shunt large enough to read easily is large enough to waste power, which is the trade-off in every current sensor.

The bench supply limit is the simplest protection. Set it to a value above the running current and below the stall current, such as 1.2 A for the motor in this lesson. When the motor stalls, the supply holds the current at the limit and drops its voltage, so the motor stops and the supply does not overload. A battery has no such limit, so the battery branch needs a fuse, sized as Day 12 described: 125 to 150 percent of the continuous current, which for the 1.0 A stall motor and 0.25 A running current is about 2 A, placed on the battery's positive lead.

A fuse's limit is its slowness. A fuse takes time to open, and a short that delivers 30 A can damage a MOSFET before the fuse opens. The bench supply limit is faster, but it only protects if the bench supply is in constant-current mode. The layered approach is the answer: fuse for the wiring, driver limits for the chip, and firmware limits for the motor.

The mistake is to set the bench limit for the running current. The motor then stops during a normal load spike, and the test appears to show that the motor is weak. Set the limit above the expected stall current for short tests, and reduce it once you know the motor's behavior.

On the job, current limiting is required by every motor controller. The best drivers report their current on a sense pin, so the firmware can stop a motor before the fuse opens.

---

### Electrical noise

A brushed motor is an electrical noise source. As the brushes cross the commutator segments, the current switches, and small sparks create voltage spikes that travel along the power wires and radiate into nearby sensors. The motor's PWM adds a second source at the switching frequency, and the two together can shift a sensor reading by tens of millivolts, as Day 11's noise measurements showed.

The physics is the same as the flyback problem. A fast change in current, di/dt, induces a voltage across any inductance in the path. Wires have inductance of about 1 µH per metre, so a 10 cm loop of motor wire and its return has an inductance of roughly 0.1 µH. A current changing by 1 A in 10 ns induces V = 0.1 × 10^-6 × 1 / 10 × 10^-9 = 10 V. That is a large number for a small loop, and it explains why short, tight wiring matters.

The standard fixes are short motor wires, a 100 nF ceramic capacitor across the motor's terminals, and a capacitor from each motor terminal to the motor's metal case, which is connected to ground. A bulk electrolytic capacitor on the motor supply, of 470 µF or more, absorbs the stall surge. Twisting the motor wires together reduces the loop area, and therefore the induced voltage.

Noise fixes trade cost, space and sometimes response time. A large capacitor slows the supply's response to a stall, and a series ferrite bead reduces high-frequency noise but may heat under current. Measure the noise after each change, and keep the change that does the most good for the least cost.

The mistake is to route the motor wires alongside the sensor wires. The noise then appears on the sensor line regardless of the sensor's own quality. The way to check is to run the motor and watch the sensor reading. If the reading changes with the motor, the wiring or the grounding is the cause, and a better sensor will not fix it.

EMC testing for every robot product checks the motor noise against standards that limit emissions and immunity. The noise fixes in this section are the first steps in passing those tests.

---

## Hands-on exercises

Keep the motor's supply current limited, as set up in the exercises, and keep the fuse in the circuit. The motor's stall current is below 1.0 A, and the bench supply should be set to 1.2 A or lower for every test. Never run the motor from an unprotected battery.

1. **Measure the motor's resistance.** With the motor disconnected from everything, set the multimeter to ohms and measure the winding across its two terminals. Record the value. For a 6 V motor with a 1.0 A stall, expect about 6 Ω, but the real value varies. Use the measured value in every calculation that follows.

2. **Measure the no-load current and compute the back-EMF.** Connect the motor to the bench supply at 6.0 V with the current limit at 1.2 A. Let the motor spin free, and read the current on the bench supply's display or with the multimeter's DC amps mode in series. Compute E = V − I × R using your measured R. Compare the E you calculated with what you expect from the no-load speed, if your motor's datasheet lists it.

3. **Measure the stall current briefly.** Hold the shaft gently so it cannot turn, and read the bench supply's current for no more than two seconds. The current should rise toward the stall value and be limited by the bench supply if the motor's stall exceeds the limit. Release the shaft immediately. Record the current, and compare it with V / R. If the motor is warm after the test, wait for it to cool before the next step.

4. **Build the H-bridge test.** Mount the TB6612FNG breakout on the breadboard. Connect VM to the bench supply's 6 V output through the fuse, VCC to the Uno's 5 V, GND to the Uno's GND and the bench supply's negative. Connect STBY to 5 V. Connect the motor to AO1 and AO2. Connect AIN1 to D7, AIN2 to D8 and PWMA to D9. Upload a sketch that sets AIN1 high and AIN2 low, then reverses them every three seconds, with analogWrite on D9 at 180 (about 70 percent duty). Confirm the motor reverses direction on each change.

5. **Test the braking and the PWM steps.** Add a function that sets AIN1 and AIN2 both high for braking. Then step PWMA through 0, 64, 128, 191 and 255, holding each for two seconds. With a multimeter in DC volts mode across the motor terminals, read the average voltage at each step. The readings should be close to the duty cycle times 6 V, with some loss from the driver. Record the results in a table.

6. **Build the switching stage with a flyback diode.** On a second breadboard area, wire a motor from the 6 V supply to the drain of the IRLZ44N, its source to GND, and its gate to the Uno's D5 through 100 Ω, with a 10 kΩ pull-down from gate to GND. Place the 1N5819 across the motor with its cathode (the striped end) to the 6 V supply and its anode to the MOSFET drain. Run a PWM sketch at 50 percent duty on D5 and confirm the motor runs. Do not run a test with the diode removed. The calculation in the lesson shows the spike can exceed the transistor's rating.

Stop immediately if any component becomes hot to the touch, if the bench supply's voltage drops much below its setting, or if the motor makes a grinding sound. Disconnect power, check the wiring against the diagrams, and check the motor for jamming.

---

## Project: a motor driver characterization rig

This project is a measurement rig that you can reuse for any small motor and driver. It is a practical build, so the deliverable is a set of recorded measurements with the calculations that predict them, and a README that explains where the driver's limits sit.

**Goal:** build a repeatable test rig for one motor and one driver, which records the stall current, the no-load current, the PWM voltage response and the driver temperature under a sustained load. The output is a characterization table you can compare against the datasheet and against the power budget from Day 12.

**Repo layout:**

```
100-days-robotics-engineering-day-13-motor-electronics/
├── README.md
├── firmware/
│   └── driver_rig/
│       └── driver_rig.ino
├── measurements/
│   ├── resistance_and_stall.md
│   ├── pwm_voltage_response.csv
│   └── sustained_load_log.csv
├── calculations/
│   └── back_emf_and_shunt.md
├── docs/
│   └── switching_stage_checklist.md
└── images/
    ├── h_bridge_wiring.jpg
    └── switching_stage_wiring.jpg
```

**Starter code:** `firmware/driver_rig/driver_rig.ino`

```cpp
const int AIN1 = 7;
const int AIN2 = 8;
const int PWMA = 9;      // PWM-capable pin on the Uno
const int STBY = 4;      // Driver standby pin, wire it to D4 for this rig (the exercises tie it to 5 V instead)

const int DUTY_STEPS[] = {0, 64, 128, 191, 255};
const int NUM_STEPS = 5;
const unsigned long HOLD_MS = 2000;

void setForward() {
  digitalWrite(AIN1, HIGH);
  digitalWrite(AIN2, LOW);
}

void setStop() {
  digitalWrite(AIN1, LOW);
  digitalWrite(AIN2, LOW);
}

void setup() {
  Serial.begin(115200);
  pinMode(AIN1, OUTPUT);
  pinMode(AIN2, OUTPUT);
  pinMode(PWMA, OUTPUT);
  pinMode(STBY, OUTPUT);
  digitalWrite(STBY, HIGH);
  setForward();
  Serial.println("step,pwm,elapsed_ms");
}

void loop() {
  for (int i = 0; i < NUM_STEPS; i++) {
    analogWrite(PWMA, DUTY_STEPS[i]);
    unsigned long start = millis();
    while (millis() - start < HOLD_MS) {
      // Hold this step; take DMM readings by hand or with a logger
    }
    Serial.print(i);
    Serial.print(",");
    Serial.print(DUTY_STEPS[i]);
    Serial.print(",");
    Serial.println(millis());
  }
  analogWrite(PWMA, 0);
  setStop();
  while (true) { }   // Run once, then stop
}
```

The rig holds each PWM step for two seconds and prints a marker, so you can line up multimeter readings with the step. The loop stops after one pass, which keeps the motor from running unattended.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the wiring, the bench supply limit and the fuse rating.
2. Fill `measurements/resistance_and_stall.md` with the measured winding resistance, the calculated stall current at 6 V, and the measured current from the exercise.
3. Run the PWM sweep. Read the motor terminal voltage at each step and record it in `measurements/pwm_voltage_response.csv` with the step, the duty cycle and the average voltage.
4. Run a sustained load test. Lightly load the shaft with a finger or a 3D-printed wheel for five minutes at 50 percent duty, and record the bench supply current every 30 seconds in `measurements/sustained_load_log.csv`. Touch the driver breakout and the motor after the test, and note their temperature in the README.
5. Write `calculations/back_emf_and_shunt.md`, with the back-EMF calculation from your measured no-load current and the shunt-resistor design for a 2 A limit.
6. Complete `docs/switching_stage_checklist.md`, with the flyback diode orientation, the MOSFET drain-source rating, the gate pull-down and the fuse.

**README checklist:**

- [ ] Goal and the motor's stated specification
- [ ] Wiring diagram and photo
- [ ] Measured winding resistance and stall current
- [ ] PWM voltage response table
- [ ] Sustained load log and the driver's temperature
- [ ] Switching stage checklist
- [ ] One limit of the driver, stated with a number from its datasheet

**Done when:**

- The measured stall current is within 15 percent of V / R using your measured resistance.
- The PWM average voltage at each step is within 10 percent of the duty cycle times the supply voltage, or the difference is explained by driver loss.
- The sustained load test runs for five minutes with the bench supply limit in place and no fuse blow.
- The README states the driver's average current rating from the datasheet and shows that the test current is below 60 percent of it.

**Evidence to commit to GitHub:** the firmware, the measurement files, the calculations, the switching checklist, the photos and the README. The reviewer should be able to see that every limit in the README came from a datasheet or a measurement.

This project uses a motor and a driver that can heat, and it uses a supply that can deliver more current than the wiring is sized for. Keep the bench supply limit set, keep the fuse in the circuit, and stop the test if the driver or the motor becomes too hot to touch comfortably.

---

## Check your understanding

1. A 12 V motor has a winding resistance of 4 Ω and a no-load current of 0.3 A. Calculate its stall current, its back-EMF at no load, and the power dissipated in the winding at stall. Explain why the stall power is the figure that limits how long a stall can be held.
2. A motor's back-EMF falls from 5.4 V to 4.5 V when a load is applied at 6 V with a 6 Ω winding. Calculate the current before and after the load, and explain why the current rises when the speed falls.
3. A driver is set to 75 percent PWM duty from a 12 V supply. Calculate the average voltage across the motor, and explain why the motor's actual speed may not match this average voltage under load.
4. A 2 mH motor coil carries 1.5 A and the driving transistor switches it off in 2 microseconds. Calculate the voltage spike without a flyback diode, and explain what a flyback diode does to it.
5. A motor driver is rated for 1.2 A average and 3.2 A peak. A motor draws 0.6 A running and 2.8 A stall for 0.5 seconds at a time. Decide whether the driver is suitable, and state the test that would confirm your decision.
6. Explain why a motor's noise can appear on an analog sensor even when the motor is on a separate supply. Name two fixes that reduce the coupling without changing the sensor.

---

### Answers

1. The stall current is 12 / 4 = 3.0 A. The no-load back-EMF is E = V − I × R = 12 − 0.3 × 4 = 12 − 1.2 = 10.8 V. The stall dissipation is I² × R = 3.0² × 4 = 36 W. A motor dissipating 36 W in its winding heats quickly, and a stall held for more than a few seconds can damage the insulation, so the stall time is limited by the heat, not by the current.
2. Before the load, I = (6 − 5.4) / 6 = 0.6 / 6 = 0.10 A. After the load, I = (6 − 4.5) / 6 = 1.5 / 6 = 0.25 A. The current rises because the back-EMF falls with the speed, so less voltage opposes the supply, and the net voltage across the winding rises.
3. The average voltage is 0.75 × 12 = 9 V. Under load the motor slows, and its back-EMF falls, so the current rises. The speed depends on the back-EMF and the load, not only on the average voltage, so the actual speed will not match the duty cycle exactly.
4. The voltage spike is V = L × di / dt = 0.002 × 1.5 / 0.000002 = 0.003 / 0.000002 = 1500 V. A flyback diode across the coil clamps the voltage to about the supply plus the diode's forward drop, so the transistor sees only a small voltage above the supply.
5. The driver is suitable for the running current, which is below 1.2 A average. For the stall, the peak is 2.8 A, which is below the 3.2 A peak rating, and the 0.5-second stall is brief. The average current over a cycle depends on the duty of the stall, so the test is to run the real stall pattern for a few minutes and measure the chip's temperature and the average current. If the chip stays within its rated temperature and the average is under the rating, the driver is suitable.
6. The motor's switching current couples into the sensor through the shared ground and through the magnetic and electric fields of nearby wires. Two fixes that do not change the sensor are to route the sensor wires away from the motor wires and to add a capacitor across the motor terminals, which reduces the switching edges that radiate. A star ground and a bulk capacitor on the motor supply also help.

---

## Sources to read and watch

*Read*

- [Wikipedia: DC motor](https://en.wikipedia.org/wiki/DC_motor) for the basic construction and the back-EMF relation.
- [Wikipedia: Counter-electromotive force](https://en.wikipedia.org/wiki/Counter-electromotive_force) for the principle and its use in speed estimation.
- [Wikipedia: H-bridge](https://en.wikipedia.org/wiki/H-bridge) for the switch arrangement and the shoot-through hazard.
- [Wikipedia: Pulse-width modulation](https://en.wikipedia.org/wiki/Pulse-width_modulation) for the duty-cycle relation.
- [Wikipedia: Freewheeling diode](https://en.wikipedia.org/wiki/Freewheeling_diode) for the flyback diode.
- [Wikipedia: Power MOSFET](https://en.wikipedia.org/wiki/Power_MOSFET) for the switching characteristics.
- The TB6612FNG datasheet from Toshiba. Search "TB6612FNG datasheet Toshiba" for the ratings and the logic table.
- The IRLZ44N datasheet from Infineon or its original manufacturer. Search "IRLZ44N datasheet" for the gate threshold and the drain-source rating.
- The 1N5819 datasheet from a major diode maker. Search "1N5819 datasheet" for the reverse voltage and forward drop.

*Tutorials*

- SparkFun Learn, the tutorials on motor driver basics and on H-bridges. Search "SparkFun Learn H-bridge motor driver".
- Adafruit Learning System, the guides on DC motors and on motor driver selection. Search "Adafruit DC motor driver guide".
- Arduino Language Reference, the page for analogWrite and the PWM frequencies on the Uno. Site: docs.arduino.cc.

*Watch*

- MIT OpenCourseWare, 6.002 Circuits and Electronics. The lectures on inductors and the transient response explain the flyback spike.
- Ben Eater's YouTube channel. Search his videos on transistors and on building logic switching circuits, which show the same switching habits on breadboard.

*Practice*

- Falstad circuit simulator at falstad.com/circuit. Build the MOSFET switch with an inductor and watch the voltage spike appear when the switch opens, with and without the diode.

---

## Tomorrow

Today you drove a motor safely and measured how it responds to voltage and load. Tomorrow you will read the documents that describe every part you have used so far. Day 14 covers schematics and datasheets: how to read the symbols, nets and rails in a circuit diagram, how to find the pin numbers and the absolute maximum ratings in a datasheet, and how to read a timing diagram. The TB6612FNG and IRLZ44N datasheets you looked up today are the first documents you will read in that lesson.

**Next:** [Day 14: Schematics and Datasheets](Robotics-Day-14-Schematics-and-Datasheets.md)
