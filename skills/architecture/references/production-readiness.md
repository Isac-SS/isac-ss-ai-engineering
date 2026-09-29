# Skill: Production Readiness — Michael T. Nygard ("Release It!")

## Perspectiva do Autor

Michael Nygard aborda software com a mentalidade de quem já foi acordado às 3h da manhã
por alertas de produção. Sua visão central é que **a maioria dos sistemas é projetada para
passar nos testes, não para sobreviver em produção**. O ambiente de QA é um mundo
idealizado — rede confiável, carga previsível, dependências sempre disponíveis. Produção
é o oposto: hostil, imprevisível e implacável.

Nygard cunha o conceito de **Cynical Software** — software que assume o pior sobre toda
interação externa. Cada chamada de rede pode falhar. Cada dependência pode travar. Cada
usuário pode se comportar de forma inesperada. O software cínico não confia em nada e
se protege proativamente.

A filosofia pode ser resumida em uma frase: **"O software que sobrevive em produção é
aquele projetado para falhar graciosamente, não aquele projetado para nunca falhar."**

Prioridades do autor, em ordem:
1. **Estabilidade** — o sistema deve se manter estável sob condições adversas
2. **Capacidade** — o sistema deve lidar com carga real, não apenas carga de teste
3. **Transparência** — se você não consegue ver o que está acontecendo, não consegue consertar
4. **Deploy seguro** — mudanças não devem exigir downtime nem introduzir risco desnecessário
5. **Operabilidade** — o sistema deve ser operável por humanos sob pressão

O autor rejeita a separação entre "desenvolvimento" e "operações" — todo desenvolvedor
deve pensar em como seu código se comporta em produção. Código que funciona no laptop
mas derruba produção não é código funcional.

---

## Princípios Fundamentais

### 1. Estabilidade como Propriedade Emergente

Estabilidade não é uma feature que se adiciona ao final — é uma propriedade que emerge
de decisões de design tomadas ao longo de todo o desenvolvimento.

- **Failures are inevitable; outages are not.** Falhas individuais vão acontecer.
  O que importa é se uma falha isolada se propaga e derruba o sistema inteiro.
- Todo sistema tem um **failure mode** — a questão é se você o projetou
  intencionalmente ou se ele vai se revelar em produção às 3h da manhã.
- **Longevity testing** é essencial: sistemas que funcionam por 1 hora podem
  falhar após 72 horas por memory leaks, connection pool exhaustion,
  log files enchendo disco, etc.
- Um sistema estável absorve perturbações transitórias sem necessidade de
  intervenção humana.

### 2. Anti-patterns de Estabilidade (Os Assassinos de Produção)

#### 2.1 Integration Points (Pontos de Integração)

Cada integração com sistema externo é um risco de estabilidade. Não importa se é
um banco de dados, uma API REST, um serviço gRPC, um message broker ou um sistema
de arquivos remoto — **toda integração vai falhar de alguma forma, em algum momento**.

Modos de falha comuns:
- **Connection refused** — o serviço está fora do ar
- **Connection timeout** — o serviço está lento ou inalcançável
- **Slow response** — responde, mas demora 30 segundos em vez de 200ms
- **Garbage response** — retorna HTML de uma página de erro onde se esperava JSON
- **Partial response** — conexão cai no meio da transmissão
- **Success response with wrong data** — HTTP 200 mas o conteúdo está errado

Cada ponto de integração precisa de: timeout, circuit breaker, retry com backoff,
fallback definido, e monitoramento independente.

#### 2.2 Chain Reactions (Reações em Cadeia)

Quando um nó em um cluster falha, a carga é redistribuída para os nós restantes.
Se a falha foi causada por sobrecarga, os nós restantes agora recebem MAIS carga
e têm MAIOR probabilidade de falhar — criando uma cascata que derruba o cluster
inteiro.

Cenário clássico: 5 servidores atrás de um load balancer. Um cai por memory leak.
Os 4 restantes recebem 25% mais carga. O segundo cai. Os 3 restantes recebem 67%
mais carga que o original. Colapso total em minutos.

Defesa: health checks agressivos, auto-scaling, bulkheads, circuit breakers que
removem nós problemáticos antes que contaminem os demais.

#### 2.3 Cascading Failures (Falhas em Cascata)

