# 7.4 - Soft Skills e Carreira

## Code Review: como dar e receber feedback

### O que verificar em um Code Review

**Correção e lógica:**
- O código resolve o problema descrito no ticket/PR?
- Existem edge cases não tratados (null, lista vazia, valores negativos)?
- As condições de erro estão tratadas adequadamente?

**Segurança:**
- Existem SQL Injections, XSS, deserialização insegura?
- Dados sensíveis sendo logados?
- Autorização adequada (um usuário pode acessar dados de outro)?
- Secrets hardcoded no código?

**Performance:**
- Consultas N+1 ao banco?
- Loops desnecessários ou complexidade O(n²) onde O(n) seria suficiente?
- Objetos grandes sendo criados desnecessariamente?
- Cache seria útil aqui?

**Legibilidade:**
- Os nomes de variáveis e métodos comunicam claramente a intenção?
- Métodos muito longos (acima de 20-30 linhas)?
- Comentários explicando "o quê" em vez do "por quê"?
- Complexidade acidental (código mais complexo que o problema requer)?

**Cobertura de testes:**
- Os casos de sucesso estão testados?
- Os casos de erro estão testados?
- Os testes testam comportamento ou implementação?

---

### Como dar feedback construtivo

**Modelo PNSE (Pergunta, Não, Sugestão, Elogio):**

```
# Ruim: feedback agressivo
"Esse código está errado, nunca faça isso."
"Por que você não usou streams aqui?"

# Bom: feedback construtivo e específico
"Observei que o método `calcularTotal` não trata o caso de lista vazia.
O que acontece se `itens` for null? Poderia adicionar uma verificação?"

# Pergunta ao invés de afirmação (quando há dúvida)
"Essa abordagem funciona bem aqui, mas fiquei curioso:
você considerou usar `Optional` para tornar a ausência de valor mais explícita?"

# Elogio genuíno
"Gostei muito da forma como você separou as responsabilidades aqui —
o serviço ficou muito mais testável assim."
```

**Classificação por severidade:**

```
# [BLOQUEADOR] — precisa ser resolvido antes do merge
[BLOQUEADOR] Esse endpoint não valida se o usuário tem permissão para
acessar o recurso. Um usuário poderia acessar dados de outro.

# [SUGESTÃO] — melhoria desejada, mas não obrigatória agora
[SUGESTÃO] Esse loop poderia ser simplificado com .stream().filter().toList()

# [NITPICK] — questão menor de estilo
[NITPICK] Prefiro o nome `clienteId` a `idCliente` por consistência com
o restante da codebase, mas pode ficar assim se você preferir.

# [PERGUNTA] — genuinamente não entendeu
[PERGUNTA] Por que usou `synchronized` aqui? Essa operação pode ser
chamada concorrentemente?
```

### Como receber feedback

- **Não leve para o pessoal:** o reviewer está criticando o código, não você como pessoa
- **Peça clareza:** "Poderia dar um exemplo de como você faria isso?"
- **Agradeça:** especialmente por feedback difícil — levou tempo e cuidado
- **Discorde com argumentos:** "Entendo o ponto, mas optei por essa abordagem porque..."
- **Não argumente sobre nitpicks:** se é questão de estilo, ceda e siga em frente

---

## Trabalho em equipe: Git Flow em times

### Fluxo recomendado para times pequenos (GitHub Flow)

```bash
# 1. Sempre partir da main atualizada
git checkout main
git pull origin main

# 2. Criar branch descritiva
git checkout -b feat/adicionar-filtro-por-categoria
# ou: fix/corrigir-calculo-desconto
# ou: refactor/extrair-servico-pagamento
# ou: docs/atualizar-readme-deploy

# 3. Trabalhar com commits pequenos e frequentes
git add src/main/java/com/minhaapp/service/ProdutoService.java
git commit -m "feat: adicionar filtro por categoria no serviço de produtos"

git add src/test/java/com/minhaapp/service/ProdutoServiceTest.java
git commit -m "test: adicionar testes para filtro por categoria"

# 4. Manter branch atualizada com a main
git fetch origin
git rebase origin/main  # Prefira rebase a merge para histórico linear

# 5. Abrir PR com descrição clara
# 6. Code review
# 7. Merge (squash ou merge commit, conforme a política do time)

# 8. Deletar a branch após merge
git branch -d feat/adicionar-filtro-por-categoria
git push origin --delete feat/adicionar-filtro-por-categoria
```

### Convenção de commits (Conventional Commits)

```
tipo(escopo): descrição curta no imperativo

Corpo opcional explicando o porquê da mudança
(não o quê — o diff já mostra o quê)

Rodapé opcional:
Closes #42
BREAKING CHANGE: remove parâmetro `email` do construtor de Cliente
```

