# ADR-006 — Reuso dos padrões do projeto

## Status

Aceito

## Contexto

O OMS organiza cada domínio em controller, service, repository, routes e schemas. Já possui autenticação JWT, autorização por papel, validação Zod, erros estruturados, middleware centralizado, logger Pino e Prisma.

Introduzir padrões paralelos aumentaria a curva de aprendizado, a superfície operacional e o risco de respostas inconsistentes.

## Decisão

Implementar webhooks como um módulo em `src/modules/webhooks`, seguindo a separação já usada pelos demais domínios.

Reutilizar:

- `AppError` e as classes HTTP de `src/shared/errors/`;
- códigos específicos com prefixo `WEBHOOK_`;
- `errorMiddleware` para respostas de erro;
- Zod e `validate` para contratos de entrada;
- `authenticate` no CRUD e `requireRole('ADMIN')` no replay da DLQ;
- Pino para logs estruturados;
- Prisma e a `DATABASE_URL` existente, com um `PrismaClient` próprio no processo worker;
- a composição de controllers e rotas de `src/app.ts` e `src/routes/index.ts`;
- o estilo de testes de integração com Vitest e Supertest.

## Alternativas consideradas

### Framework ou serviço separado

Descartado porque duplicaria autenticação, observabilidade, validação, deploy e padrões sem benefício demonstrado nesta fase.

### Erros e logger exclusivos do módulo

Descartados porque produziriam respostas e telemetria inconsistentes com o restante da aplicação.

## Consequências

### Positivas

- Menor custo de implementação e manutenção.
- Contratos de erro e autenticação consistentes com a API.
- Testes e composição seguem padrões conhecidos pelo time.
- Menos dependências e infraestrutura novas.

### Negativas

- O módulo herda limitações da arquitetura e do middleware atuais.
- `ValidationError` usa hoje o código genérico `VALIDATION_ERROR`; erros específicos de webhook precisarão de classes adequadas para manter o prefixo acordado.
- O worker precisa inicializar suas próprias instâncias por ser outro processo, ainda que reutilize os mesmos módulos.

## Referências

- `TRANSCRICAO.md`: `[09:27] Bruno` a `[09:30] Larissa` e `[09:36] Larissa`.
- Código: `src/app.ts`, `src/routes/index.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/errors/app-error.ts`, `src/shared/logger/index.ts`, `src/config/database.ts` e `tests/orders.test.ts`.

