# Skill: Patterns of Enterprise Application Architecture — Martin Fowler

## Perspectiva do Autor

Martin Fowler escreve a partir de décadas de consultoria em sistemas corporativos de grande escala.
Sua visão de mundo é pragmática e orientada a trade-offs: não existe solução universalmente correta,
apenas soluções adequadas a um contexto específico de complexidade, equipe e requisitos de negócio.

Prioridades centrais de Fowler:

1. **Separação de responsabilidades em camadas** — a arquitetura deve organizar código em camadas
   com direção de dependência controlada. A camada de domínio nunca depende da apresentação.
2. **Complexidade proporcional** — a sofisticação da solução técnica deve ser proporcional à
   complexidade do problema de negócio. Usar Domain Model para um CRUD simples é overengineering;
   usar Transaction Script para regras de negócio intrincadas é suicídio de manutenção.
3. **Distribuição como último recurso** — a primeira lei de objetos distribuídos é "não distribua".
   Performance local com clustering quase sempre vence performance distribuída com latência de rede.
4. **Patterns como vocabulário compartilhado** — patterns não são receitas; são nomes para soluções
   recorrentes que permitem comunicação precisa entre desenvolvedores e arquitetos.
5. **Decisões em cascata** — as escolhas arquiteturais se encadeiam: a decisão sobre domain logic
   influencia a escolha de data source, que influencia a apresentação, que influencia a concorrência.

Fowler é cético em relação a frameworks que prometem resolver tudo e defende que o arquiteto deve
entender os mecanismos subjacentes antes de adotar qualquer ferramenta. Ele valoriza simplicidade,
testabilidade e a capacidade de mudar decisões quando o contexto muda.

---

## Princípios Fundamentais

### 1. As Três Camadas Principais

Toda aplicação enterprise pode ser decomposta em três camadas lógicas:

| Camada | Responsabilidade | Exemplos |
|--------|-----------------|----------|
| **Presentation** | Exibição de informação e interpretação de comandos do usuário | HTML, CLI, REST controllers, UI components |
| **Domain (Business Logic)** | Regras de negócio, validações, cálculos, workflows | Entities, services, policies, strategies |
| **Data Source** | Comunicação com bancos de dados, messaging, sistemas externos | Repositories, gateways, mappers, adapters |

**Regra de ouro:** dependências fluem para baixo (Presentation -> Domain -> Data Source), mas
**o Domain nunca depende de Presentation**. Se o Domain precisa notificar a UI, use Observer,
eventos ou inversão de dependência.

### 2. Layer vs. Tier

Estes termos NÃO são sinônimos:

- **Layer** = separação lógica dentro do código (pacotes, módulos, namespaces)
- **Tier** = separação física em processos/máquinas diferentes (client, app server, database server)

Um sistema pode ter 3 layers rodando em 1 tier (monólito), ou 2 layers distribuídas em 3 tiers.
A confusão entre esses conceitos leva a distribuição prematura — que é a causa número um de
problemas de performance em sistemas enterprise, segundo Fowler.

### 3. Complexity Boosters (Jens Coldewey)

Fatores que aumentam a complexidade real de um sistema e justificam soluções mais sofisticadas:

- **Volume de regras de negócio** — quanto mais regras interagem entre si, mais se justifica Domain Model
- **Variabilidade de comportamento** — polimorfismo no domínio (diferentes tipos de produto, diferentes
  regras por região) empurra para Domain Model ou Strategy patterns
- **Frequência de mudança** — domínios que mudam semanalmente precisam de isolamento forte
- **Número de integrações** — cada sistema externo adiciona complexidade na camada de Data Source
- **Requisitos de concorrência** — transações longas, múltiplos usuários editando o mesmo dado
- **Requisitos de distribuição** — quando realmente precisa distribuir (regulatório, latência geográfica)

### 4. Domain Logic Patterns

#### 4.1 Transaction Script

- Um procedimento por caso de uso/transação de negócio
- Cada script opera do início ao fim, coordenando leitura, lógica e persistência
- Simples, direto, fácil de entender isoladamente
- **Quando usar:** domínio simples, poucas regras, equipe junior, protótipos
- **Risco:** duplicação de lógica entre scripts quando a complexidade cresce

#### 4.2 Domain Model

- Objetos que representam entidades e conceitos de negócio com dados E comportamento
- Relações entre objetos modelam relações do domínio
- Estratégias, polimorfismo, herança para capturar variações de comportamento
- **Quando usar:** domínio complexo, muitas regras que interagem, equipe experiente em OO
- **Risco:** curva de aprendizado, overhead para problemas simples

#### 4.3 Table Module

