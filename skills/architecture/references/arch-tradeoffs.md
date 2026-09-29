# Skill: Software Architecture — The Hard Parts — Neal Ford & Mark Richards

## Perspectiva do Autor

Neal Ford e Mark Richards partem de uma premissa central: **nao existem "best practices" em
arquitetura de software — existem apenas trade-offs**. O titulo "The Hard Parts" refere-se
precisamente as decisoes onde nao ha resposta certa, apenas a "menos errada" dado o contexto.

Os autores valorizam:
- **Analise estruturada de trade-offs**: toda decisao arquitetural deve ser documentada com pros, contras e justificativa explicita. Intuicao nao e suficiente.
- **Mensurabilidade**: "If you can't measure it, you can't govern it." Fitness functions automatizadas sao o mecanismo de governanca.
- **Reversibilidade**: decisoes devem ser classificadas como one-way door (irreversivel) ou two-way door (reversivel). O nivel de analise deve ser proporcional a irreversibilidade.
- **Decomposicao guiada por dados, nao por hype**: a parte mais dificil da decomposicao nao e separar codigo — e separar dados. Codigo e facil de mover; dados sao frageis, acoplados e cheios de side-effects.
- **Quanta arquiteturais**: a unidade minima de analise nao e o servico, e o quantum — a menor unidade independentemente deployavel com alta coesao funcional e connascence sincrona.

Ford e Richards sao pragmaticos: eles nao advogam por microservices nem por monolitos. Eles
advogam por **decisoes conscientes baseadas em evidencia**. O livro e fundamentalmente um
**framework de decisao**, nao uma prescricao de estilo arquitetural.

---

## Principios Fundamentais

### 1. Architecture Quantum

O conceito mais original do livro. Um architecture quantum e a **menor unidade independentemente
deployavel** de um sistema que possui:
- **Alta coesao funcional**: os componentes dentro do quantum contribuem para uma mesma capability de negocio.
- **Connascence sincrona**: os componentes internos se comunicam sincronamente e dependem uns dos outros em runtime.

**Implicacoes praticas**:
- Um **monolito** tem um unico quantum. Todas as caracteristicas arquiteturais (performance, disponibilidade, escalabilidade) sao governadas pela mesma unidade.
- Um sistema **distribuido** pode ter multiplos quanta. Cada quantum pode ter caracteristicas arquiteturais diferentes (ex: quantum de pagamento com alta disponibilidade, quantum de relatorio com alta throughput).
- O numero de quanta determina a **granularidade de decisao**: com um quantum, voce tem uma unica escala, um unico SLA, um unico pipeline. Com multiplos, cada um pode evoluir independentemente.

**Decisao chave**: quantos quanta voce realmente precisa? Se a resposta e "um", um monolito
modular e provavelmente suficiente. Se a resposta e "multiplos com caracteristicas diferentes",
arquitetura distribuida e justificada.

### 2. Fitness Functions — Governanca Automatizada

Fitness functions sao **testes automatizados que validam caracteristicas arquiteturais**. Sao o
mecanismo pelo qual decisoes arquiteturais saem de documentos esquecidos e entram no pipeline
de CI/CD como gates executaveis.

#### Taxonomia de Fitness Functions

| Dimensao | Tipos | Exemplo |
|----------|-------|---------|
| **Escopo** | Atomic (testa uma unica caracteristica) vs Holistic (testa interacao entre componentes) | Atomic: "nenhum ciclo entre packages". Holistic: "latencia P99 do fluxo de checkout < 500ms" |
| **Cadencia** | Triggered (dispara em evento) vs Continuous (roda periodicamente) | Triggered: teste de dependencia no build. Continuous: scan de seguranca diario |
| **Resultado** | Static (analise de codigo/config) vs Dynamic (observacao em runtime) | Static: ArchUnit verificando camadas. Dynamic: teste de carga validando SLA |

#### Exemplos Concretos

1. **Direcao de dependencia**: ArchUnit/ArchUnitNET verificando que `domain` nunca importa `infrastructure`.
2. **Deteccao de ciclos**: JDepend, Sonargraph verificando que nao ha ciclos entre packages.
3. **SLA de latencia**: teste de carga automatizado que falha se P99 > threshold.
4. **Metricas de acoplamento**: fan-in/fan-out medido por ferramenta estatica; alerta se excede threshold.
5. **Tamanho de componente**: LOC ou numero de classes por modulo; alerta se cresce alem do limite.
6. **Cobertura de testes de contrato**: percentual de APIs inter-servico cobertas por Consumer-Driven Contracts.

**Principio**: se uma caracteristica arquitetural e importante o suficiente para ser mencionada
em um documento de arquitetura, ela e importante o suficiente para ter uma fitness function que
a valide automaticamente.

### 3. Framework de Analise de Trade-offs

#### 3.1 Toda Decisao Arquitetural e um Trade-off

> "There are no right or wrong answers in architecture — only trade-offs."

Os autores rejeitam o conceito de "best practices" arquiteturais. O que funciona em um contexto
pode ser desastroso em outro. A responsabilidade do arquiteto nao e conhecer "a resposta certa",
e saber **analisar trade-offs sistematicamente**.

#### 3.2 Analise MECE (Mutually Exclusive, Collectively Exhaustive)

