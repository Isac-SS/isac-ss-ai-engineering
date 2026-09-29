# Skill: Gestão de Complexidade de Código — John Ousterhout ("A Philosophy of Software Design")

## Perspectiva do Autor

John Ousterhout parte de uma premissa fundamental: **"A maior limitação na escrita de
software é a nossa capacidade de entender os sistemas que estamos criando."** Não é
velocidade de CPU, não é memória disponível, não é a linguagem de programação — é a
capacidade cognitiva humana de manter um modelo mental do sistema.

Complexidade é **O inimigo**. Cada decisão de design deve ser avaliada pelo critério
mais importante: **esta decisão aumenta ou diminui a complexidade para quem vai ler e
manter este código?** Não para quem está escrevendo agora, mas para o próximo
desenvolvedor que vai precisar entender e modificar.

A visão de Ousterhout é profundamente pragmática e orientada ao custo cognitivo. Ele
não busca elegância estética nem pureza teórica — busca sistemas que possam ser
compreendidos e modificados por humanos reais com capacidade cognitiva limitada.

O autor é crítico da cultura de "clean code" superficial que foca em regras mecânicas
(métodos pequenos, classes pequenas) sem considerar o custo cognitivo real. Um método
de 5 linhas que obriga o leitor a navegar entre 15 classes para entender o fluxo é
MAIS complexo que um método de 40 linhas autocontido — mesmo que o primeiro "pareça"
mais limpo.

Prioridades do autor, em ordem:
1. **Reduzir complexidade** — o objetivo supremo de todo design
2. **Módulos profundos** — interfaces simples que escondem implementações ricas
3. **Information hiding** — encapsular decisões de design dentro de módulos
4. **Código óbvio** — o leitor entende sem esforço significativo
5. **Investimento estratégico** — gastar 10-20% mais agora para reduzir custo futuro

---

## Princípios Fundamentais

### 1. A Natureza da Complexidade

Complexidade é qualquer coisa que torna software difícil de entender ou modificar.
Não é uma propriedade absoluta — é relativa ao observador e ao contexto.

**Três manifestações da complexidade:**

#### 1.1 Change Amplification (Amplificação de Mudanças)
Uma mudança aparentemente simples requer modificações em muitos lugares diferentes.
Exemplo: mudar o formato de uma data requer alterações em 15 arquivos porque o formato
está hardcoded em cada um. Se uma decisão de design requer N alterações em N arquivos,
a complexidade é multiplicada por N.

#### 1.2 Cognitive Load (Carga Cognitiva)
Quantidade de conhecimento que um desenvolvedor precisa ter para completar uma tarefa.
Não é apenas "quantidade de código" — APIs complexas, convenções inconsistentes,
dependências entre módulos, configuração implícita — tudo contribui para cognitive load.

**Mais linhas de código com lógica simples pode ser MENOS complexo do que menos linhas
com lógica densa.** O critério não é tamanho — é quanto o leitor precisa saber para
entender.

#### 1.3 Unknown Unknowns (Desconhecidos Desconhecidos)
**A pior manifestação.** O desenvolvedor não sabe o que não sabe. Não sabe que precisa
considerar um caso especial, que existe uma dependência oculta, que uma mudança aqui
quebra algo ali.

- Change amplification é irritante mas gerenciável — você descobre ao testar
- Cognitive load é pesado mas contornável — você estuda e entende
- Unknown unknowns são perigosos — **você nem sabe que precisa investigar**

**Complexidade é incremental — morte por mil cortes.** Nenhuma decisão individual parece
criar complexidade significativa. Mas cada pequena decisão de "isso é bom o suficiente"
se acumula. Depois de centenas dessas decisões, o sistema se torna impenetrável.
Reverter é exponencialmente mais difícil do que prevenir.

### 2. Programação Estratégica vs. Tática

#### 2.1 Programação Tática
Mentalidade: "Fazer funcionar AGORA. Depois eu limpo."

Características:
- Foco exclusivo na tarefa imediata
- Atalhos são aceitáveis desde que funcione
- "Eu limpo depois" — mas depois nunca chega
- Design emerge acidentalmente
- Cada feature adiciona um pouco de complexidade
- O sistema decai progressivamente

#### 2.2 Programação Estratégica
Mentalidade: "Investir 10-20% mais de tempo agora para criar design limpo."

Características:
- O objetivo primário é produzir um ótimo design que TAMBÉM funciona
- Refactoring proativo — não esperar o código apodrecer
- Consideração de alternativas de design antes de implementar
- O investimento se paga em semanas ou meses, não anos
- Cada feature deixa o sistema um pouco melhor

