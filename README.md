# Patient Management System — Project Documentation

## Overview

The Patient Management System is a backend microservices application built with Java and Spring Boot. It demonstrates a production-style distributed architecture where multiple independent services communicate via REST, gRPC, and Kafka, and are deployed to a cloud-like environment using AWS CDK with LocalStack.

The system manages patient records, handles authentication, processes billing, and feeds an analytics pipeline — all across separate, independently deployable services.

---

## Architecture

The system is composed of six distinct services, each with its own responsibility:

```
Client
  └─► API Gateway (port 4004)
        ├─► Auth Service      (port 4005)  — JWT authentication
        ├─► Patient Service   (port 4000)  — Patient CRUD operations
        │     ├─► Billing Service (gRPC, port 9001) — Billing account creation
        │     └─► Kafka Topic "patient"   — Event publishing
        └─► Analytics Service (port 4002) — Kafka event consumption
```

### Services

| Service | Port | Responsibility |
|---|---|---|
| **API Gateway** | 4004 | Routes all incoming requests; enforces JWT authentication |
| **Auth Service** | 4005 | User login, JWT generation and validation |
| **Patient Service** | 4000 | Patient CRUD; orchestrates gRPC and Kafka calls |
| **Billing Service** | 4001 / 9001 | Exposes a gRPC server to create billing accounts |
| **Analytics Service** | 4002 | Kafka consumer that processes patient events |
| **Infrastructure** | — | AWS CDK stack definition (deployed via LocalStack) |

---

## Technology Stack

### Backend
- **Java 21** with **Spring Boot 4.x**
- **Spring Cloud Gateway** (reactive, WebFlux-based) for API routing
- **Spring Data JPA** + **PostgreSQL** for persistence
- **Spring Security** for auth configuration
- **JJWT** for JWT generation and validation

### Inter-Service Communication
- **REST** — API Gateway to Auth Service and Patient Service
- **gRPC** (via Protocol Buffers) — Patient Service to Billing Service
- **Apache Kafka** — Patient Service produces events; Analytics Service consumes them

### Infrastructure & Deployment
- **Docker** — each service has its own multi-stage Dockerfile
- **AWS CDK (Java)** — infrastructure defined as code
- **LocalStack** — local AWS cloud emulation (ECS Fargate, RDS PostgreSQL, MSK Kafka, ALB)
- **H2** — in-memory database available for local development without Docker

### Testing
- **JUnit 5**
- **REST Assured** — integration tests for auth and patient flows

---

## Key Design Decisions

### JWT Authentication at the Gateway
The API Gateway intercepts all requests to `/api/patients/**` and validates the Bearer token by calling the Auth Service's `/validate` endpoint before forwarding the request downstream. This keeps authentication concerns out of individual services.

### gRPC for Billing
When a new patient is created, the Patient Service synchronously calls the Billing Service via gRPC to create a billing account. Protocol Buffers are used to define the shared contract (`billing_service.proto`), ensuring type safety across service boundaries.

### Kafka for Event-Driven Analytics
Patient creation also publishes a Protobuf-serialized `PatientEvent` to a Kafka topic. The Analytics Service consumes this topic independently, decoupling analytics processing from the core patient workflow.

### Profile-Based Gateway Configuration
The API Gateway uses Spring profiles (`application.yml` vs `application-prod.yml`) to switch between local development (localhost) and containerized (Docker internal DNS) routing without code changes.

---

## Project Structure

```
/
├── api-gateway/          # Spring Cloud Gateway — routing & JWT filter
├── auth-service/         # Login endpoint, JWT util, user repository
├── patient-service/      # Patient CRUD, gRPC client, Kafka producer
├── billing-service/      # gRPC server implementation
├── analytics-service/    # Kafka consumer
├── infrastructure/       # AWS CDK stack (LocalStack.java)
├── integration-tests/    # REST Assured end-to-end tests
├── api-requests/         # HTTP request files for manual testing
└── grpc-requests/        # gRPC request files for manual testing
```

---

## Running Locally

### Prerequisites
- Docker and Docker Compose
- Java 21+
- Maven

### With Docker (Full Stack)
Each service includes a multi-stage Dockerfile. Build and run each image, or use the infrastructure stack with LocalStack:

```bash
# Deploy via LocalStack (requires LocalStack running)
cd infrastructure
bash localstack-deploy.sh
```

### Local Development (Without Docker)
The Patient Service and Auth Service support H2 in-memory database for quick local development. Uncomment the H2 datasource properties in `application.properties` and run the service directly:

```bash
cd patient-service
./mvnw spring-boot:run
```

### Running Integration Tests
```bash
cd integration-tests
./mvnw test
```

The integration tests cover:
- Successful login returning a valid JWT
- Rejected login with invalid credentials
- Fetching patients with a valid Bearer token

---

## API Reference

All requests go through the API Gateway at `http://localhost:4004`.

### Authentication

**Login**
```
POST /auth/login
Content-Type: application/json

{ "email": "testuser@test.com", "password": "password123" }
```
Returns: `{ "token": "<jwt>" }`

**Validate Token**
```
GET /auth/validate
Authorization: Bearer <token>
```

### Patients (requires valid JWT)

| Method | Path | Description |
|---|---|---|
| GET | `/api/patients` | List all patients |
| POST | `/api/patients` | Create a new patient |
| PUT | `/api/patients/{id}` | Update an existing patient |
| DELETE | `/api/patients/{id}` | Delete a patient |

**Create Patient Body:**
```json
{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "address": "123 Main Street",
  "dateOfBirth": "1990-01-15",
  "registeredDate": "2024-11-28"
}
```

### Billing (gRPC)
```
GRPC localhost:9001/BillingService/CreateBillingAccount

{ "patientId": "...", "name": "...", "email": "..." }
```

---

## Infrastructure (AWS CDK)

The `infrastructure` module defines the full cloud topology as Java code using the AWS CDK:

- **VPC** with public and private subnets across 2 availability zones
- **RDS PostgreSQL** instances for Auth Service and Patient Service (each isolated)
- **Route53 Health Checks** on both databases
- **MSK (Kafka) Cluster** with 2 broker nodes
- **ECS Fargate** tasks for all five application services
- **Application Load Balancer** exposing the API Gateway publicly

The stack is synthesized to a CloudFormation template and deployed locally via LocalStack, enabling full cloud infrastructure simulation without an AWS account.

---

## Security Notes

- All patient endpoints require a valid JWT passed as `Authorization: Bearer <token>`
- JWT secret is configurable via environment variable (`JWT_SECRET`)
- Tokens expire after 10 hours
- Passwords are hashed with BCrypt
- CSRF protection is disabled (stateless REST API design)
- The default test user (`testuser@test.com` / `password123`) is seeded via `data.sql` at startup
