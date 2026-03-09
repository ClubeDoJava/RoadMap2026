# 1.5 - Métodos

Métodos são blocos de código nomeados que executam uma tarefa específica. Eles promovem a **reutilização** de código, melhoram a **organização** e facilitam a **manutenção** do programa.

---

## Declaração de Método

**Sintaxe básica:**

```
[modificador] tipoDeRetorno nomeDoMetodo([parâmetros]) {
    // corpo do método
    return valor; // obrigatório se tipoDeRetorno != void
}
```

```java
public class DeclaracaoMetodo {

    // Estrutura completa de um método
    public static int somar(int a, int b) {
        int resultado = a + b;
        return resultado;
    }

    public static void main(String[] args) {
        int soma = somar(5, 3);
        System.out.println("5 + 3 = " + soma); // 8
    }
}
```

Partes da declaração:
- `public` — modificador de acesso (visível de qualquer lugar)
- `static` — pertence à classe, não a uma instância
- `int` — tipo do valor retornado
- `somar` — nome do método (camelCase por convenção)
- `(int a, int b)` — lista de parâmetros

---

## Tipos de Retorno

### void — sem retorno

Métodos `void` realizam uma ação mas não devolvem nenhum valor.

```java
public class RetornoVoid {

    // Método void: apenas imprime, não retorna valor
    public static void exibirBoasVindas(String nome) {
        System.out.println("Olá, " + nome + "! Bem-vindo ao sistema.");
        // Não há return aqui (ou pode ser usado return; sem valor para sair antecipadamente)
    }

    public static void exibirLinha(int tamanho) {
        for (int i = 0; i < tamanho; i++) {
            System.out.print("-");
        }
        System.out.println();
    }

    public static void main(String[] args) {
        exibirBoasVindas("Ana");
        exibirLinha(30);
        exibirBoasVindas("Carlos");
        exibirLinha(30);
    }
}
```

### Com retorno

O método retorna um valor com a instrução `return`. O tipo do valor retornado deve corresponder ao tipo declarado.

```java
public class RetornoComValor {

    // Retorna int
    public static int calcularQuadrado(int numero) {
        return numero * numero;
    }

    // Retorna double
    public static double calcularMedia(double[] valores) {
        double soma = 0;
        for (double v : valores) {
            soma += v;
        }
        return soma / valores.length;
    }

    // Retorna boolean
    public static boolean ehPrimo(int numero) {
        if (numero < 2) return false;
        for (int i = 2; i <= Math.sqrt(numero); i++) {
            if (numero % i == 0) return false;
        }
        return true;
    }

    // Retorna String
    public static String classificarNota(double nota) {
        if (nota >= 9.0) return "A";
        if (nota >= 7.0) return "B";
        if (nota >= 5.0) return "C";
        return "D";
    }

    public static void main(String[] args) {
        System.out.println("Quadrado de 7: " + calcularQuadrado(7)); // 49

        double[] notas = {8.5, 7.0, 9.5, 6.0, 8.0};
        System.out.printf("Média: %.2f%n", calcularMedia(notas)); // 7.80

        System.out.println("17 é primo? " + ehPrimo(17)); // true
        System.out.println("18 é primo? " + ehPrimo(18)); // false

        System.out.println("Conceito 8.5: " + classificarNota(8.5)); // B
    }
}
```

---

## Parâmetros e Argumentos

- **Parâmetros** são as variáveis declaradas na assinatura do método.
- **Argumentos** são os valores passados ao chamar o método.

Java passa primitivos **por valor** (cópias) e objetos **por referência de valor** (a referência é copiada, mas aponta para o mesmo objeto).

```java
import java.util.Arrays;

public class ParametrosArgumentos {

    // Parâmetros com valores padrão simulados via sobrecarga (ver próxima seção)
    public static void imprimirInfo(String nome, int idade, String cidade) {
        System.out.printf("Nome: %s | Idade: %d | Cidade: %s%n", nome, idade, cidade);
    }

    // Passagem por valor: primitivo não é alterado fora do método
    public static void tentarAlterarPrimitivo(int numero) {
        numero = 999; // altera apenas a cópia local
        System.out.println("Dentro do método: " + numero);
    }

    // Passagem por referência de valor: array é alterado
    public static void dobrarElementos(int[] array) {
        for (int i = 0; i < array.length; i++) {
            array[i] *= 2; // altera o objeto original
        }
    }

    // Varargs: número variável de argumentos (deve ser o último parâmetro)
    public static int somarTodos(int... numeros) {
        int soma = 0;
        for (int n : numeros) {
            soma += n;
        }
        return soma;
    }

    public static void main(String[] args) {
        imprimirInfo("Maria", 28, "São Paulo");
        imprimirInfo("João", 35, "Rio de Janeiro");

        // Passagem por valor
        int valor = 10;
        tentarAlterarPrimitivo(valor);
        System.out.println("Após o método (primitivo): " + valor); // ainda 10

        // Passagem por referência de valor
        int[] numeros = {1, 2, 3, 4, 5};
        System.out.println("Antes: " + Arrays.toString(numeros));
        dobrarElementos(numeros);
        System.out.println("Depois: " + Arrays.toString(numeros)); // valores dobrados

        // Varargs
        System.out.println("Soma(1,2): "        + somarTodos(1, 2));
        System.out.println("Soma(1,2,3,4,5): "  + somarTodos(1, 2, 3, 4, 5));
        System.out.println("Soma(): "           + somarTodos()); // 0
    }
}
```

