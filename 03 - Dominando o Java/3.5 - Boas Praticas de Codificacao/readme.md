# 3.5 — Boas Práticas de Codificação

Escrever código que funciona é o mínimo. Boas práticas garantem que o código seja **legível, manutenível, extensível e correto** — e que outros desenvolvedores (incluindo você mesmo, no futuro) consigam trabalhar nele sem sofrimento.

---

## Clean Code

Clean Code é a filosofia de escrever código que se explica sozinho, sem a necessidade de comentários excessivos.

### Nomes que revelam intenção

O nome de uma variável, método ou classe deve responder: **o que é isso? para que serve? como é usado?**

```java
// RUIM — nomes sem significado
int d;              // dias? dívida? dados?
List<int[]> lista;  // lista de quê?
boolean flag;       // que flag?

void proc(List<int[]> l) {
    for (int[] x : l) {
        if (x[0] > 4) {
            // o que significa 4? o que é x[0]?
        }
    }
}

// BOM — nomes auto-explicativos
int diasAteVencimento;
List<int[]> celulasAtivas;
boolean estaLogado;

void filtrarCelulasComValorMinimo(List<int[]> celulas) {
    final int VALOR_MINIMO_ACEITAVEL = 4;
    for (int[] celula : celulas) {
        if (celula[STATUS_INDEX] > VALOR_MINIMO_ACEITAVEL) {
            processar(celula);
        }
    }
}

// RUIM — negação dupla e nome confuso
boolean naoNaoVerificado = !naoVerificado;

// BOM — nome positivo
boolean verificado = !naoVerificado;

// RUIM — abreviações enigmáticas
String nm;
int qtdPrd;
BigDecimal vlrTot;

// BOM — nomes completos
String nome;
int quantidadeDeProdutos;
BigDecimal valorTotal;
```

### Funções pequenas — fazem uma coisa só

```java
// RUIM — método que faz tudo
public void processarPedidoCompleto(UUID pedidoId) {
    // valida o pedido (30 linhas)
    var pedido = pedidoRepository.findById(pedidoId).orElseThrow();
    if (pedido.getItens().isEmpty()) throw new IllegalStateException("Pedido sem itens");
    if (pedido.getCliente() == null) throw new IllegalStateException("Cliente obrigatório");
    for (var item : pedido.getItens()) {
        if (item.getQuantidade() <= 0) throw new IllegalStateException("Quantidade inválida");
        if (item.getPreco().compareTo(BigDecimal.ZERO) <= 0) throw new IllegalStateException("Preço inválido");
    }
    // calcula o total (20 linhas)
    BigDecimal subtotal = BigDecimal.ZERO;
    for (var item : pedido.getItens()) {
        subtotal = subtotal.add(item.getPreco().multiply(new BigDecimal(item.getQuantidade())));
    }
    BigDecimal desconto = BigDecimal.ZERO;
    if (pedido.getCliente().isVip()) {
        desconto = subtotal.multiply(new BigDecimal("0.15"));
    }
    BigDecimal total = subtotal.subtract(desconto);
    pedido.setTotal(total);
    // verifica estoque (20 linhas)
    for (var item : pedido.getItens()) {
        int estoqueAtual = estoqueRepository.quantidadeDisponivel(item.getProdutoId());
        if (estoqueAtual < item.getQuantidade()) {
            throw new EstoqueInsuficienteException(item.getProdutoId());
        }
        estoqueRepository.reservar(item.getProdutoId(), item.getQuantidade());
    }
    // confirma e notifica (15 linhas)
    pedido.setStatus(Status.CONFIRMADO);
    pedidoRepository.save(pedido);
    emailService.enviarConfirmacao(pedido);
    smsService.enviarNotificacao(pedido);
}

// BOM — cada método faz UMA coisa
public void processarPedido(UUID pedidoId) {
    var pedido = pedidoRepository.findById(pedidoId).orElseThrow();

    validarPedido(pedido);
    calcularTotal(pedido);
    reservarEstoque(pedido);
    confirmarPedido(pedido);
}

private void validarPedido(Pedido pedido) {
    if (pedido.getItens().isEmpty()) {
        throw new PedidoInvalidoException("Pedido sem itens");
    }
    Objects.requireNonNull(pedido.getCliente(), "Cliente é obrigatório");
    pedido.getItens().forEach(this::validarItem);
}

private void validarItem(ItemPedido item) {
    if (item.getQuantidade() <= 0) {
        throw new PedidoInvalidoException("Quantidade inválida para: " + item.getProdutoId());
    }
    if (item.getPreco().compareTo(BigDecimal.ZERO) <= 0) {
        throw new PedidoInvalidoException("Preço inválido para: " + item.getProdutoId());
    }
}

private void calcularTotal(Pedido pedido) {
    var subtotal = calcularSubtotal(pedido.getItens());
    var desconto = calcularDesconto(pedido.getCliente(), subtotal);
    pedido.setTotal(subtotal.subtract(desconto));
}

private BigDecimal calcularSubtotal(List<ItemPedido> itens) {
    return itens.stream()
        .map(item -> item.getPreco().multiply(new BigDecimal(item.getQuantidade())))
        .reduce(BigDecimal.ZERO, BigDecimal::add);
}

private BigDecimal calcularDesconto(Cliente cliente, BigDecimal subtotal) {
    if (cliente.isVip()) {
        return subtotal.multiply(new BigDecimal("0.15"));
    }
    return BigDecimal.ZERO;
}

private void reservarEstoque(Pedido pedido) {
    pedido.getItens().forEach(item -> {
        int disponivel = estoqueRepository.quantidadeDisponivel(item.getProdutoId());
        if (disponivel < item.getQuantidade()) {
            throw new EstoqueInsuficienteException(item.getProdutoId());
        }
        estoqueRepository.reservar(item.getProdutoId(), item.getQuantidade());
    });
}

private void confirmarPedido(Pedido pedido) {
    pedido.setStatus(Status.CONFIRMADO);
    pedidoRepository.save(pedido);
    notificarCliente(pedido);
}
```

