---
titulo: "Framework Reference — Spring Boot"
fonte_primaria: "Spring Boot Reference (docs.spring.io)"
fontes_complementares:
  - "Spring Framework Reference"
  - "Spring Security Reference"
  - "Spring Data JPA Reference"
  - "Cloud Native Java (Josh Long)"
versao_framework: "3.x (Java 17+)"
---

# Spring Boot — Reference para Arquiteto Bob

## Deteccao

Um projeto Spring Boot e identificado por:

- **Maven**: `pom.xml` contendo `spring-boot-starter-parent` como parent ou dependencias `spring-boot-starter-*`
- **Gradle**: `build.gradle` / `build.gradle.kts` com plugin `org.springframework.boot` e/ou dependencias `org.springframework.boot:spring-boot-starter-*`
- **Classe principal**: presenca de `@SpringBootApplication` no pacote raiz
- **Arquivos de configuracao**: `application.properties` ou `application.yml` em `src/main/resources/`

Fonte: [Spring Boot Reference — Getting Started](https://docs.spring.io/spring-boot/reference/)

---

## Filosofia e Principios Core

Spring Boot segue quatro principios fundamentais, conforme a documentacao oficial:

1. **Opinionated defaults**: fornece configuracoes sensiveis por padrao (servidor embarcado Tomcat, Jackson para JSON, SLF4J para logging) que funcionam sem configuracao manual. O objetivo e eliminar boilerplate.

2. **Auto-configuration**: detecta automaticamente as dependencias no classpath e configura beans apropriados. `@SpringBootApplication` combina `@Configuration`, `@EnableAutoConfiguration` e `@ComponentScan`.

3. **Starter POMs**: agrupam dependencias transitivas coesas. `spring-boot-starter-web` traz Tomcat, Jackson e Spring MVC; `spring-boot-starter-data-jpa` traz Hibernate, HikariCP e Spring Data.

4. **Production-ready**: Actuator fornece endpoints de health, metrics e monitoramento prontos para producao.

**Principio arquitetural**: a arquitetura do Spring Boot favorece **convencao sobre configuracao** — o framework toma decisoes sensiveis e permite override granular quando necessario.

Fonte: [Spring Boot Reference — Overview](https://docs.spring.io/spring-boot/reference/)

---

## Estrutura Canonica

A documentacao oficial recomenda **package-by-feature** (organizacao por dominio de negocio), com a classe principal no pacote raiz:

```
com
 +- example
     +- myapplication
         +- MyApplication.java            # @SpringBootApplication no pacote raiz
         |
         +- customer
         |   +- Customer.java             # Entity
         |   +- CustomerController.java   # REST endpoint
         |   +- CustomerService.java      # Logica de negocio
         |   +- CustomerRepository.java   # Acesso a dados
         |
         +- order
             +- Order.java
             +- OrderController.java
             +- OrderService.java
             +- OrderRepository.java
```

**Regras importantes**:
- Nunca usar o default package (sem declaracao `package`) — causa problemas com `@ComponentScan`, `@EntityScan` e `@ConfigurationPropertiesScan`
- Seguir convencao de nomenclatura de pacotes com domain name reverso (ex: `com.example.project`)
- A classe `@SpringBootApplication` define implicitamente o pacote base de scan
- Para projetos maiores, considerar **Spring Modulith** para enforcement de boundaries entre modulos

Fonte: [Spring Boot Reference — Structuring Your Code](https://docs.spring.io/spring-boot/reference/using/structuring-your-code.html)

---

## Patterns Idiomaticos

### IoC/DI com Spring

O container IoC do Spring gerencia o ciclo de vida dos objetos (beans) e suas dependencias. Os stereotypes anotam beans por camada:

| Stereotype | Camada | Semantica |
|---|---|---|
| `@Component` | Generica | Bean gerenciado generico |
| `@Service` | Logica de negocio | Marca um service — sem comportamento adicional alem de `@Component` |
| `@Repository` | Acesso a dados | Traduz automaticamente excecoes de persistencia para `DataAccessException` |
| `@Controller` / `@RestController` | Web | Trata requisicoes HTTP |

**Constructor injection e a abordagem recomendada pela documentacao oficial** porque:
- Permite marcar dependencias como `final` (imutabilidade)
- Torna dependencias explicitas e obrigatorias
- Facilita testes unitarios (basta passar mocks no construtor)

```java
import org.springframework.stereotype.Service;

@Service
public class MyAccountService implements AccountService {

    private final RiskAssessor riskAssessor;

    // Com um unico construtor, @Autowired e opcional
    public MyAccountService(RiskAssessor riskAssessor) {
        this.riskAssessor = riskAssessor;
    }

    // ...
}
```

Quando ha multiplos construtores, marque o desejado com `@Autowired`:

```java
@Service
public class MyAccountService implements AccountService {

    private final RiskAssessor riskAssessor;
    private final PrintStream out;

    @Autowired
    public MyAccountService(RiskAssessor riskAssessor) {
        this.riskAssessor = riskAssessor;
        this.out = System.out;
    }

    public MyAccountService(RiskAssessor riskAssessor, PrintStream out) {
        this.riskAssessor = riskAssessor;
        this.out = out;
    }
}
```

Fonte: [Spring Boot Reference — Spring Beans and Dependency Injection](https://docs.spring.io/spring-boot/reference/using/spring-beans-and-dependency-injection.html), [Spring Framework Reference — IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html)

---

### Spring Data JPA (vs Repository Pattern de Evans)

Spring Data JPA elimina boilerplate de acesso a dados expondo interfaces que o framework implementa automaticamente em runtime.

**Hierarquia de interfaces**:

```
Repository (marker)
  └─ CrudRepository (save, findById, findAll, delete, count, existsById)
       └─ ListCrudRepository (retorna List ao inves de Iterable)
            └─ JpaRepository (flush, saveAndFlush, deleteInBatch, JPA-specific)
       └─ PagingAndSortingRepository (findAll com Sort e Pageable)
```

**Derived query methods** — o framework gera JPQL a partir do nome do metodo:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    List<User> findByEmailAddressAndLastname(String email, String lastname);
    // → select u from User u where u.emailAddress = ?1 and u.lastname = ?2

    List<User> findByLastnameOrderByFirstnameDesc(String lastname);
    // → select u from User u where u.lastname = ?1 order by u.firstname desc

    List<User> findByFirstnameContaining(String name);
    // → select u from User u where u.firstname like %?1%

    Page<User> findByLastname(String lastname, Pageable pageable);
    // → paginacao automatica
}
```

**@Query para consultas complexas**:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("select u from User u where u.firstname = :firstname or u.lastname = :lastname")
    User findByLastnameOrFirstname(@Param("lastname") String lastname,
                                   @Param("firstname") String firstname);

    @Modifying
    @Query("update User u set u.firstname = ?1 where u.lastname = ?2")
    int setFixedFirstnameFor(String firstname, String lastname);
}
```

**Reconciliacao com DDD Repository (Evans)**: o Repository Pattern de Evans e um conceito de dominio — uma abstracao que simula uma colecao de aggregates. O `JpaRepository` do Spring Data e uma **implementacao tecnica** desse conceito. A reconciliacao correta e:

- O aggregate root define o que o Repository gerencia (Evans)
- O `JpaRepository<AggregateRoot, ID>` implementa o acesso a dados (Spring)
- Query methods complexas devem usar `Specification` ou `@Query` — nao poluir a interface com dezenas de `findBy*`
- Para DDD puro, considere criar uma interface de dominio (ex: `OrderRepository`) e uma implementacao Spring Data que a estende

Fonte: [Spring Data JPA Reference — JPA Repositories](https://docs.spring.io/spring-data/jpa/reference/jpa.html), [Spring Data JPA Reference — Query Methods](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)

---

### @Transactional

O gerenciamento declarativo de transacoes do Spring usa proxies AOP. `@Transactional` e o mecanismo central.

**Propagation types**:

| Propagation | Comportamento |
|---|---|
| `REQUIRED` (padrao) | Usa transacao existente ou cria nova |
| `REQUIRES_NEW` | Sempre cria nova transacao (suspende a existente) |
| `SUPPORTS` | Usa transacao existente se houver; senao, roda sem transacao |
| `NOT_SUPPORTED` | Suspende transacao existente |
| `MANDATORY` | Exige transacao existente; lanca excecao se nao houver |
| `NEVER` | Lanca excecao se houver transacao ativa |
| `NESTED` | Cria savepoint dentro da transacao existente |

**Exemplo com propagation e read-only**:

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        paymentService.processPayment(order);
        // Ambas operacoes na mesma transacao (REQUIRED padrao)
    }

    @Transactional(readOnly = true)
    public List<Order> getAllOrders() {
        // Otimizacoes: flush mode NEVER, Hibernate pula dirty checking
        return orderRepository.findAll();
    }
}

@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;

    public PaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processPayment(Order order) {
        // Transacao independente — se falhar, nao faz rollback do pedido
        Payment payment = new Payment(order);
        paymentRepository.save(payment);
    }
}
```

**Rollback rules**:

```java
@Transactional(
    rollbackFor = {CustomBusinessException.class},
    noRollbackFor = {ValidationException.class}
)
public void processTransaction() {
    // Rollback em CustomBusinessException
    // NAO faz rollback em ValidationException
}
```

**Pitfall critico — self-invocation (proxy-based AOP)**: quando um metodo chama outro metodo `@Transactional` na **mesma classe**, a chamada nao passa pelo proxy AOP, e a anotacao `@Transactional` e **ignorada silenciosamente**.

```java
// ERRADO — self-invocation, @Transactional ignorada
@Service
public class UserService {

    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        sendWelcomeEmail(user); // chamada direta — NAO passa pelo proxy
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendWelcomeEmail(User user) {
        // Esta anotacao e IGNORADA por causa de self-invocation
    }
}

// CORRETO — mover para outro bean
@Service
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;

    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        emailService.sendWelcomeEmail(user); // chamada via proxy de outro bean
    }
}