Antes de decidir, o arquiteto deve mapear TODAS as opcoes de forma:
- **Mutuamente exclusiva**: cada opcao e distinta — nao ha sobreposicao.
- **Coletivamente exaustiva**: todas as opcoes relevantes estao na lista — nao ha lacuna.

Isso evita dois erros comuns:
1. **Falso dilema**: considerar apenas 2 opcoes quando ha 5.
2. **Opcoes redundantes**: listar variacoes da mesma opcao como se fossem distintas.

#### 3.3 Qualitativo vs Quantitativo

**Analise Qualitativa — Matriz de Pros/Contras**:
Para cada opcao, liste:
- Pros (beneficios concretos)
- Contras (custos concretos)
- Riscos (incertezas)
- Premissas (o que precisa ser verdade para esta opcao funcionar)

**Analise Quantitativa — Weighted Scoring**:
1. Liste as caracteristicas relevantes (performance, simplicidade, custo, etc).
2. Atribua peso a cada caracteristica (soma = 100%).
3. Avalie cada opcao em cada caracteristica (escala 1-5).
4. Calcule score ponderado.
5. A opcao com maior score vence — mas o processo de pontuacao e mais valioso que o resultado numerico, pois forca a equipe a explicitar prioridades.

#### 3.4 Reversibilidade: One-Way Door vs Two-Way Door

| Tipo | Descricao | Nivel de Analise | Exemplo |
|------|-----------|------------------|---------|
| **One-way door** | Decisao dificil ou impossivel de reverter | Exige analise profunda, PoC, validacao com stakeholders | Escolha de banco de dados, linguagem principal, protocolo de comunicacao |
| **Two-way door** | Decisao facilmente reversivel | Analise rapida, tente e ajuste | Escolha de library de logging, formato de serializacao interna, nome de variavel |

**Heuristica**: gaste tempo proporcional a irreversibilidade. Decisoes two-way door nao justificam
semanas de analise. Decisoes one-way door nao devem ser tomadas em uma reuniao de 30 minutos.

### 4. Decomposicao: Quando e Como Quebrar

#### 4.1 Abordagens de Decomposicao

**Component-based decomposition**: identifique componentes logicos (modulos, packages) e promova-os a servicos.
- Vantagem: respeita fronteiras existentes.
- Risco: componentes mal definidos geram servicos mal definidos.

**Tactical forking**: clone o monolito inteiro e remova o que nao pertence a cada clone.
- Vantagem: rapido para comecar.
- Risco: duplicacao massiva de codigo que diverge com o tempo.

**Domain-based decomposition**: use Bounded Contexts (DDD) para definir fronteiras de servico.
- Vantagem: alinhamento com dominio de negocio.
- Risco: requer conhecimento profundo do dominio.

#### 4.2 Quando NAO Quebrar

Decomposicao nem sempre e a resposta. NAO decomponha se:
- **Aumenta a complexidade total**: se o sistema decomposto e mais dificil de entender, operar e debugar que o monolito, a decomposicao nao valeu a pena.
- **Dados nao podem ser separados limpaente**: se duas funcionalidades compartilham as mesmas tabelas com foreign keys bidirecionais, JOINs complexos e transacoes que cruzam boundaries, separar o codigo e facil mas separar os dados e um pesadelo.
- **Transaction boundaries se tornam intoleraveis**: se a consistencia entre os dados deve ser ACID e nao pode ser eventual, manter em um unico processo e mais simples que implementar sagas complexas.
- **O time e pequeno**: microservices para um time de 3 pessoas e over-engineering. A complexidade operacional supera os beneficios.

**Principio fundamental**: "Code is easy to decompose; data is not." A decomposicao de dados
e o fator limitante, nao a decomposicao de codigo.

### 5. Analise de Acoplamento

#### 5.1 Static Coupling

| Metrica | Descricao | Analogia |
|---------|-----------|----------|
| **Afferent coupling (Fan-in)** | Quantos componentes dependem DESTE componente | Quantos clientes este servico tem |
| **Efferent coupling (Fan-out)** | De quantos componentes ESTE componente depende | De quantos servicos este servico depende |

**Instability** = Fan-out / (Fan-in + Fan-out)
- Instability = 0: componente estavel (muitos dependem dele, ele nao depende de ninguem). Deve ser abstrato.
- Instability = 1: componente instavel (ele depende de muitos, ninguem depende dele). Pode mudar livremente.
- Componentes com alto fan-in E alto fan-out sao pontos de risco: muitos dependem dele, mas ele tambem muda frequentemente.

#### 5.2 Dynamic Coupling

Alem do acoplamento no codigo (estatico), existe o acoplamento em runtime:

| Dimensao | Questao | Impacto |
|----------|---------|---------|
| **Sincrono vs Assincrono** | A chamada e bloqueante? | Sincrono = acoplamento temporal (ambos devem estar de pe) |
| **Frequencia de comunicacao** | Quantas chamadas por minuto? | Alta frequencia = alto custo de rede, latencia acumulada |
| **Volume de dados** | Quanto dado e transmitido? | Alto volume = gargalo de bandwidth, serializacao custosa |
| **Criticidade** | A falha na comunicacao impede o fluxo? | Alta criticidade = precisa de circuit breaker, fallback |

