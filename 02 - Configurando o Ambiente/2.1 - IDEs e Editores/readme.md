# 2.1 — IDEs e Editores de Código

Escolher a ferramenta certa é uma das decisões mais impactantes na sua produtividade diária. Em 2026, o cenário mudou bastante com a integração de IA diretamente nos editores. Este guia compara as três principais opções e explica como tirar o máximo de cada uma.

---

## Comparativo: Cursor vs IntelliJ IDEA Ultimate vs VS Code

| Característica            | Cursor                  | IntelliJ IDEA Ultimate   | VS Code                        |
|---------------------------|-------------------------|--------------------------|--------------------------------|
| Preço                     | Grátis / $20/mês Pro    | ~$249/ano                | Gratuito                       |
| Peso                      | Médio                   | Pesado                   | Leve                           |
| IA nativa                 | Sim (foco principal)    | Parcial (plugin)         | Parcial (GitHub Copilot)       |
| Suporte Java enterprise   | Bom                     | Excelente                | Razoável                       |
| Refatoração automática    | Boa                     | Excelente                | Boa (com extensões)            |
| Depurador visual          | Sim                     | Excelente                | Sim                            |
| Melhor para               | IA + produtividade      | Projetos grandes/Spring  | Projetos leves / aprendizado   |

---

## Cursor — IA Integrada ao Centro do Fluxo de Trabalho

O Cursor é um fork do VS Code construído do zero com IA no centro do fluxo. É uma escolha produtiva para quem valoriza edição assistida por IA e trabalha com projetos de complexidade moderada.

### Por que o Cursor se destaca

- **Edição em múltiplos arquivos com IA:** o modelo entende o contexto do projeto inteiro, não só o arquivo aberto.
- **Chat com base de código:** pergunte "onde essa interface é implementada?" e ele navega pelo código.
- **Geração e refatoração com contexto real:** não apenas snippets, mas refatorações que respeitam a arquitetura existente.
- **Composer (Agent mode):** executa tarefas complexas como criar um endpoint REST completo com testes.

### Como instalar o Cursor

```bash
# Linux (via download direto)
wget https://download.cursor.sh/linux/appImage/x64 -O cursor.AppImage
chmod +x cursor.AppImage
./cursor.AppImage

# macOS (via brew)
brew install --cask cursor

# Windows: baixe o instalador em https://cursor.sh
```

### Plugins essenciais para Java no Cursor

Como o Cursor é baseado no VS Code, instale as mesmas extensões:

```bash
# Via linha de comando
cursor --install-extension vscjava.vscode-java-pack
cursor --install-extension vscjava.vscode-spring-initializr
cursor --install-extension pivotal.vscode-spring-boot
cursor --install-extension sonarsource.sonarlint-vscode
cursor --install-extension streetsidesoftware.code-spell-checker-portuguese-brazilian
```

Ou pelo painel de extensões (Ctrl+Shift+X), busque:

- **Extension Pack for Java** — suporte completo a Java
- **Spring Boot Extension Pack** — Spring Boot + Initializr
- **SonarLint** — análise estática em tempo real
- **GitLens** — histórico Git avançado
- **Thunder Client** — testes de API REST sem sair do editor

### Configurando Claude como AI Provider no Cursor

O Cursor suporta múltiplos provedores de IA. Para usar Claude (Anthropic):

1. Abra **Cursor Settings** → `Cursor > AI > Model`
2. Em **API Keys**, adicione sua chave da Anthropic
3. Selecione o modelo de maior capacidade disponível na sua assinatura

Ou via arquivo de configuração:

```json
{
  "cursor.aiProvider": "anthropic",
  "cursor.anthropicApiKey": "sk-ant-...",
  "cursor.defaultModel": "claude-opus-4"
}
```

**Dicas de uso com Claude no Cursor:**

- `Ctrl+K` — edita o código selecionado com instrução em linguagem natural
- `Ctrl+L` — abre o chat lateral com contexto do arquivo atual
- `Ctrl+Shift+I` — modo Composer (agente que edita múltiplos arquivos)

---

## IntelliJ IDEA Ultimate — O Mais Completo para Java Enterprise

Para projetos Spring Boot complexos, microsserviços com Kafka, integração com bancos de dados e debugging avançado, o IntelliJ ainda é imbatível.

