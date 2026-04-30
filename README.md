# Smart Junkers Gas Boiler

> Auf Deutsch lesen: [README.de.md](README.de.md)

Documentation of how I made my old **Junkers/Bosch gas boiler** smart by
replacing the **Junkers TRQ 21** modulating room thermostat with a
**Shelly Plus Uni** on the **1-2-4 interface**. Four heating
levels (Off / Low / Medium / High), room temperature via **DS18B20**,
ready to integrate with **Home Assistant**, MQTT or the Shelly Cloud.

Compatible with older Junkers/Bosch boilers that use the 1-2-4 interface
– e.g. **ZWR**, **ZSBE**, **ZWN**, **ZWA** series.

> **Disclaimer:** This is my personal documentation, not a build-along
> tutorial. If you mess with your boiler, you should know what you're
> doing – recreate at your own risk.

## Goal: 4 heating levels with frost-protection failsafe

Instead of the modulating control of the TRQ 21, the Shelly Plus Uni is
used to implement **four discrete heating levels** – **High**, **Medium**,
**Low** and **Off**. The circuit is deliberately wired so that if the
Shelly fails (unpowered, broken) the boiler **keeps heating at full
power** instead of going off: frost damage to heating and water pipes is
much more expensive than temporary overheating, which is limited by the
radiator thermostatic valves anyway.

The two switchable outputs of the Shelly are used in all four
combinations – active pulldowns between terminal **2** (signal) and
terminal **4** (GND) pull the voltage down step by step from the
failsafe full-load level:

| Level      | Output 1 | Output 2 | active pulldown between 2 ↔ 4 |
| ---------- | :------: | :------: | ----------------------------- |
| **High**   |   off    |   off    | none (terminal 2 floating)    |
| **Medium** |    on    |   off    | 5.1 kΩ                        |
| **Low**    |   off    |    on    | 4 kΩ (2 kΩ + 2 kΩ in series)  |
| **Off**    |    on    |    on    | 5.1 kΩ ∥ 4 kΩ ≈ 2.22 kΩ       |

