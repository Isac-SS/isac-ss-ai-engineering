# Skill: Designing Data-Intensive Applications — Martin Kleppmann

## Perspectiva do Autor

Martin Kleppmann é um pesquisador e engenheiro que aborda sistemas de dados com rigor acadêmico
e experiência prática. Sua visão de mundo é fundamentalmente cética: ele questiona as garantias
que bancos de dados e sistemas distribuídos realmente oferecem versus o que seus manuais prometem.

Prioridades centrais de Kleppmann:

1. **Entender o que realmente acontece sob falha** — sistemas não falham binariamente (funciona/não funciona).
   Eles degradam, perdem mensagens, reordenam eventos, dividem redes. O arquiteto deve projetar para
   o comportamento REAL sob falha, não para o cenário feliz.

2. **Dados são o ativo mais duradouro** — código muda a cada sprint; dados persistem por décadas.
   Decisões sobre modelo de dados, encoding e schema evolution têm impacto desproporcional no longo prazo.

3. **Trade-offs são inevitáveis e devem ser explícitos** — não existe banco de dados perfeito.
   Cada escolha (consistência vs. disponibilidade, throughput de escrita vs. leitura, normalização vs.
   denormalização) tem custo. O arquiteto deve documentar QUAL trade-off fez e POR QUÊ.

4. **Ferramentas diferentes para problemas diferentes** — não use seu banco OLTP como fila de mensagens.
   Não use sua fila de mensagens como banco de dados. Cada ferramenta foi otimizada para um padrão de
   acesso específico.

5. **Dados derivados vs. dados de origem** — toda a arquitetura de dados pode ser pensada como
   "source of truth" (log de eventos, banco primário) e "dados derivados" (caches, search indexes,
   materialized views). Dados derivados podem ser reconstruídos; dados de origem não.

Kleppmann é o autor que faz as perguntas incômodas: "O que acontece se o líder cair entre o commit
local e a replicação? O que acontece se o clock do servidor estiver errado? O que acontece se dois
datacenters aceitarem escritas conflitantes?" Se você não tem resposta, seu sistema tem um bug latente.

---

## Princípios Fundamentais

### 1. Os Três Pilares: Reliability, Scalability, Maintainability

#### 1.1 Reliability (Confiabilidade)

O sistema deve continuar funcionando corretamente mesmo quando coisas dão errado.

**Tipos de falha:**

| Tipo | Exemplos | Mitigação |
|------|----------|-----------|
| **Hardware faults** | Disco morre, RAM corrompida, rede desconecta | Redundância (RAID, replicação, failover) |
| **Software faults** | Bug que corrompe dados, memory leak, cascade failure | Testes, monitoramento, circuit breakers, crash-only design |
| **Human faults** | Config errada, deploy com bug, delete acidental | Sandbox, rollback fácil, telemetria, deploy gradual |

**Princípio fundamental:** projete para ser fault-tolerant, não fault-free.
Falhas VÃO acontecer — a questão é como o sistema se comporta quando acontecem.

#### 1.2 Scalability (Escalabilidade)

Escalabilidade não é um atributo binário ("é escalável" / "não é escalável").
A pergunta correta é: "se o load crescer de X para Y, o que acontece com a performance?"

**Descrevendo load — Load Parameters:**
- Requests por segundo
- Ratio de leituras vs. escritas
- Número de usuários simultâneos
- Tamanho do working set (dados acessados frequentemente)
- Fan-out (quantos serviços internos uma request toca)

**Descrevendo performance — Percentis:**
- **p50 (mediana):** metade dos usuários tem experiência melhor que isso
- **p95:** 5% dos usuários sofrem latência acima deste valor
- **p99:** 1% dos usuários (mas frequentemente os mais valiosos — heavy users)
- **p999:** long tail — problemas sistêmicos, não outliers aleatórios

**Regra de Kleppmann:** média de latência é quase inútil. Use percentis.
Um p99 de 10 segundos significa que 1 em cada 100 requests é inaceitavelmente lenta.

**Head-of-line blocking:** mesmo se apenas 1% das requests do backend for lenta,
se o client faz múltiplas requests paralelas, a PROBABILIDADE de pelo menos uma ser lenta
cresce exponencialmente. Por isso p99/p999 importam tanto.

#### 1.3 Maintainability (Manutenibilidade)

A maior parte do custo de software é manutenção, não desenvolvimento inicial.

- **Operability:** facilidade de operar (monitoramento, deploy, troubleshooting)
- **Simplicity:** facilidade de entender (abstrações claras, sem complexidade acidental)
- **Evolvability:** facilidade de mudar (schema evolution, backward compatibility)

### 2. Modelos de Dados e Query Languages

#### 2.1 Relacional vs. Documento vs. Grafo

| Modelo | Melhor para | Pior para | Schema |
|--------|------------|-----------|--------|
| **Relacional** | Joins, relacionamentos N:M, integridade referencial | Dados hierárquicos profundos | Schema-on-write |
| **Documento** | Documentos auto-contidos, leitura de agregados inteiros | Joins, relações many-to-many | Schema-on-read |
| **Grafo** | Relações complexas, travessias multi-hop, dados altamente conectados | Agregações analíticas simples | Schema flexível |

**Heurística-chave:** escolha o modelo baseado no padrão de acesso, NÃO na familiaridade da equipe.

#### 2.2 Schema-on-read vs. Schema-on-write

- **Schema-on-write** (relacional): o banco valida dados na escrita. Dados sempre conformes ao schema.
  Alterações de schema requerem migration (ALTER TABLE).
- **Schema-on-read** (documento, schemaless): o código que lê interpreta a estrutura. Flexível para
  dados heterogêneos, mas a validação é responsabilidade da aplicação.

**Trade-off:** schema-on-write dá garantias fortes mas é rígido. Schema-on-read dá flexibilidade
mas transfere a complexidade de validação para o código da aplicação.

