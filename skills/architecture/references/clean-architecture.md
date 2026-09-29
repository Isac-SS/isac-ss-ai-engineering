# Skill: Clean Architecture — Robert C. Martin

## Perspectiva do Autor

Robert C. Martin ("Uncle Bob") é um engenheiro de software com mais de cinco décadas de experiência que defende, acima de tudo, que **a arquitetura de um sistema existe para servir ao negócio, não à tecnologia**. Na visão dele, o objetivo primário da arquitetura de software é minimizar os recursos humanos necessários para construir e manter o sistema ao longo do tempo. Ele acredita que o valor de um software se divide em dois tipos: o valor de comportamento (o que o software faz hoje) e o valor de estrutura (a facilidade com que o software pode ser alterado amanhã). Martin argumenta que o valor de estrutura é, paradoxalmente, o mais importante — pois um software que funciona mas não pode ser alterado eventualmente se tornará inútil, enquanto um software que não funciona mas é fácil de alterar pode ser consertado.

Martin prioriza:
- **Independência de frameworks**: frameworks são ferramentas, não o centro do sistema.
- **Testabilidade**: a arquitetura deve permitir testar regras de negócio sem UI, banco de dados ou qualquer elemento externo.
- **Independência de UI**: a interface do usuário deve poder mudar sem afetar as regras de negócio.
- **Independência de banco de dados**: deve ser possível trocar Oracle por PostgreSQL, MongoDB ou qualquer outro, sem impactar as regras de negócio.
- **Independência de agentes externos**: as regras de negócio não devem saber nada sobre o mundo exterior.

Para Martin, código limpo e arquitetura limpa não são luxos — são a forma mais econômica de desenvolver software a longo prazo. Ele vê a degradação arquitetural como a principal causa de projetos que se tornam cada vez mais caros de manter.

---

## Princípios Fundamentais

### SOLID

Os princípios SOLID, segundo Martin, não são regras para código orientado a objetos apenas — são princípios para organizar módulos de nível intermediário (classes, módulos, componentes) de forma que sejam tolerantes a mudanças, fáceis de entender e reutilizáveis.

#### SRP — Single Responsibility Principle

**Definição**: Um módulo deve ter um, e apenas um, motivo para mudar. Martin reformulou isso de forma mais precisa: **um módulo deve ser responsável por um, e apenas um, ator** (stakeholder ou grupo de stakeholders).

**Explicação prática**: SRP não significa "fazer apenas uma coisa". Significa que a classe ou módulo deve agrupar código que muda pelas mesmas razões e ao mesmo tempo. Se o departamento financeiro e o departamento de RH usam a mesma classe `Employee`, e cada um requer mudanças diferentes, essa classe viola SRP — porque dois atores distintos são fontes de mudança.

**Implicação prática**:
- Separar dados e funções que servem a atores diferentes em módulos diferentes.
- Quando um módulo muda por mais de uma razão, ele deve ser dividido.
- Fachadas (Facades) podem ser usadas para reunificar a interface sem violar SRP.

**Como verificar no código**:
1. Identifique quais atores (stakeholders) cada método da classe serve.
2. Se métodos diferentes servem atores diferentes, o SRP está violado.
3. Verifique se a descrição da responsabilidade da classe contém a palavra "e" — isso geralmente indica violação.
4. Analise o histórico de commits: se a classe muda por razões fundamentalmente diferentes, há violação.

#### OCP — Open-Closed Principle

**Definição**: Um artefato de software deve ser aberto para extensão e fechado para modificação.

**Explicação prática**: Martin defende que o objetivo é permitir que o comportamento do sistema mude adicionando código novo, sem alterar código existente. Isso é alcançado através de abstrações e da Dependency Rule. A arquitetura ideal protege os componentes de nível mais alto (regras de negócio) das mudanças nos componentes de nível mais baixo (detalhes).

**Implicação prática**:
- Use interfaces e abstrações nos pontos onde mudanças são prováveis.
- Organize dependências de forma que componentes de alta política não dependam de componentes de baixo nível.
- A direção de proteção vai do mais importante (Entities) para o menos importante (Details).

**Como verificar no código**:
1. Simule uma mudança de requisito (ex: novo formato de relatório). Quantos arquivos precisam mudar?
2. Verifique se adicionar um novo comportamento requer apenas criar novas classes/módulos, sem alterar os existentes.
3. Procure por `switch/case` ou cadeias de `if/else` sobre tipos — geralmente indicam que OCP está violado.
4. Verifique se as interfaces estão nos pontos corretos de variabilidade.

#### LSP — Liskov Substitution Principle

**Definição**: Objetos de um subtipo devem poder substituir objetos do tipo base sem alterar a correção do programa.

**Explicação prática**: Martin estende LSP além de herança de classes — aplica-o a interfaces, APIs REST, e qualquer contrato de serviço. Uma violação clássica é a interface `Rectangle` com `setWidth` e `setHeight` onde `Square` herda e viola as expectativas do chamador.

**Implicação prática**:
- Subtipos devem honrar completamente o contrato do tipo base — pré-condições, pós-condições e invariantes.
- APIs versionadas devem manter compatibilidade com versões anteriores.
- Se um subtipo precisa lançar `UnsupportedOperationException` para um método herdado, há violação.

**Como verificar no código**:
1. Para cada implementação de interface, verifique se todos os métodos têm implementação real (sem `throw UnsupportedOperationException`).
2. Verifique se o código cliente faz verificações de tipo (`instanceof`, `is`, `typeof`) antes de chamar métodos — isso indica que subtipos não são verdadeiramente substituíveis.
3. Procure por cláusulas condicionais que verificam o tipo concreto de uma abstração.
4. Valide que pré-condições não são fortalecidas e pós-condições não são enfraquecidas em subtipos.

