# 7.3 - Ferramentas de Produtividade

## Lombok

Lombok elimina boilerplate de Java (getters, setters, construtores, equals/hashCode, toString) gerando-os em tempo de compilação via processamento de anotações.

### Dependência

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

### Anotações essenciais

**@Data** — gera getters, setters, equals, hashCode, toString e construtor com campos obrigatórios:

```java
@Data
public class ProdutoDTO {
    private Long id;
    private String nome;
    private BigDecimal preco;
    private String categoria;
}
// Equivale a escrever manualmente ~80 linhas de código
```

**@Builder** — gera o padrão Builder:

```java
@Builder
@Data
public class EmailRequest {
    private String destinatario;
    private String assunto;
    private String corpo;
    @Builder.Default
    private String remetente = "noreply@minhaapp.com";
    @Singular  // Para coleções: adiciona .anexo(a) e .anexos(lista)
    private List<String> anexos;
}

// Uso fluente
EmailRequest email = EmailRequest.builder()
        .destinatario("joao@email.com")
        .assunto("Confirmação de pedido")
        .corpo("Seu pedido #42 foi aprovado!")
        .anexo("nota-fiscal.pdf")
        .build();
```

**@RequiredArgsConstructor** — gera construtor apenas para campos `final` e `@NonNull`:

```java
@Service
@RequiredArgsConstructor  // Gerado pelo Lombok — padrão recomendado no Spring
public class PedidoService {
    private final PedidoRepository repository;        // final → entra no construtor
    private final NotificacaoService notificacao;     // final → entra no construtor
    private int contadorRequisicoes = 0;              // não-final → não entra

    // Equivale a:
    // public PedidoService(PedidoRepository repository, NotificacaoService notificacao) { ... }
}
```

**@Slf4j** — injeta um logger `log` na classe:

```java
@Slf4j
@Service
public class PagamentoService {
    public void processar(Pagamento pagamento) {
        log.info("Processando pagamento de R${}", pagamento.getValor());
        try {
            // ...
            log.debug("Pagamento {} aprovado", pagamento.getId());
        } catch (Exception e) {
            log.error("Erro ao processar pagamento {}", pagamento.getId(), e);
        }
    }
}
```

**@Value** — classe imutável (todos os campos `private final`, sem setters):

```java
@Value  // Atenção: é o @Value do Lombok, não do Spring!
public class Dinheiro {
    BigDecimal valor;
    String moeda;
}
// Todos os campos são private final, construtor com todos os campos, getters sem set
```

**@AllArgsConstructor e @NoArgsConstructor:**

```java
@Entity
@Data
@NoArgsConstructor               // JPA exige construtor padrão
@AllArgsConstructor              // Útil para testes e mapeamentos
public class PedidoEntity {
    @Id @GeneratedValue
    private Long id;
    private String clienteId;
    private BigDecimal total;
}
```

### Quando usar e quando evitar

| Anotação | Usar | Evitar |
|----------|------|--------|
| `@Slf4j` | Sempre | Nunca |
| `@RequiredArgsConstructor` | Em `@Service`, `@Component` | Entidades JPA |
| `@Builder` | DTOs, Value Objects, testes | Entidades JPA (pode causar problemas com Hibernate) |
| `@Data` em DTOs | Sim | Em entidades JPA: `equals/hashCode` baseado em ID é problemático |
| `@EqualsAndHashCode` | Quando precisar personalizar | Não use com `@Data` em entidades |
| `@Value` | Value Objects imutáveis | Quando precisar de mutabilidade |

**Cuidado com `@Data` em entidades JPA:**

```java
// PROBLEMÁTICO: @Data gera equals/hashCode baseado em todos os campos
// Pode causar StackOverflow em relacionamentos bidirecionais
@Entity
@Data  // Evite isso em entidades JPA
public class Pedido {
    @OneToMany(mappedBy = "pedido")
    private List<Item> itens; // @Data vai incluir itens no hashCode → loop infinito
}

// MELHOR: controle explícito
@Entity
@Getter
@Setter
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Pedido {
    @Id
    @EqualsAndHashCode.Include  // Apenas o ID determina igualdade
    private Long id;

    @OneToMany(mappedBy = "pedido")
    private List<Item> itens;
}
```

---

## MapStruct

MapStruct gera código de mapeamento entre objetos (DTOs, entidades, etc.) em tempo de compilação. É muito mais rápido que reflexão (como ModelMapper).

### Dependência

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.6.2</version>
</dependency>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.6.2</version>
            </path>
            <!-- Se usar Lombok junto com MapStruct, o Lombok vem ANTES -->
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok-mapstruct-binding</artifactId>
                <version>0.2.0</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

---

## Exemplo completo: Entidade → DTO → Entidade com MapStruct

### Classes envolvidas

