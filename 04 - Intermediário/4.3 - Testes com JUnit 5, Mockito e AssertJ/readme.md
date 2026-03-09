# 4.3 — Testes com JUnit 5, Mockito e AssertJ

Testar não é opcional em software profissional. Código sem teste é código que você não pode alterar com confiança. Esta seção cobre as ferramentas padrão do ecossistema Java para testes unitários.

---

## Por que Testar? A Pirâmide de Testes

```
          /\
         /E2E\          ← poucos, lentos, caros
        /------\
       /Integr. \       ← alguns, moderados
      /----------\
     /  Unitários \     ← muitos, rápidos, baratos
    /--------------\
```

- **Unitários:** testam uma classe/método isolado, sem dependências reais. Rápidos (milissegundos). A base da pirâmide.
- **Integração:** testam a interação entre partes: serviço + banco, controller + Spring. Mais lentos.
- **E2E:** testam o sistema completo do ponto de vista do usuário. Lentos e frágeis.

A maioria do esforço deve estar nos testes unitários. Eles são rápidos, fáceis de manter e dão feedback imediato.

---

## Dependências Maven

```xml
<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.11.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.12.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.12.0</version>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.26.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.3.1</version>
        </plugin>
    </plugins>
</build>
```

> No Spring Boot, todas essas dependências já estão incluídas em `spring-boot-starter-test`.

---

## JUnit 5

### Estrutura básica de um teste

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

@DisplayName("Calculadora")
class CalculadoraTest {

    private Calculadora calc;

    @BeforeAll
    static void configuracaoGlobal() {
        // Executado UMA VEZ antes de todos os testes da classe
        // Deve ser static
        System.out.println("Iniciando suíte de testes");
    }

    @AfterAll
    static void limpezaGlobal() {
        // Executado UMA VEZ após todos os testes da classe
        System.out.println("Suíte finalizada");
    }

    @BeforeEach
    void setup() {
        // Executado antes de CADA teste — recriar o estado
        calc = new Calculadora();
    }

    @AfterEach
    void tearDown() {
        // Executado após CADA teste — limpar recursos
    }

    @Test
    @DisplayName("Deve somar dois números positivos")
    void deveSomarDoisNumerosPositivos() {
        int resultado = calc.somar(2, 3);
        assertEquals(5, resultado);
    }

    @Test
    @DisplayName("Deve lançar exceção ao dividir por zero")
    void deveLancarExcecaoAoDividirPorZero() {
        assertThrows(ArithmeticException.class, () -> calc.dividir(10, 0));
    }

    @Test
    @Disabled("Aguardando implementação da raiz negativa")
    void deveCalcularRaizNegativa() {
        // teste ignorado
    }

    @Test
    @Tag("regressao")
    @DisplayName("Múltiplas assertions em grupo")
    void validarVariosResultados() {
        assertAll("operações básicas",
                () -> assertEquals(5,  calc.somar(2, 3)),
                () -> assertEquals(1,  calc.subtrair(3, 2)),
                () -> assertEquals(6,  calc.multiplicar(2, 3)),
                () -> assertEquals(2,  calc.dividir(6, 3))
        );
        // assertAll executa TODOS os assertions e reporta todas as falhas de uma vez
    }

    @Test
    void deveExecutarEmTempoLimite() {
        assertTimeout(Duration.ofMillis(100), () -> {
            calc.somar(1, 1); // deve terminar em menos de 100ms
        });
    }
}
```

### `@Nested` — organizar testes por contexto

```java
@DisplayName("ServicoDeUsuario")
class ServicoDeUsuarioTest {

    @Nested
    @DisplayName("Ao criar usuário")
    class AoCriarUsuario {

        @Test
        @DisplayName("deve salvar quando dados válidos")
        void deveSalvarQuandoDadosValidos() { /* ... */ }

        @Test
        @DisplayName("deve lançar exceção quando email duplicado")
        void deveLancarExcecaoQuandoEmailDuplicado() { /* ... */ }
    }

    @Nested
    @DisplayName("Ao buscar usuário")
    class AoBuscarUsuario {

