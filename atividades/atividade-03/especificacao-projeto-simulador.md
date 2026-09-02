# ESPECIFICAÇÃO DE PROJETO: SIMULADOR DE SISTEMA OPERACIONAL EM MODO USUÁRIO

Este documento estabelece formalmente a arquitetura, as estruturas de dados e as regras de funcionamento para o desenvolvimento de um **Simulador de Sistema Operacional em Modo Usuário**. O simulador deverá ser escrito em linguagem C (padrão C99 ou superior) como um único fluxo de execução (single-thread), simulando a concorrência e o gerenciamento de recursos de forma inteiramente lógica através de tempo discreto.

O objetivo deste documento é fornecer especificações sem ambiguidades, com diagramas de transição de estado mapeados e definições de estruturas na linguagem C para guiar agentes autônomos de geração de código (como Claude Code e Open Code).

---

## 1. Visão Geral e Arquitetura do Simulador

O simulador emula o comportamento de um núcleo de sistema operacional multiprogramado gerenciando uma CPU de núcleo único (*single-core*). Toda a execução de instruções, operações de Entrada e Saída (E/S) e passagens de tempo são discretizadas por meio de uma contagem lógica de ciclos de clock.

```
       +--------------------------------------------+
       |           Inicialização do Sistema         |
       |  (Leitura de tasks.txt e alocação de PCBs) |
       +---------------------+----------------------+
                             |
                             v
                 +-----------+-----------+
                 |   Loop da Simulação   | <-------------------------+
                 |  (Clock Lógico ++ )   |                           |
                 +-----------+-----------+                           |
                             |                                       |
                             v                                       |
       +---------------------+----------------------+                |
       |  Verifica Chegada de Novos Processos (NEW)  |                |
       |      (Move-os para a Fila READY)           |                |
       +---------------------+----------------------+                |
                             |                                       |
                             v                                       |
       +---------------------+----------------------+                |
       |     Atualiza Processos Bloqueados (BLOCKED)|                |
       |      (Decrementa surto de E/S restante)    |                |
       +---------------------+----------------------+                |
                             |                                       |
                             v                                       |
       +---------------------+----------------------+                |
       |        Invocação do Escalonador            |                |
       |   (Escolhe próximo processo para RUNNING)  |                |
       +---------------------+----------------------+                |
                             |                                       |
                             v                                       |
       +---------------------+----------------------+                |
       |        Simula Execução da Instrução        |                |
       |  (Incrementa cpu_time_used e decrementa    |                |
       |         surto de CPU restante)             |                |
       +---------------------+----------------------+                |
                             |                                       |
                             v                                       |
       +---------------------+----------------------+                |
       |      Avalia Próxima Transição de Estado     |                |
       | (Preempção, Solicitação E/S ou Término)    |                |
       +---------------------+----------------------+                |
                             |                                       |
                             +---------------------------------------+
```

### 1.1 CPU Virtual e Contexto de Execução
Para emular de forma abstrata um hardware real, o simulador manterá um conjunto de registradores virtuais para cada processo criado. A CPU virtual do simulador é composta pelos seguintes componentes:
*   **Relógio Lógico Global (`global_clock`):** Um contador inteiro de 32 bits, inicializado em `0`. Cada "passo" de simulação corresponde a `1` tick de clock. Nenhuma instrução ou ação de sistema ocorre instantaneamente; tudo consome ciclos do relógio lógico global.
*   **Program Counter Virtual (`pc`):** Um registrador contendo um valor inteiro que indica qual instrução ou etapa de surto de CPU o processo está executando.
*   **Stack Pointer Virtual (`sp`):** Aponta para o topo conceitual de uma pilha de execução virtual.
*   **Registradores de Uso Geral (`R0` a `R3`):** Simulados como variáveis numéricas locais no PCB para simular o salvamento e a restauração física do contexto durante as trocas de contexto.

### 1.2 Fluxo de Execução Principal
O núcleo do simulador roda em um laço fechado (`while`) até que todos os processos inseridos tenham completado o seu ciclo de vida. As operações do loop ocorrem na seguinte sequência estrita:

1.  **Ingressar Processos (`NEW -> READY`):** Varre a lista de processos importados e verifica se algum deles possui `arrival_time == global_clock`. Se positivo, move-o para a fila correspondente do estado `READY`.
2.  **Processar Operações de E/S (`BLOCKED -> READY`):** Para cada processo presente na fila de bloqueados (`BLOCKED`), reduz em `1` unidade o seu tempo de surto de E/S restante. Se o surto de E/S zerar, o processo é movido imediatamente para a fila de `READY`.
3.  **Executar Aging (Envelhecimento):** Se a política ativa for a de *Escalonador de Prioridades*, o simulador deve varrer todos os processos na fila `READY` e incrementar sua prioridade se o limite de tempo em espera (*aging threshold*) for atingido.
4.  **Escalonamento da CPU:**
    *   Se não houver processo ativo no estado `RUNNING`, o escalonador é invocado para escolher um dos processos `READY` segundo o algoritmo configurado.
    *   Se houver um processo no estado `RUNNING`, o escalonador decide se ocorre preempção (por estouro de quantum ou chegada de processo com prioridade superior).
5.  **Simular Ciclo de Execução:** Se existir um processo rodando em `RUNNING`:
    *   Soma `1` ao `cpu_time_used` do processo.
    *   Subtrai `1` do tempo restante do surto de CPU atual do processo.
    *   Incrementa o registrador `pc` virtual.
    *   Soma `1` ao `global_clock`.
    *   *Nota:* Se a CPU estiver ociosa (nenhum processo `READY` ou `RUNNING`, mas há processos em `BLOCKED` ou esperando chegada futura em `NEW`), o simulador deve apenas incrementar o `global_clock` e avançar as simulações de E/S.

---

## 2. Bloco de Controle de Processo (PCB)

Cada processo de usuário dentro do simulador é representado exclusivamente por uma estrutura de dados dedicada: o **Bloco de Controle de Processo (PCB)**. A especificação exata em linguagem C da estrutura `PCB_t` e de seus tipos auxiliares é descrita a seguir:

```c
#include <stdint.h>

/* Enumeração para o ciclo de vida do processo */
typedef enum {
    STATE_NEW,         /* Processo criado, aguardando momento de chegada */
    STATE_READY,       /* Pronto para execução na CPU virtual */
    STATE_RUNNING,     /* Em execução ativa na CPU virtual */
    STATE_BLOCKED,     /* Bloqueado aguardando término de operação de E/S */
    STATE_TERMINATED   /* Execução concluída */
} ProcessState_t;

/* Estrutura para simular o contexto de registradores físicos salvos */
typedef struct {
    uint32_t pc;               /* Program Counter virtual do processo */
    uint32_t sp;               /* Stack Pointer virtual do processo */
    uint32_t r[4];             /* Registradores virtuais de propósito geral R0-R3 */
} CpuContext_t;

/* Enumeração para identificar o tipo de surto atual do processo */
typedef enum {
    BURST_CPU,
    BURST_IO
} BurstType_t;

/* Estrutura interna para definir um surto */
typedef struct {
    BurstType_t type;          /* Tipo do surto: CPU ou IO */
    uint32_t duration;         /* Duração total deste surto em ticks de clock */
} Burst_t;

/* Estrutura Principal do Bloco de Controle de Processo (PCB) */
typedef struct PCB {
    uint32_t pid;                  /* Identificador numérico único do processo */
    ProcessState_t state;          /* Estado de execução atual */
    uint32_t base_priority;        /* Prioridade estática (0 a 5, onde 5 é máxima) */
    uint32_t current_priority;     /* Prioridade dinâmica calculada com aging */
    
    CpuContext_t context;          /* Contexto salvo dos registradores de CPU */

    /* Definição de Surtos (Bursts) */
    Burst_t *burst_list;           /* Vetor dinâmico contendo a sequência de surtos */
    uint32_t total_bursts;         /* Quantidade de elementos na burst_list */
    uint32_t current_burst_index;  /* Índice do surto em processamento atual */
    uint32_t burst_time_remaining; /* Ciclos de clock restantes para o surto atual */

    /* Métricas de Tempo e Contabilidade */
    uint32_t arrival_time;         /* Tempo de simulação no qual o processo chega (NEW -> READY) */
    uint32_t first_run_time;       /* Tempo de simulação em que o processo roda pela primeira vez */
    uint32_t completion_time;      /* Tempo de simulação em que o processo entra em TERMINATED */
    uint32_t cpu_time_used;        /* Total de ticks consumidos executando em CPU */
    uint32_t waiting_time;         /* Total de ticks passados no estado READY */
    uint32_t io_time_used;         /* Total de ticks passados no estado BLOCKED */
    uint32_t last_ready_time;      /* Tempo de simulação da última vez que entrou na fila READY */

    /* Campo auxiliar para controle de filas dinâmicas no simulador */
    struct PCB *next;              /* Ponteiro para encadeamento em listas */
} PCB_t;
```

