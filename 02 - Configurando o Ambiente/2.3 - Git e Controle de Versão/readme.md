# 2.3 — Git e Controle de Versão

Git é a ferramenta mais importante do ecossistema de desenvolvimento moderno. Não é opcional — é obrigatório. Sem controle de versão, você não consegue colaborar em equipe, não tem histórico de mudanças, não consegue reverter erros e não tem como integrar com CI/CD.

---

## Por que Git é obrigatório

- **Histórico completo:** quem mudou o quê, quando e por quê
- **Reversão segura:** voltar a qualquer estado anterior sem perder trabalho
- **Trabalho em paralelo:** branches permitem múltiplas features ao mesmo tempo
- **Colaboração:** múltiplos devs no mesmo projeto sem sobrescrever o trabalho um do outro
- **Code review:** Pull Requests são o fluxo padrão de qualidade em times
- **CI/CD:** pipelines são acionados por commits e merges no Git
- **Rastreabilidade:** linkar commits a tasks, issues, bugs

---

## Configuração inicial

```bash
# Nome e e-mail (aparecem em cada commit — use o mesmo do GitHub)
git config --global user.name "Seu Nome"
git config --global user.email "seu@email.com"

# Editor padrão para mensagens de commit
git config --global core.editor "code --wait"        # VS Code
git config --global core.editor "cursor --wait"      # Cursor
git config --global core.editor "nano"               # Nano (simples)
git config --global core.editor "vim"                # Vim

# Branch padrão (substitui 'master' por 'main')
git config --global init.defaultBranch main

# Rebase em vez de merge no git pull (mais limpo)
git config --global pull.rebase true

# Colorir saída do terminal
git config --global color.ui auto

# Configurar aliases úteis
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.st "status -sb"
git config --global alias.unstage "reset HEAD --"

# Ver todas as configurações
git config --list --global

# Onde fica o arquivo de configuração
cat ~/.gitconfig
```

---

## Comandos essenciais com exemplos reais

### Inicializar e clonar

```bash
# Inicializar um repositório Git em um diretório existente
mkdir sistema-financeiro && cd sistema-financeiro
git init
# Initialized empty Git repository in .../sistema-financeiro/.git/

# Clonar um repositório remoto
git clone https://github.com/usuario/sistema-financeiro.git

# Clonar em um diretório com nome diferente
git clone https://github.com/usuario/sistema-financeiro.git meu-projeto

# Clonar apenas a branch específica (mais rápido para repos grandes)
git clone --branch develop --single-branch https://github.com/usuario/sistema-financeiro.git

# Clonar superficialmente (sem histórico — útil em CI)
git clone --depth 1 https://github.com/usuario/sistema-financeiro.git
```

### Status, diff e log

```bash
# Ver estado atual do repositório
git status
# On branch main
# Changes not staged for commit:
#   modified:   src/main/java/com/app/Service.java
# Untracked files:
#   src/main/java/com/app/NovaFeature.java

# Versão compacta (com o alias configurado acima)
git st
# M  src/main/java/com/app/Service.java
# ?? src/main/java/com/app/NovaFeature.java

# Ver as diferenças ainda não staged
git diff

# Ver diferenças já staged (prontas para commit)
git diff --staged
git diff --cached   # equivalente

# Log simples
git log

# Log compacto — um commit por linha (mais útil)
git log --oneline
# a3f2c1d feat: adicionar endpoint de pagamento
# b8e4f9a fix: corrigir validação de CPF
# c1d5e2b refactor: extrair serviço de notificação

# Log com grafo de branches (com o alias 'lg')
git lg
# * a3f2c1d (HEAD -> main) feat: adicionar endpoint de pagamento
# * b8e4f9a fix: corrigir validação de CPF
# | * f9a2c3d (feature/relatorio) feat: adicionar relatório mensal
# |/
# * c1d5e2b refactor: extrair serviço de notificação

# Log de um arquivo específico
git log --oneline -- src/main/java/com/app/PagamentoService.java

# Log das últimas 5 entradas
git log -5 --oneline

# Quem modificou cada linha de um arquivo (culpar)
git blame src/main/java/com/app/PagamentoService.java

# Buscar por palavra na mensagem de commit
git log --grep="pagamento" --oneline

# Buscar por mudança no conteúdo do código
git log -S "calcularJuros" --oneline
```

