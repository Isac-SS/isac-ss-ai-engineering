# Skill: Monolito vs Microservices — Vernon & Jaskula, Newman, Cases Reais

> **Fontes consolidadas:**
> - *Strategic Monoliths and Microservices* — Vaughn Vernon & Tomasz Jaskula (2022)
> - *Monolith to Microservices* — Sam Newman (2019)
> - Cases reais: Amazon Prime Video (2023), Shopify (modular monolith), Segment, Basecamp/DHH, Istio
>
> **Propósito desta skill:** ser o contrapeso crítico à hype de microservices. Questionar toda decisão
> de decomposição. Perguntar "mas vocês realmente precisam disso?" — e ao mesmo tempo reconhecer
> quando microservices SÃO a escolha correta.
>
> **Tom:** direto, desafiador, baseado em evidências. Arquitetura não é moda.

---

## Perspectiva dos Autores

### Vernon & Jaskula — Purposeful Architecture

Vernon e Jaskula defendem que arquitetura deve servir a um propósito de negócio concreto, não seguir
tendências tecnológicas. O conceito central do livro é **Purposeful Architecture**: toda decisão
arquitetural deve responder à pergunta "que capacidade de negócio estamos otimizando?".

Para Vernon, o monolito **não é um degrau** para microservices. É uma arquitetura legítima, completa
e frequentemente superior. Um monolito bem projetado com boundaries claros é objetivamente melhor do
que microservices mal projetados — e a maioria dos microservices no mercado são mal projetados.

Vernon argumenta que a indústria inverteu a lógica: em vez de escolher a arquitetura que serve o
negócio, as organizações escolhem microservices por pressão social e depois tentam justificar a
decisão. Isso é o oposto de Purposeful Architecture.

Jaskula complementa com a perspectiva prática: a complexidade operacional de microservices é um
**custo real** que precisa ser contabilizado. Não é "complexidade acidental que vamos resolver com
tooling" — é complexidade inerente à distribuição que consome budget, tempo e energia cognitiva.

### Newman — A Evolução de Quem Escreveu O Livro de Microservices

Newman é o autor de *Building Microservices* (2015), frequentemente citado como a bíblia do estilo.
Porém, a posição de Newman **evoluiu significativamente**. Em *Monolith to Microservices* (2019),
ele dedica capítulos inteiros a avisar contra decomposição prematura.

Newman afirma explicitamente: microservices não são um objetivo — são um meio. Se o meio não serve
ao fim, ele está errado. Newman defende que a migração para microservices deve ser **incremental**,
**reversível** e motivada por problemas concretos — nunca por ambição arquitetural abstrata.

Um dos pontos mais fortes de Newman é o conceito de **independent deployability** como o teste
definitivo: se os serviços não são deployados independentemente na prática, eles não são
microservices — são um distributed monolith com overhead extra.

### DHH / Basecamp — "The Majestic Monolith"

David Heinemeier Hansson (criador do Rails, CTO do Basecamp/37signals) cunhou o termo
"The Majestic Monolith" para defender que a vasta maioria das empresas não é Google, Netflix ou
Amazon — e não precisa da arquitetura delas.

A tese de DHH é provocadora e direta: times pequenos (menos de 50 desenvolvedores) quase nunca
precisam de microservices. O overhead operacional — deployment, monitoring, debugging, tracing,
service mesh — é um **imposto** que times pequenos não conseguem pagar sem sacrificar features
de negócio.

DHH não é contra microservices em princípio. Ele é contra a adoção irrefletida por empresas que
não têm escala, complexidade organizacional ou maturidade operacional para justificá-los.

---

## Princípios Fundamentais

### O Espectro Arquitetural

A decisão entre monolito e microservices **não é binária**. Existe um espectro com pelo menos
cinco pontos distintos, cada um com trade-offs específicos:

| Ponto do Espectro | Descrição | Deploy Units | Complexidade Operacional | Complexidade de Coordenação |
|---|---|---|---|---|
| **Single-process Monolith** | Uma aplicação, um processo, um deployable. Tradicional e simples. | 1 | Mínima | Cresce com o time |
| **Modular Monolith** | Um deployable com boundaries explícitos entre módulos, enforced por tooling. | 1 | Mínima | Gerenciável com enforcement |
| **Service-based Architecture** | Poucos serviços coarse-grained (4-12), cada um cobrindo um domínio amplo. | 4-12 | Moderada | Moderada |
| **Microservices** | Muitos serviços fine-grained (dezenas a centenas), cada um com uma responsabilidade específica. | 20-200+ | Alta | Baixa por serviço, alta no agregado |
| **Nano/Serverless** | Funções como unidade de deploy. Cada function é independente. | 100-1000+ | Muito alta (observability) | Muito baixa por função |

**Mover para a direita** no espectro adiciona complexidade operacional: mais deploys, mais
monitoramento, mais pontos de falha na rede, mais necessidade de distributed tracing.

**Mover para a esquerda** adiciona complexidade de coordenação: mais merge conflicts, mais deploy
locks, mais necessidade de comunicação entre times que compartilham o mesmo deployable.

A pergunta correta não é "monolito ou microservices?" — é "onde neste espectro o custo operacional
e o custo de coordenação se equilibram para NOSSA organização, NOSSO domínio e NOSSO time?"

Vernon argumenta que para a maioria das organizações, o ponto de equilíbrio está no **Modular
Monolith** ou na **Service-based Architecture** — não em microservices puros.

### Purposeful Architecture (Vernon & Jaskula)

Toda decisão arquitetural deve ser driven por propósito de negócio, não por tendência tecnológica.
Vernon e Jaskula propõem três perguntas-filtro obrigatórias:

1. **"Que capacidade de negócio estamos otimizando?"** — Se a resposta é "escalabilidade" mas o
   sistema recebe 100 requests por segundo, a otimização é prematura. Se a resposta é "velocity de
   desenvolvimento" mas o time tem 8 pessoas, microservices provavelmente vão diminuir a velocity.

2. **"Qual é o custo do overhead operacional versus o custo da coordenação?"** — Microservices
   eliminam coordenação de deploy mas adicionam coordenação operacional. A conta precisa fechar.
   Se o time gasta mais tempo em infraestrutura do que em features, a conta não fecha.

3. **"A organização tem maturidade para sistemas distribuídos?"** — Microservices exigem CI/CD por
   serviço, observability robusta, distributed tracing, estratégias de retry/circuit-breaker,
   gestão de backward compatibility de APIs. Se a organização não tem isso, microservices vão
   amplificar problemas, não resolvê-los.

O insight central de Vernon: **um monolito bem projetado com boundaries claros é MELHOR do que
microservices mal projetados**. E a maioria dos microservices no mercado são mal projetados porque
as organizações adotam a arquitetura antes de ter maturidade para operá-la.

Purposeful Architecture significa que a arquitetura deve corresponder à maturidade da organização,
ao tamanho do time e ao nível de entendimento do domínio. Adotar microservices sem entender o
domínio é como otimizar performance antes de ter testes — está resolvendo o problema errado.

### O Modular Monolith como Arquitetura de Primeira Classe

O Modular Monolith **não é "um monolito esperando para ser decomposto"**. Ele **é** a arquitetura
alvo para muitos sistemas — possivelmente para a maioria. Vernon e Jaskula são enfáticos: tratar
o modular monolith como estágio intermediário é Purposeless Architecture.

