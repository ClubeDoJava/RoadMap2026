# 1.3 - Operadores

Operadores são símbolos que realizam operações sobre um ou mais operandos. Java possui um conjunto rico de operadores organizados por categoria.

---

## Operadores Aritméticos

Realizam operações matemáticas básicas.

| Operador | Descrição       | Exemplo         | Resultado |
|----------|-----------------|-----------------|-----------|
| `+`      | Adição          | `5 + 3`         | `8`       |
| `-`      | Subtração       | `5 - 3`         | `2`       |
| `*`      | Multiplicação   | `5 * 3`         | `15`      |
| `/`      | Divisão         | `10 / 3`        | `3` (inteira) |
| `%`      | Módulo (resto)  | `10 % 3`        | `1`       |

```java
public class OperadoresAritmeticos {
    public static void main(String[] args) {
        int a = 10;
        int b = 3;

        System.out.println("a + b = " + (a + b)); // 13
        System.out.println("a - b = " + (a - b)); // 7
        System.out.println("a * b = " + (a * b)); // 30

        // Divisão inteira: descarta a parte decimal
        System.out.println("a / b = " + (a / b)); // 3

        // Para divisão real, ao menos um operando deve ser double/float
        System.out.println("a / (double)b = " + (a / (double) b)); // 3.3333...

        // Módulo: retorna o resto da divisão
        System.out.println("a % b = " + (a % b)); // 1

        // Uso prático do módulo: verificar par/ímpar
        int numero = 17;
        if (numero % 2 == 0) {
            System.out.println(numero + " é par");
        } else {
            System.out.println(numero + " é ímpar");
        }

        // Operador + com Strings: concatenação
        String texto = "Resultado: " + (a + b); // avalia (a+b) primeiro
        System.out.println(texto); // "Resultado: 13"
    }
}
```

---

## Operadores de Atribuição

Combinam uma operação com a atribuição do resultado.

| Operador | Equivalente a  | Exemplo       |
|----------|----------------|---------------|
| `=`      | —              | `x = 5`       |
| `+=`     | `x = x + n`   | `x += 3`      |
| `-=`     | `x = x - n`   | `x -= 3`      |
| `*=`     | `x = x * n`   | `x *= 3`      |
| `/=`     | `x = x / n`   | `x /= 3`      |
| `%=`     | `x = x % n`   | `x %= 3`      |

```java
public class OperadoresAtribuicao {
    public static void main(String[] args) {
        int pontos = 100;
        System.out.println("Pontos iniciais: " + pontos);

        pontos += 50;  // pontos = pontos + 50
        System.out.println("Após += 50:  " + pontos); // 150

        pontos -= 20;  // pontos = pontos - 20
        System.out.println("Após -= 20:  " + pontos); // 130

        pontos *= 2;   // pontos = pontos * 2
        System.out.println("Após *= 2:   " + pontos); // 260

        pontos /= 4;   // pontos = pontos / 4
        System.out.println("Após /= 4:   " + pontos); // 65

        pontos %= 10;  // pontos = pontos % 10
        System.out.println("Após %= 10:  " + pontos); // 5
    }
}
```

---

## Operadores de Comparação (Relacionais)

Comparam dois valores e retornam `boolean` (`true` ou `false`).

| Operador | Descrição        | Exemplo  | Resultado |
|----------|------------------|----------|-----------|
| `==`     | Igual a          | `5 == 5` | `true`    |
| `!=`     | Diferente de     | `5 != 3` | `true`    |
| `<`      | Menor que        | `3 < 5`  | `true`    |
| `>`      | Maior que        | `5 > 3`  | `true`    |
| `<=`     | Menor ou igual   | `5 <= 5` | `true`    |
| `>=`     | Maior ou igual   | `3 >= 5` | `false`   |

```java
public class OperadoresComparacao {
    public static void main(String[] args) {
        int x = 10;
        int y = 20;

        System.out.println("x == y: " + (x == y)); // false
        System.out.println("x != y: " + (x != y)); // true
        System.out.println("x < y:  " + (x < y));  // true
        System.out.println("x > y:  " + (x > y));  // false
        System.out.println("x <= y: " + (x <= y)); // true
        System.out.println("x >= y: " + (x >= y)); // false

        // ATENÇÃO: para objetos, == compara referências, não conteúdo
        String s1 = new String("Java");
        String s2 = new String("Java");
        System.out.println("s1 == s2:       " + (s1 == s2));       // false (referências diferentes)
        System.out.println("s1.equals(s2):  " + s1.equals(s2));    // true (mesmo conteúdo)

        // Comparação com primitivos funciona corretamente com ==
        int a = 5;
        int b = 5;
        System.out.println("a == b (primitivos): " + (a == b)); // true
    }
}
```

