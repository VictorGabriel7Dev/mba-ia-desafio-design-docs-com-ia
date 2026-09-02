# Da Reunião ao Documento: pacote de design docs do Sistema de Webhooks

> Entrega do **Desafio 3 (fase 299)** do MBA em Engenharia de Software com IA, da Full Cycle.
> O enunciado original está preservado em [`docs/ENUNCIADO-ORIGINAL.md`](docs/ENUNCIADO-ORIGINAL.md).

## Sobre o desafio

A tarefa é transformar a transcrição de uma reunião técnica de 55 minutos em documentação de
engenharia acionável. A reunião decidiu como construir um **Sistema de Webhooks de Notificação de
Pedidos** para um OMS em produção, e nada além da transcrição foi registrado. O trabalho é
reconstruir, a partir dela e do código existente, o pacote que o time precisaria para começar a
implementar: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

O que torna o desafio diferente de "resumir uma reunião" é a exigência de **rastreabilidade**. Cada
requisito, decisão e restrição precisa apontar para o trecho da transcrição ou o arquivo de código
que o originou. Isso muda o modo de trabalhar: em vez de produzir um documento plausível, você
produz um documento **verificável**, e qualquer coisa que não fecha com a fonte é sinal de que a IA
inventou.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (Opus) | Ferramenta principal. Leu a transcrição e o código do repositório diretamente do disco, produziu os documentos e fez a auditoria final contra os critérios de aceite |
| **Scripts próprios de verificação** | Escritos durante o trabalho para conferir o tracker: contagem de fontes, validação do formato de timestamp e checagem de que todo arquivo citado existe de fato no repositório |

A escolha do Claude Code em vez de uma interface de chat foi deliberada: o desafio exige citar
caminhos reais de arquivo, e uma ferramenta que **lê o repositório** elimina a categoria de erro
mais provável, que é citar um arquivo que não existe.

## Workflow adotado

A ordem sugerida no enunciado é ADRs → RFC → FDD → PRD → Tracker → README. **Eu não segui essa
ordem**, e vale registrar por quê e o que isso custou.

A ordem que usei foi:

1. **Leitura da transcrição inteira**, sem resumir. São 324 linhas, e cada decisão tem timestamp e
   autor. Ler inteiro é o que permite preencher a coluna `Localização` do tracker depois.
2. **Mapeamento do código**: estrutura de `src/modules/orders/`, o método `changeStatus` e sua
   transação, as classes de erro, os middlewares e o `prisma/schema.prisma`.
3. **PRD**, depois **RFC**, depois os **7 ADRs**.
4. **Auditoria contra os critérios de aceite** (ver iteração 1 abaixo).
5. **FDD**, com a seção obrigatória de integração.
6. **Tracker**, varrendo os documentos prontos.
7. **Verificação automatizada** do tracker e dos caminhos de arquivo.
8. **Este README**.

Começar pelo PRD em vez dos ADRs fez o PRD nascer com uma tabela de decisões apontando para ADRs
que ainda não existiam. Funcionou porque as decisões já estavam claras na transcrição, mas a ordem
do enunciado é melhor: escrever a decisão primeiro força o raciocínio antes da consolidação.

## Prompts customizados

### 1. Extração de decisões com âncora obrigatória

Este foi o prompt que estabeleceu a regra que valeu para o pacote inteiro: **nada entra sem
origem**.

```
Leia TRANSCRICAO.md por completo, sem resumir.

Extraia toda DECISÃO TÉCNICA tomada. Para cada uma, devolva:
  - a decisão em uma frase
  - o timestamp e o nome de quem a formulou, no formato [hh:mm] Nome
  - a alternativa que foi descartada, se houve, e o motivo declarado do descarte
  - se ficou decidido, adiado ou em aberto

REGRAS:
- Não infira decisão que não foi verbalizada. Se está implícito, marque como INFERIDO
  e explique de onde tirou.
- Item descartado na reunião (email, dashboard, rate limiting) NÃO é requisito.
  Liste separado, como "fora de escopo", com o timestamp de quando foi descartado.
- Se você não consegue apontar timestamp para um item, ele não entra na lista.
```

A última regra é a que mais rendeu. Ela transforma "a IA pode alucinar" em um teste mecânico: sem
timestamp, o item não existe.

### 2. Auditoria adversarial contra os critérios de aceite

Escrito depois de descobrir que eu tinha pulado parte do enunciado. Em vez de pedir revisão
genérica, ele pede para **procurar a falha**.

```
Leia os "Critérios de Aceite" do enunciado, item por item, e confronte com os
documentos que já produzi em docs/.

Para CADA checkbox, responda em uma linha:
  ATENDE  <critério>  <onde está no documento>
  FALHA   <critério>  <o que exatamente falta>

Não escreva nada além dessa lista. Não elogie o que está pronto.
Se um critério exige um número mínimo (8 requisitos, 4 endpoints, 5 linhas de
código no tracker), CONTE e mostre o número encontrado.
```

Foi esse prompt que encontrou a falha da iteração 1.

