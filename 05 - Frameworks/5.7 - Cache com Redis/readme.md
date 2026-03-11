# 5.7 — Cache com Redis

Um endpoint que busca os mesmos dados de configuração a cada requisição está desperdiçando latência de banco desnecessariamente. Um produto com milhares de acessos simultâneos num pico de Black Friday pode saturar o pool de conexões JDBC enquanto os dados raramente mudam. Cache de aplicação resolve esses problemas — e Redis é o padrão do ecossistema para isso.

---

## Dependências Maven

```xml
<!-- Spring Data Redis + cliente Lettuce (padrão) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- Spring Cache abstraction (necessária para @Cacheable, @CacheEvict) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

---

## Configuração

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      # password: sua-senha  # se o Redis tiver autenticação
      timeout: 2000ms   # timeout de conexão
      lettuce:
        pool:
          max-active: 10    # máximo de conexões ativas no pool
          max-idle: 5
          min-idle: 2
          max-wait: 1000ms  # tempo máximo aguardando conexão disponível no pool

  cache:
    type: redis
    redis:
      # TTL padrão para todos os caches que não especificam TTL próprio
      time-to-live: 10m
      # Prefixo adicionado a todas as chaves no Redis (evita colisões entre apps)
      key-prefix: "minhaapp::"
      use-key-prefix: true
      # Armazena o tipo da classe no JSON (necessário para desserializar corretamente)
      cache-null-values: false
```

```java
// Habilitar cache na aplicação
@SpringBootApplication
@EnableCaching
public class MinhaAplicacao {
    public static void main(String[] args) {
        SpringApplication.run(MinhaAplicacao.class, args);
    }
}
```

---

## Serialização com Jackson

Por padrão, o Spring Cache com Redis serializa objetos com Java Serialization, que produz bytes ilegíveis e é frágil a mudanças de classe. Configure Jackson para JSON:

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()
                )
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            )
            .disableCachingNullValues();
    }

    // Configurações específicas por cache (sobrescrevem o padrão)
    @Bean
    public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
        return builder -> builder
            // Cache de produtos: TTL de 30 minutos
            .withCacheConfiguration("produtos",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(30)))
            // Cache de configurações: TTL de 1 hora
            .withCacheConfiguration("configuracoes",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofHours(1)));
    }
}
```

---

## @Cacheable — cachear resultado de método

```java
@Service
public class ProdutoService {

    @Autowired
    private ProdutoRepository repository;

    // Na primeira chamada: executa o método e armazena o resultado no Redis
    // Nas chamadas seguintes com o mesmo 'id': retorna do cache, sem executar o método
    @Cacheable(value = "produtos", key = "#id")
    public ProdutoDTO buscarPorId(Long id) {
        log.info("Buscando produto {} do banco de dados", id);
        return repository.findById(id)
            .map(ProdutoDTO::from)
            .orElseThrow(() -> new ProdutoNaoEncontradoException(id));
    }

    // Chave composta
    @Cacheable(value = "produtos-por-categoria",
               key = "#categoria + ':' + #pagina + ':' + #tamanho")
    public Page<ProdutoDTO> listarPorCategoria(String categoria, int pagina, int tamanho) {
        return repository.findByCategoria(categoria, PageRequest.of(pagina, tamanho))
            .map(ProdutoDTO::from);
    }

    // Condicional: só cacheia se o resultado não for null
    @Cacheable(value = "produtos", key = "#id", unless = "#result == null")
    public ProdutoDTO buscarOuNull(Long id) {
        return repository.findById(id).map(ProdutoDTO::from).orElse(null);
    }

    // Condicional: só busca cache se o parâmetro satisfazer a condição
    // (ex: não cacheia buscas com filtros muito específicos de baixo reuso)
    @Cacheable(value = "produtos", key = "#id", condition = "#id > 0")
    public ProdutoDTO buscarCondicional(Long id) {
        return repository.findById(id).map(ProdutoDTO::from).orElseThrow();
    }
}
```

---

## @CacheEvict — invalidar cache

Quando um produto é atualizado ou deletado, o cache precisa ser invalidado. Servir dados antigos por minutos depois de uma atualização é um bug, não uma feature.

```java
@Service
public class ProdutoService {

    // Invalida a entrada específica do cache após atualizar
    @CacheEvict(value = "produtos", key = "#id")
    public ProdutoDTO atualizar(Long id, AtualizarProdutoRequest request) {
        var produto = repository.findById(id).orElseThrow();
        produto.atualizar(request);
        return ProdutoDTO.from(repository.save(produto));
    }

    // Invalida TODAS as entradas do cache "produtos"
    @CacheEvict(value = "produtos", allEntries = true)
    public void deletar(Long id) {
        repository.deleteById(id);
    }

