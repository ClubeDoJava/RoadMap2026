# 3.2 — Collections Framework e Streams

O Collections Framework e a Streams API são pilares do Java moderno. Dominar essas ferramentas permite escrever código conciso, expressivo e eficiente para manipular dados.

---

## Collections Framework

### Hierarquia completa

```
java.util.Collection<E>
├── List<E>               (elementos ordenados, duplicatas permitidas)
│   ├── ArrayList<E>
│   ├── LinkedList<E>     (também implementa Deque)
│   ├── Vector<E>         (legado, thread-safe mas lento)
│   └── CopyOnWriteArrayList<E>  (thread-safe para leitura frequente)
│
├── Set<E>                (sem duplicatas)
│   ├── HashSet<E>
│   ├── LinkedHashSet<E>
│   └── SortedSet<E>
│       └── TreeSet<E>
│
└── Queue<E>              (fila FIFO)
    ├── PriorityQueue<E>
    ├── ArrayDeque<E>     (também implementa Deque)
    └── Deque<E>          (fila dupla — insere/remove nos dois extremos)
        ├── ArrayDeque<E>
        └── LinkedList<E>

java.util.Map<K, V>       (chave → valor, chaves únicas)
├── HashMap<K, V>
├── LinkedHashMap<K, V>
├── TreeMap<K, V>         (ordenado por chave)
├── Hashtable<K, V>       (legado, evitar)
├── WeakHashMap<K, V>
└── ConcurrentHashMap<K, V> (thread-safe, alta performance)
```

---

### List — quando usar cada implementação

#### ArrayList

```java
// Backed por um array dinâmico
// Acesso por índice: O(1)
// Inserção no final: O(1) amortizado
// Inserção no meio: O(n) — precisa deslocar elementos
// Melhor para: acesso aleatório por índice, iteração sequencial

var produtos = new ArrayList<Produto>();
produtos.add(new Produto("Notebook", 3500.0));
produtos.add(new Produto("Mouse", 150.0));

// Acesso por índice — rápido
Produto primeiro = produtos.get(0);

// Busca por elemento — O(n)
int indice = produtos.indexOf(new Produto("Mouse", 150.0));

// Inicializar com capacidade conhecida (evita rehashing)
var lista = new ArrayList<Pedido>(1000);
```

#### LinkedList

```java
// Backed por lista duplamente encadeada
// Acesso por índice: O(n) — percorre desde o início ou fim
// Inserção/remoção nas extremidades: O(1)
// Inserção no meio: O(n) para encontrar, O(1) para inserir
// Melhor para: filas, pilhas, quando insere/remove muito nas extremidades

var fila = new LinkedList<Tarefa>();
fila.addLast(new Tarefa("processar pagamento"));   // enfileira
fila.addLast(new Tarefa("enviar email"));

Tarefa proxima = fila.pollFirst();   // desenfileira
```

**Comparativo prático:**

| Operação            | ArrayList | LinkedList |
|---------------------|-----------|------------|
| `get(i)` por índice | O(1)      | O(n)       |
| `add` no final      | O(1)*     | O(1)       |
| `add(i, e)` no meio | O(n)      | O(n)       |
| `remove(i)` no meio | O(n)      | O(n)       |
| Memória             | Menor     | Maior (ponteiros) |

> **Regra:** na dúvida, use `ArrayList`. `LinkedList` só vale a pena quando você insere/remove muito nas extremidades ou como fila/pilha.

---

### Set — quando usar cada implementação

```java
// HashSet — sem garantia de ordem, operações O(1)
Set<String> tags = new HashSet<>();
tags.add("java");
tags.add("spring");
tags.add("java");   // duplicata ignorada silenciosamente
System.out.println(tags.size());   // 2

// LinkedHashSet — mantém ordem de inserção
Set<String> linguagens = new LinkedHashSet<>();
linguagens.add("Java");
linguagens.add("Python");
linguagens.add("Go");
// Iteração sempre em ordem: Java, Python, Go

// TreeSet — ordenado naturalmente (ou por Comparator)
Set<String> ordenado = new TreeSet<>();
ordenado.add("Banana");
ordenado.add("Abacaxi");
ordenado.add("Caju");
// Iteração: Abacaxi, Banana, Caju

// TreeSet com Comparator personalizado
Set<Produto> porPreco = new TreeSet<>(
    Comparator.comparing(Produto::preco).thenComparing(Produto::nome)
);
```

