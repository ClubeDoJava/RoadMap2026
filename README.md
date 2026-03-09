# Roadmap Otimizado para Aprender Java em 2026

Este roadmap foi desenhado para ajudar você no aprendizado da linguagem Java e seu ecossistema de forma estruturada e eficiente.

---

> **Disclaimer:** A IA acelera muito, mas nunca use como muleta.
> Sempre peça explicação, trade-offs, como debugar e teste tudo.
> Mantenha um arquivo `decisoes.md` em cada projeto explicando suas escolhas.
> Virtual Threads só onde tem I/O bloqueante. Features preview são só para estudo.

---

## Índice

- [0. Monte sua IA de programação](#0-primeiro-passo-obrigatório-monte-sua-ia-de-programação)
- [1. Fundamentos de Programação](#1-fundamentos-de-programação-essenciais)
- [2. Configurando o Ambiente](#2-configurando-seu-ambiente-de-desenvolvimento-java)
- [3. Dominando a Linguagem Java](#3-dominando-a-linguagem-java)
- [4. Fundamentos Intermediários](#4-fundamentos-intermediários-e-ferramentas)
- [5. Frameworks Essenciais](#5-frameworks-essenciais-e-bibliotecas-do-ecossistema)
- [6. Deploy e DevOps](#6-deploy-devops-e-boas-práticas-de-entrega)
- [7. Aprendizado Contínuo](#7-aprendizado-contínuo-e-evolução)
- [8. Tópicos Avançados](#8-tópicos-avançados-e-especializações-opcional)
- [Recursos Recomendados](#recursos-recomendados)

---

## 0. Primeiro passo obrigatório: Monte sua IA de programação

Antes de escrever qualquer linha de código, configure isso:

Instale o Cursor (melhor ferramenta em 2026 para Java) ou VS Code com Continue.dev.
Conecte com Claude 4.6+ Sonnet ou Opus.

Crie este prompt fixo e use em todo projeto:

> "Você é um desenvolvedor Java sênior especializado em Java 25 LTS + Spring Boot 4.
> Sempre explique o motivo da escolha, mostre trade-offs, sugira testes e priorize código limpo e eficiente. Nunca use features preview em produção."

Regra de ouro: IA gera primeiro → você lê, entende, testa, melhora e escreve no `decisoes.md`.
Com isso pronto, você já ganha umas 4 semanas de vantagem.

---

## 1. Fundamentos de Programação Essenciais

📁 [`01 - Aprenda os Fundamentos/`](./01%20-%20Aprenda%20os%20Fundamentos/)

### Sintaxe Básica

| # | Tópico | Link |
|---|--------|------|
| 1.1 | Criando Variáveis e nomeando elas | [📄 readme](./01%20-%20Aprenda%20os%20Fundamentos/SintaxeB%C3%A1sica/1.1%20-%20Criando%20Vari%C3%A1veis%20e%20nomeando%20elas/readme.md) |
| 1.2 | Tipos de Dados | [📄 readme](./01%20-%20Aprenda%20os%20Fundamentos/SintaxeB%C3%A1sica/1.2%20-%20Tipos%20de%20Dados/readme.md) |
| 1.3 | Operadores | [📄 readme](./01%20-%20Aprenda%20os%20Fundamentos/SintaxeB%C3%A1sica/1.3%20-%20Operadores/readme.md) |
| 1.4 | Estruturas de Controle | [📄 readme](./01%20-%20Aprenda%20os%20Fundamentos/SintaxeB%C3%A1sica/1.4%20-%20Estruturas%20de%20Controle/readme.md) |
| 1.5 | Métodos | [📄 readme](./01%20-%20Aprenda%20os%20Fundamentos/SintaxeB%C3%A1sica/1.5%20-%20M%C3%A9todos/readme.md) |
| 1.6 | Strings e Arrays | [📄 readme](./01%20-%20Aprenda%20os%20Fundamentos/SintaxeB%C3%A1sica/1.6%20-%20Strings%20e%20Arrays/readme.md) |
| 1.7 | Entrada e Saída Padrão | [📄 readme](./01%20-%20Aprenda%20os%20Fundamentos/SintaxeB%C3%A1sica/1.7%20-%20Entrada%20e%20Sa%C3%ADda%20Padr%C3%A3o/readme.md) |

### Orientação a Objetos (OOP)

📄 [`POO/readme.md`](./01%20-%20Aprenda%20os%20Fundamentos/POO/readme.md) — Objetos, Classes, Herança, Interfaces, Classes Abstratas, Encapsulamento, Polimorfismo, Abstração.

**Checklist:**
- [ ] Variáveis, tipos de dados e operadores
- [ ] Estruturas de controle
- [ ] Métodos: declaração, parâmetros, retorno
- [ ] Strings e arrays básicos
- [ ] Entrada e saída padrão (console)
- [ ] Classes e objetos
- [ ] Interfaces e classes abstratas
- [ ] Encapsulamento, Herança, Polimorfismo, Abstração

**Projeto prático:** Gerenciador de tarefas em linha de comando (adicionar, listar, remover). Depois: sistema de biblioteca com Livro, Autor e Biblioteca usando Records.

---

## 2. Configurando seu Ambiente de Desenvolvimento Java

📁 [`02 - Configurando o Ambiente/`](./02%20-%20Configurando%20o%20Ambiente/)

| # | Tópico | Link |
|---|--------|------|
| 2.1 | IDEs e Editores (Cursor, IntelliJ, VS Code) | [📄 readme](./02%20-%20Configurando%20o%20Ambiente/2.1%20-%20IDEs%20e%20Editores/readme.md) |
| 2.2 | Maven e Gradle + GraalVM Native Image | [📄 readme](./02%20-%20Configurando%20o%20Ambiente/2.2%20-%20Maven%20e%20Gradle/readme.md) |
| 2.3 | Git e Controle de Versão | [📄 readme](./02%20-%20Configurando%20o%20Ambiente/2.3%20-%20Git%20e%20Controle%20de%20Vers%C3%A3o/readme.md) |

**Checklist:**
- [ ] Cursor como IDE principal
- [ ] Maven: pom.xml, ciclo de vida, dependências
- [ ] Gradle como alternativa
- [ ] GraalVM Native Image
- [ ] Git: comandos essenciais, GitHub Flow, Conventional Commits

**Prática contínua:** Crie um repositório no GitHub para cada projeto. Use GitHub Codespaces quando não quiser instalar nada.

---

## 3. Dominando a Linguagem Java

📁 [`03 - Dominando o Java/`](./03%20-%20Dominando%20o%20Java/)

| # | Tópico | Link |
|---|--------|------|
| 3.1 | Recursos Modernos do Java 21 e 25 | [📄 readme](./03%20-%20Dominando%20o%20Java/3.1%20-%20Recursos%20Modernos%20do%20Java%2021%20e%2025/readme.md) |
| 3.2 | Collections Framework e Streams | [📄 readme](./03%20-%20Dominando%20o%20Java/3.2%20-%20Collections%20Framework%20e%20Streams/readme.md) |
| 3.3 | Tratamento de Exceções | [📄 readme](./03%20-%20Dominando%20o%20Java/3.3%20-%20Tratamento%20de%20Excecoes/readme.md) |
| 3.4 | Concorrência Básica | [📄 readme](./03%20-%20Dominando%20o%20Java/3.4%20-%20Concorrencia%20Basica/readme.md) |
| 3.5 | Boas Práticas (Clean Code, SOLID) | [📄 readme](./03%20-%20Dominando%20o%20Java/3.5%20-%20Boas%20Praticas%20de%20Codificacao/readme.md) |

**Checklist:**
- [ ] var, Switch Expressions, Text Blocks, Records, Pattern Matching, Sealed Classes
- [ ] Scoped Values, Unnamed Variables, Virtual Threads
- [ ] Collections: List, Set, Map, Queue
- [ ] Functional Interfaces, Streams, Optional
- [ ] Tratamento de exceções e exceções customizadas
- [ ] Thread, ExecutorService, Virtual Threads
- [ ] Clean Code, SOLID, Object Calisthenics

**Projeto prático:** Ranking de filmes com coleções e streams. Refatore projetos anteriores usando as features modernas.

---

## 4. Fundamentos Intermediários e Ferramentas

📁 [`04 - Intermediário/`](./04%20-%20Intermedi%C3%A1rio/)

| # | Tópico | Link |
|---|--------|------|
| 4.1 | Manipulação de Arquivos e IO | [📄 readme](./04%20-%20Intermedi%C3%A1rio/4.1%20-%20Manipula%C3%A7%C3%A3o%20de%20Arquivos%20e%20IO/readme.md) |
| 4.2 | JSON com Jackson | [📄 readme](./04%20-%20Intermedi%C3%A1rio/4.2%20-%20JSON%20com%20Jackson/readme.md) |
| 4.3 | Testes com JUnit 5, Mockito e AssertJ | [📄 readme](./04%20-%20Intermedi%C3%A1rio/4.3%20-%20Testes%20com%20JUnit%205%2C%20Mockito%20e%20AssertJ/readme.md) |
| 4.4 | Banco de Dados com JDBC e PostgreSQL | [📄 readme](./04%20-%20Intermedi%C3%A1rio/4.4%20-%20Banco%20de%20Dados%20com%20JDBC%20e%20PostgreSQL/readme.md) |

**Checklist:**
- [ ] File, BufferedReader/Writer, NIO (Path, Files, WatchService)
- [ ] Jackson: ObjectMapper, anotações, TypeReference
- [ ] JUnit 5, Mockito, AssertJ, JaCoCo
- [ ] JDBC, PreparedStatement, HikariCP, PostgreSQL via Docker

**Projeto prático:** Leia um CSV de produtos, processe com Streams e salve resultado. CRUD de usuários com JDBC.

---

## 5. Frameworks Essenciais e Bibliotecas do Ecossistema

📁 [`05 - Frameworks/`](./05%20-%20Frameworks/)

| # | Tópico | Link |
|---|--------|------|
| 5.1 | Spring Boot (Core, MVC, REST, Validação) | [📄 readme](./05%20-%20Frameworks/5.1%20-%20Spring%20Boot/readme.md) |
| 5.2 | JPA e Hibernate + Spring Data + Flyway | [📄 readme](./05%20-%20Frameworks/5.2%20-%20JPA%20e%20Hibernate/readme.md) |
| 5.3 | Spring Security + JWT + OAuth2 | [📄 readme](./05%20-%20Frameworks/5.3%20-%20Spring%20Security/readme.md) |
| 5.4 | Documentação com OpenAPI e Swagger | [📄 readme](./05%20-%20Frameworks/5.4%20-%20Documenta%C3%A7%C3%A3o%20com%20OpenAPI%20e%20Swagger/readme.md) |
| 5.5 | Testes de Integração + Testcontainers | [📄 readme](./05%20-%20Frameworks/5.5%20-%20Testes%20de%20Integra%C3%A7%C3%A3o/readme.md) |

**Checklist:**
- [ ] Spring Core: IoC, DI, @Component, @Service, @Repository
- [ ] Spring Boot: starters, application.yml, profiles, Virtual Threads
- [ ] Spring MVC: REST, @Valid, ControllerAdvice, CORS
- [ ] JPA: entidades, relacionamentos, N+1, lazy loading
- [ ] Spring Data JPA: JpaRepository, query methods, paginação
- [ ] Flyway: migrations versionadas
- [ ] Spring Security + JWT + OAuth2
- [ ] springdoc-openapi + Swagger UI
- [ ] @SpringBootTest + Testcontainers

**Projeto prático:** API RESTful com autenticação JWT, paginação, validação, documentação e testes de integração.

---

## 6. Deploy, DevOps e Boas Práticas de Entrega

📁 [`06 - Deploy e DevOps/`](./06%20-%20Deploy%20e%20DevOps/)

| # | Tópico | Link |
|---|--------|------|
| 6.1 | Docker e Kubernetes | [📄 readme](./06%20-%20Deploy%20e%20DevOps/6.1%20-%20Docker%20e%20Kubernetes/readme.md) |
| 6.2 | CI/CD com GitHub Actions | [📄 readme](./06%20-%20Deploy%20e%20DevOps/6.2%20-%20CI-CD%20com%20GitHub%20Actions/readme.md) |
| 6.3 | Deploy em Nuvem (Render, AWS, GCP) | [📄 readme](./06%20-%20Deploy%20e%20DevOps/6.3%20-%20Deploy%20em%20Nuvem/readme.md) |
| 6.4 | Monitoramento e Observabilidade | [📄 readme](./06%20-%20Deploy%20e%20DevOps/6.4%20-%20Monitoramento%20e%20Observabilidade/readme.md) |

**Checklist:**
- [ ] Dockerfile multi-stage + Docker Compose
- [ ] Kubernetes: Pods, Services, Deployments, ConfigMaps, Secrets
- [ ] GitHub Actions: pipeline build → test → deploy
- [ ] Deploy no Render ou AWS
- [ ] GraalVM Native Image em produção
- [ ] Spring Boot Actuator + Prometheus + Grafana + OpenTelemetry

**Projeto prático:** API rodando em produção com pipeline CI/CD, monitoramento e alertas.

---

## 7. Aprendizado Contínuo e Evolução

📁 [`07 - Aprendizado Contínuo/`](./07%20-%20Aprendizado%20Cont%C3%ADnuo/)

| # | Tópico | Link |
|---|--------|------|
| 7.1 | Padrões de Projeto (15 Design Patterns GoF) | [📄 readme](./07%20-%20Aprendizado%20Cont%C3%ADnuo/7.1%20-%20Padr%C3%B5es%20de%20Projeto/readme.md) |
| 7.2 | Arquitetura (DDD, Hexagonal, Clean, Microsserviços) | [📄 readme](./07%20-%20Aprendizado%20Cont%C3%ADnuo/7.2%20-%20Arquitetura%20de%20Software/readme.md) |
| 7.3 | Ferramentas de Produtividade (Lombok, MapStruct) | [📄 readme](./07%20-%20Aprendizado%20Cont%C3%ADnuo/7.3%20-%20Ferramentas%20de%20Produtividade/readme.md) |
| 7.4 | Soft Skills e Carreira | [📄 readme](./07%20-%20Aprendizado%20Cont%C3%ADnuo/7.4%20-%20Soft%20Skills%20e%20Carreira/readme.md) |

**Checklist:**
- [ ] Design Patterns: Singleton, Factory, Builder, Strategy, Observer, etc.
- [ ] DDD: Entities, Value Objects, Aggregates, Bounded Contexts
- [ ] Arquitetura Hexagonal e Clean Architecture
- [ ] Lombok e MapStruct
- [ ] Code reviews, ADRs, contribuição open-source

**Projeto prático (avançado):** Refatore um módulo do projeto principal usando DDD + Arquitetura Hexagonal.

---

## 8. Tópicos Avançados e Especializações (Opcional)

📁 [`08 - Tópicos Avançados/`](./08%20-%20T%C3%B3picos%20Avan%C3%A7ados/)

| # | Tópico | Link |
|---|--------|------|
| 8.1 | Programação Reativa com Spring WebFlux | [📄 readme](./08%20-%20T%C3%B3picos%20Avan%C3%A7ados/8.1%20-%20Programa%C3%A7%C3%A3o%20Reativa%20com%20Spring%20WebFlux/readme.md) |
| 8.2 | Concorrência Avançada (CompletableFuture, StructuredTaskScope) | [📄 readme](./08%20-%20T%C3%B3picos%20Avan%C3%A7ados/8.2%20-%20Concorr%C3%AAncia%20Avan%C3%A7ada/readme.md) |
| 8.3 | Otimização de Performance e JVM | [📄 readme](./08%20-%20T%C3%B3picos%20Avan%C3%A7ados/8.3%20-%20Otimiza%C3%A7%C3%A3o%20de%20Performance%20e%20JVM/readme.md) |
| 8.4 | Nichos: Kafka, gRPC, GraphQL, DJL, Spark | [📄 readme](./08%20-%20T%C3%B3picos%20Avan%C3%A7ados/8.4%20-%20Nichos%20e%20Ecossistema%20Avan%C3%A7ado/readme.md) |

**Checklist:**
- [ ] Spring WebFlux: Mono, Flux, R2DBC
- [ ] CompletableFuture + Virtual Threads + StructuredTaskScope
- [ ] JVM: GC, VisualVM, JMH, GraalVM AOT
- [ ] Kafka, gRPC, GraphQL, Apache Spark, DJL

---

## Recursos Recomendados

### Documentação Oficial
- [Java 25 LTS — dev.java](https://dev.java)
- [Spring Boot](https://spring.io/projects/spring-boot)
- [Baeldung](https://www.baeldung.com)

### Livros
- Effective Java — Joshua Bloch
- Head First Java
- Clean Code — Robert C. Martin
- Java Concurrency in Practice — Brian Goetz

### Comunidades
- [GUJ](https://www.guj.com.br)
- Discord Java Brasil
- Stack Overflow
- Reddit [r/java](https://www.reddit.com/r/java)

### Prática
- [LeetCode](https://leetcode.com) — fácil/médio para algoritmos
- [HackerRank](https://www.hackerrank.com)
- [Learn Git Branching](https://learngitbranching.js.org)

---

## Dicas Finais para o Sucesso

Pratique todos os dias. Cada conceito vira projeto pequeno com testes e GitHub.
Sempre use a IA primeiro, mas nunca aceite sem entender.
Mantenha o GitHub atualizado com projetos bem explicados e o arquivo `decisoes.md`.
Fique de olho nas novidades (Java 25+, Spring AI, GraalVM). Com IA você acompanha sem stress.

Se seguir esse caminho com consistência, em 3 ou 4 meses você já entrega APIs profissionais e compete no mercado.

Qualquer dúvida é só falar. Bora codar!
