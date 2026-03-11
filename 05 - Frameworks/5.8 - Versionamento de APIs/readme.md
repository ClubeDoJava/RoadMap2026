# 5.8 — Versionamento de APIs

Você lançou `/api/v1/produtos`. Seis meses depois, um cliente externo usa esse endpoint em produção, e você precisa mudar o formato da resposta. Sem versionamento, você quebra esse cliente. Com versionamento ruim, você mantém código legado para sempre sem estratégia de deprecação. Esta seção discute as abordagens existentes, seus trade-offs, e como implementar versionamento de forma que permita evolução real da API.

---

## Por que `/api/v1/` não é suficiente

Colocar `v1` na URL é um começo, mas não resolve o problema por si só. Versionamento de API é uma decisão arquitetural que exige responder: quando uma mudança justifica uma nova versão? Qual é a política de deprecação? Quantas versões você mantém em paralelo? Por quanto tempo?

Uma mudança que quebra contratos existentes (breaking change) justifica nova versão:
- Remover um campo da resposta que clientes dependem
- Mudar o tipo de um campo (string → objeto)
- Mudar a semântica de um parâmetro
- Remover um endpoint

Uma mudança aditiva não precisa de nova versão:
- Adicionar um campo opcional na resposta
- Adicionar um endpoint novo
- Adicionar um parâmetro opcional com valor default

---

## Estratégia 1: Versionamento via URL

A mais comum e a mais visível. Simples de entender, simples de rotear, simples de testar em navegador.

```java
// Controller v1
@RestController
@RequestMapping("/api/v1/produtos")
public class ProdutoV1Controller {

    @GetMapping("/{id}")
    public ProdutoV1Response buscar(@PathVariable Long id) {
        return produtoService.buscarV1(id);
    }
}

// Controller v2 — resposta tem estrutura diferente
@RestController
@RequestMapping("/api/v2/produtos")
public class ProdutoV2Controller {

    @GetMapping("/{id}")
    public ProdutoV2Response buscar(@PathVariable Long id) {
        return produtoService.buscarV2(id);
    }
}
```

```java
// Resposta v1 — preço como double
public record ProdutoV1Response(Long id, String nome, double preco) {}

// Resposta v2 — preço como objeto com valor e moeda
public record ProdutoV2Response(Long id, String nome, PrecoResponse preco, String categoria) {}
public record PrecoResponse(BigDecimal valor, String moeda) {}
```

Vantagens: intuitivo, cacheável por proxies/CDNs, fácil de deprecar (redirect 301 para nova versão, depois desativar a rota antiga). Desvantagens: a URL deixa de representar apenas o recurso — carrega metadado de versão junto.

---

## Estratégia 2: Versionamento via Header

A versão é informada via request header, mantendo a URL limpa.

```java
@RestController
@RequestMapping("/api/produtos")
public class ProdutoController {

    @GetMapping(value = "/{id}", headers = "API-Version=1")
    public ProdutoV1Response buscarV1(@PathVariable Long id) {
        return produtoService.buscarV1(id);
    }

    @GetMapping(value = "/{id}", headers = "API-Version=2")
    public ProdutoV2Response buscarV2(@PathVariable Long id) {
        return produtoService.buscarV2(id);
    }
}
```

```bash
# Uso
curl -H "API-Version: 2" http://api.empresa.com/api/produtos/42
```

Vantagens: URL limpa, versão é realmente um metadado de protocolo. Desvantagens: não é cacheável por padrão (proxies ignoram headers customizados, precisam ser configurados com `Vary: API-Version`), não funciona em navegador sem extensão/Postman, mais difícil de debugar.

---

## Estratégia 3: Versionamento via Content Negotiation

Usa o mecanismo padrão HTTP de negociação de conteúdo com media type customizado.

```java
@RestController
@RequestMapping("/api/produtos")
public class ProdutoController {

    @GetMapping(value = "/{id}",
                produces = "application/vnd.empresa.produto.v1+json")
    public ProdutoV1Response buscarV1(@PathVariable Long id) {
        return produtoService.buscarV1(id);
    }

    @GetMapping(value = "/{id}",
                produces = "application/vnd.empresa.produto.v2+json")
    public ProdutoV2Response buscarV2(@PathVariable Long id) {
        return produtoService.buscarV2(id);
    }
}
```

```bash
# Uso
curl -H "Accept: application/vnd.empresa.produto.v2+json" \
     http://api.empresa.com/api/produtos/42
```

Essa é a abordagem mais "correta" do ponto de vista REST puro — o tipo de mídia descreve o que você quer, não a versão do endpoint. Na prática, é verbosa, exige disciplina dos consumidores e tooling específico para funcionar bem com OpenAPI/Swagger.

---

## Estratégia 4: Versionamento via Query Parameter

