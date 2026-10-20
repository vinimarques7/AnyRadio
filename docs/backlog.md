# AnyRadio — Backlog (rascunho)

> Baseado em `docs/arquitetura.md`. Prioridade: **P0** = sem isso não há demo · **P1** = demo completa · **P2** = extra.
> Frentes: **F1** Áudio e stream · **F2** Entradas e comunicação · **F3** Interface no Pi · **F4** Montagem e apresentação · **ALL** grupo todo.
> Cada item pode virar uma issue no GitHub (ID no título, frente como label).

## Fase 0 — Preparação

| ID | Item | Frente | Prio | Depende | Pronto quando |
| :-- | :-- | :-: | :-: | :-- | :-- |
| INF-01 | Definir o responsável de cada frente e fechar os pontos da seção 6 da arquitetura | ALL | P0 | — | Decisões registradas em `docs/arquitetura.md` |
| INF-02 | Criar a estrutura do repo (`firmware/`, `ui/`, `protocol/`, `pi/`), `.gitignore`, licença | ALL | P0 | INF-01 | Estrutura no `main` |
| INF-03 | Esqueleto PlatformIO (`espidf`, envs `wrover`/`wokwi`/`native`, `sdkconfig.defaults`, partições) com blink | F1 | P0 | INF-02 | `pio run -e wrover` compila e o LED pisca |
| INF-04 | Wokwi: duas tasks trocando mensagens por fila | F2 | P0 | INF-03 | Simulação roda pelo `wokwi.toml` |
| INF-05 | Contrato v1 do protocolo em `protocol/` (README, schema, exemplos) | F2+F3 | P0 | INF-01 | Revisado por F1, F2 e F3 |
| INF-06 | Lista de compras e divisão de custos | F4 | P0 | — | Itens pedidos |
| INF-07 | Esqueleto `ui/` (workspaces, server, web Vite, lint) | F3 | P0 | INF-02 | `npm run dev` sobe a tela vazia |
| INF-08 | CI no GitHub Actions: `pio run`, `pio test -e native`, build da UI | ALL | P2 | INF-03, INF-07 | Check verde nos PRs |

## Fase 1 — Raspberry Pi e tela

| ID | Item | Frente | Prio | Depende | Pronto quando |
| :-- | :-- | :-: | :-: | :-- | :-- |
| PI-01 | Gravar o Pi OS desktop (Wi-Fi + SSH no Imager) | F3 | P0 | — | SSH funcionando |
| PI-02 | Tela 800×480 (`cmdline.txt` `video=`; plano B `hdmi_cvt`) | F3 | P0 | PI-01 | Imagem certa e sem bordas |
| PI-03 | Testar o touch e registrar o resultado | F3 | P1 | PI-02 | Resposta da pergunta em aberto |
| PI-04 | UART: console desligado, hardware ligado, `dtoverlay=disable-bt`, loopback TX↔RX | F3 | P0 | PI-01 | `/dev/serial0` ecoa no loopback |
| PI-05 | Node.js LTS e usuário no grupo `dialout` | F3 | P0 | PI-01 | `node -v` ok e serial aberta sem sudo |
| PI-06 | `pi/setup.sh` reproduzindo PI-02..PI-05 e PI-07/PI-08 | F3 | P1 | PI-04 | Pi novo configurado com um comando |
| PI-07 | Serviço `systemd` do backend | F3 | P1 | UI-01 | Backend sobe sozinho no boot |
| PI-08 | Chromium em kiosk no boot, sem blanking nem cursor | F3 | P0 | PI-02 | Boot vai direto para a UI |

## Fase 2 — Áudio no ESP32

| ID | Item | Frente | Prio | Depende | Pronto quando |
| :-- | :-- | :-: | :-: | :-- | :-- |
| AUD-01 | Ligar o MAX98357A + capacitor e tocar um tom senoidal no I2S (`i2s_std`) | F1 | P0 | INF-03 | Tom limpo no alto-falante |
| AUD-02 | `wifi_mgr`: conectar, reconectar e bit `WIFI_UP` | F1 | P0 | INF-03 | Reconecta após queda do hotspot |
| AUD-03 | `net_stream`: HTTP GET de uma rádio fixa até o stream buffer em PSRAM | F1 | P0 | AUD-02 | Taxa de bytes estável no log |
| AUD-04 | Integrar o decoder MP3 e tocar o stream pela `audio_task` | F1 | P0 | AUD-01, AUD-03 | Rádio tocando 10 min sem engasgo |
| AUD-05 | Pré-buffer com `BUF_READY` e event group | F1 | P1 | AUD-04 | Só toca com buffer ≥ 50%; volta a tocar após queda |
| AUD-06 | Afinidade e prioridade conforme a tabela 2.3 | F1 | P0 | AUD-04 | Sem engasgo com telemetria a 20 Hz |
| AUD-07 | Volume digital 0–21 e VU (pico L/R) | F1 | P1 | AUD-04 | Volume muda sem clique; VU coerente |
| AUD-08 | Troca de estação (notificação, flush, reconexão) | F1 | P0 | AUD-04, IO-05 | Troca em menos de 3 s, sem travar |
| AUD-09 | Lista de estações (componente `stations`) e escolha das rádios | F1 | P0 | — | 5+ URLs HTTP MP3 testadas no hotspot |

## Fase 3 — Entradas