#### 2.3 O Tornado Tático
O desenvolvedor que produz volume impressionante de código mas deixa um rastro
de complexidade. É perigoso porque parece produtivo no curto prazo. O custo da
complexidade criada só aparece quando outros desenvolvimentos precisam trabalhar
no código — e a conta vem multiplicada.

**Empresas que investem em qualidade de design (Google, VMware) consistentemente
superam empresas que priorizam velocidade tática** — porque o custo da complexidade
acumulada eventualmente supera qualquer ganho de velocidade inicial.

### 3. Deep Modules vs. Shallow Modules

O conceito mais importante e original do livro.

**Módulo = Interface + Implementação**

A interface é o custo cognitivo que o módulo impõe a todos os seus usuários.
A implementação é o valor que o módulo fornece.

#### 3.1 Deep Modules (Módulos Profundos)
Interface simples, implementação rica. O módulo esconde enorme complexidade atrás
de uma interface minúscula.

**Exemplo canônico: I/O de arquivos Unix**
- Interface: `open()`, `read()`, `write()`, `close()`, `lseek()`
- 5 chamadas que escondem: file systems, buffers, caching, permissões, locks,
  journaling, block allocation, disk scheduling, device drivers
- O usuário não precisa saber NADA da implementação para usar efetivamente

**Exemplo moderno: garbage collector**
- Interface: nenhuma (automático)
- Implementação: algoritmos complexos de marking, sweeping, compaction,
  generational collection
- Profundidade máxima — complexidade enorme com zero custo para o usuário

#### 3.2 Shallow Modules (Módulos Rasos)
Interface quase tão complexa quanto a implementação. O módulo adiciona complexidade
(custo da interface) sem esconder complexidade proporcionalmente.

```
// Exemplo de módulo raso — a interface é tão complexa quanto a implementação
class FileAppender {
    void appendLine(String filePath, String line, String encoding,
                    boolean createIfNotExists, boolean appendNewline) {
        // Implementação: 5 linhas de código
    }
}
```

O módulo acima tem 5 parâmetros na interface para ~5 linhas de implementação.
O custo cognitivo para o usuário é quase igual ao custo de implementar diretamente.

#### 3.3 Classitis

Doença de excesso de classes/módulos pequenos. Cada módulo, por mais simples que seja,
tem custo cognitivo:
- O leitor precisa saber que existe
- O leitor precisa entender sua interface
- O leitor precisa navegar até ele
- O leitor precisa entender como se relaciona com outros módulos

**20 classes de 10 linhas cada são mais complexas que 2 classes de 100 linhas** se
as 20 classes forçam o leitor a navegar entre elas para entender um único fluxo.

O critério não é tamanho da classe — é profundidade. Uma classe grande que encapsula
muita complexidade atrás de uma interface simples é excelente design.

### 4. Information Hiding e Information Leakage

#### 4.1 Information Hiding (Ocultação de Informação)
Cada módulo encapsula uma decisão de design. Outros módulos não precisam (e não
devem) conhecer os detalhes dessa decisão.

Exemplos de informação que deve ser escondida:
- Formato de serialização de dados
- Estrutura de tabelas do banco de dados
- Algoritmo de implementação
- Protocolo de comunicação com serviço externo
- Mecanismo de caching
- Estratégia de retry e resiliência

Se a informação está escondida, ela pode ser mudada sem afetar outros módulos.

#### 4.2 Information Leakage (Vazamento de Informação)
Uma decisão de design é exposta em múltiplos módulos. Se a decisão muda,
múltiplos módulos precisam mudar — **change amplification**.

**Formas comuns de information leakage:**
- Formato de arquivo/protocolo conhecido por múltiplas classes
- Estrutura de banco de dados exposta em múltiplas camadas
- Constantes repetidas em múltiplos arquivos
- Assunções sobre ordem de operações espalhadas pelo código

**Exemplo clássico:** conhecimento do formato HTTP espalhado entre o módulo que
cria requests e o módulo que parsea responses. Se ambos conhecem os headers,
status codes, content-type — há leakage. A solução: **colocar ambos no mesmo
módulo** para que o conhecimento fique encapsulado.

#### 4.3 Temporal Decomposition (Decomposição Temporal)
Um dos erros de design mais comuns: dividir módulos pela **ordem de execução**
em vez de pela **informação que encapsulam**.

Exemplo: sistema que lê arquivo, processa dados e escreve resultado.
Decomposição temporal cria 3 módulos: FileReader, DataProcessor, FileWriter.
Mas FileReader e FileWriter ambos conhecem o formato do arquivo — **information
leakage**.

Melhor: um módulo FileFormat que encapsula leitura E escrita (porque ambos
dependem da mesma informação), e um módulo DataProcessor separado.

### 5. General-Purpose vs. Special-Purpose

O ideal é **"somewhat general-purpose"** — genérico o suficiente para usos
atuais e prováveis futuros, sem especular sobre necessidades hipotéticas.

