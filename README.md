# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

> Enunciado original do desafio: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)

## Sobre o desafio

O ponto de partida é um Order Management System (OMS) em Node.js + TypeScript, com Prisma e MySQL, e a transcrição de uma reunião de 55 minutos. Nela, tech lead, PM, dois engenheiros e a engenheira de segurança decidem como construir um sistema de webhooks que avisa clientes B2B quando o status dos pedidos muda. Nada foi registrado além da transcrição.

Meu trabalho foi transformar essa conversa, junto com o código existente, em um pacote de design docs acionável: PRD, RFC, FDD, 8 ADRs e um tracker que liga cada item à fala ou ao arquivo de onde ele veio. A regra mais importante era não inventar nada, e tão importante quanto registrar o que foi decidido era deixar de fora o que a reunião descartou ou adiou.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (modelo Claude Opus) | Ferramenta principal. Leu a transcrição e o código direto do repositório, fez a extração dirigida das decisões, gerou os rascunhos de cada documento e aplicou as correções. |
| **Subagente revisor do Claude Code** | Um agente separado, sem acesso ao "raciocínio" de quem escreveu, rodando como auditor cético contra a transcrição e o código. Usei em duas rodadas. |
| **Scripts Python gerados com o Claude Code** | Um gerador para o `TRACKER.md` a partir de uma lista estruturada, e um validador que confere cada `[hh:mm] Nome` contra a transcrição, cada caminho de arquivo contra o repositório, os links entre documentos e a cobertura de IDs no tracker. |

## Workflow adotado

1. **Contextualização.** Pedi ao Claude Code para ler a transcrição inteira e os arquivos que a feature toca: `order.service.ts`, `order.status.ts`, `schema.prisma`, os erros em `src/shared/errors/`, os middlewares, `app.ts`, `server.ts`, `routes/index.ts`, o logger e o `package.json`.
2. **Matriz de extração antes de qualquer documento.** Separei a reunião em quatro listas: decisões fechadas, requisitos, itens descartados, e itens adiados ou em aberto. Cada linha tinha timestamp e falante. Essa etapa revelou ambiguidades que eu teria deixado passar (ver iterações).
3. **ADRs primeiro.** Foram 8 ADRs: as 6 decisões principais, mais "snapshot e filtro na inserção" e "ordenação com worker único", que pesavam demais na implementação para ficar só no FDD.
4. **RFC** em cima dos ADRs, curto, com as alternativas descartadas e as questões em aberto da reunião.
5. **FDD**, o documento mais longo: modelo de dados, fluxos, 8 contratos, matriz de erros `WEBHOOK_*` e a seção de integração com 20 arquivos reais do código base.
6. **PRD** por último entre os grandes, como consolidação em linguagem de produto.
7. **Tracker** gerado por script, com um ID em cada item dos documentos.
8. **Validação automática**, depois **auditoria independente**, correções, **segunda auditoria** e novas correções.
9. **README** no fim, com o processo já completo.

Uma regra que mantive o tempo todo: quando o FDD precisava de um detalhe que a reunião não definiu (tamanho do lote do worker, formato exato do `X-Signature`, rota de rotação), ele entra marcado como **[proposta do FDD]**. Ele nunca aparece como se tivesse sido decidido na call.

## Prompts customizados

**1. Extração dirigida da transcrição** (usado antes de escrever qualquer documento)

```text
Leia TRANSCRICAO.md inteira. Não escreva nenhum documento ainda.
Monte uma matriz com quatro seções, cada linha com [hh:mm] e o nome de quem falou:
1. DECISÕES FECHADAS: só o que alguém explicitamente decidiu ou o grupo confirmou
   (ex.: "Decidido", "Tá decidido", resumo final da Larissa). Inclua os valores exatos.
2. REQUISITOS FUNCIONAIS pedidos pelo PM ou pela Segurança.
3. DESCARTADO: alternativas colocadas na mesa e rejeitadas, com o motivo dito na reunião.
4. ADIADO / EM ABERTO: o que ficou para depois, para observar ou sem decisão.
Por último, liste AMBIGUIDADES: falas que se contradizem ou valores que não fecham entre si.
Não complete lacunas com boas práticas de mercado. Se não foi dito, não entra.
```

