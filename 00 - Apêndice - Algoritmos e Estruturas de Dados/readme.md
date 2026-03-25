# Apêndice — Algoritmos e Estruturas de Dados em Java

Este apêndice não é sobre LeetCode por si só — é sobre ter o raciocínio algorítmico que todo desenvolvedor Java profissional precisa ao escolher entre um `ArrayList` e um `LinkedList`, ao decidir como modelar um sistema de busca, ou ao escrever código que precise ser eficiente sob carga.

**Quando estudar:** após o módulo 3.2 (Collections Framework), em paralelo com os módulos 4 e 5. Não espere terminar o roadmap inteiro.

---

## Complexidade Algorítmica — Big-O

### O que é e por que importa

Big-O descreve como o tempo de execução ou uso de memória cresce em relação ao tamanho da entrada. Não mede o tempo exato — mede a **tendência de crescimento**.

```
Entrada (n)    O(1)    O(log n)    O(n)    O(n log n)    O(n²)
10             1       3           10      33            100
100            1       7           100     664           10.000
1.000          1       10          1.000   9.966         1.000.000
10.000         1       13          10.000  132.877       100.000.000
1.000.000      1       20          1.000.000  19.931.568  10^12
```

Para 1 milhão de elementos, a diferença entre O(log n) e O(n²) é a diferença entre 20 operações e 1 trilhão.

### Complexidades das Collections do Java

| Estrutura | Acesso por índice | Busca | Inserção | Remoção | Quando usar |
|---|---|---|---|---|---|
| `ArrayList` | O(1) | O(n) | O(1) amortizado* | O(n) | Lista com acesso por índice frequente |
| `LinkedList` | O(n) | O(n) | O(1) no início/fim | O(1) quando tem referência | Fila/Deque; raramente para acesso |
| `HashMap` | — | O(1) médio | O(1) médio | O(1) médio | Busca por chave, sem ordem |
| `TreeMap` | — | O(log n) | O(log n) | O(log n) | Busca por chave **ordenada** |
| `HashSet` | — | O(1) médio | O(1) médio | O(1) médio | Verificar existência sem duplicatas |
| `TreeSet` | — | O(log n) | O(log n) | O(log n) | Conjunto **ordenado** |
| `PriorityQueue` | O(n) | O(n) | O(log n) | O(log n) | Menor/maior elemento sempre acessível |
| `ArrayDeque` | O(n) | O(n) | O(1) nas pontas | O(1) nas pontas | Pilha ou fila (mais rápido que `LinkedList`) |

*`ArrayList.add()` é O(1) amortizado: ocasionalmente precisa redimensionar o array interno (O(n)), mas na média é O(1).

```java
// RUIM — busca linear em ArrayList: O(n) por cada verificação
List<String> produtos = new ArrayList<>();
// ... carrega 100.000 produtos
for (String p : produtos) {
    if (p.equals(nomeBuscado)) { // O(n) por busca
        // processar
    }
}

// BOM — busca em HashSet: O(1) por verificação
Set<String> produtosSet = new HashSet<>(produtos);
if (produtosSet.contains(nomeBuscado)) { // O(1)
    // processar
}
```

---

## Estruturas de Dados Fundamentais

### Pilha (Stack)

LIFO — Last In, First Out. O último elemento inserido é o primeiro removido.

```java
// Em Java, use ArrayDeque como pilha (mais eficiente que Stack)
Deque<Integer> pilha = new ArrayDeque<>();

pilha.push(1);    // [1]
pilha.push(2);    // [2, 1]
pilha.push(3);    // [3, 2, 1]

System.out.println(pilha.peek());  // 3 (sem remover)
System.out.println(pilha.pop());   // 3 — remove e retorna o topo
System.out.println(pilha.pop());   // 2
// Pilha: [1]
```

**Casos de uso:** histórico de navegação (voltar), desfazer/refazer, avaliação de expressões, DFS iterativo.

### Fila (Queue)

FIFO — First In, First Out. O primeiro elemento inserido é o primeiro removido.

```java
Queue<String> fila = new ArrayDeque<>();

fila.offer("tarefa-1");   // Não lança exceção se a fila estiver cheia
fila.offer("tarefa-2");
fila.offer("tarefa-3");

System.out.println(fila.peek());   // "tarefa-1" (sem remover)
System.out.println(fila.poll());   // "tarefa-1" — remove e retorna
System.out.println(fila.poll());   // "tarefa-2"
```

**Casos de uso:** filas de processamento, BFS, sistemas de mensageria (Kafka é uma fila distribuída de alta performance).

### Fila de Prioridade (PriorityQueue)

Remove sempre o elemento de **menor valor** (min-heap por padrão).

