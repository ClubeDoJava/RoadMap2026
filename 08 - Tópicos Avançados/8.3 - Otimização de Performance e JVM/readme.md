# 8.3 - Otimização de Performance e JVM

## Como a JVM funciona

```
┌─────────────────────────────────────────────────────────────────┐
│                    JVM (Java Virtual Machine)                   │
│                                                                 │
│  ┌──────────────┐    ┌────────────────────────────────────┐    │
│  │  ClassLoader │    │         Runtime Data Areas          │    │
│  │              │    │  ┌──────────┐  ┌─────────────────┐ │    │
│  │  Bootstrap   │    │  │  Heap    │  │   Stack (Thread) │ │    │
│  │  Extension   │───>│  │  Young   │  │   Frame          │ │    │
│  │  Application │    │  │  Old Gen │  │   Local vars     │ │    │
│  └──────────────┘    │  └──────────┘  └─────────────────┘ │    │
│                      │  ┌──────────┐  ┌─────────────────┐ │    │
│  ┌──────────────┐    │  │ Metaspace│  │   PC Register   │ │    │
│  │  Execution   │    │  │ (classes)│  │   Native Stack  │ │    │
│  │  Engine      │    │  └──────────┘  └─────────────────┘ │    │
│  │  Interpreter │    └────────────────────────────────────┘    │
│  │  JIT Compiler│                                               │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
```

### ClassLoader

Carrega classes `.class` (bytecode) para a memória sob demanda. Hierarquia:
- **Bootstrap:** carrega classes do próprio Java (`java.lang`, `java.util`)
- **Extension:** carrega extensões do JDK
- **Application:** carrega as classes da sua aplicação

### Heap (onde vivem os objetos)

```
Young Generation              Old Generation (Tenured)
┌──────────────────────────┐  ┌─────────────────────────────────┐
│  Eden Space              │  │                                 │
│  (novos objetos)         │  │  Objetos que sobreviveram       │
│  ──────────────────────  │  │  múltiplos GCs                  │
│  Survivor 0  Survivor 1  │  │                                 │
└──────────────────────────┘  └─────────────────────────────────┘
        Minor GC (frequente)          Major GC / Full GC (raro, lento)
```

**Ciclo de vida de um objeto:**
1. Nasce no **Eden Space**
2. Sobrevive ao Minor GC → vai para **Survivor Space**
3. Sobrevive a N GCs (threshold) → promovido para **Old Generation**
4. Não tem mais referências → coletado pelo GC

### Stack (onde vivem os dados de execução)

Cada thread tem sua própria stack. Cada chamada de método cria um **frame** com:
- Variáveis locais
- Operandos parciais da expressão atual
- Referência para o pool de constantes da classe

`StackOverflowError` ocorre quando a stack enche (geralmente por recursão infinita).

### Metaspace

Armazena metadados das classes (estrutura, métodos, campos). Antes era `PermGen` com tamanho fixo — era comum `OutOfMemoryError: PermGen space`. Desde Java 8, Metaspace usa memória nativa e cresce automaticamente.

---

## Garbage Collectors

### Serial GC

Usa uma única thread para GC. Para tudo durante a coleta ("stop-the-world").

```bash
java -XX:+UseSerialGC -jar app.jar
# Uso: aplicações pequenas, ambientes com uma CPU
```

### Parallel GC

Usa múltiplas threads para GC, mas ainda para a aplicação durante a coleta.

```bash
java -XX:+UseParallelGC -XX:ParallelGCThreads=4 -jar app.jar
# Uso: aplicações batch que priorizam throughput
```

### G1 GC (padrão desde Java 9)

Divide o heap em regiões. Tenta atingir um objetivo de pausa configurável. Bom equilíbrio entre throughput e latência.

```bash
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar
# Uso: aplicações de propósito geral (padrão)
```

### ZGC (Java 15+ stable)

Pausa máxima de 1ms, independente do tamanho do heap. Ideal para aplicações com latência crítica ou heaps gigantes (TB de RAM).

```bash
java -XX:+UseZGC -jar app.jar
# Uso: aplicações com SLA de latência rigoroso
```

### Shenandoah

Similar ao ZGC, desenvolvido pela Red Hat. Pausas muito curtas (sub-millisecond).

```bash
java -XX:+UseShenandoahGC -jar app.jar
# Uso: similar ao ZGC
```

