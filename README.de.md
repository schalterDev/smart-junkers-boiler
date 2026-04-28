# Smart Junkers Gastherme

> Read this in English: [README.md](README.md)

Dokumentation der Umrüstung einer alten Junkers-Gastherme auf eine smarte
Steuerung. Anstelle des originalen Raumthermostats **Junkers TRQ 21** wird die 
Therme mit einem Shelly Plus Uni über die **1-2-4-Schnittstelle** angesteuert.

> **Disclaimer:** Das hier ist meine persönliche Doku, kein Nachbau-Tutorial.
> Wer an seiner Heizung schraubt, sollte wissen, was er tut – Nachbau auf
> eigene Gefahr.

## Ziel: 4 Heizstufen über zwei Shelly-Ausgänge

Statt der modulierenden Regelung des TRQ 21 sollen mit dem Shelly Plus Uni
**vier diskrete Heizstufen** abgebildet werden – **Aus**, **Niedrig**,
**Mittel** und **Hoch**. Die zwei schaltbaren Ausgänge des Shelly werden in
allen vier Kombinationen genutzt:

| Stufe       | Ausgang 1 | Ausgang 2 | aktiver Pull-up zwischen 1 ↔ 2 |
| ----------- | :-------: | :-------: | ------------------------------ |
| **Aus**     |    aus    |    aus    | keiner                         |
| **Niedrig** |    ein    |    aus    | 2 kΩ                           |
| **Mittel**  |    aus    |    ein    | 1 kΩ                           |
| **Hoch**    |    ein    |    ein    | 2 kΩ ∥ 1 kΩ ≈ 667 Ω            |

