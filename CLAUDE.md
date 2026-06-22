# CLAUDE

## Objetivo

Este repositório representa toda a infraestrutura da automação residencial.

### Diretrizes

- Priorizar estabilidade.
- Evitar versões `latest` em produção.
- Home Assistant é o núcleo da automação.
- MQTT é o barramento de mensagens.
- Zigbee2MQTT é o coordenador Zigbee principal.
- Documentar toda decisão arquitetural relevante.
- Toda mudança deve preservar a capacidade de reinstalação completa do ambiente.