#### 5.3 Connascence — Qualidade do Acoplamento

Connascence e um framework para medir a **qualidade** do acoplamento, nao apenas a quantidade.

**Tres propriedades**:
- **Strength (forca)**: quao forte e o acoplamento? (Connascence of Name e fraca; Connascence of Identity e forte)
- **Locality (localidade)**: os componentes acoplados estao proximos? (mesmo modulo = aceitavel; cross-service = perigoso)
- **Degree (grau)**: quantos componentes sao afetados? (2 = baixo; 15 = alto)

**Tipos de connascence (do mais fraco ao mais forte)**:

| Tipo | Descricao | Exemplo | Forca |
|------|-----------|---------|-------|
| **Connascence of Name (CoN)** | Componentes concordam com um nome | Metodo ou endpoint name | Fraca |
| **Connascence of Type (CoT)** | Componentes concordam com um tipo | Tipo de parametro | Fraca |
| **Connascence of Meaning (CoM)** | Componentes concordam com o significado de um valor | `status = 1` significa "ativo" | Media |
| **Connascence of Position (CoP)** | Componentes dependem da ordem de parametros | Funcao com 10 parametros posicionais | Media |
| **Connascence of Algorithm (CoA)** | Componentes devem usar o mesmo algoritmo | Mesmo hash para password nos dois lados | Forte |
| **Connascence of Execution (CoE)** | Componentes devem executar em ordem especifica | A antes de B antes de C | Forte |
| **Connascence of Timing (CoTi)** | Componentes dependem de timing relativo | Race conditions, timeouts | Forte |
| **Connascence of Value (CoV)** | Componentes devem concordar com valores especificos | Constantes duplicadas | Forte |
| **Connascence of Identity (CoI)** | Componentes devem referenciar a mesma instancia | Singleton compartilhado | Muito forte |

**Heuristica**: dentro de um quantum, connascence forte e aceitavel. Entre quanta, busque apenas
connascence fraca (Name, Type).

**Stamp coupling** e um caso especial: passar um objeto inteiro quando apenas parte dele e
necessaria. Aumenta o acoplamento desnecessariamente. Prefira passar apenas os campos necessarios.

### 6. Caracteristicas Arquiteturais

Ford e Richards organizam as "-ilities" em tres categorias:

#### Operacionais
| Caracteristica | Descricao |
|---------------|-----------|
| **Availability** | Percentual do tempo em que o sistema esta operacional |
| **Continuity** | Capacidade de recuperacao apos desastre |
| **Performance** | Latencia e throughput sob carga normal |
| **Recoverability** | Tempo e completude de recuperacao apos falha |
| **Reliability** | Frequencia com que o sistema falha |
| **Robustness** | Capacidade de lidar com condicoes de erro e limites |
| **Scalability** | Capacidade de lidar com aumento de carga |

#### Estruturais
| Caracteristica | Descricao |
|---------------|-----------|
| **Configurability** | Facilidade de alterar configuracao sem rebuild |
| **Extensibility** | Facilidade de adicionar novas funcionalidades |
| **Installability** | Facilidade de instalar em ambientes diversos |
| **Reuse** | Capacidade de reutilizar componentes em contextos diferentes |
| **Localization** | Suporte a multiplos idiomas e formatos |
| **Maintainability** | Facilidade de corrigir bugs e fazer mudancas |
| **Portability** | Capacidade de rodar em plataformas diferentes |

#### Cross-cutting
| Caracteristica | Descricao |
|---------------|-----------|
| **Accessibility** | Suporte a usuarios com necessidades especiais |
| **Archivability** | Capacidade de arquivar ou destruir dados apos periodo |
| **Authentication** | Verificacao de identidade do usuario |
| **Authorization** | Controle de acesso baseado em permissoes |
| **Legal** | Conformidade com legislacao (LGPD, GDPR, SOX) |
| **Privacy** | Protecao de dados pessoais |
| **Security** | Protecao contra ameacas e ataques |
| **Supportability** | Facilidade de diagnosticar e resolver problemas |
| **Usability** | Facilidade de uso pelo usuario final |

**Regra dos 7**: nunca escolha mais de 7 caracteristicas como drivers. Idealmente, selecione
3-5 que sao verdadeiramente diferenciadoras. Tentar otimizar para tudo e otimizar para nada.

### 7. Patterns de Arquitetura Distribuida

#### 7.1 Orchestration vs Choreography — Framework de Decisao

| Criterio | Favorece Orchestration | Favorece Choreography |
|----------|----------------------|----------------------|
| **Complexidade do fluxo** | Fluxo longo com muitos passos e condicoes | Fluxo simples com poucos passos |
| **Visibilidade** | Precisa de visao centralizada do estado | Cada servico gerencia seu estado |
| **Acoplamento** | Aceita coordenador central | Quer desacoplamento maximo |
| **Error handling** | Compensacao complexa com logica condicional | Compensacao simples e local |
| **Performance** | Latencia nao e critica | Latencia minima entre passos |
| **Evolucao** | Fluxo muda frequentemente (centralizado = facil de mudar) | Fluxo e estavel (descentralizado = menos bottleneck) |

#### 7.2 Saga Patterns — Classificacao de Ford/Richards

Ford e Richards classificam sagas por nivel de complexidade e risco:

| Saga Pattern | Comunicacao | Consistencia | Coordenacao | Complexidade |
|-------------|-------------|-------------|-------------|------------|
| **Epic Saga** | Sincrona | Atomica | Orchestrated | Baixa — mas alto acoplamento |
| **Phone Tag Saga** | Sincrona | Atomica | Choreographed | Media — acoplamento temporal mas descentralizado |
| **Fairy Tale Saga** | Sincrona | Eventual | Orchestrated | Media — coordenacao clara com eventual consistency |
| **Fantasy Fiction Saga** | Sincrona | Eventual | Choreographed | Alta — sem coordenacao central e sem consistencia forte |
| **Horror Story Saga** | Assincrona | Atomica | Orchestrated | ANTI-PATTERN — tentar atomicidade com comunicacao assincrona e ilusao perigosa |
| **Parallel Saga** | Assincrona | Eventual | Choreographed | Alta — maximo desacoplamento mas maximo esforco de observabilidade |
| **Time Travel Saga** | Assincrona | Eventual | Orchestrated | Media-alta — boa combinacao de desacoplamento e controle |
| **Anthology Saga** | Misto | Misto | Misto | Variavel — combina patterns conforme necessidade por trecho do fluxo |

**Horror Story Saga e o anti-pattern**: tentar garantir atomicidade (ACID) com comunicacao
assincrona e fundamentalmente impossivel. Mensagens podem ser perdidas, duplicadas, reordenadas.
Se voce precisa de atomicidade, use comunicacao sincrona ou aceite eventual consistency.

#### 7.3 Transactional Patterns

| Pattern | Descricao | Quando Usar |
|---------|-----------|-------------|
| **Background Synchronization** | Processo batch periodico sincroniza dados entre servicos | Consistencia eventual e aceitavel com atraso de minutos/horas |
| **Orchestrated Request-Based** | Orchestrator coordena chamadas sincronas em sequencia | Consistencia imediata necessaria, fluxo complexo |
| **Event-Based** | Servicos reagem a eventos e atualizam dados locais | Alta throughput, desacoplamento temporal desejado |

### 8. Data Ownership e Partitioning

#### 8.1 Patterns de Ownership

| Pattern | Descricao | Quando Usar | Risco |
|---------|-----------|-------------|-------|
| **Joint Ownership** | Multiplos servicos leem/escrevem os mesmos dados | ANTI-PATTERN — evitar sempre | Conflitos de escrita, acoplamento forte, sem source of truth |
| **Data Domain** | Um servico e dono dos dados; outros consultam via API | Padrao recomendado para a maioria dos casos | Latencia de API, indisponibilidade do owner |
| **Delegate** | Um servico delega operacao de dados a outro servico que e o owner | Quando logica de escrita e complexa e centralizada | Acoplamento com o delegate |
| **Service Consolidation** | Merge de servicos que compartilham dados demais | Quando dois servicos nao conseguem ser separados nos dados | Servico cresce e pode virar God Service |

#### 8.2 Database Strategies

**Database-per-service** (ideal):
- Cada servico tem seu proprio banco.
- Isolamento total. Schema e detalhe interno.
- Desafio: JOINs cross-service sao impossiveis — use API composition ou CQRS.

**Shared database** (stepping stone):
- Multiplos servicos acessam o mesmo banco, mas com schemas separados.
- Passo intermediario em migracoes de monolito.
- Risco: schemas "separados" frequentemente compartilham tabelas na pratica.

**Data mesh**:
- Dados tratados como produto, com ownership de dominio.
- **Domain-oriented**: cada dominio e responsavel por seus dados.
- **Self-serve platform**: infraestrutura de dados como plataforma.
- **Federated governance**: padroes definidos coletivamente, nao centralmente.
- **Data as product**: cada dataset tem SLA, documentacao, qualidade medida.

### 9. Architecture Decision Records (ADRs)

Ford e Richards enfatizam que decisoes arquiteturais devem ser **documentadas formalmente**:

**Estrutura de um ADR**:
- **Status**: Proposed, Accepted, Superseded.
- **Context**: por que esta decisao e necessaria?
- **Decision**: o que foi decidido?
- **Consequences**: quais sao os trade-offs aceitos? O que ganhamos e o que perdemos?

Cada decisao significativa (escolha de banco, protocolo, pattern de comunicacao) deve ter um ADR.
ADRs sao imutaveis — quando uma decisao muda, crie um novo ADR que supersede o anterior.

---

## Heuristicas Verificaveis

### Architecture Quantum e Boundaries

1. **Cada quantum tem seu proprio pipeline de deploy**
   - Como verificar: mapeie quanta para pipelines de CI/CD. Se dois quanta compartilham pipeline e sao deployados juntos, nao sao realmente quanta independentes.

2. **Caracteristicas arquiteturais sao definidas por quantum, nao globalmente**
   - Como verificar: revise documentacao de SLAs e SLOs. Se ha um unico SLA para o sistema inteiro, ha provavelmente um unico quantum. Cada quantum deve ter suas proprias metas de performance, availability, etc.

3. **Comunicacao sincrona nao cruza fronteiras de quantum**
   - Como verificar: se dois componentes se comunicam sincronamente e precisam estar de pe ao mesmo tempo, eles estao no mesmo quantum. Se estao em quanta diferentes mas se comunicam sincronamente, a fronteira esta errada.