Diferente de chain reactions (que acontecem dentro de uma camada), cascading failures
propagam entre camadas e serviços. O serviço A falha, o que causa timeout no serviço
B que depende de A, o que faz as threads do serviço B ficarem bloqueadas, o que faz
o serviço C (que depende de B) também travar.

**A cascata mais perigosa é a que cruza boundaries de times** — ninguém tem
visibilidade completa da cadeia de dependências.

#### 2.4 Users (Usuários)

Usuários reais não se comportam como scripts de teste:
- **Traffic spikes** — eventos virais, Black Friday, campanhas de marketing
- **Zombie sessions** — usuários com sessões abertas há dias, consumindo memória
- **Bots e scrapers** — gerando tráfego 10x maior que usuários reais
- **Deep linking** — entrando em páginas internas que assumem estado prévio
- **Impatient clicks** — clicando 15 vezes no botão de submit
- **Browser tabs** — 50 abas do mesmo usuário, cada uma com websocket ou polling

#### 2.5 Blocked Threads

**O problema de produção mais comum segundo Nygard.** Threads ficam bloqueadas
esperando por um recurso que nunca responde:
- Esperando resposta de um serviço externo sem timeout
- Esperando por um lock de banco de dados
- Esperando por uma posição no connection pool
- Deadlocks entre threads

O perigo: blocked threads não geram erros imediatos. O sistema simplesmente fica
mais e mais lento até parar completamente. Monitoramento tradicional (error rate)
não detecta — é preciso monitorar thread pool utilization e latência.

#### 2.6 Self-Denial Attacks (Auto-ataques)

O próprio sistema gera carga que o derruba:
- **Marketing envia email para 2 milhões de usuários** com link para uma página
  que faz 15 queries por request
- **Batch jobs** que rodam à meia-noite e competem com o tráfego noturno
- **Cache stampede** — cache expira e 10.000 requests simultâneas vão todas ao
  banco de dados
- **Retry storms** — quando um serviço fica lento, todos os clientes fazem retry
  simultaneamente, multiplicando a carga

#### 2.7 Scaling Effects

O que funciona com 2 servidores não funciona com 200.
- **Point-to-point communication** escala como O(N²) — cada nó precisa conhecer
  todos os outros
- **Shared resources** se tornam bottlenecks — um banco de dados para 200 serviços
- **Broadcast patterns** se tornam tempestades em escala
- **DNS, service discovery, load balancers** — tudo tem limites que só aparecem
  em escala

#### 2.8 Unbalanced Capacities (Capacidades Desbalanceadas)

Quando o frontend pode aceitar 10x mais requests do que o backend pode processar:
- API Gateway aceita 50.000 req/s, mas o serviço de pagamento processa 500 req/s
- O message broker aceita mensagens mais rápido do que o consumer processa
- O banco de dados aceita conexões mais rápido do que consegue executar queries

Defesa: load shedding, rate limiting, backpressure, capacity planning que
considera o sistema inteiro e não componentes isolados.

#### 2.9 Slow Responses (Respostas Lentas)

**Pior que falha.** Uma falha rápida é barata — o caller recebe o erro e segue.
Uma resposta lenta é cara — o caller fica esperando, consumindo thread, memória,
conexão, enquanto não recebe nada útil.

Respostas lentas são contagiosas: se o serviço A responde lentamente, todos os
serviços que dependem de A também ficam lentos, cada um consumindo recursos
enquanto espera.

#### 2.10 Unbounded Result Sets (Conjuntos de Resultado Sem Limite)

Queries que podem retornar milhões de registros:
- `SELECT * FROM orders` sem WHERE nem LIMIT
- APIs que retornam listas sem paginação
- Queries que em dev retornam 10 registros mas em produção retornam 10 milhões

Defesa: SEMPRE usar paginação, SEMPRE ter LIMIT, monitorar query result sizes.

#### 2.11 SLA Inversion

A disponibilidade do seu sistema é o **produto** da disponibilidade de todas as
dependências. Se você depende de 5 serviços com 99.9% de uptime cada:
0.999^5 = 99.5% — você perdeu metade do seu "budget" de downtime.

Se o seu SLA é 99.95% mas você depende de um serviço com SLA de 99.5%,
**é matematicamente impossível cumprir seu SLA** sem fallbacks.

### 3. Patterns de Estabilidade (As Defesas)

