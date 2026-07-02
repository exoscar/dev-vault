# DevSync Phase 4 Learning Checklist

> Goal: Complete the remaining Phase 4 modules:
>
> - Email Notifications
> - Dockerization
> - Integration Testing

---

# Phase 4.3 — Email Notifications

## Objective

Build a production-style email notification system powered by Spring Events.

Example flow:

```text
Issue Assigned
    ↓
IssueAssignedEvent
    ↓
Event Listener
    ↓
Email Service
    ↓
Email Sent
```

---

## Spring Mail

### Learn

- [x] spring-boot-starter-mail
- [x] JavaMailSender
- [x] SimpleMailMessage
- [x] MimeMessage
- [x] MimeMessageHelper

### Understand

```java
javaMailSender.send(...);
```

### Outcome

- [ ] Send a plain text email from Spring Boot

---

## SMTP Fundamentals

### Learn

- [x] What SMTP is
- [x] SMTP Host
- [x] SMTP Port
- [x] Authentication
- [x] App Passwords

### Providers

- [x] Mailtrap (Recommended)
- [ ] Gmail SMTP
- [ ] SendGrid (Awareness)

### Outcome

- [ ] Configure Spring Boot to send emails through Mailtrap

---

## HTML Emails

### Learn

- [ ] MIME Emails
- [ ] HTML Email Body
- [ ] Email Formatting

Example:

```html
<h2>Issue Assigned</h2>
<p>You have been assigned a new issue.</p>
```

### Outcome

- [ ] Send a styled HTML email

---

## Async Processing

### Learn

- [ ] @EnableAsync
- [ ] @Async
- [ ] Background Execution
- [ ] Task Executors

### Why

Avoid blocking API requests while sending emails.

```text
Issue Assigned
    ↓
API Response
    ↓
Email Sent Asynchronously
```

### Outcome

- [ ] Send emails in the background

---

## Event-Driven Email Notifications

### Reuse Existing Events

- [ ] IssueAssignedEvent
- [ ] IssueStatusChangedEvent
- [ ] IssuePriorityChangedEvent

### Implement

```text
Event
   ↓
Listener
   ↓
Email Service
   ↓
Email
```

### Outcome

- [ ] Issue Assignment Email
- [ ] Status Change Email
- [ ] Priority Change Email

---

# Phase 4.4 — Dockerization

## Objective

Run the complete DevSync application inside containers.

---

## Docker Fundamentals

### Learn

- [ ] Images
- [ ] Containers
- [ ] Volumes
- [ ] Networks
- [ ] Registries

### Understand

```text
Image -> Blueprint
Container -> Running Instance
```

### Outcome

- [ ] Understand Docker lifecycle

---

## Dockerfile

### Learn

- [ ] FROM
- [ ] WORKDIR
- [ ] COPY
- [ ] RUN
- [ ] EXPOSE
- [ ] ENTRYPOINT

Example Flow:

```dockerfile
FROM eclipse-temurin:21-jdk
COPY app.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

### Outcome

- [ ] Create Docker image for DevSync

---

## Containerizing Spring Boot

### Learn

- [ ] Maven Build
- [ ] JAR Packaging
- [ ] Docker Build
- [ ] Docker Run

Commands:

```bash
mvn clean package
docker build -t devsync .
docker run devsync
```

### Outcome

- [ ] Run DevSync from Docker

---

## Docker Compose

### Learn

- [ ] services
- [ ] ports
- [ ] environment
- [ ] depends_on
- [ ] volumes

Architecture:

```text
Spring Boot Container
          ↓
PostgreSQL Container
```

### Outcome

- [ ] Start application + database using one command

```bash
docker compose up
```

---

## Environment Variables

### Learn

Replace hardcoded values:

```properties
spring.datasource.url
spring.datasource.username
spring.datasource.password
```

With:

```text
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
```

### Outcome

- [ ] Externalized configuration

---

# Phase 4.5 — Integration Testing

## Objective

Verify complete application flows instead of isolated units.

---

## Testing Pyramid

### Learn

- [ ] Unit Testing
- [ ] Integration Testing
- [ ] End-to-End Testing

### Understand

```text
Unit Test
    ↓
Integration Test
    ↓
End-to-End Test
```

### Outcome

- [ ] Know when to use each test type

---

## Spring Boot Integration Testing

### Learn

- [ ] @SpringBootTest
- [ ] Test Context Loading
- [ ] Real Bean Interaction

### Outcome

- [ ] Create first integration test

---

## MockMvc

### Learn

- [ ] mockMvc.perform()
- [ ] Request Builders
- [ ] Response Validation

Example:

```java
mockMvc.perform(get("/api/issues"))
```

### Outcome

- [ ] Test REST endpoints without starting server manually

---

## Assertions

### Learn

- [ ] status()
- [ ] jsonPath()
- [ ] content()

Example:

```java
status().isOk()
jsonPath("$.data.id")
```

### Outcome

- [ ] Validate API responses

---

## Test Database

### Stage 1

- [ ] H2 Database

### Stage 2

- [ ] Testcontainers
- [ ] PostgreSQL Container

### Outcome

- [ ] Run tests against isolated databases

---

# Recommended Learning Order

## Week 1

- [ ] Spring Mail
- [ ] SMTP
- [ ] Mailtrap
- [ ] HTML Emails
- [ ] Async Processing
- [ ] Event → Email Flow

---

## Week 2

- [ ] Docker Basics
- [ ] Dockerfile
- [ ] Spring Boot Containerization
- [ ] Docker Compose
- [ ] Environment Variables

---

## Week 3

- [ ] SpringBootTest
- [ ] MockMvc
- [ ] Assertions
- [ ] H2 Testing
- [ ] Testcontainers

---

# Completion Checklist

## Email Notifications

- [ ] Mailtrap configured
- [ ] Plain text email
- [ ] HTML email
- [ ] Async email sending
- [ ] Issue assigned email
- [ ] Status change email
- [ ] Priority change email

## Dockerization

- [ ] Dockerfile created
- [ ] Image built successfully
- [ ] Spring Boot container runs
- [ ] PostgreSQL container runs
- [ ] Docker Compose works
- [ ] Environment variables configured

## Integration Testing

- [ ] First integration test written
- [ ] MockMvc configured
- [ ] Controller tests passing
- [ ] H2 tests passing
- [ ] Testcontainers learned
- [ ] Critical API flows covered

---

# Not Required Yet

Leave these for after DevSync:

- [ ] Kafka
- [ ] RabbitMQ
- [ ] Redis
- [ ] Microservices
- [ ] Kubernetes
- [ ] CQRS
- [ ] Event Sourcing

Focus on finishing DevSync first.
