# 7.2 - Arquitetura de Software

## Monolito vs Microsserviços: trade-offs reais

### Monolito

Uma única aplicação que contém todos os módulos de negócio deployada como uma unidade.

```
┌──────────────────────────────────────────────┐
│              MONOLITO                        │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Módulo   │  │ Módulo   │  │ Módulo   │  │
│  │ Pedidos  │  │ Estoque  │  │ Usuários │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│         │            │            │          │
│         └────────────┼────────────┘          │
│                      │                       │
│              ┌───────────────┐               │
│              │  Banco único  │               │
│              └───────────────┘               │
└──────────────────────────────────────────────┘
```

**Vantagens:**
- Simples de desenvolver no início
- Deploy simples: um JAR, uma configuração
- Sem complexidade de rede entre módulos
- Transações ACID entre módulos
- Fácil de debugar e testar localmente
- Menor overhead de infraestrutura

**Desvantagens:**
- Com o tempo, tende a se tornar "Big Ball of Mud"
- Um bug pode derrubar tudo
- Escala toda a aplicação, mesmo que só um módulo precise
- Deploy de uma funcionalidade requer redeployar tudo
- Adoção tecnológica inflexível (todos usam a mesma linguagem/versão)

### Microsserviços

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Serviço  │     │ Serviço  │     │ Serviço  │
│ Pedidos  │────>│ Estoque  │     │ Usuários │
│  :8081   │     │  :8082   │     │  :8083   │
│  DB-1    │     │  DB-2    │     │  DB-3    │
└──────────┘     └──────────┘     └──────────┘
      │                │                │
      └────────────────┼────────────────┘
                       │
                ┌──────────────┐
                │  API Gateway │
                └──────────────┘
```

**Vantagens:**
- Escala independente por serviço
- Deploy independente
- Cada serviço pode usar a tecnologia mais adequada
- Falha isolada: um serviço caindo não afeta os outros
- Times autônomos por serviço

**Desvantagens:**
- Complexidade de rede (latência, falhas, retries)
- Sem transações ACID entre serviços (eventual consistency)
- Infraestrutura muito mais complexa
- Debugging e tracing distribuído difícil
- Overhead operacional enorme para times pequenos

### Quando escolher cada um

| Situação | Recomendação |
|----------|-------------|
| Startup, time pequeno (< 5 devs) | Monolito |
| MVP, prazo curto | Monolito |
| Domínio ainda não bem entendido | Monolito |
| Time grande, domínios bem definidos | Microsserviços |
| Requisitos de escala muito diferentes por módulo | Microsserviços |
| Necessidade de deploys frequentes e independentes | Microsserviços |

**Regra prática:** comece com monolito bem estruturado. Extraia microsserviços apenas quando sentir a dor real.

---

## DDD — Domain-Driven Design

DDD é uma abordagem de design que coloca o **domínio do negócio** no centro das decisões de software.

### Conceitos fundamentais

**Ubiquitous Language (Linguagem Ubíqua):**
Vocabulário compartilhado entre desenvolvedores e especialistas do domínio. O código usa os mesmos termos que o negócio usa.

```java
// Ruim: terminologia técnica misturada com negócio
public class UserOrderRecord {
    private int userId;
    private List<OrderItem> items;
    private double totalAmount;
}

// Bom: linguagem do domínio (Pedido, Cliente, ValorTotal)
public class Pedido {
    private ClienteId clienteId;
    private List<ItemPedido> itens;
    private Dinheiro valorTotal;
}
```

**Entity (Entidade):**
Objeto com identidade única que persiste ao longo do tempo. Dois objetos são a mesma entidade se têm o mesmo ID, mesmo com atributos diferentes.

```java
public class Pedido {
    private final PedidoId id;      // Identidade imutável
    private StatusPedido status;    // Muda ao longo do tempo
    private List<ItemPedido> itens;