#### 3.1 Timeouts

**A regra mais importante: timeout em TUDO que é externo.**

- Toda chamada HTTP, toda query de banco, toda operação de I/O externo precisa
  de timeout explícito
- **Default timeout**: defina um timeout padrão para toda a aplicação (ex: 5s)
  e só ajuste para operações que comprovadamente precisam de mais
- **Timeout budget**: se uma operação envolve 3 chamadas sequenciais e o timeout
  total é 10s, cada chamada pode ter no máximo ~3s
- Sem timeout, uma thread pode ficar bloqueada para sempre
- Linguagens e frameworks frequentemente têm timeout padrão de infinito —
  NUNCA confie no default

#### 3.2 Circuit Breaker

O pattern mais importante de resiliência em sistemas distribuídos.

**Três estados:**
- **Closed** (Normal): requests passam normalmente. Falhas são contadas.
- **Open** (Bloqueado): requests falham imediatamente sem chamar o serviço
  downstream. Um timer define quando testar novamente.
- **Half-Open** (Testando): permite um número limitado de requests passar.
  Se tiverem sucesso, volta para Closed. Se falharem, volta para Open.

**Configuração por dependência:**
- Cada serviço downstream deve ter SEU circuit breaker
- Thresholds devem ser calibrados por dependência (um serviço crítico pode
  ter threshold de 3 falhas, um opcional pode ter 10)
- Fallback definido para cada circuit breaker (cache, valor default, degradação
  graceful, erro amigável)

**Métricas essenciais:** estado atual de cada circuit breaker, taxa de trip,
tempo médio em estado Open.

#### 3.3 Bulkheads

Inspirado em navios — compartimentos estanques que impedem que água inunde
toda a embarcação.

- **Thread pool bulkheads**: pools separados para cada dependência downstream.
  Se o serviço de pagamento travar, apenas as threads do pool de pagamento
  ficam bloqueadas — o resto do sistema continua funcionando.
- **Connection pool bulkheads**: pools de conexão separados por serviço.
- **Process bulkheads**: serviços críticos em processos/containers separados.
- **Infrastructure bulkheads**: dependências críticas em infra separada.

O princípio: **contain the blast radius** — limitar o raio de explosão de
qualquer falha.

#### 3.4 Steady State

O sistema deve rodar indefinidamente sem intervenção humana:
- **Log rotation**: logs devem ser rotacionados e comprimidos automaticamente.
  Disco cheio de logs é uma das causas mais comuns de outage.
- **Cache eviction**: caches devem ter TTL e política de eviction (LRU, LFU).
  Cache sem eviction = memory leak.
- **Session expiration**: sessões devem expirar. Sessões eternas consomem
  memória eternamente.
- **Data archival**: tabelas que só crescem eventualmente causam problemas.
  Estratégia de archival ou particionamento é essencial.
- **Disk space**: monitorar e alertar em 70%, agir automaticamente em 85%.
- **Connection pools**: conexões devem ter max-lifetime e idle-timeout.

#### 3.5 Fail Fast

Se uma operação vai falhar, que falhe imediatamente:
- Validação de input na entrada — não processe um request inválido
- Verificação de pré-condições antes de operações caras
- Se o circuit breaker está Open, falhe imediatamente
- Se o recurso necessário (conexão, thread) não está disponível, falhe
  em vez de esperar indefinidamente

Fail fast é o oposto de slow response — e slow response é o inimigo.

#### 3.6 Handshaking

Protocolo de comunicação onde o servidor sinaliza sua capacidade:
- **HTTP 503 Service Unavailable** — "estou sobrecarregado, tente mais tarde"
- **HTTP 429 Too Many Requests** com header Retry-After
- **Backpressure via streaming** — consumidor sinaliza ao produtor que está
  ficando para trás
- O cliente DEVE respeitar esses sinais — não fazer retry imediato

#### 3.7 Test Harness

Testes que simulam falhas reais de produção:
- **Slow responses** — resposta que leva 30 segundos
- **Connection refused** — serviço completamente indisponível
- **Garbage responses** — HTML onde se esperava JSON
- **Partial responses** — conexão cai no meio
- **Timeouts** — nenhuma resposta
- **TCP half-open** — conexão que parece aberta mas está morta
- **Connection reset** — RST no meio da comunicação