```java
// Entidade JPA
@Entity
@Table(name = "clientes")
@Getter @Setter @NoArgsConstructor
public class ClienteEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nome;
    private String email;

    @Column(name = "data_nascimento")
    private LocalDate dataNascimento;

    @Embedded
    private EnderecoEntity endereco;

    @Column(name = "criado_em")
    private LocalDateTime criadoEm;

    @Column(name = "ativo")
    private boolean ativo = true;
}

// Embedded do endereço
@Embeddable
@Getter @Setter
public class EnderecoEntity {
    private String logradouro;
    private String numero;
    private String cep;
    private String cidade;
    private String estado;
}

// DTO de resposta (retornado pela API)
public record ClienteResponse(
    Long id,
    String nome,
    String email,
    int idade,            // Calculado, não existe na entidade
    String enderecoCompleto, // Formatado, diferente da entidade
    boolean ativo
) {}

// DTO de criação (recebido pela API)
public record CriarClienteRequest(
    @NotBlank String nome,
    @Email String email,
    @NotNull LocalDate dataNascimento,
    @NotNull EnderecoRequest endereco
) {}

public record EnderecoRequest(
    String logradouro,
    String numero,
    String cep,
    String cidade,
    String estado
) {}
```

### Mapper com MapStruct

```java
@Mapper(componentModel = "spring")  // Gera um @Component do Spring
public interface ClienteMapper {

    // Mapeamento simples (campos com mesmo nome são automáticos)
    @Mapping(target = "idade", expression = "java(calcularIdade(entity.getDataNascimento()))")
    @Mapping(target = "enderecoCompleto", expression = "java(formatarEndereco(entity.getEndereco()))")
    ClienteResponse toResponse(ClienteEntity entity);

    // Mapeamento de lista
    List<ClienteResponse> toResponseList(List<ClienteEntity> entities);

    // Request → Entity (para criação)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "criadoEm", expression = "java(java.time.LocalDateTime.now())")
    @Mapping(target = "ativo", constant = "true")
    @Mapping(source = "endereco", target = "endereco")
    ClienteEntity toEntity(CriarClienteRequest request);

    // Mapeamento do endereço (campos com mesmo nome)
    EnderecoEntity toEntity(EnderecoRequest request);

    // Método auxiliar para calcular idade
    default int calcularIdade(LocalDate nascimento) {
        return Period.between(nascimento, LocalDate.now()).getYears();
    }

    // Método auxiliar para formatar endereço
    default String formatarEndereco(EnderecoEntity end) {
        if (end == null) return null;
        return String.format("%s, %s - %s/%s - CEP: %s",
                end.getLogradouro(), end.getNumero(),
                end.getCidade(), end.getEstado(), end.getCep());
    }
}
```

### Código gerado pelo MapStruct (para entender o que acontece)

```java
// Código gerado automaticamente em target/generated-sources/
@Component
public class ClienteMapperImpl implements ClienteMapper {

    @Override
    public ClienteResponse toResponse(ClienteEntity entity) {
        if (entity == null) return null;

        return new ClienteResponse(
            entity.getId(),
            entity.getNome(),
            entity.getEmail(),
            calcularIdade(entity.getDataNascimento()),
            formatarEndereco(entity.getEndereco()),
            entity.isAtivo()
        );
    }
    // ...
}
```

### Usando o mapper no serviço

```java
@Service
@RequiredArgsConstructor
public class ClienteService {

    private final ClienteRepository repository;
    private final ClienteMapper mapper;

    public ClienteResponse criar(CriarClienteRequest request) {
        ClienteEntity entity = mapper.toEntity(request);
        ClienteEntity salvo = repository.save(entity);
        return mapper.toResponse(salvo);
    }

    public ClienteResponse buscarPorId(Long id) {
        return repository.findById(id)
                .map(mapper::toResponse)
                .orElseThrow(() -> new ClienteNaoEncontradoException(id));
    }

    public List<ClienteResponse> listarTodos() {
        return mapper.toResponseList(repository.findAll());
    }
}
```

### Mapeamentos avançados

```java
@Mapper(componentModel = "spring")
public interface PedidoMapper {

    // Mapear campos com nomes diferentes
    @Mapping(source = "clienteId.valor", target = "clienteId")
    @Mapping(source = "valorTotal.valor", target = "totalCentavos",
             qualifiedByName = "paracentavos")
    @Mapping(source = "status", target = "statusDescricao",
             qualifiedByName = "traduzirStatus")
    PedidoResponse toResponse(Pedido pedido);

    @Named("paraCentavos")
    default long paraCentavos(BigDecimal valor) {
        return valor.multiply(BigDecimal.valueOf(100)).longValue();
    }

    @Named("traduzirStatus")
    default String traduzirStatus(StatusPedido status) {
        return switch (status) {
            case RASCUNHO -> "Rascunho";
            case CONFIRMADO -> "Confirmado";
            case ENVIADO -> "Em trânsito";
            case ENTREGUE -> "Entregue";
            case CANCELADO -> "Cancelado";
        };
    }
}
```

---

## JShell: REPL do Java

JShell é o REPL (Read-Eval-Print Loop) do Java, disponível desde o Java 9. Permite experimentar código sem criar um projeto.

```bash
# Iniciar o JShell
jshell

# Com imports automáticos
jshell --startup DEFAULT
```