- Um objeto por tabela do banco de dados, mas que opera sobre conjuntos (RecordSets)
- Meio-termo entre Transaction Script e Domain Model
- Funciona bem com tecnologias centradas em datasets (ex: .NET DataSet, ferramentas RAD)
- **Quando usar:** domínio de complexidade média, ferramentas que favorecem RecordSets
- **Risco:** perde expressividade quando o domínio diverge do schema relacional

#### 4.4 Service Layer

- Camada fina sobre o domínio que define a fronteira da aplicação
- Coordena casos de uso, segurança, transações, orquestração
- NÃO contém regras de negócio — delega ao Domain Model ou Transaction Scripts
- **Quando usar:** quase sempre que existe uma API exposta (REST, gRPC, messaging)
- **Risco:** se a Service Layer engorda, virou Transaction Script disfarçado

#### Curva Complexidade-Esforço

```
Esforço de
Manutenção
    ^
    |         Transaction Script
    |        /
    |       /        Table Module
    |      /        /
    |     /        /       Domain Model
    |    /        /       /
    |   /        /       /
    |  /--------/-------/----------->
    | /    ****       **
    |/ **      **** **
    +--*-----------*-----------------> Complexidade do Domínio
       ^           ^
       |           |
    Ponto onde     Ponto onde Domain Model
    Transaction    começa a compensar
    Script perde
```

**Heurística:** se você tem menos de 5-6 regras de negócio que não interagem entre si,
Transaction Script basta. Se regras se cruzam, condições variam por tipo de entidade,
e o domínio muda frequentemente, Domain Model é o investimento correto.

### 5. Data Source Patterns

#### 5.1 Table Data Gateway

- Um objeto por tabela, com métodos CRUD que retornam RecordSets
- A aplicação trabalha com dados tabulares, não com objetos de domínio
- **Pairing natural:** Transaction Script, Table Module

#### 5.2 Row Data Gateway

- Um objeto por linha do banco de dados, com métodos para load/save
- O objeto contém apenas dados e acesso ao banco, SEM lógica de negócio
- **Pairing natural:** Transaction Script (quando quer mais estrutura que Table Data Gateway)

#### 5.3 Active Record

- Como Row Data Gateway, mas COM lógica de domínio no mesmo objeto
- O objeto conhece como se persistir E contém regras de negócio
- **Pairing natural:** domínio simples onde a estrutura do banco espelha o domínio
- **Risco:** quando o domínio diverge do schema, Active Record vira um monstro

#### 5.4 Data Mapper

- Camada de mapeamento totalmente separada entre objetos de domínio e banco de dados
- O Domain Model não sabe que banco de dados existe
- **Pairing natural:** Domain Model complexo
- **Risco:** complexidade de implementação (ou use um ORM maduro)

#### Tabela de Pairing: Domain Logic -> Data Source

| Domain Logic Pattern | Data Source Recomendado | Alternativa | Evitar |
|----------------------|------------------------|-------------|--------|
| Transaction Script | Table Data Gateway | Row Data Gateway | Data Mapper (overkill) |
| Table Module | Table Data Gateway | — | Active Record, Data Mapper |
| Domain Model (simples) | Active Record | Row Data Gateway | — |
| Domain Model (complexo) | Data Mapper | — | Active Record (acoplamento) |
| Service Layer + Domain Model | Data Mapper | — | Table Data Gateway |

### 6. O/R Behavioral Patterns

#### 6.1 Unit of Work

- Mantém lista de objetos afetados por uma transação de negócio
- No commit, coordena a escrita no banco na ordem correta
- Resolve o problema de "quando exatamente salvar no banco?"
- **Implementações:** Hibernate Session, Entity Framework DbContext, JPA EntityManager

#### 6.2 Identity Map

- Cache de objetos já carregados na sessão/transação atual
- Garante que duas leituras da mesma entidade retornam o MESMO objeto (identidade referencial)
- Evita inconsistências e economiza queries
- **Cuidado:** escopo deve ser a transação/request, NUNCA global (memory leak + stale data)

#### 6.3 Lazy Load

- Adia o carregamento de dados relacionados até o momento do acesso
- Técnicas: virtual proxy, value holder, ghost, lazy initialization
- **Perigo clássico:** N+1 queries — lazy load dentro de um loop gera uma query por iteração
- **Regra:** use Lazy Load por padrão, mas identifique e resolva N+1 com eager loading seletivo

### 7. O/R Structural Patterns

#### 7.1 Identity Field

- Campo no objeto que corresponde à chave primária no banco
- Escolhas: chave natural vs. surrogate key (auto-increment, UUID)
- **Recomendação de Fowler:** surrogate keys na maioria dos casos — chaves naturais mudam

#### 7.2 Foreign Key Mapping

