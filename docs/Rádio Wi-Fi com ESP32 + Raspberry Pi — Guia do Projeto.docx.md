# **Rádio Wi-Fi com ESP32 \+ Raspberry Pi — Guia do Projeto**

Oct 15, 2026 · @Irvin

## **Visão geral**

Vamos construir um rádio por internet em que o **ESP32 é o rádio** e o **Raspberry Pi é a cara dele**. O foco da disciplina é o FreeRTOS, então toda a lógica de tempo real (stream, buffer, áudio, entradas) fica no ESP32. O Pi só desenha a interface na tela de 5" HDMI.

* **ESP32 (FreeRTOS):** baixa o stream da rádio, guarda num buffer, decodifica o MP3 e toca pelo I2S no MAX98357A. Também lê o encoder e os botões.  
* **Raspberry Pi:** recebe a telemetria do ESP32 pela UART e mostra na tela a estação, o volume, o VU meter e um **monitor do FreeRTOS ao vivo**. Se a tela tiver touch, manda comandos de volta.  
* **O diferencial:** a tela mostra o RTOS por dentro (estado das tasks, ocupação das filas, quem segura o mutex). Na apresentação, a teoria aparece funcionando.

Regra de ouro: o Pi **nunca** toca o áudio. Se ele fizer tudo sozinho, o FreeRTOS some do projeto.

![arquitetura · ESP32 + Raspberry Pi][image1]

arquitetura · ESP32 \+ Raspberry Pi

Tudo que é tempo real fica dentro do retângulo destacado; entre ESP32 e Pi só passam mensagens pela UART.

## **O que precisamos ter**

A tela e o Raspberry Pi já estão garantidos. O resto ainda precisa ser conseguido.

### **Hardware**

| Componente | Para que serve | Situação |
| :---- | :---- | :---- |
| ESP32 (de preferência WROVER, com PSRAM) | Wi-Fi, stream, decodificação e FreeRTOS | A conseguir |
| Tela 5" HDMI | Interface do rádio, ligada no Pi | Temos |
| Raspberry Pi 3 ou 4 | Roda a interface na tela | Professor traz |
| Cartão microSD (16 GB ou mais) | Sistema do Pi (Raspberry Pi OS com desktop) | Professor traz |
| Fonte 5 V / 3 A para o Pi | Alimenta o Pi e a tela | A conseguir |
| Cabo HDMI (micro-HDMI se for Pi 4\) | Liga o Pi na tela | A conseguir |
| Módulo MAX98357A | DAC \+ amplificador I2S para o som | A conseguir |
| Alto-falante 4–8 ohms, \~3 W | Saída de áudio | A conseguir |
| Encoder rotativo KY-040 | Volume e estação (gera interrupções) | A conseguir |
| 2–3 botões táteis | Play/pause, próxima e anterior | A conseguir |
| Capacitor 470–1000 µF | Estabiliza a alimentação do amplificador | A conseguir |
| Fonte 5 V / 2 A (USB) para o ESP32 | Alimenta ESP32 e amplificador | A conseguir |
| Protoboard \+ jumpers | Montagem, incluindo 3 fios para a UART (TX, RX, GND) | A conseguir |

### **Software**

* **ESP32:** ESP-IDF (ou Arduino IDE / PlatformIO com o core do ESP32) e uma biblioteca de áudio, como ESP32-audioI2S.  
* **Raspberry Pi:** Raspberry Pi OS com desktop, Node.js, a biblioteca serialport e um front-end em React aberto no Chromium em modo kiosk.  
* **Simulação:** Wokwi, para testar tasks, filas e semáforos antes do hardware chegar.  
* **Organização:** um repositório no GitHub com as pastas firmware/ (ESP32) e ui/ (Pi).

## **Divisão do grupo**

São quatro frentes que andam em paralelo e se encontram no protocolo UART. Preencham o responsável de cada uma.

| Frente | O que entrega | Responsável |
| :---- | :---- | :---- |
| 1\. Áudio e stream | Conexão Wi-Fi, download do stream, buffer, decodificação e saída I2S no ESP32 |  |
| 2\. Entradas e comunicação | ISR do encoder e dos botões, fila de comandos, UART e protocolo no ESP32 |  |
| 3\. Interface no Pi | Backend Node lendo a serial, front-end React, modo kiosk e tela de monitor do RTOS |  |
| 4\. Montagem e apresentação | Circuito, case, roteiro das demos e slides |  |

