# Projektdokumentation: Steuerung analoger Messinstrumente via ESP32 PWM

Diese Dokumentation beschreibt die elektrotechnischen Hintergründe, die Dimensionierung der Bauteile und die Inbetriebnahme von zwei unterschiedlichen analogen Messinstrumenten (Hartmann & Braun) mittels ESP32-PWM-Ansteuerung und IRF3708-MOSFETs.

---

## 1. Elektrotechnische Grundlagen & Berechnungen

### 1.1 Ermittlung der Innenwiderstände
Die Innenwiderstände der Spulen ($R_{\text{Spule}}$) wurden anhand der maximalen Betriebsspannung und des maximalen Stroms berechnet:

* **Minutenmesswerk** (40 mA bei 9,2 V):
  $$R_{\text{Spule, Minuten}} = \frac{9,2\,\text{V}}{0,04\,\text{A}} = \mathbf{230\,\Omega}$$

* **Stundenmesswerk** (130 mA bei 2,7 V):
  $$R_{\text{Spule, Stunden}} = \frac{2,7\,\text{V}}{0,13\,\text{A}} \approx \mathbf{20,77\,\Omega}$$

### 1.2 Vergleich der Regelungseigenschaften
Das Minutenmesswerk ermöglicht mit der PWM des ESP32 eine spürbar feinere und präzisere Einstellung als das Stundenmesswerk. Dafür gibt es zwei wesentliche Gründe:

#### Das Verhältnis von Induktivität zu Widerstand (Zeitkonstante)
Für eine ruhige Zeigerbewegung ist die Glättung des Stroms durch die Spule entscheidend.
* **Das Minutenmesswerk** besitzt durch seine dünnere, längere Wicklung eine hohe Induktivität ($L$) und einen hohen Innenwiderstand ($230\,\Omega$). Es verhält sich bei typischen PWM-Frequenzen gutmütig: Der Strom reißt in den Signalpausen nicht abrupt ab, was zu einem stabilen, hochauflösenden Zeigerstand führt.
* **Das Stundenmesswerk** ist mit $\approx 20,8\,\Omega$ extrem niederohmig. Der Strom schießt bei jedem Puls blitzschnell nach oben und fällt rasant ab (steile Flanken). Dies führt zu unruhigeren Strömen und minimalem Zeigerzittern im Grenzbereich.

#### Der relative Einfluss von Peripheriewiderständen
Der MOSFET IRF3708 wurde gezielt als Logic-Level-MOSFET gewählt, da er bereits bei den 3,3 V des ESP32 voll durchschaltet. Er besitzt einen Einschaltwiderstand ($R_{DS(on)}$) von etwa $0,01\,\Omega$ bis $0,03\,\Omega$.
* **Beim Minutenmesswerk** (Gesamtwiderstand im Kreis ca. $277\,\Omega$) ist dieser MOSFET-Widerstand absolut vernachlässigbar (unter 0,01 % Einfluss). Die Kennlinie hängt rein von der Spule ab.
* **Beim Stundenmesswerk** (Gesamtwiderstand im Kreis nur ca. $23\,\Omega$) fallen minimale Widerstände von Kabeln, Lötstellen oder der Erwärmung des MOSFETs prozentual stark ins Gewicht. Das System wird anfälliger für Schwankungen.

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
* **Eingang:** Ein 12V-Netzteil speist den Eingang des LM2596-Spannungsreglers sowie den Vorwiderstand des Minutenmesswerks.
* **LM2596-Ausgang:** Liefert stabile 3,3 V an den ESP32 und den Vorwiderstand des Stundenmesswerks.
* **Gemeinsame Masse (GND):** Alle Minuspole (12V-Netzteil, LM2596-Eingang/Ausgang, ESP32-GND und die Source-Pins der MOSFETs) müssen zwingend miteinander verbunden sein.

### 3.2 Schutzbeschaltung (Gate & Spule)
* **Gate-Serienwiderstand:** Ca. $100\,\Omega$ schützen den ESP32-Pin vor Stromspitzen beim Umschalten (Umladen der Gate-Kapazität).
* **Pull-Down-Widerstand:** Ein $10\,\text{k}\Omega$-Widerstand gegen GND verhindert unkontrolliertes Zeiger-Zucken beim Starten/Flashen des Systems.
* **Induktiver Schutz (Freilaufdioden):** Da es sich bei den Messwerken um Spulen handelt, muss unbedingt parallel zu jedem Messwerk eine schnelle Freilaufdiode geschaltet werden (Kathode an Plus, Anode an den MOSFET-Drain). Sie schützt den MOSFET vor zerstörerischen Induktionsspannungen beim Abschalten der PWM-Pulse.

### 3.3 Einkaufs- und Bauteilliste

