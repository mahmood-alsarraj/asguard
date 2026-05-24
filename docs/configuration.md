# Configuration & Options Reference

AsGuard is highly customizable. All features, performance structures, security systems, and storage settings are configured via `RequestLoggingOptions` in the `AddRequestLogging` dependency injection extension.

---

## Complete Options Reference

Here is the exhaustive reference of all properties available on `RequestLoggingOptions`:

| Option | Type | Default Value | Description |
| :--- | :--- | :--- | :--- |
| **DatabaseProvider** | `LoggingDatabaseProvider` | `LoggingDatabaseProvider.SqlServer` | The database provider to use for persisting logs. Supports `SqlServer`, `PostgreSql`, `Sqlite`, or `InMemory`. |
| **ConnectionString** | `string` | `""` | Connection string for SQL Server, PostgreSQL, or SQLite. Not used for InMemory. |
| **QueueCapacity** | `int` | `10_000` | Maximum number of log entries the in-memory request queue can hold. |
| **QueueOverflowPolicy** | `QueueOverflowPolicy` | `QueueOverflowPolicy.DropNewest` | Behavior when the request log queue reaches capacity (`DropNewest` or `DropOldest`). |
| **BatchSize** | `int` | `100` | Number of log entries flushed to the database in a single batch. |
| **EnableExceptionLogging** | `bool` | `true` | Enables/disables exception and log event persistence. |
| **EnableApmTracing** | `bool` | `true` | Enables/disables automatic APM tracing for database queries (EF Core) and downstream calls (HttpClient). |
| **EnableW3CPropagation** | `bool` | `true` | Enables/disables W3C standard traceparent header context propagation across HTTP boundaries. |
| **ExceptionQueueCapacity** | `int` | `5_000` | Maximum number of exception entries the in-memory queue can hold. |
| **ExceptionQueueOverflowPolicy** | `QueueOverflowPolicy` | `QueueOverflowPolicy.DropNewest` | Behavior when the exception log queue reaches capacity (`DropNewest` or `DropOldest`). |
| **ExceptionBatchSize** | `int` | `50` | Number of exception/log entries flushed to the database in a single batch. |
| **MaxInMemoryEntries** | `int` | `10_000` | Maximum retained rows for in-memory stores before oldest items are evicted. |
| **BroadcastQueueCapacity** | `int` | `2_048` | Live broadcast queue capacity for Server-Sent Events (SSE). |
| **BroadcastOverflowPolicy** | `BroadcastOverflowPolicy` | `BroadcastOverflowPolicy.DropOldest` | Behavior when the broadcast queue reaches capacity (`DropNewest` or `DropOldest`). |
| **BroadcastDetailMode** | `BroadcastDetailMode` | `BroadcastDetailMode.SummaryOnly` | Controls how much detail is pushed over SSE for live rows (`SummaryOnly` or `Full`). |
| **EnableDetailedLogEndpoint** | `bool` | `true` | Enables/disables endpoints used to fetch headers, bodies, stack traces, and metadata on demand. |
| **CaptureHostLogs** | `bool` | `true` | Captures host application `ILogger` events into the exceptions dashboard. |
| **HostLogMinimumLevel** | `LogLevel` | `LogLevel.Warning` | Minimum host application `ILogger` level captured by AsGuard. |
| **ExcludedHostLogCategoryPrefixes** | `HashSet<string>` | `["AsGuard."]` | Host logger categories excluded from capture (case-insensitive prefixes). |
| **CorrelationHeaderName** | `string` | `"X-Correlation-ID"` | HTTP header used to propagate correlation identifiers. |
| **RequestRetentionDays** | `int?` | `null` | Retention period (days) for request data. Disabled when null. |
| **ExceptionRetentionDays** | `int?` | `null` | Retention period (days) for exception data. Disabled when null. |
| **RetentionCleanupInterval** | `TimeSpan` | `6 hours` | How often the background retention worker deletes expired logs. |
| **QueuePressureAlertThreshold** | `double?` | `0.8` | Queue depth ratio that triggers queue pressure alerts (e.g. 0.8 = 80%). Disabled when null. |
| **ExceptionSpikeAlertThreshold** | `int?` | `null` | Number of exceptions in a rolling window that triggers an alert. Disabled when null. |
| **ExceptionSpikeAlertWindow** | `TimeSpan` | `5 minutes` | Rolling window used for exception spike alerts. |
| **AlertCooldown** | `TimeSpan` | `5 minutes` | Minimum time between repeated alerts of the same type. |
| **SamplingRate** | `double?` | `null` | Sample rate for request logging (0.0 to 1.0). Null means 100% of requests are logged. |
| **LogRequestsSlowerThan** | `TimeSpan?` | `null` | If set, only requests taking longer than this are logged (exceptions always log). |
| **LoggableStatusCodes** | `HashSet<int>` | `[]` | If populated, only requests with these status codes will be logged (exceptions always log). |
| **MaxCapturedBodyBytes** | `int` | `4_096` | Maximum body length captured in bytes before truncation. |
| **LogRequestBody** | `bool` | `false` | Enables/disables capturing and persisting request bodies. |
| **LogResponseBody** | `bool` | `false` | Enables/disables capturing and persisting response bodies. |
| **ExcludedPathPrefixes** | `HashSet<string>` | `["/health", "/swagger", "/metrics"]` | Case-insensitive path prefixes skipped by request logging. |
| **ExcludedExactPaths** | `HashSet<string>` | `["/health", "/metrics", "/swagger", "/favicon.ico"]` | Case-insensitive exact paths skipped by request logging. |
| **LoggableContentTypes** | `string[]` | `[]` | Content types eligible for body capture. If empty, all content types are captured. |
| **SensitiveHeaders** | `HashSet<string>` | `[]` | Headers redacted from logs (e.g. `"Authorization"`). `Set-Cookie` is always redacted. |
| **SensitiveBodyKeys** | `HashSet<string>` | `[]` | JSON keys whose values should be redacted in request/response bodies. |
| **EnrichLog** | `Action<HttpContext, RequestLog>?` | `null` | Custom action to enrich the captured RequestLog with extra data or tags dynamically. |
| **DashboardRoute** | `string` | `"/request-logs-ui"` | Route prefix for the monitoring dashboard. Empty disables dashboard mapping. |
| **DashboardUsername** | `string` | `""` | Basic Authentication username required to access dashboard and stream. |
| **DashboardPassword** | `string` | `""` | Basic Authentication password required to access dashboard and stream. |
| **EnableHostMetrics** | `bool` | `true` | Enables process CPU, memory, and ThreadPool resource sampling and graphing. |
| **HostMetricsSampleInterval** | `TimeSpan` | `15 seconds` | Sampling interval for host resource metrics. |
| **MaxHostMetricsStored** | `int` | `120` | Maximum time-series samples retained in the circular metrics memory buffer (default is 120 samples, which stores 30 minutes of history at a 15-second interval). |
| **EnableLiveLogConsole** | `bool` | `true` | Enables real-time ILogger console streaming to dashboard clients. |
| **LiveLogConsoleMinLevel** | `LogLevel` | `LogLevel.Information` | Minimum log level forwarded to the live console terminal. |
| **EnableHealthChecksVisualizer** | `bool` | `true` | Enables visual monitoring status cards for database and external health dependencies. |

