# 3.4 — Concorrência Básica

Concorrência é um dos tópicos mais complexos em Java — e um dos mais importantes. Com o Java 21 e Virtual Threads, escrever código concorrente ficou mais simples, mas os conceitos fundamentais continuam essenciais para entender o que está acontecendo.

---

## Thread e Runnable

```java
// Forma 1: estender Thread (não recomendado — acopla lógica à thread)
public class ProcessadorDePedido extends Thread {
    private final Pedido pedido;

    public ProcessadorDePedido(Pedido pedido) {
        super("ProcessadorPedido-" + pedido.id());
        this.pedido = pedido;
    }

    @Override
    public void run() {
        System.out.println("Processando no thread: " + Thread.currentThread().getName());
        processar(pedido);
    }
}

// Uso
var thread = new ProcessadorDePedido(pedido);
thread.start();   // start() — chama run() em uma nova thread
                  // NUNCA chame run() diretamente — executa na thread atual

// Forma 2: implementar Runnable (preferível — separa lógica da mecânica)
Runnable tarefa = () -> {
    System.out.println("Executando em: " + Thread.currentThread().getName());
    processar(pedido);
};

Thread thread = new Thread(tarefa, "processador-pedido");
thread.start();

// Forma 3: lambda direta
new Thread(() -> processar(pedido)).start();
```

### sleep e join

```java
public class ExemplosThread {

    public static void main(String[] args) throws InterruptedException {

        // Thread.sleep() — pausa a thread atual (não libera locks!)
        System.out.println("Iniciando...");
        Thread.sleep(2000);   // pausa por 2 segundos
        System.out.println("Continuando após 2s");

        // join() — aguarda a thread terminar antes de continuar
        var thread1 = new Thread(() -> {
            try {
                Thread.sleep(1000);
                System.out.println("Thread 1 concluída");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();   // restaura o flag de interrupção
                System.out.println("Thread 1 interrompida");
            }
        });

        var thread2 = new Thread(() -> {
            try {
                Thread.sleep(2000);
                System.out.println("Thread 2 concluída");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        thread1.start();
        thread2.start();

        thread1.join();   // aguarda thread1 terminar
        thread2.join();   // aguarda thread2 terminar

        System.out.println("Ambas as threads concluídas");

        // join com timeout — não espera indefinidamente
        thread1.join(5000);   // espera no máximo 5 segundos
    }
}
```

> **Sempre trate `InterruptedException` corretamente:** chame `Thread.currentThread().interrupt()` para restaurar o flag de interrupção e permita que o código chamador saiba que houve interrupção.

---

## ExecutorService

Gerenciar threads manualmente é trabalhoso e propenso a erros. `ExecutorService` fornece um pool de threads reutilizáveis.

```java
// newFixedThreadPool — número fixo de threads
// Bom para: tarefas CPU-bound onde mais threads que CPUs não ajudam
ExecutorService fixo = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()   // número de CPUs
);

// newCachedThreadPool — cria threads conforme necessário, reutiliza ociosas
// Bom para: muitas tarefas curtas de I/O
// CUIDADO: pode criar threads ilimitadas — risco de OutOfMemoryError
ExecutorService cache = Executors.newCachedThreadPool();

// newSingleThreadExecutor — uma única thread, sequencial
// Bom para: garantir ordem de execução, acesso serial a recurso compartilhado
ExecutorService serial = Executors.newSingleThreadExecutor();

// newScheduledThreadPool — execução agendada ou periódica
ScheduledExecutorService agendado = Executors.newScheduledThreadPool(2);

// Submeter tarefas
ExecutorService executor = Executors.newFixedThreadPool(4);

// submit(Runnable) — sem retorno
executor.submit(() -> processar(pedido1));
executor.submit(() -> processar(pedido2));

// Encerrando o executor (SEMPRE feche o executor para não vazar threads)
executor.shutdown();    // aguarda tarefas em andamento terminarem
// executor.shutdownNow();   // tenta interromper as tarefas imediatamente

// try-with-resources com Java 19+ (ExecutorService implementa AutoCloseable)
try (var exec = Executors.newFixedThreadPool(4)) {
    for (var pedido : listaDePedidos) {
        exec.submit(() -> processar(pedido));
    }
}   // aguarda automaticamente ao sair do bloco
```

---

## Callable e Future

`Callable<V>` é como `Runnable`, mas retorna um resultado e pode lançar exceções.