**Características essenciais:**

- **Boundaries explícitos enforced por tooling** — não por convenção, não por documentação, não
  por boa vontade. Por tooling que quebra o build se alguém violar um boundary.
- **Cada módulo tem uma API pública** — internals são privados. Nenhum código externo ao módulo
  acessa classes, funções ou constantes internas.
- **Módulos se comunicam por interfaces bem definidas** — não por acesso direto ao banco de dados
  de outro módulo, não por import de classes internas.
- **Testável independentemente** — cada módulo pode ter sua suite de testes executada isoladamente.
- **Compartilha deployment unit mas não detalhes de implementação** — um deploy, mas com isolamento
  interno real.
- **Database com schema ownership por módulo** — o banco pode ser compartilhado, mas cada módulo
  é dono das suas tabelas. Nenhum módulo faz SELECT direto em tabelas de outro módulo.

**Case: Shopify — Modular Monolith com Packwerk em Rails**

Shopify opera uma das maiores aplicações Rails do mundo. Com mais de 300 desenvolvedores
contribuindo para o mesmo codebase, a empresa deliberadamente escolheu o modular monolith em vez
de microservices. A ferramenta central é **Packwerk**, criada pelo próprio time de Shopify.

Como funciona na prática:
- Packwerk **enforce boundaries entre módulos no build time**. Se um módulo tenta acessar internals
  de outro módulo, o CI falha. Não é sugestão — é enforcement.
- Módulos declaram **dependências explícitas**. O grafo de dependências é visível e gerenciável.
- **Privacy é enforced**: não é possível acessar internals de outro módulo sem violar o build.
- O codebase é grande, complexo e atende milhões de merchants — e funciona como monolito modular.

O ponto crucial: Shopify **poderia** adotar microservices. Tem budget, tem engenharia de classe
mundial, tem escala. Mas **escolheu não fazê-lo** porque o modular monolith serve melhor. Isso
não é limitação — é Purposeful Architecture.

**Case: Basecamp / DHH — "The Majestic Monolith"**

DHH argumenta com dados da própria experiência:
- Times pequenos (menos de 50 desenvolvedores) quase nunca precisam de microservices.
- O overhead operacional de microservices — deployment pipelines por serviço, monitoring
  distribuído, debugging cross-service, distributed tracing — é um **imposto** que times
  pequenos não conseguem pagar.
- Um monolito bem estruturado é mais fácil de entender, debugar e evoluir.
- O Basecamp/HEY roda em monolito Rails com um time relativamente pequeno, serve milhões de
  usuários e é lucrativo — sem Kubernetes, sem service mesh, sem distributed tracing.

### Quando Microservices São Justificados

Newman e Vernon convergem nas condições que justificam microservices. **Todas** as condições abaixo
devem ser verdadeiras — não algumas, todas:

1. **Múltiplos times precisam de deployment verdadeiramente independente.** Conway's Law está
   pressionando: times diferentes precisam deployar em cadências diferentes sem coordenação.
   Se todos deployam juntos na sexta-feira, não há independent deployability real.

2. **Boundaries de domínio são bem compreendidos e estáveis.** Se o time ainda está explorando o
   domínio, decompor é prematuro. Newman é enfático: "decomponha o que você conhece, não o que
   você está descobrindo." Boundaries errados em microservices são ordens de grandeza mais caros
   de corrigir do que boundaries errados em um monolito.

3. **A organização tem maturidade operacional.** CI/CD por serviço, observability (métricas, logs,
   traces), distributed tracing (Jaeger, Zipkin, OpenTelemetry), estratégias de resiliência
   (circuit breakers, retries, timeouts, bulkheads), gestão de API versioning. Se algum desses
   itens é "vamos implementar depois", microservices vão causar dor.

4. **O custo de coordenação excede o custo de distribuição.** Deploy locks frequentes, merge
   conflicts diários, release trains lentos, times bloqueados esperando outros times — esses são
   sinais reais de que a coordenação está cara demais. Mas o custo de distribuição (latência de
   rede, eventual consistency, debugging distribuído) precisa ser menor.

5. **Partes diferentes do sistema têm requisitos genuinamente diferentes de escalabilidade ou
   disponibilidade.** O módulo de checkout precisa de 99.99% de uptime mas o módulo de relatórios
   pode tolerar 99.9%. O módulo de busca precisa de 10x mais instâncias que o módulo de admin.
   Se tudo escala igual, esse argumento não se aplica.

6. **O time tem experiência com failure modes de sistemas distribuídos.** Network partitions,
   partial failures, eventual consistency, message ordering, idempotency, distributed transactions
   (ou a ausência delas). Se a equipe não sabe o que é o CAP theorem na prática, microservices
   vão ensinar da pior forma possível — em produção.

**Se QUALQUER uma dessas condições é falsa, o modular monolith é provavelmente a escolha melhor.**

### Quando Microservices Fazem Mais Mal que Bem

Sinais de que a decomposição foi prematura, errada ou ambas:

- **Distributed monolith**: serviços precisam ser deployados juntos. Se o deploy do serviço A
  requer o deploy simultâneo do serviço B, eles não são independentes — são um monolito com
  latência de rede adicionada.
- **Network latency domina o response time**: o que antes era uma chamada de método local agora
  é HTTP/gRPC pela rede. Se a soma das latências de rede é maior que o tempo de processamento,
  a decomposição adicionou custo sem valor.
- **Debugging requer 5+ logs de serviços**: para entender um único request, o desenvolvedor precisa
  correlacionar logs de múltiplos serviços, abrir múltiplos dashboards, usar ferramentas de
  distributed tracing. Se isso é o dia-a-dia, o custo operacional é real e alto.
- **Mais tempo em infraestrutura do que em features de negócio**: se o time gasta mais horas
  configurando CI/CD, Kubernetes manifests, Helm charts, service mesh e monitoring do que
  escrevendo código de negócio, a arquitetura está servindo a si mesma, não ao negócio.
- **Problemas de data consistency que não existiam no monolito**: o que antes era uma transação
  ACID local agora requer sagas, compensações e eventual consistency. Se o domínio tem requisitos
  fortes de consistência, a distribuição adicionou complexidade essencial sem benefício.
- **Mais tempo em reuniões de coordenação cross-team do que codando**: irônico — microservices
  deveriam reduzir coordenação, mas se os boundaries estão errados, a coordenação aumenta.
- **Toda feature requer mudanças em 3+ serviços**: sinal clássico de boundaries errados. A feature
  deveria caber em um serviço. Se não cabe, os boundaries não refletem o domínio.
- **Testes requerem subir 10+ serviços localmente**: se o ambiente local de desenvolvimento precisa
  de docker-compose com 15 serviços para rodar um teste end-to-end, a produtividade do
  desenvolvedor está comprometida.

**Case: Amazon Prime Video (2023)**

A equipe de monitoramento de qualidade de vídeo do Amazon Prime Video migrou de microservices
(Step Functions + Lambda) para monolito. Resultados:
- **90% de redução de custo** de infraestrutura.
- Mais simples de operar, debugar e evoluir.
- O insight chave: microservices adicionaram overhead sem adicionar valor para o caso de uso
  específico. O processamento de vídeo era inerentemente sequencial e acoplado — forçá-lo em
  microservices criou complexidade artificial.