    // Invalida múltiplos caches de uma vez
    @Caching(evict = {
        @CacheEvict(value = "produtos", key = "#id"),
        @CacheEvict(value = "produtos-por-categoria", allEntries = true)
    })
    public ProdutoDTO atualizarComCategoria(Long id, AtualizarProdutoRequest request) {
        var produto = repository.findById(id).orElseThrow();
        produto.atualizar(request);
        return ProdutoDTO.from(repository.save(produto));
    }
}
```

---

## @CachePut — atualizar cache sem invalidar

`@CachePut` sempre executa o método e atualiza o cache com o resultado — útil quando você quer que as leituras seguintes já encontrem o dado atualizado, sem precisar buscá-lo novamente.

```java
// Executa sempre (não é bypass de cache como @Cacheable)
// Atualiza o cache com o resultado
@CachePut(value = "produtos", key = "#result.id()")
public ProdutoDTO criar(CriarProdutoRequest request) {
    var produto = repository.save(Produto.from(request));
    return ProdutoDTO.from(produto);
}
```

---

## RedisTemplate — acesso de baixo nível

Para operações que vão além do cache de métodos — contadores, filas, expiração condicional, estruturas de dados Redis — use `RedisTemplate` diretamente:

```java
@Service
public class RateLimitService {

    private final RedisTemplate<String, String> redisTemplate;

    public RateLimitService(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // Rate limiting simples: máximo de 100 requisições por minuto por IP
    public boolean permitirRequisicao(String clienteIp) {
        String chave = "rate_limit:" + clienteIp;
        var ops = redisTemplate.opsForValue();

        Long count = ops.increment(chave);
        if (count == 1) {
            // Primeira requisição: define expiração de 1 minuto
            redisTemplate.expire(chave, Duration.ofMinutes(1));
        }

        return count != null && count <= 100;
    }

    // Cache manual com TTL customizado por entrada
    public void armazenarTemporariamente(String chave, String valor, Duration ttl) {
        redisTemplate.opsForValue().set("temp:" + chave, valor, ttl);
    }

    public Optional<String> buscarTemporario(String chave) {
        return Optional.ofNullable(redisTemplate.opsForValue().get("temp:" + chave));
    }

    // Verificar TTL restante de uma chave
    public Duration ttlRestante(String chave) {
        Long ttl = redisTemplate.getExpire(chave, TimeUnit.SECONDS);
        return ttl != null && ttl > 0 ? Duration.ofSeconds(ttl) : Duration.ZERO;
    }
}
```

---

## Redis com Testcontainers nos testes

Assim como com PostgreSQL, não use mocks do Redis nos testes de integração — o comportamento do `RedisTemplate` com operações de TTL, expiração e estruturas de dados pode variar entre implementações. Use um container real:

```java
@SpringBootTest
@Testcontainers
class ProdutoServiceCacheTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>(
            DockerImageName.parse("redis:7-alpine"))
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void configurar(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
    }

    @Autowired
    ProdutoService produtoService;

    @Autowired
    ProdutoRepository repository;

    @Test
    void deveCachearResultadoNaSegundaChamada() {
        var produto = repository.save(new Produto("Notebook", new BigDecimal("4999")));

        // Primeira chamada: vai ao banco
        var resultado1 = produtoService.buscarPorId(produto.getId());

        // Deleta do banco sem invalidar cache
        repository.delete(produto);

        // Segunda chamada: deve retornar do cache (o banco não tem mais)
        var resultado2 = produtoService.buscarPorId(produto.getId());

        assertThat(resultado2.nome()).isEqualTo(resultado1.nome());
    }

    @Test
    void deveInvalidarCacheAposAtualizar() {
        var produto = repository.save(new Produto("Notebook", new BigDecimal("4999")));
        produtoService.buscarPorId(produto.getId()); // popula cache

        produtoService.atualizar(produto.getId(),
            new AtualizarProdutoRequest("Notebook Pro", new BigDecimal("5999")));

        // Deve buscar do banco novamente com dados atualizados
        var atualizado = produtoService.buscarPorId(produto.getId());
        assertThat(atualizado.nome()).isEqualTo("Notebook Pro");
    }
}
```

---

## O que não cachear

Cache resolve o problema de leitura frequente de dados que raramente mudam. Não use cache para:

Dados que mudam a cada requisição (como carrinho de compras ativo — use sessão no Redis, não Spring Cache). Dados com requisitos de consistência imediata (saldo bancário, estoque em operações de debito). Resultados de operações que têm efeitos colaterais — `@Cacheable` em métodos que escrevem no banco é um bug. E nunca coloque objetos JPA gerenciados (entidades com proxies Hibernate) diretamente no cache — serializa o proxy, não o dado. Sempre mapeie para DTOs antes de cachear.