        @Test
        @DisplayName("deve retornar Optional vazio quando não encontrado")
        void deveRetornarOptionalVazioQuandoNaoEncontrado() { /* ... */ }
    }
}
```

### Testes parametrizados

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;

class ValidadorCpfTest {

    @ParameterizedTest
    @ValueSource(strings = {"111.111.111-11", "000.000.000-00", "abc"})
    @DisplayName("deve rejeitar CPFs inválidos")
    void deveRejeitarCpfsInvalidos(String cpf) {
        assertFalse(ValidadorCpf.isValido(cpf));
    }

    @ParameterizedTest(name = "{0} + {1} = {2}")
    @CsvSource({
            "1, 2, 3",
            "10, 20, 30",
            "-1, 1, 0",
            "0, 0, 0"
    })
    void deveSomarCorretamente(int a, int b, int esperado) {
        assertEquals(esperado, new Calculadora().somar(a, b));
    }

    @ParameterizedTest
    @MethodSource("produtosInvalidos")
    void deveRejeitarProdutosInvalidos(Produto produto) {
        assertThrows(IllegalArgumentException.class,
                () -> new ProdutoService().validar(produto));
    }

    static Stream<Produto> produtosInvalidos() {
        return Stream.of(
                new Produto(null, 10.0),
                new Produto("", 10.0),
                new Produto("Nome", -1.0),
                new Produto("Nome", 0.0)
        );
    }
}
```

---

## Mockito

Mockito cria objetos falsos (mocks) que simulam dependências. Assim você testa a unidade isolada sem precisar do banco de dados, de APIs externas, ou de outros serviços.

### Configuração com `MockitoExtension`

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;
import org.mockito.*;

@ExtendWith(MockitoExtension.class)
class ProdutoServiceTest {

    @Mock
    ProdutoRepository repository;    // mock: objeto falso completamente controlado

    @Mock
    NotificacaoService notificacao;  // mock de outra dependência

    @Spy
    AuditoriaService auditoria = new AuditoriaService(); // spy: objeto real, mas monitorado

    @InjectMocks
    ProdutoService service;          // objeto real com os mocks injetados

    @Captor
    ArgumentCaptor<Produto> produtoCaptor; // captura argumentos passados a mocks
}
```

### `when().thenReturn()` e `when().thenThrow()`

```java
@Test
void deveBuscarProdutoPorId() {
    // Dado
    Produto produto = new Produto(1L, "Notebook", 3500.0);
    when(repository.findById(1L)).thenReturn(Optional.of(produto));

    // Quando
    Produto resultado = service.buscarPorId(1L);

    // Então
    assertEquals("Notebook", resultado.getNome());
}

@Test
void deveLancarExcecaoQuandoProdutoNaoEncontrado() {
    when(repository.findById(99L)).thenReturn(Optional.empty());

    assertThrows(ProdutoNaoEncontradoException.class,
            () -> service.buscarPorId(99L));
}

@Test
void devePropagaExcecaoDoRepositorio() {
    when(repository.findById(any()))
            .thenThrow(new RuntimeException("Banco indisponível"));

    assertThrows(RuntimeException.class, () -> service.buscarPorId(1L));
}
```

### `verify()` — verificar que métodos foram chamados

```java
@Test
void deveChamarSaveAoCriarProduto() {
    Produto produto = new Produto("Mouse", 85.0);
    when(repository.save(any(Produto.class))).thenReturn(produto);

    service.criar(produto);

    // Verifica que save foi chamado exatamente 1 vez
    verify(repository, times(1)).save(produto);

    // Verifica que notificação foi enviada
    verify(notificacao).enviarEmail(any(String.class));

    // Verifica que auditoria NÃO foi chamada com erro
    verify(auditoria, never()).registrarErro(any());
}
```

### `ArgumentCaptor` — capturar o argumento passado

```java
@Test
void deveSalvarProdutoComPrecoFormatado() {
    Produto entrada = new Produto("Teclado", 200.0);
    when(repository.save(any())).thenAnswer(i -> i.getArgument(0));

    service.criar(entrada);

    // Captura o objeto que foi passado para o save
    verify(repository).save(produtoCaptor.capture());
    Produto produtoSalvo = produtoCaptor.getValue();

    assertEquals("Teclado", produtoSalvo.getNome());
    assertNotNull(produtoSalvo.getCriadoEm());
    assertEquals("ATIVO", produtoSalvo.getStatus());
}
```

### `doNothing` e `doThrow` para métodos void

```java
@Test
void deveEnviarNotificacaoAoDeletar() {
    doNothing().when(notificacao).enviarEmail(anyString());

    service.deletar(1L);

    verify(notificacao).enviarEmail(contains("deletado"));
}

