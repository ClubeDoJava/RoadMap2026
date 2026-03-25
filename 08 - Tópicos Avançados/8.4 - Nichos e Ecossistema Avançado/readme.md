# 8.4 - Nichos e Ecossistema Avançado

> **Prioridade recomendada:** Kafka → Spring AI → gRPC/GraphQL → DJL/Spark. Kafka é infraestrutura básica de microsserviços. Spring AI tem adoção crescente e demanda de mercado real em 2026. DJL e Spark são especializações de nicho.

---

## Spring AI — Integração com LLMs em Java

Spring AI é a abstração oficial do Spring Framework para integração com modelos de linguagem. Permite usar OpenAI, Anthropic, Mistral, Ollama e outros provedores com a mesma API, trocando o provider via configuração.

### Dependência

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
<!-- ou para Anthropic Claude -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
</dependency>
```

### Chat simples

```java
@Service
@RequiredArgsConstructor
public class AssistenteService {

    private final ChatClient chatClient;

    public String responder(String pergunta) {
        return chatClient.prompt()
                .user(pergunta)
                .call()
                .content();
    }

    // Com system prompt fixo e histórico de conversa
    public String responderComContexto(String pergunta, List<Message> historico) {
        return chatClient.prompt()
                .system("Você é um especialista em Java. Seja direto e prático.")
                .messages(historico)
                .user(pergunta)
                .call()
                .content();
    }
}
```

### Padrão RAG — Retrieval-Augmented Generation

RAG resolve o problema de "o modelo não sabe sobre seus dados internos". Em vez de retreinar o modelo, você injeta o contexto relevante em cada chamada:

```
[Pergunta do usuário]
        │
        ▼
[Gerar embedding da pergunta]
        │
        ▼
[Buscar chunks similares no vector store]
        │
        ▼
[Montar prompt: contexto + pergunta]
        │
        ▼
[Enviar para LLM → Resposta contextualizada]
```

```java
@Service
@RequiredArgsConstructor
public class RagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    // Indexar documentos (executar uma vez)
    public void indexar(List<Document> documentos) {
        vectorStore.add(documentos);
    }

    // Responder com contexto dos documentos indexados
    public String responder(String pergunta) {
        // Busca os chunks mais relevantes
        List<Document> contexto = vectorStore.similaritySearch(
                SearchRequest.query(pergunta).withTopK(4)
        );

        String textoContexto = contexto.stream()
                .map(Document::getContent)
                .collect(Collectors.joining("\n\n"));

        return chatClient.prompt()
                .system("""
                        Responda com base APENAS no contexto fornecido.
                        Se a informação não estiver no contexto, diga que não sabe.
                        Contexto:
                        """ + textoContexto)
                .user(pergunta)
                .call()
                .content();
    }
}
```

### Configuração (application.yml)

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-opus-4-5  # troque pelo modelo atual disponível na sua conta
          max-tokens: 2048
    # Para vector store em memória (desenvolvimento)
    vectorstore:
      simple:
        defaults:
          top-k: 4
```

> **Atenção:** identificadores de modelos mudam com frequência (ex: `claude-opus-4-5` pode tornar-se `claude-opus-4-6` ou similar). Não hardcode versões de modelo no código de produção — leia o identificador de variável de ambiente ou configuração externa.

### Projeto prático

Construa um assistente de documentação: indexe os READMEs do seu projeto usando Spring AI + VectorStore em memória, e responda perguntas sobre o conteúdo. Depois troque para PGVector (Postgres) para persistência real.

---

## Deep Java Library (DJL) — Inferência de Modelos ML em Java

> **Contexto de uso:** DJL é adequado para inferência de modelos pre-treinados em Java puro, sem depender de um servidor externo. Tem adoção de mercado menor do que Spring AI para integrações com LLMs — avalie pelo caso de uso. Se o objetivo é integrar com APIs de LLMs (OpenAI, Anthropic), use Spring AI. Se o objetivo é rodar inferência local com PyTorch/ONNX, DJL é a escolha.

DJL é uma biblioteca open-source da Amazon que permite executar modelos de Machine Learning em Java sem precisar escrever código Python. Suporta PyTorch, TensorFlow, MXNet e ONNX Runtime.

