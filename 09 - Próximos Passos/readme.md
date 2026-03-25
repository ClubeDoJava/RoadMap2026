# 09 — Próximos Passos

Você chegou até aqui com Spring Boot em produção, CI/CD funcionando, observabilidade configurada e arquitetura hexagonal no currículo. A pergunta agora não é mais "o que estudar" — é **"para onde ir"**.

Este módulo não tem código. É um mapa de bifurcações: cada trilha aponta para um caminho de especialização diferente, com o contexto de quando faz sentido e por onde começar.

---

## Como escolher sua trilha

Não escolha pelo que parece mais avançado. Escolha pelo que o seu **contexto atual recompensa**:

- O que a sua empresa precisa agora?
- Que tipo de vaga você quer em 12 meses?
- Onde você sente a maior lacuna no seu trabalho diário?

Especialização sem contexto é estudo sem retorno. A trilha certa é a que resolve um problema real que você já está enfrentando.

---

## Trilha 1 — Cloud-Native e Serverless

**Quando faz sentido:** você trabalha com muitas instâncias de microsserviços, tem custo de infraestrutura relevante, ou precisa de startup em milissegundos (AWS Lambda, GCP Cloud Run, Azure Functions).

**O problema que essa trilha resolve:** Spring Boot com JVM tem startup de 3–8 segundos e consome ~200MB de memória em idle. Para serverless, isso significa cold start perceptível e custo alto. Para microsserviços com centenas de instâncias, o footprint total importa.

### Quarkus

Framework cloud-native da Red Hat, implementação de referência de Jakarta EE moderna. DI compilada em tempo de build (sem CGLIB em runtime), Native Image first-class, Dev Services que sobem Postgres/Kafka/Redis automaticamente no modo dev.

**Quando preferir ao Spring Boot:**
- Ambientes serverless onde cold start importa
- Microsserviços com muitas instâncias (custo de memória acumulado)
- Times que trabalham pesado com Jakarta EE (CDI, JAX-RS, MicroProfile)

**Curva de aprendizado:** baixa para quem já conhece Spring Boot — os conceitos são os mesmos, a sintaxe muda. Uma tarde lendo a documentação já deixa claro as diferenças.

