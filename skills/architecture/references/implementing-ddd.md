# Skill: Implementing Domain-Driven Design — Vaughn Vernon

## Perspectiva do Autor

Vaughn Vernon e o engenheiro que pegou a teoria de Eric Evans e a transformou em regras operacionais.
Onde Evans fala em filosofia e insight, Vernon fala em **regras, tamanhos, transacoes e wiring**.

Vernon defende que:

1. **DDD falha na pratica porque desenvolvedores nao sabem dimensionar Aggregates.** O erro mais
   comum e criar Aggregates enormes que causam contention, locks longos e falhas de concorrencia.
   As 4 regras de design de Aggregate sao o coracao do livro.

2. **Eventual consistency nao e um mal necessario — e o design correto.** A maioria dos processos
   de negocio tolera milissegundos ou segundos de inconsistencia. Transacoes distribuidas sao a
   excecao, nao a regra. Domain Events + eventual consistency e o padrao primario.

3. **Application Services e Domain Services tem papeis completamente distintos.** Confundir os
   dois e a porta de entrada para o Anemic Domain Model. Application Service orquestra;
   Domain Service encapsula logica de negocio que nao cabe em um unico Aggregate.

4. **CQRS e Event Sourcing sao ferramentas poderosas, mas opcionais.** Vernon os apresenta como
   opcoes taticas, nao como requisitos de DDD. Usar CQRS sem necessidade e overengineering.
   Event Sourcing tem custos reais que precisam ser ponderados.

5. **Bounded Context e a unidade de deploy e de autonomia de time.** Vernon trata Bounded Contexts
   como fronteiras organizacionais e tecnicas, nao apenas conceituais. Cada contexto deve poder
   evoluir independentemente.

6. **O codigo deve ser a implementacao literal do modelo.** Vernon rejeita a ideia de que o modelo
   e um diagrama e o codigo e "outra coisa". Se o codigo nao expressa o modelo, o modelo nao existe.

Vernon e prescritivo onde Evans e descritivo. Ele da regras numeradas, tamanhos recomendados,
e exemplos concretos de implementacao. Seu livro e um manual de campo para quem quer aplicar DDD
em projetos reais, com todas as decisoes de wiring e infraestrutura resolvidas.

---

## Principios Fundamentais

### 1. As 4 Regras de Design de Aggregate (Vernon's Rules)

Estas sao as regras mais importantes do livro. Vernon as trata como leis, nao sugestoes.

**Regra 1: Proteger invariantes de negocio dentro dos limites do Aggregate.**
- O Aggregate e o guardiao das regras de consistencia
- Toda operacao que modifica estado deve passar pela Aggregate Root
- A Root valida invariantes antes e depois de qualquer mudanca
- Se uma invariante envolve dois Aggregates, ou a invariante esta no Aggregate errado,
  ou os limites estao errados, ou a consistencia deve ser eventual
- Invariantes devem ser verificaveis no momento da mudanca, sem consultar outros Aggregates

**Regra 2: Desenhar Aggregates pequenos.**
- O ideal: uma Aggregate Root (Entity) + Value Objects. Sem entidades internas extras.
- Se o Aggregate tem mais de 3-4 entidades internas, provavelmente e grande demais
- Aggregates grandes causam: lock contention, transacoes longas, conflitos de concorrencia,
  dificuldade de teste, acoplamento interno excessivo
- Comece com o menor Aggregate possivel e so expanda se uma invariante de negocio exigir
- "Small Aggregate" nao significa Aggregate sem logica — significa Aggregate focado
- Um Aggregate com 1 Root Entity e 5 Value Objects ricos e perfeitamente valido

**Regra 3: Referenciar outros Aggregates apenas por identidade.**
- Nunca segure referencia de objeto para outro Aggregate (nao faca `private Pedido pedido`)
- Use o ID: `private PedidoId pedidoId`
- Isso impede que um Aggregate navegue para dentro de outro e quebre encapsulamento
- Impede que o ORM carregue grafos de objetos descontrolados (lazy loading hell)
- Torna explicito que Aggregates sao unidades independentes
- Se voce precisa do outro Aggregate, carregue-o via Repository no Application Service

**Regra 4: Usar eventual consistency para regras que cruzam limites de Aggregate.**
- Se uma acao no Aggregate A precisa atualizar o Aggregate B, use Domain Events
- O Aggregate A publica um evento; um handler assincrono atualiza o Aggregate B
- Cada Aggregate e modificado em sua propria transacao
- A maioria dos domain experts aceita eventual consistency quando explicada em termos de negocio
  ("o relatorio de estoque pode levar alguns segundos para refletir uma venda")
- Se o domain expert exige consistencia imediata, questione: e realmente imediata, ou e
  "rapida o suficiente"? Na maioria dos casos, e a segunda opcao

### 2. Dimensionamento de Aggregates

Vernon fornece heuristicas concretas para dimensionamento:

**Tamanho ideal:**
- 1 Aggregate Root (Entity) + Value Objects necessarios para invariantes
- Se possivel, a Root e a unica Entity do Aggregate
- Value Objects podem ser colecoes (ex: lista de `LineItem` como VOs dentro de `Order`)

