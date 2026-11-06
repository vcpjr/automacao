# Frigate: detecção de pessoas nas câmeras

Atualizado em 06/11/2026. **Status: em produção desde 06/11/2026.**

O Frigate analisa as 4 câmeras Intelbras (ver [CAMERAS.md](CAMERAS.md)) e detecta pessoas. Ele grava só os trechos com movimento durante os eventos e manda os eventos para o Home Assistant, que avisa no iPhone com uma foto.

## Resultado

Conferido em 06/11/2026, com as 4 câmeras ligadas:

| Item | Valor |
| --- | --- |
| Câmeras no Frigate | 4: `frente_1`, `frente_2`, `quarto_visitas`, `escritorio` |
| Tempo de inferência do detector `ov` (OpenVINO na CPU) | cerca de 7 ms. Até uns 30 ms seria aceitável. |
| CPU do detector | cerca de 1 núcleo dos 16. O servidor todo ficou em cerca de 8%. |
| Decodificação de vídeo | VAAPI na Radeon integrada. Aparece como `amd-vaapi` em Sistema → Métricas. |
| Integração no HA | Frigate 5.15.6, instalada pelo HACS |
| Teste de ponta a ponta | Uma pessoa passou na frente da câmera do Escritório: o sensor foi para `on` e a notificação chegou no iPhone com foto. |

## Decisões

| Item | Escolha | Motivo |
| --- | --- | --- |
| Versão | `ghcr.io/blakeblackshear/frigate:0.18.2` | Estável atual. A regra do repositório pede para não usar `latest`/`stable`. |
| Detector | OpenVINO na CPU | Padrão do Frigate 0.18 e roda em CPU AMD. Não há Coral, e o ROCm não tem suporte oficial à GPU integrada. |
| Decodificação de vídeo | VAAPI na Radeon integrada (`LIBVA_DRIVER_NAME=radeonsi`) | Tira a decodificação da CPU. |
| Stream de detecção | Stream extra (`subtype=1`, 704x480, H.264) a 5 FPS | É leve e é o suficiente para detectar pessoas. |
| Stream de gravação | Stream principal (`subtype=0`, H.264) | Qualidade total nos clipes. |
| Gravação | Só eventos (alertas e detecções), 7 dias, modo `motion` | O NVR já grava 24h. O disco tinha 77 GB livres, e 24h das 4 câmeras passaria de 100 GB por dia. |
| Nomes das câmeras | `frente_1` (canal 1), `frente_2` (canal 2), `quarto_visitas` (canal 3), `escritorio` (canal 4) | O nome vira o ID das entidades no HA, então não deve ser mudado depois. |
| Interface web | Porta 8971, com login e sem HTTPS (`tls.enabled: false`) | O acesso é só pela rede de casa e pelo Tailscale. O navegador mostra "Não seguro", e isso é esperado. |
| Integração com o HA | `http://172.17.0.1:5000` | O HA roda com `network_mode: host`. A porta 5000 não tem autenticação, então fica ligada só ao IP da bridge do Docker, fora da rede da casa. |
| MQTT | Mosquitto do compose (`mosquitto:1883`), sem usuário e senha | É o mesmo broker que o HA e o Zigbee2MQTT usam. |
| Notificação | Todas as câmeras, a qualquer hora, uma por ocorrência | Escolha do Vilmar. O gatilho é o tópico `frigate/reviews`, com um aviso por alerta novo e sem repetição enquanto a pessoa continua na imagem. |
| Senhas | Arquivo `.env` (fora do Git), lido pelo Frigate como `{FRIGATE_...}` | O repositório é público. |

## Endereços

| O quê | Endereço |
| --- | --- |
| Interface do Frigate em casa | `http://192.168.1.7:8971`, usuário `admin` (a senha não fica no repositório) |
| Interface do Frigate fora de casa | `http://<IP Tailscale do servidor>:8971`, com o Tailscale ligado (ver [ACESSO-REMOTO.md](ACESSO-REMOTO.md)) |
| API sem autenticação, usada pelo HA | `http://172.17.0.1:5000`, que só responde dentro do servidor |

