# Skill: Domain-Driven Design — Eric Evans

## Perspectiva do Autor

Eric Evans enxerga o desenvolvimento de software como um problema de **conhecimento**, nao de tecnologia.
O software mais valioso e aquele que resolve problemas complexos de dominio, e a unica forma de
resolve-los bem e por meio de uma colaboracao profunda entre desenvolvedores e especialistas de dominio.

Evans defende que:

1. **O modelo e o coracao do software.** Nao e o banco de dados, nao e o framework, nao e a UI.
   O modelo de dominio e a arma estrategica que diferencia software bom de software mediano.

2. **A linguagem unifica o time.** Se desenvolvedores e especialistas de dominio usam vocabularios
   diferentes, o software sera uma traducao defeituosa. A Ubiquitous Language elimina essa friccao.

3. **O design deve ser iterativo e guiado por insight.** Modelos nao nascem prontos. Eles emergem
   de ciclos de Knowledge Crunching — conversas, experimentacao, refatoracao, descoberta.

4. **Desenvolvedores devem ser modeladores.** Evans rejeita a separacao entre "arquitetos que pensam"
   e "desenvolvedores que codificam". Quem escreve o codigo deve participar da modelagem. Hands-On
   Modelers nao delegam o modelo para documentos — eles o expressam no codigo.

5. **Nem todo software merece DDD.** Evans e pragmatico: DDD e para dominios complexos onde a
   complexidade do negocio justifica o investimento em modelagem. Para CRUD simples, e overkill.

6. **Design estrategico e tao importante quanto design tatico.** Bounded Contexts, Context Maps e
   distilacao do Core Domain sao decisoes que afetam a organizacao inteira, nao so um modulo.

Evans ve o software como um espelho do conhecimento do time sobre o dominio. Se o espelho e turvo
(modelo fraco), o software sera fragil. Se o espelho e nitido (modelo rico), o software sera poderoso.

---

## Principios Fundamentais

### 1. Knowledge Crunching e Ubiquitous Language

O processo de modelagem comeca com Knowledge Crunching: sessoes intensas entre desenvolvedores e
domain experts para extrair, organizar e testar conhecimento sobre o dominio.

**Knowledge Crunching:**
- Nao e uma fase — e continuo ao longo de todo o projeto
- Envolve conversas, diagramas, prototipagem rapida e experimentacao com modelos
- O objetivo e encontrar abstraccoes que simplifiquem a complexidade do dominio
- Os melhores insights surgem quando o time descobre conceitos implicitos e os torna explicitos
- Desenvolvedores devem questionar terminologia, perguntar "por que?", e desafiar premissas

**Ubiquitous Language:**
- E um vocabulario compartilhado entre desenvolvedores e domain experts
- Deve aparecer no codigo (nomes de classes, metodos, variaveis), na documentacao, e nas conversas
- Mudancas na linguagem implicam mudancas no modelo e vice-versa
- Cada Bounded Context tem sua propria Ubiquitous Language
- Se um termo e ambiguo, e sinal de que o modelo precisa ser refinado
- A linguagem nao e imposta — ela emerge das sessoes de Knowledge Crunching

**Sinais de que Knowledge Crunching esta funcionando:**
- O time descobre contradicoes no entendimento do dominio
- Novos conceitos emergem que simplificam regras complexas
- O domain expert diz "sim, e exatamente assim que funciona"
- O codigo comeca a "contar a historia" do dominio

**Sinais de que Knowledge Crunching esta falhando:**
- Desenvolvedores nao conseguem explicar o dominio sem mencionar tecnologia
- Domain experts nao reconhecem a terminologia usada no codigo
- O modelo e uma copia 1:1 do banco de dados ou da UI
- Ninguem questiona as decisoes de modelagem

### 2. Model-Driven Design

O modelo de dominio nao e um diagrama UML em um documento separado. O modelo **e** o codigo.
Model-Driven Design significa que:

- O design do codigo reflete diretamente os conceitos do dominio
- Refatorar o modelo implica refatorar o codigo (e vice-versa)
- Diagramas sao ferramentas auxiliares, nao o modelo em si
- O modelo deve ser pratico — se nao guia decisoes de implementacao, e inutil
- Cada artefato de codigo (classe, metodo, modulo) deve ter correspondencia com um conceito do dominio

