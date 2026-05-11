# HikariCP Database Connector — Plugin Guide

**Artifact:** `gov.va.bip.empwr:mule-hikaricp-db-connector:1.0.0`  
**Namespace:** `hikari-db` (`http://www.mulesoft.org/schema/mule/hikari-db`)  
**Mule Runtime:** 4.10.x+ | **Java:** 17+  
**HikariCP:** 5.1.0 | **Oracle Driver:** ojdbc8 19.24.0.0

---

## Table of Contents

1. [Why Replace C3P0?](#1-why-replace-c3p0)
2. [Migrating an Existing API (Step-by-Step)](#2-migrating-an-existing-api-step-by-step)
3. [New API Implementation](#3-new-api-implementation)
4. [Operations Reference](#4-operations-reference)
5. [Pool Health & Monitoring](#5-pool-health--monitoring)
6. [Error Handling](#6-error-handling)
7. [Logging Configuration](#7-logging-configuration)
8. [Performance Results](#8-performance-results)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Why Replace C3P0?

| Problem | C3P0 (current) | HikariCP Plugin |
|---------|-----------------|-----------------|
| **"Connection reset by peer"** in prod | No keepalive — idle connections go stale | Built-in keepalive heartbeat every 5 min |
| **Performance** | ~140 ms avg query latency | ~77 ms avg — **44% faster** |
| **Leak detection** | Manual — no built-in detection | Configurable threshold with auto-logging |
| **JMX monitoring** | Not available | Built-in MBeans for live pool metrics |
| **Connection validation** | `testConnectionOnCheckin/Checkout` (slow) | Lightweight `isValid()` + keepalive pings |
| **Pool exhaustion** | Silent failures | Thread-awaiting metrics + leak warnings |

---

## 2. Migrating an Existing API (Step-by-Step)

### Step 1: Add Plugin Dependency

Add to your API's `pom.xml` inside `<dependencies>`:

```xml
<dependency>
    <groupId>gov.va.bip.empwr</groupId>
    <artifactId>mule-hikaricp-db-connector</artifactId>
    <version>1.0.0</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

> **Note:** The plugin is a fat jar — HikariCP and the Oracle JDBC driver are shaded inside. No additional dependencies are required.

### Step 2: Add XML Namespace

In your `global.xml` (or any XML file using the connector), add the namespace declaration:

```xml
<mule xmlns:hikari-db="http://www.mulesoft.org/schema/mule/hikari-db"
      ...
      xsi:schemaLocation="
        http://www.mulesoft.org/schema/mule/hikari-db http://www.mulesoft.org/schema/mule/hikari-db/current/mule-hikari-db.xsd
        ...">
```

### Step 3: Replace DB Configs in `global.xml`

Replace each `<db:config>` with the equivalent `<hikari-db:config>`.

**BEFORE (C3P0 — va-common-framework):**

```xml
<db:config name="Global_Database_Config" doc:name="Corp Database Config">
    <db:oracle-connection host="${db.host}" user="${db.user}"
        password="${vault::secret/empwr/kubernetes/$[envVault]/db/corp-db.password}"
        port="${db.port}" serviceName="${db.serviceName}">
        <reconnection>
            <reconnect frequency="3000" count="3" />
        </reconnection>
        <db:pooling-profile maxPoolSize="${db.poolingMax}" minPoolSize="${db.poolingMin}"
            acquireIncrement="${db.poolIncrement}" preparedStatementCacheSize="${db.poolCache}"
            maxWait="${db.maxWait}" maxIdleTime="${db.maxIdleTime}">
            <db:additional-properties>
                <db:additional-property key="testConnectionOnCheckin" value="true" />
                <db:additional-property key="testConnectionOnCheckout" value="true" />
                <db:additional-property key="idleConnectionTestPeriod" value="300" />
                <db:additional-property key="preferredTestQuery" value="SELECT 1 FROM DUAL" />
                <db:additional-property key="maxIdleTimeExcessConnections" value="60" />
                <db:additional-property key="unresolvedAddresses" value="true" />
                <db:additional-property key="numHelperThreads" value="5" />
                <db:additional-property key="maxConnectionAge" value="3600" />
                <db:additional-property key="checkoutTimeout" value="30000" />
                <db:additional-property key="breakAfterAcquireFailure" value="false" />
            </db:additional-properties>
        </db:pooling-profile>
        <db:column-types>
            <!-- Oracle column type mappings... -->
        </db:column-types>
    </db:oracle-connection>
</db:config>
```

**AFTER (HikariCP Plugin):**

```xml
<hikari-db:config name="Global_Database_Config" doc:name="HikariCP Corp Database Config">
    <hikari-db:oracle-connection
        host="${db.host}"
        port="${db.port}"
        user="${db.user}"
        password="${vault::secret/empwr/kubernetes/$[envVault]/db/corp-db.password}"
        serviceName="${db.serviceName}"
        maxPoolSize="${db.poolingMax}"
        minIdle="${db.poolingMin}"
        connectionTimeout="${db.maxWait}"
        idleTimeout="${db.maxIdleTime}"
        poolName="hikari-corp" />
</hikari-db:config>
```

### Step 4: Repeat for All 6 Pools

| Config Name (keep same) | Pool Name | Property Prefix |
|--------------------------|-----------|-----------------|
| `Global_Database_Config` | `hikari-corp` | `db.*` |
| `Global_EXT_Database_Config` | `hikari-extcorp` | `extDb.*` |
| `Global_RDS_Config` | `hikari-rds` | `rds.*` |
| `Global_EXT_RDS_Config` | `hikari-extrds` | `extrds.*` |
| `Global_PERMRDS_Config` | `hikari-permrds` | `permrds.*` |
| `Global_EXT_PERMRDS_Config` | `hikari-extpermrds` | `extpermrds.*` |

> **Critical:** Keep the same `name` attribute (e.g., `Global_Database_Config`). All existing `config-ref` references in your flows will continue to work without changes.

### Step 5: Remove C3P0 Configs

Delete the old `<db:config>` blocks and the `db:` namespace if no longer used.  
You can also remove the `db:pooling-profile` and `db:additional-properties` sections entirely — HikariCP handles all pooling internally.

### Step 6: No Flow Changes Required

Your existing flows using `<db:select>`, `<db:insert>`, `<db:update>`, `<db:delete>`, and `<db:stored-procedure>` continue to work as-is because the `config-ref` names are unchanged.

```xml
<!-- This flow works identically before and after migration -->
<db:select config-ref="Global_Database_Config" doc:name="Query Corp DB">
    <db:sql>SELECT * FROM PERSON WHERE PTCPNT_ID = :ptcpntId</db:sql>
    <db:input-parameters><![CDATA[#[{ 'ptcpntId': vars.ptcpntId }]]]></db:input-parameters>
</db:select>
```

### Step 7: Update Log4j2 Configuration

Add to `src/main/resources/log4j2.xml`:

```xml
<AsyncLogger name="com.zaxxer.hikari" level="INFO"/>
<AsyncLogger name="gov.va.bip.hikari.datasource" level="INFO"/>
<AsyncLogger name="gov.va.bip.hikari.health" level="INFO"/>
```

### Step 8: Build & Deploy

```bash
# Set Java 17 (required for build)
$env:JAVA_HOME = "C:\AnypointStudio\plugins\org.mule.tooling.jdk.win32.x86_64_1.4.1"
mvn clean package -DskipTests
```

### Migration Checklist

- [ ] Added `mule-hikaricp-db-connector` dependency to `pom.xml`
- [ ] Added `hikari-db` namespace to XML files
- [ ] Replaced all 6 `<db:config>` with `<hikari-db:config>` (same `name`)
- [ ] Removed old C3P0 `<db:config>` blocks
- [ ] Added HikariCP loggers to `log4j2.xml`
- [ ] Verified existing `config-ref` references unchanged
- [ ] Tested locally in Anypoint Studio
- [ ] Deployed to dev environment

---

## 3. New API Implementation

For a new API, start with HikariCP from the beginning.

### Step 1: Add Dependencies to `pom.xml`

```xml
<dependency>
    <groupId>gov.va.bip.empwr</groupId>
    <artifactId>mule-hikaricp-db-connector</artifactId>
    <version>1.0.0</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

### Step 2: Create `global.xml` with HikariCP Configs

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns:hikari-db="http://www.mulesoft.org/schema/mule/hikari-db"
      xmlns:http="http://www.mulesoft.org/schema/mule/http"
      xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="
        http://www.mulesoft.org/schema/mule/hikari-db http://www.mulesoft.org/schema/mule/hikari-db/current/mule-hikari-db.xsd
        http://www.mulesoft.org/schema/mule/http http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd
        http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd">

    <configuration-properties file="configurations/${env}.yaml" />

    <http:listener-config name="HTTP_Listener_Config">
        <http:listener-connection host="0.0.0.0" port="${http.port}" />
    </http:listener-config>

    <!-- Corp DB -->
    <hikari-db:config name="Global_Database_Config" doc:name="HikariCP Corp DB">
        <hikari-db:oracle-connection
            host="${db.host}" port="${db.port}"
            user="${db.user}" password="${db.password}"
            serviceName="${db.serviceName}"
            maxPoolSize="${db.poolingMax}" minIdle="${db.poolingMin}"
            connectionTimeout="${db.maxWait}" idleTimeout="${db.maxIdleTime}"
            poolName="hikari-corp" />
    </hikari-db:config>

    <!-- Ext Corp DB -->
    <hikari-db:config name="Global_EXT_Database_Config" doc:name="HikariCP Ext Corp DB">
        <hikari-db:oracle-connection
            host="${extDb.host}" port="${extDb.port}"
            user="${extDb.user}" password="${extDb.password}"
            serviceName="${extDb.serviceName}"
            maxPoolSize="${extDb.poolingMax}" minIdle="${extDb.poolingMin}"
            connectionTimeout="${extDb.maxWait}" idleTimeout="${extDb.maxIdleTime}"
            poolName="hikari-extcorp" />
    </hikari-db:config>

    <!-- RDS -->
    <hikari-db:config name="Global_RDS_Config" doc:name="HikariCP RDS">
        <hikari-db:oracle-connection
            host="${rds.host}" port="${rds.port}"
            user="${rds.user}" password="${rds.password}"
            serviceName="${rds.serviceName}"
            maxPoolSize="${rds.poolingMax}" minIdle="${rds.poolingMin}"
            connectionTimeout="${rds.maxWait}" idleTimeout="${rds.maxIdleTime}"
            poolName="hikari-rds" />
    </hikari-db:config>

    <!-- Ext RDS -->
    <hikari-db:config name="Global_EXT_RDS_Config" doc:name="HikariCP Ext RDS">
        <hikari-db:oracle-connection
            host="${extrds.host}" port="${extrds.port}"
            user="${extrds.user}" password="${extrds.password}"
            serviceName="${extrds.serviceName}"
            maxPoolSize="${extrds.poolingMax}" minIdle="${extrds.poolingMin}"
            connectionTimeout="${extrds.maxWait}" idleTimeout="${extrds.maxIdleTime}"
            poolName="hikari-extrds" />
    </hikari-db:config>

    <!-- PERMRDS -->
    <hikari-db:config name="Global_PERMRDS_Config" doc:name="HikariCP PERMRDS">
        <hikari-db:oracle-connection
            host="${permrds.host}" port="${permrds.port}"
            user="${permrds.user}" password="${permrds.password}"
            serviceName="${permrds.serviceName}"
            maxPoolSize="${permrds.poolingMax}" minIdle="${permrds.poolingMin}"
            connectionTimeout="${permrds.maxWait}" idleTimeout="${permrds.maxIdleTime}"
            poolName="hikari-permrds" />
    </hikari-db:config>

    <!-- Ext PERMRDS -->
    <hikari-db:config name="Global_EXT_PERMRDS_Config" doc:name="HikariCP Ext PERMRDS">
        <hikari-db:oracle-connection
            host="${extpermrds.host}" port="${extpermrds.port}"
            user="${extpermrds.user}" password="${extpermrds.password}"
            serviceName="${extpermrds.serviceName}"
            maxPoolSize="${extpermrds.poolingMax}" minIdle="${extpermrds.poolingMin}"
            connectionTimeout="${extpermrds.maxWait}" idleTimeout="${extpermrds.maxIdleTime}"
            poolName="hikari-extpermrds" />
    </hikari-db:config>

</mule>
```

### Step 3: Use HikariCP Plugin Operations in Flows

The plugin provides its own operations under the `hikari-db:` namespace. Use these for **direct HikariCP operations** (JSON responses, built-in pool management):

```xml
<!-- Select (returns JSON) -->
<hikari-db:select config-ref="Global_Database_Config" sql="SELECT * FROM PERSON WHERE PTCPNT_ID = :ptcpntId"
    inputParameters="#[{ 'ptcpntId': vars.ptcpntId }]" />

<!-- Insert (returns affected row count) -->
<hikari-db:insert config-ref="Global_Database_Config"
    sql="INSERT INTO AUDIT_LOG (ACTION, TS) VALUES (:action, SYSDATE)"
    inputParameters="#[{ 'action': 'LOGIN' }]" />

<!-- Update -->
<hikari-db:update config-ref="Global_Database_Config"
    sql="UPDATE PERSON SET STATUS = :status WHERE PTCPNT_ID = :id"
    inputParameters="#[{ 'status': 'ACTIVE', 'id': vars.ptcpntId }]" />

<!-- Delete -->
<hikari-db:delete config-ref="Global_Database_Config"
    sql="DELETE FROM TEMP_TABLE WHERE CREATED_DATE &lt; SYSDATE - 30" />

<!-- Query Single (returns one row as JSON object) -->
<hikari-db:query-single config-ref="Global_Database_Config"
    sql="SELECT COUNT(*) AS total FROM PERSON"  />

<!-- Stored Procedure -->
<hikari-db:stored-procedure config-ref="Global_Database_Config"
    sql="{ CALL GET_PERSON_INFO(:ptcpntId) }"
    inputParameters="#[{ 'ptcpntId': vars.ptcpntId }]" />
```

> **Important:** `inputParameters` and `bulkInputParameters` must be **attributes** on the operation element, not child elements.

### Step 4: Add Bulk Operations

```xml
<!-- Bulk Insert -->
<hikari-db:bulk-insert config-ref="Global_Database_Config"
    sql="INSERT INTO AUDIT_LOG (ACTION, USER_ID) VALUES (:action, :userId)"
    bulkInputParameters="#[payload map { 'action': $.action, 'userId': $.userId }]" />

<!-- Bulk Update -->
<hikari-db:bulk-update config-ref="Global_Database_Config"
    sql="UPDATE PERSON SET STATUS = :status WHERE PTCPNT_ID = :id"
    bulkInputParameters="#[payload map { 'status': $.newStatus, 'id': $.ptcpntId }]" />

<!-- Bulk Delete -->
<hikari-db:bulk-delete config-ref="Global_Database_Config"
    sql="DELETE FROM TEMP_TABLE WHERE ID = :id"
    bulkInputParameters="#[payload map { 'id': $.id }]" />
```

### Step 5: Add DDL Operations (if needed)

```xml
<!-- Execute DDL -->
<hikari-db:execute-ddl config-ref="Global_Database_Config"
    sql="CREATE INDEX idx_person_name ON PERSON(LAST_NAME)" />

<!-- Execute Script (multiple statements separated by ;) -->
<hikari-db:execute-script config-ref="Global_Database_Config"
    sql="CREATE TABLE temp1 (id NUMBER); CREATE TABLE temp2 (id NUMBER)" />
```

### Step 6: Add Health Check Endpoint

See [Section 5: Pool Health & Monitoring](#5-pool-health--monitoring).

---

## 4. Operations Reference

### DML Operations

| Operation | XML Element | Returns | Description |
|-----------|-------------|---------|-------------|
| **Select** | `<hikari-db:select>` | JSON array of rows | Execute SELECT, return all matching rows |
| **Query Single** | `<hikari-db:query-single>` | JSON object (one row) | Execute SELECT, return first row only |
| **Insert** | `<hikari-db:insert>` | `int` (affected rows) | Execute INSERT statement |
| **Update** | `<hikari-db:update>` | `int` (affected rows) | Execute UPDATE statement |
| **Delete** | `<hikari-db:delete>` | `int` (affected rows) | Execute DELETE statement |
| **Stored Procedure** | `<hikari-db:stored-procedure>` | JSON with resultSet + updateCount | Call Oracle stored procedure |

### Bulk Operations

| Operation | XML Element | Returns | Description |
|-----------|-------------|---------|-------------|
| **Bulk Insert** | `<hikari-db:bulk-insert>` | `int[]` (per-row counts) | Batch INSERT with multiple parameter sets |
| **Bulk Update** | `<hikari-db:bulk-update>` | `int[]` (per-row counts) | Batch UPDATE with multiple parameter sets |
| **Bulk Delete** | `<hikari-db:bulk-delete>` | `int[]` (per-row counts) | Batch DELETE with multiple parameter sets |

### DDL Operations

| Operation | XML Element | Returns | Description |
|-----------|-------------|---------|-------------|
| **Execute DDL** | `<hikari-db:execute-ddl>` | `void` | Execute CREATE, ALTER, DROP |
| **Execute Script** | `<hikari-db:execute-script>` | `void` | Execute multiple `;`-separated statements |

### Pool Management Operations

| Operation | XML Element | Returns | Description |
|-----------|-------------|---------|-------------|
| **Pool Health** | `<hikari-db:pool-health>` | JSON | Pool config + active/idle/total connections |
| **Pool Stats** | `<hikari-db:pool-stats>` | JSON | Detailed stats + connection validation |

### Common Attributes

| Attribute | Required | Description |
|-----------|----------|-------------|
| `config-ref` | Yes | Reference to `<hikari-db:config>` name |
| `sql` | Yes | SQL text with `:paramName` named parameters |
| `inputParameters` | No | DataWeave expression returning `Map<String, Object>` |
| `bulkInputParameters` | No | DataWeave expression returning `List<Map<String, Object>>` |

---

## 5. Pool Health & Monitoring

### Recommended: Add Health Check Endpoint to Every API

Create a dedicated health flow to monitor pool status in production:

```xml
<flow name="hikari-health-check-flow">
    <http:listener config-ref="HTTP_Listener_Config"
        path="/api/health/db" allowedMethods="GET" />

    <set-variable variableName="healthResults" value="#[[]]" />

    <!-- Check Corp DB -->
    <try>
        <hikari-db:pool-health config-ref="Global_Database_Config" />
        <set-variable variableName="corpHealth" value="#[payload]" />
    </try>

    <!-- Check RDS -->
    <try>
        <hikari-db:pool-health config-ref="Global_RDS_Config" />
        <set-variable variableName="rdsHealth" value="#[payload]" />
    </try>

    <set-payload value="#[output application/json --- {
        status: 'UP',
        timestamp: now(),
        pools: {
            corp: vars.corpHealth default 'UNAVAILABLE',
            rds: vars.rdsHealth default 'UNAVAILABLE'
        }
    }]" />
</flow>
```

### Pool Health Response Fields

`<hikari-db:pool-health>` returns:

```json
{
    "poolName": "hikari-corp",
    "jdbcUrl": "jdbc:oracle:thin:@//hostname:1521/SERVICENAME",
    "username": "APP_USER",
    "maximumPoolSize": 20,
    "minimumIdle": 5,
    "connectionTimeout": 30000,
    "idleTimeout": 600000,
    "maxLifetime": 1800000,
    "leakDetectionThreshold": 60000,
    "isClosed": false,
    "activeConnections": 2,
    "idleConnections": 8,
    "totalConnections": 10,
    "threadsAwaitingConnection": 0,
    "status": "RUNNING"
}
```

### Pool Stats Response Fields

`<hikari-db:pool-stats>` returns:

```json
{
    "poolName": "hikari-corp",
    "activeConnections": 2,
    "idleConnections": 8,
    "totalConnections": 10,
    "threadsAwaitingConnection": 0,
    "connectionValid": true,
    "javaVersion": "17.0.12",
    "poolType": "HikariCP 5.1.0"
}
```

### Key Metrics to Monitor

| Metric | Healthy | Warning | Action |
|--------|---------|---------|--------|
| `activeConnections` | < 80% of max | > 80% of max | Increase `maxPoolSize` or check for leaks |
| `threadsAwaitingConnection` | 0 | > 0 | Pool exhaustion — increase pool or fix leaks |
| `idleConnections` | > 0 | 0 | All connections in use — increase `minIdle` |
| `totalConnections` | = `maxPoolSize` in steady state | Fluctuating wildly | Check `maxLifetime` / `idleTimeout` |
| `connectionValid` | `true` | `false` | DB connectivity issue — check network/firewall |
| `isClosed` | `false` | `true` | Pool was shut down — restart app |

### JMX Monitoring

HikariCP registers JMX MBeans by default (`registerMbeans="true"`). Connect via JConsole or VisualVM:

**MBean path:** `com.zaxxer.hikari:type=Pool (hikari-corp)`

Available metrics:
- `ActiveConnections` — currently in use
- `IdleConnections` — available in pool
- `TotalConnections` — active + idle
- `ThreadsAwaitingConnection` — blocked threads waiting
- Pool operations: `softEvictConnections()`, `suspendPool()`, `resumePool()`

---

## 6. Error Handling

The plugin defines 9 error types for targeted error handling:

```xml
<try>
    <hikari-db:select config-ref="Global_Database_Config"
        sql="SELECT * FROM PERSON WHERE ID = :id"
        inputParameters="#[{ 'id': vars.personId }]" />

    <error-handler>
        <on-error-continue type="HIKARI-DB:CONNECTIVITY">
            <logger message="Database unreachable — failover or retry" level="ERROR" />
        </on-error-continue>

        <on-error-continue type="HIKARI-DB:INVALID_CREDENTIALS">
            <logger message="Bad credentials — check Vault secret rotation" level="ERROR" />
        </on-error-continue>

        <on-error-continue type="HIKARI-DB:POOL_EXHAUSTED">
            <logger message="All connections in use — increase maxPoolSize" level="ERROR" />
        </on-error-continue>

        <on-error-continue type="HIKARI-DB:CONNECTION_TIMEOUT">
            <logger message="Timed out waiting for connection" level="WARN" />
        </on-error-continue>

        <on-error-continue type="HIKARI-DB:QUERY_EXECUTION">
            <logger message="SQL execution error: #[error.description]" level="ERROR" />
        </on-error-continue>

        <on-error-continue type="HIKARI-DB:BAD_SQL_SYNTAX">
            <logger message="Invalid SQL: #[error.description]" level="ERROR" />
        </on-error-continue>

        <on-error-continue type="HIKARI-DB:CONNECTION_LEAK">
            <logger message="Connection leak detected!" level="ERROR" />
        </on-error-continue>
    </error-handler>
</try>
```

### Error Types Reference

| Error Type | Trigger | ORA Codes |
|------------|---------|-----------|
| `CONNECTIVITY` | Network failure, connection reset | ORA-12170, ORA-12514, ORA-03113 |
| `INVALID_CREDENTIALS` | Wrong username/password | ORA-01017 |
| `INVALID_DATABASE` | Bad SID/service name | ORA-12505, ORA-12154 |
| `QUERY_EXECUTION` | Runtime SQL error | Any during execute |
| `BAD_SQL_SYNTAX` | Malformed SQL | ORA-00900 series |
| `POOL_EXHAUSTED` | All connections in use, timeout waiting | — |
| `CONNECTION_LEAK` | Connection held past threshold | — |
| `CONNECTION_TIMEOUT` | Acquire timeout exceeded | — |
| `UNKNOWN` | Uncategorized error | — |

---

## 7. Logging Configuration

### Recommended `log4j2.xml` Configuration

```xml
<Loggers>
    <!-- HikariCP internal pool logging -->
    <AsyncLogger name="com.zaxxer.hikari" level="INFO" additivity="false">
        <AppenderRef ref="Console" />
    </AsyncLogger>

    <!-- Plugin connection lifecycle events -->
    <AsyncLogger name="gov.va.bip.hikari.datasource" level="INFO" additivity="false">
        <AppenderRef ref="Console" />
    </AsyncLogger>

    <!-- Pool health check logging -->
    <AsyncLogger name="gov.va.bip.hikari.health" level="INFO" additivity="false">
        <AppenderRef ref="Console" />
    </AsyncLogger>
</Loggers>
```

### Log Levels

| Level | What You See |
|-------|-------------|
| `INFO` | Pool initialization, shutdown, connection errors |
| `DEBUG` | Per-connection acquisition time, validation results |
| `WARN` | Slow connection acquisition, leak detection warnings |
| `ERROR` | Connection failures, pool exhaustion, ORA errors |

### Example Log Output

```
INFO  [hikari-corp] HikariCP pool initialized: jdbc:oracle:thin:@//dbhost:1521/CORPDB (pool: 5-20)
WARN  [hikari-corp] Connection leak detection triggered (held for 65000ms)
INFO  [hikari-corp] Pool stats — active: 3, idle: 7, total: 10, waiting: 0
```

---

## 8. Performance Results

Tested on 2026-05-11 with 25 `SELECT 1 FROM dual` queries per pool:

| Pool | HikariCP | C3P0 | Improvement |
|------|:--------:|:----:|:-----------:|
| Corp | 2,418 ms | 3,860 ms | **37.4%** |
| Ext Corp | 1,897 ms | 3,510 ms | **46.0%** |
| RDS | 1,916 ms | 3,465 ms | **44.7%** |
| Ext RDS | 1,921 ms | 3,751 ms | **48.8%** |
| PERMRDS | 1,968 ms | 3,483 ms | **43.5%** |
| Ext PERMRDS | 1,872 ms | 3,541 ms | **47.1%** |
| **Average** | **1,999 ms** | **3,602 ms** | **~44.6%** |

- **P50 latency:** HikariCP ~68 ms vs C3P0 ~135 ms
- **Zero failures** across all 300 queries (150 per pool type)
- **Full report:** See `db_connection_test/PERFORMANCE_REPORT.md`

---

## 9. Troubleshooting

### "Connection reset by peer" in production

This is the original problem that prompted the migration. HikariCP fixes it with:
- **Keepalive heartbeat** (`keepaliveTime=300000`) — pings idle connections every 5 min
- **Max lifetime** (`maxLifetime=1800000`) — recycles connections before the server drops them
- **Idle timeout** (`idleTimeout=600000`) — removes stale idle connections

### Pool exhaustion (all connections busy)

1. Check `threadsAwaitingConnection` via pool health endpoint
2. Look for leak detection warnings in logs
3. Ensure all DB streams are consumed (especially in `foreach` loops)
4. Increase `maxPoolSize` if legitimate load

### Anypoint Studio — connector not visible

1. Ensure `pom.xml` has the dependency with `<classifier>mule-plugin</classifier>`
2. Run `mvn clean package -DskipTests` to refresh
3. Check Studio's "Mule Palette" — search for "HikariCP"

### Build errors with Java version

The plugin requires Java 17. Set before building:

```powershell
$env:JAVA_HOME = "C:\AnypointStudio\plugins\org.mule.tooling.jdk.win32.x86_64_1.4.1"
mvn clean install -DskipTests
```

### Connection parameters reference

All parameters map to existing property prefixes. No new properties required:

| HikariCP Parameter | Maps From | Default |
|--------------------|-----------|---------|
| `host` | `${prefix.host}` | — |
| `port` | `${prefix.port}` | `1521` |
| `user` | `${prefix.user}` | — |
| `password` | `${prefix.password}` or Vault | — |
| `serviceName` | `${prefix.serviceName}` | — |
| `maxPoolSize` | `${prefix.poolingMax}` | `20` |
| `minIdle` | `${prefix.poolingMin}` | `5` |
| `connectionTimeout` | `${prefix.maxWait}` | `30000` |
| `idleTimeout` | `${prefix.maxIdleTime}` | `600000` |
| `maxLifetime` | — | `1800000` |
| `leakDetectionThreshold` | — | `60000` |
| `keepaliveTime` | — | `300000` |
| `connectionTestQuery` | — | `SELECT 1 FROM dual` |
| `poolName` | — | `hikari-pool` |
| `registerMbeans` | — | `true` |

---

*mule-hikaricp-db-connector v1.0.0 — May 2026*