@Test
void deveLidarComFalhaDeNotificacao() {
    doThrow(new RuntimeException("Falha no email"))
            .when(notificacao).enviarEmail(anyString());

    // O serviço deve continuar mesmo com falha na notificação
    assertDoesNotThrow(() -> service.deletar(1L));
}
```

---

## AssertJ

AssertJ oferece assertions fluentes e legíveis. É uma evolução sobre o `Assertions` do JUnit.

```java
import static org.assertj.core.api.Assertions.*;

// Strings
assertThat("Olá, mundo!")
        .isNotNull()
        .isNotEmpty()
        .startsWith("Olá")
        .endsWith("!")
        .contains("mundo")
        .hasSize(12);

// Números
assertThat(3.14)
        .isGreaterThan(3.0)
        .isLessThan(4.0)
        .isCloseTo(3.14159, within(0.01));

// Listas
List<String> nomes = List.of("Ana", "Bruno", "Carlos");
assertThat(nomes)
        .hasSize(3)
        .contains("Ana", "Bruno")
        .doesNotContain("Diego")
        .isSortedAccordingTo(Comparator.naturalOrder());

// Lista de objetos — extraindo campos
List<Produto> produtos = List.of(
        new Produto("Notebook", 3500.0),
        new Produto("Mouse", 85.0)
);
assertThat(produtos)
        .extracting(Produto::getNome)
        .containsExactly("Notebook", "Mouse");

assertThat(produtos)
        .extracting("nome", "preco")
        .containsExactly(
                tuple("Notebook", 3500.0),
                tuple("Mouse", 85.0)
        );

// Optional
Optional<String> resultado = Optional.of("valor");
assertThat(resultado)
        .isPresent()
        .hasValue("valor");

Optional<String> vazio = Optional.empty();
assertThat(vazio).isEmpty();

// Exceções
assertThatThrownBy(() -> service.buscarPorId(-1L))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("ID inválido")
        .hasNoCause();

// Alternativa mais expressiva
assertThatExceptionOfType(ProdutoNaoEncontradoException.class)
        .isThrownBy(() -> service.buscarPorId(999L))
        .withMessageContaining("999");
```

> **Por que AssertJ é melhor que JUnit Assertions?**
> - Mensagens de erro mais descritivas quando o teste falha
> - Autocompletar da IDE funciona melhor com a API fluente
> - Assertions específicas para cada tipo (`contains`, `extracting`, `tuple`)
> - Não precisa inverter a ordem dos argumentos (esperado vs real)

---

## JaCoCo — Cobertura de Código

### Configuração no `pom.xml`

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.12</version>
            <executions>
                <!-- Prepara o agente antes dos testes -->
                <execution>
                    <id>prepare-agent</id>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>

                <!-- Gera relatório HTML após os testes -->
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>

                <!-- Falha o build se cobertura for menor que o mínimo -->
                <execution>
                    <id>check</id>
                    <goals>
                        <goal>check</goal>
                    </goals>
                    <configuration>
                        <rules>
                            <rule>
                                <element>BUNDLE</element>
                                <limits>
                                    <limit>
                                        <counter>LINE</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.80</minimum> <!-- 80% mínimo -->
                                    </limit>
                                    <limit>
                                        <counter>BRANCH</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.70</minimum>
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                        <!-- Excluir classes que não precisam de cobertura -->
                        <excludes>
                            <exclude>**/dto/**</exclude>
                            <exclude>**/config/**</exclude>
                            <exclude>**/*Application.class</exclude>
                        </excludes>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

```bash
# Gerar relatório
mvn test