```java
// Callable retorna resultado
Callable<BigDecimal> calculadora = () -> {
    // operação demorada
    return calcularTotal(pedidoId);
};

ExecutorService executor = Executors.newFixedThreadPool(4);

// submit(Callable) retorna Future<V>
Future<BigDecimal> futuro = executor.submit(calculadora);

// ... faça outras coisas enquanto o cálculo roda em paralelo ...

// get() bloqueia até o resultado estar disponível
BigDecimal total = futuro.get();            // bloqueia indefinidamente
BigDecimal totalComTimeout = futuro.get(5, TimeUnit.SECONDS);   // timeout

// Verificar estado do Future
boolean concluido   = futuro.isDone();
boolean cancelado   = futuro.isCancelled();
futuro.cancel(true);   // tenta cancelar (true = interrompe se estiver rodando)

// invokeAll — submete todos e aguarda todos terminarem
List<Callable<String>> tarefas = List.of(
    () -> buscarDadosServico1(),
    () -> buscarDadosServico2(),
    () -> buscarDadosServico3()
);

List<Future<String>> futuros = executor.invokeAll(tarefas);
List<String> resultados = futuros.stream()
    .map(f -> {
        try {
            return f.get();
        } catch (InterruptedException | ExecutionException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Falha na tarefa", e);
        }
    })
    .toList();

// invokeAny — retorna o resultado do mais rápido (cancela os outros)
String maisRapido = executor.invokeAny(tarefas);
```

---

## Problemas clássicos de concorrência

### Race Condition (Condição de Corrida)

```java
// PROBLEMA: duas threads lendo e escrevendo no mesmo campo sem sincronização
public class ContadorInseguro {
    private int contador = 0;

    public void incrementar() {
        contador++;   // NÃO é atômico! É read-modify-write em 3 passos
        // Thread A lê:  contador = 5
        // Thread B lê:  contador = 5   ← leu antes de A escrever
        // Thread A escreve: contador = 6
        // Thread B escreve: contador = 6   ← perdeu o incremento de A!
    }

    public int valor() { return contador; }
}

// Demonstração do problema:
var contador = new ContadorInseguro();
var executor = Executors.newFixedThreadPool(10);

for (int i = 0; i < 1000; i++) {
    executor.submit(contador::incrementar);
}

executor.shutdown();
executor.awaitTermination(5, TimeUnit.SECONDS);
System.out.println(contador.valor());   // Esperado: 1000. Real: ~950 (varia)
```

### Deadlock

```java
// PROBLEMA: duas threads esperando uma pela outra indefinidamente
Object lockA = new Object();
Object lockB = new Object();

Thread thread1 = new Thread(() -> {
    synchronized (lockA) {
        System.out.println("Thread 1 adquiriu lockA");
        try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        synchronized (lockB) {   // espera lockB que Thread 2 tem
            System.out.println("Thread 1 adquiriu lockB");
        }
    }
});

Thread thread2 = new Thread(() -> {
    synchronized (lockB) {
        System.out.println("Thread 2 adquiriu lockB");
        try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        synchronized (lockA) {   // espera lockA que Thread 1 tem → DEADLOCK
            System.out.println("Thread 2 adquiriu lockA");
        }
    }
});

// SOLUÇÃO: sempre adquirir locks na mesma ordem
Thread thread1Corrigida = new Thread(() -> {
    synchronized (lockA) {       // ordem: A → B
        synchronized (lockB) {
            executar();
        }
    }
});

Thread thread2Corrigida = new Thread(() -> {
    synchronized (lockA) {       // mesma ordem: A → B
        synchronized (lockB) {
            executar();
        }
    }
});
```

### Starvation e Livelock

```java
// Starvation: thread de baixa prioridade nunca consegue CPU
Thread altaPrioridade = new Thread(tarefaPesada);
altaPrioridade.setPriority(Thread.MAX_PRIORITY);   // 10

Thread baixaPrioridade = new Thread(tarefaLeve);
baixaPrioridade.setPriority(Thread.MIN_PRIORITY);   // 1
// baixaPrioridade pode nunca executar se altaPrioridade dominar o CPU

// Livelock: threads "educadas demais" — ficam cedendo uma para a outra
// Exemplo conceitual:
// Thread A: "Vou esperar Thread B terminar primeiro"
// Thread B: "Não, vou esperar Thread A terminar primeiro"
// Resultado: ambas ficam esperando, sem deadlock mas sem progresso
```

