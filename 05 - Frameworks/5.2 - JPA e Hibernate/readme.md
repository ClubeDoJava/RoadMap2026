# 5.2 — JPA e Hibernate

JPA (Jakarta Persistence API) é uma especificação Java para mapeamento objeto-relacional (ORM). O Hibernate é a implementação mais usada. Spring Data JPA adiciona uma camada de abstração sobre o JPA, eliminando muito código repetitivo.

---

## O que é ORM e por que usar

**Sem ORM:** você escreve SQL manualmente, converte ResultSet em objetos, gerencia transações.

**Com ORM:** você trabalha com objetos Java, o framework gera o SQL.

```java
// Sem ORM (JDBC)
String sql = "INSERT INTO produtos (nome, preco) VALUES (?, ?)";
PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setString(1, produto.getNome());
stmt.setDouble(2, produto.getPreco());
stmt.executeUpdate();

// Com JPA
repository.save(produto); // uma linha
```

**Quando JPA é desvantagem:** queries muito complexas, tuning de performance avançado, batch inserts de milhões de registros. Nesses casos, JDBC ou `@Query` com SQL nativo são melhores.

---

## Entidades

### Mapeamento básico

```java
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "produtos")
public class Produto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // auto-increment do banco
    private Long id;

    @Column(name = "nome_produto", nullable = false, length = 100)
    private String nome;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal preco;

    @Column(name = "em_estoque")
    private boolean emEstoque = true;

    @Transient  // campo não persistido
    private String nomeMaiusculo;

    @Enumerated(EnumType.STRING) // salva o nome do enum ("ATIVO"), não o ordinal
    @Column(nullable = false)
    private StatusProduto status;

    @Lob  // campo grande (texto longo, binário)
    @Column(columnDefinition = "TEXT")
    private String descricaoCompleta;

    @Column(name = "criado_em", updatable = false)
    private LocalDateTime criadoEm;

    // construtores, getters, setters...
}

public enum StatusProduto {
    ATIVO, INATIVO, DESCONTINUADO
}
```

### Tipos de `GenerationType`

```java
// IDENTITY — usa auto-increment do banco (PostgreSQL, MySQL)
// Mais simples, mas impede batch inserts
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// SEQUENCE — usa sequence do banco (PostgreSQL)
// Permite batch inserts, melhor performance
@GeneratedValue(strategy = GenerationType.SEQUENCE,
                generator = "produto_seq")
@SequenceGenerator(name = "produto_seq",
                   sequenceName = "produto_id_seq",
                   allocationSize = 50) // reserva 50 ids por vez
private Long id;

// UUID — identificador universal
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

---

## Relacionamentos

### `@OneToMany` e `@ManyToOne`

O lado do `@ManyToOne` é quem contém a FK no banco.

```java
@Entity
@Table(name = "categorias")
public class Categoria {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nome;

    // Uma categoria tem muitos produtos
    // mappedBy aponta para o campo em Produto que representa o lado dono
    @OneToMany(mappedBy = "categoria", cascade = CascadeType.ALL,
               fetch = FetchType.LAZY, orphanRemoval = true)
    private List<Produto> produtos = new ArrayList<>();
}

@Entity
@Table(name = "produtos")
public class Produto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nome;

    // Muitos produtos pertencem a uma categoria (lado dono — tem a FK)
    @ManyToOne(fetch = FetchType.LAZY) // LAZY: carrega somente quando acessado
    @JoinColumn(name = "categoria_id", nullable = false)
    private Categoria categoria;
}
```

### `@OneToOne`

```java
@Entity
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(mappedBy = "usuario", cascade = CascadeType.ALL,
              fetch = FetchType.LAZY, optional = false)
    private Perfil perfil;
}

@Entity
public class Perfil {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String bio;
    private String avatar;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "usuario_id", unique = true, nullable = false)
    private Usuario usuario;
}
```

### `@ManyToMany`

```java
@Entity
public class Pedido {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "pedido_produto",
        joinColumns = @JoinColumn(name = "pedido_id"),
        inverseJoinColumns = @JoinColumn(name = "produto_id")
    )
    private Set<Produto> produtos = new HashSet<>();
}
```

### `FetchType.LAZY` vs `FetchType.EAGER`

```java
// LAZY (padrão para coleções) — carrega somente quando acessado
// RECOMENDADO: não carrega dados desnecessários
@OneToMany(fetch = FetchType.LAZY)
private List<Item> itens;