**Tipos:**
- `feat`: nova funcionalidade
- `fix`: correção de bug
- `refactor`: mudança que não adiciona feature nem corrige bug
- `test`: adiciona ou corrige testes
- `docs`: mudanças na documentação
- `chore`: mudanças de build, CI, dependências
- `perf`: melhoria de performance

### Comunicação de decisões técnicas em PRs

```markdown
## O que esse PR faz
Adiciona cache Redis no endpoint de listagem de produtos.

## Por que
A listagem de produtos estava sendo a query mais lenta (~800ms),
pois agrega dados de 3 tabelas. O catálogo muda raramente (1-2x por dia),
então caching é uma solução simples e eficaz.

## Como foi implementado
- Cache no Redis com TTL de 30 minutos
- Invalidação do cache quando um produto é criado/atualizado
- Testes de integração com Redis Testcontainers

## Como testar
1. Rodar `docker compose up -d redis`
2. `./mvnw test -Dtest=ProdutoServiceTest`
3. Verificar nos logs: `Cache HIT/MISS para produtos`

## Riscos e limitações
- Dados podem ficar desatualizados por até 30min após uma mudança
- Redis precisa estar disponível (circuit breaker não implementado nessa PR)
```

---

## Contribuindo com Open Source

### Como encontrar projetos

- **GitHub Explore:** github.com/explore
- **Good First Issues:** github.com/explore?q=good+first+issue (filtra issues marcadas para iniciantes)
- **Up For Grabs:** up-for-grabs.net (agregador de issues abertas para contribuição)
- **Projetos que você usa:** Spring Boot, Hibernate, MapStruct, etc. todos aceitam contribuições

### Fluxo de contribuição

```bash
# 1. Fork o repositório no GitHub (botão "Fork")

# 2. Clone o SEU fork (não o original)
git clone https://github.com/SEU-USUARIO/spring-boot.git
cd spring-boot

# 3. Configure o upstream (repositório original)
git remote add upstream https://github.com/spring-projects/spring-boot.git

# 4. Crie uma branch a partir da branch correta do upstream
git fetch upstream
git checkout -b fix/corrigir-nullpointer-na-configuracao upstream/main

# 5. Faça a mudança, adicione testes, rode a suíte completa
./mvnw test

# 6. Commit e push para o SEU fork
git push origin fix/corrigir-nullpointer-na-configuracao

# 7. Abra Pull Request no repositório ORIGINAL
# GitHub mostrará automaticamente a opção de abrir PR

# 8. Mantenha a branch atualizada durante o review
git fetch upstream
git rebase upstream/main
git push origin fix/corrigir-nullpointer-na-configuracao --force-with-lease
```

### Criando boas issues

```markdown
## Descrição do problema
Ao usar @SpringBootTest com configuração customizada, um NullPointerException
é lançado se a propriedade `spring.datasource.url` não estiver definida.

## Como reproduzir
1. Criar projeto Spring Boot 3.2.x com spring-boot-starter-data-jpa
2. Não definir spring.datasource.url
3. Rodar qualquer @SpringBootTest
4. Observar: `java.lang.NullPointerException at DataSourceAutoConfiguration.java:45`

## Comportamento esperado
Uma mensagem de erro clara: "spring.datasource.url é obrigatório quando JPA está no classpath"

## Comportamento atual
NullPointerException sem mensagem útil

## Ambiente
- Spring Boot: 3.2.1
- Java: 21
- OS: Linux (Ubuntu 22.04)
```

---

## Architecture Decision Records (ADRs)

ADRs documentam decisões arquiteturais importantes: o contexto, as opções consideradas, a decisão tomada e as consequências.

### Por que documentar decisões técnicas

- Novos membros do time entendem "por que está assim" sem perguntar
- Evita reabrir discussões já resolvidas
- Ajuda a identificar quando uma decisão antiga precisa ser revisitada
- Histórico auditável de evolução da arquitetura

### Estrutura de um ADR

```markdown
# ADR-001: Usar PostgreSQL como banco principal

## Status
Aceito

## Data
2026-01-15

## Contexto
Precisamos escolher um banco de dados para o sistema de pedidos.
O volume esperado é de 10.000 pedidos/dia com picos de 500 req/s.
O time tem experiência com bancos relacionais.

## Opções consideradas

### PostgreSQL
- Vantagens: ACID, excelente suporte a JSON, índices GIN para buscas textuais,
  extenso suporte a tipos, ótimo ecossistema Java (Hibernate, JPA, R2DBC)
- Desvantagens: operações de schema changes exigem migrações cuidadosas

### MongoDB
- Vantagens: esquema flexível, bom para documentos aninhados
- Desvantagens: sem transações ACID multi-documento antes do v4, time sem experiência

### MySQL
- Vantagens: muito conhecido, hosting barato
- Desvantagens: suporte a JSON inferior ao PostgreSQL, menos funcionalidades

## Decisão
Usaremos **PostgreSQL 16**.

## Consequências
- Positivas: transações ACID garantidas, suporte nativo a JSON para dados semi-estruturados,
  boa integração com Spring Data JPA
- Negativas: times precisarão aprender gestão de migrações com Flyway
- A ser monitorado: uso de extensão pgvector se precisarmos de busca semântica no futuro
```