**Hands-On Modelers:**
- Desenvolvedores que escrevem codigo devem participar da modelagem
- Arquitetos que nao codificam perdem contato com a realidade do modelo
- O feedback loop entre modelo e implementacao deve ser curto
- Separar "pensadores" de "codificadores" cria modelos desconectados da realidade

### 3. Layered Architecture

Evans propoe uma arquitetura em camadas para isolar o dominio da infraestrutura:

**User Interface (Presentation Layer):**
- Responsavel por exibir informacoes e interpretar comandos do usuario
- Nao contem logica de dominio
- Pode ser uma API REST, CLI, GUI, etc.

**Application Layer:**
- Coordena tarefas e delega trabalho ao dominio
- Nao contem regras de negocio
- Gerencia transacoes, seguranca, e orquestracao de use cases
- Deve ser fina — se esta gorda, logica de dominio vazou para ca

**Domain Layer:**
- O coracao do software
- Contem Entities, Value Objects, Domain Services, Aggregates, Repositories (interfaces), Factories
- Expressa regras de negocio e conceitos do dominio
- Nao depende de nenhuma outra camada (principio de inversao de dependencia)
- Deve ser testavel isoladamente, sem banco de dados, sem framework

**Infrastructure Layer:**
- Implementacoes tecnicas: persistencia, messaging, email, integracao externa
- Implementa interfaces definidas na camada de dominio (ex: Repository implementations)
- Contem adapters, ORM mappings, clients HTTP, etc.

**Regra fundamental:** dependencias apontam para dentro. A camada de dominio nao conhece
a infraestrutura. A infraestrutura conhece e implementa contratos do dominio.

### 4. Building Blocks — Elementos Taticos

#### Entity

- Objeto definido pela sua **identidade**, nao pelos seus atributos
- Dois objetos com os mesmos atributos mas IDs diferentes sao entidades diferentes
- Deve ter uma identidade unica e estavel ao longo do tempo
- Exemplos: Cliente, Pedido, Conta Bancaria, Produto
- A identidade pode ser natural (CPF) ou artificial (UUID)
- Entities tem ciclo de vida: sao criadas, modificadas e eventualmente descontinuadas
- Igualdade e baseada em identidade, nao em atributos

#### Value Object

- Objeto definido pelos seus **atributos**, nao por identidade
- Dois Value Objects com os mesmos atributos sao iguais
- Deve ser **imutavel** — uma vez criado, nao muda
- Exemplos: Endereco, Dinheiro, CPF, Periodo, Coordenada GPS
- Pode e deve conter logica de dominio relevante (ex: `Dinheiro.somar(outro)`)
- Facilita compartilhamento seguro (imutabilidade garante ausencia de side effects)
- Deve ser substituido inteiramente, nunca modificado parcialmente
- Pode ser implementado como record, data class, frozen dataclass, etc.

#### Domain Service

- Operacao do dominio que **nao pertence naturalmente** a nenhuma Entity ou Value Object
- E stateless — nao mantem estado proprio
- Recebe entidades/VOs como parametros e executa logica de dominio
- Exemplos: CalculadoraDeFreteService, TransferenciaEntreContasService
- Nao confundir com Application Service (que orquestra, nao executa logica de dominio)
- O nome do servico deve refletir um conceito do dominio, nao uma acao tecnica
- Se o servico nao faz sentido para o domain expert, provavelmente nao e um Domain Service

#### Module (Package)

- Agrupamento logico de conceitos relacionados do dominio
- Deve refletir a Ubiquitous Language, nao preocupacoes tecnicas
- Nomes de pacotes como `services/`, `models/`, `utils/` sao anti-patterns
- Prefira nomes como `pedidos/`, `faturamento/`, `logistica/`
- Modulos devem ter alta coesao interna e baixo acoplamento externo
- A estrutura de modulos comunica a organizacao do dominio

#### Aggregate

- Cluster de Entities e Value Objects tratados como uma unidade para mudancas de dados
- Tem uma **Aggregate Root** — a unica Entity por onde se acessa o cluster
- Regras de consistencia dentro do Aggregate sao garantidas em uma unica transacao
- Objetos externos so podem referenciar o Aggregate Root, nunca entidades internas
- O Aggregate Root e responsavel por manter invariantes do cluster inteiro
- Deletar o Aggregate Root implica deletar todo o cluster
- O limite do Aggregate define o limite da transacao

#### Factory

