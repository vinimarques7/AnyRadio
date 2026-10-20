# AnyRadio — Arquitetura (rascunho para discussão)

> Status: **proposta**. Os pontos marcados com **[DECIDIR]** precisam de acordo do grupo antes de virarem spec.
> Fontes: `README.md`, `docs/Rádio Wi-Fi com ESP32 + Raspberry Pi — Guia do Projeto.docx.md`, `images/arquitetura.png`.

## 1. Contexto em uma frase

O ESP32 é o rádio (Wi-Fi → buffer → MP3 → I2S, tudo em FreeRTOS). O Raspberry Pi é só a tela: lê telemetria pela UART e mostra "Tocando agora" e "Monitor do RTOS". O Pi **nunca** toca áudio.

O objetivo da disciplina é o FreeRTOS. Por isso a arquitetura prefere **deixar visível** cada mecanismo (tasks, stream buffer, fila, semáforo, mutex, event group) a escondê-lo dentro de uma biblioteca.

## 2. Decisões de firmware

### 2.1 Framework no PlatformIO — **[DECIDIR]**

| Opção | Prós | Contras |
| :-- | :-- | :-- |
| **A. `framework = espidf` (recomendada)** | `menuconfig`/`sdkconfig` liberados (`configUSE_TRACE_FACILITY`, `configGENERATE_RUN_TIME_STATS`); `uart_driver_install` com fila de eventos é nativo; driver I2S novo (`i2s_std`); casa com o tema da disciplina | Precisamos montar o pipeline de áudio (HTTP + decoder MP3), sem o "tudo pronto" da ESP32-audioI2S |
| B. `framework = arduino` + ESP32-audioI2S | Rádio tocando em poucas linhas | A lib faz HTTP + decode + I2S **dentro de `audio.loop()`**: o produtor-consumidor e o stream buffer ficam escondidos. As versões atuais exigem o core Arduino 3.x, que o PlatformIO oficial não traz (teria que usar o fork *pioarduino*). O `sdkconfig` do Arduino é pré-compilado, então habilitar run-time stats fica difícil |
| C. `framework = arduino, espidf` (híbrido) | Junta as duas coisas | Build lento e frágil; mais ponto de falha do que ganho |

**Recomendação:** A. O que o grupo ganharia com a lib (B) é justamente o que a disciplina quer ver implementado: a task de rede enchendo o buffer e a de áudio esvaziando.

### 2.2 Pipeline de áudio

```
[net_task] --HTTP chunks--> (StreamBuffer 96 KB em PSRAM) --> [audio_task] --MP3 decode--> PCM --> I2S --> MAX98357A
    core 0                                                     core 1, maior prioridade
```

- **HTTP:** `esp_http_client`, só streams **HTTP** MP3 128 kbps (sem HTTPS no MVP, que gasta muita RAM com TLS). Header `Icy-MetaData` desligado no MVP.
- **Decoder:** candidato `esp-libhelix-mp3` (ESP Component Registry, via `idf_component.yml`). Alternativa: `espressif/esp_audio_codec`. **[DECIDIR] na spec.**
- **Buffer:** `xStreamBufferCreate` com memória em PSRAM. 96 KB ≈ 6 s a 128 kbps, o bastante para a demo "derruba o Wi-Fi e o som continua".
- **Pré-buffer:** a reprodução só começa com o buffer ≥ 50% (bit `BUF_READY` do event group).
- **VU meter:** calculado na `audio_task` sobre o PCM (pico por canal, 0–100), publicado numa variável atômica e lido pela telemetria.
- **Volume:** escala digital no PCM, 0–21 passos (mantém a faixa do README).

### 2.3 Tasks