### Comentários — quando são necessários e quando são sintoma de código ruim

```java
// RUIM — comentário que repete o código (ruído)
// Incrementa i
i++;

// Verifica se o usuário está ativo
if (usuario.getAtivo()) { ... }

// Obtém o nome do cliente
String nomeCliente = pedido.getCliente().getNome();

// BOM — comentário explica o PORQUÊ, não o O QUÊ
// A API externa retorna HTTP 204 para recursos não encontrados (bug deles, não nosso)
if (resposta.getStatusCode() == 204) {
    return Optional.empty();
}

// O limite de 100 itens é uma restrição da API de pagamentos (documentação v2.3)
static final int LIMITE_ITENS_POR_LOTE = 100;

// Algoritmo baseado em https://example.com/algoritmo-especifico — não trivial de reescrever
double calcularVolatilidade(List<Double> precos) {
    // ... implementação complexa
}

// TODO: remover após migração para novo sistema de autenticação (Sprint 45)
@Deprecated
public String gerarTokenLegado(String usuario) { ... }

// PÉSSIMO — código comentado (use git para histórico)
// public void metodoAntigo() {
//     // ... 50 linhas comentadas
// }
```

### DRY — Don't Repeat Yourself

```java
// RUIM — lógica duplicada
public BigDecimal calcularImpostoPF(BigDecimal rendimento) {
    if (rendimento.compareTo(new BigDecimal("2112.00")) <= 0) return BigDecimal.ZERO;
    if (rendimento.compareTo(new BigDecimal("2826.65")) <= 0)
        return rendimento.multiply(new BigDecimal("0.075"));
    // ... mais faixas
    return rendimento.multiply(new BigDecimal("0.275"));
}

public String descricaoAliquotaPF(BigDecimal rendimento) {
    if (rendimento.compareTo(new BigDecimal("2112.00")) <= 0) return "Isento";
    if (rendimento.compareTo(new BigDecimal("2826.65")) <= 0) return "7,5%";
    // ... duplicação das mesmas condições
    return "27,5%";
}

// BOM — extraindo a tabela para um lugar só
record FaixaIR(BigDecimal limiteInferior, BigDecimal limiteSuperior,
               BigDecimal aliquota, String descricao) {

    public boolean abrange(BigDecimal valor) {
        return valor.compareTo(limiteInferior) > 0
            && (limiteSuperior == null || valor.compareTo(limiteSuperior) <= 0);
    }
}

public class TabelaIRPF {
    private static final List<FaixaIR> FAIXAS = List.of(
        new FaixaIR(BigDecimal.ZERO,          new BigDecimal("2112.00"),  BigDecimal.ZERO,              "Isento"),
        new FaixaIR(new BigDecimal("2112.00"), new BigDecimal("2826.65"), new BigDecimal("0.075"), "7,5%"),
        new FaixaIR(new BigDecimal("2826.65"), new BigDecimal("3751.05"), new BigDecimal("0.150"), "15%"),
        new FaixaIR(new BigDecimal("3751.05"), new BigDecimal("4664.68"), new BigDecimal("0.225"), "22,5%"),
        new FaixaIR(new BigDecimal("4664.68"), null,                       new BigDecimal("0.275"), "27,5%")
    );

    public FaixaIR obterFaixa(BigDecimal rendimento) {
        return FAIXAS.stream()
            .filter(f -> f.abrange(rendimento))
            .findFirst()
            .orElseThrow();
    }

    public BigDecimal calcularImposto(BigDecimal rendimento) {
        return rendimento.multiply(obterFaixa(rendimento).aliquota());
    }

    public String descricaoAliquota(BigDecimal rendimento) {
        return obterFaixa(rendimento).descricao();
    }
}
```

