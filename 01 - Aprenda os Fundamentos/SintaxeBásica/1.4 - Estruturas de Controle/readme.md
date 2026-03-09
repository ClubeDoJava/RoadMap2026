# 1.4 - Estruturas de Controle

Estruturas de controle determinam o fluxo de execução de um programa. Em Java, elas se dividem em **estruturas de decisão** (condicionais) e **estruturas de repetição** (loops).

---

## if / else if / else

A estrutura `if` executa um bloco de código somente se a condição for `true`. O `else if` avalia condições adicionais e o `else` captura todos os casos restantes.

```java
public class IfElse {
    public static void main(String[] args) {
        double nota = 7.5;

        // if simples
        if (nota >= 6.0) {
            System.out.println("Aprovado!");
        }

        // if / else
        if (nota >= 6.0) {
            System.out.println("Situação: Aprovado");
        } else {
            System.out.println("Situação: Reprovado");
        }

        // if / else if / else
        if (nota >= 9.0) {
            System.out.println("Conceito: A");
        } else if (nota >= 7.0) {
            System.out.println("Conceito: B");
        } else if (nota >= 5.0) {
            System.out.println("Conceito: C");
        } else {
            System.out.println("Conceito: D");
        }

        // if aninhado
        int idade = 20;
        boolean temCNH = true;

        if (idade >= 18) {
            if (temCNH) {
                System.out.println("Pode dirigir.");
            } else {
                System.out.println("Maior de idade, mas sem CNH.");
            }
        } else {
            System.out.println("Menor de idade, não pode dirigir.");
        }
    }
}
```

---

## switch

O `switch` compara uma variável com múltiplos valores possíveis. É mais legível que vários `else if` quando há muitos casos discretos.

### switch Tradicional (todas as versões)

```java
public class SwitchTradicional {
    public static void main(String[] args) {
        int diaSemana = 3;
        String nomeDia;

        switch (diaSemana) {
            case 1:
                nomeDia = "Segunda-feira";
                break; // sem break, cai no próximo case (fall-through)
            case 2:
                nomeDia = "Terça-feira";
                break;
            case 3:
                nomeDia = "Quarta-feira";
                break;
            case 4:
                nomeDia = "Quinta-feira";
                break;
            case 5:
                nomeDia = "Sexta-feira";
                break;
            case 6:
                nomeDia = "Sábado";
                break;
            case 7:
                nomeDia = "Domingo";
                break;
            default:
                nomeDia = "Dia inválido";
        }

        System.out.println("Dia: " + nomeDia); // Quarta-feira

        // Fall-through intencional: agrupar casos
        char vogal = 'a';
        switch (vogal) {
            case 'a':
            case 'e':
            case 'i':
            case 'o':
            case 'u':
                System.out.println(vogal + " é vogal");
                break;
            default:
                System.out.println(vogal + " é consoante");
        }
    }
}
```

### switch Expression (Java 14+)

A partir do Java 14, o `switch` pode ser usado como expressão, eliminando a necessidade de `break` e permitindo retornar um valor diretamente.

```java
public class SwitchExpression {
    public static void main(String[] args) {
        int mes = 4;

        // Switch expression com ->
        String nomeMes = switch (mes) {
            case 1  -> "Janeiro";
            case 2  -> "Fevereiro";
            case 3  -> "Março";
            case 4  -> "Abril";
            case 5  -> "Maio";
            case 6  -> "Junho";
            case 7  -> "Julho";
            case 8  -> "Agosto";
            case 9  -> "Setembro";
            case 10 -> "Outubro";
            case 11 -> "Novembro";
            case 12 -> "Dezembro";
            default -> "Mês inválido";
        };

        System.out.println("Mês: " + nomeMes); // Abril

        // Múltiplos valores por case
        int diasNoMes = switch (mes) {
            case 1, 3, 5, 7, 8, 10, 12 -> 31;
            case 4, 6, 9, 11           -> 30;
            case 2                     -> 28; // simplificado, sem bissexto
            default                    -> -1;
        };

        System.out.println("Dias no mês " + mes + ": " + diasNoMes); // 30

        // Switch com bloco de código e yield
        String estacao = switch (mes) {
            case 12, 1, 2 -> "Verão";
            case 3, 4, 5  -> "Outono";
            case 6, 7, 8  -> {
                System.out.println("(meses de inverno)");
                yield "Inverno"; // yield retorna o valor no bloco
            }
            case 9, 10, 11 -> "Primavera";
            default         -> "Desconhecido";
        };

        System.out.println("Estação: " + estacao);
    }
}
```

