# Tracker de Rastreabilidade

Este documento realiza o mapeamento cruzado entre cada requisito, decisão técnica, restrição ou trade-off e sua respectiva origem, seja ela na transcrição da reunião técnica (`TRANSCRICAO`) ou na base de código do sistema (`CODIGO`).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| **PRD-RF-01** | `docs/PRD.md` | Requisito Funcional | Cadastro de Webhook (POST `/webhooks`) com URL e eventos. | `TRANSCRICAO` | `[09:31] Marcos` |
| **PRD-RF-02** | `docs/PRD.md` | Requisito Funcional | Edição de Webhook (PATCH `/webhooks/:id`). | `TRANSCRICAO` | `[09:33] Bruno` |
| **PRD-RF-03** | `docs/PRD.md` | Requisito Funcional | Exclusão de Webhook (DELETE `/webhooks/:id`). | `TRANSCRICAO` | `[09:33] Bruno` |
| **PRD-RF-04** | `docs/PRD.md` | Requisito Funcional | Listagem de Webhooks por cliente (GET `/webhooks`). | `TRANSCRICAO` | `[09:33] Bruno` |
| **PRD-RF-05** | `docs/PRD.md` | Requisito Funcional | Histórico de Entregas (GET `/webhooks/:id/deliveries`) com últimos 100 envios. | `TRANSCRICAO` | `[09:34] Marcos` |
| **PRD-RF-06** | `docs/PRD.md` | Requisito Funcional | Filtragem na Outbox durante a inserção do status do pedido. | `TRANSCRICAO` | `[09:33] Marcos` |
| **PRD-RF-07** | `docs/PRD.md` | Requisito Funcional | Rotação de Secret com carência de 24 horas. | `TRANSCRICAO` | `[09:21] Sofia` |
| **PRD-RF-08** | `docs/PRD.md` | Requisito Funcional | Replay manual da DLQ (POST `/admin/webhooks/dead-letter/:id/replay`). | `TRANSCRICAO` | `[09:18] Diego` |
| **PRD-RNF-01** | `docs/PRD.md` | Requisito Não Funcional | Latência ponta a ponta abaixo de 10 segundos. | `TRANSCRICAO` | `[09:02] Marcos` |
| **PRD-RNF-02** | `docs/PRD.md` | Requisito Não Funcional | Garantia At-Least-Once com dedup por `X-Event-Id` no cliente. | `TRANSCRICAO` | `[09:24] Diego` |
| **PRD-RNF-03** | `docs/PRD.md` | Requisito Não Funcional | Protocolo HTTPS obrigatório (validação de URL no schema Zod). | `TRANSCRICAO` | `[09:23] Sofia` |
| **PRD-RNF-04** | `docs/PRD.md` | Requisito Não Funcional | Limite do Payload em 64KB (falha se exceder). | `TRANSCRICAO` | `[09:23] Sofia` |
| **PRD-RNF-05** | `docs/PRD.md` | Requisito Não Funcional | Isolamento de processos para o Worker de webhooks (`src/worker.ts`). | `TRANSCRICAO` | `[09:11] Diego` |
| **PRD-RNF-06** | `docs/PRD.md` | Requisito Não Funcional | Auditoria e log de replays manuais da DLQ. | `TRANSCRICAO` | `[09:35] Diego` |
| **RFC-ALT-01** | `docs/RFC.md` | Alternativa | Envio síncrono no OrderService (Descartada). | `TRANSCRICAO` | `[09:04] Bruno` |
| **RFC-ALT-02** | `docs/RFC.md` | Alternativa | Mensageria via Redis Streams ou RabbitMQ (Descartada). | `TRANSCRICAO` | `[09:07] Diego` |
| **RFC-QA-01** | `docs/RFC.md` | Questão em Aberto | Throttling e Rate Limiting de Saída para clientes. | `TRANSCRICAO` | `[09:38] Diego` |
| **RFC-QA-02** | `docs/RFC.md` | Questão em Aberto | Dashboard visual / Interface de gerenciamento no frontend. | `TRANSCRICAO` | `[09:39] Marcos` |
| **FDD-INT-01** | `docs/FDD.md` | Integração | Inserção transacional no `OrderService.changeStatus` usando Prisma transactional client. | `CODIGO` | `src/modules/orders/order.service.ts` |
| **FDD-INT-02** | `docs/FDD.md` | Integração | Uso de classes herdadas de `AppError` com prefixo `WEBHOOK_`. | `CODIGO` | `src/modules/errors/AppError.ts` |
| **FDD-INT-03** | `docs/FDD.md` | Integração | Restrição de replay ao papel de administrador (`requireRole('ADMIN')`). | `CODIGO` | `src/middlewares/requireRole.ts` |
| **FDD-INT-04** | `docs/FDD.md` | Integração | Captura automática de erros do tipo `AppError` no middleware centralizado. | `CODIGO` | `src/middlewares/errorHandler.ts` |
| **FDD-INT-05** | `docs/FDD.md` | Integração | Registro de logs estruturados de processamento com o Pino Logger. | `CODIGO` | `src/utils/logger.ts` |
| **ADR-001** | `docs/adrs/ADR-001-padrao-outbox-mysql.md` | Decisão | Uso do Padrão Outbox no MySQL. | `TRANSCRICAO` | `[09:06] Diego` |
| **ADR-002** | `docs/adrs/ADR-002-politica-retry-backoff-dlq.md` | Decisão | Política de retry com backoff exponencial (5 tentativas) e DLQ. | `TRANSCRICAO` | `[09:14] Diego` |
| **ADR-003** | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | Assinatura HMAC-SHA256 e secret única rotacionável por endpoint. | `TRANSCRICAO` | `[09:19] Sofia` |
| **ADR-004** | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Garantia de entrega At-Least-Once com dedup no cliente via `X-Event-Id`. | `TRANSCRICAO` | `[09:24] Diego` |
| **ADR-005** | `docs/adrs/ADR-005-worker-polling-processo-separado.md` | Decisão | Worker em processo separado com polling de 2 segundos. | `TRANSCRICAO` | `[09:08] Diego` |
| **ADR-006** | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Decisão | Reuso máximo de padrões de design e utilitários da codebase. | `TRANSCRICAO` | `[09:30] Larissa` |
