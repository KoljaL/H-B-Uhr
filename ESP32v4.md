Die Pin-Tabelle für das ESP32 DevKitC V4 (38 Pins) ist in 🟢 sicher, 🟡 Standard-Bus, 🔵 Nur-Eingang, ⚠️ Strapping-Pins und ❌ verbotene Pins unterteilt, um die Nutzung zu vereinfachen. Die ❌ Pins (GPIO 6-11) sollten frei bleiben, da sie intern mit dem Flash-Speicher verbunden sind.

| Linke Pin-Reihe | Typ / Status       | Funktion / Nutzungsempfehlung                  | 🔄   | Rechte Pin-Reihe | Typ / Status       | Funktion / Nutzungsempfehlung                  |
| --------------- | ------------------ | ---------------------------------------------- | --- | ---------------- | ------------------ | ---------------------------------------------- |
| 3V3             | Stromspannung      | Ausgang 3,3V Spannungsversorgung               |     | GND              | Masse              | Masse-Anschluss (Ground)                       |
| EN (CHIP_PU)    | Reset-Pin          | Setzt den ESP32 zurück (Reset-Taste)           |     | GPIO 23          | 🟢 Sicher / SPI     | MOSI (Standard SPI-Bus) / Digital I/O          |
| GPIO 36 (VP)    | 🔵 Nur-Eingang      | ADC1_CH0 (Perfekt für Analog-Sensoren)         |     | GPIO 22          | 🟡 Sicher / I2C     | SCL (Standard I2C-Bus) / Digital I/O           |
| GPIO 39 (VN)    | 🔵 Nur-Eingang      | ADC1_CH3 (Perfekt für Analog-Sensoren)         |     | GPIO 1 (TX0)     | ⚠️ UART0 / Meiden   | USB-Seriell-Verbindung (Blockiert IDE-Upload)  |
| GPIO 34         | 🔵 Nur-Eingang      | ADC1_CH6 (Perfekt für Analog-Sensoren)         |     | GPIO 3 (RX0)     | ⚠️ UART0 / Meiden   | USB-Seriell-Verbindung (Blockiert IDE-Upload)  |
| GPIO 35         | 🔵 Nur-Eingang      | ADC1_CH7 (Perfekt für Analog-Sensoren)         |     | GPIO 21          | 🟡 Sicher / I2C     | SDA (Standard I2C-Bus) / Digital I/O           |
| GPIO 32         | 🟢 Sicher           | ADC1_CH4 / Digital I/O                         |     | GPIO 19          | 🟡 Sicher / SPI     | MISO (Standard SPI-Bus) / Digital I/O          |
| GPIO 33         | 🟢 Sicher           | ADC1_CH5 / Digital I/O                         |     | GPIO 18          | 🟡 Sicher / SPI     | SCK (Standard SPI-Bus) / Digital I/O           |
| GPIO 25         | 🟢 Sicher           | DAC1 (Echter Analog-Ausgang) / Digital I/O     |     | GPIO 5           | ⚠️ Strapping-Pin    | Steuert Boot-Modus (Vorsicht beim Einschalten) |
| GPIO 26         | 🟢 Sicher           | DAC2 (Echter Analog-Ausgang) / Digital I/O     |     | GPIO 17          | 🟢 Sicher / UART2   | TX2 / Frei auf WROOM (Belegt auf WROVER-PSRAM) |
| GPIO 27         | 🟢 Sicher           | ADC2_CH7 (Nicht bei WLAN nutzen) / Digital I/O |     | GPIO 16          | 🟢 Sicher / UART2   | RX2 / Frei auf WROOM (Belegt auf WROVER-PSRAM) |
| GPIO 14         | 🟢 Sicher           | ADC2_CH6 / Digital I/O                         |     | GPIO 4           | 🟢 Sicher           | ADC2_CH0 / Digital I/O                         |
| GPIO 12         | ⚠️ Strapping-Pin    | Boot schlägt fehl, wenn beim Start HIGH!       |     | GPIO 0           | ⚠️ Strapping-Pin    | Bootet in Flash-Modus, wenn beim Start LOW!    |
| GND             | Masse              | Masse-Anschluss (Ground)                       |     | GPIO 2           | ⚠️ Strapping-Pin    | Onboard-LED (Muss beim Booten LOW sein)        |
| GPIO 13         | 🟢 Sicher           | ADC2_CH4 / Digital I/O                         |     | GPIO 15          | ⚠️ Strapping-Pin    | Steuert Boot-Modus (Vorsicht beim Einschalten) |
| GPIO 9 (D2)     | ❌ Absolut verboten | Interner Flash-Speicher (Systemabsturz)        |     | GPIO 8 (D1)      | ❌ Absolut verboten | Interner Flash-Speicher (Systemabsturz)        |
| GPIO 10 (D3)    | ❌ Absolut verboten | Interner Flash-Speicher (Systemabsturz)        |     | GPIO 7 (D0)      | ❌ Absolut verboten | Interner Flash-Speicher (Systemabsturz)        |
| GPIO 11 (CMD)   | ❌ Absolut verboten | Interner Flash-Speicher (Systemabsturz)        |     | GPIO 6 (CLK)     | ❌ Absolut verboten | Interner Flash-Speicher (Systemabsturz)        |
| 5V (VIN)        | Stromspannung      | Eingang für 5V Stromversorgung                 |     | GND              | Masse              | Masse-Anschluss (Ground)                       |


