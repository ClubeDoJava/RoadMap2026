# 7.1 - Padrões de Projeto (Design Patterns)

## O que são e por que importam

Design Patterns são soluções reutilizáveis para problemas recorrentes no design de software orientado a objetos. Foram catalogados pela "Gang of Four" (GoF) no livro clássico de 1994 e organizados em três categorias:

- **Criacionais:** como os objetos são criados
- **Estruturais:** como os objetos se relacionam e se compõem
- **Comportamentais:** como os objetos se comunicam e distribuem responsabilidades

**Por que importam:**
- Vocabulário comum entre desenvolvedores: dizer "use um Strategy aqui" é mais preciso que explicar o conceito do zero
- Soluções comprovadas por décadas de uso
- Código mais legível, extensível e testável
- Base para entender frameworks (Spring usa Factory, Proxy, Template Method, etc.)

---

## Padrões Criacionais

### Singleton — com thread-safety usando enum

**Problema que resolve:** garantir que uma classe tenha apenas uma instância e fornecer um ponto de acesso global a ela.

```
┌─────────────────────────────────┐
│          Singleton              │
│─────────────────────────────────│
│ - instancia: Singleton          │
│─────────────────────────────────│
│ + getInstance(): Singleton      │
│ + operacao()                    │
└─────────────────────────────────┘
```

```java
// MELHOR FORMA: usando enum (thread-safe, serialização segura, reflexão bloqueada)
public enum ConexaoBancoDados {
    INSTANCE;

    private Connection connection;

    ConexaoBancoDados() {
        // Inicialização acontece uma única vez, thread-safe pela JVM
        try {
            this.connection = DriverManager.getConnection("jdbc:...");
        } catch (SQLException e) {
            throw new RuntimeException("Erro ao conectar ao banco", e);
        }
    }

    public Connection getConnection() {
        return connection;
    }
}

// Uso
Connection conn = ConexaoBancoDados.INSTANCE.getConnection();

// ─────────────────────────────────────────────────────────────────
// Forma alternativa: double-checked locking (quando enum não serve)
// ─────────────────────────────────────────────────────────────────
public class Cache {
    // volatile garante visibilidade entre threads
    private static volatile Cache instancia;
    private final Map<String, Object> dados = new ConcurrentHashMap<>();

    private Cache() {}

    public static Cache getInstance() {
        if (instancia == null) {
            synchronized (Cache.class) {
                if (instancia == null) {  // Segunda checagem dentro do synchronized
                    instancia = new Cache();
                }
            }
        }
        return instancia;
    }

    public void put(String chave, Object valor) {
        dados.put(chave, valor);
    }

    public Object get(String chave) {
        return dados.get(chave);
    }
}
```

**No Spring:** a maioria dos beans é Singleton por padrão (`@Service`, `@Repository`, `@Controller`). Você raramente implementa Singleton manualmente.

---

### Factory Method

**Problema que resolve:** criar objetos sem expor a lógica de criação ao cliente, delegando para subclasses ou métodos especializados.

```
┌──────────────────────┐        ┌────────────────────────┐
│   NotificacaoFactory │        │      Notificacao        │
│──────────────────────│        │────────────────────────│
│ + criar(tipo): Notif.│───────>│ + enviar(msg: String)  │
└──────────────────────┘        └────────────────────────┘
           △                              △
           │                    ┌─────────┴──────────┐
     ┌─────┴──────┐         ┌───┴───┐  ┌──────┐  ┌──┴──┐
     │   criação  │         │ Email │  │ SMS  │  │Push │
     └────────────┘         └───────┘  └──────┘  └─────┘
```