## Arquivos no servidor

| Caminho em `~/automacao` | Conteúdo | No Git? |
| --- | --- | --- |
| `docker-compose.yml` | Serviço `frigate` (passo 3) | Ainda não (ver Pendências) |
| `frigate/config/config.yml` | Configuração do Frigate (passo 4) | Ainda não. O conteúdo está neste documento. |
| `frigate/config/` (demais arquivos) | Banco `frigate.db`, cache de modelos e backups que o Frigate gera | Não |
| `frigate/storage/` | Clipes e fotos | Não (`.gitignore`) |
| `.env` | `FRIGATE_RTSP_USER` e `FRIGATE_RTSP_PASSWORD` (usuário e senha do NVR) | Não (`.gitignore`, `chmod 600`) |
| `homeassistant/config/` | Configuração do HA, montada em `/config` no container | Não |

## Instalação do zero

### 1. Conferência do servidor

Do Terminal do Mac: `ssh vilmar@192.168.1.7`. Depois:

```
ls -l /dev/dri; df -h ~; free -h; nproc
```

Tem que existir `/dev/dri/renderD128`, que é a GPU usada pelo VAAPI. As câmeras precisam estar em H.264 (ver [CAMERAS.md](CAMERAS.md)).

### 2. Pastas, senha e .gitignore

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

### 3. Serviço no docker-compose

Abra `nano docker-compose.yml` e cole o bloco abaixo dentro de `services:`, antes da seção `networks:`, com o mesmo recuo dos outros serviços. Depois valide com `docker compose config -q`; se não aparecer nada, o arquivo está certo.

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
    networks:
      - automacao
```

### 4. config.yml

Crie o arquivo com `nano frigate/config/config.yml` e cole:

```
mqtt:
  host: mosquitto
  port: 1883

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
  frente_2:
    ffmpeg:
      inputs:
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=2&subtype=1
          roles:
            - detect
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=2&subtype=0
          roles:
            - record
  quarto_visitas:
    ffmpeg:
      inputs:
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=3&subtype=1
          roles:
            - detect
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=3&subtype=0
          roles:
            - record
  escritorio:
    ffmpeg:
      inputs:
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=4&subtype=1
          roles:
            - detect
        - path: rtsp://{FRIGATE_RTSP_USER}:{FRIGATE_RTSP_PASSWORD}@192.168.1.10:554/cam/realmonitor?channel=4&subtype=0
          roles:
            - record
```

Observação: no Frigate 0.18, `detect.enabled` vem desligado por padrão, por isso o `enabled: true` acima.

### 5. Subir o Frigate e acessar

```
docker compose up -d frigate
docker logs frigate 2>&1 | grep "Password:"
```

No primeiro start, o log mostra a senha gerada para o usuário `admin`. Os erros `502` do nginx logo no início do log são normais enquanto o Frigate termina de subir.

1. Acesse `http://192.168.1.7:8971`, entre com `admin` e a senha do log e troque a senha: ícone de usuário, embaixo à esquerda → Alterar senha.
2. Na tela inicial devem aparecer as 4 câmeras com imagem. Um ponto vermelho no canto da câmera indica movimento naquele momento.
3. Em Sistema → Métricas, confira o tempo de inferência do detector `ov` (abaixo de cerca de 30 ms está bom) e se aparece a GPU `amd-vaapi`.

### 6. Integração com o Home Assistant

**6.1 Testar a API que o HA vai usar.** Tem que responder `HTTP/1.1 200 OK` e `0.18.2-...`:

```
curl -sS -i http://172.17.0.1:5000/api/version; echo
```