#### ISP — Interface Segregation Principle

**Definição**: Clientes não devem ser forçados a depender de interfaces que não usam.

**Explicação prática**: Martin alerta que depender de algo que carrega funcionalidade desnecessária pode causar problemas inesperados. Se o módulo A depende de um framework F, e F contém funcionalidade usada pelo módulo B, mudanças em F para atender B podem quebrar A — mesmo que A não use essa funcionalidade. Esse princípio vale para imports de módulos, dependências Maven/npm e até microserviços.

**Implicação prática**:
- Crie interfaces específicas para cada cliente, não interfaces genéricas e gordas.
- Em linguagens dinâmicas, o risco é menor mas o princípio ainda se aplica a nível de módulo.
- Em nível arquitetural: não dependa de serviços inteiros se você precisa de apenas uma operação.

**Como verificar no código**:
1. Verifique se alguma classe implementa métodos de interface que são vazios, lançam exceção ou retornam `null`/valores default.
2. Conte quantos métodos cada cliente realmente usa da interface — se é menos da metade, a interface é gorda.
3. Analise se mudanças em uma interface forçam recompilação/redeploy de módulos que não usam os métodos alterados.
4. Verifique se interfaces têm mais de 5-7 métodos — geralmente precisam ser segregadas.

#### DIP — Dependency Inversion Principle

**Definição**: Módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de abstrações. Abstrações não devem depender de detalhes. Detalhes devem depender de abstrações.

**Explicação prática**: Martin afirma que este é o princípio mais importante para definir fronteiras arquiteturais. A inversão acontece quando, em vez de o Use Case chamar diretamente o Repository concreto, ele define uma interface (port) e o Repository concreto a implementa. O fluxo de controle vai do Use Case para o Repository, mas a dependência de código-fonte aponta do Repository para o Use Case.

**Implicação prática**:
- Regras de negócio definem as interfaces que precisam, não se adaptam a interfaces impostas por frameworks ou bancos.
- Factories e injeção de dependência são mecanismos para resolver DIP em tempo de execução.
- Elementos voláteis (concretos, frequentemente modificados) nunca devem ser importados por elementos estáveis (abstratos, raramente modificados).

**Como verificar no código**:
1. Trace os `import`/`require`/`using` de cada arquivo — os imports devem apontar em direção ao centro (Entities/Use Cases), nunca para fora.
2. Verifique se classes de regras de negócio importam frameworks, bibliotecas de ORM, HTTP, etc.
3. Confirme que interfaces estão declaradas no módulo que as usa (o consumidor), não no módulo que as implementa (o provedor).
4. Verifique se o `main` ou a camada de composição é o único lugar que conhece tipos concretos.

---

### Component Principles

Martin organiza os princípios de componentes em dois grupos: coesão (o que colocar dentro de um componente) e acoplamento (como os componentes se relacionam).

#### Princípios de Coesão

##### REP — Reuse/Release Equivalence Principle

**Definição**: A granularidade de reutilização é a granularidade de release.

**Explicação**: Classes e módulos agrupados em um componente devem ser lançáveis juntos. Se são reutilizáveis juntos, devem estar no mesmo componente com versionamento semântico. Martin argumenta que classes que não fazem sentido sendo lançadas juntas não devem estar no mesmo componente.

**Verificação**: Pergunte-se: "Faz sentido versionar e publicar todas essas classes juntas?" Se um subconjunto muda independentemente, deve ser extraído.

##### CCP — Common Closure Principle

**Definição**: Agrupe no mesmo componente as classes que mudam pelas mesmas razões e ao mesmo tempo. Separe em componentes diferentes as classes que mudam em momentos diferentes e por razões diferentes.

**Explicação**: É o SRP aplicado a componentes. Martin defende que a manutenibilidade geralmente é mais importante que a reutilizabilidade. Se uma mudança de requisito afeta apenas um componente, o custo de deploy é minimizado.

**Verificação**: Analise o histórico de commits. Se uma mudança de feature típica toca arquivos em muitos componentes, o CCP está violado. Use `git log --name-only` para rastrear quais arquivos mudam juntos frequentemente.

##### CRP — Common Reuse Principle

**Definição**: Não force os usuários de um componente a depender de coisas que não precisam.

**Explicação**: É o ISP aplicado a componentes. Se um componente contém classes que nem todos os consumidores usam, qualquer mudança nessas classes forçará recompilação/redeploy desnecessário. Martin diz que este princípio puxa na direção oposta ao CCP — é necessário encontrar o equilíbrio.

**Verificação**: Para cada consumidor do componente, verifique se ele usa todas (ou quase todas) as classes exportadas. Se usa apenas 10%, o componente provavelmente é grande demais.

#### Princípios de Acoplamento

##### ADP — Acyclic Dependencies Principle

**Definição**: O grafo de dependências entre componentes não deve ter ciclos.

**Explicação**: Martin descreve dois problemas de ciclos: (1) "morning after syndrome" — onde mudanças em um componente quebram outro inesperadamente; (2) builds se tornam impossíveis de fazer incrementalmente. Ciclos são quebrados com DIP (invertendo a dependência) ou criando um novo componente que ambos dependem.

**Verificação**: Gere o grafo de dependências (com ferramentas como `madge`, `deptree`, `jdepend`, `NDepend`). Se houver algum ciclo, é uma violação. Automatize essa verificação no CI.

##### SDP — Stable Dependencies Principle

**Definição**: Dependa na direção da estabilidade.