### Comparativo

| GC | Throughput | Latência | Memória | Uso recomendado |
|----|-----------|---------|---------|-----------------|
| Serial | Baixo | Alta | Baixo | Apps pequenas |
| Parallel | Alto | Alta | Médio | Batch processing |
| G1 | Bom | Média | Médio | Padrão, apps web |
| ZGC | Bom | Muito baixa | Alto | Latência crítica |
| Shenandoah | Bom | Muito baixa | Alto | Latência crítica |

---

## Configurações JVM essenciais

```bash
# ─── Tamanho do Heap ───────────────────────────────────────────
-Xms512m         # Heap mínimo (inicial)
-Xmx2g           # Heap máximo
# Dica: Xms = Xmx evita redimensionamento do heap

# ─── Em containers (melhor que -Xmx fixo) ─────────────────────
-XX:MaxRAMPercentage=75.0    # Usa 75% da RAM do container
-XX:InitialRAMPercentage=50.0

# ─── Garbage Collector ─────────────────────────────────────────
-XX:+UseG1GC                 # G1 (padrão Java 9+)
-XX:+UseZGC                  # ZGC (baixa latência)
-XX:MaxGCPauseMillis=200     # Objetivo de pausa máxima (G1)

# ─── GC Logging (para análise) ─────────────────────────────────
-Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=20m

# ─── JIT ───────────────────────────────────────────────────────
-XX:+TieredCompilation       # Habilitado por padrão
-XX:CompileThreshold=10000   # Métodos compilados pelo JIT após 10000 invocações

# ─── Diagnóstico ───────────────────────────────────────────────
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof
-XX:+PrintGCDetails          # Detalhes do GC no stdout

# ─── Performance ───────────────────────────────────────────────
-Djava.security.egd=file:/dev/./urandom  # Evita bloqueio em geração de números aleatórios
-server                                   # Otimizações para servidor (padrão em 64-bit)
```

### Configuração para Spring Boot em produção (Docker)

```dockerfile
ENV JAVA_OPTS="\
  -XX:+UseZGC \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -Djava.security.egd=file:/dev/./urandom \
  -Dspring.profiles.active=prod"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## VisualVM: Profiler e análise

VisualVM é uma ferramenta gratuita para monitorar e analisar a JVM em tempo real.

### Instalação

```bash
# Download
https://visualvm.github.io/download.html

# Ou via SDK Man
sdk install visualvm
```

### Como conectar

1. Inicie sua aplicação com JMX habilitado:

```bash
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9010 \
     -Dcom.sun.management.jmxremote.ssl=false \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -jar app.jar
```

2. Abra o VisualVM
3. No painel esquerdo, sua aplicação aparece automaticamente se local
4. Para remota: `File → Add JMX Connection` → `host:9010`

### Funcionalidades principais

**Monitor:** CPU, Heap, Threads em tempo real

**Sampler:** veja quais métodos consomem mais CPU sem overhead

**Profiler:** análise mais detalhada, mas com mais overhead

**Heap Dump:** tire um snapshot do heap e analise:
- Quais classes têm mais instâncias
- O que está impedindo o GC de coletar

**Thread Dump:** captura o estado de todas as threads — essencial para diagnosticar deadlocks

---

## JMH — Java Microbenchmark Harness

JMH é a ferramenta oficial para escrever benchmarks corretos em Java. Mede nanossegundos com precisão, evitando armadilhas como JIT warm-up e dead code elimination.

### Dependência

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.37</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.37</version>
    <scope>test</scope>
</dependency>
```

### Escrevendo benchmarks corretos

```java
@BenchmarkMode(Mode.AverageTime)       // Tempo médio por operação
@OutputTimeUnit(TimeUnit.NANOSECONDS)  // Unidade de saída
@State(Scope.Benchmark)               // Estado compartilhado entre iterações
@Warmup(iterations = 5, time = 1)     // 5 iterações de warm-up (descartadas)
@Measurement(iterations = 10, time = 1) // 10 iterações de medição
@Fork(2)                              // Roda em 2 JVMs separadas (evita viés)
public class StringConcatenationBenchmark {

    @Param({"10", "100", "1000"})
    private int tamanho;              // Parametrização automática

    private List<String> palavras;

    @Setup                            // Executado antes dos benchmarks
    public void setup() {
        palavras = new ArrayList<>();
        for (int i = 0; i < tamanho; i++) {
            palavras.add("palavra" + i);
        }
    }

    @Benchmark
    public String concatenacaoNaive() {
        String resultado = "";
        for (String p : palavras) {
            resultado += p;            // O(n²) — realoca a cada iteração
        }
        return resultado;
    }

    @Benchmark
    public String comStringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (String p : palavras) {
            sb.append(p);              // O(n) — amortizado
        }
        return sb.toString();
    }

    @Benchmark
    public String comStringJoin() {
        return String.join("", palavras); // API mais legível, mesmo desempenho do StringBuilder
    }
}
```

