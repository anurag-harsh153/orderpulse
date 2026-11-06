# OrderPulse

A small e-commerce order system, built step by step to learn the full Java + Angular stack.

Customers browse products, add to a cart, place orders and track their status. Admins manage products. Placing an order sends an event through Kafka to a second service that creates notifications.

## Tech stack

| Layer | Technology |
|---|---|
| Backend | Java 17, Spring Boot 4.1.2, Maven |
| Database | PostgreSQL 16 (Docker), Flyway migrations, Spring Data JPA |
| Frontend | Angular (planned, Stage 4) |
| Messaging / cache | Kafka, Redis (planned, Stage 5) |
| Infra | Docker, Docker Compose |

## Project structure

```
orderpulse/
  docker/                  docker-compose.yml (PostgreSQL)
  shop-api/                Spring Boot API (users, products, orders)
  notification-service/    Spring Boot Kafka consumer (planned)
  frontend/                Angular app (planned)
```

## Prerequisites

| Tool | Version used | Check with |
|---|---|---|
| JDK | 17 (Temurin) | `java -version` |
| Maven | 3.9.x | `mvn -version` |
| Node.js | 22.x | `node -v` |
| Git | 2.x | `git --version` |
| Docker Desktop | 29.x | `docker --version` and `docker ps` |

Java 8 is also installed on this machine, so JDK 17 is selected per terminal session:

```
set "JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-17.0.21.9-hotspot"
set "PATH=%JAVA_HOME%\bin;%PATH%"
```

In IntelliJ: File > Project Structure > Project SDK = 17. Settings > Build Tools > Maven > Runner > JRE = Project JDK.

## Setup so far

### Step 0: Create the repository

```
mkdir orderpulse
cd orderpulse
git init
mkdir docker shop-api frontend notification-service
```

One repository keeps every part of the system, its history and this README in one place.

### Step 0.3: Run PostgreSQL in Docker

Docker Desktop must be running ("Engine running") before any `docker` command works.

Create `docker/docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: orderpulse-postgres
    environment:
      POSTGRES_DB: orderpulse
      POSTGRES_USER: orderpulse
      POSTGRES_PASSWORD: orderpulse
    ports:
      - "5433:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Notes:
- Host port is **5433** because a local PostgreSQL service already uses 5432 on this machine. The container still uses 5432 inside.
- The `pgdata` volume keeps data when the container is removed and recreated. `docker compose down -v` deletes the volume and wipes the data.

Start and check:

```
docker compose -f docker/docker-compose.yml up -d
docker ps
docker exec -it orderpulse-postgres psql -U orderpulse -d orderpulse -c "select version();"
```

Expected: container `orderpulse-postgres` with `0.0.0.0:5433->5432/tcp`, and a line starting with "PostgreSQL 16".

Useful commands:

```
docker stop orderpulse-postgres      # stop the database
docker start orderpulse-postgres     # start it again
docker compose -f docker/docker-compose.yml down      # remove container, keep data
docker compose -f docker/docker-compose.yml down -v   # remove container and data
```

### Step 3: Create the Spring Boot project

Generated at start.spring.io with:

- Maven, Java 17, Spring Boot 4.1.2, packaging Jar
- Group `com.orderpulse`, artifact `shop-api`
- Dependencies: Spring Web, Spring Data JPA, Validation, PostgreSQL Driver, Spring Boot Actuator, Flyway Migration

The contents are unzipped so `pom.xml` sits directly in `shop-api/`.

If IntelliJ shows "not found" errors in `pom.xml`:

1. Run `mvn -U dependency:resolve` from `shop-api/` (the `-U` flag retries downloads Maven gave up on).
2. In IntelliJ, click Reload All Maven Projects in the Maven panel.

### Step 4: Configure the database and first run

`shop-api/src/main/resources/application.yml` (replaces `application.properties`):

```yaml
spring:
  application:
    name: shop-api
  datasource:
    url: jdbc:postgresql://localhost:5433/orderpulse
    username: orderpulse
    password: orderpulse
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    show-sql: true
    properties:
      hibernate:
        format_sql: true

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

What the settings mean:

- `datasource.url` format is `jdbc:postgresql://host:port/database`.
- `ddl-auto: validate` makes Hibernate check entities against the tables without changing them. Flyway owns the schema.
- `open-in-view: false` closes database sessions when the service method ends, so lazy-loading problems show up early.
- `show-sql` and `format_sql` print every SQL statement, which is needed to spot the N+1 problem in Stage 1.

Run `ShopApiApplication` from IntelliJ, or from `shop-api/`:

```
mvn spring-boot:run
```

Check: open http://localhost:8080/actuator/health. Expected:

```json
{"groups": ["liveness", "readiness"], "status": "UP"}
```

Spring Boot 4 enables the liveness and readiness health groups by default. Kubernetes uses them to check whether an app is alive and ready for traffic.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `docker` commands fail with a pipe or daemon error | Docker Desktop is not running. Start it and wait for "Engine running". |
| "port is already allocated" | Another program uses the host port. Change the left side of the `ports` mapping and the port in the JDBC URL. |
| `netstat -ano \| findstr :5432` shows a listener | Local PostgreSQL service. `tasklist /FI "PID eq <pid>"` shows the process. We use 5433 to avoid it. |
| App fails with "connection refused" | Postgres container not running, or wrong port in the URL. Check `docker ps`. |
| "password authentication failed" | Username or password differs from `docker-compose.yml`. |
| Maven "not found" errors in IntelliJ | Run `mvn -U dependency:resolve`, then reload Maven projects. |
| "Unable to find location: classpath:db/migration" | Create `src/main/resources/db/migration`. |
| `java -version` shows 1.8 | Set `JAVA_HOME` and `PATH` to JDK 17 in that terminal. |

## Progress

- [x] Step 0.1 Tools check (JDK 17, Maven, Node, Git, Docker)
- [x] Step 0.2 Repository and folders
- [x] Step 0.3 PostgreSQL running in Docker (port 5433)
- [x] Step 3 Spring Boot 4.1.2 project generated
- [x] Step 4 Configuration and first run (health endpoint UP)
- [ ] Step 5 Table design
- [ ] Step 6 Flyway migration
- [ ] Step 7 Product CRUD

## Commit convention

Commit after every step with a clear message, for example:

```
git add .
git commit -m "Step 0-4: docker postgres and spring boot skeleton"
```