**Explicação**: Martin define estabilidade como a dificuldade de mudar um componente — não porque é difícil de entender, mas porque muitos outros componentes dependem dele. Um componente com muitos dependentes é estável (irresponsável mudar). Um componente com poucos dependentes é instável (fácil de mudar). Componentes instáveis devem depender de componentes estáveis, nunca o contrário.

**Métricas**:
- **Fan-in**: número de classes externas que dependem de classes dentro do componente.
- **Fan-out**: número de classes externas das quais classes dentro do componente dependem.
- **I (Instabilidade)** = Fan-out / (Fan-in + Fan-out). Varia de 0 (máxima estabilidade) a 1 (máxima instabilidade).

**Verificação**: Calcule I para cada componente. Verifique que toda dependência vai de I alto para I baixo (ou igual). Se um componente com I=0.1 depende de um com I=0.9, há violação.

##### SAP — Stable Abstractions Principle

**Definição**: Um componente deve ser tão abstrato quanto estável.

**Explicação**: Martin conecta estabilidade com abstração. Componentes estáveis (I baixo) devem ser abstratos para que sua estabilidade não impeça extensão (OCP). Componentes instáveis (I alto) devem ser concretos, pois sua instabilidade permite que o código concreto seja facilmente alterado.

**Métricas**:
- **A (Abstração)** = número de classes abstratas e interfaces / número total de classes no componente. Varia de 0 (totalmente concreto) a 1 (totalmente abstrato).

#### Métricas Combinadas e Zonas

- **Main Sequence**: a linha reta de (I=1, A=0) a (I=0, A=1) no gráfico I x A. É a posição ideal.
- **D (Distância da Main Sequence)** = |A + I - 1|. Deve ser o mais próximo possível de 0.
- **Zone of Pain** (I=0, A=0): componente estável e concreto. Muito difícil de mudar e não extensível. Exemplos aceitáveis: classes utilitárias altamente estáveis como `String`. Exemplos problemáticos: schemas de banco de dados concretos dos quais tudo depende.
- **Zone of Uselessness** (I=1, A=1): componente instável e totalmente abstrato. Ninguém depende dele. Geralmente é código morto — interfaces que nunca foram implementadas.

**Como verificar**: Use ferramentas de análise estática para calcular I, A e D de cada componente. Estabeleça thresholds no CI: D > 0.3 deve gerar warning, D > 0.6 deve falhar o build.

---

### Architecture

#### The Dependency Rule — Os 4 Círculos

Martin define quatro camadas concêntricas. **A regra fundamental é: dependências de código-fonte devem apontar apenas para dentro — em direção às políticas de nível mais alto.**

1. **Entities (centro)**: Regras de negócio críticas da empresa (Enterprise Business Rules). Existem independentemente de qualquer sistema. São os objetos de negócio com os métodos que implementam regras válidas em qualquer contexto. Mudam apenas quando as regras de negócio fundamentais mudam.

2. **Use Cases**: Regras de negócio da aplicação (Application Business Rules). Orquestram o fluxo de dados de e para as Entities. Definem como o sistema automatiza um processo específico. Contêm a lógica de "quando o usuário faz X, o sistema faz Y com a Entity Z". Não sabem nada sobre UI, banco ou frameworks.

3. **Interface Adapters**: Convertem dados entre o formato conveniente para Use Cases/Entities e o formato conveniente para agentes externos (banco, web, dispositivos). Aqui ficam Controllers, Presenters, Gateways, ViewModels. O padrão MVC inteiro reside nesta camada.

4. **Frameworks & Drivers (borda externa)**: O nível mais externo. Web frameworks, banco de dados, UI frameworks, dispositivos. Pouco código próprio — são "glue code" para conectar à camada de adapters.

**Regra crítica sobre dados que cruzam boundaries**: Quando dados cruzam uma fronteira, devem estar em uma estrutura simples (DTO, hashmap, tupla) conveniente para a camada interna. A camada interna **nunca** deve receber um objeto que pertence ao framework (ex: `HttpRequest`, `ResultSet`, `ActiveRecord model`).

#### Policy vs Detail

Martin faz uma distinção crucial:
- **Policy**: as regras de negócio, o que o sistema faz. Componentes de alto nível.
- **Detail**: os mecanismos de como o sistema opera — banco de dados, protocolo web, framework, sistema operacional. Componentes de baixo nível.

A arquitetura deve deixar as decisões de Detail abertas o máximo possível. Martin argumenta que um bom arquiteto maximiza o número de decisões que **não** precisam ser tomadas cedo. O banco de dados pode ser escolhido depois. O framework web pode ser escolhido depois. O protocolo de entrega pode ser escolhido depois. Quanto mais tempo essas decisões forem adiadas, mais informação teremos para tomá-las corretamente.

#### Screaming Architecture

Martin afirma que ao olhar para a estrutura de diretórios de um projeto, a arquitetura deveria "gritar" o propósito do sistema. Se olhando para os diretórios você vê `controllers/`, `models/`, `views/`, `services/` — a arquitetura grita "sou um framework MVC". Se você vê `patients/`, `appointments/`, `prescriptions/`, `billing/` — a arquitetura grita "sou um sistema de saúde".

**Verificação**: Mostre a estrutura de diretórios de primeiro e segundo nível a alguém. Se a pessoa identificar o framework antes do domínio, a arquitetura não está "gritando" corretamente.

#### Humble Object Pattern

Martin descreve este padrão como estratégia para tornar código testável. A ideia é separar comportamentos difíceis de testar (I/O, UI, rede) em um objeto "humilde" que faz o mínimo possível, e colocar toda a lógica em um objeto testável.

