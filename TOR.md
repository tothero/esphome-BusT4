# Tor SUGAM: Nice Robus + BiDi-WiFi mit ESPHome

Lokale Steuerung des Schiebetors (Nice Robus RBS400/600/800) über den BiDi-WiFi-Stick am BusT4-/IBT4N-Port der Steuerung. Die Nice-Firmware des Sticks ist durch ESPHome ersetzt; Cloud und MyNice-App werden nicht mehr genutzt.

Konfiguration: [`tor.yaml`](tor.yaml). Allgemeine Doku der Komponente: [README.md](README.md).

## Funktionsumfang

Bewusst nur Bedienung und Status:

- Cover „Tor“: Auf, Zu, Stop, Zustand, Position
- Taster „Teil-Auf 1“ und „Schritt“
- WLAN-Signal, ESPHome-Version

Nicht enthalten sind die Konfigurationsschalter aus `example.yaml` (Auto-Close, Photo-Close, Always-Close, Standby, Peak-Mode, Pre-Flash), der Gate-Lock und das Raw-Command-Feld. Diese senden SET-Befehle an die Steuerung; die Template-Schalter könnten das beim Booten auch ungewollt tun. Die Parameter der Steuerung bleiben so unverändert.

## Hardware

| | |
|---|---|
| Stick | Nice BiDi-WiFi, ESP32-D0WD-V3 rev3.1, 8 MB Flash, MAC `6c:c8:40:08:73:ec` |
| Sicherheit | keine Flash-Verschlüsselung, kein Secure Boot, UART-Download offen |
| Bus-T4 | UART GPIO21 (TX) / GPIO18 (RX), 19200 Baud |
| Im Netz | `tor.local` |

## Sicherung der Nice-Firmware

Das vollständige Flash-Abbild (8 MB) liegt **außerhalb dieses Repos** unter `../backup/bidiwifi_6cc8400873ec_2026-11-08.bin`. Es enthält gerätespezifische Daten (NVS) und gehört nicht ins Repo oder in fremde Hände. Eine zweite Lesung (`…_read2.bin`) unterscheidet sich nur in einem NVS-Sektor, Code und Partitionen sind identisch.

Zurück zur Nice-Firmware (Stick per UART angeschlossen, siehe unten):

```sh
uvx --from esptool esptool -p /dev/cu.usbserial-XXXX -b 115200 erase-flash
uvx --from esptool esptool -p /dev/cu.usbserial-XXXX -b 115200 write-flash 0x0 ../backup/bidiwifi_6cc8400873ec_2026-11-08.bin
```

## Entwicklungsumgebung einrichten

