---
titulo: "Framework Reference — Liferay CE 7.4"
fonte_primaria: "12 Knowledge Bases corporativas do SeaKit"
fontes_complementares:
  - "Liferay Learn (learn.liferay.com)"
  - "Liferay Source Code (github.com/liferay/liferay-portal)"
versao_framework: "CE 7.4 GA 112"
---

# Liferay CE 7.4 — Reference para Arquiteto Bob

## Deteccao

Sinais de que o projeto usa Liferay:
- `gradle.properties` com `liferay.workspace.product=portal-7.4-*`
- `build.gradle` com `compileOnly group: "com.liferay.portal"` ou `release.portal.api`
- Diretorios `modules/` com `*-api/`, `*-service/`, `*-rest/`, `*-web/`
- `service.xml` (Service Builder) ou `rest-openapi.yaml` (REST Builder)
- `bnd.bnd` com headers OSGi (`Bundle-SymbolicName`, `Web-ContextPath`)
- `.client-extension.yaml` (Client Extensions)

## Filosofia e Principios Core

Liferay e uma plataforma portal baseada em OSGi com dois modelos de extensao:

1. **Modulos OSGi** (tradicional) — Service Builder, REST Builder, Model Listeners, Service Wrappers
2. **Client Extensions** (moderno) — React Custom Elements, microservicos Spring Boot, Object Actions

Principios fundamentais:
- **Service Builder** gera a camada de persistencia — voce customiza `*LocalServiceImpl`
- **REST Builder** gera a camada REST — voce customiza `*ResourceImpl`
- **`@Reference`** e o mecanismo de DI do OSGi (equivalente ao `@Autowired` do Spring)
- **Separacao api/service/web** — `-api` publica interfaces, `-service` implementa, `-web` apresenta

## Dispatcher para Knowledge Bases

O SeaKit possui **12 Knowledge Bases especializadas** em Liferay. O Arquiteto Bob deve consultar a KB apropriada para cada dimensao arquitetural:

### Por dimensao de analise

| Dimensao | KBs a consultar | Prefixos de regra |
|----------|----------------|-------------------|
| **Estrutura e componentes** | `liferay-osgi-components` | OSG-*, CFG-*, MLS-*, EXT-* |
| **Qualidade e patterns** | `liferay-architecture-quality` | DPT-*, SOL-*, LOG-*, OBS-* |
| **Dominio (Service Builder)** | `liferay-service-builder-kb` | SB-* |
| **Dominio (REST Builder)** | `liferay-rest-builder-kb` | RB-* |
| **Persistencia e cache** | `liferay-data-persistence` | UPG-*, CCH-*, TRX-* |
| **Busca e indexacao** | `liferay-search-indexing` | IDX-*, AST-*, EXP-* |
| **Mensageria e agendamento** | `liferay-messaging-scheduling` | SCH-*, MSB-*, BTC-* |
| **Seguranca** | `liferay-security-kb` | SEC-*, PRM-*, ANTI-SEC-*, TSEC-* |
| **Performance** | `liferay-architecture-quality` (subset) | PERF-JAVA-*, PERF-LIF-*, PERF-DB-*, ANTI-PERF-* |
| **Plataforma moderna** | `liferay-modern-platform` | CXB-*, OBJ-*, HLS-* |
| **Workflow e integracao** | `liferay-workflow-integration` | WFL-*, EAI-* |
| **Diagnostico OSGi** | `liferay-gogo-diagnostics` | GGS-* |
| **Testes** | `liferay-testing-kb` | TST-* (120+ docs) |

### Por tipo de modulo detectado

| Arquivo detectado | Tipo de modulo | KBs relevantes |
|-------------------|---------------|----------------|
| `service.xml` | Service Builder | `liferay-service-builder-kb`, `liferay-data-persistence` |
| `rest-openapi.yaml` | REST Builder | `liferay-rest-builder-kb` |
| `*ModelListener.java` | Model Listener | `liferay-osgi-components` (MLS-*) |
| `*ServiceWrapper.java` | Service Wrapper | `liferay-osgi-components` (EXT-*) |
| `*Configuration.java` com `@Meta.OCD` | Config API | `liferay-osgi-components` (CFG-*) |
| `*SchedulerJobConfiguration.java` | Scheduler | `liferay-messaging-scheduling` (SCH-*) |
| `*Indexer.java` ou `*ModelDocumentContributor.java` | Search | `liferay-search-indexing` (IDX-*) |
| `.client-extension.yaml` | Client Extension | `liferay-modern-platform` (CXB-*) |
| `*UpgradeProcess.java` | Upgrade | `liferay-data-persistence` (UPG-*) |

### Por sinal arquitetural

| Sinal detectado | KBs a acionar | Acao |
|----------------|---------------|------|
| Ausencia de `PermissionChecker` em `*ServiceImpl` | `liferay-security-kb` (PRM-*) | Verificar controle de acesso |
| `QueryUtil.ALL_POS` sem paginacao | `liferay-architecture-quality` (PERF-LIF-001) | Bloqueante — corrigir |
| `*LocalServiceUtil` chamado fora de OSGi | `liferay-osgi-components` (OSG-*) | Usar `@Reference` |
| Logica de negocio em `*ResourceImpl` | `liferay-rest-builder-kb` (RB-*) | Mover para `*LocalServiceImpl` |
| N+1 em loop de entidades | `liferay-architecture-quality` (PERF-*) | Usar finder ou DynamicQuery |
| Shared database entre modulos | `liferay-service-builder-kb` (SB-*) | Usar API inter-modulo |
| Ausencia de testes | `liferay-testing-kb` | Gerar testes com agente `liferay-test-writer` |