### Dependência

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>ai.djl</groupId>
            <artifactId>bom</artifactId>
            <version>0.27.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>ai.djl</groupId>
        <artifactId>api</artifactId>
    </dependency>
    <!-- PyTorch como engine de inferência -->
    <dependency>
        <groupId>ai.djl.pytorch</groupId>
        <artifactId>pytorch-engine</artifactId>
    </dependency>
    <dependency>
        <groupId>ai.djl.pytorch</groupId>
        <artifactId>pytorch-native-auto</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### Inferência com modelo pré-treinado

```java
// Classificação de imagem com modelo ResNet pré-treinado
public class ClassificadorImagem {

    public List<Classifications.Classification> classificar(Path imagemPath) throws Exception {

        // Carrega a imagem e redimensiona para o tamanho esperado pelo modelo
        Image imagem = ImageFactory.getInstance().fromFile(imagemPath);

        // Translators: pré/pós-processamento específico do modelo
        ImageClassificationTranslator translator = ImageClassificationTranslator.builder()
                .addTransform(new Resize(256))
                .addTransform(new CenterCrop(224, 224))
                .addTransform(new ToTensor())
                .addTransform(new Normalize(
                    new float[]{0.485f, 0.456f, 0.406f},  // Mean ImageNet
                    new float[]{0.229f, 0.224f, 0.225f}   // Std ImageNet
                ))
                .optApplySoftmax(true)
                .build();

        // Carrega modelo do Model Zoo (download automático na primeira vez)
        Criteria<Image, Classifications> criteria = Criteria.builder()
                .optApplication(Application.CV.IMAGE_CLASSIFICATION)
                .setTypes(Image.class, Classifications.class)
                .optFilter("layer", "50")           // ResNet-50
                .optFilter("dataset", "imagenet")
                .optTranslator(translator)
                .optProgress(new ProgressBar())
                .build();

        // Executa inferência
        try (ZooModel<Image, Classifications> modelo = ModelZoo.loadModel(criteria);
             Predictor<Image, Classifications> predictor = modelo.newPredictor()) {

            Classifications resultado = predictor.predict(imagem);
            return resultado.topK(5); // Top 5 classificações
        }
    }
}

// Uso
ClassificadorImagem classificador = new ClassificadorImagem();
List<Classifications.Classification> top5 = classificador.classificar(
        Path.of("gato.jpg"));

for (Classifications.Classification c : top5) {
    System.out.printf("%-30s %.2f%%%n", c.getClassName(), c.getProbability() * 100);
}
// tabby, tabby cat                99.23%
// Egyptian cat                     0.41%
// tiger cat                        0.21%
```

### Análise de sentimento com DJL e HuggingFace

```java
public class AnalisadorSentimento {

    public String analisar(String texto) throws Exception {
        Criteria<String, Classifications> criteria = Criteria.builder()
                .optApplication(Application.NLP.SENTIMENT_ANALYSIS)
                .setTypes(String.class, Classifications.class)
                .optEngine("PyTorch")
                .optModelUrls("djl://ai.djl.huggingface.pytorch/distilbert-base-uncased-finetuned-sst-2-english")
                .build();

        try (ZooModel<String, Classifications> modelo = ModelZoo.loadModel(criteria);
             Predictor<String, Classifications> predictor = modelo.newPredictor()) {

            Classifications resultado = predictor.predict(texto);
            return resultado.best().getClassName(); // "POSITIVE" ou "NEGATIVE"
        }
    }
}
```

---

## Apache Kafka

Kafka é uma plataforma de streaming de eventos distribuída, usada para desacoplar produtores e consumidores de dados de forma assíncrona e tolerante a falhas.

### Conceitos fundamentais

```
Produtor → [Tópico: pedidos]  → Consumidor (grupo A: microsserviço de estoque)
                              → Consumidor (grupo B: microsserviço de NF)
                              → Consumidor (grupo C: analytics)

Cada tópico tem N partições, e cada partição mantém uma sequência ordenada de mensagens.
Cada mensagem tem um offset (posição) permanente — consumidores guardam sua posição.
```

