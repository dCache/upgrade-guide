# The ultimate golden release upgrade guide

### Community effort to write an upgrade guide for dCache 11.2

How to get from dCache 10.2 to dCache 11.2

## Executive summary

### The highlights in 11.2 compared with 10.2

- NFSv4.1 read-delegation support improving file access latency for HPC workloads
- NFS read performance improvement via zero-copy I/O — up to 20% throughput gain
- Hot file replication where pools autonomously replicate popular files
- Kafka logging moved to non-blocking Billing service for stability
- Integrated Prometheus exporter for metrics
- Java 21 support alongside Java 17

## Breaking changes

### WARNING — Incompatibility between 10.2 and 11.2

You must upgrade the entire dCache instance at once. Release 11.0 removed backward compatibility with pre-11.0 message encoding, preventing mixed-version operations. Stop all services, upgrade all packages, run database migrations, and restart together.

Database schema migrations cannot be rolled back. Liquibase 4 migrations applied in 11.0 and later are irreversible. Always take complete database backups before upgrading.

### General incompatibilities (from 9.2)

- XRootD `prepare` request behavior changed.
- NFSv4.1 files layout type removed.
- File flags removed.
- `pool.enable.hsm-flag` obsolete.
- `t_storageinfo` no longer populated.
- Legacy admin shell removed.
- Pool Manager partitions now allow staging by default.
- ARGUS plugin removed.
- Access to HSM-only files denied.
- Stage cancellation propagated to pool.

## NFS

NFSv4.1 read-delegation is now supported, allowing the server to delegate OPEN-CLOSE operations to clients for files accessed multiple times. This improves latency for HPC workloads with repetitive read patterns. To enable this feature, the `nfsv4_1_files` layout type was removed. Before upgrading, remove `lt=nfsv4_1_files` from NFS export files if present. After the upgrade, NFS movers may remain active longer because clients retain delegations. Billing records now reflect delegation periods rather than individual `open→close` cycles. [11.1]

```
# Old (must be removed):
/data  *(rw,lt=nfsv4_1_files)

# New (correct):
/data  *(rw)
```

The NFS mover now uses zero-copy I/O for READ operations, eliminating unnecessary memory copies. This provides up to 20% throughput improvement for data-intensive workloads. No configuration changes required. [11.2]

When Proxy-IO is used, the NFS door now passes original request credentials to pools instead of the door's credentials, fixing permission mismatches in multi-user environments. [11.2]

Files with labels can be browsed through virtual read-only directories using `.(collection)` path syntax. Access files grouped by label independent of physical location. The WebDAV equivalent is `http://<host>:<port>/.(collection)(labelName)/`. [10.2, 11.0, 11.1]

```bash
ls -l /dcache/".(collection)"/                    # List all labels
ls -l /dcache/".(collection)"/AnnaBerthaLudwig    # List files with label
cat /dcache/".(collection)"/report.txt-9          # Access specific file
```

The NFS door exposes user and group quota information via the remote quota protocol (rquota over UDP). Configure the port with `nfs.net.rquota.port`. [10.2]

---

## Pool

Pools now autonomously detect and replicate hot files when in-flight transfers exceed a configurable threshold. The pool triggers migration to create replicas on other pools in the same group. Enable with `pool.hotfile.monitoring.enable=true`. Tune parameters via `\s <pool> hotfile` admin commands. This is particularly valuable for workloads accessing shared calibration files or common datasets. [11.2]

```properties
pool.hotfile.monitoring.enable=true
```

The `set max diskspace` command now accepts percentages of total disk space, making heterogeneous pool configurations easier. Pools also automatically shrink declared maximum space if the underlying filesystem shrinks. [11.0]

```
\s <pool> set max diskspace 90%
```

The migration module can now spread files to a desired number of pools with `wait` or `limit` strategies when insufficient pools are online. This provides better control during maintenance and rebalancing. [11.0]

Berkeley DB open-file limit increased from 100 to 512, significantly improving startup times for pools with large repositories. No action required unless previously customized. [11.1]

Remove `pool.enable.hsm-flag` from configuration files — it is obsolete and fully removed. [11.1]

```properties
# Remove this line:
# pool.enable.hsm-flag=false
```

Pools can calculate multiple checksums simultaneously on HTTP upload or TPC-pull if requested by clients. No configuration needed. [11.0]

The `sweeper purge` command accepts `-storageClass=<class>` to selectively purge cached files. The `pool.mover.nfs.multipath` property specifies IP addresses advertised to pNFS clients for multi-homed hosts. The `pp ls` command now shows source pool names during transfers. [10.2]

