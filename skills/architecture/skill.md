---
name: architecture
description: Analisa e projeta arquiteturas de software, selecionando referências especializadas conforme o problema, contexto, tecnologias, riscos e características arquiteturais envolvidas.
---

# Architecture

## Objetivo

Fornecer um processo estruturado para projetar novos sistemas, analisar arquiteturas existentes e apoiar decisões arquiteturais durante o desenvolvimento.

A análise deve utilizar apenas as referências relevantes para o problema investigado.

Não aplique patterns ou recomendações de maneira mecânica.

---

# 1. Descoberta do Contexto

Antes de qualquer análise arquitetural relevante:

1. compreenda o problema apresentado;
2. identifique requisitos e restrições conhecidas;
3. quando houver acesso ao projeto, inspecione sua implementação;
4. identifique tecnologias e frameworks utilizados;
5. identifique as dimensões arquiteturais envolvidas;
6. selecione somente as references necessárias.

Não presuma características do projeto que possam ser verificadas diretamente.

---

# 2. Identificação de Tecnologias e Frameworks

Quando houver acesso ao projeto, examine arquivos e configurações para identificar as tecnologias utilizadas.

## Frameworks conhecidos

| Evidências                                       | Framework     | Reference                      |
| ------------------------------------------------ | ------------- | ------------------------------ |
| `Gemfile`, `config/routes.rb`, `bin/rails`       | Ruby on Rails | `references/fw-rails.md`       |
| `manage.py`, `settings.py`, dependência Django   | Django        | `references/fw-django.md`      |
| `pom.xml`/`build.gradle` com Spring Boot         | Spring Boot   | `references/fw-spring-boot.md` |
| `gradle.properties` com Liferay ou `service.xml` | Liferay       | `references/fw-liferay.md`     |
| `go.mod` com `github.com/gin-gonic/gin`          | Gin           | `references/fw-gin.md`         |

A ausência de uma reference específica não impede a análise.

Nesse caso, utilize os princípios gerais disponíveis e deixe explícitas eventuais limitações.

## Compatibilidade arquitetural

Prefira mecanismos e convenções idiomáticas do framework quando eles satisfizerem os requisitos.

Evite introduzir abstrações que apenas reproduzam capacidades já fornecidas pela tecnologia.

Quando houver tensão entre uma recomendação arquitetural e a abordagem idiomática do framework:

1. identifique as alternativas;
2. explique seus benefícios;
3. explique suas limitações;
4. considere o contexto;
5. justifique a escolha.

---

# 3. Catálogo de References

## Arquitetura

| Reference                      | Dimensão                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| `clean-architecture.md`        | estrutura, boundaries, SOLID e componentes                   |
| `ddd.md`                       | domínio, strategic design e ubiquitous language              |
| `implementing-ddd.md`          | aggregates, aplicação prática de DDD, CQRS e Event Sourcing  |
| `microservices.md`             | decomposição, comunicação, sagas e independent deployability |
| `enterprise-patterns.md`       | domain logic, persistência e enterprise patterns             |
| `production-readiness.md`      | estabilidade, capacidade, deploy e operação                  |
| `data-intensive.md`            | consistência, replicação, particionamento e processamento    |
| `arch-tradeoffs.md`            | trade-offs, fitness functions e decisões arquiteturais       |
| `code-complexity.md`           | complexidade cognitiva, módulos e interfaces                 |
| `monolith-vs-microservices.md` | monólitos, modular monolith e microsserviços                 |

---

# 4. Seleção de References

Selecione as references de acordo com as dimensões envolvidas.

| Categoria                 | References principais                                               | Complementares                        |
| ------------------------- | ------------------------------------------------------------------- | ------------------------------------- |
| Revisão estrutural        | `code-complexity`, `clean-architecture`                             | `ddd`                                 |
| Modelagem de domínio      | `ddd`, `implementing-ddd`                                           | `enterprise-patterns`                 |
| Dados e persistência      | `data-intensive`, `enterprise-patterns`                             | `ddd`                                 |
| Boundaries e decomposição | `arch-tradeoffs`, `monolith-vs-microservices`, `ddd`                | `clean-architecture`, `microservices` |
| Resiliência e produção    | `production-readiness`, `data-intensive`                            | `microservices`                       |
| Evolução arquitetural     | `arch-tradeoffs`, `monolith-vs-microservices`, `clean-architecture` | conforme contexto                     |
| APIs e integração         | `microservices`, `enterprise-patterns`, `production-readiness`      | `arch-tradeoffs`                      |
| Trade-offs                | `arch-tradeoffs`                                                    | conforme dimensões analisadas         |