### Diretrizes de Implementação do PCB:
1.  **Salvamento de Contexto:** Sempre que um processo mudar de `RUNNING` para `READY` ou `BLOCKED`, o simulador deve copiar os estados atuais da CPU (valores dos registradores virtuais da simulação global) para os campos correspondentes dentro de `PCB->context`. Ao retornar a `RUNNING`, os dados devem ser restaurados.
2.  **Métricas Finais:** No instante da transição para `TERMINATED`, as métricas de tempo devem ser congeladas de acordo com as seguintes fórmulas:
    *   **Turnaround Time ($T_{turn}$):** $T_{turn} = \text{completion\_time} - \text{arrival\_time}$
    *   **Waiting Time ($T_{wait}$):** Somatório dos ciclos acumulados em que o estado era `READY`. Deve ser incrementado a cada clock que o processo passa na fila de prontos.
    *   **Response Time ($T_{resp}$):** $T_{resp} = \text{first\_run\_time} - \text{arrival\_time}$

---

## 3. Ciclo de Vida e Grafo de Transição de Estados

O ciclo de vida dos processos simulados obedece rigorosamente ao modelo clássico de 5 estados proposto por Tanenbaum, adicionando os gatilhos explícitos de transição.

```
                   +-------------+
                   |     NEW     |
                   +------+------+
                          | (T1) Chegada (arrival_time == clock)
                          v
                   +-------------+
        +--------> |    READY    | <---------+
        |          +------+------+           |
        |                 |                  |
        |                 | (T2) Despacho    | (T3) Preempção
        |                 v    (Escalonador) |      (Quantum ou Prioridade)
        |          +-------------+           |
        | (T5) E/S |   RUNNING   | ----------+
        | Concluída+------+------+
        |                 |
        |                 | (T4) Solicitação E/S (Início de surto de E/S)
        |                 v
        |          +-------------+
        +--------- |   BLOCKED   |
                   +-------------+
                          |
                          | (T6) Término (Fim do último surto de CPU)
                          v
                   +-------------+
                   | TERMINATED  |
                   +-------------+
```

### 3.1 Mapeamento Detalhado de Gatilhos e Regras de Transição

| ID | Transição Original | Gatilho Técnico (Trigger) | Ações do Simulador (Efeitos Colaterais) |
|:---|:---|:---|:---|
| **T1** | `NEW -> READY` | `arrival_time == global_clock` | O PCB é inicializado, o campo `last_ready_time` recebe o `global_clock` e o processo é anexado ao final da fila `READY`. |
| **T2** | `READY -> RUNNING` | Seleção pelo escalonador da CPU | Se `first_run_time` for indefinido, recebe o `global_clock`. O estado muda para `RUNNING`. Restaura-se o contexto salvo de `PCB->context` para os registradores de CPU da simulação. |
| **T3** | `RUNNING -> READY` | Estouro de Quantum ou Preempção de Prioridade | Salva-se o contexto da CPU de volta no PCB do processo. O processo é colocado na cauda da fila `READY` e o escalonador redefine o contador de quantum. |
| **T4** | `RUNNING -> BLOCKED`| Início de surto de E/S (Fim de surto de CPU corrente) | O contexto atual da CPU é persistido no PCB. O processo é inserido na lista de bloqueados com `burst_time_remaining = burst_list[current_burst_index].duration`. |
| **T5** | `BLOCKED -> READY` | `burst_time_remaining == 0` | O processo muda de estado para `READY`, `last_ready_time` é atualizado com o `global_clock`, e ele é anexado à respectiva fila de prontos. |
| **T6** | `RUNNING -> TERMINATED`| Processamento do último surto de CPU concluído | O campo `completion_time` é preenchido com o `global_clock` atual. O estado é alterado para `TERMINATED` e recursos virtuais (como o vetor de bursts) são liberados. |

---

## 4. Escalonador de CPU

O simulador deve implementar de forma isolada e intercambiável dois algoritmos de escalonamento principais. A lógica abstrata de ambos é definida a seguir:

### 4.1 Algoritmo Circular (Round Robin)

O algoritmo Round Robin assume que todos os processos possuem pesos iguais e os gerencia de forma preemptiva por fatia de tempo (*quantum*).