As frentes 1 e 2 dividem o mesmo firmware, então combinem desde o início os nomes das filas e o formato das mensagens.

## **Passo a passo**

Cada fase só começa quando a anterior está funcionando. Marquem os itens conforme forem concluindo.

### **Fase 0 — Preparação (antes da próxima aula)**

* ☐ Definir o responsável de cada frente  
* ☐ Criar o repositório no GitHub com firmware/ e ui/  
* ☐ Instalar o ambiente do ESP32 (ESP-IDF ou PlatformIO) e rodar um "blink"  
* ☐ Montar no Wokwi um esqueleto com duas tasks trocando mensagens por fila  
* ☐ Listar o que falta comprar e dividir os custos

### **Fase 1 — Raspberry Pi e tela (aula com o professor)**

* ☐ Gravar o Raspberry Pi OS (com desktop) no microSD com o Raspberry Pi Imager, já configurando Wi-Fi e SSH  
* ☐ Ligar o Pi na tela e ajustar o config.txt para 800×480, se a imagem não aparecer certa  
* ☐ Testar o touch, se a tela tiver  
* ☐ Instalar o Node.js e ativar a UART do Pi (raspi-config → Interface → Serial: console desligado, hardware ligado)

### **Fase 2 — Áudio no ESP32**

* ☐ Ligar o MAX98357A e o alto-falante no I2S e tocar um tom de teste  
* ☐ Conectar no Wi-Fi e tocar uma rádio fixa  
* ☐ Separar em tasks: rede (produtora) e áudio (consumidora), com stream buffer entre elas  
* ☐ Fixar a task de áudio num core, com a maior prioridade

### **Fase 3 — Entradas**

* ☐ ISR do encoder e dos botões dando semáforo (ou task notification)  
* ☐ Task de entrada colocando comandos numa fila  
* ☐ Mutex protegendo o estado compartilhado (estação e volume)  
* ☐ Trocar estação e volume pelo encoder, sem engasgar o som

### **Fase 4 — Comunicação ESP32 ↔ Pi**

* ☐ Ligar TX, RX e GND entre os dois (cruzados: TX de um no RX do outro)  
* ☐ Task de telemetria no ESP32 mandando o status a cada 200 ms com vTaskDelayUntil  
* ☐ Task lendo comandos da UART pela fila de eventos do driver  
* ☐ Script Node no Pi que lê a serial e imprime as mensagens no terminal

### **Fase 5 — Interface no Pi**

* ☐ Backend Node repassando as mensagens por WebSocket  
* ☐ Tela "Tocando agora": estação, volume, VU meter e nível do buffer  
* ☐ Tela "Monitor do RTOS": estado das tasks, filas e mutex  
* ☐ Chromium abrindo sozinho em modo kiosk no boot

### **Fase 6 — Montagem final e apresentação**

* ☐ Passar o circuito para uma montagem firme (ou case)  
* ☐ Ensaiar as demos da seção de apresentação  
* ☐ Preparar os slides com a arquitetura e os conceitos

## **Conceitos de FreeRTOS no projeto**

Cada recurso visto em aula tem um lugar natural no rádio. Esta tabela é a base dos slides.

| Recurso | Onde aparece | API principal |
| :---- | :---- | :---- |
| Tasks e prioridades | Rede, áudio, entrada, telemetria e comunicação; o áudio tem a maior prioridade | xTaskCreatePinnedToCore |
| Produtor-consumidor | Task de rede enche o buffer, task de áudio esvazia | xStreamBufferSend / xStreamBufferReceive |
| Fila | Comandos do usuário (encoder, botões, touch) até quem executa | xQueueSend / xQueueReceive |
| Semáforo binário | ISR do encoder acorda a task de entrada | xSemaphoreGiveFromISR / xSemaphoreTake |
| Mutex | Protege o estado (estação, volume) e a UART de saída | xSemaphoreCreateMutex |
| Fila de eventos do driver | Driver UART entrega os dados recebidos do Pi | uart\_driver\_install |
| Tarefa periódica | Telemetria a cada 200 ms, sem acumular atraso | vTaskDelayUntil |
| Event group (extra) | Espera Wi-Fi conectado e buffer pronto antes de tocar | xEventGroupWaitBits |
| Estatísticas do sistema | Alimenta a tela de monitor do RTOS | uxTaskGetSystemState |

