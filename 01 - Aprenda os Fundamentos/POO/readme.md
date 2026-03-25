# Objetos, Classes, Interfaces, Pacotes e Herança


Se você nunca usou uma linguagem de programação orientada a objetos antes, precisará aprender alguns conceitos básicos antes de começar a escrever qualquer código.

Esta seção o apresentará a objetos, classes, herança, interfaces e pacotes.

Cada discussão se concentra em como esses conceitos se relacionam com o mundo real
e, ao mesmo tempo, fornece uma introdução à sintaxe da linguagem de programação Java.

> **Vale saber desde o início:** Java não é POO pura. Tipos como `int`, `boolean` e `double` não são objetos — são primitivos sem métodos, sem classe, fora do modelo de objetos. O Java criou wrappers (`Integer`, `Boolean`, `Double`) para contornar isso, mas a tensão existe: em linguagens como Smalltalk e Ruby, o número `5` é um objeto que responde a mensagens. No Java, `5` é só um valor na memória. Isso não impede aprender OOP pelo Java — ele é excelente para isso — mas é bom saber que o modelo tem limites filosóficos. Você vai sentir esse atrito quando precisar converter `int` para `Integer` para usar em uma coleção genérica, ou quando encontrar uma classe cheia de métodos `static` que não pertence a nenhum objeto de verdade.

---

## Índice