**Perguntas para calibrar o nível de generalidade:**
- "Qual é a interface mais simples que cobre todas as minhas necessidades atuais?"
- "Em quantas situações este método vai ser usado?"
- "Esta API é fácil de usar para minhas necessidades atuais?"

Módulos general-purpose tendem a ser **mais profundos** — interface mais simples
(porque não tem especificidades de um caso) com implementação mais rica.

Módulos excessivamente especializados tendem a ser **mais rasos** — interface
cheia de parâmetros específicos do caso de uso atual.

**Não projete para especulação, mas não se encurrale.**

### 6. Different Layer, Different Abstraction

Cada camada do sistema deve fornecer uma abstração diferente das camadas
adjacentes. Se duas camadas oferecem abstrações semelhantes, provavelmente
há um problema de design.

#### 6.1 Pass-through Methods
Método que não faz nada além de chamar outro método com a mesma (ou muito
similar) assinatura.

```
// Red flag: pass-through method
class OrderService {
    void createOrder(Order order) {
        orderRepository.createOrder(order);  // Apenas delega
    }
}
```

Se o método não adiciona valor (validação, transformação, orquestração),
ele é apenas noise — adiciona uma camada de indireção sem adicionar
abstração.

#### 6.2 Pass-through Variables
Variáveis passadas através de múltiplas camadas para chegar ao seu destino.

```
// Red flag: pass-through variable
void handleRequest(Request req, String tenantId) {
    processOrder(req, tenantId);
}
void processOrder(Request req, String tenantId) {
    saveToDatabase(req, tenantId);
}
void saveToDatabase(Request req, String tenantId) {
    // Finalmente usa tenantId
}
```

`tenantId` atravessa duas camadas que não se importam com ele. Soluções:
context objects, thread-local storage, dependency injection.

#### 6.3 Decorators
Decorators frequentemente violam o princípio "different layer, different
abstraction" porque oferecem exatamente a mesma interface da classe decorada.
São inerentemente shallow — adicionam uma camada sem adicionar profundidade.

Use decorators com cautela e apenas quando a funcionalidade adicionada é
genuinamente transversal (logging, caching, retry) e não pode ser absorvida
pelo módulo original.

### 7. Pull Complexity Downward

Dada a escolha entre tornar algo simples para o **usuário** de um módulo ou
para o **desenvolvedor** do módulo, **simplifique para o usuário**.

O desenvolvedor do módulo paga o custo da complexidade **uma vez**.
Cada usuário do módulo paga o custo **todas as vezes** que interage com ele.
Se o módulo tem 50 usuários, o custo multiplicado é 50x.

**Configuration parameters são um exemplo clássico de pushing complexity
upward** — em vez de o módulo tomar uma decisão razoável, ele delega a
decisão para o usuário via parâmetro de configuração. Cada parâmetro de
configuração é complexidade transferida do autor para o usuário.

Não significa que configuration parameters são sempre ruins — mas cada um
deve ser justificado. Se 95% dos usuários vão usar o mesmo valor, faça
esse valor o default e permita override apenas quando necessário.

### 8. Define Errors Out of Existence

Exception handling é uma fonte enorme de complexidade. Cada exceção que um
módulo pode lançar:
- Adiciona um path que o chamador precisa considerar
- Pode não ser adequadamente tratada (catch vazio, log e ignora)
- Pode se propagar por múltiplas camadas sem tratamento adequado
- Torna o fluxo principal mais difícil de seguir

**A melhor forma de lidar com exceções é projetar o design para eliminá-las.**

**Exemplo: deletar arquivo inexistente**
- Windows: lança erro — o chamador precisa verificar existência antes de deletar
  ou tratar a exceção. Dois paths de código.
- Unix `unlink()`: se o arquivo não existe, não faz nada — não é erro.
  O chamador chama `unlink()` e segue. Um path de código.

A abordagem Unix é superior porque **o caso normal já trata o caso especial**.

**Princípios:**
- Projete APIs onde o "caso normal" absorve os edge cases naturalmente
- Mascare exceções em níveis baixos — só propague exceções que o chamador
  pode tratar de forma significativa
- Se o chamador não pode fazer nada útil com a exceção, não lance
- Reduza a superfície de exceções na interface pública

### 9. Design It Twice

Para qualquer decisão de design importante, considere **pelo menos duas
alternativas** antes de implementar.

Mesmo que uma abordagem pareça óbvia, o exercício de considerar alternativas:
- Revela trade-offs que não eram aparentes
- Pode produzir uma solução melhor
- Aumenta a confiança na decisão escolhida
- Exercita o músculo de design

**"A coisa mais importante é resistir à tentação de se agarrar à primeira ideia."**