// EAGER (padrão para @ManyToOne e @OneToOne) — carrega sempre junto
// CUIDADO: pode causar queries desnecessárias e problema N+1
@ManyToOne(fetch = FetchType.EAGER) // evite sobrescrever para EAGER
private Categoria categoria;
```

### `CascadeType`

```java
// PERSIST — salva filhos ao salvar o pai
// MERGE — atualiza filhos ao atualizar o pai
// REMOVE — deleta filhos ao deletar o pai
// ALL — todos acima (use com cuidado)
@OneToMany(mappedBy = "pedido",
           cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private List<ItemPedido> itens;
```

---

## Problema N+1

N+1 é quando você faz 1 query para buscar N entidades, e depois N queries adicionais para buscar os relacionamentos.

```java
// Cenário: buscar todos os pedidos e exibir os itens

// SEM N+1 control — dispara 1 + N queries (um por pedido)
List<Pedido> pedidos = repository.findAll();
for (Pedido p : pedidos) {
    // Aqui dispara uma query para cada pedido!
    System.out.println(p.getItens().size());
}
```

### Solução 1: `JOIN FETCH` na query

```java
@Repository
public interface PedidoRepository extends JpaRepository<Pedido, Long> {

    @Query("SELECT p FROM Pedido p JOIN FETCH p.itens WHERE p.status = :status")
    List<Pedido> findByStatusComItens(@Param("status") String status);
}
```

### Solução 2: `@EntityGraph`

```java
@Repository
public interface PedidoRepository extends JpaRepository<Pedido, Long> {

    @EntityGraph(attributePaths = {"itens", "cliente"})
    List<Pedido> findAll(); // carrega itens e cliente junto

    @EntityGraph(attributePaths = "itens")
    Optional<Pedido> findById(Long id);
}
```

---

## `@Embeddable` e `@Embedded` — Value Objects

```java
@Embeddable  // não é uma entidade, é parte de outra
public class Endereco {

    @Column(name = "logradouro", length = 200)
    private String logradouro;

    @Column(name = "numero", length = 10)
    private String numero;

    @Column(name = "cep", length = 9)
    private String cep;

    @Column(name = "cidade", length = 100)
    private String cidade;

    @Column(name = "estado", length = 2)
    private String estado;
}

@Entity
public class Cliente {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nome;

    @Embedded
    private Endereco enderecoEntrega;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "logradouro", column = @Column(name = "fat_logradouro")),
        @AttributeOverride(name = "numero", column = @Column(name = "fat_numero")),
        @AttributeOverride(name = "cep", column = @Column(name = "fat_cep"))
    })
    private Endereco enderecoFaturamento;
}
```

---

## Herança

```java
// SINGLE_TABLE — todas as subclasses em uma tabela (mais performático, colunas nullable)
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "tipo", discriminatorType = DiscriminatorType.STRING)
public abstract class Pagamento {
    @Id @GeneratedValue private Long id;
    private BigDecimal valor;
}

@Entity
@DiscriminatorValue("CARTAO")
public class PagamentoCartao extends Pagamento {
    private String numeroCartao;
    private int parcelas;
}

@Entity
@DiscriminatorValue("PIX")
public class PagamentoPix extends Pagamento {
    private String chavePix;
}

// JOINED — cada classe em sua tabela (normalizado, mais queries)
// TABLE_PER_CLASS — cada classe com todos os campos (sem joins, mas duplicação de schema)
```

---

## Spring Data JPA

### `JpaRepository`

```java
@Repository
public interface ProdutoRepository extends JpaRepository<Produto, Long> {
    // JpaRepository já fornece:
    // save, saveAll, findById, findAll, findAllById
    // deleteById, delete, deleteAll
    // existsById, count
    // flush, saveAndFlush
}
```

### Query Methods — convenção de nomes

```java
@Repository
public interface ProdutoRepository extends JpaRepository<Produto, Long> {

    // Básicos
    Optional<Produto> findByNome(String nome);
    List<Produto> findByCategoria(String categoria);
    boolean existsByEmail(String email);
    long countByStatus(StatusProduto status);

    // Comparações
    List<Produto> findByPrecoGreaterThan(BigDecimal preco);
    List<Produto> findByPrecoBetween(BigDecimal min, BigDecimal max);
    List<Produto> findByEstoqueGreaterThanEqual(int estoque);

    // Like e Containing
    List<Produto> findByNomeContainingIgnoreCase(String termo);
    List<Produto> findByNomeStartingWith(String prefixo);

    // Múltiplos campos
    List<Produto> findByCategoriaAndStatus(String categoria, StatusProduto status);
    List<Produto> findByNomeOrDescricaoContaining(String nome, String descricao);

