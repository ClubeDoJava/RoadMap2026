# Criando variáveis e nomeando-as


## Variáveis

Como você aprendeu na seção anterior, um objeto armazena seu estado em campos.

Vamos supor que temos um objeto Bicicleta:

```java
int cadencia = 0;
int velocidade = 0;
int marcha = 1; // gear em inglês
```

A seção "O que é um objeto?" apresentou os campos, mas você provavelmente ainda tem algumas perguntas, como:

- Quais são as regras e convenções para nomear um campo?
- Além de `int`, que outros tipos de dados existem?
- Os campos precisam ser inicializados quando são declarados?
- É atribuído um valor padrão aos campos se eles não forem inicializados explicitamente?

Exploraremos as respostas a essas perguntas nesta seção, mas, antes disso, há algumas distinções técnicas que você deve conhecer primeiro.

Na linguagem de programação Java, os termos "campo" e "variável" podem ser usados; essa é um ponto comum de confusão entre os novos desenvolvedores, pois ambos parecem se referir à mesma coisa.

A linguagem de programação Java define os seguintes tipos de variáveis:


## Variáveis de instância (campos não estáticos)

Tecnicamente falando, os objetos armazenam seus estados individuais em "campos não estáticos", ou seja, campos declarados sem a palavra-chave `static`.

Os campos não estáticos também são conhecidos como **variáveis de instância** porque seus valores são exclusivos para cada instância de uma classe (para cada objeto, em outras palavras).

```java
// Cada objeto Bicicleta terá sua própria velocidade
public class Bicicleta {
    int velocidade = 0; // variável de instância
    int cadencia = 0;   // variável de instância
}
```

Por exemplo, a velocidade atual de uma bicicleta é independente da velocidade de outra.


## Variáveis de classe (campos estáticos)

Uma variável de classe é qualquer campo declarado com o modificador `static`. Isso informa ao compilador que existe exatamente uma cópia dessa variável, independentemente de quantas vezes a classe tenha sido instanciada.

```java
public class Bicicleta {
    static int numMarchas = 6; // variável de classe — compartilhada por todos os objetos
    // final indica que o valor nunca mudará
    static final int MAX_VELOCIDADE = 120;
}
```

Um campo que define o número de marchas de um determinado tipo de bicicleta pode ser marcado como `static`, pois conceitualmente o mesmo número de marchas será aplicado a todas as instâncias.


## Variáveis locais

Da mesma forma que um objeto armazena seu estado em campos, um método geralmente armazena seu estado temporário em variáveis locais.

```java
public void calcularMedia() {
    int count = 0; // variável local — existe só dentro deste método
    double soma = 0.0;
    // ...
}
```

Não há nenhuma palavra-chave especial que designe uma variável como local. Essa determinação vem inteiramente do local em que a variável é declarada — entre as chaves de um método. Variáveis locais são visíveis apenas para o método em que são declaradas.


## Parâmetros

```java
public static void main(String[] args) {
    // 'args' é um parâmetro — sempre classificado como variável, não como campo
}

public void mudarMarcha(int novaMarcha) {
    // 'novaMarcha' é um parâmetro
}
```

O importante é lembrar que os parâmetros são sempre classificados como "variáveis" e não como "campos". Isso também se aplica a construtores e manipuladores de exceções.


---


# Nomeação de variáveis

As regras e convenções para nomear variáveis em Java:

- Os nomes diferenciam maiúsculas de minúsculas (`velocidade` e `Velocidade` são variáveis diferentes)
- Podem conter letras, dígitos, `$` ou `_`, mas **devem começar com uma letra**
- Por convenção, nunca use `$` ou `_` no início
- Não podem ser palavras-chave (`int`, `class`, `static`, etc.)
- Sem espaços em branco

```java
// Convenções de nomenclatura

// Uma palavra → tudo minúsculo
int velocidade;
int marcha;

// Mais de uma palavra → camelCase
int velocidadeAtual;
int numeroDeMarchas;

// Constantes → UPPER_SNAKE_CASE
static final int NUM_MARCHAS = 6;
static final double VELOCIDADE_MAXIMA = 120.0;
```

Prefira nomes completos e descritivos em vez de abreviações. `cadencia`, `velocidade` e `marcha` são muito mais intuitivos do que `c`, `v` e `m`.