    // Igualdade baseada apenas no ID
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Pedido pedido)) return false;
        return Objects.equals(id, pedido.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

**Value Object (Objeto de Valor):**
Objeto sem identidade, definido apenas pelos seus atributos. Imutável. Dois Value Objects são iguais se todos os atributos são iguais.

```java
// Value Object: imutável, sem ID próprio
public record Dinheiro(BigDecimal valor, String moeda) {

    public Dinheiro {
        if (valor.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Valor monetário não pode ser negativo");
        }
        if (moeda == null || moeda.length() != 3) {
            throw new IllegalArgumentException("Moeda deve ter 3 caracteres (ex: BRL)");
        }
    }

    public Dinheiro somar(Dinheiro outro) {
        if (!this.moeda.equals(outro.moeda)) {
            throw new IllegalArgumentException("Não é possível somar moedas diferentes");
        }
        return new Dinheiro(this.valor.add(outro.valor), this.moeda);
    }

    public Dinheiro aplicarDesconto(BigDecimal percentual) {
        BigDecimal desconto = valor.multiply(percentual).divide(BigDecimal.valueOf(100));
        return new Dinheiro(valor.subtract(desconto), moeda);
    }
}

// Uso
Dinheiro preco = new Dinheiro(new BigDecimal("99.90"), "BRL");
Dinheiro comDesconto = preco.aplicarDesconto(new BigDecimal("10"));
```

**Aggregate (Agregado):**
Cluster de entidades e Value Objects tratados como uma unidade para mudanças de dados. Tem uma **Aggregate Root** (raiz) que é o único ponto de entrada.

```java
// Aggregate Root: Pedido controla seus itens
public class Pedido {
    private final PedidoId id;
    private final ClienteId clienteId;
    private final List<ItemPedido> itens = new ArrayList<>(); // Itens são parte do agregado
    private StatusPedido status = StatusPedido.RASCUNHO;

    // Toda modificação passa pela raiz
    public void adicionarItem(ProdutoId produtoId, int quantidade, Dinheiro preco) {
        if (status != StatusPedido.RASCUNHO) {
            throw new IllegalStateException("Não é possível modificar pedido " + status);
        }
        itens.add(new ItemPedido(produtoId, quantidade, preco));
    }

    public void confirmar() {
        if (itens.isEmpty()) {
            throw new IllegalStateException("Pedido não pode ser confirmado sem itens");
        }
        this.status = StatusPedido.CONFIRMADO;
        // Registra domain event
        DomainEvents.raise(new PedidoConfirmado(this.id));
    }

    public Dinheiro calcularTotal() {
        return itens.stream()
                .map(ItemPedido::calcularSubtotal)
                .reduce(new Dinheiro(BigDecimal.ZERO, "BRL"), Dinheiro::somar);
    }
}
```

**Repository (Repositório):**
Abstrai o acesso ao armazenamento. O domínio define a interface, a infraestrutura a implementa.

```java
// Interface no domínio (sem JPA, sem SQL)
public interface PedidoRepository {
    void salvar(Pedido pedido);
    Optional<Pedido> buscarPorId(PedidoId id);
    List<Pedido> buscarPorCliente(ClienteId clienteId);
    List<Pedido> buscarPendentes();
}

// Implementação na infraestrutura (JPA)
@Repository
public class PedidoRepositoryJpa implements PedidoRepository {
    private final PedidoJpaRepository jpaRepository;

    @Override
    public Optional<Pedido> buscarPorId(PedidoId id) {
        return jpaRepository.findById(id.valor())
                .map(PedidoMapper::toDomain);
    }
    // ...
}
```

**Domain Service (Serviço de Domínio):**
Lógica de negócio que não pertence naturalmente a nenhuma entidade específica.

```java
// Lógica que envolve múltiplos agregados
public class ServicoTransferencia {

    public void transferir(Conta origem, Conta destino, Dinheiro valor) {
        if (!origem.temSaldo(valor)) {
            throw new SaldoInsuficienteException(origem.getId());
        }
        origem.debitar(valor);
        destino.creditar(valor);
    }
}
```

**Bounded Context (Contexto Delimitado):**
Limite explícito dentro do qual um modelo de domínio específico se aplica. "Produto" pode significar coisas diferentes em contextos diferentes.

```
┌─────────────────────┐    ┌─────────────────────┐
│  Contexto: Catálogo │    │  Contexto: Estoque   │
│                     │    │                     │
│  Produto            │    │  Produto            │
│  - nome             │    │  - codigo           │
│  - descricao        │    │  - quantidade       │
│  - fotos            │    │  - localizacao      │
│  - preco            │    │  - nivelMinimo      │
└─────────────────────┘    └─────────────────────┘
```

---

## Arquitetura Hexagonal (Ports & Adapters)

Criada por Alistair Cockburn. O objetivo é **isolar o núcleo de negócio** de todos os detalhes técnicos (banco, HTTP, filas, etc.).

```
                    ┌─────────────────────────────────┐
   Driver           │         APPLICATION CORE         │          Driven
   (Primário)       │                                 │          (Secundário)
                    │   ┌─────────────────────────┐   │
  ┌─────────┐       │   │                         │   │       ┌─────────────┐
  │  REST   │──────>│   │    DOMAIN (negócio)     │   │──────>│  Banco JPA  │
  │ Controller│ Port│   │    Entities             │   │ Port  │             │
  └─────────┘       │   │    Services             │   │       └─────────────┘
                    │   │    Value Objects         │   │
  ┌─────────┐       │   │                         │   │       ┌─────────────┐
  │  gRPC   │──────>│   └─────────────────────────┘   │──────>│  Kafka      │
  └─────────┘       │                                 │       └─────────────┘
                    │                                 │
  ┌─────────┐       │                                 │       ┌─────────────┐
  │  CLI    │──────>│                                 │──────>│ Email SMTP  │
  └─────────┘       │                                 │       └─────────────┘
                    └─────────────────────────────────┘
```

### Como implementar em Java

**Estrutura de pacotes:**

```
src/main/java/com/minhaapp/
├── domain/                          # Núcleo: sem dependências externas
│   ├── model/
│   │   ├── Pedido.java
│   │   ├── ItemPedido.java
│   │   └── Dinheiro.java
│   ├── port/
│   │   ├── in/                      # Ports de entrada (use cases)
│   │   │   ├── CriarPedidoUseCase.java
│   │   │   └── ConfirmarPedidoUseCase.java
│   │   └── out/                     # Ports de saída
│   │       ├── PedidoRepository.java
│   │       └── NotificacaoPort.java
│   └── service/                     # Application services
│       └── PedidoService.java
│
└── infrastructure/                  # Adaptadores: detalhes técnicos
    ├── web/                         # Adaptador HTTP
    │   └── PedidoController.java
    ├── persistence/                 # Adaptador JPA
    │   ├── PedidoRepositoryAdapter.java
    │   └── PedidoJpaRepository.java
    └── messaging/                   # Adaptador Kafka
        └── PedidoEventPublisher.java
```

**Port de entrada (use case):**

```java
// domain/port/in/CriarPedidoUseCase.java
public interface CriarPedidoUseCase {
    PedidoId executar(CriarPedidoCommand command);

    record CriarPedidoCommand(
        String clienteId,
        List<ItemCommand> itens,
        String enderecoEntrega
    ) {}
}
```

**Serviço de domínio (implementa o use case):**

```java
// domain/service/PedidoService.java
@Service
@Transactional
public class PedidoService implements CriarPedidoUseCase {

    private final PedidoRepository repository;       // Port de saída
    private final NotificacaoPort notificacao;        // Port de saída

    public PedidoService(PedidoRepository repository, NotificacaoPort notificacao) {
        this.repository = repository;
        this.notificacao = notificacao;
    }

    @Override
    public PedidoId executar(CriarPedidoCommand command) {
        Pedido pedido = new Pedido(
                PedidoId.gerar(),
                new ClienteId(command.clienteId())
        );

        for (ItemCommand item : command.itens()) {
            pedido.adicionarItem(
                    new ProdutoId(item.produtoId()),
                    item.quantidade(),
                    new Dinheiro(item.preco(), "BRL")
            );
        }

        repository.salvar(pedido);
        notificacao.notificarCriacaoPedido(pedido.getId());

        return pedido.getId();
    }
}
```

**Adaptador de entrada (Controller HTTP):**

```java
// infrastructure/web/PedidoController.java
@RestController
@RequestMapping("/api/pedidos")
public class PedidoController {

    private final CriarPedidoUseCase criarPedido;

    public PedidoController(CriarPedidoUseCase criarPedido) {
        this.criarPedido = criarPedido;
    }

    @PostMapping
    public ResponseEntity<Map<String, String>> criar(@RequestBody @Valid PedidoRequest request) {
        // Converte DTO HTTP → Command do domínio
        CriarPedidoCommand command = new CriarPedidoCommand(
                request.clienteId(),
                request.itens().stream()
                        .map(i -> new ItemCommand(i.produtoId(), i.quantidade(), i.preco()))
                        .toList(),
                request.enderecoEntrega()
        );

        PedidoId id = criarPedido.executar(command);

        return ResponseEntity.status(201).body(Map.of("id", id.valor()));
    }
}
```

**Adaptador de saída (Repositório JPA):**

```java
// infrastructure/persistence/PedidoRepositoryAdapter.java
@Component
public class PedidoRepositoryAdapter implements PedidoRepository {

    private final PedidoJpaRepository jpaRepository;
    private final PedidoMapper mapper;

    @Override
    public void salvar(Pedido pedido) {
        PedidoEntity entity = mapper.toEntity(pedido);
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Pedido> buscarPorId(PedidoId id) {
        return jpaRepository.findById(id.valor())
                .map(mapper::toDomain);
    }
}
```

---

## Clean Architecture

Robert C. Martin (Uncle Bob) formalizou uma versão com camadas concêntricas:

```
        ┌─────────────────────────────────────┐
        │         Frameworks & Drivers         │ ← Web, DB, UI
        │   ┌─────────────────────────────┐   │
        │   │      Interface Adapters      │   │ ← Controllers, Presenters
        │   │   ┌─────────────────────┐   │   │
        │   │   │   Application BL    │   │   │ ← Use Cases
        │   │   │   ┌─────────────┐   │   │   │
        │   │   │   │   Entities  │   │   │   │ ← Enterprise Rules
        │   │   │   └─────────────┘   │   │   │
        │   │   └─────────────────────┘   │   │
        │   └─────────────────────────────┘   │
        └─────────────────────────────────────┘
```

**A Regra da Dependência:** as dependências no código-fonte devem apontar **somente para dentro**. Camadas internas não sabem nada sobre camadas externas.

- Entities (centro): regras de negócio da empresa — não mudam com tecnologia
- Use Cases: regras de negócio da aplicação
- Interface Adapters: convertem dados entre use cases e camada externa
- Frameworks: Spring, JPA, REST — detalhes que podem mudar

---

## Event-Driven Architecture

Em vez de chamadas síncronas entre serviços, usamos eventos para comunicação assíncrona.

```java
// Evento de domínio
public record PedidoConfirmado(
    PedidoId pedidoId,
    ClienteId clienteId,
    Dinheiro valorTotal,
    Instant ocorridoEm
) {}

// Publisher: domínio publica eventos sem saber quem escuta
@Component
public class PedidoEventPublisher implements ApplicationEventPublisher {
    private final ApplicationEventPublisher publisher;

    public void publicar(PedidoConfirmado evento) {
        publisher.publishEvent(evento);
    }
}

// Subscriber: múltiplos serviços reagem ao evento
@Component
public class NotificacaoListener {
    @EventListener
    @Async
    public void aoConfirmarPedido(PedidoConfirmado evento) {
        emailService.enviarConfirmacao(evento.clienteId(), evento.pedidoId());
    }
}

@Component
public class EstoqueListener {
    @EventListener
    @Async
    public void aoConfirmarPedido(PedidoConfirmado evento) {
        estoqueService.reservarItens(evento.pedidoId());
    }
}
```

---

## Como refatorar um monolito para microsserviço

Este processo é chamado de **Strangler Fig Pattern** (como a figueira estranguladora que cresce ao redor de uma árvore).

**Passo a passo:**

1. **Entenda o monolito** — identifique os bounded contexts e dependências
2. **Escolha o módulo com menos acoplamento** para extrair primeiro
3. **Crie uma interface limpa** entre o módulo escolhido e o resto
4. **Adicione um Anti-Corruption Layer** se necessário
5. **Crie o novo microsserviço** com seu próprio banco
6. **Roteie o tráfego** gradualmente (API Gateway: 10% novo → 100%)
7. **Remova o código** do monolito após validação
8. **Repita** para o próximo módulo

```java
// Fase 1: Facade no monolito que pode redirecionar
@Service
public class EstoqueFacade {

    @Value("${feature.microsservico-estoque.ativo:false}")
    private boolean microsservicoAtivo;

    private final EstoqueServiceLegado legado;          // Monolito
    private final EstoqueMicrosservicoClient externo;   // Novo microsserviço

    public boolean verificarDisponibilidade(Long produtoId, int quantidade) {
        if (microsservicoAtivo) {
            return externo.verificar(produtoId, quantidade);
        }
        return legado.verificar(produtoId, quantidade);
    }
}
```

---

## Exemplo completo: estrutura de pacotes hexagonal

```
minha-app/
├── src/main/java/com/minhaapp/
│   │
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Pedido.java               # Aggregate Root
│   │   │   ├── ItemPedido.java           # Entity dentro do Aggregate
│   │   │   ├── PedidoId.java             # Value Object (ID)
│   │   │   ├── ClienteId.java
│   │   │   ├── ProdutoId.java
│   │   │   ├── Dinheiro.java             # Value Object
│   │   │   └── StatusPedido.java         # Enum do domínio
│   │   │
│   │   ├── event/
│   │   │   └── PedidoConfirmado.java     # Domain Event
│   │   │
│   │   ├── exception/
│   │   │   ├── PedidoNaoEncontradoException.java
│   │   │   └── SaldoInsuficienteException.java
│   │   │
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   ├── CriarPedidoUseCase.java
│   │   │   │   ├── ConfirmarPedidoUseCase.java
│   │   │   │   └── BuscarPedidoUseCase.java
│   │   │   └── out/
│   │   │       ├── PedidoRepository.java
│   │   │       ├── ProdutoRepository.java
│   │   │       └── NotificacaoPort.java
│   │   │
│   │   └── service/
│   │       ├── PedidoService.java
│   │       └── EstoqueService.java
│   │
│   ├── application/
│   │   └── config/
│   │       └── BeanConfig.java           # @Configuration do Spring
│   │
│   └── infrastructure/
│       ├── web/
│       │   ├── controller/
│       │   │   └── PedidoController.java
│       │   ├── dto/
│       │   │   ├── PedidoRequest.java
│       │   │   └── PedidoResponse.java
│       │   ├── mapper/
│       │   │   └── PedidoWebMapper.java
│       │   └── exception/
│       │       └── GlobalExceptionHandler.java
│       │
│       ├── persistence/
│       │   ├── adapter/
│       │   │   └── PedidoRepositoryAdapter.java
│       │   ├── entity/
│       │   │   └── PedidoEntity.java
│       │   ├── jpa/
│       │   │   └── PedidoJpaRepository.java
│       │   └── mapper/
│       │       └── PedidoJpaMapper.java
│       │
│       └── messaging/
│           ├── kafka/
│           │   └── PedidoKafkaPublisher.java
│           └── email/
│               └── EmailNotificacaoAdapter.java
│
└── src/test/java/com/minhaapp/
    ├── domain/
    │   └── PedidoTest.java              # Testes unitários do domínio (sem Spring)
    ├── application/
    │   └── PedidoServiceTest.java       # Testes de use case (mock dos ports)
    └── infrastructure/
        ├── PedidoControllerTest.java    # Testes de integração da API
        └── PedidoRepositoryTest.java    # Testes do repositório JPA
```