- Encapsula a logica complexa de criacao de objetos e Aggregates
- Garante que o objeto e criado em um estado valido
- Pode ser um metodo factory, um Factory Method, ou uma classe separada
- Util quando a criacao envolve regras de negocio ou montagem de grafos complexos
- A Factory conhece as invariantes e as garante na criacao
- Para criacao simples, um construtor e suficiente — nao criar Factories desnecessarias

#### Repository

- Abstracao que simula uma colecao em memoria de Aggregates
- Fornece a ilusao de que todos os objetos estao disponiveis em memoria
- Interface definida na camada de dominio; implementacao na infraestrutura
- Um Repository por Aggregate Root — nunca para entidades internas
- Operacoes tipicas: add, remove, findById, findByCriteria
- Nao deve expor detalhes de persistencia (SQL, conexoes, sessions)
- O Repository retorna Aggregates completamente constituidos

### 5. Design Estrategico

#### Bounded Context

- Limite explicito dentro do qual um modelo de dominio se aplica
- Cada Bounded Context tem sua propria Ubiquitous Language
- O mesmo conceito real pode ter representacoes diferentes em Bounded Contexts diferentes
  (ex: "Cliente" no contexto de Vendas vs "Cliente" no contexto de Suporte)
- Alinhar Bounded Contexts com limites de times e uma boa heuristica
- Um Bounded Context nao e necessariamente um microsservico — pode ser um modulo

#### Context Map

- Mapa que documenta os relacionamentos entre Bounded Contexts
- Tipos de relacionamento:

**Shared Kernel:**
- Dois contextos compartilham um subconjunto do modelo
- Ambos os times concordam com o codigo compartilhado
- Mudancas requerem coordenacao entre times
- Deve ser pequeno — quanto maior, maior a friccao

**Customer/Supplier:**
- Um contexto (Supplier/Upstream) fornece servicos para outro (Customer/Downstream)
- O Supplier deve considerar as necessidades do Customer
- O Customer tem influencia sobre o roadmap do Supplier
- Existe negociacao entre os times

**Conformist:**
- O Downstream aceita o modelo do Upstream sem influencia
- Nao ha negociacao — o Downstream se conforma
- Usado quando o Upstream nao tem incentivo para atender o Downstream
- Ex: integracao com API de terceiros que voce nao controla

**Anti-Corruption Layer (ACL):**
- Camada de traducao que isola o modelo downstream do modelo upstream
- Implementada com Adapter + Translator + Facade
- Essencial quando o modelo upstream e incompativel ou de baixa qualidade
- Protege o dominio downstream de ser "contaminado" por conceitos externos
- O ACL traduz entre as duas linguagens

**Separate Ways:**
- Os contextos decidem nao se integrar
- Cada um resolve o problema de forma independente
- Duplicacao de funcionalidade e aceita como trade-off
- Util quando o custo de integracao supera o beneficio

**Open Host Service:**
- O Upstream define um protocolo aberto e bem documentado
- Qualquer Downstream pode se integrar usando esse protocolo
- Desacopla o Upstream de Downstreams especificos
- Geralmente combinado com Published Language

**Published Language:**
- Formato de dados bem documentado e versionado para integracao
- Ex: XML Schema, JSON Schema, Protobuf, Avro
- Permite que multiplos contextos se comuniquem sem acoplamento direto
- Deve ser estavel e evoluir com versionamento explicito

### 6. Distilacao do Dominio

#### Core Domain

- A parte do dominio que diferencia o negocio competitivamente
- Onde o investimento em modelagem deve ser mais intenso
- Deve ter os melhores desenvolvedores trabalhando nele
- Nao e necessariamente o maior modulo — mas e o mais valioso

#### Generic Subdomain

- Funcionalidade necessaria mas nao diferenciadora
- Pode ser comprada, terceirizada ou implementada com solucoes simples
- Exemplos: autenticacao, notificacoes, geolookup
- Nao merece o mesmo nivel de investimento em modelagem que o Core Domain

#### Supporting Subdomain

- Funcionalidade que suporta o Core Domain mas nao e generica o suficiente para comprar
- Importante, mas nao e a vantagem competitiva
- Pode ter um modelo mais simples que o Core Domain

#### Domain Vision Statement

- Documento curto (1 pagina) que descreve o Core Domain e seu valor
- Usado para alinhar o time sobre o que e mais importante
- Guia decisoes de investimento e priorizacao

#### Highlighted Core

- Documento ou diagrama que destaca os elementos mais importantes do modelo
- Ajuda novos membros do time a entender rapidamente o que importa
- Pode ser um Distillation Document ou marcacoes visuais no codigo

