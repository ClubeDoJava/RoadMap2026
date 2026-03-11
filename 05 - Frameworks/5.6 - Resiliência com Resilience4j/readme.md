# 5.6 — Resiliência com Resilience4j

Microsserviços falham. Uma dependência fica lenta, um banco de dados satura, uma API externa retorna 503 às 14h numa sexta-feira. O que diferencia um sistema que degrada graciosamente de um que cascateia falhas para toda a plataforma é a implementação de padrões de resiliência. Resilience4j é a biblioteca padrão do ecossistema Spring Boot para isso.

---

## Dependências Maven

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>

<!-- Para métricas no Actuator/Micrometer -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- AOP é necessário para as anotações funcionarem -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

---

## Circuit Breaker

O Circuit Breaker é o padrão mais importante aqui, e é frequentemente mal compreendido. Não é apenas "parar de chamar quando falha" — tem estados bem definidos e transições entre eles.

```
CLOSED (fechado — operação normal)
  |
  | taxa de falhas excede threshold (ex: 50% das últimas 10 chamadas)
  v
OPEN (aberto — todas as chamadas falham imediatamente, sem tentar)
  |
  | após waitDurationInOpenState (ex: 60 segundos)
  v
HALF_OPEN (meio-aberto — permite N chamadas de teste)
  |
  |-- se as chamadas de teste passam --> CLOSED
  |-- se as chamadas de teste falham --> OPEN novamente
```

Quando o circuito está OPEN, a chamada falha imediatamente com `CallNotPermittedException` — sem esperar timeout da dependência. Isso é o que protege o sistema chamador: em vez de esperar 30 segundos por um banco saturado multiplicado por 200 threads, as chamadas falham em microssegundos e permitem que o sistema tente um fallback.

### Configuração

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      pagamento-service:
        # Janela deslizante de contagem: avalia os últimos N chamadas
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10

        # Percentual de falhas para abrir o circuito
        failureRateThreshold: 50

        # Percentual de chamadas lentas que conta como falha
        slowCallRateThreshold: 80
        slowCallDurationThreshold: 3s

        # Tempo que fica em OPEN antes de tentar HALF_OPEN
        waitDurationInOpenState: 60s

        # Número de chamadas permitidas em HALF_OPEN para testar
        permittedNumberOfCallsInHalfOpenState: 5

        # Número mínimo de chamadas antes de calcular taxa de falha
        minimumNumberOfCalls: 5

        # Exceções que contam como falha
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - feign.FeignException

        # Exceções que NÃO contam como falha (ex: 404 é resposta válida)
        ignoreExceptions:
          - com.empresa.exception.RecursoNaoEncontradoException
```

### Uso com anotação

```java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;

@Service
public class PagamentoService {

    private final PagamentoClient pagamentoClient;

    @CircuitBreaker(name = "pagamento-service", fallbackMethod = "pagamentoFallback")
    public PagamentoResponse processarPagamento(PagamentoRequest request) {
        // Se pagamentoClient estiver instável, o circuito vai abrir
        return pagamentoClient.processar(request);
    }

    // Fallback: chamado quando o circuito está OPEN ou quando a chamada falha
    // A assinatura precisa ter os mesmos parâmetros + a exceção como último argumento
    private PagamentoResponse pagamentoFallback(PagamentoRequest request, Exception ex) {
        log.warn("Circuit breaker ativado para pagamento-service: {}", ex.getMessage());
        // Dependendo do contexto: fila assíncrona, resposta cached, erro amigável
        return PagamentoResponse.pendente("Pagamento em processamento — tente novamente em instantes");
    }
}
```

### Uso programático (quando precisar de controle total)

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;

@Service
public class PagamentoService {

    private final CircuitBreaker circuitBreaker;
    private final PagamentoClient pagamentoClient;

    public PagamentoService(CircuitBreakerRegistry registry, PagamentoClient client) {
        // Pega a instância configurada no yml
        this.circuitBreaker = registry.circuitBreaker("pagamento-service");
        this.pagamentoClient = client;
    }

    public PagamentoResponse processarPagamento(PagamentoRequest request) {
        return CircuitBreaker.decorateSupplier(
            circuitBreaker,
            () -> pagamentoClient.processar(request)
        ).get();
    }

    public CircuitBreaker.State estadoAtual() {
        return circuitBreaker.getState();
    }
}
```

---

## Retry com Backoff Exponencial

Retry sem backoff é perigoso: se 100 instâncias da aplicação tentam reconectar a um banco instável a cada segundo, você cria um thundering herd que impede a recuperação. Backoff exponencial com jitter resolve isso.

```yaml
resilience4j:
  retry:
    instances:
      estoque-service:
        # Número máximo de tentativas (incluindo a primeira)
        maxAttempts: 3

        # Tempo de espera entre tentativas (dobra a cada retry com exponential)
        waitDuration: 500ms

        # Backoff exponencial: 500ms, 1000ms, 2000ms...
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2

        # Jitter aleatório (±25%) para evitar thundering herd
        enableRandomizedWait: true
        randomizedWaitFactor: 0.25

        # Só faz retry para estas exceções
        retryExceptions:
          - java.io.IOException
          - java.net.ConnectException
          - feign.RetryableException

        # Nunca faz retry para estas (erros de negócio não melhoram com retry)
        ignoreExceptions:
          - com.empresa.exception.SaldoInsuficienteException
          - com.empresa.exception.PedidoDuplicadoException
```