### Staging e commit

```bash
# Adicionar arquivo específico (preferível)
git add src/main/java/com/app/PagamentoService.java

# Adicionar múltiplos arquivos específicos
git add src/main/java/com/app/PagamentoService.java src/test/java/com/app/PagamentoServiceTest.java

# Adicionar todos os arquivos modificados (use com cuidado)
git add .

# Adicionar interativamente (escolhe partes do arquivo)
git add -p src/main/java/com/app/PagamentoService.java

# Ver o que está staged antes de commitar
git diff --staged

# Fazer o commit
git commit -m "feat: adicionar cálculo de juros compostos no serviço de pagamento"

# Commitar tudo de uma vez (add + commit — só para arquivos já rastreados)
git commit -am "fix: corrigir arredondamento no cálculo de juros"

# Abrir o editor para mensagem de commit longa
git commit
```

### Branches

```bash
# Listar branches locais
git branch

# Listar branches locais e remotas
git branch -a

# Criar uma nova branch
git branch feature/autenticacao-oauth

# Mudar para uma branch existente
git checkout feature/autenticacao-oauth
# Forma moderna (Java 21 era, use switch)
git switch feature/autenticacao-oauth

# Criar e mudar para nova branch ao mesmo tempo
git checkout -b feature/autenticacao-oauth
# Forma moderna
git switch -c feature/autenticacao-oauth

# Renomear branch atual
git branch -m novo-nome

# Deletar branch (somente se já foi merged)
git branch -d feature/autenticacao-oauth

# Forçar deleção (cuidado — pode perder trabalho)
git branch -D feature/autenticacao-oauth

# Criar branch a partir de uma tag ou commit específico
git switch -c hotfix/v1.2.1 v1.2.0
```

### Merge vs Rebase

**Merge** — preserva o histórico de todas as branches:

```bash
# Estando na branch main, integrar a feature
git switch main
git merge feature/autenticacao-oauth

# Resultado: cria um "merge commit" que une as duas linhas do histórico
# *   a3f2c1d (HEAD -> main) Merge branch 'feature/autenticacao-oauth'
# |\
# | * f9a2c3d feat: adicionar OAuth com Google
# | * b8e4f9a feat: criar tela de login
# |/
# * c1d5e2b chore: configurar dependências iniciais
```

**Rebase** — reescreve o histórico como se a feature tivesse sido criada a partir do estado atual do main:

```bash
# Estando na branch feature, atualizar com o main
git switch feature/autenticacao-oauth
git rebase main

# Resultado: histórico linear, sem merge commits
# * f9a2c3d (HEAD -> feature/autenticacao-oauth) feat: adicionar OAuth com Google
# * b8e4f9a feat: criar tela de login
# * c1d5e2b (main) chore: configurar dependências iniciais
```

**Quando usar cada um:**

| Situação                                   | Use                    |
|--------------------------------------------|------------------------|
| Integrar feature branch no main            | **Merge** (via PR)     |
| Atualizar sua feature com o main           | **Rebase**             |
| Histórico público/compartilhado            | **Merge** (não reescreve) |
| Histórico local ainda não publicado        | **Rebase** (deixa linear) |
| Squash de commits antes do PR              | **Rebase interativo**  |

> **Regra de ouro:** nunca rebase branches públicas (main, develop) — isso reescreve o histórico e causa conflitos para todos os outros devs.

### Pull e push

