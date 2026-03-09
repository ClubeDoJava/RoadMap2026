# 4.1 — Manipulação de Arquivos e IO

Java oferece duas gerações de APIs para trabalhar com arquivos: a API clássica (`java.io`) e a API moderna (`java.nio.file`). Para projetos novos, prefira sempre NIO. A API clássica ainda aparece em código legado, então é importante conhecê-la.

---

## File I/O Clássico (`java.io`)

### A classe `File`

`File` representa um caminho no sistema de arquivos. Ela não lê nem escreve nada sozinha — apenas descreve o caminho e permite operações de metadados.

```java
import java.io.File;

File arquivo = new File("/tmp/dados.txt");

System.out.println(arquivo.exists());          // true ou false
System.out.println(arquivo.isFile());          // é um arquivo?
System.out.println(arquivo.isDirectory());     // é um diretório?
System.out.println(arquivo.length());          // tamanho em bytes
System.out.println(arquivo.getAbsolutePath()); // caminho absoluto

// Criar arquivo vazio (retorna false se já existir)
boolean criado = arquivo.createNewFile();

// Deletar
boolean deletado = arquivo.delete();

// Listar arquivos de um diretório
File dir = new File("/tmp");
File[] arquivos = dir.listFiles();
if (arquivos != null) {
    for (File f : arquivos) {
        System.out.println(f.getName());
    }
}

// Filtrar por extensão
File[] txts = dir.listFiles(f -> f.getName().endsWith(".txt"));
```

### `FileReader` e `FileWriter`

Trabalham com caracteres (texto). Sempre defina o charset explicitamente para evitar comportamentos diferentes em cada sistema operacional.

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

// Escrita
try (FileWriter fw = new FileWriter("saida.txt", StandardCharsets.UTF_8)) {
    fw.write("Linha 1\n");
    fw.write("Linha 2\n");
} catch (IOException e) {
    e.printStackTrace();
}

