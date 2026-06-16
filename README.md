# demo-java-app

A simple Spring Boot 3.2.5 web application that demonstrates a CI/CD pipeline.

## Overview

This application serves a Thymeleaf-based UI at the root context (`/`). It showcases a minimal Spring Boot project structure with:
- Spring Boot 3.2.5
- Java 17
- Maven build system
- CI/CD integration

## Prerequisites

- Java 17 or later
- Maven 3.9+

## Getting Started

### Running Tests

```bash
mvn test
```

### Running Locally

```bash
mvn spring-boot:run
```

The application will start on `http://localhost:8080`.

### Building the Application

```bash
mvn clean package -DskipTests
```

This creates the JAR file in the `target` directory.

## CI/CD

This project uses GitHub Actions for continuous integration:
- **Pull Requests**: Trigger reusable Java pipeline (requires `feature/`, `release/`, `hotfix/`, or `bugfix/` prefix)
- **Main Branch**: Deploys via reusable pipeline template

## Docker

The application can be containerized using the multi-stage `Dockerfile`:

```bash
docker build -t demo-java-app .
docker run -p 8080:8080 demo-java-app
```

## Project Structure

```
demo-java-app/
├── src/
│   ├── main/java/com/example/demo/
│   │   ├── DemoApplication.java   # Spring Boot entry point
│   │   └── HomeController.java    # Web controller
│   ├── main/resources/templates/
│   │   └── index.html             # Thymeleaf template
│   └── test/java/com/example/demo/
│       └── HomeControllerTest.java # Web MVC test
├── pom.xml                        # Maven configuration
└── Dockerfile                     # Multi-stage Docker build
```
