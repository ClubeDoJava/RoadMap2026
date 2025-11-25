# Roadmap Otimizado para Aprender Java em 2026

Este roadmap foi desenhado para ajudar você no aprendizado da linguagem Java e seu ecossistema
de forma estruturada e eficiente.

---

## 1. Fundamentos de Programação Essenciais

### Conceitos básicos de programação:
- [ ] **Lógica básica: algoritmos simples, fluxogramas e resolução de problemas (use plataformas como Codecademy ou freeCodeCamp para intro gratuita).**
- [ ] Variáveis, tipos de dados (primitivos e referência) e operadores.
- [ ] Estruturas de controle (condicionais `if/else`, `switch`; laços `for`, `while`, `do-while`).
- [ ] Métodos (funções): declaração, parâmetros, retorno.
- [ ] Manipulação de *strings* e *arrays* básicos.
- [ ] Entrada e saída padrão (console).

**Projeto prático:** Desenvolva um gerenciador de tarefas simples para linha de comando (CLI) que permita adicionar, listar e remover tarefas. Integre *virtual threads* para tarefas assíncronas básicas.
**(Adicione: Teste com ferramentas como JShell para experimentação rápida de código sem IDE full.)**

### Orientação a Objetos (OOP) - O Paradigma Central do Java:
- [ ] Classes e objetos: a base da OOP.
- [ ] Interfaces: definindo contratos.
- [ ] Classes abstratas.
- [ ] **Pilares da OOP:**
    - [ ] Encapsulamento: protegendo os dados.
    - [ ] Herança: reutilizando e estendendo código.
    - [ ] Polimorfismo: múltiplas formas de um objeto.
    - [ ] Abstração: modelando o essencial.

**Projeto prático:** Crie um sistema simples de gerenciamento de uma biblioteca, com classes para `Livro`, `Autor`, `Biblioteca`, permitindo adicionar livros, buscar por título ou autor. 
Use `Scoped Values` para compartilhar dados imutáveis entre *threads*.

---

## 2. Configurando seu Ambiente de Desenvolvimento Java

### Escolha uma IDE/Editor de Código:
- [ ] **Visual Studio Code com o "Extension Pack for Java" (leve e gratuito, ótimo para starters).**
- [ ] IntelliJ IDEA Community/Ultimate (altamente recomendado pela produtividade, agora com suporte nativo a Java 25).
- [ ] Eclipse IDE for Java Developers.

### Gerenciamento de Dependências e Build do Projeto:
- [ ] **Apache Maven:** aprenda a estrutura de projetos, `pom.xml`, ciclo de vida e gerenciamento de dependências (foco inicial).
- [ ] **Gradle:** conheça como alternativa, especialmente popular em projetos Android e outros ecossistemas.
- [ ] **GraalVM Native Image:** Para compilar *apps* Java em executáveis nativos, reduzindo *startup time* e memória.
**(Adicione: Spring Native para integração com Spring Boot.)**

### Controle de Versão com Git:
- [ ] Comandos essenciais do Git (`commit`, `push`, `pull`, `branch`, `merge`, `rebase`)
Utilize o site **Learn Git Branching** para desenvolver uma intuição melhor sobre a ferramenta.
- [ ] Plataformas de hospedagem: GitHub (mais popular), GitLab, Bitbucket.
- [ ] Fluxos de trabalho: Git Flow ou GitHub Flow (simplificado).

**(Adicione: GitHub Codespaces para rodar projetos sem instalar nada localmente.)**

**Prática contínua:** Crie um repositório no GitHub para cada projeto. Faça *commits* frequentes.

Integre GitHub Copilot para sugestões de código AI-assistidas.

---

## 3. Dominando a Linguagem Java

### Sintaxe Moderna e Recursos das Versões LTS:
- [ ] Foco em recursos do **Java 21 (LTS)** e **Java 25 (LTS)**.
- [ ] `var` para inferência de tipo de variáveis locais.
- [ ] *Switch Expressions*.
- [ ] *Text Blocks* para *strings* multi-linha.
- [ ] *Record Classes* para classes de dados imutáveis.
- [ ] *Pattern Matching* para `instanceof`.
- [ ] *Sealed Classes* and *Interfaces*.
- [ ] *Scoped Values* (JEP 506), *Structured Concurrency* aprimorada, *Primitive Patterns* (preview).