- A equipe não voltou "ao legado" — construiu um monolito bem projetado com conhecimento adquirido.

**Case: Segment**

Segment é um dos casos mais documentados de ida e volta:
- Começou como monolito. Migrou para microservices.
- Microservices criaram o que a equipe chamou de **"death by a thousand queues"** — centenas de
  filas, cada uma com suas peculiaridades, failure modes e backlogs.
- O overhead operacional superou os benefícios para o tamanho do time.
- Voltaram para um monolito (repensado, não o original).
- Lição: a arquitetura deve corresponder ao time, não o contrário.

**Case: Istio**

O service mesh Istio era composto por múltiplos microservices (Pilot, Mixer, Citadel, Galley).
A equipe decidiu **reconsolidar** tudo em um único binário chamado **istiod**. Motivos:
- A complexidade operacional de deployar e configurar múltiplos componentes era alta.
- Os componentes mudavam juntos na prática — não havia independent deployability real.
- Um binário é mais simples de instalar, configurar, monitorar e debugar.
- A ironia: o projeto que facilita microservices para outros decidiu que microservices não serviam
  para si mesmo.

### Reconsolidação — Voltando ao Monolito

Quando microservices não funcionaram, existem patterns para reconsolidar sem caos:

1. **Identifique serviços que sempre mudam juntos** — se o serviço A e o serviço B são modificados
   na mesma PR em mais de 50% dos casos, eles devem ser um módulo dentro de um mesmo deployable.
   Merge them.

2. **Identifique serviços com banco de dados compartilhado** — se dois serviços acessam as mesmas
   tabelas, eles nunca foram verdadeiramente independentes. A separação é ilusória. Merge ou
   separe os dados de verdade.

3. **Identifique serviços owned pelo mesmo time** — se um time é dono de 5 serviços, considere
   mergeá-los. Conway's Law em reverso: se a estrutura do time não requer separação, a
   arquitetura provavelmente também não.

4. **Mantenha serviços que SÃO deployados independentemente por times diferentes** — se o time de
   pagamentos deploya o serviço de pagamentos independentemente do time de catálogo, essa
   separação faz sentido. Preserve-a.

5. **O resultado geralmente é uma service-based architecture** — menos serviços, mais coarse-grained
   (4-12), não de volta ao single monolith. É o meio-termo pragmático.

**A nuance de Newman:** "voltar" não significa fracasso. Significa que a arquitetura está evoluindo
para corresponder à realidade. Arquitetura é iterativa, não uma decisão permanente.

**A nuance de Vernon:** nunca foi sobre monolito OU microservices — sempre foi sobre Purposeful
Architecture correspondendo à necessidade do negócio. Se a necessidade mudou, a arquitetura muda.
Se a necessidade foi mal compreendida, a arquitetura corrige.

### Enforcement de Boundaries — A Chave do Modular Monolith

Sem enforcement, um modular monolith é apenas um monolito com boas intenções. Vernon é direto:
**boundaries não enforced são boundaries imaginários**.

**Ferramentas de enforcement por linguagem/ecossistema:**

| Linguagem | Ferramenta | Mecanismo |
|---|---|---|
| Ruby/Rails | **Packwerk** (Shopify) | Analisa referências entre packages no build; falha CI se boundary violado |
| Ruby/Rails | dry-rb boundaries | Encapsulamento via containers e dependency injection |
| Ruby/Rails | Custom RuboCop rules | Regras customizadas para proibir imports cross-module |
| Java | **ArchUnit** | Testes de arquitetura: "classes no package X não podem depender do package Y" |
| Java | Java 9 Modules (JPMS) | Module system nativo: exports explícitos, strong encapsulation |
| Java | Spring Modulith | Framework para modular monoliths em Spring Boot, com verificação de boundaries |
| .NET | Roslyn Analyzers | Analyzers customizados que falham build em violações |
| .NET | Project references | Separação por projetos com referências explícitas |
| TypeScript/Node | eslint-plugin-boundaries | Regras ESLint para controlar imports entre módulos |
| TypeScript/Node | **Nx** workspace rules | Constraints de dependência entre libraries/apps no monorepo |
| Python | **import-linter** | Contratos de importação que falham CI se violados |
| Python | tach | Enforcement de boundaries de importação |
| Go | internal packages | Convenção nativa: packages em `internal/` só são acessíveis pelo parent |
| Kotlin | ArchUnit (JVM) | Mesmas capacidades que para Java |

**O que enforçar:**

1. **Sem acesso direto a classes/constantes/funções cross-module** — toda comunicação entre módulos
   deve usar a API pública do módulo. Se o módulo de `orders` precisa de dados do módulo de
   `customers`, ele chama `Customers.API.get_customer(id)`, não `Customers::Internal::CustomerRecord.find(id)`.

2. **Sem acesso cross-module a tabelas do banco** — cada módulo é dono do seu schema. Se `orders`
   precisa de dados de `customers`, ele pede via API, não faz JOIN direto. Isso pode parecer
   ineficiente, mas é o preço da modularidade. Se a ineficiência é inaceitável, talvez os
   módulos devam ser um só.

3. **Direção de dependência: grafo acíclico** — módulos formam um DAG (Directed Acyclic Graph).
   Dependências circulares são proibidas. Se A depende de B e B depende de A, eles são um módulo
   ou precisam de uma abstração intermediária (evento, interface).

4. **Comunicação event-based para side effects cross-module** — mesmo processo, mas desacoplado.
   Quando `orders` completa um pedido, emite um evento `OrderCompleted`. O módulo `notifications`
   escuta e envia email. Não há chamada direta de `orders` para `notifications`.

**Métricas para acompanhar:**

| Métrica | O que mede | Alvo |
|---|---|---|
| Boundary violations | Número de violações de boundary detectadas pelo tooling | Zero (ou decrescente) |
| Afferent/Efferent coupling | Acoplamento entre módulos (quantos módulos dependem de X / de quantos X depende) | Baixo e estável |
| Module size (LOC, classes) | Tamanho dos módulos | Se cresce muito, precisa ser dividido |
| Change coupling | Se módulo A muda, com que frequência módulo B muda junto? | Baixo (< 20%) |
| Cycle count | Número de dependências circulares entre módulos | Zero |

### Team Topologies e Arquitetura

Vernon conecta diretamente a escolha arquitetural à estrutura de times, baseando-se em Team
Topologies de Matthew Skelton e Manuel Pais:

- **Stream-aligned teams** são donos de um módulo ou serviço end-to-end. Cada team é responsável
  por uma stream de valor de negócio, do código ao deploy ao monitoramento em produção.
- **Platform teams** fornecem infraestrutura como serviço — CI/CD, observability, deployment
  pipelines. Eles habilitam os stream-aligned teams a serem autônomos.
- **Enabling teams** ajudam stream-aligned teams a adotar novas práticas — por exemplo, ajudar
  a adotar event-driven architecture ou melhorar práticas de testing.
- **Complicated subsystem teams** são donos de componentes tecnicamente complexos — por exemplo,
  um engine de machine learning ou um algoritmo de precificação — que requerem expertise
  especializada.

**O insight chave de Vernon:**

