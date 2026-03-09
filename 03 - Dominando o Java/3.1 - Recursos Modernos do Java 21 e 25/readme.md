# 3.1 — Recursos Modernos do Java 21 e 25

Java evoluiu radicalmente nos últimos anos. O **Java 21 LTS** (setembro de 2023) e o **Java 25 LTS** (setembro de 2025) trouxeram recursos que tornam o código mais expressivo, seguro e eficiente. Este guia explora cada feature com o problema que resolve, o código antes e depois, e os casos onde você deve evitá-las em produção.

---

## Contexto: Java 21 LTS e Java 25 LTS

**Java 21 (LTS):**
- Virtual Threads (Project Loom) — finalizado
- Record Patterns, Pattern Matching para switch — finalizado
- Sequenced Collections
- String Templates (preview)

**Java 25 (LTS):**
- Scoped Values — finalizado
- Unnamed Variables e Patterns — finalizado
- Flexible Constructor Bodies
- Module Import Declarations
- Primitive Types in Patterns

> **LTS (Long-Term Support):** versões com suporte por 8+ anos — use em produção. Versões intermediárias recebem suporte por apenas 6 meses.

---

## var — Inferência de tipo local

Disponível desde Java 10. O compilador infere o tipo da variável com base na expressão do lado direito.

**Problema que resolve:** verbosidade excessiva em tipos genéricos longos.

```java
// ANTES — verboso
HashMap<String, List<PedidoDTO>> pedidosPorCliente = new HashMap<String, List<PedidoDTO>>();
Iterator<Map.Entry<String, List<PedidoDTO>>> iterator = pedidosPorCliente.entrySet().iterator();

// DEPOIS — com var
var pedidosPorCliente = new HashMap<String, List<PedidoDTO>>();
var iterator = pedidosPorCliente.entrySet().iterator();
```

### Quando usar `var`

```java
// BOM — o tipo é óbvio pela inicialização
var nome = "João";
var lista = new ArrayList<Pedido>();
var servico = new PagamentoService();
var resultado = calcularTotal(itens);

// BOM — evita repetição óbvia
var pedido = pedidoRepository.findById(id).orElseThrow();

// BOM — em loops onde o tipo é claro
for (var pedido : pedidos) {
    processar(pedido);
}

// BOM — resultado de streams onde o tipo seria gigantesco
var agrupado = pedidos.stream()
    .collect(Collectors.groupingBy(Pedido::getCliente,
             Collectors.summingDouble(Pedido::getValor)));
```

### Quando NÃO usar `var`

```java
// RUIM — tipo não é óbvio, precisa do contexto do método
var resultado = processar(dados);    // o que é 'resultado'?
var x = obterConfig();               // que tipo retorna?

// RUIM — com literais numéricos (pode mudar o tipo implicitamente)
var numero = 1;        // int, não long — pode surpreender
var pi = 3.14;         // double, não float

// RUIM — campos de classe (var só funciona em variáveis locais)
// Não compila:
// private var nome = "João";

// RUIM — parâmetros de método (não suportado)
// Não compila:
// public void processar(var dado) {}

// RUIM — quando o tipo é importante para o leitor entender o contrato
var pagamento = criarPagamento();   // é Pagamento? PagamentoDTO? PagamentoCommand?
// Prefira:
Pagamento pagamento = criarPagamento();
```

> **Regra prática:** use `var` quando o tipo for óbvio da inicialização. Se alguém precisar inspecionar o retorno do método para entender a variável, escreva o tipo explicitamente.

---

## Switch Expressions (Java 14+)

O `switch` tradicional era propenso a erros (esquecer `break`), não retornava valor e era verboso.

**Problema que resolve:** switch como expressão que retorna valor, sem fall-through acidental.