**(Adicione: Unnamed Variables e String Templates do Java 25+.)**

**Prática:** Refatore seus projetos anteriores utilizando esses recursos modernos, incluindo *Scoped Values* para compartilhamento de dados.

### APIs Fundamentais da JDK (Java Development Kit):
- [ ] **Java Collections Framework:** `List` (`ArrayList`, `LinkedList`), `Set` (`HashSet`, `TreeSet`), `Map` (`HashMap`, `TreeMap`).
- [ ] **Functional Interfaces:** base conceitual para lambdas e method references (`::`).
- [ ] **APIs de funções úteis:** `Predicate<T>` (testar condição booleana), `Function<T, R>` (transformação de valores),
`Supplier<T>` (fornecimento de valores sem exigir entrada).
- [ ] **Streams API** e **Expressões Lambda**: para processamento de coleções de forma funcional e concisa.

 **(Adicione: Parallel Streams para performance.)**
 
- [ ] Tratamento de Exceções: `try-catch-finally`, `throw`, criação de exceções customizadas.
- [ ] `Optional` para melhor manuseio de valores nulos.
- [ ] Manipulação de Datas e Horas com `java.time` (`LocalDate`, `LocalDateTime`, etc.).

**Projeto prático:** Crie um programa que gerencia um *ranking* de filmes, permitindo adicionar filmes, classificá-los, e listar os top N filmes, utilizando coleções e *streams*.

### Introdução à Concorrência (*Multithreading*):
- [ ] Conceitos de `Thread` e `Runnable`.
- [ ] `ExecutorService` e *ThreadPools* para gerenciamento eficiente de *threads*.
- [ ] Entendimento básico de problemas de concorrência (*race conditions*, *deadlocks*).
- [ ] **Virtual Threads (Java 21+)** para escalabilidade massiva sem *overhead*.

### Boas Práticas de Codificação:
- [ ] Princípios de *Clean Code*.
- [ ] Princípios **SOLID** para *design* orientado a objetos.
- [ ] Convenções de nomenclatura e formatação de código Java.
- [ ] **Green Coding** – Otimização para eficiência energética (ex: reduzir *loops* ineficientes).
- [ ] **Object Calisthenics** – Exercícios de disciplina para evoluir a qualidade e expressividade do código.

**(Adicione: Use SonarLint para análise estática de código na IDE.)**

**Prática:** Resolva problemas em plataformas como HackerRank, LeetCode (nível fácil/médio) ou Exercism, focando na clareza, eficiência e concorrência.

---

## 4. Fundamentos Intermediários e Ferramentas

### Manipulação de Arquivos e I/O (Entrada/Saída):
- [ ] Classes `File`, `FileInputStream`, `FileOutputStream`.
- [ ] `BufferedReader`, `BufferedWriter` para leitura/escrita eficiente de texto.
- [ ] **Java NIO (New I/O - `java.nio`):** `Path`, `Files`, *Channels*, *Buffers* (para operações mais avançadas).

**Projeto prático:** Desenvolva um programa que leia dados de um arquivo CSV (ex: lista de produtos) e escreva um resumo ou dados processados em um arquivo TXT ou outro CSV.

### Serialização e Desserialização de Dados:
- [ ] **JSON:** utilizando bibliotecas populares como **Jackson** ou **Gson**.
- [ ] XML: básico com JAXB (opcional, mais comum em sistemas legados). **(Adicione: Avro ou Protobuf para performance em big data.)**

**Prática:** Crie uma funcionalidade que converta objetos Java (ex: os livros da sua biblioteca) para formato JSON e vice-versa.

### Testes Unitários Automatizados:
- [ ] **JUnit 5:** para escrever e executar testes unitários.
- [ ] **Mockito:** para criar objetos *mock* (dublês) e isolar unidades de teste.
- [ ] **AssertJ** para asserções fluentes e mais legíveis (alternativa ao Hamcrest).
- [ ] Importância da cobertura de testes (*Code Coverage*).
- [ ] Pact para *contract testing* em microsserviços. **(Adicione: JaCoCo para medir cobertura.)**

