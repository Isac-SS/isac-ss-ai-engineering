# Software Engineering Toolkit

Toolkit de agentes, skills e referências voltada ao desenvolvimento de software assistido por IA.

O objetivo do projeto é fornecer um conjunto reutilizável de especialistas e procedimentos capazes de auxiliar durante diferentes etapas do desenvolvimento de software, como planejamento, decisões arquiteturais, implementação, debugging, testes e revisão de código.

A toolkit prioriza **simplicidade, contexto do projeto e decisões proporcionais ao problema**, evitando a aplicação mecânica de patterns, arquiteturas ou tecnologias.

## Objetivos

A toolkit busca:

- auxiliar na tomada de decisões técnicas e arquiteturais;
- transformar requisitos em planos de implementação;
- apoiar implementação e manutenção de código;
- investigar bugs de forma estruturada;
- auxiliar na criação e execução de testes;
- revisar alterações antes da entrega;
- reutilizar conhecimento especializado sem sobrecarregar o contexto dos agentes;
- adaptar recomendações às tecnologias e aos padrões existentes no projeto.

A toolkit deve complementar o trabalho do desenvolvedor, não substituir seu julgamento técnico.

---

## Conceitos

A toolkit utiliza três conceitos principais:

### Agent

Um **Agent** representa uma responsabilidade ou papel especializado.

Ele interpreta o contexto, toma decisões dentro de sua responsabilidade e utiliza skills quando necessário.

Exemplos:

- Architect
- Developer
- Reviewer

Um novo agente deve existir somente quando houver uma responsabilidade autônoma e recorrente que justifique sua separação.

### Skill

Uma **Skill** representa uma capacidade ou procedimento reutilizável.

Ela descreve **como executar determinado tipo de tarefa** e pode ser utilizada por diferentes agentes.

Exemplos:

- planejamento;
- análise arquitetural;
- implementação;
- debugging;
- testes;
- code review.

Uma skill não representa uma persona ou especialista independente.

### Reference

Uma **Reference** contém conhecimento especializado utilizado por uma skill.

Pode representar:

- princípios arquiteturais;
- documentação de frameworks;
- padrões;
- boas práticas;
- conhecimento extraído de literatura técnica;
- regras específicas de determinada tecnologia.

References fornecem conhecimento. Elas não executam tarefas nem tomam decisões.

---

## Relação entre os conceitos

```text
Agent
  │
  │ utiliza
  ▼
Skill
  │
  │ consulta quando necessário
  ▼
Reference
```

Exemplo:

```text
Architect
    │
    ▼
Architecture Skill
    │
    ├── DDD
    ├── Clean Architecture
    ├── Architecture Trade-offs
    ├── Production Readiness
    └── Framework Reference
```

O agente não deve carregar todo o conhecimento disponível antecipadamente.

Skills e references devem ser selecionadas conforme o problema analisado.

---

## Princípios

### Contexto antes de solução

Antes de recomendar ou implementar mudanças, os agentes devem compreender o projeto existente, suas restrições e seus padrões.

### Simplicidade primeiro

A solução deve possuir complexidade proporcional ao problema.

Patterns, abstrações, serviços, camadas ou tecnologias não devem ser introduzidos sem uma necessidade concreta.

### Evidência antes de suposição

Sempre que possível, decisões devem ser baseadas em evidências encontradas no projeto.

O agente deve distinguir claramente:

- fatos observados;
- requisitos informados;
- inferências;
- recomendações.

### Arquitetura serve ao negócio

Arquitetura não é um objetivo por si só.

Decisões arquiteturais devem responder a necessidades reais de negócio, manutenção, evolução, segurança, confiabilidade ou operação.

### Respeitar o ecossistema

Princípios gerais devem ser reconciliados com as práticas idiomáticas da linguagem, framework e plataforma utilizados pelo projeto.

### Mudanças incrementais

Prefira alterações pequenas, verificáveis e reversíveis.

Evite grandes reestruturações quando o mesmo objetivo puder ser alcançado incrementalmente.

