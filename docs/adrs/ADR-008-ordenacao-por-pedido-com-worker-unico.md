# ADR-008: Ordenação garantida apenas por pedido, com worker único

- **Status:** Aceito
- **Data:** 2026-10-14
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (Produto)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md)

## Contexto

Um pedido pode passar por `PAID`, `PROCESSING` e `SHIPPED` em sequência rápida (as transições permitidas estão em `src/modules/orders/order.status.ts`). Precisávamos decidir que garantia de ordem oferecer ao cliente e como isso se relaciona com o número de workers.

## Decisão

1. Operar com **um único worker**, que processa a outbox em ordem de `created_at`.
2. A ordem de entrega é garantida **apenas por `order_id` e enquanto houver um único worker**. Não há garantia de ordem global entre pedidos diferentes.
3. Para que a garantia valha também com retries, o worker só envia um evento quando não há evento anterior do mesmo pedido, para o mesmo endpoint, ainda pendente ou em processamento. O detalhe está no [FDD, seção 5.2](../FDD.md#52-processamento-pelo-worker-fdd-fluxo-02).
4. Registrar isso como **limitação conhecida**. Escalar para vários workers (particionando por `order_id` ou com lock pessimista) fica para o futuro.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Vários workers em paralelo desde já** | Perderia a garantia de ordem por pedido sem um mecanismo de particionamento ou lock, o que aumenta a complexidade antes de existir necessidade de volume. |
| **Garantia de ordem global** | Os clientes nunca pediram. Eles só querem saber se cada pedido mudou. |

## Consequências

**Positivas**
- Sem lógica de lock ou particionamento na primeira versão.
- O cliente recebe as mudanças de um mesmo pedido na ordem em que aconteceram, inclusive quando há retries.

**Negativas**
- A vazão de entrega fica limitada a um worker.
- Um evento em retry segura os seguintes do mesmo pedido para aquele endpoint. Se o `PAID` estiver em backoff, o `SHIPPED` espera, por até cerca de 15h no pior caso, até o `PAID` ser entregue ou ir para a DLQ.
- Quando um evento vai para a DLQ, os seguintes do mesmo pedido são liberados. Se ele sofrer replay depois, chega **depois** deles, e o cliente reconcilia pelo `timestamp` e por `from_status`/`to_status` do payload.
- Escalar exigirá revisitar esta decisão.

**Trade-off:** abrimos mão de escala horizontal imediata, e aceitamos que um evento com problema atrase os seguintes do mesmo pedido, para manter a ordem por pedido sem lock nem particionamento.

## Referências

- Transcrição: [09:12] Larissa e Diego, [09:13] Bruno, Diego e Larissa, [09:14] Marcos
- Código: `src/modules/orders/order.status.ts` (transições permitidas)
