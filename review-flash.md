# Code-Review: Hartmann & Braun Dreheisen-Uhr (H-B-Uhr)

Dieses Code-Review analysiert das Projekt zur Steuerung der Hartmann & Braun Dreheisenmesswerk-Uhr basierend auf einem ESP32. Das System verwendet ein interessantes Konzept, bei dem die physikalischen Zeiger-Nichtlinearitäten von Dreheisenmesswerken durch eine mehrpunktige Kalibrierung (13 Stützpunkte für Stunden und Minuten) in Software ausgeglichen werden.

---

## 1. Gesamtarchitektur & Struktur
### Stärken
* **Klare Trennung der Zuständigkeiten (Separation of Concerns):**
  * `main.cpp` steuert die Hardware, den Encoder, die Kalibrierungslogik und die WLAN-Verbindung.
  * `web_server.cpp` / `web_server.h` kapselt das Web-Interface und die REST-API vollständig.
  * Die Kommunikation erfolgt sauber über Funktionszeiger/Callbacks (`WebStateReader`, `WebPwmWriter`, etc.). Es gibt keine zyklischen Abhängigkeiten oder unsauberen globalen Verweise zwischen den beiden Modulen.
* **Moderne ESP32-APIs:**
  * Verwendung von `ledcAttach` (neuere ESP32-Arduino-Core-APIs ab v3.x) statt der veralteten `ledcSetup`/`ledcAttachPin`-Funktionen.
  * Nutzung der eingebauten `Preferences`-Bibliothek für das Speichern von Daten im Flash-Speicher (NVS) anstelle von veraltetem EEPROM.
* **Robuste WLAN-Verbindung:**
  * Automatischer Fallback auf einen Access Point (`HB-Uhr-Setup`), falls keine Verbindung zum konfigurierten WLAN hergestellt werden kann oder kein SSID-Name angegeben wurde.
  * Nicht-blockierender Verbindungsaufbau.

---

## 2. Detaillierte Analyse: `main.cpp`

### 2.1 Stärken & Best Practices
* **Konsistente Typisierung & Konstanten:**
  * Verwendung von `constexpr` für Pinbelegungen, Intervalle und Schwellenwerte.
  * Verwendung von stark typisierten Enums (`enum class OperatingMode : uint8_t`).
* **Lineare Interpolation:**
  * Die Funktion `interpolate()` rechnet mathematisch korrekt und nutzt `int32_t` für Zwischenrechnungen, um Overflows bei Berechnungen zu vermeiden. Sie fängt eine Division durch Null ab.
* **Interrupt-Handling (ISR):**
  * Der Encoder-Interrupt `readEncoderISR()` ist korrekt mit `IRAM_ATTR` deklariert, was für eine schnelle Ausführung im RAM sorgt.
* **Kalibrierungs-Validierung:**
  * Beim Laden der Kalibrierung aus dem NVS wird geprüft, ob die Versionsnummer und die Länge des Bytestroms mit den erwarteten Werten übereinstimmen. Zudem wird jeder einzelne PWM-Wert auf Gültigkeit geprüft (`validPwm()`).

### 2.2 Verbesserungspotenziale & Risiken

#### A. Zeitmessung / Uhrwerk-Logik fehlt komplett ⚠️
* **Problem:** In `main.cpp` existieren zwar `dummyHour` und `dummyMinute`, diese werden aber im Code **niemals automatisch hochgezählt**. Die Uhr zeigt somit permanent die im Webserver gesetzte statische Uhrzeit an.
* **Lösung:** 
  1. Integration der ESP32-internen Zeitsteuerung via NTP (Network Time Protocol) über `<time.h>`:
     ```cpp
     configTzTime("CET-1CEST,M3.5.0,M10.5.0/3", "pool.ntp.org"); // Automatische Sommerzeit für DE
     ```
  2. Auslesen der Echtzeit im `loop()` in regelmäßigen Abständen (z. B. jede Sekunde) und Aktualisierung von `dummyHour` und `dummyMinute`.