---

## Sobrecarga de Método (Method Overloading)

Sobrecarga permite criar múltiplos métodos com o **mesmo nome**, mas com **parâmetros diferentes** (tipo, quantidade ou ordem). O compilador escolhe qual versão chamar com base nos argumentos fornecidos.

```java
public class Sobrecarga {

    // Versão 1: dois inteiros
    public static int somar(int a, int b) {
        System.out.println("somar(int, int)");
        return a + b;
    }

    // Versão 2: três inteiros
    public static int somar(int a, int b, int c) {
        System.out.println("somar(int, int, int)");
        return a + b + c;
    }

    // Versão 3: dois doubles
    public static double somar(double a, double b) {
        System.out.println("somar(double, double)");
        return a + b;
    }

    // Versão 4: String (concatenação)
    public static String somar(String a, String b) {
        System.out.println("somar(String, String)");
        return a + b;
    }

    // Sobrecarga para simular parâmetro opcional
    public static void criarUsuario(String nome, int idade, String pais) {
        System.out.printf("Usuário: %s, %d anos, %s%n", nome, idade, pais);
    }

    // Versão sem o campo "pais" (valor padrão aplicado internamente)
    public static void criarUsuario(String nome, int idade) {
        criarUsuario(nome, idade, "Brasil"); // chama a versão completa
    }

    public static void main(String[] args) {
        System.out.println(somar(2, 3));             // 5
        System.out.println(somar(2, 3, 4));          // 9
        System.out.println(somar(2.5, 3.5));         // 6.0
        System.out.println(somar("Olá, ", "Mundo")); // Olá, Mundo

        System.out.println();
        criarUsuario("Alice", 25, "Portugal");
        criarUsuario("Bob", 30); // usa "Brasil" por padrão
    }
}
```

---

## Métodos Estáticos vs Métodos de Instância

### Métodos Estáticos (`static`)

Pertencem à **classe** e podem ser chamados sem criar um objeto. São ideais para utilitários e funções independentes de estado.

### Métodos de Instância

Pertencem a um **objeto** (instância) e podem acessar os campos da instância via `this`. Precisam de um objeto criado para serem chamados.

```java
public class ContaBancaria {
    // Campos de instância
    private String titular;
    private double saldo;

    // Construtor
    public ContaBancaria(String titular, double saldoInicial) {
        this.titular = titular;
        this.saldo = saldoInicial;
    }

    // --- Métodos de Instância ---
    public void depositar(double valor) {
        if (valor <= 0) {
            System.out.println("Valor inválido para depósito.");
            return;
        }
        saldo += valor;
        System.out.printf("Depósito de R$ %.2f realizado. Novo saldo: R$ %.2f%n", valor, saldo);
    }

    public boolean sacar(double valor) {
        if (valor <= 0 || valor > saldo) {
            System.out.println("Saque não autorizado.");
            return false;
        }
        saldo -= valor;
        System.out.printf("Saque de R$ %.2f realizado. Novo saldo: R$ %.2f%n", valor, saldo);
        return true;
    }

    public double getSaldo() {
        return saldo;
    }

    public void exibirExtrato() {
        System.out.printf("Titular: %s | Saldo: R$ %.2f%n", titular, saldo);
    }

    // --- Método Estático ---
    // Não precisa de instância, é um utilitário da classe
    public static double calcularJuros(double valor, double taxaAnual, int meses) {
        return valor * Math.pow(1 + taxaAnual / 12, meses) - valor;
    }

    public static void main(String[] args) {
        // Método estático: chamado diretamente na classe
        double juros = ContaBancaria.calcularJuros(1000.0, 0.12, 6);
        System.out.printf("Juros em 6 meses: R$ %.2f%n", juros);

        System.out.println();

        // Métodos de instância: precisam de um objeto
        ContaBancaria conta = new ContaBancaria("Carlos", 500.0);
        conta.exibirExtrato();
        conta.depositar(200.0);
        conta.sacar(100.0);
        conta.sacar(1000.0); // saldo insuficiente
        conta.exibirExtrato();
    }
}
```

