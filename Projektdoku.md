# Projektdokumentation: Steuerung analoger Messinstrumente via ESP32 PWM

Diese Dokumentation beschreibt die elektrotechnischen Hintergründe, die Dimensionierung der Bauteile und die Inbetriebnahme von zwei unterschiedlichen analogen Messinstrumenten (Hartmann & Braun) mittels ESP32-PWM-Ansteuerung und IRF3708-MOSFETs.

---

## 1. Elektrotechnische Grundlagen & Berechnungen

### 1.1 Ermittlung der Innenwiderstände
Die Innenwiderstände der Spulen ($R_{\text{Spule}}$) wurden anhand der maximalen Betriebsspannung und des maximalen Stroms berechnet:

* **Instrument 1** (40 mA bei 9,2 V):
  $$R_{\text{Spule1}} = \frac{9,2\,\text{V}}{0,04\,\text{A}} = \mathbf{230\,\Omega}$$

* **Instrument 2** (130 mA bei 2,7 V):
  $$R_{\text{Spule2}} = \frac{2,7\,\text{V}}{0,13\,\text{A}} \approx \mathbf{20,77\,\Omega}$$

### 1.2 Vergleich der Regelungseigenschaften
Instrument 1 ermöglicht mit der PWM des ESP32 eine spürbar feinere und präzisere Einstellung als Instrument 2. Dafür gibt es zwei wesentliche Gründe:

#### Das Verhältnis von Induktivität zu Widerstand (Zeitkonstante)
Für eine ruhige Zeigerbewegung ist die Glättung des Stroms durch die Spule entscheidend.
* **Instrument 1** besitzt durch seine dünnere, längere Wicklung eine hohe Induktivität ($L$) und einen hohen Innenwiderstand ($230\,\Omega$). Es verhält sich bei typischen PWM-Frequenzen gutmütig: Der Strom reißt in den Signalpausen nicht abrupt ab, was zu einem stabilen, hochauflösenden Zeigerstand führt.
* **Instrument 2** ist mit $\approx 20,8\,\Omega$ extrem niederohmig. Der Strom schießt bei jedem Puls blitzschnell nach oben und fällt rasant ab (steile Flanken). Dies führt zu unruhigeren Strömen und minimalem Zeigerzittern im Grenzbereich.

#### Der relative Einfluss von Peripheriewiderständen
Der MOSFET IRF3708 wurde gezielt als Logic-Level-MOSFET gewählt, da er bereits bei den 3,3 V des ESP32 voll durchschaltet. Er besitzt einen Einschaltwiderstand ($R_{DS(on)}$) von etwa $0,01\,\Omega$ bis $0,03\,\Omega$.
* **Bei Instrument 1** (Gesamtwiderstand im Kreis ca. $277\,\Omega$) ist dieser MOSFET-Widerstand absolut vernachlässigbar (unter 0,01 % Einfluss). Die Kennlinie hängt rein von der Spule ab.
* **Bei Instrument 2** (Gesamtwiderstand im Kreis nur ca. $23\,\Omega$) fallen minimale Widerstände von Kabeln, Lötstellen oder der Erwärmung des MOSFETs prozentual stark ins Gewicht. Das System wird anfälliger für Schwankungen.

---

## 2. Wahl der PWM-Frequenz

Für den stabilen Betrieb des Dreheisenmesswerks ist die richtige Frequenzwahl entscheidend:

* **Zu niedrige Frequenz (< 100 Hz):** Der mechanische Zeiger versucht den einzelnen Pulsen zu folgen und beginnt sichtbar zu vibrieren.
* **Zu hohe Frequenz (> 5 kHz):** Durch die hohe Induktivität der Spule wird der induktive Blindwiderstand so groß, dass kaum noch Strom fließt. Der Zeiger bewegt sich nicht mehr.
* **Der Sweetspot (200 Hz – 1 kHz):** Diese Frequenz ist hoch genug, damit der mechanisch träge Zeiger ruhig steht, und niedrig genug, um den Stromfluss durch die Spule nicht abzuwürgen. 

*Hinweis: Im Code wird eine 10-Bit-PWM-Auflösung (0 bis 1023 Stufen) genutzt.*

---

## 3. Schaltungsaufbau & Hardware

### 3.1 Spannungsversorgung & Masse (GND)
* **Eingang:** Ein 12V-Netzteil speist den Eingang des LM2596-Spannungsreglers sowie den Vorwiderstand von Instrument 1.
* **LM2596-Ausgang:** Liefert stabile 3,3 V an den ESP32 und den Vorwiderstand von Instrument 2.
* **Gemeinsame Masse (GND):** Alle Minuspole (12V-Netzteil, LM2596-Eingang/Ausgang, ESP32-GND und die Source-Pins der MOSFETs) müssen zwingend miteinander verbunden sein.