Para uxTaskGetSystemState funcionar, ativem configUSE\_TRACE\_FACILITY e configGENERATE\_RUN\_TIME\_STATS no menuconfig.

## **Protocolo UART entre ESP32 e Pi**

Uma linha JSON por mensagem, terminada em \\n, a 115200 baud. É simples de depurar: dá para ler tudo direto num terminal serial.

**Ligação:** TX do ESP32 no RX do Pi (GPIO15, pino 10), RX do ESP32 no TX do Pi (GPIO14, pino 8\) e GND com GND. Os dois trabalham em 3,3 V, então não precisa de conversor de nível.

| Direção | Tipo | Exemplo |
| :---- | :---- | :---- |
| ESP32 → Pi | Status (a cada 200 ms) | {"t":"status","st":"Rádio X","vol":12,"buf":78,"rssi":-61} |
| ESP32 → Pi | VU meter | {"t":"vu","l":34,"r":40} |
| ESP32 → Pi | Monitor do RTOS (a cada 1 s) | {"t":"rtos","tasks":\[{"n":"audio","s":"R","cpu":41,"stk":812}\],"q":{"cmd":2}} |
| ESP32 → Pi | Evento | {"t":"evt","e":"wifi\_lost"} |
| Pi → ESP32 | Trocar estação | {"cmd":"station","i":3} |
| Pi → ESP32 | Volume | {"cmd":"vol","v":13} |
| Pi → ESP32 | Play/pause | {"cmd":"toggle"} |

Combinem esse formato na Fase 0\. Se mudar algum campo, atualizem esta tabela primeiro.

## **Roteiro da apresentação**

A ideia é mostrar cada conceito acontecendo na tela, não só o rádio tocando.

1. **Abertura:** o rádio ligado, tocando, na tela "Tocando agora".  
2. **Arquitetura:** um slide com ESP32, Pi e o que passa entre eles.  
3. **ISR e semáforo:** abrir o monitor do RTOS e girar o encoder. A task de entrada sai de Blocked para Running, e o contador de interrupções sobe.  
4. **Produtor-consumidor:** desligar o roteador (ou o hotspot) por alguns segundos. O buffer cai na tela, o som continua por um tempo e só depois corta.  
5. **Mutex:** compilar uma versão sem o mutex da UART e mostrar as mensagens embaralhadas. Depois, com o mutex, tudo limpo.  
6. **Prioridades:** baixar a prioridade da task de áudio e mostrar o som engasgando quando a interface pede muitos dados.  
7. **Fechamento:** o que cada recurso resolveu e o que fariam diferente.

Gravem as demos 4, 5 e 6 em vídeo, como plano B caso o Wi-Fi da sala falhe.

## **Riscos e dicas**

| Risco | Como evitar |
| :---- | :---- |
| Wi-Fi da faculdade bloqueia o stream ou exige login | Usar o hotspot do celular nos testes e na apresentação |
| Som engasgando | Task de áudio fixada num core com a maior prioridade, e buffer maior (por isso a PSRAM) |
| ESP32 reiniciando quando o volume sobe | Capacitor na alimentação do amplificador e fonte de 2 A |
| UART não recebe nada | Conferir TX/RX cruzados, GND em comum e o console serial desligado no Pi |
| Tela do Pi com imagem errada | Ajustar hdmi\_cvt e hdmi\_group no config.txt para 800×480 |
| Pi demora a ficar pronto | Usar o Wokwi e um terminal serial no PC enquanto o Pi não chega |
| Integração de última hora | Combinar o protocolo cedo e testar cada fase isolada |

Perguntas em aberto:

* ☐ A tela tem touch? Se tiver, a interface pode mandar comandos.  
* ☐ Qual modelo de Raspberry Pi o professor vai trazer (3 ou 4)?  
* ☐ Quais rádios vão estar na lista de estações?

[image1]: <data:image/png;base64,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>