#### Segregated Core

- Refatoracao que separa fisicamente o Core Domain de subdominios genericos/suporte
- O Core Domain fica em modulos/pacotes dedicados, sem depender de subdominios
- Torna explicito o que e estrategicamente importante

### 7. Tornando Conceitos Implicitos Explicitos

Um dos insights mais poderosos de Evans: muito da complexidade do dominio esta escondida em
conceitos implicitos que nunca foram nomeados ou modelados.

**Specification Pattern:**
- Encapsula uma regra de negocio em um objeto que pode ser testado, combinado e reutilizado
- Metodo `isSatisfiedBy(candidate): boolean`
- Pode ser combinada com AND, OR, NOT para criar regras compostas
- Util para validacoes, filtragem, selecao de politicas
- Exemplos: `PedidoElegivelParaDesconto`, `ClienteAdimplente`, `ProdutoDisponivel`

**Outros conceitos frequentemente implicitos:**
- Restricoes (Constraints) que existem apenas como if/else espalhados
- Processos (Processes) que sao sequencias de regras nao nomeadas
- Politicas (Policies/Strategies) que variam mas nao foram abstraidas
- Regras de calculo que estao embutidas em metodos longos

**Como encontrar conceitos implicitos:**
- Ouvir a linguagem do domain expert — termos que eles usam mas que nao existem no codigo
- Procurar por codigo "desconfortavel" — metodos longos, muitos ifs, parametros demais
- Questionar por que certas regras existem — a resposta revela conceitos ocultos
- Experimentar com modelos alternativos — as vezes renomear muda tudo

---

## Heuristicas Verificaveis

### Ubiquitous Language e Nomeacao

1. **Toda classe de dominio usa termos da Ubiquitous Language.**
   Como verificar: revise nomes de classes no Domain Layer. Se algum nome nao faz sentido para
   o domain expert, o nome esta errado. Busque por nomes tecnicos (Manager, Handler, Processor,
   Helper, Utils) no dominio — eles indicam conceitos nao nomeados.

2. **Metodos de Entities e Value Objects usam verbos do dominio.**
   Como verificar: leia a assinatura dos metodos publicos. `pedido.confirmar()` e melhor que
   `pedido.setStatus(CONFIRMADO)`. Metodos devem ler como sentencas do dominio.

3. **Nao existem sinonimos no mesmo Bounded Context.**
   Como verificar: procure por classes ou metodos que usam termos diferentes para o mesmo conceito
   (ex: `Client` e `Customer` no mesmo contexto). Cada conceito deve ter exatamente um nome.

4. **Nomes de pacotes/modulos refletem conceitos do dominio, nao camadas tecnicas.**
   Como verificar: a estrutura de diretorios deve ter nomes como `pedidos/`, `faturamento/`,
   `estoque/` — nao `models/`, `services/`, `utils/`, `helpers/`.

### Entities e Value Objects

5. **Toda Entity tem identidade explicita e imutavel.**
   Como verificar: verifique se cada Entity tem um campo de ID que nao pode ser alterado apos
   criacao. O metodo `equals()` deve comparar por identidade, nao por atributos.

6. **Todo Value Object e imutavel.**
   Como verificar: verifique que Value Objects nao tem setters. Campos devem ser `final`/`readonly`.
   Operacoes retornam novos objetos em vez de modificar o existente.
   Ex: `dinheiro.somar(outro)` retorna um novo `Dinheiro`, nao modifica `this`.

7. **Value Objects contem logica de dominio relevante.**
   Como verificar: se um Value Object e apenas um DTO com getters, ele nao esta agregando valor.
   Procure por logica espalhada em Services ou Entities que deveria estar no VO.
   Ex: validacao de CPF deve estar na classe `CPF`, nao em um `CpfValidator` externo.

8. **Value Objects sao usados em vez de tipos primitivos para conceitos de dominio.**
   Como verificar: procure por `String email`, `double preco`, `String cpf` em Entities.
   Estes devem ser `Email email`, `Dinheiro preco`, `CPF cpf`. Primitive Obsession e um code smell.

### Aggregates

9. **Cada Aggregate tem exatamente uma Aggregate Root.**
   Como verificar: identifique os pontos de entrada para cada cluster de objetos. Se objetos
   internos sao acessados diretamente de fora do Aggregate, o limite esta quebrado.

