# 2.2 — Maven e Gradle

Todo projeto Java real precisa de um **gerenciador de build**. Sem ele, você teria que baixar JARs manualmente, configurar classpath na mão, compilar cada arquivo na ordem certa e distribuir o projeto de forma frágil. Maven e Gradle resolvem tudo isso — e muito mais.

---

## Por que gerenciadores de build?

Um gerenciador de build cuida de:

- **Gerenciamento de dependências:** baixa bibliotecas do repositório central e resolve conflitos de versão automaticamente
- **Compilação padronizada:** todos os desenvolvedores compilam da mesma forma
- **Execução de testes:** `mvn test` ou `gradle test` roda todos os testes automaticamente
- **Empacotamento:** gera JARs, WARs, Docker images
- **Integração com CI/CD:** pipelines dependem de comandos padronizados
- **Reprodutibilidade:** builds determinísticos — o mesmo código gera o mesmo resultado

---

## Apache Maven

Maven é o gerenciador de build mais usado no ecossistema Java enterprise. Adota a filosofia de **convenção sobre configuração**: se você segue a estrutura padrão, quase nada precisa ser configurado.

### Estrutura padrão de projeto Maven

```
meu-projeto/
├── pom.xml                          ← coração do projeto
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/empresa/app/     ← código-fonte principal
│   │   └── resources/
│   │       ├── application.properties
│   │       └── META-INF/
│   └── test/
│       ├── java/
│       │   └── com/empresa/app/     ← testes (espelha a estrutura de main)
│       └── resources/
│           └── application-test.properties
├── target/                          ← gerado pelo Maven (não versionar)
│   ├── classes/
│   ├── test-classes/
│   └── meu-projeto-1.0.jar
└── .mvn/
    └── wrapper/
        ├── maven-wrapper.jar
        └── maven-wrapper.properties
```

### pom.xml explicado linha a linha

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <!-- Versão do modelo do POM — sempre 4.0.0 para Maven 2/3 -->
    <modelVersion>4.0.0</modelVersion>

    <!-- ===== COORDENADAS DO PROJETO (GAV — obrigatórias) ===== -->

    <!-- groupId: identifica sua organização (como um package Java reverso) -->
    <groupId>com.minha-empresa</groupId>

    <!-- artifactId: nome único do artefato dentro do grupo -->
    <artifactId>sistema-financeiro</artifactId>

    <!-- version: SNAPSHOT = em desenvolvimento; sem SNAPSHOT = release -->
    <version>1.0.0-SNAPSHOT</version>

    <!-- packaging: jar (padrão), war, pom (para projetos pai) -->
    <packaging>jar</packaging>

    <!-- Metadados opcionais mas recomendados -->
    <name>Sistema Financeiro</name>
    <description>API REST para gestão financeira</description>

    <!-- ===== HERANÇA (usado com Spring Boot) ===== -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.0</version>
        <!-- relativePath vazio = buscar no repositório remoto -->
        <relativePath/>
    </parent>

    <!-- ===== PROPRIEDADES ===== -->
    <properties>
        <!-- Versão do Java — usada pelo maven-compiler-plugin -->
        <java.version>21</java.version>
        <!-- Encoding padrão — evita problemas de caracteres especiais -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!-- Versões centralizadas — boa prática para evitar inconsistências -->
        <mapstruct.version>1.6.0</mapstruct.version>
        <testcontainers.version>1.20.0</testcontainers.version>
    </properties>

    <!-- ===== DEPENDÊNCIAS ===== -->
    <dependencies>

        <!-- Dependência de compilação (scope padrão = compile) -->
        <!-- Incluída no JAR final, disponível em toda a aplicação -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- version omitida — herdada do parent (BOM) -->
        </dependency>

        <!-- scope=provided: necessária para compilar, mas o servidor fornece em runtime -->
        <!-- Ex: Servlet API em projetos WAR -->
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.1.0</version>
            <scope>provided</scope>
        </dependency>

        <!-- scope=runtime: não usada para compilar, mas necessária em runtime -->
        <!-- Ex: driver JDBC — você programa contra a interface, não a implementação -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- scope=test: só disponível durante os testes -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- optional=true: incluída no classpath mas não propagada para dependentes -->
        <!-- Ex: Lombok é processado em tempo de compilação, não precisa ir para o JAR -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Excluindo uma dependência transitiva indesejada -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.fasterxml.jackson.core</groupId>
                    <artifactId>jackson-annotations</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

    </dependencies>

    <!-- ===== BUILD ===== -->
    <build>
        <!-- Nome do artefato gerado (padrão: artifactId-version) -->
        <finalName>sistema-financeiro</finalName>

        <plugins>
            <!-- Compilador Java — define a versão de source e target -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <release>21</release>
                    <!-- Processadores de anotação (ex: Lombok + MapStruct) -->
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </path>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>

            <!-- Plugin do Spring Boot — gera fat JAR executável -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <!-- Exclui dependências optional do fat JAR -->
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>

            <!-- Plugin de testes — configurações do Surefire -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <!-- Rodar testes em paralelo (cuidado com estado compartilhado) -->
                    <parallel>methods</parallel>
                    <threadCount>4</threadCount>
                    <!-- Excluir testes de integração (rodam em fase separada) -->
                    <excludes>
                        <exclude>**/*IT.java</exclude>
                        <exclude>**/*IntegrationTest.java</exclude>
                    </excludes>
                </configuration>
            </plugin>

        </plugins>
    </build>