### KISS — Keep It Simple, Stupid

```java
// RUIM — complexidade desnecessária
public boolean verificarSeEhPar(int numero) {
    return Integer.toBinaryString(Math.abs(numero)).charAt(
        Integer.toBinaryString(Math.abs(numero)).length() - 1
    ) == '0';
}

// BOM — simples e claro
public boolean ehPar(int numero) {
    return numero % 2 == 0;
}

// RUIM — sobre-engenharia
public interface VerificadorDeParidade {
    boolean verificar(int numero);
}
public class VerificadorDeParidadeImpl implements VerificadorDeParidade {
    @Override
    public boolean verificar(int numero) { return numero % 2 == 0; }
}
public class FabricaDeVerificador {
    public VerificadorDeParidade criar() { return new VerificadorDeParidadeImpl(); }
}

// BOM — quando uma função é suficiente
Predicate<Integer> ehPar = n -> n % 2 == 0;
```

---

## SOLID

> **SOLID é heurística, não lei.** Robert Martin criou os princípios para ajudar desenvolvedores a escrever código que não vira bola de lama com o tempo — não para ser um checklist de code review. O risco real de aplicar SOLID mecanicamente não é que ele produza código ruim: é que ele dá a sensação de que código bom foi produzido. Você fragmenta classes, satisfaz as cinco letras, passa no code review — e entrega um sistema que ninguém consegue mudar sem medo, porque as peças foram cortadas no tamanho certo mas coladas no lugar errado.
>
> Quem pensa em domínio antes aplica SOLID naturalmente — o raciocínio vem primeiro, a letra é consequência. Os exemplos abaixo mostram os princípios em ação, mas o ponto de partida sempre deve ser: "quais são as responsabilidades reais do meu domínio?" — não "quantas classes estou usando?"

### S — Single Responsibility Principle

Uma classe deve ter apenas um motivo para mudar — e esse motivo deve ser rastreável a uma responsabilidade real do domínio, não a um critério de tamanho de arquivo.