### Por que usar o IntelliJ

- Análise de código mais profunda que qualquer outra IDE
- Refatorações seguras (renomear classe atualiza XML, SQL, HTML automaticamente)
- Profiler integrado, análise de heap, detecção de vazamentos de memória
- Suporte nativo a Spring, Hibernate, JPA, Docker, Kubernetes
- Banco de dados integrado (DataGrip embutido)

### Instalação

```bash
# Via Toolbox App (recomendado — gerencia versões)
# Baixe em: https://www.jetbrains.com/toolbox-app/

# Via snap no Linux
sudo snap install intellij-idea-ultimate --classic

# Via brew no macOS
brew install --cask intellij-idea
```

### Plugins recomendados

Acesse `File > Settings > Plugins > Marketplace`:

| Plugin              | Para que serve                                              |
|---------------------|-------------------------------------------------------------|
| **SonarLint**       | Análise estática, detecta bugs e code smells em tempo real |
| **Rainbow Brackets**| Colore pares de colchetes/chaves em cores diferentes       |
| **Key Promoter X**  | Mostra o atalho sempre que você usa o mouse desnecessariamente |
| **String Manipulation** | Converte camelCase, snake_case, UPPER_CASE com atalho  |
| **Lombok**          | Suporte ao processamento de anotações do Lombok            |
| **Docker**          | Gerencia containers sem sair da IDE                        |
| **HTTP Client**     | Testa APIs REST diretamente em arquivos `.http`            |

### Atalhos de teclado essenciais do IntelliJ

#### Navegação

| Atalho                    | Ação                                               |
|---------------------------|----------------------------------------------------|
| `Ctrl+N`                  | Abrir classe pelo nome                             |
| `Ctrl+Shift+N`            | Abrir qualquer arquivo pelo nome                   |
| `Ctrl+Alt+Shift+N`        | Abrir símbolo (método, variável) pelo nome         |
| `Ctrl+E`                  | Arquivos recentes                                  |
| `Ctrl+B` / `Ctrl+Click`   | Ir para declaração                                 |
| `Alt+F7`                  | Encontrar todos os usos de um símbolo              |
| `Ctrl+Alt+B`              | Ir para implementação da interface                 |
| `Ctrl+H`                  | Hierarquia de tipos                                |
| `Alt+Home`                | Navegação no breadcrumb                            |
| `Ctrl+G`                  | Ir para linha específica                           |

#### Edição e geração de código

| Atalho                    | Ação                                               |
|---------------------------|----------------------------------------------------|
| `Alt+Insert`              | Gerar código (getter, setter, constructor, equals) |
| `Ctrl+Alt+T`              | Envolver com (try-catch, if, synchronized, etc.)   |
| `Ctrl+Shift+Enter`        | Completar linha atual (adiciona `;` ou `{}`)       |
| `Ctrl+D`                  | Duplicar linha ou seleção                          |
| `Ctrl+Y`                  | Deletar linha atual                                |
| `Ctrl+Shift+Up/Down`      | Mover linha ou bloco para cima/baixo               |
| `Ctrl+/`                  | Comentar/descomentar linha                         |
| `Ctrl+Shift+/`            | Comentar bloco com `/* */`                         |
| `Tab`                     | Aceitar sugestão de autocompletar                  |
| `Ctrl+Space`              | Autocompletar básico                               |
| `Ctrl+Shift+Space`        | Autocompletar inteligente (por tipo)               |

#### Refatoração

| Atalho             | Ação                                                    |
|--------------------|---------------------------------------------------------|
| `Shift+F6`         | Renomear (refatoração segura — atualiza todas as refs)  |
| `Ctrl+Alt+M`       | Extrair método                                          |
| `Ctrl+Alt+V`       | Extrair variável local                                  |
| `Ctrl+Alt+F`       | Extrair campo (field)                                   |
| `Ctrl+Alt+C`       | Extrair constante                                       |
| `Ctrl+Alt+P`       | Extrair parâmetro                                       |
| `F6`               | Mover classe para outro pacote                          |
| `Ctrl+Alt+Shift+T` | Menu completo de refatorações                           |

#### Busca

