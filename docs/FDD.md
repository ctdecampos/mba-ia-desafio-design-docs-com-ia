# Feature Design Document (FDD)

## Detalhamento de Implementação: Sistema de Webhooks de Notificação de Pedidos

---

### 1. Contexto e Motivação Técnica
Este documento detalha o desenho técnico de baixo nível para a implementação da feature de outbound webhooks. A solução baseia-se no reaproveitamento da stack existente (Node.js, TypeScript, MySQL, Prisma) para prover entrega confiável e assíncrona de eventos de pedidos para clientes integrados.

### 2. Objetivos Técnicos
* Implementar o padrão transacional Outbox no banco de dados MySQL atual.
* Garantir isolamento de execução criando um processo de worker desacoplado.
* Definir contratos de API claros e robustos para gerenciamento dos webhooks.

### 3. Escopo e Exclusões
* **Incluso:** Criação das tabelas de configuração, outbox e dead letter; implementação do worker; endpoints de CRUD; endpoint de replay administrativo; assinatura HMAC-SHA256; histórico de envios.
* **Exclusões:** Envio de alertas de e-mail; painéis administrativos visuais; rate limiting de saída ativo.

---

### 4. Fluxos Detalhados

#### Fluxo 1: Transação e Registro de Evento (Outbox)
1. O controlador de pedidos chama o `OrderService.changeStatus`.
2. O `OrderService` abre uma transação Prisma.
3. Altera o status do pedido, atualiza estoques e insere logs históricos.
4. Verifica se o cliente do pedido possui um webhook ativo cadastrado para o novo status.
5. Se possuir, gera uma carga JSON enxuta e insere na tabela `webhook_outbox` como `PENDING`.
6. Finaliza e commita a transação SQL.

```
[Ordem Status Alterado] -> [OrderService.changeStatus]
                                │ (Inicia Transação Prisma Client)
                                ├──> Atualiza status na tabela 'Order'
                                ├──> Grava histórico na tabela 'OrderStatusHistory'
                                ├──> Decrementa estoque na tabela 'Product'
                                ├──> Consulta 'WebhookConfig' ativa do Customer
                                │       └── Se ativa + inscrita para o status:
                                │             Gerar payload snapshot e salvar 'WebhookOutbox' como PENDING
                                └── (Commit Transação)
```

#### Fluxo 2: Processamento pelo Worker, Retry e DLQ
1. O worker (`src/worker.ts`) acorda a cada 2 segundos.
2. Busca um lote (*batch*) de eventos `PENDING` ou `RETRY` (com tempo de processamento agendado menor ou igual ao horário atual) na tabela `webhook_outbox`.
3. Para cada evento, transiciona o status na outbox para `PROCESSING` com bloqueio pessimista ou controle otimista.
4. Gera a assinatura HMAC-SHA256 do payload usando a segredo exclusiva do webhook do cliente.
5. Dispara a requisição HTTP POST para a URL cadastrada com timeout de 10 segundos.
6. **Cenário Sucesso (HTTP status 2xx):** Marca o evento na outbox como `DELIVERED`, cria uma linha na tabela de logs de entrega (`webhook_deliveries`) e atualiza o histórico.
7. **Cenário Falha (HTTP status não-2xx ou Timeout/Erro de Rede):**
   * Incrementa o contador de tentativas (`attempts`).
   * Se `attempts` < 5: Atualiza o status para `RETRY` e calcula o próximo horário de disparo (`next_attempt_at`) baseado no backoff exponencial. Grava o log da falha na tabela de entregas.
   * Se `attempts` >= 5: Marca como `FAILED`, remove o registro da outbox e transfere o evento para a tabela de DLQ (`webhook_dead_letter`) registrando o motivo do erro.

---

### 5. Contratos Públicos (API Endpoints)

#### Endpoint 1: Cadastro de Webhook
* **Rota:** `POST /webhooks`
* **Autenticação:** JWT do Operador (o `customer_id` deve ser informado no corpo da requisição)
* **Request Payload (JSON):**
```json
{
  "customer_id": "893c5240-62e5-4d04-897d-411a78dc124e",
  "url": "https://api.atlascomercial.com/webhooks/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"]
}
```
* **Response (HTTP 201 Created):**
```json
{
  "id": "11ea15a9-34ba-46e3-a61b-9f4a6efc17bb",
  "customer_id": "893c5240-62e5-4d04-897d-411a78dc124e",
  "url": "https://api.atlascomercial.com/webhooks/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "whsec_b89d81d2f6027fe09322e7d7041a9cb9eb9d59e3bc839d436a5c1f513903ee41",
  "active": true,
  "created_at": "2026-08-20T16:19:00.000Z"
}
```

#### Endpoint 2: Listagem de Webhooks de um Customer
* **Rota:** `GET /webhooks?customer_id=893c5240-62e5-4d04-897d-411a78dc124e`
* **Autenticação:** JWT Autenticado
* **Response (HTTP 200 OK):**
```json
[
  {
    "id": "11ea15a9-34ba-46e3-a61b-9f4a6efc17bb",
    "customer_id": "893c5240-62e5-4d04-897d-411a78dc124e",
    "url": "https://api.atlascomercial.com/webhooks/orders",
    "events": ["PAID", "SHIPPED", "DELIVERED"],
    "active": true,
    "created_at": "2026-08-20T16:19:00.000Z"
  }
]
```

