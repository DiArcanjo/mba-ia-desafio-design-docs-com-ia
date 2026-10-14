# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autora** | Diana Arcanjo |
| **Status** | Em revisão |
| **Data** | 2026-10-14 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenharia de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |
| **Próximo passo** | Sessão de revisão do design com Bruno e Diego antes do início da implementação |

## 1. Resumo executivo (TL;DR)

Propomos notificar os clientes B2B por **webhooks de saída** sempre que o status de um pedido muda.

- A mudança de status grava o evento em uma **outbox no MySQL**, na mesma transação.
- Um **worker em processo separado** lê a outbox **a cada 2 segundos** e entrega o evento por HTTPS, **assinado com HMAC-SHA256** usando uma secret por endpoint.
- Falhas são retentadas com **backoff exponencial** (5 retentativas, até cerca de 15h) e depois vão para uma **DLQ**, com replay manual por administrador.
- A garantia é **at-least-once**, e o cliente deduplica pelo `X-Event-Id`.
- Tudo segue os padrões já existentes no projeto. Não entra infraestrutura nova.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para ser avisados quando o status dos pedidos deles muda. Hoje eles fazem polling em `GET /orders`, o que deixa a integração lenta e cara. Para eles, "tempo real" é qualquer coisa abaixo de 10 segundos. A Atlas indicou que pode migrar para um concorrente se a solução não sair até o fim do trimestre.

O OMS não tem nenhum mecanismo de notificação externa, eventos ou filas. A mudança de status é feita por `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), em uma transação que já atualiza o pedido, grava o histórico de status e movimenta estoque. Qualquer solução precisa preservar a atomicidade dessa transação e não pode deixá-la refém da disponibilidade dos clientes.

## 3. Proposta técnica

### Visão geral

```
PATCH /orders/:id/status
        │
        ▼
OrderService.changeStatus ──(mesma transação)──► publishWebhookEvent(tx, ...)
        │                                              │ insere 1 linha por endpoint
        ▼                                              ▼ que assina o status
   commit / rollback                           tabela webhook_outbox (PENDING)
                                                       │
                     src/worker.ts (processo separado, polling 2s)
                                                       │
                             POST https://cliente/... (assinado com HMAC)
                              │ 2xx            │ falha
                              ▼                ▼
                          DELIVERED      retry 1m/5m/30m/2h/12h ──► webhook_dead_letter
                                                                       │
                                            POST /admin/webhooks/dead-letter/:id/replay (ADMIN)