```bash
# Executar os benchmarks
mvn package
java -jar target/benchmarks.jar

# Executar benchmark específico
java -jar target/benchmarks.jar StringConcatenationBenchmark

# Saída típica:
# Benchmark                                  (tamanho)  Mode  Cnt      Score  Units
# StringConcatenationBenchmark.concatenacaoNaive    100  avgt   20   8423.2  ns/op
# StringConcatenationBenchmark.comStringBuilder     100  avgt   20     82.1  ns/op  ← 100x mais rápido!
# StringConcatenationBenchmark.comStringJoin        100  avgt   20     80.3  ns/op
```

---

## Problemas comuns de performance

### Memory Leak — objetos que não são coletados

```java
// PROBLEMA: cache estático cresce indefinidamente
public class CacheProblematico {
    private static final Map<String, byte[]> cache = new HashMap<>();

    public byte[] getImagem(String url) {
        return cache.computeIfAbsent(url, this::baixarImagem); // Cache sem limite!
    }
}

// SOLUÇÃO: usar WeakReference ou Caffeine com tamanho máximo
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.Cache;

public class CacheCorreto {
    private final Cache<String, byte[]> cache = Caffeine.newBuilder()
            .maximumSize(1000)                    // Tamanho máximo
            .expireAfterAccess(10, TimeUnit.MINUTES) // TTL
            .build();

    public byte[] getImagem(String url) {
        return cache.get(url, this::baixarImagem);
    }
}
```

### N+1 Queries — o assassino silencioso de performance

```java
// PROBLEMA: 1 query para buscar pedidos + N queries para buscar itens
@GetMapping("/pedidos")
public List<PedidoResponse> listar() {
    List<Pedido> pedidos = pedidoRepository.findAll(); // 1 query
    return pedidos.stream()
            .map(p -> {
                List<Item> itens = itemRepository.findByPedidoId(p.getId()); // N queries!
                return new PedidoResponse(p, itens);
            })
            .toList();
}

// SOLUÇÃO 1: JOIN FETCH no JPQL
@Query("SELECT p FROM Pedido p JOIN FETCH p.itens WHERE p.status = :status")
List<Pedido> findByStatusComItens(@Param("status") String status);

// SOLUÇÃO 2: EntityGraph (mais flexível)
@EntityGraph(attributePaths = {"itens", "cliente"})
List<Pedido> findAll();

// SOLUÇÃO 3: Ativar log de SQL para detectar em desenvolvimento
# application.yml
spring.jpa.show-sql: true
logging.level.org.hibernate.SQL: DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

### Boxing/Unboxing desnecessário

```java
// PROBLEMA: auto-boxing cria objetos Integer desnecessariamente
List<Integer> numeros = new ArrayList<>();
long soma = 0;
for (int i = 0; i < 1_000_000; i++) {
    numeros.add(i);        // boxing: int → Integer (cria objeto)
    soma += numeros.get(i); // unboxing: Integer → int
}

// SOLUÇÃO: usar tipos primitivos com coleções especializadas
// Opção 1: IntStream (para operações encadeadas)
long soma2 = IntStream.range(0, 1_000_000).asLongStream().sum();

// Opção 2: int[] (sem overhead de boxing)
int[] numeros2 = new int[1_000_000];
long soma3 = 0;
for (int i = 0; i < numeros2.length; i++) {
    numeros2[i] = i;
    soma3 += numeros2[i]; // sem boxing
}

// Opção 3: biblioteca Eclipse Collections (primitives collections)
// LongList, IntList etc. sem boxing
```

### String concatenation em loop

```java
// PROBLEMA: O(n²) — cria nova String a cada iteração
String resultado = "";
for (String linha : milharesDeLinha) {
    resultado += linha + "\n"; // Ruim!
}

