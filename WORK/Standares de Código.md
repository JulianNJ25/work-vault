
# 1. Development baseline

This should be the **foundation** of every Spring Boot service.

Define:
- Java version — **Java 21**
- Spring Boot version — **Spring Boot 4.x**
- Build system — **Maven 3.9.x**
- Dependency management strategy
- Repository structure
- IDE-independent formatting rules
- Naming conventions    
- Required build commands


For example:

> All backend services must use Java 21, Spring Boot 4.x and Maven.

Spring Boot already provides dependency management for a curated set of dependencies, and Spring recommends allowing Boot to manage those versions rather than independently specifying versions everywhere. ([Home](https://docs.spring.io/spring-boot/4.0/reference/using/build-systems.html?utm_source=chatgpt.com "Build Systems :: Spring Boot"))

This is important because you don't want developers doing:

```xml
<dependency>
    ...
    <version>random-version</version>
</dependency>
```

when Spring Boot's dependency management already provides the compatible version.

---

# 2. Project structure

This is one of the areas I would put **very early** in your standard.

For example, establish a canonical structure such as:

```text
project/
├── pom.xml
├── README.md
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com.company.project/
│   │   │       ├── ProjectApplication.java
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       ├── entity/
│   │   │       ├── dto/
│   │   │       ├── mapper/
│   │   │       ├── exception/
│   │   │       ├── config/
│   │   │       └── util/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── ...
│   └── test/
│       └── java/
└── ...
```

But there's an important architectural question you should resolve **before making this mandatory**:

### Package-by-layer vs package-by-feature

For example:

```text
controller/
service/
repository/
dto/
```

versus:

```text
printer/
    PrinterController
    PrinterService
    PrinterRepository
    PrinterDto

printing/
    PrintingController
    PrintingService
    PrintingRepository
```

For a company standard, this decision is much more important than it initially appears.

You should explicitly decide which one your organization wants and **why**.

---

# 3. Dependency policy

You already identified this, but I'd make it a substantial section.

Define:

### Allowed

For example:

- Spring Boot starters
- Spring Data
- Spring Security
- Jackson
- Validation
- database drivers
- approved logging libraries
- approved testing libraries

### Restricted

Dependencies that require architectural/security approval.

### Prohibited

For example:

- obsolete libraries
- libraries with unacceptable licensing
- libraries that duplicate functionality already provided by Spring/JDK
- dependencies with known unacceptable vulnerabilities
- arbitrary libraries added without justification
And importantly:
### Dependency version policy

Define:

> Developers must not manually specify versions for dependencies managed by Spring Boot unless there is an approved justification.

Spring Boot explicitly provides a curated dependency list for this purpose. ([Home](https://docs.spring.io/spring-boot/4.0/reference/using/build-systems.html?utm_source=chatgpt.com "Build Systems :: Spring Boot"))

You could eventually maintain an **approved dependency catalog** internally.

---

# 4. Java language standards

You should define how Java itself is supposed to be written.
### Naming

- classes → `PascalCase`
- methods → `camelCase`
- constants → `UPPER_SNAKE_CASE`
- packages → lowercase
- variables → `camelCase`

### Types

Define when to use:
- `record`
- `class`
- `enum`
- interface
- sealed classes, if applicable

### Collections

For example:

- prefer interfaces for declarations:
    

```java
List<String> names;
```

rather than:

```java
ArrayList<String> names;
```

### Date/time

Define your organization's standard:

```java
Instant
OffsetDateTime
LocalDate
LocalDateTime
```

and when each should be used.

This becomes particularly important in systems where timestamps have regulatory significance.

---

# 5. Models / Entities / DTOs

I would actually split your "models" category.

These are not necessarily the same thing.

Define standards separately for:

### JPA entities

```java
@Entity
public class Printer {
    ...
}
```

Rules for:

- IDs
- generated IDs
- relationships
- lazy/eager loading
- cascade
- `equals()` / `hashCode()`
- entity mutability
- database naming    
- column constraints


### Request DTOs

```java
public record CreatePrinterRequest(
    @NotBlank
    String name
) {}
```

### Response DTOs

```java
public record PrinterResponse(
    Long id,
    String name
) {}
```

### Domain objects

If you use them.

This distinction is important because you don't want developers accidentally exposing JPA entities directly through REST APIs.

---

# 6. Records

You specifically mentioned records, and I'd definitely standardize them.

But don't simply say:

> "Use records."

Define **when records are appropriate**.

For example:

**Good candidates**

- REST request DTOs
    
- REST response DTOs
    
- immutable value objects
    
- configuration structures
    
- event payloads
    

**Generally inappropriate**

- JPA entities
    
- objects requiring significant mutable state
    
- objects whose identity/lifecycle depends on mutation
    

Also define validation:

```java
public record CreatePrinterRequest(
    @NotBlank
    @Size(max = 100)
    String name
) {}
```

---

# 7. Controllers

This should be very explicit.

Define:

- URL conventions
    
- HTTP method conventions
    
- status codes
    
- request/response DTOs
    
- validation
    
- pagination
    
- query parameters
    
- path parameters
    
- headers
    
- content types
    
- API versioning
    
- naming
    
- controller responsibilities
    

Most importantly:

> **Controllers should be thin.**

For example:

```text
Controller
    ↓
Service
    ↓
Repository
```

The controller should not contain business logic.

---

# 8. Services

Define:

- What belongs in a service
- Transaction boundaries
- Business logic
- Interaction with repositories
- Interaction with external services
- Whether services can call other services
- Service naming
- Service interface policy
I'd explicitly answer a controversial question:
> **Do we require an interface for every service?**
Many organizations blindly create:

```text
PrinterService
PrinterServiceImpl
```

even when there is only one implementation and no actual abstraction.

You should make this a deliberate architectural decision rather than an automatic pattern.

---

# 9. Repositories / persistence

Define:
- Spring Data repository conventions
- Query method conventions
- JPQL/native SQL rules
- transaction handling
- entity relationships
- pagination
- database migrations
- database schema changes
- optimistic locking
- indexes
- N+1 prevention
- lazy loading
- connection configuration
And especially:

### Database migration standard

Decide whether you use:
- Flyway
- Liquibase
- something else

and make the process mandatory.

For a regulated environment, database changes should not simply be:

> "Developer manually changed production database."

You want reproducible, reviewable migrations.

---

# 10. Validation

Another major missing category.

Define **where validation occurs**.

For example:

```text
HTTP request
      ↓
Bean Validation
      ↓
Controller
      ↓
Service/business validation
      ↓
Repository
```

Define:

- `@NotNull`
    
- `@NotBlank`
    
- `@Size`
    
- `@Pattern`
    
- `@Valid`
    
- custom validators
    
- business validation
    

And distinguish:

**Structural validation**

> "This field cannot be empty."

from:

**Business validation**

> "A printer cannot be connected to two printing sessions."

That distinction will make your architecture much cleaner.

### Null handling

Define whether developers should:
- use `Optional`
- use `@Nullable` / `@NonNull`
- allow nulls
- validate at boundaries

---

# 11. Exception handling

You already have this, and I'd keep it as its own standard.

Include:

- custom exception naming
    
- exception hierarchy
    
- when to create custom exceptions
    
- HTTP mapping
    
- error response structure
    
- validation errors
    
- unexpected errors
    
- logging of exceptions
    
- sensitive information
    
- stack traces
    
- correlation IDs
    

You have already been working on the `ErrorResponseDto` / `@RestControllerAdvice` approach, so this can become one of your first concrete standards.

---

# 12. Logging

Also good that you identified this.

But go beyond:

> "Use SLF4J."

Define:

### Log levels

When to use:

- `TRACE`
    
- `DEBUG`
    
- `INFO`
    
- `WARN`
    
- `ERROR`
    

### Log structure

Prefer structured information such as:

```text
timestamp
level
service
traceId
requestId
operation
message
```

### What must never be logged

This is particularly relevant to ISO 27001:

- passwords
    
- authentication tokens
    
- API keys
    
- sensitive personal information
    
- sensitive medical information
    
- database credentials
    

### Logging responsibility

For example:

> Controllers should not log every request manually if centralized HTTP request logging already handles it.

---

# 13. Configuration management

**This is another important omission.**

Define:

- `application.yml`
    
- profiles
    
- environment variables
    
- secrets
    
- configuration properties
    
- default values
    
- configuration validation
    
- external configuration
    
- production configuration
    

Especially:

> **No secrets in source code.**

Bad:

```yaml
database:
  password: MySecretPassword
```

And decide how secrets are provided in your environment.

---

# 14. Security

Given ISO/IEC 27001, I would make this a **first-class category**, not an afterthought.

ISO/IEC 27001 is specifically concerned with managing information-security risks and protecting confidentiality, integrity and availability. ([ISO](https://www.iso.org/standard/27001?s=1234&utm_source=chatgpt.com "ISO/IEC 27001:2022 - Information security management systems"))

Your Spring standard should eventually cover:

- authentication
    
- authorization
    
- Spring Security
    
- password handling
    
- JWT/session handling
    
- secrets
    
- HTTPS
    
- CORS
    
- CSRF
    
- input validation
    
- injection prevention
    
- SQL injection
    
- SSRF
    
- path traversal
    
- insecure deserialization
    
- sensitive data exposure
    
- dependency vulnerabilities
    
- security headers
    
- rate limiting
    
- audit logging
    

You don't necessarily need all of this in **version 1** of your coding standard, but it absolutely belongs in the overall framework.

---

# 15. REST API standards

This deserves its own section rather than being buried inside controllers.

Define:

```text
GET
POST
PUT
PATCH
DELETE
```

and their semantics.

Also:

- URL naming
    
- pluralization
    
- HTTP status codes
    
- pagination
    
- filtering
    
- sorting
    
- error responses
    
- versioning
    
- idempotency
    
- content negotiation
    
- API documentation
    

For example:

```text
GET    /printers
GET    /printers/{id}
POST   /printers
PUT    /printers/{id}
DELETE /printers/{id}
```

And define what `200`, `201`, `204`, `400`, `401`, `403`, `404`, `409`, `422`, `500`, etc. mean **within your organization**.

---

# 16. Testing

This is probably the **biggest omission** from your original list.

You are defining code quality. You cannot define code quality without defining testing standards.

At minimum:

### Unit tests

- service tests
    
- utility tests
    
- validators
    
- business logic
    

### Integration tests

- database
    
- repositories
    
- Spring context
    

### API tests

- controller tests
    
- request validation
    
- error responses
    
- authentication/authorization
    

### Test naming

For example:

```text
shouldReturnPrinterWhenPrinterExists()
shouldThrowExceptionWhenPrinterDoesNotExist()
```

### Test coverage

Be careful here.

I would **not** simply establish:

> "All projects must have 90% coverage."

Coverage is useful, but coverage percentage alone is a poor definition of quality.

Instead define minimum expectations around **what must be tested**.

---

# 17. API documentation

You should standardize:

- OpenAPI
    
- Swagger UI
    
- endpoint documentation
    
- request examples
    
- response examples
    
- error responses
    
- authentication requirements
    

Especially because you are building web services.

You want every service to expose a predictable API contract.

---

# 18. Code quality automation

This is where your standards become enforceable rather than merely documentation.

I'd strongly recommend defining tools such as:

```text
Formatting
    ↓
Static analysis
    ↓
Unit tests
    ↓
Integration tests
    ↓
Dependency vulnerability scan
    ↓
Build
```

Possible tooling:

- Checkstyle
    
- SpotBugs
    
- PMD
    
- JaCoCo
    
- OWASP dependency scanning
    
- SonarQube
    
- Maven Enforcer
    

You don't necessarily need all of them.

The important principle is:

> **If a rule can be automatically checked, don't rely exclusively on developers remembering it.**

For example, instead of documenting:

> "Unused dependencies are prohibited."

have CI detect them.

---

# 19. Git and code review

This is another major omission.

Define:

- branch naming
    
- commit conventions
    
- pull/merge requests
    
- mandatory reviewers
    
- minimum approvals
    
- protected branches
    
- merge strategy
    
- review checklist
    
- prohibited practices
    
- emergency changes
    

For example:

```text
Developer
   ↓
Feature branch
   ↓
Pull Request
   ↓
Automated checks
   ↓
Code review
   ↓
Approval
   ↓
Merge
```

This is extremely important in a quality-management/security environment because it creates evidence that changes were reviewed rather than simply pushed directly into production.

---

# 20. CI/CD requirements

Closely related to the previous section.

Define what must happen before code can be merged/deployed.

For example:

```text
git push
   ↓
Compile
   ↓
Static analysis
   ↓
Unit tests
   ↓
Integration tests
   ↓
Dependency/security scan
   ↓
Package
   ↓
Deploy
```

You can then make these requirements enforceable through your Forgejo Actions infrastructure.

---

# 21. Observability

I'd include:

- health endpoints
    
- metrics
    
- structured logging
    
- tracing
    
- correlation IDs
    
- application startup information
    
- readiness/liveness
    
- error monitoring
    

Spring Boot itself provides production-oriented features such as health checks, metrics and externalized configuration, so this fits naturally into a Spring Boot standard. ([Home](https://docs.spring.io/spring-boot/index.html?utm_source=chatgpt.com "Spring Boot :: Spring Boot"))

---

# 22. Documentation

Define what every service must contain.

For example:

```text
README.md
API documentation
Architecture overview
Configuration documentation
Deployment documentation
Database migration information
Known limitations
```

And perhaps:

```text
docs/
├── architecture/
├── api/
├── deployment/
└── operations/
```

---

# 23. Dependency and vulnerability lifecycle

This is worth separating from basic dependency management.

Define:

- vulnerability scanning
    
- CVE severity thresholds
    
- dependency update frequency
    
- unsupported dependencies
    
- end-of-life technologies
    
- emergency vulnerability remediation
    
- exception process
    

For example:

> A dependency with a critical vulnerability may not be deployed unless a documented risk acceptance exists.

This connects your technical standards much more directly to your eventual security-management requirements.

---

# 24. Change management

This becomes particularly important for the ISO side.

You should eventually define:

- how standards change
    
- how production code changes
    
- who approves architectural exceptions
    
- how exceptions to coding standards are documented
    
- how old services are brought into compliance
    
- how technical debt is tracked
    

This leads to something extremely valuable:

### Standard → Enforcement → Evidence

For example:

|Standard|Enforcement|Evidence|
|---|---|---|
|Java 21|Maven/CI|Build logs|
|Formatting|Formatter plugin|CI result|
|Unit tests|Maven|Test reports|
|Vulnerability scanning|Scanner|Scan report|
|Code review|Forgejo|PR history|
|API contract|OpenAPI|Repository|
|Database changes|Flyway|Migration history|

That third column is especially important for an organization preparing for audits.

---

# The structure I would recommend

If I were organizing this initiative from scratch, I wouldn't make one gigantic document called **"Spring Boot Coding Standards."**

I'd create a hierarchy.

### Level 1 — Development baseline

```text
1. Technology stack
2. Project structure
3. Java conventions
4. Maven conventions
5. Dependency management
6. Naming conventions
7. Formatting
```

### Level 2 — Application architecture

```text
8. Package structure
9. Controllers
10. DTOs
11. Records
12. Services
13. Repositories
14. Entities
15. Mappers
16. Utilities
17. Configuration
18. Exceptions
```

### Level 3 — Application behavior

```text
19. Validation
20. REST API conventions
21. Error handling
22. Logging
23. Transactions
24. Database access
25. External service communication
```

### Level 4 — Quality

```text
26. Unit testing
27. Integration testing
28. API testing
29. Static analysis
30. Dependency scanning
31. Code coverage
32. Code review
```

### Level 5 — Security

```text
33. Authentication
34. Authorization
35. Secrets
36. Sensitive data
37. Secure coding
38. Security logging
39. Vulnerability management
```

### Level 6 — Operations

```text
40. Configuration
41. Health checks
42. Metrics
43. Tracing
44. Deployment
45. CI/CD
46. Monitoring
```

### Level 7 — Governance

```text
47. Change management
48. Exceptions to standards
49. Technical debt
50. Documentation
51. Versioning of the standards
52. Compliance evidence
```

---

## One thing I would change about your current approach

You currently have:

> "How models should be written, how records should be written, how controllers should be written..."

That's good, but I'd avoid making the document merely a collection of **"how to write X" rules**.

Instead, for every component, define four things:

### 1. Purpose

**What is this component for?**

### 2. Responsibilities

**What is it allowed to do?**

### 3. Restrictions

**What must it NOT do?**

### 4. Interaction

**What components is it allowed to communicate with?**

For example:

```text
Controller

Purpose:
    Handle HTTP requests.

Responsibilities:
    - Receive requests
    - Validate input
    - Invoke application/service layer
    - Return HTTP response

Must NOT:
    - Contain business logic
    - Access repositories directly
    - Execute SQL
    - Construct database entities unnecessarily

Interactions:
    Controller → Service
    Controller → DTO
```

Then:

```text
Service

Purpose:
    Implement business/application logic.

Responsibilities:
    - Business rules
    - Transaction boundaries
    - Coordinate repositories/external services

Must NOT:
    - Handle HTTP-specific concerns
    - Return ResponseEntity
    - Depend on controllers
```

That creates an actual **architecture**, rather than a style guide.

---

# And one particularly important point for ISO

Don't try to make your Spring Boot coding standard "ISO 27001 compliant."

That's the wrong abstraction.

Think of it as:

```text
                 Company QMS / ISMS
                        │
          ┌─────────────┴─────────────┐
          │                           │
      ISO 27001                    ISO 15189
          │                           │
          └─────────────┬─────────────┘
                        │
                Software governance
                        │
              Development standards
                        │
             Spring Boot standard
                        │
        ┌───────────────┼───────────────┐
        │               │               │
     Coding          Testing         Security
     standards       standards       standards
```

ISO/IEC 27001:2022 is fundamentally an **information security management system** standard, while ISO 15189:2022 establishes requirements for the quality and competence of medical laboratories. ([ISO](https://www.iso.org/standard/27001?s=1234&utm_source=chatgpt.com "ISO/IEC 27001:2022 - Information security management systems"))

So your Spring Boot standard should be **one controlled engineering standard inside the company's larger quality/security system**.

And that actually gives you a much stronger long-term strategy: when an auditor asks _"How do you ensure software is developed consistently and securely?"_, you can point not merely to a document, but to **the standard + automated enforcement + code reviews + CI evidence + change history**.

For your particular situation, I would therefore **start with the Level 1 + Level 2 material you proposed**, get those rules solid, and only then move into security/compliance-specific controls. Spring Boot 4 itself already has a well-defined baseline around Maven and dependency management that you can use as the technical foundation. ([Home](https://docs.spring.io/spring-boot/4.0/reference/using/build-systems.html?utm_source=chatgpt.com "Build Systems :: Spring Boot"))

[Spring Boot 4.0 Reference Documentation](https://docs.spring.io/spring-boot/4.0/reference/index.html?utm_source=chatgpt.com)  
[ISO/IEC 27001:2022 official page](https://www.iso.org/standard/27001?s=1234&utm_source=chatgpt.com)  
[ISO 15189:2022 official page](https://www.iso.org/standard/76677.html?utm_source=chatgpt.com)


# Dependencies

for `@OneToOne`, `@OneToMany`, `@ManyToOne`, and `@ManyToMany` annotations
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