---

## Storage Providers

AsGuard supports 4 storage providers via `DatabaseProvider`. Table structures are automatically validated and initialized at application startup, eliminating the need for manual Entity Framework migrations or script execution.

### 1. SQL Server
Best for enterprise production environments already using Microsoft SQL Server.
```csharp
builder.Services.AddRequestLogging(options =>
{
    options.DatabaseProvider = LoggingDatabaseProvider.SqlServer;
    options.ConnectionString = builder.Configuration.GetConnectionString("SqlServerConnection")!;
});
```

### 2. PostgreSQL
Best for high-performance open-source production deployments.
```csharp
builder.Services.AddRequestLogging(options =>
{
    options.DatabaseProvider = LoggingDatabaseProvider.PostgreSql;
    options.ConnectionString = builder.Configuration.GetConnectionString("PostgresConnection")!;
});
```

### 3. SQLite
Excellent for local development, staging environments, single-container Docker deployments, or small-scale applications.
```csharp
builder.Services.AddRequestLogging(options =>
{
    options.DatabaseProvider = LoggingDatabaseProvider.Sqlite;
    options.ConnectionString = "Data Source=asguard_logs.db;Cache=Shared;";
});
```

### 4. In-Memory
An ephemeral, process-local memory store. **Do not use in production.** Recommended only for:
- Integration testing environments.
- Local diagnostics where zero disk footprints are required.
- Short-lived diagnostic sessions.
```csharp
builder.Services.AddRequestLogging(options =>
{
    options.DatabaseProvider = LoggingDatabaseProvider.InMemory;
    // Cap memory utilization to prevent OutOfMemoryExceptions
    options.MaxInMemoryEntries = 5_000; 
});
```
> [!WARNING]
> When using `InMemory` storage, restarting the ASP.NET Core process completely clears all request and exception logs. 

---

## Performance & Queue Tuning

