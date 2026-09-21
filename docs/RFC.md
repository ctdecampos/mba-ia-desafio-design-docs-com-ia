# Request for Comments (RFC)

## Proposta Técnica: Sistema de Webhooks de Notificação de Pedidos

| Metadado | Detalhe |
| --- | --- |
| **Autor** | Diego (Engenheiro Sênior, Plataforma) / Bruno (Engenheiro Pleno, Pedidos) |
| **Status** | Em Revisão |
| **Data de Criação** | 20 de Agosto de 2026 |
| **Revisores Obrigatórios** | Larissa (Tech Lead), Marcos (PM), Sofia (Segurança) |

---

### 1. Resumo Executivo (TL;DR)
Esta RFC propõe a implementação de um sistema robusto de **outbound webhooks** para notificar clientes B2B em tempo real sobre mudanças no status de seus pedidos. Para evitar sobrecarga operacional e infraestrutura extra, utilizaremos o padrão **Transactional Outbox** persistido na nossa base MySQL existente, consumido de forma assíncrona por um **Worker separado** executando em *polling* de 2 segundos. A segurança será garantida através de conexões HTTPS obrigatórias e assinaturas digitais **HMAC-SHA256** com chaves únicas por endpoint cadastrado.

### 2. Contexto e Problema
Hoje, os clientes B2B realizam requisições frequentes para consultar se houve alteração no status de seus pedidos, o que gera consumo excessivo de recursos e lentidão na sincronização. Precisamos de uma solução ativa que empurre essas atualizações aos clientes, mitigando riscos comerciais de *churn* e reduzindo a latência percebida para menos de 10 segundos.

Como a transação de mudança de status de pedido atual (`changeStatus` no `OrderService`) já executa diversas tarefas críticas (atualizações de estoque, histórico, auditoria), qualquer integração síncrona com serviços HTTP externos degradaria gravemente a performance e comprometeria a estabilidade do sistema.

### 3. Proposta Técnica
A solução é dividida em três pilares arquiteturais:

#### A. Transação e Persistência Atômica (Padrão Outbox)
No momento em que o status do pedido é alterado, o serviço de pedidos realiza a persistência e, dentro da mesma transação SQL (usando o cliente Prisma transacional), insere um registro na tabela `webhook_outbox`. O payload enviado é serializado como um snapshot imutável no momento da mudança.
* Detalhado no [ADR-001](./adrs/ADR-001-padrao-outbox-mysql.md).

```
[Mudar Status Pedido] ──> (Abrir Transação SQL)
                                │
                                ├──> Atualizar Tabela 'orders'
                                ├──> Inserir Histórico 'order_status_history'
                                ├──> Decrementar Estoque 'stock_quantity'
                                └──> Inserir na Tabela 'webhook_outbox' (Snapshot)
                                │
                          (Commit da Transação)
```

#### B. Worker Desconectado em Polling
Um processo Node.js apartado da API principal (`src/worker.ts`) realiza consultas periódicas (*polling*) a cada 2 segundos buscando registros pendentes na outbox. Isso isola falhas de rede ou de infraestrutura do cliente, garantindo que a API permaneça rápida e disponível.
* Detalhado no [ADR-005](./adrs/ADR-005-worker-polling-processo-separado.md).

#### C. Política de Resiliência (Retries e DLQ)
Se uma chamada HTTP falhar, o worker executará até 5 retentativas utilizando backoff exponencial com janelas progressivas: **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas**. Se o limite for esgotado, o evento é transferido para a tabela `webhook_dead_letter` (DLQ) para tratamento administrativo e reprocessamento manual via endpoint administrativo.
* Detalhado no [ADR-002](./adrs/ADR-002-politica-retry-backoff-dlq.md).

#### D. Segurança e Integridade
O payload enviado conterá uma assinatura digital criptográfica enviada no cabeçalho `X-Signature`, gerada via HMAC-SHA256 utilizando uma chave secreta exclusiva por webhook. O endpoint do cliente é validado com esquema estrito do Zod para exigir o protocolo HTTPS.
* Detalhado no [ADR-003](./adrs/ADR-003-autenticacao-hmac-sha256.md).