---

## Operadores Lógicos

Combinam expressões booleanas.

| Operador | Descrição           | Resultado                                 |
|----------|---------------------|-------------------------------------------|
| `&&`     | E lógico (AND)      | `true` somente se ambos forem `true`      |
| `\|\|`     | OU lógico (OR)      | `true` se ao menos um for `true`          |
| `!`      | NÃO lógico (NOT)    | Inverte o valor booleano                  |

```java
public class OperadoresLogicos {
    public static void main(String[] args) {
        boolean temConta = true;
        boolean temSaldo = false;
        boolean maiorDeIdade = true;

        // AND: todos devem ser verdadeiros
        System.out.println("Pode sacar (tem conta E saldo): "
            + (temConta && temSaldo)); // false

        // OR: basta um ser verdadeiro
        System.out.println("Pode continuar (conta OU saldo): "
            + (temConta || temSaldo)); // true

        // NOT: inverte
        System.out.println("Não tem saldo: " + !temSaldo); // true

        // Combinando operadores
        boolean podeAbrirContaJunior = !maiorDeIdade && temConta;
        System.out.println("Pode conta junior: " + podeAbrirContaJunior); // false

        // Curto-circuito (short-circuit evaluation)
        // Em &&: se o primeiro for false, o segundo NÃO é avaliado
        // Em ||: se o primeiro for true, o segundo NÃO é avaliado
        int divisor = 0;
        boolean resultado = (divisor != 0) && (10 / divisor > 1); // seguro: não divide por zero
        System.out.println("Resultado seguro: " + resultado); // false

        // Tabela verdade
        System.out.println("\n--- Tabela Verdade AND ---");
        System.out.println("true  && true  = " + (true && true));   // true
        System.out.println("true  && false = " + (true && false));  // false
        System.out.println("false && true  = " + (false && true));  // false
        System.out.println("false && false = " + (false && false)); // false

        System.out.println("\n--- Tabela Verdade OR ---");
        System.out.println("true  || true  = " + (true || true));   // true
        System.out.println("true  || false = " + (true || false));  // true
        System.out.println("false || true  = " + (false || true));  // true
        System.out.println("false || false = " + (false || false)); // false
    }
}
```

---

## Operador Ternário

Uma forma compacta de escrever um `if-else` simples em uma única linha.

**Sintaxe:** `condição ? valorSeVerdadeiro : valorSeFalso`

```java
public class OperadorTernario {
    public static void main(String[] args) {
        int idade = 20;

        // Forma tradicional com if-else
        String status;
        if (idade >= 18) {
            status = "Maior de idade";
        } else {
            status = "Menor de idade";
        }

        // Equivalente com operador ternário
        String statusTernario = (idade >= 18) ? "Maior de idade" : "Menor de idade";

        System.out.println(status);        // Maior de idade
        System.out.println(statusTernario); // Maior de idade

        // Exemplos práticos
        int a = 15;
        int b = 28;
        int maior = (a > b) ? a : b;
        System.out.println("Maior entre " + a + " e " + b + ": " + maior); // 28

        double nota = 7.5;
        String aprovado = (nota >= 6.0) ? "Aprovado" : "Reprovado";
        System.out.println("Situação: " + aprovado); // Aprovado

        // Ternário aninhado (use com moderação — pode prejudicar a legibilidade)
        int numero = 0;
        String sinal = (numero > 0) ? "positivo" : (numero < 0) ? "negativo" : "zero";
        System.out.println("O número é: " + sinal); // zero
    }
}
```

---

## Operadores de Incremento e Decremento

Aumentam ou diminuem o valor de uma variável numérica em 1.

| Operador | Descrição           | Forma       |
|----------|---------------------|-------------|
| `++`     | Incremento de 1     | Pré ou pós  |
| `--`     | Decremento de 1     | Pré ou pós  |

- **Pré-fixado** (`++x`): incrementa **antes** de usar o valor na expressão.
- **Pós-fixado** (`x++`): usa o valor na expressão e incrementa **depois**.

