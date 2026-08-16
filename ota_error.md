# OTA-Update funktioniert nicht

## Beobachteter Fehler

Der OTA-Build ist erfolgreich. Der Upload scheitert erst beim Netzwerkzugriff:

```text
Sending invitation to HB-Uhr failed
[ERROR]: Host HB-Uhr Not Found
*** [upload] Error 1
```

Damit erreicht `espota` den ESP32 nicht. Es wurde noch keine Firmware uebertragen.

## Probleme

1. **Das Ziel `HB-Uhr` wird nicht aufgeloest.**
   In `platformio.ini` steht `upload_port = HB-Uhr`. PlatformIO beziehungsweise `espota` findet diesen Hostnamen beim Upload nicht. Der Hostname ist damit aktuell kein funktionierender Upload-Endpunkt.

2. **Die mDNS-Erreichbarkeit ist nicht nachgewiesen.**
   `ArduinoOTA.setHostname(OTA_HOSTNAME)` setzt den OTA-Hostnamen nur auf dem laufenden ESP32. Der ESP32 muss dafuer bereits erfolgreich mit dem WLAN verbunden sein, der OTA-Dienst muss laufen und der Rechner muss mDNS im selben Netz aufloesen koennen. Fehlt einer dieser Punkte, bleibt `HB-Uhr` unbekannt.

3. **Die IP-Adresse ist nicht dauerhaft konsistent.**
   In einer frueheren Konfiguration wurde `192.168.178.151` als Upload-Ziel verwendet. Die aktuelle Konfiguration verwendet dagegen den Hostnamen `HB-Uhr`, waehrend `192.168.178.149` als `--host_ip` gesetzt ist. `--host_ip` ist die Rueckmelde-Adresse des Rechners fuer `espota` und nicht die IP-Adresse des ESP32. Ohne DHCP-Reservierung kann sich die ESP32-Adresse zudem aendern.

4. **Der OTA-Dienst kann nur nach einem funktionierenden USB-Flash getestet werden.**
   Die OTA-Initialisierung erfolgt erst nach erfolgreicher WLAN-Verbindung in `aktualisiereOtaDienst()`. Wenn auf dem ESP32 noch keine aktuelle OTA-faehige Firmware laeuft oder WLAN beziehungsweise OTA dort nicht bereit sind, kann der Upload nicht funktionieren.

## Naechste Pruefungen

- Seriellen Monitor beim USB-Start beobachten. Erwartet wird eine Meldung wie:
  ` [OTA] Aktiviert auf <ESP32-IP>:3232 mit Hostname 'HB-Uhr'`
- Vom Mac pruefen, ob `HB-Uhr.local` beziehungsweise `HB-Uhr` per mDNS aufloesbar ist.
- Die aktuell vom ESP32 ausgegebene IP-Adresse mit der Upload-Konfiguration abgleichen.
- Eine feste DHCP-Zuweisung fuer den ESP32 einrichten oder die aktuelle IP temporaer als `upload_port` verwenden.
- Sicherstellen, dass Mac und ESP32 im selben Netzwerk/VLAN sind und UDP-Port `3232` erreichbar ist.
- Erst danach den OTA-Upload erneut ausfuehren.