Se a organização tem 3 stream-aligned teams, provavelmente precisa de 3 módulos ou serviços.
**Não 30.** O número de unidades independentemente deployáveis deve corresponder aproximadamente
ao número de stream-aligned teams.

- **Mais serviços do que times = overhead sem benefício.** Se um time é responsável por 8 serviços,
  ele carrega o custo operacional de 8 mas não obtém o benefício de independent deployability
  (porque é o mesmo time deployando tudo).
- **Menos serviços do que times = acoplamento e overhead de coordenação.** Se 5 times trabalham no
  mesmo serviço, há merge conflicts, deploy locks e release trains.

A regra prática: **1 stream-aligned team : 1-3 módulos/serviços**. Se a razão é muito diferente
disso, há um descompasso entre arquitetura e organização.

---

## Heurísticas Verificáveis

Regras práticas para tomada de decisão e validação contínua. Cada heurística inclui fonte,
critério e método de verificação.

### Decisão Arquitetural Inicial

**H01.** Se o time tem menos de 50 desenvolvedores E não há requisitos de deployment independente
por equipe, o **modular monolith é o default**. (Vernon)
- *Verificação:* Conte o número de desenvolvedores. Pergunte: "Quais times precisam deployar
  independentemente hoje — não em teoria, na prática?"

**H02.** Se o domínio ainda não é bem compreendido (produto novo, pivotando, explorando), **não
decomponha em microservices**. Comece com monolito modular e decomponha depois, se necessário.
(Newman)
- *Verificação:* "O Bounded Context map está estável há pelo menos 6 meses? Se não, os boundaries
  vão mudar — e mudar boundaries em microservices é 10x mais caro."

**H03.** Se a organização não tem CI/CD por serviço, observability e distributed tracing em
produção **hoje**, microservices vão amplificar problemas. (Vernon & Newman)
- *Verificação:* "Existe pipeline de CI/CD independente para cada serviço? Existe dashboard de
  métricas por serviço? Existe distributed tracing configurado e sendo usado?"

**H04.** Antes de adotar microservices, calcule o **custo total de ownership**: infraestrutura,
tooling (service mesh, API gateway, message broker), tempo do time em operação, on-call rotation
expandida. Se o custo excede 30% do budget de engenharia, reconsidere. (Vernon)
- *Verificação:* Planilha de custo real — horas de engenharia em infra vs features, custos de
  cloud por serviço, custo de tooling.

### Saúde do Modular Monolith

**H05.** Se não existe enforcement de boundaries (Packwerk, ArchUnit, import-linter, etc.), o
"modular monolith" é **wishful thinking**. Boundaries não enforced degradam em semanas.
- *Verificação:* "Existe tooling no CI que falha o build em violação de boundary? Qual? Quando
  foi a última vez que falhou?"

**H06.** Cada módulo deve ter uma **API pública explícita**. Se qualquer código externo acessa
internals de outro módulo, o boundary está quebrado.
- *Verificação:* Code review + tooling. Grep por imports/requires de caminhos internos de outros
  módulos.

**H07.** Cada módulo deve ter **schema ownership** no banco de dados. Se o módulo A faz SELECT em
tabelas do módulo B, a modularidade é ilusória.
- *Verificação:* Análise de queries. Grep por nomes de tabelas de outros módulos nas queries.

**H08.** O grafo de dependências entre módulos deve ser um **DAG** (sem ciclos). Dependências
circulares indicam que os módulos não estão bem delimitados.
- *Verificação:* Tooling de análise de dependências (Packwerk gera o grafo, ArchUnit testa ciclos).

**H09.** Se um módulo tem mais de 50.000 linhas de código, ele provavelmente precisa ser dividido.
Se tem menos de 500, ele provavelmente deveria ser absorvido por outro. (Heurística de tamanho)
- *Verificação:* `cloc` ou ferramenta equivalente por módulo.

**H10.** Módulos devem poder ser **testados isoladamente**. Se o teste de um módulo requer setup
de outros módulos, o acoplamento é muito alto.
- *Verificação:* Execute a suite de testes de cada módulo individualmente. Se falha sem os outros,
  há acoplamento implícito.

### Saúde de Microservices

**H11.** Se mais de 30% das features exigem mudanças em 3+ serviços, **a decomposição está
errada**. Os boundaries não refletem o domínio. (Newman)
- *Verificação:* Análise de PRs/commits dos últimos 3 meses. Quantas features tocaram 3+ repos
  de serviços?

**H12.** Se os serviços **compartilham banco de dados**, eles são um distributed monolith.
Consolide ou separe os dados de verdade. (Newman)
- *Verificação:* "Quantos serviços se conectam ao mesmo database instance? Quantos acessam as
  mesmas tabelas?"

**H13.** Se um serviço não é deployado independentemente pelo menos **1x por semana**, ele não
precisa ser um serviço separado. O custo operacional não se justifica.
- *Verificação:* Histórico de deploys por serviço nos últimos 3 meses.

**H14.** Se o **rollback de um deploy** requer rollback de outros serviços, independent
deployability é ilusão. (Newman)
- *Verificação:* "Qual é o procedimento de rollback? Envolve outros serviços? O rollback do
  serviço A já quebrou o serviço B?"

**H15.** Cada serviço deve ser **ownable por um único time**. Se dois times são donos do mesmo
serviço, há confusão de ownership. Se um time é dono de 8+ serviços, há overhead excessivo.
(Vernon / Team Topologies)
- *Verificação:* Mapeie serviço-para-time. Cada serviço tem exatamente um time dono?

**H16.** Se o distributed tracing é **necessário para debugar qualquer issue**, a complexidade
operacional de microservices está consumindo recursos significativos.
- *Verificação:* "Quantos minutos leva para debugar um bug end-to-end? Quantos dashboards e
  ferramentas precisa abrir?"

**H17.** Um serviço deve ter **mais código de negócio do que código de infraestrutura** (configs
Kubernetes, Helm charts, Dockerfiles, CI/CD pipelines). Se a razão se inverte, é um nano-service.
- *Verificação:* `cloc` separando código de negócio de configuração/infra.

**H18.** Latência de rede entre serviços deve ser **medida e monitorada**. Se a soma de latências
inter-service representa mais de 50% do response time total, a decomposição está custando
performance sem justificativa.
- *Verificação:* Distributed tracing — waterfall de um request típico. Quanto tempo é rede?

### Migração e Transição

**H19.** A migração de monolito para microservices deve ser **incremental** (Strangler Fig Pattern),
**nunca big bang**. (Newman)
- *Verificação:* "O plano de migração é incremental? Cada passo é revertível? Existe um estado
  intermediário funcional?"

**H20.** Antes de extrair um serviço, **extraia o módulo** dentro do monolito. Se não consegue
criar um módulo com boundaries claros internamente, não vai conseguir criar um serviço
independente. (Newman)
- *Verificação:* "O módulo candidato à extração já existe como módulo isolado no monolito?
  Tem API pública? É testável independentemente?"

**H21.** A migração deve ser **motivada por uma dor concreta**, não por ambição arquitetural.
"O deploy do módulo X está bloqueando o time Y 3x por sprint" é motivo. "Microservices são mais
modernos" não é. (Vernon)
- *Verificação:* "Qual é a dor concreta que estamos resolvendo? Está documentada? Tem métricas?"