10. **Objetos externos nao referenciam entidades internas do Aggregate.**
    Como verificar: busque por referencias diretas a entidades que nao sao Aggregate Roots.
    Se `PedidoItem` e interno ao Aggregate `Pedido`, ninguem fora deve ter referencia a `PedidoItem`.

11. **O Aggregate Root valida invariantes antes de persistir.**
    Como verificar: todas as operacoes que modificam o Aggregate devem passar pela Root.
    Verifique que nao ha forma de modificar entidades internas sem passar pelo metodo da Root.

12. **Transacoes respeitam limites de Aggregate.**
    Como verificar: cada transacao deve modificar no maximo um Aggregate. Se uma transacao
    modifica multiplos Aggregates, os limites estao errados ou falta eventual consistency.

### Repositories e Factories

13. **Existe exatamente um Repository por Aggregate Root.**
    Como verificar: conte Repositories e Aggregate Roots. Se os numeros nao batem, algo esta errado.
    Nao deve existir Repository para Value Objects ou entidades internas.

14. **A interface do Repository esta na camada de dominio; a implementacao na infraestrutura.**
    Como verificar: verifique que a interface do Repository nao importa nenhum pacote de
    infraestrutura (JPA, Hibernate, Mongo, SQL). A implementacao deve estar em outro pacote.

15. **Repositories retornam Aggregates completos.**
    Como verificar: verifique que o Repository nunca retorna entidades internas parciais ou
    projecoes. O retorno deve ser o Aggregate Root com todo o grafo interno montado.

16. **Factories garantem que objetos sao criados em estado valido.**
    Como verificar: apos chamar a Factory, o objeto resultante deve satisfazer todas as invariantes.
    Nao deve ser possivel criar um Aggregate em estado inconsistente.

### Layered Architecture

17. **A camada de dominio nao depende de infraestrutura.**
    Como verificar: analise os imports da camada de dominio. Se ha imports de frameworks ORM,
    HTTP clients, messaging libraries, a dependencia esta invertida.

18. **Application Services nao contem logica de dominio.**
    Como verificar: procure por if/else com semantica de negocio em Application Services.
    Se encontrar `if (pedido.getTotal() > 1000)`, essa logica pertence ao dominio.

19. **A camada de dominio e testavel sem infraestrutura.**
    Como verificar: testes unitarios do dominio devem rodar sem banco de dados, sem servidor HTTP,
    sem broker de mensagens. Se precisam de @SpringBootTest ou container Docker, algo esta errado.

20. **Controllers/Endpoints nao acessam Repositories diretamente.**
    Como verificar: controllers devem chamar Application Services, que por sua vez usam
    Repositories. Acesso direto do controller ao Repository pula a camada de aplicacao.

### Design Estrategico

21. **Cada Bounded Context tem limites claros e documentados.**
    Como verificar: deve ser possivel identificar quais modulos/pacotes pertencem a qual
    Bounded Context. Se a resposta e "tudo e um contexto so", provavelmente ha contextos implicitos.

22. **O Context Map esta atualizado e reflete integrações reais.**
    Como verificar: compare o Context Map com as integracoes reais (APIs chamadas, filas consumidas,
    schemas compartilhados). Se ha divergencia, o mapa esta obsoleto.

23. **Anti-Corruption Layers existem para integracoes com modelos externos.**
    Como verificar: quando o sistema consome APIs externas, deve existir uma camada de traducao
    que converte DTOs externos em objetos do dominio interno. O dominio nunca usa tipos externos.

24. **O Core Domain esta identificado e recebe o maior investimento.**
    Como verificar: pergunte ao time "qual e o Core Domain?". Se nao ha consenso, a distilacao
    nao foi feita. Verifique se o codigo do Core Domain tem melhor qualidade que subdominios genericos.

### Specification e Conceitos Implicitos

25. **Regras de negocio complexas sao encapsuladas em Specifications.**
    Como verificar: procure por regras de filtragem ou validacao espalhadas em multiplos metodos.
    Se a mesma regra aparece em mais de um lugar, ela deve ser uma Specification.

26. **Conceitos do dominio que existem na conversa existem no codigo.**
    Como verificar: liste termos que o domain expert usa. Cada termo deve ter uma classe,
    metodo ou modulo correspondente. Termos orfaos indicam conceitos implicitos nao modelados.

27. **Nao existem metodos com mais de 3 parametros booleanos.**
    Como verificar: metodos com muitos flags geralmente escondem variantes de comportamento
    que deveriam ser Policies, Strategies ou Specifications nomeadas.