---

### Map — quando usar cada implementação

```java
// HashMap — acesso O(1), sem ordem garantida
Map<String, Integer> estoque = new HashMap<>();
estoque.put("Notebook", 10);
estoque.put("Mouse", 50);

// Leitura segura com default
int qtd = estoque.getOrDefault("Teclado", 0);

// Atualizar com lógica
estoque.merge("Mouse", 5, Integer::sum);   // adiciona 5 ao valor existente

// Computar se ausente (cache, agrupamento)
Map<String, List<Pedido>> pedidosPorCliente = new HashMap<>();
pedidosPorCliente
    .computeIfAbsent("João", k -> new ArrayList<>())
    .add(novoPedido);

// LinkedHashMap — mantém ordem de inserção (útil para cache LRU)
Map<String, String> cache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
        return size() > 100;   // LRU com máximo 100 entradas
    }
};

// TreeMap — ordenado por chave
Map<LocalDate, List<Pedido>> pedidosPorData = new TreeMap<>();
// Útil para range queries:
pedidosPorData.subMap(dataInicio, dataFim)
              .forEach((data, pedidos) -> processar(data, pedidos));

// ConcurrentHashMap — thread-safe, melhor que Collections.synchronizedMap()
Map<String, Integer> contadores = new ConcurrentHashMap<>();
contadores.compute("requisicoes", (k, v) -> v == null ? 1 : v + 1);
```

---

### Queue e Deque

```java
// ArrayDeque — mais eficiente que LinkedList para filas e pilhas
Deque<Tarefa> pilha = new ArrayDeque<>();
pilha.push(new Tarefa("A"));   // empilha
pilha.push(new Tarefa("B"));
pilha.pop();                   // desempilha B (LIFO)

Deque<Tarefa> fila = new ArrayDeque<>();
fila.offer(new Tarefa("A"));   // enfileira
fila.offer(new Tarefa("B"));
fila.poll();                   // desenfileira A (FIFO)

// PriorityQueue — processa pelo menor elemento primeiro
PriorityQueue<Pedido> filaPrioridade = new PriorityQueue<>(
    Comparator.comparing(Pedido::prioridade)
);
filaPrioridade.offer(new Pedido("P1", 3));
filaPrioridade.offer(new Pedido("P2", 1));
filaPrioridade.offer(new Pedido("P3", 2));

while (!filaPrioridade.isEmpty()) {
    System.out.println(filaPrioridade.poll().nome());
    // P2, P3, P1 — pelo menor valor de prioridade
}
```

---

### Collections utilitárias

```java
// Ordenar
List<String> nomes = new ArrayList<>(List.of("Carlos", "Ana", "Bruno"));
Collections.sort(nomes);                                      // ordena in-place
Collections.sort(nomes, Comparator.reverseOrder());           // ordem inversa
nomes.sort(Comparator.comparing(String::length));             // por comprimento

// Busca binária (lista deve estar ordenada)
int pos = Collections.binarySearch(nomes, "Ana");

// Inverter
Collections.reverse(nomes);

// Embaralhar
Collections.shuffle(nomes);
Collections.shuffle(nomes, new Random(42));   // com semente reprodutível

// Mínimo e máximo
String menor = Collections.min(nomes);
String maior = Collections.max(nomes);

// Wrappers de imutabilidade (lança UnsupportedOperationException em modificações)
List<String>       imutavel   = Collections.unmodifiableList(nomes);
Set<String>        semModif   = Collections.unmodifiableSet(conjunto);
Map<String, Pedido> protegido = Collections.unmodifiableMap(mapa);

// Wrappers thread-safe (mais lentos que ConcurrentHashMap)
List<Pedido> sincronizado = Collections.synchronizedList(new ArrayList<>());

// Coleções vazias (singletons — não alocam memória)
List<String>  vazia  = Collections.emptyList();
Set<Integer>  vazioS = Collections.emptySet();
Map<?, ?>     vazioM = Collections.emptyMap();

// Lista com um único elemento
List<String> unitaria = Collections.singletonList("único");
```

### Coleções imutáveis com factory methods (Java 9+)

