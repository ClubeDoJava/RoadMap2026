# 5.4 — Documentação com OpenAPI e Swagger

Uma API sem documentação é uma API que ninguém consegue usar. OpenAPI é o padrão da indústria para descrever APIs REST. Swagger UI é a interface gráfica que permite explorar e testar a API diretamente no browser.

---

## Por que documentar APIs?

- **Onboarding rápido:** novos desenvolvedores entendem a API sem precisar ler o código
- **Contrato:** frontend e backend podem trabalhar em paralelo com base no contrato OpenAPI
- **Teste interativo:** Swagger UI permite testar endpoints sem ferramentas externas
- **Geração de clientes:** o arquivo `openapi.json` pode gerar SDKs automaticamente para várias linguagens

---

## springdoc-openapi

A biblioteca `springdoc-openapi` gera automaticamente a documentação OpenAPI 3.0 inspecionando os controllers Spring MVC. Não é necessário escrever YAML manualmente.

### Dependência Maven

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

### Acesso automático

Após adicionar a dependência e subir a aplicação:

- **Swagger UI:** `http://localhost:8080/swagger-ui.html`
- **Spec OpenAPI (JSON):** `http://localhost:8080/v3/api-docs`
- **Spec OpenAPI (YAML):** `http://localhost:8080/v3/api-docs.yaml`

Nenhuma configuração adicional é necessária para o caso básico.

### Personalizar URLs no `application.yml`

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
    enabled: true
  swagger-ui:
    path: /swagger-ui.html
    enabled: true
    operations-sorter: method     # ordena por método HTTP
    tags-sorter: alpha             # ordena tags alfabeticamente
    try-it-out-enabled: true       # habilita "Try it out" por padrão
    display-request-duration: true # exibe tempo de resposta
  show-actuator: false             # não exibe endpoints do Actuator
```

---

## Anotações OpenAPI

### `@Tag` — agrupar endpoints

```java
import io.swagger.v3.oas.annotations.tags.Tag;

@RestController
@RequestMapping("/api/v1/produtos")
@Tag(name = "Produtos", description = "Endpoints para gerenciamento de produtos")
public class ProdutoController {
    // ...
}
```

### `@Operation` e `@ApiResponse`

```java
import io.swagger.v3.oas.annotations.*;
import io.swagger.v3.oas.annotations.responses.*;
import io.swagger.v3.oas.annotations.media.*;

@Operation(
    summary = "Buscar produto por ID",
    description = "Retorna os detalhes de um produto específico. Requer autenticação."
)
@ApiResponses({
    @ApiResponse(
        responseCode = "200",
        description = "Produto encontrado",
        content = @Content(schema = @Schema(implementation = ProdutoResponse.class))
    ),
    @ApiResponse(
        responseCode = "404",
        description = "Produto não encontrado",
        content = @Content(schema = @Schema(implementation = ErroResponse.class))
    ),
    @ApiResponse(responseCode = "401", description = "Não autenticado"),
    @ApiResponse(responseCode = "403", description = "Sem permissão")
})
@GetMapping("/{id}")
public ResponseEntity<ProdutoResponse> buscarPorId(@PathVariable Long id) {
    return ResponseEntity.ok(service.buscarPorId(id));
}
```

### `@Parameter`

```java
import io.swagger.v3.oas.annotations.Parameter;

@Operation(summary = "Listar produtos com filtros e paginação")
@GetMapping
public ResponseEntity<Page<ProdutoResponse>> listar(
        @Parameter(description = "Número da página (0-indexed)", example = "0")
        @RequestParam(defaultValue = "0") int pagina,

        @Parameter(description = "Quantidade de itens por página", example = "10")
        @RequestParam(defaultValue = "10") int tamanho,

        @Parameter(description = "Filtrar por categoria")
        @RequestParam(required = false) String categoria
) {
    return ResponseEntity.ok(service.listar(pagina, tamanho, categoria));
}
```

### `@Schema` em DTOs

```java
import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.*;

