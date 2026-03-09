# 6.3 - Deploy em Nuvem

## Opções e comparação

| Plataforma | Tipo | Free Tier | Complexidade | Melhor para |
|-----------|------|-----------|--------------|-------------|
| **Render** | PaaS | Sim (dorme após 15min) | Baixa | Projetos pessoais, portfólio |
| **Railway** | PaaS | Sim ($5/mês de crédito) | Baixa | Projetos pequenos/médios |
| **Fly.io** | PaaS | Sim (3 VMs pequenas) | Média | Apps globalmente distribuídos |
| **AWS Elastic Beanstalk** | PaaS sobre IaaS | 12 meses free | Média | Apps corporativos na AWS |
| **AWS EC2** | IaaS | 12 meses free (t2.micro) | Alta | Controle total do servidor |
| **GCP Cloud Run** | Serverless Container | Sim (generoso) | Média | Apps com tráfego variável |
| **Heroku** | PaaS | Não (pago) | Baixa | Projetos que pagam pelo conforto |

---

## Deploy no Render (passo a passo)

Render é a plataforma mais simples para começar. Suporta deploy direto do GitHub.

### Pré-requisitos
- Conta no [render.com](https://render.com)
- Repositório no GitHub com a aplicação Spring Boot
- `Dockerfile` na raiz do projeto (ou Render detecta automaticamente projetos Maven)

### Passo a passo

**1. Criar o serviço no Render**

```
Dashboard → New + → Web Service → Connect GitHub → Selecionar repositório
```

**2. Configurar o serviço**

```
Name: minha-app
Region: Oregon (US West) ou Frankfurt (Europa)
Branch: main
Runtime: Docker (se tiver Dockerfile) ou Java
Build Command: ./mvnw package -DskipTests
Start Command: java -jar target/minha-app.jar
Instance Type: Free (0.1 CPU, 512 MB RAM)
```

**3. Configurar variáveis de ambiente**

No painel do Render, vá em `Environment`:

```
SPRING_DATASOURCE_URL=jdbc:postgresql://dpg-xxx.oregon-postgres.render.com/minhadb
SPRING_DATASOURCE_USERNAME=appuser
SPRING_DATASOURCE_PASSWORD=senhaGeradaPeloRender
SPRING_JPA_HIBERNATE_DDL_AUTO=update
JWT_SECRET=minha-chave-jwt-super-secreta-longa
SPRING_PROFILES_ACTIVE=prod
```

**4. Adicionar banco PostgreSQL no Render**

```
Dashboard → New + → PostgreSQL
Name: minha-app-db
Region: mesma do serviço
Plan: Free (1GB storage, expira em 90 dias no free tier)
```

Após criar, copie a "Internal Database URL" e use como `SPRING_DATASOURCE_URL`.

**5. application.properties para produção**

```yaml
# src/main/resources/application-prod.yml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: ${SPRING_JPA_HIBERNATE_DDL_AUTO:validate}
    show-sql: false

server:
  port: ${PORT:8080}

logging:
  level:
    root: INFO
    com.minhaempresa: INFO
```

**Observação importante sobre o free tier:** a instância "dorme" após 15 minutos sem requisições. A primeira requisição após o sono pode levar 30-60 segundos para responder (cold start da JVM).

---

## Deploy no AWS Elastic Beanstalk (passo a passo)

Elastic Beanstalk (EB) gerencia automaticamente EC2, Load Balancer, Auto Scaling e deployment, mas você tem mais controle que no Render.

### Pré-requisitos

```bash
# Instalar AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configurar credenciais
aws configure
# AWS Access Key ID: AKIA...
# AWS Secret Access Key: xxx...
# Default region: us-east-1
# Default output format: json

# Instalar EB CLI
pip install awsebcli
```

### Passo a passo

**1. Inicializar o projeto EB**

```bash
# Na raiz do projeto
eb init

# Responder as perguntas:
# Region: us-east-1
# Application name: minha-app
# Platform: Java 21 running on 64bit Amazon Linux 2023
# SSH: sim (para debug)
```

**2. Criar o ambiente de produção**

```bash
eb create minha-app-prod \
  --instance-type t3.micro \
  --platform "java-21" \
  --region us-east-1
```

**3. Configurar variáveis de ambiente no EB**

```bash
eb setenv \
  SPRING_DATASOURCE_URL=jdbc:postgresql://xxx.rds.amazonaws.com:5432/minhadb \
  SPRING_DATASOURCE_USERNAME=appuser \
  SPRING_DATASOURCE_PASSWORD=senhaSegura \
  SPRING_PROFILES_ACTIVE=prod \
  JWT_SECRET=minha-chave-jwt
```

**4. Fazer o deploy**

```bash
# Empacotar o JAR
./mvnw package -DskipTests

# Deploy do JAR para o EB
eb deploy

# Ver logs
eb logs

# Abrir a aplicação no navegador
eb open

# Status do ambiente
eb status
```

**5. Arquivo .elasticbeanstalk/config.yml gerado automaticamente**

```yaml
branch-defaults:
  main:
    environment: minha-app-prod
    group_suffix: null
global:
  application_name: minha-app
  branch: null
  default_ec2_keyname: minha-chave
  default_platform: java-21
  default_region: us-east-1
  repository: null
  sc: git
```

**6. Arquivo Procfile para o EB saber como iniciar**

```
# Procfile (na raiz do projeto)
web: java -Xms256m -Xmx512m -jar target/minha-app.jar
```

---

## Variáveis de ambiente em produção

Boas práticas para gerenciar configurações sensíveis:

**Nunca commite valores sensíveis.** Use variáveis de ambiente ou serviços de secrets.

```java
// Spring lê variáveis de ambiente automaticamente
// DATABASE_URL → spring.datasource.url (se configurado assim)
// Ou use ${VAR_NAME} no application.yml:

// application.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
```

```bash
# Verificar variáveis disponíveis em produção (AWS EB)
eb ssh
env | grep SPRING

# Render: visível no painel em Environment
# Railway: visível em Variables
```

**Hierarquia de configuração do Spring Boot** (maior prioridade primeiro):
1. Argumentos da linha de comando (`--server.port=8081`)
2. Variáveis de ambiente do SO (`SERVER_PORT=8081`)
3. `application-{profile}.yml` (ex: `application-prod.yml`)
4. `application.yml`
5. Valores padrão do código (`@Value("${port:8080}")`)

---

## GraalVM Native Image em produção

### O que é e por que usar

GraalVM AOT (Ahead-of-Time) compilation transforma seu JAR em um binário nativo para o SO alvo. O resultado é uma aplicação que:

- **Inicia em ~50ms** (vs ~3-5s com JVM)
- **Usa ~50-100MB de RAM** (vs ~200-400MB com JVM)
- **Não precisa de JVM instalada** no servidor
- Ideal para **containers**, **serverless (Lambda)** e **microsserviços**

### Build nativo com Spring Boot

```xml
<!-- pom.xml: adicionar plugin nativo -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```bash
# Build nativo (requer GraalVM instalado ou Docker)
./mvnw native:compile -Pnative -DskipTests

# Build nativo via Docker (não precisa GraalVM local)
./mvnw spring-boot:build-image -Pnative -DskipTests

# Executar o binário nativo
./target/minha-app
```

### Comparativo de performance

| Métrica | JVM (JRE) | Native Image |
|---------|-----------|--------------|
| Startup | ~3-5s | ~50ms |
| RAM idle | ~200MB | ~50MB |
| Tamanho imagem Docker | ~300MB | ~80MB |
| Throughput máximo | Alto (JIT) | Ligeiramente menor |
| Tempo de build | ~30s | ~5-15min |
| Debug | Fácil | Mais difícil |

**Quando usar native:**
- Serverless (AWS Lambda, Cloud Run)
- Muitos microsserviços (custo de infra)
- Requisitos de startup rápido

**Quando ficar com JVM:**
- Aplicações de longa duração e alto throughput
- Debugging e profiling frequentes
- Bibliotecas com reflection extensiva

---

## Configuração de domínio e HTTPS

### Render

1. Vá em `Settings > Custom Domains` do seu serviço
2. Adicione seu domínio (ex: `api.minhaapp.com`)
3. Render fornece o CNAME para configurar no seu DNS
4. No seu provedor DNS (Cloudflare, Namecheap, etc.), adicione:
   ```
   Tipo: CNAME
   Nome: api
   Valor: minha-app.onrender.com
   ```
5. SSL/HTTPS é automático (Render usa Let's Encrypt)

### AWS (com Route 53 + ACM)

```bash
# 1. Solicitar certificado SSL no AWS Certificate Manager
aws acm request-certificate \
  --domain-name api.minhaapp.com \
  --validation-method DNS \
  --region us-east-1

# 2. Validar o domínio via DNS (adicionar registro CNAME fornecido pela AWS)

# 3. Adicionar o certificado ao Load Balancer do Elastic Beanstalk
# Painel EB → Configuration → Load Balancer → Listeners → Add (HTTPS:443)

# 4. Criar registro DNS no Route 53 apontando para o EB
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch file://dns-change.json
```

---

## Rollback de deploy

### Render

```
Dashboard → Events → Clique no deploy anterior → "Rollback to this deploy"
```

### AWS Elastic Beanstalk

```bash
# Listar versões da aplicação
eb appversion

# Fazer rollback para versão específica
eb deploy --version "versão-anterior"

# Ou pelo console AWS:
# Elastic Beanstalk → Application versions → Selecionar versão → Deploy
```

### Kubernetes (mais robusto)

```bash
# Ver histórico de rollouts
kubectl rollout history deployment/minha-app

# Rollback para a versão anterior
kubectl rollout undo deployment/minha-app

# Rollback para uma revisão específica
kubectl rollout undo deployment/minha-app --to-revision=2

# Verificar status após rollback
kubectl rollout status deployment/minha-app
```

### Estratégia de blue-green deploy

Mantém duas versões em produção ao mesmo tempo (blue = atual, green = nova), e troca o tráfego instantaneamente:

```yaml
# Kubernetes: dois deployments, um service que aponta para o ativo
apiVersion: v1
kind: Service
metadata:
  name: minha-app-service
spec:
  selector:
    app: minha-app
    version: blue  # Mude para "green" para fazer o swap
  ports:
    - port: 80
      targetPort: 8080
```

Para rollback: mude `version: green` para `version: blue` no Service — instantâneo, sem downtime.