### 3. Storage e Retrieval

#### 3.1 Duas Famílias de Storage Engines

| Aspecto | Log-structured (LSM-trees) | Page-oriented (B-trees) |
|---------|---------------------------|------------------------|
| **Otimizado para** | Escrita | Leitura |
| **Estrutura** | SSTables, memtable + compaction | Páginas de tamanho fixo, in-place update |
| **Write amplification** | Menor (sequential writes) | Maior (random writes) |
| **Read amplification** | Pode precisar checar múltiplos SSTables | Geralmente um seek na árvore |
| **Exemplos** | LevelDB, RocksDB, Cassandra, HBase | PostgreSQL, MySQL (InnoDB), Oracle |
| **Compaction** | Necessário (background), pode causar latency spikes | Não necessário |

#### 3.2 OLTP vs. OLAP

| Aspecto | OLTP | OLAP |
|---------|------|------|
| **Padrão de acesso** | Muitas leituras/escritas pequenas | Poucas queries que escaneiam muitos dados |
| **Queries** | Lookup por chave, insert, update | Agregações, GROUP BY, joins de muitas tabelas |
| **Volume por query** | Dezenas de registros | Milhões de registros |
| **Usado por** | Aplicação (via ORM/API) | Analistas (via SQL/BI) |
| **Storage engine** | B-tree, LSM-tree (row-oriented) | Column-oriented storage |
| **Schema** | Normalizado (3NF) | Star schema / Snowflake schema (denormalizado) |

**Regra:** não rode queries OLAP no banco OLTP. Use data warehouse, réplica de leitura,
ou pipeline de dados para analytics.

#### 3.3 Indexes — O Trade-off Fundamental

Indexes aceleram leituras mas desaceleram escritas. Cada index adicional:
- Ocupa espaço em disco e memória
- Precisa ser atualizado em cada INSERT/UPDATE/DELETE
- Pode causar write amplification

**Heurística:** não crie index "por precaução". Crie baseado em queries reais medidas.

### 4. Encoding e Schema Evolution

#### 4.1 Compatibilidade Bidirecional

- **Backward compatibility:** código NOVO lê dados escritos por código ANTIGO.
  Essencial — sempre precisa funcionar.
- **Forward compatibility:** código ANTIGO lê dados escritos por código NOVO.
  Necessário em deploys rolling (nem todos os nós atualizam ao mesmo tempo).

#### 4.2 Formatos de Encoding

| Formato | Schema evolution | Eficiência | Human-readable | Uso ideal |
|---------|-----------------|-----------|----------------|-----------|
| **JSON** | Difícil (sem schema formal) | Baixa | Sim | APIs REST, configs |
| **XML** | Difícil (schemas complexos) | Baixa | Sim | Legado, SOAP |
| **Protocol Buffers** | Boa (field tags + required/optional) | Alta | Não | gRPC, APIs internas |
| **Thrift** | Boa (field tags) | Alta | Não | Facebook ecosystem |
| **Avro** | Excelente (sem field tags, writer/reader schema) | Alta | Não | Kafka, data warehouse, event stores |

**Insight de Kleppmann sobre Avro:** Avro é ideal para event stores e data lakes porque o schema
evolui sem necessidade de field tags numéricos. O writer schema é armazenado junto aos dados,
e o reader schema resolve as diferenças automaticamente.

#### 4.3 Regras de Ouro para Schema Evolution

1. Nunca remova um campo obrigatório
2. Novos campos devem ser opcionais ou ter default value
3. Nunca reutilize um field tag/number que já foi usado (mesmo que o campo tenha sido removido)
4. Valide backward e forward compatibility em CI/CD

### 5. Replicação

#### 5.1 Topologias de Replicação

```
Single-leader:   [Leader] ---> [Follower 1]
                     |-------> [Follower 2]
                     |-------> [Follower 3]

Multi-leader:    [Leader A] <---> [Leader B]
                     |               |
                 [Follower]      [Follower]

Leaderless:      [Node 1] <---> [Node 2] <---> [Node 3]
                 (qualquer nó aceita leituras e escritas)
```

#### 5.2 Síncrono vs. Assíncrono

| Aspecto | Síncrono | Assíncrono |
|---------|----------|------------|
| **Durabilidade** | Dado confirmado em N nós antes do ACK | Dado confirmado apenas no líder |
| **Latência de escrita** | Alta (espera replicação) | Baixa (ACK imediato) |
| **Risco** | Se follower cair, escrita trava | Se líder cair, dados recentes podem ser perdidos |
| **Comum em** | Transações financeiras, ZooKeeper | Maioria dos bancos relacionais (default) |

**Na prática:** a maioria dos sistemas usa "semi-síncrono" — 1 follower síncrono (garantia mínima)
e os demais assíncronos.

#### 5.3 Problemas de Replication Lag

| Problema | Descrição | Garantia necessária |
|----------|-----------|-------------------|
| **Read-after-write** | Usuário escreve e não vê sua própria escrita | Read-after-write consistency |
| **Monotonic reads** | Usuário vê dado novo, refresh mostra dado antigo | Monotonic reads |
| **Consistent prefix** | Observador vê resposta antes da pergunta (causalidade violada) | Consistent prefix reads |

**Soluções comuns:**
- Read-after-write: ler do líder para dados recém-escritos pelo mesmo usuário
- Monotonic reads: rotear mesmo usuário para mesma réplica (session affinity)
- Consistent prefix: usar particionamento causal ou sequence numbers

#### 5.4 Conflitos em Multi-leader

Conflitos são inevitáveis em multi-leader e leaderless. Estratégias de resolução:

- **Last-write-wins (LWW):** usa timestamp para resolver. **PERDE DADOS silenciosamente.**
  Kleppmann é enfático: LWW é uma fonte de bugs sutis e perda de dados.