# Relatório HTML disponível em:
# target/site/jacoco/index.html
```

---

## Boas Práticas de Teste

### F.I.R.S.T

- **Fast** — testes devem rodar em milissegundos. Nunca dorme (`Thread.sleep`) em testes unitários.
- **Independent** — cada teste deve ser independente. Não dependa da ordem de execução.
- **Repeatable** — o mesmo resultado toda vez que rodar, em qualquer ambiente.
- **Self-validating** — o teste diz se passou ou falhou. Não exige inspeção manual.
- **Timely** — escreva o teste antes (TDD) ou junto com o código.

### Nomenclatura: `dado_quando_então`

```java
// Ruim
@Test
void testCriar() { }

// Bom
@Test
void dadoProdutoValido_quandoCriar_entaoSalvaNoBanco() { }

@Test
void dadoPrecoNegativo_quandoCriar_entaoLancaIllegalArgumentException() { }
```

### Um assert lógico por teste

```java
// Ruim: múltiplas verificações em um único teste
@Test
void testProduto() {
    Produto p = service.criar(new Produto("X", 10.0));
    assertNotNull(p.getId());
    assertEquals("X", p.getNome());
    assertEquals(10.0, p.getPreco());
    assertEquals("ATIVO", p.getStatus());
    assertNotNull(p.getCriadoEm());
}

// Bom: um teste por comportamento
@Test
void dadoProdutoValido_quandoCriar_entaoAtribuiId() {
    Produto p = service.criar(new Produto("X", 10.0));
    assertThat(p.getId()).isNotNull();
}

@Test
void dadoProdutoValido_quandoCriar_entaoDefineSatusAtivo() {
    Produto p = service.criar(new Produto("X", 10.0));
    assertThat(p.getStatus()).isEqualTo("ATIVO");
}
```

> "Um assert por teste" se refere ao comportamento verificado, não ao número de linhas. `assertAll` e assertions encadeadas do AssertJ são válidos quando verificam o mesmo comportamento.

---

## Exemplo Completo

### Classe de domínio e serviço

```java
// Produto.java
public class Produto {
    private Long id;
    private String nome;
    private double preco;
    private String status;
    private LocalDateTime criadoEm;

    public Produto(String nome, double preco) {
        if (nome == null || nome.isBlank())
            throw new IllegalArgumentException("Nome obrigatório");
        if (preco <= 0)
            throw new IllegalArgumentException("Preço deve ser positivo");
        this.nome = nome;
        this.preco = preco;
    }
    // getters e setters...
}

// ProdutoRepository.java
public interface ProdutoRepository {
    Produto save(Produto produto);
    Optional<Produto> findById(Long id);
    void deleteById(Long id);
    boolean existsById(Long id);
}

// ProdutoService.java
public class ProdutoService {

    private final ProdutoRepository repository;
    private final NotificacaoService notificacao;

    public ProdutoService(ProdutoRepository repository,
                          NotificacaoService notificacao) {
        this.repository = repository;
        this.notificacao = notificacao;
    }

    public Produto criar(Produto produto) {
        produto.setStatus("ATIVO");
        produto.setCriadoEm(LocalDateTime.now());
        Produto salvo = repository.save(produto);
        notificacao.enviarEmail("Produto criado: " + salvo.getNome());
        return salvo;
    }

