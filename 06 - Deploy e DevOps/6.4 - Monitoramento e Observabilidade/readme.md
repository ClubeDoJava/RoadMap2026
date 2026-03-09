# 6.4 - Monitoramento e Observabilidade

## Os 3 pilares da Observabilidade

```
┌─────────────────────────────────────────────────────────────────┐
│                     OBSERVABILIDADE                             │
│                                                                 │
│   📊 MÉTRICAS          📄 LOGS           🔍 TRACES             │
│   ──────────           ────────          ─────────             │
│   O que está           O que             Por onde               │
│   acontecendo?         aconteceu?        passou a               │
│   (números             (eventos          requisição?            │
│   ao longo do          pontuais)         (rastreio              │
│   tempo)                                 distribuído)           │
│                                                                 │
│   Ferramentas:         Ferramentas:      Ferramentas:           │
│   Prometheus           Logback           Jaeger                 │
│   Micrometer           ELK Stack         Zipkin                 │
│   Grafana              Loki              OpenTelemetry           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Spring Boot Actuator

O Actuator expõe endpoints HTTP com informações sobre o estado da aplicação.

### Dependência

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Para métricas no formato Prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### Configuração

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info,env,prometheus,loggers,threaddump,heapdump
      base-path: /actuator
  endpoint:
    health:
      show-details: always  # Mostra detalhes como status do banco
      probes:
        enabled: true       # Habilita /health/liveness e /health/readiness (Kubernetes)
  info:
    env:
      enabled: true
    git:
      mode: full

# Informações da aplicação exibidas em /actuator/info
info:
  app:
    name: Minha App
    version: '@project.version@'
    java-version: '@java.version@'
  build:
    time: '@maven.build.timestamp@'
```

### Endpoints disponíveis

| Endpoint | Método | Descrição |
|----------|--------|-----------|
| `/actuator/health` | GET | Status geral da aplicação e dos componentes |
| `/actuator/health/liveness` | GET | Está vivo? (usado pelo Kubernetes) |
| `/actuator/health/readiness` | GET | Pronto para receber tráfego? |
| `/actuator/metrics` | GET | Lista todas as métricas disponíveis |
| `/actuator/metrics/{nome}` | GET | Valor de uma métrica específica |
| `/actuator/prometheus` | GET | Métricas no formato Prometheus (scraping) |
| `/actuator/info` | GET | Informações da aplicação |
| `/actuator/env` | GET | Variáveis de ambiente e propriedades |
| `/actuator/loggers` | GET/POST | Ver e alterar nível de log em runtime |
| `/actuator/threaddump` | GET | Dump de todas as threads |
| `/actuator/heapdump` | GET | Dump do heap (arquivo binário para análise) |

### Métricas personalizadas com Micrometer

```java
@Service
public class PedidoService {

    private final MeterRegistry meterRegistry;
    private final Counter pedidosCriados;
    private final Timer tempoProcessamento;

    public PedidoService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;

        // Contador: incrementa a cada pedido criado
        this.pedidosCriados = Counter.builder("pedidos.criados.total")
                .description("Total de pedidos criados")
                .tag("aplicacao", "minha-app")
                .register(meterRegistry);

        // Timer: mede o tempo de processamento
        this.tempoProcessamento = Timer.builder("pedidos.processamento.tempo")
                .description("Tempo de processamento de pedidos")
                .register(meterRegistry);
    }

    public Pedido criarPedido(PedidoRequest request) {
        return tempoProcessamento.record(() -> {
            Pedido pedido = processarPedido(request);
            pedidosCriados.increment();
            return pedido;
        });
    }

    // Gauge: valor que varia (ex: tamanho de uma fila)
    @PostConstruct
    public void registrarGauge() {
        Gauge.builder("pedidos.fila.tamanho", this, PedidoService::getTamanhoDaFila)
                .description("Tamanho atual da fila de pedidos")
                .register(meterRegistry);
    }

    private double getTamanhoDaFila() {
        return filaDePedidos.size();
    }
}
```

---

## Prometheus

Prometheus é um banco de dados de séries temporais que "raspa" (scrape) métricas dos endpoints das aplicações em intervalos regulares.

### Como funciona

```
Aplicação → /actuator/prometheus → Prometheus (coleta a cada 15s) → Grafana (visualiza)
```

### Configuração do Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s      # Coleta métricas a cada 15 segundos
  evaluation_interval: 15s  # Avalia alertas a cada 15 segundos

scrape_configs:
  - job_name: 'minha-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']  # Nome do serviço no Docker Compose
    scrape_interval: 10s

  - job_name: 'prometheus-self'
    static_configs:
      - targets: ['localhost:9090']
```

### Queries PromQL essenciais

```promql
# Taxa de requisições por segundo (últimos 5 minutos)
rate(http_server_requests_seconds_count[5m])

# Percentil 95 do tempo de resposta
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))

# Requisições com erro (status 5xx)
rate(http_server_requests_seconds_count{status=~"5.."}[5m])

# Uso de memória heap em MB
jvm_memory_used_bytes{area="heap"} / 1024 / 1024

# Número de threads ativas
jvm_threads_live_threads

# Taxa de erro de um endpoint específico
rate(http_server_requests_seconds_count{uri="/api/pedidos", status=~"5.."}[5m])
```

---

## Grafana: dashboards

### Acesso e configuração inicial

1. Acesse `http://localhost:3000`
2. Login padrão: `admin` / `admin`
3. Adicione o Prometheus como datasource:
   - `Configuration → Data Sources → Add data source → Prometheus`
   - URL: `http://prometheus:9090`
   - Clique em `Save & Test`

### Importar dashboard pronto para Spring Boot