28. **Enums que controlam fluxo de negocio sao candidatos a polimorfismo ou Strategy.**
    Como verificar: procure por `switch/case` ou `if/else` sobre enums que determinam
    comportamento de negocio. Considere substituir por Strategy ou polimorfismo.

---

## Anti-patterns com Sinais no Codigo

| Anti-pattern | Sinal no Codigo | Severidade | Acao |
|---|---|---|---|
| **Anemic Domain Model** | Entities com apenas getters/setters, logica toda em Services | CRITICA | Mover logica de negocio para Entities e Value Objects |
| **Smart UI** | Regras de negocio no controller ou na camada de apresentacao | CRITICA | Extrair logica para Domain Layer |
| **Big Ball of Mud** | Sem Bounded Contexts, tudo no mesmo modelo monolitico | CRITICA | Identificar Bounded Contexts e separar modelos |
| **Primitive Obsession** | `String cpf`, `double preco`, `int quantidade` sem Value Objects | ALTA | Criar Value Objects para cada conceito de dominio |
| **Repository para entidade interna** | `PedidoItemRepository`, `EnderecoRepository` (quando sao internos) | ALTA | Remover e acessar via Aggregate Root |
| **God Aggregate** | Aggregate com 10+ entidades, transacoes lentas | ALTA | Dividir em Aggregates menores com eventual consistency |
| **Leaking Domain Logic** | if/else de negocio em Application Service ou Controller | ALTA | Mover logica para Domain Service, Entity ou Specification |
| **Database-Driven Design** | Classes sao reflexo 1:1 de tabelas, modelo guiado pelo schema | ALTA | Redesenhar modelo a partir do dominio, nao do banco |
| **Ubiquitous Language ausente** | Nomes tecnicos (Manager, Handler, Processor) em classes de dominio | MEDIA | Renomear usando termos do domain expert |
| **Shared Kernel descontrolado** | Biblioteca compartilhada que cresce sem limites | MEDIA | Reduzir ao minimo, considerar Published Language |
| **Acoplamento entre Aggregates** | Um Aggregate segura referencia de objeto para outro | MEDIA | Substituir por referencia por identidade (ID) |
| **Transacao cross-Aggregate** | Uma transacao modifica multiplos Aggregates | MEDIA | Usar Domain Events e eventual consistency |
| **Specification implicita** | Mesma condicao de negocio copiada em 3+ lugares | MEDIA | Extrair para Specification nomeada |
| **Context Map ausente** | Integracoes entre contextos sem documentacao de relacionamento | MEDIA | Criar e manter Context Map |
| **Conformist nao intencional** | Dominio interno usa tipos/DTOs de API externa diretamente | MEDIA | Implementar Anti-Corruption Layer |
| **Factory ausente** | Criacao de Aggregate complexo espalhada em multiplos pontos | BAIXA | Centralizar criacao em Factory |
| **Modulos tecnicos** | Pacotes chamados `models/`, `services/`, `repositories/` | BAIXA | Reorganizar por conceito de dominio |

---

## Checklist de Revisao

### Linguagem e Modelagem

- [ ] Os nomes de classes, metodos e variaveis usam termos da Ubiquitous Language?
- [ ] Um domain expert conseguiria ler o codigo e reconhecer os conceitos?
- [ ] Existem termos ambiguos ou sinonimos no mesmo Bounded Context?
- [ ] Conceitos mencionados pelo domain expert estao modelados explicitamente no codigo?
- [ ] O modelo foi refinado recentemente com base em feedback de Knowledge Crunching?

### Building Blocks

- [ ] Entities tem identidade explicita e igualdade baseada em ID?
- [ ] Value Objects sao imutaveis e comparam por atributos?
- [ ] Value Objects contem logica de dominio (nao sao meros DTOs)?
- [ ] Tipos primitivos foram substituidos por Value Objects onde apropriado?
- [ ] Domain Services sao stateless e recebem objetos do dominio como parametros?
- [ ] Domain Services tem nomes que fazem sentido para o domain expert?

### Aggregates

- [ ] Cada Aggregate tem uma unica Aggregate Root?
- [ ] Limites de Aggregate estao definidos com base em invariantes de negocio?
- [ ] Entidades internas nao sao acessadas diretamente de fora do Aggregate?
- [ ] Aggregates sao pequenos (preferencia por Root + Value Objects)?
- [ ] Transacoes modificam no maximo um Aggregate?
- [ ] Aggregates referenciam outros Aggregates por ID, nao por objeto?