```java
// Experimentar expressões imediatamente
jshell> 2 + 2
$1 ==> 4

// Testar streams
jshell> List.of(1, 2, 3, 4, 5).stream().filter(n -> n % 2 == 0).toList()
$2 ==> [2, 4]

// Definir métodos
jshell> String saudar(String nome) {
   ...>     return "Olá, " + nome + "!";
   ...> }
|  created method saudar(String)

jshell> saudar("Java")
$3 ==> "Olá, Java!"

// Testar Records
jshell> record Ponto(int x, int y) { }
|  created record Ponto

jshell> new Ponto(3, 4)
$4 ==> Ponto[x=3, y=4]

// Ver variáveis definidas
jshell> /vars

// Ver métodos definidos
jshell> /methods

// Listar histórico
jshell> /list

// Sair
jshell> /exit
```

**Casos de uso práticos:**
- Testar expressões regulares sem criar um projeto
- Experimentar a API de Streams
- Verificar comportamento de métodos de String/Math
- Prototipar lógica rapidamente

---

## HTTPie e curl para testar APIs

### curl

```bash
# GET simples
curl http://localhost:8080/api/pedidos

# GET com headers e formatação JSON
curl -s -H "Accept: application/json" http://localhost:8080/api/pedidos | jq .

# POST com JSON
curl -X POST http://localhost:8080/api/pedidos \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGc..." \
  -d '{
    "clienteId": "abc-123",
    "itens": [{"produtoId": 1, "quantidade": 2}],
    "enderecoEntrega": "Rua das Flores, 123"
  }'

# PUT
curl -X PUT http://localhost:8080/api/pedidos/42 \
  -H "Content-Type: application/json" \
  -d '{"status": "CONFIRMADO"}'

# DELETE
curl -X DELETE http://localhost:8080/api/pedidos/42

# Ver headers de resposta
curl -I http://localhost:8080/actuator/health

# Seguir redirects
curl -L http://localhost:8080/api/redirect

# Salvar resposta em arquivo
curl -o resposta.json http://localhost:8080/api/pedidos
```

### HTTPie (mais amigável)

```bash
# Instalar
pip install httpie

# GET (simples e limpo)
http localhost:8080/api/pedidos

# POST com JSON (HTTPie infere Content-Type automaticamente)
http POST localhost:8080/api/pedidos \
  clienteId="abc-123" \
  enderecoEntrega="Rua das Flores, 123"

# Com array aninhado
http POST localhost:8080/api/pedidos \
  clienteId="abc-123" \
  itens:='[{"produtoId": 1, "quantidade": 2}]'

# Com Bearer token
http GET localhost:8080/api/pedidos \
  Authorization:"Bearer eyJhbGc..."

# Modo verbose (ver request e response completos)
http -v POST localhost:8080/api/pedidos clienteId="abc"

# Apenas status code
http --check-status localhost:8080/actuator/health
```

---

## Postman / Bruno

### Postman

Ferramenta gráfica para criar e organizar coleções de requisições HTTP. Ideal para equipes.

**Funcionalidades principais:**
- Organizar endpoints em coleções e pastas
- Variáveis de ambiente (desenvolvimento, staging, produção)
- Testes automáticos em JavaScript após cada requisição
- Documentação automática da API
- Mock servers para desenvolvimento sem backend

```javascript
// Test script no Postman (aba "Tests")
pm.test("Status 200", () => {
    pm.response.to.have.status(200);
});

pm.test("Resposta tem ID", () => {
    const json = pm.response.json();
    pm.expect(json).to.have.property("id");
});

// Salva o ID para usar em requisições seguintes
const pedidoId = pm.response.json().id;
pm.environment.set("pedidoId", pedidoId);
```

### Bruno (alternativa open-source)

Bruno é uma alternativa open-source ao Postman com uma abordagem diferente: **as coleções são arquivos de texto** (formato `.bru`) que ficam no repositório Git junto com o código.

```
minha-app/
├── src/
├── bruno-collection/              # Fica no Git!
│   ├── bruno.json
│   ├── environments/
│   │   ├── local.bru
│   │   └── prod.bru
│   └── pedidos/
│       ├── criar-pedido.bru
│       ├── buscar-pedido.bru
│       └── cancelar-pedido.bru
```

```
# criar-pedido.bru
meta {
  name: Criar Pedido
  type: http
  seq: 1
}

post {
  url: {{baseUrl}}/api/pedidos
  body: json
  auth: bearer
}

auth:bearer {
  token: {{token}}
}

body:json {
  {
    "clienteId": "abc-123",
    "itens": [
      {"produtoId": 1, "quantidade": 2}
    ],
    "enderecoEntrega": "Rua das Flores, 123"
  }
}

tests {
  test("Status 201", function() {
    expect(res.getStatus()).to.equal(201);
  });

  test("Resposta tem ID", function() {
    expect(res.getBody()).to.have.property("id");
  });
}
```

**Por que Bruno para projetos Java:**
- Coleção no Git = histórico de evolução da API junto com o código
- Sem dependência de conta em nuvem
- CLI (`bru run`) para rodar testes em CI/CD
- Formato texto = fácil de fazer diff no PR
