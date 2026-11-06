# 🏠 Automação Residencial

Projeto de Infraestrutura como Código (IaC) para a automação residencial.

## Objetivos

- Home Assistant como plataforma central
- Zigbee2MQTT como controlador Zigbee
- Frigate para análise de vídeo
- Docker Compose como orquestrador
- Toda a infraestrutura versionada no Git

## Hardware

- AMD Ryzen 7 PRO 5750E
- Ubuntu Server

## Serviços

- Home Assistant
- Mosquitto
- Zigbee2MQTT
- Frigate
- Portainer
- Homepage
- Node-RED
- Dozzle

Consulte a pasta `docs/` para a documentação.

- [Integrações no Home Assistant: interfone Hikvision, fechadura C90, NSPanel Pro, Yoosee, Rossi Nitro e Alexa](docs/INTEGRACOES-HA.md)
- [Coordenador Zigbee: incidente de set–nov/2026, atualização de firmware e recuperação](docs/ZIGBEE-COORDENADOR.md)
- [Acesso ao Home Assistant: rede local e remoto via Tailscale](docs/ACESSO-REMOTO.md)
- [Câmeras Intelbras no Home Assistant: NVR, canais, codificação e RTSP](docs/CAMERAS.md)
- [Frigate: detecção de pessoas nas câmeras e aviso no iPhone](docs/FRIGATE.md)