| Atalho               | Ação                                           |
|----------------------|------------------------------------------------|
| `Shift+Shift`        | Search Everywhere (busca tudo)                 |
| `Ctrl+F`             | Buscar no arquivo atual                        |
| `Ctrl+Shift+F`       | Buscar em todos os arquivos (com filtros)      |
| `Ctrl+R`             | Substituir no arquivo atual                    |
| `Ctrl+Shift+R`       | Substituir em todos os arquivos                |

#### Debug

| Atalho           | Ação                                             |
|------------------|--------------------------------------------------|
| `F9`             | Rodar com debug                                  |
| `F8`             | Step over (próxima linha)                        |
| `F7`             | Step into (entrar no método)                     |
| `Shift+F8`       | Step out (sair do método)                        |
| `Alt+F9`         | Run to cursor (executar até o cursor)            |
| `Ctrl+F8`        | Adicionar/remover breakpoint                     |
| `Alt+F8`         | Avaliar expressão no modo debug                  |

---

## VS Code — Leve, Gratuito e Suficiente para Aprender

Para quem está começando ou trabalha em projetos menores, o VS Code com as extensões certas é uma excelente opção.

### Instalação

```bash
# Linux (Debian/Ubuntu)
sudo apt install code

# Linux via snap
sudo snap install code --classic

# macOS
brew install --cask visual-studio-code

# Windows: https://code.visualstudio.com/
```

### Extension Pack for Java

Instale o pacote oficial da Microsoft que inclui tudo necessário:

```bash
code --install-extension vscjava.vscode-java-pack
```

O pack inclui:
- **Language Support for Java** (Red Hat) — análise semântica, autocompletar
- **Debugger for Java** — debug visual
- **Test Runner for Java** — rodar JUnit direto no editor
- **Maven for Java** — suporte ao Maven
- **Project Manager for Java** — gerenciar projetos

### Configurações mínimas para Java no VS Code

Adicione ao `settings.json` (Ctrl+Shift+P → "Open User Settings JSON"):

```json
{
  "java.jdt.ls.java.home": "/usr/lib/jvm/java-21-openjdk-amd64",
  "java.configuration.runtimes": [
    {
      "name": "JavaSE-21",
      "path": "/usr/lib/jvm/java-21-openjdk-amd64",
      "default": true
    }
  ],
  "editor.formatOnSave": true,
  "editor.tabSize": 4,
  "java.format.settings.profile": "GoogleStyle",
  "java.saveActions.organizeImports": true,
  "java.completion.importOrder": [
    "java",
    "javax",
    "org",
    "com",
    ""
  ]
}
```

---

## SonarLint — Análise Estática de Código

O SonarLint é um plugin disponível para todas as IDEs que detecta bugs, vulnerabilidades de segurança e code smells **enquanto você digita**, antes do código ir para produção.

### Instalação

**IntelliJ:** `File > Settings > Plugins > Marketplace > SonarLint`

**VS Code / Cursor:**
```bash
code --install-extension sonarsource.sonarlint-vscode
# ou
cursor --install-extension sonarsource.sonarlint-vscode
```

### O que o SonarLint detecta

```java
// RUIM — SonarLint vai reclamar
public void processarDados(String dado) {
    if (dado == null) {
        System.out.println("dado nulo"); // Regra: use um logger, não System.out
        return;
    }

    String resultado = dado.toString(); // Regra: toString() desnecessário em String

    try {
        // ... processamento
    } catch (Exception e) {
        // Regra: não engula exceções silenciosamente
    }
}

// BOM — sem alertas do SonarLint
private static final Logger log = LoggerFactory.getLogger(MinhaClasse.class);

public void processarDados(String dado) {
    if (dado == null) {
        log.warn("Dado recebido nulo, ignorando processamento");
        return;
    }

    try {
        // ... processamento
    } catch (IOException e) {
        log.error("Erro ao processar dado: {}", dado, e);
        throw new ProcessamentoException("Falha ao processar: " + dado, e);
    }
}
```

### Conectar ao SonarQube/SonarCloud (para times)

No IntelliJ, clique no ícone do SonarLint na barra lateral → **Bind to SonarQube/SonarCloud** → insira a URL do servidor e o token. Assim as regras do time ficam sincronizadas com o CI/CD.

---

## JShell — REPL Java para Testar Código Rapidamente