## Agentes Especializados

Alem das KBs, o SeaKit oferece agentes que podem ser delegados:

| Agente | Funcao |
|--------|--------|
| `liferay-backend-kb` | Consultor read-only que navega as 12 KBs para responder duvidas |
| `liferay-security-perf-reviewer` | Code review automatizado com 118+ regras de seguranca e 87+ de performance |
| `liferay-test-writer` | Gerador de testes unitarios e de integracao |
| `liferay-reviewer` | Code review geral Liferay |

## Reconciliacao com Livros de Arquitetura

| Conceito (livro) | Equivalente Liferay | Nota |
|---|---|---|
| Repository (Evans) | `*Persistence` (gerado pelo Service Builder) | Nunca implemente manualmente — SB gera |
| Domain Service (Evans) | `*LocalServiceImpl` | Logica de negocio mora aqui |
| Application Service (Evans) | `*ServiceImpl` (remote) ou `*ResourceImpl` (REST) | Coordena + controle de acesso |
| Use Case (Martin) | `*LocalServiceImpl` metodo publico | Cada metodo = um use case |
| Entity (Evans) | `*Model` + `*ModelImpl` | Gerado pelo SB; customizar `*ModelImpl` somente |
| Value Object (Evans) | Classes em `-api` sem `service.xml` | Criar manualmente |
| Aggregate (Evans/Vernon) | Entidade raiz no `service.xml` com `<column>` e finders | Uma entidade por `*LocalServiceImpl` |
| Bounded Context (Evans) | Modulo OSGi (`-api` + `-service`) | Um modulo = um contexto |
| Anti-Corruption Layer (Evans) | Adapter em `-service` que chama API externa | Usar `HttpUtil` ou `JSONFactoryUtil` |
| Circuit Breaker (Nygard) | Nao nativo — implementar manualmente ou via Client Extension | Considerar para integracoes externas |
| Observer (GoF) | `ModelListener` | `onAfterCreate`, `onAfterUpdate`, `onAfterRemove` |
| Strategy (GoF) | OSGi service ranking | `@Reference` com `target` filter |
| Template Method (GoF) | `*BaseImpl` → `*Impl` | Padrao central do Service Builder |
| Decorator (GoF) | `*ServiceWrapper` | Intercepta chamadas ao service original |
| Factory (GoF) | `*LocalServiceUtil` | Acesso estatico (evitar — preferir `@Reference`) |

## Estrutura Canonica de Modulo Liferay

```
meu-modulo/
├── meu-modulo-api/           # Interfaces publicas, models, exceptions
│   ├── bnd.bnd
│   ├── build.gradle
│   └── src/main/java/
│       └── com/empresa/modulo/
│           ├── model/        # Interfaces de entidade (geradas pelo SB)
│           ├── service/      # Interfaces de servico (geradas pelo SB)
│           └── exception/    # Exceptions customizadas
│
├── meu-modulo-service/       # Implementacao + persistencia
│   ├── bnd.bnd
│   ├── build.gradle
│   ├── service.xml           # Definicao do Service Builder
│   └── src/
│       ├── main/java/
│       │   └── com/empresa/modulo/
│       │       ├── model/impl/       # *ModelImpl (customizavel)
│       │       ├── service/impl/     # *LocalServiceImpl (logica de negocio)
│       │       └── service/persistence/impl/  # *PersistenceImpl (gerado)
│       └── test/java/                # Testes unitarios
│
├── meu-modulo-rest/          # API REST (REST Builder)
│   ├── rest-openapi.yaml     # Spec OpenAPI
│   ├── rest-config.yaml
│   └── src/
│       ├── main/java/.../resource/v1_0/  # *ResourceImpl
│       └── test/java/                     # Testes unitarios
│
└── meu-modulo-web/           # UI (portlet, fragments, client extensions)
    ├── bnd.bnd
    └── src/main/
        ├── java/             # Controllers (MVCPortlet)
        └── resources/META-INF/resources/  # JSP/FreeMarker
```

## Checklist Arquitetural Liferay

- [ ] Cada modulo tem separacao `-api` / `-service` (nunca logica em `-api`)
- [ ] Logica de negocio em `*LocalServiceImpl`, nunca em `*ResourceImpl` ou controller
- [ ] `*ServiceImpl` (remote) aplica `PermissionChecker` antes de delegar a `*LocalServiceImpl`
- [ ] Nenhum `*LocalServiceUtil` — usar `@Reference` em todo componente OSGi
- [ ] Queries paginadas — nunca `QueryUtil.ALL_POS` em producao
- [ ] `ServiceContext` sempre com `companyId > 0` e `userId > 0`
- [ ] ThreadLocals limpos apos uso (`@AfterEach` em testes, `finally` em codigo)
- [ ] Testes unitarios para todo `*LocalServiceImpl` e `*ResourceImpl`
- [ ] `bnd.bnd` com `Export-Package` apenas para `-api`
- [ ] Client Extensions para frontend novo (nao portlets JSP)

## Quando Liferay Nao Basta

| Sinal | Alternativa |
|-------|-------------|
| API pura sem portal/CMS | Spring Boot standalone ou Client Extension microservice |
| Frontend complexo (SPA) | Client Extension React + headless APIs |
| Processamento massivo de dados | Job externo (Spring Batch) integrado via REST |
| Real-time / WebSockets | Servico separado integrado via Message Bus ou Client Extension |
| ML/AI pipeline | Servico Python/FastAPI integrado via Object Action ou REST |
