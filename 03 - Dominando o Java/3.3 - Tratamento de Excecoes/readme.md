# 3.3 — Tratamento de Exceções

Tratamento de exceções é um dos tópicos mais mal aplicados em Java. Código com exceções malfeitas é difícil de depurar, esconde bugs reais e cria sistemas frágeis. Este guia ensina como fazer certo.

---

## Hierarquia: Throwable, Error e Exception

```
java.lang.Throwable
├── java.lang.Error                  ← NUNCA capture (problemas da JVM)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   ├── VirtualMachineError
│   └── AssertionError
│
└── java.lang.Exception              ← Problemas que o código pode tratar
    ├── CHECKED EXCEPTIONS           ← Compilador obriga a tratar ou declarar
    │   ├── IOException
    │   │   ├── FileNotFoundException
    │   │   └── SocketException
    │   ├── SQLException
    │   ├── ParseException
    │   ├── ClassNotFoundException
    │   └── InterruptedException
    │
    └── RuntimeException             ← UNCHECKED — compilador não obriga
        ├── NullPointerException
        ├── IllegalArgumentException
        ├── IllegalStateException
        ├── IndexOutOfBoundsException
        ├── ClassCastException
        ├── ArithmeticException
        ├── UnsupportedOperationException
        └── NumberFormatException
```

**A regra fundamental:**
- **Error:** nunca capture. A JVM está em estado irrecuperável.
- **Checked Exception:** o compilador força você a lidar. Representa falhas antecipáveis externas ao código (arquivo não encontrado, banco indisponível).
- **Unchecked (RuntimeException):** erro de programação ou condição impossível de recuperar. Bugs que deveriam ser corrigidos, não capturados.

---

## try-catch-finally

```java
// Estrutura básica
public String lerArquivo(String caminho) {
    FileReader leitor = null;
    try {
        leitor = new FileReader(caminho);
        var buffer = new BufferedReader(leitor);
        return buffer.lines().collect(Collectors.joining("\n"));

    } catch (FileNotFoundException e) {
        // Exceção específica primeiro (mais específica antes da mais genérica)
        log.warn("Arquivo não encontrado: {}", caminho);
        return "";

    } catch (IOException e) {
        // Exceção mais genérica depois
        log.error("Erro de I/O ao ler {}: {}", caminho, e.getMessage(), e);
        throw new LeituraArquivoException("Falha ao ler arquivo: " + caminho, e);

    } finally {
        // finally SEMPRE executa — ideal para liberar recursos
        // Mas: try-with-resources é melhor para AutoCloseable
        if (leitor != null) {
            try {
                leitor.close();
            } catch (IOException e) {
                log.error("Erro ao fechar o arquivo", e);
            }
        }
    }
}
```

**Quando usar finally:**
- Liberar recursos que não implementam `AutoCloseable` (legado)
- Logging de auditoria que deve ocorrer sempre
- Métricas de tempo de execução

---

## try-with-resources (AutoCloseable)

Qualquer classe que implementa `AutoCloseable` (ou `Closeable`) pode ser usada em try-with-resources. O recurso é fechado automaticamente ao sair do bloco, mesmo com exceção.