```java
import io.github.resilience4j.retry.annotation.Retry;

@Service
public class EstoqueService {

    @Retry(name = "estoque-service", fallbackMethod = "estoqueIndisponivel")
    public EstoqueResponse verificarEstoque(UUID produtoId) {
        return estoqueClient.verificar(produtoId);
    }

    private EstoqueResponse estoqueIndisponivel(UUID produtoId, Exception ex) {
        log.error("Estoque indisponível após retries para produto {}: {}", produtoId, ex.getMessage());
        // Estoque assumido disponível (otimista) ou exceção de negócio
        throw new ServicoIndisponivelException("Serviço de estoque temporariamente indisponível");
    }
}
```

---

## Combinando Circuit Breaker + Retry

A ordem importa: o Retry deve estar dentro do Circuit Breaker, não fora. Se o circuito está OPEN, não faz sentido tentar novamente.

```java
// A anotação @CircuitBreaker engloba @Retry quando ambas são usadas no mesmo método
// Use a ordem correta via configuração:

@CircuitBreaker(name = "pagamento-service", fallbackMethod = "fallback")
@Retry(name = "pagamento-service")
@TimeLimiter(name = "pagamento-service")
public CompletableFuture<PagamentoResponse> processarPagamento(PagamentoRequest req) {
    return CompletableFuture.supplyAsync(() -> pagamentoClient.processar(req));
}
```

Ordem de execução (de fora para dentro): `TimeLimiter → CircuitBreaker → Retry → função`.

---

## Bulkhead — Isolamento de recursos

Bulkhead impede que uma dependência lenta consuma todas as threads do pool, afetando outras funcionalidades da aplicação. O nome vem de compartimentos estanques de navios: um vazamento não afunda o navio inteiro.

```yaml
resilience4j:
  bulkhead:
    instances:
      relatorio-service:
        # Máximo de chamadas concorrentes permitidas
        maxConcurrentCalls: 10
        # Tempo máximo de espera para entrar na fila
        maxWaitDuration: 500ms
```

```java
@Bulkhead(name = "relatorio-service", type = Bulkhead.Type.SEMAPHORE,
          fallbackMethod = "relatorioIndisponivel")
public RelatorioResponse gerarRelatorio(RelatorioRequest request) {
    // Se já houver 10 chamadas concorrentes, a 11ª recebe o fallback imediatamente
    return relatorioService.gerar(request);
}

private RelatorioResponse relatorioIndisponivel(RelatorioRequest request,
                                                 BulkheadFullException ex) {
    return RelatorioResponse.emFila("Relatório enfileirado — você receberá por email");
}
```

---

## Monitoramento via Actuator

Com Resilience4j + Spring Boot Actuator, você tem métricas e estado dos circuitos expostos automaticamente:

```bash
# Estado atual de todos os circuit breakers
curl http://localhost:8080/actuator/circuitbreakers

# Métricas de um circuito específico
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.state?tag=name:pagamento-service
```

No Prometheus/Grafana, as métricas aparecem como:
```
resilience4j_circuitbreaker_state{name="pagamento-service"} 0  # 0=CLOSED, 1=OPEN, 2=HALF_OPEN
resilience4j_circuitbreaker_failure_rate{name="pagamento-service"} 45.5
resilience4j_retry_calls_total{name="estoque-service",kind="successful_without_retry"} 142
resilience4j_retry_calls_total{name="estoque-service",kind="successful_with_retry"} 8
resilience4j_retry_calls_total{name="estoque-service",kind="failed_with_retry"} 2
```

---

## Exemplo completo: serviço com resiliência em produção

```java
@Service
@Slf4j
public class CheckoutService {

    private final PagamentoClient pagamentoClient;
    private final EstoqueClient estoqueClient;
    private final PedidoRepository pedidoRepository;

    // Circuit breaker protege contra falhas do serviço de pagamento
    @CircuitBreaker(name = "pagamento-service", fallbackMethod = "pagamentoFallback")
    // Retry com backoff para falhas transientes de rede
    @Retry(name = "pagamento-service")
    public PedidoResponse finalizarPedido(FinalizarPedidoRequest request) {
        // 1. Verifica estoque (com seu próprio circuit breaker configurado separado)
        verificarEstoque(request.itens());

        // 2. Processa pagamento
        var pagamento = pagamentoClient.processar(
            new PagamentoRequest(request.clienteId(), request.total(), request.cartao())
        );

        if (!pagamento.aprovado()) {
            throw new PagamentoRecusadoException(pagamento.motivo());
        }

        // 3. Persiste pedido
        var pedido = pedidoRepository.save(Pedido.from(request, pagamento.transacaoId()));
        log.info("Pedido {} finalizado. Transação: {}", pedido.getId(), pagamento.transacaoId());

        return PedidoResponse.from(pedido);
    }

    private PedidoResponse pagamentoFallback(FinalizarPedidoRequest request, Exception ex) {
        if (ex instanceof CallNotPermittedException) {
            // Circuito aberto: pagamento-service está instável
            log.warn("Circuito aberto para pagamento-service. Pedido {} enfileirado", request.clienteId());
            enfileirarParaProcessamentoAsync(request);
            return PedidoResponse.pendente("Pedido recebido — processamento em até 5 minutos");
        }
        // Falha após retries esgotados
        log.error("Falha definitiva no pagamento: {}", ex.getMessage());
        throw new PagamentoIndisponivelException("Serviço de pagamento temporariamente indisponível");
    }

    @CircuitBreaker(name = "estoque-service")
    @Bulkhead(name = "estoque-service")
    private void verificarEstoque(List<ItemPedido> itens) {
        for (var item : itens) {
            var estoque = estoqueClient.verificar(item.produtoId());
            if (estoque.quantidade() < item.quantidade()) {
                throw new EstoqueInsuficienteException(item.produtoId(), item.quantidade(), estoque.quantidade());
            }
        }
    }
}
```
