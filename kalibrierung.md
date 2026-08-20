# Kalibrierung der H&B-Uhr

Die Firmware kalibriert die beiden Dreheisenmesswerke über 13 PWM-Stützpunkte je Instrument. Die Werte werden dauerhaft im Flash des ESP32 gespeichert.

## Aktuelle Pinbelegung

| Funktion             | GPIO |
| -------------------- | ---: |
| Minutenmesswerk, PWM |   14 |
| Stundenmesswerk, PWM |   13 |
| Encoder A            |   26 |
| Encoder B            |   25 |
| Encoder-Taster       |   27 |

Die PWM arbeitet mit 500 Hz und 10 Bit. Der gültige Bereich ist `0` bis `1023`. Die elektrische Beschaltung mit MOSFETs, Vorwiderständen und Freilaufdioden muss vor der Kalibrierung geprüft werden.

## Vorbereitung

1. Firmware mit PlatformIO auf den ESP32 übertragen.
2. Seriellen Monitor mit 115200 Baud öffnen.
3. Weboberfläche über die IP-Adresse des ESP32 öffnen. Wenn kein konfiguriertes WLAN erreichbar ist, verbindet sich der Rechner mit dem Access Point `HB-Uhr-Setup`; Passwort: `hb-uhr-setup`.
4. Die Zeiger niemals gegen einen mechanischen Anschlag fahren. Den PWM-Wert vorsichtig erhöhen und den maximal sicheren Wert notieren.

## Encoder-Ablauf

Die Kalibrierung wird im Webinterface mit **Minuten kalibrieren** gestartet. Der Encoder kann danach ohne Weboberfläche verwendet werden.

### Minutenmesswerk

1. Die Firmware startet bei Minute 0.
2. Drehen verändert den PWM-Wert live.
3. Kurzer Tastendruck speichert den Wert und wechselt zum nächsten Punkt.
4. Die Punkte sind `0, 5, 10, ..., 60` Minuten. Der Punkt 60 wird separat gespeichert.
5. Nach dem Speichern von Punkt 60 einen langen Tastendruck ausführen, um zur Stundenkalibrierung zu wechseln.

### Stundenmesswerk

1. Die Firmware startet bei Stunde 0.
2. Drehen verändert den PWM-Wert des Stundenmesswerks live.
3. Kurzer Tastendruck speichert den Wert und wechselt zum nächsten Punkt.
4. Die Punkte sind `0, 1, 2, ..., 12` Uhr.
5. Nach dem Speichern von Punkt 12 wechselt die Firmware automatisch in den Normalbetrieb. Ein langer Tastendruck kann die Stundenkalibrierung vorher beenden.

Die Werte werden bei jedem kurzen Tastendruck in den Flash geschrieben. Die Weboberfläche bietet zusätzlich einen gemeinsamen Speichern-Befehl.

## Normalbetrieb

Die Dummy-Uhrzeit wird im Webinterface gesetzt und bleibt danach stehen. Die Minutenanzeige wird für jede Minute neu berechnet:

- Minute 0 bis 5 liegt zwischen den Punkten 0 und 5.
- Minute 1 bis 4 wird linear zwischen diesen Punkten interpoliert.
- Minute 59 liegt zwischen den Punkten 55 und 60.

Der Stundenzeiger wird nur in 15-Minuten-Schritten aktualisiert. Die Position zwischen zwei vollen Stunden wird linear interpoliert. Beispiel: Bei `14:30` wird zwischen dem kalibrierten Wert für 2 Uhr und dem für 3 Uhr die Hälfte ausgegeben. Für den Übergang `12:00` zu `13:00` werden Punkt 12 und Punkt 1 verwendet.

## Weboberfläche

Die Oberfläche enthält:

- Umschaltung zwischen Normalbetrieb, Minuten- und Stundenkalibrierung
- Einstellung der Dummy-Uhrzeit
- Tabelle aller 13 Minuten- und 13 Stundenpunkte
- Live-Test eines beliebigen PWM-Wertes
- Live-Test einzelner Tabellenpunkte
- dauerhaftes Speichern aller Tabellenwerte

Die Tabellenwerte werden erst durch den Speichern-Befehl dauerhaft übernommen. Ein Live-Test verändert den Wert zunächst nur im RAM.

## API-Beispiele

```text
GET  /api/state
POST /api/state              {"instrument":1,"pwm":450}
POST /api/mode               {"mode":1}
POST /api/time               {"hour":14,"minute":30}
POST /api/calibration        {"instrument":1,"index":3,"pwm":450,"persist":false}
POST /api/calibration/save   {}
```

`mode` ist `0` für Normalbetrieb, `1` für Minutenkalibrierung und `2` für Stundenkalibrierung. PWM-Werte, Indizes und Uhrzeiten werden serverseitig validiert.