**H22.** Nunca migre o domínio inteiro de uma vez. **Comece pelo contexto com menor acoplamento**
com o resto do sistema — o que Newman chama de "the easiest seam to find." (Newman)
- *Verificação:* Análise de acoplamento. Qual módulo tem menos dependências aferentes e eferentes?

**H23.** Se a migração está parada há mais de 6 meses (alguns serviços extraídos, o resto no
monolito, sem progresso), tome uma decisão: **completar ou reverter**. Ficar no meio é o pior
dos mundos — custo operacional de ambas as arquiteturas. (Strangler Fig Abandonment)
- *Verificação:* "Quando foi o último serviço extraído? Há plano e timeline para os próximos?"

### Reconsolidação

**H24.** Serviços que **sempre mudam juntos** (change coupling > 50%) devem ser mergeados. Eles
não são independentes. (Vernon)
- *Verificação:* Análise de change coupling — correlação de PRs/deploys entre serviços.

**H25.** Serviços owned pelo **mesmo time** que não têm requisitos de escala diferentes podem ser
mergeados. Conway's Law em reverso. (Vernon)
- *Verificação:* Mapeamento serviço-para-time + análise de requisitos de escala.

**H26.** Ao reconsolidar, **não simplesmente copie o código junto**. Refatore para criar
boundaries internos (módulos) que preservem a separação conceitual. Evite o Frankenstein Monolith.
- *Verificação:* "Os serviços mergeados foram transformados em módulos com boundaries enforced?
  Ou foi só copy-paste?"

### Comunicação e Dados

**H27.** Se serviços se comunicam de forma **síncrona em cadeia** (A chama B que chama C que chama
D), a latência empilha e a falha cascateia. Use comunicação assíncrona onde possível. (Newman)
- *Verificação:* Mapeie os call chains síncronos. Qual é o chain mais longo? O que acontece se
  o último da cadeia cai?

**H28.** Cada serviço deve ter seu **próprio data store**. Se dois serviços precisam dos mesmos
dados, use eventos para replicar, não compartilhe o banco. (Newman)
- *Verificação:* "Quantos serviços se conectam ao mesmo banco? Existe um plano de separação?"

**H29.** **Eventual consistency** é um trade-off, não um default. Se o domínio exige strong
consistency (transferências financeiras, estoque crítico), microservices adicionam complexidade
sem ganho — e sagas são inerentemente complexas de implementar e debugar.
- *Verificação:* "Quais operações exigem strong consistency? Essas operações cruzam boundaries
  de serviços? Se sim, a decomposição é questionável."

**H30.** Eventos entre serviços devem ter **schema registrado e versionado** (Schema Registry,
AsyncAPI, Protobuf). Eventos sem schema são bomba-relógio de backward compatibility.
- *Verificação:* "Existe schema registry? Os eventos têm versão? O que acontece se o produtor
  muda o formato do evento?"

### Operação e Observability

**H31.** Se o time não tem **runbooks** para cada serviço em produção, microservices vão causar
incidentes prolongados. Cada serviço é um componente que pode falhar independentemente — e cada
falha precisa de procedimento documentado.
- *Verificação:* "Existe runbook para cada serviço? Quando foi atualizado pela última vez?"

**H32.** O **on-call rotation** deve ser sustentável. Se microservices triplicaram o número de
alertas e o time está burned out, a arquitetura está custando pessoas.
- *Verificação:* Número de alertas por semana antes e depois de microservices. Tempo médio de
  resolução de incidentes.

**H33.** Se precisa de **docker-compose com 10+ serviços** para rodar o ambiente local de
desenvolvimento, a produtividade do desenvolvedor está comprometida.
- *Verificação:* "Quanto tempo leva para um novo desenvolvedor ter o ambiente local funcionando?
  Se é mais de meio dia, há um problema."

---

## Anti-patterns com Sinais no Código

| # | Anti-pattern | Descrição | Sinais no Código/Infra | Fonte |
|---|---|---|---|---|
| AP01 | **Distributed Monolith** | Serviços que precisam ser deployados juntos. A pior combinação: complexidade de microservices sem os benefícios. | Lock-step deploys; CI/CD que builda/deploya todos os serviços juntos; integration tests requerem todos os serviços; shared database; versões hardcoded entre serviços. | Newman |
| AP02 | **Premature Decomposition** | Decompor antes de entender os boundaries do domínio. Boundaries errados em microservices são 10x mais caros de corrigir. | Serviços com nomes genéricos ("common-service", "util-service"); boundaries que mudam a cada quarter; features frequentemente requerem mudanças em 3+ serviços. | Newman |
| AP03 | **Microservices Envy** | Adotar microservices porque "Netflix/Google/Amazon fazem". Ignorar que essas empresas têm 10.000+ engenheiros e plataformas internas. | Time de 10 pessoas com Kubernetes, service mesh, API gateway, message broker, distributed tracing e event sourcing; mais YAML do que código; reuniões sobre infraestrutura > reuniões sobre produto. | Vernon / DHH |
| AP04 | **Boundary-less Monolith** | Chamar de "modular monolith" mas sem enforcement. Todo mundo importa tudo de todo lugar. | Sem Packwerk/ArchUnit/import-linter no CI; módulos existem como diretórios mas sem API pública definida; qualquer arquivo importa qualquer outro; circular dependencies abundantes. | Vernon |
| AP05 | **Database Integration** | Serviços comunicando-se via tabelas compartilhadas no banco. O banco é o contrato de integração. | Múltiplos serviços com connection strings para o mesmo banco; JOINs cross-schema; triggers que afetam outros serviços; migrations que requerem coordenação entre times. | Newman |
| AP06 | **Shared Library Coupling** | Lógica de negócio em shared libraries que forçam releases coordenados. A library vira um ponto de acoplamento. | Uma library `common` ou `shared` com lógica de negócio (não só utils); mudança na library requer rebuild/redeploy de múltiplos serviços; library com 50+ classes de domínio. | Newman |
| AP07 | **Nano-service** | Serviço tão pequeno que tem mais código de infraestrutura do que código de negócio. | Dockerfile + Helm chart + CI/CD pipeline + Kubernetes manifests + monitoring config > linhas de código de negócio; serviço com 3 endpoints e 200 linhas de lógica; time gastando mais tempo configurando o serviço do que desenvolvendo features. | Vernon |
| AP08 | **The Distributed Big Ball of Mud** | Sem boundaries nem no monolito nem nos microservices. Microservices que chamam qualquer outro microservice sem design de API. | Grafo de dependências que parece uma teia de aranha; qualquer serviço chama qualquer outro; sem API contracts; sem ownership claro; "quem é dono disso?" é pergunta comum. | Vernon |
| AP09 | **Conway's Law Denial** | Arquitetura que não corresponde à estrutura de times. 3 times mantendo 30 serviços, ou 10 times trabalhando em 1 monolito. | Razão serviço-para-time > 5:1 ou < 1:3; um time fazendo PR em repos de outros times regularmente; ninguém sabe quem é dono de qual serviço. | Vernon / Team Topologies |
| AP10 | **Strangler Fig Abandonment** | Migração começada e abandonada no meio. O sistema agora roda duas arquiteturas — monolito parcial + alguns serviços — com o custo operacional de ambas. | Monolito ainda em produção com serviços parcialmente extraídos; requests que vão do monolito para serviço e voltam; último serviço extraído há 6+ meses; sem roadmap de migração. | Newman |
| AP11 | **Over-orchestration** | Camadas excessivas de API gateways, service meshes, sidecars, message brokers para uma aplicação que não precisa de tudo isso. | 5+ componentes de infraestrutura para rotear um request; latência de 100ms+ só em overhead de mesh/gateway; time dedicado a manter o service mesh; mais configuração de Istio/Envoy do que routes de negócio. | Vernon / DHH |
| AP12 | **Sync Chain** | A chama B que chama C que chama D sincronamente. Latência empilha. Falha cascateia. | Chain de 4+ chamadas síncronas em um request; p99 latency > 2s por causa de cascading calls; um serviço fora do ar derruba a chain inteira; retry storms quando um nó falha. | Newman |
| AP13 | **Entity Service** | Um serviço por entidade de banco (CustomerService, OrderService, ProductService) em vez de por business capability. CRUD distribuído. | Serviços com nomes de entidades; serviços que são basicamente CRUD REST wrappers; features de negócio requerem orquestrar 5 entity services; anemic domain model distribuído. | Vernon |
| AP14 | **The Frankenstein Monolith** | Microservices mergeados de volta sem boundaries internos. Volta a ser big ball of mud mas agora com cicatrizes. | Módulos sem boundaries enforced; código misturando conceitos de diferentes ex-serviços; nomes de packages refletindo serviços antigos sem coesão; sem Packwerk/ArchUnit. | Original |
| AP15 | **The Golden Hammer** | Usar o mesmo estilo arquitetural para tudo — microservices para o CRUD admin, microservices para o batch job, microservices para o script de migração. | Todo componente do sistema é um "serviço" — inclusive ferramentas internas, scripts, jobs; 50 serviços onde 40 são ferramentas internas com 1 deploy por ano. | Vernon |
| AP16 | **API Gateway as Business Logic** | Lógica de negócio no API Gateway (transformações, orquestração, regras de roteamento complexas). O gateway vira um distributed monolith disfarçado. | API Gateway com centenas de regras de transformação; lógica de request/response mapping complexa; quando o gateway cai, nada funciona; deploy do gateway requer testes de regressão de negócio. | Newman |
| AP17 | **Event Soup** | Comunicação event-driven sem schema, sem governance, sem documentação. Eventos viram o novo acoplamento — invisível e impossível de rastrear. | Eventos sem schema registry; sem versionamento de eventos; produtores mudam formato sem avisar consumidores; ninguém sabe quais serviços consomem qual evento; debugging requer correlacionar 10 tópicos de Kafka. | Vernon |
| AP18 | **Resume-Driven Architecture** | Escolhas arquiteturais motivadas pelo que o time quer aprender ou ter no currículo, não pelo que o negócio precisa. | Kubernetes para 3 serviços; event sourcing para um CRUD; CQRS para read-heavy com 100 users; GraphQL federation para 2 serviços; Rust rewrite "para performance" de um sistema I/O-bound. | DHH |