- Mapeia referências entre objetos para chaves estrangeiras no banco
- Relações 1:N e N:1 mapeiam diretamente
- Relações N:M precisam de tabela associativa

#### 7.3 Embedded Value

- Mapeia um Value Object para colunas da tabela da entidade proprietária
- Ex: `Address` (rua, cidade, CEP) como colunas na tabela `Customer`
- **Quando usar:** value objects pequenos, sempre carregados junto

#### 7.4 Serialized LOB (Large Object)

- Serializa um grafo de objetos em um campo BLOB/CLOB (XML, JSON, binário)
- **Vantagem:** simplicidade para estruturas complexas que não precisam de query
- **Desvantagem:** perda total de capacidade de query sobre os dados internos
- **Regra:** se você precisa fazer WHERE em campos internos, NÃO use Serialized LOB

### 8. Inheritance Mapping

| Estratégia | Tabelas | Prós | Contras |
|-----------|---------|------|---------|
| **Single Table Inheritance** | 1 tabela para toda a hierarquia | Sem joins, simples | Muitos NULLs, tabela enorme |
| **Class Table Inheritance** | 1 tabela por classe na hierarquia | Normalizado, sem NULLs | Joins em toda leitura |
| **Concrete Table Inheritance** | 1 tabela por classe concreta | Sem NULLs, sem joins para tipo específico | Mudança no pai = alterar todas as tabelas |

**Recomendação:** comece com Single Table Inheritance. Mude para Class Table se os NULLs ficarem
inaceitáveis ou a tabela exceder milhões de registros com muitas colunas vazias.

### 9. Web Presentation Patterns

#### 9.1 Model-View-Controller (MVC)

- **Model:** Domain Model ou representação dos dados
- **View:** renderização visual (HTML, JSON, XML)
- **Controller:** recebe input, coordena Model e View
- Princípio central: separação entre Model e View permite múltiplas apresentações

#### 9.2 Page Controller

- Um controller por página/endpoint
- Simples, direto, fácil de mapear URL -> handler
- **Quando usar:** aplicações com mapeamento 1:1 entre URL e ação

#### 9.3 Front Controller

- Um único ponto de entrada para todas as requests
- Centraliza cross-cutting concerns: autenticação, logging, roteamento
- **Quando usar:** quando precisa de processamento uniforme antes de cada request

#### 9.4 Template View

- View com marcadores que são preenchidos com dados em tempo de renderização
- Ex: JSP, Thymeleaf, ERB, Jinja2, Blade
- **Risco:** lógica de negócio infiltrando nos templates

#### 9.5 Transform View

- Transforma o Model em output via transformação (ex: XSLT, serializers)
- Mais testável que Template View, porém menos intuitivo para designers

#### 9.6 Two Step View

- Primeiro passo: transforma dados de domínio em uma representação lógica
- Segundo passo: transforma representação lógica em output visual específico
- **Quando usar:** múltiplos canais de saída (web, mobile, PDF) a partir da mesma lógica

### 10. Distribuição

#### A Primeira Lei dos Objetos Distribuídos

> **"Não distribua seus objetos."** — Martin Fowler

Chamadas remotas são ordens de magnitude mais lentas que chamadas locais. A tentação de
distribuir "para escalar" resulta em sistemas lentos, frágeis e difíceis de debugar.

**Alternativas à distribuição:**
- **Clustering:** múltiplas instâncias do mesmo monólito atrás de um load balancer
- **Scaling vertical:** antes de distribuir, verifique se hardware melhor resolve
- **Cache:** muitas vezes, cache elimina a necessidade de distribuir

#### Quando Realmente Distribuir

- Requisitos regulatórios (dados em jurisdições específicas)
- Latência geográfica irredutível
- Times independentes com ciclos de deploy separados (microservices legítimos)

#### Remote Facade

- Interface granular (coarse-grained) para acesso remoto
- Agrupa múltiplas operações finas em uma chamada grossa
- NÃO contém lógica — apenas traduz e delega para a camada de domínio local
- **Regra:** uma chamada remota deve fazer o trabalho de 10+ chamadas locais

#### Data Transfer Object (DTO)

- Objeto serializável que transporta dados entre processos
- Sem comportamento, sem lógica — apenas dados
- **Assembler:** componente que converte entre Domain Objects e DTOs
- **Erro comum:** usar DTOs quando não há fronteira remota — overhead sem benefício

### 11. Concorrência

#### Vocabulário de Concorrência

- **Transação de negócio:** uma sequência de ações do ponto de vista do usuário
  (ex: "registrar um pedido")
- **Transação de sistema:** uma transação ACID do banco de dados
- Uma transação de negócio pode abranger múltiplas transações de sistema

#### Controle Otimista vs. Pessimista

