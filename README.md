# Smart Junkers Boiler

> Auf Deutsch lesen: [README.de.md](README.de.md)

Documentation of converting an old Junkers gas boiler to a smart control.
Instead of the original **Junkers TRQ 21** room thermostat, the boiler is
driven by a Shelly Plus Uni via the **1-2-4 interface**.

> **Disclaimer:** This is my personal documentation, not a build-along
> tutorial. If you mess with your boiler, you should know what you're
> doing – recreate at your own risk.

## Goal: 4 heating levels via two Shelly outputs

Instead of the modulating control of the TRQ 21, the Shelly Plus Uni is
used to implement **four discrete heating levels** – **Off**, **Low**,
**Medium** and **High**. The two switchable outputs of the Shelly are
used in all four combinations:

| Level      | Output 1 | Output 2 | active pull-up between 1 ↔ 2 |
| ---------- | :------: | :------: | ---------------------------- |
| **Off**    |   off    |   off    | none                         |
| **Low**    |    on    |   off    | 2 kΩ                         |
| **Medium** |   off    |    on    | 1 kΩ                         |
| **High**   |    on    |    on    | 2 kΩ ∥ 1 kΩ ≈ 667 Ω          |

Each level corresponds to a fixed voltage at terminal 2 and therefore a
fixed burner power – the matching voltage levels are documented further
down under [My measurements](#my-measurements).

## The replaced room thermostat: Junkers TRQ 21

The TRQ 21 (variant *T* with a daily program, *W* with a weekly program)
is a **modulating room temperature controller** for older Junkers/Bosch
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

- **below ~5 V**: burner **off**
- **up to ~15 V**: burner power increases up to **100 %** (higher voltage  
  produces no additional power)

> **Failsafe behavior:** If no controller is connected – terminal 2 left
> open – the boiler interprets this as a full-load request and heats at
> 100 %.

## Control via Shelly Plus Uni

Instead of a TRQ 21, a **Shelly Plus Uni** now sits at the 1-2-4
interface. The circuit uses no active electronics – the Shelly does not
drive terminal 2 "analog", it only modifies the voltage divider formed
by the boiler's internal pull-up and external resistors:

- **Failsafe pulldown (permanent):** 1 kΩ between terminal **2** and
  terminal **4**. With nothing else attached, this leaves only about
  **3.5 V** on terminal 2 – so the boiler stays off when the Shelly is
  unpowered or has no output switched on.
- **Pull-up via the Shelly:** the two switchable outputs of the Shelly
  Plus Uni each connect a resistor between terminal **1** (+24 V) and
  terminal **2**. The smaller the resistor, the higher the voltage on
  terminal 2 – and the higher the burner power.

### My measurements

Voltage between terminal **2** and terminal **4**, with the 1 kΩ pulldown
between 2 and 4 permanently in place:

| Pull-up between 1 ↔ 2 | Voltage at terminal 2 |
| --------------------- | --------------------- |
| none                  | **~3.5 V**            |
| 5 kΩ                  | **6.4 V**             |
| 2 kΩ                  | **9.5 V**             |
| 1 kΩ                  | **13 V**              |
| 667 Ω (2 kΩ ∥ 1 kΩ)   | **14.9 V**            |
| 330 Ω                 | **18.2 V**            |

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

- **Terminal 1 (24 V)** powers both the Shelly (via VAC1) and – through
  the OUT 1 / OUT 2 relays – the pull-up resistors going to terminal 2.
- **Terminal 2 (signal)** is tied to terminal 4 via a fixed
  **1 kΩ pulldown** and is pulled up towards +24 V via the Shelly outputs
  on demand. The same line also feeds the Shelly's **ANALOG IN** pin for
  voltage measurement.
- **Terminal 4 (GND)** is the common ground for the boiler, the Shelly's
  power supply and the DS18B20.

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

In addition, the Shelly has **two potential-free relay outputs**:

| Output        | switched pull-up between terminal 1 ↔ terminal 2 | Heating level |
| ------------- | ------------------------------------------------ | ------------- |
| **OUT 1**     | **2 kΩ**                                         | Low           |
| **OUT 2**     | **1 kΩ**                                         | Medium        |
| OUT 1 + OUT 2 | 2 kΩ ∥ 1 kΩ ≈ 667 Ω                              | High          |

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
