# Lab: Building a Local PDF Summarizer with Spring Boot & Docker-Hosted LLM

Welcome to this hands-on lab! You'll create a **Spring Boot 3.5** microservice that transforms any PDF into a concise summary using a **local Large Language Model** (LLM) served through Docker Model Runner. By the end, you'll have a running REST endpoint (`POST /api/summarize`) that accepts a PDF and returns a summary – **without ever sending data to the cloud**.

## Learning Objectives

1. Scaffold a modern Spring Boot project with proper dependency management
2. Configure Spring AI to communicate with a locally running LLM (Mistral) via Docker Model Runner
3. Implement PDF processing with page-by-page text extraction using Spring AI utilities
4. Build a multi-stage summarization pipeline with prompt chaining
5. Expose functionality through a robust REST API with proper error handling
6. Package the complete solution with Docker Compose for easy deployment

## Prerequisites & Environment Setup

| Tool           | Minimum Version | Check Command        | Installation Notes |
| -------------- | --------------- | -------------------- | ------------------ |
| Java (Temurin) | 21              | `java --version`     | Use SDKMAN or official installer |
| Maven          | 3.9             | `mvn --version`      | Bundled with most IDEs |
| Docker Desktop | 4.40            | `docker --version`   | Required for Model Runner |
| IDE            | Latest          | N/A                  | IntelliJ IDEA recommended |

**Verification Task:** Run all check commands. If any fail, install the missing tools before proceeding.

## Step 0: Download and Verify LLM Model

We'll use **Mistral 7B** - it's lightweight (~4GB) yet powerful enough for quality summarization.

**Your Task:**
1. Use `docker model pull` to download the `ai/mistral` model
2. Verify the download with `docker model list`
3. Check if Docker Model Runner is active with `docker model status`

**Common Issues:**
- Command not recognized? Your Docker Desktop version is too old
- Download hanging? Check your internet connection and Docker daemon

**Success Indicator:** You should see "Docker Model Running" and the mistral model listed.

## Step 1: Create Project Structure

**Your Task:**
1. Choose meaningful names:
   - Group ID: Use your domain (e.g., `com.yourname.pdfsummarizer`)
   - Artifact ID: Something descriptive (e.g., `pdf-summarizer-service`)

2. Generate a Maven project using the `maven-archetype-quickstart` archetype

```bash
mvn archetype:generate                                   \
  -DgroupId=com.yourname.pdfsummarizer                   \
  -DartifactId=pdf-summarizer-service                    \
  -DarchetypeGroupId=org.apache.maven.archetypes         \
  -DarchetypeArtifactId=maven-archetype-quickstart       \
  -DarchetypeVersion=1.4 -DinteractiveMode=false
```

3. Open the project in your IDE and run an initial `mvn clean compile` to verify setup

## Step 2: Configure Dependencies (pom.xml)

**Your Task:** Transform the basic Maven project into a Spring Boot application by modifying `pom.xml`:

### Required Changes:
1. **Parent Configuration:** Set Spring Boot as the parent with version 3.5.0
2. **Properties Block:** Define Java version (21) and Spring AI version (1.0.0)
3. **Dependency Management:** Import the Spring AI BOM to manage versions
4. **Core Dependencies:** Add these three essential starters:
   - Web starter (for REST endpoints)
   - Spring AI OpenAI starter (for LLM communication)
   - PDF document reader (for PDF processing)
5. **Optional Enhancement:** Add Lombok for cleaner code

```xml
<!-- parent -->
<parent>...Spring Boot 3.5.0...</parent>

<!-- properties -->
<java.version>21</java.version>
<spring-ai.version>1.0.0</spring-ai.version>

<!-- dependencyManagement -->
<dependencyManagement>...spring-ai-bom...</dependencyManagement>

<!-- starters you’ll need -->
<dependency> spring-boot-starter-web </dependency>
<dependency> spring-ai-openai-spring-boot-starter </dependency>
<dependency> spring-ai-pdf-document-reader </dependency>
<!-- optional -->
<dependency scope="provided"> lombok </dependency>
```

**Verification:** `mvn clean package` should complete without errors.

## Step 3: Application Configuration

**Your Task:** Create `src/main/resources/application.yml` with proper Spring AI configuration.

### Configuration Requirements:
1. **Server Settings:** Configure the application port
2. **Multipart Configuration:** Set appropriate file size limits for PDF uploads
3. **Spring AI OpenAI Settings:** Configure four key properties:
   - `base-url`: Where is Docker Model Runner listening? (Hint: localhost with a specific port and path)
   - `api-key`: What dummy value works for local usage?
   - `model`: Which model did you download in Step 0?
   - `temperature`: What value gives consistent summaries? (0.0-1.0 range)

```yaml
server:
  port: 8080

spring:
  servlet:
    multipart:
      max-file-size: 20MB    
  ai:
    openai:
      base-url:  http://localhost:12434/
      api-key:   dummy
      model:     ai/istral
      temperature: 0.0
```   

