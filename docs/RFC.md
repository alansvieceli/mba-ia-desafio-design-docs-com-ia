# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| Autor | Alan Vieceli |
| Status | Proposto para revisão |
| Data | 27/07/2026 |
| Revisores | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenharia de Pedidos), Diego (Plataforma), Sofia (Segurança) |

## TL;DR

Propomos webhooks outbound para notificar clientes B2B quando pedidos mudarem de status. A mudança de status e o snapshot do evento serão persistidos atomicamente em uma outbox no MySQL. Um único worker, executado em processo separado, consultará a outbox a cada 2 segundos e fará entregas HTTP com HMAC-SHA256.

A entrega será at-least-once, com UUID no `X-Event-Id`, cinco retentativas e DLQ após o esgotamento. A solução usa a stack e os padrões atuais do OMS, sem Redis, broker externo ou chamada síncrona no fluxo de pedidos.

## Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo solicitaram notificação em tempo real das mudanças de status. Hoje esses clientes consultam repetidamente `GET /orders`, uma integração lenta e cara. Para eles, latência inferior a 10 segundos atende à expectativa de tempo real.

O OMS não possui eventos, filas ou notificações externas. O fluxo atual de `OrderService.changeStatus` é transacional e pode atualizar pedido, histórico e estoque. A nova capacidade precisa preservar essa consistência e não pode depender da disponibilidade do endpoint do cliente.

## Objetivos da proposta

- Emitir notificações outbound de mudança de status em menos de 10 segundos em condições normais.
- Garantir que toda mudança confirmada e assinada por ao menos um webhook ativo tenha um evento persistido.
- Isolar a operação de pedidos de latência e falhas externas.
- Permitir autenticação do emissor, deduplicação, retentativas, diagnóstico e replay controlado.
- Reutilizar MySQL, Prisma, autenticação, erros, validação e observabilidade existentes.

## Não objetivos

- Webhooks inbound.
- Email como fallback.
- Dashboard visual.
- Broker ou Redis nesta fase.
- Rate limiting de saída nesta fase.
- Escala horizontal do worker ou ordenação com múltiplos consumidores.
- Arquivamento automático de entregas após 30 dias.

## Proposta técnica

### Visão geral

1. Uma mudança válida de status é executada por `OrderService.changeStatus`.
2. Na mesma transação Prisma, o OMS identifica webhooks ativos do cliente que assinam o novo status.
3. Para cada configuração aplicável, persiste na outbox um evento UUID com snapshot JSON de até 64 KB.
4. Após o commit, um worker separado consulta eventos pendentes a cada 2 segundos, em lote pequeno e por ordem de criação.
5. O worker assina os bytes do corpo com HMAC-SHA256 e envia o evento ao endpoint HTTPS.
6. Sucesso registra a entrega. Falha agenda nova tentativa em `1m / 5m / 30m / 2h / 12h`.
7. Após as cinco retentativas, o evento é movido para uma DLQ persistida. Um ADMIN pode solicitar replay manual, com auditoria.

```text
changeStatus
    │
    ├── atualiza pedido, histórico e estoque
    └── grava snapshot na outbox
              │
              ▼
       worker separado
              │
       ┌──────┴──────┐
       │             │
    sucesso        falha
       │             │
   delivery       backoff
   entregue          │
                 5 retries
                     │
                    DLQ
                     │
               replay ADMIN
```

### Semântica de entrega

A garantia é at-least-once. Retentativas do mesmo evento usam o mesmo `event_id`; portanto, o consumidor deve deduplicar por `X-Event-Id`. Exactly-once não faz parte da proposta.

Um único worker preserva a ordem de criação para eventos do mesmo pedido. Essa propriedade não deve ser tratada como garantia global ou assumida após futura escala horizontal.

### Segurança