Jede Stufe entspricht damit einem festen Spannungswert an Klemme 2 und
einer entsprechend festen Brennerleistung – die zugehörigen Spannungspegel
sind weiter unten unter [Eigene Messwerte](#eigene-messwerte) dokumentiert.

## Das ersetzte Raumthermostat: Junkers TRQ 21

Der TRQ 21 (Varianten *T* mit Tagesprogramm bzw. *W* mit Wochenprogramm) ist
ein **modulierender Raumtemperaturregler** für ältere Junkers-/Bosch-Gas­thermen.
Er misst die Raumtemperatur, vergleicht sie mit dem Sollwert und gibt der
Therme über eine analoge Stellgröße vor, **wie viel** sie heizen soll.

## Die 1-2-4-Schnittstelle

Der TRQ 21 ist über eine **dreiadrige Junkers-Schnittstelle** mit
der Therme verbunden. Der Name kommt von der Klemmenbezeichnung im
Anschlussplan: belegt sind die Klemmen **1**, **2** und **4** (Klemme 3
existiert auf der Leiste, wird aber nicht verwendet).

| Klemme | Bedeutung       | Beschreibung                                  |
| :----: | --------------- | --------------------------------------------- |
| **1**  | **+24 V DC**    | Versorgungsspannung von der Therme zum Regler |
| **2**  | **Stellsignal** | Analoge Rückspannung vom Regler zur Therme    |
| **4**  | **GND / Masse** | gemeinsame Bezugsmasse                        |

### So wird moduliert

Die Therme legt zwischen Klemme 1 und Klemme 4 ihre 24 V Versorgungsspannung
an und misst, **welche Spannung der Regler an Klemme 2 zurückgibt**. Diese
Spannung steuert direkt die Brennermodulation:

- **unter ~5 V**: Brenner **aus**
- **bis ~15 V**: Brennerleistung steigt bis **100 %** (höhere Spannung bringt keine  
  zusätzliche Leistung)

> **Sicherheits- bzw. Failsafe-Verhalten:** Ist gar kein Regler angeschlossen
> – Klemme 2 also offen –, interpretiert die Therme das als Volllast­anforderung
> und heizt mit 100 %. 

## Ansteuerung mit Shelly Plus Uni

Statt eines TRQ 21 sitzt jetzt ein **Shelly Plus Uni** an der
1-2-4-Schnittstelle. Die Schaltung kommt ganz ohne aktive Elektronik aus –
der Shelly steuert nichts „analog" an Klemme 2, sondern verändert nur den
Spannungsteiler aus dem internen Pull-up der Therme und externen
Widerständen:

- **Sicherheits-Pulldown (dauerhaft):** 1 kΩ zwischen Klemme **2** und
  Klemme **4**. Damit liegen ohne weiteren Eingriff nur ca. **3,5 V** an
  Klemme 2 – die Therme bleibt also aus, wenn der Shelly stromlos ist oder
  keinen Ausgang schaltet.
- **Pull-up über den Shelly:** Die beiden schaltbaren Ausgänge des Shelly
  Plus Uni legen jeweils einen Widerstand zwischen Klemme **1** (+24 V) und
  Klemme **2**. Je kleiner dieser Widerstand, desto höher die Spannung an
  Klemme 2 – und damit die Brennerleistung.

### Eigene Messwerte

Spannung zwischen Klemme **2** und Klemme **4**, bei dauerhaft gestecktem
1 kΩ-Pulldown zwischen 2 und 4:

| Pull-up zwischen 1 ↔ 2 | Spannung an Klemme 2 |
| ---------------------- | -------------------- |
| keiner                 | **~3,5 V**           |
| 5 kΩ                   | **6,4 V**            |
| 2 kΩ                   | **9,5 V**            |
| 1 kΩ                   | **13 V**             |
| 667 Ω (2 kΩ ∥ 1 kΩ)    | **14,9 V**           |
| 330 Ω                  | **18,2 V**           |

### Spannungsüberwachung

Der Shelly Plus Uni hat zusätzlich zu den beiden Schaltausgängen einen
**analogen Spannungseingang (0–30 V DC)**. Damit lässt sich die Spannung
zwischen Klemme **2** und Klemme **4** direkt mitloggen – also genau das
Signal, mit dem die Therme moduliert. So kann jederzeit überprüft werden,
in welcher Stufe sich die Therme tatsächlich befindet, und sowohl die
Therme als auch die eigene Schaltung lassen sich aus der Ferne überwachen.

## Raumtemperatur per DS18B20

Mit dem Wegfall des TRQ 21 entfällt auch dessen interne
Raumtemperatur­messung. Diese Aufgabe übernimmt ein **DS18B20**, der direkt am Shelly hängt.  
Diese Temperatur kann später in einer Automatisierung verwendet werden, um zu entscheiden,
welcher Ausgang geschaltet werden soll.

Die exakte Verdrahtung ist im [Schaltplan](#schaltplan-und-pinbelegung)
zu sehen.

## Schaltplan und Pinbelegung

Die komplette Verdrahtung ist in [`images/wiring.png`](images/wiring.png)
dargestellt (bearbeitbare Quelle: [`images/wiring.drawio`](images/wiring.drawio)):

![Schaltplan der Shelly-Anbindung an die Junkers-1-2-4-Schnittstelle](images/wiring.png)

### Anschluss an die Therme

Oben im Bild liegen die vier Klemmen der Therme. Genutzt werden nur **1**,
**2** und **4**; **Klemme 3** bleibt frei.

- **Klemme 1 (24 V)** versorgt sowohl den Shelly (über VAC1) als auch –
  über die Relais OUT 1 / OUT 2 – die Pull-up-Widerstände nach Klemme 2.
- **Klemme 2 (Signal)** ist über einen festen **1 kΩ-Pulldown** mit
  Klemme 4 verbunden und wird über die Shelly-Ausgänge bei Bedarf nach
  +24 V gezogen. Die gleiche Leitung geht zusätzlich auf den Shelly-Pin
  **ANALOG IN** zur Spannungsmessung.
- **Klemme 4 (GND)** ist die gemeinsame Masse für Therme, Shelly-Versorgung
  und DS18B20.

### Pinbelegung Shelly Plus Uni

Der Shelly Plus Uni hat einen 10-poligen Stecker mit farbcodierten Adern.
In dieser Schaltung sind sechs der zehn Adern belegt:

|  Pin  | Bezeichnung | Aderfarbe | Verwendung                                        |
| :---: | ----------- | --------- | ------------------------------------------------- |
|   1   | VAC1        | rot       | → Therme **Klemme 1** (+24 V) – Shelly-Versorgung |
|   2   | VAC2        | schwarz   | → Therme **Klemme 4** (GND) – Shelly-Versorgung   |
|   3   | ANALOG IN   | grau      | → Therme **Klemme 2** (Signal) – Spannungsmessung |
|   4   | SENSOR VCC  | gelb      | → DS18B20 **rote** Ader (Sensor-Versorgung)       |
|   5   | DATA        | blau      | → DS18B20 **gelbe** Ader (1-Wire Datenleitung)    |
|   6   | +5 VDC      | -         | unbenutzt                                         |
|   7   | GND         | grün      | → DS18B20 **schwarze** Ader (Sensor-Masse)        |
|   8   | COUNT IN    | -         | unbenutzt                                         |
|   9   | IN 1        | -         | unbenutzt                                         |
|  10   | IN 2        | -         | unbenutzt                                         |

Zusätzlich hat der Shelly **zwei potentialfreie Relaisausgänge**:

| Ausgang       | geschalteter Pull-up zwischen Klemme 1 ↔ Klemme 2 | Heizstufe |
| ------------- | ------------------------------------------------- | --------- |
| **OUT 1**     | **2 kΩ**                                          | Niedrig   |
| **OUT 2**     | **1 kΩ**                                          | Mittel    |
| OUT 1 + OUT 2 | 2 kΩ ∥ 1 kΩ ≈ 667 Ω                               | Hoch      |

## Quellen und weiterführende Informationen

- [Ansteuerung Junkers 1-2-4-Schnittstelle – mikrocontroller.net](https://www.mikrocontroller.net/topic/450569)
  – ausführliche Diskussion mit Messwerten, Spannungs- und Stromkennlinien
- [Control Junkers gas heating (1-2-4 interface) with Home Assistant – Home Assistant Community](https://community.home-assistant.io/t/control-junkers-gas-heating-1-2-4-interface-with-home-assistant/354658)
  – praktische Umsetzung per ESP8266/PWM und Optokoppler
- [Schaltplan Platine Junkers ZSBE 16-3 – roter-unimog.de](http://www.roter-unimog.de/p4/46-hzg-technik.htm)
  – Original-Schaltplan einer kompatiblen Therme inkl. Klemmenbelegung
- [Bedienungsanleitung TRQ 21 W (PDF) – ManualsLib](https://www.manualslib.com/manual/1229445/Junkers-Trq-21-W.html)
  – Hersteller­dokumentation des Original­reglers
- [KNX-User-Forum: TRQ 21 T und Gastherme ZWR 18-3/24-3](https://knx-user-forum.de/forum/%C3%B6ffentlicher-bereich/knx-eib-forum/knx-einsteiger/1132705-raumtemperaturregler-trq-21-t-und-gastherme-zwr-18-3-24-3-von-junkers-bosch)
  – Diskussion zur KNX-Anbindung der gleichen Schnittstelle