4. **Numero de caracteristicas arquiteturais drivers <= 7 (idealmente 3-5)**
   - Como verificar: revise o documento de arquitetura. Conte quantas "-ilities" sao listadas como "criticas". Se mais de 7, force priorizacao. Nao e possivel otimizar para tudo.

5. **Cada decisao arquitetural significativa tem um ADR**
   - Como verificar: busque por diretorio `/docs/adr/` ou `/architecture/decisions/`. Cada decisao de banco, protocolo, framework, pattern deve ter ADR correspondente.

### Fitness Functions e Governanca

6. **Fitness functions existem para cada caracteristica arquitetural declarada como driver**
   - Como verificar: para cada "-ility" listada como driver, identifique o teste automatizado correspondente. Se nao existe, a caracteristica e aspiracional, nao governada.

7. **Direcao de dependencia e validada automaticamente no build**
   - Como verificar: busque por ArchUnit, ArchUnitNET, Dependency-Cruiser ou equivalente no pipeline. Regras como "domain nao importa infrastructure" devem ser testes que falham o build.

8. **Ciclos entre packages/modulos sao detectados e bloqueados**
   - Como verificar: busque por JDepend, Sonargraph, Madge, ou script customizado no pipeline que detecta ciclos. Ciclos entre packages sao anti-pattern — devem bloquear o build.

9. **Metricas de acoplamento sao medidas e tem thresholds**
   - Como verificar: fan-in e fan-out sao calculados por ferramenta estatica. Existe alerta quando um componente excede threshold de fan-out (ex: fan-out > 10).

10. **Testes de performance automatizados validam SLAs no pipeline**
    - Como verificar: busque por k6, Gatling, JMeter ou equivalente integrado ao CI/CD. Testes devem falhar se P99 excede SLA declarado.

### Trade-off Analysis

11. **Decisoes one-way door tem PoC documentada**
    - Como verificar: para decisoes irreversiveis (banco, linguagem, protocolo), busque por PoC report ou spike com metricas comparativas. Se a decisao foi tomada sem PoC, o risco nao foi mitigado.

12. **Analise de trade-off inclui pelo menos 3 opcoes (MECE)**
    - Como verificar: revise ADRs. A secao "Alternatives Considered" deve listar pelo menos 3 opcoes mutuamente exclusivas. Se lista apenas "opcao escolhida vs. nao fazer nada", a analise e superficial.

13. **Weighted scoring documenta pesos e pontuacoes explicitamente**
    - Como verificar: para decisoes complexas, busque por matriz de scoring com pesos explicitos. Se a justificativa e apenas prosa ("escolhemos X porque e melhor"), falta rigor.

14. **Premissas de decisoes arquiteturais sao listadas e revisadas periodicamente**
    - Como verificar: ADRs devem ter secao "Assumptions". Crie calendar reminder para revisar premissas trimestralmente. Se premissa invalida, re-avalie a decisao.

### Decomposicao

15. **Decomposicao foi precedida por analise de acoplamento de dados**
    - Como verificar: antes de separar servicos, deve existir mapeamento de tabelas compartilhadas, foreign keys cross-boundary, queries JOIN cross-boundary. Se esse mapeamento nao existe, a decomposicao e prematura.

16. **Servicos nao fazem JOIN cross-database em runtime**
    - Como verificar: revise queries SQL de cada servico. Se ha queries acessando tabelas de outro servico (mesmo que indiretamente via view), os dados nao foram separados.

17. **Transacoes que cruzam boundaries de servico usam Saga (nao 2PC)**
    - Como verificar: busque por two-phase commit, XA transactions, distributed transactions. Se encontrar, substitua por Saga com compensating transactions.

18. **Decomposicao nao aumentou a complexidade total**
    - Como verificar: compare metricas antes/depois: numero de deploys com falha, tempo medio de resolucao de incidentes, tempo de onboarding de novo dev. Se pioraram, a decomposicao pode ter sido prematura.

### Acoplamento

19. **Fan-out de nenhum componente excede threshold (ex: 10)**
    - Como verificar: use ferramenta de analise estatica para calcular fan-out. Componentes com fan-out > 10 sao candidatos a refatoracao (quebrar em componentes menores ou introduzir abstracoes).

20. **Connascence forte nao cruza fronteiras de quantum**
    - Como verificar: mapeie connascence entre componentes de quanta diferentes. Se ha Connascence of Algorithm, Timing, ou Identity cross-quantum, e um sinal de fronteira errada ou acoplamento perigoso.

21. **Stamp coupling e identificado e removido em APIs publicas**
    - Como verificar: revise payloads de API. Se um endpoint recebe `CustomerFullObject` quando precisa apenas de `customerId` e `email`, ha stamp coupling. Crie DTOs minimos.

22. **Comunicacao assincrona nao tenta garantir atomicidade**
    - Como verificar: busque por patterns que tentam simular transacao ACID sobre mensagens assincronas. Se ha logica de "rollback" em consumers de fila, a consistencia e eventual — aceite isso explicitamente.

23. **Instability metric de componentes de dominio e baixa (proximo de 0)**
    - Como verificar: calcule Instability = Fan-out / (Fan-in + Fan-out) para componentes de dominio core. Devem ter Instability baixa (muitos dependem deles, eles dependem de poucos).