```java
// Produto: interface comum
public interface Notificacao {
    void enviar(String destinatario, String mensagem);
}

// Produtos concretos
public class NotificacaoEmail implements Notificacao {
    @Override
    public void enviar(String destinatario, String mensagem) {
        System.out.println("Email para " + destinatario + ": " + mensagem);
        // lógica de envio de email...
    }
}

public class NotificacaoSMS implements Notificacao {
    @Override
    public void enviar(String destinatario, String mensagem) {
        System.out.println("SMS para " + destinatario + ": " + mensagem);
    }
}

public class NotificacaoPush implements Notificacao {
    @Override
    public void enviar(String destinatario, String mensagem) {
        System.out.println("Push para " + destinatario + ": " + mensagem);
    }
}

// Factory: esconde a lógica de criação
public class NotificacaoFactory {
    public static Notificacao criar(String tipo) {
        return switch (tipo.toLowerCase()) {
            case "email" -> new NotificacaoEmail();
            case "sms"   -> new NotificacaoSMS();
            case "push"  -> new NotificacaoPush();
            default      -> throw new IllegalArgumentException("Tipo desconhecido: " + tipo);
        };
    }
}

// Uso
Notificacao notif = NotificacaoFactory.criar("email");
notif.enviar("joao@email.com", "Seu pedido foi aprovado!");
```

---

### Abstract Factory

**Problema que resolve:** criar famílias de objetos relacionados sem especificar as classes concretas.

```java
// Famílias de UI para diferentes plataformas
public interface BotaoUI { void renderizar(); }
public interface InputUI  { void renderizar(); }

// Família Windows
public class BotaoWindows implements BotaoUI {
    public void renderizar() { System.out.println("Botão Windows [▣]"); }
}
public class InputWindows implements InputUI {
    public void renderizar() { System.out.println("Input Windows [________]"); }
}

// Família macOS
public class BotaoMacOS implements BotaoUI {
    public void renderizar() { System.out.println("Botão macOS (●)"); }
}
public class InputMacOS implements InputUI {
    public void renderizar() { System.out.println("Input macOS ⌈________⌉"); }
}

// Abstract Factory
public interface UIFactory {
    BotaoUI criarBotao();
    InputUI criarInput();
}

// Factories concretas
public class WindowsUIFactory implements UIFactory {
    public BotaoUI criarBotao() { return new BotaoWindows(); }
    public InputUI criarInput()  { return new InputWindows(); }
}

public class MacOSUIFactory implements UIFactory {
    public BotaoUI criarBotao() { return new BotaoMacOS(); }
    public InputUI criarInput()  { return new InputMacOS(); }
}

// Aplicação usa apenas a interface
public class Aplicacao {
    private final UIFactory factory;

    public Aplicacao(UIFactory factory) {
        this.factory = factory;
    }

    public void renderizarFormulario() {
        factory.criarInput().renderizar();
        factory.criarBotao().renderizar();
    }
}

// Uso
String os = System.getProperty("os.name");
UIFactory factory = os.contains("Windows") ? new WindowsUIFactory() : new MacOSUIFactory();
new Aplicacao(factory).renderizarFormulario();
```

---

### Builder — com inner class fluente

**Problema que resolve:** construir objetos complexos passo a passo, evitando construtores com muitos parâmetros ("telescoping constructor").

```java
public class Pedido {
    private final Long id;
    private final String clienteId;
    private final List<ItemPedido> itens;
    private final String enderecoEntrega;
    private final String metodoPagamento;
    private final BigDecimal desconto;
    private final String observacoes;

    // Construtor privado: só o Builder pode criar
    private Pedido(Builder builder) {
        this.id = builder.id;
        this.clienteId = builder.clienteId;
        this.itens = Collections.unmodifiableList(builder.itens);
        this.enderecoEntrega = builder.enderecoEntrega;
        this.metodoPagamento = builder.metodoPagamento;
        this.desconto = builder.desconto;
        this.observacoes = builder.observacoes;
    }

    // Getters...
    public Long getId() { return id; }
    public String getClienteId() { return clienteId; }
    public List<ItemPedido> getItens() { return itens; }

    // Inner Builder class
    public static class Builder {
        // Campos obrigatórios
        private final Long id;
        private final String clienteId;
        private final List<ItemPedido> itens = new ArrayList<>();

        // Campos opcionais com valores padrão
        private String enderecoEntrega;
        private String metodoPagamento = "CARTAO";
        private BigDecimal desconto = BigDecimal.ZERO;
        private String observacoes;

        // Construtor do Builder exige apenas os campos obrigatórios
        public Builder(Long id, String clienteId) {
            this.id = id;
            this.clienteId = clienteId;
        }

        public Builder adicionarItem(ItemPedido item) {
            this.itens.add(item);
            return this;  // Retorna o próprio Builder para encadear chamadas
        }

        public Builder enderecoEntrega(String endereco) {
            this.enderecoEntrega = endereco;
            return this;
        }

        public Builder metodoPagamento(String metodo) {
            this.metodoPagamento = metodo;
            return this;
        }

        public Builder desconto(BigDecimal desconto) {
            this.desconto = desconto;
            return this;
        }

        public Builder observacoes(String obs) {
            this.observacoes = obs;
            return this;
        }

        public Pedido build() {
            // Validações antes de construir
            if (itens.isEmpty()) {
                throw new IllegalStateException("Pedido deve ter ao menos um item");
            }
            if (enderecoEntrega == null || enderecoEntrega.isBlank()) {
                throw new IllegalStateException("Endereço de entrega é obrigatório");
            }
            return new Pedido(this);
        }
    }
}

// Uso — legível e fluente
Pedido pedido = new Pedido.Builder(1L, "cliente-42")
        .adicionarItem(new ItemPedido("produto-1", 2, BigDecimal.valueOf(29.90)))
        .adicionarItem(new ItemPedido("produto-2", 1, BigDecimal.valueOf(59.90)))
        .enderecoEntrega("Rua das Flores, 123 - São Paulo")
        .metodoPagamento("PIX")
        .desconto(BigDecimal.valueOf(10))
        .observacoes("Entregar após 18h")
        .build();
```