Sem test harness, você está testando o happy path e rezando para produção
se comportar da mesma forma.

#### 3.8 Decoupling Middleware

Comunicação assíncrona para desacoplamento temporal:
- Message queues (RabbitMQ, SQS, Kafka) permitem que produtor e consumidor
  operem em tempos diferentes
- Se o consumidor está fora, as mensagens aguardam na fila
- Elimina a necessidade de ambos estarem disponíveis simultaneamente
- **Trade-off**: complexidade adicional (delivery guarantees, ordering,
  dead letter queues, idempotência)

#### 3.9 Shed Load (Descarte de Carga)

Quando a carga excede a capacidade, **rejeitar requests é melhor que atender
todos lentamente**:
- Rate limiting por cliente, por endpoint, por IP
- Priorização de requests (health checks, pagamentos > navegação)
- HTTP 503 com Retry-After para requests rejeitados
- Load shedding deve ser automático — não depender de intervenção humana

#### 3.10 Create Back Pressure

Quando o consumidor não consegue acompanhar o produtor:
- **Bounded queues** — filas com tamanho máximo; quando cheia, o produtor
  é forçado a desacelerar
- **Reactive Streams** — consumidor informa ao produtor quantos itens
  pode processar
- **TCP flow control** — o próprio TCP faz backpressure via window size
- Sem backpressure, a alternativa é out-of-memory ou perda de dados

### 4. Capacidade

- **Connection pooling**: SEMPRE usar pools. Configurar min, max, idle timeout,
  max lifetime, wait timeout.
- **Caching**: agressivo mas com eviction. Cache sem TTL é memory leak.
  Cache stampede prevention (locking, probabilistic early expiration).
- **Precomputation**: views materializadas, agregações pré-calculadas,
  static site generation onde possível.
- **Garbage Collection tuning**: em JVM/Go/C#, entender e tunar o GC é
  essencial para performance previsível. GC pauses podem parecer outages.

### 5. Deploy e Operações

- **Zero-downtime deployment**: blue-green, canary, rolling updates.
  Se o deploy exige downtime, o design está errado.
- **Feature flags**: separar deploy (colocar código em produção) de release
  (ativar funcionalidade para usuários). Permite rollback instantâneo sem
  redeploy.
- **Health checks**:
  - **Shallow** (`/health`): "o processo está vivo e aceitando conexões"
  - **Deep** (`/health/deep`): "todas as dependências estão acessíveis e
    o sistema está funcional"
  - Load balancers usam shallow; monitoring usa deep
- **Observabilidade** (o tripé):
  - **Logs estruturados**: JSON com correlation ID, timestamp, nível, contexto
  - **Métricas**: latência (p50, p95, p99), throughput, error rate, saturation
  - **Distributed tracing**: trace ID propagado entre serviços para
    reconstruir a jornada de um request

**"If you can't see it, you can't fix it."**

### 6. Networking

Realidades de produção que não existem em dev:
- **DNS failures** são reais — cache DNS com TTL, fallback para IP
- **TLS handshake** adiciona latência significativa — connection pooling
  e keep-alive são essenciais
- **Connection pooling através de firewalls/load balancers** pode falhar
  silenciosamente — idle connections são fechadas por intermediários
  sem notificação
- **NICs parcialmente falhas** — aceitam conexões mas dropam pacotes.
  O cenário mais difícil de diagnosticar.
- **MTU mismatches** causam pacotes fragmentados e retransmissões
- **TCP keepalive** é diferente de HTTP keepalive — configure ambos

---

## Heurísticas Verificáveis

### Timeouts e Resiliência

1. **Toda chamada HTTP externa tem timeout explícito.**
   *Como verificar:* buscar todas as chamadas HTTP (RestTemplate, HttpClient,
   fetch, axios, etc.) e confirmar que timeout está configurado. Grep por
   configuração de timeout no client. Ausência de timeout = violação.

2. **Toda query de banco de dados tem statement timeout.**
   *Como verificar:* verificar configuração do connection pool (HikariCP,
   pgBouncer, etc.) para statement_timeout. Verificar se queries longas
   têm timeout específico. `SHOW statement_timeout` no PostgreSQL.

3. **Todo circuit breaker tem fallback definido.**
   *Como verificar:* buscar anotações/configurações de circuit breaker
   (@CircuitBreaker, resilience4j config, Polly policies) e confirmar
   que cada um define fallbackMethod ou equivalente.