@Service
public class EmailService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendWelcomeEmail(User user) {
        // Transacao nova, funciona corretamente
    }
}
```

Fonte: [Spring Framework Reference — Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)

---

### Controllers e REST

`@RestController` combina `@Controller` + `@ResponseBody`, retornando dados serializados (JSON por padrao via Jackson).

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserRepository userRepository;
    private final CustomerRepository customerRepository;

    public UserController(UserRepository userRepository, CustomerRepository customerRepository) {
        this.userRepository = userRepository;
        this.customerRepository = customerRepository;
    }

    @GetMapping("/{userId}")
    public User getUser(@PathVariable Long userId) {
        return this.userRepository.findById(userId).orElseThrow(
            () -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    }

    @GetMapping("/{userId}/customers")
    public List<Customer> getUserCustomers(@PathVariable Long userId) {
        return this.userRepository.findById(userId)
            .map(this.customerRepository::findByUser)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    }

    @DeleteMapping("/{userId}")
    public void deleteUser(@PathVariable Long userId) {
        this.userRepository.deleteById(userId);
    }
}
```

**Validacao com `@Valid` e Bean Validation**:

```java
@PostMapping
public ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest request) {
    User user = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(user);
}

public record CreateUserRequest(
    @NotBlank String name,
    @Email String email,
    @Min(18) int age
) {}
```