**No Spring:** Lombok `@Builder` gera isso automaticamente. Na prática você usa `@Builder` nas entidades e DTOs.

---

### Prototype

**Problema que resolve:** criar novos objetos copiando (clonando) um objeto existente, evitando operações de inicialização caras.

```java
public abstract class Relatorio implements Cloneable {
    protected String titulo;
    protected List<String> colunas = new ArrayList<>();
    protected Map<String, Object> filtros = new HashMap<>();

    // Retorna uma cópia do objeto
    @Override
    public Relatorio clone() {
        try {
            Relatorio clone = (Relatorio) super.clone();
            // Deep copy das coleções mutáveis
            clone.colunas = new ArrayList<>(this.colunas);
            clone.filtros = new HashMap<>(this.filtros);
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}

public class RelatorioVendas extends Relatorio {
    private String periodoReferencia;

    public RelatorioVendas(String titulo) {
        this.titulo = titulo;
        // Simulando inicialização cara (busca de configurações no banco, etc.)
        this.colunas = List.of("Data", "Produto", "Quantidade", "Valor");
    }
}

// Uso: clone é muito mais rápido que criar do zero
RelatorioVendas template = new RelatorioVendas("Relatório de Vendas");
template.filtros.put("ativo", true);

// Clonar e personalizar para cada usuário
RelatorioVendas relatorioJoao = (RelatorioVendas) template.clone();
relatorioJoao.filtros.put("vendedor", "João");

RelatorioVendas relatorioMaria = (RelatorioVendas) template.clone();
relatorioMaria.filtros.put("vendedor", "Maria");
```

---

## Padrões Estruturais

### Adapter

**Problema que resolve:** fazer dois objetos com interfaces incompatíveis trabalharem juntos.

```
┌──────────┐    usa    ┌──────────┐  adapta  ┌─────────────────┐
│  Cliente │─────────>│ Adapter  │─────────>│ ServicoExterno  │
└──────────┘           └──────────┘          │ (interface diff) │
                                             └─────────────────┘
```

```java
// Interface que nosso código espera
public interface GatewayPagamento {
    boolean processar(String cartao, BigDecimal valor);
}

// SDK de terceiro com interface diferente (não podemos modificar)
public class PagSeguroSDK {
    public PagSeguroResposta realizarTransacao(PagSeguroRequest request) {
        // lógica interna do PagSeguro...
        return new PagSeguroResposta("APROVADO", "TXN-12345");
    }
}

// Adapter: faz o PagSeguro parecer um GatewayPagamento
public class PagSeguroAdapter implements GatewayPagamento {
    private final PagSeguroSDK sdk;

    public PagSeguroAdapter(PagSeguroSDK sdk) {
        this.sdk = sdk;
    }

    @Override
    public boolean processar(String cartao, BigDecimal valor) {
        PagSeguroRequest request = new PagSeguroRequest(cartao, valor.doubleValue());
        PagSeguroResposta resposta = sdk.realizarTransacao(request);
        return "APROVADO".equals(resposta.getStatus());
    }
}

// Uso: o cliente só conhece GatewayPagamento
GatewayPagamento gateway = new PagSeguroAdapter(new PagSeguroSDK());
boolean aprovado = gateway.processar("4111-1111-1111-1111", BigDecimal.valueOf(150.00));
```

