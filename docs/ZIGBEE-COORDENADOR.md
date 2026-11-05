# Coordenador Zigbee (Sonoff ZBDongle-P)

Atualizado em 05/11/2026.

O coordenador Zigbee travou em setembro de 2026 e deixou a rede Zigbee fora do ar por cerca de 6 semanas. Em 05/11/2026 ele foi recuperado com a atualização do firmware para a versão 20240710, sem precisar parear nenhum dispositivo de novo.

## Configuração atual

| Item | Valor |
| --- | --- |
| Adaptador | Sonoff Zigbee 3.0 USB Dongle Plus (ZBDongle-P), CC2652P, ponte USB CP2102N |
| Caminho no host | `/dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_e40511843ff5ef118f60a2a29ed47d52-if00-port0` (aponta para `/dev/ttyUSB0`) |
| Driver no Zigbee2MQTT | `zstack` |
| IEEE do coordenador | `0x00124b0039de5029` |
| Firmware | Z-Stack 3.x.0, revision `20240710` (antes era `20210708`) |
| Zigbee2MQTT | 2.12.0 |
| Pasta de dados | `./zigbee2mqtt/data` (montada em `/app/data`) |
| Reserva | Um segundo ZBDongle-P (ver [INVENTARIO.md](INVENTARIO.md)) |

Dispositivos na rede em 05/11/2026:

| Nome | IEEE | Modelo |
| --- | --- | --- |
| Sacada do escritório | `0xc02cedfffe748f51` | ZBMINIR2 (roteador) |
| Sacada da suíte | `0xc02cedfffe74122a` | ZBMINIR2 (roteador) |
| Suíte | `0xc02cedfffeec9570` | ZBMINIR2 (roteador) |

## Incidente de set–nov/2026

| Data e hora | Evento |
| --- | --- |
| 02/09/2026 16:41 | O Zigbee2MQTT inicia com o adaptador `zstack` em `/dev/ttyUSB0`. |
| 03/09 a 20/09/2026 | O backup diário do coordenador (`coordinator_backup.json`) é gravado normalmente. |
| Após 20/09/2026 | O backup falha em `AdapterBackup.getAdapterVersion` e o dongle para de responder. A rede Zigbee fica sem coordenador. |
| 05/11/2026 16:20 | O `docker compose up -d` falha no zigbee2mqtt com `Error gathering device information while adding custom device "/dev/serial/..."`, porque o dongle não aparece no sistema. |
| 05/11/2026 16:22 | O dongle volta a ser reconhecido (o link em `/dev/serial/by-id/` é recriado) e o container sobe. |
| 05/11/2026 16:25 | O Zigbee2MQTT inicia, mas os 3 relés ficam offline com `NWK_NO_ROUTE`. |
| 05/11/2026 16:54 | O firmware 20240710 é gravado com o cc2538-bsl. |
| 05/11/2026 16:56 | O log mostra `zigbee-herdsman started (restored)`: a rede foi restaurada do backup. |
| 05/11/2026 16:57 | Os 3 relés voltam a ficar online. |

**Causa provável:** o dongle travou rodando o firmware de fábrica (`20210708`), que é antigo. Com isso, ele deixou de ser reconhecido no USB. O caminho no `docker-compose.yml` estava correto: o erro do Docker apareceu porque o dispositivo não existia no momento.

**Decisão:** manter o firmware Z-Stack `20240710`, que é a versão estável e mais testada para o ZBDongle-P. A `20250321`, mais nova, foi deixada de lado por prioridade à estabilidade.

## Como atualizar o firmware do dongle

Tudo é feito no servidor, dentro de `~/automacao`. A gravação apaga a memória do dongle. Na partida seguinte, o Zigbee2MQTT restaura a rede a partir do `coordinator_backup.json`. Isso só funciona com o mesmo dongle, porque o IEEE do coordenador precisa continuar igual.

1. Pare o Zigbee2MQTT e faça backup dos dados:

```
cd ~/automacao
docker compose stop zigbee2mqtt
sudo cp -a ./zigbee2mqtt/data ~/z2m-backup-antes-firmware
ls -l ~/z2m-backup-antes-firmware/coordinator_backup.json
```