### Repositories e Factories

- [ ] Ha exatamente um Repository por Aggregate Root?
- [ ] Interfaces de Repository estao no dominio, implementacoes na infraestrutura?
- [ ] Repositories retornam Aggregates completos?
- [ ] Factories garantem criacao em estado valido?
- [ ] Repositories nao expoe detalhes de persistencia?

### Arquitetura em Camadas

- [ ] O Domain Layer nao depende de frameworks ou infraestrutura?
- [ ] Application Services sao finos (orquestram, nao decidem)?
- [ ] Controllers delegam para Application Services, nao acessam Repositories?
- [ ] Testes do dominio rodam sem infraestrutura externa?
- [ ] A direcao de dependencias aponta para o dominio?

### Design Estrategico

- [ ] Bounded Contexts estao identificados e documentados?
- [ ] O Context Map reflete integracoes reais?
- [ ] Integracoes com sistemas externos usam ACL?
- [ ] O Core Domain esta identificado?
- [ ] O Core Domain tem a melhor qualidade de codigo?
- [ ] Generic Subdomains usam solucoes proporcionais (compradas/simples)?

### Conceitos Implicitos

- [ ] Regras de negocio repetidas foram extraidas para Specifications?
- [ ] Variacoes de comportamento usam Strategies/Policies nomeadas?
- [ ] Metodos longos com muitos ifs foram decompostos em conceitos nomeados?
- [ ] Processos de negocio com multiplas etapas estao modelados explicitamente?

---

## Perguntas que esta Skill Faz

1. **Qual e a Ubiquitous Language deste Bounded Context?** Ela esta documentada? O time a usa
   consistentemente nas conversas, no codigo e nos testes?

2. **Um domain expert reconheceria este codigo?** Se mostrarmos as classes e metodos do dominio
   para o especialista, ele entenderia os nomes e a estrutura?

3. **Onde esta o Core Domain?** O time sabe qual parte do sistema e a vantagem competitiva?
   O Core Domain recebe os melhores desenvolvedores e o maior investimento?

4. **Este conceito e uma Entity ou um Value Object?** A identidade importa, ou so os atributos?
   Se dois objetos tem os mesmos atributos, eles sao intercambiaveis?

5. **Por que este Aggregate e tao grande?** Todas as entidades internas realmente precisam de
   consistencia transacional com a Root? Poderiamos usar eventual consistency?

6. **Que conceito implicito esta escondido neste metodo longo?** O metodo tem 50 linhas com
   varios ifs — que conceitos de dominio estao implicitos aqui que deveriam ser classes?

7. **Por que existe logica de negocio neste Application Service?** Este if/else tem semantica
   de dominio — nao deveria estar em uma Entity, Domain Service ou Specification?

8. **Como este Bounded Context se relaciona com os outros?** O relacionamento e
   Customer/Supplier? Conformist? Tem ACL? O Context Map esta atualizado?

9. **Estamos usando DDD onde nao precisamos?** Este modulo e CRUD simples? DDD estaria
   adicionando complexidade sem beneficio?

10. **O modelo reflete o dominio ou o banco de dados?** As classes sao copias das tabelas?
    O modelo foi desenhado a partir de conversas com domain experts ou do schema do banco?

11. **Quem e o dono deste Bounded Context?** Ha um time claro responsavel? Ou multiplos
    times modificam o mesmo contexto sem coordenacao?

12. **Os Repositories estao escondendo a persistencia adequadamente?** O dominio sabe que
    usa banco relacional? Ha annotations JPA/ORM nas Entities?

13. **Podemos testar o dominio isoladamente?** Os testes unitarios do dominio dependem de
    banco, framework, ou containers? Se sim, a camada de dominio nao esta isolada.

14. **Este Domain Service deveria existir?** A logica poderia pertencer a uma Entity ou
    Value Object? Domain Services sao para operacoes que genuinamente nao pertencem a um objeto.

15. **Quando foi a ultima sessao de Knowledge Crunching?** Se ninguem lembra, o modelo esta
    estagnado. Modelos bons evoluem continuamente com novos insights.

16. **Existem Bounded Contexts implicitos neste monolito?** Termos ambiguos, modelos que tentam
    ser tudo para todos, e times que nao concordam sobre conceitos sao sinais de contextos ocultos.