---

## Checklist de Revisão

### Checklist 1: Devemos Usar Microservices?

- [ ] O time tem mais de 50 desenvolvedores contribuindo para o mesmo codebase?
- [ ] Múltiplos times (3+) precisam de independent deployability na prática (não na teoria)?
- [ ] Os Bounded Contexts estão estáveis há pelo menos 6 meses?
- [ ] A organização tem CI/CD por serviço, observability e distributed tracing em produção HOJE?
- [ ] O custo de coordenação (deploy locks, merge conflicts, release trains) foi medido e é alto?
- [ ] Partes do sistema têm requisitos genuinamente diferentes de escala ou disponibilidade?
- [ ] O time tem experiência prática com failure modes distribuídos (network partitions, eventual
  consistency, distributed transactions)?
- [ ] O custo total estimado de microservices (infra + tooling + overhead operacional) foi
  calculado e é aceitável?
- [ ] Alternativas foram consideradas (modular monolith, service-based architecture)?

**Se mais de 3 respostas são "não", microservices provavelmente não são a escolha certa.**

### Checklist 2: Saúde do Modular Monolith

- [ ] Existe tooling de enforcement de boundaries (Packwerk, ArchUnit, import-linter) no CI?
- [ ] O número de boundary violations é zero (ou está decrescendo continuamente)?
- [ ] Cada módulo tem API pública explícita documentada?
- [ ] Nenhum módulo acessa tabelas de outro módulo diretamente?
- [ ] O grafo de dependências é um DAG (sem ciclos)?
- [ ] Cada módulo pode ser testado isoladamente?
- [ ] Existe ownership claro (1 time : 1-3 módulos)?
- [ ] Módulos comunicam-se por interfaces ou eventos, não por internals?
- [ ] Change coupling entre módulos é baixo (< 20%)?
- [ ] Novos módulos são criados com boundary enforcement desde o início (não retrofit)?

### Checklist 3: Saúde de Microservices

- [ ] Cada serviço é deployado independentemente (sem lock-step)?
- [ ] Cada serviço tem seu próprio data store?
- [ ] Cada serviço é ownable por exatamente um time?
- [ ] O rollback de um serviço não requer rollback de outros?
- [ ] Menos de 30% das features requerem mudanças em 3+ serviços?
- [ ] O tempo de debugging end-to-end é razoável (< 30 min para issues típicos)?
- [ ] Distributed tracing está configurado e é usado regularmente?
- [ ] O ambiente local de desenvolvimento não requer 10+ serviços?
- [ ] Cada serviço tem mais código de negócio do que código de configuração/infra?
- [ ] Existe schema registry para eventos e API contracts versionados?
- [ ] Runbooks estão atualizados para cada serviço?
- [ ] O on-call rotation é sustentável (sem burnout)?

### Checklist 4: Readiness para Migração (Monolito para Serviços)

- [ ] A dor concreta que motiva a migração está documentada com métricas?
- [ ] O módulo candidato já existe como módulo isolado no monolito com boundaries enforced?
- [ ] O módulo candidato tem o menor acoplamento possível com o resto do sistema?
- [ ] O Strangler Fig Pattern está planejado (incremental, revertível)?
- [ ] A infraestrutura target está pronta (CI/CD, observability, tracing)?
- [ ] O time que será dono do novo serviço tem experiência com operação distribuída?
- [ ] Existe plano de rollback se a extração falhar?
- [ ] O prazo é realista (extração de um serviço leva semanas a meses, não dias)?

### Checklist 5: Sinais de Reconsolidação

- [ ] Serviços que mudam juntos em > 50% dos casos foram identificados?
- [ ] Serviços que compartilham banco de dados foram identificados?
- [ ] Serviços do mesmo time sem requisitos de escala diferentes foram identificados?
- [ ] O custo operacional de microservices está excedendo o benefício?
- [ ] A produtividade do desenvolvedor caiu após a adoção de microservices?
- [ ] O time está gastando mais de 30% do tempo em infra/tooling?
- [ ] A reconsolidação é planejada como "modular monolith" (com enforcement), não "big ball of mud"?

---

## Perguntas que esta Skill Faz

Perguntas provocativas para desafiar decisões arquiteturais. Cada pergunta deve ser feita no
momento apropriado — antes da decisão, durante a operação, ou quando há sinais de problema.