```java
// ANTES — propenso a vazamentos de recurso
public List<String> lerLinhas(Path arquivo) throws IOException {
    BufferedReader reader = new BufferedReader(new FileReader(arquivo.toFile()));
    try {
        return reader.lines().toList();
    } finally {
        reader.close();   // e se o close() lançar exceção?
    }
}

// DEPOIS — try-with-resources
public List<String> lerLinhas(Path arquivo) throws IOException {
    try (var reader = new BufferedReader(new FileReader(arquivo.toFile()))) {
        return reader.lines().toList();
    }
    // reader.close() é chamado automaticamente
}

// Múltiplos recursos — fechados na ordem inversa de abertura
public void copiarArquivo(Path origem, Path destino) throws IOException {
    try (
        var entrada = new FileInputStream(origem.toFile());
        var saida   = new FileOutputStream(destino.toFile())
    ) {
        entrada.transferTo(saida);
    }
    // saida.close() primeiro, depois entrada.close()
}

// Com banco de dados — garantindo fechamento de Connection, Statement e ResultSet
public Optional<Cliente> buscarCliente(UUID id) {
    String sql = "SELECT id, nome, email FROM clientes WHERE id = ?";

    try (
        var conn = dataSource.getConnection();
        var stmt = conn.prepareStatement(sql)
    ) {
        stmt.setObject(1, id);

        try (var rs = stmt.executeQuery()) {
            if (rs.next()) {
                return Optional.of(new Cliente(
                    UUID.fromString(rs.getString("id")),
                    rs.getString("nome"),
                    rs.getString("email")
                ));
            }
            return Optional.empty();
        }

    } catch (SQLException e) {
        throw new DatabaseException("Erro ao buscar cliente: " + id, e);
    }
}

// Criando sua própria classe AutoCloseable
public class ConexaoLegada implements AutoCloseable {
    private final Socket socket;

    public ConexaoLegada(String host, int porta) throws IOException {
        this.socket = new Socket(host, porta);
        log.info("Conexão aberta para {}:{}", host, porta);
    }

    public String enviarComando(String comando) throws IOException {
        // ... envio e leitura
        return resposta;
    }

    @Override
    public void close() throws IOException {
        log.info("Fechando conexão");
        socket.close();
    }
}

// Uso — fecha automaticamente
try (var conexao = new ConexaoLegada("servidor.com", 8080)) {
    String resposta = conexao.enviarComando("PING");
    processar(resposta);
}
```

---

## Multi-catch

```java
// ANTES — catch duplicado
public void processarArquivoXml(Path arquivo) {
    try {
        var doc = DocumentBuilderFactory.newInstance()
            .newDocumentBuilder()
            .parse(arquivo.toFile());
        processar(doc);
    } catch (IOException e) {
        log.error("Erro de I/O: {}", e.getMessage(), e);
        throw new ProcessamentoException("Falha no processamento", e);
    } catch (SAXException e) {
        log.error("Erro de I/O: {}", e.getMessage(), e);   // duplicação!
        throw new ProcessamentoException("Falha no processamento", e);
    } catch (ParserConfigurationException e) {
        log.error("Erro de I/O: {}", e.getMessage(), e);   // duplicação!
        throw new ProcessamentoException("Falha no processamento", e);
    }
}

// DEPOIS — multi-catch
public void processarArquivoXml(Path arquivo) {
    try {
        var doc = DocumentBuilderFactory.newInstance()
            .newDocumentBuilder()
            .parse(arquivo.toFile());
        processar(doc);
    } catch (IOException | SAXException | ParserConfigurationException e) {
        log.error("Erro ao processar XML {}: {}", arquivo, e.getMessage(), e);
        throw new ProcessamentoException("Falha ao processar arquivo XML", e);
    }
}
```

> **Atenção:** no multi-catch, o parâmetro `e` é efetivamente final — não pode ser reatribuído.

---

## Criando exceções customizadas

### Exceção unchecked (RuntimeException) — mais comum em aplicações modernas

```java
// Base para exceções da aplicação — facilita tratamento global
public class AppException extends RuntimeException {

    private final String codigo;   // código de erro para a API

    public AppException(String codigo, String mensagem) {
        super(mensagem);
        this.codigo = codigo;
    }

    public AppException(String codigo, String mensagem, Throwable causa) {
        super(mensagem, causa);
        this.codigo = codigo;
    }

    public String getCodigo() {
        return codigo;
    }
}

// Exceções específicas de domínio — herdam da base
public class RecursoNaoEncontradoException extends AppException {

    public RecursoNaoEncontradoException(String recurso, Object id) {
        super("RECURSO_NAO_ENCONTRADO",
              String.format("%s com id '%s' não foi encontrado", recurso, id));
    }
}

public class RegraDeNegocioException extends AppException {

    public RegraDeNegocioException(String mensagem) {
        super("REGRA_DE_NEGOCIO", mensagem);
    }
}

public class PedidoNaoEncontradoException extends RecursoNaoEncontradoException {

    public PedidoNaoEncontradoException(UUID id) {
        super("Pedido", id);
    }
}

public class SaldoInsuficienteException extends RegraDeNegocioException {

    private final BigDecimal saldoAtual;
    private final BigDecimal valorSolicitado;

    public SaldoInsuficienteException(BigDecimal saldoAtual, BigDecimal valorSolicitado) {
        super(String.format(
            "Saldo insuficiente. Disponível: R$ %.2f, Solicitado: R$ %.2f",
            saldoAtual, valorSolicitado
        ));
        this.saldoAtual     = saldoAtual;
        this.valorSolicitado = valorSolicitado;
    }

    public BigDecimal getSaldoAtual()      { return saldoAtual; }
    public BigDecimal getValorSolicitado() { return valorSolicitado; }
}
```