| Task | Core | Prioridade* | Disparo | Papel |
| :-- | :-: | :-: | :-- | :-- |
| `audio` | 1 | 10 | bloqueia no stream buffer | decodifica MP3 e escreve no I2S |
| `net` | 0 | 8 | bloqueia no socket | baixa o stream e escreve no buffer (produtora) |
| `input` | 1 | 7 | semáforo dado pela ISR | lê encoder/botões e põe `cmd_t` na fila |
| `uart_rx` | 0 | 6 | fila de eventos do driver UART | lê linha JSON do Pi e põe `cmd_t` na fila |
| `control` | 0 | 6 | `xQueueReceive` na fila de comandos | **única** task que altera o estado (estação, volume, play) |
| `telemetry` | 0 | 5 | `vTaskDelayUntil` 200 ms | envia `status` e `vu` |
| `rtos_mon` | 0 | 3 | `vTaskDelayUntil` 1 s | `uxTaskGetSystemState` → mensagem `rtos` |

\* Relativas entre si; ficam abaixo das tasks do Wi-Fi/LwIP do ESP-IDF. A `audio` fica sozinha no core 1 com a `input` (curta e rara).

> **Novidade em relação ao diagrama:** a task `control`. No diagrama, a fila de comandos aponta para o "Estado", mas alguém precisa consumir a fila. Ter um único consumidor deixa o fluxo claro: entrada e UART produzem, o `control` executa.

### 2.4 Primitivas de sincronização

| Primitiva | Nome | Quem usa |
| :-- | :-- | :-- |
| Stream buffer | `audio_sb` | `net` → `audio` |
| Fila | `cmd_q` (8 × `cmd_t`) | `input`, `uart_rx` → `control` |
| Semáforo binário | `input_sem` | ISR do encoder/botões → `input` |
| Mutex | `state_mtx` | `control` escreve; `telemetry` lê |
| Mutex | `uart_tx_mtx` | `telemetry`, `rtos_mon`, `control` (ack/evt) e `net` (evt) escrevem na UART |
| Event group | `sys_eg`: `WIFI_UP`, `BUF_READY`, `PLAYING` | `audio` espera `WIFI_UP \| BUF_READY` |

O `uart_tx_mtx` tem vários escritores de propósito, para a demo 3 (mutex) ser real.

**Troca de estação:** o `control` atualiza o estado e notifica a `net` (task notification). A `net` fecha a conexão, limpa `BUF_READY`, a `audio` para de consumir e o buffer é esvaziado (`xStreamBufferReset` só funciona sem task bloqueada; a spec define a sequência).

### 2.5 Pinagem proposta (ESP32-WROVER) — **conferir na placa**

| Função | GPIO | Observação |
| :-- | :-: | :-- |
| I2S BCLK | 26 | |
| I2S LRC/WS | 25 | |
| I2S DOUT | 22 | → DIN do MAX98357A |
| Encoder CLK (A) | 34 | input-only; KY-040 já tem pull-up |
| Encoder DT (B) | 35 | input-only |
| Encoder SW | 39 | input-only; KY-040 já tem pull-up |
| Botão play/pause | 32 | pull-up interno |
| Botão próxima | 33 | pull-up interno |
| Botão anterior | 27 | pull-up interno |
| UART1 TX → Pi RX (GPIO15, pino 10) | 19 | |
| UART1 RX ← Pi TX (GPIO14, pino 8) | 18 | |

⚠️ **GPIO16/17 (UART2 padrão) são usados pela PSRAM no WROVER.** Por isso a UART com o Pi vai na UART1 remapeada. A UART0 (USB) fica para log e gravação.

### 2.6 Organização do projeto PlatformIO

