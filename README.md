# Steinel BLE Mesh → Home Assistant Bridge

ESP32-basierte Bridge zwischen Steinel Bluetooth Mesh Leuchten und Home Assistant via MQTT.

![Architektur](docs/architecture.png)

```
Steinel L 810 SC
  (BLE Mesh SIG v1.1)
        │
        │  Bluetooth Mesh
        ▼
    ESP32
  (Provisioner + WiFi)
        │
        │  WiFi / MQTT
        ▼
  Home Assistant
  (MQTT Autodiscovery)
```

## ✨ Unterstützte Funktionen (Steinel L 810 SC)

| Funktion | HA-Entity | Steuerbar |
|---|---|---|
| Licht an/aus | `light` | ✅ |
| Helligkeit (Hauptlicht) | `light` (brightness) | ✅ |
| Grundlicht-Level | `number` | ✅ |
| Dauerlicht-Modus | `select` (Modus) | ✅ |
| Softlichtstart (Übergangszeit) | automatisch | ✅ |
| Nachlaufzeit | `number` | ✅ |
| Empfindlichkeit (HF-Sensor) | `select` | ✅ |
| Dämmerungsschwelle | `number` | ✅ |
| Bewegungsmelder | `binary_sensor` | gelesen |
| Beleuchtungsstärke (Lux) | `sensor` | gelesen |
| Zeitplan-Funktion | HA-Automatisierung | ✅ |

---

## 📦 Voraussetzungen

### Hardware
- **ESP32** (WROOM-32 oder WROVER empfohlen – kein S2/S3/C3, da diese kein klassisches BLE haben)
- Steinel L 810 SC (oder andere Steinel Connected Lighting Leuchten mit BLE Mesh)
- Home Assistant mit MQTT-Integration (Mosquitto Broker Add-on empfohlen)

### Software
- [ESP-IDF v5.x](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/)
- Python 3.8+

---

## 🚀 Schnellstart

### 1. ESP-IDF installieren

```bash
# ESP-IDF klonen (falls noch nicht vorhanden)
git clone --recursive https://github.com/espressif/esp-idf.git ~/esp/esp-idf
cd ~/esp/esp-idf
./install.sh esp32
. ./export.sh
```

### 2. Projekt klonen und konfigurieren

```bash
git clone https://github.com/<dein-name>/steinel-ble-mesh-ha.git
cd steinel-ble-mesh-ha/esp32_firmware

# Konfiguration öffnen
idf.py menuconfig
```

In `menuconfig` anpassen:
- **Component config → Steinel Mesh Config**
  - WiFi SSID und Passwort
  - MQTT Broker URL (z.B. `mqtt://192.168.1.10:1883`)
  - MQTT Benutzername/Passwort (falls gesetzt)

Oder direkt in `sdkconfig.defaults`:
```
CONFIG_WIFI_SSID="MeinWLAN"
CONFIG_WIFI_PASSWORD="MeinPasswort"
CONFIG_MQTT_BROKER_URL="mqtt://192.168.1.10:1883"
```

### 3. Flashen

```bash
idf.py -p /dev/ttyUSB0 flash monitor
```

### 4. Steinel-Leuchte provisionieren

1. **Steinel L 810 SC in den Provisionierungs-Modus** setzen:
   - Leuchte ausschalten und 3× schnell ein/aus schalten
   - Die Leuchte blinkt kurz → Provisionierungs-Modus aktiv (30 Sekunden)

2. **ESP32 erkennt die Leuchte automatisch** und provisioniert sie ins Mesh-Netzwerk.
   Im seriellen Monitor erscheint:
   ```
   I (12345) BLE_MESH: Unversioniertes Gerät: UUID=...
   I (12890) BLE_MESH: ✅ Gerät provisioniert: Adresse=0x0001
   I (13100) MQTT_BRIDGE: ✅ HA Discovery für Steinel_0001 veröffentlicht
   ```

3. **In Home Assistant** erscheinen die Entities automatisch unter
   *Einstellungen → Geräte & Dienste → MQTT*.

> **Hinweis:** Der Provisionierungsvorgang muss nur einmal durchgeführt werden.
> Die Mesh-Konfiguration wird im NVS-Flash des ESP32 gespeichert.

---

## 🏠 Home Assistant Einrichtung

### MQTT-Broker (Mosquitto)

Falls noch nicht installiert, im Add-on-Store nach „Mosquitto Broker" suchen.

`configuration.yaml`:
```yaml
mqtt:
  broker: localhost
  port: 1883
  # username: ha_user
  # password: geheim
  discovery: true
  discovery_prefix: homeassistant
```

### Package einbinden (optional)

```yaml
# configuration.yaml
homeassistant:
  packages:
    steinel: !include packages/steinel_mesh.yaml
```

---

## 📡 MQTT Topic-Struktur

