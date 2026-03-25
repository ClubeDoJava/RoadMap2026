# 8.2 - Concorrência Avançada

## CompletableFuture

`CompletableFuture` é a API de programação assíncrona do Java, introduzida no Java 8. Permite encadear operações assíncronas sem bloquear threads.

### Operações básicas

```java
// Executar tarefa assíncrona
CompletableFuture<String> futuro = CompletableFuture.supplyAsync(() -> {
    // Simula operação demorada (ex: chamada HTTP)
    Thread.sleep(1000);
    return "resultado";
});

// Bloquear e obter o resultado (evite em produção — bloqueia a thread)
String resultado = futuro.get();          // lança Exception checada
String resultado2 = futuro.join();        // lança unchecked exception (preferível)

// Com timeout
String resultado3 = futuro.get(5, TimeUnit.SECONDS); // lança TimeoutException
```

### thenApply — transforma o resultado (síncrono)

```java
CompletableFuture<Integer> futuro = CompletableFuture
        .supplyAsync(() -> "42")
        .thenApply(s -> Integer.parseInt(s))    // String → Integer
        .thenApply(n -> n * 2);                  // Integer → Integer

System.out.println(futuro.join()); // 84
```

### thenCompose — encadeia futuros (evita aninhamento)

```java
// Sem thenCompose: aninhamento feio
CompletableFuture<CompletableFuture<String>> errado =
    CompletableFuture.supplyAsync(() -> buscarUsuarioId())
                     .thenApply(id -> buscarNome(id)); // retorna CF<CF<String>>

// Com thenCompose: "aplana" o resultado
CompletableFuture<String> correto =
    CompletableFuture.supplyAsync(() -> buscarUsuarioId())
                     .thenCompose(id -> buscarNome(id)); // retorna CF<String>
```

### thenCombine — combina dois futuros independentes

```java
CompletableFuture<String> nomeF = CompletableFuture.supplyAsync(() -> buscarNome(userId));
CompletableFuture<String> endF  = CompletableFuture.supplyAsync(() -> buscarEndereco(userId));

// Executa em paralelo, combina quando ambos terminam
CompletableFuture<String> resultado = nomeF.thenCombine(
        endF,
        (nome, endereco) -> nome + " mora em " + endereco
);
System.out.println(resultado.join());
```

### allOf — espera todos completarem

```java
List<Long> ids = List.of(1L, 2L, 3L, 4L, 5L);

List<CompletableFuture<Produto>> futuros = ids.stream()
        .map(id -> CompletableFuture.supplyAsync(() -> buscarProduto(id)))
        .toList();

// Espera todos terminarem
CompletableFuture<Void> todos = CompletableFuture.allOf(
        futuros.toArray(new CompletableFuture[0])
);

// Coleta os resultados após todos terminarem
List<Produto> produtos = todos.thenApply(v ->
        futuros.stream().map(CompletableFuture::join).toList()
).join();

System.out.println("Buscados " + produtos.size() + " produtos em paralelo");
```

### anyOf — retorna o mais rápido

```java
// Útil para circuit breaker simples: usa o servidor mais rápido
CompletableFuture<Object> maisRapido = CompletableFuture.anyOf(
        CompletableFuture.supplyAsync(() -> chamarServidorA()),
        CompletableFuture.supplyAsync(() -> chamarServidorB()),
        CompletableFuture.supplyAsync(() -> chamarServidorC())
);
String resultado = (String) maisRapido.join();
```

### exceptionally — tratamento de erros

```java
CompletableFuture<Produto> futuro = CompletableFuture
        .supplyAsync(() -> buscarProduto(id))
        .exceptionally(ex -> {
            log.error("Erro ao buscar produto {}: {}", id, ex.getMessage());
            return Produto.padrao(); // valor de fallback
        });

// handle: trata tanto sucesso quanto erro
CompletableFuture<Produto> futuro2 = CompletableFuture
        .supplyAsync(() -> buscarProduto(id))
        .handle((produto, ex) -> {
            if (ex != null) {
                log.error("Erro: {}", ex.getMessage());
                return Produto.padrao();
            }
            return produto;
        });
```

---

## CompletableFuture + Virtual Threads