#### B. Redundantes Polling & Entprellen des Encoders im Haupt-Loop
* **Problem:** Der Encoder-Taster (`ENCODER_SW_PIN`) wird im `loop()` mittels Software-Debounce über `millis()` abgefragt. Wenn der Loop durch Netzwerkaktivitäten, langsame HTTP-Anfragen oder serielle Prints blockiert wird (auch wenn diese meist asynchron laufen), kann die Tastenerkennung träge wirken.
* **Lösung:** Auch den Button an einen Interrupt binden oder eine bewährte Bibliothek wie `EasyButton` oder `OneButton` nutzen, um Klicks, Doppelklicks und lange Tastendrücke ereignisgesteuert abzufangen.

#### C. Skalierbarkeit & Globale Variablen
* **Problem:** Es gibt sehr viele globale Variablen im globalen Namespace (`calibration`, `operatingMode`, `dummyHour`, `dummyMinute` etc.).
* **Lösung:** Kapselung der Zustände in einer Klasse `ClockController` oder zumindest in einem Namespace, um Namenskollisionen zu vermeiden und die Testbarkeit zu erhöhen.

---

## 3. Detaillierte Analyse: `web_server.cpp`

### 3.1 Stärken & Best Practices
* **Speichereffizienz (PROGMEM):**
  * Das gesamte HTML/CSS/JS-Webinterface ist über das `PROGMEM`-Makro im Flash-Speicher hinterlegt. Dadurch bleibt der wertvolle SRAM des ESP32 frei.
* **Kompaktes Responsive Design:**
  * Das Web-Interface ist modern und responsiv gestaltet. Es nutzt CSS-Variablen für ein konsistentes Dark-Theme und Media-Queries für mobile Endgeräte.
* **JSON-REST-Schnittstelle:**
  * Saubere Verwendung von `ArduinoJson` (v7) für die Kommunikation zwischen Web-Interface und ESP32.

### 3.2 Verbesserungspotenziale & Risiken

#### A. Speicherverbrauch von ArduinoJson
* **Sicherheit:** `JsonDocument` in ArduinoJson v7 verwaltet den Speicher dynamisch auf dem Stack/Heap. In `sendState()` wird `JsonDocument status;` deklariert. Da die Größe nicht mehr vorab fest definiert werden muss (wie bei `StaticJsonDocument` in v6), ist dies zwar komfortabler, bei hohem Durchsatz sollte jedoch auf Speicher-Fragmentierung geachtet werden.
* **Optimierung:** Bei sehr knappen RAM-Ressourcen könnte man die JSON-Antwort auch manuell oder über einen vordefinierten Stream schreiben, dies ist hier bei einem ESP32 mit 520 KB SRAM jedoch unkritisch.

#### B. Fehlende Absicherung des Access Points
* **Problem:** Der Access Point startet mit der SSID `HB-Uhr-Setup` und dem Passwort `hb-uhr-setup`. Dies ist im Quellcode hartcodiert.
* **Lösung:** Ermögliche das Konfigurieren der AP-Sicherheitsdaten oder generiere ein einzigartiges Passwort basierend auf der MAC-Adresse des ESP32 (z. B. `HB-Uhr-[Last3Bytes]`).

---

## 4. Architektonische Empfehlungen & Roadmap

| Priorität  | Thema            | Beschreibung                                                      | Nutzen                                                                   |
| :--------- | :--------------- | :---------------------------------------------------------------- | :----------------------------------------------------------------------- |
| **Hoch**   | NTP-Integration  | Einbau von NTP-Client zur automatischen Zeitsynchronisation.      | Macht aus dem statischen Demogerät eine echte, funktionierende Uhr.      |
| **Mittel** | Button-Interrupt | Auslagern des Encoder-Buttons in einen Interrupt.                 | Verhindert verpasste Klicks während hoher WLAN-/Webserver-Auslastung.    |
| **Mittel** | OTA-Updates      | Integration von `ArduinoOTA` in `platformio.ini` und Code.        | Ermöglicht kabelloses Flashen von neuer Firmware im eingebauten Zustand. |
| **Gering** | Code-Kapselung   | Strukturierung der globalen Variablen in Namespaces oder Klassen. | Verbessert die Wartbarkeit und Lesbarkeit des Codes.                     |

---
*Review erstellt durch GitHub Copilot (Modell: Gemini 3.5 Flash) am 18. August 2026.*