| Anzahl | Bauteil / Wert                  | Funktion / Beschreibung                                     | Erwartete Verlustleistung |
| :----- | :------------------------------ | :---------------------------------------------------------- | :------------------------ |
| **2x** | Diode (1N4148 oder 1N5819)      | Freilaufdiode für die Spulen (Schutz vor Induktionsspitzen) | -                         |
| **2x** | MOSFET IRF3708                  | N-Channel Logic-Level MOSFET                                | -                         |
| **2x** | $100\,\Omega$ Widerstand        | Gate-Schutz für die MOSFETs (Standard 0,25 W)               | -                         |
| **2x** | $10\,\text{k}\Omega$ Widerstand | Pull-Down gegen GND für definierten Pegel (Standard 0,25 W) | -                         |
| **1x** | $47\,\Omega$ Widerstand         | Vorwiderstand für das Minutenmesswerk (Standard 0,25 W)     | ca. 0,075 W (unkritisch)  |
| **1x** | $2,2\,\Omega$ Widerstand        | Vorwiderstand für das Stundenmesswerk (Standard 0,25 W)     | ca. 0,04 W (sicher)       |

---

## 4. Software-Inbetriebnahme (ESP32-Code)

Nach dem Verbauen der Vorwiderstände muss die Ansteuerung im Code kalibriert werden, um mechanische Beschädigungen an den Instrumenten zu verhindern und Spannungsdifferenzen auszugleichen:

* **PWM-Bereich:** Die Firmware verwendet für beide Instrumente den nativen 10-Bit-Bereich von `0` bis `1023`. Die zulässigen Endpunkte werden bei der Kalibrierung vorsichtig ermittelt; ein mechanischer Anschlag ist zu vermeiden.
* **Kalibrierungsvorgang:** Taste dich in der Weboberfläche oder mit dem Encoder für beide Instrumente langsam von einem niedrigen PWM-Wert nach oben vor, bis der Zeiger jeweils exakt auf dem letzten Skalenstrich steht. Dies berücksichtigt die Ungenauigkeiten der Hartmann-&-Braun-Spulen sowie die Hardware-Toleranzen.


## Pinbelegung (ESP32):
14 Minutenzeiger
13 Stundenzeiger
21 SCL (I²C) für RTC DS3231
22 SDA (I²C) 
25 Encoder A
26 Encoder B
27 Encoder Button

## WLAN und Weboberflaeche

Die Firmware verbindet sich im Station-Modus mit einem bestehenden WLAN. Die lokalen
Zugangsdaten werden in einer nicht versionierten `secrets.ini` im Projektverzeichnis
eingetragen. Als Vorlage dient `secrets.ini.example`:

```ini
[secrets]
wifi_ssid = mein-wlan
wifi_password = mein-passwort
```

Bei fehlender Verbindung versucht der ESP32 alle 10 Sekunden automatisch erneut zu
verbinden. Der Encoder bleibt dabei lokal bedienbar; der Webserver blockiert den
Hauptloop nicht. Die Weboberflaeche ist unter der IP-Adresse des ESP32 erreichbar.

### JSON-API

`GET /api/state` liefert die aktive Achse und PWM-Werte. `POST /api/state` akzeptiert
beispielsweise `{"axis":"minutes","pwm":450}`. Der PWM-Wert muss im gemeinsamen
10-Bit-Bereich von `0` bis `1023` liegen.

## Firmware-Architektur

Die Firmware verwendet drei Betriebszustände:

* `0` Normalbetrieb
* `1` Minutenkalibrierung
* `2` Stundenkalibrierung

Die Kalibrierungstabellen enthalten jeweils 13 Werte. Für das Minutenmesswerk
werden die Punkte `0, 5, ..., 60` Minuten gespeichert. Für das Stundenmesswerk
werden die Punkte `0, 1, ..., 12` Uhr gespeichert. Die Werte liegen als
`uint16_t`-Arrays im NVS-Namespace `hb-uhr` und werden beim Start validiert.
Fehlende oder ungültige Werte werden mit `0` initialisiert.

Im Normalbetrieb berechnet die Firmware den Minutenwert per linearer
Interpolation zwischen benachbarten 5-Minuten-Punkten. Der Stundenwert wird für
die Viertelstunden `00`, `15`, `30` und `45` zwischen den benachbarten
Stundenpunkten interpoliert. Die Dummy-Uhrzeit wird über `POST /api/time` gesetzt
und bleibt anschließend unverändert, bis sie erneut gesetzt wird.

## WLAN und Access Point

Zuerst versucht die Firmware, sich mit den Zugangsdaten aus `secrets.ini` im
Station-Modus zu verbinden. Nach einem nicht-blockierenden Timeout von 15
Sekunden startet sie den Access Point `HB-Uhr-Setup` mit dem Passwort
`hb-uhr-setup`. Der Webserver ist dann unter `192.168.4.1` erreichbar.

Die WLAN-Wiederverbindung, der Webserver, die Encoder-Abfrage und die
Zeitberechnung laufen ohne `delay()` in `loop()`.

## Erweiterte JSON-API

`GET /api/state` liefert Betriebsart, Kalibrierindex, Dummy-Uhrzeit, aktuelle
PWM-Werte und beide vollständigen Kalibrierungstabellen.

Weitere Endpunkte:

* `POST /api/mode` mit `{"mode":0}`, `{"mode":1}` oder `{"mode":2}`
* `POST /api/time` mit `{"hour":14,"minute":30}`
* `POST /api/calibration` mit `axis` (`"minutes"` oder `"hours"`), Index, PWM und optionalem `persist`
* `POST /api/calibration/save` zum dauerhaften Speichern aller Werte

