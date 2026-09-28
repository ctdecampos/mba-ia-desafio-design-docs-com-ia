# Architecture Decision Record (ADR)

## ADR-006: Reuso dos Padroes de Codigo Existentes

* **Status:** Decidido
* **Data:** 20 de Agosto de 2026
* **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)

---

### Contexto
A codebase atual do OMS possui padrões consolidados para erros, autenticação, logs e validações. Criar utilitários duplicados para o módulo de webhooks traria inconsistência e dívida técnica.

### Decisão
Adoção de **reuso estrito e alinhamento total** com a estrutura existente:
1. **Controle de Erros (`AppError`):** Reuso da classe base de erros em `src/shared/errors/app-error.ts` com o prefixo corporativo `WEBHOOK_`.
2. **Middleware de Autenticação (`auth.middleware.ts`):** Proteção da rota administrativa via `src/middlewares/auth.middleware.ts`.
3. **Middleware de Tratamento Global de Erros (`error.middleware.ts`):** Captura automática de `AppError` via `src/middlewares/error.middleware.ts`.
4. **Logger Pino (`src/shared/logger/index.ts`):** Emissão de logs estruturados via `src/shared/logger/index.ts`.
5. **Prisma Client e Transações (`OrderService`):** Acoplamento do evento outbox no método `changeStatus` em `src/modules/orders/order.service.ts`.

### Consequências
* **Positivas:** Consistência perfeita da codebase e facilidade de manutenção.
* **Negativas:** Dependência acoplada às abstrações globais existentes.