Cada endpoint possui uma secret própria. A assinatura HMAC-SHA256 é calculada sobre o corpo enviado e transmitida em `X-Signature`. Também são enviados `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.

Somente URLs HTTPS são aceitas. A rotação mantém a secret anterior válida por 24 horas. O timeout da chamada é de 10 segundos e o payload máximo é 64 KB.

### Gestão

Usuários autenticados podem criar, listar, alterar, remover e rotacionar configurações, indicando o cliente e os status de interesse. Também podem consultar as últimas 100 entregas de um webhook.

O replay da DLQ é separado do CRUD comum: exige papel `ADMIN` e gera registro de auditoria.

## Alternativas consideradas

### RFC-ALT-01 — Entrega síncrona em `changeStatus`

Descartada porque a chamada externa manteria o fluxo de pedidos e sua transação dependentes da latência e disponibilidade do cliente. Uma indisponibilidade externa não pode causar rollback da mudança do pedido.

### RFC-ALT-02 — Redis Streams ou broker

Descartado nesta fase porque exige nova infraestrutura para um time pequeno e ainda introduz o problema de atomicidade entre MySQL e o broker. A outbox no banco existente atende ao volume e à meta inicial.

### RFC-ALT-03 — Trigger de banco para acordar o worker

Descartada porque o MySQL não oferece notificação nativa adequada a processos externos; improvisar arquivo ou callback aumentaria a fragilidade.

### RFC-ALT-04 — Exactly-once

Descartada porque exigiria coordenação entre OMS e consumidor. At-least-once com identificador estável oferece uma relação melhor entre confiabilidade e complexidade.

## Questões em aberto

| ID | Questão | Encaminhamento |
| --- | --- | --- |
| RFC-OPEN-01 | Será necessário rate limiting de saída por cliente ou endpoint? | Observar volume e falhas após o lançamento; definir apenas se virar problema. |
| RFC-OPEN-02 | Como preservar a ordem por pedido com múltiplos workers? | Antes de escalar, avaliar particionamento por `order_id` ou lock pessimista. |
| RFC-OPEN-03 | Qual será a política definitiva de retenção e arquivamento? | Arquivamento em torno de 30 dias foi sugerido, mas está fora desta fase. |
| RFC-OPEN-04 | Replay da DLQ preserva o `event_id` original ou cria um novo? | Fechar antes da implementação do replay e documentar para os consumidores. |

## Impacto e riscos

| Risco/impacto | Efeito | Tratamento |
| --- | --- | --- |
| Crescimento da outbox e histórico | Consultas do worker e banco podem degradar | Índices por estado e criação, lotes pequenos, métricas de backlog e futura retenção |
| Endpoint lento ou indisponível | Atraso de entrega e pressão sobre o worker | Timeout de 10s, backoff finito e DLQ |
| Duplicatas | Cliente pode processar a mesma mudança mais de uma vez | `X-Event-Id`, documentação e testes de retentativa |
| Vazamento de secret | Falsificação de eventos de um endpoint | Secret isolada, HTTPS, rotação e redação de logs |
| Worker único indisponível | Backlog cresce enquanto o processo estiver parado | Processo separado, health check, métricas e reinício operacional |
| Falha ao persistir a outbox | Mudança de status não pode ficar sem evento | Rollback da transação completa |

## Implantação e validação

A entrega foi estimada em três sprints, com pelo menos dois dias úteis para revisão de segurança antes do deploy.

A validação deve cobrir atomicidade da outbox, filtro de status, assinatura, timeout, retentativas, DLQ, replay autorizado, duplicatas e regressão do fluxo de pedidos. A ativação deve ser acompanhada por latência, backlog, taxa de sucesso, retries e tamanho da DLQ.

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker separado com polling](adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — Autenticação HMAC e rotação de secret](adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md)
- [ADR-005 — Entrega at-least-once com Event ID](adrs/ADR-005-entrega-at-least-once-com-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)

## Referências

- `TRANSCRICAO.md`: reunião completa, com decisões confirmadas em `[09:48] Larissa` e `[09:49] Diego`.
- `src/modules/orders/order.service.ts`
- `src/middlewares/auth.middleware.ts`
- `src/shared/errors/app-error.ts`
- `src/shared/logger/index.ts`
- `src/config/database.ts`
- `prisma/schema.prisma`