```java
// VIOLANDO SRP — classe que faz tudo
public class PedidoService {
    public void processar(Pedido pedido) {
        // Validação
        if (pedido.getItens().isEmpty()) throw new IllegalStateException("Sem itens");

        // Cálculo de preço
        BigDecimal total = pedido.getItens().stream()
            .map(i -> i.getPreco().multiply(BigDecimal.valueOf(i.getQtd())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        // Persistência
        String sql = "INSERT INTO pedidos (id, total) VALUES (?, ?)";
        jdbcTemplate.update(sql, pedido.getId(), total);

        // Envio de e-mail
        String corpo = "Seu pedido foi recebido. Total: R$ " + total;
        emailSender.send(pedido.getCliente().getEmail(), "Pedido confirmado", corpo);

        // Geração de relatório
        relatorioService.gerarComprovante(pedido);
    }
}

// CORRETO — cada classe tem uma responsabilidade
public class PedidoService {
    private final ValidadorDePedido validador;
    private final CalculadoraDePreco calculadora;
    private final PedidoRepository repository;
    private final NotificacaoService notificacao;

    public void processar(Pedido pedido) {
        validador.validar(pedido);
        var total = calculadora.calcularTotal(pedido);
        pedido.confirmar(total);
        repository.salvar(pedido);
        notificacao.notificarConfirmacao(pedido);
    }
}

public class ValidadorDePedido {
    public void validar(Pedido pedido) { /* somente validações */ }
}

public class CalculadoraDePreco {
    public BigDecimal calcularTotal(Pedido pedido) { /* somente cálculos */ }
}

public class NotificacaoService {
    public void notificarConfirmacao(Pedido pedido) { /* somente notificações */ }
}
```

### O — Open/Closed Principle

Aberto para extensão, fechado para modificação.

```java
// VIOLANDO OCP — cada novo tipo de desconto exige modificar a classe
public class CalculadoraDeDesconto {
    public BigDecimal calcular(Pedido pedido) {
        if (pedido.getTipoCliente() == TipoCliente.VIP) {
            return pedido.getTotal().multiply(new BigDecimal("0.20"));
        } else if (pedido.getTipoCliente() == TipoCliente.PREMIUM) {
            return pedido.getTotal().multiply(new BigDecimal("0.10"));
        } else if (pedido.getTipoCliente() == TipoCliente.FUNCIONARIO) {   // adicionado depois
            return pedido.getTotal().multiply(new BigDecimal("0.30"));
        }
        return BigDecimal.ZERO;
    }
}

// CORRETO — estende sem modificar
public interface RegraDeDesconto {
    boolean seAplica(Pedido pedido);
    BigDecimal calcular(Pedido pedido);
}

public class DescontoVIP implements RegraDeDesconto {
    @Override
    public boolean seAplica(Pedido pedido) {
        return pedido.getCliente().getTipo() == TipoCliente.VIP;
    }

    @Override
    public BigDecimal calcular(Pedido pedido) {
        return pedido.getTotal().multiply(new BigDecimal("0.20"));
    }
}

public class DescontoFuncionario implements RegraDeDesconto {
    @Override
    public boolean seAplica(Pedido pedido) {
        return pedido.getCliente().getTipo() == TipoCliente.FUNCIONARIO;
    }

    @Override
    public BigDecimal calcular(Pedido pedido) {
        return pedido.getTotal().multiply(new BigDecimal("0.30"));
    }
}

// Calculadora não precisa ser modificada para novos descontos
public class CalculadoraDeDesconto {
    private final List<RegraDeDesconto> regras;

    public CalculadoraDeDesconto(List<RegraDeDesconto> regras) {
        this.regras = regras;
    }

    public BigDecimal calcular(Pedido pedido) {
        return regras.stream()
            .filter(r -> r.seAplica(pedido))
            .map(r -> r.calcular(pedido))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

### L — Liskov Substitution Principle

Subtipos devem ser substituíveis pelos tipos base sem quebrar o comportamento do programa.

```java
// VIOLANDO LSP
public class Retangulo {
    protected int largura, altura;
    public void setLargura(int l) { this.largura = l; }
    public void setAltura(int a)  { this.altura  = a; }
    public int calcularArea()     { return largura * altura; }
}

