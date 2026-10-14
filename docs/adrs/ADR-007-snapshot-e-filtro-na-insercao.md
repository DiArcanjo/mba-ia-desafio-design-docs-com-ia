# ADR-007: Payload enxuto renderizado como snapshot e filtro de eventos na inserção

- **Status:** Aceito
- **Data:** 2026-10-14
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (Produto)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md)

## Contexto

Cada endpoint de webhook escolhe quais status quer receber (por exemplo, apenas `SHIPPED` e `DELIVERED`). Também precisávamos definir:

- em que momento aplicar esse filtro;
- se a outbox guarda o payload pronto ou só a referência ao pedido;
- quanto do pedido vai no payload.

## Decisão

1. **Filtro na inserção.** Ao publicar o evento, `publishWebhookEvent` só grava linhas na outbox para os endpoints ativos do cliente que assinam o status de destino. Se nenhum endpoint quer aquele status, nada é inserido.
2. **Snapshot na inserção.** O payload é renderizado e salvo no momento da mudança de status. Ele reflete o pedido naquele instante, mesmo que o pedido mude depois.
3. **Payload enxuto.** JSON com `event_id`, `event_type` (`order.status_changed`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`. **Sem itens do pedido.** Para detalhes, o cliente consulta `GET /orders/:id`.
4. **Limite de 64 KB.** Um payload acima disso não é enviado nem truncado. É tratado como erro (`WEBHOOK_PAYLOAD_TOO_LARGE`).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Filtrar na hora do envio** | Gravaria linhas que nunca serão entregues, ocupando a outbox sem necessidade. |
| **Guardar só o `order_id` e renderizar no envio** | Com retries de até cerca de 15 horas, o evento poderia descrever um estado diferente do que gerou a notificação, o que é inconsistente para o cliente. |
| **Incluir os itens do pedido no payload** | Infla o payload sem necessidade. O detalhe já está disponível em `GET /orders/:id`. |
| **Truncar payloads acima do limite** | Um payload desse tamanho indica erro, e entregar dados cortados é pior do que não entregar. |

## Consequências

**Positivas**
- A outbox só contém trabalho real.
- O evento é imutável e fiel ao momento da mudança, inclusive em replays.
- O payload pequeno reduz o custo de envio e de armazenamento.

**Negativas**
- O filtro passa a rodar dentro da transação de `changeStatus`, com uma consulta aos endpoints do cliente.
- Uma mudança de assinatura (filtro) só vale para eventos futuros. Eventos já gravados seguem como estão.
- Clientes que querem os itens precisam de uma chamada extra à API.

**Trade-off:** pagamos uma consulta a mais na transação de status para ter uma outbox enxuta e eventos imutáveis.

## Referências

- Transcrição: [09:23] Sofia, [09:24] Diego e Larissa, [09:33] Marcos, [09:34] Diego e Bruno, [09:43] Diego, [09:44] Bruno, [09:52] Larissa, Diego e Bruno
- Código: `src/modules/orders/order.routes.ts` (`GET /orders/:id`), `prisma/schema.prisma` (`Order.orderNumber`, `Order.totalCents`)