```java
// ANTES — switch tradicional (verboso e perigoso)
String descricao;
switch (status) {
    case PENDENTE:
        descricao = "Aguardando pagamento";
        break;
    case PAGO:
        descricao = "Pagamento confirmado";
        break;
    case CANCELADO:
        descricao = "Pedido cancelado";
        break;
    default:
        descricao = "Status desconhecido";
        break;
}

// DEPOIS — switch expression com ->
String descricao = switch (status) {
    case PENDENTE  -> "Aguardando pagamento";
    case PAGO      -> "Pagamento confirmado";
    case CANCELADO -> "Pedido cancelado";
};
// Se for enum exaustivo, o default é desnecessário
```

### Com `yield` para blocos complexos

```java
double desconto = switch (categoria) {
    case VIP -> 0.20;
    case PREMIUM -> 0.15;
    case COMUM -> {
        // lógica mais complexa exige yield em vez de ->
        double base = 0.05;
        double extra = calcularExtraComum(cliente);
        yield base + extra;   // yield retorna o valor do bloco
    }
    case NOVO -> 0.0;
};
```

### Retornando objetos

```java
// Switch expression pode retornar qualquer coisa
Notificacao notificacao = switch (evento) {
    case PAGAMENTO_APROVADO -> new NotificacaoEmail("Pagamento aprovado!");
    case PAGAMENTO_RECUSADO -> new NotificacaoSMS("Pagamento recusado.");
    case ENTREGA_REALIZADA  -> new NotificacaoPush("Seu pedido chegou!");
};

// Pattern Matching no switch (Java 21)
String formato = switch (objeto) {
    case Integer i -> "Inteiro: " + i;
    case String s  -> "Texto com " + s.length() + " caracteres";
    case null      -> "Nulo";
    default        -> "Tipo desconhecido: " + objeto.getClass().getSimpleName();
};
```

### Quando NÃO usar em produção

Switch expression não apresenta riscos em produção — é uma melhoria direta. Use sem receio. O cuidado é garantir exaustividade: para `enum`, o compilador garante; para outros tipos, adicione `default`.

---

## Text Blocks (Java 15+)

Strings multi-linha sem escape manual de aspas e quebras de linha.

**Problema que resolve:** SQL, JSON, HTML e XML embutidos em código Java são ilegíveis com concatenação tradicional.

```java
// ANTES — ilegível
String sql = "SELECT u.id, u.nome, u.email\n" +
             "FROM usuarios u\n" +
             "JOIN pedidos p ON p.usuario_id = u.id\n" +
             "WHERE u.ativo = true\n" +
             "  AND p.criado_em >= :dataInicio\n" +
             "ORDER BY u.nome ASC";

String json = "{\n" +
              "  \"nome\": \"João\",\n" +
              "  \"email\": \"joao@email.com\",\n" +
              "  \"ativo\": true\n" +
              "}";

// DEPOIS — text blocks
String sql = """
        SELECT u.id, u.nome, u.email
        FROM usuarios u
        JOIN pedidos p ON p.usuario_id = u.id
        WHERE u.ativo = true
          AND p.criado_em >= :dataInicio
        ORDER BY u.nome ASC
        """;

String json = """
        {
          "nome": "João",
          "email": "joao@email.com",
          "ativo": true
        }
        """;
```

### Indentação inteligente

O Java remove automaticamente a indentação comum a todas as linhas:

```java
// A indentação "real" é determinada pela posição do """ de fechamento
// Estas três formas geram strings diferentes:

// Forma 1: """ no final — remove toda a indentação relativa
String texto = """
        Linha 1
        Linha 2
        """;
// Resultado: "Linha 1\nLinha 2\n"

// Forma 2: """ mais à esquerda — preserva indentação adicional
String texto = """
        Linha 1
        Linha 2
""";
// Resultado: "        Linha 1\n        Linha 2\n"
```

### formatted() para interpolação

```java
String template = """
        Olá, %s!
        Seu pedido #%d foi confirmado.
        Valor total: R$ %.2f
        Previsão de entrega: %s
        """.formatted(cliente.getNome(), pedido.getId(),
                      pedido.getValor(), pedido.getPrevisaoEntrega());
```

### Quando NÃO usar