Compare as alternativas em termos de:
- Simplicidade da interface
- Profundidade do módulo (informação escondida vs. exposta)
- Flexibilidade para mudanças futuras prováveis
- Impacto na complexidade do sistema como um todo

Não confunda "Design It Twice" com analysis paralysis. O objetivo não é
considerar 10 alternativas durante semanas — é investir 15-30 minutos
considerando 2-3 alternativas antes de cada decisão significativa.

### 10. Comentários como Ferramenta de Design

#### 10.1 Bons Comentários
Descrevem o que **NÃO é óbvio** a partir do código:
- **WHY** — a razão por trás de uma decisão, o racional que motivou o design
- **WHAT** em alto nível — documentação de interface, o que o módulo faz
  (não como faz)
- **Caveats** — edge cases, limitações, pré-condições não óbvias
- **Invariantes** — condições que devem ser mantidas

#### 10.2 Maus Comentários
Descrevem o que o código JÁ diz:
- Linha por linha descrevendo o que cada instrução faz
- Repetição do nome do método em prosa
- Informação que pode ser inferida do código

#### 10.3 Comentários como Ferramenta de Design
**Escreva comentários ANTES do código.**

Se o comentário da interface é difícil de escrever, provavelmente o design está
errado. A dificuldade de descrever o módulo em palavras simples revela que o
módulo não tem uma abstração clara.

Teste: se o comentário da interface requer muitas cláusulas condicionais
("faz X, exceto quando Y, a menos que Z"), a interface é muito complexa.

### 11. Consistência

Consistência reduz cognitive load drasticamente. Quando o leitor aprende um
padrão, ele o reconhece em todo o codebase sem custo adicional.

Áreas onde consistência importa:
- **Naming conventions** — mesmas coisas com mesmos nomes
- **Coding style** — indentação, formatação, organização de arquivo
- **Design patterns** — mesmos problemas resolvidos da mesma forma
- **Invariantes** — condições que sempre são verdadeiras
- **Interfaces** — métodos similares com assinaturas similares

**"Se um sistema não é consistente, mesmo as partes que são bem projetadas
serão mais difíceis de entender."**

Inconsistência força o leitor a tratar cada parte do código como nova,
mesmo que seja conceitualmente similar a algo que já viu.

### 12. Código Deve Ser Óbvio

O teste final: **um desenvolvedor competente consegue entender rapidamente
o que este código faz?**

Se não consegue, o código precisa ser redesenhado — não apenas comentado.
Comentários são um band-aid para código obscuro; o tratamento real é tornar
o código óbvio.

**Fontes comuns de obscuridade:**
- **Event-driven programming** — difícil seguir o fluxo quando handlers são
  registrados em um lugar e invocados em outro
- **Generic containers** — `Pair<String, Integer>` não diz nada sobre o
  significado. Use named types: `record OrderCount(String productId, int count)`
- **Code that does more than expected** — efeitos colaterais não óbvios
- **Inconsistency** — padrões diferentes para problemas similares
- **Missing context** — informação necessária para entender está em outro arquivo

---

## Heurísticas Verificáveis

### Profundidade de Módulos

1. **A interface pública de cada módulo/classe é significativamente menor que sua implementação.**
   *Como verificar:* contar métodos públicos vs. linhas de implementação. Se um
   módulo tem 15 métodos públicos e 50 linhas de implementação, é raso. Se tem
   3 métodos públicos e 300 linhas de implementação, é profundo.

2. **Métodos públicos não expõem detalhes de implementação na assinatura.**
   *Como verificar:* examinar tipos de parâmetros e retorno dos métodos públicos.
   Se expõem tipos internos (DTOs internos, estruturas de banco, objetos de
   framework), há leakage.

3. **Classes "orquestradoras" não são apenas listas de delegações.**
   *Como verificar:* se um método em uma service class contém apenas uma sequência
   de chamadas para outros objetos sem lógica própria (validação, transformação,
   decisão), questionar se a orquestração agrega valor ou é apenas indireção.

4. **O número de módulos é proporcional à complexidade do domínio, não ao número de operações.**
   *Como verificar:* contar módulos/classes. Se um CRUD simples tem 15 classes
   (Controller, Service, ServiceImpl, Repository, RepositoryImpl, Mapper, DTO,
   Entity, Validator, Exception, etc.), há classitis.

5. **Nenhum módulo existe apenas para satisfazer uma regra de arquitetura.**
   *Como verificar:* para cada classe, perguntar "se eu inlinasse este código no
   chamador, o que eu perderia?". Se a resposta é "nada", o módulo é desnecessário.

### Information Hiding

6. **Decisões de formato de dados (JSON, XML, CSV, Protobuf) estão encapsuladas em um único módulo.**
   *Como verificar:* buscar imports de bibliotecas de serialização (Jackson,
   Gson, etc.). Se aparecem em mais de um package/módulo, há information leakage.

