# Lab: Building a Local PDF Summarizer with Spring Boot & Docker‑Hosted LLM

Welcome to this hands‑on lab! You’ll create a **Spring Boot 3.5** micro‑service that turns any PDF into a concise summary using a **local Large Language Model** (LLM) served through Docker Model Runner.
By the end you will have a running REST endpoint (`POST /api/summarize`) that accepts a PDF and returns a summary – **without ever sending data to the cloud**.

## Learning Objectives

1. Scaffold a modern Spring Boot project with Maven.
2. Wire Spring AI to a locally running LLM (Mistral) via Docker Model Runner.
3. Read and split PDFs page‑by‑page with Spring AI utilities.
4. Chain multiple prompts to build a global summary.
5. Expose the functionality as a clean REST API.
6. (Bonus) Package everything with Docker Compose for a one‑command spin‑up.

## Prerequisites

| Tool           | Minimum Version | Check Command      |
| -------------- | --------------- | ------------------ |
| Java (Temurin) | 21              | `java‑‑version`    |
| Maven          | 3.9             | `mvn ‑‑version`    |
| Docker Desktop | 4.40            | `docker ‑‑version` |

If any command fails, fix it **before** moving on.

## 0 · Download an LLM Once

We’ll use the **Mistral 7B** model because it’s small enough for a laptop yet surprisingly capable.

```bash
# Run this, watch Netflix while ~4 GiB downloads
$ docker model pull ai/mistral
```

*Troubleshooting hint:* If `docker model` isn’t recognised, your Docker Desktop is too old. Upgrade.

## 1 · Create the Project Skeleton

1. Decide on a group and artifact ID that makes sense for you (write them down – you need them later).
2. Run the Maven archetype generator:

```bash
mvn archetype:generate \
  -DgroupId=<your.group> \
  -DartifactId=<your-artifact> \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false
```

> **Hint** Any modern IDE (IntelliJ, VS Code + Extension Pack for Java, etc.) lets you run Maven goals from its GUI.

Open the generated project in your IDE and do an initial *Run* to ensure nothing is broken.

## 2 · Add Dependencies (pom.xml)

Open `pom.xml` and make it Spring‑Boot‑aware:

* Set the parent to `org.springframework.boot:spring-boot-starter-parent:3.5.0`.
* Declare a `<properties>` block with `java.version` = **21** and `spring-ai.version` = **1.0.0**.
* Import the Spring AI BOM (hint: `<dependencyManagement>`).
* Add these **three** starters/dependencies:

  1. `spring-boot-starter-web`
  2. `spring-ai-starter-model-openai`
  3. `spring-ai-pdf-document-reader`
* (Optional but recommended) Add Lombok as *provided* scope.

Let IntelliJ’s auto‑complete do the heavy lifting. When finished, `mvn package` should succeed with no errors.

> **Self‑check:** Does the effective POM show *exactly* Spring Boot 3.5.0 and Spring AI 1.0.0? If not, revisit your `<parent>` and BOM sections.

## 3 · Configure Spring AI (application.yml)

Inside `src/main/resources`, create `application.yml` with **four** keys under `spring.ai.openai`:

```yaml
spring:
  ai:
    openai:
      base-url: <???>
      api-key: <???>
      init:
        pull-model-strategy: when_missing
      chat:
        options:
          model: ai/mistral
```

Fill the two `<?>` placeholders:

* `base-url`: the endpoint where Docker Model Runner exposes models (hint: `http://localhost:12434/engines`).
* `api-key`: a dummy value – local usage doesn’t enforce auth.

Save‑and‑sync. No compilation needed yet.

## 4 · Business Logic — Service Layer

Create `org.<your.group>.service.PdfSummarizationService`.

```java
@Service
@RequiredArgsConstructor
public class PdfSummarizationService {

    private final ChatModel chatModel; // injected by Spring

    public String summarize(MultipartFile pdfFile) throws IOException {
        // 1. Build a ChatClient from chatModel
        // 2. Split the PDF into List<Document> pages using PagePdfDocumentReader
        // 3. For each page, call the LLM with prompt "Summarize this page: ..."
        // 4. Collect page summaries
        // 5. Call the LLM again with prompt "Summarize the whole document: ..."
        // 6. Return the global summary
        return null; // TODO replace
    }
}
```

**Hints:**

* The `PagePdfDocumentReader` constructor takes an `InputStreamResource`, wrap `pdfFile.getInputStream()`.
* `ChatClient.builder(chatModel).build()` gives you a fluent prompt API.
* Use `pages.stream().map(Document::getText)` to get raw page text.
* `String.join("\n", summaries)` glues individual summaries together.

Compile often. IntelliJ will tell you what to import.

## 5 · REST Layer — Controller

Create `org.<your.group>.rest.SummarizationController`.

```java
@RestController
@RequestMapping("/api/summarize")
@RequiredArgsConstructor
public class SummarizationController {

    private final PdfSummarizationService service;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<String> summarizePdf(@RequestParam("file") MultipartFile file) throws IOException {
        // Delegate to service and return
        return null; // TODO replace
    }
}
```

*Return `ResponseEntity.ok(service.summarize(file))` once the service works.*

---

## 6 · Bootstrapping

Replace the archetype’s default `App.java` with:

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

Make sure it lives in a **root package** that encloses `service` and `rest` packages so component scanning picks them up.

## 7 · Run & Test Locally

1. **Start** Docker Model Runner in a separate terminal:

   ```bash
   docker model status
   # if you see "Docker Model Running" you’re good
   ```

2. **Run** the Spring Boot application:

   ```bash
   mvn clean package && java -jar target/<your-artifact>-0.0.1-SNAPSHOT.jar
   ```

3. **POST** a PDF:

   ```bash
   curl -F file=@sample.pdf http://localhost:8080/api/summarize
   ```

   You should receive plaintext JSON‑escaped summary lines.

*Troubleshooting tip:* if you get `ResourceAccessException` pointing to `localhost:12434`, ensure step 1 is really running.

## 8 · Containerise with Docker Compose

Inside the project root, run:

```bash
docker init
```

Choose **Java**, port **8080**, Java version **21**. Accept the defaults.

Open `compose.yaml` and set an environment override so the app inside the container can reach the model on the host:

```yaml
environment:
  SPRING_AI_OPENAI_BASE_URL: http://host.docker.internal:12434/engines
```

Finally:

```bash
docker compose up --build
```

Then hit the same `curl` command but expect a slightly longer cold‑start.

## 9 · Wrap‑Up & Next Steps

* You built an *offline* summariser that keeps data on‑prem.
* You integrated Spring AI with Docker Model Runner in under 150 lines of code.
* You practised *typing* real code – the only reliable way to learn.

Take it further by exploring **vector stores** and **chunking strategies** for larger documents, or swap in a different model (`ai/gemma`, `ai/llama‑3‑8b`, etc.).