### Data e Ownership

24. **Cada entidade de dados tem um unico owner (servico)**
    - Como verificar: crie matriz entidade-servico. Cada entidade deve ter exatamente um servico na coluna "write". Se ha dois servicos que escrevem na mesma entidade, ha joint ownership (anti-pattern).

25. **Shared database tem plano de migracao com data**
    - Como verificar: se multiplos servicos compartilham banco, deve existir roadmap documentado com datas para database-per-service. Shared database e stepping stone, nao estado final.

26. **Data mesh domains tem SLA e documentacao como "produto"**
    - Como verificar: para cada dataset exposto, verifique: existe SLA de freshness? Existe documentacao de schema? Existe owner com contato? Se nao, nao e data-as-product.

27. **API composition substitui JOINs cross-service**
    - Como verificar: onde antes havia JOIN entre tabelas de servicos diferentes, deve haver API composition (BFF ou aggregation layer) que chama ambos servicos e combina resultados em memoria.

28. **CQRS e usado quando read model e write model divergem significativamente**
    - Como verificar: se ha materialized views complexas ou queries com 5+ JOINs para leitura mas escrita e simples, CQRS e justificado. Se read e write sao similares, CQRS e over-engineering.

29. **Nenhum servico faz escrita direta no banco de outro servico**
    - Como verificar: revise connection strings e grants de banco. Nenhum servico deve ter permissao de escrita em schema de outro servico. Read-only access e toleravel como stepping stone.

30. **Architecture Decision Records (ADRs) sao criados para decisoes de decomposicao**
    - Como verificar: busque no diretorio de ADRs por decisoes de "split", "extract", "merge" de servicos. Cada decisao de decomposicao deve ter ADR com Context, Decision, Consequences.

---

## Anti-patterns com Sinais no Codigo

| # | Anti-pattern | Sinal no Codigo/Arquitetura | Severidade | Acao |
|---|---|---|---|---|
| 1 | **Joint Ownership de dados** | Dois ou mais servicos com write access ao mesmo schema/tabela | CRITICA | Defina um unico owner. Outros servicos leem via API ou eventos |
| 2 | **Horror Story Saga** | Logica de "rollback" ou "compensacao atomica" em consumers de fila assincrona | CRITICA | Aceite eventual consistency ou mude para comunicacao sincrona |
| 3 | **Fitness Function vaporware** | Documento de arquitetura lista 12 "-ilities" como drivers, mas nenhuma tem teste automatizado | ALTA | Para cada driver, crie pelo menos uma fitness function executavel |
| 4 | **Analysis Paralysis** | Decisao two-way door com 3 semanas de analise, 5 reunioes e 20 paginas de documentacao | MEDIA | Classifique reversibilidade. Two-way door: decida em horas, nao semanas |
| 5 | **Falso dilema arquitetural** | ADR com apenas 2 opcoes: "escolhida" e "nao fazer nada" | MEDIA | Exija pelo menos 3 opcoes mutuamente exclusivas (MECE) |
| 6 | **God Component** | Componente com fan-out > 15 e fan-in > 15 simultaneamente | ALTA | Decomponha. Um componente nao pode ser simultaneamente dependencia de todos E dependente de todos |
| 7 | **Stamp Coupling em APIs** | Endpoint recebe `CustomerFullObject` (30 campos) quando precisa de 3 campos | MEDIA | Crie request DTOs minimos com apenas campos necessarios |
| 8 | **Premature decomposition** | 15 microservices para sistema com 3 desenvolvedores e sem trafego relevante | ALTA | Consolide em monolito modular. Decomponha quando houver razao concreta |
| 9 | **Ciclos de dependencia** | Package A importa B, B importa C, C importa A | ALTA | Quebre ciclo com inversao de dependencia (DIP) ou extraia interface comum |
| 10 | **ADR ausente para decisao critica** | Banco de dados escolhido sem documentacao de alternativas avaliadas | MEDIA | Crie ADR retroativo. Documente premissas para futura reavaliacao |
| 11 | **Quantum com multiplos SLAs** | Um unico servico com endpoints que tem SLAs radicalmente diferentes (1ms vs 5s) | ALTA | Separe em quanta diferentes. Cada quantum deve ter SLA coerente |
| 12 | **Connascence of Algorithm cross-service** | Dois servicos implementam independentemente o mesmo algoritmo de hash/criptografia | ALTA | Centralize em shared library de infra (nao dominio) ou use servico dedicado |
| 13 | **Database monolith** | Servicos separados no codigo mas todos apontam para o mesmo PostgreSQL com 200 tabelas | CRITICA | Inicie data decomposition. Comece pelo dominio com menos foreign keys |
| 14 | **Over-optimization** | Documento de arquitetura lista 15 caracteristicas como "criticas" | MEDIA | Force priorizacao: no maximo 7, idealmente 3-5. Otimizar tudo = otimizar nada |
| 15 | **Invisible trade-off** | Decisao tomada sem registro de contras ou riscos; apenas justificativa positiva | MEDIA | Reescreva ADR incluindo trade-offs explicitos. Toda decisao tem custo |
| 16 | **Two-phase commit em sistema distribuido** | Uso de XA transactions ou distributed transaction manager entre servicos | CRITICA | Substitua por Saga pattern com compensating transactions |
| 17 | **Metricas de acoplamento nao monitoradas** | Nenhuma ferramenta de analise estatica mede fan-in/fan-out; ninguem sabe quais componentes sao mais acoplados | MEDIA | Integre ferramenta de analise estatica no pipeline (Sonargraph, JDepend, NDepend) |
| 18 | **Data decomposition ignorada** | Servicos decompostos no codigo, mas migration scripts ainda rodam no mesmo banco central | ALTA | Planeje data decomposition com Change Data Capture ou eventos |