**Research Required:**
- What port does Docker Model Runner use by default?
- What's the correct API endpoint path for OpenAI-compatible APIs?
- How does temperature affect LLM output consistency?

## Step 4: Core Business Logic - Service Layer

**Your Task:** Create a `PdfSummarizationService` class that orchestrates the summarization process.

### Class Structure:

```java
@Service
@RequiredArgsConstructor
public class PdfSummarizationService {

    private final ChatModel chatModel;

    public String summarize(MultipartFile pdf) throws IOException {
        // TODO 1: validate pdf (size, emptiness)
        // TODO 2: use PagePdfDocumentReader to pull List<Document>
        // TODO 3: loop over pages → ask LLM for 2-3 sentence summary each
        //         Hint: new ChatClient(chatModel).call("Prompt" + pageContent)
        // TODO 4: combine page summaries → ask LLM again for global summary
        // Return final summary
        return null; // placeholder
    }
}
```

### Implementation Strategy:
1. **Input Validation:** Check if the file is valid and not empty
2. **PDF Processing:** Use `PagePdfDocumentReader` to extract text from each page
3. **Page Summarization:** Create individual summaries for each page using the LLM
4. **Global Summarization:** Combine page summaries into a comprehensive document summary

### Key Classes to Research:
- `ChatClient`: How do you build one from a `ChatModel`?
- `PagePdfDocumentReader`: What constructor parameters does it need?
- `Document`: How do you extract text content?
- `InputStreamResource`: How do you create one from a `MultipartFile`?

**Design Questions:**
- Why summarize page-by-page instead of the entire document at once?
- What happens if a PDF has no readable text?
- How should you handle very large PDFs?

**Prompt Engineering Tips:**
- Be specific about desired summary length
- Ask for key points and main ideas

## Step 5: REST API Layer - Controller

**Your Task:** Create a REST controller that exposes your summarization service.

### Class Structure:

```java
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class SummarizationController {

    private final PdfSummarizationService service;

    @PostMapping("/summarize")
    public ResponseEntity<String> summarize(@RequestParam("file") MultipartFile file) {
        // TODO: call service
        return null;
    }
}
```

### Controller Requirements:
1. **Annotations:** Use appropriate Spring annotations for REST endpoints
2. **Endpoint Mapping:** Map to `/api/summarize` with POST method
3. **File Handling:** Accept multipart file uploads
4. **Error Handling:** Return appropriate HTTP status codes
5. **Response Format:** Return plain text summaries

### Implementation Considerations:
- What `@RequestMapping` configuration do you need?
- How do you handle `MultipartFile` parameters?
- What should happen when summarization fails?
- Should you return `ResponseEntity` or plain `String`?

## Step 6: Application Bootstrap

**Your Task:** Replace the default `App.java` with a proper Spring Boot application class.

### Class Structure:

```java
@SpringBootApplication
public class PdfSummarizerApplication {
    public static void main(String[] args) { SpringApplication.run(PdfSummarizerApplication.class, args); }
}
```

*(This one is boilerplate; left intact so the app actually starts.)*

## Step 7: Local Testing & Debugging

**Your Task:** Get everything running and test the complete workflow.

### Testing Checklist:
1. **Verify Model Runner:** Confirm Docker Model Runner is active
2. **Start Application:** Use Maven to run your Spring Boot app
3. **Test Endpoint:** Use curl or Postman to send a PDF file

### Sample Test Command Structure:

```bash
# You'll need to construct the proper curl command
curl -F file=@your-test.pdf http://localhost:YOUR_PORT/YOUR_ENDPOINT
```

**Troubleshooting Guide:**
- `ResourceAccessException`: Check Docker Model Runner status and base-url configuration
- `404 Not Found`: Verify your controller mappings and component scanning
- `400 Bad Request`: Check file upload configuration and request format
- Empty response: Examine your service logic and LLM prompts

**Testing Tips:**
- Start with a small, simple PDF
- Check application logs for detailed error information
- Verify each step independently (PDF reading, LLM calls, etc.)

## Step 8: Containerization with Docker Compose

**Your Task:** Package your application for easy deployment.

### Docker Setup:
1. **Initialize Docker:** Use `docker init` to generate Docker configuration
2. **Configure Environment:** Set environment variables for container-to-host communication
3. **Network Configuration:** Ensure the containerized app can reach Docker Model Runner on the host

### Key Considerations:
- What base URL should the containerized app use to reach the host?
- How do you override Spring configuration with environment variables?
- What port mapping do you need in Docker Compose?

## Success Criteria

**Functional Requirements:**
- Application starts without errors
- PDF files can be uploaded and processed
- Summaries are generated and returned
- Local LLM integration works correctly

**Technical Requirements:**
- Proper Spring Boot project structure
- Clean separation of concerns (Controller → Service → AI)
- Appropriate error handling
- Containerized deployment option

**Learning Outcomes:**
- Understanding of Spring AI framework
- Experience with local LLM deployment
- REST API development skills
- Docker containerization knowledge

> Remember: The goal is to understand each component and how they work together. Don't hesitate to experiment, break things, and fix them – that's how real learning happens!