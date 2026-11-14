# Day 26: DC Motors

*100 Days of Robotics Engineering, Module 4: Sensors and Actuators*

**Time:** about 4 hours
**Level:** intermediate. You need Day 13 (motor electronics), Day 12 (the power budget) and Day 14 (reading datasheets). This day turns the electrical behavior you measured into a mechanical picture: torque, speed and gearing. Those three numbers decide which motor a robot wheel or joint needs.

**By the end of this day, you will have:**

- A torque constant and a speed constant for one small brushed DC gearmotor, measured from your own bench data
- A gearbox calculation that converts the motor's stall torque into output torque at the wheel
- A torque-speed curve, plotted from a measurement log and a short Python script
- A written sizing note that picks a motor and driver for a 2 kg robot wheel on a flat floor

**What you need:**

- The small 6 V brushed DC gearmotor and TB6612FNG driver from Day 13, plus the bench supply set to 6.0 V with a 1.2 A current limit
- A digital multimeter with DC volts, DC amps and ohms modes, about $15 to $25
- A phone with a free optical tachometer app. Several are on the Google Play Store and the Apple App Store. Search "optical tachometer" and check the reviews, since accuracy varies.
- A 10 cm wheel or pulley taped to the motor shaft, and a 500 g to 1 kg spring scale, about $10 to $15 for a hanging scale
- A string, a cardboard lever arm and some tape
- Python 3.12 with NumPy 2.1 or newer and Matplotlib 3.9 or newer, both free and open source
- Arduino IDE 2.3 or newer if you reuse the Day 13 sketch for the H-bridge
- Optional: a gearmotor with a published gear ratio, such as 30:1 or 48:1. Check the listed ratio before you run the lesson's numbers on a different motor.

**Prerequisites:**

