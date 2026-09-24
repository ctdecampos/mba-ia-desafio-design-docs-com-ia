# Jornada de Desenvolvimento: Design Docs Gerados por IA

Este repositório contém a entrega completa do desafio técnico de modelagem e design para o **Sistema de Webhooks de Notificação de Pedidos** integrado ao Order Management System (OMS) existente da Full Cycle.

---

## 1. Sobre o Desafio
O objetivo deste desafio foi transformar a transcrição literal de uma reunião técnica de engenharia (composta por PM, Tech Lead, Engenheiro de Segurança e Desenvolvedores) em uma documentação técnica completa, detalhada e de nível acionável. A feature projetada preenche a ausência de um sistema de notificações na aplicação, estabelecendo uma solução confiável e escalável baseada no padrão **Transactional Outbox**, garantindo que as mudanças de status dos pedidos de clientes B2B sejam entregues de forma robusta.

O maior foco consistiu em orquestrar diferentes níveis de abstração técnica e de negócio, garantindo que cada documento possuísse uma finalidade clara: o **PRD** com o foco no produto, a **RFC** propondo alternativas de arquitetura de alto nível, os **ADRs** formalizando as decisões específicas, o **FDD** detalhando o "como construir" no nível do código e o **Tracker** amarrando toda a rastreabilidade contra alucinações.

---

## 2. Ferramentas de IA Utilizadas
Durante toda a jornada, operamos sob o papel de maestros de engenharia com a assistência ativa das seguintes ferramentas de Inteligência Artificial:
* **Gemini (Notebook):** Atuou como nosso repositório central de inteligência e base de conhecimento. Foi fundamental para indexar a transcrição da chamada e cruzar as referências do código.
* **Claude-3.5-Sonnet:** Utilizado como motor de raciocínio principal para redigir os documentos de design técnico, gerando diagramas Mermaid e refinando as estruturas JSON das APIs com estrita consistência.
* **Markdown Tools:** Auxiliou na montagem estruturada das tabelas e na consistência das referências cruzadas entre os arquivos de documentação.

---

## 3. Workflow Adotado
Organizamos o desenvolvimento de forma progressiva e estruturada para garantir consistência lógica e semântica de ponta a ponta:

1. **Fase de Mapeamento e Extração:** Fizemos uma varredura completa da transcrição literal (`TRANSCRICAO.md`) identificando todos os requisitos, restrições, nomes de participantes, decisões acordadas, pontos em aberto e descarte de ideias.
2. **Fase de ADRs Primeiro (Esqueleto Técnico):** Geramos primeiro os 6 Architecture Decision Records (ADRs). Como as decisões arquiteturais guiam toda a implementação física e as regras de negócio, ter esses documentos fechados evitou contradições futuras.
3. **Fase da RFC (Proposta Arquitetural):** Compilamos a RFC apresentando a visão unificada da solução técnica, referenciando diretamente os ADRs criados e deixando claro quais as alternativas descartadas e pontos suspensos.
4. **Fase do FDD (Detalhamento Técnico):** Descemos ao nível de baixo nível de código, detalhando payloads, rotas, matriz de erros com prefixo `WEBHOOK_`, tratamento de retries, seções completas de dependências, critérios de aceite, riscos e integração com arquivos reais da codebase (`order.service.ts`, `auth.middleware.ts`, `error.middleware.ts`, `src/shared/logger/index.ts`).
5. **Fase do PRD (Visão de Negócio):** Consolidamos as métricas quantitativas, escopo funcional e os critérios de aceitação de negócio para o PRD, amarrando as pontas com as visões técnicas já desenhadas.
6. **Fase do Tracker de Rastreabilidade:** Mapeamos item por item, garantindo correspondência absoluta das tabelas e gerando o `TRACKER.md` final.
7. **Refatoração do README e Empacotamento:** Finalizamos documentando nosso processo neste README e criando um pacote automático de entrega.

---

## 4. Prompts Customizados

Apresentamos abaixo dois prompts customizados extremamente relevantes que escrevemos para orientar a IA a produzir documentos altamente detalhados e livres de alucinações:

### Prompt 1: Mapeador de Rastreabilidade e Identificação de Contradições
```markdown
Você é um Engenheiro de Qualidade de Software especializado em revisar especificações técnicas contra transcrições literais de reuniões de arquitetura.
Dado o arquivo 'TRANSCRICAO.md' e o código base da aplicação:
1. Extraia todas as decisões fechadas com timestamps e nomes de quem concordou.
2. Identifique quais ideias foram explicitamente descartadas (mínimo 2) e quais pontos foram deixados como 'questões em aberto' (mínimo 2).
3. Verifique se o código existente possui referências reais a: 'src/middlewares/auth.middleware.ts', 'src/middlewares/error.middleware.ts', 'src/modules/orders/order.service.ts', 'src/shared/logger/index.ts' e 'AppError'.
Gere uma tabela estruturada mapeando cada item a ser documentado ao seu respectivo ponto de origem, garantindo que não haja invenção ou suposição.
```

### Prompt 2: Engenharia de Baixo Nível para o FDD (Resiliência, Contratos e Integração)
```markdown
Atue como um Engenheiro de Software Sênior. Quero que você desenhe os contratos HTTP e a lógica do Worker para a feature de Webhooks de acordo com a transcrição técnica.
O FDD deve conter:
- 4 endpoints HTTP completos em formato JSON contendo payload de request, response e headers esperados (incluindo X-Signature, X-Event-Id e X-Timestamp).
- Uma matriz de tratamento de erros detalhada usando a convenção de códigos de erro do projeto, iniciando todos com 'WEBHOOK_'.
- Detalhamento matemático de como o algoritmo de backoff exponencial se comportará após cada falha de envio para as 5 retentativas planejadas (1m, 5m, 30m, 2h, 12h), especificando o tempo total de resiliência.
- A seção obrigatória 'Integração com o sistema existente' referenciando caminhos de arquivos reais (src/middlewares/auth.middleware.ts, src/middlewares/error.middleware.ts, src/shared/logger/index.ts).
- Seções completas de 'Dependências e compatibilidade', 'Critérios de aceite técnicos' e 'Riscos e mitigação'.
```

---

## 5. Iterações e Ajustes
A qualidade e densidade da documentação exigiram ciclos completos de iterações e correções críticas:
* **Ajuste 1 (Níveis de Abstração):** Na primeira geração, a IA incluiu um excesso de diagramas de fluxo de implementação e payloads detalhados dentro da RFC. Solicitamos uma refatoração rigorosa para mover todo o detalhamento de implementação HTTP para o FDD, mantendo a RFC concisa (focada em decisões, alternativas e debates de alto nível).
* **Ajuste 2 (Filtro de Escopo e Alucinações):** A IA sugeriu de forma autônoma a inclusão de um sistema de alertas de webhook por e-mail e uma interface administrativa visual no PRD. Solicitamos a remoção, mantendo o foco estrito no que foi decidido em reunião.
* **Ajuste 3 (Padronização de Erros e Caminhos de Código):** No FDD e no Tracker, alinhamos rigorosamente a matriz de erros ao prefixo corporativo `WEBHOOK_*` e validamos todos os caminhos de arquivo da pasta `src/` com a estrutura real da codebase (`src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`).
* **Ajuste 4 (Validação do Tracker e Seções Faltantes):** Corrigimos o mapeamento de localização do Tracker com indicação exata de timestamps (ex: `[09:18] Diego`) e caminhos reais de código, além de completar no FDD as seções de dependências, critérios de aceite técnicos e matriz de riscos e mitigações.

---

## 6. Como Navegar a Entrega
Para analisar o pacote de documentos completo, sugerimos a leitura na seguinte sequência lógica estruturada:

1. **Visão de Negócio:** [docs/PRD.md](./docs/PRD.md) - Define os requisitos funcionais e não funcionais, escopo e metas.
2. **Proposta Arquitetural:** [docs/RFC.md](./docs/RFC.md) - Apresenta a visão geral da solução técnica e alternativas avaliadas.
3. **Decisões Técnicas:** [docs/adrs/](./docs/adrs/) - Registro isolado e justificado das 6 decisões fundamentais de engenharia.
4. **Detalhamento de Implementação:** [docs/FDD.md](./docs/FDD.md) - Especificação matemática de resiliência, contratos JSON, integração com o código existente, critérios de aceite e riscos.
5. **Rastreabilidade:** [docs/TRACKER.md](./docs/TRACKER.md) - Tabela cruzando a documentação com a transcrição e o código fonte.