| Conceito | Descrição |
|----------|-----------|
| **Topic** | Canal lógico de mensagens (ex: `pedidos`, `pagamentos`) |
| **Partition** | Sub-divisão do tópico para paralelismo |
| **Offset** | Posição de uma mensagem na partição (nunca muda) |
| **Producer** | Envia mensagens para um tópico |
| **Consumer** | Lê mensagens de um tópico |
| **Consumer Group** | Grupo de consumidores que divide as partições (cada mensagem vai para 1 consumidor do grupo) |
| **Broker** | Servidor Kafka |

### Dependência Spring Kafka

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### Configuração

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all              # Aguarda confirmação de todas as réplicas
      retries: 3
      properties:
        enable.idempotence: true   # Evita mensagens duplicadas em caso de retry
    consumer:
      group-id: minha-app
      auto-offset-reset: earliest  # Começa a ler do início quando não há offset salvo
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.minhaapp.event"
```

### Produtor — Exemplo completo

```java
// Evento de domínio
public record PedidoCriadoEvent(
    String pedidoId,
    String clienteId,
    BigDecimal valorTotal,
    List<ItemEvent> itens,
    Instant criadoEm
) {}

// Produtor Spring Kafka
@Service
@RequiredArgsConstructor
@Slf4j
public class PedidoEventProducer {

    private final KafkaTemplate<String, PedidoCriadoEvent> kafkaTemplate;

    @Value("${app.kafka.topics.pedidos-criados}")
    private String topicPedidosCriados;

    public void publicarPedidoCriado(PedidoCriadoEvent evento) {
        // A chave garante que mensagens do mesmo pedido vão para a mesma partição
        kafkaTemplate.send(topicPedidosCriados, evento.pedidoId(), evento)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Erro ao publicar evento do pedido {}: {}",
                                evento.pedidoId(), ex.getMessage());
                        // Em prod: salvar em outbox table para retry
                    } else {
                        log.info("Evento publicado: pedido={}, partition={}, offset={}",
                                evento.pedidoId(),
                                result.getRecordMetadata().partition(),
                                result.getRecordMetadata().offset());
                    }
                });
    }
}
```

### Consumidor — Exemplo completo

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class PedidoCriadoConsumer {

    private final EstoqueService estoqueService;
    private final NotificacaoService notificacaoService;

    @KafkaListener(
        topics = "${app.kafka.topics.pedidos-criados}",
        groupId = "${spring.kafka.consumer.group-id}",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consumir(
            @Payload PedidoCriadoEvent evento,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int particao,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        log.info("Recebido evento: pedido={}, partition={}, offset={}",
                evento.pedidoId(), particao, offset);

        try {
            // Processamento idempotente (pode ser chamado múltiplas vezes com segurança)
            estoqueService.reservarItens(evento.pedidoId(), evento.itens());
            notificacaoService.enviarConfirmacao(evento.clienteId(), evento.pedidoId());

            // Confirma o processamento apenas após sucesso
            ack.acknowledge();

        } catch (EstoqueInsuficienteException e) {
            log.error("Estoque insuficiente para pedido {}: {}", evento.pedidoId(), e.getMessage());
            ack.acknowledge(); // Confirma mesmo assim — não tem como tentar de novo
        } catch (Exception e) {
            log.error("Erro ao processar pedido {}: {}", evento.pedidoId(), e.getMessage());
            // NÃO confirma → Kafka vai retentar a mensagem
        }
    }

    // Consumidor com múltiplas partições em paralelo
    @KafkaListener(
        topics = "pedidos-alta-prioridade",
        groupId = "processador-alta-prioridade",
        concurrency = "3"  // 3 threads consumindo em paralelo
    )
    public void consumirAltaPrioridade(PedidoCriadoEvent evento) {
        processarComPrioridade(evento);
    }
}
```

### docker-compose para Kafka em desenvolvimento

```yaml
version: '3.9'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: true
    depends_on:
      - zookeeper

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    depends_on:
      - kafka
```

---

## Apache Spark com Java

Spark é uma engine de processamento distribuído para Big Data. Processa datasets que não cabem na memória de uma única máquina.

### Dependência

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql_2.13</artifactId>
    <version>3.5.0</version>
</dependency>
```

### Processamento com DataFrame

```java
public class ProcessadorVendas {

