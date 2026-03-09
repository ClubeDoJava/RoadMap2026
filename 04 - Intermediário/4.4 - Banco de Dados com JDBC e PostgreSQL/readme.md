# 4.4 — Banco de Dados com JDBC e PostgreSQL

JDBC (Java Database Connectivity) é a API padrão do Java para comunicar com bancos de dados relacionais. Mesmo que você use JPA ou Spring Data no dia a dia, entender JDBC é fundamental para compreender o que acontece por baixo dos frameworks.

---

## Conceitos JDBC

| Componente | Papel |
|---|---|
| `Driver` | Implementação específica do banco (PostgreSQL, MySQL, etc.) |
| `Connection` | Sessão com o banco de dados |
| `Statement` | Executa SQL sem parâmetros (evitar em produção) |
| `PreparedStatement` | Executa SQL com parâmetros (usar sempre) |
| `ResultSet` | Cursor para navegar os resultados de uma query |

---

## Dependências Maven

```xml
<!-- Driver PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.3</version>
</dependency>

<!-- HikariCP — Connection Pool -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>

<!-- H2 para testes -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.2.224</version>
    <scope>test</scope>
</dependency>
```

---

## PostgreSQL via Docker

Não instale PostgreSQL diretamente na sua máquina para desenvolvimento. Use Docker:

```bash
# Iniciar PostgreSQL com Docker
docker run -d \
  --name postgres-dev \
  -e POSTGRES_DB=appdb \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=apppass \
  -p 5432:5432 \
  postgres:16-alpine

# Verificar se está rodando
docker ps

# Parar e iniciar novamente
docker stop postgres-dev
docker start postgres-dev

# Acessar o psql dentro do container
docker exec -it postgres-dev psql -U appuser -d appdb
```

### Comandos `psql` essenciais

```sql
-- Listar bancos
\l

-- Conectar em um banco
\c appdb

-- Listar tabelas
\dt

-- Descrever estrutura de uma tabela
\d usuarios

-- Listar usuários do PostgreSQL
\du

-- Sair
\q
```

### Tipos de dados PostgreSQL mais usados

```sql
-- Tipos numéricos
BIGSERIAL           -- auto-increment 64-bit (use para PKs)
INTEGER / INT       -- inteiro 32-bit
DECIMAL(10, 2)      -- precisão exata (use para dinheiro)
DOUBLE PRECISION    -- ponto flutuante (evitar para valores monetários)

-- Texto
VARCHAR(255)        -- texto com limite
TEXT                -- texto sem limite
UUID                -- identificador universal

-- Data e hora
TIMESTAMP           -- data + hora sem timezone
TIMESTAMPTZ         -- data + hora com timezone (preferível)
DATE                -- apenas data
BOOLEAN             -- true/false

-- Outros
JSONB               -- JSON binário (pesquisável e indexável)
```

---

## Connection Pool com HikariCP

Abrir uma nova `Connection` a cada operação é muito lento. O connection pool mantém um conjunto de conexões abertas e prontas para uso.

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import javax.sql.DataSource;

public class DataSourceFactory {

    private static HikariDataSource dataSource;

    public static DataSource getDataSource() {
        if (dataSource == null) {
            HikariConfig config = new HikariConfig();
            config.setJdbcUrl("jdbc:postgresql://localhost:5432/appdb");
            config.setUsername("appuser");
            config.setPassword("apppass");

            // Configurações do pool
            config.setMaximumPoolSize(10);        // máximo de conexões simultâneas
            config.setMinimumIdle(2);             // mínimo de conexões ociosas
            config.setConnectionTimeout(30_000);  // timeout para obter conexão (ms)
            config.setIdleTimeout(600_000);       // timeout de conexão ociosa (ms)
            config.setMaxLifetime(1_800_000);     // tempo máximo de vida da conexão (ms)
            config.setPoolName("AppPool");

            // Validação da conexão
            config.setConnectionTestQuery("SELECT 1");

            dataSource = new HikariDataSource(config);
        }
        return dataSource;
    }
}
```

> **Por que HikariCP?** É o pool mais rápido disponível para Java. É o padrão do Spring Boot. Benchmarks mostram que ele é 2-5x mais rápido que alternativas como DBCP2 e c3p0.

---

## CRUD Completo com PreparedStatement

### Criando a tabela

```sql
CREATE TABLE usuarios (
    id          BIGSERIAL PRIMARY KEY,
    nome        VARCHAR(100) NOT NULL,
    email       VARCHAR(150) UNIQUE NOT NULL,
    idade       INT CHECK (idade >= 0),
    criado_em   TIMESTAMPTZ DEFAULT NOW()
);
```

### Classe de domínio

```java
import java.time.OffsetDateTime;

public class Usuario {
    private Long id;
    private String nome;
    private String email;
    private int idade;
    private OffsetDateTime criadoEm;

    // construtores, getters, setters...
}
```

### DAO (Data Access Object) com JDBC

```java
import javax.sql.DataSource;
import java.sql.*;
import java.util.*;

public class UsuarioDAO {

    private final DataSource dataSource;