- Day 13: the back-EMF relation E = V minus I times R, the stall current and the H-bridge
- Day 12: the power budget and the idea of a fuse sized for the continuous current
- Day 14: reading a datasheet for the rated voltage, stall current and no-load speed
- Basic Python: running a script, reading a CSV file and plotting a line

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How electric motors became robot actuators](#how-electric-motors-became-robot-actuators)
- [Glossary](#glossary)
- [Construction of a brushed DC motor](#construction-of-a-brushed-dc-motor)
- [Torque and current](#torque-and-current)
- [Speed and back-EMF](#speed-and-back-emf)
- [The torque-speed curve](#the-torque-speed-curve)
- [Gearing](#gearing)
- [Stall current and heat](#stall-current-and-heat)
- [Drivers and the operating envelope](#drivers-and-the-operating-envelope)
- [Encoders and where they fit](#encoders-and-where-they-fit)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a DC motor torque and speed characterization](#project-a-dc-motor-torque-and-speed-characterization)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Almost every mobile robot moves because a motor turns a wheel through a gearbox. The motor's electrical data on the nameplate tells you very little about whether it will move the robot up a ramp. Three numbers matter: how much torque it can give at the shaft, how fast it turns when unloaded, and how much current it draws while it works. A team that picks a motor from its price or its size alone finds out in testing that the wheels slip, the robot stalls on carpet or the driver overheats.

Today you connect Day 13's electrical model to the mechanical side. You will measure the motor's torque constant and speed constant, then use them to predict the torque at the wheel after a gearbox. The motor you use is the same one from Day 13, so every number you measure today can be checked against the driver limits and the fuse rating you already set.

---

## How electric motors became robot actuators

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1820s | Early experiments by Hans Christian Ørsted and Michael Faraday show that current produces motion in a magnetic field. | The force on a current-carrying wire in a field is the principle behind every DC motor. |
| 1830s to 1870s | Early DC motors with electromagnets and mechanical commutators appear, and the first practical machines are built in the following decades. | The brushed DC motor is still the simplest motor for a small robot. |
| 1880s to 1900s | Electric motors replace steam and belt drives in factories, and speed control through field and armature voltage becomes common. | Engineers learn to size motors by torque and speed, the same way you will in this lesson. |
| 1960s to 1980s | Permanent-magnet DC motors with ferrite and then rare-earth magnets become cheap. | Small hobby gearmotors with permanent magnets come from this lineage. |
| Today | Brushless DC motors, with electronic commutation and sensors, dominate high-performance robots and drones. | Brushed motors remain common in education and low-cost mobile robots. |

The physics has not changed. What changed is the cost of the magnets, the electronics that drive them, and the sensors that report their speed. Today's exercises use a brushed motor because the same torque and speed relations apply, and the hardware costs a few dollars.

---

## Glossary

- **Torque:** the turning force at a shaft, measured in newton-metres (N·m).
- **Speed constant (Ke):** the back-EMF per unit of shaft speed, in volts per radian per second.
- **Torque constant (Kt):** the torque per unit of current, in newton-metres per ampere.
- **No-load speed:** the speed the motor reaches with no external load, where its torque only has to overcome friction.
- **Stall torque:** the torque at zero speed, which is the largest torque the motor can give at its supply voltage.
- **Gearbox ratio:** the number of motor turns for each output turn. A 48:1 ratio means the output turns once for every 48 motor turns.
- **Gearbox efficiency:** the fraction of the input power that reaches the output. Spur gearboxes are often 80 to 90 percent efficient when new.
- **Rpm (revolutions per minute):** a speed in turns per minute. One rpm equals 2π divided by 60 radians per second, about 0.1047 rad/s.
- **Duty cycle:** the fraction of time a PWM signal is on, as Day 13 defined it.
- **Continuous current:** the current the motor can draw for a long time without overheating.
- **Spring scale:** a hanging scale that reads force in newtons or grams-force, used here to measure torque at a known lever arm.

---

## Construction of a brushed DC motor

A brushed DC motor has a stator and a rotor. The stator is a set of permanent magnets fixed to the motor's outer can. The rotor, also called the armature, is a shaft carrying coils of wire wound on an iron core. The end of the shaft carries a commutator, which is a ring of copper segments. Two brushes, usually made of carbon or graphite, press against the commutator and carry current into the coils. As the rotor turns, the brushes move from one segment to the next, which reverses the current in each coil at the right moment so that the force keeps pushing the shaft in one direction.

The analogy is a revolving door with a set of hands that push it. The hands are the coils, and the commutator is a timing mechanism that switches which hand pushes at each point in the turn. Without the commutator, the coils would push the shaft in one direction for half a turn and then push back, and the motor would stop.

The first principle is the force on a current-carrying wire in a magnetic field. For a wire of length L carrying current I at right angles to a field B, the force is F = B × L × I. A coil of radius r gives a torque of about T = B × L × I × r, where L is the length of wire in the field. The torque is proportional to the current, which is why the motor's torque constant Kt is a simple number once the magnet and winding are fixed.

The worked example uses the motor from Day 13. Suppose the motor produces 0.027 N·m of torque at a current of 1.0 A. Its torque constant is Kt = 0.027 / 1.0 = 0.027 N·m per ampere. At a current of 0.25 A, the torque is 0.027 × 0.25 = 0.0068 N·m. The number is small, which is why most small brushed motors are paired with a gearbox before they drive anything heavy.

The common mistake is to assume the torque constant is the same for every motor of the same size. Winding turns, magnet strength and the rotor's diameter all change Kt, and a datasheet's torque figure is a result of those choices. Measure it, or use the datasheet's value only after checking the units.

On the job, datasheets from robotics suppliers list stall torque, stall current, no-load speed and no-load current. Those four numbers are enough to estimate Kt, Ke and the resistance, and the rest of today's work follows from them.

---

## Torque and current

Think of pulling a rope with a hand and feeling the force. The harder you pull, the more the rope pushes back, and the more force you need from your arm. A motor's current is the effort it spends to produce torque, so current and torque rise together. The relation is linear for a brushed motor at a fixed magnetic field.

The first principle is the torque relation T = Kt × I. The current I is the electrical current into the winding, and the torque T is the torque at the shaft, before the gearbox. Friction in the bearings and brushes takes a small part of the torque away, so the useful output is T minus the friction torque. At no load, the friction torque equals the torque the motor produces, and the no-load current I0 is the current needed to overcome that friction.

Here is a worked example with the numbers you will measure in the exercises. The no-load current is 0.10 A, so the motor's friction torque is 0.027 × 0.10 = 0.0027 N·m. At stall, the current is 1.0 A and the shaft torque is 0.027 N·m. The useful torque at a load is the shaft torque minus 0.0027 N·m. If the load requires 0.012 N·m, the current is I = (0.012 + 0.0027) / 0.027 = 0.544 A. Rounded, the motor draws about 0.54 A at that load.

The common mistake is to read the motor's current as a measure of the load alone. Current also includes the no-load friction, and the friction changes with temperature, brush wear and the lubricant in the gearbox. A motor that draws more current at the same load may be warm or worn, not overloaded.

Current sensing is the way to see this in a robot. A driver with a current-sense output, or a shunt resistor, gives the current in real time. Torque is then estimated as Kt times the measured current, minus the friction. The estimate is useful for detecting a jam, which shows up as a sudden rise in current with no change in command. It is not a precise torque sensor, because Kt drifts with temperature.

In the robotics market, current-based torque estimation is used in many low-cost manipulators and in the collision detection of cobots, where the joint motors' current is the cheapest signal for contact.

---

## Speed and back-EMF

A spinning motor is also a generator. When it turns, the coils cut through the magnetic field and produce a voltage that opposes the supply. Day 13 called this back-EMF. The faster the motor spins, the larger the back-EMF, and the smaller the current that the supply can push through the winding.

The first principle is E = Ke × ω, where E is the back-EMF in volts, ω is the shaft speed in radians per second, and Ke is the speed constant in volts per radian per second. For a motor, Ke equals Kt when both are in SI units. That identity comes from energy conservation: the electrical power converted to mechanical power, E × I, equals the mechanical power, T × ω, which gives Kt = Ke.

The worked example uses the no-load data. At 6.0 V with a winding resistance of 6.0 Ω and a no-load current of 0.10 A, the back-EMF is E = 6.0 minus 0.10 times 6.0, which equals 5.4 V. Using Ke = 0.027 V·s/rad, the no-load speed is ω = 5.4 / 0.027 = 200 rad/s. In rpm that is 200 times 60 divided by 2π, about 1910 rpm. The same motor with a 48:1 gearbox turns at 1910 / 48, about 40 rpm, at the output.

The common mistake is to read the motor's speed from the supply voltage alone. The no-load speed is proportional to the supply voltage only when the winding drop is small, and the winding drop is I times R, which grows with load. A robot that drives at a fixed PWM duty will therefore slow down when the load rises, and the speed is set by the load as well as the duty cycle.

Speed is measured in the real world with a tachometer. A phone app that counts reflections from a strip of tape on the shaft is accurate to a few rpm for a steady motor, and it is free. An encoder on the shaft is the usual measurement for a robot, and Day 29 covers it. Back-EMF estimation, which Day 13 introduced, is a cheaper but less accurate method.

---

## The torque-speed curve

A motor's behavior is summarized by one line on a plot. Torque on the horizontal axis and speed on the vertical axis, with the no-load speed at zero torque and the stall torque at zero speed, and a straight line between them. That line is the torque-speed curve, and it is the single most useful picture of a DC motor for a robot designer.

The first principle comes from combining the two equations. From E = V minus I times R and E = Ke times ω, the current is I = (V minus Ke times ω) / R. Substituting into T = Kt times I gives T = Kt times (V minus Ke times ω) / R. This is a straight line in T and ω. Its slope is minus Kt times Ke over R, and its intercepts are the stall torque at ω = 0 and the no-load speed at T = 0.

The worked example gives the same numbers as before. Stall torque is T_stall = 0.027 × 6.0 / 6.0 = 0.027 N·m. No-load speed is ω0 = 6.0 / 0.027 = 222 rad/s, which is a little above the 200 rad/s we found earlier, because the no-load friction was ignored. The real curve bends slightly near no load because of friction, but the straight line is a good engineering approximation.

The common mistake is to pick a motor that gives the required torque at its no-load speed. The required torque and speed must lie inside the curve at the operating point, which is the point where the load's torque line crosses the motor's curve. A motor that works at 20 rpm with 0.01 N·m may fail at 5 rpm with 0.02 N·m, because the curve gives less torque at low speed and more torque at high speed.

The torque-speed curve is also how datasheets are read. A gearbox shifts the curve, and the PWM duty cycle scales it. The operating envelope is the region under the curve that the driver can supply, which Day 13's current limits bound.

In the job market, the torque-speed curve is the first plot in every motor selection report, and it is the figure a reviewer asks for when a robot is underpowered.

---

## Gearing

A bicycle lets you trade speed for force. In a low gear you pedal faster and push less hard; in a high gear you push harder and spin slower. A gearbox does the same trade for a motor. The motor is fast and weak, and the gearbox turns it into a slow and strong output.

The first principle is that power is conserved apart from losses. The gearbox ratio N is the motor speed divided by the output speed, and the output torque is the motor torque times N times the gearbox efficiency η. So T_out = T_motor times N times η, and ω_out = ω_motor divided by N. Multiplying the two gives T_out times ω_out equal to T_motor times ω_motor times η, which is the power equation with losses.

The worked example uses a 48:1 spur gearbox with 80 percent efficiency. The stall torque at the output is 0.027 × 48 × 0.80 = 1.04 N·m. The no-load speed at the output is 222 / 48 = 4.6 rad/s, about 44 rpm. The motor's stall current stays at 1.0 A, so the driver still sees the same current, while the wheel now has far more torque.

The common mistake is to forget the efficiency and to assume the output torque is exactly N times the motor torque. A spur gearbox with a few stages loses 10 to 20 percent to friction, and a worm gearbox can lose more than half. Efficiency also changes with temperature, load and lubricant, and a gearbox that is new and clean is more efficient than one that has been run dry.

Gearboxes also add backlash, which is the small free movement between meshing teeth. Backlash makes a position estimate from a motor encoder wrong by an amount equal to the backlash divided by N, and it causes a small dead zone when the motor reverses. Design the gearbox and the sensor placement together, and mention backlash in every position-control requirement.

On the job, gear ratio is the first design choice for a drivetrain. A warehouse robot's wheel gearbox is chosen so the robot starts with the load it carries, and the robot's top speed is then set by the motor's no-load speed divided by the ratio.

---

## Stall current and heat

A motor at stall draws its largest current and produces its largest torque, but it also converts all of its electrical power into heat in the winding. The stall power is I squared times R, which for the Day 13 motor at 1.0 A and 6.0 Ω is 6.0 W. A motor is designed to take that power only for a short time.

The first principle is that the winding's temperature rises with the power dissipated and falls with the heat carried away. The rise follows a thermal time constant, which for a small motor is often tens of seconds. Energy in the winding is E = P times t. If the winding can store, say, 20 joules before it reaches its insulation limit, the motor can take 6 W for about 3.3 seconds. The number is an estimate, but it shows that stall time is limited by the rating.

The worked example applies the same reasoning to the duty cycle. If a robot repeatedly stalls against a wall for 2 seconds and then runs free for 8 seconds, the average power is 6.0 times 2 / 10 = 1.2 W. That average is well inside the motor's thermal limit, but the 2-second peak is the one that matters for insulation, so the stall must not be repeated without a rest.

The common mistake is to design from the average current alone. A driver rated for 1.2 A average and 3.2 A peak may still fail if the stall lasts long enough to overheat the chip. Check both the peak and the time at peak, and use the datasheet's thermal figures for the chip.

Heat also reduces torque. Magnets lose strength as the temperature rises, so a hot motor gives less torque at the same current. The effect is small for a ferrite magnet at moderate temperatures and larger for some rare-earth magnets. The practical rule is to keep the motor cool, because a hot motor is also a weak one.

In the market, stall and thermal behavior decides whether an actuator survives field testing. Motor suppliers publish duty cycle curves for this reason.

---

## Drivers and the operating envelope

A driver is the amplifier between the microcontroller and the motor, as Day 13 explained. From the motor's point of view, the driver sets the supply voltage and limits the current. The motor's torque-speed curve is then the region that the driver can supply, which is called the operating envelope.

The first principle is that the envelope is set by three limits. The torque limit is the stall torque at the supply voltage, or the current limit times Kt, whichever is lower. The speed limit is the no-load speed at the supply voltage, reduced by the gearbox ratio. The thermal limit is the time the motor can spend in the high-current region before its temperature exceeds the rated value.

The worked example uses the TB6612FNG from Day 13, which is rated for about 1.2 A average and a higher peak. With the 6.0 V supply, the current limit of 1.2 A sets a torque ceiling of 0.027 times 1.2, which is 0.032 N·m at the motor shaft. The motor's own stall current of 1.0 A is below that limit, so the motor is the limit, not the driver. A larger motor on the same driver would make the driver the limit.

The common mistake is to pick a driver from its voltage rating alone. The voltage rating must exceed the battery's full charge voltage, and the current rating must exceed the stall current for the stall duration. A 2S lithium polymer pack at 8.4 V full may exceed a driver's motor-supply limit even when the motor is 6 V, so check the battery's full voltage, not its nominal voltage.

Drivers also differ in how they handle regeneration. When a geared motor is pushed by its load, as on a downhill ramp, it generates current back into the driver. A driver without a path for that current can see a voltage rise on its supply. Many drivers include a flyback diode or a synchronous path for this, and the datasheet says which.

On the job, drivers are matched to the motor by voltage, current and envelope. The envelope plot is part of every motor and driver selection document.

---

## Encoders and where they fit

A DC motor by itself does not report its position or speed. The back-EMF estimate works for some tasks, but it is noisy and it depends on the motor constant, which drifts. An encoder is a sensor on the motor shaft that reports the rotation in pulses, and it is the standard way to measure speed and position in a robot.

The first principle is that an encoder counts events, and the speed is the number of counts per second divided by the counts per revolution. Day 29 covers encoders in detail, including quadrature signals and direction. For today's characterization, a simple tachometer is enough, and an optical tachometer app gives an accurate speed reading for a steady motor.

The worked example is the conversion from counts to speed. A motor encoder with 11 pulses per revolution on the motor shaft, read at the output of a 48:1 gearbox with 4 times quadrature decoding, gives 11 times 48 times 4 counts per output revolution. That is 2112 counts per output turn. A count rate of 1408 counts per second corresponds to 1408 / 2112 = 0.667 output turns per second, or 40 rpm.

The common mistake is to forget the quadrature factor or the gearbox ratio when converting counts to speed. The result is a speed that is off by a factor of four or 48, and it looks plausible enough to pass review. Write the conversion as a function with the counts per revolution as a named constant, and test it with a known speed.

Encoders are a part of the motor system, and they are often sold with the motor. Choosing an encoder that matches the motor's shaft size, the required resolution and the signal voltage is part of the motor selection.

---

## Hands-on exercises

Keep the motor's supply current limited and keep the fuse in the circuit, as Day 13 set up. The bench supply stays at 6.0 V with the current limit at 1.2 A. Never run the motor from an unprotected battery, and stop the test if the motor or the driver gets too hot to hold comfortably.

1. **Measure the winding resistance and the no-load data.** With the motor disconnected from the driver, measure the winding resistance with the multimeter's ohms mode and record it. Connect the motor to the bench supply at 6.0 V, let it spin free for 30 seconds, and record the no-load current. Measure the no-load speed with the optical tachometer app and a strip of reflective tape on the shaft. Record three readings and take the average.

2. **Compute Kt and Ke.** Use the measured resistance R, the no-load current I0 and the no-load speed in rad/s. Compute E = V minus I0 times R, then Ke = E / ω0. Set Kt equal to Ke, because both are in SI units. Write both numbers in a table with their units. If they differ by more than 15 percent, recheck the speed reading before you continue.

3. **Measure the stall torque with a lever arm.** Attach a lever arm of known length, for example 5 cm, to the output shaft, which you can make from cardboard taped to the wheel hub. Hang the spring scale from the end of the arm and hold the shaft so it cannot turn. Read the force for no more than two seconds, with the bench supply at 1.2 A. Compute the torque as the force times the lever arm. Release the shaft and let the motor cool.

4. **Compare the measured torque with Kt times the stall current.** Use the stall current from the bench supply display and compute Kt times the current. Note the difference and explain it in one sentence. A difference of 10 to 20 percent is normal, because the measurement uses a hand-held scale and a short hold.

5. **Plot a torque-speed curve.** Collect at least five points by holding the motor at different loads using the spring scale, and read the speed with the tachometer at each. Put torque on the horizontal axis and speed on the vertical axis. Draw the straight line between the no-load speed and the stall torque, and compare your points with the line.

6. **Gearbox calculation.** Using the same motor and a 48:1 gearbox with 80 percent efficiency, compute the output stall torque and the output no-load speed. If your gearbox ratio differs, use your number. Write the calculation on paper before checking it against a measurement, if you have a geared motor available.

---

## Project: a DC motor torque and speed characterization

This project is a repeatable measurement of one motor and its gearbox. It produces a torque-speed curve and a set of constants that a robot designer can use for sizing. It is a practical build with a short analysis script, so the deliverable is a dataset, a plot and a README that states the limits of the measurement.

**Goal:** measure the Kt, Ke, stall torque, no-load speed and the torque-speed curve for one gearmotor on a 6.0 V supply, then compare the measured values with the datasheet and the calculations in this lesson.

**Repo layout:**

```
100-days-robotics-engineering-day-26-dc-motors/
├── README.md
├── analysis/
│   ├── fit_torque_constant.py
│   └── requirements.txt
├── data/
│   ├── no_load_speed.csv
│   └── torque_speed_points.csv
├── calculations/
│   └── gearbox_and_sizing.md
├── plots/
│   └── torque_speed_curve.png
└── images/
    └── test_rig.jpg
```

**Starter code:** `analysis/fit_torque_constant.py`

```python
"""Fit a torque-speed line to measured points and report Kt, Ke and stall torque.

Expected CSV columns in data/torque_speed_points.csv:
    torque_nm, speed_rad_s, current_a
"""
import csv
from pathlib import Path

import numpy as np
import matplotlib.pyplot as plt

DATA = Path(__file__).resolve().parent.parent / "data" / "torque_speed_points.csv"
PLOTS = Path(__file__).resolve().parent.parent / "plots"

def load_points(path):
    torque, speed, current = [], [], []
    with open(path, newline="") as f:
        for row in csv.DictReader(f):
            torque.append(float(row["torque_nm"]))
            speed.append(float(row["speed_rad_s"]))
            current.append(float(row["current_a"]))
    return np.array(torque), np.array(speed), np.array(current)

def main():
    torque, speed, current = load_points(DATA)

    # Fit a line: speed = a * torque + b. Then the no-load speed is b and
    # the stall torque is -b / a.
    a, b = np.polyfit(torque, speed, 1)
    no_load_speed = b
    stall_torque = -b / a

    # Kt from the slope of torque against current (least squares through zero).
    kt = np.sum(torque * current) / np.sum(current * current)

    print(f"Torque constant Kt   : {kt:.4f} N·m/A")
    print(f"No-load speed (fit)  : {no_load_speed:.1f} rad/s")
    print(f"Stall torque (fit)   : {stall_torque:.4f} N·m")

    PLOTS.mkdir(exist_ok=True)
    fig, ax = plt.subplots(figsize=(6, 4))
    ax.scatter(torque, speed, label="measured")
    xs = np.linspace(0, stall_torque, 100)
    ax.plot(xs, a * xs + b, color="C1", label="linear fit")
    ax.set_xlabel("Shaft torque (N·m)")
    ax.set_ylabel("Speed (rad/s)")
    ax.set_title("Torque-speed curve, 6.0 V")
    ax.legend()
    fig.tight_layout()
    fig.savefig(PLOTS / "torque_speed_curve.png", dpi=150)

if __name__ == "__main__":
    main()
```

The script fits a line to the measured torque and speed points, reads the no-load speed and stall torque from the intercepts, and estimates Kt from the torque and current. The Kt estimate uses a least-squares fit through zero, which is correct because the torque is proportional to current with no offset in the ideal model.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the motor's datasheet values, the wiring and the bench supply limit.
2. Record the winding resistance and the no-load data in `data/no_load_speed.csv`, with the voltage, current, speed and the time of each reading.
3. Collect at least eight torque-speed points using the spring scale and the lever arm. Hold each load for five seconds, read the speed and the current, and save the rows to `data/torque_speed_points.csv`.
4. Run `analysis/fit_torque_constant.py` and save the printed numbers in the README.
5. Write `calculations/gearbox_and_sizing.md` with the output stall torque for your gearbox ratio and efficiency, and a sizing check for a 2 kg robot wheel of 3 cm radius on a flat floor. The rolling resistance coefficient for a rubber tire on a hard floor is commonly taken as about 0.01 to 0.02, so use a range and state it.
6. Compare the measured Kt and stall torque with the datasheet, and state any difference with a reason.

**README checklist:**

- [ ] Goal and the motor's datasheet specification, with the source and the date
- [ ] Wiring diagram and a photo of the test rig
- [ ] Measured winding resistance, Kt, Ke and no-load speed
- [ ] Torque-speed table and the plot
- [ ] Gearbox calculation with the ratio and efficiency stated
- [ ] Sizing check for the wheel, with the assumptions listed
- [ ] One limitation of the measurement, such as the hand-held spring scale or the optical tachometer's accuracy

**Done when:**

- The script runs from the repo root with `python analysis/fit_torque_constant.py` and produces the plot and the three printed numbers.
- The measured stall torque is within 25 percent of Kt times the measured stall current, and the README explains any larger difference.
- The torque-speed points lie within about 15 percent of the fitted line, or the README explains which point is an outlier and why.
- The README states the gearbox ratio and efficiency used, and every output torque in the calculations uses them.

**Evidence to commit to GitHub:** the script, the CSV data, the plot, the calculation file, the test rig photo and the README. A reviewer should be able to rerun the script and get the same numbers from the same data.

This project is light on hazard, but the stall tests heat the motor. Keep each stall under two seconds and let the motor cool between tests.

---

## Check your understanding

1. A 12 V motor has a winding resistance of 4 Ω and a no-load current of 0.3 A. Calculate its stall current, its no-load back-EMF and its stall torque if Kt equals 0.05 N·m per ampere. Explain why the stall torque is not the right number to size the wheel's torque at the output.
2. A motor with a 30:1 gearbox at 85 percent efficiency produces 0.020 N·m at the motor shaft at a certain current. Calculate the output torque. Then explain why a gearbox with a large ratio can make a robot slower even when its motor is fast.
3. A motor's no-load speed is 210 rad/s at 6.0 V. Estimate its no-load speed at 4.5 V, assuming the back-EMF is proportional to the supply voltage. Explain why the real speed at 4.5 V under a light load is lower than your estimate.
4. A robot's drive motor stalls for 3 seconds at 2.5 A against a wall, then runs at 0.4 A for 27 seconds. Calculate the average current over the 30 seconds and the average power in a 6 Ω winding. Explain why the average is not the right figure for deciding whether the motor is safe.
5. Two motors have the same stall torque, but one has a higher no-load speed. Explain which motor is better for a robot that must climb a 15-degree ramp, and which is better for a robot that drives on a flat floor at high speed. Use the torque-speed curve in your answer.
6. A current-based torque estimate reads 0.02 N·m, but the robot's wheel is not moving. Name two causes that could explain the reading without a true load, and state one test that separates them.

---

### Answers

1. The stall current is 12 / 4 = 3.0 A. The no-load back-EMF is E = V minus I0 times R = 12 minus 0.3 times 4 = 10.8 V. The stall torque is Kt times the stall current, 0.05 times 3.0 = 0.15 N·m. The stall torque is at the motor shaft, before the gearbox, so the wheel torque depends on the gearbox ratio and efficiency as well.
2. The output torque is 0.020 times 30 times 0.85 = 0.51 N·m. A large ratio slows the output, because the output speed is the motor speed divided by the ratio. The motor's fast speed is traded for torque, so the robot's top speed falls with the ratio.
3. The estimate is 210 times 4.5 / 6.0 = 157.5 rad/s. The real speed under a light load is lower because the winding drop I times R takes a larger share of the lower voltage, and friction and load take more of the available torque.
4. The average current is (3 times 2.5 + 27 times 0.4) / 30 = (7.5 + 10.8) / 30 = 0.61 A. The stall period dissipates I squared times R, which at 2.5 A and 6 Ω is 37.5 W during the stall. The average hides that peak, and the thermal limit depends on the peak power and its duration, so the safety check must use the stall power and the time.
5. For the ramp, the higher-torque motor is better, because the robot needs torque at low speed to climb. For the flat floor, the higher-speed motor is better because the robot needs speed at a moderate torque. The torque-speed curve shows that the operating point on the ramp lies at low speed and high torque, and the flat-floor operating point lies at high speed and low torque. A gearbox can move the curve to suit either need.
6. A stuck wheel, a binding gearbox, a worn brush or friction in the bearings can each raise the current, and the reading looks like a load. A test that separates them is to spin the motor with the wheel removed. If the current stays high, the motor or gearbox is at fault. If the current falls, the load is external.

---

## Sources

*Read*

- [Wikipedia: DC motor](https://en.wikipedia.org/wiki/DC_motor) for the construction, the commutator and the back-EMF relation.
- [Wikipedia: Torque](https://en.wikipedia.org/wiki/Torque) for the definition and units.
- [Wikipedia: Gear train](https://en.wikipedia.org/wiki/Gear_train) for the ratio and efficiency of gearboxes.
- [Wikipedia: Back-EMF](https://en.wikipedia.org/wiki/Back-EMF) for the speed relation, although the article title may vary by language. Search the title if the link moves.
- The datasheet of the motor you bought. Search the supplier's name and the motor's part number. Pololu and SparkFun list the stall torque, stall current and no-load speed on their product pages.
- The TB6612FNG datasheet from Toshiba, from Day 13, for the current and thermal limits.

*Tutorials*

- GeeksforGeeks, the articles on DC motors and on the working principle of electric motors. Search "GeeksforGeeks DC motor working principle".
- SparkFun Learn, the tutorial on motor basics and the one on gearmotors. Search "SparkFun Learn motor basics".
- Adafruit Learning System, the guides on DC motors and motor selection. Search "Adafruit DC motor guide".

*Watch*

- MIT OpenCourseWare, 6.002 Circuits and Electronics, the lectures on electromechanical systems and motors. Search the course on ocw.mit.edu.
- Stanford Engineering, the robotics course lectures on actuators, if available on the Stanford Engineering YouTube channel. Search the course title, since lecture numbers change.

*Practice*

- Falstad circuit simulator at falstad.com/circuit. Build a DC motor model from a resistor, an inductor and a voltage source, and watch the current change as the load changes.

---

## Tomorrow

Tomorrow you will use the servo motor, which adds a position feedback loop to a motor and a gearbox in one package. Day 27 covers the servo's command signal, its feedback concept, its torque rating and its calibration. Where a DC motor needs an encoder to know its position, a servo has the feedback built in, and today's torque-speed curve is still the test for whether a servo has enough torque for the joint.

**Next:** [Day 27: Servo Motors](Robotics-Day-27-Servo-Motors.md)
