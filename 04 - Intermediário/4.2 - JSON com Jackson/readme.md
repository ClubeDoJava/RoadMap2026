# 4.2 — JSON com Jackson

Jackson é a biblioteca que o Spring Boot usa para converter objetos Java em JSON e vice-versa. Na maioria dos projetos, ela funciona transparentemente — até o dia em que não funciona. `UnrecognizedPropertyException` quando o JSON de entrada tem um campo que seu DTO não mapeia. `InvalidDefinitionException` porque o `ObjectMapper` não sabe serializar `LocalDateTime` sem o `JavaTimeModule`. `MismatchedInputException` ao deserializar um array para um tipo genérico sem `TypeReference`. Este guia cobre o suficiente para você entender o que está acontecendo quando essas exceções aparecem, e como configurar o Jackson corretamente desde o início.

---

## Dependência Maven

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.2</version>
</dependency>

<!-- Para suporte a tipos de data/hora do Java 8+ (LocalDate, LocalDateTime) -->
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.17.2</version>
</dependency>
```

> No Spring Boot, `jackson-databind` já é incluído automaticamente pelo `spring-boot-starter-web`. O `jackson-datatype-jsr310` também é configurado automaticamente se estiver no classpath.

---

## `ObjectMapper` — a classe principal

`ObjectMapper` é thread-safe após a configuração inicial. O ideal é criar uma instância única como bean (singleton) na aplicação.

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.databind.SerializationFeature;

ObjectMapper mapper = new ObjectMapper()
        .registerModule(new JavaTimeModule())
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
```

### Objeto → JSON (`writeValueAsString`)

```java
import com.fasterxml.jackson.databind.ObjectMapper;

record Produto(String nome, double preco, int estoque) {}

ObjectMapper mapper = new ObjectMapper();
Produto produto = new Produto("Notebook", 3500.00, 10);

String json = mapper.writeValueAsString(produto);
// {"nome":"Notebook","preco":3500.0,"estoque":10}

// Pretty print
String jsonFormatado = mapper.writerWithDefaultPrettyPrinter()
        .writeValueAsString(produto);
// {
//   "nome" : "Notebook",
//   "preco" : 3500.0,
//   "estoque" : 10
// }
```

### JSON → Objeto (`readValue`)

```java
String json = """
        {
            "nome": "Cadeira",
            "preco": 850.00,
            "estoque": 25
        }
        """;

Produto produto = mapper.readValue(json, Produto.class);
System.out.println(produto.nome());  // Cadeira
System.out.println(produto.preco()); // 850.0
```

### Objeto → Arquivo JSON (`writeValue`)

```java
import java.io.File;

Produto produto = new Produto("Monitor", 1200.00, 5);
mapper.writeValue(new File("/tmp/produto.json"), produto);

// Ler de volta do arquivo
Produto lido = mapper.readValue(new File("/tmp/produto.json"), Produto.class);
```

### Navegar JSON sem POJO (`readTree`)

Útil quando a estrutura do JSON é desconhecida ou variável.

```java
import com.fasterxml.jackson.databind.JsonNode;

String json = """
        {
            "usuario": {
                "nome": "Ana",
                "email": "ana@email.com",
                "roles": ["ADMIN", "USER"]
            },
            "ativo": true
        }
        """;

JsonNode root = mapper.readTree(json);

String nome = root.path("usuario").path("nome").asText();
String email = root.path("usuario").path("email").asText();
boolean ativo = root.path("ativo").asBoolean();

// Iterar array
JsonNode roles = root.path("usuario").path("roles");
for (JsonNode role : roles) {
    System.out.println(role.asText());
}

// Verificar se campo existe
if (root.has("usuario")) {
    System.out.println("Campo usuario encontrado");
}

// path() vs get(): path() retorna MissingNode (nunca null), get() pode retornar null
```

---

## Anotações Jackson

### `@JsonProperty` e `@JsonAlias`

```java
import com.fasterxml.jackson.annotation.*;

public class UsuarioDTO {

    @JsonProperty("nome_completo")  // nome no JSON é diferente do campo Java
    private String nomeCompleto;

    @JsonAlias({"email", "e_mail", "emailAddress"}) // aceita múltiplos nomes na leitura
    @JsonProperty("email")                           // mas escreve sempre como "email"
    private String email;
}
```

### `@JsonIgnore` e `@JsonIgnoreProperties`