@Schema(description = "Dados para criação ou atualização de produto")
public record ProdutoRequest(

    @Schema(description = "Nome do produto", example = "Notebook Dell XPS 15",
            minLength = 2, maxLength = 100)
    @NotBlank
    String nome,

    @Schema(description = "Preço em reais", example = "4999.90", minimum = "0.01")
    @NotNull
    @DecimalMin("0.01")
    BigDecimal preco,

    @Schema(description = "Categoria do produto", example = "Eletrônicos",
            allowableValues = {"Eletrônicos", "Móveis", "Vestuário", "Alimentos"})
    @NotBlank
    String categoria,

    @Schema(description = "Quantidade em estoque", example = "50", minimum = "0")
    @Min(0)
    int estoque

) {}

@Schema(description = "Dados retornados de um produto")
public record ProdutoResponse(

    @Schema(description = "Identificador único", example = "1")
    Long id,

    @Schema(description = "Nome do produto", example = "Notebook Dell XPS 15")
    String nome,

    @Schema(description = "Preço formatado", example = "4999.90")
    BigDecimal preco,

    String categoria,
    int estoque,

    @Schema(description = "Data e hora de cadastro (ISO 8601)",
            example = "2026-03-09T14:30:00")
    LocalDateTime criadoEm

) {}
```

### `@SecurityRequirement` — endpoints protegidos

```java
import io.swagger.v3.oas.annotations.security.SecurityRequirement;