```
steinel_mesh/
├── bridge/
│   └── status              ← online / offline (LWT)
└── <ADDR>/                 ← 4-stellige Hex-Adresse (z.B. 0001)
    ├── light/
    │   ├── state           ← {"state":"ON","brightness":75}
    │   └── set             ← {"state":"ON"} oder {"state":"ON","brightness":50}
    ├── motion/
    │   └── state           ← ON / OFF
    ├── illuminance/
    │   └── state           ← 42.5  (lux, float)
    ├── mode/
    │   ├── state           ← Auto / Dauerlicht / Deaktiviert
    │   └── set             ← Auto / Dauerlicht / Deaktiviert
    ├── hold_time/
    │   ├── state           ← 300  (Sekunden)
    │   └── set             ← 120
    ├── sensitivity/
    │   ├── state           ← Niedrig / Mittel / Hoch
    │   └── set             ← Niedrig / Mittel / Hoch
    ├── twilight/
    │   ├── state           ← 50  (lux)
    │   └── set             ← 30
    └── base_light/
        ├── state           ← 10  (Prozent)
        └── set             ← 15
```

---

## 🔧 Mehrere Leuchten

Der ESP32 kann bis zu **16 Steinel-Leuchten** gleichzeitig verwalten. Jede Leuchte erhält beim Provisionieren eine eindeutige Mesh-Adresse (0x0001, 0x0002, ...) und wird automatisch in HA als separates Gerät angelegt.

---

## 🔐 Sicherheitshinweise

Die Standard-Netzwerk- und App-Keys in `ble_mesh.c` sollten vor dem produktiven Einsatz geändert werden:

```c
// esp32_firmware/main/ble_mesh.c
static uint8_t s_net_key[16] = { /* eigene 16 Bytes */ };
static uint8_t s_app_key[16] = { /* eigene 16 Bytes */ };
```

Zufällige Keys generieren:
```bash
python3 -c "import os; print(', '.join(hex(b) for b in os.urandom(16)))"
```

---

## 🏗️ Architektur

```
ESP32 Firmware
├── main.c            – Einstiegspunkt, Initialisierungsreihenfolge
├── wifi_manager.c    – WiFi-Verbindung mit Reconnect-Logik
├── ble_mesh.c        – BLE Mesh Provisioner
│   ├── Provisioning (PB-ADV + PB-GATT)
│   ├── Generic OnOff Client (Licht an/aus)
│   ├── Light Lightness Client (Dimmen)
│   ├── Sensor Client (Bewegung, Lux)
│   ├── Scene Client (Szenen/Dauerlicht)
│   ├── Scheduler Client (Zeitplan)
│   └── Generic DTT Client (Softlichtstart)
└── mqtt_bridge.c     – MQTT ↔ BLE Mesh Bridge
    ├── HA Autodiscovery (8 Entities pro Gerät)
    ├── Zustandsveröffentlichung
    └── Befehlsempfang & -weitergabe
```

---

## 🐛 Fehlersuche

### Gerät wird nicht gefunden
- Leuchte in den Provisionierungs-Modus versetzen (3× ein/aus)
- Abstand zwischen ESP32 und Leuchte prüfen (< 5 m beim Provisioning)
- Seriellen Monitor auf Fehlermeldungen prüfen

### MQTT-Verbindung schlägt fehl
- Broker-URL in `sdkconfig.defaults` prüfen
- Mosquitto-Logs in HA prüfen: *Supervisor → Mosquitto → Logs*
- Firewall-Regeln prüfen (Port 1883)

### Keine Entities in HA
- MQTT Discovery aktiviert? (`discovery: true` in `configuration.yaml`)
- MQTT-Integration in HA prüfen: *Einstellungen → Geräte & Dienste → MQTT*
- Topics im MQTT Explorer überprüfen

---

## 📋 Bekannte Einschränkungen

- **Vendor-spezifische Modelle:** Steinel hat möglicherweise proprietäre Opcodes für einige Einstellungen (z.B. bestimmte Sensor-Parameter). Diese müssen ggf. durch BLE-Sniffing der Steinel Connect App ermittelt werden (nRF Sniffer + Wireshark).
- **Zeitplan-Funktion:** Wird über HA-Automationen abgebildet, da der BLE Mesh Scheduler sehr gerätespezifisch ist.
- **OTA-Updates:** Noch nicht implementiert.

---

## 🤝 Mitwirken

Pull Requests und Issues sind willkommen! Insbesondere:
- Testergebnisse mit anderen Steinel-Modellen (L 840 SC, L 820 SC)
- Reverse-Engineering-Ergebnisse der Steinel Connect App
- Unterstützung für weitere BLE Mesh Modelle

---

## 📄 Lizenz

MIT License – siehe [LICENSE](LICENSE)

---

## 🙏 Danksagung

- [Espressif ESP-BLE-MESH](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/esp-ble-mesh/) – Open-Source BLE Mesh Stack
- [CFenner/HomeAssistant-Steinel-BT](https://github.com/CFenner/HomeAssistant-Steinel-BT) – Inspiration
- Loxforum-Community für Reverse-Engineering-Hinweise