```
\s <pool> sweeper purge -storageClass=atlas:default
```

---

## Pool Manager

When a pool comes back online during an active stage request, PoolManager now retries the request to use the pool copy instead of waiting for tape restore. Clients receive data from whichever source becomes ready first (tape or pool). This improves user experience during maintenance windows. [10.2, 11.0]

The `stage-allowed` partition option now defaults to `yes`. Partitions without explicit settings will allow staging. Add `stage-allowed=no` to partition definitions if you need the old blocking behavior. [10.2]

Stage cancellations in PoolManager now propagate to pools, preventing orphaned HSM restore operations and saving tape bandwidth. [10.2]

---

## Chimera / Namespace

File flags functionality is removed. The `t_level_2` table is no longer read or written but remains in the database. Remove any scripts using `set flags`, `get flags`, or `delete flags` commands. Optionally truncate the table to reclaim space. [11.1]

```sql
TRUNCATE t_level_2;
```

The `t_storageinfo` table is no longer populated or read. Storage info is determined from directory tags only. The table can be truncated if desired. [11.1]

```sql
TRUNCATE t_storageinfo;
```

The QoS database name is now configurable with `qos.db.name`, allowing consolidation of multiple instances or compliance with naming conventions. [10.2, 11.1]

```properties
qos.db.name=my_qos_db
```

TransferManagers no longer use database back-ends. They operate entirely in-memory or with file-based configuration. [10.2, 11.0]

---

## Cleaner

The `cleaner-hsm` can now delete files in parallel via multiple pools per HSM instead of sequentially through one pool. Decrease `cleaner-hsm.period` to increase parallelism. This significantly improves throughput for sites with heavy tape delete workloads. [11.1]

```properties
cleaner-hsm.period=60
cleaner-hsm.period.unit=SECONDS
```

As a reminder from 9.2: property prefixes changed from `cleaner.<something>` to `cleaner-disk.<something>` and `cleaner-hsm.<something>`. Check custom configurations accordingly.

---

## QoS

Files with QoS `HSM`-only are denied access even if cached on disk, enforcing intended policy lifecycle. Stage files to disk before reading or adjust policies to allow temporary caching. [10.2]

The QoS database name is now configurable. [10.2]

QoS skips unpinning unless performed by the scanner to avoid race conditions during migrations. This prevents unexpected file removal during active transfers. [10.2, 11.0, 11.1]

---

## gPlazma

The `gplazma2-argus` plugin is removed. Migrate to `gplazma2-multimap` (recommended), `gplazma2-grid-mapfile`, or `gplazma2-vorolemap`. The multimap plugin supports role-based authorization and can express most ARGUS policies more simply. [10.2]

Python scripting support for `auth` and `map` plugins is now available via `gplazma2-pyscript` / `gplazma2-jython`. This enables rapid prototyping of custom authentication logic without Java compilation. [10.2]

gPlazma now consistently sets `RolePrincipal` when the `role` attribute is present, fixing inconsistent role enforcement across protocols. Verify gPlazma configuration after upgrading if you rely on roles for admin access. The frontend and dCacheView now use multimap instead of the old roles plugin. [10.2, 11.0]

The `gplazma2-voms` plugin now accepts only FQANs matching the configured VO name, preventing cross-VO authorization issues. Ensure each VO has its own plugin instance. [11.1]

The LDAP plugin supports more flexible search base DN configurations. [11.2]

---

## Billing / Monitoring

Kafka logging is consolidated into the non-blocking Billing service. Previously, Kafka producers in doors and pools could block under adverse conditions, causing service instability. Now, Kafka outages do not impact door or pool stability. Enable with `billing.enable.kafka=true`. Remove Kafka configuration from other services. [11.2]

```properties
billing.enable.kafka=true
```

Billing JSON records now include a `transferTag` field provided by XRootD and HTTP clients. This enables tracking of experiment-specific metadata (experiment ID, workflow ID) for monitoring and usage analysis. [11.2]

The integrated Prometheus exporter exposes JVM memory, thread counts, and open file handles. Enable with `dcache.enable.prometheus.exporter=true`. Configure endpoint with `dcache.enable.prometheus.exporter.endpoint` (default `localhost:9876`). [11.1]

```properties
dcache.enable.prometheus.exporter=true
dcache.enable.prometheus.exporter.endpoint=localhost:9876
```

---

## SciTags / Firefly