</project>
```

### Ciclo de vida do Maven

O Maven possui 3 ciclos de vida independentes. O mais usado é o **default**:

```
validate      → Valida se o projeto está correto e todas as informações estão disponíveis
    ↓
initialize    → Inicializa o estado do build (cria diretórios, define propriedades)
    ↓
compile       → Compila o código-fonte em /target/classes
    ↓
test-compile  → Compila os testes em /target/test-classes
    ↓
test          → Executa os testes com um framework adequado (JUnit, TestNG)
    ↓
package       → Empacota o código compilado (JAR, WAR) em /target
    ↓
verify        → Executa verificações adicionais (testes de integração, quality gates)
    ↓
install       → Instala o pacote no repositório local (~/.m2/repository)
    ↓
deploy        → Envia o pacote para o repositório remoto (Nexus, Artifactory)
```

> **Importante:** ao executar uma fase, o Maven executa todas as fases anteriores automaticamente. `mvn package` também compila e testa.

### Comandos essenciais do Maven

```bash
# Compilar o projeto
mvn compile

# Executar apenas os testes
mvn test

# Compilar, testar e gerar o JAR em /target
mvn package

# Pular os testes (útil para builds rápidos — não recomendado em CI)
mvn package -DskipTests

# Instalar no repositório local (~/.m2)
mvn install

# Limpar a pasta /target
mvn clean

# Limpar e empacotar (mais comum no dia a dia)
mvn clean package

# Ver a árvore de dependências (identifica conflitos de versão)
mvn dependency:tree

# Ver dependências desatualizadas
mvn versions:display-dependency-updates

# Gerar relatório de site com documentação e métricas
mvn site

# Executar a aplicação Spring Boot
mvn spring-boot:run

# Baixar fontes das dependências (útil para depuração)
mvn dependency:sources
```

### Como adicionar dependências

Sempre busque as coordenadas no [MVN Repository](https://mvnrepository.com/) ou [search.maven.org](https://search.maven.org/).

**Exemplo: adicionando Jackson, Lombok e Spring Boot Data JPA:**

```xml
<dependencies>
    <!-- Jackson — serialização/deserialização JSON -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.18.0</version>
    </dependency>

    <!-- Lombok — reduz boilerplate com anotações -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.34</version>
        <optional>true</optional>
    </dependency>

    <!-- Spring Data JPA — repositórios com JPA/Hibernate -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
        <!-- versão gerenciada pelo spring-boot-starter-parent -->
    </dependency>
</dependencies>
```

### Profiles Maven para ambientes diferentes

Profiles permitem ter configurações diferentes para desenvolvimento, testes e produção:

```xml
<profiles>

    <!-- Perfil de desenvolvimento (ativado por padrão) -->
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <db.url>jdbc:h2:mem:devdb</db.url>
            <log.level>DEBUG</log.level>
        </properties>
    </profile>

    <!-- Perfil de produção -->
    <profile>
        <id>prod</id>
        <activation>
            <!-- Ativar quando a variável de ambiente AMBIENTE=prod -->
            <property>
                <name>AMBIENTE</name>
                <value>prod</value>
            </property>
        </activation>
        <properties>
            <db.url>jdbc:postgresql://prod-db:5432/financeiro</db.url>
            <log.level>WARN</log.level>
        </properties>
        <build>
            <plugins>
                <!-- Habilitar checagens adicionais em prod -->
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-enforcer-plugin</artifactId>
                    <executions>
                        <execution>
                            <id>enforce-no-snapshots</id>
                            <goals><goal>enforce</goal></goals>
                            <configuration>
                                <rules>
                                    <requireReleaseDeps/>
                                </rules>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>

</profiles>
```

```bash
# Ativar o perfil de produção explicitamente
mvn clean package -Pprod

