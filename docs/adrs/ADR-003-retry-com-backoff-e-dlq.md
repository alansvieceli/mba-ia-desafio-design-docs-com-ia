# ADR-003 — Retry com backoff e DLQ

## Status

Aceito

## Contexto

Endpoints dos clientes podem ficar indisponíveis ou ultrapassar o timeout. Descartar a entrega na primeira falha prejudicaria a confiabilidade, enquanto retentar indefinidamente manteria eventos presos e consumiria recursos sem limite.

Também é necessário preservar evidências de falhas permanentes e permitir recuperação operacional controlada.

## Decisão

Após uma tentativa de entrega malsucedida, reagendar o evento seguindo cinco intervalos de backoff: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.

Depois de esgotar as cinco retentativas, mover o evento para uma tabela `webhook_dead_letter`, preservando ao menos payload, motivo e timestamp da falha. O replay será manual por endpoint administrativo e recolocará uma cópia como pendente na outbox.

O replay exigirá papel `ADMIN` e deverá registrar quem executou a ação.

## Alternativas consideradas

### Retry indefinido

Descartado porque eventos de endpoints abandonados permaneceriam ativos para sempre.

### Três retentativas

Descartado porque não cobre indisponibilidades conhecidas de aproximadamente duas horas.

### Marcar falha permanente na própria outbox

Descartado em favor de uma DLQ separada, que mantém a consulta de pendentes mais simples e preserva evidências para diagnóstico e replay.

## Consequências

### Positivas

- Falhas temporárias têm uma janela de recuperação de aproximadamente 15 horas.
- Falhas permanentes ficam isoladas e auditáveis.
- O replay manual permite recuperação controlada.

### Negativas

- A entrega pode ocorrer muitas horas após o evento original.
- São necessários estado de tentativas, agendamento e tabela adicional.
- O replay pode gerar nova duplicata, compatível com a garantia at-least-once.
- A operação passa a exigir monitoramento da DLQ.

## Referências

- `TRANSCRICAO.md`: `[09:15] Diego`, `[09:16] Diego`, `[09:17] Larissa`, `[09:18] Diego`, `[09:35] Diego` e `[09:36] Sofia`.
- Código: `src/middlewares/auth.middleware.ts`.