---

## for

O loop `for` é ideal quando se conhece antecipadamente o número de iterações.

**Sintaxe:** `for (inicialização; condição; atualização) { ... }`

```java
public class LoopFor {
    public static void main(String[] args) {
        // Contagem crescente
        System.out.print("Crescente: ");
        for (int i = 0; i < 5; i++) {
            System.out.print(i + " ");
        }
        System.out.println(); // 0 1 2 3 4

        // Contagem decrescente
        System.out.print("Decrescente: ");
        for (int i = 5; i > 0; i--) {
            System.out.print(i + " ");
        }
        System.out.println(); // 5 4 3 2 1

        // Passo diferente de 1
        System.out.print("Pares até 10: ");
        for (int i = 0; i <= 10; i += 2) {
            System.out.print(i + " ");
        }
        System.out.println(); // 0 2 4 6 8 10

        // Tabuada do 7
        System.out.println("\nTabuada do 7:");
        for (int i = 1; i <= 10; i++) {
            System.out.println("7 x " + i + " = " + (7 * i));
        }

        // For aninhado: tabuada completa
        System.out.println("\nTabuada (2 a 4):");
        for (int i = 2; i <= 4; i++) {
            for (int j = 1; j <= 5; j++) {
                System.out.printf("%d x %d = %2d  ", i, j, i * j);
            }
            System.out.println();
        }
    }
}
```

---

## while

O `while` repete o bloco enquanto a condição for `true`. Use quando não se sabe antecipadamente quantas iterações ocorrerão.

```java
public class LoopWhile {
    public static void main(String[] args) {
        // Contagem básica
        int contador = 1;
        while (contador <= 5) {
            System.out.print(contador + " ");
            contador++;
        }
        System.out.println(); // 1 2 3 4 5

        // Simulação de saque bancário
        double saldo = 500.0;
        double valorSaque = 120.0;

        System.out.println("\n=== Simulação de Saques ===");
        System.out.println("Saldo inicial: R$ " + saldo);

        while (saldo >= valorSaque) {
            saldo -= valorSaque;
            System.out.printf("Saque de R$ %.2f | Saldo restante: R$ %.2f%n",
                valorSaque, saldo);
        }

        System.out.printf("Saldo insuficiente. Restam R$ %.2f%n", saldo);

        // Encontrando a primeira potência de 2 maior que 1000
        int potencia = 1;
        int expoente = 0;
        while (potencia <= 1000) {
            potencia *= 2;
            expoente++;
        }
        System.out.println("\nPrimeira potência de 2 > 1000: 2^" + expoente + " = " + potencia);
    }
}
```

---

## do-while

O `do-while` garante que o bloco seja executado **pelo menos uma vez**, pois a condição é verificada ao final.

```java
public class LoopDoWhile {
    public static void main(String[] args) {
        // Comparação: while vs do-while com condição falsa desde o início
        int x = 10;

        System.out.print("while (x < 5): ");
        while (x < 5) {
            System.out.print(x + " ");
            x++;
        }
        System.out.println("(não executou)");

        x = 10;
        System.out.print("do-while (x < 5): ");
        do {
            System.out.print(x + " "); // executa ao menos uma vez
            x++;
        } while (x < 5);
        System.out.println("(executou uma vez)");

        // Caso de uso clássico: menu de opções
        // (simulado sem Scanner para manter o exemplo autocontido)
        int opcao;
        int tentativa = 0;
        int[] entradas = {0, 0, 2}; // simula o usuário digitando 0, 0 e depois 2

        System.out.println("\n=== Menu ===");
        do {
            opcao = entradas[tentativa++];
            System.out.println("Opção escolhida: " + opcao);
            if (opcao == 0) {
                System.out.println("Opção inválida, tente novamente.");
            }
        } while (opcao == 0);

        System.out.println("Opção válida selecionada: " + opcao);
    }
}
```

---

## break e continue

### break

Encerra o loop ou bloco `switch` imediatamente.

```java
public class UsandoBreak {
    public static void main(String[] args) {
        // Encontrar o primeiro múltiplo de 7 maior que 50
        for (int i = 51; i <= 200; i++) {
            if (i % 7 == 0) {
                System.out.println("Primeiro múltiplo de 7 > 50: " + i); // 56
                break; // sai do loop
            }
        }

        // Break em loop aninhado: sai apenas do loop mais interno
        System.out.println("\nMatriz até encontrar 6:");
        for (int i = 1; i <= 3; i++) {
            for (int j = 1; j <= 3; j++) {
                if (i * j >= 6) {
                    System.out.println("Parou em i=" + i + ", j=" + j);
                    break; // sai apenas do for interno
                }
                System.out.print(i * j + " ");
            }
        }
        System.out.println();
    }
}
```

