# Acesso ao Home Assistant (local e remoto)

Atualizado em 05/11/2026.

Em casa, o Home Assistant é acessado pelo IP local do servidor. Fora de casa, o acesso passa pelo Tailscale. Assim nenhuma porta fica aberta no modem, e o acesso funciona mesmo que a operadora use CGNAT.

## Endereços

| Onde | URL |
| --- | --- |
| Rede local | `http://192.168.1.7:8123` |
| Fora de casa (Tailscale) | `http://<IP Tailscale do servidor>:8123`. O IP aparece com `tailscale ip -4` no servidor. |

**Decisão:** usar o Tailscale, que é gratuito e não expõe nenhuma porta. O Home Assistant Cloud (Nabu Casa) é pago, e o redirecionamento de portas expõe o servidor à internet.

## Servidor (Ubuntu)

1. Instale o Tailscale:

```
curl -fsSL https://tailscale.com/install.sh | sudo sh
tailscale version
```

2. Conecte o servidor à conta Tailscale. O comando mostra um link de autenticação; abra no navegador e autorize.

```
sudo tailscale up
```

3. Anote o IP Tailscale do servidor:

```
tailscale ip -4
```

4. Desative a expiração da chave do servidor: em https://login.tailscale.com/admin/machines, abra o menu ⋯ da máquina e escolha **Disable key expiry**. Sem isso, o servidor sai da rede Tailscale a cada 180 dias.

O Home Assistant roda com `network_mode: host`, então já responde também na interface do Tailscale. Não é preciso nenhuma configuração extra.

## iPhone

1. Instale o app **Tailscale**, entre com a mesma conta e mantenha a conexão ligada. A opção "VPN sob demanda" reconecta sozinha.
2. No app **Home Assistant**, vá em Configurações → Servidores → **Casa** e preencha:
   - **URL interna:** `http://192.168.1.7:8123`. Em SSIDs, coloque o nome do Wi-Fi de casa e dê permissão de localização "Sempre".
   - **URL externa:** `http://<IP Tailscale do servidor>:8123`.
3. Teste com o Wi-Fi desligado, só no 4G/5G e com o Tailscale ligado: abra o app e acione um relé.

## Diagnóstico rápido

| Sintoma | O que verificar |
| --- | --- |
| O app e o navegador não abrem o HA na rede local. | No servidor, `curl -sI http://localhost:8123 \| head -1` deve responder `HTTP/1.1 405` (o HA está no ar). Depois confira o IP com `hostname -I`; a URL tem que ter os 4 números do IP. Por fim, `sudo ufw status`. |
| Dispositivos aparecem como "Indisponível" e a entidade diz que "não está mais sendo fornecida pela integração mqtt". | O HA não está conectado ao Mosquitto. Como ele roda com `network_mode: host`, não enxerga o nome `mosquitto`. Na integração MQTT, use o broker `192.168.1.7`, porta `1883`. |
| Fora de casa não abre. | O Tailscale tem que estar ligado no iPhone, e o servidor tem que aparecer como conectado no painel do Tailscale. |
| Esqueceu a senha do HA. | `docker exec homeassistant hass --script auth -c /config change_password <usuario> '<nova senha>'` e depois `docker compose restart homeassistant`. |

## Pendências

- [ ] Reservar no DHCP do modem os IPs `192.168.1.7` (servidor) e `192.168.1.10` (NVR), para que não mudem.