- **Merge automático:** CRDTs, union de valores
- **Custom resolution:** lógica de aplicação que apresenta conflitos ao usuário
- **Evitar conflitos:** rotear todas as escritas de um usuário para o mesmo líder

### 6. Partitioning (Sharding)

#### 6.1 Estratégias de Particionamento

| Estratégia | Distribuição | Range queries | Hot spots |
|-----------|-------------|--------------|-----------|
| **Key range** | Por intervalos da chave (A-G, H-N, O-Z) | Eficientes | Alto risco (chaves populares) |
| **Hash** | Hash da chave distribui uniformemente | Ineficientes | Baixo risco (exceto chaves extremamente quentes) |
| **Composto** | Hash da primeira parte + range da segunda | Parcialmente eficientes | Equilibrado |

**Hot spots:** mesmo com hash partitioning, uma chave extremamente popular (ex: celebridade no Twitter)
cria hot spot em uma única partição. Solução: adicionar random suffix à chave (split da partição lógica),
mas isso complica leituras.

#### 6.2 Secondary Indexes em Sistemas Particionados

| Tipo | Como funciona | Escrita | Leitura |
|------|--------------|---------|---------|
| **Local (document-partitioned)** | Cada partição mantém seu próprio index | Rápida (atualiza 1 partição) | Scatter/gather (consulta TODAS as partições) |
| **Global (term-partitioned)** | Index particionado separadamente dos dados | Lenta (pode precisar atualizar múltiplas partições de index) | Eficiente (consulta apenas partições relevantes do index) |

**Trade-off clássico:** indexes locais tornam escritas simples e leituras caras.
Indexes globais tornam leituras eficientes e escritas complexas (e geralmente eventualmente consistentes).

#### 6.3 Rebalancing

- **Fixed number of partitions:** cria-se muito mais partições que nós; novos nós recebem partições existentes.
  Simples, mas número de partições deve ser escolhido corretamente no início.
- **Dynamic partitioning:** partições se dividem quando crescem demais (split) e fundem quando encolhem (merge).
  Adapta-se ao volume de dados.
- **Partitioning proporcional a nós:** número fixo de partições por nó; novos nós roubam partições existentes.

**Regra:** NUNCA rebalanceie automaticamente com migração de dados E rerouting simultâneo.
Se um nó sobrecarregado for erroneamente considerado morto, rebalancing automático piora a situação (cascade failure).

### 7. Transactions

#### 7.1 ACID — O Que Realmente Significa

| Letra | Significado Comum | Significado Real (Kleppmann) |
|-------|-------------------|------------------------------|
| **A** (Atomicity) | "Tudo ou nada" | **Abortability** — se algo dá errado, todas as escritas são desfeitas. Não tem relação com concorrência. |
| **C** (Consistency) | "Banco consistente" | **Invariante da aplicação** — é responsabilidade da APLICAÇÃO, não do banco. O banco apenas garante A, I e D. |
| **I** (Isolation) | "Transações não se veem" | **Serializability** — o resultado é como se transações executassem sequencialmente. Mas muitos bancos NÃO oferecem Serializable por padrão. |
| **D** (Durability) | "Dados salvos para sempre" | **Escrito em disco (ou replicado)** — mas discos falham, backups corrompem, e replicação tem lag. Durabilidade absoluta não existe. |

**Alerta de Kleppmann:** o "C" em ACID é marketing. Consistência é responsabilidade da aplicação.
Não confie que o banco garante suas business rules — ele garante constraints (FK, unique), não invariantes de negócio.

#### 7.2 Isolation Levels — A Realidade

| Nível | Previne | NÃO previne | Implementação |
|-------|---------|-------------|--------------|
| **Read Committed** | Dirty reads, dirty writes | Read skew, write skew, phantoms | Row-level locks + snapshot para leitura |
| **Snapshot Isolation (SI)** | Dirty reads, dirty writes, read skew | Write skew, phantoms | MVCC (Multi-Version Concurrency Control) |
| **Serializable** | Tudo | — | Serial execution, 2PL, ou SSI |

**ATENÇÃO:** muitos bancos chamam Snapshot Isolation de "Repeatable Read" (ex: PostgreSQL, MySQL/InnoDB).
O nome no banco NÃO corresponde ao padrão SQL. Verifique o comportamento real, não o nome.

#### 7.3 Write Skew e Phantoms

**Write skew** ocorre quando duas transações leem o mesmo dado, tomam decisões baseadas nessa leitura,
e escrevem em registros DIFERENTES — mas as duas escritas juntas violam um invariante.

**Exemplo clássico:** dois médicos de plantão verificam que há 2 médicos ativos e decidem sair.
Cada transação vê 2 médicos, permite a saída, e o resultado é 0 médicos de plantão.

**Phantom:** um write skew onde a decisão é baseada na AUSÊNCIA de registros (SELECT que retorna vazio).
Não há registro para lockar — o lock não pode prevenir o problema.

**Snapshot Isolation NÃO previne write skew.** Este é um dos insights mais importantes do livro.
Somente Serializable resolve.

#### 7.4 Implementações de Serializable

| Abordagem | Mecanismo | Throughput | Latência | Melhor para |
|-----------|-----------|-----------|---------|-------------|
| **Actual Serial Execution** | Uma thread processa tudo | Baixo (single-threaded) | Baixa (sem contenção) | Workloads simples (Redis, VoltDB) |
| **Two-Phase Locking (2PL)** | Locks compartilhados e exclusivos | Baixo (muita contenção) | Alta (espera locks) | Workloads com muitas escritas longas |
| **Serializable Snapshot Isolation (SSI)** | MVCC + detecção otimista de conflitos | Alto (otimista) | Baixa (retries em conflito) | Workloads mistas, melhor opção geral |