```java
// Min-heap: retorna sempre o menor
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);
minHeap.offer(1);
minHeap.offer(3);

System.out.println(minHeap.poll()); // 1 (menor)
System.out.println(minHeap.poll()); // 3
System.out.println(minHeap.poll()); // 5

// Max-heap: retorna sempre o maior
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
maxHeap.offer(5);
maxHeap.offer(1);
maxHeap.offer(3);
System.out.println(maxHeap.poll()); // 5 (maior)

// Com objetos customizados
PriorityQueue<Pedido> pedidosPorPrioridade = new PriorityQueue<>(
    Comparator.comparingInt(Pedido::getPrioridade)
);
```

**Casos de uso:** agendamento de tarefas por prioridade, algoritmo de Dijkstra, k elementos menores/maiores de um stream.

---

## Árvore Binária de Busca (BST)

```
         8
        / \
       3   10
      / \    \
     1   6    14
        / \   /
       4   7 13
```

Propriedade: para cada nó, todos os valores à esquerda são menores, todos à direita são maiores.

```java
public class NoBST {
    int valor;
    NoBST esquerda, direita;

    NoBST(int valor) { this.valor = valor; }
}

public class BST {

    private NoBST raiz;

    public void inserir(int valor) {
        raiz = inserirRec(raiz, valor);
    }

    private NoBST inserirRec(NoBST no, int valor) {
        if (no == null) return new NoBST(valor);
        if (valor < no.valor) no.esquerda = inserirRec(no.esquerda, valor);
        else if (valor > no.valor) no.direita = inserirRec(no.direita, valor);
        return no; // valor == no.valor: duplicata ignorada
    }

    public boolean contem(int valor) {
        NoBST atual = raiz;
        while (atual != null) {
            if (valor == atual.valor) return true;
            atual = valor < atual.valor ? atual.esquerda : atual.direita;
        }
        return false;
    }

    // Percurso em-ordem (in-order): retorna valores em ordem crescente
    public List<Integer> inOrder() {
        List<Integer> resultado = new ArrayList<>();
        inOrderRec(raiz, resultado);
        return resultado;
    }

    private void inOrderRec(NoBST no, List<Integer> resultado) {
        if (no == null) return;
        inOrderRec(no.esquerda, resultado);
        resultado.add(no.valor);
        inOrderRec(no.direita, resultado);
    }
}
```

> **Na prática em Java:** você raramente implementa BST do zero. `TreeMap` e `TreeSet` usam Red-Black Tree (BST auto-balanceada) internamente. Entender BST é pré-requisito para entender por que `TreeMap.get()` é O(log n) e quando preferi-lo ao `HashMap`.

---

## Grafos — BFS e DFS

Grafos representam conexões entre entidades. São ubíquos em sistemas reais: rotas entre cidades, dependências entre serviços, relacionamentos em redes sociais, caminho entre páginas.

```java
// Representação: lista de adjacência
Map<String, List<String>> grafo = new HashMap<>();
grafo.put("A", List.of("B", "C"));
grafo.put("B", List.of("A", "D", "E"));
grafo.put("C", List.of("A", "F"));
grafo.put("D", List.of("B"));
grafo.put("E", List.of("B", "F"));
grafo.put("F", List.of("C", "E"));
```

### BFS — Busca em Largura (menor caminho)

Visita todos os nós a distância 1, depois distância 2, etc. Encontra o **caminho mais curto** em grafos não-ponderados.

```java
public List<String> bfs(Map<String, List<String>> grafo, String inicio, String destino) {
    Queue<List<String>> fila = new ArrayDeque<>();
    Set<String> visitados = new HashSet<>();

    fila.offer(List.of(inicio));
    visitados.add(inicio);

    while (!fila.isEmpty()) {
        List<String> caminho = fila.poll();
        String atual = caminho.getLast();

        if (atual.equals(destino)) return caminho;

        for (String vizinho : grafo.getOrDefault(atual, List.of())) {
            if (!visitados.contains(vizinho)) {
                visitados.add(vizinho);
                List<String> novoCaminho = new ArrayList<>(caminho);
                novoCaminho.add(vizinho);
                fila.offer(novoCaminho);
            }
        }
    }
    return List.of(); // Caminho não encontrado
}

// Uso:
List<String> caminho = bfs(grafo, "A", "F");
// ["A", "C", "F"] — caminho mais curto
```

### DFS — Busca em Profundidade (exploração completa)

Vai o mais fundo possível antes de retroceder. Útil para detectar ciclos, encontrar componentes conectados, verificar se um caminho existe.

```java
public boolean dfs(Map<String, List<String>> grafo, String atual,
                   String destino, Set<String> visitados) {
    if (atual.equals(destino)) return true;
    visitados.add(atual);

    for (String vizinho : grafo.getOrDefault(atual, List.of())) {
        if (!visitados.contains(vizinho)) {
            if (dfs(grafo, vizinho, destino, visitados)) return true;
        }
    }
    return false;
}

// Uso:
Set<String> visitados = new HashSet<>();
boolean existe = dfs(grafo, "A", "F", visitados);
```

**Quando usar BFS vs DFS:**