Because AsGuard persists logs asynchronously, it handles high-traffic bursts with ease. To optimize memory footprint and write performance, tune the background workers based on your load patterns.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Storage Engine Settings
    options.DatabaseProvider = LoggingDatabaseProvider.PostgreSql;
    options.ConnectionString = "...";

    // 1. Request Logging Queue Tuning
    options.QueueCapacity = 20_000;               // Buffer up to 20,000 logs in memory
    options.QueueOverflowPolicy = QueueOverflowPolicy.DropOldest; // Evict oldest if full
    options.BatchSize = 250;                       // Bulk-insert 250 records per batch

    // 2. Exception Logging Queue Tuning (Fewer items, needs reliability)
    options.ExceptionQueueCapacity = 2_000;
    options.ExceptionQueueOverflowPolicy = QueueOverflowPolicy.DropNewest; // Maintain original errors
    options.ExceptionBatchSize = 10;               // Write exceptions faster to database
});
```

### Queue Overflow Policies:
-   **`QueueOverflowPolicy.DropNewest`**: When the queue is full, new logs are silently discarded. This is the safest setting to prevent high memory pressure from impacting application memory during downstream database outages.
-   **`QueueOverflowPolicy.DropOldest`**: When the queue is full, the oldest items are removed to make room for new items. Best if you always need the most recent diagnostics.

---

## Retention & Automatic Cleanup

To prevent logs from consuming unlimited database storage, configure the built-in **Retention Cleanup Worker**. This background worker runs on a recurring schedule and deletes logs that exceed your configured threshold.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // ... Database configuration ...

    // Keep HTTP Request logs for 14 days
    options.RequestRetentionDays = 14;

    // Keep Exception & Event logs longer for auditing (e.g. 60 days)
    options.ExceptionRetentionDays = 60;

    // Run the cleanup routine every 12 hours (Default: 6 hours)
    options.RetentionCleanupInterval = TimeSpan.FromHours(12);
});
```

### Deactivating Retention
By default, `RequestRetentionDays` and `ExceptionRetentionDays` are set to `null` (disabled). If you want to disable automatic deletion and manage tables yourself (e.g., via database partitioning or custom cron scripts), leave these values as `null`.

---

## Performance Metrics & Host Diagnostics

AsGuard provides extensive process resource monitoring out-of-the-box. Performance records are cached entirely **in-memory** in a concurrent, thread-safe circular ring buffer, ensuring **zero database migrations** and a negligible storage footprint.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Enable system and process diagnostics graphing
    options.EnableHostMetrics = true;

    // Sample resource parameters every 30 seconds (Default: 15 seconds)
    options.HostMetricsSampleInterval = TimeSpan.FromSeconds(30);

    // Keep up to 240 samples in memory (2 hours of history at a 30s interval)
    options.MaxHostMetricsStored = 240;
});
```

Metrics captured include process CPU usage (adjusted for environment core counts), working set vs. private memory footprint, ThreadPool pending items, active threads, and GC counts.

---

## Live Application Console Logger

You can stream live `ILogger` console logs straight onto your dashboard with sub-millisecond latencies using Server-Sent Events (SSE). 

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Stream real-time logs to connected clients
    options.EnableLiveLogConsole = true;

    // Minimum severity level captured for the streaming terminal (Default: Information)
    options.LiveLogConsoleMinLevel = LogLevel.Debug;
});
```

---

## Distributed Tracing & W3C Trace Context

AsGuard supports standard **W3C Trace Context** out-of-the-box, allowing you to propagate trace identifiers across service boundaries using `traceparent` headers. This is essential for auditing execution graphs in microservices.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Enable W3C traceparent header propagation across downstream HttpClients (Default: true)
    options.EnableW3CPropagation = true;
});
```

### Traceparent Format & MD5 Detour:
- Conforms perfectly to W3C `00-{traceId}-{spanId}-01` specifications.
- If incoming traditional correlation headers (e.g., `X-Correlation-ID`) are present but do not comply with the 32-character hexadecimal requirement, AsGuard dynamically hashes the value using MD5. This derives a deterministic and valid 32-character hex traceId without altering the underlying correlation ID.

---

## Programmatic Dependency Health Checks

Integrate programmatic dependencies status boards directly into the AsGuard dashboard with **zero configurations**:

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Enable visual status cards for external databases and external APIs
    options.EnableHealthChecksVisualizer = true;
});
```

> [!NOTE]
> This capability hooks programmatically into standard ASP.NET Core `HealthCheckService`. Make sure to register standard health checks on your application builder startup:
> ```csharp
> builder.Services.AddHealthChecks()
>     .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")!)
>     .AddRedis(builder.Configuration.GetConnectionString("Redis")!);
> ```