2. Instale a ferramenta de gravação (só na primeira vez):

```
sudo apt update && sudo apt install -y python3-venv git unzip wget
python3 -m venv ~/cc2538
~/cc2538/bin/pip install "git+https://github.com/JelmerT/cc2538-bsl.git" intelhex
```

3. Baixe e extraia o firmware:

```
mkdir -p ~/firmware && cd ~/firmware
wget https://github.com/Koenkk/Z-Stack-firmware/releases/download/Z-Stack_3.x.0_coordinator_20240710/CC1352P2_CC2652P_launchpad_coordinator_20240710.zip
unzip CC1352P2_CC2652P_launchpad_coordinator_20240710.zip
```

4. Grave o firmware. Leva cerca de 1 minuto e deve terminar com `Verified`. Não desconecte o dongle durante a gravação.

```
sudo ~/cc2538/bin/cc2538-bsl -p /dev/ttyUSB0 --bootloader-sonoff-usb -e -w -v CC1352P2_CC2652P_launchpad_coordinator_20240710.hex
```

5. Tire o dongle do USB, espere 5 segundos, coloque de novo e confira:

```
ls /dev/serial/by-id/
```

6. Suba o Zigbee2MQTT e confira o log:

```
cd ~/automacao && docker compose start zigbee2mqtt
docker compose logs --tail 50 zigbee2mqtt | grep -i -E "revision|restored|resumed|started"
```

O log deve mostrar `zigbee-herdsman started (restored)`, `"revision":20240710` e `Zigbee2MQTT started!`.

7. Acione os interruptores dos relés, espere 2 minutos e confirme que voltaram:

```
docker compose logs --since 3m zigbee2mqtt | grep -i -E "availability|NO_ROUTE"
```

Cada dispositivo deve aparecer com `{"state":"online"}`.

## Diagnóstico rápido

| Sintoma | Causa | O que fazer |
| --- | --- | --- |
| `Error gathering device information while adding custom device "/dev/serial/..."` | O dongle não está aparecendo no sistema. | Rode `ls -l /dev/serial/by-id/` e `lsusb`. Se não aparecer, reconecte o dongle ou troque de porta USB. Se o `dmesg` citar `brltty`, rode `sudo apt remove brltty`. |
| O caminho existe, mas o erro continua. | O `by-id` no compose é diferente do dongle conectado. | Copie o caminho exato de `ls /dev/serial/by-id/` para o `devices:` do serviço zigbee2mqtt. |
| Dispositivos offline com `NWK_NO_ROUTE` | O coordenador ficou fora do ar ou a rota se perdeu. | Acione o interruptor físico e espere até 10 min (intervalo do ping do Zigbee2MQTT). Se não voltar, desligue o disjuntor do relé por 10 s. Parear de novo é o último recurso. |
| O backup diário do coordenador para de aparecer no log. | O dongle travou. | Siga o passo a passo de atualização acima. |

## Prevenção e pendências

- [ ] Ligar o dongle com um cabo extensor USB de 1 a 2 m, longe das portas USB 3.0 e do gabinete.
- [ ] Apagar `~/z2m-backup-antes-firmware` depois de alguns dias de operação estável.
- Conferir de vez em quando se o backup diário continua aparecendo: `docker compose logs zigbee2mqtt | grep "Wrote coordinator backup" | tail -3`.
- Se o dongle der defeito, use o ZBDongle-P reserva. Ele tem outro IEEE, então antes de restaurar a rede é preciso gravar nele o IEEE `0x00124b0039de5029` com o `cc2538-bsl` (opção `--ieee-address`). Sem isso, todos os dispositivos precisam ser pareados de novo.

## Fontes

- [Koenkk/Z-Stack-firmware — coordinator 20240710](https://github.com/Koenkk/Z-Stack-firmware/releases/tag/Z-Stack_3.x.0_coordinator_20240710)
- [JelmerT/cc2538-bsl](https://github.com/JelmerT/cc2538-bsl)
- [Koenkk/zigbee2mqtt — Discussion #20724 (restauração e IEEE)](https://github.com/Koenkk/zigbee2mqtt/discussions/20724)
