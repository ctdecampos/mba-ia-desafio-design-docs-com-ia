# Architecture Decision Record (ADR)

## ADR-002: Política de Retry com Backoff e DLQ

* **Status:** Decidido
* **Data:** 20 de Agosto de 2026
* **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)

---

### Contexto
Endpoints HTTP fornecidos por clientes B2B estão sujeitos a instabilidades temporárias de rede, janelas de manutenção planejada e problemas gerais de disponibilidade. Para garantir alta confiabilidade na entrega das notificações de pedidos (garantia At-Least-Once), precisamos de uma política estruturada de retentativas automáticas em vez de descartar o evento na primeira falha de comunicação HTTP.

### Decisão
Implementaremos uma política de retentativa automática utilizando **Backoff Exponencial** progressivo contendo **5 tentativas ao total**. Se todas as tentativas falharem, o evento será movido de forma atômica para uma tabela de Dead Letter Queue (DLQ) denominada `webhook_dead_letter` e removido da outbox.

A progressão do backoff após as falhas ocorrerá nos seguintes intervalos:
1. **Tentativa 1:** Reagendar para +1 minuto
2. **Tentativa 2:** Reagendar para +5 minutos
3. **Tentativa 3:** Reagendar para +30 minutos
4. **Tentativa 4:** Reagendar para +2 horas
5. **Tentativa 5:** Reagendar para +12 horas

Isso fornece uma janela de aproximadamente 15 horas entre o primeiro disparo malsucedido e a última tentativa. Se o cliente falhar em todas as tentativas, a mensagem entra em estado inativo na tabela de DLQ.
Disponibilizaremos um endpoint administrativo seguro `POST /admin/webhooks/dead-letter/:id/replay` para permitir que usuários com papel de `ADMIN` possam reenviar manualmente os eventos da DLQ de volta para a outbox principal.

### Alternativas Consideradas
* **Fila de Retry Infinita:** Tentar reenviar o webhook indefinitely até obter sucesso. Isso foi descartado, pois causaria congestionamento na outbox por causa de endpoints de clientes abandonados ou desativados permanentemente, bloqueando recursos do worker.
* **Retry Linear (curto):** Retentar de 3 em 3 minutos por 3 vezes. Descartado porque 3 tentativas cobrem uma janela curta demais (9 minutos). Nossos clientes possuem rotinas de manutenção interna que podem durar de 1 a 2 horas, o que geraria falhas permanentes de forma indevida.

### Consequências
* **Positivas:**
  * Janela de resiliência de 15 horas, cobrindo com folga interrupções noturnas ou manutenções prolongadas dos clientes.
  * Isola registros falhos persistentes na DLQ, mantendo a tabela ativa de outbox limpa.
  * Capacidade de replay manual por administradores após a correção das falhas pelos clientes.
* **Negativas:**
  * Requer a criação e manutenção de uma estrutura de tabela dedicada para a DLQ (`webhook_dead_letter`).