---

## Recursão

Um método é recursivo quando **chama a si mesmo**. Toda recursão precisa de uma **condição de parada** (caso base) para evitar loop infinito.

```java
public class Recursao {

    // Fatorial: n! = n * (n-1) * ... * 1
    // Caso base: 0! = 1
    public static long fatorial(int n) {
        if (n < 0) throw new IllegalArgumentException("n deve ser >= 0");
        if (n == 0 || n == 1) return 1;        // caso base
        return n * fatorial(n - 1);             // chamada recursiva
    }

    // Fibonacci: fib(n) = fib(n-1) + fib(n-2)
    // Casos base: fib(0) = 0, fib(1) = 1
    public static int fibonacci(int n) {
        if (n <= 0) return 0; // caso base
        if (n == 1) return 1; // caso base
        return fibonacci(n - 1) + fibonacci(n - 2); // recursão
    }

    // Soma dos dígitos: soma(123) = 1 + 2 + 3 = 6
    public static int somaDigitos(int n) {
        if (n < 10) return n;             // caso base: um único dígito
        return n % 10 + somaDigitos(n / 10); // último dígito + recursão no restante
    }

    // Potência: base^expoente
    public static double potencia(double base, int expoente) {
        if (expoente == 0) return 1;           // caso base
        if (expoente < 0) return 1 / potencia(base, -expoente); // expoente negativo
        return base * potencia(base, expoente - 1); // recursão
    }

    public static void main(String[] args) {
        // Fatorial
        System.out.println("=== Fatorial ===");
        for (int i = 0; i <= 10; i++) {
            System.out.println(i + "! = " + fatorial(i));
        }

        // Fibonacci
        System.out.println("\n=== Fibonacci ===");
        System.out.print("Sequência: ");
        for (int i = 0; i <= 10; i++) {
            System.out.print(fibonacci(i) + " ");
        }
        System.out.println();

        // Soma de dígitos
        System.out.println("\n=== Soma de Dígitos ===");
        System.out.println("somaDigitos(1234) = " + somaDigitos(1234)); // 10
        System.out.println("somaDigitos(9999) = " + somaDigitos(9999)); // 36

        // Potência
        System.out.println("\n=== Potência ===");
        System.out.println("2^10 = " + (int) potencia(2, 10)); // 1024
        System.out.println("3^5  = " + (int) potencia(3, 5));  // 243
    }
}
```

> **Atenção:** A recursão tem custo de memória (cada chamada ocupa espaço na call stack). Para casos com muitas iterações, prefira abordagens iterativas. A recursão de Fibonacci acima, por exemplo, é O(2ⁿ) — para valores grandes, use programação dinâmica ou iteração.

---

## Exemplo Completo

```java
public class CalculadoraCientifica {

    // Utilitários estáticos
    public static boolean ehPar(int n)         { return n % 2 == 0; }
    public static boolean ehPositivo(double n) { return n > 0; }

    // Operações básicas (sobrecarga)
    public static double calcular(double a, double b, char operacao) {
        return switch (operacao) {
            case '+' -> a + b;
            case '-' -> a - b;
            case '*' -> a * b;
            case '/' -> b != 0 ? a / b : Double.NaN;
            case '%' -> a % b;
            default  -> throw new IllegalArgumentException("Operação desconhecida: " + operacao);
        };
    }

    // Potência recursiva
    public static double potencia(double base, int exp) {
        if (exp == 0) return 1;
        if (exp < 0)  return 1 / potencia(base, -exp);
        return base * potencia(base, exp - 1);
    }

    // Formatação de resultado
    public static String formatarResultado(double valor) {
        if (valor == Math.floor(valor) && !Double.isInfinite(valor)) {
            return String.valueOf((long) valor); // sem casas decimais desnecessárias
        }
        return String.format("%.4f", valor);
    }

    public static void main(String[] args) {
        System.out.println("=== Calculadora Científica ===\n");

        double a = 15.0, b = 4.0;
        char[] ops = {'+', '-', '*', '/', '%'};

        for (char op : ops) {
            double resultado = calcular(a, b, op);
            System.out.printf("%.0f %c %.0f = %s%n", a, op, b, formatarResultado(resultado));
        }

        System.out.println();
        System.out.println("2^10 = " + formatarResultado(potencia(2, 10)));
        System.out.println("5^3  = " + formatarResultado(potencia(5, 3)));
        System.out.println("4^-2 = " + formatarResultado(potencia(4, -2)));

        System.out.println();
        System.out.println("16 é par? "      + ehPar(16));
        System.out.println("-5 é positivo? " + ehPositivo(-5));
    }
}
```