**Quando expandir o Aggregate:**
- Apenas se uma invariante de negocio genuina exige consistencia transacional
- Exemplo valido: `Order` precisa garantir que `total == sum(lineItems.price)` atomicamente
- Exemplo invalido: `Order` segura referencia para `Customer` porque "precisa do nome"

**Quando o Aggregate esta grande demais:**
- Transacoes demoram mais que 200-500ms regularmente
- Conflitos de concorrencia (OptimisticLockException) sao frequentes
- O Aggregate tem mais de 3-4 entidades internas
- Testes do Aggregate exigem setup complexo com muitos objetos
- Mudancas em uma parte do Aggregate quebram testes de outra parte

**Como dividir um Aggregate grande:**
1. Identifique quais invariantes realmente precisam de consistencia transacional
2. Separe entidades que nao participam dessas invariantes em Aggregates proprios
3. Conecte os novos Aggregates por ID (Regra 3)
4. Use Domain Events para manter consistencia eventual (Regra 4)
5. Valide com o domain expert que a eventual consistency e aceitavel

### 3. Domain Events

Vernon trata Domain Events como cidadaos de primeira classe no design tatico.

**Definicao e convencoes:**
- Um Domain Event e um fato que aconteceu no dominio, expresso no passado
- Nomeacao: verbo no participio passado — `PedidoRealizado`, `PagamentoAprovado`,
  `EstoqueReservado`, `ClienteCadastrado`
- Cada evento e um objeto imutavel com timestamp e dados relevantes
- Eventos nunca devem ser modificados apos publicacao
- Eventos carregam dados suficientes para que o consumidor aja sem consultar a origem

**Ciclo de vida do evento:**
1. O Aggregate executa a operacao de negocio e muda seu estado
2. O Aggregate cria o Domain Event como parte da operacao
3. O evento e publicado (dentro da mesma transacao via outbox, ou apos commit)
4. Subscribers recebem o evento e executam reacoes
5. Cada subscriber roda em sua propria transacao

**Estrategias de publicacao:**
- **In-process synchronous:** simples, mas acopla publisher e subscriber na mesma transacao.
  Util para Bounded Contexts dentro do mesmo processo.
- **Outbox Pattern:** o evento e salvo na mesma transacao que a mudanca de estado.
  Um processo separado le a outbox e publica no broker. Garante at-least-once delivery.
- **Event Store:** o evento E o estado (Event Sourcing). Publicacao e persistencia sao a mesma coisa.

**Versionamento de eventos:**
- Necessario desde o dia 1. Eventos sao contratos — consumidores dependem de seu schema.
- Estrategias: schema evolution (adicionar campos opcionais), upcasting (converter versao antiga
  para nova no momento da leitura), versioned event types (`PedidoRealizadoV2`)
- Nunca remover campos de eventos publicados — apenas depreciar e adicionar novos

**Domain Events como mecanismo de integracao:**
- Entre Aggregates do mesmo Bounded Context: ideal para eventual consistency (Regra 4)
- Entre Bounded Contexts: mecanismo primario de integracao assincrona
- O evento publicado e parte da Published Language do contexto
- Consumidores em outros contextos devem ter ACL para traduzir eventos

### 4. Application Services vs Domain Services

Vernon faz uma distincao rigida entre os dois tipos de servico.

**Application Service:**
- Orquestra o fluxo de um use case
- Gerencia transacoes (begin, commit, rollback)
- Aplica seguranca e autorizacao
- Converte DTOs de entrada para chamadas ao dominio
- Converte resultado do dominio para DTOs de saida
- **NAO contem logica de negocio. Zero. Nenhuma.**
- Se um Application Service tem `if (pedido.getTotal() > limite)`, logica vazou
- O Application Service e como um maestro: coordena, mas nao toca nenhum instrumento

**Padrao tipico de Application Service:**
```
1. Receber command/DTO de entrada
2. Carregar Aggregate(s) do Repository
3. Chamar metodo do Aggregate (que executa logica de negocio)
4. Salvar Aggregate no Repository
5. Publicar Domain Events
6. Retornar resultado (DTO de saida)
```

**Domain Service:**
- Encapsula logica de negocio que nao pertence a nenhum Aggregate individual
- E stateless — nao mantem estado entre chamadas
- Recebe Aggregates, Entities ou Value Objects como parametros
- Exemplos:
  - `TransferenciaBancariaService.transferir(contaOrigem, contaDestino, valor)` —
    envolve dois Aggregates, nenhum deles e dono natural da operacao
  - `CalculadoraDeImpostosService.calcular(pedido, regiao)` — regra que depende de
    informacoes externas ao Aggregate Pedido
  - `PoliticaDeDescontoService.aplicar(pedido, cliente)` — regra que cruza dois Aggregates

**Regra de ouro:** se voce pode mover a logica para dentro de um Aggregate sem criar
dependencias estranhas, mova. Domain Services sao o ultimo recurso para logica de dominio,
nao o primeiro destino.

**Red flags de que logica vazou para Application Service:**
- O Application Service tem if/else com condicoes de negocio
- O Application Service calcula valores, valida regras, ou toma decisoes de dominio
- O Application Service faz mais de 10 linhas de logica alem de load/save/publish
- Voce nao consegue testar a regra de negocio sem mockar Repository e Transaction Manager