**6.2 Instalar o HACS**, que é a loja de integrações da comunidade. A integração do Frigate não é oficial, por isso vem por ele. Só é preciso fazer isso uma vez:

```
docker exec -it homeassistant bash -c "wget -O - https://get.hacs.xyz | bash -"
cd ~/automacao
docker compose restart homeassistant
```

**6.3 Ativar o HACS.** No HA (recarregue a página com Cmd+Shift+R), vá em Configurações → Dispositivos e serviços → Adicionar integração → **HACS**. Marque as caixas de confirmação e clique em Enviar. Abra `https://github.com/login/device`, entre na conta `vcpjr`, digite o código e clique em **Authorize HACS**. O HACS passa a aparecer na barra lateral.

**6.4 Baixar a integração Frigate.** Vá em HACS → busque **Frigate** (autor Blake Blackshear; não confunda com o Frigate Card ou o Advanced Camera Card) → Baixar. Depois reinicie o HA: `docker compose restart homeassistant`.

**6.5 Adicionar a integração.** Vá em Configurações → Dispositivos e serviços → Adicionar integração → **Frigate**. Use a URL `http://172.17.0.1:5000` e deixe usuário e senha em branco.

> **Atenção:** na tela "Nomear e atribuir", **não renomeie os dispositivos**. Use "Pular e concluir" ou escolha só a área. Na instalação de 06/11/2026, renomear "Escritorio" para "Escritório" fez o HA refazer os IDs como `escritorio_escritorio_*` (ver Diagnóstico).

**6.6 Conferir.** Em Ferramentas de desenvolvedor → Estados, filtre por `person_occupancy`. Têm que aparecer 4 sensores com estado `off`, um por câmera, e o da câmera tem que ir para `on` quando alguém passar na frente dela.

### 7. Notificação no iPhone

Vá em Configurações → Automações e cenas → Criar automação → Criar nova automação → ⋮ → **Editar em YAML**, cole o bloco abaixo e salve. O serviço `notify.mobile_app_iphone_de_vilmar` é o iPhone do Vilmar no app Home Assistant; o nome aparece em Ferramentas de desenvolvedor → Ações, buscando `mobile_app`.

```
alias: Frigate - pessoa detectada
description: Avisa no iPhone, com foto, quando o Frigate detecta uma pessoa em qualquer câmera.
mode: parallel
max: 10
triggers:
  - trigger: mqtt
    topic: frigate/reviews
conditions:
  - condition: template
    value_template: >-
      {{ trigger.payload_json.type == 'new'
         and trigger.payload_json.after.severity == 'alert'
         and 'person' in trigger.payload_json.after.data.objects }}
actions:
  - variables:
      camera: "{{ trigger.payload_json.after.camera }}"
      evento: "{{ trigger.payload_json.after.data.detections[0] }}"
      nomes:
        frente_1: Frente 1
        frente_2: Frente 2
        quarto_visitas: Quarto de visitas
        escritorio: Escritório
  - action: notify.mobile_app_iphone_de_vilmar
    data:
      title: Pessoa detectada
      message: "{{ nomes.get(camera, camera) }} às {{ now().strftime('%H:%M') }}"
      data:
        image: "/api/frigate/notifications/{{ evento }}/snapshot.jpg"
        url: /dashboard-cameras
        group: "frigate-{{ camera }}"
```

Como funciona:

- O Frigate publica em `frigate/reviews` quando começa uma ocorrência. A automação só reage a `type: new` com severidade `alert`, por isso manda um aviso por ocorrência.
- A foto vem do proxy de notificações da integração (`/api/frigate/notifications/<evento>/snapshot.jpg`). O app do iPhone completa o endereço do HA sozinho, então funciona em casa e pelo Tailscale.
- Tocar na notificação abre o painel de câmeras. Segurar a notificação mostra a foto grande.

Para testar, fique fora de todas as câmeras por cerca de 1 minuto e depois passe na frente de uma delas.

## Entidades no Home Assistant