```
firmware/
├── platformio.ini
├── CMakeLists.txt            # exigido pelo framework espidf
├── sdkconfig.defaults        # PSRAM, trace facility, run-time stats, CPU 240 MHz
├── partitions.csv            # app grande (decoder + Wi-Fi)
├── wokwi.toml + diagram.json # simulação
├── include/
│   ├── pins.h                # pinagem única (tabela 2.5)
│   └── app_config.h          # tamanhos de buffer, períodos, prioridades
├── src/
│   ├── CMakeLists.txt
│   ├── idf_component.yml     # decoder MP3, cJSON se necessário
│   └── main.c                # cria primitivas e tasks, nada mais
├── components/
│   ├── app_state/            # estado + state_mtx
│   ├── wifi_mgr/             # Wi-Fi + bit WIFI_UP
│   ├── net_stream/           # net_task (produtora)
│   ├── audio_out/            # audio_task, decoder, I2S, VU, volume
│   ├── input/                # ISR + input_task (encoder com debounce)
│   ├── control/              # control_task (consumidor da cmd_q)
│   ├── uart_link/            # driver UART1, uart_rx_task, envio com uart_tx_mtx
│   ├── protocol/             # encode/parse JSON (C puro, testável no PC)
│   ├── telemetry/            # telemetry_task e rtos_mon_task
│   └── stations/             # lista de estações
└── test/
    └── test_protocol/        # Unity, roda em env:native
```

`platformio.ini` (esboço):

```ini
[platformio]
default_envs = wrover

[env]
platform = espressif32 @ ^6        ; pinar a versão exata na spec
framework = espidf
board = esp-wrover-kit             ; ajustar para a placa real
monitor_speed = 115200
board_build.partitions = partitions.csv

[env:wrover]                       ; hardware real

[env:wokwi]                        ; simulação: Wi-Fi Wokwi-GUEST, áudio stub
build_flags = -DANYRADIO_SIM=1

[env:demo_nomutex]                 ; demo 3: UART sem mutex
build_flags = -DANYRADIO_UART_NO_MUTEX=1

[env:native]                       ; testes do protocolo no PC
platform = native
framework =
test_filter = test_protocol
```

Credenciais Wi-Fi: `secrets.ini` fora do git (via `extra_configs`) ou menuconfig. **[DECIDIR]**

## 3. Protocolo UART — ajustes propostos

A base do README continua valendo (JSON por linha, `\n`, 115200 8N1). Banda: status 5 Hz + VU 20 Hz + rtos 1 Hz ≈ 1,5 KB/s, cerca de 13% dos ~11,5 KB/s disponíveis. Cabe com folga.

Lacunas que encontrei:

| # | Problema | Proposta |
| :-: | :-- | :-- |
| 1 | O Pi não sabe o nome das estações nem a versão do firmware | `{"t":"hello","v":1,"fw":"0.1.0","stations":["Rádio X","..."]}` no boot e quando o Pi pedir com `{"cmd":"hello"}` |
| 2 | `"st"` é ambíguo (station? state?) | `status`: `{"t":"status","si":2,"sn":"Rádio X","play":true,"vol":12,"buf":78,"rssi":-61}` |
| 3 | Faltam próxima/anterior pelo Pi | `{"cmd":"next"}`, `{"cmd":"prev"}` |
| 4 | O monitor quer mostrar "quem segura o mutex" e o contador de interrupções (demo 3) | `rtos` ganha `"mtx":{"state":"control","uart":null}` (via `xSemaphoreGetMutexHolder`) e `"isr":1532` |
| 5 | A ocupação da fila sem a capacidade não diz nada | `"q":{"cmd":[2,8]}` (usados, capacidade) |
| 6 | Unidades indefinidas | `cpu` = % do intervalo, somando os 2 cores; `stk` = menor folga de stack em **bytes**; `s` ∈ `R,r,B,S,D` (Running, Ready, Blocked, Suspended, Deleted); incluir `"c"` = core |
| 7 | Sem confirmação de comando | `{"t":"ack","cmd":"vol","ok":true}` / `{"t":"err","e":"bad_json"}` |
| 8 | Sem limite de linha | máximo de 512 bytes; linha maior é descartada com `err` |
| 9 | O Pi não detecta ESP desligado | sem `status` por 1 s → UI mostra "ESP32 desconectado" |
| 10 | Demo 6 exige recompilar | comando de debug `{"cmd":"prio","task":"audio","p":1}` (só em build de demo) **[DECIDIR]** |