### 5. Repository — Implementacao Pratica

Vernon detalha como implementar Repositories na pratica.

**Principios de interface:**
- Uma interface por Aggregate Root. Nunca para entidades internas ou Value Objects.
- Interface no Domain Layer. Implementacao no Infrastructure Layer.
- Nomear com o conceito de dominio: `Pedidos`, `Clientes` (ou `PedidoRepository`, `ClienteRepository`)

**Dois estilos de Repository:**

*Collection-Oriented (estilo Evans):*
- Simula uma colecao em memoria: `add(aggregate)`, `remove(aggregate)`, `findById(id)`
- O Aggregate nao "sabe" que e persistido — o Repository rastreia mudancas (Unit of Work / dirty checking)
- Combina bem com ORMs que tem change tracking (ex: JPA/Hibernate)
- `add()` registra o objeto; as mudancas sao persistidas automaticamente no commit

*Persistence-Oriented (estilo NoSQL/Event Sourcing):*
- Requer chamada explicita de save: `save(aggregate)`, `findById(id)`
- Cada `save()` substitui ou persiste o estado completo do Aggregate
- Combina bem com document stores (MongoDB), key-value stores, Event Stores
- Mais explicito sobre quando a persistencia acontece

**Regras criticas:**
- Nunca retornar Aggregates parciais. O Repository sempre retorna o Aggregate Root
  com todo o grafo interno completamente constituido.
- Nunca expor detalhes de persistencia na interface (nao receber SQL, Session, Connection)
- Metodos de consulta devem expressar conceitos de dominio:
  `findPedidosPendentes()` em vez de `findByStatusIn(List<String> statuses)`

**Use Case Optimal Query:**
- Para leituras complexas que nao mapeiam para a estrutura do Aggregate, Vernon
  recomenda queries otimizadas por use case
- Essas queries podem retornar DTOs/projecoes diretamente, sem passar pelo Aggregate
- Sao usadas no lado de leitura (especialmente com CQRS)
- Nao violam DDD porque leitura nao precisa de invariantes

### 6. CQRS — Command Query Responsibility Segregation

Vernon apresenta CQRS como uma opcao tatica poderosa, nao como requisito.

**O que e CQRS:**
- Separar o modelo de escrita (Command) do modelo de leitura (Query)
- Command side: Aggregates, Repositories, Domain Services — logica de negocio completa
- Query side: modelos desnormalizados otimizados para leitura, sem logica de negocio
- As duas "sides" podem ter datastores diferentes

**Quando usar CQRS:**
- Read:write ratio alto (muitas leituras para poucas escritas)
- Queries complexas que nao mapeiam para a estrutura dos Aggregates
- Necessidade de views materializadas para performance
- Necessidade de escalar leitura e escrita independentemente
- Quando Event Sourcing e usado (CQRS e quase obrigatorio com ES)

**Quando NAO usar CQRS:**
- CRUD simples onde leitura e escrita tem a mesma estrutura
- Dominios com poucas queries complexas
- Times pequenos que nao podem manter dois modelos
- Quando a complexidade adicional nao traz beneficio mensuravel

**Implementacao pratica:**
- Command side: Application Service recebe Command -> carrega Aggregate -> executa -> salva
- Query side: Query Service le diretamente do read model (pode ser view SQL, Elasticsearch, cache)
- Sincronizacao: Domain Events atualizam o read model (eventual consistency)
- O read model e descartavel — pode ser reconstruido a partir dos eventos ou do command store

**Cuidados:**
- Eventual consistency entre command e query side: o usuario pode fazer uma acao e nao ver
  o resultado imediatamente. Mitigacoes: redirect-after-post, optimistic UI, polling
- Manter dois modelos tem custo de desenvolvimento e operacional
- Nao comece com CQRS — comece simples e migre quando a necessidade surgir

### 7. Event Sourcing

Vernon apresenta Event Sourcing como complemento opcional, com beneficios e custos claros.

**O que e Event Sourcing:**
- Em vez de armazenar o estado atual do Aggregate, armazena a sequencia de eventos que
  levaram ao estado atual
- O estado e reconstruido "replaying" (reproduzindo) os eventos do inicio ao fim
- O Event Store e append-only — eventos nunca sao modificados ou deletados

**Beneficios:**
- Audit trail completo: cada mudanca e um evento registrado
- Temporal queries: reconstruir o estado do sistema em qualquer ponto no tempo
- Debugging: reproduzir a sequencia exata de eventos que levou a um bug
- Event-driven naturalmente: eventos ja existem, prontos para publicacao
- Nao ha impedance mismatch ORM — eventos sao a persistencia

**Custos reais:**
- Complexidade de implementacao significativamente maior
- Event schema evolution: mudar a estrutura de eventos antigos e doloroso
- Snapshots: reconstruir Aggregates com milhares de eventos e lento sem snapshots
- Queries: impossivel fazer queries SQL tradicionais no Event Store (por isso CQRS e quase obrigatorio)
- Mindset diferente: desenvolvedores precisam pensar em termos de eventos, nao de estado
- Eventual deletion/GDPR: apagar dados de eventos append-only e complexo (crypto-shredding)