| Aspecto | Otimista | Pessimista |
|---------|----------|------------|
| **Mecanismo** | Detecta conflito no commit (versão/timestamp) | Previne conflito com locks |
| **Throughput** | Alto (sem locks) | Menor (contenção de locks) |
| **Conflitos** | Detectados tarde (re-trabalho) | Prevenidos (bloqueio) |
| **Melhor para** | Leitura intensa, poucos conflitos | Escrita intensa, muitos conflitos |
| **Risco** | Retry storms sob alta contenção | Deadlocks, starvation |

#### ACID e Isolation Levels

| Nível | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|-------------------|-------------|
| Read Uncommitted | Sim | Sim | Sim |
| Read Committed | Não | Sim | Sim |
| Repeatable Read | Não | Não | Sim |
| Serializable | Não | Não | Não |

**Regra de Fowler:** use o nível de isolamento mais baixo que mantém a corretude.
Serializable é correto mas lento; Read Committed é o default pragmático na maioria dos bancos.

### 12. Offline Concurrency Patterns

Estes patterns tratam de transações de negócio que abrangem múltiplas transações de sistema
(ex: wizard multi-step, edição de documento com "salvar rascunho").

#### 12.1 Optimistic Offline Lock

- Valida que o dado não mudou desde a leitura no momento do commit
- Implementação: coluna de versão ou timestamp comparada no UPDATE
- `UPDATE ... WHERE id = ? AND version = ?` — se afeta 0 linhas, houve conflito

#### 12.2 Pessimistic Offline Lock

- Adquire lock no início da transação de negócio, impede acesso concorrente
- Implementação: tabela de locks ou flag "editando por"
- **Risco:** locks órfãos (usuário fecha o browser sem liberar o lock) — precisa de timeout

#### 12.3 Coarse-Grained Lock

- Um único lock cobre um agregado inteiro (ex: lock no Pedido inclui todos os Itens)
- Reduz complexidade de lock management
- **Trade-off:** reduz concorrência em entidades filhas

#### 12.4 Implicit Lock

- Framework aplica locks automaticamente sem o desenvolvedor solicitar explicitamente
- Reduz erro humano (esquecer de adquirir lock)
- **Risco:** comportamento implícito dificulta debugging

### 13. Session State

| Estratégia | Prós | Contras | Melhor para |
|-----------|------|---------|-------------|
| **Client Session State** | Sem estado no servidor, clustering trivial | Segurança (dados no client), bandwidth | APIs stateless, JWT |
| **Server Session State** | Rápido, fácil de implementar | Afinidade de sessão, failover complexo | Apps web tradicionais |
| **Database Session State** | Sem afinidade, failover automático | Latência de banco para cada acesso | Sistemas com muitos servidores |

**Recomendação moderna:** prefira Client Session State (tokens) para APIs e
Database Session State (cache como Redis) quando precisa de estado server-side.

### 14. Performance — Vocabulário Preciso

Fowler insiste em vocabulário preciso para discussões de performance:

| Termo | Definição |
|-------|----------|
| **Response time** | Tempo total para o sistema processar uma requisição externa |
| **Responsiveness** | Quão rápido o sistema reconhece a requisição (pode ser antes de completá-la) |
| **Latency** | Tempo de espera antes que o processamento comece (rede, enfileiramento) |
| **Throughput** | Quantidade de trabalho processado por unidade de tempo |
| **Load** | Quantidade de stress atual no sistema (requests/segundo, conexões simultâneas) |
| **Capacity** | Throughput máximo que o sistema suporta |
| **Scalability** | Capacidade de manter performance ao adicionar recursos |
| **Efficiency** | Performance dividida por recursos consumidos |

**Regra:** "O sistema está lento" não é um diagnóstico. Especifique: response time alto?
Throughput insuficiente? Load acima da capacity? Cada problema tem solução diferente.

### 15. Master Decision Framework — Putting It All Together

Fowler propõe um processo decisório em cascata:

#### Passo 1: Escolha o Domain Logic Pattern

```
Complexidade do domínio?
    |
    +-- Baixa (CRUD, poucas regras) ---------> Transaction Script
    |
    +-- Média (regras moderadas, algum
    |   polimorfismo, ferramentas RAD) -------> Table Module
    |
    +-- Alta (muitas regras interativas,
        polimorfismo, domínio rico) ----------> Domain Model
                                                  |
                                              Service Layer por cima
```

#### Passo 2: Escolha o Data Source Pattern

```
Domain Logic Pattern escolhido?
    |
    +-- Transaction Script ----> Table Data Gateway (ou Row Data Gateway)
    |
    +-- Table Module ----------> Table Data Gateway
    |
    +-- Domain Model (simples) -> Active Record
    |
    +-- Domain Model (complexo) -> Data Mapper (com ORM)
```