Virtual Threads (Java 21) são threads extremamente leves gerenciadas pela JVM. Combinados com `CompletableFuture`, eliminam a preocupação com bloqueio de threads do pool.

```java
// Executor de Virtual Threads
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// CompletableFuture usando Virtual Threads
CompletableFuture<String> futuro = CompletableFuture.supplyAsync(
        () -> {
            // Pode bloquear à vontade: Virtual Thread, não thread do pool
            return chamarAPIExterna();
        },
        executor  // Usa o executor de Virtual Threads
);

// Processamento em paralelo com Virtual Threads
List<String> resultados = ids.parallelStream()
        .map(id -> CompletableFuture.supplyAsync(
                () -> buscarDadosExternos(id), executor))
        .map(CompletableFuture::join)
        .toList();
```

---

## StructuredTaskScope — Java 25 LTS

> **Status:** `StructuredTaskScope` ficou em preview no Java 21, 22, 23 e 24. Assumindo Java 25 LTS como base, use com `--enable-preview` em versões anteriores ao 25. Verifique sempre o JEP correspondente antes de usar em produção.

`StructuredTaskScope` é a nova API de concorrência estruturada que garante que subtarefas não "escapem" do escopo pai. Ao sair do bloco `try`, todas as tarefas filhas estão terminadas ou canceladas — elimina o problema de threads "soltas" do estilo `CompletableFuture` tradicional.

```java
import java.util.concurrent.StructuredTaskScope;

// Exemplo: buscar usuário e permissões em paralelo
// ShutdownOnFailure: cancela todas se qualquer uma falhar
public record DadosUsuario(Usuario usuario, List<Permissao> permissoes) {}

public DadosUsuario buscarDadosUsuario(Long userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

        // Fork: inicia as tarefas em paralelo (em Virtual Threads)
        StructuredTaskScope.Subtask<Usuario> usuarioTask =
                scope.fork(() -> usuarioRepository.buscar(userId));

        StructuredTaskScope.Subtask<List<Permissao>> permissoesTask =
                scope.fork(() -> permissaoService.listar(userId));

        // Espera todas terminarem (ou falhar)
        scope.join().throwIfFailed(); // Lança exceção se alguma falhou

        // Aqui todas as tarefas estão concluídas com sucesso
        return new DadosUsuario(usuarioTask.get(), permissoesTask.get());

    } // Ao sair do try, todas as tarefas são garantidamente concluídas
}

// ShutdownOnSuccess: retorna o primeiro que completar com sucesso
public String buscarDoServidorMaisRapido() throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {

        scope.fork(() -> buscarDeServidorA());
        scope.fork(() -> buscarDeServidorB());
        scope.fork(() -> buscarDeServidorC());

        scope.join(); // Espera o primeiro completar
        return scope.result(); // Retorna o primeiro resultado
    }
}
```

---

## Semáforo, CountDownLatch, CyclicBarrier, Phaser

### Semaphore — controla acesso a recursos limitados

```java
// Limita a 3 conexões simultâneas ao banco legado
Semaphore semaforo = new Semaphore(3);

public Resultado consultarBancoLegado(String query) throws InterruptedException {
    semaforo.acquire(); // Bloqueia se já há 3 usando
    try {
        return executarQuery(query);
    } finally {
        semaforo.release(); // Libera uma vaga
    }
}

// Versão não-bloqueante
if (semaforo.tryAcquire(500, TimeUnit.MILLISECONDS)) {
    try {
        return executarQuery(query);
    } finally {
        semaforo.release();
    }
} else {
    throw new RecursoOcupadoException("Banco legado sobrecarregado");
}
```

### CountDownLatch — aguarda N eventos

```java
// Inicializa o sistema somente quando todos os 3 serviços estiverem prontos
CountDownLatch latch = new CountDownLatch(3);

// Serviços notificam quando prontos
executor.submit(() -> { inicializarBanco(); latch.countDown(); });
executor.submit(() -> { inicializarCache(); latch.countDown(); });
executor.submit(() -> { inicializarKafka(); latch.countDown(); });

// Thread principal espera todos estarem prontos
boolean ok = latch.await(30, TimeUnit.SECONDS);
if (!ok) throw new TimeoutException("Inicialização não completou em 30s");

System.out.println("Sistema pronto!");
// CountDownLatch não pode ser reutilizado (ao contrário do CyclicBarrier)
```

