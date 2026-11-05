# Frigate: detecção de pessoas nas câmeras

Atualizado em 05/11/2026. **Status: em andamento.** O passo 1 está concluído e o passo 2 precisa ser conferido. Os blocos dos passos 3 em diante estão prontos para colar.

O Frigate vai analisar as 4 câmeras Intelbras (ver [CAMERAS.md](CAMERAS.md)), detectar pessoas, guardar só os trechos com movimento durante eventos e mandar os eventos para o Home Assistant.

## Decisões

| Item | Escolha | Motivo |
| --- | --- | --- |
| Versão | `ghcr.io/blakeblackshear/frigate:0.18.2` | Estável atual. A regra do repositório pede para não usar `latest`/`stable`. |
| Detector | OpenVINO na CPU | Padrão do Frigate 0.18 e roda em CPU AMD. Não há Coral, e o ROCm não tem suporte oficial à GPU integrada. |
| Decodificação de vídeo | VAAPI na Radeon integrada (`LIBVA_DRIVER_NAME=radeonsi`) | Tira a decodificação da CPU. |
| Stream de detecção | Stream extra (`subtype=1`, 704x480, H.264) a 5 FPS | É leve e é o suficiente para detectar pessoas. |
| Stream de gravação | Stream principal (`subtype=0`, H.264) | Qualidade total nos clipes. |
| Gravação | Só eventos (alertas e detecções), 7 dias, modo `motion` | O NVR já grava 24h. O disco tem 77 GB livres, e 24h das 4 câmeras passaria de 100 GB por dia. |
| Integração com o HA | `http://172.17.0.1:5000` | O HA roda com `network_mode: host`. A porta 5000 não tem autenticação, então fica ligada só ao IP da bridge do Docker, fora da rede da casa. |
| Senhas | Arquivo `.env` (fora do Git), lido pelo Frigate como `{FRIGATE_...}` | O repositório é público. |

Servidor conferido em 05/11/2026: GPU em `/dev/dri/renderD128`, 30 GB de RAM e 77 GB livres em `/`.

## Passos

### 1. SSH e conferência do servidor (concluído)

Do Terminal do Mac: `ssh vilmar@192.168.1.7`. Depois: `ls -l /dev/dri; df -h ~; free -h; nproc`.

### 2. Pastas, senha e .gitignore (conferir)

```
cd ~/automacao
mkdir -p frigate/config frigate/storage
nano .env
```

No fim do `.env`, acrescente as linhas abaixo. Se a senha tiver `@`, `#` ou `:`, escreva `%40`, `%23` e `%3A`.

```
FRIGATE_RTSP_USER=admin
FRIGATE_RTSP_PASSWORD=SENHA_DO_NVR
```

```
chmod 600 .env
grep -q '^.env$' .gitignore 2>/dev/null || printf '.env\nfrigate/storage/\n' >> .gitignore
```

Confira também se o Mosquitto exige usuário e senha:

```
grep -A6 "^mqtt:" zigbee2mqtt/data/configuration.yaml
```

Se aparecerem `user` e `password`, acrescente ao `.env`:

```
FRIGATE_MQTT_USER=usuario_do_mosquitto
FRIGATE_MQTT_PASSWORD=senha_do_mosquitto
```

### 3. Serviço no docker-compose

Abra `nano docker-compose.yml` e cole o bloco abaixo dentro de `services:`, com o mesmo recuo dos outros serviços. Depois valide com `docker compose config -q`; se não aparecer nada, o arquivo está certo.

```
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:0.18.2
    restart: unless-stopped
    stop_grace_period: 30s
    shm_size: "512mb"
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./frigate/config:/config
      - ./frigate/storage:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "8971:8971"
      - "172.17.0.1:5000:5000"
    env_file:
      - .env
    environment:
      LIBVA_DRIVER_NAME: radeonsi
```

### 4. config.yml

Antes de colar, confira no HA qual canal é cada câmera: em Configurações → Dispositivos e serviços → Câmera genérica → Configurar, o canal aparece na URL. Depois troque `canal_2`, `canal_3` e `canal_4` por nomes definitivos, em minúsculas, sem acento nem espaço (por exemplo `frente_2`, `escritorio`, `quarto_visitas`). O nome vira o ID das entidades no HA, então é melhor não mudar depois.

Se o Mosquitto exigir senha, tire o `#` das linhas `user` e `password`. Crie o arquivo com `nano frigate/config/config.yml` e cole:

```
mqtt:
  host: mosquitto
  port: 1883
  # user: "{FRIGATE_MQTT_USER}"
  # password: "{FRIGATE_MQTT_PASSWORD}"

tls:
  enabled: false

ffmpeg:
  hwaccel_args: preset-vaapi

detectors:
  ov:
    type: openvino
    device: CPU

model:
  width: 300
  height: 300
  input_tensor: nhwc
  input_pixel_format: bgr
  path: /openvino-model/ssdlite_mobilenet_v2.xml
  labelmap_path: /openvino-model/coco_91cl_bkgr.txt

objects:
  track:
    - person

detect:
  enabled: true
  width: 704
  height: 480
  fps: 5

record:
  enabled: true
  continuous:
    days: 0
  motion:
    days: 0
  alerts:
    retain:
      days: 7
      mode: motion
  detections:
    retain:
      days: 7
      mode: motion

snapshots:
  enabled: true
  retain:
    default: 7

cameras:
  frente_1:
    ffmpeg:
      inputs:
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=1&subtype=1
          roles:
            - detect
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=1&subtype=0
          roles:
            - record
  canal_2:
    ffmpeg:
      inputs:
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=2&subtype=1
          roles:
            - detect
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=2&subtype=0
          roles:
            - record
  canal_3:
    ffmpeg:
      inputs:
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=3&subtype=1
          roles:
            - detect
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=3&subtype=0
          roles:
            - record
  canal_4:
    ffmpeg:
      inputs:
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=4&subtype=1
          roles:
            - detect
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=4&subtype=0
          roles:
            - record
```

### 5. Subir o Frigate e acessar

```
docker compose up -d frigate
docker logs frigate 2>&1 | grep -i -A2 password
```

No primeiro start, o log mostra a senha gerada para o usuário `admin`. Acesse `http://192.168.1.7:8971` e troque a senha. Em Sistema → Métricas, confira o tempo de inferência do detector: abaixo de cerca de 30 ms está bom. Em seguida, passe na frente de uma câmera e veja se a pessoa é detectada.

### 6. Integração com o Home Assistant

No HA, abra HACS → busque **Frigate** → Baixar → reinicie o HA (`docker compose restart homeassistant`). Depois vá em Configurações → Dispositivos e serviços → Adicionar integração → **Frigate** e use a URL `http://172.17.0.1:5000`. A integração cria, para cada câmera, um sensor de presença de pessoa (`binary_sensor.<camera>_person_occupancy`).

### 7. Notificação no iPhone

Automação que avisa no iPhone, com foto, quando uma pessoa for detectada. Ainda falta escrever; ela usa as entidades criadas no passo 6.

### 8. Documentação

Atualizar este arquivo com o resultado, marcar "Frigate" no [ROADMAP](ROADMAP.md) e versionar o `docker-compose.yml` real do servidor. Hoje o arquivo no repositório ainda é um modelo vazio.

## Melhorias futuras

- Restream via go2rtc no Frigate, para ter vídeo ao vivo de baixa latência na interface do Frigate e uma única conexão por câmera com o NVR.
- Zonas e máscaras por câmera, para ignorar a rua e reduzir falsos positivos.