    public Produto buscarPorId(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new ProdutoNaoEncontradoException(id));
    }

    public void deletar(Long id) {
        if (!repository.existsById(id))
            throw new ProdutoNaoEncontradoException(id);
        repository.deleteById(id);
        try {
            notificacao.enviarEmail("Produto " + id + " deletado");
        } catch (Exception e) {
            // falha na notificação não impede a deleção
        }
    }
}
```

### Testes completos com JUnit 5 + Mockito + AssertJ

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("ProdutoService")
class ProdutoServiceTest {

    @Mock
    ProdutoRepository repository;

    @Mock
    NotificacaoService notificacao;

    @InjectMocks
    ProdutoService service;

    @Captor
    ArgumentCaptor<Produto> produtoCaptor;

    @Nested
    @DisplayName("criar()")
    class Criar {

        @Test
        @DisplayName("deve definir status ATIVO ao criar")
        void dadoProdutoValido_quandoCriar_entaoDefineSatusAtivo() {
            Produto produto = new Produto("Mouse", 85.0);
            when(repository.save(any())).thenAnswer(i -> {
                Produto p = i.getArgument(0);
                p.setId(1L);
                return p;
            });

            Produto criado = service.criar(produto);

            assertThat(criado.getStatus()).isEqualTo("ATIVO");
        }

        @Test
        @DisplayName("deve definir data de criação")
        void dadoProdutoValido_quandoCriar_entaoDefineCriadoEm() {
            when(repository.save(any())).thenAnswer(i -> i.getArgument(0));

            Produto criado = service.criar(new Produto("Mouse", 85.0));

            assertThat(criado.getCriadoEm())
                    .isNotNull()
                    .isBeforeOrEqualTo(LocalDateTime.now());
        }

        @Test
        @DisplayName("deve enviar notificação após criar")
        void dadoProdutoValido_quandoCriar_entaoEnviaNotificacao() {
            when(repository.save(any())).thenAnswer(i -> i.getArgument(0));

            service.criar(new Produto("Teclado", 200.0));

            verify(notificacao).enviarEmail(contains("Teclado"));
        }

        @ParameterizedTest
        @ValueSource(doubles = {0.0, -1.0, -100.0})
        @DisplayName("deve rejeitar preço não positivo")
        void dadoPrecoInvalido_quandoCriar_entaoLancaExcecao(double preco) {
            assertThatThrownBy(() -> new Produto("Produto", preco))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("Preço deve ser positivo");
        }
    }

    @Nested
    @DisplayName("buscarPorId()")
    class BuscarPorId {

        @Test
        @DisplayName("deve retornar produto quando encontrado")
        void dadoIdValido_quandoBuscar_entaoRetornaProduto() {
            Produto esperado = new Produto("Notebook", 3500.0);
            when(repository.findById(1L)).thenReturn(Optional.of(esperado));

            Produto resultado = service.buscarPorId(1L);

            assertThat(resultado.getNome()).isEqualTo("Notebook");
        }

        @Test
        @DisplayName("deve lançar exceção quando não encontrado")
        void dadoIdInexistente_quandoBuscar_entaoLancaExcecao() {
            when(repository.findById(99L)).thenReturn(Optional.empty());

            assertThatExceptionOfType(ProdutoNaoEncontradoException.class)
                    .isThrownBy(() -> service.buscarPorId(99L))
                    .withMessageContaining("99");
        }
    }

    @Nested
    @DisplayName("deletar()")
    class Deletar {

        @Test
        @DisplayName("deve deletar produto existente")
        void dadoIdExistente_quandoDeletar_entaoChamaRepository() {
            when(repository.existsById(1L)).thenReturn(true);

            service.deletar(1L);

            verify(repository).deleteById(1L);
        }

        @Test
        @DisplayName("deve continuar mesmo com falha na notificação")
        void dadoFalhaNotificacao_quandoDeletar_entaoNaoPropagaExcecao() {
            when(repository.existsById(1L)).thenReturn(true);
            doThrow(new RuntimeException("Falha no email"))
                    .when(notificacao).enviarEmail(anyString());

            assertThatCode(() -> service.deletar(1L))
                    .doesNotThrowAnyException();
        }

        @Test
        @DisplayName("deve lançar exceção quando produto não existe")
        void dadoIdInexistente_quandoDeletar_entaoLancaExcecao() {
            when(repository.existsById(99L)).thenReturn(false);

            assertThatThrownBy(() -> service.deletar(99L))
                    .isInstanceOf(ProdutoNaoEncontradoException.class);

            verify(repository, never()).deleteById(any());
        }
    }
}
```
