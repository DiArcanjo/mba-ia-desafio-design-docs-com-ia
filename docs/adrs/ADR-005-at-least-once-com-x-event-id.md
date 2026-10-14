# ADR-005: Garantia de entrega at-least-once com X-Event-Id

- **Status:** Aceito
- **Data:** 2026-10-14
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (Produto)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Contexto

Com outbox, worker e retries, existem cenários em que o mesmo evento é enviado mais de uma vez. Por exemplo, o cliente processa a requisição mas a resposta não chega em 10 segundos, então o worker considera falha e retenta. Precisamos decidir qual garantia de entrega oferecer e como o cliente lida com duplicatas.

## Decisão

1. A plataforma garante entrega **at-least-once**. O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado.
2. Cada evento recebe um **`event_id` (UUID)** no momento em que entra na outbox. Ele é enviado no header **`X-Event-Id`** e no campo `event_id` do payload.
3. O `event_id` não muda entre retentativas nem em replays da DLQ. O cliente deduplica do lado dele pelo `event_id`.
4. Produto documenta esse comportamento com destaque no portal do desenvolvedor.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exigiria coordenação dos dois lados (confirmação e estado compartilhado com o cliente), com complexidade muito maior. O at-least-once com identificador resolve a grande maioria dos casos e é o modelo usado por Stripe e GitHub. |

## Consequências

**Positivas**
- O worker fica simples: na dúvida, retenta.
- Nenhum evento é perdido por falha de rede na resposta.
- Modelo conhecido pelos clientes, por ser o mesmo de grandes provedores.

**Negativas**
- A responsabilidade de deduplicar passa para o cliente, ponto levantado pela Segurança.
- Clientes que não deduplicam podem processar o mesmo evento duas vezes.

**Trade-off:** preferimos simplicidade e robustez do nosso lado a uma garantia de exatamente uma vez, e transferimos a deduplicação para o cliente com um identificador estável.

## Referências

- Transcrição: [09:24] Diego, [09:25] Bruno, Diego e Sofia, [09:26] Marcos e Larissa, [09:42] Diego
