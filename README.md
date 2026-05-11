# mule-hikaricp-db-connector

High-performance HikariCP database connection pool plugin for Mule 4 applications.  
Drop-in replacement for C3P0 — **40-48% faster** with superior connection management.

Modeled after [mulesoft/mule-db-connector](https://github.com/mulesoft/mule-db-connector) but uses the `<object>` + `<db:data-source-connection>` pattern for Java 17 compatibility.

## Features

- **HikariCP 5.1.0** — fastest JVM connection pool
- **Java 17+** sealed interfaces, records, text blocks, pattern matching
- **Connection event logging** — acquisition time, slow warnings, error classification
- **Oracle error classification** — ORA-01017 (credentials), ORA-12505 (SID), connection resets
- **Pool health metrics** — active/idle/waiting connections, error counts, uptime
- **Leak detection** — configurable threshold with automatic logging
- **JMX MBeans** — GUI monitoring via JConsole/VisualVM
- **Keepalive heartbeat** — prevents idle connection staleness (HikariCP 5.x)
- **6 pre-configured pools** — Corp, ExtCorp, RDS, ExtRDS, PermRDS, ExtPermRDS
- **Zero API changes** — same `config-ref` names as va-common-framework

## Installation

Add to your API's `pom.xml`:

```xml
<dependency>
    <groupId>gov.va.bip.empwr</groupId>
    <artifactId>mule-hikaricp-db-connector</artifactId>
    <version>1.0.0</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

Add shared libraries to your `mule-maven-plugin` configuration:

```xml
<sharedLibrary>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
</sharedLibrary>
<sharedLibrary>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc8</artifactId>
</sharedLibrary>
```

## Configuration

Properties are resolved in order:
1. **System properties** (`-Ddb.host=xxx`)
2. **Environment variables** (`DB_HOST=xxx`) — K8s/Vault in production
3. **Classpath** `hikari-db.properties` — dev fallback

### Property Prefixes

| Pool | Prefix | Example Properties |
|------|--------|--------------------|
| Corp DB | `db` | `db.host`, `db.port`, `db.serviceName`, `db.user`, `db.password` |
| Ext Corp DB | `extDb` | `extDb.host`, `extDb.port`, ... |
| RDS | `rds` | `rds.host`, `rds.port`, ... |
| Ext RDS | `extrds` | `extrds.host`, `extrds.port`, ... |
| Perm RDS | `permrds` | `permrds.host`, `permrds.port`, ... |
| Ext Perm RDS | `extpermrds` | `extpermrds.host`, `extpermrds.port`, ... |

### Pool Tuning Properties

| Property | Default | Description |
|----------|---------|-------------|
| `{prefix}.poolingMin` | 5 | Minimum idle connections |
| `{prefix}.poolingMax` | 20 | Maximum pool size |
| `{prefix}.connectionTimeout` | 30000 | Connection acquire timeout (ms) |
| `{prefix}.idleTimeout` | 600000 | Idle connection timeout (ms) |
| `{prefix}.maxLifetime` | 1800000 | Max connection lifetime (ms) |
| `{prefix}.validationTimeout` | 5000 | Validation timeout (ms) |
| `{prefix}.leakDetectionThreshold` | 60000 | Leak detection threshold (ms) |
| `{prefix}.keepaliveTime` | 300000 | Keepalive heartbeat interval (ms) |
| `{prefix}.registerMbeans` | true | Register JMX MBeans |

## DB Config Names

These match existing va-common-framework names — no flow changes needed:

| Config Name | Pool |
|-------------|------|
| `Global_Database_Config` | Corp |
| `Global_EXT_Database_Config` | Ext Corp |
| `Global_RDS_Config` | RDS |
| `Global_EXT_RDS_Config` | Ext RDS |
| `Global_PERMRDS_Config` | Perm RDS |
| `Global_EXT_PERMRDS_Config` | Ext Perm RDS |

## Health Check Sub-Flows

Call from your API flows:

```xml
<!-- Test all pool connections -->
<flow-ref name="hikari-test-all-connections" />

<!-- Check specific pool health -->
<set-variable variableName="poolName" value="corp" />
<flow-ref name="hikari-health-pool" />
```

## Log4j Configuration

Add to your `log4j2.xml`:

```xml
<AsyncLogger name="com.zaxxer.hikari" level="INFO"/>
<AsyncLogger name="gov.va.bip.hikari.datasource" level="INFO"/>
<AsyncLogger name="gov.va.bip.hikari.health" level="INFO"/>
```

Set `gov.va.bip.hikari.datasource` to `DEBUG` for per-connection acquisition logs.

## Build

```bash
cd mule-hikaricp-db-connector
mvn clean install -DskipTests
```

## Performance

Tested against C3P0 with 25 queries across all 6 Oracle databases:

| Pool | HikariCP | C3P0 | Improvement |
|------|----------|------|-------------|
| Corp | 117ms avg | 198ms avg | **41% faster** |
| ExtCorp | 86ms | 155ms | **45% faster** |
| RDS | 97ms | 171ms | **43% faster** |
| ExtRDS | 93ms | 162ms | **43% faster** |
| PermRDS | 86ms | 166ms | **48% faster** |
| ExtPermRDS | 97ms | 161ms | **40% faster** |