4. **Circuit breakers existem por dependência, não genéricos.**
   *Como verificar:* contar o número de dependências externas e o número
   de circuit breakers. Deve haver ao menos um circuit breaker por
   dependência crítica.

5. **Retry tem backoff exponencial com jitter.**
   *Como verificar:* buscar configuração de retry. Retry com intervalo
   fixo = retry storm em potencial. Deve haver multiplicador e
   componente aleatório (jitter).

### Pools e Recursos

6. **Connection pools têm max, min, idle timeout e max lifetime configurados.**
   *Como verificar:* inspecionar configuração de connection pool (database,
   HTTP, Redis). Todos os quatro parâmetros devem estar presentes.
   Defaults de framework geralmente são inadequados para produção.

7. **Thread pools têm tamanho limitado e bounded queue.**
   *Como verificar:* buscar criação de thread pools. `Executors.newCachedThreadPool()`
   = unbounded = perigoso. Deve usar `ThreadPoolExecutor` com parâmetros
   explícitos ou equivalente.

8. **Bulkheads separam pools por dependência downstream.**
   *Como verificar:* verificar se chamadas para diferentes serviços usam
   pools de recursos independentes. Um pool compartilhado para todas as
   dependências = uma dependência lenta trava tudo.

### Dados e Queries

9. **Toda query que retorna lista tem LIMIT ou paginação.**
   *Como verificar:* buscar queries SQL sem LIMIT e endpoints de API
   que retornam arrays sem parâmetro de paginação (page, offset, cursor).

10. **Endpoints de listagem têm tamanho máximo de página.**
    *Como verificar:* verificar se parâmetros como `pageSize` ou `limit`
    têm valor máximo enforced no backend. `?pageSize=999999` não deve
    ser permitido.

11. **Result sets grandes usam streaming ou cursor, não load completo em memória.**
    *Como verificar:* queries que podem retornar > 10.000 registros devem
    usar cursor-based pagination ou streaming. Grep por `fetchAll`,
    `toList()` em queries potencialmente grandes.

### Health e Observabilidade

12. **Existe health check shallow (`/health`) e deep (`/health/deep`).**
    *Como verificar:* acessar ambos os endpoints. Shallow deve retornar
    < 100ms. Deep deve verificar conectividade com dependências.

13. **Logs são estruturados (JSON) com correlation ID.**
    *Como verificar:* examinar output de log. Deve ser JSON parseável.
    Deve conter campo de correlation/trace ID propagado entre serviços.

14. **Métricas de latência incluem percentis (p50, p95, p99), não apenas média.**
    *Como verificar:* inspecionar dashboards e configuração de métricas.
    Média mascara outliers. p99 revela a experiência dos piores 1%.

15. **Alertas existem para thread pool saturation, não apenas error rate.**
    *Como verificar:* verificar configuração de alertas. Se só há alerta
    para erros HTTP 5xx, blocked threads passam despercebidas.

### Deploy e Operações

16. **Deploy é zero-downtime (blue-green, canary ou rolling).**
    *Como verificar:* inspecionar pipeline de deploy. Se há step de
    "manutenção" ou "downtime programado", é violação.

17. **Feature flags separam deploy de release.**
    *Como verificar:* verificar se funcionalidades novas são deployadas
    desativadas e ativadas via flag. Buscar por sistema de feature
    flags (LaunchDarkly, Unleash, ConfigCat, config própria).

18. **Rollback é possível sem redeploy.**
    *Como verificar:* confirmar que a versão anterior pode ser restaurada
    em < 5 minutos. Feature flags permitem rollback instantâneo.

### Steady State

19. **Logs têm rotação automática configurada.**
    *Como verificar:* verificar logrotate, configuração de logging framework
    (maxSize, maxHistory), ou log shipping para serviço externo.

20. **Caches têm TTL e política de eviction.**
    *Como verificar:* buscar criação de caches. Cache sem TTL ou maxSize
    = memory leak eventual. Toda entrada de cache deve ter expiração.

21. **Sessões têm timeout de expiração.**
    *Como verificar:* verificar configuração de sessão. Sessões sem
    timeout = consumo de memória crescente infinitamente.

