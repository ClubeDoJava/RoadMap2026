# 1.6 - Strings e Arrays

`String` e arrays são duas das estruturas de dados mais fundamentais em Java. Entender como criá-los, manipulá-los e percorrê-los é essencial para qualquer programa.

---

## String

### Criação de Strings

Em Java, `String` é uma classe — não um tipo primitivo. Há duas formas principais de criar Strings:

```java
public class CriacaoString {
    public static void main(String[] args) {
        // Forma 1: literal (recomendada) — usa o String Pool
        String nome = "Java";
        String linguagem = "Java"; // reutiliza o mesmo objeto do pool

        // Forma 2: com new — cria sempre um novo objeto no heap
        String nome2 = new String("Java");

        // Comparação de referências
        System.out.println(nome == linguagem);  // true (mesmo objeto no pool)
        System.out.println(nome == nome2);       // false (objetos diferentes)
        System.out.println(nome.equals(nome2));  // true (mesmo conteúdo)

        // String vazia vs null
        String vazia = "";         // String sem caracteres
        String nula = null;        // referência que não aponta para nenhum objeto

        System.out.println("Vazia: \"" + vazia + "\"");
        System.out.println("Nula: " + nula);
        System.out.println("Vazia está vazia: " + vazia.isEmpty());
        // nula.isEmpty() lançaria NullPointerException!
    }
}
```

---

### Principais Métodos

```java
public class MetodosString {
    public static void main(String[] args) {
        String texto = "  Olá, Mundo Java!  ";
        String frase = "Aprender Java é incrível";

        // --- Tamanho ---
        System.out.println("length():     " + texto.length());        // 20 (com espaços)
        System.out.println("trim():       \"" + texto.trim() + "\""); // remove espaços extras
        System.out.println("strip():      \"" + texto.strip() + "\""); // idem, suporte a Unicode

        String s = texto.trim(); // "Olá, Mundo Java!"

        // --- Acesso a caracteres ---
        System.out.println("\ncharAt(4):    " + s.charAt(4));          // ','
        System.out.println("indexOf('o'): " + s.indexOf('o'));         // 1
        System.out.println("indexOf(\"Java\"): " + s.indexOf("Java")); // 11
        System.out.println("lastIndexOf('a'): " + s.lastIndexOf('a')); // 15

        // --- Substrings ---
        System.out.println("\nsubstring(5):     " + s.substring(5));     // "Mundo Java!"
        System.out.println("substring(5, 10): " + s.substring(5, 10)); // "Mundo"

        // --- Verificações ---
        System.out.println("\ncontains(\"Java\"): " + s.contains("Java"));       // true
        System.out.println("startsWith(\"Olá\"): " + s.startsWith("Olá"));     // true
        System.out.println("endsWith(\"!\"):     " + s.endsWith("!"));          // true
        System.out.println("isEmpty():          " + s.isEmpty());               // false
        System.out.println("isBlank():          " + "   ".isBlank());           // true (Java 11+)

        // --- Transformações ---
        System.out.println("\ntoUpperCase(): " + s.toUpperCase()); // OLÁ, MUNDO JAVA!
        System.out.println("toLowerCase(): " + s.toLowerCase()); // olá, mundo java!
        System.out.println("replace('a','@'): " + s.replace('a', '@')); // substituição de char
        System.out.println("replace(\"Java\",\"Python\"): " + s.replace("Java", "Python"));

        // --- Split ---
        String csv = "maçã,banana,cereja,damasco";
        String[] frutas = csv.split(",");
        System.out.println("\nsplit(\",\") resulta em " + frutas.length + " partes:");
        for (String f : frutas) {
            System.out.println("  " + f);
        }

        // --- Comparação ---
        String a = "Java";
        String b = "java";
        System.out.println("\nequals:           " + a.equals(b));             // false
        System.out.println("equalsIgnoreCase: " + a.equalsIgnoreCase(b));    // true
        System.out.println("compareTo:        " + a.compareTo(b));           // negativo (J < j em ASCII)

        // --- Conversão ---
        System.out.println("\nvalueOf(42):    " + String.valueOf(42));
        System.out.println("valueOf(3.14):  " + String.valueOf(3.14));
        System.out.println("valueOf(true):  " + String.valueOf(true));

        // --- Outros ---
        System.out.println("\nrepeat (Java 11+): " + "ab".repeat(4)); // abababab
        System.out.println("formatted (Java 15+): "
            + "Nota: %.1f".formatted(8.5));                           // Nota: 8.5
    }
}
```

