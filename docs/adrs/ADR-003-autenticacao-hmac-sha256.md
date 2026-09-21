# Architecture Decision Record (ADR)

## ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

* **Status:** Decidido
* **Data:** 20 de Agosto de 2026
* **Decisores:** Larissa (Tech Lead), Sofia (Segurança), Bruno (Pedidos)

---

### Contexto
Disparar notificações contendo informações sensíveis de pedidos (identificadores, status, valores financeiros) para URLs fora da nossa infraestrutura privada introduz riscos críticos de segurança. O receptor precisa de uma forma segura e eficiente para validar que:
1. A requisição HTTP realmente se originou nos nossos servidores legítimos (autenticidade).
2. O corpo da mensagem (*payload*) não sofreu alterações no caminho (integridade).
3. Terceiros maliciosos não consigam forjar requisições fakes para simular faturamentos ou envios de pedidos.

### Decisão
Decidimos que todas as requisições HTTP de webhook incluirão obrigatoriamente uma assinatura digital computada no cabeçalho `X-Signature`. A assinatura será baseada em **HMAC-SHA256** calculada sobre o corpo da requisição (*raw payload* JSON recebido) utilizando uma chave secreta (*secret*) exclusiva do endpoint cadastrado do cliente.

Regras de Segurança Adicionais:
* **Secrets Únicas:** Cada endpoint cadastrado possuirá sua própria secret criptográfica forte (gerada na criação com prefixo `whsec_`), impedindo que o vazamento da chave de um cliente comprometa outros.
* **Rotação Segura:** Disponibilizaremos um endpoint para rotação da secret de um webhook. Ao rotacionar, implementaremos um período de carência (*grace period*) de 24 horas onde ambas as chaves (a antiga e a nova) serão válidas em paralelo, permitindo migração de sistemas do cliente sem indisponibilidade.
* **HTTPS Obrigatório:** A URL configurada deve obrigatoriamente usar protocolo `https`. A validação ocorrerá em nível de entrada usando esquemas de dados estritos do Zod.

### Alternativas Consideradas
* **Segredo Global da Plataforma:** Usar uma única secret global idêntica para todos os clientes cadastrados. Descartado pois viola o princípio do menor privilégio; se um cliente vazar sua chave acidentalmente em logs, todos os outros ficariam vulneráveis.
* **Autenticação via Cabeçalho Token Simples (Static Bearer Token):** Mandar um token estático na requisição (tipo `Authorization: Bearer xyz`). Descartado porque tokens estáticos são facilmente interceptados em trânsito se houver falha de proxy e não garantem a integridade do corpo do payload (um invasor no meio poderia alterar o status do pedido).

### Consequências
* **Positivas:**
  * Segurança robusta e em conformidade com as melhores práticas de mercado (Stripe, GitHub, Shopify).
  * Garantia matemática de integridade do payload e autenticidade de origem.
  * Rotação segura de chaves sem fricção operacional ou interrupções.
* **Negativas:**
  * Adiciona a necessidade do cliente de implementar uma rotina simples de cálculo de HMAC (SHA-256) na ponta receptora.
  * A rotação de chaves com período de carência de 24 horas exige controle no banco para armazenar a secret atual e a anterior temporariamente com seu timestamp de expiração.