- Strings de uma única linha — use aspas normais
- Strings que precisam de trim no final — text blocks incluem a newline final por padrão

---

## Record Classes (Java 16+)

Records são classes imutáveis para transportar dados. Eliminam boilerplate de `equals()`, `hashCode()`, `toString()` e construtores.

**Problema que resolve:** DTOs, Value Objects e POJOs com toneladas de código repetitivo.

```java
// ANTES — classe DTO manual (55+ linhas de boilerplate)
public final class EnderecoDTO {
    private final String logradouro;
    private final String numero;
    private final String cep;
    private final String cidade;
    private final String estado;

    public EnderecoDTO(String logradouro, String numero, String cep,
                       String cidade, String estado) {
        this.logradouro = logradouro;
        this.numero = numero;
        this.cep = cep;
        this.cidade = cidade;
        this.estado = estado;
    }

    public String getLogradouro() { return logradouro; }
    public String getNumero()     { return numero; }
    public String getCep()        { return cep; }
    public String getCidade()     { return cidade; }
    public String getEstado()     { return estado; }

    @Override
    public boolean equals(Object o) { /* ... 10 linhas ... */ }

    @Override
    public int hashCode() { /* ... 5 linhas ... */ }

    @Override
    public String toString() {
        return "EnderecoDTO{logradouro='" + logradouro + "', ...}";
    }
}

// DEPOIS — record (1 linha)
public record EnderecoDTO(
    String logradouro,
    String numero,
    String cep,
    String cidade,
    String estado
) {}
```

### Compact Constructor — validação

```java
public record Dinheiro(BigDecimal valor, String moeda) {

    // Compact constructor — sem parâmetros, executa antes da atribuição automática
    public Dinheiro {
        Objects.requireNonNull(valor, "valor não pode ser nulo");
        Objects.requireNonNull(moeda, "moeda não pode ser nula");

        if (valor.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Valor não pode ser negativo: " + valor);
        }

        // Normalização — os campos ainda não foram atribuídos aqui
        moeda = moeda.toUpperCase().trim();
    }
}
```

### Métodos adicionais em records

```java
public record Periodo(LocalDate inicio, LocalDate fim) {

    // Validação no compact constructor
    public Periodo {
        if (fim.isBefore(inicio)) {
            throw new IllegalArgumentException("Fim não pode ser antes do início");
        }
    }

    // Métodos de negócio são permitidos
    public long diasDeDuracao() {
        return ChronoUnit.DAYS.between(inicio, fim);
    }

    public boolean contem(LocalDate data) {
        return !data.isBefore(inicio) && !data.isAfter(fim);
    }

    public boolean sobrepoeCom(Periodo outro) {
        return !this.fim.isBefore(outro.inicio) && !outro.fim.isBefore(this.inicio);
    }

    // "with" pattern — retorna nova instância com campo modificado (imutável)
    public Periodo comInicio(LocalDate novoInicio) {
        return new Periodo(novoInicio, this.fim);
    }

    public Periodo comFim(LocalDate novoFim) {
        return new Periodo(this.inicio, novoFim);
    }
}
```

### Records com interfaces

```java
// Records podem implementar interfaces
public sealed interface Resultado<T> permits Resultado.Sucesso, Resultado.Falha {

    record Sucesso<T>(T valor) implements Resultado<T> {}
    record Falha<T>(String mensagem, Exception causa) implements Resultado<T> {}
}

// Uso
Resultado<Pedido> resultado = criarPedido(command);
String resposta = switch (resultado) {
    case Resultado.Sucesso<Pedido> s -> "Pedido criado: " + s.valor().getId();
    case Resultado.Falha<Pedido> f  -> "Erro: " + f.mensagem();
};
```

### Quando NÃO usar records

- Entidades JPA/Hibernate — precisam ser mutáveis e ter construtor padrão
- Classes com lógica complexa de negócio — prefira uma classe normal
- Quando você precisa de herança de classe (records não podem estender outras classes)

---

## Pattern Matching para instanceof (Java 16+)

Elimina o cast manual após verificação de tipo.