**2. Geração dos ADRs** (adaptado do template MADR do curso)

```text
Escreva um ADR por decisão da matriz, em docs/adrs/ADR-NNN-titulo-em-kebab-case.md.
Seções obrigatórias: Status, Data, Decisores, Contexto, Decisão, Alternativas Consideradas,
Consequências (Positivas, Negativas e uma linha "Trade-off:" explícita), Referências.
Regras:
- Alternativas só podem ser as que foram discutidas na reunião, com o motivo real do descarte.
- Contexto deve citar o código existente pelo caminho do arquivo quando a decisão toca nele
  (ex.: OrderService.changeStatus em src/modules/orders/order.service.ts).
- Referências: liste os timestamps no formato [hh:mm] Nome. Não cite um falante num
  timestamp em que ele não fala.
- Se a reunião deixou algo ambíguo, registre como "Nota de interpretação", não como decisão.
```

**3. Auditoria independente** (subagente revisor, sem permissão de edição)

```text
Você é um revisor técnico cético. Audite um pacote de design docs contra a fonte de verdade.
NÃO edite nenhum arquivo, apenas reporte.
Fontes de verdade: TRANSCRICAO.md e o código em src/, prisma/, tests/, package.json.
Procure, com evidência (arquivo:linha + timestamp ou caminho do código):
1. Contradições com a transcrição ou o código.
2. Itens descartados ou adiados aparecendo como escopo.
3. Invenções não sinalizadas (o que estiver marcado como "[proposta do FDD]" é aceitável,
   a menos que contradiga algo).
4. Atribuições erradas de timestamp.
5. Duplicação entre documentos (RFC descendo ao nível do FDD, PRD repetindo detalhe técnico).
6. Lacunas: algo decidido que não aparece em nenhum documento.
7. Incoerência interna (tentativas, nomes de tabela, rotas, códigos de erro, IDs).
Para cada achado: severidade, local, evidência e correção sugerida em uma frase.
```

**4. Reverificação das correções**

```text
Apliquei correções para os 18 achados da sua auditoria. Releia os trechos afetados:
1. Para cada achado, diga se foi RESOLVIDO, PARCIAL ou NÃO RESOLVIDO.
2. Verifique o novo desenho do worker (envio assíncrono + NOT EXISTS por webhook_id/order_id):
   ele garante a ordem por pedido e o isolamento entre clientes? Há bug lógico novo
   (deadlock, evento preso para sempre, contradição com at-least-once ou com "worker único")?
3. Aponte qualquer problema NOVO introduzido pelas correções.
```

## Iterações e ajustes

Foram **4 iterações principais** até a versão final.

**Iteração 1: a matriz revelou uma contradição na própria reunião.** O Diego fala em "5 tentativas", mas também em "quase 15 horas entre primeira falha e última tentativa" com os intervalos 1m/5m/30m/2h/12h. Somando os cinco intervalos dá 14h36, o que só fecha se forem 5 **retentativas** depois da tentativa inicial. Em vez de deixar a IA escolher uma leitura em silêncio, registrei a interpretação no ADR-003, deixei o ADR como "aceito com contagem a confirmar" e abri a questão RFC-Q-06. Fiz o mesmo com o `customer_id` "no body ou no path", que a Larissa deixou em aberto.

**Iteração 2: o validador pegou uma fala inventada.** Na primeira versão, o ADR-006 citava "[09:28] Larissa" como fonte do prefixo `WEBHOOK_`, mas às 09:28 só o Bruno fala; a Larissa diz isso às 09:29. O mesmo script mostrou que 10 IDs do FDD estavam sem linha no tracker (cobertura de 84%) e que o RFC usava IDs curtos (`ALT-01`), incompatíveis com o tracker (`RFC-ALT-01`). Antes disso eu já tinha corrigido dois links de âncora quebrados entre RFC, PRD e ADR-003, e um "três tabelas novas" que na verdade eram quatro.