@Operation(summary = "Deletar produto")
@SecurityRequirement(name = "bearerAuth") // nome definido na config global
@DeleteMapping("/{id}")
public ResponseEntity<Void> deletar(@PathVariable Long id) {
    service.deletar(id);
    return ResponseEntity.noContent().build();
}
```

---

## Configuração Completa da Documentação

```java
import io.swagger.v3.oas.models.*;
import io.swagger.v3.oas.models.info.*;
import io.swagger.v3.oas.models.security.*;
import io.swagger.v3.oas.models.servers.*;
import org.springframework.context.annotation.*;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("API de Produtos")
                        .description("""
                                API REST para gerenciamento de produtos.

                                ## Autenticação
                                Esta API usa JWT Bearer Token.
                                Para autenticar, faça POST em `/api/v1/auth/login`
                                e use o token retornado no header `Authorization: Bearer <token>`.
                                """)
                        .version("v1.0.0")
                        .contact(new Contact()
                                .name("Time de Backend")
                                .email("backend@empresa.com")
                                .url("https://empresa.com"))
                        .license(new License()
                                .name("Apache 2.0")
                                .url("https://www.apache.org/licenses/LICENSE-2.0")))

                .addServersItem(new Server()
                        .url("http://localhost:8080")
                        .description("Desenvolvimento"))
                .addServersItem(new Server()
                        .url("https://api.empresa.com")
                        .description("Produção"))

                // Configuração global de segurança (JWT)
                .components(new Components()
                        .addSecuritySchemes("bearerAuth",
                                new SecurityScheme()
                                        .type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer")
                                        .bearerFormat("JWT")
                                        .description("Token JWT obtido no endpoint de login")))

                // Aplicar segurança globalmente (todos os endpoints requerem token)
                .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
    }
}
```

### Excluir endpoints do Swagger no `application.yml`

```yaml
springdoc:
  paths-to-exclude: /api/v1/internal/**, /actuator/**
  packages-to-scan: com.exemplo.app.controller
```

---

## Gerar o Arquivo `openapi.json` / `openapi.yaml`

O arquivo pode ser usado para:
- Gerar clientes SDK (TypeScript, Python, etc.) com ferramentas como `openapi-generator`
- Configurar API Gateway (AWS API Gateway, Kong)
- Validação de contratos entre times

```bash
# Baixar o spec em JSON enquanto a aplicação está rodando
curl http://localhost:8080/v3/api-docs -o openapi.json

# Baixar em YAML
curl http://localhost:8080/v3/api-docs.yaml -o openapi.yaml
```

### Gerar spec sem subir o servidor (via Maven plugin)

```xml
<plugin>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-maven-plugin</artifactId>
    <version>1.4</version>
    <executions>
        <execution>
            <id>integration-test</id>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <apiDocsUrl>http://localhost:8080/v3/api-docs</apiDocsUrl>
        <outputFileName>openapi.json</outputFileName>
        <outputDir>${project.build.directory}</outputDir>
    </configuration>
</plugin>
```

---

## Versionamento de API

### Via path (mais simples e explícito)

```java
// v1
@RestController
@RequestMapping("/api/v1/produtos")
public class ProdutoV1Controller {}

// v2 — quando o contrato mudou incompativelmente
@RestController
@RequestMapping("/api/v2/produtos")
public class ProdutoV2Controller {}
```

### Via header (mais limpo, URLs menores)

```java
@GetMapping(headers = "Accept-Version=v2")
public ResponseEntity<ProdutoV2Response> buscarV2(@PathVariable Long id) {}

// Chamada: GET /api/produtos/1 com header Accept-Version: v2
```

### Documentar múltiplas versões com springdoc

```java
@Bean
public GroupedOpenApi apiV1() {
    return GroupedOpenApi.builder()
            .group("v1")
            .pathsToMatch("/api/v1/**")
            .build();
}

@Bean
public GroupedOpenApi apiV2() {
    return GroupedOpenApi.builder()
            .group("v2")
            .pathsToMatch("/api/v2/**")
            .build();
}
```

---

## Exemplo Completo — API de Produtos com Autenticação JWT Documentada

```java
@RestController
@RequestMapping("/api/v1/produtos")
@Tag(name = "Produtos", description = "CRUD de produtos do catálogo")
public class ProdutoController {

    private final ProdutoService service;

    public ProdutoController(ProdutoService service) {
        this.service = service;
    }

    @Operation(summary = "Listar produtos", description = "Retorna lista paginada de produtos ativos")
    @ApiResponse(responseCode = "200", description = "Lista retornada com sucesso")
    @GetMapping
    public ResponseEntity<Page<ProdutoResponse>> listar(
            @Parameter(description = "Página (0-indexed)", example = "0")
            @RequestParam(defaultValue = "0") int pagina,

            @Parameter(description = "Tamanho da página", example = "10")
            @RequestParam(defaultValue = "10") int tamanho) {

        return ResponseEntity.ok(service.listar(pagina, tamanho));
    }

    @Operation(summary = "Buscar produto por ID")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Produto encontrado"),
        @ApiResponse(responseCode = "404", description = "Produto não encontrado",
            content = @Content(schema = @Schema(implementation = ErroResponse.class)))
    })
    @GetMapping("/{id}")
    public ResponseEntity<ProdutoResponse> buscar(
            @Parameter(description = "ID do produto", example = "1")
            @PathVariable Long id) {
        return ResponseEntity.ok(service.buscarPorId(id));
    }

    @Operation(summary = "Criar produto")
    @SecurityRequirement(name = "bearerAuth")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Produto criado"),
        @ApiResponse(responseCode = "422", description = "Dados inválidos",
            content = @Content(schema = @Schema(implementation = ErroResponse.class))),
        @ApiResponse(responseCode = "401", description = "Não autenticado")
    })
    @PostMapping
    public ResponseEntity<ProdutoResponse> criar(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Dados do produto",
                required = true,
                content = @Content(
                    schema = @Schema(implementation = ProdutoRequest.class),
                    examples = @ExampleObject(value = """
                        {
                            "nome": "Notebook Dell XPS 15",
                            "preco": 4999.90,
                            "categoria": "Eletrônicos",
                            "estoque": 10
                        }
                        """)
                )
            )
            @Valid @RequestBody ProdutoRequest request) {

        ProdutoResponse criado = service.criar(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(criado);
    }

    @Operation(summary = "Deletar produto (somente ADMIN)")
    @SecurityRequirement(name = "bearerAuth")
    @ApiResponses({
        @ApiResponse(responseCode = "204", description = "Produto deletado"),
        @ApiResponse(responseCode = "404", description = "Produto não encontrado"),
        @ApiResponse(responseCode = "403", description = "Sem permissão (requer ADMIN)")
    })
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deletar(@PathVariable Long id) {
        service.deletar(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Resultado no Swagger UI

Após subir a aplicação, acesse `http://localhost:8080/swagger-ui.html` para ver:

- Título, versão e descrição da API
- Servidores disponíveis (dev e prod)
- Endpoints agrupados por tag (`Produtos`, `Autenticação`, etc.)
- Botão "Authorize" para inserir o token JWT
- Para cada endpoint: parâmetros, corpo da requisição com exemplo, e todos os possíveis responses
- Botão "Try it out" para testar diretamente no browser
