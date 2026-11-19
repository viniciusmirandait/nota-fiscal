# GEMINI.md - CodeNotaFiscal Project Guidelines

## Overview
**CodeNotaFiscal** is a Spring Boot application designed for processing Brazilian Tax Invoices (Nota Fiscal).

## Tech Stack
- **Language**: Java 17
- **Framework**: Spring Boot 3.5.x (Spring Web, Spring Data JPA, Spring Security)
- **Database**: PostgreSQL
- **Security**: JWT (`io.jsonwebtoken`)
- **Documentation**: OpenAPI 3 / Swagger (`springdoc-openapi-starter-webmvc-ui`)
- **Build Tool**: Apache Maven (`./mvnw`)

## Common Commands
```bash
# Build project
./mvnw clean compile

# Run unit and integration tests
./mvnw test

# Run application locally
./mvnw spring-boot:run
```

## Structure
- `com.projeto.notafiscal.controller` - REST API endpoints
- `com.projeto.notafiscal.service` - Business rules and calculations
- `com.projeto.notafiscal.repository` - Data access interfaces
- `com.projeto.notafiscal.model` - JPA Entities
- `com.projeto.notafiscal.security` - Security configuration & JWT authentication
- `com.projeto.notafiscal.exceptions` - Exception handling
