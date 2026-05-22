# HTTP Traffic Monitoring

AsGuard provides deep visibility into HTTP request and response cycles. This guide details how to control payload capture, protect user privacy with advanced redaction and JSON body masking, and implement sampling filters to manage storage growth.

---

## 1. Request & Response Body Capture

By default, body capture is **disabled** to avoid high storage usage and inadvertent logging of sensitive information. 

You can selectively enable request and/or response body capture, configure character limits, and restrict capture to safe content types:

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Enable body capture
    options.LogRequestBody = true;
    options.LogResponseBody = true;

    // Set maximum body capture size to 8 KB (Default: 4_096 bytes)
    // Payloads exceeding this size are cleanly truncated with a marker.
    options.MaxCapturedBodyBytes = 8_192;

    // Limit body capture to specific content types.
    // If empty (Default), AsGuard attempts to log ALL content types, 
    // which can lead to capturing binary files or media.
    options.LoggableContentTypes =
    [
        "application/json",
        "application/xml",
        "text/",
        "application/x-www-form-urlencoded"
    ];
});
```

### Skipped Content Types
If a request or response content type does not match the prefixes defined in `LoggableContentTypes`, AsGuard skips the body and records a `"[Body capture skipped: unsupported content type]"` marker in the storage.

---

## 2. Header Redaction

HTTP headers often carry credentials, cookies, session identifiers, or API keys. AsGuard prioritizes security by providing robust header redaction utilities.

*   **Always Redacted**: The `Set-Cookie` response header is **always** redacted automatically by AsGuard.
*   **Opt-In Redaction**: Other sensitive headers must be explicitly registered in `SensitiveHeaders` to be replaced with a `"[REDACTED]"` value.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // Add common sensitive headers to redaction list
    options.SensitiveHeaders.Add("Authorization");
    options.SensitiveHeaders.Add("Cookie");
    options.SensitiveHeaders.Add("X-Api-Key");
    options.SensitiveHeaders.Add("X-Session-Token");
    options.SensitiveHeaders.Add("X-Profile-ID");
});
```

---

## 3. Body Data Masking

When capturing request and response JSON bodies, it is common for sensitive variables (such as `"password"`, `"token"`, or `"creditCard"`) to slip into logs. AsGuard provides two complementary layers of data masking to cleanse payloads before they are committed to database storage:

### A. Attribute-Based Model Masking (Recommended)
You can annotate property definitions in your DTOs or Request models using the `[AsGuardMasked]` attribute. 

AsGuard scans all application assemblies at startup. When a matching key is found in any serialized JSON request or response, AsGuard replaces the value with `"[REDACTED]"` in the logged payload. The actual HTTP response returned to the client is completely untouched.

```csharp
using AsGuard.Domain.RequestLogging;
using System.Text.Json.Serialization;

public sealed class RegisterUserRequest
{
    public string Email { get; set; } = string.Empty;

    [AsGuardMasked]
    public string Password { get; set; } = string.Empty;

    // AsGuard respects [JsonPropertyName] properties automatically
    [AsGuardMasked]
    [JsonPropertyName("security_pin")]
    public string SecurityPin { get; set; } = string.Empty;
}
```

### B. Option-Based Manual Masking
If you cannot modify the model source code directly, you can register key names to be masked via options:

```csharp
builder.Services.AddRequestLogging(options =>
{
    // ...
    // Mask these keys in any JSON body captured
    options.SensitiveBodyKeys.Add("creditCardNumber");
    options.SensitiveBodyKeys.Add("cvv");
    options.SensitiveBodyKeys.Add("ssn");
});
```
*Note: Key matching is case-insensitive.*

---

## 4. Advanced Sampling & Filtering

For high-traffic systems, logging 100% of HTTP traffic can quickly saturate databases and exhaust worker queues. AsGuard provides multiple runtime sampling and filtering mechanisms to target only what is necessary.

```csharp
builder.Services.AddRequestLogging(options =>
{
    // 1. Log only 15% of successful requests (0.0 to 1.0)
    // Value of 0.15 means a random 15% selection of traffic will be stored.
    options.SamplingRate = 0.15;

    // 2. Slow Request Capture
    // Log only requests taking longer than 500ms
    options.LogRequestsSlowerThan = TimeSpan.FromMilliseconds(500);

    // 3. Status Code Specific Filter
    // If populated, only requests returning these codes are captured
    options.LoggableStatusCodes.Add(StatusCodes.Status400BadRequest);
    options.LoggableStatusCodes.Add(StatusCodes.Status401Unauthorized);
    options.LoggableStatusCodes.Add(StatusCodes.Status403Forbidden);
    options.LoggableStatusCodes.Add(StatusCodes.Status500InternalServerError);
});
```

### Exception Bypass Rule:
> [!IMPORTANT]
> **Exceptions Are Exempt from Filters**:
> Any request that encounters an unhandled exception or results in a captured system failure **bypasses all sampling and filtering rules** and is **always logged in full**. This guarantees you will never miss a crash or fatal error due to active sampling rates.