### Exceção checked — use apenas quando o chamador PODE e DEVE tratar

```java
// Exceção checked: o compilador força o tratamento
// Use quando o chamador tem algo concreto a fazer com o erro
public class ErroDeIntegracaoException extends Exception {

    private final int codigoHttp;

    public ErroDeIntegracaoException(String servico, int codigoHttp, String mensagem) {
        super(String.format("[%s] HTTP %d: %s", servico, codigoHttp, mensagem));
        this.codigoHttp = codigoHttp;
    }

    public ErroDeIntegracaoException(String servico, String mensagem, Throwable causa) {
        super(String.format("[%s] %s", servico, mensagem), causa);
        this.codigoHttp = -1;
    }

    public int getCodigoHttp() { return codigoHttp; }
    public boolean ehRetentavel() { return codigoHttp >= 500; }
}

// Declaração e uso
public Pagamento consultarPagamento(UUID id) throws ErroDeIntegracaoException {
    try {
        return httpClient.get("/pagamentos/" + id, Pagamento.class);
    } catch (HttpClientException e) {
        throw new ErroDeIntegracaoException("PagamentosAPI", e.getStatusCode(), e.getMessage());
    }
}

// Chamador é forçado a tratar
try {
    var pagamento = consultarPagamento(id);
    processar(pagamento);
} catch (ErroDeIntegracaoException e) {
    if (e.ehRetentavel()) {
        agendarRetentativa(id);
    } else {
        notificarFalha(id, e.getMessage());
    }
}
```

---

## Boas práticas

### Não engolir exceções

```java
// PÉSSIMO — esconde erros, impossível de depurar
try {
    processarPedido(pedido);
} catch (Exception e) {
    // nada aqui — o bug desaparece silenciosamente
}

// PÉSSIMO — captura genérica sem reraise
} catch (Exception e) {
    System.out.println("deu erro");
}

// BOM — loga com contexto e rethrow ou trata adequadamente
} catch (PedidoInvalidoException e) {
    log.warn("Pedido inválido recebido — id: {}, motivo: {}", pedido.id(), e.getMessage());
    // Retorna erro para o chamador (não relança)
    return ResultadoProcessamento.falha(e.getMessage());

} catch (IOException e) {
    log.error("Erro de I/O ao processar pedido {}", pedido.id(), e);
    throw new ProcessamentoPedidoException("Falha técnica ao processar pedido", e);
}
```

### Não usar exceções para controle de fluxo

```java
// RUIM — exceção como controle de fluxo (lento e semanticamente errado)
public boolean clienteExiste(String cpf) {
    try {
        buscarClientePorCpf(cpf);
        return true;
    } catch (ClienteNaoEncontradoException e) {
        return false;   // exceção como if/else — errado
    }
}

// BOM — retorno semântico
public boolean clienteExiste(String cpf) {
    return clienteRepository.existsByCpf(cpf);
}

// BOM — Optional
public Optional<Cliente> buscarPorCpf(String cpf) {
    return clienteRepository.findByCpf(cpf);
}
```

### Logar corretamente — com a causa e contexto

```java
private static final Logger log = LoggerFactory.getLogger(PagamentoService.class);

// RUIM — perde a stack trace original
} catch (SQLException e) {
    log.error("Erro no banco: " + e.getMessage());  // sem stack trace
    throw new DatabaseException(e.getMessage());     // sem causa encadeada
}

// BOM — preserva a causa original (e) na stack trace e no encadeamento
} catch (SQLException e) {
    log.error("Erro ao persistir pagamento. pedidoId={}, sql={}", pedidoId, sql, e);
    //                                                                          ^
    //                                             último argumento = Throwable → inclui stack trace
    throw new DatabaseException("Falha ao registrar pagamento", e);
    //                                                            ^ causa encadeada (getCause())
}
```

### Hierarquia de captura

