# Observability, Alerts & Production Guide

Observing the operational health of your monitoring infrastructure is just as important as monitoring your application. This page details AsGuard's built-in alert engine, explains how to route system alerts to external channels like Slack or Teams, troubleshoots common setups, and outlines production checklist guidelines.

---

## 1. Built-In Alerting Engine

AsGuard runs an internal background monitor service (`AsGuardAlertMonitorService`) that watches diagnostic counters and raises **System Alerts** under four scenarios:

1.  **Queue Pressure Alert**: Raised if the internal request queue depth ratio exceeds your threshold (Default: `0.8`, meaning 80% capacity). Indicates your database cannot keep up with traffic surges.
2.  **Exception Spike Alert**: Raised if the number of unhandled or manually logged exceptions within a rolling window exceeds your limit.
3.  **Persistence Failure Alert**: Raised if the background workers encounter database exceptions while writing logs.
4.  **Broadcast Failure Alert**: Raised if Server-Sent Events transmission errors occur repeatedly.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // ... Storage and security settings ...

    // 1. Queue Pressure Alert Limit (80% full)
    options.QueuePressureAlertThreshold = 0.8;

    // 2. Exception Spike Alert Limit
    // Triggers if 25 exceptions occur within a 5-minute rolling window
    options.ExceptionSpikeAlertThreshold = 25;
    options.ExceptionSpikeAlertWindow = TimeSpan.FromMinutes(5);

    // 3. Alert Cooldown
    // Prevent spamming your team channels; wait at least 5 minutes before re-alerting
    options.AlertCooldown = TimeSpan.FromMinutes(5);
});
```

### Deactivating Alerts
To disable queue pressure or exception spike alerts, set their respective thresholds (`QueuePressureAlertThreshold` or `ExceptionSpikeAlertThreshold`) to `null`.

---

## 2. Custom Alert Sinks (`IAsGuardAlertSink`)

By default, all raised alerts are written to standard application logs via `ILogger`. 

To push alerts directly to Slack, MS Teams, internal paging services, or email, implement the **`IAsGuardAlertSink`** interface and register it in the dependency injection container.

### Step A: Implement the Webhook Sink
Create a custom class to serialize the `AsGuardAlert` model and POST it to a chat webhook:

```csharp
using AsGuard.Services;
using System.Net.Http;
using System.Text;
using System.Text.Json;
using System.Threading;
using System.Threading.Tasks;

public sealed class TeamsWebhookAlertSink : IAsGuardAlertSink
{
    private readonly HttpClient _httpClient;
    private readonly string _webhookUrl;

    public TeamsWebhookAlertSink(HttpClient httpClient, IConfiguration config)
    {
        _httpClient = httpClient;
        _webhookUrl = config["AsGuard:AlertsWebhookUrl"]!;
    }

    public async ValueTask PublishAsync(AsGuardAlert alert, CancellationToken cancellationToken)
    {
        // Construct a clean, readable message for chat channels
        var payload = new
        {
            title = $"[AsGuard Alert] {alert.Type}",
            text = $"**Message**: {alert.Message}\n\n**Occurred (UTC)**: {alert.OccurredOnUtc:yyyy-MM-dd HH:mm:ss}\n\n**Details**: {JsonSerializer.Serialize(alert.Properties)}"
        };

        var json = JsonSerializer.Serialize(payload);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        try
        {
            var response = await _httpClient.PostAsync(_webhookUrl, content, cancellationToken);
            response.EnsureSuccessStatusCode();
        }
        catch
        {
            // Fail silently or fallback to standard console logs to prevent 
            // alert delivery failures from crashing the host application.
        }
    }
}
```

### Step B: Register the Custom Sink
Add your sink class in `Program.cs`. Ensure you register it **after** `AddRequestLogging(...)` to override the default console provider:

```csharp
builder.Services.AddHttpClient();

// 1. Add AsGuard Services
builder.Services.AddRequestLogging(options => { /* ... */ });

// 2. Override default sink with your custom webhook sink
builder.Services.AddSingleton<IAsGuardAlertSink, TeamsWebhookAlertSink>();
```

---

## 3. Production Guidelines & Best Practices

Before deploying AsGuard to a production environment, review the following checklist:

### A. GDPR & Privacy Compliance
*   **Opt-out of Body Capture**: In public-facing portals or applications dealing with PII or HIPAA, keep `LogRequestBody` and `LogResponseBody` set to `false`.
*   **Enforce Masking**: If you must enable body capture, annotate all sensitive variables (passwords, social security numbers, bank cards) with the `[AsGuardMasked]` attribute.
*   **Redact Headers**: Explicitly register `Authorization`, `Cookie`, `X-Api-Key`, and custom session tokens in the `SensitiveHeaders` collection.

### B. Storage Scaling
*   **SQLite Limitations**: SQLite works well for microservices or single containers under moderate load. However, SQLite locks the database file during writes. For high-concurrency systems, migrate to **PostgreSQL** or **SQL Server**.
*   **Define Retention**: Always configure `RequestRetentionDays` (e.g. `14` days) and `ExceptionRetentionDays` (e.g. `30` days) to prevent databases from exhausting disk storage space over time.

### C. Queue Pressure Mitigation
*   If your stats endpoint (`GET /request-logs-api/stats`) reports `droppedRequests` > 0, your in-memory channel is full.
*   Increase persistence throughput by increasing `BatchSize` (e.g. from `100` to `250`).
*   Verify your database cluster's CPU utilization and latency indices.

---

## 4. Troubleshooting Common Issues

### Issue 1: Application Warning/Error logs do not appear in the dashboard
1.  Verify that `CaptureHostLogs` is set to `true`.
2.  Ensure your code or environment configuration does not call `builder.Logging.ClearProviders()` *after* `AddRequestLogging(...)`. 
3.  Check if `EnableExceptionLogging` is enabled, as application logs use the exceptions background persistence queue.
4.  Ensure that the logging namespace is not excluded by `ExcludedHostLogCategoryPrefixes`.

### Issue 2: Dashboard/API returns `401 Unauthorized`
1.  Verify that `DashboardUsername` and `DashboardPassword` are correctly configured.
2.  Confirm that your browser or API client is sending standard HTTP Basic Authentication headers (`Authorization: Basic <base64_credentials>`).
3.  Check if you are attempting to log in using the legacy credentials `admin` / `admin`. If configured, AsGuard will reject these credentials and fail to boot at startup.

### Issue 3: Missing Request or Response Bodies
1.  Confirm that `LogRequestBody` or `LogResponseBody` is set to `true`.
2.  Check that the HTTP request's `Content-Type` header matches one of the values listed in your `LoggableContentTypes` array.
3.  Confirm that the payload size did not exceed the limit set by `MaxCapturedBodyBytes`.