### Antes de Adotar Microservices

1. **"Quantas pessoas estão no time? Se são menos de 30, por que microservices?"** —
   Vernon e DHH concordam: times pequenos não têm massa crítica para justificar o overhead
   operacional de microservices.

2. **"Se o Shopify roda com 300+ desenvolvedores em um monolito modular Rails, por que vocês
   com 10 desenvolvedores precisam de microservices?"** — Não é retórica. É um data point real.
   Se a empresa com uma das maiores aplicações Rails do mundo não precisa, por que vocês
   precisariam?

3. **"Qual é a dor concreta que microservices resolvem? Está documentada? Tem métricas?"** —
   Se a resposta é "queremos escalar" mas o sistema recebe 100 rps, a dor é imaginária. Se a
   resposta é "times bloqueados esperando deploy" mas há 2 times, o custo da solução excede o
   problema.

4. **"Se eu remover o service mesh, o API gateway, o distributed tracing, o Kubernetes e o
   message broker, quanto do orçamento de infra sobra para features?"** — Forçar a conta do
   custo real de microservices. Muitas vezes, mais de 40% do esforço de engenharia vai para
   manter a arquitetura funcionando.

5. **"Os Bounded Contexts estão estáveis? Ou vocês vão decompor boundaries que vão mudar no
   próximo quarter?"** — Newman é enfático: decompor boundaries instáveis é garantia de
   refatoração distribuída futura — ordens de grandeza mais cara.

6. **"Vocês já tentaram enforçar boundaries dentro do monolito? Se não conseguem nem com
   Packwerk/ArchUnit, o que faz acreditar que vão manter boundaries entre serviços?"** —
   Se o time não tem disciplina para boundaries internos, boundaries externos (APIs HTTP)
   não vão ajudar — vão apenas adicionar latência.

7. **"Quem vai operar esses microservices às 3h da manhã quando o PagerDuty tocar?"** —
   Microservices multiplicam pontos de falha. Se o on-call rotation já é doloroso, vai piorar.

### Durante a Operação

8. **"Quantos serviços são deployados independentemente por semana? Se a resposta é 'todos
   juntos na sexta', vocês têm um distributed monolith."** — Independent deployability é o
   teste que define microservices. Se não existe na prática, os serviços são um monolito com
   latência de rede.

9. **"Quantos minutos leva para debugar um bug end-to-end? Se precisa de 5 dashboards e 3
   ferramentas de tracing, o custo operacional está justificado?"** — Debugging distribuído
   é inerentemente mais difícil. Se isso está consumindo horas do time, o custo é real.

10. **"Esse serviço tem mais YAML de configuração (Kubernetes, Helm, CI/CD) do que linhas de
    código de negócio?"** — Se sim, é um nano-service. O serviço existe para justificar sua
    própria infraestrutura, não para resolver um problema de negócio.

11. **"Se um serviço cai, quantos outros são afetados? Se a resposta é 'quase todos', a
    independência é ilusória."** — Fault isolation é um dos benefícios prometidos de
    microservices. Se não existe, o benefício não se materializou.

12. **"Qual é a latência total de um request end-to-end? Quanto disso é overhead de rede
    entre serviços?"** — Se 60% da latência é rede entre serviços, a decomposição está
    custando performance sem dar nada em troca.

13. **"Quantas features do último quarter exigiram mudanças em 3+ serviços?"** — Se é mais de
    30%, os boundaries estão errados. As features deveriam caber em 1-2 serviços.

### Sobre Boundaries e Modularidade

14. **"Qual o enforcement real de boundaries no monolito? Packwerk? ArchUnit? Ou convenção
    e boa vontade?"** — Boundaries sem enforcement degradam exponencialmente. Após 3 meses
    sem enforcement, o "módulo" é uma pasta com nome bonito.

15. **"Se dois módulos mudam juntos em 80% das PRs, eles são um módulo. Por que a separação
    existe?"** — Change coupling é o indicador mais honesto de acoplamento real — mais do que
    qualquer diagrama de arquitetura.

16. **"Existe um diagrama atualizado do grafo de dependências entre módulos/serviços? Ele
    mostra ciclos?"** — Se o diagrama não existe ou está desatualizado, ninguém sabe o estado
    real da arquitetura.

### Sobre Reconsolidação

17. **"Microservices estão há 2 anos e a produtividade caiu. Já consideraram voltar?"** —
    "Voltar" não é fracasso. É maturidade. Amazon Prime Video voltou. Segment voltou. Istio
    voltou. Se o modelo não serve, mude o modelo.

18. **"Se o time pudesse começar do zero hoje, com o conhecimento atual, escolheria
    microservices de novo?"** — A pergunta mais honesta. Se a resposta é "não", está na hora
    de planejar a reconsolidação.

19. **"Quanto do orçamento de engenharia vai para manter os microservices funcionando versus
    entregar valor para o cliente?"** — Se a resposta é "> 40%", a arquitetura está servindo
    a si mesma. Purposeful Architecture serve ao negócio.

20. **"O on-call está sustentável? Quantos alertas por semana? O time está burned out?"** —
    Microservices multiplicam alertas. Se a operação está insustentável, a arquitetura precisa
    mudar — ou a equipe vai mudar (para outro emprego).

---

## Vocabulário