```java
// List.of(), Set.of(), Map.of() — imutáveis desde o início
// Não permitem null, são mais compactas em memória
List<String> linguagens = List.of("Java", "Python", "Go", "Rust");
Set<Integer> primos     = Set.of(2, 3, 5, 7, 11, 13);
Map<String, Integer>  idades = Map.of(
    "Ana", 30,
    "Bruno", 25,
    "Carlos", 35
);

// Para Map com muitas entradas, use Map.ofEntries()
Map<String, String> capitais = Map.ofEntries(
    Map.entry("Brasil",    "Brasília"),
    Map.entry("Argentina", "Buenos Aires"),
    Map.entry("Chile",     "Santiago"),
    Map.entry("Peru",      "Lima"),
    Map.entry("Colômbia",  "Bogotá")
);

// Cópia imutável de uma coleção existente (Java 10+)
List<Pedido> copia = List.copyOf(pedidosMutaveis);
```

---

## Functional Interfaces

As interfaces funcionais são a base das lambdas e da Streams API.

```java
// Predicate<T> — testa uma condição, retorna boolean
Predicate<String>  naoVazio    = s -> !s.isBlank();
Predicate<Integer> maiorQueZero = n -> n > 0;
Predicate<Pedido>  pago        = p -> p.status() == Status.PAGO;

// Composição de Predicates
Predicate<String> emailValido = naoVazio.and(s -> s.contains("@"));
Predicate<Integer> negativo   = maiorQueZero.negate();
Predicate<Pedido> naoPago     = pago.negate();

// Function<T, R> — transforma T em R
Function<String, Integer>  comprimento     = String::length;
Function<Pedido, String>   clienteDoPedido = Pedido::cliente;
Function<String, String>   maiusculo       = String::toUpperCase;

// Composição de Functions
Function<String, String> limparEMaiusculo = String::trim
    .andThen(String::toUpperCase);

// BiFunction<T, U, R> — transforma T e U em R
BiFunction<String, Integer, String> repetir = String::repeat;
String resultado = repetir.apply("Java ", 3);   // "Java Java Java "

// Supplier<T> — fornece um valor sem receber argumentos
Supplier<LocalDateTime> agora  = LocalDateTime::now;
Supplier<List<Pedido>>  lista  = ArrayList::new;
Supplier<UUID>          novoId = UUID::randomUUID;

// Consumer<T> — consome um valor, não retorna nada
Consumer<String>  imprimir   = System.out::println;
Consumer<Pedido>  salvar     = pedidoRepository::save;
Consumer<Produto> atualizar  = p -> { p.aplicarDesconto(0.10); notificar(p); };

// Composição de Consumers
Consumer<Pedido> salvarENotificar = salvar.andThen(p -> notificarCliente(p));

// UnaryOperator<T> — especialização de Function<T, T>
UnaryOperator<String> trim     = String::trim;
UnaryOperator<Integer> dobrar  = n -> n * 2;
UnaryOperator<Pedido> aplicarDesconto = p -> p.comDesconto(0.10);
```

### Method References — as quatro formas

```java
// 1. Referência a método estático: Classe::metodoEstatico
Function<String, Integer> parseInteiro = Integer::parseInt;
Comparator<String> comparar            = String::compareTo;

// 2. Referência a método de instância (de um objeto específico)
String prefixo = "BR-";
Predicate<String> temPrefixo = prefixo::startsWith;  // instância 'prefixo'

// 3. Referência a método de instância (de um tipo arbitrário)
// O primeiro parâmetro da lambda se torna a instância chamadora
Function<String, String>      maiusculo = String::toUpperCase;  // s -> s.toUpperCase()
Function<String, Integer>     tamanho   = String::length;       // s -> s.length()
BiFunction<String, String, Boolean> contem = String::contains;  // (s, t) -> s.contains(t)

// 4. Referência a construtor: Classe::new
Supplier<ArrayList<String>>          novaLista   = ArrayList::new;
Function<String, StringBuilder>      novoBuilder = StringBuilder::new;
BiFunction<String, Integer, String>  repetir     = String::new;  // não usual
```

---

## Streams API

Streams permitem processar coleções de forma declarativa — você descreve **o que** quer, não **como** fazer.

### Pipeline: source → intermediate → terminal

