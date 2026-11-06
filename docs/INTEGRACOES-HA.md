# Home Assistant — Interfone, NSPanel, Yoosee e Rossi

Atualizado em 03/11/2026.

## Antes de começar

O guia considera o HA rodando em Docker no Ubuntu (instalação Container, sem add-ons). "Reiniciar o HA" = `docker restart homeassistant` (troque pelo nome do seu container, veja em `docker ps`).

1. Confirme que o container do HA roda com `network_mode: host` no docker-compose. Sem isso a descoberta de dispositivos e o Matter não funcionam.
2. No modem, reserve IP fixo (reserva DHCP) para cada equipamento e preencha a tabela abaixo.
3. Instale o HACS, se ainda não tiver:

```
docker exec -it homeassistant bash
wget -O - https://get.hacs.xyz | bash -
exit
docker restart homeassistant
```

4. No HA: Configurações → Dispositivos e serviços → Adicionar integração → HACS → autorize com sua conta GitHub.
5. Senhas vão no `secrets.yaml` (mesma pasta do `configuration.yaml`), nunca direto no YAML.

| Equipamento | IP fixo | Usuário | Senha |
| --- | --- | --- | --- |
| Servidor HA |  | — | — |
| Interfone Hikvision |  | admin |  |
| NSPanel Pro |  | — | — |
| Câmera Yoosee |  | admin |  |
| Módulo do portão (Rossi) |  | — | — |

## Parte 1 — Interfone Hikvision (vídeo e campainha)

O HA fala direto com o interfone pela rede local: vídeo por RTSP e status da chamada por ISAPI, sem nuvem Hik-Connect.

1. Pelo navegador, entre no IP do interfone como admin. Confirme vídeo em H.264. Se existir a opção de protocolo de integração (ISAPI) em Rede → Avançado, habilite.
2. No servidor Ubuntu, teste o status de chamada (troque IP e senha). Deve voltar `idle`; aperte o botão do interfone e repita: deve voltar `ring`.

```
curl -s --digest -u admin:SENHA "http://IP_INTERFONE/ISAPI/VideoIntercom/callStatus?format=json"
```

3. Acrescente no `secrets.yaml`:

```
hik_user: admin
hik_pass: SENHA_DO_INTERFONE
hik_callstatus: http://IP_INTERFONE/ISAPI/VideoIntercom/callStatus?format=json
hik_porta: http://IP_INTERFONE/ISAPI/AccessControl/RemoteControl/door/1
```

4. Acrescente no `configuration.yaml` e reinicie o HA. Cria o `sensor.interfone_chamada` (idle / ring / onCall):

```
rest:
  - resource: !secret hik_callstatus
    authentication: digest
    username: !secret hik_user
    password: !secret hik_pass
    scan_interval: 2
    sensor:
      - name: "Interfone chamada"
        unique_id: interfone_chamada
        value_template: "{{ value_json.CallStatus.status }}"
```

5. Câmera: Configurações → Dispositivos e serviços → Adicionar integração → Câmera genérica (Generic Camera). Preencha:
   - URL do stream: `rtsp://admin:SENHA@IP_INTERFONE:554/Streaming/Channels/101`
   - URL da imagem estática: em branco
   - Protocolo RTSP: TCP · Autenticação: digest
6. Em Configurações → Entidades, renomeie a câmera para `camera.interfone`. O canal `102` é o substream, mais leve, se o vídeo engasgar.

Senha com `@`, `#` ou `:` quebra a URL RTSP: troque a senha ou codifique (`@` vira `%40`).