---

### Decorator

**Problema que resolve:** adicionar comportamentos a objetos dinamicamente, sem modificar a classe original nem usar herança.

```java
// Componente base
public interface Pedido {
    BigDecimal calcularTotal();
    String getDescricao();
}

public class PedidoSimples implements Pedido {
    private final List<Item> itens;

    public PedidoSimples(List<Item> itens) {
        this.itens = itens;
    }

    @Override
    public BigDecimal calcularTotal() {
        return itens.stream()
                .map(Item::getPreco)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    @Override
    public String getDescricao() {
        return "Pedido com " + itens.size() + " itens";
    }
}

// Decorator base
public abstract class PedidoDecorator implements Pedido {
    protected final Pedido pedidoDecorado;

    public PedidoDecorator(Pedido pedido) {
        this.pedidoDecorado = pedido;
    }
}

// Decoradores concretos
public class PedidoComFrete extends PedidoDecorator {
    private final BigDecimal frete;

    public PedidoComFrete(Pedido pedido, BigDecimal frete) {
        super(pedido);
        this.frete = frete;
    }

    @Override
    public BigDecimal calcularTotal() {
        return pedidoDecorado.calcularTotal().add(frete);
    }

    @Override
    public String getDescricao() {
        return pedidoDecorado.getDescricao() + " + frete R$" + frete;
    }
}

public class PedidoComDesconto extends PedidoDecorator {
    private final BigDecimal percentualDesconto;

    public PedidoComDesconto(Pedido pedido, BigDecimal percentualDesconto) {
        super(pedido);
        this.percentualDesconto = percentualDesconto;
    }

    @Override
    public BigDecimal calcularTotal() {
        BigDecimal total = pedidoDecorado.calcularTotal();
        BigDecimal desconto = total.multiply(percentualDesconto.divide(BigDecimal.valueOf(100)));
        return total.subtract(desconto);
    }

    @Override
    public String getDescricao() {
        return pedidoDecorado.getDescricao() + " (desconto " + percentualDesconto + "%)";
    }
}

// Uso: composição em camadas
Pedido pedido = new PedidoSimples(itens);
pedido = new PedidoComFrete(pedido, BigDecimal.valueOf(15.90));
pedido = new PedidoComDesconto(pedido, BigDecimal.valueOf(10));

System.out.println(pedido.getDescricao());
// "Pedido com 3 itens + frete R$15.90 (desconto 10%)"
System.out.println(pedido.calcularTotal());
```

---

### Facade

**Problema que resolve:** fornecer uma interface simplificada para um subsistema complexo.

```java
// Subsistemas complexos (classes existentes)
public class ServicoEstoque { public boolean verificarDisponibilidade(Long produtoId, int qtd) { ... } }
public class ServicoFinanceiro { public boolean processarPagamento(String cartao, BigDecimal valor) { ... } }
public class ServicoLogistica { public String agendarEntrega(String endereco, List<Item> itens) { ... } }
public class ServicoNotificacao { public void enviarConfirmacao(String email, String codigoPedido) { ... } }

// Facade: coordena todos os subsistemas com uma interface simples
@Service
public class FachadePedido {

    private final ServicoEstoque estoque;
    private final ServicoFinanceiro financeiro;
    private final ServicoLogistica logistica;
    private final ServicoNotificacao notificacao;

    // Construtor com todas as dependências...

    // Uma chamada simples que orquestra tudo
    public ResultadoPedido realizarCompra(CompraRequest request) {
        // 1. Verifica estoque
        for (ItemRequest item : request.getItens()) {
            if (!estoque.verificarDisponibilidade(item.getProdutoId(), item.getQuantidade())) {
                throw new EstoqueInsuficienteException("Produto " + item.getProdutoId() + " indisponível");
            }
        }

        // 2. Processa pagamento
        boolean pagamentoAprovado = financeiro.processarPagamento(
                request.getCartao(),
                request.calcularTotal()
        );
        if (!pagamentoAprovado) {
            throw new PagamentoRecusadoException("Pagamento não autorizado");
        }

        // 3. Agenda entrega
        String codigoRastreio = logistica.agendarEntrega(
                request.getEnderecoEntrega(),
                request.getItens()
        );

        // 4. Notifica cliente
        notificacao.enviarConfirmacao(request.getEmail(), codigoRastreio);

        return new ResultadoPedido(codigoRastreio, "APROVADO");
    }
}

// Uso: o cliente não precisa conhecer os subsistemas
ResultadoPedido resultado = fachadePedido.realizarCompra(request);
```

