# ADR-001 — Outbox transacional no MySQL

## Status

Aceito

## Contexto

O OMS precisa emitir webhooks quando um pedido muda de status sem aumentar a latência da operação nem permitir que uma mudança confirmada fique sem evento. Hoje, `OrderService.changeStatus` executa a mudança do pedido, o histórico e os ajustes de estoque dentro de uma transação Prisma em `src/modules/orders/order.service.ts`.

Uma chamada HTTP síncrona dentro dessa transação faria a disponibilidade e a latência do cliente externo interferirem na operação do OMS. Publicar em uma infraestrutura separada após o commit, por outro lado, criaria uma janela na qual o status poderia ser confirmado sem que o evento fosse persistido.

## Decisão

Persistir cada evento aplicável em uma tabela de outbox no MySQL, dentro da mesma transação que atualiza `orders` e insere em `order_status_history`.

O evento terá UUID e armazenará um snapshot do payload no momento da mudança de status. A inserção será filtrada pelas configurações ativas e pelos status assinados pelo cliente. Se a persistência na outbox falhar, toda a transação deverá sofrer rollback.

## Alternativas consideradas

### Chamada HTTP síncrona

Descartada porque um endpoint lento ou indisponível bloquearia a mudança de status e poderia manter aberta uma transação que também altera estoque.

### Redis Streams ou broker externo

Descartado nesta fase por adicionar infraestrutura e operação desproporcionais ao tamanho do time. Além disso, seria necessário resolver a atomicidade entre MySQL e broker.

### Publicação assíncrona após o commit

Descartada porque existe uma janela de falha entre confirmar a mudança e publicar o evento.

## Consequências

### Positivas

- A mudança do pedido e o registro do evento são atômicos.
- A entrega externa não afeta a latência da transação de pedidos.
- O sistema reaproveita o MySQL e o Prisma existentes.
- O snapshot preserva o estado exato que originou o evento.

### Negativas

- O MySQL passa a acumular dados operacionais de entrega.
- É necessário indexar e monitorar a outbox para evitar degradação.
- Retenção e arquivamento serão necessários, mas não fazem parte desta fase.
- O acoplamento transacional exige que o enfileiramento aceite o `Prisma.TransactionClient` vigente.

## Referências

- `TRANSCRICAO.md`: `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Larissa`, `[09:08] Larissa`, `[09:40] Bruno`, `[09:41] Diego`, `[09:51] Larissa` e `[09:52] Larissa`.
- Código: `src/modules/orders/order.service.ts` e `prisma/schema.prisma`.
