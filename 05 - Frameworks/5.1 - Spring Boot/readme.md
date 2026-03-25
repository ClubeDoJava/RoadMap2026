# 5.1 — Spring Boot

Spring Boot é o framework mais usado no ecossistema Java. Ele simplifica a criação de aplicações Spring ao eliminar a configuração manual: servidor embutido, auto-configuração, e convenções sensatas prontas para uso.

> **Antes de começar:** adicione Lombok ao seu `pom.xml` agora. Você vai escrever muitas classes com construtores, getters e builders — o Lombok elimina esse boilerplate via anotações (`@Data`, `@Builder`, `@RequiredArgsConstructor`, `@Slf4j`). O módulo 7.3 explica as anotações em detalhe. Para Records (DTOs imutáveis), prefira o Record do Java 16+ ao `@Data` do Lombok — mais simples e sem dependência extra.
>
> ```xml
> <dependency>
>     <groupId>org.projectlombok</groupId>
>     <artifactId>lombok</artifactId>
>     <optional>true</optional>
> </dependency>
> ```

---

## Spring Core — IoC e Injeção de Dependência

### IoC (Inversão de Controle)

Sem IoC, você cria e gerencia seus objetos manualmente:

```java
// Sem IoC — você controla a criação
EmailService emailService = new EmailService();
UsuarioService usuarioService = new UsuarioService(emailService);
```

Com IoC, o container Spring cria e conecta os objetos para você:

```java
// Com IoC — o Spring gerencia
@Service
public class UsuarioService {
    private final EmailService emailService;

    public UsuarioService(EmailService emailService) { // Spring injeta automaticamente
        this.emailService = emailService;
    }
}
```

### DI (Injeção de Dependência) — as três formas

```java
// 1. Constructor Injection (PREFERIDA)
// Vantagens: imutabilidade, facilita testes, dependências visíveis
@Service
public class PedidoService {

    private final ProdutoRepository produtoRepository;
    private final NotificacaoService notificacaoService;

    // @Autowired é opcional quando há apenas um construtor (Spring 4.3+)
    public PedidoService(ProdutoRepository produtoRepository,
                         NotificacaoService notificacaoService) {
        this.produtoRepository = produtoRepository;
        this.notificacaoService = notificacaoService;
    }
}

// 2. Field Injection (EVITAR)
// Ruim: não funciona sem Spring, dificulta testes unitários, esconde dependências
@Service
public class PedidoService {
    @Autowired
    private ProdutoRepository produtoRepository; // não faça isso
}

// 3. Setter Injection (usar raramente, para dependências opcionais)
@Service
public class ReportService {

    private EmailService emailService;

    @Autowired(required = false)
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

### Estereótipos — quando usar cada anotação

```java
@Component    // componente genérico — uso geral
@Service      // lógica de negócio — semântica clara
@Repository   // acesso a dados — traduz exceções de banco para DataAccessException
@Controller   // controller MVC (retorna views)
@RestController // controller REST (retorna JSON/XML automaticamente)
```

### `@Bean` em `@Configuration`

Para criar beans de classes externas (que você não pode anotar com `@Component`):

```java
@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean(name = "emailAsync")
    public EmailService emailServiceAsync() {
        return new EmailService(true); // versão assíncrona
    }
}
```

### Scopes

```java
@Component
@Scope("singleton")  // padrão — uma instância para toda a aplicação
public class CacheService {}

@Component
@Scope("prototype") // nova instância a cada injeção
public class RelatorioBuilder {}

// Em aplicações web
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {} // nova instância por requisição HTTP

@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class CarrinhoCompras {} // nova instância por sessão HTTP
```

### `@Profile` e `@Conditional`

```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        // HikariCP com PostgreSQL real
    }
}

// Ativar um profile
// Via propriedade: spring.profiles.active=dev
// Via anotação no teste: @ActiveProfiles("dev")
```

---

## Spring Boot

### Por que Spring Boot?

| Sem Spring Boot | Com Spring Boot |
|---|---|
| Configurar web.xml | Nenhuma configuração XML |
| Fazer deploy em Tomcat externo | Tomcat embutido (fat jar) |
| Configurar DataSource manualmente | Auto-configuração detecta dependências |
| Gerenciar versões de dependências | BOM gerencia versões compatíveis |

### Estrutura de um projeto Spring Boot

```
src/
├── main/
│   ├── java/
│   │   └── com/exemplo/app/
│   │       ├── AppApplication.java       ← ponto de entrada
│   │       ├── config/                   ← configurações
│   │       ├── controller/               ← endpoints REST
│   │       ├── service/                  ← lógica de negócio
│   │       ├── repository/               ← acesso a dados
│   │       ├── model/                    ← entidades JPA
│   │       └── dto/                      ← Data Transfer Objects
│   └── resources/
│       ├── application.yml               ← configuração principal
│       ├── application-dev.yml           ← config de dev
│       ├── application-prod.yml          ← config de produção
│       └── db/migration/                 ← scripts Flyway
└── test/
    └── java/