**Exemplos**:
- **Presenters**: o Presenter formata dados (lógica testável), o View apenas renderiza (humble object).
- **Database Gateways**: o Gateway implementa queries (humble object), o Use Case contém a lógica de orquestração (testável).
- **Service Listeners**: o listener faz unmarshal de dados e encaminha (humble object), o service executor contém a lógica (testável).

**Verificação**: Se uma classe que lida com I/O contém `if/else` com lógica de negócio, o Humble Object pattern não foi aplicado. Toda lógica condicional deve estar no objeto testável.

#### Boundaries — Tipos de Fronteiras

Martin descreve diferentes graus de separação entre componentes:

1. **Full Boundary (Fronteira Completa)**: Interfaces bidirecionais com Data Transfer Objects cruzando a fronteira. Custo mais alto, máxima flexibilidade. Use quando a separação é crítica e provável que se torne deployment independente.

2. **Partial Boundary (Fronteira Parcial)**: Variantes com custo menor:
   - **Strategy Pattern**: uma interface com uma implementação. Prepara o ponto de extensão sem o custo total.
   - **Facade Pattern**: uma classe fachada que oculta os serviços internos. Perde a inversão de dependência.

3. **Monolith (Source-Level Boundary)**: Componentes separados por disciplina de código-fonte (packages, namespaces, módulos) mas deployados como um único artefato. A separação é mantida por convenção e regras de import.

4. **Deployment Boundary**: Componentes deployados como DLLs, JARs, gems, node_modules independentes. Custo intermediário.

5. **Service Boundary**: Comunicação via rede (REST, gRPC, mensageria). Custo mais alto. Martin alerta: **serviços não são automaticamente arquitetura** — se os serviços são acoplados no nível de dados (compartilham schemas), eles são apenas um monolito distribuído mais caro.

#### Frameworks, DB e Web como Details

**Frameworks**: Martin argumenta que o autor de um framework não conhece seus problemas específicos. O framework quer que você case com ele — que suas classes herdem dele, que seus módulos importem seus decorators. Resista. Trate frameworks como detalhes. Não deixe que o framework "invada" suas Entities e Use Cases. Use o framework na camada externa e interaja com ele via adapters.

**Banco de Dados**: O modelo relacional é um detail. SQL é um detail. O banco é uma ferramenta de armazenamento de dados, não uma arquitetura. Martin defende que as Entities devem usar estruturas de dados simples (plain objects) e nunca annotations de ORM, herança de `ActiveRecord`, ou decorators de frameworks de persistência.

**Web**: HTTP, REST, GraphQL são mecanismos de delivery. A web é um I/O device. A arquitetura não deve girar em torno do protocolo de entrega. Use Cases não devem saber se os dados vêm de uma requisição HTTP ou de uma fila de mensagens.

---

### Code Organization (Simon Brown — Chapter 34)

Martin incluiu o capítulo de Simon Brown sobre organização de código, reconhecendo a importância de traduzir princípios arquiteturais em estrutura de diretórios e regras de visibilidade.

#### Package by Layer

Organização horizontal por camada técnica:
```
com.app.controllers/
com.app.services/
com.app.repositories/
com.app.models/
```

**Problemas**: Violação de Screaming Architecture. Fácil de "pular" camadas. Alta probabilidade de mudanças em cascata entre packages.

#### Package by Feature

Organização vertical por feature de negócio:
```
com.app.orders/
com.app.payments/
com.app.customers/
```

**Vantagem**: Screaming Architecture. Mudanças tendem a ficar contidas em um único package.
**Problema**: Sem separação clara entre regras de negócio e detalhes de infraestrutura dentro do package.

#### Ports and Adapters

Separação explícita entre interior (domínio) e exterior (infraestrutura):
```
com.app.domain/
    models/
    ports/
    services/
com.app.infrastructure/
    persistence/
    web/
    messaging/
```

**Vantagem**: Dependency Rule visível na estrutura. Interior não conhece o exterior.
**Problema**: Pode degenerar se a disciplina de imports não for mantida.

#### Package by Component

Abordagem de Simon Brown: empacotamento que agrupa "dentro do componente" a regra de negócio, a interface de persistência e a implementação, expondo apenas uma interface limpa:
```
com.app.orders/
    OrderService.java       (interface pública)
    OrderServiceImpl.java   (package-private)
    OrderRepository.java    (package-private interface)
    OrderRepositoryImpl.java (package-private)
```

**Vantagem**: O compilador enforce que só a interface pública é acessível. Encapsulamento real.

#### Encapsulation > Organization

Brown enfatiza (e Martin concorda) que a organização de diretórios é inútil se os modificadores de acesso não reforçam as regras. Em Java, tudo `public` destrói qualquer intenção arquitetural. Em linguagens sem modificadores de acesso fortes (Python, JavaScript), é preciso usar ferramentas de lint, regras de import e CI para compensar.

**Verificação**: Examine quantas classes de um package são `public`. Se todas são públicas, a organização é decorativa — qualquer coisa pode depender de qualquer coisa. Use ferramentas como ArchUnit, deptrac, eslint-plugin-import para enforcement automatizado.

#### Compiler as Enforcer

O ideal é que o compilador (ou analisador estático) impeça violações da arquitetura. Se uma classe interna de um componente não é visível fora dele (package-private, internal), nenhum desenvolvedor pode acidentalmente criar uma dependência indevida.

**Verificação**: Tente, a partir de um módulo externo, importar uma classe interna. Se o compilador/IDE permite, o enforcement falhou.

---

## Heurísticas Verificáveis

