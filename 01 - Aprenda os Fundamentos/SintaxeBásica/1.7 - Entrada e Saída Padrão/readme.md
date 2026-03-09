# 1.7 - Entrada e Saída Padrão

Todo programa precisa interagir com o mundo externo. Em Java, as saídas de texto são feitas via `System.out` e a leitura de dados do teclado é feita com a classe `Scanner`.

---

## Saída Padrão

### System.out.println

Imprime o valor e **adiciona uma nova linha** ao final.

```java
public class SaidaPrintln {
    public static void main(String[] args) {
        // println com diferentes tipos
        System.out.println("Texto como String");
        System.out.println(42);
        System.out.println(3.14);
        System.out.println(true);
        System.out.println('J');

        // println sem argumento: apenas quebra de linha
        System.out.println("Linha 1");
        System.out.println(); // linha em branco
        System.out.println("Linha 3");

        // Concatenação automática
        String nome = "Java";
        int versao = 21;
        System.out.println("Linguagem: " + nome + " | Versão: " + versao);
    }
}
```

### System.out.print

Imprime o valor **sem** adicionar nova linha ao final. O cursor fica na mesma linha.

```java
public class SaidaPrint {
    public static void main(String[] args) {
        // print: sem quebra de linha
        System.out.print("A");
        System.out.print("B");
        System.out.print("C");
        System.out.println(); // quebra de linha manual

        // Útil para construir saída na mesma linha
        System.out.print("Carregando");
        for (int i = 0; i < 5; i++) {
            System.out.print(".");
        }
        System.out.println(" Pronto!");

        // Comparação visual
        System.out.print("println: ");
        System.out.println("com quebra");
        System.out.print("print: ");
        System.out.print("sem quebra | ");
        System.out.print("continua na mesma linha");
        System.out.println();
    }
}
```

### System.out.printf

Imprime texto formatado, similar ao `printf` do C. **Não adiciona nova linha** automaticamente — use `%n` ou `\n` para isso.

```java
public class SaidaPrintf {
    public static void main(String[] args) {
        String nome = "Maria";
        int idade = 28;
        double salario = 5750.50;
        boolean ativo = true;

        // Especificadores de formato comuns:
        // %s  → String
        // %d  → inteiro decimal (int, long)
        // %f  → ponto flutuante (float, double)
        // %c  → caractere
        // %b  → boolean
        // %n  → nova linha (independente do SO)
        // %%  → literal %

        System.out.printf("Nome:    %s%n", nome);
        System.out.printf("Idade:   %d anos%n", idade);
        System.out.printf("Salário: R$ %f%n", salario);     // muitas casas decimais
        System.out.printf("Salário: R$ %.2f%n", salario);   // 2 casas decimais
        System.out.printf("Ativo:   %b%n", ativo);

        // Largura e alinhamento
        System.out.println("\n=== Tabela de Produtos ===");
        System.out.printf("%-15s %8s %10s%n", "Produto", "Qtd", "Preço");
        System.out.println("-".repeat(35));
        System.out.printf("%-15s %8d %10.2f%n", "Notebook",    5, 3499.90);
        System.out.printf("%-15s %8d %10.2f%n", "Mouse",      20,   89.99);
        System.out.printf("%-15s %8d %10.2f%n", "Teclado",    15,  149.90);
        System.out.printf("%-15s %8d %10.2f%n", "Monitor",     3, 1299.00);

        // Modificadores de largura:
        // %10d  → inteiro com largura 10 (alinhado à direita)
        // %-10d → inteiro com largura 10 (alinhado à esquerda)
        // %010d → inteiro com largura 10, preenchido com zeros
        // %.2f  → float com 2 casas decimais
        // %10.2f → float com largura 10 e 2 casas decimais

        System.out.println("\nExemplos de formatação numérica:");
        System.out.printf("'%10d'%n",   42);    // alinhado à direita
        System.out.printf("'%-10d'%n",  42);    // alinhado à esquerda
        System.out.printf("'%010d'%n",  42);    // preenchido com zeros
        System.out.printf("'%+d'%n",    42);    // com sinal
        System.out.printf("'%.5f'%n",  3.14);   // 5 casas decimais
        System.out.printf("'%e'%n",    12345.6789); // notação científica
    }
}
```

---

## Scanner — Entrada do Usuário

A classe `Scanner` (pacote `java.util`) permite ler dados digitados pelo usuário no console.

### Configuração Básica

```java
import java.util.Scanner;

public class ConfiguracaoScanner {
    public static void main(String[] args) {
        // Criando o Scanner conectado à entrada padrão (teclado)
        Scanner scanner = new Scanner(System.in);

        // ... leitura de dados ...

        // IMPORTANTE: sempre feche o Scanner ao terminar
        scanner.close();
    }
}
```

---

### Lendo String