```

### `@SpringBootApplication`

```java
// É equivalente a:
// @Configuration + @ComponentScan + @EnableAutoConfiguration
@SpringBootApplication
public class AppApplication {
    public static void main(String[] args) {
        SpringApplication.run(AppApplication.class, args);
    }
}
```

### Starters — o que cada um inclui

```xml
<!-- Web: Spring MVC + Tomcat + Jackson -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- JPA: Hibernate + Spring Data + JDBC -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Segurança: Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Testes: JUnit 5 + Mockito + AssertJ + MockMvc -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Validação: Hibernate Validator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- Actuator: endpoints de monitoramento -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- DevTools: hot reload -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

### `application.yml` — configuração principal

```yaml
spring:
  application:
    name: minha-api

  datasource:
    url: jdbc:postgresql://localhost:5432/appdb
    username: appuser
    password: apppass
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2

  jpa:
    hibernate:
      ddl-auto: validate       # nunca use create ou update em produção
    show-sql: false
    open-in-view: false        # importante: desligar para evitar lazy loading fora da transação

  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080
  error:
    include-message: always    # inclui mensagem de erro no response

logging:
  level:
    root: INFO
    com.exemplo.app: DEBUG
```

### Profiles com `application-dev.yml`

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb
    username: sa
    password:
  h2:
    console:
      enabled: true
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop

logging:
  level:
    com.exemplo.app: DEBUG
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DB_URL}            # variável de ambiente
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    hibernate:
      ddl-auto: validate

logging:
  level:
    root: WARN
    com.exemplo.app: INFO
```

```bash
# Ativar profile
java -jar app.jar --spring.profiles.active=prod

# Ou via variável de ambiente
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

### Virtual Threads no Spring Boot 3.2+

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

Com Virtual Threads habilitadas, o Spring Boot usa threads virtuais do Java 21 para processar requisições HTTP. O resultado é maior throughput em cargas de trabalho I/O-bound (banco, HTTP externo) sem mudança no código da aplicação.

---

## Spring MVC / REST

### Controller completo

```java
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.Valid;
import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/v1/produtos")
public class ProdutoController {

    private final ProdutoService service;

    public ProdutoController(ProdutoService service) {
        this.service = service;
    }

    @GetMapping
    public ResponseEntity<List<ProdutoResponse>> listar() {
        return ResponseEntity.ok(service.listarTodos());
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProdutoResponse> buscarPorId(@PathVariable Long id) {
        return ResponseEntity.ok(service.buscarPorId(id));
    }

    @GetMapping("/buscar")
    public ResponseEntity<List<ProdutoResponse>> buscarPorCategoria(
            @RequestParam String categoria,
            @RequestParam(defaultValue = "0") int pagina,
            @RequestParam(defaultValue = "10") int tamanho) {
        return ResponseEntity.ok(service.buscarPorCategoria(categoria, pagina, tamanho));
    }

    @PostMapping
    public ResponseEntity<ProdutoResponse> criar(
            @Valid @RequestBody ProdutoRequest request) {
        ProdutoResponse criado = service.criar(request);
        URI location = URI.create("/api/v1/produtos/" + criado.id());
        return ResponseEntity.created(location).body(criado);
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProdutoResponse> atualizar(
            @PathVariable Long id,
            @Valid @RequestBody ProdutoRequest request) {
        return ResponseEntity.ok(service.atualizar(id, request));
    }

    @PatchMapping("/{id}/estoque")
    public ResponseEntity<Void> atualizarEstoque(
            @PathVariable Long id,
            @RequestParam int quantidade) {
        service.atualizarEstoque(id, quantidade);
        return ResponseEntity.noContent().build();
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deletar(@PathVariable Long id) {
        service.deletar(id);
        return ResponseEntity.noContent().build();
    }
}
```

### DTOs com Bean Validation

```java
import jakarta.validation.constraints.*;

public record ProdutoRequest(

    @NotBlank(message = "Nome é obrigatório")
    @Size(min = 2, max = 100, message = "Nome deve ter entre 2 e 100 caracteres")
    String nome,

    @NotNull(message = "Preço é obrigatório")
    @DecimalMin(value = "0.01", message = "Preço deve ser maior que zero")
    BigDecimal preco,

    @NotBlank(message = "Categoria é obrigatória")
    String categoria,

    @Min(value = 0, message = "Estoque não pode ser negativo")
    int estoque,

    @Email(message = "Email do fornecedor inválido")
    String emailFornecedor

) {}

public record ProdutoResponse(
    Long id,
    String nome,
    BigDecimal preco,
    String categoria,
    int estoque,
    LocalDateTime criadoEm
) {}
```

### Tratamento global de erros com `@ControllerAdvice`