**Quando usar:**
- Dominios onde auditoria completa e requisito (financeiro, saude, legal)
- Dominios com logica temporal complexa
- Quando CQRS ja esta em uso e os beneficios justificam
- Quando event-driven architecture ja e o padrao do sistema

**Quando NAO usar:**
- CRUD simples
- Quando a equipe nao tem experiencia com o pattern
- Quando nao ha necessidade de auditoria ou temporal queries
- Como primeira escolha "porque e elegante" — Event Sourcing e uma decisao de trade-off

**Event Sourcing NAO e requisito para DDD.** Vernon e explicito: DDD funciona perfeitamente
com persistencia tradicional (relacional, document store, etc.).

### 8. Integracao entre Bounded Contexts

Vernon detalha como implementar integracoes na pratica.

**Mecanismos de integracao (do mais ao menos desejavel):**

1. **Messaging (assincrono via Domain Events):**
   - Mecanismo primario recomendado por Vernon
   - Publish/Subscribe via broker (RabbitMQ, Kafka, SQS, etc.)
   - Desacoplamento temporal: contextos nao precisam estar online simultaneamente
   - Idempotencia no consumidor e obrigatoria (mensagens podem ser entregues mais de uma vez)
   - Ordenacao de mensagens deve ser tratada (sequence number, idempotent consumer)

2. **RESTful HTTP (sincrono):**
   - Para integracoes request/response
   - Open Host Service com Published Language (JSON Schema versionado)
   - Downstream deve implementar ACL para traduzir DTOs REST em objetos do dominio interno
   - Circuit breaker e retry sao obrigatorios para resiliencia

3. **Shared Database (anti-pattern):**
   - Dois contextos acessando as mesmas tabelas
   - Cria acoplamento forte: mudancas no schema quebram ambos os contextos
   - Impossibilita evolucao independente
   - Vernon classifica explicitamente como anti-pattern a ser evitado

**Anti-Corruption Layer — implementacao:**
- **Adapter:** recebe dados do contexto externo no formato externo
- **Translator:** converte dados do formato externo para objetos do dominio interno
- **Facade (opcional):** simplifica a interface do sistema externo para o dominio interno
- A ACL vive no Infrastructure Layer do contexto downstream
- Testes da ACL devem verificar que a traducao produz objetos de dominio validos

**Published Language:**
- Schema versionado (JSON Schema, Protobuf, Avro) que define o contrato de integracao
- Versioning: major.minor. Major = breaking change; minor = backwards-compatible
- O Upstream e dono do schema; o Downstream se adapta
- Schemas devem estar em repositorio compartilhado ou schema registry

### 9. Specification Pattern — Uso Pratico

Vernon expande o Specification de Evans com orientacoes de implementacao.

**Usos do Specification:**
- **Validacao:** `PedidoValido.isSatisfiedBy(pedido)` — substituir validacao espalhada
- **Selecao/Filtragem:** `ClienteAdimplente` para filtrar listas de clientes
- **Construcao de queries:** Specification pode ser traduzida para SQL/criteria pelo Repository
- **Politicas de negocio:** combinar Specifications com AND/OR/NOT

**Composicao:**
```
elegivelParaPromocao = clienteAdimplente
    .and(comprouNoUltimoMes)
    .and(not(jaUsouPromocao))
```

**Cuidados:**
- Nao criar Specification para regras triviais (um unico if nao justifica)
- Specification deve ser nomeada com linguagem do dominio
- A mesma Specification pode ser usada para validar E para consultar

### 10. Double Dispatch e Operacoes Multi-Aggregate

Quando uma operacao envolve logica que depende de multiplos Aggregates:

**Double Dispatch:**
- O Aggregate A recebe o Domain Service como parametro
- O metodo do Aggregate chama o Domain Service passando a si mesmo
- O Domain Service acessa dados de ambos os Aggregates para executar a logica
- Isso preserva o encapsulamento: o Aggregate continua como ponto de entrada

**Exemplo conceitual:**
- `pedido.aplicarDesconto(politicaDeDescontoService)`
- Internamente: `politicaDeDescontoService.calcular(this)` acessa dados do pedido
  e dados externos (cliente, promocoes ativas) para calcular o desconto
- O pedido recebe o resultado e atualiza seu estado

**Alternativa: Domain Events**
- Se a operacao pode ser eventual, use Domain Events em vez de Double Dispatch
- Double Dispatch e para quando a resposta e necessaria na mesma operacao sincrona

---

## Heuristicas Verificaveis

### Aggregate Design

1. **Todo Aggregate segue as 4 regras de Vernon.**
   Como verificar: para cada Aggregate, cheque (a) invariantes protegidas pela Root,
   (b) tamanho pequeno (idealmente Root + VOs), (c) referencias por ID para outros Aggregates,
   (d) eventual consistency cross-Aggregate via Domain Events.

2. **Nenhum Aggregate tem mais de 3-4 entidades internas.**
   Como verificar: conte o numero de Entity classes dentro de cada Aggregate. Se passar de 3-4,
   avalie se todas realmente precisam de consistencia transacional com a Root.

3. **Aggregates referenciam outros Aggregates por ID, nunca por objeto.**
   Como verificar: busque campos em Aggregates cujo tipo e outro Aggregate Root ou Entity de
   outro Aggregate. Devem ser `XxxId`, nao `Xxx`.