```java
@GetMapping("/produtos/{id}")
public ResponseEntity<?> buscar(@PathVariable Long id,
                                  @RequestParam(defaultValue = "2") int version) {
    return switch (version) {
        case 1 -> ResponseEntity.ok(produtoService.buscarV1(id));
        case 2 -> ResponseEntity.ok(produtoService.buscarV2(id));
        default -> ResponseEntity.badRequest()
            .body(Map.of("erro", "Versão " + version + " não suportada"));
    };
}
```

```bash
curl http://api.empresa.com/api/produtos/42?version=2
```

Simples de implementar e de testar, mas polui a URL do recurso com parâmetro de protocolo. Útil para APIs internas ou scripts de migração, não recomendado para APIs públicas.

---

## Qual estratégia usar

Para APIs públicas com clientes externos: versionamento via URL é o padrão mais amplamente adotado. É o que GitHub, Stripe, Twilio e a maioria das APIs REST de referência usam. O custo de manter `/v1/` e `/v2/` em paralelo existe de qualquer forma — a URL explícita pelo menos torna isso visível.

Para APIs internas entre microsserviços: header ou Content Negotiation podem fazer mais sentido, já que você controla todos os clientes e pode coordenar a migração.

---

## Política de deprecação

Ter versões numeradas sem política de deprecação é acumular dívida técnica. Defina e comunique:

```java
// Marcar endpoints como deprecated no OpenAPI
@GetMapping("/{id}")
@Deprecated
@Operation(
    summary = "Buscar produto por ID (deprecated)",
    description = "Deprecated desde 2026-01. Use /api/v2/produtos/{id}. " +
                  "Será removido em 2027-01.",
    deprecated = true
)
@ApiResponse(
    responseCode = "200",
    headers = @Header(
        name = "Deprecation",
        description = "Timestamp de quando a versão foi deprecada",
        schema = @Schema(type = "string")
    )
)
public ProdutoV1Response buscar(@PathVariable Long id, HttpServletResponse response) {
    // Adiciona headers de deprecação em cada resposta
    response.addHeader("Deprecation", "2026-01-01");
    response.addHeader("Sunset", "2027-01-01");  // data de remoção planejada
    response.addHeader("Link", "</api/v2/produtos/" + id + ">; rel=\"successor-version\"");

    return produtoService.buscarV1(id);
}
```

O header `Sunset` (RFC 8594) informa quando o endpoint será desativado. Clientes bem implementados podem ler esse header e alertar seus times antes da remoção.

---

## Estrutura de projeto para múltiplas versões

```
src/main/java/com/empresa/api/
├── v1/
│   ├── controller/
│   │   └── ProdutoV1Controller.java
│   ├── dto/
│   │   ├── ProdutoV1Request.java
│   │   └── ProdutoV1Response.java
│   └── mapper/
│       └── ProdutoV1Mapper.java
├── v2/
│   ├── controller/
│   │   └── ProdutoV2Controller.java
│   ├── dto/
│   │   ├── ProdutoV2Request.java
│   │   └── ProdutoV2Response.java
│   └── mapper/
│       └── ProdutoV2Mapper.java
└── shared/
    ├── service/
    │   └── ProdutoService.java    ← lógica de negócio compartilhada
    └── domain/
        └── Produto.java           ← entidade compartilhada
```

A lógica de negócio não deve ser duplicada — apenas os contratos de entrada e saída (DTOs e controllers) variam por versão. O `ProdutoService` serve ambas as versões, e os mappers de cada versão traduzem entre o domínio e o formato específico da versão.

---

## Spec-First com OpenAPI

Uma prática que evita quebrar contratos sem perceber é definir a especificação OpenAPI antes do código e gerar os stubs a partir dela. Quando a spec é a fonte de verdade, qualquer mudança breaking fica explícita no diff.

```yaml
# openapi.yml — defina a API antes de implementar
openapi: 3.1.0
info:
  title: API de Produtos
  version: "2.0"

paths:
  /api/v2/produtos/{id}:
    get:
      summary: Buscar produto por ID
      operationId: buscarProduto
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        "200":
          description: Produto encontrado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProdutoV2Response'
        "404":
          description: Produto não encontrado

components:
  schemas:
    ProdutoV2Response:
      type: object
      required: [id, nome, preco]
      properties:
        id:
          type: integer
        nome:
          type: string
        preco:
          $ref: '#/components/schemas/Preco'
        categoria:
          type: string

    Preco:
      type: object
      required: [valor, moeda]
      properties:
        valor:
          type: number
          format: decimal
        moeda:
          type: string
          example: BRL
```

Com o plugin `openapi-generator-maven-plugin`, você gera interfaces Java a partir dessa spec e implementa os controllers contra as interfaces geradas — o compilador garante que o código está em conformidade com o contrato definido.