```java
// SEMPRE mais específico antes do mais genérico
try {
    executar();
} catch (FileNotFoundException e) {   // filho
    // trata arquivo não encontrado especificamente
} catch (IOException e) {             // pai
    // trata demais erros de I/O
} catch (Exception e) {               // avô — captura qualquer coisa restante
    // último recurso
}

// Erro de compilação: FileNotFoundException após IOException não compila
// (FileNotFoundException é subclasse de IOException — nunca seria alcançado)
```

---

## Quando usar checked vs unchecked

| Critério                                               | Checked                           | Unchecked                           |
|--------------------------------------------------------|-----------------------------------|-------------------------------------|
| O chamador pode se recuperar?                          | Sim → checked                     | Não → unchecked                     |
| É um bug de programação?                               | Não                               | Sim → unchecked                     |
| É uma falha externa (rede, arquivo, banco)?            | Sim → checked                     | Depende do contrato da API          |
| Código chamador tem ação concreta a tomar?             | Sim → checked                     | Não → unchecked                     |
| Aplicações Spring Boot modernas                        | Raramente                         | Preferência na maioria dos casos    |

**Tendência moderna:** a maioria das aplicações Spring Boot usa **somente unchecked exceptions**, convertendo checked em unchecked no ponto de integração (adapter/repository). O Spring Data, por exemplo, converte `SQLException` em `DataAccessException` (unchecked).

---

## Optional vs exceção

```java
// Use Optional quando:
// - O valor pode ou não existir e isso é normal (não é erro)
// - O chamador vai decidir o que fazer com a ausência

// BOM com Optional — "não encontrar um usuário por ID é normal"
public Optional<Usuario> buscarPorId(UUID id) {
    return usuarioRepository.findById(id);
}

// Chamador:
buscarPorId(id)
    .map(UsuarioDTO::fromEntity)
    .orElseThrow(() -> new UsuarioNaoEncontradoException(id));

// Use exceção quando:
// - A ausência é uma violação de regra de negócio ou estado inválido
// - Não faz sentido continuar a execução sem o dado

// BOM com exceção — "ao processar um pagamento, o pedido DEVE existir"
public void processarPagamento(UUID pedidoId, BigDecimal valor) {
    Pedido pedido = pedidoRepository.findById(pedidoId)
        .orElseThrow(() -> new PedidoNaoEncontradoException(pedidoId));
    // Se chegou aqui, temos o pedido com certeza
    realizarCobranca(pedido, valor);
}
```

---

## Exemplos completos com cenários reais

### Leitura de arquivo CSV

```java
@Service
public class ImportacaoProdutoService {

    private static final Logger log = LoggerFactory.getLogger(ImportacaoProdutoService.class);

    public ResultadoImportacao importar(Path arquivoCsv) {
        var erros    = new ArrayList<String>();
        var sucessos = new ArrayList<Produto>();

        if (!Files.exists(arquivoCsv)) {
            throw new RecursoNaoEncontradoException("Arquivo CSV", arquivoCsv);
        }

        try (var linhas = Files.lines(arquivoCsv, StandardCharsets.UTF_8)) {

            linhas
                .skip(1)           // pula o cabeçalho
                .filter(l -> !l.isBlank())
                .forEach(linha -> {
                    try {
                        var produto = parsearLinha(linha);
                        sucessos.add(produto);
                    } catch (DadoInvalidoException e) {
                        erros.add("Linha inválida: " + linha + " — " + e.getMessage());
                        log.warn("Linha ignorada na importação: '{}' — {}", linha, e.getMessage());
                    }
                });

        } catch (IOException e) {
            log.error("Falha ao ler arquivo de importação: {}", arquivoCsv, e);
            throw new ImportacaoException("Não foi possível ler o arquivo: " + arquivoCsv, e);
        }

        log.info("Importação concluída. Sucesso: {}, Erros: {}", sucessos.size(), erros.size());
        return new ResultadoImportacao(sucessos, erros);
    }

    private Produto parsearLinha(String linha) {
        var campos = linha.split(",", -1);

        if (campos.length < 3) {
            throw new DadoInvalidoException("Esperado 3 campos, encontrado: " + campos.length);
        }

        try {
            var nome   = campos[0].trim();
            var preco  = new BigDecimal(campos[1].trim());
            var estoque = Integer.parseInt(campos[2].trim());

            if (nome.isBlank()) {
                throw new DadoInvalidoException("Nome do produto não pode ser vazio");
            }
            if (preco.compareTo(BigDecimal.ZERO) < 0) {
                throw new DadoInvalidoException("Preço não pode ser negativo: " + preco);
            }

            return new Produto(nome, preco, estoque);

        } catch (NumberFormatException e) {
            throw new DadoInvalidoException("Formato numérico inválido na linha: " + linha, e);
        }
    }
}
```

