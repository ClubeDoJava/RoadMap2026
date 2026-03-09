# 8.1 - Programação Reativa com Spring WebFlux

## Por que reativo? Imperativo vs Reativo

### Modelo imperativo (bloqueante)

```
Thread 1: ──[recebe req]──[aguarda DB]──────────────[processa]──[responde]──>
                                ↑
                         Thread bloqueada aqui
                         (consumindo memória, sem trabalhar)
```

No modelo imperativo, cada requisição ocupa uma thread durante toda a sua duração. Quando a thread espera I/O (banco, HTTP externo, arquivo), ela fica bloqueada mas ainda consumindo memória (~500KB por thread).

Com 1.000 requisições simultâneas: 1.000 threads × 500KB = 500MB só para gerenciar threads.

### Modelo reativo (não-bloqueante)

```
Thread 1: ──[recebe req]──[registra callback]──[processa outra req]──>
                                                      ↑
                                              Thread livre para trabalhar
                                              enquanto espera o DB

               [DB responde] → [callback executado em thread disponível]
```

No modelo reativo, a thread não fica bloqueada esperando I/O. Quando o banco responde, uma thread disponível executa o callback. Poucas threads conseguem processar muitas requisições simultâneas.

**Quando usar reativo:**
- Alta concorrência com I/O intensivo (muitas chamadas a bancos, APIs externas)
- Streaming de dados (SSE, WebSocket)
- Microsserviços que fazem muitas chamadas para outros serviços

**Quando ficar com MVC síncrono:**
- Lógica de negócio simples com poucos I/Os
- Time sem experiência em reativo (curva de aprendizado é real)
- CPU-bound (processamento intenso): reativo não ajuda aqui

---

## Project Reactor: Mono e Flux

**Mono:** 0 ou 1 elemento (como um `Optional` assíncrono)

```java
Mono<String> mono = Mono.just("Olá")
        .map(s -> s.toUpperCase())
        .doOnNext(s -> System.out.println("Valor: " + s));
// Nota: nada acontece até alguém se "inscrever" (subscribe)

// Mono vazio (equivale a Optional.empty())
Mono<String> vazio = Mono.empty();

// Mono de erro
Mono<String> erro = Mono.error(new RuntimeException("Algo falhou"));
```

**Flux:** 0 a N elementos (stream assíncrono)

```java
Flux<Integer> flux = Flux.range(1, 10)    // 1, 2, 3, ..., 10
        .filter(n -> n % 2 == 0)           // 2, 4, 6, 8, 10
        .map(n -> n * n);                   // 4, 16, 36, 64, 100

// Flux a partir de lista
Flux<String> frutas = Flux.fromIterable(List.of("maçã", "banana", "laranja"));

// Flux de intervalos (emite a cada segundo)
Flux<Long> tick = Flux.interval(Duration.ofSeconds(1));
```

### Operadores essenciais

```java
// ─── Transformação ─────────────────────────────────────────────
Flux<Integer> numeros = Flux.range(1, 5);

// map: transforma cada elemento (síncrono)
numeros.map(n -> n * 2)
       .subscribe(System.out::println); // 2, 4, 6, 8, 10

// flatMap: transforma e "achata" (útil para operações assíncronas)
Flux.just(1, 2, 3)
    .flatMap(n -> buscarDadosPorId(n))  // buscarDadosPorId retorna Mono
    .subscribe(System.out::println);

// flatMap executa em paralelo (sem ordem garantida)
// concatMap executa em sequência (com ordem garantida, mais lento)

// ─── Filtragem ──────────────────────────────────────────────────
numeros.filter(n -> n > 2)
       .subscribe(System.out::println); // 3, 4, 5

// take: pega apenas os primeiros N
numeros.take(3).subscribe(System.out::println); // 1, 2, 3

// ─── Combinação ─────────────────────────────────────────────────
Mono<String> usuario = buscarUsuario(1L);
Mono<String> endereco = buscarEndereco(1L);

// zip: combina dois Monos em um par (espera ambos)
Mono.zip(usuario, endereco)
    .map(tuple -> tuple.getT1() + " mora em " + tuple.getT2())
    .subscribe(System.out::println);

// merge: combina dois Flux sem ordem garantida (mais rápido)
Flux.merge(
    Flux.just("A", "B").delayElements(Duration.ofMillis(100)),
    Flux.just("1", "2").delayElements(Duration.ofMillis(150))
).subscribe(System.out::println); // A, 1, B, 2 (ordem por tempo)

// ─── Tratamento de erros ────────────────────────────────────────
buscarProduto(id)
    .onErrorReturn(new Produto("default")) // valor padrão em caso de erro
    .subscribe(System.out::println);

buscarProduto(id)
    .onErrorResume(e -> {
        log.error("Erro ao buscar produto: {}", e.getMessage());
        return Mono.just(new Produto("fallback"));
    })
    .subscribe(System.out::println);

// retry: tenta novamente em caso de erro
buscarProduto(id)
    .retry(3)  // tenta até 3 vezes
    .subscribe(System.out::println);

// ─── Logs para debug ────────────────────────────────────────────
Flux.range(1, 3)
    .log()  // Loga todos os eventos do pipeline (subscribe, next, complete)
    .map(n -> n * 2)
    .subscribe(System.out::println);
```