    public void analisarVendas() {
        // Inicializa Spark Session
        SparkSession spark = SparkSession.builder()
                .appName("AnalisadorVendas")
                .master("local[*]")  // Em produção: "spark://master:7077"
                .getOrCreate();

        // Carrega dados (pode ser S3, HDFS, banco, Kafka, etc.)
        Dataset<Row> vendas = spark.read()
                .option("header", "true")
                .option("inferSchema", "true")
                .csv("s3://meu-bucket/vendas-2025/*.csv");

        // Schema: data, produto, categoria, vendedor, valor, quantidade

        // Transformações (lazy — só executam quando há uma ação)
        Dataset<Row> vendasFiltradas = vendas
                .filter(col("data").between("2025-01-01", "2025-12-31"))
                .filter(col("valor").gt(0));

        // Agregação: vendas por categoria
        Dataset<Row> porCategoria = vendasFiltradas
                .groupBy("categoria")
                .agg(
                    sum("valor").alias("total_vendas"),
                    count("*").alias("qtd_vendas"),
                    avg("valor").alias("ticket_medio")
                )
                .orderBy(col("total_vendas").desc());

        porCategoria.show(20);

        // Salva resultado
        porCategoria.write()
                .mode(SaveMode.Overwrite)
                .parquet("s3://meu-bucket/resultados/vendas-por-categoria/");

        // SQL sobre DataFrames
        vendasFiltradas.createOrReplaceTempView("vendas");
        Dataset<Row> top10Vendedores = spark.sql(
            "SELECT vendedor, SUM(valor) as total " +
            "FROM vendas " +
            "GROUP BY vendedor " +
            "ORDER BY total DESC " +
            "LIMIT 10"
        );
        top10Vendedores.show();

        spark.stop();
    }
}
```

---

## gRPC em Java

gRPC é um framework de RPC (Remote Procedure Call) de alta performance que usa Protocol Buffers para serialização binária. Muito mais eficiente que JSON/REST.

### Dependência

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
```

### Definição do contrato em Protocol Buffers

```protobuf
// src/main/proto/produto.proto
syntax = "proto3";

package com.minhaapp.produto;

option java_package = "com.minhaapp.produto.grpc";
option java_outer_classname = "ProdutoProto";

service ProdutoService {
  // Unário: uma requisição, uma resposta
  rpc BuscarPorId (BuscarProdutoRequest) returns (ProdutoResponse);

  // Server Streaming: uma requisição, N respostas (stream)
  rpc ListarPorCategoria (ListarRequest) returns (stream ProdutoResponse);

  // Client Streaming: N requisições, uma resposta
  rpc CriarEmLote (stream CriarProdutoRequest) returns (ResultadoLote);

  // Bidirectional Streaming: N requisições, N respostas
  rpc AtualizarPrecos (stream AtualizacaoPreco) returns (stream ResultadoAtualizacao);
}

message BuscarProdutoRequest {
  int64 id = 1;
}

message ListarRequest {
  string categoria = 1;
  int32 pagina = 2;
  int32 tamanho = 3;
}

message ProdutoResponse {
  int64 id = 1;
  string nome = 2;
  double preco = 3;
  string categoria = 4;
  bool ativo = 5;
}

message CriarProdutoRequest {
  string nome = 1;
  double preco = 2;
  string categoria = 3;
}

message ResultadoLote {
  int32 criados = 1;
  int32 falhas = 2;
  repeated string erros = 3;
}

message AtualizacaoPreco {
  int64 produto_id = 1;
  double novo_preco = 2;
}

message ResultadoAtualizacao {
  int64 produto_id = 1;
  bool sucesso = 2;
  string mensagem = 3;
}
```

### Implementação do servidor gRPC com Spring Boot