Each level corresponds to a fixed voltage at terminal 2 and therefore a
fixed burner power – the matching voltage levels are documented further
down under [Voltages and measurements](#voltages-and-measurements).

## The replaced room thermostat: Junkers TRQ 21

The TRQ 21 is a **modulating room temperature controller** for older Junkers/Bosch
gas boilers. It measures the room temperature, compares it to the
setpoint and tells the boiler via an analog control signal **how much**
to heat.

## The 1-2-4 interface

The TRQ 21 is connected to the boiler via a **three-wire Junkers
interface**. The name comes from the terminal numbers in the wiring
diagram: terminals **1**, **2** and **4** are used (terminal 3 exists on
the strip but is not connected).

| Terminal | Meaning            | Description                                        |
| :------: | ------------------ | -------------------------------------------------- |
|  **1**   | **+24 V DC**       | supply voltage from the boiler to the controller   |
|  **2**   | **control signal** | analog feedback voltage from the controller        |
|  **4**   | **GND / ground**   | common reference ground                            |

### How modulation works

The boiler applies its 24 V supply voltage between terminal 1 and
terminal 4 and measures **what voltage the controller returns on
terminal 2**. This voltage directly controls burner modulation:

- **below ~7 V**: burner **off**
- **up to ~15 V**: burner power increases up to **100 %** (higher voltage  
  produces no additional power)

> **Failsafe behavior:** If no controller is connected – terminal 2 left
> open – the boiler interprets this as a full-load request and heats at
> 100 %. This circuit deliberately leverages that behavior as frost
> protection: if the Shelly fails, the boiler automatically falls back
> to full power.

## Control via Shelly Plus Uni

Instead of a TRQ 21, a **Shelly Plus Uni** now sits at the 1-2-4
interface. The circuit uses no active electronics – the Shelly does not
drive terminal 2 "analog", it only modifies the voltage divider formed
by the boiler's internal pull-up and external pulldown resistors:

- **No permanent pulldown:** terminal 2 is left floating in the idle
  state. The boiler's internal pull-up pulls it to +21 V – so the
  boiler heats at full power on any fault (frost-protection failsafe).
- **Pulldowns via the Shelly:** the two switchable outputs each connect
  a resistor between terminal **2** (signal) and terminal **4** (GND).
  When the outputs are not switched on – or the Shelly is unpowered –
  the pulldowns are automatically disconnected. The smaller the
  effective pulldown, the harder terminal 2 is pulled toward GND – and
  the lower the burner power.

### Voltages and measurements

Voltage at terminal **2** for the four heating levels, measured on my
boiler:

| Pulldown 2 ↔ 4           | Voltage at terminal 2 | Level             | Burner behaviour       |
| ------------------------ | --------------------- | ----------------- | ---------------------- |
| none (floating)          | **21.4 V**            | High (failsafe)   | full power             |
| 5.1 kΩ                   | **10.5 V**            | Medium            | on, modulated          |
| 4 kΩ (2 kΩ + 2 kΩ)       | **9.3 V**             | Low               | on, ignites cold       |
| 2.22 kΩ (5.1 kΩ ∥ 4 kΩ)  | **6.4 V**             | Off               | shuts off              |

The values were measured on my boiler – they may vary slightly
between models.

#### Hysteresis: two different thresholds

Modulating burners typically have a hysteresis – the voltage at which
a **cold** burner ignites is higher than the voltage at which a
**running** burner shuts off. For my boiler the measurements yield:

- **Cut-off threshold:** ~6 V → running burner shuts off
- **Hysteresis zone:** ~6 – 9 V → burner stays on if already running,
  but does **not** restart from the off state
- **Re-ignition threshold:** ~9 V → cold burner ignites reliably

**This drives the choice of R_b:** R_b is deliberately picked large
enough (4 kΩ) so that the Low voltage (9.3 V) sits **above the
re-ignition threshold** – the burner therefore ignites reliably from
the off state even when Low is selected. A smaller R_b such as 3 kΩ
would push Low down to 7.8 V, right into the hysteresis zone: the
burner would stay on if already running but would not restart by
itself from the off state.

#### Characterisation series

Full measurement series:

| Pulldown 2 ↔ 4    | Voltage at terminal 2 | Burner behaviour                |
| ----------------- | --------------------- | ------------------------------- |
| none (open)       | 21.4 V                | on, full power                  |
| 10 kΩ             | 14 V                  | on (modulation saturation zone) |
| 5.1 kΩ            | 10.5 V                | on, modulated                   |
| 4 kΩ              | 9.3 V                 | ignites cold reliably           |
| 3 kΩ              | 7.8 V                 | stays on, won't ignite cold     |
| 2.22 kΩ (5.1 ∥ 4) | 6.4 V                 | shuts off (from running)        |
| 2 kΩ              | 5.9 V                 | shuts off (from running)        |
| 1.88 kΩ (5.1 ∥ 3) | 5.7 V                 | shuts off (from running)        |
| 1 kΩ              | 3.5 V                 | shuts off (from running)        |

### Voltage monitoring

In addition to the two switching outputs, the Shelly Plus Uni has an
**analog voltage input (0–30 V DC)**. This allows logging the voltage
between terminal **2** and terminal **4** directly – i.e. exactly the
signal the boiler modulates with. This makes it possible to verify at
any time which level the boiler is actually in, and to monitor both the
boiler and the circuit remotely.

## Room temperature via DS18B20

With the TRQ 21 gone, its built-in room temperature measurement is also
gone. This job is taken over by a **DS18B20** connected directly to the
Shelly.  
The temperature reading can later be used in an automation to decide
which output should be switched.

The exact wiring is shown in the [schematic](#schematic-and-pinout).

## Schematic and pinout

The complete wiring is shown in [`images/wiring.png`](images/wiring.png)
(editable source: [`images/wiring.drawio`](images/wiring.drawio)):

![Schematic of the Shelly connection to the Junkers 1-2-4 interface](images/wiring.png)

### Connection to the boiler

At the top of the image are the four boiler terminals. Only **1**, **2**
and **4** are used; **terminal 3** is left open.

- **Terminal 1 (24 V)** powers the Shelly via VAC1.
- **Terminal 2 (signal)** is pulled towards terminal 4 (GND) on demand
  through the pulldown resistors switched by the two Shelly outputs.
  The same line also feeds the Shelly's **ANALOG IN** pin for voltage
  measurement.
- **Terminal 4 (GND)** is the common ground for the boiler, the Shelly's
  power supply, the DS18B20 and the switched pulldowns.

### Shelly Plus Uni pinout

The Shelly Plus Uni has a 10-pin connector with color-coded wires. Six
of the ten wires are used in this circuit:

|  Pin  | Label       | Wire color | Use                                                       |
| :---: | ----------- | ---------- | --------------------------------------------------------- |
|   1   | VAC1        | red        | → boiler **terminal 1** (+24 V) – Shelly power            |
|   2   | VAC2        | black      | → boiler **terminal 4** (GND) – Shelly power              |
|   3   | ANALOG IN   | grey       | → boiler **terminal 2** (signal) – voltage measurement    |
|   4   | SENSOR VCC  | yellow     | → DS18B20 **red** wire (sensor power)                     |
|   5   | DATA        | blue       | → DS18B20 **yellow** wire (1-Wire data line)              |
|   6   | +5 VDC      | -          | unused                                                    |
|   7   | GND         | green      | → DS18B20 **black** wire (sensor ground)                  |
|   8   | COUNT IN    | -          | unused                                                    |
|   9   | IN 1        | -          | unused                                                    |
|  10   | IN 2        | -          | unused                                                    |

In addition, the Shelly has **two potential-free relay outputs**. Each
switches its resistor between terminal **2** and terminal **4** (pulldown):

| Output        | switched pulldown between terminal 2 ↔ terminal 4 | Heating level   |
| ------------- | ------------------------------------------------- | --------------- |
| both off      | open (no pulldown)                                | High (failsafe) |
| **OUT 1**     | **5.1 kΩ**                                        | Medium          |
| **OUT 2**     | **4 kΩ** (2 kΩ + 2 kΩ in series)                  | Low             |
| OUT 1 + OUT 2 | 5.1 kΩ ∥ 4 kΩ ≈ 2.22 kΩ                           | Off             |

## 3D-printed cover

To replace the TRQ 21 visually, I modeled a **3D-printable cover** that
clips onto the **existing wall bracket** of the original thermostat – no
new holes. The Shelly Plus Uni and all wiring fit underneath.

![Preview of the 3D-printed cover](3d-printing/cover.png)

The print files live in [`3d-printing/`](3d-printing/):

- [`cover.stl`](3d-printing/cover.stl) – ready-to-slice model
- [`cover.png`](3d-printing/cover.png) – preview render

## Sources and further reading

- [Driving the Junkers 1-2-4 interface – mikrocontroller.net](https://www.mikrocontroller.net/topic/450569) (German)
  – detailed discussion with measurements, voltage and current characteristics
- [Control Junkers gas heating (1-2-4 interface) with Home Assistant – Home Assistant Community](https://community.home-assistant.io/t/control-junkers-gas-heating-1-2-4-interface-with-home-assistant/354658)
  – practical implementation with an ESP8266 (PWM) and opto-coupler
- [Schematic of the Junkers ZSBE 16-3 board – roter-unimog.de](http://www.roter-unimog.de/p4/46-hzg-technik.htm) (German)
  – original schematic of a compatible boiler including terminal layout
- [TRQ 21 W user manual (PDF) – ManualsLib](https://www.manualslib.com/manual/1229445/Junkers-Trq-21-W.html) (German)
  – manufacturer documentation of the original controller
- [KNX user forum: TRQ 21 T and ZWR 18-3/24-3 boiler](https://knx-user-forum.de/forum/%C3%B6ffentlicher-bereich/knx-eib-forum/knx-einsteiger/1132705-raumtemperaturregler-trq-21-t-und-gastherme-zwr-18-3-24-3-von-junkers-bosch) (German)
  – discussion about KNX integration of the same interface