---

### Proxy

**Problema que resolve:** fornecer um substituto ou representante de outro objeto para controlar o acesso a ele.

```java
// Interface comum
public interface ServicoCliente {
    Cliente buscarPorId(Long id);
}

// Implementação real
public class ServicoClienteImpl implements ServicoCliente {
    @Override
    public Cliente buscarPorId(Long id) {
        // Simulando busca cara no banco
        System.out.println("Buscando cliente " + id + " no banco...");
        return new Cliente(id, "João Silva");
    }
}

// Proxy com cache
public class ServicoClienteProxy implements ServicoCliente {
    private final ServicoCliente real;
    private final Map<Long, Cliente> cache = new HashMap<>();

    public ServicoClienteProxy(ServicoCliente real) {
        this.real = real;
    }

    @Override
    public Cliente buscarPorId(Long id) {
        if (cache.containsKey(id)) {
            System.out.println("Cliente " + id + " retornado do cache");
            return cache.get(id);
        }
        Cliente cliente = real.buscarPorId(id);
        cache.put(id, cliente);
        return cliente;
    }
}

// Uso
ServicoCliente servico = new ServicoClienteProxy(new ServicoClienteImpl());
servico.buscarPorId(1L); // Busca no banco
servico.buscarPorId(1L); // Retorna do cache
```

**No Spring:** `@Transactional`, `@Cacheable`, `@Async` são implementados com Proxy dinâmico do Spring AOP.

---

### Composite

**Problema que resolve:** tratar objetos individuais e composições de objetos de forma uniforme (estruturas de árvore).

```java
// Componente
public abstract class CategoriaMenu {
    protected String nome;
    public abstract void exibir(int nivel);
}

// Folha (sem filhos)
public class ItemMenu extends CategoriaMenu {
    private final String url;

    public ItemMenu(String nome, String url) {
        this.nome = nome;
        this.url = url;
    }

    @Override
    public void exibir(int nivel) {
        System.out.println("  ".repeat(nivel) + "- " + nome + " → " + url);
    }
}

// Composite (tem filhos)
public class GrupoMenu extends CategoriaMenu {
    private final List<CategoriaMenu> filhos = new ArrayList<>();

    public GrupoMenu(String nome) {
        this.nome = nome;
    }

    public void adicionar(CategoriaMenu item) { filhos.add(item); }

    @Override
    public void exibir(int nivel) {
        System.out.println("  ".repeat(nivel) + "▼ " + nome);
        for (CategoriaMenu filho : filhos) {
            filho.exibir(nivel + 1);
        }
    }
}

// Uso
GrupoMenu raiz = new GrupoMenu("Menu Principal");
raiz.adicionar(new ItemMenu("Home", "/"));

GrupoMenu admin = new GrupoMenu("Administração");
admin.adicionar(new ItemMenu("Usuários", "/admin/usuarios"));
admin.adicionar(new ItemMenu("Permissões", "/admin/permissoes"));

raiz.adicionar(admin);
raiz.exibir(0);
```

---

## Padrões Comportamentais

### Strategy

**Problema que resolve:** definir uma família de algoritmos, encapsular cada um e torná-los intercambiáveis sem alterar o código que os usa.