**Iteração 3: a auditoria independente encontrou 18 problemas, 2 graves.**
- **A ordem por pedido não era garantida.** O ADR-008 prometia ordem por `order_id`, mas o desenho deixava um `SHIPPED` sair enquanto o `PAID` do mesmo pedido estava em backoff. Corrigi com uma regra de elegibilidade no worker: só envia se não houver evento anterior pendente do mesmo pedido para o mesmo endpoint.
- **Um cliente lento travava todos.** O worker processava o lote em sequência, com timeout de 10 s. Passei para envio assíncrono com limite de envios em andamento.
- **Itens médios corrigidos:**
  - o PRD tinha uma meta "p95 < 10 s" que ninguém disse;
  - a listagem usava `customerId` na query, que não era nenhuma das duas opções da reunião;
  - faltava `onDelete: Cascade` na FK para `Customer`, o que quebraria o `CustomerService.delete` existente;
  - o ADR-006 sugeria estender `NotFoundError`, que fixa o código `NOT_FOUND`;
  - o RFC ainda trazia headers e timeout, que são detalhe de FDD;
  - várias regras do FDD não estavam marcadas como proposta.

**Iteração 4: a reauditoria pegou um erro no meu próprio redesenho.** Ela confirmou 15 dos 18 achados como resolvidos, mas mostrou que:
- o limite de envios em andamento era **global**, então um endpoint travado com 10 pedidos pendentes ainda ocupava todas as vagas;
- dois eventos no mesmo milissegundo podiam sair fora de ordem;
- uma exceção fora do `fetch` deixava a linha presa em `PROCESSING` e bloqueava o pedido.

Corrigi com:
- um limite de envios **por endpoint**;
- uma coluna `seq` para desempate (o `id` continua UUID, como decidido às 09:51);
- `try/finally` liberando a vaga;
- um índice para a consulta nova;
- uma exceção explícita no requisito de 10 s quando um evento anterior do mesmo pedido está retido.

O que aprendi: as duas falhas de desenho mais sérias (ordem e isolamento) passaram pela primeira escrita *e* pela primeira correção. Só apareceram porque um segundo agente leu o documento sem saber o que eu "quis dizer". Em documentação gerada com IA, a revisão adversarial rende mais do que pedir para o mesmo modelo "revisar o que escreveu".

## Como navegar a entrega

| Ordem | Arquivo | O que contém |
|---|---|---|
| 1 | [`docs/PRD.md`](docs/PRD.md) | Problema, público, objetivos com metas, escopo e o que ficou de fora, requisitos, riscos e plano de 3 sprints |
| 2 | [`docs/RFC.md`](docs/RFC.md) | A proposta em nível de arquitetura, 6 alternativas descartadas e 9 questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/) | 8 decisões, uma por arquivo, de [`ADR-001`](docs/adrs/ADR-001-outbox-no-mysql.md) a [`ADR-008`](docs/adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md) |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Modelo de dados, fluxos, 8 contratos com exemplos, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e integração com o código existente |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | 233 itens rastreados: 87% na transcrição e 31 no código |
| — | [`TRANSCRICAO.md`](TRANSCRICAO.md) | A reunião original, sem alterações |

**ADRs**

| ADR | Decisão |
|---|---|
| [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL |
| [ADR-002](docs/adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2s |
| [ADR-003](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ |
| [ADR-004](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação |
| [ADR-005](docs/adrs/ADR-005-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md) | Payload snapshot e filtro de eventos na inserção |
| [ADR-008](docs/adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md) | Ordenação por pedido com worker único |

A entrega é só documental: nenhum arquivo em `src/`, `prisma/`, `tests/` ou de configuração foi alterado.