```bash
# Baixar mudanças do remoto e integrar (com rebase — mais limpo)
git pull --rebase origin main

# Baixar mudanças sem integrar (só atualiza as referências remotas)
git fetch origin

# Ver diferença entre local e remoto antes de integrar
git fetch origin
git diff main origin/main

# Enviar para o remoto
git push origin feature/autenticacao-oauth

# Publicar uma nova branch no remoto e configurar tracking
git push -u origin feature/autenticacao-oauth

# Depois de configurar com -u, pode usar apenas:
git push
git pull
```

### Stash — salvar trabalho temporariamente

```bash
# Guardar as mudanças sem commitar (útil para trocar de branch com urgência)
git stash

# Stash com uma mensagem descritiva
git stash push -m "WIP: implementando validação de CPF"

# Ver a lista de stashes
git stash list
# stash@{0}: On feature/pagamento: WIP: implementando validação de CPF
# stash@{1}: On main: ajustes temporários na configuração

# Recuperar o stash mais recente (mantém na lista)
git stash apply

# Recuperar um stash específico
git stash apply stash@{1}

# Recuperar e remover da lista
git stash pop

# Remover um stash específico
git stash drop stash@{0}

# Remover todos os stashes
git stash clear

# Criar uma branch a partir de um stash
git stash branch feature/nova-branch stash@{0}
```

### Reset — desfazendo mudanças locais

```bash
# --soft: desfaz o commit, mantém as mudanças staged
# Útil para reescrever a mensagem de commit ou adicionar mais arquivos
git reset --soft HEAD~1

# --mixed (padrão): desfaz o commit, mantém as mudanças mas tira do stage
# Útil para reorganizar commits antes de publicar
git reset HEAD~1
git reset --mixed HEAD~1  # equivalente

# --hard: desfaz o commit E descarta todas as mudanças
# PERIGOSO: perda permanente de trabalho não commitado
git reset --hard HEAD~1

# Tirar um arquivo do stage (sem perder as mudanças)
git reset HEAD src/main/java/com/app/Service.java
# Forma moderna
git restore --staged src/main/java/com/app/Service.java

# Descartar mudanças em um arquivo (volta ao estado do último commit)
git checkout -- src/main/java/com/app/Service.java
# Forma moderna
git restore src/main/java/com/app/Service.java
```

> **Aviso sobre `--hard`:** `git reset --hard` descarta permanentemente as mudanças não commitadas. Antes de usar, verifique se há algo importante com `git status` e `git diff`. Você pode recuperar commits "perdidos" com `git reflog` por até 30 dias, mas mudanças nunca commitadas são irrecuperáveis.

### Revert — desfazer com segurança em produção

```bash
# Reverter um commit específico (cria um novo commit que desfaz as mudanças)
git revert a3f2c1d

# Reverter sem abrir o editor (usa mensagem padrão)
git revert --no-edit a3f2c1d

# Reverter múltiplos commits de uma vez
git revert HEAD~3..HEAD

# Preparar o revert sem commitar automaticamente
git revert --no-commit a3f2c1d
```

**Diferença crucial entre reset e revert:**

| Comando        | O que faz                      | Quando usar                              |
|----------------|--------------------------------|------------------------------------------|
| `git reset`    | Reescreve o histórico          | Commits locais, ainda não publicados     |
| `git revert`   | Adiciona um commit que desfaz  | Branches públicas, produção              |

> Em produção, **sempre use `git revert`**. O `git reset` em branch pública quebra o histórico de todos os outros devs.

---

## .gitignore para projetos Java

O `.gitignore` define o que o Git deve ignorar. Nunca versione arquivos compilados, configurações locais de IDE ou segredos.