**Recomendação de Kleppmann:** SSI é a melhor opção para a maioria dos workloads.
Combina a performance de Snapshot Isolation com a corretude de Serializable.

#### 7.5 Distributed Transactions e 2PC

**Two-Phase Commit (2PC):**
1. Fase Prepare: coordinator pergunta a todos os participantes se podem commitar
2. Fase Commit: se todos disseram sim, coordinator manda commitar

**Problemas:**
- Se o coordinator cair após o prepare mas antes do commit, participantes ficam em dúvida (in-doubt)
  e segurando locks indefinidamente
- O coordinator é um single point of failure
- Performance degradada (latência = RTT para cada participante + tempo de log de cada um)

**Kleppmann é cético sobre distributed transactions:** elas são corretas em teoria, mas
frágeis na prática. Prefira designs que não precisem de transações distribuídas.

### 8. Consistency e Consensus

#### 8.1 Linearizability

"O sistema se comporta como se houvesse apenas uma cópia dos dados, e todas as operações
são atômicas."

**Requer:** após uma escrita ser confirmada, TODAS as leituras subsequentes (de qualquer client)
devem retornar o valor novo. Não existe "janela de inconsistência" visível.

**Usos que REQUEREM linearizability:**
- Eleição de líder (quem é o líder AGORA?)
- Constraints de unicidade (este username está disponível?)
- Cross-channel timing (enviei notificação push E gravei no banco — o push deve ver o dado)

**Custo:** linearizability requer coordenação entre todos os nós, o que limita performance e
disponibilidade durante partições de rede.

#### 8.2 CAP Theorem — O Que Realmente Diz

O CAP theorem é frequentemente mal interpretado. O que ele REALMENTE diz:

> **Durante uma network partition, você deve escolher entre Consistency (linearizability) e
> Availability (todos os nós respondem).**

**O que CAP NÃO diz:**
- NÃO é "escolha 2 de 3" — network partitions ACONTECEM, então a escolha é C ou A durante partições
- NÃO se aplica em operação normal (sem partição, você pode ter C E A)
- NÃO cobre todos os tipos de consistency — CAP fala apenas de linearizability

**Regra prática:** em vez de pensar em CAP, pense: "o que acontece quando a rede falha?"
Se sua resposta é "o sistema para de aceitar escritas" = CP.
Se é "nós aceitam escritas independentemente, resolvemos conflitos depois" = AP.

#### 8.3 Causal Consistency

Mais fraca que linearizability, mas mais prática:
- Operações causalmente relacionadas são vistas na mesma ordem por todos
- Operações concorrentes (sem relação causal) podem ser vistas em qualquer ordem

**Vantagem:** pode ser implementada sem coordenação global, ou seja, sem sacrificar disponibilidade.

#### 8.4 Total Order Broadcast e Consensus

- **Total Order Broadcast:** todos os nós recebem as mesmas mensagens na mesma ordem.
  Equivalente a consensus e a um log replicado (como Kafka com replicação).
- **Consensus:** algoritmos como Raft, Paxos, Zab garantem que todos os nós concordam em uma decisão.
- **Implementações:** ZooKeeper (Zab), etcd (Raft), Consul (Raft)

**Insight:** se você precisa de eleição de líder, locks distribuídos, ou sequenciamento total,
use uma ferramenta de consensus (ZooKeeper, etcd). Não tente implementar consensus manualmente.

### 9. Stream Processing

#### 9.1 Event Logs vs. Message Queues Tradicionais

| Aspecto | Log-based (Kafka) | Traditional (RabbitMQ) |
|---------|-------------------|----------------------|
| **Persistência** | Mensagens retidas por tempo configurável | Mensagens deletadas após consumo (ACK) |
| **Replay** | Sim — consumidor pode rebobinar e reprocessar | Não — uma vez consumida, sumiu |
| **Ordering** | Garantida por partição | Não garantida (exceto em filas FIFO) |
| **Consumer groups** | Cada grupo lê independentemente, sem interferir | Mensagem entregue a um consumidor por fila |
| **Throughput** | Muito alto (sequential I/O) | Moderado |
| **Use case** | Event sourcing, CDC, stream processing | Task queues, work distribution |

**Heurística fundamental de Kleppmann:**
> "Não use seu banco OLTP como fila de mensagens. Não use sua fila de mensagens como banco de dados."

#### 9.2 Event Sourcing, CQRS e CDC

- **Event Sourcing:** armazena a sequência de eventos que levaram ao estado atual, não o estado em si.
  O estado é derivado reproduzindo eventos. Permite auditoria completa e reconstrução.

- **CQRS (Command Query Responsibility Segregation):** modelo de escrita (commands) separado do
  modelo de leitura (queries). A escrita emite eventos, e views materializadas otimizadas
  são construídas para leitura.

- **Change Data Capture (CDC):** captura mudanças no banco de dados (INSERTs, UPDATEs, DELETEs)
  como stream de eventos. Permite alimentar caches, search indexes, data warehouse a partir do
  banco primário sem acoplamento.

**Relação:** CDC é a ponte entre o mundo de bancos de dados e o mundo de streams.
Event Sourcing é o modelo nativo de streams. CQRS é a arquitetura que conecta escrita e leitura.

#### 9.3 Joins em Stream Processing

| Tipo de join | O que combina | Exemplo |
|-------------|--------------|---------|
| **Stream-stream** | Dois streams de eventos | Correlacionar click com impressão de anúncio |
| **Stream-table** | Stream de eventos com lookup table | Enriquecer evento de pedido com dados do cliente |
| **Table-table** | Duas materialized views (realmente dois CDC streams) | Manter view que junta duas tabelas |

#### 9.4 Exactly-once Semantics

"Exactly-once delivery" é tecnicamente impossível em sistemas distribuídos.
O que sistemas reais oferecem é **effectively-once processing:**