---

## Spring WebFlux vs Spring MVC

### Dependência

```xml
<!-- WebFlux (reativo) — substitui spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- R2DBC para banco de dados reativo -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

### Controller reativo

```java
// Anotações idênticas ao MVC, mas retorna Mono/Flux
@RestController
@RequestMapping("/api/produtos")
@RequiredArgsConstructor
public class ProdutoController {

    private final ProdutoService service;

    // Retorna Mono para um único recurso
    @GetMapping("/{id}")
    public Mono<ResponseEntity<ProdutoResponse>> buscarPorId(@PathVariable Long id) {
        return service.buscarPorId(id)
                .map(ResponseEntity::ok)
                .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // Retorna Flux para coleção
    @GetMapping
    public Flux<ProdutoResponse> listarTodos(
            @RequestParam(defaultValue = "0") int pagina,
            @RequestParam(defaultValue = "20") int tamanho) {
        return service.listarTodos(pagina, tamanho);
    }

    // Criação com POST
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<ProdutoResponse> criar(@RequestBody @Validated CriarProdutoRequest request) {
        return service.criar(request);
    }

    // Server-Sent Events: streaming em tempo real
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ProdutoResponse> stream() {
        return service.listarTodos()
                .delayElements(Duration.ofMillis(500)); // 1 produto a cada 500ms
    }
}
```

---

## R2DBC: banco de dados reativo

R2DBC (Reactive Relational Database Connectivity) é a alternativa não-bloqueante ao JDBC.

### Configuração

```yaml
# application.yml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/minhadb
    username: appuser
    password: senha123
    pool:
      max-size: 10
      initial-size: 5

  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
```

### Entidade e Repository

```java
// Entidade (não usa JPA, mas Spring Data R2DBC)
@Table("produtos")
public class Produto {
    @Id
    private Long id;
    private String nome;
    private BigDecimal preco;
    private String categoria;

    @Column("criado_em")
    private LocalDateTime criadoEm;

    @Column("ativo")
    private boolean ativo = true;
}

// Repository reativo
public interface ProdutoRepository extends ReactiveCrudRepository<Produto, Long> {

    // Derivação de query automática (funciona igual ao JPA)
    Flux<Produto> findByCategoria(String categoria);
    Flux<Produto> findByAtivoTrue();
    Mono<Produto> findByNome(String nome);

    // Query customizada
    @Query("SELECT * FROM produtos WHERE preco BETWEEN :min AND :max AND ativo = true")
    Flux<Produto> findByPrecoBetween(BigDecimal min, BigDecimal max);

    // Paginação
    Flux<Produto> findAllByOrderByNomeAsc(Pageable pageable);
}
```

### Serviço reativo

```java
@Service
@RequiredArgsConstructor
public class ProdutoService {

    private final ProdutoRepository repository;
    private final ProdutoMapper mapper;

    @Transactional(readOnly = true)
    public Mono<ProdutoResponse> buscarPorId(Long id) {
        return repository.findById(id)
                .map(mapper::toResponse)
                .switchIfEmpty(Mono.error(
                    new ProdutoNaoEncontradoException("Produto " + id + " não encontrado")
                ));
    }

    @Transactional(readOnly = true)
    public Flux<ProdutoResponse> listarTodos(int pagina, int tamanho) {
        return repository.findAllByOrderByNomeAsc(PageRequest.of(pagina, tamanho))
                .map(mapper::toResponse);
    }

    @Transactional
    public Mono<ProdutoResponse> criar(CriarProdutoRequest request) {
        Produto produto = mapper.toEntity(request);
        produto.setCriadoEm(LocalDateTime.now());

        return repository.save(produto)
                .map(mapper::toResponse);
    }

    // Combinando múltiplas fontes reativas
    public Mono<ProdutoComEstoqueResponse> buscarComEstoque(Long id) {
        Mono<Produto> produtoMono = repository.findById(id)
                .switchIfEmpty(Mono.error(new ProdutoNaoEncontradoException(id)));

        Mono<Integer> estoqueMono = estoqueClient.buscarEstoque(id)
                .onErrorReturn(0); // Se estoque falhar, retorna 0

        return Mono.zip(produtoMono, estoqueMono)
                .map(tuple -> new ProdutoComEstoqueResponse(
                        mapper.toResponse(tuple.getT1()),
                        tuple.getT2()
                ));
    }
}
```

---

## Backpressure

Backpressure é o mecanismo pelo qual um consumidor sinaliza ao produtor que está sobrecarregado e precisa que ele desacelere ou descarte dados.

```
Produtor          Consumidor
  │                   │
  │───[1000 msg/s]───>│ ← Consumidor processa apenas 100/s
  │                   │ ← Backpressure: "Me manda só 100/s"
  │────[100 msg/s]───>│
  │                   │ OK!
```

```java
// Estratégias de backpressure no Project Reactor

// BUFFER: armazena em buffer (cuidado: pode gerar OutOfMemoryError)
Flux.range(1, 10000)
    .onBackpressureBuffer(1000)  // Buffer de até 1000 elementos
    .subscribe(n -> processar(n));

// DROP: descarta elementos quando o consumidor não acompanha
Flux.range(1, 10000)
    .onBackpressureBuffer(100, DroppedElementException::new, BufferOverflowStrategy.DROP_OLDEST)
    .subscribe(n -> processar(n));

// LATEST: mantém apenas o último elemento quando sobrecarregado
Flux.range(1, 10000)
    .onBackpressureLatest()
    .subscribe(n -> processar(n));

// Controle explícito via BaseSubscriber
Flux.range(1, 100)
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            request(10); // Solicita 10 elementos por vez
        }

        @Override
        protected void hookOnNext(Integer value) {
            processar(value);
            if (value % 10 == 0) {
                request(10); // Solicita mais 10 quando terminar os 10 anteriores
            }
        }
    });