A matriz orienta a seleção. Ela não substitui julgamento técnico.

Não carregue todas as references apenas porque estão disponíveis.

---

# 5. Indicadores para aprofundamento

Durante a inspeção, utilize sinais encontrados no projeto para decidir quais dimensões precisam de investigação adicional.

| Indício                                         | References                                                     |
| ----------------------------------------------- | -------------------------------------------------------------- |
| Organização exclusivamente por camadas técnicas | `clean-architecture`                                           |
| Modelo anêmico                                  | `ddd`, `code-complexity`                                       |
| Aggregates excessivamente grandes               | `implementing-ddd`                                             |
| Banco utilizado como fila                       | `data-intensive`                                               |
| Integrações sem timeout                         | `production-readiness`                                         |
| Banco compartilhado entre serviços              | `microservices`, `data-intensive`                              |
| Decisão sem trade-offs explícitos               | `arch-tradeoffs`                                               |
| Interfaces excessivamente complexas             | `code-complexity`                                              |
| Domínio fortemente acoplado ao schema           | `enterprise-patterns`, `data-intensive`                        |
| Discussão monólito × microsserviços             | `monolith-vs-microservices`, `arch-tradeoffs`, `microservices` |
| Modular monolith sem enforcement de boundaries  | `monolith-vs-microservices`, `clean-architecture`              |
| Deploy coordenado entre serviços                | `monolith-vs-microservices`, `microservices`                   |
| Infraestrutura desproporcional ao problema      | `monolith-vs-microservices`, `arch-tradeoffs`                  |

Esses indicadores são pontos de investigação, não diagnósticos automáticos.

---

# 6. Arbitragem entre Perspectives

Quando references apresentarem abordagens diferentes:

1. identifique claramente a divergência;
2. determine quais requisitos cada abordagem privilegia;
3. avalie os custos;
4. avalie os benefícios;
5. considere reversibilidade;
6. considere complexidade operacional;
7. considere experiência da equipe;
8. considere características do domínio;
9. escolha de acordo com o contexto.

Não trate nenhuma escola arquitetural como universalmente correta.

## Critérios gerais

### Complexidade estrutural

A complexidade da solução deve acompanhar a complexidade do problema.

Para operações simples, considere estruturas diretas.

Para domínios complexos, considere Domain Model, boundaries explícitos e isolamento das regras de negócio.

### Persistência

Não introduza Repository ou abstrações semelhantes apenas por convenção.

Escolha Active Record, Data Mapper, Repository ou outras abordagens conforme complexidade, acoplamento, testabilidade e necessidades de evolução.

### Monólito e microsserviços

Não considere microsserviços como arquitetura padrão.

Avalie:

- autonomia de implantação;
- estabilidade dos boundaries;
- observabilidade;
- custo operacional;
- maturidade da equipe;
- necessidade real de distribuição.

Um modular monolith pode ser uma arquitetura completa quando atende adequadamente aos requisitos.

### Event Sourcing

Não utilize Event Sourcing como mecanismo padrão de persistência.

Considere-o quando requisitos como histórico completo, reconstrução temporal ou rastreabilidade justificarem sua complexidade.

Avalie alternativas mais simples antes da adoção.

### Validação

Distribua validações conforme sua finalidade:

- fronteiras: formato e estrutura;
- domínio: invariantes;
- aplicação: coordenação e pré-condições do caso de uso.

Evite duplicar a mesma regra sem justificativa.

---

# 7. Protocolo — Projetar Sistema Novo

## 7.1 Entender contexto

Consulte principalmente:

- `arch-tradeoffs`;
- `ddd`.

Identifique:

- regras de negócio;
- architecture characteristics;
- estrutura organizacional;
- restrições;
- volume de dados;
- padrões de acesso.

## 7.2 Definir boundaries

Consulte conforme necessário:

- `ddd`;
- `implementing-ddd`;
- `arch-tradeoffs`;
- `microservices`.

Analise:

- Bounded Contexts;
- Aggregate boundaries;
- Context Map;
- Core Domain;
- Architecture Quantum;
- deployment model.

## 7.3 Definir arquitetura interna

Considere:

- Domain Logic Pattern;
- Building Blocks;
- Dependency Rule;
- Data Source Pattern;
- persistência;
- complexidade das interfaces.

## 7.4 Definir comunicação

Avalie:

- contratos;
- comunicação síncrona × assíncrona;
- request-response × eventos;
- workflows cross-context;
- integração com legado.

## 7.5 Projetar para produção

Avalie quando aplicável:

- timeouts;
- circuit breakers;
- bulkheads;
- retries;
- graceful degradation;
- observabilidade;
- health checks;
- consistência;
- durabilidade.

## 7.6 Validar

Valide:

- architecture characteristics;
- trade-offs;
- isolamento do domínio;
- dependency direction;
- deployment model;
- decisões que precisam de ADR.

---

# 8. Protocolo — Sistema Existente

## 8.1 Inspecionar

Antes de recomendar refatorações:

- examine estrutura;
- dependências;
- domínio;
- persistência;
- integrações;
- deployment;
- comportamento conhecido em produção.

## 8.2 Diagnosticar

Para cada problema:

1. apresente evidência;
2. diferencie fato de hipótese;
3. explique impacto;
4. avalie severidade;
5. explique causa provável quando houver evidência suficiente;
6. apresente alternativas.

## 8.3 Priorizar

Priorize conforme impacto real.

Considere especialmente:

1. falhas críticas de confiabilidade;
2. integridade e perda de dados;
3. segurança;
4. boundaries que impedem evolução segura;
5. problemas de domínio;
6. decomposição;
7. complexidade local.

Essa ordem não é absoluta.

Incidentes, compliance, segurança ou dependências entre alterações podem mudar a prioridade.

## 8.4 Evoluir incrementalmente

Considere estratégias como:

- Strangler Fig;
- Bubble Context;
- Expand and Contract;
- migrações incrementais.

Evite Big Bang Rewrite como estratégia padrão.

Preserve comportamento quando possível e estabeleça:

- critérios de aceitação;
- validação;
- rollback;
- rastreabilidade.

---

# 9. Protocolo — Mentoria durante Desenvolvimento

Durante a implementação, avalie somente as dimensões relevantes.

## Estrutura

Verifique:

- direção das dependências;
- ciclos;
- boundaries;
- enforcement;
- organização do projeto.

## Domínio

Quando houver domínio significativo, verifique:

- Entities;
- Value Objects;
- Aggregates;
- invariantes;
- Ubiquitous Language;
- Application Services.

## Dados

Quando pertinente, verifique:

- padrões de acesso;
- consistência;
- paginação;
- transações;
- concorrência;
- volume.

## Resiliência

Para dependências externas, considere:

- timeouts;
- retries;
- circuit breakers;
- isolamento;
- idempotência;
- comportamento durante indisponibilidade.

## Complexidade

Questione:

- interfaces excessivamente complexas;
- information leakage;
- abstrações sem finalidade;
- complexidade não justificada.

## Trade-offs

Verifique:

- custos;
- benefícios;
- limitações;
- reversibilidade;
- fitness functions quando relevantes.

---

# 10. Calibração de Complexidade

Utilize a complexidade do problema para calibrar a solução.

| Cenário                    | Abordagem inicial                                               |
| -------------------------- | --------------------------------------------------------------- |
| CRUD simples               | estrutura direta / Transaction Script                           |
| CRUD com regras moderadas  | Service Layer enxuta e objetos de domínio quando necessários    |
| Domínio complexo           | Domain Model e boundaries explícitos                            |
| Múltiplos Bounded Contexts | Context Map e isolamento                                        |
| Sistema distribuído        | resiliência, consistência e boundaries explícitos               |
| Escala elevada             | avaliar particionamento e outras estratégias conforme evidência |

Não transforme essa tabela em regra automática.

## Princípio

A arquitetura deve oferecer proteção proporcional aos riscos e complexidade proporcional às necessidades.

Evite tanto overengineering quanto simplificações que comprometam requisitos reais.

---

# 11. Resultado da Análise

Não utilize um template grande automaticamente.

Adapte a resposta à solicitação.

Para decisões arquiteturais, normalmente apresente:

1. contexto;
2. evidências;
3. alternativas;
4. trade-offs;
5. recomendação;
6. consequências;
7. pontos que precisam de validação.

Para análise de sistema existente:

1. diagnóstico;
2. evidências;
3. severidade quando pertinente;
4. prioridades;
5. plano incremental;
6. riscos;
7. critérios de validação.

Para mentoria:

1. problema observado;
2. princípio envolvido;
3. impacto;
4. alternativas;
5. orientação.

Use estruturas maiores somente quando a complexidade da solicitação justificar.