    public UsuarioDAO(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    // CREATE
    public Usuario criar(Usuario usuario) {
        String sql = "INSERT INTO usuarios (nome, email, idade) VALUES (?, ?, ?) RETURNING id, criado_em";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, usuario.getNome());
            stmt.setString(2, usuario.getEmail());
            stmt.setInt(3, usuario.getIdade());

            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    usuario.setId(rs.getLong("id"));
                    usuario.setCriadoEm(rs.getObject("criado_em", OffsetDateTime.class));
                }
            }

            return usuario;

        } catch (SQLException e) {
            throw new RuntimeException("Erro ao criar usuário", e);
        }
    }

    // READ por ID
    public Optional<Usuario> buscarPorId(Long id) {
        String sql = "SELECT id, nome, email, idade, criado_em FROM usuarios WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, id);

            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(mapearUsuario(rs));
                }
                return Optional.empty();
            }

        } catch (SQLException e) {
            throw new RuntimeException("Erro ao buscar usuário", e);
        }
    }

    // READ todos
    public List<Usuario> listarTodos() {
        String sql = "SELECT id, nome, email, idade, criado_em FROM usuarios ORDER BY id";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql);
             ResultSet rs = stmt.executeQuery()) {

            List<Usuario> usuarios = new ArrayList<>();
            while (rs.next()) {
                usuarios.add(mapearUsuario(rs));
            }
            return usuarios;

        } catch (SQLException e) {
            throw new RuntimeException("Erro ao listar usuários", e);
        }
    }

    // UPDATE
    public boolean atualizar(Usuario usuario) {
        String sql = "UPDATE usuarios SET nome = ?, email = ?, idade = ? WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, usuario.getNome());
            stmt.setString(2, usuario.getEmail());
            stmt.setInt(3, usuario.getIdade());
            stmt.setLong(4, usuario.getId());

            int linhasAfetadas = stmt.executeUpdate();
            return linhasAfetadas > 0;

        } catch (SQLException e) {
            throw new RuntimeException("Erro ao atualizar usuário", e);
        }
    }

    // DELETE
    public boolean deletar(Long id) {
        String sql = "DELETE FROM usuarios WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, id);
            int linhasAfetadas = stmt.executeUpdate();
            return linhasAfetadas > 0;

        } catch (SQLException e) {
            throw new RuntimeException("Erro ao deletar usuário", e);
        }
    }

    // Mapeamento ResultSet → Usuario
    private Usuario mapearUsuario(ResultSet rs) throws SQLException {
        Usuario u = new Usuario();
        u.setId(rs.getLong("id"));
        u.setNome(rs.getString("nome"));
        u.setEmail(rs.getString("email"));
        u.setIdade(rs.getInt("idade"));
        u.setCriadoEm(rs.getObject("criado_em", OffsetDateTime.class));
        return u;
    }
}
```

### Navegação e extração do ResultSet

```java
// Tipos mais comuns
rs.getLong("id")
rs.getString("nome")
rs.getInt("idade")
rs.getDouble("saldo")
rs.getBigDecimal("preco")           // para valores monetários (nunca double)
rs.getBoolean("ativo")
rs.getDate("data_nascimento")       // java.sql.Date
rs.getTimestamp("criado_em")        // java.sql.Timestamp
rs.getObject("criado_em", OffsetDateTime.class)  // tipos modernos do Java 8+

// Verificar nulos
int idade = rs.getInt("idade");
if (rs.wasNull()) {
    // campo era NULL no banco
}

// Ou use getObject para nullable
Integer idadeNullable = (Integer) rs.getObject("idade");
```

---

## Transactions manuais

```java
public void transferir(Long idOrigem, Long idDestino, double valor) {
    try (Connection conn = dataSource.getConnection()) {
        conn.setAutoCommit(false); // inicia transação manual

        try {
            debitar(conn, idOrigem, valor);
            creditar(conn, idDestino, valor);

            conn.commit(); // confirma se tudo deu certo

        } catch (Exception e) {
            conn.rollback(); // desfaz se algo falhou
            throw new RuntimeException("Transferência falhou", e);
        } finally {
            conn.setAutoCommit(true); // restaura comportamento padrão
        }

    } catch (SQLException e) {
        throw new RuntimeException("Erro de conexão", e);
    }
}

private void debitar(Connection conn, Long id, double valor) throws SQLException {
    String sql = "UPDATE contas SET saldo = saldo - ? WHERE id = ? AND saldo >= ?";
    try (PreparedStatement stmt = conn.prepareStatement(sql)) {
        stmt.setDouble(1, valor);
        stmt.setLong(2, id);
        stmt.setDouble(3, valor);
        if (stmt.executeUpdate() == 0) {
            throw new RuntimeException("Saldo insuficiente ou conta não encontrada");
        }
    }
}

