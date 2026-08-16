```cpp
#include <Arduino.h>#include "AiEsp32RotaryEncoder.h"
// PIN-Definitionen für die Instrumente (IRF3708 Gates)const int PIN_INST_1 = 18;  // Instrument 1 (Minuten / 12V Schiene)const int PIN_INST_2 = 19;  // Instrument 2 (Stunden / 3.3V Schiene)
// PWM-Einstellungen (ca. 500 Hz für Dreheisenmesswerke)const int PWM_FREQ = 500;       const int PWM_RESOLUTION = 10;   // 10 Bit = 0 bis 1023
// PIN-Definitionen für den Drehencoderconst int ENCODER_A_PIN = 25;const int ENCODER_B_PIN = 26;const int ENCODER_SW_PIN = 27; // Taster des Encodersconst int ENCODER_VCC_PIN = -1; // -1 wenn direkt an 3V3 angeschlossenconst int ENCODER_STEPS = 4;    // Schritte pro Raste (meist 4)
// Encoder-Objekt initialisierenAiEsp32RotaryEncoder rotaryEncoder = AiEsp32RotaryEncoder(ENCODER_A_PIN, ENCODER_B_PIN, ENCODER_SW_PIN, ENCODER_VCC_PIN, ENCODER_STEPS);
// Variablen für die Steuerungint aktuellesInstrument = 1; // 1 = Instrument 1, 2 = Instrument 2int pwmWertInst1 = 0;int pwmWertInst2 = 0;int alterWert = -1;
// ISR (Interrupt Service Routine) für den Encodervoid IRAM_ATTR readEncoderISR() {
    rotaryEncoder.readEncoder_ISR();
}
void setup() {
    Serial.begin(115200);
    delay(1000);
    Serial.println("\n=== Hartmann & Braun Kalibrierung gestartet ===");

    // PWM-Kanäle auf dem ESP32 einrichten
    ledcAttach(PIN_INST_1, PWM_FREQ, PWM_RESOLUTION);
    ledcAttach(PIN_INST_2, PWM_FREQ, PWM_RESOLUTION);

    // Instrumente initial auf 0 setzen
    ledcWrite(PIN_INST_1, 0);
    ledcWrite(PIN_INST_2, 0);

    // Encoder initialisieren
    rotaryEncoder.begin();
    rotaryEncoder.setup(readEncoderISR);
    rotaryEncoder.setBoundaries(0, 1023, false); // Wertebereich 0-1023, kein Kreis-Umlauf
    rotaryEncoder.setAcceleration(50);          // Schnelles Drehen erhöht die Schrittweite
    rotaryEncoder.setEncoderValue(0);

    Serial.println("Steuerung bereit!");
    Serial.println("-> Druecken: Wechselt zwischen Instrument 1 und 2");
    Serial.println("-> Drehen  : Verandert den PWM-Wert (0 bis 1023)");
    Serial.println("AKTUELL: Instrument 1 (Minuten) gewaehlt.");
}
void loop() {
    // 1. Taster des Encoders abfragen (Umschalten des Instruments)
    if (rotaryEncoder.isEncoderButtonClicked()) {
        if (aktuellesInstrument == 1) {
            // Sichern des aktuellen Wertes für Instrument 1
            pwmWertInst1 = rotaryEncoder.getEncoderValue();
            aktuellesInstrument = 2;
            // Encoder auf den alten Wert von Instrument 2 setzen
            rotaryEncoder.setEncoderValue(pwmWertInst2);
            Serial.println("\n[WECHSEL] -> Jetzt Instrument 2 (Stunden) gewaehlt.");
        } else {
            // Sichern des aktuellen Wertes für Instrument 2
            pwmWertInst2 = rotaryEncoder.getEncoderValue();
            aktuellesInstrument = 1;
            // Encoder auf den alten Wert von Instrument 1 setzen
            rotaryEncoder.setEncoderValue(pwmWertInst1);
            Serial.println("\n[WECHSEL] -> Jetzt Instrument 1 (Minuten) gewaehlt.");
        }
        alterWert = -1; // Ausgabe im Serial Monitor erzwingen
        delay(200);     // Software-Entprellung für den Taster
    }

    // 2. Drehbewegung abfragen und PWM ausgeben
    if (rotaryEncoder.encoderChanged()) {
        int aktuellerWert = rotaryEncoder.getEncoderValue();

        if (aktuellerWert != alterWert) {
            if (aktuellesInstrument == 1) {
                pwmWertInst1 = aktuellerWert;
                ledcWrite(PIN_PIN_INST_1, pwmWertInst1);
                Serial.printf("Instrument 1 (Minuten) | PWM-Wert: %4d / 1023\n", pwmWertInst1);
            } else {
                pwmWertInst2 = aktuellerWert;
                ledcWrite(PIN_INST_2, pwmWertInst2);
                Serial.printf("Instrument 2 (Stunden) | PWM-Wert: %4d / 1023\n", pwmWertInst2);
            }
            alterWert = aktuellerWert;
        }
    }
}
```


## So gehst du bei der Kalibrierung vor:

   1. Flashe den Code über VSCode/PlatformIO auf den ESP32 und öffne den Serial Monitor (unten in der Leiste das Stecker-Symbol oder Alt+Shift+M).
   2. Du startest bei Instrument 1. Drehe den Encoder langsam hoch, bis sich der Zeiger bewegt. Notiere dir den PWM-Wert für den Nullpunkt (falls dieser nicht bei 0 liegt).
   3. Drehe weiter, bis der Zeiger exakt auf dem ersten Skalenstrich (z. B. 10 % oder 5 Minuten) steht. Notiere den PWM-Wert aus dem Serial Monitor.
   4. Wiederhole das in Schritten bis zum Vollausschlag (100 %). Vorsicht am Ende: Wegen der neuen Vorwiderstände erreichst du den Vollausschlag deutlich vor 1023. Drehe nicht weiter, sobald der Zeiger am Anschlag steht!
   5. Drücke einmal auf den Encoder-Knopf. Du wechselst zu Instrument 2 und wiederholst das Ganze für die Stunden.
   6. Die notierten Werte trägst du anschließend einfach in die Arrays (KALIBRIERUNG_INST_1 und _2) des finalen Uhren-Codes ein.