*   **Configuração:** O simulador aceita um parâmetro inteiro positivo global chamado `quantum_size` (em ciclos de clock).
*   **Gerenciamento da Fila:** Mantém uma fila linear FIFO clássica de processos `READY`.
*   **Funcionamento:**
    1.  Ao despachar um processo, o simulador inicia um contador interno `quantum_counter = 0`.
    2.  A cada tick do relógio de simulação em que o processo execute, `quantum_counter` e `cpu_time_used` são incrementados em 1.
    3.  Se `quantum_counter == quantum_size` e o processo em execução ainda possuir surto de CPU pendente:
        *   Ocorre a preempção do processo (`RUNNING -> READY`).
        *   O processo preemptado é reinserido no fim da fila `READY`.
        *   O escalonador busca o próximo processo da cabeça da fila `READY` para rodar.
    4.  Se o surto de CPU do processo terminar antes que `quantum_counter == quantum_size`, o processo voluntariamente abdica da CPU (`RUNNING -> BLOCKED` ou `RUNNING -> TERMINATED`), e o contador de quantum é reiniciado para o próximo processo despachado.

### 4.2 Algoritmo de Prioridades com Aging (Preemptivo)

O escalonador por prioridades atribui a cada processo um nível de importância, onde processos mais prioritários monopolizam o processamento sobre os menos prioritários.

*   **Configuração de Prioridades:** Níveis inteiros de `0` (menor prioridade) a `5` (maior prioridade).
*   **Múltiplas Filas:** O simulador deve manter **6 filas de prontos independentes**, uma para cada nível de prioridade (de 0 a 5).
*   **Decisão de Escalonamento:**
    *   O escalonador sempre inspeciona as filas de maior índice para menor índice. O processo a ser despachado será sempre o da cabeça da fila de maior prioridade não vazia.
    *   Se um processo estiver executando e um novo processo ingressar no estado `READY` com uma prioridade dinâmica estritamente superior ao processo em execução, ocorre **preempção imediata** (preempção por prioridade). O processo executando retorna à sua respectiva fila `READY`.
*   **Mecanismo de Aging (Envelhecimento):**
    Para evitar a inanição (*starvation*) crônica de processos de baixa prioridade, o simulador deve conter a lógica de envelhecimento:
    *   **Regra:** Se um processo permanecer esperando na fila `READY` por mais de `AGING_THRESHOLD` ciclos de clock simulados (calculados como `global_clock - last_ready_time > AGING_THRESHOLD`), sua prioridade dinâmica (`current_priority`) deve ser incrementada em `1` unidade (até o teto de `5`).
    *   Ao sofrer o incremento, o processo é movido da sua fila de prontos atual para a fila correspondente ao seu novo nível de prioridade mais alto. Seu campo `last_ready_time` é atualizado para o clock atual.
    *   **Restauração da Prioridade:** Assim que o processo que sofreu aging for selecionado e entrar em execução (`STATE_RUNNING`), sua prioridade dinâmica (`current_priority`) deve ser imediatamente redefinida para seu valor de prioridade inicial (`base_priority`).

---

## 5. Entradas, Testes e Saídas

Para garantir a portabilidade com ferramentas automatizadas e testes rigorosos, o simulador deve ler configurações padronizadas de entrada de arquivo e gravar os resultados em formatos de log específicos.

### 5.1 Sintaxe do Arquivo de Entrada (`tasks.txt`)

O simulador lerá as tarefas do arquivo de texto `tasks.txt`. Linhas iniciadas com `#` ou vazias serão ignoradas. A sintaxe de cada linha útil segue o formato CSV delimitado por ponto e vírgula:

`PID;arrival_time;base_priority;burst_sequence`

*   `PID`: Inteiro único de 32 bits identificador da tarefa.
*   `arrival_time`: Inteiro não negativo (tempo de chegada do processo).
*   `base_priority`: Inteiro de 0 a 5.
*   `burst_sequence`: Surtos separados por vírgula. A letra `C` indica um surto de CPU e a letra `I` indica um surto de E/S (*I/O*). Surtos devem obrigatoriamente intercalar entre `C` e `I`, iniciando sempre com `C`.

**Exemplo de `tasks.txt`:**
```text
# Exemplo de arquivo de carga de trabalho para simulação
# PID;arrival_time;base_priority;burst_sequence
1;0;3;C5,I10,C3
2;2;1;C2,I5,C4
3;5;5;C4
```

### 5.2 Log de Transições de Estado

O simulador gravará no console (stdout) e em um arquivo chamado `simulation_trace.log` a ocorrência exata das transições de estado a cada tick de relógio global. Cada entrada do log deve conter estritamente a sintaxe:

