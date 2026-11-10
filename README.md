# Session 04 / Exercise 01 — StoreX Docker Compose

This repository uses a minimal Spring Boot test fixture because the original StoreX backend image/source was not provided. The fixture exists solely to demonstrate Docker Compose service discovery and PostgreSQL connectivity.

## Objective and root cause

The original JDBC URL was `jdbc:postgresql://localhost:5432/storex`.
Containers have separate network namespaces: `localhost` inside `backend` means
the backend container, not the PostgreSQL container or host.

Compose creates the project-scoped `rikkei-ss04-storex_default` network.
Both services join it, and Docker's internal DNS resolves the service name
`postgres`. The corrected JDBC URL is:

```text
jdbc:postgresql://postgres:5432/storex
```

Container-to-container traffic uses PostgreSQL's internal port `5432`; it does
not require a published host port. PostgreSQL host-port publishing was removed
with authorization because the host PostgreSQL already uses `5432`. Only
backend port `8080:8080` is published. No host IP or fixed container IP is used.

DNS resolution and network reachability do not imply database readiness.
`pg_isready` plus `depends_on: condition: service_healthy` delay backend startup
until PostgreSQL accepts TCP connections. Spring datasource initialization and
real SQL execution are verified separately. The named volume preserves database
data; Compose scopes the network, containers, and volume to this project.

## Fixture

- Java 21 (verified runtime: Temurin **21.0.13**), Spring Boot **3.5.16**,
  Maven **3.9.11** build container.
- Spring Web, Spring JDBC / HikariCP, PostgreSQL JDBC driver, and Actuator.
- Images: locally built `storex-backend:latest` and official `postgres:13`.
- The backend's `pull_policy: never` prevents pulling an unavailable public image.
- The assignment's database/user/password are `storex` / `postgres` / `secret`.
  These are fixed lab values from the assignment, not personal credentials.

On startup and on every `GET /db-check`, `JdbcTemplate` executes:

```sql
SELECT 1 AS query_result, current_database() AS database_name,
       inet_server_addr()::text AS server_address, inet_server_port() AS server_port;
```

HTTP 200 reports the actual query result, database name, and database server
address. HTTP 503 reports a failed query without returning credentials or
exception details. A failed startup query prevents successful application
startup. `/actuator/health` also checks the configured datasource.

## Build and start

Requires Docker with Compose, registry/Maven Central access, and a free host TCP
port `8080`. No local Maven installation is needed.

```bash
ss -ltnp '( sport = :8080 )'
docker build -t storex-backend:latest .
docker image inspect storex-backend:latest
docker compose config -q
docker compose config
docker compose up -d
docker compose ps
docker compose logs --tail 100 postgres
docker compose logs --tail 100 backend
```

The Compose file sets the unique project name `rikkei-ss04-storex`.
The multi-stage Dockerfile compiles the included source and runs its executable
JAR in a Java 21 JRE as an unprivileged user. Maven dependencies are versioned
through the Spring Boot parent. Base-image tags remain mutable; this is a
repeatable build procedure, not a claim of byte-identical future image builds.

## Verify real database connectivity

Wait for application startup, then run:

```bash
curl -i http://localhost:8080/db-check
curl -i http://localhost:8080/actuator/health
docker compose exec -T backend getent hosts postgres
docker compose exec -T postgres psql -U postgres -d storex -c 'SELECT 1, current_database();'
```

The HTTP URL above accesses the published backend from the host; the backend
itself always uses `postgres:5432` for JDBC. Container status `Up` alone is not
database connectivity evidence.

### Optional controlled failure/recovery test

This only stops this assignment's database, preserves its volume, and restores
it immediately. Do not use these commands against another project's services.

```bash
docker compose stop postgres
curl -i http://localhost:8080/db-check  # expect HTTP 503, queryExecuted=false
docker compose start postgres
# Wait for PostgreSQL readiness AND HikariCP reconnection; retry until HTTP 200.
curl -i http://localhost:8080/db-check
```

## Actual test results

Executed on **2026-11-11, UTC+07:00**, using NixOS 26.11, Docker 29.5.3,
and Docker Compose 5.1.4. Container/HTTP timestamps below are UTC.