```java
public class IncrementoDecremento {
    public static void main(String[] args) {
        int x = 5;

        // Pós-fixado: retorna o valor atual, depois incrementa
        System.out.println("x++: " + x++); // imprime 5, depois x vira 6
        System.out.println("x agora: " + x); // 6

        // Pré-fixado: incrementa primeiro, depois retorna o valor
        System.out.println("++x: " + ++x); // x vira 7, imprime 7
        System.out.println("x agora: " + x); // 7

        // Decremento pós-fixado
        System.out.println("x--: " + x--); // imprime 7, depois x vira 6
        System.out.println("x agora: " + x); // 6

        // Decremento pré-fixado
        System.out.println("--x: " + --x); // x vira 5, imprime 5
        System.out.println("x agora: " + x); // 5

        // Uso clássico em loops
        System.out.println("\nContagem com ++:");
        for (int i = 0; i < 5; i++) {
            System.out.print(i + " ");
        }

        System.out.println("\nContagem regressiva com --:");
        for (int i = 5; i > 0; i--) {
            System.out.print(i + " ");
        }
        System.out.println();
    }
}
```

---

## Precedência de Operadores

Quando uma expressão contém múltiplos operadores, a ordem de avaliação segue uma hierarquia. Operadores com maior precedência são avaliados primeiro.

| Precedência | Operadores                             |
|-------------|----------------------------------------|
| 1 (maior)   | `++` `--` (pós-fixado)                 |
| 2           | `++` `--` `+` `-` `!` (pré-fixado/unário) |
| 3           | `*` `/` `%`                            |
| 4           | `+` `-`                                |
| 5           | `<` `>` `<=` `>=`                      |
| 6           | `==` `!=`                              |
| 7           | `&&`                                   |
| 8           | `\|\|`                                   |
| 9           | `?:` (ternário)                        |
| 10 (menor)  | `=` `+=` `-=` `*=` `/=` `%=`          |

```java
public class PrecedenciaOperadores {
    public static void main(String[] args) {
        // Sem parênteses: * tem precedência sobre +
        int resultado1 = 2 + 3 * 4;
        System.out.println("2 + 3 * 4 = " + resultado1); // 14, não 20

        // Com parênteses: forçamos a ordem
        int resultado2 = (2 + 3) * 4;
        System.out.println("(2 + 3) * 4 = " + resultado2); // 20

        // Misturando lógicos e comparação
        // && tem precedência sobre ||
        boolean r = true || false && false;
        // Equivale a: true || (false && false) = true || false = true
        System.out.println("true || false && false = " + r); // true

        boolean r2 = (true || false) && false;
        System.out.println("(true || false) && false = " + r2); // false

        // Expressão complexa
        int a = 5, b = 3, c = 2;
        int calc = a + b * c - a / c;
        // Avaliação: a + (b*c) - (a/c) = 5 + 6 - 2 = 9
        System.out.println("5 + 3 * 2 - 5 / 2 = " + calc); // 9

        // Dica: use parênteses para deixar a intenção clara
        int calcClaro = a + (b * c) - (a / c);
        System.out.println("Com parênteses explícitos: " + calcClaro); // 9
    }
}
```

> **Regra de ouro:** quando tiver dúvida sobre precedência, use parênteses. Além de garantir o resultado correto, o código fica mais legível.

---

## Exemplo Completo

```java
public class CalculadoraSimples {
    public static void main(String[] args) {
        double a = 15.0;
        double b = 4.0;

        // Operações aritméticas
        System.out.println("=== Calculadora ===");
        System.out.println("a = " + a + ", b = " + b);
        System.out.println("Soma:       " + (a + b));
        System.out.println("Subtração:  " + (a - b));
        System.out.println("Produto:    " + (a * b));
        System.out.println("Divisão:    " + (a / b));
        System.out.println("Resto:      " + (a % b));

        // Atribuição composta
        double acumulador = 100.0;
        acumulador += a;   // 115.0
        acumulador -= b;   // 111.0
        acumulador *= 2;   // 222.0
        acumulador /= 3;   // 74.0
        System.out.println("\nAcumulador final: " + acumulador);

        // Comparações e lógicos
        boolean maiorQueB = a > b;
        boolean ambosPositivos = (a > 0) && (b > 0);
        System.out.println("\na > b:                " + maiorQueB);
        System.out.println("a e b são positivos:  " + ambosPositivos);

        // Ternário para classificação
        String classificacao = (a > 10) ? "grande" : "pequeno";
        System.out.println("a é: " + classificacao);

        // Incremento em contador
        int contador = 0;
        contador++;
        contador++;
        contador++;
        System.out.println("\nOperações realizadas: " + contador);
    }
}
```