```java
@GrpcService
@RequiredArgsConstructor
@Slf4j
public class ProdutoGrpcService extends ProdutoServiceGrpc.ProdutoServiceImplBase {

    private final ProdutoRepository repository;
    private final ProdutoMapper mapper;

    @Override
    public void buscarPorId(BuscarProdutoRequest request,
                            StreamObserver<ProdutoResponse> responseObserver) {
        try {
            repository.findById(request.getId())
                    .ifPresentOrElse(
                            produto -> {
                                responseObserver.onNext(mapper.toGrpc(produto));
                                responseObserver.onCompleted();
                            },
                            () -> responseObserver.onError(
                                Status.NOT_FOUND
                                    .withDescription("Produto " + request.getId() + " não encontrado")
                                    .asRuntimeException()
                            )
                    );
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.getMessage()).asRuntimeException()
            );
        }
    }

    @Override
    public void listarPorCategoria(ListarRequest request,
                                   StreamObserver<ProdutoResponse> responseObserver) {
        // Server streaming: envia múltiplos produtos
        repository.findByCategoria(request.getCategoria())
                .stream()
                .map(mapper::toGrpc)
                .forEach(responseObserver::onNext);

        responseObserver.onCompleted();
    }

    @Override
    public StreamObserver<CriarProdutoRequest> criarEmLote(
            StreamObserver<ResultadoLote> responseObserver) {

        // Client streaming: recebe stream de requisições
        return new StreamObserver<>() {
            private int criados = 0;
            private int falhas = 0;
            private final List<String> erros = new ArrayList<>();

            @Override
            public void onNext(CriarProdutoRequest request) {
                try {
                    repository.save(mapper.fromGrpc(request));
                    criados++;
                } catch (Exception e) {
                    falhas++;
                    erros.add("Falha em '" + request.getNome() + "': " + e.getMessage());
                }
            }

            @Override
            public void onError(Throwable t) {
                log.error("Erro no streaming de criação: {}", t.getMessage());
            }

            @Override
            public void onCompleted() {
                responseObserver.onNext(ResultadoLote.newBuilder()
                        .setCriados(criados)
                        .setFalhas(falhas)
                        .addAllErros(erros)
                        .build());
                responseObserver.onCompleted();
            }
        };
    }
}
```

```yaml
# application.yml — configuração do servidor gRPC
grpc:
  server:
    port: 9090
```

---

## GraphQL com Spring for GraphQL

### Dependência

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
```

### Schema GraphQL

```graphql
# src/main/resources/graphql/schema.graphqls
type Query {
    produto(id: ID!): Produto
    produtos(categoria: String, pagina: Int = 0, tamanho: Int = 20): [Produto!]!
}

type Mutation {
    criarProduto(input: CriarProdutoInput!): Produto!
    atualizarPreco(id: ID!, novoPreco: Float!): Produto!
}

type Subscription {
    precoAtualizado(categoria: String): Produto!
}

type Produto {
    id: ID!
    nome: String!
    preco: Float!
    categoria: String!
    itens: [ItemEstoque!]!  # Campo resolvido separadamente (evita N+1)
}

type ItemEstoque {
    quantidade: Int!
    localizacao: String!
}

input CriarProdutoInput {
    nome: String!
    preco: Float!
    categoria: String!
}
```

### Controller GraphQL

```java
@Controller
@RequiredArgsConstructor
public class ProdutoGraphQLController {

    private final ProdutoService produtoService;
    private final EstoqueService estoqueService;

    @QueryMapping
    public Produto produto(@Argument Long id) {
        return produtoService.buscarPorId(id)
                .orElseThrow(() -> new GraphQLException("Produto não encontrado: " + id));
    }

    @QueryMapping
    public List<Produto> produtos(
            @Argument String categoria,
            @Argument int pagina,
            @Argument int tamanho) {
        return produtoService.listar(categoria, pagina, tamanho);
    }

    @MutationMapping
    public Produto criarProduto(@Argument CriarProdutoInput input) {
        return produtoService.criar(input);
    }

    // BatchMapping: resolve o campo itens de múltiplos produtos de uma vez
    // Evita o problema N+1 do GraphQL
    @BatchMapping
    public Map<Produto, List<ItemEstoque>> itens(List<Produto> produtos) {
        Set<Long> ids = produtos.stream().map(Produto::getId).collect(Collectors.toSet());
        Map<Long, List<ItemEstoque>> itensPorProduto = estoqueService.buscarPorProdutos(ids);

        return produtos.stream().collect(Collectors.toMap(
                p -> p,
                p -> itensPorProduto.getOrDefault(p.getId(), List.of())
        ));
    }
}
```

---

## Exemplo completo: Produtor e Consumidor Kafka com Spring Boot

### Estrutura do projeto

```
kafka-demo/
├── src/main/java/com/minhaapp/
│   ├── config/
│   │   └── KafkaConfig.java
│   ├── event/
│   │   └── PedidoCriadoEvent.java
│   ├── producer/
│   │   └── PedidoProducer.java
│   ├── consumer/
│   │   └── PedidoConsumer.java
│   └── controller/
│       └── PedidoController.java
└── src/main/resources/
    └── application.yml
