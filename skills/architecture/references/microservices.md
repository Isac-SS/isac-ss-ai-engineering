# Skill: Building Microservices — Sam Newman (2nd Edition)

## Perspectiva do Autor

Sam Newman e um pragmatico. Ele nao vende microservices como bala de prata — pelo contrario,
dedica capitulos inteiros a explicar quando NAO usar microservices. Sua visao central e que
**independent deployability** e o unico principio que realmente importa: se voce nao consegue
fazer deploy de um servico sem coordenar com outros times, voce nao tem microservices — voce
tem um monolito distribuido.

Newman valoriza:
- **Incrementalismo**: migrar aos poucos, nunca reescrever do zero ("Big Bang Rewrite" e anti-pattern).
- **Modelagem por dominio**: Bounded Contexts do DDD sao a melhor heuristica para definir fronteiras de servico.
- **Autonomia de times**: a arquitetura deve refletir a estrutura organizacional desejada (Inverse Conway Maneuver).
- **Acoplamento minimo e coesao maxima**: codigo que muda junto deve morar junto.
- **Observabilidade sobre adivinhacao**: se voce nao consegue ver o que acontece em producao, nao consegue operar microservices.
- **Resiliencia como propriedade emergente**: nao basta um servico ser robusto — o sistema precisa se adaptar a falhas inesperadas.