---

### String é Imutável

Uma vez criado, o conteúdo de um objeto `String` **jamais pode ser alterado**. Toda operação que "modifica" uma String na verdade cria um novo objeto.

```java
public class StringImutavel {
    public static void main(String[] args) {
        String original = "Java";
        String modificada = original.toUpperCase(); // cria NOVO objeto

        System.out.println("original:   " + original);  // Java (não mudou)
        System.out.println("modificada: " + modificada); // JAVA

        // Concatenação também cria novos objetos
        String s = "Olá";
        s = s + ", Mundo"; // s aponta para um NOVO objeto; o antigo é descartado
        System.out.println(s); // Olá, Mundo

        // Concatenação em loop: ineficiente (cria muitos objetos temporários)
        String resultado = "";
        for (int i = 0; i < 5; i++) {
            resultado += i; // BAD: cria novo String a cada iteração
        }
        System.out.println("Resultado (ineficiente): " + resultado); // 01234
    }
}
```

---

### StringBuilder para Concatenação Eficiente

`StringBuilder` é mutável e muito mais eficiente quando há muitas concatenações, pois modifica o mesmo buffer interno.

```java
public class UsandoStringBuilder {
    public static void main(String[] args) {
        // Criação
        StringBuilder sb = new StringBuilder();

        // Append: adiciona ao final
        sb.append("Java");
        sb.append(" é ");
        sb.append("incrível");
        sb.append("!");
        System.out.println(sb.toString()); // Java é incrível!

        // Outros métodos úteis
        StringBuilder sb2 = new StringBuilder("Olá, Mundo!");

        sb2.insert(5, " querido"); // insere na posição 5
        System.out.println(sb2); // Olá,  querido Mundo!

        sb2.delete(4, 12); // remove do índice 4 ao 12 (exclusive)
        System.out.println(sb2);

        sb2.reverse(); // inverte
        System.out.println(sb2);

        // Exemplo prático: construção eficiente de String grande
        int n = 10_000;

        // Versão com String (lenta — cria n objetos)
        long inicio1 = System.currentTimeMillis();
        String strSimples = "";
        for (int i = 0; i < n; i++) {
            strSimples += i;
        }
        long tempo1 = System.currentTimeMillis() - inicio1;

        // Versão com StringBuilder (rápida — modifica o mesmo buffer)
        long inicio2 = System.currentTimeMillis();
        StringBuilder sbRapido = new StringBuilder();
        for (int i = 0; i < n; i++) {
            sbRapido.append(i);
        }
        String strRapida = sbRapido.toString();
        long tempo2 = System.currentTimeMillis() - inicio2;

        System.out.println("\nTempo com String:        " + tempo1 + " ms");
        System.out.println("Tempo com StringBuilder: " + tempo2 + " ms");
        System.out.println("Resultado igual? " + strSimples.equals(strRapida));
    }
}
```

> **Regra prática:** Use `String` para valores fixos. Use `StringBuilder` quando precisar construir Strings dinamicamente (loops, concatenações repetidas).

---

## Arrays

### Declaração e Inicialização

Um array armazena múltiplos valores do **mesmo tipo** em posições contíguas de memória, acessadas por índice (começando em 0).

```java
import java.util.Arrays;

public class DeclaracaoArray {
    public static void main(String[] args) {
        // Declaração e inicialização separadas
        int[] numeros;              // declaração
        numeros = new int[5];       // inicialização: 5 posições, todas com 0

        // Declaração e inicialização juntas
        int[] pares = new int[]{2, 4, 6, 8, 10};

        // Forma mais concisa (shorthand) — só funciona na declaração
        String[] frutas = {"Maçã", "Banana", "Cereja"};
        double[] notas = {8.5, 7.0, 9.5, 6.0};

        // Acesso por índice (começa em 0)
        System.out.println("Primeiro elemento: " + frutas[0]); // Maçã
        System.out.println("Último elemento:   " + frutas[frutas.length - 1]); // Cereja

        // Modificação
        frutas[1] = "Manga";
        System.out.println("Após modificação:  " + Arrays.toString(frutas));

        // Tamanho do array
        System.out.println("Tamanho: " + frutas.length); // 3

        // ArrayIndexOutOfBoundsException se acessar índice inválido
        // frutas[10]; // lança exceção em runtime!
    }
}
```

