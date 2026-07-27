# Design Docs com IA — Sistema de Webhooks

## Sobre o desafio

Este repositório contém um pacote de design docs para adicionar webhooks outbound de mudanças de status a um Order Management System. O ponto de partida foi uma transcrição de reunião técnica e uma aplicação Node.js + TypeScript já funcional. A entrega transforma essas fontes em requisitos, proposta arquitetural, decisões isoladas e uma especificação acionável, sem alterar o código da aplicação.

O principal cuidado foi manter rastreabilidade: requisito, decisão e restrição precisam apontar para um timestamp da reunião ou para um arquivo real do código. Alternativas descartadas e assuntos adiados foram tratados separadamente para não virarem requisitos por engano.

## Ferramentas de IA utilizadas

- **OpenAI Codex:** leitura do repositório e da transcrição, extração de evidências, estruturação dos documentos, revisão cruzada e automação das validações.
- **Prompts customizados e plano de execução:** usados para separar os níveis de PRD, RFC, ADR e FDD e exigir fonte antes de aceitar afirmações normativas.

Ferramentas auxiliares, não generativas:

- **ripgrep (`rg`):** busca de timestamps, decisões, caminhos, endpoints e códigos de erro.
- **Prettier:** normalização e validação da formatação Markdown.
- **Git:** branch de documentação e commits separados por fase.

## Workflow adotado

O trabalho foi organizado de forma que decisões mais estáveis sustentassem os documentos posteriores:

1. Leitura integral da transcrição e inspeção dos fluxos reais de pedidos, autenticação, erros, logs, Prisma e testes.
2. Classificação das evidências em requisito, decisão, restrição, alternativa, questão em aberto e integração.
3. Criação dos seis ADRs que formalizam as decisões arquiteturais.
4. Produção do RFC, consolidando proposta, alternativas e questões abertas.
5. Produção do FDD, detalhando fluxos, contratos, erros, resiliência, observabilidade e integração.
6. Produção do PRD, mantendo foco em problema, público, escopo, requisitos e métricas.
7. Atualização contínua do tracker; ele não foi deixado para o final.
8. Registro deste README somente depois que o processo real já podia ser descrito.
9. Auditoria automatizada e manual contra cada critério de aceite.

Os documentos foram produzidos em quatro ciclos: extração, estrutura, consistência e aceite. Cada fase relevante recebeu um commit próprio para preservar a evolução do trabalho.

## Prompts customizados

### Extração de evidências

```text
Leia integralmente TRANSCRICAO.md e os módulos existentes do OMS.
Extraia somente fatos verificáveis e classifique cada item como requisito,
decisão, restrição, alternativa descartada, questão em aberto ou integração.

Para itens da reunião, informe [hh:mm] e falante. Para itens do código,
informe um caminho que exista. Não transforme hipótese, sugestão adiada
ou alternativa descartada em decisão. Aponte contradições e lacunas.
```

### Separação entre documentos

```text
Produza os documentos sem duplicar suas responsabilidades:

- PRD: problema, público, escopo, métricas e critérios de produto;
- RFC: proposta arquitetural, alternativas, riscos e questões abertas;
- ADR: uma decisão isolada, seu contexto e suas consequências;
- FDD: fluxos, contratos, erros, integração e testes acionáveis.

Antes de incluir uma afirmação normativa, associe-a à transcrição ou
ao código. Se a fonte não fechar uma decisão, registre questão em aberto.
```

### Auditoria de aceite

```text
Audite o pacote contra o enunciado item por item. Conte requisitos,
endpoints, ADRs e fontes do tracker; valide timestamps, links relativos
e caminhos do código; procure códigos de erro fora do padrão WEBHOOK_*.

Não considere concluído enquanto houver contradição, caminho inexistente,
item sem fonte ou seção obrigatória ausente.
```

## Iterações e ajustes

### 1. Semântica do Event ID no replay

Uma primeira versão inferiu que o replay da DLQ preservaria o `event_id`. A reunião define Event ID estável nas retentativas comuns, mas não fecha o comportamento do replay manual. A inferência foi removida e o tema passou a constar como questão aberta no ADR, RFC, FDD e PRD.

### 2. Caminho futuro tratado como arquivo existente

A especificação inicialmente citou a sugestão `src/worker.ts` como se o arquivo já existisse. A auditoria de caminhos detectou que ele é apenas uma entry point proposta. A documentação foi revisada para separar claramente arquivos reais de componentes a criar.

### 3. Fronteira entre RFC e FDD

A proposta inicial acumulava detalhes de contrato no RFC. Na revisão estrutural, payloads, status codes, matriz de erros e passos operacionais foram mantidos no FDD; o RFC ficou restrito à abordagem, aos trade-offs e às questões abertas.

### 4. Tracker construído cedo

O tracker começou junto com os ADRs. Isso expôs rapidamente trechos sem fonte direta e evitou que decisões plausíveis, mas não confirmadas, fossem repetidas nos demais documentos. Ao final, ele foi consolidado e medido automaticamente.

Foram quatro iterações principais até a revisão final: extração, estrutura, consistência e aceite.

## Como navegar a entrega

Ordem de leitura sugerida:

1. [PRD — requisitos de produto](docs/PRD.md): problema, público, escopo e sucesso.
2. [RFC — proposta técnica](docs/RFC.md): arquitetura, alternativas e questões abertas.
3. [ADRs — decisões arquiteturais](docs/adrs/): justificativa de cada decisão.
4. [FDD — desenho de implementação](docs/FDD.md): fluxos, contratos, erros e integração.
5. [Tracker — rastreabilidade](docs/TRACKER.md): origem de cada item relevante.

### ADRs

- [ADR-001 — Outbox transacional no MySQL](docs/adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker separado com polling](docs/adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — Autenticação HMAC e rotação de secret](docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md)
- [ADR-005 — Entrega at-least-once com Event ID](docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)

## Estrutura

```text
.
├── README.md
├── TRANSCRICAO.md
└── docs/
    ├── PRD.md
    ├── RFC.md
    ├── FDD.md
    ├── TRACKER.md
    └── adrs/
        ├── ADR-001-outbox-transacional-no-mysql.md
        ├── ADR-002-worker-separado-com-polling.md
        ├── ADR-003-retry-com-backoff-e-dlq.md
        ├── ADR-004-autenticacao-hmac-e-rotacao-de-secret.md
        ├── ADR-005-entrega-at-least-once-com-event-id.md
        └── ADR-006-reuso-dos-padroes-do-projeto.md
```

## Limites da entrega

Esta entrega é documental. `src/`, `prisma/`, `tests/` e configurações foram usados como fontes de contexto e não foram alterados.
