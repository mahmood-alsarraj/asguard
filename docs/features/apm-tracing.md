# Application Performance Monitoring (APM) & Trace Timelines

AsGuard includes a built-in, zero-configuration **Application Performance Monitoring (APM)** engine. It automatically tracks database transactions and downstream API interactions executed within the context of an HTTP request, and visualizes them on a responsive, pure CSS/HTML Gantt chart timeline.

This gives developers immediate visibility into slow requests: whether a route was blocked by a poorly optimized SQL query, a slow external payment gateway API, or high database transaction concurrency.

---

## Technical Features

### 1. Automatic Diagnostic Interception
AsGuard hooks directly into .NET's native `DiagnosticListener` framework. When you register AsGuard, it registers an startup observer service (`AsGuardDiagnosticsSubscriber`) that automatically attaches to two major telemetry producers:

*   **Entity Framework Core (`Microsoft.EntityFrameworkCore`)**: Captures execution lifecycle events of every raw SQL query, database migration, or LINQ expression translated to SQL.
*   **HttpClient (`HttpHandlerDiagnosticListener`)**: Captures outgoing HTTP queries made via the BCL `HttpClient` library (e.g. calling Stripe, Auth0, or internal microservices).

This is **fully automatic**. You do not need to register database command interceptors, add custom delegating handlers to your HTTP clients, or modify any business logic.

### 2. Lock-Free Scoped Trace Collector
Trace spans are collected inside a request-scoped collector (`IApmSpanCollector`). 
*   **Thread Safety**: It uses concurrent lock-free collections (`ConcurrentQueue` and `ConcurrentDictionary`) to support asynchronous, multi-threaded request blocks (like calling `Task.WhenAll`).
*   **Zero Leakage**: Since the collection container is registered with `Scoped` lifetime, all gathered spans are garbage-collected immediately once the request completes, eliminating any risks of memory leakage.

### 3. Asynchronous Performance Queue
Instead of saving query timings synchronously and adding overhead to your request threads, AsGuard queues collected trace spans into an internal `System.Threading.Channels` queue at request completion.
A long-running hosted persistence service (`ApmSpanPersistenceWorker`) drains this queue in the background and commits trace logs to storage in bulk batches, maintaining **100% fast request execution speeds**.

---

## Database Schema & Tables

When you enable AsGuard, it automatically creates the `AsGuardApmSpans` table on startup within your configured database.

### SQL Table Schema:
```sql
CREATE TABLE [Logging].[AsGuardApmSpans] (
    [Id] bigint IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [CorrelationId] nvarchar(64) NOT NULL,
    [SpanType] nvarchar(32) NOT NULL, -- 'DbQuery' or 'HttpClient'
    [Name] nvarchar(max) NOT NULL, -- SQL script or full URI
    [DurationInMilliseconds] float NOT NULL,
    [StartOffsetInMilliseconds] float NOT NULL,
    [IsSuccess] bit NOT NULL,
    [ErrorMessage] nvarchar(max) NULL,
    [OccurredOnUtc] datetime2 NOT NULL
);
CREATE INDEX [IX_AsGuardApmSpans_CorrelationId] ON [Logging].[AsGuardApmSpans] ([CorrelationId]);
```

*   **CorrelationId**: Links each trace span directly to its parent `RequestLog`.
*   **StartOffsetInMilliseconds**: Timing offset representing precisely *when* the command executing started relative to the start of the HTTP request. This is crucial for rendering Gantt timelines accurately.

---

## Razor Dashboard Gantt Charts

In the AsGuard Dashboard details panel, requests that generated DB queries or outgoing HTTP calls will display a **Trace Spans Timeline**:

```text
Trace Spans Timeline
Total Duration: 450 ms | Spans Count: 3

[SQL] SELECT * FROM [Users] WHERE [Id] = @p0
=== [50ms (offset 10ms)]

[HTTP] POST https://api.stripe.com/v1/charges
======================== [300ms (offset 70ms)]

[SQL] UPDATE [Orders] SET [Status] = @p0 WHERE [Id] = @p1
== [20ms (offset 380ms)]
```

### Timeline Visual Highlights:
*   **Color-Coded Tracks**: Database queries display as elegant **purple** bars, while outgoing HTTP calls render as **amber** bars.
*   **Failure Highlighting**: Spans that threw exceptions or returned non-success HTTP status codes (4xx/5xx) display a warning emoji (`⚠️`) and a glowing **rose border**.
*   **Interactive Drill-Down**: Clicking on any timeline bar collapses or expands a detailed code block showing the full formatted SQL query text or downstream URI, alongside exception trace details if it failed.

---

## Configuration & Disabling Tracing

APM tracing is **enabled by default**. If you wish to disable database and HttpClient telemetry (for instance, to reduce database writes or completely isolate request logs), set the `EnableApmTracing` flag to `false`:

```csharp
builder.Services.AddRequestLogging(options =>
{
    // ... Database configuration ...

    // Deactivate all automatic EF Core & HttpClient tracing
    options.EnableApmTracing = false;
});
```

When `EnableApmTracing` is set to `false`:
*   `AsGuardDiagnosticsSubscriber` completely bypasses diagnostic listener subscriptions at startup, resulting in **zero telemetry hookup overhead**.
*   `ApmSpanPersistenceWorker` and the request logging middleware completely skip span timing, collection, and background queuing activities.

---

## Performance & Security Considerations

### Memory Footprint
Spans are bounded inside `RequestLoggingOptions.MaxInMemoryEntries * 10` for `InMemory` storage to protect server RAM. For persistent databases, bulk batches flush spans instantly, keeping heap utilization light.

### Query Parameters & sensitive data
By default, AsGuard logs the raw database command text and downstream request URIs exactly as intercepted to maximize debugging utility. 

> [!WARNING]
> Captured SQL texts can contain parameter details if database-level sensitive logging is enabled. Always ensure that `SensitiveHeaders` redact keys like `Authorization` or `X-Api-Key` to keep external HTTP call URLs safe.