---

### Percorrendo Arrays

```java
import java.util.Arrays;

public class PercorrendoArray {
    public static void main(String[] args) {
        int[] numeros = {15, 3, 42, 8, 27, 1, 99, 56};

        // 1. for tradicional (acesso ao índice)
        System.out.println("for tradicional:");
        for (int i = 0; i < numeros.length; i++) {
            System.out.printf("  [%d] = %d%n", i, numeros[i]);
        }

        // 2. for-each (mais legível, sem acesso ao índice)
        System.out.print("\nfor-each: ");
        for (int n : numeros) {
            System.out.print(n + " ");
        }
        System.out.println();

        // 3. Calcular estatísticas
        int soma = 0;
        int min = numeros[0];
        int max = numeros[0];

        for (int n : numeros) {
            soma += n;
            if (n < min) min = n;
            if (n > max) max = n;
        }

        System.out.println("\nEstatísticas:");
        System.out.println("  Soma:   " + soma);
        System.out.println("  Média:  " + (double) soma / numeros.length);
        System.out.println("  Mínimo: " + min);
        System.out.println("  Máximo: " + max);
    }
}
```

---

### Arrays Multidimensionais

Arrays de arrays. O mais comum é o bidimensional (matriz).

```java
import java.util.Arrays;

public class ArraysMultidimensionais {
    public static void main(String[] args) {
        // Matriz 3x3
        int[][] matriz = {
            {1, 2, 3},
            {4, 5, 6},
            {7, 8, 9}
        };

        // Acesso: matriz[linha][coluna]
        System.out.println("Centro da matriz: " + matriz[1][1]); // 5

        // Percorrendo com for aninhado
        System.out.println("\nMatriz 3x3:");
        for (int i = 0; i < matriz.length; i++) {          // linhas
            for (int j = 0; j < matriz[i].length; j++) {   // colunas
                System.out.printf("%3d", matriz[i][j]);
            }
            System.out.println();
        }

        // Percorrendo com for-each aninhado
        System.out.println("\nFor-each aninhado:");
        for (int[] linha : matriz) {
            System.out.println(Arrays.toString(linha));
        }

        // Array "jagged" (linhas com tamanhos diferentes)
        int[][] triangulo = {
            {1},
            {1, 2},
            {1, 2, 3},
            {1, 2, 3, 4}
        };

        System.out.println("\nTriângulo:");
        for (int[] linha : triangulo) {
            for (int val : linha) {
                System.out.print(val + " ");
            }
            System.out.println();
        }
    }
}
```

---

### Arrays.sort e Arrays.toString

A classe utilitária `java.util.Arrays` oferece métodos prontos para operações comuns.

