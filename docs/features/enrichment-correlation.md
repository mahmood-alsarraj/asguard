# Log Enrichment & Correlation

Connecting related logs across microservices and tagging records with business context (such as User IDs, active tenants, or deployment regions) is critical for effective system triage. This guide explains how AsGuard manages correlation headers and provides multiple mechanisms to enrich logs dynamically.

---

## 1. Correlation IDs

When an HTTP request enters the middleware pipeline, AsGuard's `CorrelationMiddleware` executes:
1.  **Read Header**: It scans incoming HTTP request headers for the configured header name (Default: `X-Correlation-ID`).
2.  **Validate**: It sanitizes the ID. Accepted IDs must contain only alphanumeric characters, hyphens, and underscores, and must be 128 characters or shorter. Invalid correlation IDs are ignored.
3.  **Assign**: If a valid ID is present, AsGuard adopts it. Otherwise, it generates a fresh `Guid`.
4.  **Trace Identifier**: It overwrites `HttpContext.TraceIdentifier` with this correlation ID. This automatically aligns the internal ASP.NET Core log scopes with the AsGuard Correlation ID.
5.  **Propagate Header**: It injects the correlation ID back into the outgoing HTTP response headers, allowing frontend clients or gateway APIs to reference it on failure.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Configure a custom correlation header name (e.g. for AWS or Azure environments)
    options.CorrelationHeaderName = "X-Amz-Cf-Id"; 
});
```

### W3C Distributed Tracing (Trace Context)

In modern microservice architectures, incoming requests may originate from external applications or other services. To coordinate traces across these distributed boundaries, AsGuard supports the standard **W3C Trace Context** out-of-the-box when `EnableW3CPropagation` is enabled:

1. **Incoming traceparent**: If an incoming request contains a `traceparent` header (e.g. `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`), AsGuard extracts the 32-character hexadecimal `traceId` (`4bf92f3577b34da6a3ce929d0e0e4736`) and assigns it as the active `CorrelationId`.
2. **Deterministic MD5 Hashing (MD5 Detour)**: If traditional correlation headers (e.g. `X-Correlation-ID`) are present but do not comply with the W3C 32-character hexadecimal format, AsGuard dynamically hashes the value using MD5. This derives a deterministic and compliant 32-character hex `traceId` so the request can still be traced cleanly across downstream boundaries.
3. **Outgoing traceparent**: When communicating with other services via `HttpClient`, AsGuard automatically injects the `traceparent` header (`00-{traceId}-{newSpanId}-01`) where `newSpanId` is a random 8-byte hexadecimal string generated specifically for the outgoing span context.
4. **Response Propagation**: The W3C compliant `traceparent` is automatically appended to the outgoing HTTP response headers.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Enable W3C traceparent header context propagation (Default: true)
    options.EnableW3CPropagation = true;
});
```

---

## 2. User Identification

By default, AsGuard attempts to resolve the active user identifier making the HTTP request without requiring custom code.

During the enrichment phase, AsGuard scans the claims collection on the active `ClaimsPrincipal` (`HttpContext.User`). It maps the first matching claim value to the log's searchable `UserId` property:
-   `System.Security.Claims.ClaimTypes.NameIdentifier` (standard OAuth/OIDC username claim)
-   `sub` (standard OpenID Connect subject claim)

If your authentication provider stores user IDs in a non-standard claim (e.g. `"client_id"` or `"email"`), you can override this behavior using custom enrichers.

---

## 3. Inline Option-Based Enrichment

For simple, lightweight enrichment that does not require external service lookups, use the `EnrichLog` action delegate inside `AddRequestLogging`.

This delegate runs on the active `HttpContext` immediately before the request log is enqueued, making route details, query strings, headers, and claims fully accessible:

```csharp
builder.Services.AddRequestLogging(options =>
{
    // ... Database settings ...

    options.EnrichLog = (httpContext, requestLog) =>
    {
        // 1. Custom User ID Resolution
        if (httpContext.Request.Headers.TryGetValue("X-Client-Org", out var orgHeader))
        {
            requestLog.UserId = orgHeader.ToString();
        }

        // 2. Custom Tags (Comma-separated searchable labels)
        requestLog.Tags = "retail,public-api,v2";

        // 3. Custom Structured Metadata (Automatically saved as JSON)
        requestLog.Metadata["Environment"] = "Staging";
        requestLog.Metadata["ServerNode"] = Environment.MachineName;
        requestLog.Metadata["RequestProtocol"] = httpContext.Request.Protocol;
    };
});
```

---

## 4. Custom DI-Based Enrichers (`IAsGuardRequestEnricher`)

For complex enrichment scenarios requiring dependency-injectable services (such as looking up database records, querying memory caches, or resolving configuration files), define an enricher service by implementing `IAsGuardRequestEnricher`.

### Step A: Define the Enricher Service
Implement the `IAsGuardRequestEnricher` interface:

```csharp
using AsGuard.Domain.RequestLogging;
using AsGuard.Services;
using Microsoft.AspNetCore.Http;

public sealed class TenantRequestEnricher : IAsGuardRequestEnricher
{
    private readonly ITenantLookupService _tenantLookup;

    public TenantRequestEnricher(ITenantLookupService tenantLookup)
    {
        _tenantLookup = tenantLookup;
    }

    public void Enrich(HttpContext context, RequestLog log)
    {
        // Resolve the active tenant from route headers or custom subdomains
        var tenantCode = context.Request.Headers["X-Tenant-Code"].ToString();
        
        if (!string.IsNullOrWhiteSpace(tenantCode))
        {
            var tenantInfo = _tenantLookup.GetTenantDetails(tenantCode);
            if (tenantInfo != null)
            {
                // Assign a searchable tag
                log.Tags = string.IsNullOrEmpty(log.Tags) 
                    ? tenantInfo.Slug 
                    : $"{log.Tags},{tenantInfo.Slug}";

                // Inject structured parameters into the Metadata dictionary
                log.Metadata["TenantId"] = tenantInfo.Id;
                log.Metadata["SubscriptionTier"] = tenantInfo.Tier;
                log.Metadata["BillingStatus"] = tenantInfo.Status;
            }
        }
    }
}
```

### Step B: Register in Dependency Injection
Register your class in the application container. AsGuard detects all registered implementations of `IAsGuardRequestEnricher` and executes them automatically:

```csharp
// Register custom services needed by your enricher
builder.Services.AddScoped<ITenantLookupService, TenantLookupService>();

// Register your custom AsGuard enricher (Scoped or Singleton)
builder.Services.AddScoped<IAsGuardRequestEnricher, TenantRequestEnricher>();

// Add AsGuard request logging
builder.Services.AddRequestLogging(options => { /* options... */ });
```

---

## 5. Performance Guidelines

> [!IMPORTANT]
> **Enrichers Run Synchronously inside the Middleware Pipeline**:
> All registered custom enrichers (`IAsGuardRequestEnricher`) and inline delegates (`EnrichLog`) execute **on the active HTTP request thread** immediately after the response is completed. 
> 
> To ensure AsGuard does not add latency to your API endpoints:
> -   **Avoid blocking I/O**: Do not execute synchronous database queries, slow HTTP calls, or file writes inside your enrichers.
> -   **Use Caching**: If you must fetch metadata from database stores, utilize `IMemoryCache` or fast key-value caches to resolve information in under a millisecond.
> -   **Handle Exceptions**: Wrap custom enrichment logic in `try-catch` scopes to guarantee a custom logic crash never prevents the main HTTP request from logging.