4. **Cada transacao modifica exatamente um Aggregate.**
   Como verificar: revise Application Services. Se uma transacao carrega e salva mais de um
   Aggregate, os limites estao errados ou falta eventual consistency.

5. **Aggregates pequenos sao a regra; Aggregates grandes sao excecao justificada.**
   Como verificar: para cada Aggregate com mais de 1 Entity interna, deve existir uma invariante
   de negocio documentada que justifique o tamanho. Sem justificativa = candidato a split.

6. **A Aggregate Root e o unico ponto de modificacao do Aggregate.**
   Como verificar: entidades internas nao devem ter metodos publicos que modificam estado
   sem passar pela Root. Se `PedidoItem` tem `setPreco()` publico, o encapsulamento esta quebrado.

7. **Value Objects sao usados no lugar de entidades internas sempre que possivel.**
   Como verificar: se uma "entidade interna" nao precisa de identidade propria (nao e
   referenciada individualmente de fora), ela deveria ser Value Object.

### Domain Events

8. **Todo Domain Event e nomeado no passado e e imutavel.**
   Como verificar: revise nomes de eventos. `RealizarPedido` esta errado; `PedidoRealizado` esta
   correto. Verifique que a classe do evento nao tem setters.

9. **Domain Events carregam dados suficientes para acao independente.**
   Como verificar: o consumidor do evento deve conseguir agir sem fazer callback para a origem.
   Se o handler precisa chamar uma API para obter dados que deveriam estar no evento, o evento
   esta pobre.

10. **Eventos sao publicados apos mudanca de estado, na mesma unidade de trabalho.**
    Como verificar: verifique que o evento e criado dentro do metodo do Aggregate que muda estado.
    Se o evento e criado no Application Service separadamente, ha risco de inconsistencia.

11. **Existe estrategia de versionamento de eventos desde o dia 1.**
    Como verificar: verifique se eventos tem campo de versao ou se ha convencao de naming
    versionado. Se nao ha nenhuma estrategia, a evolucao futura sera dolorosa.

12. **Consumidores de eventos sao idempotentes.**
    Como verificar: revise handlers de eventos. Devem tratar duplicatas graciosamente
    (check-and-skip, idempotent write). Se processar o mesmo evento 2x causa side effects
    duplicados, ha um bug.

### Application Service vs Domain Service

13. **Application Services nao contem if/else com logica de negocio.**
    Como verificar: revise Application Services linha por linha. Qualquer condicional que
    expressa regra de dominio (`if (saldo < valor)`, `if (pedido.expirado())`) deve estar
    no Aggregate ou Domain Service, nao aqui.

14. **Application Services seguem o padrao load-execute-save.**
    Como verificar: o fluxo tipico deve ser: receber command -> carregar Aggregate do Repository
    -> chamar metodo do Aggregate -> salvar -> publicar eventos -> retornar. Desvios indicam
    logica de dominio vazando.

15. **Domain Services sao stateless.**
    Como verificar: Domain Services nao devem ter campos de instancia mutaveis. Se tem estado,
    provavelmente e um Aggregate ou Entity disfarçado.

16. **Domain Services existem apenas quando a logica nao cabe em nenhum Aggregate.**
    Como verificar: para cada Domain Service, pergunte "essa logica poderia estar em um Aggregate?"
    Se sim, mova. Domain Service e ultimo recurso, nao primeiro destino.

### Repository

17. **Existe exatamente um Repository por Aggregate Root.**
    Como verificar: conte Repositories e Aggregate Roots. Numeros devem bater. Repository para
    entidade interna e violacao.

18. **Interface do Repository no Domain Layer, implementacao no Infrastructure Layer.**
    Como verificar: a interface nao deve importar pacotes de framework/ORM. A implementacao
    deve estar em pacote de infraestrutura.

19. **Repository nunca retorna Aggregate parcial.**
    Como verificar: todo metodo find* retorna o Aggregate Root completo com grafo interno
    inteiro. Se retorna projecao parcial, deve ser um Query Service (CQRS), nao Repository.

20. **Metodos de consulta expressam conceitos do dominio.**
    Como verificar: `findPedidosAtrasados()` e melhor que `findByStatusAndDataMenorQue()`.
    Nomes de metodos devem usar Ubiquitous Language.

### CQRS e Event Sourcing

21. **CQRS e usado apenas quando read/write patterns divergem significativamente.**
    Como verificar: se o read model e identico ao write model (mesmo schema, mesmas queries),
    CQRS esta adicionando complexidade sem valor. Justifique a separacao.

22. **Read models sao descartaveis e reconstruiveis.**
    Como verificar: deve ser possivel deletar o read model e reconstrui-lo a partir dos eventos
    ou do command store. Se nao e possivel, o read model se tornou source of truth — perigo.

23. **Event Sourcing tem estrategia de snapshots para Aggregates de alta atividade.**
    Como verificar: se o Aggregate pode ter centenas+ de eventos, reconstruir do zero sera lento.
    Verifique que existe mecanismo de snapshot periodico.

### Integracao

24. **Integracoes com contextos externos usam Anti-Corruption Layer.**
    Como verificar: DTOs de APIs externas nunca devem aparecer no Domain Layer. Deve existir
    Adapter + Translator que converte para tipos do dominio interno.

