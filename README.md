# Estacionamento Inteligente com FreeRTOS e Cheat Sincronizado

Projeto embarcado desenvolvido e validado por **simulação no Wokwi**, utilizando **ESP32**, **Arduino Framework** e **FreeRTOS**. O sistema simula um estacionamento com vagas comuns e PCD, controle de entrada e saída, fila de espera prioritária e um subsistema de manipulação de sensor denominado **cheat**.

O projeto demonstra comunicação entre tarefas, exclusão mútua, filas, semáforos, notificações diretas e processamento periódico em tempo real. Toda a parte de hardware, incluindo ESP32, botões, LEDs, sensor simulado e análise dos sinais, é executada no ambiente virtual Wokwi.

## Vídeo de demonstração


**YouTube:** `https://youtu.be/COLOCAR_ID_DO_VIDEO`


## Integrantes

| Nome | Matrícula |
|---|---|
|Cleisson de Alencar Ramos | 122211354 | 
| Nome do integrante 2 | Matrícula | 


## Objetivo

Implementar um sistema embarcado baseado em FreeRTOS que simule o funcionamento de um estacionamento e demonstre sincronização e manipulação de sinais em tempo real.

Além do controle de vagas, o sistema possui três modos de leitura de um sensor:

1. **Normal:** o valor observado é igual ao valor físico do sensor;
2. **Cheat assíncrono:** o valor do sensor permanece adulterado continuamente;
3. **Cheat sincronizado:** a adulteração é ativada somente na janela da amostragem.

## Funcionalidades

- oito vagas comuns;
- duas vagas PCD;
- entrada de veículo comum;
- entrada de veículo PCD;
- preferência por vaga PCD;
- fallback opcional para vaga comum;
- fila de espera exclusiva para PCD;
- saída de veículos;
- reset completo;
- sinalização individual das vagas;
- indicação de lotação e espera PCD;
- amostragem periódica de um sensor;
- três modos de operação;
- sincronização do cheat com a amostragem;
- saída de diagnóstico no monitor serial.

## Ambiente de simulação e componentes virtuais

A implementação do hardware é realizada integralmente no **Wokwi**.

Componentes utilizados na simulação:

- 1 ESP32 DevKit virtual;
- 6 pushbuttons;
- 13 LEDs virtuais;
- 13 resistores virtuais de 220 Ω a 330 Ω;
- analisador lógico virtual do Wokwi;
- monitor serial virtual em 115200 baud.

> Os componentes representam o mesmo circuito que poderia ser montado fisicamente. A simulação permite demonstrar entradas, saídas, concorrência das tarefas e sincronização temporal sem utilizar protoboard ou instrumentos externos.

## Mapeamento de pinos

### Entradas

| Função | GPIO | Configuração |
|---|---:|---|
| Entrada de veículo comum | 25 | `INPUT_PULLUP` |
| Entrada de veículo PCD | 26 | `INPUT_PULLUP` |
| Saída de veículo | 27 | `INPUT_PULLUP` |
| Reset | 14 | `INPUT_PULLUP` |
| Sensor real da vaga monitorada | 32 | `INPUT_PULLUP` |
| Seleção do modo | 33 | `INPUT_PULLUP` |

### Saídas

| Função | GPIO |
|---|---:|
| Vaga comum 1 | 2 |
| Vaga comum 2 | 4 |
| Vaga comum 3 | 5 |
| Vaga comum 4 | 18 |
| Vaga comum 5 | 19 |
| Vaga comum 6 | 21 |
| Vaga comum 7 | 22 |
| Vaga comum 8 | 23 |
| Vaga PCD 1 | 12 |
| Vaga PCD 2 | 13 |
| Lotado/espera PCD | 15 |
| Cheat ativo | 16 |
| Pulso de amostragem | 17 |

> Cada LED externo deve possuir um resistor em série. Todos os switches são ligados entre o GPIO correspondente e o GND.

## Arquitetura das tarefas

| Tarefa | Prioridade | Responsabilidade |
|---|---:|---|
| `taskEntrada` | 2 | Detecta entradas comuns e PCD e envia eventos ao controlador |
| `taskControlador` | 3 | Processa a fila de entrada e atribui vagas |
| `taskSaida` | 2 | Retira veículos da fila de estacionados e libera vagas |
| `taskReset` | 3 | Limpa vagas, filas e estados do sistema |
| `taskBlinkStatus` | 1 | Controla a sinalização de lotação e espera PCD |
| `taskCheat` | 3 | Seleciona o modo e gera a adulteração do sensor |
| `taskAmostragem` | 4 | Executa a leitura periódica e registra o resultado |

O diagrama detalhado está em [`docs/DIAGRAMA.md`](docs/DIAGRAMA.md).

## Comunicação e sincronização

### Filas

- `filaEntrada`: transporta novos veículos para o controlador;
- `filaEstacionados`: armazena veículos atualmente estacionados;
- `filaEsperaPcd`: guarda veículos que aceitam somente vaga PCD.

### Mutex

O `mutexVagas` protege os dados compartilhados relacionados às vagas. Assim, atribuição, liberação e reset não alteram simultaneamente o mesmo estado.

### Semáforo contador

O `semEvento` informa à tarefa controladora que existe pelo menos um novo evento na fila de entrada. O controlador permanece bloqueado quando não há trabalho pendente.

### Notificação direta

No modo sincronizado, `taskAmostragem` utiliza uma notificação direta para desbloquear `taskCheat`. A tarefa de cheat ativa o sinal adulterado antes do instante da leitura e o mantém ativo durante a janela de amostragem.

## Estratégia de sincronização do cheat

Os parâmetros temporais usados pelo código são:

```cpp
#define PERIODO_AMOSTRAGEM_MS   1000
#define ANTECEDENCIA_CHEAT_MS     50
#define DURACAO_PULSO_CHEAT_MS   120
```

No modo sincronizado, a sequência é:

1. a tarefa de amostragem inicia um novo ciclo;
2. uma notificação é enviada à tarefa de cheat;
3. o cheat é ativado;
4. a tarefa de amostragem aguarda 50 ms;
5. o sensor é lido durante o pulso adulterado;
6. após 120 ms, a tarefa de cheat desativa o sinal.

Essa abordagem reduz a dependência de atrasos iniciados separadamente, pois o evento de amostragem dispara diretamente a tarefa responsável pela fraude.

## Modos de operação

### Modo 0 — Normal

- cheat desativado;
- LED de cheat apagado;
- valor observado igual ao valor real.

### Modo 1 — Cheat assíncrono

- cheat ativo continuamente;
- LED de cheat aceso continuamente;
- valor observado é o inverso do sensor real.

### Modo 2 — Cheat sincronizado

- cheat normalmente inativo;
- tarefa de amostragem dispara a tarefa de cheat;
- LED de cheat produz pulsos próximos às amostragens;
- a leitura ocorre dentro da janela de fraude.

Cada toque no switch de modo alterna entre os três modos.


**Simulação Wokwi:** `(https://wokwi.com/projects/470478981345713153)`