| ID | Item | Frente | Prio | Depende | Pronto quando |
| :-- | :-- | :-: | :-: | :-- | :-- |
| IO-01 | ISR do encoder e dos botões dando `input_sem`, com contador de ISR | F2 | P0 | INF-03 | Contador sobe ao girar o encoder |
| IO-02 | `input_task`: decodificar a quadratura, debounce, gerar `cmd_t` | F2 | P0 | IO-01 | 1 passo = 1 comando, sem repique |
| IO-03 | `cmd_q` e tipo `cmd_t` compartilhados | F2 | P0 | INF-05 | Header revisado por F1 |
| IO-04 | `app_state` com `state_mtx` (getters e setters) | F2 | P0 | — | Nenhum acesso ao estado fora da API |
| IO-05 | `control_task`: consumir a `cmd_q` e aplicar volume, estação e play | F2 | P0 | IO-03, IO-04 | Encoder muda o volume sem engasgar |

## Fase 4 — Comunicação ESP32 ↔ Pi

| ID | Item | Frente | Prio | Depende | Pronto quando |
| :-- | :-- | :-: | :-: | :-- | :-- |
| COM-01 | Componente `protocol` (encode/parse) com testes Unity em `env:native` | F2 | P0 | INF-05 | `pio test -e native` verde |
| COM-02 | `uart_link`: driver UART1, pattern detect `\n`, `uart_rx_task` → `cmd_q` | F2 | P0 | COM-01, IO-03 | Comando digitado no terminal muda o volume |
| COM-03 | Envio com `uart_tx_mtx` + flag `ANYRADIO_UART_NO_MUTEX` | F2 | P0 | COM-02 | `env:demo_nomutex` embaralha as linhas |
| COM-04 | `telemetry_task` 200 ms (`status`, `vu`) com `vTaskDelayUntil` | F2 | P0 | COM-03, IO-04 | Período medido de 200 ± 2 ms |
| COM-05 | `rtos_mon_task` 1 s: tasks, CPU %, stack, filas, holders, ISR | F2 | P1 | COM-03 | Mensagem `rtos` válida pelo schema |
| COM-06 | `hello`, `ack`/`err`, `evt` (`wifi_lost`, `wifi_up`, `buf_low`) | F2 | P1 | COM-03 | Exemplos do `protocol/` reproduzidos |
| COM-07 | Ligar os fios ESP32↔Pi e testar com script Node imprimindo as mensagens | F2+F3 | P0 | COM-04, PI-04 | Mensagens no terminal do Pi |

## Fase 5 — Interface no Pi

| ID | Item | Frente | Prio | Depende | Pronto quando |
| :-- | :-- | :-: | :-: | :-- | :-- |
| UI-01 | Backend: serial (com reconexão) → validação → WebSocket; comandos de volta | F3 | P0 | INF-07, INF-05 | Mensagens chegam no browser |
| UI-02 | `mock/fake-esp.js` e replay de `.jsonl` | F3 | P0 | INF-05 | UI funciona inteira sem hardware |
| UI-03 | Hook `useRadio`: estado, reconexão do WS, "ESP32 desconectado" | F3 | P0 | UI-01 | Desligar o ESP mostra o aviso em ≤ 1 s |
| UI-04 | Tela "Tocando agora": estação, volume, VU (canvas), buffer, RSSI | F3 | P0 | UI-03 | Fluido a 20 Hz no Pi |
| UI-05 | Tela "Monitor do RTOS": tabela de tasks, CPU, stack, filas, mutex, ISR | F3 | P0 | UI-03, COM-05 | Estado Blocked→Running visível ao girar |
| UI-06 | Navegação entre telas (botão físico e/ou touch) | F3 | P1 | UI-04, UI-05 | Troca sem recarregar a página |
| UI-07 | Controles touch (volume, estação, play) | F3 | P2 | PI-03, UI-04 | Comando gera `ack` |
| UI-08 | Gravação da serial em `.jsonl` | F3 | P2 | UI-01 | Arquivo reproduzível pelo mock |
| UI-09 | Identidade visual (cores, logo N) | F3+F4 | P1 | UI-04 | Bate com o banner |

## Fase 6 — Montagem e apresentação

| ID | Item | Frente | Prio | Depende | Pronto quando |
| :-- | :-- | :-: | :-: | :-- | :-- |
| HW-01 | Esquemático com a pinagem final (Fritzing ou KiCad) em `docs/` | F4 | P1 | Pinagem fechada | Imagem no repo |
| HW-02 | Montagem firme ou case | F4 | P1 | HW-01 | Transportável sem soltar fio |
| DEM-01 | Roteiro das 7 demos com o passo a passo de cada uma | F4 | P0 | — | Ensaiado 1× |
| DEM-02 | Demo de prioridade (comando `prio` ou build separado) | F1 | P1 | AUD-06 | Engasgo reproduzível |
| DEM-03 | Gravar as demos 4, 5 e 6 em vídeo (plano B) | F4 | P0 | DEM-01 | Vídeos salvos |
| DEM-04 | Slides: arquitetura e tabela de conceitos | F4 | P0 | — | Revisados pelo grupo |
| DOC-01 | Atualizar o README (PlatformIO, protocolo v1, setup do Pi) | ALL | P1 | — | README reflete o que roda |

## Caminho crítico

`INF-01 → INF-05 → (AUD-01..04 ∥ IO-01..05 ∥ UI-01..02) → COM-02..04 → COM-07 → UI-04/05 → DEM-01`

O **contrato do protocolo (INF-05)** e o **mock (UI-02)** são o que deixa as três frentes de software andarem em paralelo desde a primeira semana.