```java
import java.util.Scanner;

public class LendoString {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // next(): lê uma palavra (para ao encontrar espaço)
        System.out.print("Digite seu primeiro nome: ");
        String primeiroNome = scanner.next();

        // nextLine(): lê a linha inteira (incluindo espaços)
        scanner.nextLine(); // consome o \n que ficou após next()

        System.out.print("Digite seu nome completo: ");
        String nomeCompleto = scanner.nextLine();

        System.out.println("\nPrimeiro nome:  " + primeiroNome);
        System.out.println("Nome completo:  " + nomeCompleto);
        System.out.println("Letras no nome: " + nomeCompleto.replace(" ", "").length());

        scanner.close();
    }
}
```

> **Armadilha comum:** Após chamar `nextInt()`, `nextDouble()` ou `next()`, o caractere de nova linha (`\n`) permanece no buffer. Se o próximo método for `nextLine()`, ele lerá essa linha vazia. Solução: adicione `scanner.nextLine()` para consumir o `\n` restante antes de chamar `nextLine()`.

---

### Lendo int e double

```java
import java.util.Scanner;

public class LendoNumericos {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Lendo int
        System.out.print("Digite sua idade: ");
        int idade = scanner.nextInt();

        // Lendo long
        System.out.print("Digite um número grande: ");
        long grande = scanner.nextLong();

        // Lendo double
        System.out.print("Digite sua altura (ex: 1.75): ");
        double altura = scanner.nextDouble();

        // Lendo float
        System.out.print("Digite seu peso (ex: 70.5): ");
        float peso = scanner.nextFloat();

        // Lendo boolean
        System.out.print("Você tem CPF? (true/false): ");
        boolean temCPF = scanner.nextBoolean();

        System.out.println("\n=== Dados informados ===");
        System.out.println("Idade:    " + idade + " anos");
        System.out.println("Grande:   " + grande);
        System.out.printf("Altura:   %.2f m%n", altura);
        System.out.printf("Peso:     %.1f kg%n", peso);
        System.out.println("Tem CPF:  " + temCPF);

        // Cálculo do IMC
        double imc = peso / (altura * altura);
        System.out.printf("IMC:      %.2f%n", imc);

        scanner.close();
    }
}
```

---

### Verificando Tipo antes de Ler

Use os métodos `hasNextInt()`, `hasNextDouble()`, etc., para verificar se a entrada é do tipo esperado antes de ler, evitando `InputMismatchException`.

```java
import java.util.Scanner;

public class VerificandoTipo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Digite um número inteiro: ");
        if (scanner.hasNextInt()) {
            int numero = scanner.nextInt();
            System.out.println("Você digitou: " + numero);
        } else {
            System.out.println("Isso não é um inteiro!");
            scanner.next(); // consome a entrada inválida
        }

        scanner.close();
    }
}
```

---

## Formatação de Saída com printf

### Tabela de Especificadores de Formato

| Especificador | Tipo              | Exemplo               | Saída           |
|---------------|-------------------|-----------------------|-----------------|
| `%s`          | String            | `"%-10s", "Java"`     | `"Java      "`  |
| `%d`          | int / long        | `"%05d", 42`          | `"00042"`       |
| `%f`          | float / double    | `"%.2f", 3.14159`     | `"3.14"`        |
| `%e`          | notação científica| `"%e", 12345.0`       | `"1.234500e+04"` |
| `%c`          | char              | `"%c", 'A'`           | `"A"`           |
| `%b`          | boolean           | `"%b", true`          | `"true"`        |
| `%n`          | nova linha        | —                     | (quebra de linha)|
| `%%`          | literal %         | `"%.0f%%", 75.0`      | `"75%"`         |

```java
public class FormatacaoAvancada {
    public static void main(String[] args) {
        // Formatando moeda brasileira
        double valor = 1234567.89;
        System.out.printf("Valor:     R$ %,.2f%n", valor); // com separador de milhar

        // Formatando porcentagem
        double taxa = 0.0875;
        System.out.printf("Taxa:      %.2f%%%n", taxa * 100); // 8.75%

        // Alinhamento em relatório
        System.out.println("\n=== Relatório de Vendas ===");
        System.out.printf("%-20s %6s %12s %10s%n",
            "Vendedor", "Vendas", "Faturamento", "Comissão");
        System.out.println("=".repeat(52));

        Object[][] dados = {
            {"Ana Silva",       45, 32500.00, 0.05},
            {"Bruno Costa",     38, 28750.00, 0.05},
            {"Carla Mendes",    62, 47200.00, 0.07},
            {"Diego Oliveira",  29, 19800.00, 0.04},
        };

        double totalFaturamento = 0;

        for (Object[] linha : dados) {
            String vendedor = (String) linha[0];
            int vendas      = (Integer) linha[1];
            double fat      = (Double) linha[2];
            double comissao = (Double) linha[3];
            totalFaturamento += fat;

            System.out.printf("%-20s %6d %,12.2f %9.1f%%%n",
                vendedor, vendas, fat, comissao * 100);
        }

        System.out.println("-".repeat(52));
        System.out.printf("%-20s %6s %,12.2f%n",
            "TOTAL", "", totalFaturamento);
    }
}
```

---

## Exemplo Completo — Programa Interativo

Um programa completo que combina entrada com `Scanner` e saída formatada com `printf`.