O JShell (disponível desde Java 9) é uma ferramenta de linha de comando que permite executar código Java interativamente, sem precisar criar um projeto ou uma classe com `main()`.

### Iniciando o JShell

```bash
jshell
# Saída:
# |  Welcome to JShell -- Version 21.0.1
# |  For an introduction type: /help intro
# jshell>
```

### Exemplos práticos

```bash
# Calcular um valor diretamente
jshell> 2 + 2
$1 ==> 4

# Criar uma variável
jshell> var nome = "Java"
nome ==> "Java"

# Testar um método de String
jshell> nome.toUpperCase().repeat(3)
$3 ==> "JAVAJAVAJAVА"

# Testar Streams
jshell> import java.util.List
jshell> List.of(1, 2, 3, 4, 5).stream().filter(n -> n % 2 == 0).toList()
$5 ==> [2, 4]

# Definir um método
jshell> int fatorial(int n) { return n <= 1 ? 1 : n * fatorial(n - 1); }
|  created method fatorial(int)

jshell> fatorial(10)
$7 ==> 3628800

# Listar o que foi definido
jshell> /vars
jshell> /methods
jshell> /list

# Sair
jshell> /exit
```

### Quando usar o JShell

- Testar uma API que você não conhece (ex: "como funciona `String.format`?")
- Verificar comportamento de um método antes de escrever o código de produção
- Aprender features novas do Java sem criar um projeto
- Cálculos rápidos durante o desenvolvimento

---

## GitHub Codespaces — Ambiente na Nuvem

O GitHub Codespaces oferece um ambiente de desenvolvimento completo no navegador (ou conectado ao VS Code local), sem precisar instalar nada na sua máquina.

### Como criar um Codespace para Java

1. Acesse qualquer repositório no GitHub
2. Clique em **Code > Codespaces > Create codespace on main**
3. O ambiente inicia com VS Code no navegador em ~30 segundos

### Configurar um Codespace Java com devcontainer

Crie o arquivo `.devcontainer/devcontainer.json` no seu repositório:

```json
{
  "name": "Java 21 Dev Environment",
  "image": "mcr.microsoft.com/devcontainers/java:21",
  "features": {
    "ghcr.io/devcontainers/features/java:1": {
      "version": "21",
      "installMaven": true,
      "installGradle": true
    },
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "vscjava.vscode-java-pack",
        "pivotal.vscode-spring-boot",
        "sonarsource.sonarlint-vscode",
        "eamodio.gitlens"
      ],
      "settings": {
        "java.jdt.ls.java.home": "/usr/lib/jvm/msopenjdk-current",
        "editor.formatOnSave": true
      }
    }
  },
  "postCreateCommand": "mvn dependency:resolve",
  "forwardPorts": [8080, 5432, 6379],
  "portsAttributes": {
    "8080": { "label": "App Java" },
    "5432": { "label": "PostgreSQL" },
    "6379": { "label": "Redis" }
  }
}
```

### Quando o Codespace é útil

- Apresentações e demos sem risco de "funciona na minha máquina"
- Contribuir para projetos open source rapidamente
- Trabalhar em computadores que não são seus (bibliotecas, trabalho temporário)
- Onboarding de novos devs — ambiente pronto em minutos

---

## Resumo: Qual escolher?

| Situação                                        | Recomendação             |
|-------------------------------------------------|--------------------------|
| Aprendendo Java, projetos pequenos              | VS Code com Extension Pack |
| Foco em produtividade com IA, projetos novos    | **Cursor**               |
| Projetos Spring Boot complexos, enterprise, legado | **IntelliJ IDEA Ultimate** |
| Não quer instalar nada                          | GitHub Codespaces        |
| Analisar qualidade de código em qualquer IDE    | + SonarLint (plugin)     |

> **Nota sobre escolha de IDE:** Cursor e IntelliJ Ultimate têm casos de uso complementares, não excludentes. Para refatorações cross-module, análise de heap, profiling integrado e suporte nativo a Spring/Hibernate em projetos de grande porte, o IntelliJ continua sendo a ferramenta mais completa disponível. Muitos profissionais usam Cursor no dia a dia e IntelliJ para debugging avançado e análise de projetos legados. Escolha com base no seu projeto atual, não em tendência.