#### Passo 3: Escolha o Presentation Pattern

```
Tipo de interface?
    |
    +-- API REST/gRPC ---------> Front Controller + Transform View (serializer)
    |
    +-- Web tradicional -------> Front Controller + Template View
    |
    +-- Múltiplos canais ------> Two Step View
    |
    +-- SPA + Backend ---------> Front Controller + JSON serialization
```

#### Passo 4: Escolha a Estratégia de Concorrência

```
Transações de negócio cabem em uma transação de sistema?
    |
    +-- Sim ----> ACID com isolation level adequado (Read Committed como default)
    |
    +-- Não ----> Offline Concurrency
                    |
                    +-- Conflitos raros? -----> Optimistic Offline Lock
                    |
                    +-- Conflitos frequentes? -> Pessimistic Offline Lock
                    |
                    +-- Agregado grande? -----> Coarse-Grained Lock
```

---

## Heurísticas Verificáveis

### Domain e Layers

1. **Dependência unidirecional das camadas.** A camada de Domain NUNCA importa classes de Presentation.
   - *Como verificar:* grep por imports — nenhum arquivo em `domain/` deve importar de `controllers/`, `views/`, `routes/` ou `api/`.

2. **Service Layer fina.** Services não contêm condicionais de negócio (if/else sobre regras de domínio).
   - *Como verificar:* se um método de service tem mais de 3 condicionais de negócio, a lógica pertence ao Domain Model.

3. **Transaction Script isolado.** Cada Transaction Script opera independentemente e não chama outros scripts.
   - *Como verificar:* se Transaction Scripts chamam outros scripts, você precisa de Service Layer ou Domain Model.

4. **Active Record apenas para domínios simples.** Se a entidade tem mais de 5 métodos de negócio não-triviais, precisa de Data Mapper.
   - *Como verificar:* conte métodos que não são getters/setters/save/delete. Mais de 5? Migre para Data Mapper.

5. **DTO somente na fronteira.** DTOs não devem ser usados entre camadas internas.
   - *Como verificar:* se Domain retorna DTOs para outro objeto Domain, há acoplamento desnecessário.

### Data Source

6. **Identity Map por transação.** O Identity Map deve ter escopo de request/transação, nunca global.
   - *Como verificar:* verifique se o cache de entidades é criado/destruído a cada request. Cache global = memory leak.

7. **Lazy Load com proteção contra N+1.** Todo uso de Lazy Load deve ter análise de queries.
   - *Como verificar:* ative SQL logging em testes e verifique se loops geram N queries. Se sim, use eager loading.

8. **Unit of Work explícito.** Alterações ao banco devem ser coordenadas por um Unit of Work, não por saves individuais.
   - *Como verificar:* se há múltiplos `save()` soltos em um caso de uso, falta Unit of Work.

9. **Surrogate keys como padrão.** Use chaves naturais apenas quando há justificativa forte.
   - *Como verificar:* verifique se PKs são auto-increment ou UUID. Chaves naturais (CPF, email) como PK são sinais de alerta.

10. **Serialized LOB não-queryável.** Dados em campos JSON/BLOB nunca devem ser usados em WHERE clauses.
    - *Como verificar:* procure queries com `JSON_EXTRACT`, `->`, `->>` em campos LOB usados como filtro. Se existem, normalize.

### Presentation

11. **Zero lógica de negócio em templates.** Templates devem conter apenas lógica de apresentação (formatação, iteração simples).
    - *Como verificar:* se um template tem cálculos, validações ou condicionais de negócio, extraia para o domain.

12. **Controllers delegam, não implementam.** Controllers chamam Services/Domain; não contêm lógica de negócio.
    - *Como verificar:* se um controller tem mais de 20 linhas de lógica (excluindo parsing e response), está gordo.

13. **Front Controller para cross-cutting concerns.** Autenticação, logging, rate limiting devem ser centralizados.
    - *Como verificar:* se múltiplos controllers repetem checks de autenticação, falta middleware/front controller.

### Distribuição

14. **Chamadas remotas justificadas.** Cada fronteira remota deve ter justificativa documentada.
    - *Como verificar:* para cada microservice, pergunte: "por que isso não pode ser um módulo local?" Se não há resposta clara, consolide.

15. **Remote Facade coarse-grained.** Interfaces remotas devem ser granulares (poucos métodos, muitos dados por chamada).
    - *Como verificar:* se uma operação de negócio requer >3 chamadas remotas sequenciais, a API é fine-grained demais.

16. **DTO justificado por fronteira.** DTOs existem somente onde há serialização cross-process.
    - *Como verificar:* se DTO é usado entre módulos do mesmo processo, remova.