```bash
# Arquivo .gitignore na raiz do projeto

# ===== Java / JVM =====
*.class
*.jar
*.war
*.ear
*.nar
hs_err_pid*
replay_pid*

# ===== Maven =====
target/
pom.xml.tag
pom.xml.releaseBackup
pom.xml.versionsBackup
pom.xml.next
release.properties
dependency-reduced-pom.xml
buildNumber.properties
.mvn/timing.properties
.mvn/wrapper/maven-wrapper.jar

# ===== Gradle =====
.gradle/
build/
!gradle/wrapper/gradle-wrapper.jar
!**/src/main/**/build/
!**/src/test/**/build/
gradle-app.setting
.gradletasknamecache

# ===== IntelliJ IDEA =====
.idea/
*.iws
*.iml
*.ipr
out/
!**/src/main/**/out/
!**/src/test/**/out/

# ===== VS Code / Cursor =====
.vscode/
!.vscode/settings.json
!.vscode/tasks.json
!.vscode/launch.json
!.vscode/extensions.json
*.code-workspace

# ===== Eclipse =====
.apt_generated
.classpath
.factorypath
.project
.settings
.springBeans
.sts4-cache
bin/
!**/src/main/**/bin/
!**/src/test/**/bin/

# ===== Spring Boot =====
spring-shell.log

# ===== Logs =====
*.log
logs/
log/

# ===== Variáveis de ambiente e segredos =====
.env
.env.*
!.env.example
*.key
*.pem
secrets/
application-local.properties
application-local.yml

# ===== macOS =====
.DS_Store
.AppleDouble
.LSOverride

# ===== Windows =====
Thumbs.db
ehthumbs.db
Desktop.ini

# ===== Docker =====
docker-compose.override.yml

# ===== Testes e cobertura =====
/target/
jacoco.exec
coverage/
```

---

## GitHub Flow

O GitHub Flow é o fluxo de trabalho mais simples e eficaz para a maioria das equipes:

```
main (sempre deployável)
  │
  ├─── feature/autenticacao-oauth
  │         └── commits de trabalho
  │              └── Pull Request → code review → merge → delete branch
  │
  ├─── fix/corrigir-calculo-juros
  │         └── commits de correção
  │              └── Pull Request → code review → merge → delete branch
  │
  └─── (sempre integrar em main via PR)
```

### Passo a passo

```bash
# 1. Atualizar o main local
git switch main
git pull origin main

# 2. Criar branch para a feature
git switch -c feature/autenticacao-oauth

# 3. Desenvolver e commitar
git add src/main/java/com/app/auth/
git commit -m "feat: adicionar configuração do Spring Security"

git add src/main/java/com/app/auth/OAuthController.java
git commit -m "feat: implementar endpoint de callback OAuth"

git add src/test/java/com/app/auth/
git commit -m "test: adicionar testes de integração OAuth"

# 4. Publicar a branch
git push -u origin feature/autenticacao-oauth

# 5. Abrir Pull Request no GitHub
# (via interface web ou GitHub CLI)
gh pr create --title "feat: autenticação OAuth com Google" \
  --body "Implementa login com Google usando OAuth 2.0..."

# 6. Após aprovação e merge, limpar
git switch main
git pull origin main
git branch -d feature/autenticacao-oauth
```

---

## Conventional Commits

Conventional Commits é um padrão de mensagens de commit que torna o histórico legível por humanos e máquinas. O CI/CD pode usar o histórico para gerar changelogs automaticamente e determinar a versão (semver).

### Formato

```
<tipo>[escopo opcional]: <descrição>

[corpo opcional]

[rodapé opcional]
```

### Tipos principais

| Tipo         | Quando usar                                                   | Impacto no semver |
|--------------|---------------------------------------------------------------|-------------------|
| `feat`       | Nova funcionalidade para o usuário                            | MINOR (1.X.0)     |
| `fix`        | Correção de bug                                               | PATCH (1.0.X)     |
| `docs`       | Apenas documentação                                           | Nenhum            |
| `style`      | Formatação, ponto-e-vírgula faltando (sem mudança de lógica)  | Nenhum            |
| `refactor`   | Refatoração sem mudar comportamento externo                   | Nenhum            |
| `test`       | Adicionar ou corrigir testes                                  | Nenhum            |
| `chore`      | Atualização de dependências, configuração de CI               | Nenhum            |
| `perf`       | Melhoria de performance                                       | PATCH             |
| `ci`         | Mudanças nos arquivos e scripts de CI/CD                      | Nenhum            |
| `build`      | Mudanças no sistema de build (Maven, Gradle)                  | Nenhum            |
| `BREAKING CHANGE` | Mudança que quebra compatibilidade (no rodapé)           | MAJOR (X.0.0)     |