Lista numerada de regras concretas que podem ser verificadas no código. Cada item inclui o método de verificação.

1. **Nenhum import de framework nas Entities.** Verificação: `grep` por imports de frameworks (Spring, Django, Rails, Express, etc.) nos arquivos de Entity. Resultado esperado: zero ocorrências.

2. **Nenhum import de framework nos Use Cases.** Verificação: mesma abordagem. Use Cases não devem ter annotations como `@Transactional`, `@Controller`, `@Injectable` etc.

3. **Use Cases recebem e retornam DTOs, nunca objetos de framework.** Verificação: verifique os tipos dos parâmetros e retornos dos métodos de Use Case. Nenhum deve ser `HttpRequest`, `Request`, `ResultSet`, etc.

4. **Entities não têm dependência em nenhuma camada externa.** Verificação: analise o grafo de dependências da Entity. Fan-out para qualquer coisa fora do domínio deve ser zero.

5. **Toda dependência de código-fonte aponta para dentro (em direção ao centro).** Verificação: use ferramentas de análise de dependências e confirme que nenhum módulo central importa módulos de camadas externas.

6. **Interfaces são declaradas no lado do consumidor, não do implementador.** Verificação: a interface `OrderRepository` deve estar no package de Use Cases (ou Domain), não no package de persistência. Verifique a localização física do arquivo.

7. **Não existem ciclos no grafo de dependências entre componentes.** Verificação: execute análise de dependências com detecção de ciclos (`madge --circular`, ArchUnit, deptrac). Resultado esperado: zero ciclos.

8. **Cada Use Case é uma classe/função separada.** Verificação: busque por classes com mais de um método público de "execução" (ex: `execute`, `handle`, `run`). Cada Use Case deve ter um único ponto de entrada.

9. **O `main` (composition root) é o único lugar que instancia dependências concretas.** Verificação: busque por `new ConcreteClass()` fora do composition root ou factory. Se Use Cases ou Entities fazem `new`, há violação.

10. **Nenhum Use Case conhece o mecanismo de delivery (HTTP, CLI, gRPC).** Verificação: busque referências a status codes HTTP, headers, métodos HTTP em arquivos de Use Case. Resultado esperado: zero.

11. **Testes de Use Cases rodam sem banco de dados, sem HTTP, sem filesystem.** Verificação: execute os testes de Use Case isoladamente. Se precisam de Docker, banco em memória ou servidor HTTP para rodar, há acoplamento.

12. **A estrutura de diretórios reflete o domínio, não o framework.** Verificação: liste os diretórios de primeiro nível dentro de `src/`. Se a maioria tem nomes de domínio (`orders`, `payments`, `users`), está correto. Se tem nomes técnicos (`controllers`, `models`, `services`), está incorreto.

13. **DTOs que cruzam boundaries não contêm lógica.** Verificação: busque métodos (além de getters/construtores) em DTOs. Resultado esperado: zero lógica de negócio.

14. **Nenhuma Entity herda de uma classe de framework (ActiveRecord, Model, Document).** Verificação: verifique as superclasses das Entities. Não devem ser classes de ORM/framework.

15. **A Instabilidade (I) de componentes de domínio é menor que a de componentes de infraestrutura.** Verificação: calcule I para cada componente. Domínio deve ter I baixo, infraestrutura deve ter I alto.

16. **Componentes estáveis (I baixo) são proporcionalmente abstratos (A alto).** Verificação: calcule A e I. A Distância D da Main Sequence deve ser < 0.3.

17. **Nenhum componente está na Zone of Pain (I baixo, A baixo) exceto utilitários estáveis.** Verificação: identifique componentes com I < 0.2 e A < 0.2. Se não são bibliotecas utilitárias fundamentais, há problema.

18. **Mudanças em um Use Case não exigem alteração em mais de um Adapter.** Verificação: simule uma mudança de regra de negócio. Conte quantos adapters precisam mudar. Idealmente zero; aceitável um.

19. **Presenters não contêm lógica de negócio; apenas formatam dados para exibição.** Verificação: busque condicionais de negócio em Presenters. Regras como "se o saldo é negativo, aplique juros" não pertencem ao Presenter.

20. **Controllers são finos — delegam imediatamente para Use Cases.** Verificação: conte linhas de código em Controllers. Se um método de Controller tem mais de ~15 linhas (excluindo boilerplate de framework), provavelmente contém lógica que deveria estar no Use Case.

21. **O banco de dados é tratado como plugin.** Verificação: verifique se é possível trocar o mecanismo de persistência (ex: de PostgreSQL para MongoDB) alterando apenas o composition root e os adapters, sem tocar em Entities ou Use Cases.

22. **Não há ORM annotations nas Entities de domínio.** Verificação: busque `@Entity`, `@Column`, `@Table`, `@Field`, `mapped_column`, etc. nas classes de domínio. Use modelos de persistência separados.

23. **Objetos de valor (Value Objects) são imutáveis.** Verificação: busque setters ou mutação de estado em Value Objects. Todos os campos devem ser `final`/`readonly`/`const`.

24. **Exceções de negócio são definidas no domínio, não nos adapters.** Verificação: verifique onde as exceptions de domínio (`InsufficientFundsException`, `OrderNotFoundException`) estão declaradas. Devem estar no package de domínio.

25. **Não há lógica condicional em Humble Objects (Views, DB Gateways, Message Listeners).** Verificação: busque `if`, `switch`, ternários em Views e Gateways. Se contêm lógica além de mapeamento trivial, o padrão está sendo violado.

26. **Testes unitários do domínio representam mais de 60% dos testes totais.** Verificação: conte os testes por camada. A maioria deve testar Entities e Use Cases, não Controllers e integrações.