25. **Shared Database entre Bounded Contexts nao existe.**
    Como verificar: cada Bounded Context deve ter seu proprio datastore (ou ao minimo, schema
    separado). Se dois contextos fazem JOIN nas mesmas tabelas, ha acoplamento.

26. **Consumers de mensagens entre contextos sao idempotentes e tolerantes a falha.**
    Como verificar: revise handlers de mensagens. Devem tratar duplicatas, mensagens fora de
    ordem, e falhas temporarias com retry + dead letter queue.

27. **Published Language tem schema versionado.**
    Como verificar: eventos/DTOs de integracao devem ter versao explicita. Mudancas breaking
    devem incrementar a major version. Schemas devem estar documentados.

### Tactical Patterns

28. **Specification pattern e usado para regras de negocio reutilizaveis.**
    Como verificar: se a mesma condicao de negocio aparece em 2+ lugares, deve ser uma
    Specification nomeada com linguagem do dominio.

29. **Double Dispatch e usado quando operacao sincrona envolve multiplos Aggregates.**
    Como verificar: se um Application Service extrai dados de dois Aggregates e aplica logica
    sobre eles, a logica deveria estar em um Domain Service invocado via Double Dispatch.

30. **Use Case Optimal Query e usado para leituras complexas.**
    Como verificar: se um Repository tem metodos de query que retornam projecoes complexas
    para telas especificas, considere migrar para Query Service dedicado (CQRS query side).

---

## Anti-patterns com Sinais no Codigo

| Anti-pattern | Sinal no Codigo | Severidade | Acao |
|---|---|---|---|
| **God Aggregate** | Aggregate com 5+ entidades internas, transacoes > 500ms, OptimisticLockException frequente | CRITICA | Aplicar Regra 2 de Vernon: dividir em Aggregates menores, usar Domain Events para eventual consistency |
| **Referencia de objeto entre Aggregates** | `private OutroAggregate outro` em vez de `private OutroAggregateId outroId` | CRITICA | Aplicar Regra 3 de Vernon: substituir por referencia por ID |
| **Transacao cross-Aggregate** | Application Service que carrega e salva 2+ Aggregates na mesma transacao | CRITICA | Aplicar Regra 4: usar Domain Events para eventual consistency entre Aggregates |
| **Business logic em Application Service** | if/else com regras de negocio, calculos de valores, validacoes de dominio em Application Service | CRITICA | Mover para Aggregate, Domain Service ou Specification |
| **Anemic Domain Model** | Aggregates com apenas getters/setters, toda logica em Services | CRITICA | Mover comportamento de volta para os Aggregates |
| **Domain Event mutavel** | Eventos com setters ou campos nao-final | ALTA | Tornar eventos imutaveis (final fields, sem setters, construtor completo) |
| **Domain Event sem dados suficientes** | Handler de evento que faz callback para obter dados que o evento deveria ter | ALTA | Enriquecer o evento com dados necessarios para acao independente |
| **Sem versionamento de eventos** | Eventos sem campo de versao, sem estrategia de evolucao de schema | ALTA | Implementar versionamento (campo version, naming versionado, upcasters) |
| **Shared Database entre Bounded Contexts** | Dois contextos fazendo queries nas mesmas tabelas | ALTA | Separar datastores; usar messaging para sincronizacao |
| **Repository para entidade interna** | `PedidoItemRepository`, `LinhaDeEnderecoRepository` | ALTA | Remover; acessar via Aggregate Root |
| **Consumer nao idempotente** | Handler de mensagem que gera side effects duplicados ao receber mensagem repetida | ALTA | Implementar idempotent consumer (dedup table, idempotent write) |
| **CQRS sem necessidade** | Read model identico ao write model; mesma estrutura, mesmas queries | MEDIA | Remover CQRS; usar Repository diretamente para leitura |
| **Event Sourcing sem snapshots** | Aggregates com centenas de eventos, reconstrucao lenta | MEDIA | Implementar snapshots periodicos |
| **ACL ausente** | DTOs de API externa usados diretamente no Domain Layer | MEDIA | Implementar Anti-Corruption Layer (Adapter + Translator) |
| **Domain Service com estado** | Domain Service com campos de instancia mutaveis | MEDIA | Refatorar para stateless; estado deve estar em Aggregates |
| **Application Service gordo** | Application Service com 50+ linhas de logica alem de load/save/publish | MEDIA | Extrair logica para Domain Service ou Aggregate |
| **Aggregate parcial retornado por Repository** | Repository com metodos que retornam entidades internas ou projecoes parciais | MEDIA | Repository retorna Aggregate completo; para projecoes, usar Query Service |
| **Invariante cross-Aggregate com consistencia forte** | Regra de negocio que exige lock em 2+ Aggregates simultaneamente | MEDIA | Questionar se a consistencia forte e realmente necessaria; refatorar para eventual |

---

## Checklist de Revisao

### Aggregate Design (4 Regras de Vernon)

