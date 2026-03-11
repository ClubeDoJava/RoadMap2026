# 6.2 - CI/CD com GitHub Actions

## O que é CI/CD e por que implementar

**CI (Continuous Integration):** prática de integrar o código de todos os desenvolvedores em um repositório central com frequência (várias vezes ao dia), executando builds e testes automaticamente a cada integração.

**CD (Continuous Delivery/Deployment):** extensão do CI que automatiza a entrega do software até o ambiente de produção.

O mecanismo que torna CI efetivo é simples: quando o desenvolvedor integra com frequência, o diff entre sua branch e a principal é pequeno. Quando um teste falha, o conjunto de mudanças que pode ter causado a falha é pequeno — frequentemente um único commit ou algumas linhas. Isso é o que "detectar bugs cedo" significa na prática: não é magia, é redução do espaço de busca do debug. Em times que integram semanalmente, um bug pode estar escondido em centenas de mudanças. Em times que integram várias vezes ao dia, geralmente está no último commit.

O mesmo princípio se aplica ao deploy: entregas frequentes e incrementais são menos arriscadas do que releases grandes e infrequentes. Um deploy de 50 linhas de mudança tem superfície de problema pequena e é trivial de reverter. Um deploy de 3 semanas de desenvolvimento tem superfície enorme, e reverter é quase sempre mais doloroso do que corrigir em frente.

**Diferença entre Delivery e Deployment:**
- **Continuous Delivery:** o pipeline prepara tudo até estar pronto para produção, mas um humano aprova o deploy final
- **Continuous Deployment:** o deploy para produção é totalmente automático, sem aprovação manual

---

## GitHub Actions: conceitos

| Conceito | Descrição |
|----------|-----------|
| **Workflow** | Processo automatizado definido em um arquivo YAML na pasta `.github/workflows/` |
| **Trigger (on)** | Evento que dispara o workflow: `push`, `pull_request`, `schedule`, `workflow_dispatch` |
| **Job** | Conjunto de steps que rodam em um mesmo runner (máquina) |
| **Step** | Comando individual ou action dentro de um job |
| **Action** | Unidade reutilizável de código (ex: `actions/checkout@v4`) |
| **Runner** | Máquina virtual onde o job executa (ubuntu-latest, windows-latest, macos-latest) |
| **Secret** | Variável sensível configurada no repositório, acessada via `${{ secrets.NOME }}` |
| **Context** | Objetos com informações do workflow: `github`, `env`, `secrets`, `steps`, `jobs` |

---

## Workflow completo: build → test → package → deploy

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

# Evita execuções paralelas do mesmo workflow na mesma branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  JAVA_VERSION: '21'
  JAVA_DISTRIBUTION: 'temurin'