#### Endpoint 3: Histórico de Entregas por Webhook
* **Rota:** `GET /webhooks/:id/deliveries`
* **Autenticação:** JWT Autenticado
* **Response (HTTP 200 OK):**
```json
[
  {
    "id": "e2ba3472-ee1d-403d-bfca-4512c98a0d42",
    "webhook_id": "11ea15a9-34ba-46e3-a61b-9f4a6efc17bb",
    "event_id": "55c2f38d-0da4-44b2-a42e-be25da8934df",
    "event_type": "order.status_changed",
    "status_code": 200,
    "response_body": "{\"received\": true}",
    "execution_time_ms": 124,
    "success": true,
    "attempt_number": 1,
    "created_at": "2026-08-20T16:21:02.000Z"
  }
]
```

#### Endpoint 4: Replay Manual da DLQ (Admin)
* **Rota:** `POST /admin/webhooks/dead-letter/:id/replay`
* **Autenticação:** JWT Obrigatório, Role `ADMIN` (via `requireRole('ADMIN')`)
* **Response (HTTP 200 OK):**
```json
{
  "success": true,
  "message": "Evento re-enfileirado com sucesso para a outbox principal.",
  "event_id": "55c2f38d-0da4-44b2-a42e-be25da8934df",
  "replayed_by_user_id": "77bc54e9-9a2d-45db-9ee9-4bc78a2e4d9c"
}
```

---

### 6. Matriz de Erros Previstos

O módulo de webhooks deve lançar exceções mapeadas no padrão de erros do projeto (`AppError`), utilizando o prefixo `WEBHOOK_` e o código HTTP apropriado:

| Código de Erro | Status HTTP | Mensagem | Causa |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Webhook configuration not found. | O ID de configuração de webhook fornecido não existe no banco. |
| `WEBHOOK_INVALID_URL` | 400 | Invalid webhook URL. Must use HTTPS protocol. | A URL cadastrada não possui protocolo https ou é inválida. |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Webhook secret is required for signing. | Tentativa de processar assinatura HMAC sem uma secret válida configurada. |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | Dead Letter Queue event not found. | Tentativa de realizar replay de um ID de DLQ inexistente. |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 413 | Webhook payload size exceeds 64KB limit. | O payload gerado ultrapassou o limite físico de segurança de 64KB. |

---

### 7. Estratégias de Resiliência
* **Timeouts:** Timeout HTTP de 10 segundos para cada chamada do worker para evitar que conexões lentas de terceiros travem o pool de threads do processo.
* **Retries & Exponential Backoff:** 5 tentativas ao total. Se falhar, reagenda para retentativa em:
  * Tentativa 1: +1 minuto
  * Tentativa 2: +5 minutos
  * Tentativa 3: +30 minutos
  * Tentativa 4: +2 horas
  * Tentativa 5: +12 horas
* **Dead Letter Queue (DLQ):** Após a 5ª tentativa mal-sucedida, o registro é movido de forma atômica para a tabela `webhook_dead_letter` e removido da outbox.

### 8. Observabilidade
* **Métricas:** 
  * Contador de disparos enviados: `webhooks_dispatched_total` (labels: `status_code`, `success`, `customer_id`).
  * Histograma de latência de disparo HTTP: `webhooks_dispatch_duration_seconds`.
  * Contador de eventos na DLQ: `webhooks_dead_letter_total`.
* **Logs (reutilizando Pino):**
  * Info: `[WebhookWorker] Buscando lote de eventos pendentes. Lote: N itens encontrados.`
  * Error: `[WebhookWorker] Falha no disparo HTTP para o webhook: ID. Status: 500. Retentando em X minutos. Erro: [erro].`
  * Warn: `[WebhookWorker] Limite de tentativas esgotado. Movendo evento EVENT_ID para a DLQ.`
* **Tracing:** Registro do ID do evento (`X-Event-Id`) no cabeçalho HTTP e nos metadados de trace para correlação completa ponta a ponta.

---

### 9. Integração com o Sistema Existente

O módulo de webhooks se integra perfeitamente com os seguintes caminhos físicos da codebase:

1. **`src/modules/orders/order.service.ts`:**
   O método `changeStatus` executa a transição de estado da ordem em uma transação do Prisma. Devemos estender esse método adicionando uma chamada para a função utilitária do webhook:
   ```typescript
   // Dentro do OrderService.changeStatus
   await this.prisma.$transaction(async (tx) => {
     // ... lógica existente de histórico, atualização de estoque ...
     
     // Nova chamada integrando o webhook na mesma transação
     await publishWebhookEvent(tx, orderId, fromStatus, toStatus);
   });
   ```
2. **`src/errors/AppError.ts` (ou similar):**
   Utilizado para lançar exceções estruturadas. Reutilizaremos as classes herdadas de `AppError` para padronizar nossos retornos HTTP e códigos de erro com prefixo `WEBHOOK_`.
3. **`src/middlewares/requireRole.ts`:**
   O middleware de validação de papéis de usuários do JWT será acoplado diretamente à rota de replay de mensagens mortas na DLQ para restringir o acesso unicamente a usuários administradores:
   ```typescript
   router.post('/admin/webhooks/dead-letter/:id/replay', requireRole('ADMIN'), webhookController.replay);
   ```
4. **`src/middlewares/errorHandler.ts`:**
   O middleware de tratamento de exceções global da aplicação capturará de forma automática erros do tipo `AppError` gerados pelo módulo de webhooks e também erros de validação do Zod, injetando segurança e respostas estruturadas.
5. **`src/logger.ts` (Pino):**
   A instância existente do logger Pino será importada no `webhook.worker.ts` para capturar todos os estágios de processamento dos eventos da outbox de forma estruturada.