7. **Schema de banco de dados é conhecido apenas pelo módulo de persistência.**
   *Como verificar:* buscar referências a nomes de tabelas e colunas. Se aparecem
   fora do repository/DAO layer, o schema vazou.

8. **Detalhes de integração com serviços externos estão encapsulados em um módulo.**
   *Como verificar:* buscar URLs, headers, formatos de request/response de APIs
   externas. Se aparecem fora de um módulo dedicado de integração, há leakage.

9. **Constantes que representam a mesma decisão de design não estão duplicadas.**
   *Como verificar:* buscar strings/números mágicos duplicados que representam
   a mesma coisa (formato de data, nome de header, tamanho de página default).

10. **Enums ou tipos que representam estados internos não são expostos na API pública.**
    *Como verificar:* examinar tipos nos DTOs/responses. Se enums internos de
    estado (PROCESSING, RETRY_PENDING) aparecem na API, a decisão de design vazou.

### Pass-through e Layers

11. **Não existem pass-through methods que apenas delegam sem agregar valor.**
    *Como verificar:* buscar métodos cuja implementação inteira é uma única chamada
    a outro método com os mesmos (ou subconjunto dos) parâmetros.

12. **Cada camada do sistema fornece uma abstração diferente da camada adjacente.**
    *Como verificar:* comparar as assinaturas de métodos entre camadas (Controller
    vs. Service vs. Repository). Se são praticamente idênticas, as camadas não
    estão agregando abstração.

13. **Variáveis não são passadas através de múltiplas camadas sem uso.**
    *Como verificar:* buscar parâmetros que atravessam 3+ métodos antes de serem
    usados. São pass-through variables que poluem assinaturas.

14. **Decorators são usados apenas para concerns genuinamente transversais.**
    *Como verificar:* para cada decorator/wrapper, verificar se a funcionalidade
    adicionada (logging, caching, metrics) não poderia ser absorvida pelo módulo
    original sem complicá-lo significativamente.

### Exception Handling

15. **Exceções na interface pública são apenas aquelas que o chamador pode tratar de forma significativa.**
    *Como verificar:* para cada exceção declarada em método público, perguntar
    "o que o chamador faz com esta exceção?". Se a resposta é "log e rethrow"
    ou "catch vazio", a exceção não deveria existir na interface.

16. **APIs são projetadas para que o caso normal absorva edge cases.**
    *Como verificar:* buscar patterns onde o chamador precisa fazer check antes de
    chamar (`if (exists()) then delete()` vs. `deleteIfExists()`). O segundo
    elimina uma classe de erros.

17. **Exceções de baixo nível são mascaradas e traduzidas, não propagadas diretamente.**
    *Como verificar:* verificar se `SQLException`, `IOException`, `HttpException`
    vazam para camadas superiores. Devem ser capturadas e traduzidas em exceções
    de domínio ou tratadas internamente.

### Cognitive Load e Obviousness

18. **Tipos genéricos (`Pair`, `Tuple`, `Map<String, Object>`) não são usados como tipos de retorno em interfaces públicas.**
    *Como verificar:* buscar retornos de `Pair`, `Tuple`, `Triple`,
    `Map<String, Object>`, `Object[]` em métodos públicos. Cada um é uma
    abstração perdida — substitua por named types.

19. **Nenhum método tem mais de 3-4 parâmetros sem agrupamento semântico.**
    *Como verificar:* buscar métodos com 5+ parâmetros. Parâmetros relacionados
    devem ser agrupados em objetos (parameter objects) com nomes significativos.

20. **Efeitos colaterais são explícitos no nome do método ou na documentação.**
    *Como verificar:* verificar se métodos que modificam estado, enviam
    notificações, ou fazem I/O deixam isso claro. `calculateTotal()` que
    também salva no banco é um efeito colateral oculto.

21. **Fluxo de controle event-driven tem documentação de quais eventos disparam quais handlers.**
    *Como verificar:* em sistemas event-driven, verificar se existe documentação
    (ou código óbvio) que mapeia evento -> handler. Se o leitor precisa buscar
    em todo o codebase para encontrar quem trata um evento, há obscuridade.

### Consistência

22. **Naming conventions são consistentes em todo o codebase.**
    *Como verificar:* buscar inconsistências como `getUserById` vs. `fetchUser`
    vs. `findUserByIdentifier` para operações equivalentes. Mesma operação =
    mesmo padrão de nome.

23. **Problemas similares são resolvidos com patterns similares.**
    *Como verificar:* comparar como diferentes módulos lidam com o mesmo tipo
    de problema (error handling, validação, persistência). Se cada módulo faz
    de forma diferente, há inconsistência.