### CyclicBarrier — sincroniza N threads em um ponto de encontro

```java
// 4 threads de processamento que se sincronizam a cada batch
CyclicBarrier barreira = new CyclicBarrier(4, () -> {
    // Ação executada quando todas as threads chegam à barreira
    System.out.println("Todos os workers terminaram o batch, iniciando próximo...");
});

for (int i = 0; i < 4; i++) {
    final int workerId = i;
    executor.submit(() -> {
        while (!terminado) {
            processarBatch(workerId);
            barreira.await(); // Espera os outros 3 workers terminarem
            // Após a barreira, todos os workers recomeçam juntos
        }
    });
}
// CyclicBarrier pode ser reutilizado (diferente do CountDownLatch)
```

### Phaser — versão flexível e reutilizável

```java
// Pipeline em 3 fases, onde novos participantes podem entrar/sair
Phaser phaser = new Phaser(1); // 1 = thread principal registrada

for (int i = 0; i < 5; i++) {
    phaser.register(); // Registra cada worker
    final int workerId = i;
    executor.submit(() -> {
        // Fase 1: preparação
        preparar(workerId);
        phaser.arriveAndAwaitAdvance(); // Espera todos terminarem fase 1

        // Fase 2: processamento
        processar(workerId);
        phaser.arriveAndAwaitAdvance(); // Espera todos terminarem fase 2

        // Fase 3: finalização
        finalizar(workerId);
        phaser.arriveAndDeregister();  // Remove do phaser ao terminar
    });
}

phaser.arriveAndDeregister(); // Thread principal sai do phaser
```

---

## Queues concorrentes

```java
// ConcurrentLinkedQueue: não-bloqueante, sem limite de tamanho
// Ideal para produtor/consumidor de alta velocidade
Queue<Tarefa> fila = new ConcurrentLinkedQueue<>();
fila.offer(novaTarefa);         // Adiciona (nunca bloqueia)
Tarefa t = fila.poll();         // Remove ou retorna null

// ArrayBlockingQueue: bloqueante, tamanho fixo
// Ideal para controlar pressão do produtor
BlockingQueue<Tarefa> fila2 = new ArrayBlockingQueue<>(100);
fila2.put(novaTarefa);          // Bloqueia se fila cheia
Tarefa t2 = fila2.take();       // Bloqueia se fila vazia

// LinkedBlockingQueue: bloqueante, tamanho opcional
// Mais performance que Array em alta concorrência
BlockingQueue<Tarefa> fila3 = new LinkedBlockingQueue<>(1000);

// PriorityBlockingQueue: bloqueante, ordena por prioridade
BlockingQueue<Tarefa> filaPrioridade = new PriorityBlockingQueue<>();
```

---

## Locks avançados

### ReentrantLock — alternativa flexível ao synchronized

```java
ReentrantLock lock = new ReentrantLock();

public void operacaoCritica() {
    lock.lock();
    try {
        // Seção crítica
        modificarRecursoCompartilhado();
    } finally {
        lock.unlock(); // Sempre no finally!
    }
}

// tryLock: tenta adquirir sem bloquear indefinidamente
public boolean tentarAtualizarSaldo(BigDecimal valor) {
    if (lock.tryLock()) {
        try {
            saldo = saldo.add(valor);
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false; // Outro está usando, tente mais tarde
}
```

### ReadWriteLock — múltiplos leitores, um escritor

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();

// Múltiplos leitores simultâneos
public Produto buscar(Long id) {
    rwLock.readLock().lock();
    try {
        return cache.get(id); // Múltiplas threads podem ler ao mesmo tempo
    } finally {
        rwLock.readLock().unlock();
    }
}