jobs:
  # ─────────────────────────────────────────────────────────
  # Job 1: Build e Testes
  # ─────────────────────────────────────────────────────────
  build-and-test:
    name: Build e Testes
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout do código
        uses: actions/checkout@v4

      - name: Configurar JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          cache: maven  # Cache automático do Maven

      # Cache manual para maior controle (alternativa ao cache do setup-java)
      - name: Cache de dependências Maven
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Compilar projeto
        run: ./mvnw compile -B

      - name: Executar testes unitários
        run: ./mvnw test -B
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass

      - name: Executar testes de integração
        run: ./mvnw verify -B -Pintegration-tests
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass

      - name: Publicar relatório de cobertura
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: target/site/jacoco/
          retention-days: 7

  # ─────────────────────────────────────────────────────────
  # Job 2: Análise de qualidade de código
  # ─────────────────────────────────────────────────────────
  code-quality:
    name: Qualidade de Código
    runs-on: ubuntu-latest
    needs: build-and-test

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Necessário para SonarCloud

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          cache: maven

      - name: Análise SonarCloud
        run: ./mvnw sonar:sonar -B
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # ─────────────────────────────────────────────────────────
  # Job 3: Empacotar e publicar imagem Docker
  # ─────────────────────────────────────────────────────────
  package:
    name: Empacotar imagem Docker
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          cache: maven

      - name: Gerar JAR
        run: ./mvnw package -DskipTests -B

      - name: Login no Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extrair metadados da imagem
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKERHUB_USERNAME }}/minha-app
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build e push da imagem
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ─────────────────────────────────────────────────────────
  # Job 4: Deploy em produção
  # ─────────────────────────────────────────────────────────
  deploy:
    name: Deploy em Produção
    runs-on: ubuntu-latest
    needs: package
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://minha-app.onrender.com

    steps:
      - name: Deploy no Render
        run: |
          curl -X POST "${{ secrets.RENDER_DEPLOY_HOOK_URL }}"

      - name: Aguardar deploy finalizar
        run: sleep 60

      - name: Health check pós-deploy
        run: |
          response=$(curl -s -o /dev/null -w "%{http_code}" \
            https://minha-app.onrender.com/actuator/health)
          if [ "$response" != "200" ]; then
            echo "Health check falhou! Status: $response"
            exit 1
          fi
          echo "Deploy bem-sucedido! Status: $response"
```

---

## Cache de dependências Maven

O cache é um dos maiores ganhos de performance no pipeline. Sem cache, cada run baixa todas as dependências do zero (pode levar 2-5 minutos). Com cache, reutiliza o `.m2` do run anterior.

```yaml
# Opção 1: Cache automático via setup-java (mais simples)
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: maven  # Faz o cache automaticamente

# Opção 2: Cache manual (mais controle)
- name: Cache Maven
  uses: actions/cache@v4
  with:
    path: |
      ~/.m2/repository
      ~/.sonar/cache
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-
```

**Estratégia de chave:**
- `hashFiles('**/pom.xml')` cria um hash de todos os pom.xml
- Se o pom.xml mudar (nova dependência), o cache é invalidado e recriado
- `restore-keys` permite usar um cache parcial como fallback

---

## Secrets no GitHub Actions

Secrets são variáveis sensíveis que nunca aparecem nos logs.

**Como configurar:**
1. Vá em `Settings > Secrets and variables > Actions` no repositório
2. Clique em `New repository secret`
3. Adicione o nome e valor

**Secrets comuns para aplicações Java:**

| Secret | Descrição |
|--------|-----------|
| `DATABASE_URL` | URL do banco em produção |
| `JWT_SECRET` | Chave secreta para assinar tokens JWT |
| `DOCKERHUB_USERNAME` | Usuário do Docker Hub |
| `DOCKERHUB_TOKEN` | Token de acesso do Docker Hub |
| `RENDER_DEPLOY_HOOK_URL` | URL do webhook de deploy do Render |
| `SONAR_TOKEN` | Token do SonarCloud |
| `AWS_ACCESS_KEY_ID` | Chave de acesso AWS |
| `AWS_SECRET_ACCESS_KEY` | Chave secreta AWS |

**Como usar no workflow:**

```yaml
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  JWT_SECRET: ${{ secrets.JWT_SECRET }}

# Ou diretamente em um step
- name: Deploy
  run: ./deploy.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

**Importante:** secrets nunca são expostos em logs. Se você tentar imprimir um secret, o GitHub mascara com `***`.

---

## Deploy automático para Render

```yaml
# No job de deploy, use o Deploy Hook do Render
- name: Trigger deploy no Render
  run: |
    curl -X POST \
      -H "Authorization: Bearer ${{ secrets.RENDER_API_KEY }}" \
      "https://api.render.com/v1/services/${{ secrets.RENDER_SERVICE_ID }}/deploys" \
      -d '{}'
```

## Deploy automático para AWS (Elastic Beanstalk)

```yaml
- name: Deploy no AWS Elastic Beanstalk
  uses: einaregilsson/beanstalk-deploy@v22
  with:
    aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    application_name: minha-app
    environment_name: minha-app-prod
    version_label: ${{ github.sha }}
    region: us-east-1
    deployment_package: target/minha-app.jar
```

---

## Badge de status no README

Adicione o badge no seu README.md para mostrar o status do CI:

```markdown
[![CI/CD Pipeline](https://github.com/seu-usuario/seu-repo/actions/workflows/ci-cd.yml/badge.svg)](https://github.com/seu-usuario/seu-repo/actions/workflows/ci-cd.yml)
```

Resultado visual: ![CI/CD Pipeline](https://img.shields.io/badge/CI%2FCD-passing-brightgreen)

---

## GitLab CI: equivalente em .gitlab-ci.yml

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - package
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  JAVA_VERSION: "21"

cache:
  key: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
  paths:
    - .m2/repository/

build:
  stage: build
  image: eclipse-temurin:21-jdk-alpine
  script:
    - ./mvnw compile -B
  artifacts:
    paths:
      - target/
    expire_in: 1 hour

test:
  stage: test
  image: eclipse-temurin:21-jdk-alpine
  services:
    - postgres:16-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/testdb
    SPRING_DATASOURCE_USERNAME: testuser
    SPRING_DATASOURCE_PASSWORD: testpass
  script:
    - ./mvnw test -B
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml

package:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  only:
    - main
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

deploy-production:
  stage: deploy
  only:
    - main
  when: manual  # Requer aprovação manual para deploy em prod
  script:
    - curl -X POST "$RENDER_DEPLOY_HOOK_URL"
  environment:
    name: production
    url: https://minha-app.onrender.com
```

---

## Exemplo completo: .github/workflows/ci-cd.yml funcional

Este é o arquivo completo pronto para usar em um projeto Spring Boot real:

```yaml
name: Pipeline CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:  # Permite executar manualmente pela interface do GitHub

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    name: Integração Contínua
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Executar testes e gerar JAR
        run: ./mvnw verify -B
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass
          JWT_SECRET: ${{ secrets.JWT_SECRET }}

      - name: Upload do JAR
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 1

  cd:
    name: Deploy Contínuo
    runs-on: ubuntu-latest
    needs: ci
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Download do JAR
        uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: target/

      - name: Login no Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build e push da imagem
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/minha-app:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/minha-app:${{ github.sha }}

      - name: Deploy no Render
        run: curl -X POST "${{ secrets.RENDER_DEPLOY_HOOK_URL }}"

      - name: Notificar sucesso
        if: success()
        run: |
          echo "Deploy realizado com sucesso!"
          echo "Commit: ${{ github.sha }}"
          echo "Autor: ${{ github.actor }}"
```
