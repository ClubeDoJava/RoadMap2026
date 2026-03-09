# 5.5 — Testes de Integração

Testes unitários verificam classes isoladas. Testes de integração verificam que as peças funcionam corretamente juntas: controller + service + banco de dados, por exemplo. Esta seção cobre as ferramentas para testes de integração no ecossistema Spring Boot.

---

## Pirâmide de Testes Revisitada

```
            /\
           /E2E\            ← Cypress, Playwright, Selenium
          /------\            poucos, lentos, caros, frágeis
         /        \
        / Integração\       ← @SpringBootTest, Testcontainers, WireMock
       /------------\         alguns, moderados, testam o sistema real
      /              \
     /   Unitários    \     ← JUnit 5, Mockito, AssertJ
    /------------------\      muitos, rápidos, baratos, precisos
```

### O que testar com unitário, o que testar com integração

| Cenário | Tipo de Teste |
|---|---|
| Lógica de negócio numa classe Service | Unitário (Mockito) |
| Validações de DTO | Unitário |
| Query JPQL complexa | Integração (banco real) |
| Endpoint REST completo (request → response) | Integração (MockMvc) |
| Integração com API externa | Unitário (WireMock) ou Integração |
| Migrations Flyway aplicadas corretamente | Integração (Testcontainers) |
| Fluxo completo de autenticação JWT | Integração |

---

## Dependências Maven

```xml
<!-- Já inclui JUnit 5, Mockito, AssertJ, MockMvc, Testcontainers e WireMock -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Testcontainers — módulos específicos -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <scope>test</scope>
</dependency>

<!-- WireMock -->
<dependency>
    <groupId>org.wiremock.integrations</groupId>
    <artifactId>wiremock-spring-boot</artifactId>
    <version>3.1.0</version>
    <scope>test</scope>
</dependency>
```

---

## `@SpringBootTest`

`@SpringBootTest` sobe o contexto completo do Spring para os testes. É mais lento que testes unitários, mas verifica que todos os componentes estão corretamente configurados e interagindo.

### Modos `webEnvironment`

```java
// MOCK (padrão) — sobe o contexto web mas sem servidor HTTP real
// Use com MockMvc para testar controllers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
class ProdutoControllerTest {}

// RANDOM_PORT — sobe servidor HTTP real em porta aleatória
// Use com TestRestTemplate ou WebTestClient para testes mais reais
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProdutoIntegrationTest {}

// DEFINED_PORT — usa a porta definida no application.yml (8080)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class ProdutoE2ETest {}

// NONE — sobe o contexto sem suporte web (para testes de serviços puros)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class ProdutoServiceIntegrationTest {}
```

### MockMvc — testando controllers

```java
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.test.web.servlet.MockMvc;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.*;

@SpringBootTest
@AutoConfigureMockMvc
class ProdutoControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Autowired
    ObjectMapper objectMapper;

    @Test
    @DisplayName("GET /api/v1/produtos deve retornar 200 com lista")
    void deveRetornarListaDeProdutos() throws Exception {
        mockMvc.perform(get("/api/v1/produtos")
                        .param("pagina", "0")
                        .param("tamanho", "10")
                        .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.content").isArray())
                .andExpect(jsonPath("$.totalElements").isNumber())
                .andDo(print()); // imprime request/response no console
    }

    @Test
    @DisplayName("POST /api/v1/produtos deve criar produto e retornar 201")
    void deveCriarProduto() throws Exception {
        ProdutoRequest request = new ProdutoRequest("Notebook", new BigDecimal("4999.90"),
                "Eletrônicos", 10, null);

        mockMvc.perform(post("/api/v1/produtos")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").isNumber())
                .andExpect(jsonPath("$.nome").value("Notebook"))
                .andExpect(jsonPath("$.preco").value(4999.90))
                .andExpect(header().exists("Location"));
    }

    @Test
    @DisplayName("POST /api/v1/produtos deve retornar 422 quando dados inválidos")
    void deveRetornar422QuandoDadosInvalidos() throws Exception {
        String jsonInvalido = """
                {
                    "nome": "",
                    "preco": -10.0
                }
                """;

        mockMvc.perform(post("/api/v1/produtos")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(jsonInvalido))
                .andExpect(status().isUnprocessableEntity())
                .andExpect(jsonPath("$.campos.nome").exists())
                .andExpect(jsonPath("$.campos.preco").exists());
    }

    @Test
    @DisplayName("GET /api/v1/produtos/{id} deve retornar 404 quando não existe")
    void deveRetornar404QuandoProdutoNaoExiste() throws Exception {
        mockMvc.perform(get("/api/v1/produtos/99999")
                        .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.mensagem").value(containsString("99999")));
    }
}
```

