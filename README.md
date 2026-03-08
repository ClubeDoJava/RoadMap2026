
**Roadmap Otimizado para Aprender Java em 2026**  
(com IA como pair programmer inteligente)

**Disclaimer:** A IA acelera muito, mas nunca use como muleta. 

Sempre peça explicação, trade-offs, como debugar e teste tudo. 

Mantenha um arquivo “decisoes.md” em cada projeto explicando por que escolheu X ou Y. 

Isso é o que transforma você em dev de verdade.

### 0. Setup obrigatório (1 dia)
- Instale Cursor (ou VS Code + Continue.dev).  
- Conecte Claude 4 Sonnet ou Grok.  
- Prompt fixo para colar em todo projeto:  
  “Você é um Java senior em Java 25 LTS + Spring Boot 4. Sempre explique o motivo da escolha, mostre trade-offs, sugira testes e priorize código limpo e eficiente. Nunca use features preview em produção.”

Regra de ouro: IA gera primeiro ----> você lê, entende, testa, melhora e escreve no decisoes.md.

Pronto. Agora você tem 4 semanas de vantagem.

---

### Fase 1 – Semana 1-2: Java básico + Git + CLI (entregável real)
Objetivo: escrever código confortável sem framework.

**Entregáveis:**
- Gerenciador de tarefas CLI (CRUD em memória + salvar em arquivo JSON).
- Testes unitários básicos com JUnit 5.
- Repositório no GitHub com README claro (“como rodar”, decisões e melhorias futuras).

**Checklist mínimo:**
- Variáveis, controle de fluxo, métodos, arrays/collections
- String, Optional, exceções
- List/Set/Map
- Leitura/escrita de arquivo com java.nio.file
- Uso de Virtual Threads só para simular tarefas assíncronas (ex: salvar arquivo)

**Checkpoint:** Commit tudo, rode os testes e publique. 

Peça para a IA revisar e você explica as mudanças no README.

---

### Fase 2 – Semana 3-4: OOP de verdade + design básico
Objetivo: parar de fazer “script” e começar a modelar.

**Entregáveis:**
- Sistema de biblioteca (Livro, Autor, Empréstimo, com regras de negócio).
- Pelo menos 1 padrão (Factory ou Builder) aplicado de forma consciente.
- Testes cobrindo as regras principais.

**Checklist:**
- Classes, interfaces, encapsulamento
- Records onde faz sentido
- Composição > herança
- SOLID (foco em SRP e DIP)
- Scoped Values só para compartilhar dados imutáveis entre threads (quando necessário)

**Checkpoint:** Publique no GitHub. Escreva no decididoes.md por que usou record ou classe normal. 

Refatore um pedaço com ajuda da IA e explique o antes/depois.

---

### Fase 3 – Semana 5-6: API REST com Spring Boot + testes + docs
Objetivo: aprender o que o mercado pede no dia a dia.

**Entregáveis:**
- API REST simples (ex: cadastro de usuários ou blog básico).
- OpenAPI/Swagger completo.
- Testes unitários + integração com @SpringBootTest.

**Checklist:**
- Spring Boot 4 (Web, Validation, Actuator)
- Camadas controller/service/repository
- DTOs de entrada e saída
- Tratamento de erro padronizado
- Logging decente
- Spring AI (integre um endpoint simples de chat com memória – é o diferencial 2026)

**Checkpoint:** Rode localmente, documente no Swagger, publique no GitHub. 

Adicione pipeline básico de GitHub Actions que só faz build + testes.

---

### Fase 4 – Semana 7-8: Banco + migrations + segurança (backend completo)
**Entregáveis:**
- PostgreSQL + Flyway
- Spring Data JPA com paginação e filtros
- Autenticação JWT + roles
- Testcontainers (banco real nos testes)

**Checklist:**
- Entidades e relacionamentos corretos
- Migrations sempre versionadas
- OWASP Top 10 básico (validação, não logar senha, auth correta)
- Virtual Threads aplicados onde tem I/O (HTTP + DB)

**Checkpoint:** Tudo rodando com Docker Compose. 

Testes passando com banco real.

Publique e escreva no README o que mudou da fase anterior.

---

### Fase 5 – Semana 9-10: Deploy + CI/CD + observabilidade (“shippar” como profissional)
**Entregáveis:**
- Dockerfile otimizado + docker-compose (app + postgres)
- GitHub Actions completo (build, tests, lint, imagem Docker)
- Métricas com Spring Boot Actuator + OpenTelemetry básico
- Aplicação rodando em nuvem (Render ou Railway – grátis e simples)

**Checkpoint final:** Link da API em produção no README + dashboard de métricas funcionando. 

Grave um vídeo de 2 minutos mostrando tudo (opcional, mas impressiona muito).

---

### Depois das 10 semanas: Aprendizado contínuo (faça no seu ritmo)
- Padrões de projeto (aplique nos projetos existentes)
- Arquitetura (DDD + Hexagonal em um microsserviço pequeno)
- Lombok + MapStruct
- WebFlux (só se quiser reativo)
- Profiling com VisualVM

**Prática semanal obrigatória:**
- 2-3 problemas no LeetCode/HackerRank (manter afiado)
- Contribuição pequena em open-source ou issue no seu próprio repo

---

### Recursos que valem ouro
- Documentação: Java 25 LTS e Spring Boot 4 (oficiais)
- Baeldung e Dev.java
- Livros: Effective Java, Clean Code, Java Concurrency in Practice
- Comunidades: GUJ, Discord Java Brasil, Reddit r/java
- Cursos: Alura/Udemy que tenham módulo “Java + IA”

### Dicas que realmente aceleram

- Todo projeto nasce com README + testes + CI desde o dia 1.
- Sempre termine a semana com o projeto publicado e o arquivo decisoes.md atualizado.
- Use a IA todo dia, mas nunca aceite sem entender.
- O caderno de decisões é o que separa quem copia de quem aprende.
- Em 3-4 meses você entrega APIs profissionais de verdade.