17. **Clustering antes de distribuição.** Tente escalar horizontalmente o monólito antes de quebrar em serviços.
    - *Como verificar:* pergunte: "testamos com 2+ instâncias do monólito antes de propor microservices?"

### Concorrência

18. **Isolation level explícito.** Toda transação que não é trivial deve declarar seu isolation level.
    - *Como verificar:* procure transações sem isolation level explícito que operam em dados com concorrência.

19. **Optimistic lock em entidades concorridas.** Entidades editadas por múltiplos usuários devem ter coluna de versão.
    - *Como verificar:* procure tabelas sem coluna `version` ou `updated_at` usadas em WHERE de UPDATE.

20. **Timeout em Pessimistic Offline Lock.** Todo lock pessimista deve ter timeout para evitar locks órfãos.
    - *Como verificar:* se há tabela de locks, deve haver coluna de expiração e job de cleanup.

21. **Transação de sistema curta.** Transações de banco não devem esperar input do usuário.
    - *Como verificar:* se uma transação SQL aberta depende de ação do usuário (ex: wizard), redesenhe para Offline Concurrency.

### Mapeamento O/R

22. **Herança mapeada consistentemente.** Todas as classes de uma hierarquia usam a mesma estratégia.
    - *Como verificar:* misturar Single Table e Class Table na mesma hierarquia gera queries inconsistentes.

23. **Embedded Value para Value Objects.** Value Objects pequenos (Money, Address, DateRange) devem ser embedded.
    - *Como verificar:* se Value Objects têm tabela própria com relação 1:1, use Embedded Value.

24. **Foreign Key Mapping explícito.** Relações N:M devem ter tabela associativa; nunca serializar IDs em array.
    - *Como verificar:* se há campo com lista de IDs separados por vírgula ou array JSON para relação N:M, normalize.

### Performance

25. **Vocabulário preciso em discussões de performance.** Equipe deve diferenciar latência, throughput e response time.
    - *Como verificar:* em documentos de arquitetura, se "lento" aparece sem qualificação, exija métricas específicas.

26. **Métricas antes de otimizações.** Nenhuma otimização deve ser feita sem medição prévia.
    - *Como verificar:* se houve mudança arquitetural "por performance" sem benchmark antes/depois, reverta a discussão.

27. **Capacity planning documentado.** O sistema deve ter projeções de load vs. capacity.
    - *Como verificar:* deve existir documento ou teste de carga que relaciona load esperado com throughput medido.

---

## Anti-patterns com Sinais no Código

| # | Anti-pattern | Sinal no Código | Severidade | Ação |
|---|-------------|-----------------|------------|------|
| 1 | **Anemic Domain Model** | Entidades com apenas getters/setters; toda lógica em Services | Alta | Mover comportamento para entidades de domínio |
| 2 | **Fat Service Layer** | Services com centenas de linhas de if/else de negócio | Alta | Extrair lógica para Domain Model; Service Layer apenas orquestra |
| 3 | **God Controller** | Controller com validação, lógica de negócio e formatação de resposta | Alta | Separar em Service + Domain + Serializer |
| 4 | **Active Record com Domain complexo** | Model que herda de ORM E tem >200 linhas de regras de negócio | Alta | Migrar para Domain Model + Data Mapper |
| 5 | **Distribuição prematura** | Microservices que se chamam sincronamente em cadeia para completar uma request | Crítica | Avaliar consolidação em monólito modular ou comunicação assíncrona |
| 6 | **Chatty Remote Interface** | 5+ chamadas HTTP/gRPC para uma única operação de negócio | Alta | Consolidar em Remote Facade com endpoint coarse-grained |
| 7 | **DTO Everywhere** | DTOs mapeando entre camadas do mesmo processo sem fronteira remota | Média | Remover DTOs internos; usar domain objects diretamente |
| 8 | **Transaction Script Spaghetti** | Scripts que chamam outros scripts com shared mutable state | Alta | Refatorar para Service Layer + Domain Model |
| 9 | **N+1 Lazy Load** | Log de SQL mostra queries repetidas dentro de loops | Alta | Substituir por eager loading ou batch loading |
| 10 | **Identity Map global** | Cache de entidades compartilhado entre requests/threads | Crítica | Escopo de request/transaction; nunca singleton global |
| 11 | **Lógica em template** | Templates com cálculos de impostos, validações, regras de desconto | Alta | Mover para domain; template recebe dados prontos |
| 12 | **Natural key como PK** | Email ou CPF como primary key, sem surrogate | Média | Adicionar surrogate key; usar natural key como unique constraint |
| 13 | **Lock sem timeout** | Tabela de locks sem coluna de expiração | Alta | Adicionar TTL e job de limpeza de locks expirados |
| 14 | **Long transaction** | `BEGIN` ... espera input do usuário ... `COMMIT` | Crítica | Redesenhar com Offline Concurrency patterns |
| 15 | **CSV no banco** | Coluna com valores separados por vírgula para relação N:M | Alta | Normalizar com tabela associativa |
| 16 | **Mixed inheritance mapping** | Parte da hierarquia em Single Table, parte em Class Table | Média | Unificar estratégia para toda a hierarquia |
| 17 | **Business logic no SQL** | Stored procedures com centenas de linhas de regras de negócio | Alta | Extrair para camada de domínio; manter SQL para acesso a dados |
| 18 | **Missing Unit of Work** | Múltiplos `repository.save()` sem coordenação transacional | Alta | Introduzir Unit of Work ou transaction scope explícito |

