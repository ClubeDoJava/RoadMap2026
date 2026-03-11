# 6.1 - Docker e Kubernetes

## O que é containerização e por que usar

Containerização é a prática de empacotar uma aplicação junto com todas as suas dependências (runtime, bibliotecas, configurações) em uma unidade isolada chamada **container**. Ao contrário de máquinas virtuais, containers compartilham o kernel do sistema operacional host, tornando-os muito mais leves e rápidos.

**Por que usar containers:**

- **Consistência de ambiente:** "funciona na minha máquina" deixa de existir
- **Isolamento:** cada aplicação roda em seu próprio ambiente sem conflitos
- **Portabilidade:** roda igual em desenvolvimento, staging e produção
- **Escalabilidade:** fácil de criar e destruir instâncias sob demanda
- **Velocidade de deploy:** imagens prontas são iniciadas em segundos

---

## Dockerfile para Spring Boot (multi-stage build otimizado)

O multi-stage build é essencial: a primeira etapa compila o projeto e a segunda gera a imagem final enxuta, sem ferramentas de build.

```dockerfile
# =====================================================
# Etapa 1: Build (usa imagem completa com Maven + JDK)
# =====================================================
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /app

# Copia apenas os arquivos de dependências primeiro (melhor cache)
COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .

# Baixa as dependências sem compilar o código (cache de dependências)
RUN ./mvnw dependency:go-offline -B

# Copia o código-fonte e compila
COPY src ./src
RUN ./mvnw package -DskipTests -B

# Extrai as camadas do JAR para melhor cache nas próximas builds
RUN java -Djarmode=layertools -jar target/*.jar extract

# =====================================================
# Etapa 2: Runtime (imagem mínima, apenas JRE)
# =====================================================
FROM eclipse-temurin:21-jre-alpine AS runtime

WORKDIR /app

# Cria usuário não-root por segurança
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copia as camadas extraídas (dependências mudam menos que código)
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

# Expõe a porta da aplicação
EXPOSE 8080

# Health check integrado
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

# Usa JarLauncher para aproveitar as camadas
ENTRYPOINT ["java", \
  "-XX:+UseZGC", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## .dockerignore

Evita copiar arquivos desnecessários para o contexto de build, tornando-o mais rápido e seguro:

```
# Arquivos de build local
target/
.mvn/wrapper/
*.iml

# IDE
.idea/
.vscode/
*.class

# Git
.git/
.gitignore

# Ambiente local
.env
.env.*
docker-compose.yml
docker-compose.*.yml

# Documentação e testes
README.md
*.md
```

---

## Docker Compose: PostgreSQL + Spring Boot

```yaml
version: '3.9'

services:
  # ──────────────────────────────────────────
  # Banco de dados PostgreSQL
  # ──────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    container_name: minha-app-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: minhaapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: senhaSegura123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d minhaapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ──────────────────────────────────────────
  # Aplicação Spring Boot
  # ──────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    container_name: minha-app
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/minhaapp
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: senhaSegura123
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/actuator/health || exit 1"]
      interval: 30s
      timeout: 10s
      start_period: 40s
      retries: 3

volumes:
  postgres_data:
    driver: local
```

---

## Comandos essenciais Docker

```bash
# ─── Imagens ───────────────────────────────────────
# Construir imagem a partir do Dockerfile
docker build -t minha-app:1.0 .

# Construir com target específico do multi-stage
docker build --target builder -t minha-app:builder .

# Listar imagens locais
docker images

# Remover imagem
docker rmi minha-app:1.0

# ─── Containers ────────────────────────────────────
# Rodar container em background
docker run -d -p 8080:8080 --name minha-app minha-app:1.0

# Rodar passando variáveis de ambiente
docker run -d -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DATABASE_URL=jdbc:postgresql://host:5432/db \
  --name minha-app minha-app:1.0

# Listar containers em execução
docker ps

# Listar todos os containers (inclusive parados)
docker ps -a

# Ver logs em tempo real
docker logs -f minha-app

# Ver últimas 100 linhas de log
docker logs --tail 100 minha-app

# Executar comando dentro do container
docker exec -it minha-app sh

# Copiar arquivo do container para o host
docker cp minha-app:/app/logs/app.log ./app.log

# ─── Ciclo de vida ─────────────────────────────────
# Parar container
docker stop minha-app

# Iniciar container parado
docker start minha-app

# Remover container parado
docker rm minha-app

# Parar e remover de uma vez
docker stop minha-app && docker rm minha-app

# ─── Docker Compose ────────────────────────────────
# Subir todos os serviços (em background)
docker compose up -d

# Subir reconstruindo as imagens
docker compose up -d --build

# Ver logs de todos os serviços
docker compose logs -f

# Parar todos os serviços
docker compose down

# Parar e remover volumes (cuidado: apaga dados!)
docker compose down -v

# Status dos serviços
docker compose ps

