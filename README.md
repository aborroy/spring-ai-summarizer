# AI Summarizer

This project is a simple Spring Boot application that:

* Accepts PDF files via a REST API
* Splits the PDF by page
* Uses a local Large Language Model (via [Docker Model Runner](https://docs.docker.com/ai/model-runner/)) to summarize each page
* Produces a final global summary of the entire document

## Requirements

Before you begin, install the following tools:

| Tool       | Version        | Notes                          |
|------------|----------------|--------------------------------|
| Java       | 21             | Tested with OpenJDK 21         |
| Maven      | 3.9+           | For building the project       |
| Docker     | 4.40+          | For running containers & LLM   |

> You need to **download** a model with Docker first:

```bash
docker model pull ai/mistral
````

This will download the Mistral model and keep it running on `http://localhost:12434/engines`.

## Project Setup

You can generate the project structure manually or use your preferred IDE (IntelliJ, VS Code, etc.).

Scaffold the Maven project:

```bash
mvn archetype:generate \
  -DgroupId=org.alfresco.ai.summarize \
  -DartifactId=springai-pdf-summary \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false
```

## `pom.xml` Configuration

Replace your `pom.xml` with the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.0</version>
  </parent>

  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>springai-pdf-summary</artifactId>
  <version>0.8.0</version>

  <properties>
    <java.version>21</java.version>
    <spring-ai.version>1.0.0</spring-ai.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-bom</artifactId>
        <version>${spring-ai.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-pdf-document-reader</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <scope>provided</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>21</source>
          <target>21</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

## Configuration (`application.yml`)

Create the file `src/main/resources/application.yml` with the following content:

```yaml
spring:
  ai:
    openai:                                     
      base-url: http://localhost:12434/engines  
      api-key: nokeyrequired                    
      init:
        pull-model-strategy: when_missing       
      chat:
        options:
          model: ai/mistral                     
```

## Business Logic

### `PdfSummarizationService.java`

```java
package org.alfresco.ai.summarize.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.document.Document;
import org.springframework.ai.reader.pdf.PagePdfDocumentReader;
import org.springframework.core.io.InputStreamResource;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class PdfSummarizationService {

    private final ChatModel chatModel;

    public String summarize(MultipartFile pdfFile) throws IOException {
        ChatClient chatClient = ChatClient.builder(chatModel).build();

        List<Document> pages = new PagePdfDocumentReader(
            new InputStreamResource(pdfFile.getInputStream())
        ).get();

        List<String> summaries = pages.stream()
            .map(Document::getText)
            .map(content -> chatClient.prompt()
                .user("Summarize this page: " + content)
                .call()
                .content())
            .collect(Collectors.toList());

        return chatClient.prompt()
            .user("Summarize the whole document: " + String.join("\n", summaries))
            .call()
            .content();
    }
}
```

### `SummarizationController.java`

```java
package org.alfresco.ai.summarize.rest;

import lombok.RequiredArgsConstructor;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import org.alfresco.ai.summarize.service.PdfSummarizationService;

import java.io.IOException;

@RestController
@RequestMapping("/api/summarize")
@RequiredArgsConstructor
public class SummarizationController {

    private final PdfSummarizationService service;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<String> summarizePdf(@RequestParam("file") MultipartFile file) throws IOException {
        return ResponseEntity.ok(service.summarize(file));
    }
}
```

### `App.java`

```java
package org.alfresco.ai.summarize;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

## Run the App

**Step 1:** Start Docker and make sure the Docker Model Runner is running and Mistral AI is available:

```bash
docker model status
Docker Model Runner is running

docker model list
MODEL NAME            PARAMETERS  QUANTIZATION    ARCHITECTURE  MODEL ID      SIZE
ai/mistral            7.25 B      IQ2_XXS/Q4_K_M  llama         395e9e2070c7  4.07 GiB
```

**Step 2:** Start your Spring Boot app:

```bash
mvn clean package 
java -jar target/springai-pdf-summary-0.8.0.jar
```

## Test the Endpoint

Use `curl`:

```bash
curl -F file=@sample.pdf http://localhost:8080/api/summarize
```

Or test it in Postman or HTTPie:

* Method: POST
* URL: `http://localhost:8080/api/summarize`
* Body > form-data > key: `file`, type: *File*, value: select your PDF

## Run the App with Containers

Create Docker assets with `docker init`

```sh
docker init

Welcome to the Docker Init CLI!

This utility will walk you through creating the following files with sensible defaults for your project:
  - .dockerignore
  - Dockerfile
  - compose.yaml
  - README.Docker.md

Let's get started!

? What application platform does your project use? Java
? What's the relative directory (with a leading .) for your app? ./src
? What version of Java do you want to use? 21
? What port does your server listen on? 8080

Create them using the Maven Wrapper plugin: mvn wrapper:wrapper.

What's next?
  Start your application by running → docker compose up --build
  Your application will be available at http://localhost:8080
```

Create missing Maven Wrapper resources:

```sh
mvn wrapper:wrapper
```

Start the application using Docker Compose:

```sh
docker compose up --build
...
dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed: org.springframework.web.client.ResourceAccessException: I/O error on POST request for "http://localhost:12434/engines/v1/chat/completions": null] with root cause
```

Update Docker Model Runner URL adding the value `http://host.docker.internal:12434/engines` as environment variable `SPRING_AI_OPENAI_BASE_URL` in `compose.yaml`

```yaml
services:
  server:
    build:
      context: .
    environment:
      SPRING_AI_OPENAI_BASE_URL: http://host.docker.internal:12434/engines
    ports:
      - 8080:8080
```