---

## Checklist de Revisão

### Arquitetura Geral
- [ ] As três camadas (Presentation, Domain, Data Source) estão claramente separadas em pacotes/módulos?
- [ ] A direção de dependência vai de Presentation -> Domain -> Data Source, sem ciclos?
- [ ] A complexidade da solução técnica é proporcional à complexidade do domínio?
- [ ] Existe Service Layer definindo a fronteira da aplicação?
- [ ] A Service Layer é fina (orquestração, não lógica)?

### Domain Logic
- [ ] O Domain Logic Pattern escolhido é adequado à complexidade? (Transaction Script vs Domain Model vs Table Module)
- [ ] Entidades de domínio encapsulam dados E comportamento (não são anêmicas)?
- [ ] Value Objects são usados para conceitos sem identidade (Money, DateRange, Address)?
- [ ] Regras de negócio estão no domínio, não espalhadas em controllers/services/templates?

### Data Source
- [ ] O Data Source Pattern é compatível com o Domain Logic Pattern? (Tabela de Pairing)
- [ ] Há Unit of Work coordenando persistência dentro de cada caso de uso?
- [ ] Identity Map tem escopo de request/transação?
- [ ] Lazy Load é usado conscientemente, com proteção contra N+1?
- [ ] Surrogate keys são usadas como PKs?
- [ ] Mapeamento de herança usa estratégia consistente para toda a hierarquia?

### Apresentação
- [ ] Controllers são finos (delegam para services/domain)?
- [ ] Templates não contêm lógica de negócio?
- [ ] Cross-cutting concerns são centralizados (Front Controller/middleware)?
- [ ] DTOs são usados apenas em fronteiras remotas?

### Distribuição
- [ ] Toda fronteira remota tem justificativa clara?
- [ ] Interfaces remotas são coarse-grained (Remote Facade)?
- [ ] Clustering foi considerado antes de distribuição?
- [ ] Não há chamadas remotas síncronas em cadeia?

### Concorrência
- [ ] Isolation levels são explícitos para transações não-triviais?
- [ ] Entidades concorridas têm controle otimista (coluna de versão)?
- [ ] Transações de sistema são curtas (não esperam input humano)?
- [ ] Pessimistic locks têm timeout?
- [ ] Offline Concurrency Patterns são usados quando transação de negócio > transação de sistema?

### Performance
- [ ] Métricas são definidas com vocabulário preciso (response time, throughput, latency)?
- [ ] Otimizações são baseadas em medições, não em intuição?
- [ ] Existe capacity planning com load testing?

---

## Perguntas que esta Skill Faz

1. **"A complexidade da solução técnica é proporcional à complexidade do domínio?"**
   Domain Model para CRUD é overengineering. Transaction Script para 50 regras interativas é suicídio.

2. **"A Service Layer contém lógica de negócio ou apenas orquestra?"**
   Se a resposta é "contém lógica", você tem Transaction Script disfarçado de Domain Model.

3. **"Por que este sistema é distribuído? Testamos clustering primeiro?"**
   A primeira lei: não distribua. Microservices não são um default — são um trade-off.

4. **"Quantas chamadas remotas esta operação de negócio faz?"**
   Se >3, a interface é chatty. Consolide em Remote Facade.

5. **"O domínio conhece o mecanismo de persistência?"**
   Se a entidade de domínio faz `import SQLAlchemy`, há acoplamento. Data Mapper resolve isso.

6. **"As transações de sistema são curtas?"**
   Transação SQL aberta enquanto o usuário preenche um formulário = deadlocks garantidos.

7. **"Há coluna de versão nas tabelas que sofrem edição concorrente?"**
   Sem controle otimista, o último a salvar sobrescreve silenciosamente o primeiro.

8. **"O Identity Map é por-request ou global?"**
   Global = memory leak e dados stale. Sempre per-request.

9. **"Lazy Load está causando N+1?"**
   Ative SQL logging nos testes e conte queries. Se um loop gera N selects, use eager loading.

