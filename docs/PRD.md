# Product Requirements Document (PRD)

## Sistema de Webhooks de Notificação de Pedidos

| Campo | Detalhe |
| --- | --- |
| **Status** | Aprovado |
| **Autor** | Larissa (Tech Lead) / Marcos (PM) |
| **Data** | 20 de Agosto de 2026 |
| **Revisores** | Sofia (Segurança), Bruno (Pedidos), Diego (Plataforma) |

---

### 1. Resumo e Contexto da Feature
O **Sistema de Webhooks de Notificação de Pedidos** é uma nova feature voltada a clientes B2B (como *Atlas Comercial*, *MaxDistribuição* e *Nova Cargo*) que necessitam acompanhar o ciclo de vida dos seus pedidos em tempo real. Atualmente, esses clientes realizam chamadas recorrentes (*polling*) no endpoint `GET /orders`, gerando sobrecarga desnecessária na API e atrasos na ponta do cliente. O sistema proposto enviará notificações HTTP do tipo *outbound* de forma ativa e assíncrona sempre que houver mudanças de status de um pedido.

### 2. Problema e Motivação
* **Sobrecarga de Servidor:** Clientes B2B fazem requisições frequentes para verificar alterações de status, degradando a performance geral.
* **Atrasos de Integração:** O modelo atual de *polling* é ineficiente e lento.
* **Risco de Churn:** Clientes cruciais (ex: *Atlas Comercial*) expressaram risco de migrar para concorrentes caso essa funcionalidade de notificações em tempo real não seja entregue até o fim do trimestre.

### 3. Público-alvo e Cenários de Uso
* **Público-alvo:** Desenvolvedores e equipes de TI de clientes B2B integrados ao nosso Order Management System (OMS).
* **Cenário de Uso:** Um pedido é marcado como faturado (`PAID`) ou enviado (`SHIPPED`) no OMS. O sistema do cliente é notificado em menos de 10 segundos, disparando fluxos automáticos de separação ou logística do lado deles.

### 4. Objetivos e Métricas de Sucesso
* **Objetivo:** Notificar os clientes sobre mudanças de status de forma robusta, escalável e segura.
* **Métricas de Sucesso:**
  * **Latência de Envio:** Enviar pelo menos **99.9%** das notificações de webhook em **menos de 10 segundos** a partir do momento de alteração de status no banco de dados.
  * **Taxa de Sucesso de Entrega:** Alcançar no mínimo **98%** de entregas bem-sucedidas (HTTP status 2xx) na primeira tentativa ou através de retentativas automáticas dentro da janela de retry.

### 5. Escopo
#### Em Escopo:
* CRUD completo de configurações de webhook por cliente.
* Persistência transacional dos eventos usando o padrão **Outbox** no MySQL.
* Worker em processo separado para consumo assíncrono e disparo das requisições HTTP.
* Política de retentativas automáticas com backoff exponencial e Dead Letter Queue (DLQ) para falhas persistentes.
* Assinatura criptográfica dos payloads usando **HMAC-SHA256** com chaves únicas por endpoint.
* Endpoint de rotação de segredo (*secret*) com período de carência (*grace period*) de 24 horas.
* Endpoint administrativo restrito a administradores (`ADMIN`) para reprocessar itens na DLQ.
* Histórico de entregas contendo as últimas 100 tentativas por webhook.

#### Fora de Escopo / Adiado:
* **Envio de e-mail de alerta em caso de falhas consecutivas:** Adiado para fases futuras após medição de uso.
* **Dashboard visual / Painel de controle no Frontend:** A cargo de um projeto futuro e independente da equipe de Frontend. Esta entrega é estritamente de backend e API.
* **Rate limiting de saída para endpoints dos clientes:** Identificado como ponto em aberto para monitoramento inicial antes de implementar regras de throttling.

---

### 6. Requisitos Funcionais (RF)