A fonte da verdade fica em `protocol/` (README + JSON Schema + exemplos). O firmware (testes nativos) e a UI (validação e mock) usam os mesmos exemplos.

## 4. Raspberry Pi

### 4.1 Sistema (Raspberry Pi OS Bookworm, desktop)

- **UART:** `raspi-config` → console serial **desligado**, hardware **ligado**. No Pi 3/4, a UART boa (PL011) está com o Bluetooth; adicionar `dtoverlay=disable-bt` no `config.txt` para ter `/dev/ttyAMA0` estável em GPIO14/15. O backend abre sempre `/dev/serial0`.
- **Tela 800×480:** no Bookworm (driver KMS), `hdmi_cvt`/`hdmi_group` do guia **podem ser ignorados**. Usar `video=HDMI-A-1:800x480@60` no `cmdline.txt` e manter `hdmi_cvt` só como plano B.
- **Kiosk:** o Bookworm atual usa Wayland (labwc). Autostart em `~/.config/labwc/autostart` com `chromium-browser --kiosk --noerrdialogs --disable-infobars http://localhost:8080`. Desligar o blanking de tela e esconder o cursor. Se o Pi vier com X11, o caminho é o autostart do LXDE.
- **Backend como serviço:** unit `systemd` (`anyradio.service`), usuário no grupo `dialout`, reinício automático.
- **Node.js:** LTS (22) via NodeSource.

### 4.2 Software

```
ui/                           # npm workspaces
├── package.json
├── server/                   # Node (ESM)
│   ├── serial.js             # serialport + ReadlineParser, reconexão automática
│   ├── protocol.js           # valida e normaliza mensagens (usa protocol/)
│   ├── ws.js                 # ws: broadcast ESP→telas, comandos tela→ESP
│   └── index.js              # HTTP: serve web/dist e /ws na porta 8080
├── web/                      # React + Vite
│   └── src/
│       ├── hooks/useRadio.ts # WebSocket, estado, reconexão, "desconectado"
│       ├── screens/NowPlaying.tsx
│       ├── screens/RtosMonitor.tsx
│       └── components/       # VuMeter (canvas), BufferBar, TaskTable, QueueGauge
└── mock/
    └── fake-esp.js           # gera telemetria falsa ou reproduz um log gravado
```

- **Modo mock** (`SERIAL_PORT=mock`): a frente 3 desenvolve no notebook sem ESP32 nem Pi. É o que destrava o trabalho em paralelo.
- **Gravação:** o backend pode gravar a serial em `.jsonl`. Serve para depurar e para o plano B das demos.
- **Desempenho no Pi 3 (1 GB):** VU em `<canvas>` com `requestAnimationFrame`, sem biblioteca de gráficos pesada, e build de produção servido localmente.

## 5. Repositório

```
anyradio/
├── firmware/    # projeto PlatformIO (seção 2.6)
├── ui/          # server + web + mock (seção 4.2)
├── protocol/    # contrato UART: README, schema, exemplos
├── pi/          # config.txt, cmdline, autostart, anyradio.service, setup.sh
├── docs/        # guia, arquitetura, backlog, specs
└── images/
```

O README atual usa `idf.py`. Com PlatformIO, passa a ser `pio run -e wrover -t upload -t monitor`.

## 6. Pontos para decidir

1. **Framework:** ESP-IDF puro no PlatformIO (recomendado) ou Arduino + ESP32-audioI2S?
2. **Pipeline de áudio próprio** (net → stream buffer → audio) ou aceitar a lib como caixa-preta?
3. **Task `control`** como único consumidor da fila: de acordo?
4. **Estações:** fixas no firmware (MVP), em NVS, ou enviadas pelo Pi?
5. **Ajustes do protocolo** (seção 3): quais entram na v1?
6. **Demo de prioridade:** comando de debug pela UART ou build separado?
7. **Perguntas em aberto do guia:** a tela tem touch? O Pi é 3 ou 4? Quais rádios entram na lista?