private void creditar(Connection conn, Long id, double valor) throws SQLException {
    String sql = "UPDATE contas SET saldo = saldo + ? WHERE id = ?";
    try (PreparedStatement stmt = conn.prepareStatement(sql)) {
        stmt.setDouble(1, valor);
        stmt.setLong(2, id);
        stmt.executeUpdate();
    }
}
```

---

## H2 para Testes

H2 é um banco de dados em memória escrito em Java. Não precisa de Docker nem configuração externa — ideal para testes automatizados.

### Configuração para testes

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import javax.sql.DataSource;

public class TestDataSourceFactory {

    public static DataSource criarDataSourceH2() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL");
        config.setUsername("sa");
        config.setPassword("");
        config.setDriverClassName("org.h2.Driver");
        return new HikariDataSource(config);
    }
}
```

### Teste com H2

```java
import org.junit.jupiter.api.*;
import javax.sql.DataSource;
import java.sql.*;

class UsuarioDAOTest {

    private static DataSource dataSource;
    private UsuarioDAO dao;

    @BeforeAll
    static void setupBanco() throws SQLException {
        dataSource = TestDataSourceFactory.criarDataSourceH2();

        // Criar tabela no H2
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("""
                CREATE TABLE IF NOT EXISTS usuarios (
                    id BIGINT AUTO_INCREMENT PRIMARY KEY,
                    nome VARCHAR(100) NOT NULL,
                    email VARCHAR(150) UNIQUE NOT NULL,
                    idade INT,
                    criado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
                """);
        }
    }

    @BeforeEach
    void setup() {
        dao = new UsuarioDAO(dataSource);
    }

    @AfterEach
    void limparTabela() throws SQLException {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("DELETE FROM usuarios");
        }
    }

    @Test
    void deveCriarEBuscarUsuario() {
        Usuario usuario = new Usuario("Ana", "ana@email.com", 30);
        Usuario criado = dao.criar(usuario);

        assertNotNull(criado.getId());

        Optional<Usuario> buscado = dao.buscarPorId(criado.getId());
        assertTrue(buscado.isPresent());
        assertEquals("Ana", buscado.get().getNome());
    }
}
```

> **Por que H2 nos testes e PostgreSQL em produção?** H2 em memória é muito mais rápido para testes unitários. Mas atenção: H2 não suporta todos os recursos do PostgreSQL. Para testes de integração que testam queries específicas do Postgres, use Testcontainers (ver seção 5.5).

---

## Boas Práticas JDBC

### Nunca concatene SQL com dados do usuário

```java
// ERRADO — vulnerável a SQL Injection
String nome = request.getParameter("nome");
String sql = "SELECT * FROM usuarios WHERE nome = '" + nome + "'";
// Se nome = "'; DROP TABLE usuarios; --", você perde o banco

// CORRETO — PreparedStatement escapa automaticamente
String sql = "SELECT * FROM usuarios WHERE nome = ?";
try (PreparedStatement stmt = conn.prepareStatement(sql)) {
    stmt.setString(1, nome); // seguro, qualquer valor é tratado como dado
}
```

### Sempre feche os recursos

```java
// ERRADO — pode vazar conexões se exception ocorrer antes do close()
Connection conn = dataSource.getConnection();
PreparedStatement stmt = conn.prepareStatement(sql);
ResultSet rs = stmt.executeQuery();
// ... se lançar exception aqui, conn/stmt/rs jamais serão fechados
conn.close();

// CORRETO — try-with-resources garante fechamento mesmo com exception
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql);
     ResultSet rs = stmt.executeQuery()) {
    // recursos fechados automaticamente ao sair do bloco
}
```

---

## Exemplo Completo — Aplicação de Console com CRUD

```java
public class AplicacaoUsuarios {

    public static void main(String[] args) {
        DataSource ds = DataSourceFactory.getDataSource();
        UsuarioDAO dao = new UsuarioDAO(ds);

        // Criar
        Usuario ana = dao.criar(new Usuario("Ana Silva", "ana@email.com", 28));
        Usuario bruno = dao.criar(new Usuario("Bruno Costa", "bruno@email.com", 35));
        System.out.println("Criados: " + ana.getId() + ", " + bruno.getId());

        // Listar
        System.out.println("\nTodos os usuários:");
        dao.listarTodos().forEach(u ->
                System.out.printf("  [%d] %s (%d anos)%n",
                        u.getId(), u.getNome(), u.getIdade()));

        // Buscar por ID
        dao.buscarPorId(ana.getId()).ifPresent(u ->
                System.out.println("\nEncontrado: " + u.getNome()));

        // Atualizar
        ana.setNome("Ana Silva Santos");
        ana.setIdade(29);
        boolean atualizado = dao.atualizar(ana);
        System.out.println("\nAtualizado: " + atualizado);

        // Deletar
        boolean deletado = dao.deletar(bruno.getId());
        System.out.println("Deletado: " + deletado);

        // Listar após deleção
        System.out.println("\nApós deleção:");
        dao.listarTodos().forEach(u ->
                System.out.printf("  [%d] %s%n", u.getId(), u.getNome()));
    }
}
```

### Saída esperada

```
Criados: 1, 2

Todos os usuários:
  [1] Ana Silva (28 anos)
  [2] Bruno Costa (35 anos)

Encontrado: Ana Silva

Atualizado: true
Deletado: true

Após deleção:
  [1] Ana Silva Santos
```
