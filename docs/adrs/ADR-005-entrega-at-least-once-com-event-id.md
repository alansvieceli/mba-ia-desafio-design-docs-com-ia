# ADR-005 — Entrega at-least-once com Event ID

## Status

Aceito

## Contexto

Em uma entrega HTTP, o worker pode perder a confirmação depois que o cliente já processou o evento. Retentar nesse cenário produz duplicatas. Garantir exactly-once exigiria coordenação e estado transacional nos dois sistemas.

## Decisão

Oferecer garantia de entrega at-least-once. Cada evento receberá um UUID imutável no momento em que entrar na outbox, enviado no corpo como `event_id` e no header `X-Event-Id`.

Todas as tentativas e retentativas do mesmo evento usarão o mesmo identificador. O consumidor será responsável por deduplicar o processamento com base nesse valor.

A reunião não definiu se um replay manual da DLQ preserva o `event_id` original ou cria um novo. Esse comportamento deverá ser fechado antes da implementação do endpoint de replay.

## Alternativas consideradas

### Exactly-once

Descartada por exigir coordenação entre OMS e consumidor e por aumentar muito a complexidade.

### At-most-once

Descartada porque uma falha de rede poderia perder definitivamente uma notificação.

### Novo Event ID a cada tentativa

Descartado porque impediria o consumidor de reconhecer duplicatas do mesmo evento.

## Consequências

### Positivas

- Falhas ambíguas podem ser retentadas sem perder a identidade do evento.
- O protocolo permanece simples e independente da tecnologia do consumidor.
- O UUID segue o padrão de identificadores do projeto.

### Negativas

- O cliente precisa implementar armazenamento e deduplicação.
- Duplicatas continuam possíveis no transporte.
- A semântica do identificador precisa ser documentada de forma inequívoca.

## Referências

- `TRANSCRICAO.md`: `[09:24] Diego` a `[09:26] Larissa` e `[09:51] Larissa`.
- Código: `prisma/schema.prisma`.
