# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Apache Polaris is an open-source catalog for Apache Iceberg that implements the Iceberg REST API. It's built with Java 21+ using Gradle and Quarkus, enabling multi-engine interoperability across platforms like Apache Spark, Flink, Trino, and Dremio.

## Build Requirements

- **Java 21+** (required) - The build will fail with Java versions below 21
- **Docker 27+** (for integration tests and container builds)
- **Gradle** (via wrapper `./gradlew`)

## Common Development Commands

### Build and Test
```bash
# Full build with tests (requires Docker running)
./gradlew build

# Build without running tests
./gradlew assemble

# Run all checks (formatting, tests, integration tests)
./gradlew check

# Format code and compile
./gradlew format compileAll
```

### Running Polaris Locally
```bash
# Run Polaris server on localhost:8181
./gradlew run

# Run with custom credentials (REALM, CLIENT_ID, CLIENT_SECRET)
./gradlew run -Dpolaris.bootstrap.credentials=POLARIS,root,secret
# Default credentials if not specified: POLARIS,root,s3cr3t
```

### Testing

#### Unit and Integration Tests
```bash
# Run all checks including tests
./gradlew check

# Run tests for a specific module
./gradlew :polaris-core:test
```

#### Regression Tests
```bash
# Run regression tests locally (requires Polaris running on localhost:8181)
env POLARIS_HOST=localhost ./regtests/run.sh

# Run specific regression tests in verbose mode
env VERBOSE=1 POLARIS_HOST=localhost ./regtests/run.sh t_spark_sql/src/spark_sql_basic.sh

# Run regression tests with Docker Compose
./gradlew :polaris-server:assemble :polaris-server:quarkusAppPartsBuild --rerun -Dquarkus.container-image.build=true
docker compose -f ./regtests/docker-compose.yml up --build --exit-code-from regtest
```

#### Spark SQL Shell
```bash
# Run interactive Spark SQL shell (requires Polaris running)
./regtests/run_spark_sql.sh
```

### Docker Image Building
```bash
# Build server Docker image
./gradlew :polaris-server:assemble :polaris-server:quarkusAppPartsBuild --rerun -Dquarkus.container-image.build=true

# Build admin tool Docker image
./gradlew :polaris-admin:assemble :polaris-admin:quarkusAppPartsBuild --rerun -Dquarkus.container-image.build=true

# Use Makefile shortcuts
make build-server  # Builds server and container image
make build-admin   # Builds admin and container image
```

### Python Client
```bash
# Regenerate Python client from OpenAPI specs
make client-regenerate

# Run Python client unit tests
make client-unit-test

# Run Python client integration tests (requires server running)
make client-integration-test
```

### Documentation
```bash
# Run Hugo documentation site locally (in Docker)
site/bin/run-hugo-in-docker.sh
```

## Architecture Overview

### Module Organization

**Core Business Logic:**
- `polaris-core` - Entity definitions, core business logic, authorization (RBAC), persistence interfaces, and storage integration

**API Modules** (generated from OpenAPI specs in `spec/`):
- `polaris-api-management-model` - Management API model classes
- `polaris-api-management-service` - Management API service classes
- `polaris-api-iceberg-service` - Iceberg REST API service classes
- `polaris-api-catalog-service` - Polaris Catalog API service classes

**Runtime Modules:**
- `polaris-server` - Quarkus-based server (main runtime)
- `polaris-admin` - Admin tool for bootstrapping persistence
- `polaris-runtime-service` - Core Polaris service package
- `polaris-runtime-defaults` - Default runtime configurations
- `polaris-distribution` - Distribution packaging

**Persistence Implementations:**
- `persistence/relational-jdbc` - JDBC persistence implementation
- `persistence/nosql/*` - NoSQL persistence implementations with multiple submodules for authz, nodes, realms, etc.

**Extensions:**
- `extensions/auth/opa` - Open Policy Agent (OPA) authorization extension
- `extensions/federation/hive` - Hive federation extension
- `extensions/federation/hadoop` - Hadoop federation extension

**Plugins:**
- `plugins/spark/` - Spark plugin for Polaris (built for multiple Spark/Scala version combinations)

### Key Architecture Concepts