---

### 4. Alternativas Consideradas

#### Alternativa 1: Envio Síncrono no OrderService
* **Descrição:** Disparar a chamada HTTP diretamente de dentro do fluxo de mudança de status no `OrderService`.
* **Prós:** Simplicidade de código, sem tabelas ou workers adicionais.
* **Contras:** Clientes com endpoints lentos travariam a transação do banco de dados, correndo risco de causar bloqueio de conexões e rollbacks indevidos na venda de ingressos/pedidos em caso de queda do cliente.
* **Decisão de Descarte:** Rejeitada por unanimidade pela equipe técnica devido aos altos riscos de performance e resiliência.

#### Alternativa 2: Mensageria via Redis Streams / RabbitMQ
* **Descrição:** Publicar os eventos em uma fila dedicada no Redis Streams ou RabbitMQ diretamente do service de pedidos.
* **Prós:** Latência ultrabaixa e alta reatividade de eventos.
* **Contras:** Exige provisionar, configurar e monitorar um novo cluster de infraestrutura (Redis/RabbitMQ). Como a equipe é pequena, isso geraria uma complexidade operacional desnecessária nesta fase. Além disso, a escrita na fila e a transação no banco não seriam transacionalmente atômicas nativamente (risco de salvar no banco e falhar o envio para a fila).
* **Decisão de Descarte:** Descartada para priorizar simplicidade operacional e robustez transacional via padrão Outbox no MySQL.

---

### 5. Questões em Aberto

#### Ponto 1: Throttling e Rate Limiting de Saída
* **Descrição:** Se um cliente gerar milhares de mudanças de status em rajadas, devemos aplicar um limite de taxa (*rate limiting*) nos disparos HTTP para não derrubar o endpoint do cliente?
* **Encaminhamento:** Não será implementado nesta primeira fase. Monitoraremos o volume inicial e o comportamento do worker para projetar uma política de vazão controlada na fase 2.

#### Ponto 2: Dashboard Visual para o Cliente
* **Descrição:** Disponibilizar uma interface administrativa dentro do portal do cliente para gerenciar webhooks, visualizar histórico e rodar testes de ping.
* **Encaminhamento:** Adiado para discussão futura com a equipe de Frontend. A API fornecerá todos os endpoints de backend funcionais (`GET /webhooks`, `GET /webhooks/:id/deliveries`), permitindo a integração visual assim que o projeto de frontend for priorizado.

---

### 6. Impacto e Riscos
* **Impacto Operacional:** Baixo, pois utiliza a mesma infraestrutura de banco de dados MySQL e stack Node.js/TypeScript.
* **Risco de Performance:** O polling de 2 segundos adicionará consultas leves frequentes ao banco de dados MySQL.
  * *Mitigação:* Criação de índices específicos nas colunas `status` e `created_at` na tabela `webhook_outbox`.
* **Risco de Sobrecarga de Rede:** Chamadas para endpoints de terceiros.
  * *Mitigação:* Timeout rígido de 10 segundos configurado no cliente HTTP do worker.

---

### 7. Decisões Relacionadas (Links para ADRs)
* [ADR-001: Padrão Outbox no MySQL](./adrs/ADR-001-padrao-outbox-mysql.md)
* [ADR-002: Política de Retry com Backoff e DLQ](./adrs/ADR-002-politica-retry-backoff-dlq.md)
* [ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](./adrs/ADR-003-autenticacao-hmac-sha256.md)
* [ADR-004: Garantia At-Least-Once com X-Event-Id](./adrs/ADR-004-garantia-at-least-once.md)
* [ADR-005: Worker em Processo Separado com Polling](./adrs/ADR-005-worker-polling-processo-separado.md)
* [ADR-006: Reuso dos Padrões de Código Existentes](./adrs/ADR-006-reuso-padroes-existentes.md)