### 3. Verificação de existência dos arquivos citados

O critério de consistência exige que nenhum arquivo mencionado seja inexistente. Em vez de conferir
no olho, virou script:

```python
# extrai todo caminho .ts/.prisma citado no tracker e testa no disco
for l in linhas_com_fonte_codigo:
    for p in re.findall(r'`([a-z][\w/.-]+\.(?:ts|prisma))`', l):
        print('OK' if pathlib.Path(p).exists() else 'NAO EXISTE', p)
```

## Iterações e ajustes

Foram quatro ciclos relevantes. Os dois primeiros são erros meus, corrigidos.

### Iteração 1: eu não tinha lido o enunciado inteiro

**O erro.** Produzi PRD, RFC e os 7 ADRs tendo lido **175 das 294 linhas** do enunciado. As 119
linhas que faltavam continham justamente os **Critérios de Aceite**, ou seja, a régua da correção.
Escrevi os documentos sem saber como seriam avaliados.

**Como apareceu.** Uma pergunta direta: *"você leu as introduções com atenção? elas estão realmente
longas."* Fui contar as linhas em vez de responder de memória, e o número mostrou o buraco.

**O que estava errado.** O critério do PRD exige que a seção de Riscos tenha **probabilidade,
impacto e mitigação**. Minha tabela tinha só impacto e mitigação. Um critério objetivo, reprovado.

**A correção.** Reescrevi a tabela com as três colunas, e a probabilidade de cada risco vem
justificada, não é um rótulo solto. Exemplo: vazamento de secret é *"Média, já aconteceu antes,
cliente vazou secret em log de aplicação"*, com o timestamp `[09:22]`.

**A lição.** Ler o enunciado inteiro **antes** de produzir é mais barato que auditar depois. E
serve para a IA e para quem a dirige: eu pedi para gerar antes de ter lido a régua.

### Iteração 2: ordem de produção invertida

Comecei pelo PRD, quando o enunciado sugere ADRs primeiro. O sintoma apareceu na tabela de decisões
do PRD, que citava sete ADRs que ainda não existiam. Não invalidou o resultado, porque as decisões
estavam explícitas na transcrição, mas obrigou a voltar e conferir se cada link tinha virado
arquivo. Registrado aqui em vez de escondido: a ordem sugerida existe por um motivo.

### Iteração 3: alternativas que a IA inventou

Na primeira versão dos ADRs, apareceram alternativas plausíveis que **ninguém discutiu na reunião**
(mTLS no ADR de segurança, circuit breaker na resiliência). São ideias razoáveis, e é exatamente
por isso que passariam batido.

A correção não foi apagar, foi **rotular**: as duas continuam nos documentos, marcadas
explicitamente como *"não discutida na reunião, registrada aqui como alternativa plausível"*. E
entraram na seção "Itens verificados e descartados" do tracker, para que ninguém as leia como
decisão da equipe.

### Iteração 4: verificação mecânica em vez de leitura

A checagem final do tracker não foi feita lendo. Um script contou as fontes (85 linhas: 88%
`TRANSCRICAO`, 10 `CODIGO`), validou que os 75 timestamps estão no formato `[hh:mm] Nome` e testou
no disco cada um dos 12 caminhos de arquivo citados. Os três critérios numéricos do tracker viraram
um comando que passa ou falha, em vez de uma impressão.

## Como navegar a entrega

Ordem de leitura sugerida:

| # | Arquivo | O que é |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](docs/PRD.md) | O produto: problema, escopo, 12 requisitos funcionais, riscos e critérios de aceitação |
| 2 | [`docs/RFC.md`](docs/RFC.md) | A proposta técnica: a abordagem, 7 alternativas descartadas com o trade-off de cada uma, e 5 questões deixadas em aberto |
| 3 | [`docs/adrs/`](docs/adrs/) | As 7 decisões, uma por arquivo, em formato MADR |
| 4 | [`docs/FDD.md`](docs/FDD.md) | O "como construir": modelo de dados, fluxos, 6 contratos HTTP, matriz de erros e a **integração com o código existente** |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | A rastreabilidade de tudo acima |

**Se você só vai ler um arquivo**, leia o [TRACKER](docs/TRACKER.md). Ele mostra de onde veio cada
item do pacote e, no fim, o que foi **deliberadamente deixado de fora**.

### Os ADRs

| ADR | Decisão |
| --- | --- |
| [001](docs/adrs/ADR-001-outbox-no-mysql.md) | Padrão outbox no MySQL existente |
| [002](docs/adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2s |
| [003](docs/adrs/ADR-003-retry-backoff-e-dlq.md) | Retry com backoff e DLQ em tabela separada |
| [004](docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint |
| [005](docs/adrs/ADR-005-at-least-once-com-event-id.md) | At-least-once com `X-Event-Id` |
| [006](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões do projeto |
| [007](docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md) | Snapshot do payload na inserção |

> O código da aplicação (`src/`, `prisma/`, `tests/`) **não foi alterado**, conforme a restrição do
> enunciado. A entrega é puramente documental.