**Entity Management:**
- Entity types defined in `PolarisEntityType` at `org.apache.polaris.core.entity`
- Core entities include catalogs, principals, namespaces, tables, and views

**Authorization:**
- Role-Based Access Control (RBAC) model
- `PolarisAuthorizer` (interface at `polaris-core/src/main/java/org/apache/polaris/core/auth/PolarisAuthorizer.java`)
- `PolarisAuthorizerImpl` provides default implementation
- `PolarisPrivilege` defines available privileges
- Can be extended via `PolarisAuthorizerFactory`

**Persistence Layer:**
- Built on `TransactionalPersistence` interface
- Common persistence logic in `org.apache.polaris.core.persistence`
- Multiple backend implementations: JDBC, NoSQL (MongoDB, in-memory)
- Transactional semantics with optimistic concurrency control

**Storage Integration:**
- `PolarisStorageIntegration` interface at `polaris-core/src/main/java/org/apache/polaris/core/storage/PolarisStorageIntegration.java`
- Supports AWS S3, GCS, Azure (ADLS/Blob Storage), and file-based storage
- Handles credential vending and storage configuration

**API Generation:**
- Java service/model classes are generated from OpenAPI specifications in `spec/`
- Specs include: Polaris Management API, Polaris Catalog API, and Iceberg REST API
- Use `make client-regenerate` to regenerate Python client after spec changes

## Configuration

**Default Configuration:**
- Located in `runtime/defaults/src/main/resources/application.properties`
- Quarkus configuration properties (build-time vs runtime)
- Server runs on port 8181, management/metrics on 8182

**Runtime Configuration:**
- Feature flags: `polaris.features.*`
- Storage backends: S3, GCS, Azure, FILE
- OIDC/authentication settings
- See [Configuration Guide](site/content/in-dev/unreleased/configuration.md)

## Testing Notes

**Integration Tests Require Docker:**
- Test containers are used extensively (MinIO, databases, etc.)
- Ensure Docker daemon is running before executing `./gradlew build` or `./gradlew check`

**Regression Test Environment Variables:**
For cloud storage tests, create `.env` file in `regtests/` with:
- `AWS_TEST_ENABLED`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_STORAGE_BUCKET`, `AWS_ROLE_ARN`
- `GCS_TEST_ENABLED`, `GCS_TEST_BASE`, `GOOGLE_APPLICATION_CREDENTIALS`
- `AZURE_TEST_ENABLED`, `AZURE_TENANT_ID`, `AZURE_DFS_TEST_BASE`, `AZURE_BLOB_TEST_BASE`

**Catalog Federation Tests:**
Require these properties in `application.properties`:
```
polaris.features."ENABLE_CATALOG_FEDERATION"=true
polaris.features."ALLOW_OVERLAPPING_CATALOG_URLS"=true
polaris.features."ALLOW_NAMESPACE_CUSTOM_LOCATION"=true
```

## Code Style

- Code formatting enforced via Spotless plugin
- Checkstyle configuration in `codestyle/checkstyle.xml`
- Copyright headers required (see `codestyle/copyright-header-java.txt`)
- Run `./gradlew format` before committing
- ErrorProne static analysis enabled

## Contribution Workflow

1. Code changes should be based on `main` branch
2. Run `./gradlew format compileAll` before committing
3. Run `./gradlew check` to verify all checks pass
4. Commits follow [Conventional Commits](https://www.conventionalcommits.org/) style
5. PRs require passing CI (GitHub Actions)

## Important File Locations

- OpenAPI specs: `spec/polaris-management-service.yml`, `spec/polaris-catalog-service.yaml`
- Version: `version.txt`
- Gradle projects mapping: `gradle/projects.main.properties`
- Spark/Scala versions: `plugins/spark/spark-scala.properties`
- Server startup banner: `runtime/service/src/main/resources/org/apache/polaris/service/banner.txt`

## Makefile Shortcuts

A root-level Makefile provides convenient wrappers for common tasks:
```bash
make help                    # Show all available targets
make build                   # Build server and admin
make build-server            # Build Polaris server
make client-regenerate       # Regenerate Python client
make helm-unittest           # Run Helm chart tests
make minikube-start-cluster  # Start local Kubernetes cluster
```