**Prática:** Escreva testes unitários para todos os projetos que você desenvolveu até agora, buscando boa cobertura das funcionalidades.

### Acesso Básico a Banco de Dados Relacional:
- [ ] **JDBC (Java Database Connectivity):** API fundamental para conectar e interagir com bancos de dados.
- [ ] Conexão com bancos como **PostgreSQL** (recomendado) ou MySQL. **(Adicione: H2 para testes in-memory.)**
- [ ] Execução de *queries* SQL (`SELECT`, `INSERT`, `UPDATE`, `DELETE`).
- [ ] `PreparedStatement` para evitar *SQL Injection*.

**Projeto prático:** Crie um **CRUD** (*Create, Read, Update, Delete*) simples para cadastro de usuários, armazenando os dados em um banco de dados relacional.

---

## 5. Frameworks Essenciais e Bibliotecas do Ecossistema

### Desenvolvimento Web com Spring Framework (Foco Principal):
- [ ] **Spring Core:** Inversão de Controle (IoC), Injeção de Dependência (DI).
- [ ] **Spring MVC:** para construção de aplicações web tradicionais.
- [ ] **Spring Boot:** para desenvolvimento rápido de aplicações *stand-alone* e microsserviços (**foco em 3.3+ ou 4.0**, com suporte a *virtual threads* e *SBOM*).
- [ ] Alternativas modernas: **Quarkus** (prioridade para performance e *cloud-native*), **Micronaut**. **(Adicione: Helidon para lightweight.)**
- [ ] Jakarta EE: conhecer caso precise trabalhar com sistemas legados (opcional).

**Projeto prático:** Desenvolva uma **API RESTful** para um *blog* (posts, comentários, usuários) ou um sistema de e-commerce simplificado (produtos, pedidos) usando Spring Boot.
Compile para *native image* com GraalVM.

### Persistência de Dados com JPA e Hibernate:
- [ ] **JPA (Jakarta Persistence API):** especificação para ORM (*Object-Relational Mapping*).
- [ ] **Hibernate:** implementação JPA mais popular.
- [ ] **Spring Data JPA:** simplifica o uso de JPA/Hibernate com Spring.
- [ ] Gerenciamento de transações (`@Transactional`).
- [ ] Ferramentas de Migração de Banco de Dados: **Flyway** ou **Liquibase** (atualizado para v10+).

### Segurança de Aplicações:
- [ ] **Spring Security:** para adicionar autenticação e autorização às suas aplicações.
- [ ] Conceitos de **OAuth2** e **JWT** (*JSON Web Tokens*) para APIs seguras.
- [ ] Configuração de HTTPS.
- [ ] Princípios de segurança web (**OWASP Top 10**, atualizado para 2025).

**(Adicione: Keycloak para gerenciamento de identidade.)**

**Prática:** Adicione autenticação e autorização baseada em papéis (*roles*) à API REST que você construiu.

### Desenvolvimento e Documentação de APIs:
- [ ] Construção de APIs RESTful robustas com Spring Boot.
- [ ] **OpenAPI Specification (Swagger):** para documentar e testar suas APIs.

**Prática:** Documente sua API REST utilizando Swagger/OpenAPI.

### Testes Avançados e de Integração:
- [ ] JUnit 5 para testes de integração.
- [ ] Mockito para *mocks* em testes de integração.
- [ ] **Testcontainers:** para testes de integração com dependências reais (ex: banco de dados) em *containers* Docker.
- [ ] Spring Boot Test Utilities (`@SpringBootTest`).

**Prática:** Escreva testes de integração para sua API completa, incluindo a camada de persistência e segurança, utilizando Testcontainers para o banco de dados.

---

## 6. Deploy, DevOps e Boas Práticas de Entrega

### Containerização de Aplicações:
- [ ] **Docker:** criação de imagens Docker para suas aplicações Java.
- [ ] Comandos básicos do Docker (`build`, `run`, `push`, `pull`).
- [ ] **Docker Compose** para orquestrar múltiplos *containers* localmente.
- [ ] **Kubernetes (K8s):** conceitos básicos de orquestração de *containers* (Pods, Services, Deployments) - agora com ArgoCD para CD GitOps.
- [ ] Serverless com Java (ex: AWS Lambda ou GCP Cloud Functions). **(Adicione: Knative para serverless em K8s.)**