```java
public class Usuario {

    private String nome;

    @JsonIgnore  // nunca serializa nem desserializa este campo
    private String senha;

    private String email;
}

// Para ignorar campos desconhecidos no JSON de entrada (evita UnrecognizedPropertyException)
@JsonIgnoreProperties(ignoreUnknown = true)
public class ProdutoDTO {
    private String nome;
    private double preco;
    // campos extras no JSON são ignorados
}
```

### `@JsonSerialize` e `@JsonDeserialize`

Para customizar completamente a conversão de um campo.

```java
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.core.*;
import java.io.IOException;
import java.math.BigDecimal;

// Serializer customizado: salva preço em centavos
public class PrecoEmCentavosSerializer extends JsonSerializer<BigDecimal> {
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen,
            SerializerProvider provider) throws IOException {
        gen.writeNumber(value.multiply(BigDecimal.valueOf(100)).longValue());
    }
}

// Deserializer customizado: lê centavos e converte para reais
public class CentavosParaRealDeserializer extends JsonDeserializer<BigDecimal> {
    @Override
    public BigDecimal deserialize(JsonParser p,
            DeserializationContext ctx) throws IOException {
        long centavos = p.getLongValue();
        return BigDecimal.valueOf(centavos).divide(BigDecimal.valueOf(100));
    }
}

public class Pedido {
    @JsonSerialize(using = PrecoEmCentavosSerializer.class)
    @JsonDeserialize(using = CentavosParaRealDeserializer.class)
    private BigDecimal total;
}
```

### `@JsonFormat` — datas

```java
import com.fasterxml.jackson.annotation.JsonFormat;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class EventoDTO {

    @JsonFormat(pattern = "dd/MM/yyyy")
    private LocalDate data;

    @JsonFormat(pattern = "dd/MM/yyyy HH:mm:ss")
    private LocalDateTime criadoEm;
}
```

### `@JsonInclude` — omitir campos nulos ou vazios

```java
import com.fasterxml.jackson.annotation.JsonInclude;

// Omite campos nulos apenas desta classe
@JsonInclude(JsonInclude.Include.NON_NULL)
public class RespostaDTO {
    private String dados;
    private String erro;       // omitido se null
    private List<String> tags; // omitido se null
}

// Outras opções úteis:
// NON_EMPTY — omite nulos, strings vazias, coleções vazias
// NON_DEFAULT — omite valores padrão do tipo (0, false, etc.)
```

---

## Tipos Genéricos com `TypeReference`

O problema: por causa do type erasure do Java, `mapper.readValue(json, List.class)` retorna `List<Object>` sem informação do tipo real. Use `TypeReference` para preservar o tipo genérico.

```java
import com.fasterxml.jackson.core.type.TypeReference;
import java.util.*;

// JSON de array
String jsonArray = """
        [
            {"nome": "Notebook", "preco": 3500.0},
            {"nome": "Mouse", "preco": 85.0}
        ]
        """;

// ERRADO: retorna List<LinkedHashMap>, não List<Produto>
List produtos = mapper.readValue(jsonArray, List.class);

// CORRETO: TypeReference preserva o tipo genérico
List<Produto> listaProdutos = mapper.readValue(jsonArray,
        new TypeReference<List<Produto>>() {});

// Para Map<String, Produto>
String jsonMap = """
        {
            "p1": {"nome": "Notebook", "preco": 3500.0},
            "p2": {"nome": "Mouse", "preco": 85.0}
        }
        """;

Map<String, Produto> mapaProdutos = mapper.readValue(jsonMap,
        new TypeReference<Map<String, Produto>>() {});

mapaProdutos.forEach((chave, produto) ->
        System.out.println(chave + " → " + produto.nome()));
```

---

## Records + Jackson

A partir do Java 16 e Jackson 2.12+, records funcionam nativamente como POJOs para serialização e desserialização. O Jackson usa o construtor canônico automaticamente.

```java
// Record simples — funciona sem anotações
record Produto(String nome, double preco, int estoque) {}

ObjectMapper mapper = new ObjectMapper();

// Serialização
Produto p = new Produto("Teclado", 250.0, 50);
String json = mapper.writeValueAsString(p);
// {"nome":"Teclado","preco":250.0,"estoque":50}

// Desserialização
Produto lido = mapper.readValue(json, Produto.class);

// Record com anotações
record UsuarioDTO(
    @JsonProperty("nome_completo") String nome,
    @JsonIgnore String senha,
    @JsonFormat(pattern = "dd/MM/yyyy") LocalDate nascimento
) {}
```