`[CLOCK: <clock_atual>] PROCESS <pid>: <ESTADO_ANTERIOR> -> <NOVO_ESTADO> (<Motivo da Transição>)`

**Exemplo Real de Execução:**
```text
[CLOCK: 0] PROCESS 1: NEW -> READY (Process arrived)
[CLOCK: 0] PROCESS 1: READY -> RUNNING (Scheduled by CPU Scheduler)
[CLOCK: 2] PROCESS 2: NEW -> READY (Process arrived)
[CLOCK: 5] PROCESS 1: RUNNING -> BLOCKED (Requesting I/O burst of 10)
[CLOCK: 5] PROCESS 3: NEW -> READY (Process arrived)
[CLOCK: 5] PROCESS 3: READY -> RUNNING (Scheduled by CPU Scheduler)
[CLOCK: 9] PROCESS 3: RUNNING -> TERMINATED (Completed final execution)
```

### 5.3 Gráfico de Gantt Textual

Para permitir a análise visual do fluxo de tempo e facilitar a validação de comportamento por agentes de inteligência artificial, o simulador produzirá uma linha textual simulando o Gráfico de Gantt da CPU de forma discretizada. Cada caractere ou bloco representa o processo ativo naquele tick de tempo (ticks ociosos devem ser representados pelo caractere `_` ou `Idle`).

**Exemplo de Formato Gráfico de Gantt Textual:**
```text
======================= GRÁFICO DE GANTT (CPU) =======================
0    5    10   15   20   25   30   35   40   45   50
|P1111|P3333|P22|P111|P2222|Idle|P222|...
======================================================================
```
*(Nota: No exemplo acima, o Processo 1 rodou do clock 0 ao 5, seguido pelo Processo 3 do clock 5 ao 9, seguido pelo Processo 2 do clock 9 ao 11, etc).*

### 5.4 Relatório Final de Métricas de Desempenho

Ao concluir a simulação de todas as tarefas carregadas, o programa gerará na saída padrão um painel consolidado contendo métricas do sistema e estatísticas detalhadas por processo:

```text
========================================================================
              RELATÓRIO DE DESEMPENHO DA SIMULAÇÃO (SO)
========================================================================
Algoritmo de Escalonamento Ativo: ROUND ROBIN (Quantum = 4)
Tempo Total de Simulação:         28 unidades de clock
Vazão do Sistema (Throughput):    0.11 processos/clock
Grau de Utilização de CPU:        89.29%
Grau de Ociosidade de CPU:        10.71%

Métricas de Desempenho por Processo:
------------------------------------------------------------------------
PID | Chegada | Término | Turnaround | Espera (Ready) | Resposta | E/S
------------------------------------------------------------------------
1   | 0       | 23      | 23         | 5              | 0        | 10
2   | 2       | 28      | 26         | 15             | 7        | 5
3   | 5       | 9       | 4          | 0              | 0        | 0
------------------------------------------------------------------------

Métricas Estatísticas Consolidadas (Médias):
> Tempo Médio de Turnaround (Average Turnaround):  17.67 unidades
> Tempo Médio de Espera (Average Waiting Time):    6.67 unidades
> Tempo Médio de Resposta (Average Response Time):  2.33 unidades
========================================================================
```

---

## 6. Critérios de Robustez e Validação Técnico-Algorítmica

Para garantir que o código gerado por ferramentas de automação seja resiliente, os seguintes testes de estresse conceituais devem ser contemplados pelo código fonte:

1.  **Inexistência de Deadlocks:** O escalonador deve sempre possuir uma condição de saída válida e o tempo de ociosidade (`Idle`) deve ser contabilizado se houver processos apenas na fila de bloqueados (`BLOCKED`) ou aguardando chegada futura.
2.  **Validação de Entrada:** Se `tasks.txt` contiver formatação corrompida, PIDs repetidos ou tempos de chegada desordenados, o programa deve acusar erro em tempo de carregamento e abortar a execução com código de saída adequado.
3.  **Integridade das Métricas:** O somatório do tempo total gasto na simulação deve ser coerente com a soma de ticks de CPU ativos, tempo ocioso total e o tempo de conclusão do último processo.
4.  **Resgate de Contexto:** Os valores mantidos na struct `CpuContext_t` não podem sofrer *leaks* ou corrupção entre as chamadas de escalonamento. Os registradores virtuais simulados precisam refletir fielmente o progresso linear dos registradores no instante que o processo for restaurado para a CPU.