```java
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.bind.*;
import java.time.LocalDateTime;
import java.util.*;

@RestControllerAdvice
public class GlobalExceptionHandler {

    record ErroResponse(
        LocalDateTime timestamp,
        int status,
        String erro,
        String mensagem,
        Map<String, String> campos
    ) {}

    @ExceptionHandler(ProdutoNaoEncontradoException.class)
    public ResponseEntity<ErroResponse> handleProdutoNaoEncontrado(
            ProdutoNaoEncontradoException ex) {
        var erro = new ErroResponse(
                LocalDateTime.now(), 404, "Not Found", ex.getMessage(), null);
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(erro);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErroResponse> handleValidacao(
            MethodArgumentNotValidException ex) {
        Map<String, String> campos = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err ->
                campos.put(err.getField(), err.getDefaultMessage()));

        var erro = new ErroResponse(
                LocalDateTime.now(), 422, "Unprocessable Entity",
                "Dados inválidos", campos);
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(erro);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErroResponse> handleGenerico(Exception ex) {
        var erro = new ErroResponse(
                LocalDateTime.now(), 500, "Internal Server Error",
                "Erro interno", null);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(erro);
    }
}
```

### CORS

```java
// Por controller
@CrossOrigin(origins = "https://meusite.com")
@RestController
public class ProdutoController {}

// Global — via configuração
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://meusite.com", "https://admin.meusite.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

---

## Exemplo Completo — API REST de TODO List

### `pom.xml`

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.4</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Modelo e repositório em memória

```java
// Todo.java
public class Todo {
    private Long id;
    private String titulo;
    private boolean concluido;
    private LocalDateTime criadoEm;
    // construtores, getters, setters...
}

// TodoRepository.java (repositório em memória para o exemplo)
@Repository
public class TodoRepository {
    private final Map<Long, Todo> store = new ConcurrentHashMap<>();
    private final AtomicLong counter = new AtomicLong(0);

    public Todo save(Todo todo) {
        if (todo.getId() == null) {
            todo.setId(counter.incrementAndGet());
            todo.setCriadoEm(LocalDateTime.now());
        }
        store.put(todo.getId(), todo);
        return todo;
    }

    public Optional<Todo> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }

    public List<Todo> findAll() {
        return new ArrayList<>(store.values());
    }

    public boolean delete(Long id) {
        return store.remove(id) != null;
    }
}
```

### DTOs e Service

```java
// Request
public record TodoRequest(
    @NotBlank(message = "Título é obrigatório")
    @Size(max = 200) String titulo
) {}

// Response
public record TodoResponse(Long id, String titulo, boolean concluido, LocalDateTime criadoEm) {
    static TodoResponse de(Todo todo) {
        return new TodoResponse(todo.getId(), todo.getTitulo(),
                todo.isConcluido(), todo.getCriadoEm());
    }
}

// Service
@Service
public class TodoService {

    private final TodoRepository repository;

    public TodoService(TodoRepository repository) {
        this.repository = repository;
    }

    public List<TodoResponse> listarTodos() {
        return repository.findAll().stream()
                .map(TodoResponse::de)
                .toList();
    }

    public TodoResponse buscarPorId(Long id) {
        return repository.findById(id)
                .map(TodoResponse::de)
                .orElseThrow(() -> new TodoNaoEncontradoException(id));
    }

    public TodoResponse criar(TodoRequest request) {
        Todo todo = new Todo(null, request.titulo(), false, null);
        return TodoResponse.de(repository.save(todo));
    }

    public TodoResponse concluir(Long id) {
        Todo todo = repository.findById(id)
                .orElseThrow(() -> new TodoNaoEncontradoException(id));
        todo.setConcluido(true);
        return TodoResponse.de(repository.save(todo));
    }

    public void deletar(Long id) {
        if (!repository.delete(id)) {
            throw new TodoNaoEncontradoException(id);
        }
    }
}
```

### Controller

```java
@RestController
@RequestMapping("/api/v1/todos")
public class TodoController {

    private final TodoService service;

    public TodoController(TodoService service) {
        this.service = service;
    }

    @GetMapping
    public ResponseEntity<List<TodoResponse>> listar() {
        return ResponseEntity.ok(service.listarTodos());
    }

    @GetMapping("/{id}")
    public ResponseEntity<TodoResponse> buscar(@PathVariable Long id) {
        return ResponseEntity.ok(service.buscarPorId(id));
    }

    @PostMapping
    public ResponseEntity<TodoResponse> criar(@Valid @RequestBody TodoRequest request) {
        TodoResponse criado = service.criar(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(criado);
    }

    @PatchMapping("/{id}/concluir")
    public ResponseEntity<TodoResponse> concluir(@PathVariable Long id) {
        return ResponseEntity.ok(service.concluir(id));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deletar(@PathVariable Long id) {
        service.deletar(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Testando com curl

```bash
# Criar
curl -X POST http://localhost:8080/api/v1/todos \
  -H "Content-Type: application/json" \
  -d '{"titulo": "Estudar Spring Boot"}'

# Listar
curl http://localhost:8080/api/v1/todos

# Concluir
curl -X PATCH http://localhost:8080/api/v1/todos/1/concluir

# Deletar
curl -X DELETE http://localhost:8080/api/v1/todos/1
```
