# Architecture Decision Record (ADR)

## ADR-001: Padrão Outbox no MySQL

* **Status:** Decidido
* **Data:** 20 de Agosto de 2026
* **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Sofia (Segurança)

---

### Contexto
A alteração de status de pedidos na nossa aplicação já é uma transação SQL complexa, realizando inserções no histórico e alterações de estoque físico. Adicionar requisições HTTP síncronas de notificação para sistemas externos de terceiros dentro deste mesmo fluxo traria graves problemas de performance e estabilidade, como:
1. Bloqueio de conexões do banco de dados devido a tempos de resposta lentos dos clientes.
2. Risco de causar rollbacks indevidos na venda de pedidos caso o endpoint do cliente esteja temporariamente instável.
3. Impossibilidade de realizar retentativas resilientes de entrega em caso de indisponibilidade parcial do receptor.

Precisamos de uma solução assíncrona que garanta atomicidade: se o status do pedido foi salvo no banco de dados, a intenção de envio do webhook deve obrigatoriamente ser salva de forma inseparável.

### Decisão
Decidimos implementar o padrão **Transactional Outbox** utilizando o banco de dados MySQL existente. Toda alteração de status realizada no método `changeStatus` de `src/modules/orders/order.service.ts` incluirá, dentro do mesmo bloco de transação do Prisma client, a inserção de uma linha na tabela `webhook_outbox` contendo os dados imutáveis do evento serializados na criação.

A tabela de outbox contará com a seguinte estrutura física inicial:
* `id` (UUID - UUID nativo do sistema)
* `webhook_config_id` (UUID - ID da configuração do endpoint)
* `event_type` (String - tipo de evento)
* `payload` (JSON - snapshot serializado dos dados do pedido no momento exato do evento)
* `status` (Enum - PENDING, PROCESSING, RETRY, DELIVERED, FAILED)
* `attempts` (Integer - contador de tentativas)
* `next_attempt_at` (Timestamp - agendamento de retentativa)
* `created_at` / `updated_at` (Timestamps)

### Alternativas Consideradas
* **Chamadas HTTP Síncronas (Inline):** Descartado pois prejudicaria a resiliência e a latência da API.
* **Mensageria Dedicada (Redis Streams ou RabbitMQ):** Exige provisionamento e monitoramento de uma nova infraestrutura de mensageria em produção. Como somos um time pequeno, essa complexidade e custos operacionais foram considerados desnecessários para o volume atual. Além disso, a gravação na fila não compartilharia da mesma atomicidade transacional ácida do MySQL, correndo o risco de falhar após o commit do banco.

### Consequências
* **Positivas:**
  * **Atomicidade Garantida:** Zero inconsistência. Se o pedido mudou, o evento é registrado de forma inquebrável.
  * **Isolamento Completo:** A API de pedidos permanece rápida e independente das falhas de rede dos clientes.
  * **Sem Novas Dependências:** Reuso total do MySQL e Prisma existentes no projeto.
* **Negativas:**
  * Adiciona concorrência de leitura e escrita na base MySQL por conta das frequentes verificações do worker.
  * *Mitigação:* Criação de índices compostos em `status` e `created_at` na tabela `webhook_outbox`, além de limpeza sistemática dos dados antigos de logs após 30 dias.
