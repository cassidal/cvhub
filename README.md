# CVHub Auth Service

A Java/Spring Boot authentication backend for CVHub, a personal project for creating and sharing resumes. The implemented scope in this repository is the authentication service: gRPC APIs, JWT access tokens, persistent sessions, and automated tests.

## What it does

- Registers and authenticates users through a Protobuf-defined gRPC API, with input validation and BCrypt password hashing.
- Issues short-lived JWT access tokens and UUID refresh tokens backed by database sessions. Refresh requests check session validity, expiry, and user status.
- Protects a private gRPC endpoint with a JWT interceptor and maps application errors to gRPC status codes.
- Persists users and sessions through Spring Data JPA, with PostgreSQL for the application and Liquibase schema migrations.
- Tests service logic with JUnit/Mockito and exercises gRPC calls through in-process servers with a Spring context and H2 database.

## Stack and layout

**Java 24 · Spring Boot 3.5.6 · gRPC · Protocol Buffers · PostgreSQL · Spring Data JPA · Liquibase · JUnit 5 · Mockito**

| Path | Purpose |
| --- | --- |
| [`grpc/`](grpc) | Public/private API contracts and generated Java stubs |
| [`auth-service/`](auth-service) | gRPC handlers, application services, JWT authentication, repositories, and migrations |
| [`auth-service/src/test/`](auth-service/src/test) | Unit tests and Spring/gRPC integration tests |
| [`.github/workflows/auth-service-per-pr.yml`](.github/workflows/auth-service-per-pr.yml) | Pull-request test, verification, and static-analysis jobs |

Requests flow from gRPC handlers through an application facade and user/session services to JPA repositories. Interceptors handle authentication, error mapping, and request logging.

## Run locally

Requirements: **JDK 24**, **Maven 3.9+**, and PostgreSQL. Docker is optional for the database example; `grpcurl` is optional for the API examples. Run the following commands from the repository root.

### 1. Build the API module

The service depends on the local `ru.cassidal:grpc:1.0-SNAPSHOT` artifact. Install it before building or testing the service:

```sh
mvn -f grpc/pom.xml clean install -DskipTests
```

### 2. Start a local database

For example, using Docker with credentials intended only for local development:

```sh
docker run --name cvhub-auth-postgres \
  -e POSTGRES_DB=cvhub_auth \
  -e POSTGRES_USER=cvhub \
  -e POSTGRES_PASSWORD=local-development-only \
  -p 127.0.0.1:5432:5432 \
  -d postgres:16
```

### 3. Configure and start the service

```sh
export POSTGRES_HOST=localhost
export POSTGRES_PORT=5432
export POSTGRES_DB=cvhub_auth
export POSTGRES_USER=cvhub
export POSTGRES_PASSWORD=local-development-only
export JWT_SECRET="$(openssl rand -hex 32)"
export SERVER_PORT=9090

mvn -f auth-service/pom.xml spring-boot:run
```

Liquibase applies the schema on startup. The default configuration serves gRPC on port `9090`, enables server reflection, uses 15-minute access tokens, and gives sessions a one-day lifetime. These commands use the default profile; the checked-in `local` profile has separate database settings and port `9091`.

## API examples

The examples use plaintext transport on a local development machine. The contracts are in [`public/auth_service.proto`](grpc/src/main/proto/public/auth_service.proto) and [`private/auth_service.proto`](grpc/src/main/proto/private/auth_service.proto).

| Method | Input | Result |
| --- | --- | --- |
| `auth.AuthService/Register` | Email and password | Access token and refresh token |
| `auth.AuthService/Login` | Email and password | Access token and refresh token |
| `auth.AuthService/RefreshToken` | Refresh token | New access token and the existing refresh token |
| `auth.PrivateAuthService/IsAuthenticated` | Empty message, plus bearer-token metadata | Authentication status |

Register a user (passwords require at least 12 characters, with uppercase, lowercase, a digit, and a special character):

```sh
grpcurl -plaintext \
  -d '{"email":"demo@example.com","password":"LocalDemoPass123!"}' \
  localhost:9090 auth.AuthService/Register
```

Log in with the same credentials:

```sh
grpcurl -plaintext \
  -d '{"email":"demo@example.com","password":"LocalDemoPass123!"}' \
  localhost:9090 auth.AuthService/Login
```

Use the returned `accessToken` and `refreshToken` values in the following calls:

```sh
grpcurl -plaintext -H 'authorization: Bearer <accessToken>' \
  -d '{}' localhost:9090 auth.PrivateAuthService/IsAuthenticated

grpcurl -plaintext -d '{"refreshToken":"<refreshToken>"}' \
  localhost:9090 auth.AuthService/RefreshToken
```

Validation failures map to `INVALID_ARGUMENT`, duplicate registration to `ALREADY_EXISTS`, authentication failures to `UNAUTHENTICATED`, and inactive users to `PERMISSION_DENIED`.

## Tests and verification

After installing the API module:

```sh
mvn -f auth-service/pom.xml clean test
```

The test profile uses H2 in PostgreSQL compatibility mode and test-only JWT settings, so no external database or credentials are required. Coverage includes registration and login, duplicate/invalid inputs, inactive users, session reuse, invalid or expired refresh tokens, and private calls with invalid or expired JWTs. H2-based tests do not replace testing against PostgreSQL.

To also run the configured Checkstyle and POM-formatting checks:

```sh
mvn -f auth-service/pom.xml clean verify
```

The SortPom plugin may reformat POM files during a build; review the working tree afterward. The same build order is used by the pull-request workflow for changes under `auth-service/` and `grpc/`.

## Current scope

This is an authentication backend portfolio project; resume editing and sharing are not implemented here yet. Refresh tokens are reused until session expiry rather than rotated on each refresh, and the current API does not expose logout or session revocation.