```java
// Strategy interface
public interface CalculadorFrete {
    BigDecimal calcular(Pedido pedido);
}

// Estratégias concretas
public class FreteCorreios implements CalculadorFrete {
    @Override
    public BigDecimal calcular(Pedido pedido) {
        return pedido.getPeso().multiply(BigDecimal.valueOf(2.50));
    }
}

public class FreteTransportadora implements CalculadorFrete {
    @Override
    public BigDecimal calcular(Pedido pedido) {
        return pedido.getPeso().multiply(BigDecimal.valueOf(1.80))
                .add(BigDecimal.valueOf(5.00)); // taxa fixa
    }
}

public class FreteGratis implements CalculadorFrete {
    @Override
    public BigDecimal calcular(Pedido pedido) {
        return BigDecimal.ZERO;
    }
}

// Contexto: usa a estratégia sem saber qual é
public class PedidoService {
    private CalculadorFrete calculadorFrete;

    public void setCalculadorFrete(CalculadorFrete calculadorFrete) {
        this.calculadorFrete = calculadorFrete;
    }

    public BigDecimal calcularTotal(Pedido pedido) {
        BigDecimal subtotal = pedido.calcularSubtotal();
        BigDecimal frete = calculadorFrete.calcular(pedido);
        return subtotal.add(frete);
    }
}

// Uso: troca de estratégia em runtime
PedidoService service = new PedidoService();

if (pedido.getSubtotal().compareTo(BigDecimal.valueOf(200)) > 0) {
    service.setCalculadorFrete(new FreteGratis());
} else if (pedido.isPremium()) {
    service.setCalculadorFrete(new FreteTransportadora());
} else {
    service.setCalculadorFrete(new FreteCorreios());
}
```

---

### Observer

**Problema que resolve:** definir uma dependência um-para-muitos entre objetos, de forma que quando um objeto muda de estado, todos os seus dependentes são notificados automaticamente.

```java
// Observer interface
public interface ObservadorPedido {
    void aoAtualizarStatus(Long pedidoId, String novoStatus);
}

// Subject
public class GerenciadorPedidos {
    private final List<ObservadorPedido> observadores = new ArrayList<>();

    public void adicionarObservador(ObservadorPedido obs) {
        observadores.add(obs);
    }

    public void removerObservador(ObservadorPedido obs) {
        observadores.remove(obs);
    }

    public void atualizarStatus(Long pedidoId, String novoStatus) {
        // Lógica de negócio...
        System.out.println("Status do pedido " + pedidoId + " → " + novoStatus);

        // Notifica todos os observadores
        for (ObservadorPedido obs : observadores) {
            obs.aoAtualizarStatus(pedidoId, novoStatus);
        }
    }
}

// Observadores concretos
public class NotificadorEmail implements ObservadorPedido {
    @Override
    public void aoAtualizarStatus(Long pedidoId, String novoStatus) {
        System.out.println("Email: Pedido " + pedidoId + " está " + novoStatus);
    }
}

public class AtualizadorEstoque implements ObservadorPedido {
    @Override
    public void aoAtualizarStatus(Long pedidoId, String novoStatus) {
        if ("CANCELADO".equals(novoStatus)) {
            System.out.println("Estoque: devolvendo itens do pedido " + pedidoId);
        }
    }
}

// Uso
GerenciadorPedidos gerenciador = new GerenciadorPedidos();
gerenciador.adicionarObservador(new NotificadorEmail());
gerenciador.adicionarObservador(new AtualizadorEstoque());

gerenciador.atualizarStatus(42L, "ENVIADO");
gerenciador.atualizarStatus(43L, "CANCELADO");
```

**No Spring:** `ApplicationEventPublisher` + `@EventListener` implementam Observer de forma elegante.

---

### Command

**Problema que resolve:** encapsular uma requisição como objeto, permitindo parametrizar clientes com diferentes requisições, enfileirar ou registrar requisições e suportar operações desfazíveis.

```java
// Command interface
public interface Comando {
    void executar();
    void desfazer();
}

// Comandos concretos
public class TransferenciaComando implements Comando {
    private final ContaBancaria origem;
    private final ContaBancaria destino;
    private final BigDecimal valor;

    public TransferenciaComando(ContaBancaria origem, ContaBancaria destino, BigDecimal valor) {
        this.origem = origem;
        this.destino = destino;
        this.valor = valor;
    }

    @Override
    public void executar() {
        origem.debitar(valor);
        destino.creditar(valor);
        System.out.println("Transferência de R$" + valor + " realizada");
    }

    @Override
    public void desfazer() {
        destino.debitar(valor);
        origem.creditar(valor);
        System.out.println("Transferência de R$" + valor + " desfeita");
    }
}

// Invoker: gerencia e executa comandos
public class GerenciadorTransacoes {
    private final Deque<Comando> historico = new ArrayDeque<>();

    public void executar(Comando comando) {
        comando.executar();
        historico.push(comando);
    }

    public void desfazerUltimo() {
        if (!historico.isEmpty()) {
            historico.pop().desfazer();
        }
    }
}

// Uso
GerenciadorTransacoes gerenciador = new GerenciadorTransacoes();
gerenciador.executar(new TransferenciaComando(contaA, contaB, BigDecimal.valueOf(500)));
gerenciador.desfazerUltimo(); // Reverte a transferência
```