### Organização no projeto

```
minha-app/
├── docs/
│   └── adr/
│       ├── ADR-001-banco-postgresql.md
│       ├── ADR-002-arquitetura-hexagonal.md
│       ├── ADR-003-autenticacao-jwt.md
│       └── ADR-004-cache-redis.md
└── src/
```

---

## Comunicação técnica para não-técnicos

### Princípios

**Use analogias familiares:**
```
# Técnico (evite):
"Precisamos implementar um circuit breaker no serviço de pagamento
para evitar cascading failures quando o gateway estiver com latência alta."

# Para não-técnicos:
"Quando a operadora de cartão começa a demorar muito para responder,
nossa loja para de tentar processar pagamentos por alguns minutos.
É como um disjuntor elétrico: ao invés de sobrecarregar o sistema,
ele 'desarma' e espera para tentar de novo quando a operadora se recuperar."
```

**Foque no impacto no negócio:**
```
# Técnico:
"Vamos adicionar índices compostos na tabela de pedidos e otimizar
as queries N+1 no Hibernate."

# Para o negócio:
"A tela de listagem de pedidos está demorando 4 segundos.
Identificamos o problema: vamos fazer uma otimização que deve
reduzir para menos de 500ms. Isso melhora a experiência do operador
que consulta pedidos dezenas de vezes por dia."
```

**Apresente opções com trade-offs:**
```
"Para o problema de performance, temos 2 caminhos:

Opção A (rápida): Adicionar cache — 1 dia de trabalho, reduz o problema
em 80%, mas dados podem ficar desatualizados por 5 minutos.

Opção B (estrutural): Refatorar as queries — 3 dias de trabalho,
resolve o problema definitivamente, sem dados desatualizados.

Minha recomendação: Opção A agora para resolver o urgente,
Opção B no próximo sprint."
```

---

## Como usar IA de forma saudável

A IA é uma ferramenta poderosa, mas usada sem critério pode prejudicar seu desenvolvimento como engenheiro.

### O ciclo saudável: Gerar → Entender → Adaptar → Documentar

**1. Gerar:**
Use a IA para acelerar tarefas repetitivas, explorar abordagens e desbloquear quando estiver travado.

```
# Prompt eficaz
"Preciso de uma implementação de Repository em Spring Data JPA para
a entidade Pedido, com métodos para buscar por clienteId, por status,
e por período de data. Use Java 21 e Spring Boot 3.x."
```

**2. Entender:**
*Nunca copie código sem entender o que ele faz.*

- Leia linha por linha
- Identifique cada decisão: por que essa anotação? por que esse tipo?
- Faça perguntas: "Explique por que usou `@Transactional(readOnly = true)` aqui"
- Execute e inspecione o comportamento

**3. Adaptar:**
O código gerado é um ponto de partida, não o produto final.

- Ajuste para o seu contexto (nomes do domínio, regras de negócio)
- Adicione tratamento de erros adequado
- Escreva os testes (a IA pode ajudar, mas você precisa validar)
- Remova o que não precisa

**4. Documentar:**
Se a IA ajudou a resolver um problema não-óbvio, documente a decisão.

```java
/**
 * Usa @Transactional(readOnly = true) aqui para ativar otimizações
 * do Hibernate (snapshot desnecessário não é criado) e permitir que
 * o connection pool direcione para réplicas de leitura em produção.
 * Ver: ADR-005-estrategia-leitura-escrita.md
 */
@Transactional(readOnly = true)
public List<PedidoResponse> listarPorCliente(String clienteId) {
    // ...
}
```

### Sinais de que você está usando IA de forma prejudicial

- Você não consegue explicar o código que commitou
- Você copia e cola sem rodar localmente
- Você não escreve testes porque "a IA disse que está certo"
- Você usa IA para responder a perguntas que deveriam estar na sua cabeça (como funciona um HashMap?)
- Você não sente que está aprendendo — só "entregando"

### O que a IA não substitui

- O julgamento sobre o que construir (e o que **não** construir)
- A capacidade de debugar quando algo quebra em produção às 2h da manhã
- O entendimento do negócio para fazer perguntas certas
- A responsabilidade pelo código em produção
- O aprendizado que vem de errar, debugar e entender o porquê