- **Idempotência:** projetar operações para que executar 2x tenha o mesmo efeito que 1x.
  Ex: `SET balance = 100` é idempotente; `SET balance = balance + 10` não é.
- **Transactional outbox:** escrita no banco + publicação de evento na mesma transação.
- **Deduplication:** usar ID único por evento e ignorar duplicatas.

### 10. Batch Processing

#### 10.1 Filosofia Unix como Fundamento

Kleppmann traça um paralelo direto entre o Unix philosophy e batch processing moderno:

- **Cada programa faz uma coisa bem feita** -> cada job de batch tem uma função clara
- **Saída de um programa é entrada de outro** -> dados fluem entre jobs via arquivos/datasets
- **Composição via pipes** -> DAGs de jobs (Airflow, Spark pipelines)

#### 10.2 MapReduce e Além

- **MapReduce:** modelo de programação onde Map extrai key-value pairs e Reduce agrega por chave.
  Simples mas verboso; adequado para Hadoop.
- **Spark:** substitui MapReduce com operações mais ricas (filter, join, groupBy) e execução
  in-memory. Muito mais performático para pipelines iterativos (ML, grafos).
- **Flink:** stream processing com batch como caso especial. Modelo unificado.

#### 10.3 Derived Data — O Conceito Unificador

> "Caches, search indexes, materialized views, data warehouses são TODOS dados derivados
> do source of truth."

**Implicação arquitetural:** se o source of truth é um log de eventos (event log, CDC),
então QUALQUER dado derivado pode ser reconstruído a partir desse log.

Isso habilita:
- Reconstrução de search indexes sem downtime
- Migração de schemas reproduzindo o log para o novo formato
- Debugging reproduzindo eventos até o ponto do bug
- Novos consumidores que processam o histórico completo

---

## Heurísticas Verificáveis

### Reliability

1. **Todo serviço externo deve ter timeout configurado.** Chamada sem timeout = possibilidade de thread/connection hanging forever.
   - *Como verificar:* grep por chamadas HTTP/gRPC/DB sem timeout explícito. Cada uma deve ter timeout configurado.

2. **Circuit breaker em integrações.** Serviços downstream com falhas devem ser cortados, não retentados infinitamente.
   - *Como verificar:* verifique se existe circuit breaker (Hystrix, Resilience4j, Polly) em cada chamada externa.

3. **Retries com backoff exponencial e jitter.** Retries imediatos ou com intervalo fixo causam thundering herd.
   - *Como verificar:* procure políticas de retry. Se o intervalo é fixo (ex: `sleep(1)`), adicione exponential backoff + jitter.

4. **Health checks distinguem liveness de readiness.** Liveness = "o processo está vivo"; readiness = "posso processar requests".
   - *Como verificar:* verifique se há endpoints separados para liveness e readiness no Kubernetes/load balancer.

5. **Crash-only design.** O sistema deve ser projetado para ser desligado a qualquer momento sem corrupção.
   - *Como verificar:* mate o processo com `kill -9` em teste. O sistema deve recuperar-se automaticamente no restart.

### Scalability

6. **Load parameters documentados.** Todo sistema deve ter seus load parameters definidos e medidos.
   - *Como verificar:* existe documento com: requests/s, ratio read/write, tamanho do dataset, fan-out?

7. **Percentis, não médias.** Métricas de latência devem usar percentis (p50, p95, p99).
   - *Como verificar:* dashboards mostram percentis? Se mostram apenas média, corrija.

8. **Hot spot awareness em particionamento.** Se há sharding, hot spots devem ser identificados e mitigados.
   - *Como verificar:* verifique se há métricas por partição. Se uma partição tem 10x mais load que as outras, há hot spot.

### Data Modeling

9. **Modelo de dados baseado em access pattern.** A escolha relacional/documento/grafo deve ser justificada pelo padrão de acesso.
   - *Como verificar:* pergunte: "quais são as 5 queries mais frequentes?" O modelo deve otimizar para essas.

10. **Normalização consciente.** Denormalização deve ser explícita, documentada e justificada por performance.
    - *Como verificar:* se há dados duplicados entre tabelas, deve haver comentário ou documento explicando por quê.

11. **Schema evolution testada.** Backward e forward compatibility devem ser validadas automaticamente.
    - *Como verificar:* há testes que verificam: "código novo lê dados antigos" e "código antigo lê dados novos"?

12. **Field tags nunca reutilizados.** Em Protobuf/Thrift/Avro, field tags removidos nunca devem ser reusados.
    - *Como verificar:* mantenha lista de field tags "aposentados" no .proto/.avsc. CI deve rejeitar reuso.

### Storage e Indexes

13. **OLTP e OLAP separados.** Queries analíticas não devem rodar no banco transacional.
    - *Como verificar:* procure queries com GROUP BY, window functions ou full table scans no banco OLTP em produção.

14. **Indexes justificados por queries reais.** Cada index deve corresponder a uma query em uso.
    - *Como verificar:* liste todos os indexes e mapeie cada um para pelo menos uma query. Index sem query = overhead desnecessário.

15. **Sem SELECT * em produção.** Queries devem especificar colunas para aproveitar column-oriented storage e covering indexes.
    - *Como verificar:* grep por `SELECT *` no código de produção. Cada ocorrência deve ser substituída por colunas explícitas.

### Replicação

16. **Replication lag monitorado.** Deve haver alerta quando lag excede threshold aceitável.
    - *Como verificar:* existe métrica de replication lag em segundos no dashboard? Existe alerta configurado?

17. **Read-after-write para dados do próprio usuário.** Após escrita, o mesmo usuário deve ler do líder ou de réplica atualizada.
    - *Como verificar:* após criar um recurso, a UI faz redirect e lê de réplica? Se sim, pode mostrar "não encontrado". Force leitura do líder.