    // Ordenação
    List<Produto> findByCategoriaOrderByPrecoAsc(String categoria);
    List<Produto> findAllByOrderByCriadoEmDesc();

    // Nulos
    List<Produto> findByDescricaoIsNull();
    List<Produto> findByDescricaoIsNotNull();
}
```

### `@Query` com JPQL e SQL nativo

```java
@Repository
public interface ProdutoRepository extends JpaRepository<Produto, Long> {

    // JPQL — usa nomes das classes e campos Java, não do banco
    @Query("SELECT p FROM Produto p WHERE p.preco > :precoMin AND p.status = 'ATIVO'")
    List<Produto> findAtivosComPrecoMaiorQue(@Param("precoMin") BigDecimal precoMin);

    // JPQL com JOIN
    @Query("SELECT p FROM Produto p JOIN FETCH p.categoria c WHERE c.nome = :nomeCategoria")
    List<Produto> findPorCategoriaNome(@Param("nomeCategoria") String nomeCategoria);

    // SQL nativo (nativeQuery = true) — usa nomes reais do banco
    @Query(value = "SELECT * FROM produtos WHERE LOWER(nome_produto) LIKE %:termo%",
           nativeQuery = true)
    List<Produto> buscarPorTermoNativo(@Param("termo") String termo);

    // Modifying + Transactional para UPDATE/DELETE
    @Modifying
    @Transactional
    @Query("UPDATE Produto p SET p.status = :status WHERE p.id = :id")
    int atualizarStatus(@Param("id") Long id, @Param("status") StatusProduto status);
}
```

### Paginação

```java
@Repository
public interface ProdutoRepository extends JpaRepository<Produto, Long> {

    // Pageable aceita página, tamanho e ordenação
    Page<Produto> findByCategoria(String categoria, Pageable pageable);
    Page<Produto> findByStatus(StatusProduto status, Pageable pageable);
}

// No service
public Page<ProdutoResponse> listar(int pagina, int tamanho, String ordenacao) {
    Pageable pageable = PageRequest.of(pagina, tamanho, Sort.by(ordenacao).ascending());
    return repository.findAll(pageable).map(ProdutoResponse::de);
}

// Resposta da Page
Page<ProdutoResponse> pagina = service.listar(0, 10, "nome");
pagina.getContent();        // List<ProdutoResponse>
pagina.getTotalElements();  // total de registros
pagina.getTotalPages();     // total de páginas
pagina.getNumber();         // página atual (0-indexed)
pagina.isFirst();           // é a primeira página?
pagina.isLast();            // é a última?
pagina.hasNext();           // tem próxima?
```

### Auditoria

```java
// Habilitar na aplicação principal
@SpringBootApplication
@EnableJpaAuditing
public class AppApplication {}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(name = "criado_em", updatable = false)
    private LocalDateTime criadoEm;

    @LastModifiedDate
    @Column(name = "atualizado_em")
    private LocalDateTime atualizadoEm;

    @CreatedBy
    @Column(name = "criado_por", updatable = false)
    private String criadoPor;

    @LastModifiedBy
    @Column(name = "atualizado_por")
    private String atualizadoPor;
}

@Entity
public class Produto extends BaseEntity {
    // herda os campos de auditoria
}
```

---

## `@Transactional`

```java
@Service
public class PedidoService {

    // REQUIRED (padrão): usa transação existente ou cria uma nova
    @Transactional
    public Pedido criar(PedidoRequest request) {
        Pedido pedido = new Pedido(request);
        repository.save(pedido);
        estoqueService.reduzir(request.itens()); // participa da mesma transação
        return pedido;
    }

    // REQUIRES_NEW: sempre cria nova transação independente
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void registrarAuditoria(String evento) {
        // salvo mesmo se a transação principal fizer rollback
        auditoriaRepository.save(new LogAuditoria(evento));
    }

    // readOnly: otimização para queries (sem dirty checking)
    @Transactional(readOnly = true)
    public List<Pedido> listarPorCliente(Long clienteId) {
        return repository.findByClienteId(clienteId);
    }

    // rollbackFor: por padrão só faz rollback em RuntimeException
    @Transactional(rollbackFor = Exception.class)
    public void processarComCheckedExceptions() throws IOException {
        // faz rollback também para IOException
    }
}
```

---

## Flyway — Migrations

### Por que usar migrations?

Sem migrations, você precisa executar SQL manualmente em cada ambiente (dev, staging, produção). Com Flyway, as alterações no banco são versionadas junto com o código.

### Dependência

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

### Nomenclatura obrigatória

```
V{versão}__{descrição}.sql

