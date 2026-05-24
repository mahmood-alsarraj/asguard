# Dashboard & Web API Reference

AsGuard features a premium built-in Razor dashboard alongside a secure, highly-performant HTTP REST API and Server-Sent Events (SSE) stream. This page details dashboard navigation, security configurations, API routing schemas, query specifications, and live streaming capabilities.

---

## 1. Dashboard UI Features

The built-in dashboard auto-loads at your configured `DashboardRoute`. It requires zero external scripts or assets to be compiled and includes:

*   **Responsive Theme Engine**: Auto-detects system light or dark preferences with an instant toggle.
*   **Request Logs Terminal**: Lists monitored HTTP requests with real-time response codes, speed metrics, user tracking, and expandable detail boxes (headers, query parameters, redacted bodies, and correlated trace files).
*   **Exceptions & Event Terminal**: Lists caught unhandled crashes, manually pushed errors, and standard logger events. Includes fully interactive stack trace viewing boxes.
*   **Multi-Layer Filtering**: Support for immediate search, level/severity filter drops, HTTP method checkboxes, status code selections, exact correlation ID queries, and UTC date ranges.
*   **Data Controls**: Clear all logs or filter-selectively clear storage files directly from the UI interface.

---

## 2. Basic Authentication Security

To protect production diagnostics, the dashboard, the REST API endpoints, and the SSE live stream are shielded behind an HTTP Basic Authentication middleware.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Route pattern to mount the dashboard UI
    options.DashboardRoute = "/request-logs-ui";

    // Set secure Basic Auth credentials
    options.DashboardUsername = builder.Configuration["AsGuard:Username"]!;
    options.DashboardPassword = builder.Configuration["AsGuard:Password"]!;
});
```

### Startup Security Validation
To prevent developers from accidentally exposing logging assets to production, AsGuard executes an automated validation service at application boot:
1.  **Requirement**: If `DashboardRoute` is set to any non-empty path, both `DashboardUsername` and `DashboardPassword` **must** be populated. If empty, the application throws an `InvalidOperationException` and fails to start.
2.  **Legacy Credentials Rejection**: If you set username to `"admin"` and password to `"admin"`, the startup validation service **rejects the execution** and throws an error warning you to set custom values.

---

## 3. Web API Reference

All endpoints require HTTP Basic Authentication headers using the credentials configured above.

### Summary Table of Endpoints

| Method | Endpoint Route | Description |
| :--- | :--- | :--- |
| **GET** | `/request-logs-api` | Query and page request logs. |
| **GET** | `/request-logs-api/{id}` | Retrieve full request log details (headers, bodies). |
| **DELETE** | `/request-logs-api` | Delete request logs matching query filters (or clear all). |
| **GET** | `/request-logs-api/stats` | Retrieve diagnostic queue depths and counter metrics. |
| **GET** | `/request-logs-api/exceptions` | Query and page captured exceptions and logger events. |
| **GET** | `/request-logs-api/exceptions/{id}` | Retrieve full stack trace and metadata for an exception. |
| **GET** | `/request-logs-api/exceptions/summary` | Retrieve log count statistics grouped by severity levels. |
| **GET** | `/request-logs-api/exceptions/trends` | Retrieve exceptions timeline trends (hourly or daily). |
| **DELETE** | `/request-logs-api/exceptions` | Delete exceptions matching query filters (or clear all). |
| **GET** | `/request-logs-api/host-environment` | Retrieve system host diagnostics (uptime, OS version, .NET framework, CPU cores, machine/env). |

---

## 4. Query Parameters Reference

### Querying Request Logs (`GET /request-logs-api`)
*   `PageIndex` (`int`, default: `1`): 1-based index page.
*   `PageSize` (`int`, default: `20`): Page size limit (1 to 100).
*   `Search` (`string`): Filters by path, query string, or tags.
*   `Method` (`string`): Filter by HTTP verb (e.g. `"GET"`, `"POST"`).
*   `StatusCode` (`int`): Filter by exact HTTP response status code.
*   `FromUtc` (`DateTime`): Filter logs occurring on or after timestamp.
*   `ToUtc` (`DateTime`): Filter logs occurring on or before timestamp.
*   `UserId` (`string`): Filter logs matching specific user identity.
*   `ExceptionsOnly` (`bool`, default: `false`): If true, returns only requests that threw unhandled exceptions.
*   `IncludeDetails` (`bool`, default: `false`): If true, includes body payloads and HTTP headers in list results.
*   `SkipTotalCount` (`bool`, default: `false`): Skips running the SQL total count query for faster paging.

### Querying Exceptions (`GET /request-logs-api/exceptions`)
*   `PageIndex`, `PageSize`, `FromUtc`, `ToUtc`, `Search`, `SkipTotalCount` (same specifications as request logs).
*   `Level` (`LogLevel`): Filter by exact LogLevel string (`Warning`, `Error`, etc.).
*   `CorrelationId` (`string`): Filter by exact correlation trace string.
*   `IncludeDetails` (`bool`, default: `false`): If true, returns stack trace and inner details.

---

## 5. API Response Payload Schemas

### Request Log Detail Response (`GET /request-logs-api/123`)
```json
{
  "id": 123,
  "correlationId": "8f2d59ac-b49d-47fe-902e-b615177894a8",
  "method": "POST",
  "path": "/api/users/register",
  "queryString": "?source=web",
  "requestHeaders": "Host: localhost:5001\r\nContent-Type: application/json\r\nAuthorization: [REDACTED]\r\n",
  "requestBody": "{\"email\":\"demo@asguard.io\",\"password\":\"[REDACTED]\"}",
  "responseBody": "{\"id\":9912,\"status\":\"Created\"}",
  "clientIp": "127.0.0.1",
  "statusCode": 201,
  "durationInMilliseconds": 42.15,
  "occurredOnUtc": "2026-05-22T13:15:00Z",
  "exceptionType": null,
  "exceptionMessage": null,
  "exceptionStackTrace": null,
  "exceptionDetails": null,
  "userId": "user_demo_9912",
  "tags": "public,auth",
  "metadataJson": "{\"tenantId\":\"emea-77\",\"environment\":\"production\"}"
}
```

### Diagnostic Stats Response (`GET /request-logs-api/stats`)
Useful for integration into external dashboard solutions like Datadog or Prometheus:
```json
{
  "requestQueueDepth": 0,
  "enqueuedRequests": 45102,
  "droppedRequests": 0,
  "exceptionQueueDepth": 0,
  "enqueuedExceptions": 322,
  "droppedExceptions": 0,
  "persistedRequests": 45102,
  "persistedExceptions": 322,
  "persistenceFailures": 0,
  "broadcastFailures": 0,
  "activeSignalRConnections": 2
}
```
*Note: `activeSignalRConnections` tracks active HTTP Server-Sent Event streaming client channels.*

---

## 6. Live Server-Sent Events (SSE) Stream

AsGuard exposes a real-time Server-Sent Events stream at `/request-logs-stream`. 

### SSE Streaming Setup
The stream checks Basic Authentication credentials provided in the query string or authorization headers. Frontend applications (like the built-in dashboard) establish an `EventSource` connection to update UI tables without manual polling.

```text
GET /request-logs-stream?groups=logs,exceptions
```

*   **Query Parameters**: `groups` (Optional. Comma-separated list of events to subscribe to: `"logs"`, `"exceptions"`. If omitted, subscribes to all events).
*   **Hearbeat (Ping)**: The server pushes a `: ping\n\n` event every 15 seconds to keep idle connections alive through reverse proxies and load balancers.

### SSE Message Schema
Pushed messages are structured JSON envelopes containing a `method` descriptor and a typed list `payload`:

```text
data: {"method":"NewLogs","payload":[{"id":124,"correlationId":"a1b2c3d4...","method":"GET","path":"/api/greet","statusCode":200,"durationInMilliseconds":2.8,"occurredOnUtc":"2026-05-22T13:20:00Z"}]}
```

### Broadcast Queue Pressure Settings
To ensure broadcasting updates to slow web client connections never hurts server memory, configure the live broadcast queues:

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Buffer size for SSE events awaiting transmission (Default: 2_048)
    options.BroadcastQueueCapacity = 1_000;

    // Evict oldest SSE frames if client experiences network lag
    options.BroadcastOverflowPolicy = BroadcastOverflowPolicy.DropOldest;

    // Set how much detail is broadcast live to dashboard grids
    // Options: SummaryOnly (Minimal payload, default) or Full (Includes headers/bodies)
    options.BroadcastDetailMode = BroadcastDetailMode.SummaryOnly;
});
```

---

## 7. Advanced Diagnostics & Profiling Endpoints

### A. Host Environment Summary (`GET /request-logs-api/host-environment`)
Returns high-level system diagnostics in a sanitized format. Sensitive environmental settings are redacted, showing only safe keys.

**Response Schema:**
```json
{
  "machineName": "PROD-SERVER-01",
  "osVersion": "Microsoft Windows 10.0.22631",
  "frameworkDescription": ".NET 8.0.4",
  "processorCount": 16,
  "environment": "Production",
  "uptime": "2d 04h 12m 10s"
}
```