**Por onde começar:**
- [quarkus.io/guides](https://quarkus.io/guides) — guias oficiais, excelente qualidade
- Guia de migração Spring → Quarkus na documentação oficial
- Livro: *Quarkus in Action* (Manning)

### Micronaut

Framework da Object Computing (OCI). DI e AOP resolvidos em tempo de compilação via APT — sem reflexão em runtime, sem proxy CGLIB. Menor consumo de memória baseline que o Spring, startup mais rápido mesmo sem Native Image.

**Quando preferir:**
- Times que querem performance de Quarkus mas com API mais próxima do Spring
- Projetos que usam Kotlin intensivamente (suporte de primeira classe)
- Quando Native Image não é obrigatório mas performance importa

**Por onde começar:**
- [micronaut.io/launch](https://micronaut.io/launch) — equivalente ao Spring Initializr
- [docs.micronaut.io](https://docs.micronaut.io) — documentação oficial

### GraalVM Native Image em profundidade

Já mencionado no módulo 8.3 como introdução. Aqui o foco é operacional: configurar hints de reflexão para bibliotecas que o Spring ainda não suporta nativamente, analisar o relatório de build, otimizar o tempo de compilação AOT (5–15 minutos), e entender os trade-offs reais entre throughput de pico (JIT ganha) e startup/footprint (AOT ganha).

**Por onde começar:**
- [graalvm.org/native-image](https://www.graalvm.org/latest/reference-manual/native-image/)
- Spring Boot Native Image support: `spring.aot.enabled=true`

---

## Trilha 2 — Dados, Streaming e Processamento Distribuído

**Quando faz sentido:** você trabalha com volumes grandes de dados, pipelines de eventos em tempo real, ou sistemas que processam mais do que uma API REST consegue absorver de forma síncrona.

### Kafka Streams e ksqlDB

Além do Producer/Consumer básico do módulo 8.4, Kafka tem uma camada de processamento de streams nativa: **Kafka Streams** (biblioteca Java) e **ksqlDB** (SQL sobre tópicos Kafka).

Com Kafka Streams você constrói topologias de processamento: filtros, agregações, joins entre tópicos, janelas de tempo — tudo dentro da JVM, sem cluster separado de processamento.

**Casos de uso:** detecção de fraude em tempo real, agregação de métricas por janela de tempo, enriquecimento de eventos com dados de um banco.

**Por onde começar:**
- [kafka.apache.org/documentation/streams](https://kafka.apache.org/documentation/streams/)
- Livro: *Kafka: The Definitive Guide* — O'Reilly (gratuito no site da Confluent)

### Apache Flink

Processamento de streams distribuído com estado gerenciado, checkpointing e tolerância a falhas. Mais poderoso que Kafka Streams para casos complexos, mas requer infraestrutura própria.

**Quando usar sobre Kafka Streams:** processamento com estado muito grande (não cabe em memória de uma instância), joins complexos entre múltiplos streams, machine learning em streaming.

**Por onde começar:**
- [flink.apache.org/learn-flink](https://flink.apache.org/learn-flink/)
- Flink tem API Java nativa e suporte a Table API (SQL)

### Apache Spark

Processamento em batch de grandes volumes — análise de dados históricos, ETL, relatórios. Tem API Java mas é usado majoritariamente com Scala ou Python (PySpark). Para um dev Java, vale conhecer o modelo de execução (RDD, DataFrame, Dataset) e quando Spark faz sentido vs. uma query SQL bem indexada.

**Quando estudar:** se você vai trabalhar com dados em escala de TBs, engenharia de dados, ou data lakehouse.

---

## Trilha 3 — Arquitetura Avançada

**Quando faz sentido:** você já domina DDD e arquitetura hexagonal (módulo 7.2) e quer resolver problemas de consistência em sistemas distribuídos.

### CQRS — Command Query Responsibility Segregation

Separar o modelo de escrita (Commands) do modelo de leitura (Queries). O modelo de leitura é uma projeção desnormalizada otimizada para consulta — não há joins lentos, só leitura direta.

**Problema que resolve:** em sistemas com leitura muito mais frequente que escrita, o mesmo modelo que garante consistência transacional na escrita é lento para consultas complexas. CQRS elimina esse compromisso.

**Referência:** *Implementing Domain-Driven Design* — Vaughn Vernon (capítulos sobre CQRS)

### Event Sourcing

Em vez de armazenar o estado atual de uma entidade, armazena a **sequência de eventos** que levou a esse estado. O estado atual é reconstruído replaying os eventos.

**Problema que resolve:** auditoria completa de qualquer entidade, capacidade de reconstruir o estado em qualquer ponto no tempo, base natural para CQRS.

**Custo real:** complexidade operacional alta. Não use sem sentir a dor que resolve. Sistemas simples não precisam de Event Sourcing.

**Framework Java:** Axon Framework tem suporte nativo a CQRS + Event Sourcing + Saga.

**Por onde começar:**
- [axoniq.io/axon-framework](https://www.axoniq.io/products/axon-framework)
- Artigo de Martin Fowler sobre Event Sourcing: martinfowler.com/eaaDev/EventSourcing.html

### Saga Pattern

Coordenação de transações distribuídas sem 2PC (Two-Phase Commit). Quando uma operação envolve múltiplos microsserviços e precisa de rollback em caso de falha, o Saga orquestra compensações.

**Duas implementações:**
- **Choreography:** cada serviço publica eventos e reage a eventos dos outros (sem coordenador central — Kafka)
- **Orchestration:** um orquestrador central comanda os passos e as compensações (Axon, Temporal)

**Por onde começar:**
- [microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html) — Chris Richardson
- Livro: *Microservices Patterns* — Chris Richardson

---

## Trilha 4 — IA e LLMs em Sistemas Java

**Quando faz sentido:** você quer construir sistemas que integram modelos de linguagem como parte da lógica de negócio, não apenas como chatbot.

### Spring AI avançado

Além do RAG básico do módulo 8.4:
- **Function Calling:** o modelo chama funções Java definidas por você para buscar dados em tempo real
- **Structured Output:** o modelo retorna JSON mapeado diretamente para Records Java
- **Advisors:** interceptores no pipeline de chat (logging, guardrails, memória persistente)
- **Multi-modality:** imagem + texto como input

### LangChain4j

Alternativa ao Spring AI com filosofia diferente: mais baixo nível, mais controle sobre o pipeline. Tem suporte a agentes com ferramentas, memória de conversação, e integração com mais provedores.

**Quando preferir ao Spring AI:** projetos sem Spring Boot, mais controle sobre o pipeline de RAG, integração com Chroma/Qdrant como vector store.

**Por onde começar:**
- [docs.langchain4j.dev](https://docs.langchain4j.dev)
- Repositório de exemplos: github.com/langchain4j/langchain4j-examples

### Padrões de sistema com LLMs

Os padrões que aparecem em sistemas de produção com LLMs:
- **RAG** (Retrieval-Augmented Generation) — já coberto no 8.4
- **Agentic workflows** — LLM que decide qual ferramenta chamar em loop
- **Guardrails** — validação de input/output para evitar injeção de prompt e respostas inadequadas
- **Observabilidade de LLMs** — rastrear latência, tokens, custo por chamada (Langfuse, Phoenix)

---

## Trilha 5 — Certificações

**Quando faz sentido:** você quer validar conhecimento formalmente, trabalha em empresas que valorizam certificações, ou precisa de diferenciação em processos seletivos.

### OCP 21 — Oracle Certified Professional Java SE 21

A certificação mais reconhecida do ecossistema Java. Cobre os fundamentos da linguagem em profundidade: módulos Java, concorrência, I/O, generics, lambdas, streams, e features do Java 21.

**Pré-requisito:** ter passado pelo OCA/OCA 11 ou ter boa base nos fundamentos.

**Materiais:**
- *OCP Oracle Certified Professional Java SE 21 Developer Study Guide* — Jeanne Boyarsky e Scott Selikoff (referência padrão do mercado)
- Mock exams: Enthuware (mais próximo da prova real)

### AWS/GCP/Azure — Cloud Professional

Para quem já domina o deploy com Docker e Kubernetes (módulo 6), a certificação cloud faz sentido como próximo passo natural.

- **AWS Certified Developer – Associate:** foco em desenvolvimento e deploy na AWS. Cobre Lambda, ECS, RDS, SQS, SNS, API Gateway.
- **Google Cloud Professional Developer:** similar, com foco em GKE e Cloud Run.

**Ordem recomendada:** faça o exame AWS Cloud Practitioner antes do Developer Associate — é mais rápido e valida o vocabulário básico.

### Spring Professional Certification (VMware)

Certificação oficial do Spring Framework. Cobre IoC, AOP, Spring MVC, Spring Boot, Spring Data, Spring Security e testes.

**Relevante para:** empresas que usam Spring intensivamente e valorizam certificação formal. Menos reconhecida no mercado geral do que OCP, mas específica para o ecossistema.

---

## Trilha 6 — Contribuição Open Source

**Quando faz sentido:** você quer aprender como grandes projetos são organizados, ter portfólio público de código revisado por engineers seniores, e dar de volta ao ecossistema que você usa.

### Como encontrar uma issue para começar

```
1. Escolha um projeto que você já usa (Spring Boot, Resilience4j, Testcontainers, etc.)
2. Vá em Issues → filtre por label "good first issue" ou "help wanted"
3. Leia o CONTRIBUTING.md do projeto antes de qualquer coisa
4. Comente na issue: "I'd like to work on this" — espere confirmação
5. Fork → branch → implementação → testes → PR com descrição clara do problema e solução
```

### Projetos com boa experiência para primeiro PR

| Projeto | Dificuldade | Por onde começar |
|---|---|---|
| **Testcontainers** | Baixa | Melhorar documentação, adicionar módulo de container novo |
| **Resilience4j** | Média | Issues de compatibilidade, exemplos na doc |
| **OpenFeature Java SDK** | Baixa/Média | SDK relativamente novo, comunidade receptiva |
| **Spring Boot** | Alta | Leia o processo de contribuição em detalhes — rigoroso |
| **LangChain4j** | Média | Projeto em crescimento, issues frequentes de novos providers |

### O que você vai aprender contribuindo

- Como projetos grandes organizam testes (unitários, integração, compatibilidade)
- Code review de engineers seniores no seu código — feedback gratuito e de qualidade
- Como escrever uma descrição de PR que facilita o review
- O processo de manter compatibilidade retroativa em APIs públicas

---

## Trilha 7 — Staff/Principal Engineer

**Quando faz sentido:** você já tem 5+ anos de experiência, domina pelo menos um domínio técnico profundamente, e quer crescer para impacto de sistema e organização, não apenas de código.

O salto de Senior para Staff não é técnico — é de **escopo de influência**.

### O que muda no Staff

| Senior | Staff |
|---|---|
| Resolve problemas dentro do time | Resolve problemas que afetam múltiplos times |
| Entrega features | Define como features devem ser construídas |
| Code review do time | Decide padrões técnicos da organização |
| Foco no sprint atual | Foco nos próximos 6–12 meses |
| Executa o plano técnico | Cria o plano técnico |

### Habilidades que você precisa desenvolver

**Escrita técnica:** RFCs, ADRs, tech specs. Documentos que convencem outros engineers sem você estar na sala. Comece escrevendo o `decisoes.md` de cada projeto de forma rigorosa.

**Sistemas de larga escala:** como sistemas que funcionam com 10 usuários falham com 10 milhões. Leia *Designing Data-Intensive Applications* (já na lista de livros) e *System Design Interview* de Alex Xu para o vocabulário de entrevistas.

**Influência sem autoridade:** como fazer outros times adotarem uma prática que você acredita ser certa, sem poder de hierarquia. Isso se aprende na prática — comece com tech talks internos no seu time.

**Materiais:**
- *The Staff Engineer's Path* — Tanya Reilly
- *An Elegant Puzzle* — Will Larson
- staffeng.com — cases reais de Staff Engineers em empresas conhecidas
- Blog de Will Larson: lethain.com

---

## Resumo — por onde começar depois do roadmap

| Objetivo em 12 meses | Trilha prioritária |
|---|---|
| Emprego em empresa de produto cloud-native | Trilha 1 (Quarkus) + Certificação AWS |
| Arquiteto de microsserviços | Trilha 3 (CQRS/Event Sourcing) + Kafka Streams |
| Trabalhar com dados em escala | Trilha 2 (Flink/Spark) |
| Produtos com IA embarcada | Trilha 4 (Spring AI/LangChain4j) |
| Diferenciação formal no currículo | Trilha 5 (OCP 21) |
| Crescer para Staff/Principal | Trilha 7 + escrita técnica + contribuição OSS |

> Escolha uma trilha, não todas. Profundidade em uma direção vale mais do que superfície em cinco.