- [ ] Regra 1: Invariantes de negocio estao protegidas dentro do limite do Aggregate?
- [ ] Regra 2: O Aggregate e pequeno? (Idealmente Root + Value Objects, max 3-4 entidades internas)
- [ ] Regra 3: Outros Aggregates sao referenciados por ID, nunca por objeto?
- [ ] Regra 4: Consistencia cross-Aggregate e eventual, via Domain Events?
- [ ] A Aggregate Root e o unico ponto de entrada para modificacoes?
- [ ] Entidades internas que nao precisam de identidade sao Value Objects?
- [ ] O Aggregate cabe em uma unica transacao curta (< 200-500ms)?
- [ ] Conflitos de concorrencia (OptimisticLock) sao raros?

### Domain Events

- [ ] Eventos sao nomeados no passado? (`PedidoRealizado`, nao `RealizarPedido`)
- [ ] Eventos sao imutaveis? (sem setters, campos final)
- [ ] Eventos carregam dados suficientes para acao independente do consumidor?
- [ ] Existe estrategia de versionamento de eventos?
- [ ] Eventos sao publicados na mesma unidade de trabalho que a mudanca de estado?
- [ ] Consumidores sao idempotentes?
- [ ] Existe dead letter queue para eventos que falharam no processamento?

### Application Service vs Domain Service

- [ ] Application Service tem ZERO logica de negocio?
- [ ] Application Service segue padrao load-execute-save-publish?
- [ ] Domain Services sao stateless?
- [ ] Domain Services existem apenas para logica que nao cabe em Aggregates?
- [ ] Nao ha if/else de negocio em Application Services?

### Repository

- [ ] Um Repository por Aggregate Root? (e somente para Aggregate Roots)
- [ ] Interface no dominio, implementacao na infraestrutura?
- [ ] Retorna Aggregates completos, nunca parciais?
- [ ] Metodos de consulta usam linguagem do dominio?
- [ ] Detalhes de persistencia nao vazam para a interface?

### CQRS (se aplicavel)

- [ ] A separacao command/query e justificada por patterns de uso distintos?
- [ ] Read models sao atualizados via Domain Events?
- [ ] Read models sao reconstruiveis?
- [ ] Eventual consistency entre command e query side e tratada na UX?
- [ ] Read models sao otimizados por use case (Use Case Optimal Query)?

### Event Sourcing (se aplicavel)

- [ ] Existe mecanismo de snapshots para Aggregates de alta atividade?
- [ ] Event schema evolution esta planejada?
- [ ] CQRS esta implementado para suportar queries?
- [ ] Estrategia de GDPR/data deletion esta definida?
- [ ] Existe processo de replay para reconstrucao de read models?

### Integracao entre Bounded Contexts

- [ ] Anti-Corruption Layer existe para cada integracao externa?
- [ ] Shared Database entre contextos NAO existe?
- [ ] Messaging e o mecanismo primario de integracao assincrona?
- [ ] Published Language tem schema versionado?
- [ ] Consumers sao idempotentes e tolerantes a falha?
- [ ] Circuit breaker e retry existem para integracoes sincronas?

---

## Perguntas que esta Skill Faz

1. **Quantas entidades internas tem este Aggregate?** Se tem mais de 3-4, qual invariante de
   negocio justifica o tamanho? Pode ser dividido?

2. **Este Aggregate referencia outro Aggregate por objeto ou por ID?** Se por objeto, a
   Regra 3 de Vernon esta sendo violada. Qual e o impacto no lazy loading e acoplamento?

3. **Quantos Aggregates esta transacao modificando?** Se mais de um, a Regra 4 esta sendo
   violada. O domain expert realmente exige consistencia imediata entre eles?

4. **Existe if/else de negocio neste Application Service?** Onde essa logica deveria estar?
   Em um Aggregate? Em um Domain Service? Em uma Specification?

5. **Este Domain Event e imutavel?** Tem setters? Pode ser modificado apos publicacao?
   Carrega dados suficientes para o consumidor agir sem callback?

6. **Qual e a estrategia de versionamento dos Domain Events?** Se um evento mudar de schema,
   o que acontece com consumidores existentes? Existe upcasting?

7. **Este consumer de evento e idempotente?** O que acontece se a mesma mensagem for entregue
   duas vezes? Ha side effects duplicados?

8. **Por que estamos usando CQRS aqui?** O read model e realmente diferente do write model?
   A complexidade adicional e justificada pelo beneficio?

9. **Por que estamos usando Event Sourcing?** Ha requisito de auditoria completa? Temporal
   queries? Ou e complexidade gratuita?

10. **Este Repository retorna Aggregates completos?** Ha metodos que retornam entidades internas
    ou projecoes parciais? Isso deveria ser um Query Service?

11. **Existe Anti-Corruption Layer para esta integracao externa?** DTOs de APIs externas estao
    sendo usados no dominio? O dominio esta sendo "contaminado" por modelos externos?

12. **Dois Bounded Contexts compartilham banco de dados?** Se sim, como evoluir schemas
    independentemente? Como garantir que mudancas em um nao quebram o outro?

13. **Este Domain Service e realmente necessario?** A logica poderia estar em um Aggregate?
    O Domain Service e stateless? Ele existe porque a logica genuinamente cruza Aggregates?

14. **Qual e o tempo medio de transacao deste Aggregate?** Se e > 500ms, o Aggregate esta
    grande demais ou faz I/O que nao deveria.