27. **Nenhum import entre features/módulos passa por camadas de infraestrutura.** Verificação: se o módulo `orders` precisa do módulo `payments`, a dependência deve ser entre seus domínios (via interfaces), não entre seus adapters.

28. **O tempo de build e teste do domínio isolado é significativamente menor que o do sistema inteiro.** Verificação: execute apenas os testes de domínio e compare com o tempo total. Se não há diferença significativa, o domínio provavelmente não está isolado.

---

## Anti-patterns com Sinais no Código

| # | Anti-pattern | Sinal no Código | Severidade | O que Fazer |
|---|-------------|-----------------|------------|-------------|
| 1 | **Framework Coupling** | Entities com `@Entity`, `@Column`, annotations de ORM | Alta | Criar modelos de persistência separados e mappers na camada de adapters |
| 2 | **Smart Controller** | Controllers com mais de 20 linhas de lógica, condicionais de negócio, cálculos | Alta | Extrair lógica para Use Case dedicado; Controller deve apenas converter request e chamar Use Case |
| 3 | **Anemic Domain Model** | Entities com apenas getters/setters; toda lógica em "services" externos | Média | Mover comportamento de negócio para dentro das Entities; services orquestram, não calculam |
| 4 | **God Class** | Classe com mais de 500 linhas ou mais de 10 dependências injetadas | Alta | Identificar responsabilidades distintas e extrair em classes separadas aplicando SRP |
| 5 | **Dependency Cycle** | Componente A importa B que importa C que importa A | Alta | Aplicar DIP para inverter uma das dependências ou criar componente intermediário |
| 6 | **Leaky Abstraction** | Use Case retorna `ResultSet`, `HttpResponse`, `QueryBuilder` | Alta | Criar DTO específico para a fronteira; mapear na camada de adapters |
| 7 | **Shotgun Surgery** | Uma mudança de requisito simples toca 8+ arquivos em 4+ packages | Alta | Reagrupar código por CCP — o que muda junto fica junto |
| 8 | **Feature Envy** | Método de uma classe usa mais dados de outra classe do que da própria | Média | Mover o método para a classe cujos dados ele mais utiliza |
| 9 | **Primitive Obsession** | CPF como `String`, dinheiro como `double`, email como `String` por todo o domínio | Média | Criar Value Objects tipados (`Cpf`, `Money`, `Email`) com validação interna |
| 10 | **Boolean Parameters** | `processOrder(order, true, false, true)` | Média | Substituir por objetos de configuração, enums ou métodos separados |
| 11 | **Service Locator** | `ServiceLocator.get(OrderRepository.class)` dentro de Use Cases | Alta | Substituir por constructor injection; apenas o composition root conhece o container |
| 12 | **Shared Database Integration** | Dois serviços/módulos acessam as mesmas tabelas diretamente | Alta | Introduzir APIs entre os módulos; cada módulo é dono exclusivo de suas tabelas |
| 13 | **Premature Microservices** | Sistema com 3 desenvolvedores e 12 microserviços | Média | Consolidar em monolito modular com boundaries source-level; extrair serviço só quando houver motivo real de deploy independente |
| 14 | **Circular Reference entre Layers** | Controller importa Entity diretamente, Entity importa DTO de API | Alta | Respeitar Dependency Rule; converter dados nas fronteiras |
| 15 | **Test Coupling to Implementation** | Testes que fazem mock de métodos privados ou verificam chamadas internas | Média | Testar comportamento (output dado input), não implementação; usar mocks apenas nas fronteiras |
| 16 | **Config in Business Logic** | Use Case lê variáveis de ambiente ou arquivos de configuração diretamente | Média | Injetar configuração via construtor como valores tipados; Use Case não conhece a fonte |
| 17 | **Mixed Levels of Abstraction** | Mesmo método faz validação de negócio, query SQL e formatação de JSON | Alta | Separar em camadas: validação no Use Case, query no Gateway, formatação no Presenter |
| 18 | **Interface with Single Implementation (sem justificativa)** | `IOrderService` com apenas `OrderServiceImpl`, sem variação previsível e sem necessidade de teste double | Baixa | Interfaces sem purpose arquitetural adicionam complexidade; manter apenas nas fronteiras arquiteturais |
| 19 | **Barrel Exports que Quebram Encapsulamento** | `index.ts` que re-exporta tudo do módulo, incluindo classes internas | Média | Exportar apenas a API pública do módulo; manter internals privados |
| 20 | **Entity como DTO** | Mesma classe usada como Entity de domínio, modelo de ORM e payload de API | Alta | Criar classes separadas para cada contexto com mappers explícitos nas fronteiras |
| 21 | **Transaction Script Disfarçado** | Toda "arquitetura" é um Controller chamando um Service procedural que faz tudo sequencialmente | Média | Identificar Entities com comportamento; distribuir lógica conforme Clean Architecture |
| 22 | **Dependency on Concrete Volatile Classes** | Use Case faz `new PostgresOrderRepository()` diretamente | Alta | Depender de interface; injetar implementação concreta via construtor a partir do composition root |

---

## Checklist de Revisão

### Entities

- [ ] Entities contêm regras de negócio críticas (validações, cálculos, invariantes)?
- [ ] Entities são independentes de qualquer framework, ORM, ou biblioteca externa?
- [ ] Entities não possuem annotations de persistência, serialização ou framework?
- [ ] Value Objects são imutáveis e possuem igualdade por valor (não por referência)?
- [ ] Exceções de domínio são definidas junto às Entities?
- [ ] Entities possuem construtores que garantem estado válido (no invalid state)?
- [ ] Não há getters/setters desnecessários que expõem detalhes internos?
- [ ] Entities podem ser instanciadas e testadas sem nenhuma dependência externa?

