# Architecture Decision Record (ADR)

## ADR-006: Reuso dos Padroes de Codigo Existentes

* **Status:** Decidido
* **Data:** 20 de Agosto de 2026
* **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)

---

### Contexto
A codebase atual do OMS possui padrões consolidados para erros (`src/shared/errors/app-error.ts`), autenticação (`src/middlewares/auth.middleware.ts`), tratamento global de erros (`src/middlewares/error.middleware.ts`) e logs (`src/shared/logger/index.ts`). Criar utilitários duplicados ou um framework isolado para o módulo de webhooks traria inconsistência e dívida técnica.

### Decisão
Adoção de **reuso estrito e alinhamento total** com a estrutura e padrões existentes na codebase:
1. **Controle de Erros (`AppError`):** Reuso da classe base de erros em `src/shared/errors/app-error.ts` com o prefixo corporativo `WEBHOOK_`.
2. **Middleware de Autenticação e Autorização (`auth.middleware.ts`):** Proteção da rota administrativa de replay de DLQ via `src/middlewares/auth.middleware.ts`.
3. **Middleware de Tratamento Global de Erros (`error.middleware.ts`):** Captura automática de `AppError` via `src/middlewares/error.middleware.ts`.
4. **Logger Pino (`src/shared/logger/index.ts`):** Emissão de logs estruturados via `src/shared/logger/index.ts`.
5. **Prisma Client e Transações (`OrderService`):** Acoplamento da gravação do evento outbox no método `changeStatus` em `src/modules/orders/order.service.ts`.

### Alternativas Consideradas
* **Alternativa 1: Criar abstrações customizadas e um framework próprio de erros e logging para o módulo de webhooks**
  * *Descrição:* Desenvolver classes e utilitários isolados (ex: `WebhookError`, `WebhookLogger`, middlewares dedicados) exclusivos para a feature de webhooks.
  * *Trade-offs:* Proporcionaria um isolamento estrito do módulo em relação ao restante do monólito. Contudo, aumentaria a complexidade cognitiva do projeto, geraria duplicação de lógica e criaria múltiplos pipelines de tratamento de erro e logs, violando os princípios de simplicidade e manutenibilidade do time.
* **Alternativa 2: Extração para um microserviço/serviço independente**
  * *Descrição:* Criar um novo serviço isolado em outra aplicação/linguagem com seu próprio ecossistema de erros e logs.
  * *Trade-offs:* Desacoplamento total de infraestrutura, mas traria alto custo operacional, necessidade de novas esteiras de CI/CD e complexidade de rede desnecessária para o escopo e tamanho atual do time.

### Consequências
* **Positivas:** Consistência perfeita da codebase, curva de aprendizado nula para os desenvolvedores e facilidade de manutenção.
* **Negativas:** Dependência acoplada às abstrações globais existentes da aplicação.