public class Quadrado extends Retangulo {
    @Override
    public void setLargura(int l) {
        this.largura = l;
        this.altura  = l;   // viola a expectativa do Retangulo
    }

    @Override
    public void setAltura(int a) {
        this.largura = a;
        this.altura  = a;   // viola a expectativa do Retangulo
    }
}

// Código que usa Retangulo é surpreendido por Quadrado
Retangulo r = new Quadrado();
r.setLargura(5);
r.setAltura(10);
System.out.println(r.calcularArea());   // Esperado: 50. Real: 100

// CORRETO — usar composição em vez de herança problemática
public interface Forma {
    int calcularArea();
}

public record Retangulo(int largura, int altura) implements Forma {
    @Override
    public int calcularArea() { return largura * altura; }
}

public record Quadrado(int lado) implements Forma {
    @Override
    public int calcularArea() { return lado * lado; }
}
```

### I — Interface Segregation Principle

Nenhum cliente deve ser forçado a depender de interfaces que não usa.

```java
// VIOLANDO ISP — interface "gorda"
public interface Trabalhador {
    void trabalhar();
    void comer();
    void dormir();
    void receberSalario();
}

// Robô implementa Trabalhador mas não come nem dorme
public class Robo implements Trabalhador {
    @Override public void trabalhar()       { /* ok */ }
    @Override public void comer()           { throw new UnsupportedOperationException("Robôs não comem!"); }
    @Override public void dormir()          { throw new UnsupportedOperationException("Robôs não dormem!"); }
    @Override public void receberSalario()  { throw new UnsupportedOperationException("Robôs não recebem!"); }
}

// CORRETO — interfaces coesas e específicas
public interface Trabalhavel { void trabalhar(); }
public interface Alimentavel  { void comer(); }
public interface Descansavel  { void dormir(); }
public interface Remuneravel  { void receberSalario(); }

public class FuncionarioHumano implements Trabalhavel, Alimentavel, Descansavel, Remuneravel {
    @Override public void trabalhar()       { /* trabalha */ }
    @Override public void comer()           { /* come */ }
    @Override public void dormir()          { /* dorme */ }
    @Override public void receberSalario()  { /* recebe */ }
}

public class Robo implements Trabalhavel {
    @Override public void trabalhar() { /* trabalha */ }
    // Não precisa implementar o que não faz
}
```

### D — Dependency Inversion Principle

Módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de abstrações.

```java
// VIOLANDO DIP — alto nível depende de baixo nível
public class PedidoService {
    // Dependência direta na implementação concreta
    private final MySQLPedidoRepository repository = new MySQLPedidoRepository();
    private final SmtpEmailService emailService    = new SmtpEmailService();

    public void processar(Pedido pedido) {
        repository.salvar(pedido);     // acoplado ao MySQL
        emailService.enviar(pedido);   // acoplado ao SMTP
    }
}

// CORRETO — dependência nas abstrações (injeção de dependência)
public interface PedidoRepository {
    void salvar(Pedido pedido);
    Optional<Pedido> findById(UUID id);
}

public interface NotificacaoService {
    void notificar(Pedido pedido);
}

public class PedidoService {
    private final PedidoRepository  repository;    // interface
    private final NotificacaoService notificacao;  // interface

    // Dependências injetadas (Spring faz isso automaticamente com @Autowired)
    public PedidoService(PedidoRepository repository, NotificacaoService notificacao) {
        this.repository  = repository;
        this.notificacao = notificacao;
    }

    public void processar(Pedido pedido) {
        repository.salvar(pedido);
        notificacao.notificar(pedido);
    }
}