---

## synchronized — método e bloco

```java
// synchronized em método — lock é o objeto 'this' (métodos de instância)
public class ContadorSincronizado {
    private int contador = 0;

    // Lock: this (toda a instância)
    public synchronized void incrementar() {
        contador++;
    }

    public synchronized int valor() {
        return contador;
    }
}

// synchronized em método estático — lock é o objeto Class
public class Singleton {
    private static Singleton instancia;

    public static synchronized Singleton getInstance() {
        if (instancia == null) {
            instancia = new Singleton();
        }
        return instancia;
    }
}

// synchronized em bloco — granularidade fina, melhor performance
public class GerenciadorDeEstoque {
    private final Map<String, Integer> estoque = new HashMap<>();
    private final Object lockEstoque = new Object();   // lock dedicado

    public void atualizarQuantidade(String produto, int quantidade) {
        // Código sem sincronização (validações, log) roda fora do bloco
        if (quantidade < 0) {
            throw new IllegalArgumentException("Quantidade não pode ser negativa");
        }

        synchronized (lockEstoque) {
            // Somente a seção crítica está sincronizada
            int atual = estoque.getOrDefault(produto, 0);
            estoque.put(produto, atual + quantidade);
        }
        // Código pós-atualização roda fora do bloco
        log.info("Estoque atualizado: {} → {}", produto, quantidade);
    }

    // EVITE: synchronized em um método grande — degrada performance
    // A seção crítica deve ser a menor possível
}
```

---

## volatile — visibilidade entre threads

```java
// PROBLEMA: sem volatile, cada thread pode ter uma cópia em cache
// A mudança de uma thread pode não ser visível para outra
public class SemVolatile {
    private boolean rodando = true;   // pode ficar em cache da thread

    public void parar() {
        rodando = false;   // outra thread pode não ver isso
    }

    public void executar() {
        while (rodando) {   // pode rodar para sempre se rodando não sair do cache
            processar();
        }
    }
}

// SOLUÇÃO: volatile garante visibilidade — toda leitura vai à memória principal
public class ComVolatile {
    private volatile boolean rodando = true;   // sem cache

    public void parar() {
        rodando = false;   // visível imediatamente para todas as threads
    }

    public void executar() {
        while (rodando) {
            processar();
        }
    }
}

// LIMITAÇÃO do volatile:
// Garante visibilidade, mas NÃO garante atomicidade
// volatile int contador; contador++; ainda é race condition
// Para operações compostas, use synchronized ou classes Atomic
```

---

## Classes Atomic

As classes `java.util.concurrent.atomic` oferecem operações atômicas sem `synchronized`, usando instruções especiais do processador (CAS — Compare-And-Swap).

```java
// AtomicInteger — para contadores concorrentes
AtomicInteger contador = new AtomicInteger(0);

contador.incrementAndGet();   // equivalente a ++contador
contador.getAndIncrement();   // equivalente a contador++
contador.addAndGet(5);        // adiciona e retorna o novo valor
contador.getAndAdd(5);        // retorna o valor atual e adiciona
contador.compareAndSet(10, 20);   // só muda se o valor atual for 10

// AtomicLong — para IDs, timestamps
AtomicLong sequencia = new AtomicLong(0);
long proximoId = sequencia.incrementAndGet();

// AtomicBoolean — flags thread-safe
AtomicBoolean processando = new AtomicBoolean(false);

// Garante que apenas uma thread entre no bloco
if (processando.compareAndSet(false, true)) {
    try {
        executarProcessamentoUnico();
    } finally {
        processando.set(false);
    }
}

// AtomicReference — para qualquer objeto
AtomicReference<String> referencia = new AtomicReference<>("inicial");
referencia.set("novo valor");
referencia.compareAndSet("novo valor", "atualizado");

// LongAdder — melhor que AtomicLong para contadores com alta contenção
LongAdder visitas = new LongAdder();
visitas.increment();
long total = visitas.sum();

// Exemplo completo: contador thread-safe
public class EstatisticasDeRequisicao {

    private final AtomicLong totalRequisicoes = new AtomicLong(0);
    private final AtomicLong requisicoesFalhas = new AtomicLong(0);
    private final LongAdder tempoTotalMs = new LongAdder();

    public void registrarSucesso(long duracaoMs) {
        totalRequisicoes.incrementAndGet();
        tempoTotalMs.add(duracaoMs);
    }

    public void registrarFalha(long duracaoMs) {
        totalRequisicoes.incrementAndGet();
        requisicoesFalhas.incrementAndGet();
        tempoTotalMs.add(duracaoMs);
    }

    public double taxaDeFalhas() {
        long total = totalRequisicoes.get();
        return total == 0 ? 0.0 : (double) requisicoesFalhas.get() / total * 100;
    }

    public double tempoMedioMs() {
        long total = totalRequisicoes.get();
        return total == 0 ? 0.0 : (double) tempoTotalMs.sum() / total;
    }
}
```