**Problema que resolve:** código verbose e propenso a erros com `instanceof` + cast.

```java
// ANTES — verboso e repetitivo
Object objeto = obterObjeto();
if (objeto instanceof String) {
    String texto = (String) objeto;   // cast redundante
    System.out.println(texto.toUpperCase());
}

// DEPOIS — pattern matching
if (objeto instanceof String texto) {
    System.out.println(texto.toUpperCase());
}

// Com condição adicional (guard)
if (objeto instanceof String texto && texto.length() > 10) {
    System.out.println("Texto longo: " + texto);
}
```

### Uso real em método equals

```java
public record Ponto(int x, int y) {

    @Override
    public boolean equals(Object obj) {
        // ANTES
        // if (!(obj instanceof Ponto)) return false;
        // Ponto outro = (Ponto) obj;
        // return this.x == outro.x && this.y == outro.y;

        // DEPOIS — pattern matching
        return obj instanceof Ponto outro
            && this.x == outro.x
            && this.y == outro.y;
    }
}
```

---

## Sealed Classes and Interfaces (Java 17+)

Sealed classes permitem definir **exatamente quais classes podem estender ou implementar** um tipo, criando hierarquias fechadas.

**Problema que resolve:** hierarquias de tipos onde você quer exaustividade garantida pelo compilador (sem `default` esquecido).

```java
// Definindo a hierarquia selada
public sealed interface Forma
    permits Circulo, Retangulo, Triangulo {}

public record Circulo(double raio) implements Forma {}
public record Retangulo(double largura, double altura) implements Forma {}
public record Triangulo(double base, double altura) implements Forma {}

// Se tentar criar:
// public record Hexagono(double lado) implements Forma {}
// Erro: Hexagono não está na lista de 'permits'
```

### Exhaustive switch — o compilador garante cobertura

```java
// O compilador sabe todos os subtipos de Forma
// Se você esquecer um case, ERRO DE COMPILAÇÃO
double calcularArea(Forma forma) {
    return switch (forma) {
        case Circulo c     -> Math.PI * c.raio() * c.raio();
        case Retangulo r   -> r.largura() * r.altura();
        case Triangulo t   -> (t.base() * t.altura()) / 2;
        // Sem default — se adicionar Hexagono sem tratar aqui, não compila
    };
}
```

### Hierarquia de eventos de domínio

```java
// Modelo de domínio rico com sealed interfaces
public sealed interface EventoPedido
    permits EventoPedido.Criado, EventoPedido.Pago,
            EventoPedido.Enviado, EventoPedido.Entregue, EventoPedido.Cancelado {

    record Criado(UUID pedidoId, LocalDateTime em, String cliente) implements EventoPedido {}
    record Pago(UUID pedidoId, LocalDateTime em, BigDecimal valor) implements EventoPedido {}
    record Enviado(UUID pedidoId, LocalDateTime em, String codigoRastreio) implements EventoPedido {}
    record Entregue(UUID pedidoId, LocalDateTime em) implements EventoPedido {}
    record Cancelado(UUID pedidoId, LocalDateTime em, String motivo) implements EventoPedido {}
}

// Processador de eventos — compilador garante que todos os casos são tratados
public void processarEvento(EventoPedido evento) {
    switch (evento) {
        case EventoPedido.Criado e -> {
            log.info("Pedido {} criado por {}", e.pedidoId(), e.cliente());
            notificarCliente(e.cliente(), "Pedido recebido!");
        }
        case EventoPedido.Pago e -> {
            log.info("Pedido {} pago: R$ {}", e.pedidoId(), e.valor());
            iniciarSeparacao(e.pedidoId());
        }
        case EventoPedido.Enviado e -> {
            log.info("Pedido {} enviado. Rastreio: {}", e.pedidoId(), e.codigoRastreio());
            notificarRastreio(e.pedidoId(), e.codigoRastreio());
        }
        case EventoPedido.Entregue e -> finalizarPedido(e.pedidoId());
        case EventoPedido.Cancelado e -> processarCancelamento(e.pedidoId(), e.motivo());
    }
}
```