// Leitura
try (FileReader fr = new FileReader("saida.txt", StandardCharsets.UTF_8)) {
    int caractere;
    while ((caractere = fr.read()) != -1) {
        System.out.print((char) caractere);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

### `BufferedReader` e `BufferedWriter`

O `FileReader` lê um caractere por vez, o que é lento para arquivos grandes. O `BufferedReader` adiciona um buffer interno e disponibiliza `readLine()`, tornando a leitura muito mais eficiente.

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

// Escrita com buffer
try (BufferedWriter bw = new BufferedWriter(
        new FileWriter("arquivo.txt", StandardCharsets.UTF_8))) {
    bw.write("Primeira linha");
    bw.newLine(); // separador correto para o SO atual
    bw.write("Segunda linha");
    bw.newLine();
} catch (IOException e) {
    e.printStackTrace();
}

// Leitura linha a linha (ideal para arquivos grandes)
try (BufferedReader br = new BufferedReader(
        new FileReader("arquivo.txt", StandardCharsets.UTF_8))) {
    String linha;
    while ((linha = br.readLine()) != null) {
        System.out.println(linha);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

> **Por que `try-with-resources`?** Garante que o arquivo seja fechado mesmo se uma exceção for lançada. Sem isso, você pode deixar file descriptors abertos — um recurso escasso do sistema operacional.

---

## Java NIO (`java.nio.file`)

Introduzida no Java 7, a API NIO é mais expressiva, segura e poderosa. Prefira-a em todo código novo.

### `Path` e `Paths`

`Path` representa um caminho de forma imutável e orientada a objetos.

```java
import java.nio.file.*;

Path home = Paths.get("/home/usuario");
Path arquivo = Paths.get("/home/usuario/docs/relatorio.txt");

// Operações de caminho (não tocam o disco)
Path relativo = home.relativize(arquivo);
System.out.println(relativo); // docs/relatorio.txt

Path absoluto = Paths.get("docs/relatorio.txt").toAbsolutePath();
System.out.println(absoluto); // /caminho/atual/docs/relatorio.txt

Path normalizado = Paths.get("/tmp/../home/./usuario").normalize();
System.out.println(normalizado); // /home/usuario

Path combinado = home.resolve("docs/relatorio.txt");
System.out.println(combinado); // /home/usuario/docs/relatorio.txt

// Informações
System.out.println(arquivo.getFileName()); // relatorio.txt
System.out.println(arquivo.getParent());   // /home/usuario/docs
System.out.println(arquivo.getRoot());     // /
```

### `Files` — operações no disco

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.util.List;

Path path = Paths.get("/tmp/exemplo.txt");

// Verificações
Files.exists(path);
Files.isRegularFile(path);
Files.isDirectory(path);
Files.isReadable(path);

// Leitura completa (para arquivos pequenos)
String conteudo = Files.readString(path, StandardCharsets.UTF_8);
List<String> linhas = Files.readAllLines(path, StandardCharsets.UTF_8);

// Escrita
Files.writeString(path, "Conteúdo do arquivo", StandardCharsets.UTF_8);

// Escrita de múltiplas linhas
List<String> dados = List.of("linha 1", "linha 2", "linha 3");
Files.write(path, dados, StandardCharsets.UTF_8);

// Append (não sobrescreve)
Files.writeString(path, "\nNova linha", StandardCharsets.UTF_8,
        StandardOpenOption.APPEND, StandardOpenOption.CREATE);

// Copiar e mover
Path destino = Paths.get("/tmp/copia.txt");
Files.copy(path, destino, StandardCopyOption.REPLACE_EXISTING);
Files.move(path, Paths.get("/tmp/movido.txt"), StandardCopyOption.REPLACE_EXISTING);

// Deletar
Files.delete(path);               // lança exceção se não existir
Files.deleteIfExists(path);       // não lança exceção

// Criar diretórios (incluindo os intermediários)
Files.createDirectories(Paths.get("/tmp/a/b/c"));
```

### `Files.walk` e `Files.find` — percorrer diretórios recursivamente

```java
import java.nio.file.*;
import java.io.IOException;

Path raiz = Paths.get("/home/usuario/projetos");

// Percorrer todos os arquivos recursivamente
try (var stream = Files.walk(raiz)) {
    stream
        .filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
} catch (IOException e) {
    e.printStackTrace();
}

// Files.find com atributos (mais eficiente para filtros por metadados)
try (var stream = Files.find(raiz, Integer.MAX_VALUE,
        (path, attrs) -> attrs.isRegularFile() && attrs.size() > 1024)) {
    stream.forEach(p -> System.out.println(p + " (" + p.toFile().length() + " bytes)"));
} catch (IOException e) {
    e.printStackTrace();
}
```

### Channels e Buffers — `FileChannel` e `ByteBuffer`

Para transferência de alta performance (arquivos grandes, operações binárias, ou quando você precisa de mapeamento de memória).

```java
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;
import java.io.IOException;

Path origem = Paths.get("/tmp/grande.bin");
Path destino = Paths.get("/tmp/grande_copia.bin");

// Cópia eficiente com transferTo (pode usar DMA do SO)
try (FileChannel src = FileChannel.open(origem, StandardOpenOption.READ);
     FileChannel dst = FileChannel.open(destino,
             StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
    src.transferTo(0, src.size(), dst);
} catch (IOException e) {
    e.printStackTrace();
}

// Leitura com ByteBuffer
try (FileChannel channel = FileChannel.open(origem, StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(8192); // buffer de 8KB
    while (channel.read(buffer) > 0) {
        buffer.flip();   // prepara para leitura
        while (buffer.hasRemaining()) {
            byte b = buffer.get();
            // processa byte
        }
        buffer.clear();  // prepara para próxima escrita no buffer
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

> **Quando usar Channels/Buffers?** Para arquivos binários grandes (imagens, vídeos, dumps), transferências entre canais, ou quando precisa de `MappedByteBuffer` (arquivo mapeado em memória). Para texto comum, `Files.readString` e `Files.writeString` são suficientes.

### `WatchService` — monitorar mudanças em tempo real

```java
import java.nio.file.*;
import java.io.IOException;

Path diretorio = Paths.get("/tmp/monitorado");
Files.createDirectories(diretorio);

try (WatchService watcher = FileSystems.getDefault().newWatchService()) {

    diretorio.register(watcher,
            StandardWatchEventKinds.ENTRY_CREATE,
            StandardWatchEventKinds.ENTRY_MODIFY,
            StandardWatchEventKinds.ENTRY_DELETE);

    System.out.println("Monitorando: " + diretorio);

    while (true) {
        WatchKey key = watcher.take(); // bloqueia até haver evento

        for (WatchEvent<?> evento : key.pollEvents()) {
            WatchEvent.Kind<?> tipo = evento.kind();
            Path arquivo = (Path) evento.context();

            if (tipo == StandardWatchEventKinds.ENTRY_CREATE) {
                System.out.println("Criado: " + arquivo);
            } else if (tipo == StandardWatchEventKinds.ENTRY_MODIFY) {
                System.out.println("Modificado: " + arquivo);
            } else if (tipo == StandardWatchEventKinds.ENTRY_DELETE) {
                System.out.println("Deletado: " + arquivo);
            }
        }

        boolean valido = key.reset();
        if (!valido) break; // diretório foi deletado
    }

} catch (IOException | InterruptedException e) {
    e.printStackTrace();
}
```

---

## Serialização

### `Serializable` — e por que evitar em projetos novos

```java
import java.io.*;

// Objeto serializável
public class Usuario implements Serializable {
    // serialVersionUID evita InvalidClassException ao versionar a classe
    @Serial
    private static final long serialVersionUID = 1L;

    private String nome;
    private int idade;
    private transient String senha; // transient = não serializado

    // construtores, getters, setters...
}

// Serializar (objeto → bytes)
try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("usuario.ser"))) {
    oos.writeObject(new Usuario("Ana", 30, "segredo"));
} catch (IOException e) {
    e.printStackTrace();
}

// Desserializar (bytes → objeto)
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("usuario.ser"))) {
    Usuario u = (Usuario) ois.readObject();
    System.out.println(u.getNome());
} catch (IOException | ClassNotFoundException e) {
    e.printStackTrace();
}
```

> **Por que evitar serialização Java em projetos novos?**
> - Frágil: qualquer mudança na classe pode quebrar arquivos existentes
> - Insegura: desserialização de dados externos é vetor de ataque RCE (Remote Code Execution)
> - Verbosa: o formato binário não é legível nem interoperável
>
> **Alternativas modernas:** JSON com Jackson, Protocol Buffers, ou Avro.

---

## Exemplos Práticos

### Lendo CSV e processando com Streams

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.util.*;
import java.util.stream.*;
import java.io.IOException;

record Produto(String nome, String categoria, double preco) {}

public class LeitorCSV {

    public static void main(String[] args) throws IOException {
        Path csvPath = Paths.get("produtos.csv");

        // Conteúdo do CSV:
        // nome,categoria,preco
        // Notebook,Eletrônicos,3500.00
        // Cadeira,Móveis,850.00
        // Monitor,Eletrônicos,1200.00

        List<Produto> produtos = Files.lines(csvPath, StandardCharsets.UTF_8)
                .skip(1) // pula o cabeçalho
                .filter(linha -> !linha.isBlank())
                .map(linha -> {
                    String[] partes = linha.split(",");
                    return new Produto(
                            partes[0].trim(),
                            partes[1].trim(),
                            Double.parseDouble(partes[2].trim())
                    );
                })
                .toList();

        // Total por categoria
        Map<String, Double> totalPorCategoria = produtos.stream()
                .collect(Collectors.groupingBy(
                        Produto::categoria,
                        Collectors.summingDouble(Produto::preco)
                ));

        totalPorCategoria.forEach((cat, total) ->
                System.out.printf("%-15s R$ %.2f%n", cat, total));
    }
}
```

### Gerando relatório em arquivo de texto

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.io.IOException;

public class GeradorRelatorio {

    public static void gerarRelatorio(List<Produto> produtos, Path destino) throws IOException {
        var sb = new StringBuilder();
        var agora = LocalDateTime.now().format(DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm"));

        sb.append("=".repeat(50)).append("\n");
        sb.append("RELATÓRIO DE PRODUTOS — ").append(agora).append("\n");
        sb.append("=".repeat(50)).append("\n\n");

        for (Produto p : produtos) {
            sb.append(String.format("%-20s %-15s R$ %8.2f%n",
                    p.nome(), p.categoria(), p.preco()));
        }

        double total = produtos.stream().mapToDouble(Produto::preco).sum();
        sb.append("\n").append("-".repeat(50)).append("\n");
        sb.append(String.format("TOTAL: R$ %.2f%n", total));

        Files.writeString(destino, sb.toString(), StandardCharsets.UTF_8);
        System.out.println("Relatório gerado em: " + destino.toAbsolutePath());
    }
}
```

### Copiando arquivo com NIO

```java
import java.nio.file.*;
import java.io.IOException;
import java.time.Instant;

public class CopiadorArquivo {

    public static void copiarComBackup(Path origem, Path destino) throws IOException {
        // Se o destino já existe, cria backup com timestamp
        if (Files.exists(destino)) {
            String timestamp = String.valueOf(Instant.now().getEpochSecond());
            Path backup = destino.resolveSibling(destino.getFileName() + "." + timestamp + ".bak");
            Files.move(destino, backup);
            System.out.println("Backup criado: " + backup.getFileName());
        }

        // Cria diretórios intermediários se necessário
        Files.createDirectories(destino.getParent());

        // Copia preservando atributos
        Files.copy(origem, destino,
                StandardCopyOption.REPLACE_EXISTING,
                StandardCopyOption.COPY_ATTRIBUTES);

        System.out.printf("Copiado: %s → %s (%d bytes)%n",
                origem.getFileName(),
                destino.toAbsolutePath(),
                Files.size(destino));
    }

    public static void main(String[] args) throws IOException {
        copiarComBackup(
                Paths.get("/tmp/dados.csv"),
                Paths.get("/tmp/backup/dados.csv")
        );
    }
}
```