### Human in the loop

Decisões relevantes permanecem sob responsabilidade humana.

Os agentes devem apresentar contexto, alternativas, consequências e trade-offs quando uma decisão exigir julgamento do desenvolvedor.

---

## Estrutura inicial

```text
.
├── README.md
│
├── agents/
│   ├── architect.md
│   ├── developer.md
│   └── reviewer.md
│
├── skills/
│   ├── architecture/
│   │   ├── SKILL.md
│   │   └── references/
│   │
│   ├── planning/
│   │   └── SKILL.md
│   │
│   ├── implementation/
│   │   └── SKILL.md
│   │
│   ├── debugging/
│   │   └── SKILL.md
│   │
│   ├── testing/
│   │   └── SKILL.md
│   │
│   └── code-review/
│       └── SKILL.md
│
└── docs/
```

Essa estrutura representa o ponto de partida e pode evoluir conforme necessidades reais forem identificadas durante o uso da toolkit.

---

## Agentes iniciais

### Architect

Responsável por análise e decisões arquiteturais.

Deve ajudar a:

- analisar arquiteturas existentes;
- identificar trade-offs;
- avaliar alternativas;
- detectar complexidade desnecessária;
- orientar boundaries e responsabilidades;
- avaliar impactos arquiteturais de mudanças;
- apoiar decisões que possam exigir ADRs.

### Developer

Responsável pela implementação.

Deve:

- compreender a solicitação;
- investigar o código relacionado;
- respeitar padrões existentes;
- planejar a alteração;
- implementar a menor mudança adequada;
- validar a implementação;
- recorrer ao Architect quando surgir uma decisão arquitetural relevante.

### Reviewer

Responsável pela revisão independente das alterações.

Deve procurar principalmente:

- bugs;
- regressões;
- problemas de segurança;
- problemas de integridade de dados;
- problemas de concorrência;
- tratamento inadequado de erros;
- testes ausentes ou insuficientes;
- violações arquiteturais;
- complexidade desnecessária.

O objetivo do Reviewer não é reescrever o código conforme preferências pessoais, mas encontrar problemas concretos e relevantes.

---

## Fluxo inicial

O fluxo básico da toolkit é:

```text
Requisito
   │
   ▼
Planning
   │
   ▼
Developer
   │
   ├──────── decisão arquitetural ────────► Architect
   │                                          │
   ◄──────────────────────────────────────────┘
   │
   ▼
Implementation
   │
   ▼
Reviewer
   │
   ▼
Code Review
```

Nem toda tarefa precisa utilizar todos os agentes ou skills.

Uma alteração simples deve permanecer simples.

---

## Quando criar algo novo?

Antes de adicionar um novo componente à toolkit, considere:

```text
É apenas conhecimento?
        │
       Sim
        ▼
    Reference

É um procedimento reutilizável?
        │
       Sim
        ▼
      Skill

É uma responsabilidade autônoma,
recorrente e suficientemente distinta?
        │
       Sim
        ▼
      Agent
```

### Regra geral

> Não crie um novo Agent quando uma Skill resolver o problema.

> Não crie uma nova Skill quando uma Reference ou uma evolução de uma skill existente resolver o problema.

A própria toolkit deve seguir os mesmos princípios de simplicidade e baixo acoplamento que busca aplicar aos projetos analisados.

---

## Estado do projeto

A toolkit está em desenvolvimento inicial.

A primeira versão tem como objetivo validar o fluxo:

```text
Planejar → Implementar → Revisar
```

utilizando projetos reais.

Novos agentes, skills e especializações devem surgir a partir de necessidades observadas durante o uso, e não apenas de possibilidades teóricas.

---

## Filosofia

Esta toolkit não busca produzir a arquitetura mais sofisticada possível.

Busca ajudar a produzir **software adequado ao problema, compreensível, sustentável e seguro para evoluir**.

A melhor solução não é necessariamente aquela com mais patterns, abstrações ou tecnologias.

É aquela cuja complexidade é justificada pelo problema que resolve.