Se o sensor nunca sair de `idle` ao tocar, o firmware não expõe o status de chamada. Alternativa: o container [hikvision-doorbell](https://github.com/pergolafabio/Hikvision-SDK-Addon) roda standalone no Docker e entrega campainha e abertura via MQTT.

## Parte 2 — Fechadura HDL C90 (abrir pelo HA)

A C90 não conversa com o HA: ela está ligada no relé Door 1 do interfone. Abrir pelo HA é pedir ao interfone que pulse esse relé (1 s, já configurado na instalação).

1. Teste no servidor. Deve responder `HTTP/1.1 200 OK` e a fechadura destravar. Erro 401 = usuário/senha; 403 ou 404 = ISAPI desabilitado ou número da porta errado.

```
curl -i --digest -u admin:SENHA -X PUT -d '<RemoteControlDoor><cmd>open</cmd></RemoteControlDoor>' http://IP_INTERFONE/ISAPI/AccessControl/RemoteControl/door/1
```

2. Acrescente no `configuration.yaml` e reinicie o HA:

```
rest_command:
  abrir_portao_pedestre:
    url: !secret hik_porta
    method: put
    authentication: digest
    username: !secret hik_user
    password: !secret hik_pass
    payload: "<RemoteControlDoor><cmd>open</cmd></RemoteControlDoor>"
    content_type: "application/xml"
```

3. Crie o script: Configurações → Automações e cenas → Scripts → Novo → menu ⋮ → Editar em YAML. Salve com o ID `abrir_portao_pedestre`.

```
alias: Abrir portão pedestre
icon: mdi:door-open
sequence:
  - action: rest_command.abrir_portao_pedestre
```

4. No seu painel (dashboard), crie uma aba chamada Portão com caminho de URL `portao`. Nela, adicione um card Manual:

```
type: vertical-stack
cards:
  - type: picture-entity
    entity: camera.interfone
    camera_view: live
    show_state: false
    show_name: false
  - type: button
    name: Abrir portão
    icon: mdi:door-open
    tap_action:
      action: perform-action
      perform_action: script.abrir_portao_pedestre
      confirmation:
        text: Abrir o portão de pedestre?
```

5. A C90 não informa se o portão ficou aberto. Se quiser esse estado, cole um sensor de porta Zigbee no portão e pareie no seu dongle Sonoff.

## Parte 3 — NSPanel Pro

O caminho mais simples é rodar o app oficial do HA no próprio painel, instalado pela loja F-Droid que vem no firmware 4.x. Ele mostra o mesmo dashboard do celular, com vídeo e botões.

**Passos 1 a 6 feitos em 06/11/2026.** O resultado e os problemas encontrados estão em [NSPANEL-PRO.md](NSPANEL-PRO.md). O serviço do painel ficou `notify.mobile_app_painel`, e o do iPhone é `notify.mobile_app_iphone_de_vilmar`. O passo 7 espera a instalação do interfone.

1. No app eWeLink, atualize o NSPanel Pro para o firmware mais recente (4.2 ou superior permite escolher o app de inicialização).
2. No HA, crie um usuário só para o painel: Configurações → Pessoas → Adicionar → usuário `painel`, sem permissão de admin.
3. No painel: deslize de cima para baixo → F-Droid → busque Home Assistant → Instalar.
4. Abra o app. Ele acha o servidor sozinho; se não achar, digite `http://IP_DO_HA:8123`. Entre com o usuário `painel` e dê um nome ao dispositivo. O nome vira o serviço `notify.mobile_app_<nome>`; na instalação de 06/11/2026 ficou `painel`.
5. No app: Configurações → Companion app → seu servidor → Conexão persistente → Sempre. Sem isso, os comandos de tela chegam atrasados.
6. Nas configurações avançadas do painel, em gerenciamento de apps, defina o Home Assistant como app de inicialização. O botão lateral volta para a tela nativa da Sonoff.
7. Crie a automação que acorda o painel na aba Portão quando tocam a campainha (Automações → Nova → ⋮ → Editar em YAML). Os nomes dos serviços aparecem em Ferramentas do desenvolvedor → Ações → `notify.mobile_app_`. Confira o caminho da aba abrindo-a no navegador (ex.: `/lovelace/portao`).

```
alias: Campainha - mostrar portão
mode: single
triggers:
  - trigger: state
    entity_id: sensor.interfone_chamada
    to: ring
actions:
  - action: notify.mobile_app_painel
    data:
      message: command_screen_on
  - action: notify.mobile_app_painel
    data:
      message: command_webview
      data:
        command: /lovelace/portao
  - action: notify.mobile_app_iphone_de_vilmar
    data:
      title: Campainha
      message: Tem alguém no portão
      data:
        image: /api/camera_proxy/camera.interfone
  - action: notify.mobile_app_painel
    data:
      message: TTS
      data:
        tts_text: Campainha. Tem alguém no portão.
        media_stream: alarm_stream_max
```

8. Teste: toque o interfone. Em até 2 s o painel acende na aba Portão, fala o aviso de campainha e o celular recebe a foto.

O NSPanel não tem campainha nativa para o Hikvision: a campainha dele só dispara por cenas do eWeLink, e o interfone não é dispositivo eWeLink. Por isso o aviso vem do HA. O painel também não atende a chamada; para falar com o visitante, use o Hik-Connect no celular.

Como o HA já tem o dongle Zigbee, não pareie dispositivos Zigbee no NSPanel: evita duas redes Zigbee concorrendo. Se um dia quiser o vídeo sem o HA no meio, o painel também exibe câmera RTSP nativamente (use o canal `102` do interfone).

## Parte 4 — Câmera Wi-Fi Yoosee

A Yoosee entrega vídeo RTSP local na porta 554 e ONVIF na porta 5000; o HA lê direto da câmera, sem nuvem.

1. No app Yoosee, conecte a câmera ao Wi-Fi 2,4 GHz da casa.
2. Nas configurações da câmera no app, ative o RTSP (em algumas versões aparece como Conexão NVR) e defina uma senha. Anote na tabela do início.
3. Reserve IP fixo para a câmera no modem.
4. Teste no VLC do PC (Mídia → Abrir fluxo de rede). `onvif1` é o stream principal (HD); `onvif2` é o secundário, mais leve.

```
rtsp://admin:SENHA@IP_YOOSEE:554/onvif1
```

5. No HA: Adicionar integração → Câmera genérica. Preencha:
   - URL do stream: `rtsp://admin:SENHA@IP_YOOSEE:554/onvif1`
   - URL da imagem estática: em branco
   - Protocolo RTSP: UDP (a Yoosee costuma transmitir só por UDP; se não abrir, tente TCP ou `onvif2`)
6. Renomeie a entidade para `camera.yoosee`.
7. Se o modelo gira (PTZ), adicione também a integração ONVIF: host = IP da câmera, porta `5000`, usuário `admin`, mesma senha. Ela cria os comandos de movimento.
8. Card (Manual), na aba que preferir:

```
type: picture-entity
entity: camera.yoosee
camera_view: live
show_state: false
```

Opcional: bloquear a internet da câmera no modem deixa tudo local; em troca, o app Yoosee para de funcionar fora de casa.

## Parte 5 — Motor Rossi Nitro (portão da garagem)

O HA aciona o motor pela entrada de botoeira da central, o mesmo ponto usado para botão externo ou interfone. Um Shelly Plus Uni fecha esse contato por 1 s; funciona com qualquer central Nitro e não interfere no controle remoto.

Material:

- Shelly Plus Uni (alimentação 9–28 Vcc, saídas de contato seco até 30 V)
- Sensor magnético com fio, tipo alarme (opcional, para saber se o portão está fechado)
- Fio de 2 vias

1. Desligue o disjuntor do motor.
2. Na etiqueta ou no manual da central, localize: botoeira `BOT` (centrais mais novas: `BOT_A`), o comum `GND` (algumas usam `CM`) e a saída de acessórios `12V`/`GND`.
3. Faça a ligação. Atenção: o Shelly tem vários fios pretos; siga a etiqueta dele.

```
Central 12V  -> Shelly VAC1 (vermelho)
Central GND  -> Shelly VAC2 (preto)
Shelly OUT1 (par de fios) -> BOT (ou BOT_A) e GND (ou CM)
Sensor magnetico (opcional) -> Shelly IN1 (laranja) e GND (verde)
```

4. Religue o disjuntor. No celular, conecte na rede Wi-Fi `ShellyPlusUni-XXXX`, abra `192.168.33.1` e configure o Wi-Fi 2,4 GHz da casa. Reserve o IP no modem.
5. Abra a página do Shelly pelo novo IP e ajuste:
   - Output 1: Auto off = 1 s (vira pulso de botoeira)
   - Input 1: tipo Switch, desvinculado da saída (detached)
   - Atualize o firmware
6. No HA, o Shelly aparece como descoberto em Dispositivos e serviços → Configurar. Renomeie as entidades para `switch.portao_garagem_pulso` (saída) e `binary_sensor.portao_garagem_fechado` (entrada; ative-a se vier desativada).
7. Crie o script com o ID `acionar_portao_garagem`:

```
alias: Acionar portão garagem
icon: mdi:gate
sequence:
  - action: switch.turn_on
    target:
      entity_id: switch.portao_garagem_pulso
```

8. Card (Manual), na aba Portão para aparecer também no NSPanel:

```
type: tile
entity: binary_sensor.portao_garagem_fechado
name: Portão garagem
tap_action:
  action: perform-action
  perform_action: script.acionar_portao_garagem
  confirmation:
    text: Acionar o portão da garagem?
```

9. Teste: cada toque equivale a um aperto de botoeira (abre, para ou fecha, conforme o modo da central). Se o sensor mostrar invertido, marque Invert no Input 1 do Shelly.

Como o portão vai se mover sem você estar vendo, confirme que fotocélula e auto-reverso da central estão ativos. Se o kit veio com a placa Wi-Fi Simply Connect, ela pode continuar; não encontrei integração dela com o HA, por isso o caminho aqui é a botoeira.

## Parte 6 — Alexa (aviso de campainha)

A Echo de 2ª geração anuncia quando alguém aperta o interfone. Quem faz isso é a integração oficial Alexa Devices (HA 2025.6 ou superior), via conta Amazon; não precisa de HACS.

1. Nas configurações de segurança da conta Amazon, deixe o app autenticador como método preferido da verificação em duas etapas. A integração exige.
2. No app Alexa, confirme que a Echo está na sua própria conta (dispositivo compartilhado pelo Amazon Família não funciona) e que o Não perturbe está desligado.
3. No HA: Configurações → Dispositivos e serviços → Adicionar integração → Alexa Devices. Entre com e-mail, senha e o código do autenticador.
4. Abra o dispositivo da Echo e anote a entidade de notificação: `notify.NOME_announce` (anúncio com bipe antes) ou `notify.NOME_speak` (fala direta).
5. Teste em Ferramentas do desenvolvedor → Ações, modo YAML:

```
action: notify.send_message
target:
  entity_id: notify.echo_dot_announce
data:
  message: Teste da campainha
```

6. Abra a automação "Campainha - mostrar portão" (Parte 3), modo YAML, e acrescente no fim de `actions`:

```
  - action: notify.send_message
    target:
      entity_id: notify.echo_dot_announce
    data:
      message: Tem alguém no portão
```

7. Teste: aperte o botão do interfone. A Alexa anuncia em poucos segundos; o aviso passa pela nuvem da Amazon, por isso chega um pouco depois do NSPanel.

Se a Alexa ficar muda: confira o Não perturbe (a integração cria um switch para ele) e o volume da Echo. Se o anúncio não sair, troque `_announce` por `_speak`.

## Checklist final

Rode na ordem; cada item depende do anterior.

- [ ] curl do status de chamada responde `idle` e, ao tocar, `ring`
- [ ] curl de abrir porta destrava a C90
- [ ] `camera.interfone` com vídeo ao vivo no HA
- [ ] Botão Abrir portão pede confirmação e abre a C90
- [ ] NSPanel abre o HA ao ligar e mostra a aba Portão
- [ ] Tocar o interfone acende o NSPanel na aba Portão, a Alexa anuncia e o celular recebe a foto
- [ ] `camera.yoosee` com vídeo ao vivo
- [ ] Card do portão da garagem aciona o motor; o controle remoto continua funcionando
- [ ] Sensor do portão da garagem mostra aberto/fechado certo
- [ ] Reinicie o HA e repita os testes de abrir portão e campainha

## Fontes

- [Comandos ISAPI de abrir porta e status de chamada (fórum HA)](https://community.home-assistant.io/t/ds-kd8003-ds-kv8113-wme1-integration-hikvision-hik-connect-video-intercom/238535)
- [hikvision-doorbell: add-on e container standalone](https://github.com/pergolafabio/Hikvision-SDK-Addon)
- [RESTful Command com autenticação digest](https://www.home-assistant.io/integrations/rest_command)
- [App do HA no NSPanel Pro via F-Droid](https://sonoff.tech/en-eu/blogs/news/f-droid-new-features-bring-unlimited-possibilities-your-nspanel-pro-more-than-just-a-control-panel)
- [Notas de versão do NSPanel Pro](https://sonoff.tech/en-de/blogs/news/sonoff-nspanel-pro-version-update-information-and-faq)
- [Conexão persistente e comandos de tela no app Android](https://github.com/thomasloven/hass-browser_mod/wiki/Android-Devices-'Always-On')
- [URLs RTSP da Yoosee (onvif1/onvif2)](https://community.home-assistant.io/t/trying-to-get-ip-camera-to-work-yoosee/32842?page=2)
- [Manual da central W Nitro: bornes BOT\_A, BOT\_F e GND](https://www.rossiportoes.com.br/uploads/68af5432e2471_19498_.pdf)
- [Especificações do Shelly Plus Uni](https://www.shelly.com/blogs/documentation/shelly-plus-uni)
- [Integração Alexa Devices](https://home-assistant.io/integrations/alexa_devices)
