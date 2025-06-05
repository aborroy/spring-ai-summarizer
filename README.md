# AI Summarizer

[![Java 21](https://img.shields.io/badge/java-21-blue.svg)](https://adoptium.net/en-GB/temurin/releases/)
[![Spring Boot](https://img.shields.io/badge/spring--boot-3.5.0-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Docker Model Runner](https://img.shields.io/badge/Docker-Model--Runner-blue.svg)](https://docs.docker.com/ai/model-runner/)
[![License](https://img.shields.io/badge/license-Apache--2.0-green.svg)](LICENSE)

> Summarize PDF documents using a local LLM with Spring Boot + Docker Model Runner


## Overview

This project is a **tutorial** to build lightweight Spring Boot application that transforms PDF documents into concise summaries using a **local Large Language Model**. It's built on top of [Spring AI](https://docs.spring.io/spring-ai/) and [Docker Model Runner](https://docs.docker.com/ai/model-runner/), making it easy to run offline with full control over your models.

This project is a simple Spring Boot application that:

* Accepts PDF files via a REST API
* Splits the PDF by page
* Uses a local Large Language Model (via [Docker Model Runner](https://docs.docker.com/ai/model-runner/)) to summarize each page
* Produces a final global summary of the entire document

---

## 📚 Table of Contents

1. [Requirements](#requirements)  
2. [Project Setup](#project-setup)  
3. [pom.xml Configuration](#pomxml-configuration)  
4. [Application Configuration](#configuration-applicationyml)  
5. [Business Logic](#business-logic)  
6. [Run the App](#run-the-app)  
7. [Test the Endpoint](#test-the-endpoint)  
8. [Run the App with Containers](#run-the-app-with-containers)  
9. [License](#license)

---

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

  <!-- Inherit configuration from the Spring Boot parent POM, which simplifies plugin and dependency management -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.0</version>
  </parent>

  <!-- Maven model version -->
  <modelVersion>4.0.0</modelVersion>

  <!-- Project coordinates (group ID, artifact ID, and version) -->
  <groupId>com.example</groupId>
  <artifactId>springai-pdf-summary</artifactId>
  <version>0.8.0</version>

  <!-- Define reusable properties, including Java version and Spring AI BOM version -->
  <properties>
    <java.version>21</java.version>
    <spring-ai.version>1.0.0</spring-ai.version>
  </properties>

  <!-- Import Spring AI BOM (Bill Of Materials) to manage dependency versions centrally -->
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

  <!-- Declare project dependencies -->
  <dependencies>
    <!-- Spring Boot starter for building RESTful web applications -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring AI integration for OpenAI-compatible models (also works with Docker Model Runner) -->
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>

    <!-- Document reader that extracts content from PDF pages using Apache PDFBox -->
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-pdf-document-reader</artifactId>
    </dependency>

    <!-- Lombok reduces boilerplate code by generating getters/setters/constructors at compile-time -->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <scope>provided</scope> <!-- Not included in the final JAR, only needed at compile time -->
    </dependency>
  </dependencies>

  <!-- Build configuration with required plugins -->
  <build>
    <plugins>
      <!-- Spring Boot plugin to enable executable JAR creation and dependency resolution -->
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>

      <!-- Compiler plugin to set Java language level explicitly -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>21</source> <!-- Compile using Java 21 -->
          <target>21</target> <!-- Generate bytecode compatible with Java 21 -->
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
    openai:                                      # Configure the OpenAI-compatible model provider for Docker Model Runner
      base-url: http://localhost:12434/engines   # Base URL for the model server (e.g., Docker Model Runner exposes "/engines" endpoint)
      api-key: nokeyrequired                     # Dummy API key; Docker Model Runner does not require authentication

      init:
        pull-model-strategy: when_missing        # Automatically pull the model if it's not already available on the server

      chat:
        options:
          model: ai/mistral                      # Name of the model to use for chat (as exposed by the server)            
```

* It's configuring Spring AI to use Docker Model Runner running locally, pointing to a model named `ai/mistral`
* The model is pulled only if missing (`when_missing`) which avoids re-pulling each time
* No API key is needed for local use

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

@Service // Makes this class injectable in other Spring components (e.g., controllers)
@RequiredArgsConstructor // Lombok: generates a constructor with all final fields (chatModel)
public class PdfSummarizationService {

    // LLM model interface injected by Spring (configured in application.yml)
    private final ChatModel chatModel;

    /**
     * Summarizes a PDF file by:
     * 1. Splitting it into pages
     * 2. Summarizing each page separately
     * 3. Producing a global summary from all page summaries
     *
     * @param pdfFile the uploaded PDF to summarize
     * @return a final summary of the entire document
     * @throws IOException if the file can't be read
     */
    public String summarize(MultipartFile pdfFile) throws IOException {
        // Initialize the chat client using the configured model
        ChatClient chatClient = ChatClient.builder(chatModel).build();

        // Extract pages from the PDF using Spring AI's PDF reader
        List<Document> pages = new PagePdfDocumentReader(
            new InputStreamResource(pdfFile.getInputStream()) // Wrap file input as a Spring resource
        ).get(); // Returns a list of Document objects (one per page)

        // Summarize each page using the LLM, producing one summary per page
        List<String> summaries = pages.stream()
            .map(Document::getText) // Get plain text of each page
            .map(content -> chatClient.prompt() // Create a prompt for each page
                .user("Summarize this page: " + content) // Instruction to the model
                .call() // Call the LLM
                .content()) // Extract the result text
            .collect(Collectors.toList());

        // Ask the LLM to summarize the full document using the previous page summaries
        return chatClient.prompt()
            .user("Summarize the whole document: " + String.join("\n", summaries))
            .call()
            .content(); // Return the global summary
    }
}
```

* Uses Spring AI’s PDF reader to break a document into pages.
* Sends each page to the LLM to get a per-page summary.
* Then creates a final global summary by feeding all page summaries back to the LLM.

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

@RestController // Marks this class as a REST controller (JSON responses by default)
@RequestMapping("/api/summarize") // Base URL for all endpoints in this controller
@RequiredArgsConstructor // Lombok: generates a constructor for 'service' (final field)
public class SummarizationController {

    // Injected service that performs the actual summarization logic
    private final PdfSummarizationService service;

    /**
     * HTTP POST endpoint that receives a PDF file and returns its summary.
     *
     * @param file the uploaded PDF file (multipart/form-data)
     * @return a plain text summary of the PDF, generated via LLM
     * @throws IOException if reading the file fails
     */
    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE) // Accepts file uploads
    public ResponseEntity<String> summarizePdf(@RequestParam("file") MultipartFile file) throws IOException {
        // Delegate to the service and return the result in a 200 OK response
        return ResponseEntity.ok(service.summarize(file));
    }
}

```

* Defines a single POST `/api/summarize` endpoint.
* Accepts a file upload using `multipart/form-data`.
* Delegates the actual PDF processing and summarization to `PdfSummarizationService`.
* Returns the resulting summary as a plain String in the HTTP response body.

### `App.java`

```java
package org.alfresco.ai.summarize;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // Marks this class as the main configuration and bootstrap class
public class App {

    /**
     * Main method — entry point for the Spring Boot application.
     *
     * @param args command-line arguments passed to the app (if any)
     */
    public static void main(String[] args) {
        // Bootstraps the Spring Boot application, starting the embedded server and scanning components
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

## License

This project is licensed under the **Apache License 2.0**.
See the [LICENSE](LICENSE) file for details.