```java
import java.util.Arrays;

public class UtilitariosArrays {
    public static void main(String[] args) {
        int[] numeros = {42, 7, 99, 3, 55, 1, 28};

        // Arrays.toString: representação legível
        System.out.println("Original: " + Arrays.toString(numeros));

        // Arrays.sort: ordenação in-place (modifica o array original)
        Arrays.sort(numeros);
        System.out.println("Ordenado: " + Arrays.toString(numeros));

        // Arrays.sort com intervalo (ordena apenas parte do array)
        int[] parcial = {5, 3, 8, 1, 9, 2, 7, 4};
        Arrays.sort(parcial, 2, 6); // ordena do índice 2 ao 5 (exclusive 6)
        System.out.println("Parcial:  " + Arrays.toString(parcial));

        // Arrays.binarySearch: busca binária (array deve estar ordenado!)
        int[] ordenado = {10, 20, 30, 40, 50, 60, 70};
        int posicao = Arrays.binarySearch(ordenado, 40);
        System.out.println("\nBusca por 40: índice " + posicao); // 3

        // Arrays.fill: preenche com um valor
        int[] zerado = new int[5];
        Arrays.fill(zerado, -1);
        System.out.println("fill(-1): " + Arrays.toString(zerado)); // [-1,-1,-1,-1,-1]

        // Arrays.copyOf: cria uma cópia com tamanho especificado
        int[] copia = Arrays.copyOf(ordenado, 4); // copia os 4 primeiros
        System.out.println("copyOf(4): " + Arrays.toString(copia));

        // Arrays.copyOfRange: copia um intervalo
        int[] intervalo = Arrays.copyOfRange(ordenado, 2, 5); // índices 2 a 4
        System.out.println("copyOfRange(2,5): " + Arrays.toString(intervalo));

        // Arrays.equals: compara o conteúdo de dois arrays
        int[] a = {1, 2, 3};
        int[] b = {1, 2, 3};
        int[] c = {1, 2, 4};
        System.out.println("\na == b (ref):       " + (a == b));           // false
        System.out.println("Arrays.equals(a,b): " + Arrays.equals(a, b)); // true
        System.out.println("Arrays.equals(a,c): " + Arrays.equals(a, c)); // false

        // deepToString para arrays multidimensionais
        int[][] matriz = {{1, 2}, {3, 4}, {5, 6}};
        System.out.println("\nMatriz: " + Arrays.deepToString(matriz)); // [[1,2],[3,4],[5,6]]
    }
}
```

---

## Exemplo Completo

```java
import java.util.Arrays;

public class SistemaNotas {
    public static void main(String[] args) {
        String[] alunos = {"Ana", "Bruno", "Carlos", "Diana", "Eduardo"};
        double[][] notas = {
            {8.5, 7.0, 9.0},   // Ana
            {6.0, 7.5, 8.0},   // Bruno
            {5.5, 6.0, 5.0},   // Carlos
            {9.5, 10.0, 9.0},  // Diana
            {7.0, 6.5, 8.5}    // Eduardo
        };

        System.out.println("=== Sistema de Notas ===\n");
        System.out.printf("%-10s | %-8s | %-8s | %-8s | %-8s | %s%n",
            "Aluno", "Nota 1", "Nota 2", "Nota 3", "Média", "Situação");
        System.out.println("-".repeat(65));

        double[] medias = new double[alunos.length];

        for (int i = 0; i < alunos.length; i++) {
            double soma = 0;
            for (double nota : notas[i]) {
                soma += nota;
            }
            medias[i] = soma / notas[i].length;

            String situacao = medias[i] >= 6.0 ? "Aprovado" : "Reprovado";

            System.out.printf("%-10s | %-8.1f | %-8.1f | %-8.1f | %-8.2f | %s%n",
                alunos[i],
                notas[i][0], notas[i][1], notas[i][2],
                medias[i],
                situacao);
        }

        // Estatísticas
        double[] mediasCopia = Arrays.copyOf(medias, medias.length);
        Arrays.sort(mediasCopia);

        double mediaTurma = 0;
        for (double m : medias) mediaTurma += m;
        mediaTurma /= medias.length;

        System.out.println("-".repeat(65));
        System.out.printf("Média da turma:  %.2f%n", mediaTurma);
        System.out.printf("Maior média:     %.2f%n", mediasCopia[mediasCopia.length - 1]);
        System.out.printf("Menor média:     %.2f%n", mediasCopia[0]);

        // Melhor aluno
        int indiceMelhor = 0;
        for (int i = 1; i < medias.length; i++) {
            if (medias[i] > medias[indiceMelhor]) {
                indiceMelhor = i;
            }
        }
        System.out.println("Melhor aluno:    " + alunos[indiceMelhor]
            + " (" + String.format("%.2f", medias[indiceMelhor]) + ")");

        // StringBuilder para relatório final
        StringBuilder relatorio = new StringBuilder();
        relatorio.append("\n=== Resumo ===\n");
        int aprovados = 0;
        for (double m : medias) {
            if (m >= 6.0) aprovados++;
        }
        relatorio.append("Aprovados:  ").append(aprovados).append("\n");
        relatorio.append("Reprovados: ").append(alunos.length - aprovados).append("\n");
        relatorio.append("Taxa de aprovação: ")
                 .append(String.format("%.0f%%", (double) aprovados / alunos.length * 100));
        System.out.println(relatorio.toString());
    }
}
```