| ID | Título | Descrição | Origem |
| --- | --- | --- | --- |
| **RF-01** | Cadastro de Webhook | Permite ao cliente cadastrar um endpoint HTTP informando a `url` e uma lista de `status` de interesse. A `secret` é gerada pelo sistema e devolvida na resposta. O `customer_id` deve ser passado no corpo/path, pois o JWT representa o operador. | `TRANSCRICAO` `[09:31] Marcos` |
| **RF-02** | Edição de Webhook | Permite atualizar a URL, lista de eventos de interesse e estado ativo/inativo da configuração. | `TRANSCRICAO` `[09:33] Bruno` |
| **RF-03** | Exclusão de Webhook | Permite excluir a configuração de um webhook pelo seu identificador único. | `TRANSCRICAO` `[09:33] Bruno` |
| **RF-04** | Listagem de Webhooks | Permite listar todas as configurações de webhook associadas a um determinado cliente (`customer_id`). | `TRANSCRICAO` `[09:33] Bruno` |
| **RF-05** | Histórico de Entregas | Endpoint `GET /webhooks/:id/deliveries` para retornar o histórico das últimas 100 tentativas de disparo, contendo status, payload, resposta do cliente e tempo de resposta. | `TRANSCRICAO` `[09:34] Marcos` |
| **RF-06** | Filtragem na Outbox | No momento de alteração de status do pedido, o sistema só insere o evento na outbox se houver um webhook ativo cadastrado para aquele cliente e inscrito para aquele status específico. | `TRANSCRICAO` `[09:33] Marcos` |
| **RF-07** | Rotação de Secret | Permite rotacionar a chave secreta de assinatura. Ao rotacionar, a chave antiga permanece válida por um período de carência de 24 horas para evitar indisponibilidade. | `TRANSCRICAO` `[09:21] Sofia` |
| **RF-08** | Replay Manual da DLQ | Endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay` restrito à role `ADMIN` para re-enfileirar manualmente mensagens falhas na tabela de outbox. | `TRANSCRICAO` `[09:18] Diego` |

---

### 7. Requisitos Não Funcionais (RNF)

| ID | Título | Descrição | Origem |
| --- | --- | --- | --- |
| **RNF-01** | Latência de Notificação | A latência entre a mudança do status no banco de dados e a requisição HTTP deve ser inferior a 10 segundos (SLA de "tempo real"). | `TRANSCRICAO` `[09:02] Marcos` |
| **RNF-02** | Garantia At-Least-Once | Garantia de que toda notificação será entregue ao menos uma vez. Casos de duplicidade devem ser gerenciados pelo cliente através do cabeçalho `X-Event-Id`. | `TRANSCRICAO` `[09:24] Diego` |
| **RNF-03** | Protocolo HTTPS Obrigatório | O endpoint cadastrado pelo cliente deve obrigatoriamente utilizar o protocolo `https`. Requisições para `http` devem falhar na validação do schema Zod. | `TRANSCRICAO` `[09:23] Sofia` |
| **RNF-04** | Limite do Payload | O tamanho máximo do payload gerado não deve ultrapassar 64KB. Se ultrapassar, a tentativa de envio é registrada como erro sem disparo da chamada. | `TRANSCRICAO` `[09:23] Sofia` |
| **RNF-05** | Isolamento de Processos | O Worker de disparo de webhooks deve rodar em processo apartado do servidor principal da API (`src/worker.ts`), evitando impacto na API em caso de reinício. | `TRANSCRICAO` `[09:11] Diego` |
| **RNF-06** | Auditoria e Logs | Toda ação de replay manual na DLQ deve ser auditada e registrada contendo o identificador do administrador responsável. | `TRANSCRICAO` `[09:35] Diego` |

---

### 8. Decisões e Trade-offs Principais
* **Outbox Transacional no MySQL:** Decidido em detrimento de ferramentas de mensageria dedicadas (como Redis Streams ou RabbitMQ) para evitar sobrecarga operacional em um time pequeno. A escrita na tabela de outbox é transacional com a mudança de status do pedido, prevenindo inconsistências.
* **Processamento Single-Worker:** Adotado inicialmente para garantir a ordem lógica de eventos por pedido (`order_id`) sem necessidade de travas complexas ou particionamento.

### 9. Dependências
* **Banco de dados existente (MySQL):** Necessário para criar as tabelas `webhook_configs`, `webhook_outbox` e `webhook_dead_letter`.
* **Prisma ORM:** Para controle transacional e consultas ao banco.

### 10. Riscos e Mitigação
* **Risco 1: Endpoints de Clientes Lentos/Instáveis:** Clientes que demoram para responder podem prender conexões do worker.
  * *Mitigação:* Timeout rígido de 10 segundos por chamada e isolamento do worker em processo separado com pool próprio de conexões Prisma.
* **Risco 2: Vazamento de Secrets de Clientes:** Secrets expostas em logs dos clientes podem comprometer a autenticidade das mensagens.
  * *Mitigação:* Secrets únicas por endpoint (nunca globais) e capacidade de rotação com grace period de 24 horas para migração suave.

### 11. Critérios de Aceitação
* Qualquer transação de mudança de status de pedido (`changeStatus`) deve sofrer rollback completo se a inserção correspondente na tabela de outbox falhar.
* O payload enviado ao cliente deve ser exatamente o snapshot gerado na criação do evento, mantendo a consistência dos dados daquele momento histórico.
* A validação de assinatura pelo cliente usando HMAC-SHA256 com o cabeçalho `X-Signature` deve ser viável com o payload JSON bruto recebido.

### 12. Estratégia de Testes e Validação
* **Testes de Integração:** Validar se a transação do `OrderService.changeStatus` insere corretamente na tabela de outbox sob diferentes cenários de transição de status.
* **Testes de Unidade:** Validar a correta geração e verificação da assinatura HMAC-SHA256 e a rotação de chaves.
* **Testes Ponta a Ponta (E2E):** Subir um mock server HTTP e simular o ciclo completo: mudança de status do pedido -> inserção na outbox -> processamento pelo worker -> envio HTTP -> validação de headers e payload -> tratamento de retries -> envio para DLQ -> replay manual por usuário ADMIN.