```
Fonte (Source)           → Operações Intermediárias  → Operação Terminal
List.of(1,2,3,4,5)          .filter(n -> n % 2 == 0)    .collect(toList())
Files.lines(path)            .map(String::toUpperCase)   .forEach(System.out::println)
IntStream.range(1, 100)      .sorted()                   .count()
                             .distinct()                  .reduce(0, Integer::sum)
                             .limit(10)                   .findFirst()
```

> **Importante:** operações intermediárias são **lazy** — só executam quando uma operação terminal é chamada. Sem terminal, nenhum processamento acontece.

### Criando Streams

```java
// A partir de coleção
Stream<Pedido> stream1 = pedidos.stream();
Stream<Pedido> paralelo = pedidos.parallelStream();

// A partir de valores diretos
Stream<String> nomes = Stream.of("Ana", "Bruno", "Carlos");

// Stream vazio
Stream<String> vazio = Stream.empty();

// Stream infinita (com limite obrigatório)
Stream<Integer> pares = Stream.iterate(0, n -> n + 2);
Stream<Double>  aleatorios = Stream.generate(Math.random());

// Streams numéricas especializadas (evitam boxing/unboxing)
IntStream  inteiros  = IntStream.range(1, 101);       // 1 a 100
LongStream longos    = LongStream.rangeClosed(1, 10); // 1 a 10 inclusive
DoubleStream doubles = DoubleStream.of(1.5, 2.5, 3.5);

// A partir de arquivo (fecha automaticamente)
try (Stream<String> linhas = Files.lines(Path.of("dados.csv"))) {
    linhas.filter(l -> !l.isBlank())
          .map(l -> l.split(","))
          .forEach(campos -> processar(campos));
}
```

### Operações intermediárias

```java
List<Pedido> pedidos = obterPedidos();

// filter — mantém elementos que satisfazem o predicado
pedidos.stream()
    .filter(p -> p.valor() > 100.0)
    .filter(p -> p.status() == Status.PAGO)

// map — transforma cada elemento
pedidos.stream()
    .map(Pedido::cliente)           // Pedido → String
    .map(String::toUpperCase)

// flatMap — transforma cada elemento em uma stream e "achata" o resultado
// Útil quando cada elemento é uma coleção
List<List<Item>> listasDeListas = ...;
Stream<Item> todos = listasDeListas.stream()
    .flatMap(Collection::stream);

// Ou: todos os itens de todos os pedidos
Stream<Item> itens = pedidos.stream()
    .flatMap(p -> p.itens().stream());

// distinct — remove duplicatas (usa equals/hashCode)
Stream<String> unicos = Stream.of("a", "b", "a", "c", "b")
    .distinct();   // a, b, c

// sorted — ordena (usa Comparable ou Comparator)
pedidos.stream()
    .sorted(Comparator.comparing(Pedido::valor).reversed())

// sorted com múltiplos critérios
pedidos.stream()
    .sorted(Comparator.comparing(Pedido::cliente)
                      .thenComparing(Pedido::data)
                      .thenComparing(Pedido::valor, Comparator.reverseOrder()))

// peek — inspeciona sem modificar (útil para debug)
pedidos.stream()
    .peek(p -> log.debug("Processando pedido: {}", p.id()))
    .filter(p -> p.valor() > 100)
    .peek(p -> log.debug("Passou no filtro: {}", p.id()))

// limit e skip — paginação
pedidos.stream()
    .skip((pagina - 1) * tamanhoPagina)
    .limit(tamanhoPagina)

// Streams numéricas especializadas (sem boxing)
pedidos.stream()
    .mapToDouble(Pedido::valor)        // Stream<Pedido> → DoubleStream
    .average()
    .orElse(0.0);

pedidos.stream()
    .mapToInt(p -> p.itens().size())   // contagem de itens por pedido
    .sum();
```

### Operações terminais