---

## Checklist de Revisao

### Architecture Quantum
- [ ] Os quanta estao claramente identificados e documentados?
- [ ] Cada quantum e independentemente deployavel?
- [ ] Cada quantum tem suas proprias caracteristicas arquiteturais definidas?
- [ ] Comunicacao sincrona ocorre apenas dentro de um quantum (nao entre quanta)?
- [ ] O numero de quanta e justificado por necessidades reais (nao por hype)?

### Fitness Functions
- [ ] Cada caracteristica arquitetural driver tem pelo menos uma fitness function?
- [ ] Fitness functions rodam no pipeline de CI/CD (nao manualmente)?
- [ ] Direcao de dependencia e validada automaticamente (ArchUnit ou equivalente)?
- [ ] Ciclos entre packages sao detectados e bloqueiam o build?
- [ ] Metricas de acoplamento (fan-in/fan-out) sao medidas e tem thresholds?
- [ ] Testes de performance automatizados validam SLAs?
- [ ] Fitness functions sao revisadas quando caracteristicas arquiteturais mudam?

### Trade-off Analysis
- [ ] Decisoes one-way door passaram por PoC ou spike?
- [ ] Decisoes two-way door foram tomadas rapidamente (sem over-analysis)?
- [ ] Analise MECE lista pelo menos 3 opcoes mutuamente exclusivas?
- [ ] Weighted scoring documenta pesos e pontuacoes explicitamente?
- [ ] Premissas de decisoes sao revisadas periodicamente?
- [ ] ADRs existem para todas as decisoes significativas?
- [ ] ADRs incluem trade-offs (nao apenas justificativa positiva)?

### Decomposicao
- [ ] Analise de acoplamento de dados foi feita antes da decomposicao?
- [ ] Tabelas com foreign keys cross-boundary foram mapeadas?
- [ ] Transacoes que cruzam boundaries tem strategy definida (Saga, eventual consistency)?
- [ ] A decomposicao nao aumentou complexidade total (metricas comprovam)?
- [ ] O time tem tamanho suficiente para suportar a complexidade operacional?
- [ ] Database-per-service e o target (mesmo que shared database seja stepping stone)?

### Acoplamento
- [ ] Fan-out de cada componente esta dentro do threshold?
- [ ] Connascence forte (Algorithm, Timing, Identity) nao cruza boundaries de quantum?
- [ ] Stamp coupling foi eliminado de APIs publicas?
- [ ] Instability de componentes core e baixa (proximo de 0)?
- [ ] Dynamic coupling (frequencia, volume, criticidade) esta mapeado?
- [ ] Comunicacao assincrona nao tenta simular atomicidade?

### Data Ownership
- [ ] Cada entidade de dados tem um unico servico owner?
- [ ] Nenhum servico escreve diretamente no banco de outro?
- [ ] JOINs cross-service foram substituidos por API composition ou CQRS?
- [ ] Shared databases tem plano de migracao documentado?
- [ ] Datasets expostos como "produto" tem SLA e documentacao?

### Documentacao e Governanca
- [ ] Architecture Decision Records estao atualizados?
- [ ] Caracteristicas arquiteturais drivers estao priorizadas (max 7, ideal 3-5)?
- [ ] Decisoes superseded estao marcadas no historico de ADRs?
- [ ] Premissas de ADRs tem data de revisao agendada?

---

## Perguntas que esta Skill Faz

1. **Quantos architecture quanta este sistema realmente tem?** Se a resposta e "um", por que estamos complicando com arquitetura distribuida?

2. **Quais sao as 3-5 caracteristicas arquiteturais que sao verdadeiramente drivers?** Se a lista tem mais de 7, nao ha priorizacao real.

3. **Esta decisao e one-way door ou two-way door?** Estamos gastando tempo proporcional a irreversibilidade?

4. **Quais fitness functions validam essa caracteristica arquitetural?** Se nao existe fitness function, a caracteristica e aspiracional.

5. **Quais alternativas foram consideradas (MECE)?** Se so ha "opcao escolhida" e "nao fazer nada", a analise e incompleta.

6. **Quais sao os trade-offs desta decisao?** Se so ha beneficios listados e nenhum custo, a analise e desonesta.

7. **Qual e o fan-out deste componente?** Se excede 10, ele esta acoplado a coisas demais e e fragil a mudancas.

8. **A decomposicao de dados foi analisada, ou apenas a decomposicao de codigo?** Codigo e facil; dados sao o problema real.

9. **Qual tipo de connascence existe entre estes dois servicos?** Se e forte (Algorithm, Timing), eles deveriam estar no mesmo quantum.

10. **Este endpoint recebe mais dados do que precisa (stamp coupling)?** Se sim, reduza o payload ao minimo necessario.