---

### Template Method

**Problema que resolve:** definir o esqueleto de um algoritmo na classe base, deixando subclasses preencherem etapas específicas.

```java
// Classe abstrata com o esqueleto do algoritmo
public abstract class RelatorioBase {

    // Template method: define a sequência
    public final void gerar() {
        obterDados();
        processar();
        formatarSaida();
        exportar();
    }

    // Etapas que as subclasses DEVEM implementar
    protected abstract List<Object[]> obterDados();
    protected abstract void formatarSaida();

    // Etapas com implementação padrão (subclasses podem sobrescrever)
    protected void processar() {
        System.out.println("Processamento padrão dos dados...");
    }

    protected void exportar() {
        System.out.println("Exportando para PDF (padrão)...");
    }
}

// Subclasses preenchem apenas as etapas específicas
public class RelatorioVendas extends RelatorioBase {
    @Override
    protected List<Object[]> obterDados() {
        System.out.println("Buscando dados de vendas no banco...");
        return List.of(); // dados reais aqui
    }

    @Override
    protected void formatarSaida() {
        System.out.println("Formatando como tabela de vendas mensais...");
    }

    @Override
    protected void exportar() {
        System.out.println("Exportando para Excel...");
    }
}

public class RelatorioFinanceiro extends RelatorioBase {
    @Override
    protected List<Object[]> obterDados() {
        System.out.println("Buscando dados financeiros...");
        return List.of();
    }

    @Override
    protected void formatarSaida() {
        System.out.println("Formatando como balanço patrimonial...");
    }
    // exportar() usa o padrão (PDF)
}
```

---

### Chain of Responsibility

**Problema que resolve:** passar uma requisição por uma cadeia de handlers, onde cada um decide processar ou passar adiante.

```java
// Handler base
public abstract class ValidadorPedido {
    private ValidadorPedido proximo;

    public ValidadorPedido setProximo(ValidadorPedido proximo) {
        this.proximo = proximo;
        return proximo; // Permite encadeamento
    }

    public final boolean validar(Pedido pedido) {
        if (!checar(pedido)) {
            return false; // Este handler rejeitou
        }
        if (proximo != null) {
            return proximo.validar(pedido); // Passa para o próximo
        }
        return true; // Chegou ao fim da cadeia: aprovado
    }

    protected abstract boolean checar(Pedido pedido);
}

// Handlers concretos
public class ValidadorEstoque extends ValidadorPedido {
    @Override
    protected boolean checar(Pedido pedido) {
        boolean ok = pedido.getItens().stream().allMatch(item ->
                estoque.temDisponibilidade(item.getProdutoId(), item.getQuantidade())
        );
        if (!ok) System.out.println("Validação de estoque FALHOU");
        return ok;
    }
}

public class ValidadorLimiteCredito extends ValidadorPedido {
    @Override
    protected boolean checar(Pedido pedido) {
        boolean ok = pedido.calcularTotal().compareTo(pedido.getLimiteCredito()) <= 0;
        if (!ok) System.out.println("Limite de crédito excedido");
        return ok;
    }
}

public class ValidadorEndereco extends ValidadorPedido {
    @Override
    protected boolean checar(Pedido pedido) {
        boolean ok = pedido.getEnderecoEntrega() != null;
        if (!ok) System.out.println("Endereço de entrega não informado");
        return ok;
    }
}

// Uso: montando a cadeia
ValidadorPedido cadeia = new ValidadorEstoque();
cadeia.setProximo(new ValidadorLimiteCredito())
      .setProximo(new ValidadorEndereco());

boolean valido = cadeia.validar(pedido);
```