1. [O que é um Objeto?](#o-que-é-um-objeto)
2. [O que é uma Classe?](#o-que-é-uma-classe)
3. [Herança](#herança)
4. [Interfaces](#interfaces)
5. [Classes Abstratas](#classes-abstratas)
6. [Os 4 Pilares da OOP](#os-4-pilares-da-oop)
   - [Encapsulamento](#1-encapsulamento)
   - [Herança](#2-herança-aprofundado)
   - [Polimorfismo](#3-polimorfismo)
   - [Abstração](#4-abstração)

---

# O que é um Objeto?

Um objeto é um pacote de software com estado e comportamento relacionados.

Esta seção explica como o estado e o comportamento são representados em um objeto,
apresenta o conceito de encapsulamento de dados e explica os benefícios de projetar o software dessa maneira.

Os objetos compartilham duas características: todos eles têm estado e comportamento.
Os cães têm estado (nome, cor, raça, fome) e comportamento (latir, buscar, abanar a cauda).
As bicicletas também têm estado (marcha atual, cadência atual do pedal, velocidade atual) e comportamento (mudança de marcha, mudança de cadência do pedal, aplicação de freios).
Identificar o estado e o comportamento de objetos do mundo real é uma ótima maneira de começar a pensar em termos de programação orientada a objetos.

Reserve um minuto agora mesmo para observar os objetos do mundo real que estão em sua área imediata.

Para cada objeto que você vir, faça a si mesmo duas perguntas: "Em que estados possíveis esse objeto pode estar?" e "Que comportamento possível esse objeto pode executar?".
Não se esqueça de anotar suas observações. Ao fazer isso, você perceberá que os objetos do mundo real variam em complexidade;
sua lâmpada de casa pode ter apenas três estados possíveis (ligado, desligado ou queimada) e dois comportamentos possíveis (ligar, desligar),
mas seu rádio pode ter estados adicionais (ligado, desligado, volume atual, estação atual) e comportamentos (ligar, desligar, aumentar o volume, diminuir o volume, buscar, procurar e sintonizar).
Você também pode notar que alguns objetos, por sua vez, também conterão outros objetos.
Essas observações do mundo real são um ponto de partida para entender o mundo da programação orientada a objetos.

Os objetos de software consistem em um estado e em um comportamento relacionado.
Um objeto armazena seu estado em campos (variáveis em algumas linguagens de programação) e expõe seu comportamento por meio de métodos (funções em algumas linguagens de programação).
Os métodos operam no estado interno de um objeto e servem como o principal mecanismo de comunicação entre objetos.

Ocultar o estado interno e exigir que toda a interação seja realizada por meio dos métodos de um objeto é conhecido como encapsulamento de dados — um princípio fundamental da programação orientada a objetos.

Considere uma bicicleta, por exemplo:

Ao atribuir um estado (velocidade atual, cadência atual do pedal e marcha atual) e fornecer métodos para alterar esse estado, o objeto permanece no controle de como o mundo externo pode usá-lo.

Por exemplo, se a bicicleta tiver apenas 6 marchas, um método para mudar as marchas poderá rejeitar qualquer valor menor que 1 ou maior que 6.

O agrupamento de código em objetos de software individuais oferece vários benefícios:

- **Modularidade:** O código-fonte de um objeto pode ser escrito e mantido independentemente de outros objetos.
- **Ocultação de informações:** Ao interagir apenas com os métodos de um objeto, os detalhes internos ficam ocultos do mundo externo.
- **Reutilização de código:** Um objeto já existente pode ser usado em outros programas sem reescrever sua lógica.
- **Facilidade de depuração:** Se um objeto se mostrar problemático, você o substitui sem precisar refazer a máquina inteira.

## Exemplo em Java

```java
public class Bicicleta {

    // Estado (campos)
    private int marcha;
    private int cadencia;
    private int velocidade;

    // Construtor
    public Bicicleta(int marcha, int cadencia, int velocidade) {
        this.marcha = marcha;
        this.cadencia = cadencia;
        this.velocidade = velocidade;
    }

    // Comportamento (métodos)
    public void mudarMarcha(int novaMarcha) {
        if (novaMarcha >= 1 && novaMarcha <= 6) {
            this.marcha = novaMarcha;
        }
    }

    public void acelerar(int incremento) {
        this.velocidade += incremento;
    }

    public void frear(int decremento) {
        this.velocidade = Math.max(0, this.velocidade - decremento);
    }

    public void exibirStatus() {
        System.out.println("Marcha: " + marcha + " | Cadência: " + cadencia + " | Velocidade: " + velocidade);
    }
}
```

```java
// Criando e usando objetos
public class Main {
    public static void main(String[] args) {
        Bicicleta bike1 = new Bicicleta(1, 50, 0);
        Bicicleta bike2 = new Bicicleta(3, 70, 0);

        bike1.acelerar(20);
        bike2.acelerar(35);

        bike1.exibirStatus(); // Marcha: 1 | Cadência: 50 | Velocidade: 20
        bike2.exibirStatus(); // Marcha: 3 | Cadência: 70 | Velocidade: 35
    }
}
```

> Cada objeto tem seu próprio estado independente — a velocidade de `bike1` não afeta `bike2`.

---

# O que é uma Classe?

No mundo real, você frequentemente encontra vários objetos individuais do mesmo tipo. Existem milhares de bicicletas no mundo, todas com o mesmo modelo e mecânica. Cada bicicleta foi construída a partir do mesmo conjunto de planos e, portanto, contém os mesmos componentes.

Em termos de programação orientada a objetos, dizemos que **sua bicicleta é uma instância da classe de objetos conhecida como bicicleta**. Uma classe é o projeto (blueprint) a partir do qual os objetos individuais são criados.

Pense na classe como uma **forma de bolo**: a forma define como o bolo será, mas não é o bolo em si. Cada bolo assado com essa forma é um objeto (instância) diferente.

## Anatomia de uma Classe

Uma classe em Java é composta por:

- **Campos (atributos):** representam o estado do objeto
- **Construtores:** responsáveis por inicializar o objeto
- **Métodos:** representam os comportamentos do objeto

## Exemplo em Java

```java
public class Bicicleta {

    // Campos — definem o estado de cada bicicleta
    private int marcha;
    private int cadencia;
    private int velocidade;
    private String cor;

    // Construtor — executado ao criar uma nova instância
    public Bicicleta(int marcha, int cadencia, int velocidade, String cor) {
        this.marcha = marcha;
        this.cadencia = cadencia;
        this.velocidade = velocidade;
        this.cor = cor;
    }

    // Métodos — definem os comportamentos da bicicleta
    public void mudarMarcha(int novaMarcha) {
        if (novaMarcha >= 1 && novaMarcha <= 6) {
            this.marcha = novaMarcha;
        }
    }

    public void acelerar(int incremento) {
        this.velocidade += incremento;
    }

    public void frear(int decremento) {
        this.velocidade = Math.max(0, this.velocidade - decremento);
    }

    // Getters — permitem acesso controlado aos campos privados
    public int getMarcha()     { return marcha; }
    public int getCadencia()   { return cadencia; }
    public int getVelocidade() { return velocidade; }
    public String getCor()     { return cor; }

    public void exibirStatus() {
        System.out.println("Cor: " + cor + " | Marcha: " + marcha +
                           " | Cadência: " + cadencia + " | Velocidade: " + velocidade);
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        // Cada "new Bicicleta(...)" cria uma instância diferente da mesma classe
        Bicicleta bicicleta1 = new Bicicleta(1, 50, 0, "Vermelha");
        Bicicleta bicicleta2 = new Bicicleta(3, 70, 0, "Azul");
        Bicicleta bicicleta3 = new Bicicleta(2, 60, 0, "Preta");

        bicicleta1.acelerar(20);
        bicicleta2.acelerar(35);
        bicicleta3.acelerar(15);

        bicicleta1.exibirStatus(); // Cor: Vermelha | Marcha: 1 | Cadência: 50 | Velocidade: 20
        bicicleta2.exibirStatus(); // Cor: Azul | Marcha: 3 | Cadência: 70 | Velocidade: 35
        bicicleta3.exibirStatus(); // Cor: Preta | Marcha: 2 | Cadência: 60 | Velocidade: 15
    }
}
```

> A classe `Bicicleta` é o projeto. `bicicleta1`, `bicicleta2` e `bicicleta3` são três objetos distintos criados a partir desse projeto — cada um com seu próprio estado independente.

---

# Herança

Diferentes tipos de objetos frequentemente têm uma certa quantidade de coisas em comum entre si. Bicicletas de montanha, bicicletas de estrada e bicicletas tandem, por exemplo, todas compartilham as características de bicicletas (velocidade atual, cadência atual do pedal, marcha atual). Porém, cada uma também define características adicionais que as tornam diferentes: as bicicletas tandem têm dois assentos e dois conjuntos de guidões; as bicicletas de estrada têm guidões drop-bar; algumas bicicletas de montanha têm um anel a mais, dando a elas uma proporção de engrenagem mais baixa.

A programação orientada a objetos permite que as classes **herdem** estado e comportamento comumente usados de outras classes. Neste exemplo, `BicicletaDeMontanha`, `BicicletaDeEstrada` e `BicicletaTandem` se tornam subclasses de `Bicicleta`.

## Como funciona a Herança em Java

Em Java, usamos a palavra-chave `extends` para indicar que uma classe herda de outra.

- A classe que é herdada é chamada de **superclasse** (ou classe pai)
- A classe que herda é chamada de **subclasse** (ou classe filha)
- A subclasse herda todos os campos e métodos **não privados** da superclasse

## Exemplo em Java

```java
// Superclasse (classe pai)
public class Bicicleta {

    protected int marcha;
    protected int cadencia;
    protected int velocidade;

    public Bicicleta(int marcha, int cadencia, int velocidade) {
        this.marcha = marcha;
        this.cadencia = cadencia;
        this.velocidade = velocidade;
    }

    public void mudarMarcha(int novaMarcha) {
        if (novaMarcha >= 1 && novaMarcha <= 6) {
            this.marcha = novaMarcha;
        }
    }

    public void acelerar(int incremento) {
        this.velocidade += incremento;
    }

    public void frear(int decremento) {
        this.velocidade = Math.max(0, this.velocidade - decremento);
    }

    public void exibirStatus() {
        System.out.println("Marcha: " + marcha + " | Cadência: " + cadencia + " | Velocidade: " + velocidade);
    }
}
```

```java
// Subclasse — herda tudo de Bicicleta e adiciona características próprias
public class BicicletaDeMontanha extends Bicicleta {

    private int alturaDoSela; // característica exclusiva da bicicleta de montanha

    public BicicletaDeMontanha(int marcha, int cadencia, int velocidade, int alturaDoSela) {
        super(marcha, cadencia, velocidade); // chama o construtor da superclasse
        this.alturaDoSela = alturaDoSela;
    }

    public void ajustarSela(int novaAltura) {
        this.alturaDoSela = novaAltura;
    }

    @Override
    public void exibirStatus() {
        super.exibirStatus(); // reutiliza o comportamento da superclasse
        System.out.println("Altura do Selim: " + alturaDoSela + " cm");
    }
}
```

```java
public class BicicletaTandem extends Bicicleta {

    private int numeroDePedais; // sempre 2 para um tandem

    public BicicletaTandem(int marcha, int cadencia, int velocidade) {
        super(marcha, cadencia, velocidade);
        this.numeroDePedais = 2;
    }

    public int getNumeroDePedais() {
        return numeroDePedais;
    }

    @Override
    public void exibirStatus() {
        super.exibirStatus();
        System.out.println("Pedais: " + numeroDePedais + " (Tandem)");
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        BicicletaDeMontanha mtb = new BicicletaDeMontanha(2, 60, 0, 75);
        BicicletaTandem tandem = new BicicletaTandem(1, 50, 0);

        mtb.acelerar(25);       // método herdado de Bicicleta
        mtb.ajustarSela(80);    // método próprio de BicicletaDeMontanha
        mtb.exibirStatus();
        // Marcha: 2 | Cadência: 60 | Velocidade: 25
        // Altura do Selim: 80 cm

        tandem.acelerar(10);
        tandem.exibirStatus();
        // Marcha: 1 | Cadência: 50 | Velocidade: 10
        // Pedais: 2 (Tandem)
    }
}
```

> A subclasse herda todos os comportamentos da superclasse e pode adicionar novos ou modificar os existentes com `@Override`.

---

# Interfaces

Uma interface é um contrato que define **o que** uma classe deve fazer, sem especificar **como** ela deve fazer. Em Java, uma interface é um tipo de referência, semelhante a uma classe, que pode conter apenas constantes, assinaturas de métodos, métodos default, métodos estáticos e tipos aninhados.

Pense em uma interface como um **manual de especificações**: qualquer bicicleta que queira ser "certificada como segura" deve ter freios, luzes e buzina — a interface exige esses itens, mas cada fabricante implementa do seu jeito.

## Como funciona em Java

- Usamos a palavra-chave `interface` para definir uma interface
- Uma classe implementa uma interface usando `implements`
- Uma classe pode implementar **múltiplas interfaces**
- Todos os métodos de uma interface são implicitamente `public abstract`

## Exemplo em Java

```java
// Interface define o contrato: toda bicicleta "segura" deve ter esses comportamentos
public interface Seguranca {
    void acionarFreioDeEmergencia();
    void ligarLuzes();
    boolean verificarFreios();
}
```

```java
// Outra interface
public interface Conectavel {
    void conectarAoAplicativo(String nomeApp);
    int obterVelocidadeGPS();
}
```

```java
// A classe implementa múltiplas interfaces
public class BicicletaEletrica extends Bicicleta implements Seguranca, Conectavel {

    private int nivelDaBateria;
    private boolean luzesLigadas;
    private String appConectado;

    public BicicletaEletrica(int marcha, int cadencia, int velocidade, int nivelDaBateria) {
        super(marcha, cadencia, velocidade);
        this.nivelDaBateria = nivelDaBateria;
        this.luzesLigadas = false;
    }

    // Implementação dos métodos da interface Seguranca
    @Override
    public void acionarFreioDeEmergencia() {
        this.velocidade = 0;
        System.out.println("Freio de emergência acionado! Bicicleta parada.");
    }

    @Override
    public void ligarLuzes() {
        this.luzesLigadas = true;
        System.out.println("Luzes ativadas.");
    }

    @Override
    public boolean verificarFreios() {
        System.out.println("Verificando sistema de freios...");
        return true; // freios em perfeito estado
    }

    // Implementação dos métodos da interface Conectavel
    @Override
    public void conectarAoAplicativo(String nomeApp) {
        this.appConectado = nomeApp;
        System.out.println("Conectada ao app: " + nomeApp);
    }

    @Override
    public int obterVelocidadeGPS() {
        return this.velocidade; // retorna a velocidade atual via GPS
    }

    @Override
    public void exibirStatus() {
        super.exibirStatus();
        System.out.println("Bateria: " + nivelDaBateria + "% | Luzes: " + (luzesLigadas ? "ON" : "OFF"));
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        BicicletaEletrica eBike = new BicicletaEletrica(3, 80, 0, 90);

        eBike.ligarLuzes();
        eBike.conectarAoAplicativo("Strava");
        eBike.acelerar(40);

        System.out.println("Velocidade GPS: " + eBike.obterVelocidadeGPS() + " km/h");

        eBike.acionarFreioDeEmergencia();
        eBike.exibirStatus();
    }
}
```

> Uma interface garante que qualquer classe que a implemente terá os comportamentos esperados — isso é especialmente útil quando trabalhamos com polimorfismo.

---

# Classes Abstratas

Uma classe abstrata é um meio-termo entre uma classe concreta e uma interface. Ela pode ter:

- Métodos com implementação (concretos)
- Métodos sem implementação (abstratos), que as subclasses são obrigadas a implementar
- Campos e construtores normais

**Você não pode instanciar uma classe abstrata diretamente** — ela serve como base para outras classes.

Pense assim: uma "Bicicleta genérica" é um conceito abstrato. Você nunca compra uma "bicicleta genérica" — você compra uma bicicleta de montanha, uma bicicleta de estrada ou uma bicicleta elétrica. A classe abstrata representa esse conceito genérico.

## Diferença entre Classe Abstrata e Interface

| Característica          | Classe Abstrata                         | Interface                              |
|-------------------------|-----------------------------------------|----------------------------------------|
| Instanciável?           | Não                                     | Não                                    |
| Herança múltipla?       | Não (apenas uma superclasse)            | Sim (uma classe pode implementar várias)|
| Pode ter campos?        | Sim (inclusive privados)                | Apenas constantes (public static final)|
| Pode ter construtor?    | Sim                                     | Não                                    |
| Métodos com corpo?      | Sim                                     | Apenas métodos `default` ou `static`   |

## Exemplo em Java

```java
// Classe abstrata — define o "esqueleto" de uma bicicleta
public abstract class BicicletaAbstrata {

    protected int marcha;
    protected int cadencia;
    protected int velocidade;

    public BicicletaAbstrata(int marcha, int cadencia, int velocidade) {
        this.marcha = marcha;
        this.cadencia = cadencia;
        this.velocidade = velocidade;
    }

    // Métodos concretos — comportamento padrão herdado por todas as subclasses
    public void acelerar(int incremento) {
        this.velocidade += incremento;
    }

    public void frear(int decremento) {
        this.velocidade = Math.max(0, this.velocidade - decremento);
    }

    // Método abstrato — cada subclasse DEVE implementar do seu jeito
    public abstract void mudarMarcha(int novaMarcha);

    // Método abstrato — cada tipo de bicicleta exibe seu status de forma diferente
    public abstract void exibirStatus();
}
```

```java
// Subclasse concreta — obrigada a implementar os métodos abstratos
public class BicicletaDeEstrada extends BicicletaAbstrata {

    private int numeroDeVelocidades; // bicicletas de estrada têm mais velocidades

    public BicicletaDeEstrada(int marcha, int cadencia, int velocidade, int numeroDeVelocidades) {
        super(marcha, cadencia, velocidade);
        this.numeroDeVelocidades = numeroDeVelocidades;
    }

    @Override
    public void mudarMarcha(int novaMarcha) {
        // Bicicleta de estrada pode ter até 21 velocidades
        if (novaMarcha >= 1 && novaMarcha <= numeroDeVelocidades) {
            this.marcha = novaMarcha;
            System.out.println("Marcha alterada para " + novaMarcha + " (de " + numeroDeVelocidades + ")");
        } else {
            System.out.println("Marcha inválida! Range: 1 a " + numeroDeVelocidades);
        }
    }

    @Override
    public void exibirStatus() {
        System.out.println("[Bicicleta de Estrada] Marcha: " + marcha + "/" + numeroDeVelocidades +
                           " | Cadência: " + cadencia + " | Velocidade: " + velocidade + " km/h");
    }
}
```

```java
// Outra subclasse concreta
public class BicicletaInfantil extends BicicletaAbstrata {

    private boolean temRodinhas;

    public BicicletaInfantil(int marcha, int cadencia, int velocidade, boolean temRodinhas) {
        super(marcha, cadencia, velocidade);
        this.temRodinhas = temRodinhas;
    }

    @Override
    public void mudarMarcha(int novaMarcha) {
        // Bicicleta infantil tem apenas 3 marchas
        if (novaMarcha >= 1 && novaMarcha <= 3) {
            this.marcha = novaMarcha;
        }
    }

    @Override
    public void exibirStatus() {
        System.out.println("[Bicicleta Infantil] Marcha: " + marcha +
                           " | Velocidade: " + velocidade +
                           " | Rodinhas: " + (temRodinhas ? "Sim" : "Não"));
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        // BicicletaAbstrata bike = new BicicletaAbstrata(...); // ERRO! Não pode instanciar

        BicicletaDeEstrada estrada = new BicicletaDeEstrada(1, 90, 0, 21);
        BicicletaInfantil infantil = new BicicletaInfantil(1, 30, 0, true);

        estrada.acelerar(50);
        estrada.mudarMarcha(15);
        estrada.exibirStatus();
        // [Bicicleta de Estrada] Marcha: 15/21 | Cadência: 90 | Velocidade: 50 km/h

        infantil.acelerar(8);
        infantil.mudarMarcha(2);
        infantil.exibirStatus();
        // [Bicicleta Infantil] Marcha: 2 | Velocidade: 8 | Rodinhas: Sim
    }
}
```

> Classes abstratas são ideais quando você quer compartilhar código entre subclasses relacionadas, mas também quer forçar que cada uma implemente certos comportamentos de forma própria.

---

# Os 4 Pilares da OOP

A Programação Orientada a Objetos é sustentada por quatro princípios fundamentais. Vamos explorar cada um com exemplos baseados na nossa `Bicicleta`.

---

## 1. Encapsulamento

O **encapsulamento** é o princípio de **ocultar o estado interno** de um objeto e exigir que toda interação aconteça por meio dos seus métodos públicos.

É como o motor de uma bicicleta elétrica: você não precisa saber como a bateria é gerenciada internamente — você apenas aperta o botão de acelerar. Os detalhes ficam encapsulados dentro do objeto.

### Por que usar?

- Protege o estado interno contra modificações inválidas
- Reduz o acoplamento entre partes do sistema
- Facilita a manutenção: você pode mudar a implementação interna sem afetar quem usa a classe

### Modificadores de Acesso em Java

| Modificador | Mesma Classe | Mesmo Pacote | Subclasses | Qualquer lugar |
|-------------|:------------:|:------------:|:----------:|:--------------:|
| `private`   | Sim          | Não          | Não        | Não            |
| (padrão)    | Sim          | Sim          | Não        | Não            |
| `protected` | Sim          | Sim          | Sim        | Não            |
| `public`    | Sim          | Sim          | Sim        | Sim            |

### Exemplo em Java

```java
public class Bicicleta {

    // Campos privados — o mundo externo não acessa diretamente
    private int marcha;
    private int cadencia;
    private int velocidade;
    private int nivelDoCombustivel;

    public Bicicleta(int marcha, int cadencia, int velocidade) {
        this.marcha = marcha;
        this.cadencia = cadencia;
        this.velocidade = velocidade;
        this.nivelDoCombustivel = 100;
    }

    // Getter — permite leitura do estado sem expor o campo diretamente
    public int getMarcha()     { return marcha; }
    public int getCadencia()   { return cadencia; }
    public int getVelocidade() { return velocidade; }

    // Setter com validação — garante que o estado só mude de forma válida
    public void setMarcha(int novaMarcha) {
        if (novaMarcha >= 1 && novaMarcha <= 6) {
            this.marcha = novaMarcha;
        } else {
            System.out.println("Marcha inválida! Deve ser entre 1 e 6.");
        }
    }

    public void acelerar(int incremento) {
        if (incremento > 0) {
            this.velocidade += incremento;
        }
    }

    public void frear(int decremento) {
        this.velocidade = Math.max(0, this.velocidade - decremento);
    }

    public void exibirStatus() {
        System.out.println("Marcha: " + marcha + " | Cadência: " + cadencia +
                           " | Velocidade: " + velocidade + " | Combustível: " + nivelDoCombustivel + "%");
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        Bicicleta bike = new Bicicleta(1, 50, 0);

        // bike.marcha = 10; // ERRO de compilação! Campo privado não é acessível

        bike.setMarcha(10); // "Marcha inválida! Deve ser entre 1 e 6."
        bike.setMarcha(4);  // OK — validação passou
        bike.acelerar(30);

        System.out.println("Velocidade atual: " + bike.getVelocidade()); // 30
        bike.exibirStatus();
    }
}
```

> O encapsulamento garante que ninguém possa colocar a bicicleta em uma "marcha 10" que não existe — a lógica de validação fica dentro do objeto.

---

## 2. Herança (Aprofundado)

A **herança** permite que uma classe (subclasse) herde atributos e métodos de outra classe (superclasse), promovendo o reuso de código e estabelecendo uma hierarquia "é um tipo de".

Uma `BicicletaDeMontanha` **é um tipo de** `Bicicleta`. Uma `BicicletaEletrica` **é um tipo de** `Bicicleta`. Essa relação "é-um" é o coração da herança.

### Cadeia de Herança

Em Java, a herança pode ser encadeada — uma subclasse pode ser superclasse de outra:

```
Bicicleta
    └── BicicletaEletrica
            └── BicicletaEletricaDeMontanha
```

### Exemplo em Java

```java
// Nível 1: Superclasse base
public class Bicicleta {

    protected int marcha;
    protected int cadencia;
    protected int velocidade;

    public Bicicleta(int marcha, int cadencia, int velocidade) {
        this.marcha = marcha;
        this.cadencia = cadencia;
        this.velocidade = velocidade;
    }

    public void acelerar(int incremento) {
        this.velocidade += incremento;
    }

    public void frear(int decremento) {
        this.velocidade = Math.max(0, this.velocidade - decremento);
    }

    public void exibirStatus() {
        System.out.println("[Bicicleta] Marcha: " + marcha + " | Velocidade: " + velocidade);
    }
}
```

```java
// Nível 2: Subclasse de Bicicleta
public class BicicletaEletrica extends Bicicleta {

    protected int nivelDaBateria;
    protected int potenciaDoMotor; // watts

    public BicicletaEletrica(int marcha, int cadencia, int velocidade, int nivelDaBateria, int potenciaDoMotor) {
        super(marcha, cadencia, velocidade);
        this.nivelDaBateria = nivelDaBateria;
        this.potenciaDoMotor = potenciaDoMotor;
    }

    public void ativarAssistencia() {
        if (nivelDaBateria > 0) {
            this.velocidade += potenciaDoMotor / 100;
            this.nivelDaBateria -= 5;
            System.out.println("Assistência elétrica ativada! +" + (potenciaDoMotor / 100) + " km/h");
        } else {
            System.out.println("Bateria descarregada!");
        }
    }

    @Override
    public void exibirStatus() {
        super.exibirStatus(); // reutiliza o exibirStatus da Bicicleta
        System.out.println("[Elétrica] Bateria: " + nivelDaBateria + "% | Motor: " + potenciaDoMotor + "W");
    }
}
```

```java
// Nível 3: Subclasse de BicicletaEletrica
public class BicicletaEletricaDeMontanha extends BicicletaEletrica {

    private int suspensao; // milímetros de curso de suspensão
    private boolean modoOffRoad;

    public BicicletaEletricaDeMontanha(int marcha, int cadencia, int velocidade,
                                        int nivelDaBateria, int potenciaDoMotor, int suspensao) {
        super(marcha, cadencia, velocidade, nivelDaBateria, potenciaDoMotor);
        this.suspensao = suspensao;
        this.modoOffRoad = false;
    }

    public void ativarModoOffRoad() {
        this.modoOffRoad = true;
        System.out.println("Modo Off-Road ativado! Suspensão ajustada para terrenos difíceis.");
    }

    @Override
    public void exibirStatus() {
        super.exibirStatus(); // chama BicicletaEletrica.exibirStatus(), que chama Bicicleta.exibirStatus()
        System.out.println("[MTB Elétrica] Suspensão: " + suspensao + "mm | Off-Road: " + (modoOffRoad ? "ON" : "OFF"));
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        BicicletaEletricaDeMontanha emtb = new BicicletaEletricaDeMontanha(2, 70, 0, 95, 500, 120);

        emtb.acelerar(20);          // herdado de Bicicleta
        emtb.ativarAssistencia();   // herdado de BicicletaEletrica
        emtb.ativarModoOffRoad();   // próprio de BicicletaEletricaDeMontanha
        emtb.exibirStatus();
    }
}
```

### A palavra-chave `super`

- `super(...)` — chama o construtor da superclasse
- `super.metodo()` — chama o método da superclasse, mesmo que tenha sido sobrescrito

> A herança cria uma hierarquia de especialização: quanto mais baixo na árvore, mais específico o tipo.

---

## 3. Polimorfismo

**Polimorfismo** significa "muitas formas". Em OOP, é a capacidade de um objeto ser referenciado pelo tipo de sua superclasse, mas se comportar de acordo com sua implementação real (tipo concreto).

Em outras palavras: você pode tratar uma `BicicletaDeMontanha` e uma `BicicletaDeEstrada` como se fossem simplesmente `Bicicleta`, mas cada uma reagirá de forma diferente quando você chamar o mesmo método.

### Tipos de Polimorfismo em Java

- **Polimorfismo de sobrescrita (Override):** A subclasse redefine um método da superclasse
- **Polimorfismo de sobrecarga (Overload):** A mesma classe tem vários métodos com o mesmo nome, mas parâmetros diferentes

### Exemplo — Polimorfismo com Override

```java
public class Bicicleta {
    protected int velocidade;

    public Bicicleta(int velocidade) {
        this.velocidade = velocidade;
    }

    public void exibirStatus() {
        System.out.println("[Bicicleta Genérica] Velocidade: " + velocidade);
    }

    public void acelerar(int incremento) {
        this.velocidade += incremento;
    }
}
```

```java
public class BicicletaDeMontanha extends Bicicleta {
    private int alturaDoSela;

    public BicicletaDeMontanha(int velocidade, int alturaDoSela) {
        super(velocidade);
        this.alturaDoSela = alturaDoSela;
    }

    @Override
    public void exibirStatus() {
        System.out.println("[MTB] Velocidade: " + velocidade + " | Selim: " + alturaDoSela + "cm");
    }
}
```

```java
public class BicicletaDeEstrada extends Bicicleta {
    private int numeroDeMarchas;

    public BicicletaDeEstrada(int velocidade, int numeroDeMarchas) {
        super(velocidade);
        this.numeroDeMarchas = numeroDeMarchas;
    }

    @Override
    public void exibirStatus() {
        System.out.println("[Estrada] Velocidade: " + velocidade + " | Marchas: " + numeroDeMarchas);
    }
}
```

```java
public class BicicletaEletrica extends Bicicleta {
    private int bateria;

    public BicicletaEletrica(int velocidade, int bateria) {
        super(velocidade);
        this.bateria = bateria;
    }

    @Override
    public void exibirStatus() {
        System.out.println("[Elétrica] Velocidade: " + velocidade + " | Bateria: " + bateria + "%");
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        // Array do tipo Bicicleta, mas cada elemento é um tipo diferente
        Bicicleta[] frota = {
            new BicicletaDeMontanha(0, 75),
            new BicicletaDeEstrada(0, 21),
            new BicicletaEletrica(0, 90)
        };

        // Polimorfismo em ação: o mesmo código chama implementações diferentes
        for (Bicicleta bike : frota) {
            bike.acelerar(30);
            bike.exibirStatus(); // cada objeto executa SUA versão do método
        }
    }
}
// Saída:
// [MTB] Velocidade: 30 | Selim: 75cm
// [Estrada] Velocidade: 30 | Marchas: 21
// [Elétrica] Velocidade: 30 | Bateria: 90%
```

### Exemplo — Polimorfismo com Sobrecarga (Overload)

```java
public class Bicicleta {
    private int velocidade;

    // Sobrecarga: mesmo nome, parâmetros diferentes
    public void acelerar() {
        this.velocidade += 10; // incremento padrão
        System.out.println("Acelerou +10 (padrão). Velocidade: " + velocidade);
    }

    public void acelerar(int incremento) {
        this.velocidade += incremento;
        System.out.println("Acelerou +" + incremento + ". Velocidade: " + velocidade);
    }

    public void acelerar(int incremento, String modo) {
        if (modo.equals("turbo")) {
            this.velocidade += incremento * 2;
            System.out.println("Modo TURBO! Acelerou +" + (incremento * 2) + ". Velocidade: " + velocidade);
        } else {
            this.velocidade += incremento;
            System.out.println("Acelerou +" + incremento + ". Velocidade: " + velocidade);
        }
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        Bicicleta bike = new Bicicleta();

        bike.acelerar();              // chama o método sem parâmetros
        bike.acelerar(20);            // chama o método com int
        bike.acelerar(15, "turbo");   // chama o método com int e String
    }
}
// Saída:
// Acelerou +10 (padrão). Velocidade: 10
// Acelerou +20. Velocidade: 30
// Modo TURBO! Acelerou +30. Velocidade: 60
```

> O polimorfismo é o que permite escrever código genérico que funciona com qualquer tipo da hierarquia — o Java descobre em tempo de execução qual implementação chamar.

---

## 4. Abstração

A **abstração** é o princípio de expor apenas **o que é essencial** e ocultar os detalhes desnecessários. O objetivo é reduzir a complexidade e permitir que o programador trabalhe em um nível mais alto de pensamento.

Quando você pedala uma bicicleta, você não precisa entender como o sistema de transmissão transforma o movimento dos pedais em rotação da roda. Você simplesmente pedala. Esse é o conceito de abstração: você interage com uma interface simplificada e os detalhes complexos ficam escondidos.

Em Java, a abstração é alcançada principalmente por:

- **Classes abstratas** — definem comportamentos obrigatórios sem implementá-los
- **Interfaces** — definem contratos que as classes devem cumprir

### Exemplo em Java

```java
// Interface abstrai "o que" uma bicicleta deve fazer
public interface Veiculo {
    void mover();
    void parar();
    String obterDescricao();
}
```

```java
// Classe abstrata abstrai comportamentos comuns com algumas implementações
public abstract class BicicletaBase implements Veiculo {

    protected int velocidade;

    public BicicletaBase() {
        this.velocidade = 0;
    }

    // Implementação concreta — compartilhada por todas as subclasses
    @Override
    public void parar() {
        this.velocidade = 0;
        System.out.println("Bicicleta parada.");
    }

    // Método abstrato — cada tipo define como "mover"
    @Override
    public abstract void mover();

    // Método abstrato — cada tipo descreve a si mesmo
    @Override
    public abstract String obterDescricao();

    // Método concreto com lógica interna abstrata para o usuário
    public void iniciarPercurso(int distanciaKm) {
        System.out.println("Iniciando percurso de " + distanciaKm + "km...");
        mover(); // chama a implementação específica de cada subclasse
        System.out.println("Percurso concluído com: " + obterDescricao());
        parar();
    }
}
```

```java
public class BicicletaDeMontanha extends BicicletaBase {

    @Override
    public void mover() {
        this.velocidade = 25;
        System.out.println("MTB em movimento — modo trilha ativado. Velocidade: " + velocidade + " km/h");
    }

    @Override
    public String obterDescricao() {
        return "Bicicleta de Montanha | Velocidade: " + velocidade + " km/h";
    }
}
```

```java
public class BicicletaEletrica extends BicicletaBase {

    private int potencia;

    public BicicletaEletrica(int potencia) {
        this.potencia = potencia;
    }

    @Override
    public void mover() {
        this.velocidade = 40;
        System.out.println("E-Bike em movimento — motor de " + potencia + "W ativado. Velocidade: " + velocidade + " km/h");
    }

    @Override
    public String obterDescricao() {
        return "Bicicleta Elétrica " + potencia + "W | Velocidade: " + velocidade + " km/h";
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        // Trabalhamos com o tipo abstrato — não nos importa o detalhe interno
        BicicletaBase mtb   = new BicicletaDeMontanha();
        BicicletaBase eBike = new BicicletaEletrica(500);

        System.out.println("=== Percurso 1 ===");
        mtb.iniciarPercurso(10);

        System.out.println();

        System.out.println("=== Percurso 2 ===");
        eBike.iniciarPercurso(25);
    }
}
// Saída:
// === Percurso 1 ===
// Iniciando percurso de 10km...
// MTB em movimento — modo trilha ativado. Velocidade: 25 km/h
// Percurso concluído com: Bicicleta de Montanha | Velocidade: 25 km/h
// Bicicleta parada.
//
// === Percurso 2 ===
// Iniciando percurso de 25km...
// E-Bike em movimento — motor de 500W ativado. Velocidade: 40 km/h
// Percurso concluído com: Bicicleta Elétrica 500W | Velocidade: 40 km/h
// Bicicleta parada.
```

> A abstração permite que `iniciarPercurso()` funcione para qualquer tipo de bicicleta sem precisar saber os detalhes de como cada uma se move. O código que usa a abstração é mais simples, mais legível e mais fácil de manter.

---

## Resumo dos 4 Pilares

| Pilar            | O que faz                                                             | Como aplicar em Java                           |
|------------------|-----------------------------------------------------------------------|------------------------------------------------|
| **Encapsulamento** | Oculta o estado interno e protege com validações                    | `private` + getters/setters                    |
| **Herança**        | Reutiliza código e cria hierarquias "é-um"                          | `extends`                                      |
| **Polimorfismo**   | O mesmo código funciona com tipos diferentes                        | `@Override`, sobrecarga de métodos             |
| **Abstração**      | Expõe só o essencial, esconde a complexidade                        | `abstract class`, `interface`                  |

---

> Dominar esses quatro pilares é o que separa quem escreve código que funciona de quem escreve código que é profissional, manutenível e escalável.