```

### KafkaConfig

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    public NewTopic topicPedidosCriados() {
        return TopicBuilder.name("pedidos-criados")
                .partitions(3)
                .replicas(1)
                .build();
    }

    @Bean
    public NewTopic topicPedidosProcessados() {
        return TopicBuilder.name("pedidos-processados")
                .partitions(3)
                .replicas(1)
                .build();
    }

    // Consumer factory com Acknowledgment manual (controle preciso de offset)
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, PedidoCriadoEvent>
    kafkaListenerContainerFactory(ConsumerFactory<String, PedidoCriadoEvent> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, PedidoCriadoEvent> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        // Retry com backoff exponencial
        factory.setCommonErrorHandler(new DefaultErrorHandler(
                new FixedBackOff(1000L, 3L) // 3 retries, 1s de intervalo
        ));

        return factory;
    }
}
```

### Fluxo completo end-to-end

```java
// Controller que dispara o fluxo
@RestController
@RequestMapping("/api/pedidos")
@RequiredArgsConstructor
public class PedidoController {

    private final PedidoProducer producer;

    @PostMapping
    @ResponseStatus(HttpStatus.ACCEPTED)
    public Map<String, String> criarPedido(@RequestBody CriarPedidoRequest request) {
        String pedidoId = UUID.randomUUID().toString();

        PedidoCriadoEvent evento = new PedidoCriadoEvent(
                pedidoId,
                request.clienteId(),
                request.valorTotal(),
                request.itens(),
                Instant.now()
        );

        producer.publicar(evento);

        return Map.of(
                "pedidoId", pedidoId,
                "status", "PROCESSANDO",
                "mensagem", "Pedido recebido e sendo processado"
        );
    }
}

// Produtor
@Service
@RequiredArgsConstructor
@Slf4j
public class PedidoProducer {

    private final KafkaTemplate<String, PedidoCriadoEvent> kafkaTemplate;

    public void publicar(PedidoCriadoEvent evento) {
        kafkaTemplate.send("pedidos-criados", evento.pedidoId(), evento)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Falha ao publicar evento: {}", ex.getMessage());
                    } else {
                        log.info("Evento publicado com sucesso: pedido={}",
                                evento.pedidoId());
                    }
                });
    }
}

// Consumidor
@Component
@RequiredArgsConstructor
@Slf4j
public class PedidoConsumer {

    private final PedidoProcessorService processor;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(
        topics = "pedidos-criados",
        groupId = "${spring.kafka.consumer.group-id}"
    )
    public void processar(
            @Payload PedidoCriadoEvent evento,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topico,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int particao,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        log.info("Processando pedido={}, topic={}, partition={}, offset={}",
                evento.pedidoId(), topico, particao, offset);

        try {
            ResultadoProcessamento resultado = processor.processar(evento);

            // Publica resultado no próximo tópico do pipeline
            kafkaTemplate.send("pedidos-processados", evento.pedidoId(), resultado);

            ack.acknowledge(); // Confirma processamento
            log.info("Pedido {} processado com sucesso", evento.pedidoId());

        } catch (Exception e) {
            log.error("Erro ao processar pedido {}: {}", evento.pedidoId(), e.getMessage(), e);
            // Não confirma → DefaultErrorHandler fará retry automático
            throw e;
        }
    }
}
```

```bash
# Subir Kafka localmente para testar
docker compose up -d kafka zookeeper kafka-ui

# Interface Kafka UI: http://localhost:8090
# Ver tópicos, mensagens, consumer groups e offsets em tempo real

# Publicar mensagem de teste via CLI
docker exec -it kafka kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic pedidos-criados

# Consumir mensagens do início
docker exec -it kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic pedidos-criados \
  --from-beginning

# Listar consumer groups e seus offsets
docker exec -it kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group minha-app
```