```

### Componentes

1. **Configuração de endpoints.** Os clientes cadastram, editam, listam e removem endpoints pela API autenticada do OMS. Cada endpoint tem URL `https`, uma secret própria gerada pela plataforma (rotacionável, com 24h de convivência) e a lista de status que deseja receber.
2. **Publicação transacional.** `changeStatus` chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação existente. A função aplica o filtro de status e grava um snapshot enxuto do evento. Se falhar, a mudança de status sofre rollback.
3. **Worker de entrega.** Um processo próprio (`npm run worker`), com o mesmo banco e um `PrismaClient` próprio. Ele busca pendentes em ordem de criação, envia cada evento com identificador, assinatura e identificação do endpoint, e registra cada tentativa.
4. **Resiliência.** Timeout de 10s, 5 retentativas com backoff exponencial, DLQ em tabela própria e replay manual auditado.
5. **Observabilidade e consulta.** O cliente consulta o histórico das últimas entregas de cada endpoint. Os logs usam o Pino existente.

Contratos, modelo de dados, códigos de erro e fluxos detalhados estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte |
|---|---|---|
| RFC-ALT-01 | **Disparo HTTP síncrono em `changeStatus`** | Acoplaria a transação de status à latência e à disponibilidade dos clientes. Um cliente lento travaria mudanças de outros pedidos, e um cliente fora do ar não justifica rollback. |
| RFC-ALT-02 | **Redis Streams / Redis Cluster como fila** | Entregaria a mesma função com mais infraestrutura para operar. Para um time pequeno, foi considerado overengineering, já que o MySQL atual resolve. |
| RFC-ALT-03 | **Trigger de banco para acordar o worker** | O MySQL não tem `LISTEN/NOTIFY`. A trigger não avisa processo externo e exigiria improvisos. O polling de 2s já atende o requisito de 10s. |
| RFC-ALT-04 | **Retry indefinido ou só 3 tentativas** | O indefinido deixa eventos pendurados para sempre. Com 3, as tentativas se esgotam em cerca de 30 minutos, menos que indisponibilidades já observadas (2h). |
| RFC-ALT-05 | **Garantia exactly-once** | Exige coordenação dos dois lados e muito mais complexidade. O at-least-once com `event_id` resolve a grande maioria dos casos e é o padrão de mercado. |
| RFC-ALT-06 | **Secret global da plataforma** | Um único vazamento comprometeria todos os clientes. |

## 5. Questões em aberto

| # | Questão | Origem | Encaminhamento |
|---|---|---|---|
| RFC-Q-01 | **Rate limiting de saída por cliente.** Um cliente com 50 pedidos mudando por minuto recebe 50 chamadas. | [09:38] Diego, [09:39] Larissa | Fora do escopo. Observar em produção e decidir se virar problema. |
| RFC-Q-02 | **Aviso ao cliente quando o webhook falha repetidamente** (ex.: e-mail após 3 falhas). | [09:37] Marcos e Larissa | Adiado para a próxima fase, depois de medir o impacto. |
| RFC-Q-03 | **Escala para múltiplos workers.** Particionar por `order_id` ou usar lock pessimista. | [09:13] Bruno e Diego | Adiado. Hoje a premissa é worker único ([ADR-008](adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md)). |
| RFC-Q-04 | **Endurecer a autorização do CRUD de webhooks.** Hoje qualquer usuário autenticado gerencia endpoints de qualquer cliente. | [09:37] Sofia | Adiado: "mais pra frente a gente pode endurecer". |
| RFC-Q-05 | **Arquivamento das linhas entregues da outbox** (cerca de 30 dias). | [09:08] Diego | Fora do escopo desta feature. |
| RFC-Q-06 | **Contagem de tentativas.** "5 tentativas" vs. "quase 15 horas". O FDD assume tentativa inicial + 5 retentativas. | [09:15] e [09:17] Diego | Confirmar com Diego na revisão deste RFC. |
| RFC-Q-07 | **`customer_id` no body ou no path** dos endpoints de configuração. | [09:32] Larissa | O FDD propõe body na criação e path na listagem. Validar com Larissa e Bruno na sessão de revisão. |
| RFC-Q-08 | **Pontos para a revisão de segurança.** (a) O `X-Timestamp` não entra no HMAC, que cobre só o corpo; assim ele não protege sozinho contra replay. (b) Formato de duas assinaturas no `X-Signature` durante o grace period. (c) Uma segunda rotação dentro de 24h encurta o prazo da secret mais antiga. | [09:44] Diego, [09:21] Sofia, [09:46] Sofia | Levar para a revisão da Sofia antes do deploy. |
| RFC-Q-09 | **Listagem da DLQ via API.** Só o replay foi discutido; sem listagem, o admin descobre o ID da dead letter pelo log ou pelo banco. | [09:18] Diego | Decidir se entra nesta fase ou na próxima. |

## 6. Impacto e riscos

**Impacto no sistema existente**
- Mais uma escrita, e uma leitura dos endpoints do cliente, dentro da transação de `changeStatus`.
- Quatro tabelas novas no MySQL: configuração de endpoints, outbox, histórico de entregas e DLQ.
- Um novo processo para operar (`npm run worker`).
- Nenhuma mudança de comportamento nos endpoints atuais.

**Riscos principais** (a matriz completa com probabilidade e impacto está no [PRD](PRD.md#10-riscos-e-mitigação))
- **Cliente recebe duplicatas.** Mitigação: `X-Event-Id` estável e documentação no portal do desenvolvedor.
- **Vazamento de secret.** Mitigação: secret por endpoint, rotação com grace period e revisão de segurança de 2 dias antes do deploy.
- **Ordem quebrada ao escalar workers.** Mitigação: manter worker único até existir a estratégia da RFC-Q-03.
- **Prazo.** A estimativa é de 3 sprints, incluindo a revisão da Sofia, contra o pedido da Atlas para o fim de novembro.

## 7. Decisões relacionadas

| ADR | Decisão |
|---|---|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2s |
| [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ |
| [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação |
| [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões do projeto |
| [ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md) | Snapshot do payload e filtro na inserção |
| [ADR-008](adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md) | Ordenação por pedido com worker único |