// Agora você pode injetar qualquer implementação:
// - Em produção: JpaPedidoRepository + SmtpNotificacaoService
// - Em testes:   InMemoryPedidoRepository + FakeNotificacaoService
```

---

## Object Calisthenics — as 9 regras

Object Calisthenics é um conjunto de exercícios de código que, quando seguidos, levam naturalmente a código orientado a objetos de alta qualidade.

### Regra 1: Um nível de indentação por método

```java
// RUIM — múltiplos níveis de indentação
public void processarPedidos(List<Pedido> pedidos) {
    for (Pedido pedido : pedidos) {                    // nível 1
        if (pedido.isAtivo()) {                        // nível 2
            for (ItemPedido item : pedido.getItens()) {// nível 3
                if (item.getEstoque() > 0) {           // nível 4 — extrair método!
                    processar(item);
                }
            }
        }
    }
}

// BOM — um nível por método
public void processarPedidos(List<Pedido> pedidos) {
    pedidos.stream()
        .filter(Pedido::isAtivo)
        .forEach(this::processarItensPedido);
}

private void processarItensPedido(Pedido pedido) {
    pedido.getItens().stream()
        .filter(item -> item.getEstoque() > 0)
        .forEach(this::processar);
}
```

### Regra 2: Não use else

```java
// RUIM — else aninhado
public String classificarCliente(Cliente cliente) {
    if (cliente.getComprasTotal() > 10000) {
        return "VIP";
    } else {
        if (cliente.getComprasTotal() > 5000) {
            return "PREMIUM";
        } else {
            return "COMUM";
        }
    }
}

// BOM — early return elimina else
public String classificarCliente(Cliente cliente) {
    if (cliente.getComprasTotal() > 10000) return "VIP";
    if (cliente.getComprasTotal() > 5000)  return "PREMIUM";
    return "COMUM";
}
```

### Regra 3: Encapsule todos os primitivos e Strings (Value Objects)

```java
// RUIM — primitivos espalhados
public void criarUsuario(String email, String cpf, int idade) {
    if (!email.contains("@")) throw new IllegalArgumentException("Email inválido");
    if (cpf.length() != 11)   throw new IllegalArgumentException("CPF inválido");
    if (idade < 18)            throw new IllegalArgumentException("Menor de idade");
    // ... validações duplicadas em toda a aplicação
}

// BOM — Value Objects com validação encapsulada
public record Email(String valor) {
    public Email {
        Objects.requireNonNull(valor, "Email não pode ser nulo");
        if (!valor.matches("^[\\w.-]+@[\\w.-]+\\.[a-z]{2,}$")) {
            throw new IllegalArgumentException("Email inválido: " + valor);
        }
        valor = valor.toLowerCase().trim();
    }
}

public record CPF(String valor) {
    public CPF {
        Objects.requireNonNull(valor, "CPF não pode ser nulo");
        var soDigitos = valor.replaceAll("[^0-9]", "");
        if (soDigitos.length() != 11) {
            throw new IllegalArgumentException("CPF inválido: " + valor);
        }
        valor = soDigitos;
    }
}

public record Idade(int valor) {
    public Idade {
        if (valor < 0 || valor > 150) {
            throw new IllegalArgumentException("Idade inválida: " + valor);
        }
    }

    public boolean ehMaiorDeIdade() { return valor >= 18; }
}

// Uso — a validação acontece na construção dos VOs
public void criarUsuario(Email email, CPF cpf, Idade idade) {
    if (!idade.ehMaiorDeIdade()) {
        throw new RegraDeNegocioException("Usuário deve ser maior de 18 anos");
    }
    // email, cpf e idade já foram validados
}
```

### Regra 4: Coleções de primeira classe

```java
// RUIM — lista crua espalhada pela aplicação
List<Item> itens = new ArrayList<>();
if (itens.size() > 10) { /* regra de negócio em vários lugares */ }

// BOM — coleção com comportamento
public class ListaDeItens {
    private static final int LIMITE = 10;
    private final List<Item> itens;

    public ListaDeItens() {
        this.itens = new ArrayList<>();
    }

    public void adicionar(Item item) {
        if (itens.size() >= LIMITE) {
            throw new LimiteDePedidoExcedido("Máximo de " + LIMITE + " itens por pedido");
        }
        itens.add(item);
    }