SciTag support was enhanced with experiment ID and activity tracking. Doors and pools propagate SciTag information from clients to flow markers for network monitoring. Configure with `pool.enable.firefly`, `pool.firefly.vo-mapping`, and `pool.firefly.excludes` properties. The `vo-mapping` property maps VO names to experiment IDs for automatic tagging. [10.2, 11.0]

```properties
pool.enable.firefly=false
pool.firefly.vo-mapping=atlas:2, cms:3, lsst:9
pool.firefly.excludes=a.b.c.d/16
```

---

## XRootD

XRootD `prepare` requests now return `Unsupported` instead of `OK`. This prevents clients from issuing `open` calls on files not yet staged from tape, which previously caused blocking and inefficient tape usage. Use the WLCG Tape REST API for staging with proper asynchronous status tracking. Update workflows before upgrading. [11.2]

```bash
# Old (no longer works):
xrdfs <host> prepare -s <file>

# Use instead:
# POST /api/v1/tape/stage
# GET /api/v1/tape/stage/<request-id>
```

The XRootD door correctly handles HAProxy when `xrootd.enable.proxy-protocol=true` is set, returning the destination address and handling the `checksum` command through the proxy. [10.2]

---

## Frontend / REST API

A request rate limiter protects against brute-force attacks. By default, rate limiting applies only to authentication errors, allowing legitimate traffic to flow freely. Configure limits per source IP if needed. [11.2]

The frontend supports OAuth2 authorization code flow for `dcache-view`, enabling browser-based authentication with federated identity providers. [11.2]

The REST API `id` resource is now admin-only. JSON validation was added to the migration request endpoint. [10.2, 11.2]

---

## Admin Interface

The legacy admin shell (using `cd` instead of `\c`) is removed. Update scripts to use standard `\c` syntax or direct `\s` commands. [11.1]

```bash
# Old (no longer works):
cd PnfsManager

# New:
\c PnfsManager
\s PnfsManager pnfsidof /path/to/file
```

File flags commands (`set flags`, `get flags`, `delete flags`) are removed from all components. Update any scripts using these commands. [11.1]

Java Flight Recorder now records door interactions with other cells and message round-trip durations for latency analysis. [11.0]

---

## Runtime Environment

Java 21 is now supported alongside Java 17. The minimum required version remains Java 17. [10.2]

The JVM option `UseCompressedOops` was removed from startup scripts. Remove it from custom configurations if present. [10.2, 11.0]

Liquibase was upgraded to version 4.29.2. This migration is irreversible. Back up all databases before running `dcache database update`. The SRM and spacemanager databases required a fix applied in 11.0.10 — ensure you're on 11.0.10 or later if encountering checksum mismatch errors. [11.0]

```bash
dcache database update
```

Jetty version numbers are no longer sent in HTTP response headers for security. [10.2, 11.0]

---

## Upgrade procedure

### 1. Pre-upgrade preparation

Back up all dCache databases (Chimera, billing, SRM, spacemanager, QoS). The Liquibase 4 migration is irreversible.

Audit configuration:
- Remove `lt=nfsv4_1_files` from NFS exports (required for 11.1+)
- Remove `pool.enable.hsm-flag` from pool configuration (required for 11.1+)
- Remove scripts using file flags commands
- Update scripts using legacy admin shell `cd` syntax
- Review XRootD staging workflows using `prepare` (breaks in 11.2)
- Migrate from ARGUS to multimap/grid-mapfile/vorolemap if used
- Review partition `stage-allowed` settings if relying on old default

### 2. Stop dCache

Stop the entire instance. Rolling upgrades are not supported.

```bash
dcache stop
```

### 3. Install packages

Install dCache 11.2 packages and verify version.

```bash
yum upgrade dcache       # RPM-based
apt-get upgrade dcache   # DEB-based
dcache version
```

### 4. Run database migrations

Apply Liquibase 4 migrations. This cannot be undone.

```bash
dcache database update
```

### 5. Review configuration

Verify NFS exports and pool configuration are updated. Consider enabling:
- Hot file replication: `pool.hotfile.monitoring.enable=true`
- Prometheus exporter: `dcache.enable.prometheus.exporter=true`
- Kafka logging: `billing.enable.kafka=true`

### 6. Start dCache

Start and monitor startup logs carefully.

```bash
dcache start
tail -f /var/log/dcache/dcache.log
```

### 7. Post-upgrade verification

Test file operations through all doors. Verify stage requests work with Tape REST API. Check billing records, Kafka logging, and Prometheus metrics if enabled.

Optionally reclaim space:

```sql
-- In Chimera database:
TRUNCATE t_level_2;
TRUNCATE t_storageinfo;
```