11. **Esta saga tenta garantir atomicidade com comunicacao assincrona (Horror Story)?** Se sim, ou mude para sincrono ou aceite eventual consistency.

12. **Temos Joint Ownership de alguma entidade de dados?** Se dois servicos escrevem na mesma tabela, nao ha source of truth.

13. **O shared database tem plano de migracao para database-per-service?** Se nao, e "temporario" ha quanto tempo?

14. **Esta decisao tem ADR?** Se nao, quando as premissas mudarem, ninguem vai lembrar por que foi tomada.

15. **Quantos microservices temos e quantas pessoas temos?** Se a razao servicos/pessoas e > 3, ha complexidade operacional desproporcional.

16. **As premissas que justificaram esta decisao arquitetural ainda sao validas?** Contexto muda — decisoes devem ser reavaliadas.

17. **Estamos decompondo porque precisamos ou porque e o pattern da moda?** Se o monolito modular atende, a complexidade distribuida nao e justificada.

18. **Qual e o custo total de ownership desta arquitetura (infra + operacao + cognitive load)?** A conta mais cara de microservices nao e a de cloud — e a de complexidade cognitiva.

---

## Vocabulario

| Termo | Definicao |
|-------|-----------|
| **Architecture Quantum** | Menor unidade independentemente deployavel com alta coesao funcional e connascence sincrona. Determina a granularidade de caracteristicas arquiteturais |
| **Fitness Function** | Teste automatizado que valida uma caracteristica arquitetural. Mecanismo de governanca executavel |
| **Trade-off** | Ganho em uma dimensao ao custo de perda em outra. Toda decisao arquitetural e um trade-off |
| **MECE** | Mutually Exclusive, Collectively Exhaustive — framework para garantir que todas as opcoes sao distintas e nenhuma foi omitida |
| **One-way Door** | Decisao dificil ou impossivel de reverter (ex: escolha de banco, protocolo principal). Exige analise profunda |
| **Two-way Door** | Decisao facilmente reversivel (ex: library de logging, formato interno). Pode ser tomada rapidamente |
| **Weighted Scoring** | Tecnica quantitativa de avaliacao onde criterios tem pesos e opcoes recebem notas, gerando score ponderado |
| **Architecture Decision Record (ADR)** | Documento formal que registra uma decisao arquitetural, seu contexto, alternativas e consequencias |
| **Connascence** | Metrica de qualidade de acoplamento entre componentes. Mede forca, localidade e grau de interdependencia |
| **Afferent Coupling (Fan-in)** | Numero de componentes que dependem de um dado componente. Alto fan-in = componente influente e estavel |
| **Efferent Coupling (Fan-out)** | Numero de componentes dos quais um dado componente depende. Alto fan-out = componente fragil a mudancas |
| **Instability Metric** | Fan-out / (Fan-in + Fan-out). 0 = maximamente estavel. 1 = maximamente instavel |
| **Stamp Coupling** | Passar um objeto inteiro quando apenas parte dele e necessaria, criando acoplamento desnecessario |
| **Connascence of Name (CoN)** | Tipo mais fraco de connascence: componentes concordam com o nome de algo (metodo, endpoint) |
| **Connascence of Algorithm (CoA)** | Tipo forte: componentes devem usar o mesmo algoritmo (hash, criptografia, serializacao) |
| **Connascence of Timing (CoTi)** | Tipo forte: componentes dependem de timing relativo (race conditions, timeouts coordenados) |
| **Epic Saga** | Saga com comunicacao sincrona, consistencia atomica e coordenacao orchestrated — simples mas altamente acoplada |
| **Horror Story Saga** | Anti-pattern: saga que tenta garantir atomicidade com comunicacao assincrona — fundamentalmente impossivel |
| **Fairy Tale Saga** | Saga sincrona com eventual consistency e orchestration — bom equilibrio para muitos casos |
| **Joint Ownership** | Anti-pattern: multiplos servicos com escrita no mesmo dado, sem source of truth claro |
| **Data Domain** | Pattern onde um servico e dono dos dados e outros consultam via API — padrao recomendado |
| **Data Mesh** | Abordagem de dados descentralizada: domain-oriented, self-serve, federated governance, data as product |
| **API Composition** | Pattern que substitui JOINs cross-service: um componente chama multiplas APIs e combina resultados em memoria |
| **Background Synchronization** | Pattern transacional: processo batch periodico sincroniza dados entre servicos com atraso toleravel |
| **Component-based Decomposition** | Abordagem de decomposicao que promove componentes logicos existentes (modulos, packages) a servicos independentes |
| **Tactical Forking** | Abordagem de decomposicao que clona o monolito e remove codigo nao pertinente de cada clone |
| **Domain-based Decomposition** | Abordagem de decomposicao que usa Bounded Contexts do DDD como fronteiras de servico |
| **Operational Characteristics** | Caracteristicas arquiteturais relacionadas a operacao: availability, performance, scalability, reliability, etc |
| **Structural Characteristics** | Caracteristicas arquiteturais relacionadas a estrutura: maintainability, extensibility, portability, etc |
| **Cross-cutting Characteristics** | Caracteristicas arquiteturais transversais: security, privacy, accessibility, legal compliance, etc |