# ─── Limpeza ───────────────────────────────────────
# Remover tudo que não está em uso (imagens, containers, networks, cache)
docker system prune -a
```

---

## Kubernetes básico

Kubernetes (K8s) é um orquestrador de containers: gerencia deploy, escalabilidade, networking e recuperação automática de falhas.

### Conceitos fundamentais

**Pod:** menor unidade do K8s. Contém um ou mais containers que compartilham rede e armazenamento.

**Service:** expõe um conjunto de Pods com um endereço estável. Tipos:
- `ClusterIP` — acesso apenas dentro do cluster (padrão)
- `NodePort` — expõe em uma porta do node (acesso externo básico)
- `LoadBalancer` — cria um load balancer na nuvem (AWS ELB, GCP LB)

**Deployment:** gerencia a criação e atualização de réplicas de Pods com zero-downtime.

**ConfigMap:** armazena configurações não-sensíveis como variáveis de ambiente ou arquivos de config.

**Secret:** armazena dados sensíveis (senhas, tokens) codificados em base64.

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
  namespace: default
  labels:
    app: minha-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: minha-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: minha-app
    spec:
      containers:
        - name: minha-app
          image: minha-app:1.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: db-password
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-app-service
  namespace: default
spec:
  selector:
    app: minha-app
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80        # Porta externa
      targetPort: 8080 # Porta do container
```

### ConfigMap e Secret

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"
  LOG_LEVEL: "INFO"

---
# secret.yaml (valores devem ser base64: echo -n "valor" | base64)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  db-password: c2VuaGFTZWd1cmExMjM=
  jwt-secret: bWluaGFDaGF2ZUpXVFNlY3JldGE=
```

### Comandos kubectl essenciais

```bash
# Aplicar arquivos de configuração
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ./k8s/  # aplica todos os YAMLs da pasta

# Verificar status
kubectl get pods
kubectl get services
kubectl get deployments

# Ver logs de um pod
kubectl logs -f nome-do-pod

# Executar shell dentro de um pod
kubectl exec -it nome-do-pod -- sh

# Escalar deployment
kubectl scale deployment minha-app --replicas=5

# Atualizar imagem (rolling update)
kubectl set image deployment/minha-app minha-app=minha-app:2.0

# Verificar rollout
kubectl rollout status deployment/minha-app

# Reverter para versão anterior
kubectl rollout undo deployment/minha-app
```

---

## ArgoCD: CD declarativo

ArgoCD é uma ferramenta de Continuous Delivery que implementa o padrão GitOps: o estado desejado do cluster fica no Git, e o ArgoCD garante que o cluster sempre corresponda a esse estado.

**Como funciona:**
1. Você faz push de um novo `deployment.yaml` (com nova tag de imagem) para o repositório Git
2. ArgoCD detecta a mudança (polling ou webhook)
3. ArgoCD aplica automaticamente as mudanças no cluster Kubernetes
4. Se o estado do cluster divergir do Git, ArgoCD detecta e pode corrigir automaticamente (auto-sync)

**Vantagens:**
- Histórico completo de deploys rastreado pelo Git
- Rollback instantâneo: basta reverter o commit
- Interface web para visualizar o estado de todos os recursos
- Suporte a múltiplos clusters e múltiplos ambientes

---

## Serverless com Java

### AWS Lambda com Java

AWS Lambda executa código sem gerenciar servidores. Para Java, a AWS oferece runtime para Java 21 com suporte a Virtual Threads.

```java
// Handler de uma função Lambda
public class ProdutoHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event,
            Context context) {

        String id = event.getPathParameters().get("id");
        // lógica de negócio...
        String corpo = "{\"id\": \"" + id + "\", \"nome\": \"Produto X\"}";

        return new APIGatewayProxyResponseEvent()
                .withStatusCode(200)
                .withBody(corpo)
                .withHeaders(Map.of("Content-Type", "application/json"));
    }
}
```

**Problema com Java em Lambda:** cold start — JVM demora para inicializar. Solução: GraalVM Native Image reduz o cold start de ~3s para ~100ms.

### Knative

Knative é uma plataforma serverless sobre Kubernetes que permite escalar pods até zero quando não há tráfego e escalar rapidamente quando o tráfego volta.

---

## Exemplo completo: Dockerfile com GraalVM Native

```dockerfile
# =====================================================
# Etapa 1: Build nativo com GraalVM
# =====================================================
FROM ghcr.io/graalvm/native-image-community:21 AS builder

WORKDIR /app

COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .
RUN ./mvnw dependency:go-offline -B

COPY src ./src

# Compila para binário nativo (leva vários minutos, mas o resultado é ultra-rápido)
RUN ./mvnw native:compile -Pnative -DskipTests -B

# =====================================================
# Etapa 2: Imagem mínima — apenas o binário
# =====================================================
FROM debian:bookworm-slim AS runtime

WORKDIR /app

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

# Copia apenas o binário nativo compilado
COPY --from=builder /app/target/minha-app ./minha-app

EXPOSE 8080

HEALTHCHECK --interval=15s --timeout=5s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

# Startup em ~50ms, uso de memória ~50MB (vs ~300MB com JVM)
ENTRYPOINT ["./minha-app"]
```

**Resultados com Native Image:**
- Startup: ~50ms (vs ~3s com JVM)
- Memória em idle: ~50MB (vs ~200-300MB com JVM)
- Imagem Docker: ~80MB (vs ~300MB+ com JRE)
- Custo em AWS Lambda: muito menor (menos tempo de execução)

**Limitações:**
- Build demora mais (5-15 minutos)
- Reflection precisa de configuração extra
- Algumas bibliotecas não são compatíveis
