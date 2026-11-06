# NSPanel Pro (painel da cozinha)

Atualizado em 06/11/2026. **Status:** o painel está configurado e falando com o Home Assistant. Falta a automação da campainha, que depende do interfone Hikvision, ainda não instalado.

O painel roda o app oficial do Home Assistant, que abre sozinho quando o painel liga. Ele mostra os dashboards do HA, recebe notificações e comandos de tela e fala avisos (TTS). Vai ser usado para mostrar o portão quando tocarem a campainha (ver [INTEGRACOES-HA.md](INTEGRACOES-HA.md), Partes 1 e 3).

## Resultado

| Item | Situação em 06/11/2026 |
| --- | --- |
| Firmware | 4.4.8. Já inclui a loja F-Droid, o app do HA e a escolha do app inicial. |
| Conta eWeLink | Painel pareado pelo QR code. |
| App | Home Assistant (app oficial, instalado pelo F-Droid), definido como **app inicial** |
| Usuário no HA | Pessoa **NS Panel**, usuário `painel`, sem administrador e com login só pela rede local |
| Nome no app / serviço de notificação | `painel` → `notify.mobile_app_painel` |
| Conexão persistente | Always |
| Nível de segurança da conexão | O menos seguro, que permite HTTP. O painel nunca sai de casa e o HA usa `http://192.168.1.7:8123`. |
| Testes | TTS ("Teste do painel da cozinha") e notificação comum funcionaram. |

## Decisões

| Item | Escolha | Motivo |
| --- | --- | --- |
| Como mostrar o HA | App oficial do HA, e não uma página web no painel | Só o app recebe notificações, comandos de tela (`command_screen_on`, `command_webview`) e TTS. |
| Usuário | Usuário próprio do painel, sem administrador | O painel fica na parede da cozinha. Logado como administrador, qualquer pessoa poderia mudar configurações ou apagar automações. |
| Zigbee | Não parear dispositivos Zigbee no painel | A rede Zigbee da casa é a do Zigbee2MQTT. Duas redes concorrendo dão problema. |

## Passo a passo

### 1. Alimentação

O painel é ligado direto na rede elétrica (100–240 V AC), pelos bornes **L** e **N** atrás dele, numa caixinha de parede. Ele não vem com plugue. Para ligar em cima da mesa, prenda um cabo com plugue nos bornes L e N com tudo desligado e isole os bornes. Não pegue no painel com ele ligado.

### 2. Configuração inicial

1. Na tela, escolha o país (Brasil) e o idioma (Português).
2. Conecte no Wi-Fi de casa. O painel só funciona em **2,4 GHz**.
3. No app **eWeLink** do iPhone, toque em **+** e leia o QR code da tela.
4. Atualize o firmware em Configurações → Sobre. **Não tire da tomada durante a atualização**: ele reinicia sozinho.

### 3. Usuário do painel no HA

Configurações → Pessoas → Adicionar pessoa. Ligue **Permitir login**, crie o usuário `painel`, ligue **Só pode fazer login pela rede local** e deixe **Administrador** desligado.

### 4. App do HA

1. No painel, deslize de cima para baixo, abra o **F-Droid**, busque **Home Assistant** e instale.
2. Abra o app e conecte em `http://192.168.1.7:8123`. Entre com o usuário `painel`.
3. Dê um nome ao dispositivo. Esse nome vira o serviço `notify.mobile_app_<nome>`.

### 5. Ajustes do app

No menu lateral do HA, no painel, vá até o fim e abra **Configurações do aplicativo** (Companion app) → servidor **Casa**:

- **Persistent connection:** Always. Autorize ignorar a otimização de bateria, se ele pedir.
- **Remote connection security level:** a opção menos segura.
- Na tela anterior, ligue **Fullscreen**.

### 6. Testes

Em Ferramentas de desenvolvedor → Ações → Modo YAML:

```
action: notify.mobile_app_painel
data:
  message: TTS
  data:
    tts_text: Teste do painel da cozinha
    media_stream: alarm_stream_max
```

```
action: notify.mobile_app_painel
data:
  title: Teste
  message: Notificação do Home Assistant
```

O primeiro bloco faz o painel falar a frase. O segundo mostra uma notificação na tela.

### 7. App inicial

Nas configurações nativas da Sonoff, defina o **Home Assistant** como app inicial (Home App). Ao ligar, o painel abre direto no HA; o gesto de voltar leva para a tela da Sonoff.

## Diagnóstico

| Sintoma | Causa e solução |
| --- | --- |
| A página do HA abre numa janela com barra de endereço e o título "WebView Browser Tester" | Esse é o navegador de teste do Android, não o app do HA. Nele não existem as configurações do app, as notificações nem os comandos de tela. Abra o app **Home Assistant** (ícone azul da casinha). |
| Entrou no painel com o usuário administrador | No app, toque no nome do usuário → **Sair**. Exclua o dispositivo em Configurações → Dispositivos e serviços → Mobile App e entre de novo com o usuário `painel`. |
| Não acha as configurações do app | No menu lateral (☰), elas ficam no fim da lista, logo acima do nome do usuário: **Configurações do aplicativo** / Companion app. |
| O serviço `notify.mobile_app_...` do painel não aparece | O registro do painel no HA foi apagado. No app, vá em Companion app → **Casa** → **Delete server** e conecte de novo. Depois recarregue a página das Ferramentas de desenvolvedor com Cmd+Shift+R, porque a lista de ações só é carregada quando a página abre. |

## Próximos passos

- [ ] Instalar o interfone Hikvision (DS-KB8113-IME1) e achar o IP dele: `sudo nmap -sn 192.168.1.0/24 | grep -B2 -i hikvision`.
- [ ] Seguir a Parte 1 de [INTEGRACOES-HA.md](INTEGRACOES-HA.md) (sensor de chamada e câmera do interfone).
- [ ] Criar a automação "Campainha - mostrar portão" (Parte 3, passo 7), que já está com `notify.mobile_app_painel` e `notify.mobile_app_iphone_de_vilmar`.
- [ ] Reservar no DHCP do modem o IP do painel e o do interfone.