### Quando NÃO usar em produção

- Hierarquias que precisam ser extensíveis por clientes/usuários do framework — use interfaces abertas
- Quando você quer permitir que outras equipes/libs estendam o tipo

---

## Scoped Values (Java 21+)

Substituto mais seguro e eficiente que `ThreadLocal` para compartilhar dados imutáveis em Virtual Threads.

**Problema que resolve:** `ThreadLocal` vaza memória com Virtual Threads (são milhares delas) e permite modificação acidental.

```java
// ANTES — ThreadLocal (problemático com Virtual Threads)
public class ContextoUsuario {
    private static final ThreadLocal<Usuario> usuarioAtual = new ThreadLocal<>();

    public static void set(Usuario usuario) { usuarioAtual.set(usuario); }
    public static Usuario get() { return usuarioAtual.get(); }
    public static void clear() { usuarioAtual.remove(); }  // Não pode esquecer isso!
}

// DEPOIS — ScopedValue (Java 21)
public class ContextoUsuario {
    public static final ScopedValue<Usuario> USUARIO_ATUAL = ScopedValue.newInstance();
}

// Uso: o valor só existe dentro do escopo definido
ScopedValue.where(ContextoUsuario.USUARIO_ATUAL, usuarioLogado)
    .run(() -> {
        // Dentro deste bloco, USUARIO_ATUAL está disponível
        processarRequisicao();
        // Ao sair do bloco, o valor some automaticamente — sem vazamento
    });

// Leitura segura
public void processarRequisicao() {
    var usuario = ContextoUsuario.USUARIO_ATUAL.get();
    log.info("Processando para: {}", usuario.getNome());
}
```

---

## Unnamed Variables (Java 22+)

Permite usar `_` para ignorar variáveis que você precisa declarar mas não vai usar.

**Problema que resolve:** warnings de "variável não utilizada" e código que deixa claro a intenção de ignorar.

```java
// ANTES — variável obrigatória mas não usada
try {
    int resultado = Integer.parseInt(texto);
    return true;
} catch (NumberFormatException e) {  // 'e' não é usado
    return false;
}

// DEPOIS — unnamed variable
try {
    Integer.parseInt(texto);
    return true;
} catch (NumberFormatException _) {  // explicitamente ignorado
    return false;
}

// Em loops onde o índice não importa
for (int _ : lista) {
    contador++;
}

// Em pattern matching quando o valor não interessa
if (objeto instanceof String _) {
    // só me importa que é uma String, não o valor
    tratarString();
}

// Em lambdas
mapa.forEach((_, valor) -> processar(valor));
```

---

## Virtual Threads (Java 21)

Virtual Threads são threads "leves" gerenciadas pela JVM, criadas em milhares sem o custo das threads de sistema operacional.

**Problema que resolve:** escalabilidade de I/O bloqueante sem precisar de programação reativa complexa.

### O problema que Virtual Threads resolvem

```
Thread de S.O. tradicional:
- ~1MB de memória de stack
- Custo alto de criação e troca de contexto
- Servidor com 16GB de RAM: máximo ~16.000 threads simultâneas

Virtual Thread:
- ~few KB de memória
- Gerenciada pela JVM sobre um pool de carrier threads
- Servidor com 16GB de RAM: milhões de virtual threads simultâneas
```

### Como criar Virtual Threads

```java
// Forma 1: Thread.ofVirtual()
Thread vt = Thread.ofVirtual()
    .name("vt-processamento")
    .start(() -> {
        // Código que pode fazer I/O bloqueante com segurança
        var resultado = repositorio.buscarDados(id);   // bloqueia, mas não desperdiça
        processar(resultado);
    });

// Forma 2: ExecutorService com virtual threads (mais comum)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var pedido : listaDePedidos) {
        executor.submit(() -> processarPedido(pedido));
    }
}   // aguarda todas terminarem ao fechar o try

// Forma 3: com structured concurrency (Java 21+)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var futuroPagamento  = scope.fork(() -> verificarPagamento(pedidoId));
    var futuroEstoque    = scope.fork(() -> verificarEstoque(pedidoId));
    var futuroFraude     = scope.fork(() -> analisarFraude(pedidoId));

    scope.join().throwIfFailed();   // aguarda todos e propaga erros

    var pagamento = futuroPagamento.get();
    var estoque   = futuroEstoque.get();
    var fraude    = futuroFraude.get();

    return new ResultadoVerificacao(pagamento, estoque, fraude);
}
```