### Use Cases

- [ ] Cada Use Case é uma classe/função com responsabilidade única e clara?
- [ ] Use Cases dependem apenas de interfaces (ports), nunca de implementações concretas?
- [ ] Use Cases definem seus próprios Input/Output DTOs?
- [ ] Use Cases não contêm referências a HTTP, SQL, filas, filesystem ou qualquer mecanismo de I/O?
- [ ] Use Cases orquestram Entities sem duplicar a lógica de negócio delas?
- [ ] Use Cases podem ser testados com mocks/stubs simples, sem infraestrutura?
- [ ] A nomenclatura dos Use Cases reflete ações de negócio (`CriarPedido`, `CancelarAssinatura`), não operações CRUD?
- [ ] Use Cases não retornam Entities diretamente, mas DTOs de output?
- [ ] Transações e controle de persistência estão fora do Use Case (no adapter ou decorator)?

### Interface Adapters

- [ ] Controllers convertem dados de entrada (HTTP, CLI, evento) para o formato do Use Case e nada mais?
- [ ] Presenters/ViewModels convertem dados de saída do Use Case para o formato da View e nada mais?
- [ ] Gateways/Repositories implementam interfaces definidas na camada de domínio/Use Case?
- [ ] Não há lógica de negócio em Controllers, Presenters ou Gateways?
- [ ] Mappers entre modelos de domínio e modelos de persistência/API são explícitos?
- [ ] Erros de domínio são traduzidos para formatos apropriados do mecanismo de delivery (ex: HTTP status codes)?

### Frameworks & Drivers

- [ ] Configuração de framework (routes, middleware, ORM config) está isolada na camada mais externa?
- [ ] O composition root (main, bootstrap, app factory) é o único lugar que monta o grafo de dependências?
- [ ] Mudança de framework (ex: Express para Fastify, Spring para Quarkus) exigiria alteração apenas nesta camada e nos adapters?
- [ ] Migrations de banco de dados são tratadas como infraestrutura, não como domínio?
- [ ] Libraries de terceiros são wrappadas em adapters quando usadas em lógica de negócio?

### Components

- [ ] O grafo de dependências entre componentes é acíclico?
- [ ] Componentes de domínio têm Instabilidade (I) menor que componentes de infraestrutura?
- [ ] Componentes estáveis são suficientemente abstratos (perto da Main Sequence)?
- [ ] Nenhum componente está na Zone of Pain sem justificativa?
- [ ] Componentes que mudam juntos estão no mesmo package (CCP)?
- [ ] Componentes podem ser compilados e testados independentemente?
- [ ] A visibilidade de classes/módulos reforça os boundaries (package-private, internal, module exports)?

---

## Perguntas que esta Skill Faz

1. **"Se eu olhar a estrutura de diretórios, consigo saber que é um sistema de pagamentos ou apenas que é Rails/Spring/Express?"** — Screaming Architecture.

2. **"Consigo testar as regras de negócio sem subir o banco de dados, sem iniciar o servidor HTTP e sem carregar o framework?"** — Testabilidade e isolamento do domínio.

3. **"Se amanhã precisarmos trocar PostgreSQL por DynamoDB, quantos arquivos precisam mudar e nenhum deles é de regra de negócio?"** — Banco de dados como Detail.

4. **"Quem define a interface do Repository — o domínio que a consome ou a infraestrutura que a implementa?"** — DIP e ownership de interfaces.

5. **"Esta Entity sabe que existe um banco de dados? Ela sabe que existe HTTP? Ela sabe que existe um framework?"** — Dependency Rule.

6. **"Se eu remover o framework web e substituir por uma CLI, quanto do sistema continua funcionando sem alteração?"** — Web como Detail.

7. **"Esta classe muda por qual motivo? Atende a quais atores? Mais de um?"** — SRP.

8. **"O que acontece se eu adicionar um novo tipo de relatório — preciso modificar código existente ou apenas adicionar código novo?"** — OCP.

9. **"Existem ciclos no grafo de dependências entre os componentes? Posso compilar cada componente isoladamente?"** — ADP.

10. **"O Use Case sabe que os dados vêm de uma requisição HTTP? Ele retorna HTTP status codes?"** — Isolamento de mecanismo de delivery.

11. **"Há lógica de negócio no Controller? O Controller decide algo além de 'para qual Use Case encaminhar'?"** — Smart Controller anti-pattern.

12. **"Os modelos de persistência (com annotations de ORM) são os mesmos usados como Entities de domínio?"** — Entity como DTO anti-pattern.

13. **"Este componente estável é suficientemente abstrato, ou está na Zone of Pain?"** — SAP e métricas.

14. **"Se um novo desenvolvedor entrar no time amanhã, ele precisa entender o framework antes de entender o negócio?"** — Framework como Detail.

15. **"As boundaries entre componentes são enforced pelo compilador/tooling ou dependem apenas de convenção e disciplina humana?"** — Encapsulation > Organization.

16. **"Quantos Use Cases existem com nomes como 'CRUD de X' em vez de ações de negócio significativas como 'Aprovar Pedido' ou 'Calcular Frete'?"** — Use Cases devem representar intenções do negócio.

17. **"Se este serviço que consumimos ficar fora do ar, o domínio continua consistente ou corrompe?"** — Boundaries e resiliência.

18. **"As exceções que o domínio lança são exceções de negócio com semântica clara ou são exceções técnicas repackaged?"** — Exceções como parte do domínio.