18. **Resolução de conflitos explícita em multi-leader/leaderless.** Nunca use last-write-wins sem entender que perde dados.
    - *Como verificar:* se usa Cassandra, DynamoDB ou similar, verifique a política de conflito. LWW é aceitável? Se não, implemente merge.

### Transactions

19. **Isolation level explícito e justificado.** Toda transação crítica deve declarar e documentar seu isolation level.
    - *Como verificar:* se transações que protegem invariantes de negócio usam Read Committed, write skew é possível. Avalie Serializable ou SSI.

20. **Write skew analisado.** Para invariantes que envolvem leitura + escrita em registros diferentes, Snapshot Isolation é insuficiente.
    - *Como verificar:* identifique invariantes tipo "no mínimo N registros devem satisfazer condição X". Verifique se o isolation level previne violação.

21. **2PC evitado quando possível.** Distributed transactions com 2PC devem ter justificativa forte.
    - *Como verificar:* se há 2PC (XA transactions), pergunte: "podemos redesenhar com sagas ou eventual consistency?"

22. **Transações curtas.** Transações não devem aguardar I/O externo ou input humano.
    - *Como verificar:* se dentro de uma transação há chamada HTTP, envio de email ou espera por resposta, redesenhe.

### Partitioning

23. **Rebalancing não-automático para movimentação de dados.** Rebalancing automático pode causar cascade failure.
    - *Como verificar:* verifique se o sistema auto-rebalanceia sob detecção de falha. Se sim, adicione human-in-the-loop ou delay.

24. **Scatter/gather limitado.** Queries que tocam todas as partições devem ser identificadas e minimizadas.
    - *Como verificar:* monitore fan-out de queries. Se query de listagem toca TODAS as partições, considere index global ou reorganização.

### Stream Processing

25. **Processamento idempotente.** Toda operação de consumo de eventos deve ser segura para re-execução.
    - *Como verificar:* procure operações não-idempotentes em consumers (ex: `balance += amount`). Redesenhe com `balance = computed_value` ou dedup.

26. **Separação entre OLTP e message queue.** O banco não é fila; a fila não é banco.
    - *Como verificar:* se há polling de tabela "outbox" sem CDC, ou queries SQL contra Kafka, há mistura de responsabilidades.

27. **CDC para manter derived data.** Caches, search indexes e materialized views devem ser alimentados via CDC, não dual-write.
    - *Como verificar:* se o código escreve no banco E no Elasticsearch na mesma operação, dual-write causa inconsistência. Use CDC.

28. **Event schema versionado.** Eventos em streams devem ter schema registrado e versionado.
    - *Como verificar:* existe schema registry (Confluent, Apicurio)? Eventos sem schema estão sujeitos a quebra silenciosa.

### Consistency e Consensus

29. **Linearizability apenas quando necessária.** Nem toda operação precisa de linearizability — o custo é alto.
    - *Como verificar:* identifique onde linearizability é usada (locks distribuídos, eleição de líder). Se é usada para leitura genérica, está overengineered.

30. **Consensus delegado a ferramentas especializadas.** Não implemente Raft/Paxos manualmente.
    - *Como verificar:* se há código custom de eleição de líder ou lock distribuído sem ZooKeeper/etcd/Consul, refatore.

---

## Anti-patterns com Sinais no Código

| # | Anti-pattern | Sinal no Código | Severidade | Ação |
|---|-------------|-----------------|------------|------|
| 1 | **OLTP como fila** | Tabela `jobs`/`queue` com polling (`SELECT ... WHERE status='pending'`) sem CDC | Alta | Usar message broker (Kafka, RabbitMQ) ou CDC sobre outbox table |
| 2 | **Message queue como banco** | Dados lidos de Kafka por key lookup em vez de consumidos como stream | Alta | Armazenar estado em banco; Kafka é log, não key-value store |
| 3 | **Dual-write** | Código que escreve no banco E no cache/search/queue em sequência, sem transação | Crítica | Usar CDC ou transactional outbox para manter consistência |
| 4 | **Last-write-wins sem consciência** | Multi-leader ou leaderless com LWW sem análise de perda de dados | Crítica | Avaliar merge semântico, CRDTs, ou resolução de conflito custom |
| 5 | **SELECT * em produção** | Queries que buscam todas as colunas de tabelas largas | Média | Especificar colunas; aproveitar covering indexes |
| 6 | **Chamada sem timeout** | `httpClient.get(url)` sem timeout configurado | Alta | Adicionar timeout em toda chamada externa |
| 7 | **Retry sem backoff** | `while(!success) { retry(); sleep(1); }` | Alta | Exponential backoff com jitter |
| 8 | **Média como métrica de latência** | Dashboard mostrando "avg response time: 200ms" | Média | Substituir por percentis p50/p95/p99 |
| 9 | **Snapshot Isolation tratado como Serializable** | Invariante de negócio protegido apenas por transaction sem `SERIALIZABLE` | Crítica | Analisar se write skew é possível; usar `SERIALIZABLE` ou `SELECT FOR UPDATE` |
| 10 | **Index especulativo** | Indexes em colunas que não aparecem em nenhuma query WHERE/JOIN | Média | Remover indexes não utilizados; monitorar com `pg_stat_user_indexes` |
| 11 | **Schema evolution sem teste** | Protobuf/Avro schema modificado sem teste de backward/forward compatibility | Alta | Adicionar testes de compatibilidade em CI/CD |
| 12 | **2PC desnecessário** | XA transactions entre serviços que poderiam usar sagas | Alta | Redesenhar com sagas e compensação |
| 13 | **Rebalancing automático agressivo** | Auto-sharding que move dados imediatamente ao detectar nó lento | Alta | Adicionar delay e human approval para movimentação massiva de dados |
| 14 | **Hot spot ignorado** | Partição com 10x mais carga que as demais sem ação | Alta | Monitorar per-partition metrics; adicionar salt key ou repartionar |
| 15 | **Read-after-write em réplica** | Redirect após POST lê de réplica antes da replicação completar | Alta | Forçar leitura do líder para dados recém-escritos pelo mesmo usuário |
| 16 | **Event sem schema** | Eventos publicados como JSON sem schema registry ou validação | Média | Implementar schema registry; validar na publicação |
| 17 | **Consumer não-idempotente** | Consumer que faz `UPDATE balance = balance + amount` sem deduplication | Crítica | Redesenhar com idempotência: dedup por event ID ou operação set-based |
| 18 | **Lack of replication lag monitoring** | Nenhuma métrica sobre lag entre líder e followers | Alta | Implementar monitoramento de lag com alerta |