---

## ReentrantLock — controle avançado de locks

```java
import java.util.concurrent.locks.ReentrantLock;

public class FilaDeImpressao {
    private final ReentrantLock lock = new ReentrantLock();
    private final List<Documento> fila = new ArrayList<>();

    public boolean adicionarComTimeout(Documento doc) throws InterruptedException {
        // Tenta adquirir o lock por até 2 segundos
        if (lock.tryLock(2, TimeUnit.SECONDS)) {
            try {
                fila.add(doc);
                return true;
            } finally {
                lock.unlock();   // SEMPRE no finally
            }
        }
        return false;   // não conseguiu o lock no tempo
    }

    public void processar() throws InterruptedException {
        lock.lockInterruptibly();   // pode ser interrompido enquanto espera
        try {
            if (!fila.isEmpty()) {
                imprimir(fila.remove(0));
            }
        } finally {
            lock.unlock();
        }
    }
}
```

---

## Virtual Threads (Java 21)

Virtual Threads são a revolução do Java 21 para concorrência de I/O.

### Carrier thread vs Virtual Thread

```
Threads de S.O. (Platform Threads / Carrier Threads):
- Gerenciadas pelo sistema operacional
- ~1MB de stack por thread
- Criação e troca de contexto são operações custosas
- Limite prático: alguns milhares por JVM

Virtual Threads:
- Gerenciadas pela JVM
- Stack minúscula e dinâmica (cresce conforme necessário)
- Mapeadas para um pool pequeno de carrier threads
- Quando bloqueia em I/O: a virtual thread é suspensa,
  a carrier thread fica livre para outra virtual thread
- Limite: milhões por JVM
```

### Criando Virtual Threads

```java
// Forma 1: Thread.ofVirtual()
Thread vt = Thread.ofVirtual()
    .name("vt-processamento-", 1)   // nome com contador
    .start(() -> processar(pedido));

vt.join();   // aguarda terminar

// Forma 2: ExecutorService — mais comum e idiomático
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {

    // Cada submit() cria uma nova virtual thread — custo quase zero
    var futures = pedidos.stream()
        .map(pedido -> executor.submit(() -> processarPedido(pedido)))
        .toList();

    // Coleta resultados
    var resultados = futures.stream()
        .map(f -> {
            try {
                return f.get(30, TimeUnit.SECONDS);
            } catch (Exception e) {
                log.error("Falha ao processar pedido", e);
                return ResultadoProcessamento.falha(e.getMessage());
            }
        })
        .toList();
}

// Forma 3: Structured Concurrency — tarefas relacionadas
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

    // Dispara 3 chamadas em paralelo
    var futuroPerfil    = scope.fork(() -> buscarPerfil(clienteId));
    var futuroHistorico = scope.fork(() -> buscarHistorico(clienteId));
    var futuroLimite    = scope.fork(() -> buscarLimiteCredito(clienteId));

    scope.join()           // aguarda todos terminarem
         .throwIfFailed(); // lança exceção se algum falhou

    // Se chegou aqui, todos tiveram sucesso
    return new DashboardCliente(
        futuroPerfil.get(),
        futuroHistorico.get(),
        futuroLimite.get()
    );
}
```

### Por que Virtual Threads são leves

```
Cenário: 1000 requisições HTTP simultâneas, cada uma demora 200ms (I/O)

Com Platform Threads (thread-per-request):
- 1000 threads de SO × 1MB = 1GB de memória só para stacks
- SO gerencia 1000 threads → overhead de troca de contexto alto
- Pode esgotar threads do pool e fazer requisições esperarem

Com Virtual Threads:
- 1000 virtual threads com apenas algumas KB de stack cada
- JVM mantém pool de 8 carrier threads (uma por CPU)
- Quando vt1 bloqueia em I/O, carrier thread aceita vt2
- Todas as 1000 virtual threads progridem sem desperdiçar CPU ou memória
```