### MockMvc com autenticação

```java
// Usando @WithMockUser para simular usuário autenticado
import org.springframework.security.test.context.support.WithMockUser;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;

@Test
@WithMockUser(username = "ana@email.com", roles = "USER")
void deveAcessarEndpointProtegido() throws Exception {
    mockMvc.perform(get("/api/v1/perfil"))
            .andExpect(status().isOk());
}

@Test
@WithMockUser(roles = "ADMIN")
void adminDeveConseguirDeletar() throws Exception {
    mockMvc.perform(delete("/api/v1/produtos/1"))
            .andExpect(status().isNoContent());
}

@Test
void semTokenDeveRetornar401() throws Exception {
    mockMvc.perform(get("/api/v1/perfil"))
            .andExpect(status().isUnauthorized());
}

// Passando token JWT real no header
@Test
void comTokenValidoDeveAcessar() throws Exception {
    String token = gerarTokenParaTeste("ana@email.com", "USER");

    mockMvc.perform(get("/api/v1/perfil")
                    .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
}
```

### `TestRestTemplate` para `RANDOM_PORT`

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProdutoIntegrationTest {

    @Autowired
    TestRestTemplate restTemplate;

    @LocalServerPort
    int port;

    @Test
    void deveListarProdutos() {
        ResponseEntity<String> response = restTemplate.getForEntity(
                "/api/v1/produtos", String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }

    @Test
    void deveCriarProduto() {
        ProdutoRequest request = new ProdutoRequest("Mouse", new BigDecimal("85.0"),
                "Periféricos", 100, null);

        ResponseEntity<ProdutoResponse> response = restTemplate.postForEntity(
                "/api/v1/produtos", request, ProdutoResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().id()).isNotNull();
    }
}
```

---

## Testcontainers

Testcontainers sobe containers Docker reais dentro dos testes. Isso elimina a diferença entre o banco de teste (H2) e o banco de produção (PostgreSQL), tornando os testes de integração muito mais confiáveis.

### Por que não usar H2 para testes de integração?

- H2 não suporta funções específicas do PostgreSQL (`JSONB`, `UUID`, `ON CONFLICT`, extensões)
- Migrations Flyway podem usar sintaxe que só funciona no PostgreSQL
- O comportamento de locks, transações e índices pode diferir
- Com Testcontainers, você testa contra o banco **idêntico ao de produção**

### `@SpringBootTest` + `PostgreSQLContainer`

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.*;

@SpringBootTest
@Testcontainers
class ProdutoRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("testuser")
            .withPassword("testpass");

    // Injeta as propriedades do container no Spring antes de subir o contexto
    @DynamicPropertySource
    static void configurarPropriedades(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    ProdutoRepository repository;

    @Test
    void deveSalvarEBuscarProduto() {
        Produto produto = new Produto();
        produto.setNome("Teclado Mecânico");
        produto.setPreco(new BigDecimal("350.00"));
        produto.setStatus(StatusProduto.ATIVO);

        Produto salvo = repository.save(produto);

        assertThat(salvo.getId()).isNotNull();
        assertThat(salvo.getCriadoEm()).isNotNull();

        Optional<Produto> buscado = repository.findById(salvo.getId());
        assertThat(buscado).isPresent();
        assertThat(buscado.get().getNome()).isEqualTo("Teclado Mecânico");
    }
}
```

### Reutilização de container entre testes (evitar subir um container por classe)

```java
// Classe base compartilhada por todos os testes de integração
@SpringBootTest
@Testcontainers
public abstract class IntegrationTestBase {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withReuse(true); // reutiliza o container se já estiver rodando

    @DynamicPropertySource
    static void configurarPropriedades(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}

// Cada classe de teste herda a configuração
class ProdutoRepositoryTest extends IntegrationTestBase {
    @Autowired ProdutoRepository repository;
    // ...
}

class PedidoRepositoryTest extends IntegrationTestBase {
    @Autowired PedidoRepository repository;
    // ...
}
```

### Integração com Spring Boot 3.1+ — `@ServiceConnection`

A partir do Spring Boot 3.1, há uma forma ainda mais simples usando `@ServiceConnection`. O Spring detecta o tipo do container e configura o datasource automaticamente.

```java
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;

@SpringBootTest
@Testcontainers
class ProdutoRepositoryTest {

    @Container
    @ServiceConnection  // Spring Boot configura datasource automaticamente
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    // Não precisa de @DynamicPropertySource!

    @Autowired
    ProdutoRepository repository;
}
```

### `KafkaContainer` e `RedisContainer`

```java
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.DockerImageName;

@Testcontainers
class KafkaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @Container
    static GenericContainer<?> redis = new GenericContainer<>(
            DockerImageName.parse("redis:7-alpine"))
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void configurar(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
    }
}
```

---

## WireMock — Mockar APIs Externas

WireMock intercepta chamadas HTTP para APIs externas nos testes, sem precisar subir serviços reais.

```java
import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest
class ViaCepServiceTest {

    static WireMockServer wireMock = new WireMockServer(8089);

    @BeforeAll
    static void iniciarWireMock() {
        wireMock.start();
        WireMock.configureFor("localhost", 8089);
    }

    @AfterAll
    static void pararWireMock() {
        wireMock.stop();
    }

    @BeforeEach
    void resetarStubs() {
        wireMock.resetAll();
    }

    @Test
    void deveBuscarEnderecoPorCep() {
        // Configurar o stub
        stubFor(get(urlEqualTo("/ws/01310-100/json/"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {
                                    "cep": "01310-100",
                                    "logradouro": "Avenida Paulista",
                                    "bairro": "Bela Vista",
                                    "localidade": "São Paulo",
                                    "uf": "SP"
                                }
                                """)));

        // Executar
        Endereco endereco = viaCepService.buscar("01310-100");

        // Verificar
        assertThat(endereco.logradouro()).isEqualTo("Avenida Paulista");
        assertThat(endereco.uf()).isEqualTo("SP");

        // Verificar que a chamada foi feita
        verify(1, getRequestedFor(urlEqualTo("/ws/01310-100/json/")));
    }

    @Test
    void deveLancarExcecaoQuandoCepInvalido() {
        stubFor(get(urlMatching("/ws/.*/json/"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withBody("{\"erro\": true}")));

        assertThatThrownBy(() -> viaCepService.buscar("00000-000"))
                .isInstanceOf(CepNaoEncontradoException.class);
    }

    @Test
    void deveLidarComTimeoutDaAPI() {
        stubFor(get(urlMatching("/ws/.*/json/"))
                .willReturn(aResponse()
                        .withFixedDelay(5000) // 5 segundos de delay
                        .withStatus(200)));

        assertThatThrownBy(() -> viaCepService.buscar("01310-100"))
                .isInstanceOf(TimeoutException.class);
    }
}
```

---

## Exemplo Completo — Teste de Integração com Testcontainers

### Configuração do `application-test.yml`

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
  flyway:
    enabled: true
    clean-on-validation-error: true  # limpa o banco se migração falhar (apenas em teste)
```

### Teste de integração completo

```java
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.http.MediaType;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.web.servlet.MockMvc;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.assertj.core.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
@Testcontainers
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class ProdutoIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @Autowired ProdutoRepository repository;

    @AfterEach
    void limpar() {
        repository.deleteAll();
    }

    @Test
    @Order(1)
    @DisplayName("Fluxo completo: criar, buscar e deletar produto")
    void fluxoCompletoDeUmProduto() throws Exception {
        // 1. Criar produto
        ProdutoRequest request = new ProdutoRequest(
                "Notebook Pro", new BigDecimal("5999.00"), "Eletrônicos", 20, null);

        String responseBody = mockMvc.perform(post("/api/v1/produtos")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").isNumber())
                .andReturn()
                .getResponse()
                .getContentAsString();

        Long id = objectMapper.readTree(responseBody).get("id").asLong();

        // 2. Buscar produto criado
        mockMvc.perform(get("/api/v1/produtos/" + id))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.nome").value("Notebook Pro"))
                .andExpect(jsonPath("$.preco").value(5999.00))
                .andExpect(jsonPath("$.categoria").value("Eletrônicos"));

        // 3. Verificar no banco diretamente
        assertThat(repository.findById(id)).isPresent();
        assertThat(repository.count()).isEqualTo(1);

        // 4. Deletar
        mockMvc.perform(delete("/api/v1/produtos/" + id))
                .andExpect(status().isNoContent());

        // 5. Verificar que foi deletado
        mockMvc.perform(get("/api/v1/produtos/" + id))
                .andExpect(status().isNotFound());

        assertThat(repository.findById(id)).isEmpty();
    }

    @Test
    @DisplayName("Buscar produto inexistente deve retornar 404 com mensagem")
    void buscarInexistente_deveRetornar404ComMensagem() throws Exception {
        mockMvc.perform(get("/api/v1/produtos/99999")
                        .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.status").value(404))
                .andExpect(jsonPath("$.mensagem").value(containsString("99999")));
    }

    @Test
    @DisplayName("Criar produto com dados inválidos deve retornar 422 com erros de campo")
    void criarComDadosInvalidos_deveRetornar422() throws Exception {
        String jsonInvalido = """
                {
                    "nome": "",
                    "preco": -5.00,
                    "categoria": "",
                    "estoque": -1
                }
                """;

        mockMvc.perform(post("/api/v1/produtos")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(jsonInvalido))
                .andExpect(status().isUnprocessableEntity())
                .andExpect(jsonPath("$.campos.nome").exists())
                .andExpect(jsonPath("$.campos.preco").exists())
                .andExpect(jsonPath("$.campos.categoria").exists())
                .andExpect(jsonPath("$.campos.estoque").exists());
    }

    @Test
    @DisplayName("Listar produtos deve paginar corretamente")
    void listar_devePaginarCorretamente() throws Exception {
        // Inserir 15 produtos
        for (int i = 1; i <= 15; i++) {
            ProdutoRequest req = new ProdutoRequest("Produto " + i,
                    new BigDecimal("10.00"), "Geral", 10, null);
            mockMvc.perform(post("/api/v1/produtos")
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(req)))
                    .andExpect(status().isCreated());
        }

        // Buscar primeira página com 10 itens
        mockMvc.perform(get("/api/v1/produtos")
                        .param("pagina", "0")
                        .param("tamanho", "10"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content").isArray())
                .andExpect(jsonPath("$.content.length()").value(10))
                .andExpect(jsonPath("$.totalElements").value(15))
                .andExpect(jsonPath("$.totalPages").value(2))
                .andExpect(jsonPath("$.first").value(true))
                .andExpect(jsonPath("$.last").value(false));

        // Buscar segunda página
        mockMvc.perform(get("/api/v1/produtos")
                        .param("pagina", "1")
                        .param("tamanho", "10"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content.length()").value(5))
                .andExpect(jsonPath("$.last").value(true));
    }
}
```

### Executando os testes

```bash
# Rodar todos os testes (unitários e de integração)
mvn test

# Rodar apenas testes de integração (marcados com @Tag)
mvn test -Dgroups=integracao

# Rodar com log do Testcontainers visível
mvn test -Dtestcontainers.reuse.enable=true

# Pular testes de integração
mvn test -DexcludedGroups=integracao
```

> **Atenção:** Testcontainers requer Docker instalado e rodando na máquina de desenvolvimento e no ambiente de CI/CD. Em pipelines como GitHub Actions, use o `ubuntu-latest` que já vem com Docker disponível.