# Ativar múltiplos perfis
mvn clean package -Pprod,docker
```

### Maven Wrapper — Por que usar

O Maven Wrapper (`mvnw`) garante que todos os desenvolvedores e o CI/CD usem **exatamente a mesma versão do Maven**, sem precisar instalá-lo globalmente.

```bash
# Gerar o wrapper no projeto (precisa ter Maven instalado uma vez)
mvn wrapper:wrapper

# A partir daí, usar sempre o wrapper
./mvnw clean package          # Linux/macOS
.\mvnw.cmd clean package      # Windows

# O wrapper.properties define qual versão baixar automaticamente
cat .mvn/wrapper/maven-wrapper.properties
# distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip
```

> Sempre versionar os arquivos `mvnw`, `mvnw.cmd` e `.mvn/` no Git.

---

## Gradle

O Gradle é mais moderno que o Maven, usa uma DSL Groovy ou Kotlin em vez de XML, e é mais rápido graças ao **incremental build** — só recompila o que mudou.

### build.gradle vs build.gradle.kts

**Groovy DSL** (`build.gradle`) — mais comum em projetos legados:

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.4.0'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'com.minha-empresa'
version = '1.0.0-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
    maxParallelForks = Runtime.runtime.availableProcessors()
}
```

**Kotlin DSL** (`build.gradle.kts`) — recomendado para novos projetos (type-safe, melhor autocomplete):

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.4.0"
    id("io.spring.dependency-management") version "1.1.6"
}

group = "com.minha-empresa"
version = "1.0.0-SNAPSHOT"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.test {
    useJUnitPlatform()
    maxParallelForks = Runtime.getRuntime().availableProcessors()
}
```

### Comandos Gradle

```bash
# Compilar
gradle compileJava

# Executar testes
gradle test

# Build completo (compila + testa + gera JAR)
gradle build

# Pular testes
gradle build -x test

# Limpar e rebuildar
gradle clean build

# Ver dependências
gradle dependencies
gradle dependencies --configuration runtimeClasspath

# Rodar a aplicação
gradle bootRun   # Spring Boot

# Listar todas as tasks disponíveis
gradle tasks

# Ver quais tasks seriam executadas sem executar (dry run)
gradle build --dry-run
```

### Gradle Wrapper

```bash
# Gerar o wrapper
gradle wrapper --gradle-version 8.10

# Usar o wrapper (recomendado sempre)
./gradlew build          # Linux/macOS
.\gradlew.bat build      # Windows
```

O arquivo `gradle/wrapper/gradle-wrapper.properties`:

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.10-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

---

## Quando usar Maven vs Gradle

| Critério                         | Maven                                | Gradle                              |
|----------------------------------|--------------------------------------|-------------------------------------|
| **Projetos existentes/legados**  | Muito comum, manter consistência     | Migrar só se houver ganho real      |
| **Projetos Android**             | Não usado                            | Padrão obrigatório                  |
| **Performance em builds grandes**| Mais lento (sempre recompila tudo)   | Muito mais rápido (incremental)     |
| **Configuração complexa**        | XML verboso, mas previsível          | Kotlin DSL flexível e expressivo    |
| **Curva de aprendizado**         | Menor (convenção clara)              | Maior (mais poder = mais opções)    |
| **Ecossistema Spring Boot**      | Ambos suportados oficialmente        | Ambos suportados oficialmente       |
| **Equipes grandes**              | Menos variação = mais previsível     | Pode ficar complexo demais          |
| **Projetos novos em 2026**       | Ainda válido                         | Recomendado pelo desempenho         |

> **Regra prática:** em projeto legado, siga o que já existe. Em projeto novo, use Gradle com Kotlin DSL se a equipe topar a curva de aprendizado; caso contrário, Maven é sempre uma escolha segura.


---

## GraalVM Native Image

GraalVM Native Image usa AOT (Ahead-of-Time compilation) para transformar sua aplicação em um executável nativo, sem JVM em runtime. O resultado é startup em ~50ms, consumo de memória dramaticamente menor e um binário único — características valiosas para serverless e microsserviços com muitas instâncias.

Entender por que o Native Image existe e por que ele tem limitações específicas com reflexão requer compreender o que o ClassLoader faz, o ciclo de JIT compilation e por que um fat JAR do Spring Boot pode levar vários segundos para inicializar. Esse contexto está coberto no módulo 8.3 (Otimização de Performance e JVM), que inclui os detalhes completos de configuração, hints de reflexão, instalação do GraalVM e a decisão de quando o custo de build de 5-15 minutos compensa.