### 3.2 Schutzbeschaltung (Gate & Spule)
* **Gate-Serienwiderstand:** Ca. $100\,\Omega$ schützen den ESP32-Pin vor Stromspitzen beim Umschalten (Umladen der Gate-Kapazität).
* **Pull-Down-Widerstand:** Ein $10\,\text{k}\Omega$-Widerstand gegen GND verhindert unkontrolliertes Zeiger-Zucken beim Starten/Flashen des Systems.
* **Induktiver Schutz (Freilaufdioden):** Da es sich bei den Messwerken um Spulen handelt, muss unbedingt parallel zu jedem Instrument eine schnelle Freilaufdiode geschaltet werden (Kathode an Plus, Anode an den MOSFET-Drain). Sie schützt den MOSFET vor zerstörerischen Induktionsspannungen beim Abschalten der PWM-Pulse.

### 3.3 Einkaufs- und Bauteilliste

| Anzahl | Bauteil / Wert                  | Funktion / Beschreibung                                     | Erwartete Verlustleistung |
| :----- | :------------------------------ | :---------------------------------------------------------- | :------------------------ |
| **2x** | Diode (1N4148 oder 1N5819)      | Freilaufdiode für die Spulen (Schutz vor Induktionsspitzen) | -                         |
| **2x** | MOSFET IRF3708                  | N-Channel Logic-Level MOSFET                                | -                         |
| **2x** | $100\,\Omega$ Widerstand        | Gate-Schutz für die MOSFETs (Standard 0,25 W)               | -                         |
| **2x** | $10\,\text{k}\Omega$ Widerstand | Pull-Down gegen GND für definierten Pegel (Standard 0,25 W) | -                         |
| **1x** | $47\,\Omega$ Widerstand         | Vorwiderstand für Instrument 1 (Standard 0,25 W)            | ca. 0,075 W (unkritisch)  |
| **1x** | $2,2\,\Omega$ Widerstand        | Vorwiderstand für Instrument 2 (Standard 0,25 W)            | ca. 0,04 W (sicher)       |

---

## 4. Software-Inbetriebnahme (ESP32-Code)

Nach dem Verbauen der Vorwiderstände muss die Ansteuerung im Code kalibriert werden, um mechanische Beschädigungen an den Instrumenten zu verhindern und Spannungsdifferenzen auszugleichen:

* **Ausschlag begrenzen (Allgemein):** Trage in der Look-Up-Table für den maximalen Ausschlag (100 %) nicht blind den Maximalwert des PWM-Registers (`1023`) ein. Der Zeiger würde sonst mechanisch hart anschlagen.
* **Besonderheit bei Instrument 2:** Da der $2,2\,\Omega$-Vorwiderstand an 3,3 V hardwareseitig etwas zu klein dimensioniert ist (rechnerisch lägen bei 100 % Duty Cycle ca. 3,0 V statt der erlaubten 2,7 V am Instrument an), muss dieser Überschuss rein über die Software abgefangen werden. Der maximale PWM-Wert für Instrument 2 darf im Code den rechnerischen Grenzwert von **ca. 85 % (ca. 870 bei 10-Bit)** nicht überschreiten.
* **Kalibrierungsvorgang:** Taste dich im Code für beide Instrumente langsam von einem PWM-Wert von ca. 800 nach oben vor, bis der Zeiger jeweils exakt auf dem letzten Skalenstrich steht. Dies fängt jede Ungenauigkeit der alten Hartmann & Braun Spulen sowie die Hardware-Toleranzen perfekt ab.


## Pinbelegung (ESP32):
14 Stundenzeiger (Instrument 2)
13 Minutenzeiger (Instrument 1)
21 SCL (I²C) für RTC DS3231
22 SDA (I²C) 
25 Encoder A
26 Encoder B
27 Encoder Button

## WLAN, Weboberflaeche und OTA

Die Firmware verbindet sich im Station-Modus mit einem bestehenden WLAN. Die lokalen
Zugangsdaten werden in einer nicht versionierten `secrets.ini` im Projektverzeichnis
eingetragen. Als Vorlage dient `secrets.ini.example`:

```ini
[secrets]
wifi_ssid = mein-wlan
wifi_password = mein-passwort
ota_hostname = hb-uhr
```

Bei fehlender Verbindung versucht der ESP32 alle 10 Sekunden automatisch erneut zu
verbinden. Der Encoder bleibt dabei lokal bedienbar; Webserver und OTA blockieren den
Hauptloop nicht. Die Weboberflaeche ist unter der IP-Adresse des ESP32 erreichbar.

### JSON-API

`GET /api/state` liefert Instrument, PWM-Werte, Grenzwerte sowie WLAN-Status und IP.
`POST /api/state` akzeptiert beispielsweise `{"instrument":1,"pwm":450}`. Der PWM-Wert
wird auf die jeweilige Instrumentengrenze begrenzt.

### OTA

Nach erfolgreicher WLAN-Verbindung kann die Firmware mit dem in `secrets.ini` gesetzten
Hostnamen oder der IP-Adresse ueber ArduinoOTA hochgeladen werden. PlatformIO kann dafuer
als Upload-Port den Hostnamen verwenden, zum Beispiel:

```ini
upload_protocol = espota
upload_port = hb-uhr.local
```

Alternativ wird die IP-Adresse des ESP32 als `upload_port` gesetzt. Fuer die erste
Inbetriebnahme bleibt der serielle Upload verfuegbar, wenn `upload_protocol` nicht
gesetzt ist.

