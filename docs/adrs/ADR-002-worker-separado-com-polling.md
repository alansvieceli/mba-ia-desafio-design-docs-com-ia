# ADR-002 — Worker separado com polling

## Status

Aceito

## Contexto

Os eventos persistidos na outbox precisam ser entregues sem depender do ciclo de vida do processo HTTP. O requisito de produto considera tempo real uma entrega em menos de 10 segundos.

O MySQL não oferece um mecanismo equivalente a `LISTEN/NOTIFY` para acordar um processo externo. Executar o consumidor dentro da API também vincularia sua disponibilidade e seus reinícios ao servidor HTTP.

## Decisão

Executar um único worker em processo Node.js separado da API. O processo consultará o mesmo banco por polling a cada 2 segundos, buscará lotes pequenos de eventos pendentes em ordem de criação e os processará.

O worker terá seu próprio `PrismaClient`, apontando para a mesma `DATABASE_URL`, e uma entry point dedicada equivalente a `src/server.ts`.

Enquanto houver apenas um worker, será preservada a ordem de criação para eventos do mesmo pedido. Não há garantia de ordenação global.

## Alternativas consideradas

### Worker dentro do processo da API

Descartado porque reinícios, escalabilidade e falhas da API afetariam o consumidor.

### Trigger no MySQL

Descartado porque triggers executam SQL, mas não notificam de forma adequada um processo externo.

### Vários workers desde a primeira versão

Adiado porque adicionaria coordenação por `order_id`, particionamento ou locking. Os clientes não pediram ordenação global e o volume inicial não justificou essa complexidade.

## Consequências

### Positivas

- O intervalo de 2 segundos é compatível com a meta de menos de 10 segundos.
- API e worker podem reiniciar e operar de forma independente.
- A solução usa a mesma linguagem, banco e stack operacional.
- Um único worker simplifica a ordenação por pedido.

### Negativas

- Polling gera consultas mesmo quando não há eventos.
- Um único worker limita vazão e disponibilidade horizontal.
- A ordem por pedido deixa de ser implícita quando houver escala horizontal.
- O processo precisa de lifecycle, health check e observabilidade próprios.

## Referências

- `TRANSCRICAO.md`: `[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa`, `[09:11] Diego`, `[09:12] Diego` e `[09:13] Larissa`.
- Código: `src/server.ts` e `src/config/database.ts`.