### Conexão com banco de dados

```java
@Repository
public class PedidoRepository {

    private static final Logger log = LoggerFactory.getLogger(PedidoRepository.class);

    private final DataSource dataSource;

    public PedidoRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Optional<Pedido> findById(UUID id) {
        var sql = "SELECT id, cliente_id, valor, status, criado_em FROM pedidos WHERE id = ?";

        try (
            var conn = dataSource.getConnection();
            var stmt = conn.prepareStatement(sql)
        ) {
            stmt.setObject(1, id);

            try (var rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(mapearResultSet(rs));
                }
                return Optional.empty();
            }

        } catch (SQLException e) {
            // Converte checked (SQLException) em unchecked de domínio
            log.error("Erro ao buscar pedido id={}", id, e);
            throw new DatabaseException("Falha ao buscar pedido", e);
        }
    }

    public Pedido salvar(Pedido pedido) {
        var sql = """
                INSERT INTO pedidos (id, cliente_id, valor, status, criado_em)
                VALUES (?, ?, ?, ?, ?)
                ON CONFLICT (id) DO UPDATE
                SET valor = EXCLUDED.valor, status = EXCLUDED.status
                """;

        try (
            var conn = dataSource.getConnection();
            var stmt = conn.prepareStatement(sql)
        ) {
            stmt.setObject(1, pedido.id());
            stmt.setObject(2, pedido.clienteId());
            stmt.setBigDecimal(3, pedido.valor());
            stmt.setString(4, pedido.status().name());
            stmt.setTimestamp(5, Timestamp.valueOf(pedido.criadoEm()));

            int linhasAfetadas = stmt.executeUpdate();

            if (linhasAfetadas == 0) {
                throw new DatabaseException("Nenhuma linha afetada ao salvar pedido: " + pedido.id());
            }

            log.debug("Pedido {} salvo com sucesso", pedido.id());
            return pedido;

        } catch (SQLException e) {
            // Detecta violação de constraint específica
            if (e.getSQLState() != null && e.getSQLState().startsWith("23")) {
                throw new ViolacaoDeConstraintException(
                    "Pedido com dados conflitantes: " + pedido.id(), e
                );
            }
            log.error("Erro ao salvar pedido id={}", pedido.id(), e);
            throw new DatabaseException("Falha ao persistir pedido", e);
        }
    }

    private Pedido mapearResultSet(ResultSet rs) throws SQLException {
        return new Pedido(
            UUID.fromString(rs.getString("id")),
            UUID.fromString(rs.getString("cliente_id")),
            rs.getBigDecimal("valor"),
            Status.valueOf(rs.getString("status")),
            rs.getTimestamp("criado_em").toLocalDateTime()
        );
    }
}
```

### Tratamento global com Spring Boot

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(RecursoNaoEncontradoException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErroResponse handleNaoEncontrado(RecursoNaoEncontradoException e) {
        return new ErroResponse(e.getCodigo(), e.getMessage());
    }

    @ExceptionHandler(RegraDeNegocioException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    public ErroResponse handleRegraDeNegocio(RegraDeNegocioException e) {
        return new ErroResponse(e.getCodigo(), e.getMessage());
    }

    @ExceptionHandler(DatabaseException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErroResponse handleDatabase(DatabaseException e) {
        log.error("Erro de banco de dados", e);
        // Não expõe detalhes técnicos para o cliente
        return new ErroResponse("ERRO_INTERNO", "Erro interno. Tente novamente.");
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErroResponse handleGenerico(Exception e) {
        log.error("Erro não tratado", e);
        return new ErroResponse("ERRO_INESPERADO", "Ocorreu um erro inesperado.");
    }

    public record ErroResponse(String codigo, String mensagem) {}
}
```