| Check | Observed result |
|---|---|
| Multi-stage Docker build | Maven `BUILD SUCCESS`; `storex-backend:latest` inspected successfully |
| Compose validation | `docker compose config -q` passed; effective configuration assertions passed |
| PostgreSQL | Official `postgres:13` started PostgreSQL **13.23**; healthcheck healthy; `storex` exists |
| Database host publishing | Docker inspect reports `{"5432/tcp":null}`: no published host database port |
| Backend | Started on `8080`, completed startup SQL, remained running with zero restarts |
| Docker DNS | `getent hosts postgres` in backend returned `172.24.0.2` during this run |
| Positive SQL check | HTTP **200**, `databaseReachable=true`, `queryExecuted=true`, `database=storex`, `queryResult=1` |
| Connection attribution | SQL server address `172.24.0.2/32:5432` matched this Compose PostgreSQL container; `pg_stat_activity` showed JDBC sessions from backend `172.24.0.3` |
| Actuator | HTTP **200**, overall `UP`, database component `UP` / PostgreSQL |
| Negative path | Stopped only the assignment database; HTTP **503**, `status=DOWN`, `queryExecuted=false` |
| Recovery | Restarted the same database with its volume intact; HTTP **200** and SQL result `1`, without restarting backend |
| Resource preservation | Compared pre/post Docker metadata; all 29 existing containers, 9 networks, 40 volumes, and 25 images preserved; host PostgreSQL PID and global Git identity unchanged |

Actual final controlled test timestamps:

- Positive: `2026-11-10T18:12:53.305136233Z`.
- Negative: `2026-11-10T18:12:53.618527137Z`.
- Recovery: `2026-11-10T18:12:59.085847373Z`.

Actual backend log excerpts:

```text
2026-11-10T18:11:39.447Z ... StoreXPool - Added connection org.postgresql.jdbc.PgConnection@615b5480
2026-11-10T18:11:39.451Z ... StoreXPool - Start completed.
2026-11-10T18:12:53.615Z ... DB_CHECK FAILURE type=DataAccessResourceFailureException
2026-11-10T18:12:59.085Z ... DB_CHECK SUCCESS database=storex queryResult=1 server=172.24.0.2/32:5432
```

The excerpts abbreviate logger/thread fields with `...`; the screenshots show
genuine terminal log output. IPs above are observations, not configuration.
The first recovery attempt in an earlier test was made too soon and returned
503; a later request succeeded. The final integration test waited for readiness
and application-level recovery and passed. Expected connection warnings during
the deliberate outage are not concealed as normal startup output.

New persistent runtime resources are only this project's two containers,
default network, named volume, and the backend/PostgreSQL images. Docker build
cache is also retained. No temporary test containers were needed. Integration
and audit helpers/results were kept outside the repository in `/tmp/opencode/`.

## Evidence

Authentic, unmodified 2560×1440 PNGs were captured from Foot on Hyprland with
`grim`. PNG decoding/integrity and local OCR confirmed the captured terminal
content. They are distributed as assets of
[GitHub Release ss04-bai-01](https://github.com/ng-trng-phcc/md05_ss04_bai_01/releases/tag/ss04-bai-01),
rather than committed as source files:

- [evidence_01.png](https://github.com/ng-trng-phcc/md05_ss04_bai_01/releases/download/ss04-bai-01/evidence_01.png):
  Compose status, HTTP 200 real SQL result, and successful datasource/query logs.
- [evidence_01_details.png](https://github.com/ng-trng-phcc/md05_ss04_bai_01/releases/download/ss04-bai-01/evidence_01_details.png):
  DNS, Actuator, JDBC sessions, absent database host-port binding, and recorded
  controlled failure/recovery results.

SHA-256:

```text
2fa414135597ef0e63c9e32b3aa495e95935636592a5b66d098778f09e5213fe  evidence_01.png
d14776e37d512904bb8bc40d38ae053706157e5d75f800a4010a499c07f68977  evidence_01_details.png
```

## Scope and limitations

This is an authorized functional demonstration, not the original StoreX
business application or a production deployment. It does not verify StoreX
business behavior. The lab retains the required `postgres:13` image and fixed
assignment credentials. No claim of production readiness is made.

Keep the verified stack running. If intentionally stopping it later, use
`docker compose stop`. Do not delete the database volume or run global Docker
cleanup commands.