15. **Qual e a taxa de OptimisticLockException neste Aggregate?** Se e alta, o Aggregate esta
    grande demais (muitos usuarios modificando a mesma instancia concorrentemente).

16. **Estamos usando eventual consistency onde o domain expert exige consistencia forte?**
    Ou o contrario: estamos forçando transacoes distribuidas onde eventual bastaria?

17. **Como este sistema se comporta se o message broker ficar indisponivel por 5 minutos?**
    Ha fallback? Dead letter queue? Os eventos sao persistidos antes de publicar (outbox)?

18. **Se deletarmos o read model, conseguimos reconstrui-lo?** Se nao, o read model se tornou
    source of truth acidentalmente.

---

## Vocabulario

| Termo | Definicao |
|---|---|
| **Aggregate** | Cluster de Entities e Value Objects com uma Root que protege invariantes de negocio dentro de uma unica transacao |
| **Aggregate Root** | A Entity que serve como unico ponto de entrada para leitura e modificacao de um Aggregate |
| **Aggregate Design Rules (Vernon)** | As 4 regras: proteger invariantes, desenhar pequeno, referenciar por ID, eventual consistency cross-Aggregate |
| **Invariante de Negocio** | Regra que deve ser verdadeira a todo momento dentro do limite do Aggregate |
| **Referencia por Identidade** | Referenciar outro Aggregate pelo seu ID em vez de segurar referencia ao objeto completo |
| **Eventual Consistency** | Modelo de consistencia onde diferentes partes do sistema convergem para um estado consistente ao longo do tempo, nao instantaneamente |
| **Domain Event** | Registro imutavel de algo que aconteceu no dominio, nomeado no passado (ex: `PedidoRealizado`) |
| **Outbox Pattern** | Padrao onde o Domain Event e salvo na mesma transacao que a mudanca de estado, e publicado assincronamente por processo separado |
| **Event Store** | Armazenamento append-only de Domain Events, usado em Event Sourcing como fonte primaria de verdade |
| **Application Service** | Servico que orquestra um use case: recebe command, carrega Aggregate, delega ao dominio, salva, publica eventos. Zero logica de negocio |
| **Domain Service** | Servico stateless que encapsula logica de negocio que nao pertence a nenhum Aggregate individual |
| **Command** | Objeto que representa a intencao de executar uma acao no sistema (ex: `RealizarPedidoCommand`) |
| **Query** | Solicitacao de leitura de dados, sem efeitos colaterais no estado do sistema |
| **CQRS** | Command Query Responsibility Segregation — separacao do modelo de escrita (Aggregates) do modelo de leitura (read models desnormalizados) |
| **Event Sourcing** | Padrao onde o estado do Aggregate e reconstruido pela reproducao sequencial de seus Domain Events, em vez de ser armazenado diretamente |
| **Snapshot** | Captura periodica do estado do Aggregate em Event Sourcing para evitar replay de todos os eventos desde o inicio |
| **Read Model** | Modelo desnormalizado otimizado para leitura em CQRS, atualizado por Domain Events |
| **Use Case Optimal Query** | Query otimizada para um use case especifico, retornando exatamente os dados necessarios para aquela tela/operacao |
| **Anti-Corruption Layer (ACL)** | Camada de traducao (Adapter + Translator + Facade) que protege o dominio interno de modelos externos |
| **Published Language** | Schema versionado (JSON Schema, Protobuf, Avro) que define o contrato de integracao entre Bounded Contexts |
| **Idempotent Consumer** | Consumidor de mensagens que produz o mesmo resultado independentemente de quantas vezes a mesma mensagem for processada |
| **Dead Letter Queue (DLQ)** | Fila para mensagens que falharam no processamento apos N tentativas, para analise e reprocessamento manual |
| **Circuit Breaker** | Padrao que interrompe chamadas a um servico externo que esta falhando, prevenindo cascata de falhas |
| **Specification** | Objeto que encapsula uma regra de negocio como predicado testavel (`isSatisfiedBy`), combinavel com AND/OR/NOT |
| **Double Dispatch** | Tecnica onde o Aggregate recebe um Domain Service como parametro e delega parte da logica a ele, preservando encapsulamento |
| **Bounded Context** | Limite explicito dentro do qual um modelo de dominio e sua linguagem se aplicam, alinhado com autonomia de time e deploy |
| **Context Map** | Mapa de relacionamentos entre Bounded Contexts: Shared Kernel, Customer/Supplier, Conformist, ACL, etc. |
| **Collection-Oriented Repository** | Estilo de Repository que simula colecao em memoria (add/remove/find), com change tracking automatico |
| **Persistence-Oriented Repository** | Estilo de Repository com save explicito, adequado para document stores e Event Stores |
| **Optimistic Locking** | Estrategia de concorrencia que detecta conflitos no momento do commit, usando version number no Aggregate |
| **Crypto-Shredding** | Tecnica para GDPR em Event Sourcing: dados pessoais sao criptografados com chave por usuario; deletar a chave torna os dados irrecuperaveis |
| **Upcasting** | Tecnica de Event Sourcing onde eventos de versoes antigas sao convertidos para a versao mais recente no momento da leitura |
| **Schema Registry** | Servico centralizado que armazena e versiona schemas de eventos/mensagens, garantindo compatibilidade entre producers e consumers |
