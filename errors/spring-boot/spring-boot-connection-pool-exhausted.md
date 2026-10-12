---
title: "Spring Boot Connection Pool Exhausted"
slug: spring-boot-connection-pool-exhausted
technologies: [spring-boot, hikaricp, jdbc]
severity: high
tags: [spring-boot, database, connection-pool, hikaricp, production]
related: [postgresql-too-many-connections, mysql-too-many-connections]
last_reviewed: 2026-10-12
---

# Spring Boot Connection Pool Exhausted

## Error Message

```text
org.springframework.jdbc.CannotGetJdbcConnectionException: Failed to obtain JDBC Connection
```

```text
HikariPool-1 - Exception during pool initialization.
java.sql.SQLException: HikariPool-1 - Connection is not available, request timed out after 30000ms.
```

```text
Caused by: java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30001ms.
```

## Description

The application's HikariCP connection pool (default in Spring Boot) has exhausted all available database connections. New connection requests are queuing and timing out because all connections are either in use or have leaked. This typically manifests as 503 errors in API responses, slow response times, or complete service unavailability.

## Technologies

- Spring Boot (2.x/3.x)
- HikariCP (default connection pool)
- JDBC
- PostgreSQL / MySQL / any relational database

## Severity

**high** — application cannot serve database-dependent requests. If this is a critical service, it may cause partial or full outage for dependent services. Response times degrade as requests queue for connections.

## Common Causes

1. Connection leaks: application code obtains a connection but never closes it (missing `try-with-resources`, improper exception handling).
2. Connection pool size too small for concurrent load (default is 10).
3. Long-running queries/transactions holding connections longer than expected.
4. Database itself has reached its maximum connection limit (`max_connections`).
5. Misconfigured connection timeout (`connectionTimeout`) or pool validation settings.
6. Connection borrowed but never returned due to deadlocks or infinite loops in application logic.

## Root Cause Analysis

HikariCP maintains a fixed-size pool of connections. When a request for a connection comes in:
- If an idle connection exists, it is returned immediately.
- If all connections are in use, the request waits up to `connectionTimeout` (default 30s).
- If no connection becomes available within the timeout, the exception is thrown.

Connection leaks occur when the application fails to call `close()` on a connection (or let the `try-with-resources` block do it). The pool still counts the connection as "in use" indefinitely, slowly starving the pool. Leaked connections are only reclaimed when the pool is closed or the connection is explicitly killed by the database server.

## Diagnostic Commands

```bash
# Check Spring Boot actuator metrics for pool usage (if actuator is enabled)
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
curl http://localhost:8080/actuator/metrics/hikaricp.connections.idle
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending

# Check database server current connections (PostgreSQL)
psql -U postgres -c "SELECT count(*) FROM pg_stat_activity WHERE datname = 'your_db';"

# Check database max_connections setting (PostgreSQL)
psql -U postgres -c "SHOW max_connections;"

# List long-running queries that may be holding connections (PostgreSQL)
psql -U postgres -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query FROM pg_stat_activity WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';"

# Kill specific long-running or idle connection (PostgreSQL) - USE WITH CAUTION
psql -U postgres -c "SELECT pg_terminate_backend(<pid>);"

# Java thread dump to see what threads are waiting for connections
jstack <pid> | grep -A 5 "HikariPool"
```

## Expected Results

**Healthy pool:**
```text
hikaricp.connections.active: 5
hikaricp.connections.idle: 5
hikaricp.connections.pending: 0
```

**Exhausted pool:**
```text
hikaricp.connections.active: 10 (at maximum)
hikaricp.connections.idle: 0
hikaricp.connections.pending: 15+ (requests waiting)
```

Database shows active connections equal to or exceeding pool size, and application logs show repeated timeout exceptions.

## Resolution

1. **Immediate relief (risky):** Increase pool size temporarily in `application.properties`:
   ```properties
   spring.datasource.hikari.maximum-pool-size=20
   ```
   This buys time but does not fix leaks.

2. **Find and fix connection leaks:**
   - Search code for `@Transactional` methods that never complete (infinite loops, deadlocks).
   - Ensure all JDBC operations use `try-with-resources` or `@Transactional` properly.
   - Review custom `DataSource` usage - never bypass the pool.

3. **Set appropriate pool size:**
   ```properties
   spring.datasource.hikari.maximum-pool-size=<CPU cores * 2 + effective disk count>
   spring.datasource.hikari.minimum-idle=5
   spring.datasource.hikari.connection-timeout=30000
   ```
   Common formula: `cores * 2 + 1` for CPU-bound, higher for I/O-bound workloads.

4. **Add connection validation:**
   ```properties
   spring.datasource.hikari.connection-test-query=SELECT 1
   spring.datasource.hikari.validation-timeout=5000
   spring.datasource.hikari.leak-detection-threshold=60000
   ```
   `leak-detection-threshold` logs warnings when connections are held longer than specified (helps identify leaks).

5. **If database max_connections is the limit:**
   - Increase `max_connections` in database config (requires restart).
   - Ensure pool size across all applications < database max_connections.

6. **Review long-running queries:**
   - Add indexes to slow queries.
   - Break large transactions into smaller ones.
   - Use batch operations instead of single-row loops.

## Validation

```bash
# After fixes, monitor pool metrics under load
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending
# Should remain at 0 or very low during normal traffic.

# Verify no connection leak warnings in logs after load test
# grep for "Leak detection" or "appears to have leaked a connection"
```

Application should handle traffic without queuing connection requests. Logs should show no timeout exceptions.

## Prevention

- Enable HikariCP leak detection in development/staging (`leak-detection-threshold`).
- Set up alerting on `hikaricp.connections.pending` metric (threshold: > 5 for > 1 minute).
- Use connection pool monitoring in production (Prometheus + Grafana dashboards).
- Code review: enforce `try-with-resources` pattern for manual JDBC operations.
- Load test with realistic concurrency to size the pool correctly before production.
- Set `maximum-pool-size` as a fraction of database `max_connections` (e.g., 20% per app).

## Related Errors

- [PostgreSQL Too Many Connections](../postgresql/postgresql-too-many-connections.md)
- [MySQL Too Many Connections](../mysql/mysql-too-many-connections.md)

## References

- [HikariCP Configuration Guide](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)
- [Spring Boot DataSource Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.sql.datasource.connection-pool)
- [Connection Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

## Tags

`spring-boot` · `hikaricp` · `connection-pool` · `database` · `production` · `performance`