---

## Vocabulário

| Termo | Definição |
|-------|-----------|
| **Entity** | Objeto de negócio que contém regras de negócio críticas da empresa (Enterprise Business Rules). Existe independentemente de qualquer sistema ou aplicação. Possui identidade única. |
| **Use Case** | Regra de negócio específica da aplicação. Define como o sistema automatiza um processo, orquestrando Entities. Também chamado de Interactor. |
| **Interface Adapter** | Componente que converte dados entre o formato do domínio e o formato de agentes externos (web, banco, dispositivos). Inclui Controllers, Presenters, Gateways. |
| **Dependency Rule** | Regra fundamental da Clean Architecture: dependências de código-fonte devem apontar apenas para dentro — camadas externas dependem de internas, nunca o contrário. |
| **Boundary** | Linha de separação entre componentes com diferentes níveis de política. Pode ser source-level, deployment-level ou service-level. |
| **Policy** | As regras de negócio do sistema; o "o quê" e "por quê" — componentes de alto nível. |
| **Detail** | Mecanismos de implementação: banco de dados, web framework, UI, dispositivos — componentes de baixo nível. |
| **Humble Object** | Padrão que separa comportamento difícil de testar (I/O) em um objeto "humilde" com lógica mínima, mantendo a lógica real em um objeto testável. |
| **Screaming Architecture** | Princípio de que a estrutura do projeto deve comunicar o domínio do negócio, não o framework ou tecnologia utilizada. |
| **Interactor** | Sinônimo de Use Case. Classe que contém a lógica de aplicação para um caso de uso específico. |
| **Input Port** | Interface que define o contrato de entrada de um Use Case. Declarada na camada de Use Cases. |
| **Output Port** | Interface que define o contrato de saída que um Use Case precisa (ex: Repository, Gateway). Declarada na camada de Use Cases, implementada na camada de adapters. |
| **Input Boundary** | Sinônimo de Input Port. |
| **Output Boundary** | Sinônimo de Output Port. |
| **Presenter** | Componente da camada de adapters que converte o output do Use Case em um formato adequado para a View. |
| **ViewModel** | Estrutura de dados preparada pelo Presenter, contendo dados formatados prontos para renderização, sem lógica. |
| **Gateway** | Componente da camada de adapters que implementa uma Output Port, fornecendo acesso a recursos externos (banco, API, filesystem). |
| **DTO (Data Transfer Object)** | Estrutura de dados simples (sem comportamento) usada para transferir dados entre camadas/boundaries. |
| **Composition Root** | Ponto único do sistema (geralmente `main` ou app factory) onde o grafo de dependências é montado, conectando interfaces a implementações concretas. |
| **Main Sequence** | Linha ideal no gráfico I x A onde componentes estão em equilíbrio entre estabilidade e abstração. D = |A + I - 1| deve ser próximo de 0. |
| **Zone of Pain** | Região do gráfico I x A com I baixo e A baixo. Componente estável e concreto — difícil de mudar e não extensível. |
| **Zone of Uselessness** | Região do gráfico I x A com I alto e A alto. Componente instável e abstrato — provavelmente código morto. |
| **Instability (I)** | Métrica: Fan-out / (Fan-in + Fan-out). Mede a vulnerabilidade de um componente a mudanças. 0 = máxima estabilidade, 1 = máxima instabilidade. |
| **Abstractness (A)** | Métrica: classes abstratas e interfaces / total de classes no componente. 0 = totalmente concreto, 1 = totalmente abstrato. |
| **Distance (D)** | Métrica: \|A + I - 1\|. Distância perpendicular à Main Sequence. Quanto menor, melhor o equilíbrio. |
| **Fan-in** | Número de classes externas que dependem de classes dentro do componente. Indica responsabilidade (quantos dependem de mim). |
| **Fan-out** | Número de classes externas das quais o componente depende. Indica dependência (de quantos eu dependo). |
| **REP (Reuse/Release Equivalence Principle)** | Princípio de coesão: a granularidade de reutilização é a granularidade de release. Classes no mesmo componente devem ser lançáveis juntas. |
| **CCP (Common Closure Principle)** | Princípio de coesão: agrupe classes que mudam juntas. SRP para componentes. |
| **CRP (Common Reuse Principle)** | Princípio de coesão: não force usuários a depender de coisas que não usam. ISP para componentes. |
| **ADP (Acyclic Dependencies Principle)** | Princípio de acoplamento: o grafo de dependências entre componentes não deve conter ciclos. |
| **SDP (Stable Dependencies Principle)** | Princípio de acoplamento: dependa na direção da estabilidade. |
| **SAP (Stable Abstractions Principle)** | Princípio de acoplamento: componentes estáveis devem ser abstratos; componentes instáveis devem ser concretos. |
| **Clean Architecture** | Arquitetura proposta por Robert C. Martin baseada em camadas concêntricas com a Dependency Rule, onde o domínio está no centro e detalhes na periferia. |
| **Hexagonal Architecture** | Arquitetura de Alistair Cockburn com conceito similar: interior (domínio) conectado ao exterior (infraestrutura) via Ports e Adapters. Fortemente alinhada com Clean Architecture. |
| **Ports and Adapters** | Sinônimo de Hexagonal Architecture. Ports são interfaces; Adapters são implementações. |
| **Package by Component** | Abordagem de Simon Brown para organizar código empacotando juntos domínio, ports e adapters de um componente, expondo apenas interface pública. |
| **Monolith Modular** | Aplicação deployada como artefato único mas com boundaries source-level bem definidos entre componentes. Frequentemente o melhor ponto de partida antes de microserviços. |
