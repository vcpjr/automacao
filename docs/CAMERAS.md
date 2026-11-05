# Câmeras Intelbras no Home Assistant

Atualizado em 05/11/2026.

As 4 câmeras Intelbras chegam ao Home Assistant através do NVR NVD 1408-P, por RTSP. O HA usa o stream extra (704x480, H.264) e mostra as câmeras no painel **Câmeras** (`/dashboard-cameras`).

## Topologia

As câmeras ficam nas portas PoE do NVR, numa sub-rede interna (`10.1.1.x`) que não é acessível pela rede da casa. Na rede da casa só aparece o NVR, em `192.168.1.10`, e o HA pega o vídeo de cada câmera pelo número do canal no NVR.

| Canal | IP interno | Modelo | Situação em 05/11/2026 |
| --- | --- | --- | --- |
| 1 | 10.1.1.128 | VIP Intelbras (RXXK0600797LR), 4MP 2560x1440 | Online. No HA: "Câmera da frente 1". |
| 2 | 10.1.1.129 | VIPC-1430-B | Online |
| 3 | 10.1.1.132 | VIPC-1230-B-G2 | Online |
| 4 | 10.1.1.130 | VIPC-1230-B-G2 | Online |
| 5 | 10.1.1.133 | VIPC-1230-B-G2 | Offline. Provavelmente é um cadastro antigo (ver Pendências). |
| 7 | 10.1.1.128 | Mesma câmera do canal 1 | Cadastro duplicado (ver Pendências). |

No HA, as câmeras dos canais 2 a 4 aparecem como "Câmera da frente 2", "Câmera Escritório" e "Câmera quarto de visitas".

## Codificação

**Decisão:** os dois streams de cada câmera ficam em **H.264**. O stream extra fica em 704x480; o recomendado é 15 FPS e 1024 kb/s. O H.265 só funciona em alguns navegadores e aparelhos e costuma dar tela preta no HA. O custo do H.264 é ocupar mais espaço nas gravações do NVR.

Para alterar: na interface web do NVR (`http://192.168.1.10`), vá em Câmera → **Encoder**, escolha o canal, ajuste **Stream principal** e **Stream Extra** e clique em **Salvar**. Em seguida clique em **Atualizar** e confira se a mudança ficou.

**Câmera do canal 1:** ela não aceita a configuração enviada pelo NVR e volta para H.265. Faça a alteração na própria câmera: Câmera → Dispositivos → ícone **"e"** (coluna Página) na linha do canal 1 → Câmera → Vídeo. Desligue o Smart Codec, se estiver ligado.

## RTSP

```
rtsp://192.168.1.10:554/cam/realmonitor?channel=<canal>&subtype=1
```

`subtype=1` é o stream extra, usado no HA. `subtype=0` é o stream principal.

Para testar os 4 canais a partir do servidor (todos devem mostrar `codec_name=h264`):

```
for c in 1 2 3 4; do echo "canal $c:"; docker exec homeassistant ffprobe -v error -rtsp_transport tcp -show_entries stream=codec_name,width,height -of compact "rtsp://<usuario>:<senha>@192.168.1.10:554/cam/realmonitor?channel=$c&subtype=1"; done
```

Se a senha tiver `@`, `#` ou `:`, escreva-os como `%40`, `%23` e `%3A`.

## Configuração no Home Assistant

Cada câmera é uma integração **Câmera genérica** (Configurações → Dispositivos e serviços):

- **URL do stream:** a URL RTSP acima, com o canal da câmera e sem usuário e senha.
- **Protocolo RTSP:** TCP.
- **Usuário** e **Senha:** os do NVR, preenchidos nos campos próprios.
- **URL da imagem estática:** em branco.

O painel usa cards de imagem: cada câmera mostra uma foto atualizada a cada 10 segundos e, ao tocar, abre o vídeo ao vivo. O estado "Ocioso" é normal; ele só indica que o HA não está gravando.

## Pendências

- [ ] Canal 5 offline: se não houver uma 5ª câmera instalada, apagar o cadastro no NVR. Se houver, verificar o cabo e a porta PoE.
- [ ] Confirmar a remoção do canal 7, que é o cadastro duplicado da câmera do canal 1.
- [ ] Criar no NVR um usuário só de visualização, com senha própria, e usá-lo no HA no lugar do `admin`.
- [ ] Frigate, para detecção de pessoas e veículos e gravação inteligente (Sprint 3 do [ROADMAP](ROADMAP.md)).