| Problema | Algoritmo |
|---|---|
| Caminho mais curto em grafo não-ponderado | BFS |
| Verificar se caminho existe | DFS (mais simples) |
| Detectar ciclos | DFS |
| Componentes conectados | BFS ou DFS |
| Percorrer toda a árvore de dependências | DFS |
| Nível de relacionamento em rede social | BFS |

---

## Exercícios Fundamentais (com Java)

Estes 15 problemas cobrem os padrões que aparecem repetidamente em entrevistas técnicas e em código de produção real:

### 1. Dois Ponteiros — Pares com soma alvo

```java
// Dado array ordenado, encontrar par que some ao alvo
// O(n) tempo, O(1) espaço
public int[] doisPonteiros(int[] nums, int alvo) {
    int esq = 0, dir = nums.length - 1;
    while (esq < dir) {
        int soma = nums[esq] + nums[dir];
        if (soma == alvo) return new int[]{esq, dir};
        if (soma < alvo) esq++;
        else dir--;
    }
    return new int[]{};
}
```

### 2. Janela Deslizante — Maior substring sem repetição

```java
// O(n) tempo, O(k) espaço onde k = tamanho do charset
public int maiorSubstringSemRepeticao(String s) {
    Map<Character, Integer> mapa = new HashMap<>();
    int max = 0, esq = 0;

    for (int dir = 0; dir < s.length(); dir++) {
        char c = s.charAt(dir);
        if (mapa.containsKey(c)) {
            esq = Math.max(esq, mapa.get(c) + 1);
        }
        mapa.put(c, dir);
        max = Math.max(max, dir - esq + 1);
    }
    return max;
}
```

### 3. Pilha — Parênteses válidos

```java
public boolean parentesesValidos(String s) {
    Deque<Character> pilha = new ArrayDeque<>();
    Map<Character, Character> pares = Map.of(')', '(', '}', '{', ']', '[');

    for (char c : s.toCharArray()) {
        if ("({[".indexOf(c) >= 0) {
            pilha.push(c);
        } else {
            if (pilha.isEmpty() || pilha.pop() != pares.get(c)) return false;
        }
    }
    return pilha.isEmpty();
}
```

### 4. HashMap — Anagramas

```java
public boolean saoAnagramas(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] contagem = new int[26];
    for (char c : s.toCharArray()) contagem[c - 'a']++;
    for (char c : t.toCharArray()) contagem[c - 'a']--;
    for (int n : contagem) if (n != 0) return false;
    return true;
}
```

### 5. Ordenação — K elementos mais frequentes

```java
// PriorityQueue para os k mais frequentes: O(n log k)
public List<Integer> kMaisFrequentes(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    PriorityQueue<Integer> heap = new PriorityQueue<>(
        Comparator.comparingInt(freq::get) // min-heap por frequência
    );

    for (int num : freq.keySet()) {
        heap.offer(num);
        if (heap.size() > k) heap.poll(); // Remove o menos frequente
    }

    return new ArrayList<>(heap);
}
```

### 6. Busca Binária

```java
// O(log n) — funciona apenas em arrays ordenados
public int buscaBinaria(int[] nums, int alvo) {
    int esq = 0, dir = nums.length - 1;
    while (esq <= dir) {
        int meio = esq + (dir - esq) / 2; // Evita overflow vs (esq+dir)/2
        if (nums[meio] == alvo) return meio;
        if (nums[meio] < alvo) esq = meio + 1;
        else dir = meio - 1;
    }
    return -1; // Não encontrado
}
```

### 7. Programação Dinâmica — Maior subsequência comum (LCS)

```java
// Clássico de DP — O(m*n) tempo e espaço
public int lcs(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[m][n];
}
```

---

## Padrões de resolução — resumo rápido

| Padrão | Quando usar | Complexidade típica |
|---|---|---|
| **Dois Ponteiros** | Array/string ordenada, par com soma | O(n) |
| **Janela Deslizante** | Substring/subarray contíguo | O(n) |
| **Pilha** | Parênteses, temperatura próxima maior, avaliação de expressões | O(n) |
| **HashMap** | Frequência, anagramas, dois somas | O(n) |
| **BFS** | Menor caminho em grafo, nível em árvore | O(V+E) |
| **DFS** | Existência de caminho, componentes, ciclos | O(V+E) |
| **Busca Binária** | Encontrar posição em array ordenado | O(log n) |
| **PriorityQueue** | k menores/maiores, merge de listas ordenadas | O(n log k) |
| **DP** | Subproblemas sobrepostos, otimização | O(n²) em geral |

---

## Recursos para praticar

- **LeetCode:** comece pelos problemas marcados "Easy" e "Medium" com os tags: Array, String, HashMap, Stack, Queue, Tree
- **NeetCode.io:** roadmap organizado por padrão — melhor estrutura para quem está começando
- **Advent of Code:** problemas de dezembro, bom para grafos e DP com contexto narrativo
- **Inside Java Puzzlers:** exercícios específicos de comportamento da JVM e Collections Java