17. **O que acontece se este Bounded Context precisar escalar independentemente?** Os limites
    atuais permitiriam extrair este contexto para um servico independente?

---

## Vocabulario

| Termo | Definicao |
|---|---|
| **Knowledge Crunching** | Processo iterativo de exploracao e refinamento do conhecimento do dominio entre desenvolvedores e domain experts |
| **Ubiquitous Language** | Vocabulario compartilhado entre desenvolvedores e domain experts, usado no codigo, na documentacao e nas conversas |
| **Model-Driven Design** | Abordagem onde o codigo reflete diretamente o modelo de dominio, e mudancas no modelo implicam mudancas no codigo |
| **Hands-On Modeler** | Desenvolvedor que participa ativamente da modelagem de dominio e implementa o modelo no codigo |
| **Entity** | Objeto de dominio definido por sua identidade unica, nao por seus atributos |
| **Value Object** | Objeto de dominio imutavel definido por seus atributos, sem identidade propria |
| **Domain Service** | Operacao stateless do dominio que nao pertence naturalmente a nenhuma Entity ou Value Object |
| **Application Service** | Servico que orquestra use cases, gerencia transacoes e seguranca, sem conter logica de dominio |
| **Module** | Agrupamento logico de conceitos relacionados do dominio, nomeado com termos da Ubiquitous Language |
| **Aggregate** | Cluster de Entities e Value Objects tratados como unidade de consistencia, com uma Aggregate Root |
| **Aggregate Root** | Entity que serve como ponto de entrada e guardiao de invariantes de um Aggregate |
| **Factory** | Objeto ou metodo responsavel por criar Aggregates e Entities em estado valido |
| **Repository** | Abstracao que simula colecao em memoria de Aggregates, com interface no dominio e implementacao na infraestrutura |
| **Bounded Context** | Limite explicito dentro do qual um modelo de dominio e sua Ubiquitous Language se aplicam |
| **Context Map** | Mapa que documenta relacionamentos e integracoes entre Bounded Contexts |
| **Shared Kernel** | Subconjunto de modelo compartilhado entre dois Bounded Contexts com acordo mutuo |
| **Customer/Supplier** | Relacionamento onde um contexto upstream fornece servicos para um downstream com negociacao |
| **Conformist** | Relacionamento onde o downstream aceita o modelo do upstream sem poder de influencia |
| **Anti-Corruption Layer (ACL)** | Camada de traducao que protege o modelo do dominio de modelos externos incompativeis |
| **Separate Ways** | Decisao de nao integrar dois contextos, aceitando duplicacao como trade-off |
| **Open Host Service** | Protocolo aberto e documentado oferecido por um contexto para integracao |
| **Published Language** | Formato de dados versionado e documentado para comunicacao entre contextos |
| **Core Domain** | Parte do dominio que diferencia o negocio competitivamente e merece o maior investimento |
| **Generic Subdomain** | Funcionalidade necessaria mas nao diferenciadora, candidata a compra ou terceirizacao |
| **Supporting Subdomain** | Funcionalidade que suporta o Core Domain mas nao e generica o suficiente para comprar |
| **Domain Vision Statement** | Documento curto que descreve o Core Domain e seu valor estrategico |
| **Highlighted Core** | Documento ou diagrama que destaca os elementos mais importantes do modelo de dominio |
| **Segregated Core** | Refatoracao que separa fisicamente o Core Domain de subdominios |
| **Specification** | Objeto que encapsula uma regra de negocio como predicado testavel e combinavel |
| **Layered Architecture** | Arquitetura que separa UI, Application, Domain e Infrastructure em camadas com dependencias controladas |
| **Distillation** | Processo estrategico de identificar e isolar o Core Domain dos subdominios genericos e de suporte |
| **Supple Design** | Conjunto de tecnicas (Intention-Revealing Interfaces, Side-Effect-Free Functions, Assertions) que tornam o modelo facil de usar e evoluir |
| **Intention-Revealing Interface** | Interface cujos nomes de metodos e parametros comunicam claramente o proposito sem necessidade de ler a implementacao |
| **Side-Effect-Free Function** | Funcao que retorna resultado sem modificar estado observavel, facilitando composicao e raciocinio |
| **Closure of Operations** | Operacao cujo tipo de retorno e o mesmo dos operandos, permitindo composicao fluente (ex: `Dinheiro + Dinheiro = Dinheiro`) |
