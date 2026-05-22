# Exceptions & Host Log Capture

AsGuard provides dual exception monitoring: capturing unhandled web failures via middleware, and absorbing host application warnings/errors using a custom `ILoggerProvider`. This page explains how both features function and how to log exceptions manually outside HTTP requests.

---

## 1. Centralized Exception Capture Middleware

When you register `app.UseRequestLogging()`, AsGuard automatically places the `AsGuardExceptionHandlingMiddleware` inside your request pipeline. 

This middleware intercepts **all unhandled exceptions** thrown by your controllers or endpoints:
1.  It records the exception details (type, message, stack trace, and full string details).
2.  It maps the exception directly to the current HTTP request log's fields (`ExceptionType`, `ExceptionMessage`, `ExceptionStackTrace`, `ExceptionDetails`).
3.  It queues a separate `ExceptionLog` event for the exceptions dashboard.
4.  **Crucially, it rethrows the exception.** This guarantees that AsGuard never interferes with your application's normal HTTP response handler (such as `app.UseExceptionHandler("/error")` or `app.UseDeveloperExceptionPage()`).

---

## 2. Host ILogger Capture

AsGuard registers a high-performance, thread-safe `ILoggerProvider` inside your dependency injection container. By default, it captures all log messages produced by the host application that meet your minimum severity rules.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Enable host log capture
    options.CaptureHostLogs = true;

    // Capture logs at warning level and above (Default: LogLevel.Warning)
    options.HostLogMinimumLevel = LogLevel.Warning;

    // Prevent noisy framework logs from cluttering storage
    options.ExcludedHostLogCategoryPrefixes.Add("Microsoft.AspNetCore.DataProtection");
    options.ExcludedHostLogCategoryPrefixes.Add("System.Net.Http");
});
```

### Automatic Self-Exclusion
AsGuard automatically adds `"AsGuard."` to `ExcludedHostLogCategoryPrefixes` on startup. This prevents infinite loops where AsGuard attempts to log its own database insertion or queue flushing logs.

### Important: ClearProviders() Warning
> [!WARNING]
> **ILogger Configuration Ordering**:
> If your application calls `builder.Logging.ClearProviders()`, make sure you call it **before** registering `builder.Services.AddRequestLogging(...)`. 
> 
> If you call `ClearProviders()` after `AddRequestLogging()`, the AsGuard logging provider will be removed from your logger factory, and application warnings/errors will not appear in the dashboard.

---

## 3. Manual Exception Logging (`IExceptionLogger`)

Observability isn't just about web endpoints. Many production applications run background queues, scheduled cron workers, RabbitMQ/Kafka message consumers, or SignalR connections that exist outside HTTP request contexts.

For these processes, AsGuard exposes the `IExceptionLogger` interface. You can inject this interface into any transient, scoped, or singleton service.

```csharp
using AsGuard.Domain.RequestLogging;
using AsGuard.Services;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;

public sealed class PaymentQueueConsumer
{
    private readonly IExceptionLogger _exceptionLogger;
    private readonly ILogger<PaymentQueueConsumer> _logger;

    public PaymentQueueConsumer(IExceptionLogger exceptionLogger, ILogger<PaymentQueueConsumer> logger)
    {
        _exceptionLogger = exceptionLogger;
        _logger = logger;
    }

    public async Task ProcessOrderAsync(Order order)
    {
        try
        {
            // Execute order processing logic
            await ProcessPaymentGatewayAsync(order);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process payment for Order {OrderId}", order.Id);

            // Log manually to AsGuard with enriched business context
            await _exceptionLogger.LogAsync(
                ex,
                LogLevel.Critical,
                new ExceptionLogContext(
                    CorrelationId: order.CorrelationId, // Maintain correlation across boundaries
                    Path: "QueueConsumer/ProcessOrder",
                    Method: "CONSUME",
                    UserId: order.CustomerId,
                    Tags: "billing,payment-gateway",
                    MetadataJson: $$"""
                    {
                        "orderId": "{{order.Id}}",
                        "amount": {{order.TotalAmount}},
                        "attempts": {{order.RetryCount}}
                    }
                    """
                ));
        }
    }
}
```

### ExceptionLogContext Reference
The `ExceptionLogContext` record allows you to associate request-like properties with your exceptions for unified filtering on the dashboard:

| Property | Type | Description |
| :--- | :--- | :--- |
| **CorrelationId** | `string?` | Existing correlation ID. If omitted, AsGuard will attempt to resolve it from the active `CorrelationContext` or generate a new one. |
| **Path** | `string?` | A descriptor representing the background routine or channel name. |
| **Method** | `string?` | An operation verb or background process name (e.g. `"CRON"`, `"CONSUME"`, `"JOB"`). |
| **UserId** | `string?` | The customer or tenant identifier associated with the event. |
| **ClientIp** | `string?` | IP address of the client (if applicable). |
| **RequestId** | `string?` | Original HTTP Request ID (if applicable). |
| **Tags** | `string?` | A comma-separated list of searchable tags. |
| **MetadataJson** | `string?` | A structured JSON string representing custom variables. |
| **OccurredOnUtc** | `DateTime?` | Explicit event timestamp. If omitted, AsGuard uses the current system time (`DateTime.UtcNow`). |
