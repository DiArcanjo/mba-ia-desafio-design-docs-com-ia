# ADR-001: Padrão Outbox no MySQL para eventos de webhook

- **Status:** Aceito
- **Data:** 2026-10-14
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Três clientes B2B pediram para ser notificados quando o status dos pedidos muda, em menos de 10 segundos. Hoje a mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já faz três coisas: atualiza `orders`, insere em `order_status_history` e debita ou repõe `stock_quantity` dos produtos.

Precisamos registrar um evento de notificação para cada mudança de status sem:

- deixar a transação de status dependente da disponibilidade do cliente;
- perder eventos quando a transação commita;
- emitir eventos de mudanças que sofreram rollback.

A aplicação não tem hoje nenhuma fila, broker ou mecanismo de eventos.

## Decisão

Usar o **padrão Outbox na própria base MySQL**. Dentro da mesma transação de `changeStatus`, gravamos uma linha na tabela `webhook_outbox` com o evento. Um worker separado ([ADR-002](ADR-002-worker-separado-em-polling.md)) lê essa tabela e faz as chamadas HTTP.

- A gravação acontece pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `Prisma.TransactionClient` da transação corrente.
- Se a inserção na outbox falhar, a transação inteira sofre rollback. Não existe cenário de status alterado sem evento registrado.
- A tabela tem índices em `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) e em `created_at`, para o worker buscar lotes de pendentes mais antigos.
- Os IDs seguem o padrão UUID do projeto (`@db.Char(36)`).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo HTTP síncrono dentro de `changeStatus`** | A transação já é pesada. Um cliente lento travaria a mudança de status de outros pedidos, e com o cliente fora do ar não há saída aceitável: dar rollback na mudança de status não faz sentido para o negócio. |
| **Redis Streams (ou fila dedicada)** | Exigiria subir e operar infraestrutura nova (Redis Cluster) para um time pequeno. Foi considerado overengineering, já que o MySQL existente resolve. |

## Consequências

**Positivas**
- Atomicidade: o evento existe se, e somente se, a mudança de status foi commitada.
- Nenhuma infraestrutura nova. Usa o MySQL e o Prisma que o projeto já opera.
- A mudança de status deixa de depender da latência e da disponibilidade dos clientes.

**Negativas**
- A tabela `webhook_outbox` cresce continuamente. O arquivamento de linhas entregues (cerca de 30 dias) fica fora desta feature.
- Cada mudança de status passa a fazer mais uma escrita na transação de pedidos.
- A entrega passa a ser assíncrona, com latência mínima igual ao intervalo de polling.

**Trade-off:** trocamos entrega imediata e infraestrutura dedicada de mensageria por consistência transacional e simplicidade operacional.

## Referências

- Transcrição: [09:04] Bruno, [09:06] Diego, [09:07] Larissa e Diego, [09:08] Diego e Larissa, [09:40] Bruno, [09:41] Bruno e Diego, [09:51] Larissa
- Código: `src/modules/orders/order.service.ts` (`changeStatus`), `prisma/schema.prisma`