| Termo | Definição | Contexto |
|---|---|---|
| **Purposeful Architecture** | Abordagem de Vernon & Jaskula onde toda decisão arquitetural deve ser dirigida por propósito de negócio concreto, não por tendência tecnológica. | Conceito central do livro *Strategic Monoliths and Microservices*. |
| **The Majestic Monolith** | Termo cunhado por DHH para defender que um monolito bem estruturado é uma arquitetura legítima e frequentemente superior para times pequenos e médios. | Manifesto publicado por DHH no blog do Basecamp/Signal v. Noise. |
| **Modular Monolith** | Monolito com boundaries explícitos entre módulos, enforced por tooling. Cada módulo tem API pública, internals privados e schema ownership. Um deployable, muitos módulos. | Arquitetura defendida por Vernon como default para a maioria das organizações. |
| **Distributed Monolith** | Anti-pattern: serviços que precisam ser deployados juntos, compartilham banco ou não podem funcionar independentemente. Tem o custo de microservices sem os benefícios. | Descrito por Newman como o principal risco de decomposição prematura. |
| **Boundary Enforcement** | Uso de tooling para garantir que boundaries entre módulos/serviços sejam respeitados. Sem enforcement, boundaries degradam rapidamente. | Packwerk, ArchUnit, import-linter, Nx, Spring Modulith. |
| **Packwerk** | Ferramenta criada pelo Shopify para enforçar boundaries entre packages em aplicações Rails. Falha o CI se uma violação de boundary é detectada. | Usado pelo Shopify em produção para manter seu modular monolith. |
| **ArchUnit** | Library Java para testes de arquitetura. Permite definir regras como "classes no package X não podem depender do package Y". Roda como teste unitário. | Padrão de mercado para enforcement de boundaries em projetos Java/Kotlin. |
| **Spring Modulith** | Framework do ecossistema Spring para construir modular monoliths em Java. Oferece verificação de boundaries, eventos entre módulos e documentação automática. | Alternativa oficial do Spring para ArchUnit no contexto de modular monoliths. |
| **Service-based Architecture** | Arquitetura com poucos serviços coarse-grained (4-12), cada um cobrindo um domínio amplo. Meio-termo entre monolito e microservices. | Frequentemente o resultado de reconsolidação de microservices. |
| **Architecture Quantum** | Unidade mínima de arquitetura que pode ser deployada independentemente. Em um monolito, o quantum é o sistema inteiro. Em microservices, cada serviço é um quantum. | Conceito de Neal Ford, usado por Vernon para discutir granularidade. |
| **Strangler Fig Pattern** | Padrão de migração incremental onde funcionalidade é gradualmente movida do sistema legado para o novo, até o legado ser desligado. | Descrito por Newman como o único padrão seguro de migração para microservices. |
| **Bubble Context** | Padrão DDD onde um novo Bounded Context é criado como uma "bolha" isolada dentro ou ao lado do legado, com uma camada de tradução (Anti-Corruption Layer). | Usado em combinação com Strangler Fig para migração incremental. |
| **Team Topologies** | Framework de Skelton & Pais para design organizacional. Define 4 tipos de times: stream-aligned, platform, enabling e complicated subsystem. | Vernon usa Team Topologies para conectar estrutura de times com decisão arquitetural. |
| **Stream-aligned Team** | Time organizado em torno de uma stream de valor de negócio. É dono de um módulo ou serviço end-to-end, do código ao deploy ao monitoramento. | Tipo fundamental de time em Team Topologies. |
| **Cognitive Load** | Quantidade de esforço mental necessário para um time operar seus componentes. Microservices com muito overhead operacional aumentam cognitive load. | Conceito central de Team Topologies, usado por Vernon para argumentar contra decomposição excessiva. |
| **Independent Deployability** | Capacidade de um serviço ser deployado em produção sem requerer deploy simultâneo de outros serviços. O teste definitivo de microservices segundo Newman. | Se não existe na prática, os serviços são distributed monolith. |
| **Lock-step Deploy** | Anti-pattern onde múltiplos serviços precisam ser deployados juntos, na mesma ordem e no mesmo momento. Indica distributed monolith. | Sinal principal de que independent deployability não existe. |
| **Conway's Law** | "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations." — Melvin Conway. | Usada por Vernon para argumentar que arquitetura deve refletir estrutura de times. |
| **Inverse Conway Maneuver** | Reorganizar os times para que a estrutura de comunicação produza a arquitetura desejada. Em vez de adaptar a arquitetura aos times, adapta os times à arquitetura. | Conceito de Skelton & Pais, aplicado por Vernon no contexto de microservices. |
| **Entity Service** | Anti-pattern: um serviço por entidade de banco (CustomerService, OrderService). Resulta em CRUD distribuído sem encapsulamento de lógica de negócio. | Descrito por Vernon como resultado de modelar serviços por dados em vez de por business capability. |
| **Nano-service** | Anti-pattern: serviço tão pequeno que tem mais código de infraestrutura do que código de negócio. O overhead de existir supera o valor que entrega. | Extremo do espectro de granularidade. |
| **Change Coupling** | Métrica que mede com que frequência dois componentes (módulos, serviços, arquivos) mudam juntos. Alta change coupling indica acoplamento real, independente da arquitetura formal. | Indicador mais honesto de acoplamento do que diagramas de arquitetura. |
| **Strangler Fig Abandonment** | Anti-pattern: migração para microservices iniciada e abandonada no meio, resultando em dois sistemas (monolito parcial + serviços) com o custo de ambos. | Um dos anti-patterns mais comuns e caros. |
| **Schema Ownership** | Princípio de que cada módulo/serviço é o dono exclusivo das suas tabelas no banco de dados. Nenhum outro módulo/serviço acessa essas tabelas diretamente. | Fundamental para modular monolith e para microservices. |
| **Saga Pattern** | Padrão para gerenciar transações distribuídas em microservices. Uma sequência de transações locais onde cada passo publica evento que dispara o próximo. Compensações para rollback. | Substituto de ACID transactions em sistemas distribuídos — inerentemente mais complexo. |
| **Circuit Breaker** | Padrão de resiliência que impede chamadas a um serviço que está falhando. Após N falhas, o circuit "abre" e retorna erro imediatamente sem tentar a chamada. | Essencial em microservices para evitar cascading failures. |
| **Bulkhead** | Padrão de resiliência que isola recursos (threads, connections) para evitar que a falha de um serviço consuma todos os recursos e afete outros. | Inspirado em compartimentos de navios. |
| **Event Sourcing** | Padrão onde o estado é derivado de uma sequência de eventos, não de uma tabela de estado atual. Poderoso mas complexo — frequentemente over-engineered. | Muitas vezes adotado junto com microservices sem necessidade real. |
| **CQRS (Command Query Responsibility Segregation)** | Separação entre modelo de escrita (command) e modelo de leitura (query). Justificado quando read e write têm requisitos muito diferentes. | Frequentemente combinado com Event Sourcing e microservices — mas cada um é independente. |
| **Anti-Corruption Layer (ACL)** | Camada de tradução entre dois Bounded Contexts que protege um contexto de ser contaminado pelo modelo do outro. | Essencial em migrações incrementais (Strangler Fig). |
| **Backward Compatibility** | Capacidade de uma nova versão de API/evento ser consumida por clientes que usam a versão anterior. Essencial em microservices para independent deployability. | Sem backward compatibility, independent deployability é impossível. |
| **Contract Testing** | Testes que verificam o contrato (API, evento) entre produtor e consumidor. Ferramentas: Pact, Spring Cloud Contract. | Substituem integration tests cross-service, que são lentos e frágeis. |

---

## Referências de Aplicação Rápida

### Quando o Arquiteto Agente Deve Ativar Esta Skill

1. Qualquer discussão sobre **adotar microservices** — ativar como contrapeso crítico.
2. Qualquer reclamação sobre **complexidade operacional** de microservices — investigar sinais
   de distributed monolith ou premature decomposition.
3. Discussão sobre **migração de monolito** — enforçar que o Strangler Fig Pattern seja usado
   e que a dor concreta esteja documentada.
4. Revisão de arquitetura onde o sistema tem **mais serviços do que times** — questionar.
5. Qualquer menção a **modular monolith sem mencionar enforcement** — alertar que boundaries
   sem tooling são boundaries imaginários.
6. Projeto greenfield onde microservices são o **default sem discussão** — forçar a discussão.
7. Time pequeno (< 30 devs) com plano de adotar microservices — desafiar fortemente.
8. Sinais de **reconsolidação** — orientar o processo, não como fracasso, mas como evolução.

### Resumo da Posição

A posição consolidada de Vernon, Newman e DHH pode ser resumida em uma frase:

**Comece com o monolito modular (com boundaries enforced), evolua para microservices apenas
quando a dor de coordenação for concreta, medida e superior ao custo de distribuição — e esteja
preparado para voltar se a conta não fechar.**

Microservices são uma ferramenta poderosa no lugar certo. Mas o lugar certo é mais raro do que
a indústria faz parecer. A maioria dos sistemas seria melhor servida por um modular monolith
bem projetado com boundaries enforced por tooling. Isso não é conservadorismo — é Purposeful
Architecture.