---

## Checklist de Revisão

### Reliability
- [ ] Toda chamada externa tem timeout configurado?
- [ ] Existe circuit breaker para cada dependência downstream?
- [ ] Retries usam exponential backoff com jitter?
- [ ] O sistema sobrevive a kill -9 sem corrupção de dados?
- [ ] Há distinção entre health check de liveness e readiness?
- [ ] Existe runbook para os cenários de falha mais prováveis?

### Scalability
- [ ] Load parameters estão documentados (requests/s, ratio read/write, dataset size)?
- [ ] Métricas de latência usam percentis (p50, p95, p99), não média?
- [ ] Existe capacity planning com projeção de crescimento?
- [ ] Hot spots em partições estão identificados e monitorados?

### Data Model
- [ ] O modelo de dados é otimizado para os access patterns reais?
- [ ] Denormalizações são explícitas e documentadas?
- [ ] Schema evolution mantém backward e forward compatibility?
- [ ] Field tags/numbers removidos não são reutilizados?

### Storage
- [ ] Workloads OLTP e OLAP estão separados em engines diferentes?
- [ ] Indexes correspondem a queries reais em produção?
- [ ] Não há SELECT * em queries de produção?
- [ ] Write amplification é monitorada para workloads write-heavy?

### Replication
- [ ] Replication lag é monitorado e alertado?
- [ ] Read-after-write consistency está garantida para o usuário que escreveu?
- [ ] Conflitos em multi-leader/leaderless têm resolução explícita (não LWW cego)?
- [ ] A topologia de replicação é adequada ao use case (single-leader para maioria)?

### Transactions
- [ ] Isolation levels são explícitos para transações que protegem invariantes?
- [ ] Write skew é analisado para invariantes multi-registro?
- [ ] Transações são curtas (sem I/O externo ou espera humana)?
- [ ] 2PC é evitado quando sagas ou eventual consistency bastam?

### Partitioning
- [ ] Estratégia de particionamento é adequada aos access patterns (range vs. hash)?
- [ ] Scatter/gather queries são identificadas e minimizadas?
- [ ] Rebalancing tem human-in-the-loop ou delay de segurança?
- [ ] Secondary indexes são local ou global conforme trade-off analisado?

### Stream Processing
- [ ] Processamento de eventos é idempotente?
- [ ] O banco OLTP não está sendo usado como fila?
- [ ] Derived data (cache, search, views) é alimentado via CDC, não dual-write?
- [ ] Eventos têm schema versionado em schema registry?
- [ ] Exactly-once semantics é implementada via idempotência ou deduplication?

### Consistency e Consensus
- [ ] Linearizability é usada apenas onde necessária (eleição de líder, unicidade)?
- [ ] Consensus é delegado a ferramentas especializadas (ZooKeeper, etcd, Consul)?
- [ ] CAP trade-off está documentado: durante partição de rede, sistema é CP ou AP?
- [ ] Clocks não são usados para ordenação em decisões de corretude?

---

## Perguntas que esta Skill Faz

1. **"O que acontece quando o líder do banco cai entre o commit local e a replicação para os followers?"**
   Se a resposta é "não sabemos", há risco de perda de dados com replicação assíncrona.

2. **"Qual o p99 de latência desta operação? E o p999?"**
   Média esconde problemas. Percentis revelam a experiência dos piores casos.

3. **"Se eu rodar esta query no banco OLTP em produção, quantas linhas ela escaneia?"**
   Uma query analítica no banco OLTP pode derrubar o sistema transacional inteiro.

4. **"Estamos usando o banco como fila de mensagens?"**
   Polling de tabela "outbox" sem CDC é um anti-pattern clássico que não escala e causa lock contention.

5. **"O que acontece se este evento for processado duas vezes?"**
   Se a resposta é "duplica o efeito", falta idempotência. Em sistemas distribuídos, delivery duplicada é inevitável.

6. **"Last-write-wins é aceitável para este dado?"**
   Se dois usuários editam o mesmo documento e LWW descarta uma edição, isso é perda de dados.

7. **"Este isolation level previne write skew para este invariante?"**
   Snapshot Isolation NÃO previne write skew. Somente Serializable ou SELECT FOR UPDATE resolves.

8. **"Precisamos realmente de linearizability aqui, ou causal consistency basta?"**
   Linearizability custa caro em disponibilidade e performance. Use apenas quando necessário.

9. **"Se adicionarmos um campo ao schema, código antigo ainda lê corretamente?"**
   Forward compatibility é essencial em deploys rolling e sistemas com múltiplas versões simultâneas.

10. **"Estamos fazendo dual-write (banco + cache/search na mesma operação)?"**
    Dual-write sem transação distribuída GARANTE inconsistência eventual. Use CDC.

11. **"Quantas partições esta query precisa consultar?"**
    Se a resposta é "todas" (scatter/gather), há problema de design no particionamento ou nos indexes.

12. **"O rebalancing de partições é automático? O que acontece se ele se enganar?"**
    Rebalancing automático sob falha detectada erroneamente causa cascade failure.