**Prática:** Crie um `Dockerfile` para sua API Spring Boot e execute-a em um *container* Docker ou *serverless*.

### Integração Contínua e Entrega Contínua (CI/CD):
- [ ] **GitHub Actions:** para automação de *build*, teste e *deploy* (agora com *workflows* para GraalVM).
- [ ] Jenkins: outra ferramenta popular de CI/CD (opcional). **(Adicione: GitLab CI como alternativa open-source.)**

**Prática:** Crie um *pipeline* de CI/CD simples no GitHub Actions para sua aplicação, que rode os testes, construa a imagem Docker e *deploy* para nuvem a cada *push*.

### Estratégias de Build e Deploy em Nuvem:
- [ ] Provedores de Cloud: AWS, GCP, Azure.
- [ ] Plataformas como Serviço (PaaS): Heroku, Render (para *deploy* simplificado).

**Projeto prático:** Faça o *deploy* da sua API REST (containerizada ou *native*) em uma plataforma de nuvem (ex: AWS Lambda, GCP Cloud Run).

### Monitoramento e Observabilidade (Básico):
- [ ] Métricas, Logs e *Traces*.
- [ ] **Spring Boot Actuator** para expor métricas da aplicação (atualizado com suporte a Prometheus 1.x).
- [ ] Introdução a ferramentas como **Prometheus** (métricas) e **Grafana** (dashboards).
- [ ] Elastic Stack (ELK - Elasticsearch, Logstash, Kibana) para gerenciamento de logs. **(Adicione: OpenTelemetry para traces padronizados.)**

**Prática:** Configure o Spring Boot Actuator na sua API e integre com Prometheus e Grafana para visualizar métricas básicas, incluindo *traces* de AI.

---

## 7. Aprendizado Contínuo e Evolução

### Padrões de Projeto (*Design Patterns*):
- [ ] Fundamentais: Singleton, Factory Method, Abstract Factory, Builder, Prototype.
- [ ] Estruturais: Adapter, Decorator, Facade, Proxy.
- [ ] Comportamentais: Observer, Strategy, Template Method, Iterator, Command.

**Prática:** Identifique e aplique padrões de projeto relevantes em seus projetos existentes ou em novos desafios.

### Arquitetura de Software:
- [ ] Comparativo: Arquitetura Monolítica vs. Microsserviços.
- [ ] Arquitetura Orientada a Eventos (Event-Driven Architecture - EDA).
- [ ] **DDD (Domain-Driven Design)** - conceitos fundamentais. **(Adicione: Hexagonal Architecture para decoupling.)**

**Projeto prático (avançado):** Considere refatorar uma parte de um projeto monolítico para um microsserviço, ou desenhe uma nova funcionalidade usando princípios de EDA.

### Ferramentas de Produtividade:
- [ ] **Lombok:** para reduzir código *boilerplate* (getters, setters, construtores, etc.).
- [ ] **MapStruct:** para mapeamento eficiente entre objetos (DTOs, Entidades).

### Desenvolvimento de Soft Skills:
- [ ] Comunicação eficaz em revisões de código (*code reviews*).
- [ ] Habilidade de ler, entender e reportar *bugs* de forma clara.
- [ ] Trabalho em equipe e colaboração.

**Prática:** Contribua para projetos *open-source* (mesmo que com pequenas melhorias ou documentação) ou participe ativamente em discussões técnicas em fóruns e comunidades.

---

## 8. Tópicos Avançados e Especializações (Opcional, conforme interesse)

### Programação Reativa:
- [ ] Project Reactor (`Flux`, `Mono`).
- [ ] Spring WebFlux para APIs reativas.

**Projeto prático:** Desenvolva uma API reativa simples usando Spring WebFlux.

### Concorrência Avançada:
- [ ] `CompletableFuture` para programação assíncrona.
- [ ] *Parallel Streams* e seus casos de uso.
- [ ] **Virtual Threads (Project Loom - parte do Java 21+):** entenda os benefícios para concorrência. Integração com *Scoped Values*.

