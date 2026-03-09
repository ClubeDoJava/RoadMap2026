# 1.2 - Tipos de Dados

Em Java, toda variável precisa ter um tipo definido em tempo de compilação. Os tipos se dividem em dois grandes grupos: **tipos primitivos** e **tipos de referência**.

---

## Tipos Primitivos

Java possui 8 tipos primitivos. Eles armazenam valores diretamente na memória (stack) e não são objetos.

| Tipo      | Tamanho  | Intervalo                                          | Valor padrão |
|-----------|----------|----------------------------------------------------|--------------|
| `byte`    | 8 bits   | -128 a 127                                         | `0`          |
| `short`   | 16 bits  | -32.768 a 32.767                                   | `0`          |
| `int`     | 32 bits  | -2.147.483.648 a 2.147.483.647                     | `0`          |
| `long`    | 64 bits  | -9,2 × 10¹⁸ a 9,2 × 10¹⁸                          | `0L`         |
| `float`   | 32 bits  | ~±3,4 × 10³⁸ (6-7 dígitos decimais)               | `0.0f`       |
| `double`  | 64 bits  | ~±1,8 × 10³⁰⁸ (15-16 dígitos decimais)            | `0.0d`       |
| `char`    | 16 bits  | '\u0000' a '\uffff' (0 a 65.535)                   | `'\u0000'`   |
| `boolean` | ~1 bit   | `true` ou `false`                                  | `false`      |

```java
public class TiposPrimitivos {
    public static void main(String[] args) {
        byte idade = 25;
        short ano = 2026;
        int populacao = 215000000;
        long distanciaEstelar = 9_460_730_472_580_800L; // underscores para legibilidade
        float preco = 19.99f;
        double pi = 3.141592653589793;
        char letra = 'J';
        boolean ativo = true;

        System.out.println("byte:    " + idade);
        System.out.println("short:   " + ano);
        System.out.println("int:     " + populacao);
        System.out.println("long:    " + distanciaEstelar);
        System.out.println("float:   " + preco);
        System.out.println("double:  " + pi);
        System.out.println("char:    " + letra);
        System.out.println("boolean: " + ativo);
    }
}
```

---

## Tipos de Referência

Tipos de referência armazenam o **endereço** do objeto na memória (heap), não o valor diretamente. O valor padrão de qualquer tipo de referência é `null`.

### String

`String` é uma sequência de caracteres. É um dos tipos mais usados em Java.

```java
String nome = "Java";
String vazio = null; // valor padrão para referências
```

### Arrays

Arrays armazenam múltiplos valores do mesmo tipo em uma sequência indexada.

```java
int[] numeros = {1, 2, 3, 4, 5};
String[] frutas = new String[3]; // array de 3 posições, cada uma inicia com null
```

### Objetos

Qualquer instância de uma classe é um tipo de referência.

```java
Scanner scanner = new Scanner(System.in);
ArrayList<String> lista = new ArrayList<>();
```

---

## Valores Padrão

Quando uma variável de instância (campo de classe) é declarada sem inicialização, Java atribui um valor padrão automaticamente. Variáveis locais **não** recebem valor padrão e precisam ser inicializadas antes do uso.

```java
public class ValoresPadrao {
    // Campos de instância recebem valores padrão
    byte campoByte;       // 0
    short campoShort;     // 0
    int campoInt;         // 0
    long campoLong;       // 0L
    float campoFloat;     // 0.0f
    double campoDouble;   // 0.0d
    char campoChar;       // '\u0000'
    boolean campoBoolean; // false
    String campoString;   // null

    public static void main(String[] args) {
        ValoresPadrao obj = new ValoresPadrao();

        System.out.println("int padrão:     " + obj.campoInt);
        System.out.println("double padrão:  " + obj.campoDouble);
        System.out.println("boolean padrão: " + obj.campoBoolean);
        System.out.println("String padrão:  " + obj.campoString);

        // Variável local: DEVE ser inicializada antes do uso
        int localSemValor;
        // System.out.println(localSemValor); // Erro de compilação!
        localSemValor = 10;
        System.out.println("local:          " + localSemValor);
    }
}
```

---

## Conversão de Tipos (Casting)

### Conversão Implícita (Widening)

Quando convertemos um tipo menor para um tipo maior, Java faz a conversão automaticamente sem perda de dados.

```
byte → short → int → long → float → double
char → int
```

```java
public class ConversaoImplicita {
    public static void main(String[] args) {
        int numeroInt = 100;
        long numeroLong = numeroInt;   // int → long (automático)
        double numeroDouble = numeroLong; // long → double (automático)

        System.out.println("int:    " + numeroInt);
        System.out.println("long:   " + numeroLong);
        System.out.println("double: " + numeroDouble);

        char c = 'A';
        int codigoAscii = c; // char → int (automático)
        System.out.println("char 'A' como int: " + codigoAscii); // 65
    }
}
```

