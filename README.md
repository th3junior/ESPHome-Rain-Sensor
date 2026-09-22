# ESPHome Rain Sensor

**PT** | [EN](README.en.md) | [ES](README.es.md)

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/G7B227FYIH)

Detector de chuva barato com **ESP32** + módulo **MH-RD** + Home Assistant (ESPHome).

Não é pluviômetro: não mede milímetros. Diz se a placa está molhada e dispara automação.

Alimentação: **carregador USB de celular** no DevKit. **3 fios.** O pino **DO fica solto**.

![Esquema elétrico](docs/esquema-eletrico.svg)

![Pinos do DevKit 30 pinos](docs/diagrama-ligacao.svg)

## Material

- ESP32 DevKit **30 pinos** (DOIT / WROOM-32)
- Kit raindrops **MH-RD** (placa de trilhas + plaquinha azul LM393)
- 3 jumpers + os 2 fios da placa de trilhas
- Carregador USB 5 V / 1 A e cabo
- PC com Python (para gravar) e Home Assistant (Docker ou HAOS)

## Ligação por escrito

USB do carregador no **USB do ESP32**. Depois:

```
ESP32  3V3   →  MH-RD  VCC
ESP32  GND   →  MH-RD  GND
ESP32  VP    →  MH-RD  AO      GPIO36 (2º pino da esquerda, USB em cima, abaixo do EN)

MH-RD  S+ / S-  →  placa de trilhas
MH-RD  DO           sem fio
```

| Fio | De | Para | Função |
| --- | --- | --- | --- |
| Vermelho | 3V3 | VCC | 3,3 V para o módulo |
| Preto | GND | GND | Terra |
| Verde | VP | AO | Tensão = umidade |

**Não** ligue VCC no 5 V, no VIN nem no VP.

No telhado: cobre para o céu, placa a 30–45°, conector para cima. ESP e LM393 na caixa.

## Como o firmware decide chuva

O AO cai quando molha (~3,2 V seco → ~1,2 V encharcado), lido a cada 1 s.

| Entidade | Função |
| --- | --- |
| **Chuva** | On se V &lt; 2,50 V por 2 s. Off se V &gt; 2,90 V por 20 s |
| **Umidade da placa** | 3,17 V = 0 % · 1,25 V = 100 % |
| **Estado** | Chovendo / Seco |
| **Última chuva** | Data/hora do último on, ou **Nunca** |

Automações: `binary_sensor` **Chuva** ou evento `esphome.its_raining`. Não use o %.

O trimpot só mexe no LED do DO. Sem o fio DO, ignore o trimpot.

---

## 1. Instalar o ESPHome no Windows

O Home Assistant em **Docker não tem add-on**. Grave neste PC.

1. Instale o [Python](https://www.python.org/downloads/). Marque **Add python.exe to PATH**.
2. PowerShell:

```powershell
pip install esphome
```

3. Clone:

```powershell
git clone https://github.com/th3junior/ESPHome-Rain-Sensor.git
cd ESPHome-Rain-Sensor\esphome
```

## 2. Gerar a chave da API

A chave criptografa a conversa ESP ↔ Home Assistant (32 bytes em Base64).

**Windows (PowerShell):**

```powershell
python -c "import secrets,base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

**Linux / macOS:**

```bash
openssl rand -base64 32
```

Copie a linha inteira (termina com `=`).

## 3. Criar o secrets.yaml

```powershell
copy secrets.yaml.example secrets.yaml
notepad secrets.yaml
```

Preencha:

```yaml
wifi_ssid: "nome da sua rede"
wifi_password: "senha do wifi"
api_encryption_key: "cole a chave gerada no passo 2"
fallback_password: "minimo8chars"
```

Não envie `secrets.yaml` para o GitHub.

O mesmo `api_encryption_key` vale para API e OTA.

## 4. Gravar na placa

Plugue o ESP32 neste PC (já com os 3 fios no MH-RD).

```powershell
cd ESPHome-Rain-Sensor\esphome
esphome run rain-sensor.yaml
```

Se pedir porta, escolha o **COMx** (Silicon Labs CP210x), não OTA.

Se falhar: segure **BOOT**, rode de novo, solte quando começar a gravar.

No log, espere:

```
WiFi Connected
IP Address: 192.168.x.x
```

Anote o IP.

## 5. Adicionar no Home Assistant

1. **Configurações → Dispositivos e serviços → Adicionar integração → ESPHome**
2. **Host:** o IP do log (ou `rain-sensor.local`)
3. **Porta:** `6053`
4. Cole a **mesma** `api_encryption_key` do `secrets.yaml`

Se pedir Host e a placa ainda não foi gravada, cancele: essa tela não grava firmware.

**IP fixo:** no roteador, reserve o MAC do ESP no mesmo endereço. Senão o DHCP pode mudar e a integração cai.

Entidades que devem aparecer: Chuva, Umidade da placa, Estado, Última chuva. O resto é diagnóstico.

## 6. Testar

Placa deitada, **cobre para cima**.

1. Seco: Chuva off, tensão ~2,8–3,2 V, umidade baixa.
2. Água **nas trilhas** (não só no conector): em ~2 s Chuva on, tensão cai, umidade sobe.
3. Seque: em até ~20 s volta a off.

## 7. Dashboard (opcional)

```yaml
type: vertical-stack
cards:
  - type: glance
    title: Chuva
    entities:
      - entity: binary_sensor.detector_de_chuva_chuva
        name: Chuva
      - entity: sensor.detector_de_chuva_umidade_da_placa
        name: Umidade
      - entity: sensor.detector_de_chuva_estado
        name: Estado
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

Se os IDs forem outros, use os do seu dispositivo.

## 8. Automação

```yaml
automation:
  - alias: Começou a chover
    trigger:
      - platform: event
        event_type: esphome.its_raining
    action:
      - action: persistent_notification.create
        data:
          message: Detector de chuva acionou
```

Ou `binary_sensor.detector_de_chuva_chuva` → `on`.

## Ajustar sensibilidade

No YAML (`esphome/rain-sensor.yaml`):

```yaml
  rain_voltage_wet: "2.50"   # abaixo disto = chuva
  rain_voltage_dry: "2.90"   # acima disto = seco
```

Mais sensível: suba o `wet` (ex. `2.70`). Grave de novo com `esphome run`.

## Problemas

| Sintoma | O que checar |
| --- | --- |
| Tensão ~0,14 V | AO solto ou fora do **VP** |
| HA: unable to connect / falta `api` | Placa sem firmware, IP errado, ou ESP dormindo (este YAML não dorme) |
| Wi-Fi não conecta | `secrets.yaml`; AP `ESPHome Rain Fallback` |
| Chuva não muda com água | Água no **cobre**, cruzando trilhas |
| Gravação USB falha | BOTÃO BOOT; cabo de dados |

## Licença

MIT. Copyright (c) 2026 [th3junior](https://github.com/th3junior). Ver [LICENSE](LICENSE).