```java
// forEach — efeito colateral para cada elemento
pedidos.stream()
    .filter(p -> p.status() == Status.PENDENTE)
    .forEach(notificacaoService::notificar);

// collect — materializa o resultado em uma coleção
List<String>  nomes   = pedidos.stream().map(Pedido::cliente).collect(Collectors.toList());
List<String>  nomesJ  = pedidos.stream().map(Pedido::cliente).toList();   // Java 16+ (imutável)
Set<String>   unicos  = pedidos.stream().map(Pedido::cliente).collect(Collectors.toSet());

// count — conta os elementos
long quantidade = pedidos.stream()
    .filter(p -> p.valor() > 1000)
    .count();

// reduce — combina elementos em um único resultado
Optional<BigDecimal> total = pedidos.stream()
    .map(Pedido::valor)
    .reduce(BigDecimal::add);

BigDecimal totalComInicial = pedidos.stream()
    .map(Pedido::valor)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// min e max
Optional<Pedido> maisBarato = pedidos.stream()
    .min(Comparator.comparing(Pedido::valor));

Optional<Pedido> maisCaro = pedidos.stream()
    .max(Comparator.comparing(Pedido::valor));

// findFirst e findAny
Optional<Pedido> primeiro = pedidos.stream()
    .filter(p -> p.cliente().equals("Ana"))
    .findFirst();

// anyMatch, allMatch, noneMatch (short-circuit — para ao encontrar a resposta)
boolean temPendente  = pedidos.stream().anyMatch(p -> p.status() == Status.PENDENTE);
boolean todosPagos   = pedidos.stream().allMatch(p -> p.status() == Status.PAGO);
boolean semCancelado = pedidos.stream().noneMatch(p -> p.status() == Status.CANCELADO);
```

### Collectors avançados

```java
// toMap — coleta em Map (cuidado com chaves duplicadas)
Map<UUID, Pedido> pedidoPorId = pedidos.stream()
    .collect(Collectors.toMap(Pedido::id, Function.identity()));

// toMap com merge function (resolve chaves duplicadas)
Map<String, Double> totalPorCliente = pedidos.stream()
    .collect(Collectors.toMap(
        Pedido::cliente,
        Pedido::valor,
        Double::sum   // se houver chave duplicada, soma os valores
    ));

// groupingBy — agrupa em Map<K, List<V>>
Map<Status, List<Pedido>> pedidosPorStatus = pedidos.stream()
    .collect(Collectors.groupingBy(Pedido::status));

// groupingBy com downstream collector
Map<String, Long> quantidadePorCliente = pedidos.stream()
    .collect(Collectors.groupingBy(Pedido::cliente, Collectors.counting()));

Map<String, Double> totalPorClienteG = pedidos.stream()
    .collect(Collectors.groupingBy(
        Pedido::cliente,
        Collectors.summingDouble(Pedido::valor)
    ));

Map<Status, Double> mediaPorStatus = pedidos.stream()
    .collect(Collectors.groupingBy(
        Pedido::status,
        Collectors.averagingDouble(Pedido::valor)
    ));

// groupingBy aninhado
Map<String, Map<Status, List<Pedido>>> porClientePorStatus = pedidos.stream()
    .collect(Collectors.groupingBy(
        Pedido::cliente,
        Collectors.groupingBy(Pedido::status)
    ));

// partitioningBy — divide em true/false
Map<Boolean, List<Pedido>> particao = pedidos.stream()
    .collect(Collectors.partitioningBy(p -> p.valor() > 1000));

List<Pedido> caros   = particao.get(true);
List<Pedido> baratos = particao.get(false);

// joining — concatena strings
String nomesConcatenados = pedidos.stream()
    .map(Pedido::cliente)
    .collect(Collectors.joining(", ", "[", "]"));
// Resultado: "[Ana, Bruno, Carlos]"

// summarizingInt — estatísticas completas
IntSummaryStatistics stats = pedidos.stream()
    .collect(Collectors.summarizingInt(p -> p.itens().size()));
// stats.getCount(), getSum(), getMin(), getMax(), getAverage()

// toUnmodifiableList/Set/Map (Java 10+)
List<String> imutavel = pedidos.stream()
    .map(Pedido::cliente)
    .collect(Collectors.toUnmodifiableList());
```

### Parallel Streams