Cada câmera vira um dispositivo da integração Frigate. As entidades mais úteis:

| Câmera | Presença de pessoa | Outras |
| --- | --- | --- |
| Frente 1 | `binary_sensor.frente_1_person_occupancy` | `camera.frente_1`, `image.frente_1_person` (última foto de pessoa), `sensor.frente_1_person_count`, `switch.frente_1_detect`, `switch.frente_1_recordings`, `switch.frente_1_snapshots` |
| Frente 2 | `binary_sensor.frente_2_person_occupancy` | mesmo padrão, com `frente_2` |
| Quarto de visitas | `binary_sensor.quarto_visitas_person_occupancy` | mesmo padrão, com `quarto_visitas` |
| Escritório | `binary_sensor.escritorio_person_occupancy` | mesmo padrão, com `escritorio` |

As câmeras genéricas do painel (por exemplo `camera.escritorio_camera_escritorio`) continuam existindo e são independentes do Frigate.

## Diagnóstico

| Sintoma | Causa e solução |
| --- | --- |
| `curl -s http://172.17.0.1:5000/api/version` parece não responder | A resposta não tem quebra de linha e fica grudada no prompt. Use `curl -sS -i ...; echo` para ver o status e o corpo. |
| Entidades com o nome dobrado (`escritorio_escritorio_*`) | O dispositivo foi renomeado no assistente da integração. A correção abaixo ajusta todas de uma vez no registro de entidades, com o HA parado e com backup. Antes, confira se não há conflito: `docker exec homeassistant grep -oE '[a-z_]+\.[a-z0-9_]*escritorio[a-z0-9_]*' /config/.storage/core.entity_registry \| sort \| uniq -c`. |
| Esqueceu a senha do `admin` do Frigate | Acrescente ao `config.yml` uma seção `auth:` com a opção `reset_admin_password: true` (recuada embaixo dela), rode `docker compose restart frigate` e veja a nova senha com `docker logs frigate 2>&1 \| grep "Password:"`. Depois remova as duas linhas. |
| Câmera sem imagem no Frigate | Confira o canal com o `ffprobe` de [CAMERAS.md](CAMERAS.md) e a senha no `.env`. Depois veja `docker logs frigate 2>&1 \| grep -i error`. |
| Sensor não muda para `on` | Confira se o HA está conectado ao Mosquitto (broker `192.168.1.7`, ver [ACESSO-REMOTO.md](ACESSO-REMOTO.md)) e se `switch.<camera>_detect` está ligado. |

A correção dos IDs aplicada em 06/11/2026 (o backup ficou em `~/core.entity_registry.antes-escritorio`):

```
cd ~/automacao
docker compose stop homeassistant
sudo cp homeassistant/config/.storage/core.entity_registry ~/core.entity_registry.antes-escritorio
sudo sed -i 's/\.escritorio_escritorio/.escritorio/g' homeassistant/config/.storage/core.entity_registry
sudo grep -c 'escritorio_escritorio' homeassistant/config/.storage/core.entity_registry
docker compose start homeassistant
```

O `grep -c` tem que mostrar `0`.

## Pendências

- [ ] Versionar o `docker-compose.yml` real e o `frigate/config/config.yml` do servidor. Hoje o `docker-compose.yml` do repositório ainda é um modelo vazio.
- [ ] Zonas e máscaras por câmera, para ignorar a rua (principalmente na `frente_1`) e reduzir falsos positivos.
- [ ] Restream via go2rtc no Frigate, para ter vídeo ao vivo de baixa latência e uma única conexão por câmera com o NVR.
- [ ] Avaliar trocar as câmeras genéricas do painel do HA pelas entidades `camera.*` do Frigate, o que deixaria uma conexão a menos com o NVR.
- [ ] Usuário só de visualização no NVR, no lugar do `admin` (ver [CAMERAS.md](CAMERAS.md)).