24. **Imports e dependências seguem direção consistente (sem dependências circulares).**
    *Como verificar:* analisar o grafo de dependências. Dependências circulares
    entre módulos indicam abstrações mal definidas.

### Investimento Estratégico

25. **Refactoring é feito proativamente, não apenas quando "não tem mais jeito".**
    *Como verificar:* verificar histórico de commits. Se há commits de refactoring
    regulares (misturados com features), a equipe programa estrategicamente. Se
    refactoring só aparece em "sprints de dívida técnica", é tático.

26. **Cada PR reduz ou mantém a complexidade — nunca aumenta sem justificativa.**
    *Como verificar:* em code review, perguntar "este PR adiciona complexidade?
    Se sim, a complexidade é justificada pela funcionalidade entregue?"

27. **Antes de implementar, pelo menos duas alternativas de design foram consideradas.**
    *Como verificar:* verificar se PRs de features significativas mencionam
    alternativas consideradas. Se o design sempre é "a primeira ideia que
    funcionou", não há prática de Design It Twice.

### Comentários e Documentação

28. **Interfaces públicas têm comentários que descrevem WHAT e WHY, não HOW.**
    *Como verificar:* examinar comentários de classes e métodos públicos. Devem
    explicar o que o módulo faz e por que decisões foram tomadas — não como a
    implementação funciona.

29. **Comentários não repetem o que o código já diz.**
    *Como verificar:* para cada comentário inline, perguntar "este comentário
    me diz algo que eu não poderia inferir lendo o código?". Se não, é noise.

30. **Ausência de comentários em código não-óbvio é tratada como bug.**
    *Como verificar:* código com lógica complexa (bitwise operations, algoritmos
    otimizados, workarounds) deve ter comentário explicando o WHY. Ausência
    não é "código auto-documentado" — é código que será mal interpretado.

---

## Anti-patterns com Sinais no Código

| # | Anti-pattern | Sinal no Código | Severidade | Ação |
|---|-------------|-----------------|------------|------|
| 1 | **Shallow Module** | Classe com 10 métodos públicos e 30 linhas de implementação total | ALTA | Repensar o módulo: absorver em outro ou tornar mais profundo |
| 2 | **Classitis** | CRUD simples com 12+ classes (Controller, Service, ServiceImpl, Mapper, DTO, Entity, Validator...) | ALTA | Eliminar camadas que não agregam abstração distinta |
| 3 | **Information Leakage** | Formato de serialização (JSON annotations) em classes de múltiplos packages | ALTA | Centralizar conhecimento de formato em um módulo |
| 4 | **Temporal Decomposition** | Módulos nomeados por fase (Reader, Processor, Writer) que compartilham formato | ALTA | Reorganizar por informação encapsulada, não por ordem de execução |
| 5 | **Pass-through Method** | Método cuja implementação é uma chamada com mesma assinatura | MÉDIA | Eliminar o método ou adicionar valor (validação, transformação) |
| 6 | **Pass-through Variable** | Parâmetro passado por 3+ métodos antes de uso | MÉDIA | Context object, dependency injection, ou thread-local |
| 7 | **Tactical Programming** | PRs com "TODO: refactor later", workarounds, hacks sem justificativa | ALTA | Investir tempo no design correto agora; refusar PR tático |
| 8 | **Tactical Tornado** | Desenvolvedor com alto volume de commits mas crescente complexidade no módulo | ALTA | Code review mais rigoroso; métricas de complexidade |
| 9 | **Exceções desnecessárias** | `FileNotFoundException` quando o contrato permite arquivo ausente | MÉDIA | Redesenhar API para absorver o edge case (define errors out of existence) |
| 10 | **Generic Container como retorno** | `Map<String, Object>` ou `Pair<X, Y>` como tipo de retorno público | MÉDIA | Criar named type que documenta a semântica |
| 11 | **Configuration Parameter excessivo** | Módulo com 15+ parâmetros de configuração, maioria nunca alterada | MÉDIA | Defaults razoáveis internos; expor apenas o necessário |
| 12 | **Inconsistência de naming** | `getUser`, `fetchOrder`, `retrieveProduct` para mesma operação | MÉDIA | Padronizar naming conventions e documentar |
| 13 | **Comentário que repete código** | `// incrementa o contador` acima de `counter++` | BAIXA | Remover comentários redundantes; adicionar WHY onde falta |
| 14 | **Obscuridade por event dispatch** | Handler registrado em arquivo A, disparado em arquivo B, sem documentação do vínculo | ALTA | Documentar mapa de eventos ou tornar o binding explícito |
| 15 | **Layer sem abstração própria** | Controller, Service e Repository com assinaturas quase idênticas | MÉDIA | Questionar se todas as camadas são necessárias para este caso |
| 16 | **Efeito colateral oculto** | Método `calculate()` que também persiste, envia email ou modifica estado global | ALTA | Separar side effects ou torná-los explícitos no nome |
| 17 | **Dependência circular** | Módulo A importa B, B importa A | ALTA | Extrair interface ou reorganizar responsabilidades |
| 18 | **Complexity pushed upward** | Chamador precisa fazer 5 chamadas em sequência específica para usar o módulo | ALTA | Pull complexity downward — simplificar a interface para o chamador |

