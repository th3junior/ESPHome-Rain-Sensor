# ESPHome Rain Sensor

[PT](README.md) | **EN** | [ES](README.es.md)

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/G7B227FYIH)

Cheap rain detector with **ESP32** + **MH-RD** module + Home Assistant (ESPHome).

This is not a rain gauge: it does not measure millimetres. It tells you if the plate is wet and can fire automations.

Power: **phone USB charger** on the DevKit. **3 wires.** Leave the **DO pin unconnected**.

![Wiring diagram](docs/esquema-eletrico.svg)

![30-pin DevKit pin locations](docs/diagrama-ligacao.svg)

## Parts

- ESP32 DevKit **30-pin** (DOIT / WROOM-32)
- **MH-RD** raindrops kit (trace plate + blue LM393 board)
- 3 jumper wires + the 2 wires that come with the plate
- 5 V / 1 A USB charger and cable
- PC with Python (to flash) and Home Assistant (Docker or HAOS)

## Wiring (written)

USB charger into the **ESP32 USB port**. Then:

```
ESP32  3V3   ->  MH-RD  VCC
ESP32  GND   ->  MH-RD  GND
ESP32  VP    ->  MH-RD  AO      GPIO36 (2nd pin on the left, USB on top, below EN)

MH-RD  S+ / S-  ->  rain plate
MH-RD  DO           leave open
```

| Wire | From | To | Role |
| --- | --- | --- | --- |
| Red | 3V3 | VCC | 3.3 V to the module |
| Black | GND | GND | Ground |
| Green | VP | AO | Voltage = wetness |

**Do not** put VCC on 5 V, VIN, or VP.

On the roof: copper facing the sky, plate at 30-45 degrees, connector at the top. ESP and LM393 in a box.

## How firmware decides rain

AO drops when wet (~3.2 V dry to ~1.2 V soaked), read every 1 s.

| Entity | Meaning |
| --- | --- |
| **Chuva** (Rain) | On if V &lt; 2.50 V for 2 s. Off if V &gt; 2.90 V for 20 s |
| **Umidade da placa** | 3.17 V = 0 % · 1.25 V = 100 % |
| **Estado** | Chovendo / Seco (raining / dry) |
| **Última chuva** | Date/time of last on, or **Never** |

Automations: **Chuva** binary sensor or `esphome.its_raining`. Do not use the %.

The trimpot only affects the DO LED. With no DO wire, ignore it.

---

## 1. Install ESPHome on Windows

Home Assistant **Docker has no add-on**. Flash from this PC.

1. Install [Python](https://www.python.org/downloads/). Tick **Add python.exe to PATH**.
2. PowerShell:

```powershell
pip install esphome
```

3. Clone:

```powershell
git clone https://github.com/th3junior/ESPHome-Rain-Sensor.git
cd ESPHome-Rain-Sensor\esphome
```

## 2. Generate the API key

The key encrypts ESP &lt;-&gt; Home Assistant (32 bytes Base64).

**Windows (PowerShell):**

```powershell
python -c "import secrets,base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

**Linux / macOS:**

```bash
openssl rand -base64 32
```

Copy the whole line (it ends with `=`).

## 3. Create secrets.yaml

```powershell
copy secrets.yaml.example secrets.yaml
notepad secrets.yaml
```

Fill in:

```yaml
wifi_ssid: "your network name"
wifi_password: "wifi password"
api_encryption_key: "paste the key from step 2"
fallback_password: "atleast8chars"
```

Do not commit `secrets.yaml` to GitHub.

The same `api_encryption_key` is used for API and OTA.

## 4. Flash the board

Plug the ESP32 into this PC (3 wires already on the MH-RD).

```powershell
cd ESPHome-Rain-Sensor\esphome
esphome run rain-sensor.yaml
```

If asked for a port, pick **COMx** (Silicon Labs CP210x), not OTA.

If it fails: hold **BOOT**, run again, release when flashing starts.

In the log wait for:

```
WiFi Connected
IP Address: 192.168.x.x
```

Write down the IP.

## 5. Add in Home Assistant

1. **Settings → Devices & services → Add integration → ESPHome**
2. **Host:** the IP from the log (or `rain-sensor.local`)
3. **Port:** `6053`
4. Paste the **same** `api_encryption_key` from `secrets.yaml`

If it asks for Host and the board is not flashed yet, cancel: that screen does not flash firmware.

**Stable IP:** reserve the ESP MAC on the router. Otherwise DHCP can change the address and the integration drops.

Entities: Chuva, Umidade da placa, Estado, Última chuva. The rest is diagnostic.

## 6. Test

Plate flat, **copper up**.

1. Dry: Rain off, voltage ~2.8-3.2 V, low %.
2. Water **on the traces** (not only the header): in ~2 s Rain on, voltage drops, % rises.
3. Dry it: within ~20 s it goes off.

## 7. Dashboard (optional)

```yaml
type: vertical-stack
cards:
  - type: glance
    title: Rain
    entities:
      - entity: binary_sensor.detector_de_chuva_chuva
        name: Rain
      - entity: sensor.detector_de_chuva_umidade_da_placa
        name: Wetness
      - entity: sensor.detector_de_chuva_estado
        name: State
  - type: gauge
    entity: sensor.detector_de_chuva_umidade_da_placa
    min: 0
    max: 100
    needle: true
  - type: entities
    entities:
      - text_sensor.detector_de_chuva_ultima_chuva
      - sensor.detector_de_chuva_tensao_da_placa
      - sensor.detector_de_chuva_ip
```

If HA generated other entity IDs, use those.

## 8. Automation

```yaml
automation:
  - alias: It started raining
    trigger:
      - platform: event
        event_type: esphome.its_raining
    action:
      - action: persistent_notification.create
        data:
          message: Rain detector triggered
```

Or `binary_sensor.detector_de_chuva_chuva` → `on`.

## Sensitivity

In `esphome/rain-sensor.yaml`:

```yaml
  rain_voltage_wet: "2.50"   # below this = rain
  rain_voltage_dry: "2.90"   # above this = dry
```

More sensitive: raise `wet` (e.g. `2.70`). Flash again with `esphome run`.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Voltage ~0.14 V | AO loose or not on **VP** |
| HA: unable to connect / missing `api` | Board not flashed, wrong IP (this YAML does not deep-sleep) |
| Wi-Fi fails | `secrets.yaml`; AP `ESPHome Rain Fallback` |
| Rain does not change | Water on **copper**, bridging traces |
| USB flash fails | BOOT button; data cable |

## License

MIT. Copyright (c) 2026 [th3junior](https://github.com/th3junior). See [LICENSE](LICENSE).