22. **Tabelas de dados transacionais têm estratégia de archival ou particionamento.**
    *Como verificar:* verificar se tabelas que crescem continuamente
    (logs, eventos, audit trail) têm policy de retenção ou particionamento.

### Rede e Integração

23. **Connection keep-alive está configurado para chamadas HTTP frequentes.**
    *Como verificar:* verificar se HTTP clients reutilizam conexões.
    Criar nova conexão TCP+TLS para cada request é desperdício.

24. **DNS caching tem TTL adequado (não infinito, não zero).**
    *Como verificar:* verificar configuração de DNS cache da JVM
    (`networkaddress.cache.ttl`), do sistema operacional, ou do
    HTTP client.

25. **Existe rate limiting em endpoints públicos.**
    *Como verificar:* testar envio de 1000 requests em 1 segundo para
    endpoint público. Se todos retornarem 200, não há rate limiting.

### Capacidade e Proteção

26. **Self-denial attacks são prevenidos (cache warming, rate limit em campanhas).**
    *Como verificar:* verificar se há procedimento para pré-aquecer caches
    antes de campanhas. Verificar se links de marketing passam por rate
    limiter ou CDN.

27. **Backpressure está implementado em filas e streams.**
    *Como verificar:* verificar se message queues têm bounded size e se
    producers são notificados quando a fila está cheia. Kafka consumer
    lag é monitorado.

28. **Load shedding está configurado para proteger o sistema sob stress.**
    *Como verificar:* verificar se existe mecanismo de rejeição de requests
    quando a carga excede capacidade (429, 503). Verificar se o sistema
    prefere rejeitar requests a ficar lento para todos.

---

## Anti-patterns com Sinais no Código

| # | Anti-pattern | Sinal no Código | Severidade | Ação |
|---|-------------|-----------------|------------|------|
| 1 | **Integration Point sem proteção** | Chamada HTTP/gRPC sem timeout, sem circuit breaker, sem retry | CRÍTICA | Adicionar timeout + circuit breaker + retry com backoff |
| 2 | **Blocked Threads** | `synchronized` sem timeout, `Future.get()` sem timeout, `Thread.sleep()` em request thread | CRÍTICA | Usar variantes com timeout (`get(5, SECONDS)`), async processing |
| 3 | **Unbounded Result Set** | `SELECT *` sem LIMIT, `findAll()` sem paginação, API retornando lista sem limite | ALTA | Adicionar paginação obrigatória e LIMIT em queries |
| 4 | **Unbounded Thread Pool** | `Executors.newCachedThreadPool()`, thread creation sem limite | ALTA | Usar `ThreadPoolExecutor` com max threads e bounded queue |
| 5 | **Retry sem backoff** | `retry(3)` sem delay, loop de retry com `sleep(1000)` fixo | ALTA | Implementar exponential backoff com jitter |
| 6 | **Cache sem eviction** | `HashMap` usado como cache, cache sem TTL nem maxSize | ALTA | Usar cache library com TTL e maxSize (Caffeine, Guava, Redis com TTL) |
| 7 | **Log sem rotation** | Logs escritos em arquivo sem logrotate, sem maxSize, sem maxHistory | ALTA | Configurar rotação automática ou log shipping |
| 8 | **SLA Inversion** | SLA prometido é mais alto que o SLA de qualquer dependência sem fallback | ALTA | Implementar fallback para dependências ou reduzir SLA prometido |
| 9 | **Single point of failure** | Uma instância de serviço crítico, banco sem replica, cache sem redundância | ALTA | Redundância em componentes críticos |
| 10 | **Deploy com downtime** | Script de deploy com `stop server`, `deploy`, `start server` | MÉDIA | Migrar para blue-green ou rolling deployment |
| 11 | **Health check inadequado** | Endpoint `/health` que retorna 200 fixo sem verificar dependências | MÉDIA | Implementar deep health check que verifica conectividade |
| 12 | **Shared resource pool** | Um único connection pool usado para todas as dependências downstream | MÉDIA | Bulkhead pattern — pool separado por dependência |
| 13 | **Log não-estruturado** | `System.out.println()`, `console.log()` com string concatenada | MÉDIA | Migrar para structured logging (JSON) com campos definidos |
| 14 | **Sessão sem expiração** | Sessão HTTP sem `maxInactiveInterval`, tokens sem expiração | MÉDIA | Configurar TTL para sessões e tokens |
| 15 | **Capacidade desbalanceada** | Frontend com auto-scaling mas backend com instâncias fixas | MÉDIA | Capacity planning end-to-end, rate limiting entre camadas |
| 16 | **Feature deployada sem flag** | Funcionalidade nova ativada automaticamente em produção | MÉDIA | Wrapping em feature flag para controle de release |
| 17 | **DNS hardcoded** | IP address hardcoded em configuração, sem service discovery | BAIXA | Usar DNS ou service discovery |
| 18 | **Monitoring apenas de erros** | Alertas só para HTTP 5xx, sem monitoramento de latência ou saturação | MÉDIA | Adicionar alertas para p99 latência, thread pool usage, connection pool usage |