### Exemplos reais

```bash
# Funcionalidade simples
git commit -m "feat: adicionar endpoint de criação de pagamento"

# Com escopo (o módulo ou componente afetado)
git commit -m "feat(pagamento): implementar cálculo de juros compostos"

# Correção de bug
git commit -m "fix(auth): corrigir validação de token JWT expirado"

# Documentação
git commit -m "docs: atualizar README com instruções de instalação"

# Refatoração
git commit -m "refactor(usuario): extrair validação para classe separada"

# Testes
git commit -m "test(pagamento): adicionar testes de integração com banco H2"

# Dependências
git commit -m "chore: atualizar Spring Boot para 3.4.0"

# Breaking change — descrição no rodapé
git commit -m "feat(api)!: alterar formato da resposta de pagamento

BREAKING CHANGE: o campo 'valor' foi renomeado para 'montante'
para alinhar com o padrão ISO 20022.
Clientes precisam atualizar suas integrações."
```

---

## GitHub — Trabalhando com repositórios remotos

### Criar repositório e conectar ao local

```bash
# Via GitHub CLI (recomendado)
gh repo create sistema-financeiro --public --clone

# Ou conectar um projeto local a um repositório já criado no GitHub
git remote add origin https://github.com/usuario/sistema-financeiro.git
git branch -M main
git push -u origin main

# Ver os remotos configurados
git remote -v
# origin  https://github.com/usuario/sistema-financeiro.git (fetch)
# origin  https://github.com/usuario/sistema-financeiro.git (push)

# Adicionar um segundo remoto (ex: fork)
git remote add upstream https://github.com/original/sistema-financeiro.git
```

### Pull Requests — como criar e revisar

**Via GitHub CLI:**

```bash
# Criar PR
gh pr create \
  --title "feat: autenticação OAuth com Google" \
  --body "## O que foi feito
- Implementa login com Google usando OAuth 2.0
- Adiciona tela de callback
- Salva token de acesso no banco

## Como testar
1. Configure CLIENT_ID e CLIENT_SECRET no .env
2. Rode a aplicação
3. Acesse /login e clique em 'Entrar com Google'

## Checklist
- [x] Testes unitários
- [x] Testes de integração
- [x] Documentação atualizada"

# Ver PRs abertos
gh pr list

# Ver detalhes de um PR
gh pr view 42

# Fazer checkout de um PR para revisar localmente
gh pr checkout 42

# Aprovar um PR
gh pr review 42 --approve

# Solicitar mudanças
gh pr review 42 --request-changes --body "Falta tratamento de erro no callback"

# Fazer merge
gh pr merge 42 --squash --delete-branch
```

### Issues e Projects

```bash
# Criar issue
gh issue create --title "Bug: cálculo de juros retorna valor errado" \
  --body "Ao calcular juros compostos com taxa de 0.5%, o valor retornado..."

# Listar issues
gh issue list

# Fechar issue via commit (o GitHub fecha automaticamente)
git commit -m "fix: corrigir cálculo de juros compostos

Closes #47"

# Ver projetos
gh project list
```

### GitHub Codespaces

```bash
# Criar um Codespace para o repositório atual
gh codespace create --repo usuario/sistema-financeiro

# Listar Codespaces
gh codespace list

# Abrir no navegador
gh codespace code --web

# Parar um Codespace (economiza créditos)
gh codespace stop
```

---

## .gitignore completo para Java + Maven + Gradle + IntelliJ + VS Code

Salve como `.gitignore` na raiz do projeto:

```bash
# ============================================================
# Java
# ============================================================
*.class
*.log
*.ctxt
.mtj.tmp/
*.jar
*.war
*.nar
*.ear
*.zip
*.tar.gz
*.rar
hs_err_pid*
replay_pid*

# ============================================================
# Maven
# ============================================================
target/
pom.xml.tag
pom.xml.releaseBackup
pom.xml.versionsBackup
pom.xml.next
release.properties
dependency-reduced-pom.xml
buildNumber.properties
.mvn/timing.properties
.mvn/wrapper/maven-wrapper.jar
!.mvn/wrapper/maven-wrapper.properties

# ============================================================
# Gradle
# ============================================================
.gradle/
build/
!gradle/wrapper/gradle-wrapper.jar
!gradle/wrapper/gradle-wrapper.properties
!**/src/main/**/build/
!**/src/test/**/build/
gradle-app.setting
.gradletasknamecache

# ============================================================
# IntelliJ IDEA
# ============================================================
.idea/
!.idea/codeStyles/
!.idea/inspectionProfiles/
*.iws
*.iml
*.ipr
out/
!**/src/main/**/out/
!**/src/test/**/out/

# ============================================================
# VS Code e Cursor
# ============================================================
.vscode/*
!.vscode/settings.json
!.vscode/tasks.json
!.vscode/launch.json
!.vscode/extensions.json
!.vscode/*.code-snippets
.history/
*.vsix

# ============================================================
# Eclipse
# ============================================================
.apt_generated
.classpath
.factorypath
.project
.settings
.springBeans
.sts4-cache
bin/
!**/src/main/**/bin/
!**/src/test/**/bin/

# ============================================================
# Spring Boot
# ============================================================
spring-shell.log

# ============================================================
# Segredos e configurações locais
# ============================================================
.env
.env.local
.env.*.local
!.env.example
*.pem
*.key
*.p12
*.jks
application-local.properties
application-local.yml
application-secret*.properties
secrets/

# ============================================================
# Logs e cobertura
# ============================================================
logs/
*.log.*
jacoco.exec
*.exec
/coverage/
/report/

# ============================================================
# Docker
# ============================================================
docker-compose.override.yml

# ============================================================
# Sistemas operacionais
# ============================================================
# macOS
.DS_Store
.AppleDouble
.LSOverride
Icon
._*
.Spotlight-V100
.Trashes

# Windows
Thumbs.db
Thumbs.db:encdata
ehthumbs.db
ehthumbs_vista.db
Desktop.ini
$RECYCLE.BIN/
*.lnk

# Linux
*~
.fuse_hidden*
.directory
.Trash-*
.nfs*

# ============================================================
# Node (se houver ferramentas frontend no monorepo)
# ============================================================
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.pnp/
.pnp.js
```

---

## Dicas avançadas

### Desfazer qualquer coisa com reflog

O `git reflog` é sua rede de segurança. Ele registra todos os movimentos do HEAD, mesmo commits descartados com `reset --hard`:

```bash
# Ver histórico completo de movimentos
git reflog
# a3f2c1d HEAD@{0}: commit: feat: adicionar pagamento
# b8e4f9a HEAD@{1}: reset: moving to HEAD~1
# c1d5e2b HEAD@{2}: commit: feat: endpoint que eu deletei sem querer

# Recuperar um commit "perdido"
git checkout c1d5e2b

# Ou criar uma branch a partir dele
git switch -c recuperando-feature c1d5e2b
```

### Bisect — encontrar qual commit introduziu um bug

```bash
# Iniciar a busca binária
git bisect start

# Marcar o estado atual como ruim
git bisect bad

# Marcar um commit antigo onde o bug não existia
git bisect good v1.0.0

# Git vai fazer checkout em commits intermediários
# Teste o código e marque como bom ou ruim
git bisect good   # ou
git bisect bad

# Após encontrar o commit culpado, encerrar
git bisect reset
```

### Cherry-pick — pegar um commit específico

```bash
# Aplicar um commit de outra branch na branch atual
git cherry-pick a3f2c1d

# Cherry-pick sem commitar automaticamente
git cherry-pick --no-commit a3f2c1d
```