### continue

Pula o restante do bloco atual e vai para a próxima iteração.

```java
public class UsandoContinue {
    public static void main(String[] args) {
        // Imprimir apenas números ímpares de 1 a 10
        System.out.print("Ímpares: ");
        for (int i = 1; i <= 10; i++) {
            if (i % 2 == 0) {
                continue; // pula os pares
            }
            System.out.print(i + " ");
        }
        System.out.println(); // 1 3 5 7 9

        // Processar lista pulando valores inválidos
        int[] dados = {10, -5, 30, -1, 0, 45, -20, 8};
        int soma = 0;
        int contagem = 0;

        System.out.println("\nProcessando dados (ignorando negativos e zero):");
        for (int valor : dados) {
            if (valor <= 0) {
                System.out.println("Ignorando: " + valor);
                continue;
            }
            soma += valor;
            contagem++;
            System.out.println("Processado: " + valor);
        }

        System.out.println("Soma: " + soma + ", Contagem: " + contagem);
    }
}
```

---

## for-each (Enhanced for)

O `for-each` itera sobre arrays e coleções de forma mais simples e legível, sem precisar gerenciar índices.

**Sintaxe:** `for (Tipo elemento : colecao) { ... }`

```java
import java.util.ArrayList;
import java.util.List;

public class ForEach {
    public static void main(String[] args) {
        // Com array
        String[] frutas = {"Maçã", "Banana", "Cereja", "Damasco"};

        System.out.println("Frutas disponíveis:");
        for (String fruta : frutas) {
            System.out.println("  - " + fruta);
        }

        // Com array de inteiros
        int[] numeros = {3, 7, 2, 9, 1, 5};
        int soma = 0;
        int maximo = numeros[0];

        for (int num : numeros) {
            soma += num;
            if (num > maximo) {
                maximo = num;
            }
        }

        System.out.println("\nNúmeros: {3, 7, 2, 9, 1, 5}");
        System.out.println("Soma:    " + soma);
        System.out.println("Máximo:  " + maximo);

        // Com List (coleção)
        List<String> linguagens = new ArrayList<>();
        linguagens.add("Java");
        linguagens.add("Python");
        linguagens.add("JavaScript");
        linguagens.add("Go");

        System.out.println("\nLinguagens de programação:");
        for (String linguagem : linguagens) {
            System.out.println("  * " + linguagem);
        }

        // Limitação do for-each: não fornece o índice
        // Para isso, use o for tradicional:
        System.out.println("\nCom índice:");
        for (int i = 0; i < frutas.length; i++) {
            System.out.println("  [" + i + "] " + frutas[i]);
        }
    }
}
```

---

## Exemplo Completo

```java
public class JogoAdivinhacao {
    public static void main(String[] args) {
        int numerosecreto = 42;
        int[] tentativas = {10, 75, 42}; // simula as tentativas do jogador
        int maxTentativas = 5;
        boolean acertou = false;

        System.out.println("=== Jogo de Adivinhação ===");
        System.out.println("Adivinhe o número entre 1 e 100!");

        for (int i = 0; i < maxTentativas; i++) {
            int palpite = tentativas[i < tentativas.length ? i : tentativas.length - 1];
            System.out.println("\nTentativa " + (i + 1) + ": " + palpite);

            if (palpite == numerosecreto) {
                System.out.println("Parabéns! Você acertou em " + (i + 1) + " tentativa(s)!");
                acertou = true;
                break;
            } else if (palpite < numerosecreto) {
                System.out.println("Muito baixo! Tente um número maior.");
            } else {
                System.out.println("Muito alto! Tente um número menor.");
            }

            int restantes = maxTentativas - i - 1;
            if (restantes > 0) {
                System.out.println("Tentativas restantes: " + restantes);
            }
        }

        if (!acertou) {
            System.out.println("\nGame over! O número era: " + numerosecreto);
        }

        // Mostrar estatísticas com switch expression
        int tentativasUsadas = acertou ? 2 : maxTentativas;
        String avaliacao = switch (tentativasUsadas) {
            case 1 -> "Incrível! Na primeira tentativa!";
            case 2 -> "Excelente! Muito rápido!";
            case 3 -> "Muito bom!";
            case 4 -> "Bom!";
            default -> "Continue praticando!";
        };
        System.out.println("\nAvaliação: " + avaliacao);
    }
}
```