```java
// Habilitando paralelismo
List<Pedido> resultado = pedidos.parallelStream()
    .filter(p -> p.valor() > 100)
    .map(this::enriquecerPedido)
    .toList();

// QUANDO usar parallel streams:
// - Coleções muito grandes (> 10.000 elementos)
// - Operações CPU-bound (não I/O)
// - Sem estado compartilhado mutável
// - Em máquinas com múltiplos núcleos

// CUIDADOS — efeitos colaterais são perigosos em paralelo
List<String> resultados = new ArrayList<>();
pedidos.parallelStream()
    .map(Pedido::cliente)
    .forEach(resultados::add);   // PERIGOSO — ArrayList não é thread-safe

// CORRETO em paralelo — use coletor, não forEach com mutação
List<String> correto = pedidos.parallelStream()
    .map(Pedido::cliente)
    .collect(Collectors.toList());   // Collectors são thread-safe

// Definir o pool usado pelo parallel stream
ForkJoinPool pool = new ForkJoinPool(4);
List<Pedido> resultado2 = pool.submit(() ->
    pedidos.parallelStream()
           .filter(p -> p.valor() > 100)
           .toList()
).get();
```

---

## Optional

`Optional<T>` é um container que pode ou não conter um valor. Elimina `NullPointerException` quando usado corretamente.

```java
// Criando Optional
Optional<String> comValor = Optional.of("Java");           // NullPointerException se null
Optional<String> nullable  = Optional.ofNullable(obterNome()); // null → Optional.empty()
Optional<String> vazio     = Optional.empty();

// Verificações básicas
boolean presente = comValor.isPresent();
boolean ausente  = comValor.isEmpty();    // Java 11+

// Extraindo o valor
String valor = comValor.get();                             // NoSuchElementException se vazio
String comDefault = vazio.orElse("padrão");                // retorna "padrão" se vazio
String comFornecedor = vazio.orElseGet(() -> gerarNome()); // lazy — só chama se vazio
String comExcecao = vazio.orElseThrow(
    () -> new RecursoNaoEncontradoException("Nome não encontrado")
);

// Transformações — só executam se o valor estiver presente
Optional<Integer> tamanho = comValor.map(String::length);
Optional<String>  maiusculo = comValor.map(String::toUpperCase);

// flatMap — quando o mapeamento retorna Optional (evita Optional<Optional<T>>)
Optional<Pedido> pedido = buscarCliente(id)
    .flatMap(cliente -> buscarUltimoPedido(cliente.id()));

// filter — mantém o valor só se passar no predicado
Optional<String> longo = comValor.filter(s -> s.length() > 3);

// ifPresent — executa ação apenas se o valor estiver presente
comValor.ifPresent(s -> log.info("Nome encontrado: {}", s));

// ifPresentOrElse — Java 9+
comValor.ifPresentOrElse(
    s -> log.info("Encontrado: {}", s),
    ()  -> log.warn("Não encontrado")
);

// or — Java 9+, fornece Optional alternativo
Optional<String> resultado = buscarNoBanco(id)
    .or(() -> buscarNoCache(id))
    .or(() -> Optional.of("padrão"));
```

### Uso real com repository

```java
// RUIM — retornar null
public Pedido buscarPedido(UUID id) {
    return pedidos.get(id);   // null se não encontrar
}

// RUIM — verificar null no código chamador
Pedido pedido = buscarPedido(id);
if (pedido != null) {
    processar(pedido);
}

// BOM — retornar Optional
public Optional<Pedido> buscarPedido(UUID id) {
    return Optional.ofNullable(pedidos.get(id));
}

// Código chamador expressa intenção claramente
buscarPedido(id)
    .map(this::calcularTotal)
    .ifPresent(total -> notificar(id, total));

// Ou lançar exceção de domínio
Pedido pedido = buscarPedido(id)
    .orElseThrow(() -> new PedidoNaoEncontradoException(id));
```

### Quando NÃO usar Optional

```java
// NÃO use como parâmetro de método
// RUIM:
public void processar(Optional<Pedido> pedido) { ... }
// BOM: sobrecarga ou valor padrão
public void processar(Pedido pedido) { ... }

// NÃO use em campos de classe (não é serializável)
// RUIM:
public class Cliente {
    private Optional<String> telefone;   // problema na serialização
}
// BOM: null ou use JsonInclude(NON_NULL) no Jackson
public class Cliente {
    private String telefone;   // pode ser null — documente
}

// NÃO use em coleções (use coleção vazia em vez de Optional<List<...>>)
// RUIM:
Optional<List<Pedido>> pedidos = buscarTodos();
// BOM:
List<Pedido> pedidos = buscarTodos();   // retorna List.of() se vazio
```