### Quando usar Virtual Threads

```java
// IDEAL: qualquer operação bloqueante de I/O
// - Chamadas HTTP a APIs externas (RestTemplate, HttpClient)
// - Consultas ao banco de dados (JDBC, JPA)
// - Leitura/escrita de arquivos
// - Leitura/escrita de filas (Kafka, RabbitMQ)

@Service
public class RelatorioService {

    // Com Virtual Threads, cada chamada pode ser feita de forma simples
    // e bloqueante — sem Mono/Flux/CompletableFuture
    public RelatorioCompleto gerarRelatorio(UUID clienteId) {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            var fVendas    = scope.fork(() -> vendaRepository.buscarPorCliente(clienteId));
            var fPagamentos = scope.fork(() -> pagamentoClient.buscar(clienteId));
            var fCredito   = scope.fork(() -> creditoClient.buscarLimite(clienteId));

            scope.join().throwIfFailed();

            return new RelatorioCompleto(
                fVendas.get(),
                fPagamentos.get(),
                fCredito.get()
            );
        }
    }
}
```

### Quando NÃO usar Virtual Threads

```java
// NÃO USAR 1: operações CPU-bound (sem benefício)
// Virtual threads não aumentam o paralelismo de CPU
// Para isso, use ForkJoinPool ou parallelStream()
public BigDecimal calcularRisco(List<Ativo> ativos) {
    // CPU-bound: use paralelismo de platform threads
    return ativos.parallelStream()
        .map(this::calcularRiscoAtivo)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
}

// NÃO USAR 2: com synchronized contendo I/O — causa "pinning"
// Quando uma virtual thread executa synchronized e faz I/O,
// a carrier thread fica "pregada" (pinned) e não pode servir outras vts

// PROBLEMÁTICO — pinna a carrier thread durante o I/O dentro do synchronized
public synchronized void processarComSynchronized(UUID id) {
    var dado = httpClient.buscar(id);   // I/O dentro de synchronized = pinning
    salvar(dado);
}

// CORRETO — use ReentrantLock em vez de synchronized quando há I/O interno
private final ReentrantLock lock = new ReentrantLock();

public void processarComLock(UUID id) {
    var dado = httpClient.buscar(id);   // I/O fora do lock — sem pinning

    lock.lock();
    try {
        salvar(dado);                   // operação rápida dentro do lock
    } finally {
        lock.unlock();
    }
}

// NÃO USAR 3: para substituir reactive (Webflux) quando o objetivo é backpressure
// Se você precisa de controle de backpressure, throttling de fluxo,
// operadores reativos complexos — Reactor/RxJava ainda são mais adequados
```

### Habilitando Virtual Threads no Spring Boot 3.2+

```java
// application.properties — habilita virtual threads para todos os requests HTTP
spring.threads.virtual.enabled=true

// Isso configura automaticamente:
// - Tomcat para usar virtual threads por request
// - @Async para usar virtual threads
// - @Scheduled para usar virtual threads
```

```java
// Ou manualmente, se quiser controle total
@Configuration
public class VirtualThreadsConfig {

    @Bean
    public TomcatProtocolHandlerCustomizer<?> virtualThreadsParaTomcat() {
        return protocolHandler ->
            protocolHandler.setExecutor(
                Executors.newVirtualThreadPerTaskExecutor()
            );
    }

    @Bean
    public AsyncTaskExecutor applicationTaskExecutor() {
        return new TaskExecutorAdapter(
            Executors.newVirtualThreadPerTaskExecutor()
        );
    }
}
```

---

## Resumo: qual ferramenta usar

| Situação                                         | Ferramenta recomendada              |
|--------------------------------------------------|-------------------------------------|
| I/O simples, um resultado por task               | `ExecutorService` + Virtual Threads |
| Múltiplas tarefas I/O relacionadas em paralelo   | `StructuredTaskScope`               |
| Contador thread-safe                             | `AtomicInteger` / `LongAdder`       |
| Flag thread-safe                                 | `AtomicBoolean`                     |
| Mapa thread-safe de alta contenção               | `ConcurrentHashMap`                 |
| Seção crítica simples                            | `synchronized`                      |
| Seção crítica com timeout ou interrupção         | `ReentrantLock`                     |
| Processamento CPU-bound paralelo                 | `ForkJoinPool` / `parallelStream()` |
| Compartilhar dados imutáveis entre vthreads      | `ScopedValue`                       |