---

## Checklist de Revisão

### Para cada Pull Request, avalie:

**Profundidade do Módulo**
- [ ] As classes/módulos novos ou modificados são profundos (interface simples, implementação rica)?
- [ ] Existe justificativa para cada nova classe criada além de "separação de responsabilidades"?
- [ ] O leitor consegue entender a funcionalidade sem navegar entre mais de 2-3 arquivos?
- [ ] Algum módulo novo é shallow (interface comparável à implementação)?

**Information Hiding**
- [ ] Decisões de design estão encapsuladas em módulos únicos?
- [ ] Detalhes de formato, protocolo ou schema estão vazando entre módulos?
- [ ] Mudança nesta feature exigiria alterar múltiplos módulos por causa do mesmo motivo?

**Abstrações e Camadas**
- [ ] Cada camada fornece abstração diferente das adjacentes?
- [ ] Existem pass-through methods sem valor agregado?
- [ ] Existem pass-through variables atravessando múltiplas camadas?
- [ ] Decorators são realmente necessários ou a funcionalidade pode ser absorvida?

**Complexidade e Cognitive Load**
- [ ] Um desenvolvedor novo entenderia este código em leitura linear?
- [ ] Tipos genéricos foram substituídos por named types onde apropriado?
- [ ] Parâmetros excessivos foram agrupados em objetos semânticos?
- [ ] Efeitos colaterais são explícitos?
- [ ] Event handlers são rastreáveis?

**Exception Handling**
- [ ] Exceções na interface pública são tratáveis pelo chamador?
- [ ] APIs foram projetadas para absorver edge cases naturalmente?
- [ ] Exceções de baixo nível são traduzidas antes de propagar?

**Consistência**
- [ ] Naming segue as convenções existentes do codebase?
- [ ] Problemas similares são resolvidos de forma similar?
- [ ] Novos patterns são justificados ou representam inconsistência?

**Comentários**
- [ ] Interfaces públicas têm documentação de WHAT e WHY?
- [ ] Código não-óbvio tem comentário explicando WHY?
- [ ] Comentários redundantes (que repetem o código) foram removidos?

**Investimento Estratégico**
- [ ] O design foi uma escolha deliberada ou a primeira coisa que funcionou?
- [ ] Alternativas foram consideradas (Design It Twice)?
- [ ] A complexidade total do sistema aumentou? Se sim, é justificável?
- [ ] Existe refactoring oportunístico incluído (Boy Scout Rule)?

---

## Perguntas que esta Skill Faz

1. **Este módulo é profundo ou raso?**
   A interface é significativamente mais simples que a implementação?
   Ou o usuário precisa saber quase tanto quanto o implementador?

2. **Se eu inlinasse este código no chamador, o que eu perderia?**
   Se a resposta é "nada" — o módulo é desnecessário. Se a resposta é
   "encapsulamento de uma decisão complexa" — o módulo é valioso.

3. **Quantos arquivos um desenvolvedor novo precisa abrir para entender
   este fluxo?**
   Se são mais de 4-5, a complexidade de navegação está muito alta.

4. **Qual informação está vazando entre esses módulos?**
   Se mudança X requer alteração nos módulos A, B e C pelo mesmo motivo,
   há information leakage. A informação deveria estar em um lugar só.

5. **Este módulo existe porque agrega abstração ou porque "a arquitetura
   exige"?**
   Camadas que existem para satisfazer uma regra sem agregar valor são
   complexidade pura.

6. **O que o chamador precisa saber para usar esta interface?**
   Quanto mais o chamador precisa saber, mais complexa é a interface.
   O ideal é "quase nada".

7. **Este comentário me diz algo que o código não diz?**
   Se não, é noise. Se sim, por que o código não pode dizer por si só?
   Se não pode, o comentário é essencial.

8. **Estou fazendo programação tática ou estratégica neste PR?**
   Estou investindo no design ou estou cortando caminho para entregar mais
   rápido? Qual o custo desse atalho multiplicado pelo número de vezes que
   alguém vai ler este código?

9. **Quais são os unknown unknowns neste design?**
   O que alguém que não participou do design vai errar ao modificar este
   código? Onde estão as armadilhas ocultas?

10. **Esta exceção pode ser eliminada por redesign da API?**
    Em vez de lançar e tratar exceção, posso projetar a interface para que
    o caso normal absorva o edge case?