```

---

## Exemplo completo: API reativa com WebFlux e R2DBC

### Schema do banco

```sql
-- schema.sql
CREATE TABLE IF NOT EXISTS produtos (
    id BIGSERIAL PRIMARY KEY,
    nome VARCHAR(255) NOT NULL,
    preco DECIMAL(10,2) NOT NULL,
    categoria VARCHAR(100),
    ativo BOOLEAN DEFAULT true,
    criado_em TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_produtos_categoria ON produtos(categoria);
CREATE INDEX idx_produtos_ativo ON produtos(ativo);
```

### Controller completo

```java
@RestController
@RequestMapping("/api/produtos")
@RequiredArgsConstructor
@Slf4j
public class ProdutoController {

    private final ProdutoService service;

    @GetMapping
    public Flux<ProdutoResponse> listar(
            @RequestParam Optional<String> categoria,
            @RequestParam(defaultValue = "0") int pagina,
            @RequestParam(defaultValue = "20") int tamanho) {

        return categoria
                .map(cat -> service.listarPorCategoria(cat))
                .orElseGet(() -> service.listarTodos(pagina, tamanho));
    }

    @GetMapping("/{id}")
    public Mono<ResponseEntity<ProdutoResponse>> buscar(@PathVariable Long id) {
        return service.buscarPorId(id)
                .map(ResponseEntity::ok)
                .onErrorResume(ProdutoNaoEncontradoException.class,
                    e -> Mono.just(ResponseEntity.notFound().build()));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<ProdutoResponse> criar(@RequestBody @Validated CriarProdutoRequest request) {
        return service.criar(request)
                .doOnSuccess(p -> log.info("Produto criado: {}", p.id()));
    }

    @PutMapping("/{id}")
    public Mono<ResponseEntity<ProdutoResponse>> atualizar(
            @PathVariable Long id,
            @RequestBody @Validated AtualizarProdutoRequest request) {
        return service.atualizar(id, request)
                .map(ResponseEntity::ok)
                .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public Mono<Void> deletar(@PathVariable Long id) {
        return service.deletar(id);
    }

    // Streaming: útil para dashboards em tempo real
    @GetMapping(value = "/eventos", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<ProdutoResponse>> streamEventos() {
        return service.listarTodos()
                .map(produto -> ServerSentEvent.<ProdutoResponse>builder()
                        .id(produto.id().toString())
                        .event("produto")
                        .data(produto)
                        .build());
    }
}
```

### Teste do controller reativo

```java
@WebFluxTest(ProdutoController.class)
class ProdutoControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockBean
    private ProdutoService service;

    @Test
    void deveRetornarProdutoQuandoEncontrado() {
        ProdutoResponse produto = new ProdutoResponse(1L, "Notebook", new BigDecimal("3500.00"), "Eletrônicos");
        when(service.buscarPorId(1L)).thenReturn(Mono.just(produto));

        webTestClient.get()
                .uri("/api/produtos/1")
                .exchange()
                .expectStatus().isOk()
                .expectBody(ProdutoResponse.class)
                .isEqualTo(produto);
    }

    @Test
    void deveRetornar404QuandoNaoEncontrado() {
        when(service.buscarPorId(99L))
                .thenReturn(Mono.error(new ProdutoNaoEncontradoException(99L)));

        webTestClient.get()
                .uri("/api/produtos/99")
                .exchange()
                .expectStatus().isNotFound();
    }

    @Test
    void deveRetornarListaDeProdutos() {
        List<ProdutoResponse> produtos = List.of(
                new ProdutoResponse(1L, "Notebook", new BigDecimal("3500.00"), "Eletrônicos"),
                new ProdutoResponse(2L, "Mouse", new BigDecimal("150.00"), "Eletrônicos")
        );
        when(service.listarTodos(0, 20)).thenReturn(Flux.fromIterable(produtos));

        webTestClient.get()
                .uri("/api/produtos")
                .exchange()
                .expectStatus().isOk()
                .expectBodyList(ProdutoResponse.class)
                .hasSize(2);
    }
}
```