13. **"Por que este dado está denormalizado? Há documentação sobre o trade-off?"**
    Denormalização sem justificativa se torna tech debt que ninguém sabe se pode reverter.

14. **"Usamos 2PC (distributed transactions) aqui. Podemos redesenhar com sagas?"**
    2PC é um coordinator bottleneck. Sagas com compensação são mais resilientes.

15. **"O que acontece durante uma network partition entre nossos datacenters?"**
    Se a resposta é "não sei", o sistema tem comportamento indefinido sob falha de rede.

16. **"Nossos clocks são confiáveis o suficiente para usar timestamps como critério de ordenação?"**
    Clocks em sistemas distribuídos são notoriamente imprecisos. Não use para decidir "quem chegou primeiro".

17. **"Este consumer pode reprocessar o histórico completo de eventos?"**
    Se não pode, você perdeu a capacidade de reconstruir derived data e corrigir bugs retroativamente.

18. **"Qual é a janela de replication lag aceitável para este use case?"**
    "Eventualmente consistente" não é resposta — precisa de SLA: "consistente em < 500ms em p99".

---

## Vocabulário

| Termo | Definição |
|-------|----------|
| **Reliability** | Capacidade do sistema de funcionar corretamente mesmo quando componentes falham |
| **Scalability** | Capacidade de lidar com crescimento de load mantendo performance aceitável |
| **Maintainability** | Facilidade de operar, entender e evoluir o sistema ao longo do tempo |
| **Load parameter** | Métrica que descreve o stress no sistema (requests/s, ratio read/write, fan-out) |
| **Percentil (p50, p95, p99)** | Valor abaixo do qual X% das observações caem. p99 = 99% dos requests são mais rápidos |
| **Head-of-line blocking** | Quando requests lentas bloqueiam requests rápidas na mesma fila/conexão |
| **Schema-on-read** | Estrutura dos dados é interpretada no momento da leitura (schema implícito) |
| **Schema-on-write** | Estrutura dos dados é validada no momento da escrita (schema explícito) |
| **LSM-tree** | Log-Structured Merge-tree: engine de storage otimizada para escrita sequencial |
| **B-tree** | Estrutura de dados em árvore balanceada com páginas de tamanho fixo, otimizada para leitura |
| **OLTP** | Online Transaction Processing: operações transacionais curtas e frequentes |
| **OLAP** | Online Analytical Processing: queries analíticas que escaneiam grandes volumes |
| **Write amplification** | Quando uma escrita lógica causa múltiplas escritas físicas no storage |
| **SSTable** | Sorted String Table: arquivo de dados ordenados por chave, usado em LSM-trees |
| **Backward compatibility** | Código novo consegue ler dados escritos por código antigo |
| **Forward compatibility** | Código antigo consegue ler dados escritos por código novo |
| **Replication lag** | Atraso entre a escrita no líder e a disponibilidade da escrita nos followers |
| **Read-after-write consistency** | Garantia de que o usuário vê sua própria escrita imediatamente |
| **Monotonic reads** | Garantia de que o usuário não vê dados "voltando no tempo" entre leituras |
| **Last-write-wins (LWW)** | Estratégia de resolução de conflito que mantém o valor com maior timestamp. Perde dados. |
| **Partitioning (sharding)** | Divisão de dados em subconjuntos (partições) distribuídos em diferentes nós |
| **Hot spot** | Partição que recebe carga desproporcionalmente maior que as demais |
| **Scatter/gather** | Query que precisa consultar todas as partições e combinar resultados |
| **ACID** | Atomicity (abortability), Consistency (app invariant), Isolation (serializability), Durability |
| **Write skew** | Anomalia onde duas transações leem o mesmo dado e escrevem em registros diferentes, violando invariante |
| **Phantom** | Anomalia onde uma transação toma decisão baseada na ausência de registro que outra transação cria |
| **MVCC** | Multi-Version Concurrency Control: mantém múltiplas versões de dados para leituras consistentes |
| **SSI** | Serializable Snapshot Isolation: MVCC + detecção otimista de conflitos serializáveis |
| **2PL** | Two-Phase Locking: protocolo pessimista com locks compartilhados e exclusivos |
| **2PC** | Two-Phase Commit: protocolo para transações distribuídas com coordinator |
| **Linearizability** | Garantia de que o sistema se comporta como se houvesse uma única cópia dos dados |
| **Causal consistency** | Garantia de que operações causalmente relacionadas são vistas na mesma ordem |
| **CAP theorem** | Durante network partition, deve-se escolher entre Consistency e Availability |
| **Total Order Broadcast** | Protocolo que entrega mensagens a todos os nós na mesma ordem total |
| **Consensus** | Acordo entre nós distribuídos sobre um valor/decisão (Raft, Paxos, Zab) |
| **CDC (Change Data Capture)** | Captura de mudanças no banco como stream de eventos |
| **Event Sourcing** | Armazenamento do histórico de eventos como source of truth, não do estado atual |
| **CQRS** | Command Query Responsibility Segregation: modelos separados para escrita e leitura |
| **Dual-write** | Anti-pattern: escrever em dois sistemas (banco + cache) sem coordenação transacional |
| **Idempotência** | Propriedade de operação que pode ser executada múltiplas vezes com o mesmo resultado |
| **Transactional outbox** | Pattern: escrever evento em tabela outbox na mesma transação do banco, publicar via CDC |
| **Derived data** | Dados computados a partir de source of truth (caches, indexes, materialized views) |
| **Source of truth** | O registro autoritativo e canônico de dados, do qual derivados são reconstruídos |
| **MapReduce** | Modelo de batch processing: fase Map extrai key-value, fase Reduce agrega por chave |
| **Star schema** | Schema de data warehouse com tabela fato central e tabelas dimensão ao redor |
| **Column-oriented storage** | Armazenamento que agrupa valores da mesma coluna, otimizado para queries analíticas |