### Quando usar Virtual Threads

```java
// IDEAL: operações de I/O bloqueante
// - Chamadas HTTP a APIs externas
// - Consultas ao banco de dados
// - Leitura/escrita de arquivos
// - Chamadas a filas de mensagens (Kafka, RabbitMQ)

@GetMapping("/pedido/{id}/detalhes")
public PedidoDetalhes obterDetalhes(@PathVariable UUID id) {
    // Com Virtual Threads no Spring Boot 3.2+, cada request já roda em uma virtual thread
    // Não precisa de WebFlux/Reactive — código imperativo simples
    var pedido      = pedidoRepo.findById(id).orElseThrow();    // I/O bloqueante — OK
    var pagamento   = pagamentoClient.buscar(id);                // HTTP bloqueante — OK
    var rastreio    = logisticaClient.buscar(id);                // HTTP bloqueante — OK

    return new PedidoDetalhes(pedido, pagamento, rastreio);
}
```

### Quando NÃO usar Virtual Threads

```java
// NÃO USAR 1: operações CPU-bound (não há benefício)
// Virtual threads não ajudam quando o CPU está ocupado, não esperando I/O
public BigDecimal calcularSimulacaoMonteCarlo(int iteracoes) {
    // Isso vai no pool de threads tradicional (ForkJoinPool)
    return IntStream.range(0, iteracoes)
        .parallel()
        .mapToDouble(i -> simular())
        .average()
        .orElse(0);
}

// NÃO USAR 2: synchronized com monitor (causa "pinning" da carrier thread)
// PROBLEMÁTICO com virtual threads:
public synchronized void processarComSynchronized() {
    // A virtual thread "prega" (pins) a carrier thread durante todo o bloqueio
    buscarDoBanco();  // I/O aqui bloqueia a carrier thread inteira — perde o benefício
}

// CORRETO: usar ReentrantLock em vez de synchronized quando há I/O interno
private final ReentrantLock lock = new ReentrantLock();

public void processarComLock() {
    lock.lock();
    try {
        buscarDoBanco();  // Virtual thread pode ser suspensa aqui — OK
    } finally {
        lock.unlock();
    }
}
```

### Habilitando no Spring Boot 3.2+

```java
// application.properties
// spring.threads.virtual.enabled=true

// Ou via código
@Bean
public TomcatProtocolHandlerCustomizer<?> virtualThreadsCustomizer() {
    return protocolHandler ->
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}
```

---

## Resumo visual das features por versão

| Feature                    | Versão final | Use em produção? | Principal benefício                     |
|----------------------------|--------------|------------------|-----------------------------------------|
| `var`                      | Java 10      | Sim              | Menos verbosidade em tipos longos       |
| Switch Expressions         | Java 14      | Sim              | Switch que retorna valor, sem fall-through |
| Text Blocks                | Java 15      | Sim              | Strings multi-linha legíveis            |
| Records                    | Java 16      | Sim              | DTOs sem boilerplate                    |
| Pattern Matching instanceof| Java 16      | Sim              | Sem cast manual                         |
| Sealed Classes             | Java 17      | Sim              | Hierarquias fechadas e exaustivas       |
| Virtual Threads            | Java 21      | Sim (I/O)        | Escalabilidade sem reativo              |
| Scoped Values              | Java 21      | Sim              | ThreadLocal seguro para Virtual Threads |
| Unnamed Variables (`_`)    | Java 22      | Sim              | Intenção explícita de ignorar           |
