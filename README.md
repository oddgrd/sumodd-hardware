# Schema

This repository holds the PCB schematic and PCB CAD files for the
[Sumodd](https://github.com/oddgrd/sumo) mini-sumo robot motherboard. When a new iteration of the
PCB is manufactured, the latest included commit is tagged with a version, e.g. `v0.2`.

## Power

The PCB is powered by a 2s 7.4V, 25C, 800 mAh LIPO battery. This voltage is fed to a MOSFET
transistor source, the gate of which is controlled by a SPDT switch which flips between battery
input and ground. The transistor drain feeds a 7.4V power rail, which connects directly to the 
motor driver VM input. However, the MCU and other components want a stable 3.3V, so we use a
MPM3610AGQV-P buck converter configured to step down the battery input voltage, to feed a 3.3V
power rail.

### MPM3610 Synchronous Step-Down Converter

Datasheet: https://cdn-learn.adafruit.com/assets/assets/000/127/631/original/MPM3610GQV-Z.pdf?1707519066

The MPM3610 is largely implemented as documented in the typical applications sections of the datasheet,
figure 12.

- We connect EN (enable) to IN (VIN) with a 100k resistor, to pull EN high when the device is
powered.
- We have a decoupling 10 uF capacitor on the IN (VIN) pin, because the regulator switches a MOSFET on and
off rapidly (2MHz switching frequency), which leads to ripple currents. The capacitor maintains the DC
voltage from the battery, and provides the AC current the regulator demands each switching cycle.
    - Note: a low-ESR ceramic capacitor with X5R or X7R dielectrics is recommended in the data sheet,
    and it should be placed as close to the IC as possible. It should have an RMS current rating
    greater than half of the maximum load current. Since the regulator will only supply the MCU,
    the sensors and some LEDs, it will have a low maximum load current, around 100mA.
- We have another decoupling 22uF capacitor on the output, to stabilize the DC output voltage,
providing transient current when the load suddenly changes. Ceramic, tantalum or low ESR electrolytic
capacitors are recommended, to keep the output-voltage ripple low.
    - NOTE: the OUT path should have a short, direct and wide trace. Consider using a 10V or 16V rating.
- We have the FB pin (feedback) connected to the tap of a resistor divider from the output to GND,
which is used to control the output voltage. The regulator adjusts the duty cycle to keep voltage
at the FB pin at the internal reference voltage (Vref), 0.798V. Since we want 3.3V, we need a
divider which leads to 0.8V at the tap when output is 3.3V. We use a 75k and a 24k resistor:
`VOUT = 0.8 × (1 + 75k/24k) ≈ 3.3V`.
    - Note: the resistor divider should be placed as close to FB as possible, any noise on the
    trace can lead to incorrect adjustment of output voltage. Vias should not be placed on the
    FB traces.

Note: the PGND, IN and OUT paths should have short, direct and wide traces. The IN capacitor and
connection should be as short and wide as possible.

![alt text](image.png)

## Microcontroller

### STM32F303K8T6

Hardware development documentation (an4206):
https://www.st.com/resource/en/application_note/an4206-getting-started-with-stm32f3-series-hardware-development-stmicroelectronics.pdf

Datasheet: https://www.st.com/resource/en/datasheet/stm32f303c6.pdf

The microcontroller takes its power from the 3.3V rail, supplied by the voltage regulator. It
reads all the sensors, and controls the motors via the motor controllers.

The hardware development documentation recommends:
- A multilayer PCB with a separate layer dedicated to ground, and another layer detected to supply
(VDD), to provide good decoupling and a good shielding effect.
- The PCB layout should have separate circuits for:
    - High-current circuits
    - Low-voltage circuits
    - Digital component circuits
    - Circuits separated according to their EMI contribution. This will reduce cross-coupling
    on the PCB that introduces noise
- Unused I/O, clocks or counters should not be left free, they should be pulled high or low. This
can be done through software by configuring the pins as GPIO output.

NOTE: for more detail, see the hardware development documentation section 5, where it also
elaborates on which decoupling capacitors should be used and where.

For support components, we don't use an external oscillator, so we just need decoupling capacitors.
- For both VDD/VSS pairs, we have 4.7uF (X5R, 10V) and 0.1uF (X7R, 10V) ceramic 0805 decoupling capacitors.
    - NOTE: these should be placed as close as possible to the VDD/VSS pairs, with the smaller one
    right between them. See image of this in the an4206 doc.
- For VDDA, we have 0.1uF (X7R, 10V) and 1uF (X5R, 10V)  ceramic 0805 decoupling capacitors, placed
as close to the VDDA as possible,between VDDA and nearest VSS.

On this MCU, the BOOT0 pin should not float, and to boot from flash you need to pull the BOOT0 pin low.
Therefore, we pull it low with a 10k resistor by default, but we also add an open 2 pad solder jumper,
so that we can bridge BOOT0 to 3v3 if we need to boot with a bootloader. For more details on that, see
section 3 of AN4206.

## Motors

We have one TB6612FNG motor controller, controlling two brushed DC motors, one on either side.

The motor driver has a max continuous current rating of 1.2A per channel, meaning per motor. It supports
up to 2A for up to 20ms pulses at <=20% duty cycle, and up to 3.2A absolute peak for single 10ms
pulses.

### TB6612FNG

Datasheet: https://cdn.sparkfun.com/datasheets/Robotics/TB6612FNG.pdf

The TB6612FNG is implemented as documented in the typical application diagram of the datasheet,
page 7.

- We have a 10k pull-up resistor from STBY to VCC, to always pull STBY high, which enables the IC,
which saves us having to use a GPIO output pin to enable it from our MCU.
- We have one decoupling capacitor on the VCC pin, 0.1uF ceramic, to reduce noise from the voltage
regulator, primarily due to fast and transient current demands of control logic, as well as trace
inductance.
- We have two decoupling capacitors on the VM pins, 0.1uF and 10uF ceramic, which takes
input voltage from the batteries directly. The motor is an inductive load which causes large, fast
current spikes when switching, and it can also feed noise back towards the battery. The capacitors
can supply the increased demand when current spikes, or absorb
[back-EMF](https://en.wikipedia.org/wiki/Counter-electromotive_force) spikes when the motor
generates them: when the PWM switches it off, if the motor is driven backwards by a pushing
force or any other condition where the motor shaft is spun externally, turning it into a generator.
The TB6612FNG has built-in flyback diodes to protect the IC from this phenomenon, which redirect
spikes in current back into the supply rail, where the capacitors can absorb it.
    - The datasheet recommends electrolytic for the 10uF cap, but it seems like
    overkill, it's big, and looking at prior art, adafruit uses ceramic for their breakout board, and
    sparkfun uses polymer.

NOTE: all capacitors should be placed as close to the IC as possible.

## Sensors

The final robot will have four analog line sensors, one digital IR receiver and three digital
time-of-flight distance sensors. All of them will connect to the 3.3V power rail supplied by
the voltage regulator. They will not mount directly to the PCB, the PCB will only have connectors
for them.

### QRE1113 analog line sensors

Datasheet: https://cdn.sparkfun.com/datasheets/Sensors/Proximity/QRE1113.pdf
Breakout board schematic: https://cdn.sparkfun.com/datasheets/Sensors/Infrared/QRE1113%20Line%20Sensor%20Breakout%20-%20Analog.pdf

For each sensor we have a three pin connector, which connects to the external breakout board with
the sensors, which has the required supporting components. On the schematic, we just have some
components to support the LEDs.

- We connect a LED in series with a 330Ohm resistor from VCC to OUT. When no line is detected, the 
resistance in the sensor is low, and so the output is high. When a line is detected, the resistance
is increased, and so the output is lower. When the output is lower, there is a voltage difference
across the LED, which should be enough to light it, giving us a visual indication of when the sensor
detects a line.

### TSOP38238 IR receiver

Datasheet: https://cdn.sparkfun.com/assets/c/8/5/c/8/tsop382.pdf

The TSOP38238 is implemented as documented in the typical application diagram of the datasheet,
page 1.

- We have a decoupling 1uF capacitor on Vs, to reduce input noise.
- We have a 470Ohm resistor on Vs, for protection against EOS, meaning protection against
transient spikes of high currents or voltages beyond what the device is rated for. The
datasheet recommends a resistor that is 33 Ω < R1 < 1 kΩ.
- We have a LED in series with a 4.7k Ohm resistor between OUT and Vs. Since OUT is pulled high,
and then pulled low in pulses when receiving a signal, there will only be a voltage difference
across the LED when a signal is received, giving us a visual indication when a signal is received.
We use a larger resistor, since it forms a voltage divider with the resistance of the sensor, and
we connect the tap of the divider, which is the sensor output, to the MCU. With a small resistor,
like the 330 ohm we started with, we only got a range of 3.3v to 1.5V at the tap. With 4.7k, we get
down to 266mV minimum, and so we retain most of the range. With no led and resistor we got down to
about 180mV. TODO: explain this better, capture images from scope.

## PCB

Looking at JLPCB for manufacturing, their PCB capabilities are documented in
https://jlcpcb.com/capabilities/pcb-capabilities.

For multi-layer:
- Minimum trace width is 3.5 mil
- Minimum trace spacing is 3.5 mil.
- Minimum track to pad clearance is 0.1mm, but higher is recommended.
- Minimum SMD pad-to-pad clearance is 0.15mm.
- Minimum via hole to track is 0.2mm.

To be conservative, I will start with 6 mil anyway.

### PCB layers

We will go for a 4 layer board with a GND plane and a 3.3V power plane for design simplicity.

F.Cu — components, signal traces and power traces
In1.Cu — GND plane
In2.Cu — 3.3V power plane
B.Cu — signal routing overflow