11. **Este é um pass-through method? Que valor ele agrega?**
    Se a resposta é "organização" mas não há transformação, validação ou
    decisão, questione sua existência.

12. **Se eu adicionasse mais uma feature neste módulo, ficaria mais fácil
    ou mais difícil de entender?**
    Módulos com abstração clara ficam mais fáceis com features relacionadas.
    Módulos sem abstração clara ficam mais difíceis com qualquer adição.

13. **A complexidade está empurrada para baixo (developer do módulo) ou
    para cima (usuário do módulo)?**
    Se o chamador precisa fazer uma dança de 5 passos para usar o módulo,
    a complexidade está no lugar errado.

14. **Considerei pelo menos duas alternativas de design?**
    Qual é o trade-off entre elas? Por que esta foi escolhida?
    Ou estou implementando "a primeira coisa que pensei"?

15. **Este código é óbvio para alguém que não o escreveu?**
    O teste de obscuridade: um desenvolvedor competente entende em 30
    segundos? Se não, precisa de redesign (não apenas comentários).

16. **A consistência do codebase está sendo mantida ou erodida?**
    Estou introduzindo um novo padrão quando um existente serviria?
    Se sim, a inconsistência vale a pena?

17. **Qual é o custo cognitivo total desta mudança para o próximo leitor?**
    Quantos conceitos novos, arquivos novos, patterns novos estou
    introduzindo? O benefício justifica esse custo?

---

## Vocabulário

| Termo | Definição |
|-------|-----------|
| **Complexity** | Qualquer coisa que torna software difícil de entender ou modificar — medida pela perspectiva do leitor/mantenedor |
| **Change Amplification** | Situação onde uma mudança conceitualmente simples requer alterações em muitos lugares do código |
| **Cognitive Load** | Quantidade de conhecimento que um desenvolvedor precisa ter para completar uma tarefa em um módulo |
| **Unknown Unknowns** | Informações que o desenvolvedor precisa mas não sabe que precisa — a manifestação mais perigosa de complexidade |
| **Deep Module** | Módulo com interface simples e implementação rica — esconde muita complexidade atrás de pouca superfície |
| **Shallow Module** | Módulo cuja interface é quase tão complexa quanto sua implementação — agrega pouco valor de encapsulamento |
| **Classitis** | Tendência de criar excesso de classes/módulos pequenos, cada um adicionando custo cognitivo sem profundidade proporcional |
| **Information Hiding** | Encapsulamento de decisão de design dentro de um módulo, tornando outros módulos independentes dessa decisão |
| **Information Leakage** | Exposição de decisão de design em múltiplos módulos — se a decisão muda, múltiplos módulos precisam mudar |
| **Temporal Decomposition** | Erro de dividir módulos pela ordem de execução em vez de pela informação que encapsulam |
| **Pass-through Method** | Método que não faz nada além de delegar para outro com mesma assinatura — indireção sem abstração |
| **Pass-through Variable** | Variável passada através de múltiplas camadas de métodos antes de ser usada no destino final |
| **Strategic Programming** | Mentalidade de investir 10-20% extra de tempo em design limpo, sabendo que o investimento se paga no médio prazo |
| **Tactical Programming** | Mentalidade de fazer funcionar o mais rápido possível, acumulando complexidade incremental |
| **Tactical Tornado** | Desenvolvedor que produz código rapidamente mas deixa rastro de complexidade para outros manterem |
| **Pull Complexity Downward** | Princípio de absorver complexidade na implementação do módulo em vez de expô-la na interface para o chamador |
| **Define Errors Out of Existence** | Técnica de projetar APIs onde o caso normal absorve edge cases, eliminando a necessidade de exceptions |
| **Design It Twice** | Prática de considerar pelo menos duas alternativas de design antes de implementar, comparando trade-offs |
| **Obviousness** | Propriedade do código onde um leitor competente entende rapidamente o que o código faz e por quê |
| **Obscurity** | Oposto de obviousness — código que requer esforço significativo para ser compreendido, mesmo por desenvolvedores experientes |
| **Somewhat General-Purpose** | Nível ideal de generalidade — genérico o suficiente para usos atuais e prováveis futuros, sem especulação |
| **Configuration Parameter Creep** | Acúmulo de parâmetros de configuração que transferem complexidade do módulo para o usuário |
| **Decorator Shallowness** | Tendência de decorators serem módulos inerentemente rasos — mesma interface com adição mínima de funcionalidade |
| **Different Layer Different Abstraction** | Princípio de que cada camada deve oferecer abstração distinta das adjacentes — violação indica problema de design |
| **Incremental Complexity** | A natureza acumulativa da complexidade — pequenas adições que individualmente parecem inócuas mas coletivamente tornam o sistema impenetrável |