1. `Dashboards → Import`
2. Cole o ID `12685` (Spring Boot 3.x Statistics) ou `4701`
3. Selecione o datasource Prometheus
4. Clique em `Import`

O dashboard importado inclui automaticamente:
- Taxa de requisições por segundo
- Tempo de resposta (p50, p95, p99)
- Status da JVM (heap, threads, GC)
- Status do banco de dados (connection pool)

---

## OpenTelemetry: instrumentação automática e traces

OpenTelemetry (OTel) é o padrão aberto para telemetria (traces, métricas e logs). Com o Spring Boot, você pode instrumentar automaticamente toda a aplicação sem mudar o código de negócio.

### Dependências

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### Configuração

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% das requisições (em prod, use 0.1 = 10%)

spring:
  application:
    name: minha-app  # Aparece nos traces

# Exportar traces para Jaeger/Tempo via OTLP
otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
```

### Traces manuais (quando precisar de mais detalhes)

```java
@Service
public class PagamentoService {

    private final Tracer tracer;

    public PagamentoService(Tracer tracer) {
        this.tracer = tracer;
    }

    public ResultadoPagamento processar(Pagamento pagamento) {
        // Cria um span filho do trace atual
        Span span = tracer.nextSpan().name("processar-pagamento").start();

        try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
            span.tag("pagamento.valor", pagamento.getValor().toString());
            span.tag("pagamento.metodo", pagamento.getMetodo());

            ResultadoPagamento resultado = chamarGatewayPagamento(pagamento);

            span.tag("pagamento.status", resultado.getStatus());
            return resultado;

        } catch (Exception e) {
            span.tag("erro", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## Logging estruturado com Logback + JSON

Logs em JSON são facilmente parseados por ferramentas como Elasticsearch e Loki.

### Dependência

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

### Configuração Logback para produção

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- Perfil de desenvolvimento: log humanamente legível -->
    <springProfile name="!prod">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <!-- Perfil de produção: log em JSON estruturado -->
    <springProfile name="prod">
        <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <customFields>{"aplicacao":"minha-app","ambiente":"producao"}</customFields>
                <!-- Inclui trace ID e span ID do OpenTelemetry -->
                <includeCallerData>false</includeCallerData>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON_CONSOLE"/>
        </root>
    </springProfile>

</configuration>
```

### Exemplo de log JSON gerado

```json
{
  "@timestamp": "2026-03-09T14:32:15.123Z",
  "level": "INFO",
  "logger_name": "com.minhaapp.service.PedidoService",
  "message": "Pedido criado com sucesso",
  "thread_name": "http-nio-8080-exec-1",
  "aplicacao": "minha-app",
  "ambiente": "producao",
  "traceId": "abc123def456",
  "spanId": "789xyz",
  "pedido_id": "42",
  "usuario_id": "100"
}
```

### Logging com MDC (Mapped Diagnostic Context)

```java
@Component
public class LoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        // Adiciona informações ao contexto de log de toda a requisição
        MDC.put("requestId", UUID.randomUUID().toString());
        MDC.put("ip", request.getRemoteAddr());
        MDC.put("uri", request.getRequestURI());

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear(); // Sempre limpar para evitar vazamento entre threads
        }
    }
}
```

---

## Alertas básicos no Grafana

```yaml
# Exemplo de regra de alerta no Grafana (via arquivo YAML)
# grafana/provisioning/alerting/rules.yml

apiVersion: 1
groups:
  - orgId: 1
    name: minha-app-alertas
    folder: Alertas
    interval: 1m
    rules:
      - uid: alerta-taxa-erro
        title: "Alta taxa de erros 5xx"
        condition: C
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.1
        noDataState: OK
        execErrState: Error
        for: 2m
        annotations:
          summary: "Taxa de erros acima de 10%"
          description: "A taxa de erros 5xx está em {{ $value }} req/s"

      - uid: alerta-memoria-heap
        title: "Uso de heap crítico"
        condition: C
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: |
                jvm_memory_used_bytes{area="heap"} /
                jvm_memory_max_bytes{area="heap"} > 0.85
        for: 5m
        annotations:
          summary: "Uso de heap acima de 85%"
```

---

## Exemplo completo: docker-compose com app + Prometheus + Grafana

```yaml
# docker-compose.observabilidade.yml
version: '3.9'

services:
  # ──────────────────────────────────────────
  # Aplicação Spring Boot
  # ──────────────────────────────────────────
  app:
    build: .
    container_name: minha-app
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/minhaapp
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: senha123
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - monitoring

  # ──────────────────────────────────────────
  # PostgreSQL
  # ──────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: minhaapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: senha123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - monitoring

  # ──────────────────────────────────────────
  # Prometheus
  # ──────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    networks:
      - monitoring

  # ──────────────────────────────────────────
  # Grafana
  # ──────────────────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin123
      GF_USERS_ALLOW_SIGN_UP: false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
    networks:
      - monitoring

  # ──────────────────────────────────────────
  # Jaeger (traces distribuídos)
  # ──────────────────────────────────────────
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "16686:16686"  # Interface web
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: true
    networks:
      - monitoring

volumes:
  postgres_data:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

### Arquivo de configuração Prometheus para o compose

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'minha-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### Como usar

```bash
# Subir toda a stack de observabilidade
docker compose -f docker-compose.observabilidade.yml up -d

# Acessar:
# Aplicação: http://localhost:8080
# Prometheus: http://localhost:9090
# Grafana:    http://localhost:3000 (admin/admin123)
# Jaeger:     http://localhost:16686

# Ver métricas da aplicação diretamente
curl http://localhost:8080/actuator/prometheus

# Ver saúde
curl http://localhost:8080/actuator/health | jq .

# Ver métricas de uma API específica
curl "http://localhost:8080/actuator/metrics/http.server.requests" | jq .
```
