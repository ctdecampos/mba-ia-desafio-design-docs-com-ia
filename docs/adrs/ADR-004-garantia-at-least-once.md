# Architecture Decision Record (ADR)

## ADR-004: Garantia At-Least-Once com X-Event-Id

* **Status:** Decidido
* **Data:** 20 de Agosto de 2026
* **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)

---

### Contexto
Em redes não confiáveis, falhas de conexão podem ocorrer imediatamente após o receptor processar com sucesso a requisição, mas antes do nosso worker receber a resposta HTTP 200 de confirmação. Sob essa condição de falha, o worker assumirá que a entrega falhou e tentará reenviar a mensagem. Para desenhar um sistema de webhooks de alta confiabilidade, precisamos definir o nível de entrega que asseguramos.

### Decisão
Decidimos que o sistema operará sob o nível de garantia de entrega **At-Least-Once (Pelo menos uma vez)**. Isso assegura que toda notificação será entregue com sucesso, mas abre a possibilidade de que o mesmo evento de webhook seja disparado mais de uma vez em cenários limítrofes de instabilidade.

Para mitigar problemas causados por duplicidade, todas as requisições de webhook conterão obrigatoriamente o cabeçalho HTTP `X-Event-Id` contendo um UUID único gerado no momento da inserção do evento na outbox. Cabe ao sistema receptor do cliente registrar os IDs de eventos processados e usá-los para realizar **desduplicação (*deduplication*)** em sua própria infraestrutura, agindo de forma idempotente.

A desduplicação será amplamente documentada nos guias do desenvolvedor da nossa plataforma, alertando os clientes a tratarem as mensagens de forma resiliente.

### Alternativas Consideradas
* **Exactly-Once (Exatamente uma vez):** Garantir que o evento seja entregue uma única e exclusiva vez sob quaisquer condições de falha. Descartado pois é tecnicamente inviável sem um protocolo de coordenação de transação distribuída (como 2-Phase Commit) altamente custoso, lento e impraticável para integrações HTTP públicas com sistemas de terceiros.

### Consequências
* **Positivas:**
  * Arquitetura viável e de alta confiabilidade, alinhada com os padrões dominantes da indústria.
  * Garante que nenhuma alteração de status de pedido seja "perdida" por falha silenciosa de envio.
* **Negativas:**
  * Joga a responsabilidade de implementar o controle de idempotência para o cliente integrado.
  * *Mitigação:* Documentação detalhada fornecendo exemplos práticos em pseudocódigo e em linguagens comuns sobre como armazenar e desduplicar chaves `X-Event-Id` em Redis ou bancos de dados locais.
