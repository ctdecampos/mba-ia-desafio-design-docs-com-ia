# Architecture Decision Record (ADR)

## ADR-006: Reuso dos Padrões de Código Existentes

* **Status:** Decidido
* **Data:** 20 de Agosto de 2026
* **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)

---

### Contexto
Nossa codebase atual possui padrões de arquitetura muito bem consolidados para organização de módulos, controle de erros, logs estruturados e validação de requisições. Criar caminhos alternativos, adotar novas bibliotecas de log ou inventar novos padrões de manipulação de exceções para o módulo de webhooks introduziria complexidade cognitiva para os desenvolvedores e aumentaria o custo de manutenção da aplicação.

### Decisão
Decidimos adotar um princípio de **reuso máximo e conformidade estrita** com a arquitetura existente no projeto para a implementação de toda a feature de webhooks.

Mapeamento de Reuso de Padrões da Codebase:
1. **Estrutura Modular (`src/modules/webhooks/`):** O novo domínio seguirá o mesmo padrão de pastas dos outros domínios da aplicação, contendo:
   * `webhooks.controller.ts` (controle de rotas HTTP)
   * `webhooks.service.ts` (regras de negócio)
   * `webhooks.repository.ts` (comunicação com banco)
   * `webhooks.routes.ts` (mapeamento de rotas)
   * `webhooks.schema.ts` (schemas de validação do Zod)
2. **Controle de Erros (`AppError`):** Utilizaremos a classe global `AppError` para todas as exceções de negócio lançadas pelo módulo de webhooks, criando códigos de erro padronizados usando obrigatoriamente o prefixo `WEBHOOK_` (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`). O middleware global de erro de `src/middlewares/errorHandler.ts` já está preparado para capturar `AppError` e continuará funcionando perfeitamente sem alterações.
3. **Logs Estruturados (Logger Pino):** Utilizaremos a mesma instância configurada do Pino Logger para todas as operações do worker e disparo dos webhooks, permitindo agregação simples de logs.
4. **Segurança de Rotas (`requireRole`):** O middleware existente `requireRole` será reaproveitado para blindar a rota de replay manual de DLQ administrativa, exigindo o papel `ADMIN`.
5. **Autenticação e Schemas:** Todas as validações de payloads de entrada das APIs do CRUD utilizarão schemas Zod integrados com nossos pipelines de rotas padrão.

### Alternativas Consideradas
* **Subir um Microserviço Independente em NestJS/Go:** Criar um microserviço dedicado apenas para webhooks. Descartado, pois traria alta barreira de aprendizado e novos custos de infraestrutura no início. Manter como um módulo monolítico, porém com worker isolado de processamento, é a arquitetura ideal para nosso tamanho de time.

### Consequências
* **Positivas:**
  * Curva de aprendizado nula para o time de desenvolvimento; os engenheiros conseguirão codificar de forma natural e rápida.
  * Consistência arquitetural de alto nível em toda a codebase.
  * Sem novas bibliotecas inseridas, mantendo as dependências limpas.
* **Negativas:**
  * O worker herda o peso total das dependências do monolito ao instanciar o backend para execução, porém isso é desprezível no ambiente atual.