10. **"DTOs existem porque há fronteira remota ou por cargo cult?"**
    DTO sem serialização cross-process é complexidade gratuita.

11. **"A herança é mapeada com estratégia consistente?"**
    Misturar Single Table e Class Table na mesma hierarquia cria queries imprevisíveis.

12. **"O template contém lógica de negócio?"**
    Se o template calcula impostos ou aplica descontos, houve vazamento de domínio.

13. **"Qual o isolation level desta transação e por que este nível?"**
    Se a resposta é "o default do banco", a equipe não analisou os requisitos de concorrência.

14. **"Os locks pessimistas têm timeout?"**
    Usuário fechou o browser com lock ativo. Sem timeout, o recurso fica bloqueado indefinidamente.

15. **"Sabemos diferenciar response time de throughput nesta discussão de performance?"**
    "O sistema está lento" não é diagnóstico. Precisamos de números e do vocabulário correto.

16. **"O Data Source Pattern é compatível com o Domain Logic Pattern?"**
    Active Record + Domain Model complexo = sofrimento. Verifique a tabela de pairing.

17. **"Estamos confundindo Layer com Tier?"**
    Ter 3 layers não significa precisar de 3 tiers. Distribuição é decisão de deployment, não de código.

---

## Vocabulário

| Termo | Definição |
|-------|----------|
| **Layer** | Separação lógica de responsabilidades no código (pacotes, módulos) |
| **Tier** | Separação física de componentes em processos ou máquinas diferentes |
| **Transaction Script** | Um procedimento para cada caso de uso, coordenando leitura, lógica e escrita |
| **Domain Model** | Rede de objetos com dados e comportamento que representam o domínio do negócio |
| **Table Module** | Um objeto por tabela que opera sobre conjuntos de registros (RecordSets) |
| **Service Layer** | Camada fina que define a fronteira da aplicação e orquestra casos de uso |
| **Table Data Gateway** | Objeto que encapsula todo o SQL para acessar uma tabela, retornando RecordSets |
| **Row Data Gateway** | Objeto que encapsula acesso a uma linha do banco, sem lógica de negócio |
| **Active Record** | Objeto que encapsula uma linha do banco COM lógica de domínio |
| **Data Mapper** | Camada de mapeamento separada entre Domain Model e banco de dados |
| **Unit of Work** | Objeto que rastreia alterações em uma transação e coordena a escrita no banco |
| **Identity Map** | Cache por-transação que garante identidade referencial de objetos carregados |
| **Lazy Load** | Técnica que adia carregamento de dados relacionados até o momento do acesso |
| **Identity Field** | Campo no objeto que corresponde à chave primária no banco |
| **Embedded Value** | Value Object mapeado como colunas da tabela da entidade proprietária |
| **Serialized LOB** | Grafo de objetos serializado em campo BLOB/CLOB |
| **Single Table Inheritance** | Toda a hierarquia de herança mapeada em uma única tabela |
| **Class Table Inheritance** | Uma tabela por classe na hierarquia, com joins para leitura |
| **Concrete Table Inheritance** | Uma tabela por classe concreta, com dados duplicados do pai |
| **Remote Facade** | Interface coarse-grained para acesso remoto que delega a lógica local |
| **DTO (Data Transfer Object)** | Objeto serializável para transporte de dados cross-process |
| **Front Controller** | Ponto único de entrada que centraliza processamento de requests |
| **Page Controller** | Um controller por página/endpoint específico |
| **Template View** | View com marcadores preenchidos por dados em tempo de renderização |
| **Transform View** | View que transforma Model em output via transformação programática |
| **Two Step View** | View em dois passos: dados -> representação lógica -> output visual |
| **Optimistic Offline Lock** | Detecção de conflito no momento do commit via versão/timestamp |
| **Pessimistic Offline Lock** | Prevenção de conflito via lock adquirido no início da transação de negócio |
| **Coarse-Grained Lock** | Lock único que cobre um agregado inteiro |
| **Implicit Lock** | Lock aplicado automaticamente pelo framework |
| **Complexity Booster** | Fator que justifica soluções técnicas mais sofisticadas |
| **Response time** | Tempo total para o sistema processar uma requisição completa |
| **Responsiveness** | Rapidez com que o sistema reconhece que recebeu a requisição |
| **Latency** | Tempo de espera antes que o processamento comece |
| **Throughput** | Volume de trabalho processado por unidade de tempo |
| **Load** | Quantidade de stress atual no sistema |
| **Capacity** | Throughput máximo suportado pelo sistema |
| **Scalability** | Capacidade de manter performance ao adicionar recursos |
| **Efficiency** | Razão entre performance entregue e recursos consumidos |