> Com Java 17+ e Jackson 2.15+, você pode usar `mapper.configure(MapperFeature.USE_ANNOTATIONS_IN_RECORDS, true)` para garantir que anotações em componentes de records sejam respeitadas. Na prática, já funciona por padrão.

---

## Configurações Úteis

```java
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

ObjectMapper mapper = new ObjectMapper()

    // Não falha se o JSON tiver campos que não existem no POJO
    .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)

    // Datas como string ISO, não como array de números
    .registerModule(new JavaTimeModule())
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)

    // Não falha se o objeto for vazio (sem campos serializáveis)
    .disable(SerializationFeature.FAIL_ON_EMPTY_BEANS)

    // Omitir nulls globalmente
    .setSerializationInclusion(JsonInclude.Include.NON_NULL)

    // Nomear campos em snake_case automaticamente
    .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
```

### Configuração como Bean no Spring Boot

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
```

---

## Avro e Protobuf — quando usar

| Formato | Uso ideal | Vantagem |
|---|---|---|
| **JSON (Jackson)** | APIs REST, configurações, logs | Legível, universal, sem schema obrigatório |
| **Avro** | Kafka, pipelines de dados, data lakes | Schema evolutivo, compressão, compatível com Spark/Hadoop |
| **Protobuf** | gRPC, microsserviços de alta performance | Binário compacto, tipagem forte, serialização ~3-10x mais rápida que JSON |

Use JSON para a grande maioria dos casos. Avro e Protobuf só valem o custo de complexidade quando o volume de dados ou a latência for realmente crítica.

---

## Exemplos Completos

### API de produto: serialização e desserialização com datas e validação

```java
import com.fasterxml.jackson.annotation.*;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import java.time.LocalDateTime;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = true)
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ProdutoDTO {

    private Long id;

    @JsonProperty("nome_produto")
    private String nome;

    private Double preco;

    @JsonFormat(pattern = "dd/MM/yyyy HH:mm")
    private LocalDateTime criadoEm;

    private List<String> tags;

    // construtores, getters, setters...
}

public class ExemploCompleto {

    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        // Objeto → JSON
        ProdutoDTO dto = new ProdutoDTO();
        dto.setId(1L);
        dto.setNome("Notebook Pro");
        dto.setPreco(4999.90);
        dto.setCriadoEm(LocalDateTime.now());
        dto.setTags(List.of("eletrônico", "importado"));

        String json = mapper.writerWithDefaultPrettyPrinter().writeValueAsString(dto);
        System.out.println(json);

        // JSON → Objeto
        String jsonEntrada = """
                {
                    "id": 2,
                    "nome_produto": "Mouse Gamer",
                    "preco": 299.90,
                    "criado_em": "15/01/2026 10:30",
                    "tags": ["periférico", "gamer"],
                    "campo_desconhecido": "ignorado"
                }
                """;

        ProdutoDTO lido = mapper.readValue(jsonEntrada, ProdutoDTO.class);
        System.out.println(lido.getNome());    // Mouse Gamer
        System.out.println(lido.getPreco());   // 299.9
        System.out.println(lido.getTags());    // [periférico, gamer]
    }
}
```

### Processando resposta de API externa com `readTree`

```java
import com.fasterxml.jackson.databind.*;
import java.net.URI;
import java.net.http.*;

public class ClienteViaCep {

    private static final ObjectMapper mapper = new ObjectMapper();

    public record Endereco(String cep, String logradouro, String bairro,
                           String localidade, String uf) {}

    public static Endereco buscarCep(String cep) throws Exception {
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://viacep.com.br/ws/" + cep + "/json/"))
                .GET()
                .build();

        HttpResponse<String> response = client.send(request,
                HttpResponse.BodyHandlers.ofString());

        JsonNode root = mapper.readTree(response.body());

        if (root.has("erro")) {
            throw new IllegalArgumentException("CEP não encontrado: " + cep);
        }

        return new Endereco(
                root.path("cep").asText(),
                root.path("logradouro").asText(),
                root.path("bairro").asText(),
                root.path("localidade").asText(),
                root.path("uf").asText()
        );
    }

    public static void main(String[] args) throws Exception {
        Endereco endereco = buscarCep("01310-100");
        System.out.printf("Logradouro: %s, %s - %s/%s%n",
                endereco.logradouro(), endereco.bairro(),
                endereco.localidade(), endereco.uf());
    }
}
```