Voraussetzung: [`uv`](https://docs.astral.sh/uv/). ESPHome und esptool laufen über `uvx`, eine Installation ist nicht nötig.

1. Repo klonen:
   ```sh
   git clone git@github.com:tothero/esphome-BusT4.git
   cd esphome-BusT4
   git checkout tor-sugam
   ```
2. `secrets.yaml` anlegen (wird von git ignoriert):
   ```sh
   cp secrets.yaml.example secrets.yaml
   ```
   Benötigt werden `wifi_ssid`, `wifi_password` (2,4-GHz-Netz, Groß-/Kleinschreibung beachten), `api_encryption_key`, `ota_password`, `fallback_password`, `web_password`. Neue Werte erzeugen:
   ```sh
   openssl rand -base64 32   # api_encryption_key
   openssl rand -hex 16      # ota_password
   openssl rand -hex 8       # fallback_password, web_password
   ```
   **Achtung:** Ein neuer `ota_password` passt nicht zur Firmware auf dem Stick; OTA geht dann erst wieder nach einem Flash per UART. Die aktuelle `secrets.yaml` deshalb aufbewahren (Passwortmanager).
3. `build_path` in `tor.yaml` an den eigenen Benutzer anpassen. Der Build liegt bewusst außerhalb des Repos, weil das Repo in iCloud Drive liegt und der Build mehrere hundert MB groß ist.
4. Bauen:
   ```sh
   uvx esphome compile tor.yaml
   ```
   Beim ersten Lauf lädt ESPHome ESP-IDF und die Toolchain (ca. 1,7 GB) nach `~/Library/Caches/esphome/idf/`. Die Firmware liegt danach in `<build_path>/build/firmware.factory.bin` (UART) und `firmware.ota.bin` (OTA).

### Wenn der Download von GitHub hängt

Im Heimnetz hängt HTTPS zu github.com aus Terminal-Programmen im TLS-Handshake: Pakete mit 1500 Byte kommen auf dem Weg zu GitHub nicht an (Pfad-MTU < 1500 ohne ICMP-Rückmeldung), der Browser ist nicht betroffen. Workaround:

1. Im Browser laden: `https://github.com/esphome-libs/esp-idf/releases/download/v5.5.6/esp-idf-v5.5.6.tar.xz` (Version siehe Fehlermeldung von ESPHome)
2. Lokal bereitstellen und bauen:
   ```sh
   mkdir -p /tmp/idf-mirror && cp ~/Downloads/esp-idf-v5.5.6.tar.xz /tmp/idf-mirror/
   python3 -m http.server 8765 --bind 127.0.0.1 --directory /tmp/idf-mirror &
   ESPHOME_IDF_FRAMEWORK_MIRRORS="http://127.0.0.1:8765/esp-idf-v{VERSION}.tar.xz" \
   IDF_GITHUB_ASSETS="dl.espressif.com/github_assets" \
   uvx esphome compile tor.yaml
   ```
   `IDF_GITHUB_ASSETS` holt die Toolchain vom Espressif-Mirror statt von GitHub.

Alternative: MTU der Netzwerkschnittstelle vorübergehend senken (`sudo networksetup -setMTU "<Dienst>" 1400`, danach wieder `1500`).

## Firmware aufspielen

### Per WLAN (Normalfall)

```sh
uvx esphome run tor.yaml --device tor.local
```

Fallback, falls der Stick das WLAN nicht findet: Er öffnet nach ca. 1 min den Hotspot „Tor Fallback“ (Passwort `fallback_password`) mit Captive-Portal zur Eingabe neuer WLAN-Daten.

### Per UART (erstes Flashen, Notfall)

Stick aus der Steuerung nehmen (Steuerung vorher stromlos: Sicherungsautomat bzw. F2, ggf. Pufferbatterie abklemmen). Testpunkte siehe Foto `img/bidiwifi-pinout.jpg`.

Verkabelung mit dem Lobaro USB Config Adapter V1 (CP2102, 6-pol. JST-ZH, 3,3-V-Logik):

| Adapter | BiDi-WiFi |
|---|---|
| P1 Reset (RTS) | EN |
| P2 3,3 V | +3V3 (Kühlfahne des Spannungsreglers) |
| P3 TX vom PC | Rx |
| P4 RX zum PC | Tx |
| P5 Boot (DTR) | IO0 |
| P6 GND | GND (Pin der Stiftleiste) |

Die Kabelfarben sind nicht verlässlich (bei unserem Kabel ist Blau = GND), die Pin-Position zählt. Prüfen: P2 gegen GND ≈ 3,3 V. Der Regler des Adapters liefert nur ca. 150 mA; das reichte zum Flashen und sogar für WLAN, für Dauerbetrieb aber nicht gedacht. esptool schaltet über RTS/DTR selbst in den Bootloader. **Nur 115200 Baud** verwenden, bei 460800 gab es Übertragungsfehler.

```sh
uvx --from esptool esptool -p /dev/cu.usbserial-XXXX flash-id
uvx --from esptool esptool -p /dev/cu.usbserial-XXXX -b 115200 write-flash 0x0 <build_path>/build/firmware.factory.bin
```

## Zugriff

- Weboberfläche: `http://tor.local`, Benutzer `admin`, Passwort `web_password` (Digest-Auth)
- Home Assistant: ESPHome-Integration, Schlüssel `api_encryption_key`
- REST (Entitäten werden über den **Namen** angesprochen):
  ```sh
  curl --digest -u admin:<web_password> http://tor.local/cover/Tor
  curl --digest -u admin:<web_password> -X POST http://tor.local/cover/Tor/open
  curl --digest -u admin:<web_password> -X POST http://tor.local/cover/Tor/stop
  curl --digest -u admin:<web_password> -X POST http://tor.local/cover/Tor/close
  ```

## Steuerung (Nice Robus), Vorsicht

- Einstellungen liegen in der Steuerung, nicht im Stick; Stromausfall und Flashen des Sticks ändern sie nicht.
- Tasten an der Steuerung: **[Open ▲] + [Stop/Set]** = Einlernen der Vorrichtungen (nötig z. B. nach Einbau einer MLAE-Karte), **[Open ▲] + [Close ▼]** = **Werksreset, alle Einstellungen weg**.

## Geplant

1. Parameter der Steuerung (`CFG_*`, Register 0x80–0x85) per GET auslesen, nur als Anzeige; mit den LEDs L1–L8 der Steuerung abgleichen.
2. Schreiben nur mit gelesenem Ist-Wert, `restore_mode: DISABLED` und einer Freigabe-Sperre, die nach dem Booten aus ist.
3. Steuerung vom Raspberry Pi 5 / Home Assistant, Geofence beim Heranfahren.