### Conversão Explícita (Narrowing / Casting)

Quando convertemos um tipo maior para um tipo menor, pode haver perda de dados. É necessário usar o operador de cast `(tipo)`.

```java
public class ConversaoExplicita {
    public static void main(String[] args) {
        double valorDouble = 9.99;
        int valorInt = (int) valorDouble; // trunca a parte decimal
        System.out.println("double 9.99 → int: " + valorInt); // 9

        long numeroGrande = 1_000_000_000_000L;
        int numeroInt = (int) numeroGrande; // pode perder dados!
        System.out.println("long → int: " + numeroInt);

        int numero = 65;
        char letra = (char) numero;
        System.out.println("int 65 → char: " + letra); // A

        // Conversão entre String e tipos numéricos
        String textoNumero = "42";
        int convertido = Integer.parseInt(textoNumero);
        double convertidoDouble = Double.parseDouble("3.14");

        System.out.println("String \"42\" → int: " + convertido);
        System.out.println("String \"3.14\" → double: " + convertidoDouble);

        // int → String
        int valor = 100;
        String textoValor = String.valueOf(valor);
        String textoValor2 = Integer.toString(valor);
        System.out.println("int → String: \"" + textoValor + "\"");
    }
}
```

---

## Wrapper Classes

Cada tipo primitivo possui uma **classe wrapper** correspondente no pacote `java.lang`. Elas são necessárias quando precisamos usar primitivos como objetos (por exemplo, em coleções como `ArrayList`).

| Primitivo | Wrapper Class |
|-----------|---------------|
| `byte`    | `Byte`        |
| `short`   | `Short`       |
| `int`     | `Integer`     |
| `long`    | `Long`        |
| `float`   | `Float`       |
| `double`  | `Double`      |
| `char`    | `Character`   |
| `boolean` | `Boolean`     |

### Autoboxing e Unboxing

Java converte automaticamente entre primitivos e wrappers quando necessário.

```java
import java.util.ArrayList;

public class WrapperClasses {
    public static void main(String[] args) {
        // Autoboxing: int → Integer (automático)
        Integer numero = 42;

        // Unboxing: Integer → int (automático)
        int primitivo = numero;

        System.out.println("Integer: " + numero);
        System.out.println("int:     " + primitivo);

        // Métodos úteis das wrapper classes
        System.out.println("Máximo de int:  " + Integer.MAX_VALUE);
        System.out.println("Mínimo de int:  " + Integer.MIN_VALUE);
        System.out.println("Máximo de long: " + Long.MAX_VALUE);

        // Conversões
        String binario = Integer.toBinaryString(42);
        String hexadecimal = Integer.toHexString(255);
        String octal = Integer.toOctalString(8);

        System.out.println("42 em binário:     " + binario);     // 101010
        System.out.println("255 em hexadecimal:" + hexadecimal); // ff
        System.out.println("8 em octal:        " + octal);       // 10

        // Wrappers em coleções (ArrayList não aceita primitivos)
        ArrayList<Integer> lista = new ArrayList<>();
        lista.add(10);  // autoboxing de int para Integer
        lista.add(20);
        lista.add(30);

        int soma = 0;
        for (Integer i : lista) {
            soma += i; // unboxing de Integer para int
        }
        System.out.println("Soma: " + soma);

        // Comparação de Integer: use equals(), não ==
        Integer a = 200;
        Integer b = 200;
        System.out.println("a == b:        " + (a == b));        // false (objetos diferentes)
        System.out.println("a.equals(b):   " + a.equals(b));     // true (mesmo valor)
    }
}
```

> **Dica:** Para valores de `Integer` entre -128 e 127, o Java usa um cache interno, então `==` pode retornar `true`. Para valores fora desse intervalo, `==` compara referências. Sempre use `.equals()` para comparar wrappers.

---

## Exemplo Completo

```java
public class TiposDeDados {
    public static void main(String[] args) {
        // --- Primitivos ---
        int saldo = 1000;
        double taxaJuros = 0.05;
        boolean contaAtiva = true;
        char categoria = 'A';

        // --- Cálculo com casting ---
        double juros = saldo * taxaJuros;
        int jurosTruncado = (int) juros;

        System.out.println("=== Simulação Bancária ===");
        System.out.println("Saldo:          R$ " + saldo);
        System.out.println("Taxa de juros:  " + (taxaJuros * 100) + "%");
        System.out.println("Conta ativa:    " + contaAtiva);
        System.out.println("Categoria:      " + categoria);
        System.out.println("Juros (double): R$ " + juros);
        System.out.println("Juros (int):    R$ " + jurosTruncado);

        // --- Wrapper e conversão de String ---
        String entradaUsuario = "250";
        int deposito = Integer.parseInt(entradaUsuario);
        saldo += deposito;

        System.out.println("Após depósito:  R$ " + saldo);
        System.out.println("Saldo como String: \"" + Integer.toString(saldo) + "\"");
    }
}
```