---

## Checklist de Revisão

### Antes de cada Code Review, pergunte:

**Resiliência de Integração**
- [ ] Toda chamada externa tem timeout explícito?
- [ ] Existe circuit breaker para cada dependência?
- [ ] O fallback de cada circuit breaker está definido e testado?
- [ ] Retries usam backoff exponencial com jitter?
- [ ] O timeout budget total da operação está definido?

**Proteção de Recursos**
- [ ] Thread pools são bounded?
- [ ] Connection pools estão configurados (min, max, idle, lifetime)?
- [ ] Existe bulkhead separando pools por dependência?
- [ ] Queries retornam result sets limitados (LIMIT, paginação)?
- [ ] Caches têm TTL e maxSize?

**Steady State**
- [ ] O sistema pode rodar por 30 dias sem intervenção humana?
- [ ] Logs são rotacionados automaticamente?
- [ ] Sessões expiram?
- [ ] Dados históricos são arquivados?
- [ ] Disk space é monitorado?

**Observabilidade**
- [ ] Logs são estruturados com correlation ID?
- [ ] Métricas de latência usam percentis?
- [ ] Distributed tracing está propagando trace IDs?
- [ ] Alertas cobrem saturação além de erros?
- [ ] Dashboards mostram estado de circuit breakers?

**Deploy**
- [ ] O deploy é zero-downtime?
- [ ] Novas features estão atrás de feature flags?
- [ ] Rollback pode ser feito em < 5 minutos?
- [ ] Database migrations são backward-compatible?
- [ ] Health checks estão implementados (shallow e deep)?

**Capacidade**
- [ ] O sistema foi testado com carga realista de produção?
- [ ] Existe rate limiting em endpoints públicos?
- [ ] Backpressure está implementado em filas?
- [ ] Capacidade está balanceada entre camadas?
- [ ] Cache stampede está prevenido?

**Rede**
- [ ] Conexões HTTP usam keep-alive?
- [ ] DNS cache tem TTL adequado?
- [ ] Connection pools sobrevivem a firewall idle timeouts?
- [ ] TLS está configurado com connection reuse?

---

## Perguntas que esta Skill Faz

1. **O que acontece quando este serviço downstream fica 10x mais lento que o normal?**
   O sistema degrada graciosamente ou trava completamente?

2. **Qual é o blast radius se este componente falhar?**
   A falha é contida ou se propaga para todo o sistema?

3. **Este sistema pode rodar por 90 dias sem intervenção humana?**
   O disco vai encher? A memória vai crescer indefinidamente? Connection pools
   vão vazar?

4. **Qual é o SLA real deste sistema considerando todas as dependências?**
   Multiplique a disponibilidade de cada dependência. O resultado é menor do
   que o SLA prometido?

5. **O que acontece se o marketing enviar um email para 5 milhões de usuários
   com link direto para este endpoint?**
   O sistema aguenta? Existe rate limiting? Cache está aquecido?

6. **Existe um timeout em toda operação externa?**
   Sem exceção. HTTP, database, file system, message broker — tudo.

7. **O que o circuit breaker faz quando abre?**
   Existe fallback? O usuário vê um erro útil ou uma página em branco?

8. **Quantas threads podem ficar bloqueadas antes do sistema parar?**
   Se o thread pool tem 200 threads e a dependência trava, quanto tempo
   até esgotar?

9. **O que acontece se esta query retornar 10 milhões de registros?**
   O sistema faz pagination? Streaming? Ou carrega tudo em memória?

10. **Este sistema sobrevive a um deploy?**
    Connections são drenadas graciosamente? Sessões são preservadas?
    Requests in-flight são completados?