    public BigDecimal calcularTotal() {
        return itens.stream()
            .map(Item::subtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    public boolean estaVazia() { return itens.isEmpty(); }
    public int quantidade()    { return itens.size(); }
    public List<Item> toList() { return Collections.unmodifiableList(itens); }
}
```

### Regra 5: Um ponto por linha (Lei de Demeter)

```java
// RUIM — violação da Lei de Demeter ("acidente de trem")
String cidade = pedido.getCliente().getEndereco().getCidade().getNome();

// BOM — cada objeto expõe o que o cliente precisa
public record Cliente(String nome, Endereco endereco) {
    public String cidadeDeEntrega() {
        return endereco.cidade();
    }
}

public record Pedido(Cliente cliente, List<Item> itens) {
    public String cidadeDeEntrega() {
        return cliente.cidadeDeEntrega();
    }
}

// Código limpo
String cidade = pedido.cidadeDeEntrega();
```

### Regras 6 a 9 (resumo aplicado)

```java
// Regra 6: Não use abreviações — já demonstrado na seção de nomes
// Regra 7: Mantenha entidades pequenas (< 50 linhas por método, < 200 por classe)
// Regra 8: Não mais de 2 variáveis de instância por classe — incentiva classes focadas
// Regra 9: Sem getters/setters (Tell, don't ask) — os objetos fazem, não expõem estado

// Exemplo da Regra 9:
// RUIM — getter/setter (você busca o estado e decide de fora)
if (conta.getSaldo() >= valor) {
    conta.setSaldo(conta.getSaldo() - valor);
}

// BOM — o objeto decide e executa
conta.sacar(valor);   // a conta verifica o saldo internamente
```

---

## Green Coding — Eficiência e Sustentabilidade

```java
// Evitar loops ineficientes — preferir operações de coleção
// RUIM: O(n²) sem necessidade
List<String> resultado = new ArrayList<>();
for (String item : listaA) {
    for (String outro : listaB) {
        if (item.equals(outro)) {
            resultado.add(item);
        }
    }
}

// BOM: O(n) — usar Set para lookup
Set<String> conjuntoB = new HashSet<>(listaB);
List<String> resultado = listaA.stream()
    .filter(conjuntoB::contains)
    .toList();

// Escolha correta de coleção
// Se você precisa de lookup por chave: Map, não List
// Se precisa de unicidade: Set, não List com contains()
// Se precisa de fila: ArrayDeque, não LinkedList

// Lazy initialization — não crie objetos antes de precisar
// RUIM — cria o objeto mesmo quando não vai usar
public class RelatorioService {
    private final List<Pedido> cache = new ArrayList<>();   // criado sempre

// BOM — inicializa quando necessário
public class RelatorioService {
    private List<Pedido> cache;

    private List<Pedido> getCache() {
        if (cache == null) {
            cache = carregarDoBanco();   // só chama quando necessário
        }
        return cache;
    }
}

// Fechar recursos — evitar vazamentos de memória
// Use try-with-resources para qualquer AutoCloseable (seção 3.3)

// Evitar String concatenation em loops
// RUIM — cria N objetos String
String resultado = "";
for (var linha : linhas) {
    resultado += linha + "\n";   // O(n²) — nova String a cada iteração
}

// BOM — StringBuilder é O(n)
var sb = new StringBuilder();
for (var linha : linhas) {
    sb.append(linha).append('\n');
}
String resultado = sb.toString();

// Ou ainda melhor com Streams:
String resultado = String.join("\n", linhas);
```

---

## Convenções Java

```java
// Classes e interfaces — PascalCase
public class PedidoService { }
public interface RepositorioDePedidos { }
public record ClienteDTO(String nome) { }
public enum StatusPedido { PENDENTE, PAGO, CANCELADO }
public @interface ValidarCPF { }

// Métodos e variáveis — camelCase
public void calcularTotal() { }
private String nomeCompleto;
boolean estaAtivo;

// Constantes — UPPER_SNAKE_CASE
public static final int LIMITE_MAXIMO_ITENS = 100;
private static final String PREFIXO_ID = "BR-";

// Pacotes — lowercase, sem underscores
package com.minhaempresa.sistemafinanceiro.dominio.pedido;
package com.minhaempresa.sistemafinanceiro.aplicacao.service;
package com.minhaempresa.sistemafinanceiro.infraestrutura.repository;

// Estrutura de pacotes recomendada (por feature/domínio)
com.empresa.app
├── pedido
│   ├── Pedido.java              (entidade)
│   ├── PedidoRepository.java    (interface)
│   ├── PedidoService.java       (serviço de aplicação)
│   └── PedidoController.java    (adapter HTTP)
├── cliente
│   ├── Cliente.java
│   └── ...
└── shared
    ├── exception
    └── valueobject

// Nomes de testes — deve + cenário + resultado esperado
class PedidoServiceTest {
    @Test
    void deveLancarExcecaoQuandoPedidoNaoTemItens() { }

    @Test
    void deveAplicarDescontoVIPCorretamente() { }

    @Test
    void naoDevePermitirPedidoComValorNegativo() { }
}
```

---

## SonarLint — as regras mais importantes

O SonarLint analisa seu código em tempo real e aponta violações. Estas são as regras mais críticas:

| Regra                                       | O que detecta                                         | Por que importa                          |
|---------------------------------------------|-------------------------------------------------------|------------------------------------------|
| `java:S106`                                 | Uso de `System.out.println` em vez de logger          | Sem controle de nível, sem estrutura     |
| `java:S112`                                 | Lançar `RuntimeException` genérica                    | Perde informação, dificulta tratamento   |
| `java:S1135`                                | TODO/FIXME comments                                   | Dívida técnica não rastreada             |
| `java:S1186`                                | Método vazio sem comentário explicando                | Pode ser bug ou código incompleto        |
| `java:S1481`                                | Variável local declarada mas não usada                | Dead code, confunde leitores             |
| `java:S2095`                                | Recurso não fechado                                   | Vazamento de memória/conexão             |
| `java:S2139`                                | Exceção logada e relançada                            | Stack trace duplicada no log             |
| `java:S2259`                                | Possível NullPointerException                         | Crash em produção                        |
| `java:S3776`                                | Complexidade cognitiva muito alta                     | Código difícil de entender e manter      |
| `java:S4925`                                | Uso de `Optional.get()` sem `isPresent()`             | NoSuchElementException em produção       |

```java
// Exemplos de violações detectadas pelo SonarLint

// S106 — use logger
System.out.println("pedido processado");   // ALERTA
log.info("pedido processado");              // correto

// S112 — exceção específica
throw new RuntimeException("erro");         // ALERTA
throw new PedidoInvalidoException("erro"); // correto

// S2095 — recurso não fechado
FileReader reader = new FileReader("arquivo.txt");
reader.read();
// reader.close() esquecido — ALERTA
try (var r = new FileReader("arquivo.txt")) { r.read(); } // correto

// S2139 — não logue E relance
} catch (IOException e) {
    log.error("erro", e);   // logou
    throw e;                // e relançou — ALERTA (stack trace duplicada)
}
// Correto: logue OU relance, não ambos — exceto ao converter para outro tipo:
} catch (IOException e) {
    throw new ProcessamentoException("Falha ao processar", e);   // sem log aqui
}
// O handler global loga o erro com a causa encadeada

// S4925 — Optional.get() sem verificação
Optional<Pedido> opt = buscarPedido(id);
Pedido pedido = opt.get();   // ALERTA — pode lançar NoSuchElementException
// correto:
Pedido pedido = opt.orElseThrow(() -> new PedidoNaoEncontradoException(id));
```

> **Dica:** configure o SonarLint para se conectar ao SonarCloud/SonarQube do time e garantir que as mesmas regras estejam sendo aplicadas localmente e no CI/CD. Assim, alertas locais são garantia de que o build do CI também passará.