V1__create_users_table.sql
V2__add_email_column.sql
V3__create_products_table.sql
V4__add_index_products_category.sql
```

> O duplo underscore (`__`) é obrigatório. A versão pode ser `1`, `1.1`, `2026.03.09.01`.

### Localização dos scripts

```
src/main/resources/db/migration/
├── V1__create_schema.sql
├── V2__insert_initial_data.sql
└── V3__add_audit_columns.sql
```

### Exemplos de migrations

```sql
-- V1__create_schema.sql
CREATE TABLE categorias (
    id          BIGSERIAL PRIMARY KEY,
    nome        VARCHAR(100) NOT NULL UNIQUE,
    criado_em   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE produtos (
    id              BIGSERIAL PRIMARY KEY,
    nome_produto    VARCHAR(100) NOT NULL,
    preco           DECIMAL(10, 2) NOT NULL CHECK (preco > 0),
    status          VARCHAR(20) NOT NULL DEFAULT 'ATIVO',
    categoria_id    BIGINT REFERENCES categorias(id),
    criado_em       TIMESTAMPTZ DEFAULT NOW(),
    atualizado_em   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_produtos_categoria ON produtos(categoria_id);
CREATE INDEX idx_produtos_status ON produtos(status);
```

```sql
-- V2__add_descricao_column.sql
ALTER TABLE produtos ADD COLUMN descricao TEXT;
```

### Configuração no `application.yml`

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false    # true para migrar banco já existente
    out-of-order: false           # não permite migrations fora de ordem
    validate-on-migrate: true     # valida checksum dos scripts
```

---

## Exemplo Completo — Entidade Produto com Relacionamentos, Repositório e Paginação

```java
// Categoria.java
@Entity
@Table(name = "categorias")
public class Categoria {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String nome;

    @OneToMany(mappedBy = "categoria", fetch = FetchType.LAZY)
    private List<Produto> produtos = new ArrayList<>();
}

// Produto.java
@Entity
@Table(name = "produtos")
@EntityListeners(AuditingEntityListener.class)
public class Produto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "nome_produto", nullable = false, length = 100)
    private String nome;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal preco;

    @Column(nullable = false, length = 20)
    @Enumerated(EnumType.STRING)
    private StatusProduto status = StatusProduto.ATIVO;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "categoria_id")
    private Categoria categoria;

    @CreatedDate
    @Column(name = "criado_em", updatable = false)
    private LocalDateTime criadoEm;

    @LastModifiedDate
    @Column(name = "atualizado_em")
    private LocalDateTime atualizadoEm;
}

// ProdutoRepository.java
@Repository
public interface ProdutoRepository extends JpaRepository<Produto, Long> {

    Page<Produto> findByStatus(StatusProduto status, Pageable pageable);

    @EntityGraph(attributePaths = "categoria")
    Page<Produto> findByCategoriaNome(String nomeCategoria, Pageable pageable);

    @Query("SELECT p FROM Produto p JOIN FETCH p.categoria WHERE p.preco BETWEEN :min AND :max")
    List<Produto> findByFaixaDePreco(@Param("min") BigDecimal min, @Param("max") BigDecimal max);
}

// ProdutoService.java
@Service
@Transactional(readOnly = true)
public class ProdutoService {

    private final ProdutoRepository repository;
    private final CategoriaRepository categoriaRepository;

    public ProdutoService(ProdutoRepository repository,
                          CategoriaRepository categoriaRepository) {
        this.repository = repository;
        this.categoriaRepository = categoriaRepository;
    }

    public Page<ProdutoResponse> listar(int pagina, int tamanho) {
        Pageable pageable = PageRequest.of(pagina, tamanho, Sort.by("nome"));
        return repository.findAll(pageable).map(ProdutoResponse::de);
    }

    @Transactional
    public ProdutoResponse criar(ProdutoRequest request) {
        Categoria categoria = categoriaRepository.findById(request.categoriaId())
                .orElseThrow(() -> new CategoriaNaoEncontradaException(request.categoriaId()));

        Produto produto = new Produto();
        produto.setNome(request.nome());
        produto.setPreco(request.preco());
        produto.setCategoria(categoria);

        return ProdutoResponse.de(repository.save(produto));
    }

    @Transactional
    public void desativar(Long id) {
        Produto produto = repository.findById(id)
                .orElseThrow(() -> new ProdutoNaoEncontradoException(id));
        produto.setStatus(StatusProduto.INATIVO);
        // sem necessidade de chamar save() — JPA detecta mudança na entidade gerenciada
    }
}
```