11. **Se eu precisar fazer rollback em 2 minutos, consigo?**
    Feature flag? Blue-green switch? Ou preciso de um deploy completo?

12. **Onde estão os pontos de comunicação O(N²)?**
    Algum componente fala diretamente com todos os outros sem intermediário?

13. **Se o cache for completamente invalidado, o sistema sobrevive?**
    Cache stampede protection existe? O banco aguenta a carga total?

14. **Os health checks refletem a saúde real do sistema?**
    `/health` retorna 200 mesmo quando o banco está inacessível?

15. **Consigo distinguir um problema neste serviço de um problema em
    uma dependência olhando os logs e métricas?**
    A observabilidade permite diagnóstico rápido?

16. **Quais recursos são compartilhados entre funcionalidades e podem
    ser ponto de contenção?**
    Um thread pool, um connection pool, um banco — usado por tudo?

17. **Este sistema foi testado com as condições de falha que vai
    encontrar em produção?**
    Slow response, partial failure, garbage response — ou apenas happy path?

---

## Vocabulário

| Termo | Definição |
|-------|-----------|
| **Cynical Software** | Software projetado com a premissa de que toda interação externa pode e vai falhar de formas inesperadas |
| **Stability Pattern** | Padrão de design que aumenta a capacidade do sistema de absorver falhas sem colapsar |
| **Stability Anti-pattern** | Padrão que amplifica falhas e reduz a estabilidade do sistema |
| **Circuit Breaker** | Mecanismo que monitora falhas em uma dependência e interrompe chamadas quando o threshold é atingido, evitando propagação de falhas |
| **Bulkhead** | Isolamento de recursos (threads, conexões) por dependência para conter o blast radius de falhas |
| **Blast Radius** | Extensão do impacto de uma falha — quantos componentes, serviços ou usuários são afetados |
| **Cascading Failure** | Falha que se propaga entre serviços/camadas, onde a falha de A causa falha em B, que causa falha em C |
| **Chain Reaction** | Falha que se propaga dentro de uma camada — um nó falha e sobrecarrega os demais |
| **Fail Fast** | Princípio de que se uma operação vai falhar, deve falhar imediatamente para não consumir recursos |
| **Backpressure** | Mecanismo pelo qual o consumidor sinaliza ao produtor para desacelerar quando não consegue acompanhar |
| **Shed Load** | Rejeição deliberada de requests em excesso para proteger o sistema de sobrecarga total |
| **Steady State** | Capacidade do sistema de rodar indefinidamente sem intervenção humana para limpeza de recursos |
| **SLA Inversion** | Situação onde o SLA prometido é maior que a disponibilidade matemática das dependências |
| **Test Harness** | Simulador de falhas de produção usado em testes — slow responses, connection failures, garbage data |
| **Timeout Budget** | Distribuição do timeout total de uma operação entre as chamadas individuais que a compõem |
| **Cache Stampede** | Condição onde expiração de cache causa avalanche de requests simultâneas ao backend |
| **Retry Storm** | Amplificação de carga causada por retries simultâneos de múltiplos clientes |
| **Self-Denial Attack** | Condição onde o próprio sistema ou organização gera carga que causa outage |
| **Zero-downtime Deploy** | Estratégia de deploy que não interrompe o serviço para usuários em nenhum momento |
| **Feature Flag** | Mecanismo para ativar/desativar funcionalidades em runtime sem redeploy |
| **Health Check (Shallow)** | Verificação rápida de que o processo está vivo e aceitando conexões |
| **Health Check (Deep)** | Verificação completa que testa conectividade com todas as dependências |
| **Correlation ID** | Identificador único propagado entre serviços para rastrear um request em todo o sistema |
| **Unbalanced Capacities** | Condição onde diferentes camadas do sistema têm capacidades de processamento incompatíveis |
| **Decoupling Middleware** | Infraestrutura de mensageria (queues, topics) que desacopla temporalmente produtor e consumidor |
| **Handshaking** | Protocolo onde o servidor sinaliza sua capacidade (503, 429) e o cliente respeita essa sinalização |
| **Connection Pool** | Conjunto gerenciado de conexões reutilizáveis que evita o custo de criação/destruição por operação |
| **Longevity Testing** | Teste que roda o sistema por período prolongado (72h+) para revelar problemas de steady state |