### Otimização de Desempenho e *Tuning* da JVM:
- [ ] Ferramentas de *profiling*: JProfiler (comercial), VisualVM (gratuito, parte do JDK).
- [ ] Análise de *Garbage Collection* (GC) e ajustes da JVM.

**Prática:** Use o VisualVM para analisar o desempenho de uma de suas aplicações e identificar possíveis gargalos.

### Exploração de Nichos e Bibliotecas Específicas:
- [ ] AI/ML com Java: **Deep Java Library (DJL)** para modelos de *machine learning*.
- [ ] Processamento de Big Data com Apache Spark e Java.
- [ ] Desenvolvimento de aplicações Desktop com JavaFX. **(Adicione: Apache Kafka para streaming em EDA.)**

---

## 📚 Recursos Recomendados para sua Jornada

### Documentação Oficial:
- [ ] Documentação Oficial do Java (Oracle) – Foco em Java 25.
- [ ] Guias e Documentação do Spring – Incluindo Spring Boot 4.0.
- [ ] Documentação do **Baeldung** (excelentes tutoriais).
- [ ] **Dev.java** (tutoriais e novidades oficiais da Oracle).

### Livros Essenciais:
- [ ] *"Effective Java"* (Joshua Bloch) - Leitura obrigatória após ter uma base.
- [ ] *"Java: Como Programar"* (Deitel & Deitel) - Abrangente para iniciantes.
- [ ] *"Head First Java"* (Kathy Sierra, Bert Bates) - Didático e divertido para começar.
- [ ] *"Clean Code"* (Robert C. Martin) - Sobre qualidade de código.
- [ ] *"Spring in Action"* (Craig Walls) - Para Spring Framework (edição atualizada para 2025).
- [ ] *"Designing Data-Intensive Applications"* (Martin Kleppmann) - Para arquitetura.
- [ ] Guias de Certificação (OCP): OCP Java SE 25 Developer Study Guide (atualizado para Java 25).
- [ ] *"Java Concurrency in Practice"* (Brian Goetz) – Para *virtual threads* e *Scoped Values*.

### Comunidades e Fóruns (Português e Inglês):
- [ ] **GUJ** (Grupo de Usuários Java - Brasil).
- [ ] Discord da Comunidade Java Brasil (ex: SouJava, outras) (Verifique links atualizados de comunidades ativas). **(Adicione: Java Discord servers globais.)**
- [ ] Stack Overflow (`pt.stackoverflow.com` e `stackoverflow.com`).
- [ ] Reddit: `r/java`, `r/SpringBoot`.
- [ ] Oracle Developer Community para discussões sobre Java 25.

### Plataformas de Cursos e Prática:
- [ ] Udemy, Alura, Coursera, Pluralsight (cursos pagos com alta qualidade, agora com módulos para Java 25 e GraalVM).
- [ ] Exercism, HackerRank, LeetCode, Codewars (para praticar lógica e algoritmos). Adicione seções de AI em LeetCode.

---

## ✅ Dicas Finais para o Sucesso

1.  **Mão na Massa Sempre:** A prática leva à maestria. Cada conceito aprendido deve ser aplicado em **projetos práticos**, por menores que sejam.
2.  **Comece Simples, Evolua Gradualmente:** Para desenvolvimento web e APIs, **Spring Boot** é um excelente ponto de partida devido à sua popularidade e facilidade de configuração. Não tente aprender tudo de uma vez. Priorize **GraalVM** para *deploys* modernos.
3.  **Mantenha-se Atualizado:** O ecossistema Java é dinâmico. Acompanhe as novidades das versões do Java (ex: Java 25+ previews), *frameworks* e tendências em *blogs* (InfoQ, Baeldung, DZone) e conferências como Devoxx ou Oracle Code One.
4.  **Networking é Fundamental:** Participe de comunidades, *meetups* (online ou presenciais) e eventos. Trocar experiências com outros desenvolvedores acelera o aprendizado.
5.  **Construa seu Portfólio:** Mantenha um perfil no **GitHub** com seus projetos bem documentados, incluindo *demos* de AI e *native apps*. Isso será seu cartão de visitas para o mercado de trabalho.

**Boa sorte na sua jornada de aprendizado em Java!**