Newman se apoia fortemente em David Parnas (information hiding, 1972), Melvin Conway (Conway's Law, 1968)
e Eric Evans (Domain-Driven Design, 2003). Sua abordagem e bottom-up: comece pelo monolito,
identifique costuras (seams), extraia servicos quando houver razao concreta (escala independente,
deploy independente, isolamento de falha, autonomia de time).

---

## Principios Fundamentais

### 1. Independent Deployability — O Principio Supremo

Independent deployability significa que voce pode fazer deploy de uma alteracao em um servico
sem precisar fazer deploy de nenhum outro servico. Se isso nao e possivel, os demais principios
perdem valor.

Para atingir independent deployability:
- Cada servico deve possuir seu proprio pipeline de CI/CD.
- Contratos de API devem ser versionados e retrocompativeis.
- Dados devem ser encapsulados — nenhum servico acessa o banco de outro diretamente.
- Testes de contrato devem substituir testes end-to-end como gate de deploy.

### 2. Information Hiding (Parnas, 1972)

O modulo deve esconder suas decisoes de design e expor apenas o minimo necessario via interface.
Em microservices, isso se traduz em:
- O schema do banco e interno ao servico — nunca compartilhado.
- Estruturas de dados internas nao vazam para a API publica.
- Mudancas internas nao devem forcar mudancas em consumidores.

**Implicacao pratica**: se dois servicos compartilham um banco de dados, voce violou information hiding.
O banco e um detalhe de implementacao.

### 3. Coesao: Codigo que Muda Junto Mora Junto

A heuristica de coesao de Newman e temporal: se duas funcionalidades sempre mudam juntas quando
um requisito de negocio muda, elas devem estar no mesmo servico. Se mudam independentemente,
podem estar em servicos separados.

Isso se alinha ao Single Responsibility Principle, mas aplicado no nivel de servico:
"Gather together those things that change for the same reason, and separate those things that
change for different reasons."

### 4. Conway's Law e Inverse Conway Maneuver

> "Organizations which design systems are constrained to produce designs which are copies
> of the communication structures of these organizations." — Melvin Conway, 1968

Newman argumenta que, em vez de lutar contra Conway's Law, devemos usa-la a nosso favor:
**estruture os times da forma como voce quer que a arquitetura seja**. Se voce quer tres
servicos independentes, precisa de tres times independentes com ownership claro.

Se a estrutura organizacional nao suporta a arquitetura desejada, mude a organizacao primeiro.
Caso contrario, a arquitetura vai convergir para a estrutura de comunicacao existente.

### 5. Bounded Context como Fronteira de Servico

O Bounded Context de Eric Evans e a melhor heuristica para definir onde cortar servicos:
- Dentro de um Bounded Context, um termo tem significado preciso e consistente.
- Entre Bounded Contexts, o mesmo termo pode ter significados diferentes (ex: "Cliente" para Vendas vs. Suporte).
- A fronteira do Bounded Context e a fronteira natural do servico.

Cada Bounded Context deve ter:
- Seu proprio modelo de dominio (entidades, value objects, aggregates).
- Seu proprio banco de dados (ou schema isolado).
- Sua propria linguagem ubiqua.

### 6. Aggregate como Unidade Interna

Dentro de um servico, o Aggregate (DDD) e a unidade de consistencia:
- Transacoes devem respeitar os limites do Aggregate.
- Um Aggregate e carregado e salvo como unidade atomica.
- Referencia entre Aggregates deve ser por ID, nao por referencia direta.

Um servico pode conter multiplos Aggregates, mas um Aggregate nunca deve ser dividido entre servicos.

### 7. Tipos de Monolito

Newman identifica tres tipos:

| Tipo | Descricao | Risco |
|------|-----------|-------|
| **Single-process monolith** | Todo o codigo em um unico processo deployavel | Baixo — e o ponto de partida natural |
| **Modular monolith** | Monolito com modulos bem definidos e fronteiras claras | Baixo — muitas vezes e a melhor opcao |
| **Distributed monolith** | Multiplos servicos que precisam ser deployados juntos | ALTO — tem todas as desvantagens de microservices E do monolito |

O distributed monolith e o pior dos mundos: complexidade operacional de microservices sem
nenhum dos beneficios. Newman considera esse o anti-pattern mais comum em migracoes mal feitas.

### 8. Tipos de Acoplamento

Newman classifica acoplamento em ordem crescente de perigo:

| Tipo | Descricao | Gravidade |
|------|-----------|-----------|
| **Domain coupling** | Servico A chama Servico B para uma capability de dominio | Normal — inevitavel e aceitavel |
| **Pass-through coupling** | Servico A passa dados para B apenas para que B repasse a C | Medio — indica que A deveria falar direto com C |
| **Common coupling** | Multiplos servicos leem/escrevem no mesmo recurso compartilhado (ex: banco, arquivo) | Alto — viola information hiding |
| **Content coupling (pathological)** | Servico A acessa diretamente os internals de B (ex: banco de B, memoria de B) | Critico — destruicao total de encapsulamento |

### 9. Comunicacao entre Servicos

#### Sincrona (Request-Response)
- REST, gRPC, GraphQL.
- Simples de implementar, mas cria acoplamento temporal (ambos devem estar de pe).
- Cascading failure: se C cai, B que depende de C tambem cai, e A que depende de B tambem.

#### Assincrona (Event-Driven)
- Message brokers (Kafka, RabbitMQ, SQS).
- Desacopla temporalmente: produtor e consumidor nao precisam estar de pe ao mesmo tempo.
- Aumenta complexidade de debug e observabilidade.
- Garante melhor resiliencia e escalabilidade independente.

#### API Gateway vs Service Mesh
- **API Gateway**: ponto de entrada para clientes externos. Responsavel por autenticacao, rate limiting, roteamento.
- **Service Mesh**: infraestrutura para comunicacao service-to-service. Responsavel por mTLS, retries, circuit breaking, observabilidade. Exemplos: Istio, Linkerd.

Nao confunda os dois: API Gateway e para trafego norte-sul (externo para interno),
Service Mesh e para trafego leste-oeste (interno para interno).

### 10. Evolucao de API

Tres estrategias para evoluir APIs sem quebrar consumidores:

**Semantic Versioning**: Major.Minor.Patch. Major = breaking change. Permite multiplas versoes simultaneas, mas aumenta custo operacional.

**Expand and Contract** (tambem chamado Parallel Change):
1. Adicione o novo campo/endpoint (expand).
2. Migre consumidores para o novo campo/endpoint.
3. Remova o campo/endpoint antigo (contract).
Nunca remove antes de todos os consumidores migrarem.

**Consumer-Driven Contracts**: cada consumidor define um contrato do que espera do provider.
O provider roda os testes de contrato de todos os consumidores em seu pipeline.
Ferramentas: Pact, Spring Cloud Contract.

### 11. DRY Across Boundaries e Perigoso

> "Don't violate DRY within a microservice, but be relaxed about violating DRY across services."

Compartilhar bibliotecas de dominio entre servicos cria acoplamento. Se a biblioteca muda,
todos os servicos que a usam precisam ser atualizados e redeployados — violando independent deployability.

**Aceite duplicacao entre servicos**. E o preco da independencia.

**Client libraries** devem ser thin e focadas apenas em transporte (serializacao, HTTP, retry).
Nunca devem conter logica de dominio.

### 12. Sagas: Transacoes Distribuidas sem ACID

Em microservices, nao ha transacao distribuida ACID viavel (two-phase commit nao escala e cria
acoplamento temporal forte). Sagas sao a alternativa:

**Orchestration Saga**: um coordenador central (orchestrator) dirige a sequencia de chamadas.
- Vantagem: fluxo explicito, facil de entender.
- Desvantagem: o orchestrator pode virar um God Service.

**Choreography Saga**: cada servico reage a eventos e emite novos eventos.
- Vantagem: desacoplamento total, sem ponto central de falha.
- Desvantagem: fluxo implicito, dificil de rastrear, debugging complexo.

**Compensating Transactions**: quando um passo falha, os passos anteriores devem ser "desfeitos"
via acoes compensatorias (ex: se cobranca falhou, cancele a reserva). Nem toda acao e
perfeitamente reversivel — o design deve prever isso.

### 13. Resiliencia: Quatro Conceitos

Newman adota o framework de David Woods (resilience engineering):

| Conceito | Descricao |
|----------|-----------|
| **Robustness** | Capacidade de absorver perturbacoes conhecidas sem mudar |
| **Rebound** | Capacidade de se recuperar apos uma falha |
| **Graceful extensibility** | Capacidade de lidar com situacoes alem do design original |
| **Sustained adaptability** | Capacidade de continuar se adaptando ao longo do tempo |

Robustness e o minimo (timeouts, retries). Sustained adaptability e o ideal (cultura de
aprendizado, chaos engineering, game days).

### 14. Stability Patterns

| Pattern | Descricao | Quando Usar |
|---------|-----------|-------------|
| **Timeout** | Limite de tempo para chamadas externas | Sempre — toda chamada externa deve ter timeout |
| **Retry** | Tentar novamente apos falha transiente | Apenas para falhas transientes, com backoff exponencial |
| **Circuit Breaker** | Parar de chamar servico que esta falhando | Quando falhas sao recorrentes e voce quer fail-fast |
| **Bulkhead** | Isolar recursos para que falha em um nao afete outros | Quando compartilhar thread pools/connection pools e arriscado |
| **Idempotency** | Garantir que operacao repetida tem o mesmo efeito | Em toda operacao de escrita que pode ser retried |

### 15. CAP Theorem — Implicacoes Praticas

Em um sistema distribuido, durante uma particao de rede, voce deve escolher entre:
- **Consistency**: todos os nos veem os mesmos dados ao mesmo tempo.
- **Availability**: toda requisicao recebe resposta (mesmo que desatualizada).

Na pratica, particoes de rede ACONTECEM. A escolha real e entre CP e AP:
- **CP** (ex: banco relacional com replicacao sincrona): durante particao, o sistema recusa escrita para manter consistencia.
- **AP** (ex: Cassandra, DynamoDB): durante particao, o sistema aceita escrita e resolve conflitos depois (eventual consistency).

Newman recomenda: entenda qual e o custo de negocio de dados inconsistentes vs. indisponibilidade,
e decida caso a caso.

### 16. Migracoes: Patterns de Transicao

**Strangler Fig Pattern**: crie o novo sistema ao redor do antigo. Redirecione trafego gradualmente
do sistema legado para o novo. O legado "morre" por inanicao.
- Use um proxy/router para direcionar trafego.
- Migre uma rota/funcionalidade por vez.
- Rollback e simples: redirecione de volta.

**Parallel Run**: execute o sistema novo e o legado em paralelo com o mesmo input.
Compare outputs. So corte para o novo quando os resultados forem equivalentes.
- Util quando corretude e critica (ex: sistemas financeiros).
- Custo alto: dobra a carga de processamento.

**Feature Toggles**: use flags para ativar/desativar funcionalidades.
- Permite deploy de codigo inativo e ativacao gradual.
- Permite canary releases e A/B testing.
- CUIDADO: feature toggles acumulados viram divida tecnica. Tenha processo de limpeza.

### 17. Observabilidade

Microservices exigem observabilidade de primeira classe:

- **Logs estruturados**: JSON com correlation ID propagado entre servicos.
- **Metricas**: RED (Rate, Errors, Duration) para cada servico. USE (Utilization, Saturation, Errors) para infra.
- **Distributed Tracing**: Jaeger, Zipkin, OpenTelemetry. Trace ID propagado em headers.
- **Alertas**: baseados em SLOs, nao em thresholds arbitrarios.

### 18. Chaos Engineering e Game Days

- **Chaos Engineering**: injecao deliberada de falhas em producao para descobrir fraquezas. Ferramentas: Chaos Monkey, Litmus, Gremlin.
- **Game Days**: exercicios simulados de incidentes com o time. Praticam resposta a incidentes, nao apenas a falha tecnica.

Newman enfatiza: se voce nao pratica falhas, a primeira vez que elas acontecem sera em producao,
sob pressao, sem preparo.

---

## Heuristicas Verificaveis

### Fronteiras e Acoplamento

1. **Um servico, um repositorio (ou diretorio independente em monorepo)**
   - Como verificar: verifique se cada servico tem seu proprio `Dockerfile`, `build.gradle`/`pom.xml`/`package.json` independente, sem referencias de build cruzadas.

2. **Um servico, um banco (ou schema isolado)**
   - Como verificar: inspecione connection strings de cada servico. Nenhum servico deve apontar para o mesmo database/schema de outro. Execute query: `SELECT DISTINCT schema_name FROM information_schema.schemata` e mapeie para servicos.

3. **Nenhum servico acessa tabelas de outro servico**
   - Como verificar: revise queries SQL e ORM mappings. Busque por nomes de tabelas que pertencem a outros bounded contexts. Em ORMs, busque entidades de outro dominio importadas.

4. **Toda chamada externa tem timeout configurado**
   - Como verificar: busque por chamadas HTTP/gRPC/banco e valide que todas possuem timeout explicito. Grep por `HttpClient`, `RestTemplate`, `fetch`, `axios` sem configuracao de timeout.

5. **Retries usam backoff exponencial com jitter**
   - Como verificar: busque por configuracoes de retry. Valide que usam backoff exponencial (nao linear ou fixo) e incluem jitter para evitar thundering herd.

6. **Circuit breakers estao configurados em toda comunicacao inter-servico**
   - Como verificar: busque por uso de Resilience4j, Hystrix, Polly ou equivalente. Toda chamada a outro servico deve passar por circuit breaker.

7. **APIs nunca expoe entidades internas diretamente**
   - Como verificar: compare DTOs de API com entidades de dominio. Se forem a mesma classe, ha vazamento. Busque por `@Entity` ou modelos de banco retornados diretamente em controllers.

8. **Correlation ID propagado em todas as chamadas**
   - Como verificar: inspecione interceptors/middlewares HTTP. Verifique que um header (`X-Correlation-ID` ou `traceparent`) e gerado na entrada e propagado em todas as chamadas downstream.

9. **Consumer-Driven Contract tests existem para cada relacao provider-consumer**
   - Como verificar: busque por arquivos Pact (`*.pact.json`) ou configuracoes Spring Cloud Contract. Cada consumer deve ter pelo menos um contrato registrado com cada provider que consome.

10. **Nenhuma biblioteca compartilhada contem logica de dominio**
    - Como verificar: inspecione dependencias compartilhadas entre servicos. Bibliotecas compartilhadas devem conter apenas: serializacao, HTTP utils, logging, metricas. Nunca: entidades, regras de negocio, validacoes de dominio.

### Comunicacao e Eventos

11. **Eventos sao imutaveis e possuem schema versionado**
    - Como verificar: busque por schemas de eventos (Avro, Protobuf, JSON Schema). Verifique que existe versionamento e que schemas antigos sao mantidos para backward compatibility.

12. **Todo endpoint de escrita e idempotente**
    - Como verificar: busque por operacoes POST/PUT/PATCH. Verifique que possuem idempotency key ou que a operacao e naturalmente idempotente (PUT com ID). Busque por `Idempotency-Key` header ou mecanismo equivalente.

13. **Nao ha chamadas sincronas em cadeia com mais de 2 hops**
    - Como verificar: trace uma request tipica usando distributed tracing. Se A chama B que chama C que chama D sincronamente, ha acoplamento temporal excessivo. Considere evento ou coreografia.

14. **Eventos de dominio nao contem dados de outro bounded context**
    - Como verificar: inspecione payloads de eventos. O evento `OrderPlaced` do servico Orders nao deve conter `CustomerAddress` completo — deve conter apenas `customerId`. O consumidor busca detalhes no servico de Customers.

15. **Filas possuem Dead Letter Queue (DLQ) configurada**
    - Como verificar: inspecione configuracoes de filas (SQS, RabbitMQ, Kafka consumer groups). Toda fila deve ter DLQ associada com alertas de monitoramento.

### Deploy e Pipeline

16. **Cada servico tem seu proprio pipeline de CI/CD**
    - Como verificar: inspecione configuracoes de CI/CD. Cada servico deve ter pipeline independente que builda, testa e deploya sem depender de outros servicos.

17. **Deploy de um servico nao requer deploy de outro**
    - Como verificar: analise historico de deploys. Se servicos A e B sempre sao deployados juntos, eles estao acoplados. Verifique release notes e changelogs.

18. **Feature toggles possuem data de expiracao**
    - Como verificar: busque por feature toggles no codigo. Cada toggle deve ter metadata com data de criacao e data de expiracao planejada. Toggles sem data sao divida tecnica.

19. **Testes de contrato rodam no pipeline do provider**
    - Como verificar: verifique que o pipeline do provider executa testes de contrato de todos os consumers antes de deploy. Falha em contrato deve bloquear deploy do provider.

20. **Nenhum servico faz deploy via script manual**
    - Como verificar: verifique que todos os deploys sao automatizados via pipeline. Busque por scripts `deploy.sh` ou runbooks manuais — devem ser eliminados.

### Resiliencia e Observabilidade

21. **Logs sao estruturados (JSON) e incluem request ID**
    - Como verificar: inspecione formato de log de cada servico. Busque por `log.info("message")` sem contexto estruturado. Logs devem ser JSON com campos padronizados.

22. **Metricas RED (Rate, Errors, Duration) expostas por servico**
    - Como verificar: verifique que cada servico expoe metricas Prometheus/OpenMetrics com histogramas de latencia, contadores de erro e rate por endpoint.

23. **Health check endpoint existe e verifica dependencias**
    - Como verificar: cada servico deve ter `/health` ou `/readiness` que verifica conexao com banco, cache, filas. Nao basta retornar 200 — deve verificar dependencias reais.

24. **SLOs definidos e medidos para cada servico**
    - Como verificar: busque por definicoes de SLO (ex: "99.9% de requests < 200ms"). Verifique que dashboards exibem burn rate de error budget.

25. **Graceful shutdown implementado**
    - Como verificar: verifique que servicos interceptam SIGTERM, param de aceitar novas requests, completam requests em andamento e encerram conexoes. Busque por shutdown hooks no codigo.

26. **Testes de resiliencia automatizados (chaos tests)**
    - Como verificar: busque por testes que simulam falha de dependencias (mock de timeout, erro 500, particao de rede). Idealmente integrados ao pipeline de CI.

27. **Rate limiting implementado em APIs publicas**
    - Como verificar: verifique configuracao de rate limiting no API Gateway ou no servico. Busque por headers `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`.

28. **Alertas baseados em SLOs, nao em thresholds arbitrarios**
    - Como verificar: revise regras de alerta. Alertas devem ser baseados em burn rate de error budget, nao em CPU > 80% ou memoria > 90%.

---

## Anti-patterns com Sinais no Codigo

| # | Anti-pattern | Sinal no Codigo/Arquitetura | Severidade | Acao |
|---|---|---|---|---|
| 1 | **Distributed Monolith** | Todos os servicos sao deployados juntos; um Helm chart para todos; release train com data fixa | CRITICA | Identifique acoplamentos e elimine. Se nao for possivel, consolide de volta em monolito |
| 2 | **Shared Database** | Multiplos servicos com connection string para o mesmo banco/schema | CRITICA | Separe bancos. Use Change Data Capture ou API para comunicacao |
| 3 | **God Service** | Um servico com 50+ endpoints que cresce indefinidamente; todos os outros dependem dele | ALTA | Decomponha por Bounded Context. Extraia dominos autonomos |
| 4 | **Chatty Communication** | Trace mostra 20+ chamadas HTTP entre servicos para uma unica operacao de usuario | ALTA | Considere batch API, evento assincrono, ou merge de servicos |
| 5 | **Shared Domain Library** | Biblioteca `common-domain.jar` com entidades usadas por 5+ servicos | ALTA | Elimine. Cada servico deve ter seu proprio modelo. Duplique DTOs |
| 6 | **Synchronous Chain** | A -> B -> C -> D em chamadas sincronas; latencia = soma das latencias | ALTA | Introduza eventos assincronos ou CQRS para quebrar a cadeia |
| 7 | **Mega Event** | Evento com 50+ campos carregando dados de multiplos dominios | MEDIA | Reduza para evento fino com IDs. Consumidores buscam detalhes |
| 8 | **No Timeout** | `HttpClient` instanciado sem timeout; `fetch()` sem `AbortController` | ALTA | Configure timeout em toda chamada externa. Use defaults globais |
| 9 | **Retry Storm** | Retry sem backoff ou com retry count = 10; sem circuit breaker | ALTA | Implemente backoff exponencial com jitter e circuit breaker |
| 10 | **Entity as API** | Classe `@Entity` retornada diretamente no controller como JSON | MEDIA | Crie DTOs de API separados. Mapeie explicitamente |
| 11 | **Feature Toggle Cemetery** | 200 feature toggles sem data de expiracao, muitos nunca removidos | MEDIA | Estabeleca processo de revisao trimestral. Remova toggles expirados |
| 12 | **Log Soup** | Logs em formato texto livre sem correlation ID; `System.out.println` | MEDIA | Padronize logs estruturados com correlation ID e niveis corretos |
| 13 | **Big Bang Migration** | Branch `rewrite-v2` com 6 meses de desenvolvimento sem deploy | CRITICA | Adote Strangler Fig. Migre incrementalmente. Delete o branch |
| 14 | **Test in Production Only** | Nenhum teste de contrato; unico teste e manual em staging | ALTA | Implemente Consumer-Driven Contracts e testes de integracao automatizados |
| 15 | **Orchestrator God** | Saga orchestrator com 2000+ linhas, contendo regras de negocio de 5 dominios | ALTA | Mova regras de negocio para os servicos. Orchestrator deve conter apenas fluxo |
| 16 | **Anemic Events** | Eventos que carregam apenas `{ "id": 123, "action": "updated" }` sem contexto | MEDIA | Inclua dados relevantes no evento para que consumidores nao precisem callback sincrono |
| 17 | **Hardcoded Service URLs** | URLs de outros servicos hardcoded no codigo fonte | MEDIA | Use service discovery, DNS interno, ou variaveis de ambiente |
| 18 | **No Dead Letter Queue** | Filas sem DLQ; mensagens com erro sao descartadas silenciosamente | ALTA | Configure DLQ para toda fila. Alerte quando DLQ receber mensagens |

---

## Checklist de Revisao

### Fronteiras de Servico
- [ ] O servico representa um unico Bounded Context?
- [ ] O servico pode ser deployado independentemente de todos os outros?
- [ ] O servico possui seu proprio banco de dados (ou schema isolado)?
- [ ] O servico tem ownership claro (um time responsavel)?
- [ ] A API publica do servico usa DTOs proprios (nao entidades de banco)?
- [ ] O servico nao importa modelos de dominio de outros servicos?

### Comunicacao
- [ ] Toda chamada sincrona tem timeout configurado?
- [ ] Retries usam backoff exponencial com jitter?
- [ ] Circuit breakers estao configurados para dependencias criticas?
- [ ] Chamadas sincronas em cadeia tem no maximo 2 hops?
- [ ] Eventos sao versionados e possuem schema registrado?
- [ ] Filas tem Dead Letter Queue configurada com alertas?
- [ ] Endpoints de escrita sao idempotentes?

### API e Contratos
- [ ] Existe versionamento de API (URL, header ou content negotiation)?
- [ ] Consumer-Driven Contract tests estao implementados?
- [ ] Mudancas de API seguem Expand and Contract (nunca breaking change direta)?
- [ ] Client libraries sao thin (apenas transporte, sem logica de dominio)?
- [ ] A documentacao de API esta atualizada (OpenAPI/Swagger)?

### Resiliencia
- [ ] Bulkheads isolam recursos criticos (thread pools, connection pools)?
- [ ] Graceful shutdown esta implementado (SIGTERM handler)?
- [ ] Health checks verificam dependencias reais (banco, cache, filas)?
- [ ] Rate limiting esta configurado em APIs publicas?
- [ ] Fallbacks estao definidos para quando dependencias falham?

### Observabilidade
- [ ] Logs sao estruturados (JSON) com campos padronizados?
- [ ] Correlation ID e propagado entre todos os servicos?
- [ ] Distributed tracing esta configurado (OpenTelemetry/Jaeger)?
- [ ] Metricas RED estao expostas e dashboards configurados?
- [ ] SLOs estao definidos e alertas baseados em burn rate?
- [ ] Runbooks existem para os alertas mais criticos?

### Pipeline e Deploy
- [ ] Pipeline de CI/CD e independente por servico?
- [ ] Testes de contrato rodam no pipeline do provider?
- [ ] Feature toggles tem data de expiracao definida?
- [ ] Deploy e automatizado (zero scripts manuais)?
- [ ] Rollback e possivel em menos de 5 minutos?
- [ ] Canary deployment ou blue/green esta configurado?

### Seguranca
- [ ] Comunicacao inter-servico usa mTLS ou auth token?
- [ ] Secrets nao estao hardcoded (usam Vault, SSM, ou equivalente)?
- [ ] API Gateway valida autenticacao antes de rotear para servicos?
- [ ] Principio do menor privilegio aplicado a service accounts?

---

## Perguntas que esta Skill Faz

1. **Este servico pode ser deployado independentemente?** Se nao, por que nao? Qual e a dependencia que impede?

2. **Se eu mudar o schema do banco deste servico, algum outro servico quebra?** Se sim, voce tem um shared database — resolva isso antes de qualquer outra coisa.

3. **Quantos servicos precisam mudar para implementar este requisito de negocio?** Se mais de um, suas fronteiras de servico podem estar erradas.

4. **Qual e o Bounded Context que este servico representa?** Se voce nao consegue responder em uma frase, o servico pode estar fazendo coisas demais.

5. **O que acontece quando este servico fica indisponivel por 5 minutos?** Quais outros servicos sao afetados? Ha fallback? Ha degradacao graceful?

6. **Estamos duplicando codigo entre servicos para evitar acoplamento, ou estamos compartilhando bibliotecas que criam acoplamento?** Qual e o trade-off consciente?

7. **Este evento carrega dados suficientes para o consumidor agir sem callback sincrono?** Se o consumidor precisa chamar o produtor de volta, o evento esta anemic.

8. **Quantos hops sincronos existem no caminho critico desta request?** Cada hop adiciona latencia e ponto de falha.

9. **Se a equipe que mantem o servico X sair de ferias, conseguimos deployar nosso servico?** Se nao, ha acoplamento organizacional.

10. **Estamos usando microservices porque precisamos, ou porque e o hype?** Um modular monolith resolveria o problema com menos complexidade?

11. **Nosso orchestrator de saga contem regras de negocio?** Se sim, ele esta virando um God Service. Regras devem estar nos servicos de dominio.

12. **Quanto tempo leva para um novo desenvolvedor entender o fluxo de uma request de ponta a ponta?** Se leva mais de um dia, a observabilidade esta deficiente.

13. **Temos testes de contrato, ou dependemos de testes end-to-end para pegar quebras de API?** Testes E2E sao lentos, frageis e nao escalam.

14. **Nossos feature toggles tem data de expiracao?** Quantos toggles existem hoje e quantos ja deveriam ter sido removidos?

15. **Praticamos chaos engineering ou game days?** Se nao, a primeira falha grave sera em producao, sob pressao, sem preparo.

16. **A estrutura dos nossos times reflete a arquitetura que queremos?** Se nao, Conway's Law vai trabalhar contra nos.

17. **Estamos medindo SLOs e error budgets, ou nossos alertas sao baseados em thresholds arbitrarios?** Alertas sem SLO geram fadiga de alerta.

---

## Vocabulario

| Termo | Definicao |
|-------|-----------|
| **Independent Deployability** | Capacidade de fazer deploy de um servico sem precisar coordenar ou fazer deploy de nenhum outro servico |
| **Bounded Context** | Fronteira explicita dentro da qual um modelo de dominio e consistente e os termos tem significado unico (conceito de DDD) |
| **Aggregate** | Cluster de entidades e value objects tratados como unidade atomica de consistencia dentro de um Bounded Context (conceito de DDD) |
| **Information Hiding** | Principio de Parnas (1972): um modulo deve esconder suas decisoes de design internas e expor apenas o necessario via interface |
| **Conway's Law** | Organizacoes produzem sistemas que espelham suas estruturas de comunicacao |
| **Inverse Conway Maneuver** | Estruturar times intencionalmente para que a arquitetura emergente seja a desejada |
| **Strangler Fig Pattern** | Estrategia de migracao onde o novo sistema cresce ao redor do antigo, assumindo trafego gradualmente ate o antigo morrer |
| **Consumer-Driven Contract** | Teste de contrato definido pelo consumidor de uma API, executado no pipeline do provider para garantir compatibilidade |
| **Expand and Contract** | Estrategia de evolucao de API: adicione novo campo/endpoint, migre consumidores, remova o antigo |
| **Saga** | Padrao para gerenciar transacoes distribuidas via sequencia de transacoes locais com acoes compensatorias |
| **Orchestration** | Estilo de saga onde um coordenador central dirige a sequencia de chamadas entre servicos |
| **Choreography** | Estilo de saga onde cada servico reage a eventos e emite novos eventos, sem coordenador central |
| **Compensating Transaction** | Acao que "desfaz" semanticamente o efeito de uma transacao anterior em uma saga |
| **Circuit Breaker** | Pattern que interrompe chamadas a um servico que esta falhando repetidamente, permitindo fail-fast e recuperacao |
| **Bulkhead** | Pattern que isola recursos (threads, conexoes) para que falha em um compartimento nao afete outros |
| **Idempotency** | Propriedade de uma operacao que produz o mesmo resultado independente de quantas vezes e executada |
| **Dead Letter Queue (DLQ)** | Fila que recebe mensagens que falharam processamento apos N tentativas, para analise posterior |
| **Distributed Monolith** | Anti-pattern: sistema com multiplos servicos que precisam ser deployados juntos, combinando desvantagens de microservices e monolito |
| **CAP Theorem** | Em sistema distribuido, durante particao de rede, voce deve escolher entre Consistency e Availability |
| **Eventual Consistency** | Modelo onde, apos uma escrita, nem todos os nos refletem o valor imediatamente, mas eventualmente convergem |
| **Service Mesh** | Camada de infraestrutura para comunicacao service-to-service, gerenciando mTLS, retries, circuit breaking e observabilidade |
| **API Gateway** | Ponto de entrada unico para clientes externos, responsavel por autenticacao, roteamento e rate limiting |
| **Feature Toggle** | Mecanismo que permite ativar/desativar funcionalidades em runtime sem deploy de codigo |
| **Correlation ID** | Identificador unico propagado entre servicos para rastrear uma request de ponta a ponta |
| **SLO (Service Level Objective)** | Meta mensuravel de qualidade de servico (ex: 99.9% de requests < 200ms) |
| **Error Budget** | Quantidade de falha permitida antes de violar o SLO; quando consumido, priorize confiabilidade sobre features |
| **Chaos Engineering** | Pratica de injetar falhas deliberadamente em producao para descobrir fraquezas antes que causem incidentes reais |
| **Game Day** | Exercicio simulado de incidentes com o time, praticando resposta a falhas em ambiente controlado |
| **Semantic Versioning** | Esquema de versionamento Major.Minor.Patch onde Major indica breaking changes |
| **Change Data Capture (CDC)** | Pattern que captura mudancas em banco de dados e as propaga como eventos (ex: Debezium) |
| **CQRS** | Command Query Responsibility Segregation — separa modelos de leitura e escrita para otimizar cada um independentemente |
