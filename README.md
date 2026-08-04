# Patient Management System — Spring Boot Microservices

A distributed patient management platform built with Java Spring Boot, demonstrating
microservice architecture patterns including API gateway routing, JWT authentication,
synchronous gRPC communication, and asynchronous event streaming with Kafka.

## Architecture

The system is composed of five independently deployable services:

| Service | Responsibility | Communication |
|---|---|---|
| **api-gateway** | Single entry point; routes requests and validates JWTs before forwarding | HTTP |
| **auth-service** | User authentication, JWT issuance and validation | HTTP + PostgreSQL |
| **patient-service** | Patient CRUD operations; publishes patient events | HTTP, gRPC client, Kafka producer |
| **billing-service** | Billing account creation | gRPC server |
| **analytics-service** | Consumes patient events for downstream analytics | Kafka consumer |

### Request flow

A client authenticates against `auth-service` and receives a JWT. All subsequent
requests pass through `api-gateway`, which validates the token via a custom
`JwtValidationGatewayFilterFactory` before routing to the target service.

When a patient is created, `patient-service` makes a synchronous gRPC call to
`billing-service` to provision a billing account, then publishes a Protobuf-encoded
`PatientEvent` to Kafka. `analytics-service` consumes that event independently —
so analytics failures never block patient creation.

### Why these choices

- **gRPC for billing** — billing account creation must succeed before the patient
  record is considered complete, so it needs a synchronous call with a typed contract.
- **Kafka for analytics** — analytics is not on the critical path. Decoupling it via
  events means the patient flow stays fast and stays up if analytics is down.
- **Protobuf** — shared `.proto` definitions give both gRPC and Kafka payloads a
  single source of truth for schema.

## Tech Stack

Java 21 · Spring Boot 3.4 · Spring Cloud Gateway · Spring Security · gRPC · Protobuf ·
Apache Kafka · PostgreSQL · Docker · AWS CDK / LocalStack · JUnit 5 · Rest Assured

## Running Locally

**Prerequisites:** JDK 21, Maven, Docker Desktop

Each service is a standalone Maven module with its own `Dockerfile`. To run a
single service:

```bash
cd patient-service
./mvnw spring-boot:run
```

Services expect their dependencies (PostgreSQL, Kafka) to be reachable. Environment
variables for each service are documented in `SETUP.md`.

### Infrastructure

The `infrastructure/` module defines the AWS deployment via CDK, targeting LocalStack
for local emulation:

```bash
cd infrastructure
./localstack-deploy.sh
```

## Testing

Integration tests in `integration-tests/` exercise the auth and patient flows
end-to-end through the gateway:

```bash
cd integration-tests
mvn test
```

## API Requests

Sample HTTP requests for each endpoint live in `api-requests/` and `grpc-requests/`,
runnable directly from IntelliJ's HTTP client.

---

Built while working through Chris Blakely's Spring Boot microservices course.