// SOLUÇÃO: StringBuilder — O(n)
StringBuilder sb = new StringBuilder(milharesDeLinha.size() * 100); // capacidade inicial
for (String linha : milharesDeLinha) {
    sb.append(linha).append('\n'); // Use char em vez de String de 1 char
}
String resultado2 = sb.toString();

// Nota: dentro de um único statement, o Java otimiza automaticamente
// "A" + "B" + "C" → o compilador usa StringBuilder internamente
// O problema só existe em LOOPS
```

---

## GraalVM AOT — compilação ahead-of-time

GraalVM Native Image compila o bytecode Java para um executável nativo específico do SO, sem precisar de JVM em runtime.

```
Compilação JIT (tradicional):    Compilação AOT (Native Image):
código fonte                     código fonte
    ↓ javac                          ↓ javac
  bytecode (.class)               bytecode (.class)
    ↓ JVM                            ↓ native-image
  interpretado → JIT             binário nativo (ELF/Mach-O/PE)
  (aquece com o tempo)           (executa direto no OS)
```

### Build com Maven

```xml
<!-- pom.xml: adicionar perfil para build nativo -->
<profiles>
    <profile>
        <id>native</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.graalvm.buildtools</groupId>
                    <artifactId>native-maven-plugin</artifactId>
                    <extensions>true</extensions>
                    <executions>
                        <execution>
                            <id>build-native</id>
                            <goals><goal>compile-no-fork</goal></goals>
                            <phase>package</phase>
                        </execution>
                    </executions>
                    <configuration>
                        <imageName>minha-app</imageName>
                        <buildArgs>
                            <buildArg>--no-fallback</buildArg>
                            <buildArg>-O2</buildArg>
                        </buildArgs>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

```bash
# Build nativo (requer GraalVM instalado)
./mvnw package -Pnative -DskipTests

# Executar
./target/minha-app

# Comparação de startup:
# JVM:    Started MinhaAppApplication in 3.241 seconds
# Native: Started MinhaAppApplication in 0.048 seconds ← 67x mais rápido!
```

---

## Dicas práticas de otimização

### Antes/depois: evite criar objetos desnecessariamente

```java
// ANTES: cria String intermediária a cada chamada
public boolean isAdmin(String role) {
    return role.toUpperCase().equals("ADMIN"); // cria nova String
}

// DEPOIS: sem alocação
public boolean isAdmin(String role) {
    return "ADMIN".equalsIgnoreCase(role); // sem nova String
}
```

### Antes/depois: use List.copyOf para coleções imutáveis

```java
// ANTES: Collections.unmodifiableList ainda permite modificar a lista original
List<String> original = new ArrayList<>(List.of("a", "b", "c"));
List<String> imutavel = Collections.unmodifiableList(original);
original.add("d"); // imutavel também muda!

// DEPOIS: List.copyOf faz cópia real e é verdadeiramente imutável
List<String> imutavel2 = List.copyOf(original); // cópia independente
```

### Antes/depois: streams paralelos com cuidado

```java
// ANTES: parallelStream com operação IO-bound (causa starvation do ForkJoinPool)
List<Produto> produtos = ids.parallelStream()
        .map(id -> repository.findById(id)) // IO: bloqueia threads do ForkJoinPool!
        .toList();

// DEPOIS: para IO-bound, use CompletableFuture com executor próprio
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
List<Produto> produtos2 = ids.stream()
        .map(id -> CompletableFuture.supplyAsync(() -> repository.findById(id), executor))
        .map(CompletableFuture::join)
        .toList();

// parallelStream é bom para: operações CPU-bound sem IO
List<Integer> resultados = numeros.parallelStream()
        .map(n -> calcularComplexo(n)) // CPU-bound: OK com parallelStream
        .toList();
```

### Antes/depois: inicialização lazy

```java
// ANTES: calcula tudo no construtor, mesmo que nunca use
public class RelatorioService {
    private final List<Configuracao> configuracoes;

    public RelatorioService() {
        this.configuracoes = carregarTodasConfiguracoes(); // Sempre executado
    }
}

// DEPOIS: inicializa sob demanda
public class RelatorioService {
    private List<Configuracao> configuracoes;

    public List<Configuracao> getConfiguracoes() {
        if (configuracoes == null) {
            configuracoes = carregarTodasConfiguracoes(); // Só quando precisar
        }
        return configuracoes;
    }
}
```