**Tratamento global de erros com `@ControllerAdvice`**:

```java
@ControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(), ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    public record ErrorResponse(int status, String message) {}
}
```

Fonte: [Spring Boot Reference — Servlet Web Applications](https://docs.spring.io/spring-boot/reference/web/servlet.html)

---

### Spring Security

A partir do Spring Security 6.x, a configuracao e feita via `SecurityFilterChain` como bean (o antigo `WebSecurityConfigurerAdapter` foi removido).

**Configuracao basica**:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/users/**").hasRole("USER")
                .requestMatchers(HttpMethod.POST, "/api/users").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
            );
        return http.build();
    }
}
```

**Method security com `@PreAuthorize`**:

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}

@Service
public class AccountService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAccount(String id) {
        // Somente ADMIN pode acessar
    }

    @PreAuthorize("hasAuthority('WRITE_ACCOUNT')")
    public void updateAccount(Account account) {
        // Somente usuarios com authority WRITE_ACCOUNT
    }
}
```

**OAuth2 Resource Server (JWT)**:

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );
        return http.build();
    }

    private JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(new JwtGrantedAuthoritiesConverter());
        return converter;
    }
}
```

Fonte: [Spring Security Reference](https://docs.spring.io/spring-security/reference/)

---

### Profiles e Configuration

Profiles permitem segregar configuracao por ambiente (dev, test, prod).

**Ativacao de profiles**:

```properties
# application.properties
spring.profiles.active=dev,hsqldb
```

```bash
# Linha de comando (override sobre properties)
--spring.profiles.active=dev,hsqldb
```

**Beans condicionais com `@Profile`**:

```java
@Configuration(proxyBeanMethods = false)
@Profile("production")
public class ProductionConfiguration {
    // Configuracao carregada apenas quando profile "production" esta ativo
}
```

**Profile groups** — agrupar profiles relacionados:

```yaml
spring:
  profiles:
    group:
      production:
        - "proddb"
        - "prodmq"
```

Com `--spring.profiles.active=production`, os profiles `production`, `proddb` e `prodmq` sao ativados juntos.

**`@ConfigurationProperties` para configuracao type-safe** (preferivel a `@Value` para configuracoes hierarquicas):

```java
@ConfigurationProperties("my.service")
public class MyServiceProperties {

    private final boolean enabled;
    private final InetAddress remoteAddress;
    private final Security security;

    public MyServiceProperties(boolean enabled, InetAddress remoteAddress, Security security) {
        this.enabled = enabled;
        this.remoteAddress = remoteAddress;
        this.security = security;
    }

    public boolean isEnabled() { return this.enabled; }
    public InetAddress getRemoteAddress() { return this.remoteAddress; }
    public Security getSecurity() { return this.security; }

    public static class Security {

        private final String username;
        private final String password;
        private final List<String> roles;

        public Security(String username, String password,
                @DefaultValue("USER") List<String> roles) {
            this.username = username;
            this.password = password;
            this.roles = roles;
        }

        public String getUsername() { return this.username; }
        public String getPassword() { return this.password; }
        public List<String> getRoles() { return this.roles; }
    }
}
```

```yaml
my:
  service:
    enabled: true
    remote-address: 192.168.1.1
    security:
      username: "admin"
      password: "secret"
      roles:
        - "USER"
        - "ADMIN"
```

Habilitar scan automatico:

```java
@SpringBootApplication
@ConfigurationPropertiesScan("com.example.app")
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

Fonte: [Spring Boot Reference — Profiles](https://docs.spring.io/spring-boot/reference/features/profiles.html), [Spring Boot Reference — Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)

---

### Actuator (Production Readiness)

Spring Boot Actuator fornece endpoints de monitoramento e gerenciamento prontos para producao.

**Dependencia**:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**Endpoints principais**:

| Endpoint | Descricao |
|---|---|
| `/actuator/health` | Status de saude da aplicacao (unico exposto por padrao via HTTP) |
| `/actuator/info` | Informacoes arbitrarias da aplicacao |
| `/actuator/metrics` | Metricas (JVM, HTTP, custom) |
| `/actuator/beans` | Todos os beans do contexto |
| `/actuator/env` | Propriedades de configuracao |
| `/actuator/loggers` | Nivel de log em runtime |
| `/actuator/prometheus` | Metricas em formato Prometheus |

**Exposicao de endpoints**:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info,metrics,prometheus"
  endpoint:
    health:
      show-details: "when-authorized"
      roles: "admin"
```

**Custom health indicator**:

```java
@Component
public class MyHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        int errorCode = check();
        if (errorCode != 0) {
            return Health.down()
                .withDetail("Error Code", errorCode)
                .build();
        }
        return Health.up().build();
    }

    private int check() {
        // Verificacao de saude especifica (ex: conectividade com servico externo)
        return 0;
    }
}
```

O identificador do endpoint e o nome do bean sem o sufixo `HealthIndicator` — ex: `MyHealthIndicator` fica disponivel em `/actuator/health/my`.

**Custom actuator endpoint**:

```java
@Component
@Endpoint(id = "custom")
public class CustomEndpoint {

    @ReadOperation
    public CustomData getData() {
        return new CustomData("test", 5);
    }

    @WriteOperation
    public void updateData(String name, int counter) {
        // operacao de escrita
    }

    public record CustomData(String name, int value) {}
}
```

**Seguranca dos endpoints do Actuator**:

```java
@Configuration(proxyBeanMethods = false)
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        http.securityMatcher(EndpointRequest.toAnyEndpoint());
        http.authorizeHttpRequests(requests ->
            requests.anyRequest().hasRole("ENDPOINT_ADMIN"));
        http.httpBasic(withDefaults());
        return http.build();
    }
}
```

Fonte: [Spring Boot Reference — Actuator](https://docs.spring.io/spring-boot/reference/actuator/), [Spring Boot Reference — Actuator Endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html)

---

## Modularizacao e Boundaries

### Multi-module Maven/Gradle

Para projetos maiores, particionar em modulos Maven/Gradle:

```
my-application/
  +- pom.xml                      # parent POM
  +- my-application-domain/       # Entities, value objects, interfaces de repositorio
  +- my-application-infra/        # Implementacoes JPA, configuracoes de banco
  +- my-application-web/          # Controllers, DTOs, Spring Boot main class
```

Cada modulo declara dependencias apenas do que realmente usa, enforcing boundaries via sistema de build.

### Package-private como boundary enforcement

Em Java, visibilidade `package-private` (sem modificador) impede que classes de outros pacotes acessem implementacoes internas. Dentro de um feature package, exponha apenas a interface publica:

```java
// com.example.order (pacote publico)
public interface OrderService { ... }      // public — acessivel externamente
public record OrderCreatedEvent(...) { }   // public — evento de integracao

// com.example.order (implementacao package-private)
class DefaultOrderService implements OrderService { ... }  // package-private
class OrderValidator { ... }                               // package-private
```

### Spring Modulith

Para enforcement automatizado de boundaries, o Spring Modulith verifica em tempo de teste que modulos nao violam dependencias declaradas:

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-bom</artifactId>
    <version>2.1.0</version>
    <scope>import</scope>
    <type>pom</type>
</dependency>
```

O Modulith trata cada pacote direto abaixo do pacote raiz como um modulo de aplicacao e verifica que:
- Modulos nao acessam internals de outros modulos
- Dependencias ciclicas nao existem
- A comunicacao entre modulos pode ser feita via eventos de aplicacao (`ApplicationEventPublisher`)

Fonte: [Spring Modulith Reference](https://docs.spring.io/spring-modulith/reference/), [Spring Boot Reference — Structuring Your Code](https://docs.spring.io/spring-boot/reference/using/structuring-your-code.html)

---

## Anti-Patterns Spring Boot

### 1. Field injection

```java
// ERRADO — field injection
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // nao-final, mutavel, dificil de testar
}

// CORRETO — constructor injection
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

**Problema**: field injection esconde dependencias, impede imutabilidade e dificulta testes unitarios (requer reflection ou framework de mock especial). A documentacao oficial do Spring Boot recomenda explicitamente constructor injection.

### 2. Dependencias circulares

```java
// ERRADO — ciclo: A depende de B, B depende de A
@Service
public class ServiceA {
    private final ServiceB serviceB;
    public ServiceA(ServiceB serviceB) { this.serviceB = serviceB; }
}

@Service
public class ServiceB {
    private final ServiceA serviceA;
    public ServiceB(ServiceA serviceA) { this.serviceA = serviceA; }
}
// Spring 3.x lanca BeanCurrentlyInCreationException por padrao
```

**Solucao**: refatorar extraindo a logica compartilhada para um terceiro service, ou usar eventos (`ApplicationEventPublisher`) para desacoplar.

### 3. @Transactional em metodo privado

```java
// ERRADO — proxy AOP nao intercepta metodos privados
@Service
public class UserService {

    @Transactional
    private void saveUser(User user) {  // NUNCA funciona
        userRepository.save(user);
    }
}

// CORRETO — metodo publico
@Service
public class UserService {

    @Transactional
    public void saveUser(User user) {
        userRepository.save(user);
    }
}
```

### 4. Bean scope confusion

```java
// ERRADO — injetar bean @Scope("prototype") em singleton
// O prototype e criado uma unica vez e reutilizado
@Service // singleton por padrao
public class OrderProcessor {
    private final ShoppingCart cart;  // prototype injetado uma vez

    public OrderProcessor(ShoppingCart cart) {
        this.cart = cart;  // sempre a mesma instancia!
    }
}

// CORRETO — usar ObjectFactory ou Provider
@Service
public class OrderProcessor {
    private final ObjectFactory<ShoppingCart> cartFactory;

    public OrderProcessor(ObjectFactory<ShoppingCart> cartFactory) {
        this.cartFactory = cartFactory;
    }

    public void process() {
        ShoppingCart cart = cartFactory.getObject(); // nova instancia a cada chamada
    }
}
```

### 5. Fat @Configuration classes

```java
// ERRADO — uma classe @Configuration com 30+ @Bean methods
@Configuration
public class AppConfig {
    @Bean public DataSource dataSource() { ... }
    @Bean public SecurityFilterChain security() { ... }
    @Bean public ObjectMapper mapper() { ... }
    // ... 27 outros beans
}

// CORRETO — separar por responsabilidade
@Configuration
public class DataSourceConfig { ... }

@Configuration
public class SecurityConfig { ... }

@Configuration
public class JacksonConfig { ... }
```

---

## Reconciliacao com Livros de Arquitetura

| Conceito (livro) | Equivalente Spring | Nota |
|---|---|---|
| Repository (Evans, DDD) | `JpaRepository` / Spring Data interfaces | Spring Data implementa o pattern tecnicamente; o conceito de dominio (colecao de aggregates) requer disciplina extra do desenvolvedor |
| Use Case / Interactor (Martin, Clean Architecture) | `@Service` | O service Spring e o local natural para orquestrar use cases; manter controllers finos e services focados |
| Entity (Evans, DDD) | `@Entity` JPA | A entity JPA mistura persistencia com dominio; em DDD puro, separar domain model da entity JPA |
| Value Object (Evans, DDD) | `@Embeddable` JPA ou record Java | Records sao imutaveis por natureza — excelente fit para value objects |
| Circuit Breaker (Nygard, Release It!) | Resilience4j + `spring-cloud-circuitbreaker` | Spring Cloud Circuit Breaker abstrai Resilience4j; configuravel via properties |
| Bulkhead (Nygard) | Resilience4j `@Bulkhead` | Limita concorrencia por service; configuravel com thread pool ou semaforo |
| Strangler Fig (Newman, Building Microservices) | Spring Cloud Gateway + routing seletivo | Gateway roteia gradualmente trafego do monolito para novos services |
| Bounded Context (Evans, DDD) | Modulo Spring / Spring Modulith module | Package-by-feature + Modulith enforcements mapeia bounded contexts em codigo |
| Fitness Functions (Ford & Richards) | ArchUnit + Spring Modulith verify | Testes automatizados que verificam constraints arquiteturais |
| Deep Modules (Ousterhout, A Philosophy of Software Design) | Auto-configuration + starter POMs | Spring Boot e um exemplo de deep module: interface simples (`@SpringBootApplication`), funcionalidade rica por tras |

---

## Checklist Arquitetural Spring Boot

- [ ] **Estrutura de pacotes**: package-by-feature com classe `@SpringBootApplication` no pacote raiz
- [ ] **Injecao de dependencias**: constructor injection exclusivamente; campos `final`
- [ ] **Camadas**: controllers finos (validacao + delegacao), services com logica de negocio, repositories para acesso a dados
- [ ] **Transacoes**: `@Transactional` no service layer; `readOnly = true` para queries; sem self-invocation
- [ ] **Configuracao externalizada**: `@ConfigurationProperties` para configuracoes hierarquicas; profiles por ambiente
- [ ] **Seguranca**: `SecurityFilterChain` como bean; method security com `@PreAuthorize` onde apropriado
- [ ] **Actuator**: health check customizado para dependencias externas; endpoints sensiveis protegidos
- [ ] **Testes**: `@SpringBootTest` para integracao; slices (`@WebMvcTest`, `@DataJpaTest`) para testes focados
- [ ] **Sem anti-patterns**: sem field injection, sem dependencias circulares, sem `@Transactional` em metodo privado
- [ ] **Modularizacao**: boundaries claros entre features; Spring Modulith se o projeto crescer

---

## Quando Spring Boot Nao Basta

Spring Boot e excelente para a maioria das aplicacoes, mas sinais de que patterns adicionais sao necessarios incluem:

1. **Dominio complexo com muitas regras de negocio**: considerar Hexagonal Architecture / Ports & Adapters para isolar o dominio do framework. Spring Boot vive na camada de infraestrutura; o dominio deve ser POJO puro.

2. **Multiplos modelos de leitura e escrita**: CQRS (Command Query Responsibility Segregation) com query models separados. Spring Data projections ajudam, mas para modelos de leitura radicalmente diferentes, considerar views materializadas ou read stores separados.

3. **Consistencia eventual entre servicos**: Event-driven architecture com Spring Cloud Stream ou Spring for Apache Kafka. O `ApplicationEventPublisher` do Spring funciona para eventos intra-processo; para inter-processo, sao necessarios brokers.

4. **Alto volume de dados com requisitos de escala**: particionamento, replicacao e stream processing (Kleppmann). Spring Boot fornece o chassis, mas o design de dados precisa de atencao dedicada.

5. **Multiplas equipes no mesmo codebase**: Spring Modulith para modular monolith com boundaries enforced; considerar decomposicao em microservices apenas quando a complexidade organizacional justificar o custo operacional (Newman).

6. **Resiliencia critica (sistemas financeiros, saude)**: patterns de estabilidade de Nygard — circuit breakers, bulkheads, timeouts, retry com backoff. Resilience4j integra-se naturalmente com Spring Boot via Spring Cloud Circuit Breaker.
