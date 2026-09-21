# Architecture Decision Record (ADR)

## ADR-005: Worker em Processo Separado com Polling

* **Status:** Decidido
* **Data:** 20 de Agosto de 2026
* **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)

---

### Contexto
O consumo e o disparo das mensagens gravadas na tabela de outbox exigem rotinas de loop constante para buscar pendências, executar cálculos de HMAC, efetuar chamadas HTTP e registrar logs de entrega. Executar esse loop pesado diretamente dentro do mesmo processo Node.js que serve a API REST da nossa aplicação expõe o sistema a diversos riscos, como:
1. Competição de threads e de CPU entre o processamento de rotas de usuários e os disparos externos.
2. Quedas ou reinícios do processo da API devido a vazamento de memória interromperiam abruptamente o envio dos webhooks em andamento.
3. Dificuldade de escalar os recursos de processamento da API de forma isolada do consumo de eventos de background.

### Decisão
Decidimos que o processador de webhooks rodará como um **Worker em processo separado**, totalmente desacoplado da API.
* Criaremos um arquivo de entrada `src/worker.ts` dedicado e um script executável `npm run worker` no `package.json`.
* O worker rodará em um loop de **polling a cada 2 segundos**, executando consultas parametrizadas na tabela `webhook_outbox` para buscar lotes (*batches*) de eventos prontos para envio.
* O processo se conectará à mesma base de dados MySQL existente, porém instanciando um cliente Prisma independente para evitar concorrência de pools de conexão com a API REST.
* Manteremos o worker rodando inicialmente como uma **instância única (single-instance)**, o que garante naturalmente a ordenação implícita das entregas por `created_at` e impede que eventos do mesmo pedido sejam entregues fora de ordem por concorrência de processamento de múltiplos workers paralelos.

### Alternativas Consideradas
* **Execução In-Process (Background loop na API):** Rodar o worker como uma rotina interna em background (via `setInterval` ou `cron` do Node.js) no mesmo processo da API. Descartado devido à falta de isolamento, riscos de degradação de latência das rotas HTTP dos operadores e fragilidade operacional.
* **Triggers de Banco de Dados de Alta Reatividade:** Configurar triggers ou listeners no banco MySQL. Descartado, pois o MySQL não possui suporte nativo satisfatório para sinalizar de forma reativa aplicações fora do banco (como o `LISTEN/NOTIFY` do PostgreSQL), exigindo artifícios inseguros ou leitura em arquivo.

### Consequências
* **Positivas:**
  * **Isolamento de CPU e Memória:** Falhas, lentidões de rede ou picos de carga no envio de webhooks não afetam a disponibilidade da nossa API REST principal.
  * **Facilidade de Deploy:** O worker pode ser implantado e escalado de forma independente no ambiente de containerização (ex: Docker/Kubernetes) usando diferentes perfis de recursos.
  * **Ordenação Nativa Simples:** Manter uma instância única de worker garante que eventos de um mesmo pedido sejam disparados em sequência cronológica lógica.
* **Negativas:**
  * Aumenta a complexidade de deploy, exigindo gerenciar dois processos distintos (API e Worker).