// Escritor exclusivo
public void atualizar(Produto produto) {
    rwLock.writeLock().lock();
    try {
        cache.put(produto.getId(), produto); // Exclusivo: bloqueia leitores
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

### StampedLock — ainda mais performático

```java
StampedLock stampedLock = new StampedLock();

public Produto buscar(Long id) {
    // Leitura otimista: NÃO adquire lock, apenas um "stamp"
    long stamp = stampedLock.tryOptimisticRead();
    Produto produto = cache.get(id);

    // Verifica se houve escrita desde o stamp
    if (!stampedLock.validate(stamp)) {
        // Houve escrita: promove para leitura com lock real
        stamp = stampedLock.readLock();
        try {
            produto = cache.get(id);
        } finally {
            stampedLock.unlockRead(stamp);
        }
    }
    return produto;
}
```

---

## Exemplos completos com problemas reais

### Produtor-Consumidor com BlockingQueue

```java
public class SistemaProcessamentoEmail {

    private final BlockingQueue<Email> fila = new LinkedBlockingQueue<>(500);
    private volatile boolean ativo = true;

    // Producer: recebe emails via API e coloca na fila
    public void receberEmail(Email email) throws InterruptedException {
        if (!fila.offer(email, 1, TimeUnit.SECONDS)) {
            throw new FilaCheiaException("Fila de emails cheia, tente novamente");
        }
        System.out.println("Email enfileirado: " + email.getAssunto());
    }

    // Consumer: processa emails da fila
    public void iniciarConsumidores(int qtdConsumidores) {
        for (int i = 0; i < qtdConsumidores; i++) {
            Thread.ofVirtual().name("email-worker-" + i).start(() -> {
                while (ativo || !fila.isEmpty()) {
                    try {
                        Email email = fila.poll(500, TimeUnit.MILLISECONDS);
                        if (email != null) {
                            processarEmail(email);
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
    }

    private void processarEmail(Email email) {
        System.out.println("Processando: " + email.getAssunto());
        // envio via SMTP, etc.
    }

    public void parar() {
        this.ativo = false;
    }
}
```

### Pipeline paralelo com CompletableFuture

```java
public class PipelineProcessamentoPedido {

    public CompletableFuture<ResultadoPedido> processar(Pedido pedido) {
        ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

        // Etapa 1: Validações em paralelo
        CompletableFuture<Boolean> estoqueOk = CompletableFuture.supplyAsync(
                () -> estoqueService.verificar(pedido), executor);

        CompletableFuture<Boolean> creditoOk = CompletableFuture.supplyAsync(
                () -> creditoService.verificar(pedido.getClienteId()), executor);

        CompletableFuture<Boolean> fraudeOk = CompletableFuture.supplyAsync(
                () -> antiFraudeService.verificar(pedido), executor);

        // Aguarda todas as validações
        return CompletableFuture.allOf(estoqueOk, creditoOk, fraudeOk)
                .thenCompose(v -> {
                    // Verifica resultados
                    if (!estoqueOk.join()) return CompletableFuture.completedFuture(
                            ResultadoPedido.rejeitado("Estoque insuficiente"));
                    if (!creditoOk.join()) return CompletableFuture.completedFuture(
                            ResultadoPedido.rejeitado("Crédito insuficiente"));
                    if (!fraudeOk.join()) return CompletableFuture.completedFuture(
                            ResultadoPedido.rejeitado("Suspeita de fraude"));

                    // Etapa 2: Processamento do pagamento
                    return CompletableFuture.supplyAsync(
                            () -> pagamentoService.processar(pedido), executor);
                })
                .thenCompose(pagamento -> {
                    if (!pagamento.isAprovado()) {
                        return CompletableFuture.completedFuture(
                                ResultadoPedido.rejeitado(pagamento.getMotivoRejeicao()));
                    }

                    // Etapa 3: Ações pós-aprovação em paralelo
                    CompletableFuture<Void> reservarEstoque = CompletableFuture.runAsync(
                            () -> estoqueService.reservar(pedido), executor);
                    CompletableFuture<Void> notificarCliente = CompletableFuture.runAsync(
                            () -> notificacaoService.enviarAprovacao(pedido), executor);
                    CompletableFuture<Void> emitirNF = CompletableFuture.runAsync(
                            () -> nfService.emitir(pedido, pagamento), executor);

                    return CompletableFuture.allOf(reservarEstoque, notificarCliente, emitirNF)
                            .thenApply(v -> ResultadoPedido.aprovado(pagamento.getTransacaoId()));
                })
                .exceptionally(ex -> {
                    log.error("Erro no processamento do pedido {}", pedido.getId(), ex);
                    return ResultadoPedido.erro("Erro interno: " + ex.getMessage());
                });
    }
}
```