```java
import java.util.Scanner;

public class CalculadoraIMC {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.println("╔══════════════════════════════╗");
        System.out.println("║   Calculadora de IMC (2026)  ║");
        System.out.println("╚══════════════════════════════╝");
        System.out.println();

        // Entrada de dados
        System.out.print("Nome: ");
        String nome = scanner.nextLine();

        System.out.print("Idade: ");
        int idade = scanner.nextInt();

        System.out.print("Altura em metros (ex: 1.75): ");
        double altura = scanner.nextDouble();

        System.out.print("Peso em kg (ex: 70.5): ");
        double peso = scanner.nextDouble();

        // Cálculos
        double imc = peso / (altura * altura);
        double pesoIdeal = 22.0 * (altura * altura); // IMC ideal ~22

        // Classificação da OMS
        String classificacao;
        String recomendacao;

        if (imc < 18.5) {
            classificacao = "Abaixo do peso";
            recomendacao  = "Consulte um nutricionista para ganho de peso saudável.";
        } else if (imc < 25.0) {
            classificacao = "Peso normal";
            recomendacao  = "Parabéns! Mantenha hábitos saudáveis.";
        } else if (imc < 30.0) {
            classificacao = "Sobrepeso";
            recomendacao  = "Considere ajustar a dieta e aumentar a atividade física.";
        } else if (imc < 35.0) {
            classificacao = "Obesidade grau I";
            recomendacao  = "Consulte um médico para um plano de emagrecimento.";
        } else if (imc < 40.0) {
            classificacao = "Obesidade grau II";
            recomendacao  = "Acompanhamento médico é fortemente recomendado.";
        } else {
            classificacao = "Obesidade grau III";
            recomendacao  = "Procure auxílio médico imediatamente.";
        }

        double diferencaPeso = peso - pesoIdeal;
        String statusPeso = (diferencaPeso > 0)
            ? String.format("%.1f kg acima do ideal", diferencaPeso)
            : String.format("%.1f kg abaixo do ideal", Math.abs(diferencaPeso));

        // Exibição dos resultados
        System.out.println();
        System.out.println("┌─────────────────────────────────────┐");
        System.out.printf( "│  Resultado para: %-19s│%n", nome);
        System.out.println("├─────────────────────────────────────┤");
        System.out.printf( "│  Idade:     %-25d│%n", idade);
        System.out.printf( "│  Altura:    %-22.2f m│%n", altura);
        System.out.printf( "│  Peso:      %-21.1f kg│%n", peso);
        System.out.println("├─────────────────────────────────────┤");
        System.out.printf( "│  IMC:       %-25.2f│%n", imc);
        System.out.printf( "│  Status:    %-25s│%n", classificacao);
        System.out.printf( "│  Peso ideal: ~%-21.1f kg│%n", pesoIdeal);
        System.out.printf( "│  Diferença: %-25s│%n", statusPeso);
        System.out.println("├─────────────────────────────────────┤");
        System.out.printf( "│  %-37s│%n", "Recomendação:");

        // Quebra a recomendação em linhas de 37 chars para caber na caixa
        String[] palavras = recomendacao.split(" ");
        StringBuilder linha = new StringBuilder("│  ");
        for (String palavra : palavras) {
            if (linha.length() + palavra.length() > 39) {
                System.out.printf("%-40s│%n", linha.toString());
                linha = new StringBuilder("│  ");
            }
            linha.append(palavra).append(" ");
        }
        System.out.printf("%-40s│%n", linha.toString());
        System.out.println("└─────────────────────────────────────┘");

        scanner.close();
    }
}
```

**Exemplo de execução:**

```
╔══════════════════════════════╗
║   Calculadora de IMC (2026)  ║
╚══════════════════════════════╝

Nome: Carlos Souza
Idade: 32
Altura em metros (ex: 1.75): 1.78
Peso em kg (ex: 70.5): 85.0

┌─────────────────────────────────────┐
│  Resultado para: Carlos Souza      │
├─────────────────────────────────────┤
│  Idade:     32                      │
│  Altura:    1.78 m                  │
│  Peso:      85.0 kg                 │
├─────────────────────────────────────┤
│  IMC:       26.82                   │
│  Status:    Sobrepeso               │
│  Peso ideal: ~69.8 kg               │
│  Diferença: 15.2 kg acima do ideal  │
├─────────────────────────────────────┤
│  Recomendação:                      │
│  Considere ajustar a dieta e        │
│  aumentar a atividade física.       │
└─────────────────────────────────────┘
```

---

## Dicas Importantes

- Use `System.out.println()` para depuração rápida e mensagens simples.
- Use `System.out.printf()` quando precisar de alinhamento ou controle preciso de casas decimais.
- Sempre chame `scanner.close()` ao terminar a leitura para liberar o recurso.
- Em aplicações reais, envolva a leitura em `try-catch` para tratar `InputMismatchException` (entrada de tipo errado).
- Para projetos maiores, considere usar `BufferedReader` + `InputStreamReader` para leitura mais performática.
