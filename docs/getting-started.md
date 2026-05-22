# Getting Started with AsGuard

Setting up AsGuard in your ASP.NET Core application takes less than 5 minutes. This guide will walk you through the package installation, basic setup, middleware placement, and dashboard verification.

---

## 1. Installation

AsGuard supports **.NET 8, .NET 9, and .NET 10**.

Install the package in your Web API, Minimal API, or MVC project using the NuGet Package Manager Console or the .NET CLI:

```bash
dotnet add package AsGuard
```

---

## 2. Minimal Setup Example

For a fast start, you can configure AsGuard to use a local **SQLite** database. 

Open your application's `Program.cs` and configure the services and middleware:

```csharp
using AsGuard.Extensions;
using AsGuard.Domain.RequestLogging;
using Microsoft.Extensions.Logging;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();

// Add AsGuard request logging services
builder.Services.AddRequestLogging(options =>
{
    // Configure SQLite storage (automatically creates tables)
    options.DatabaseProvider = LoggingDatabaseProvider.Sqlite;
    options.ConnectionString = "Data Source=asguard_monitoring.db";

    // Set up secure credentials for the dashboard (Basic Authentication)
    // IMPORTANT: Rejects default admin/admin. Always use unique production credentials.
    options.DashboardRoute = "/request-logs-ui";
    options.DashboardUsername = builder.Configuration["AsGuard:Username"] ?? "asguard_admin";
    options.DashboardPassword = builder.Configuration["AsGuard:Password"] ?? "superSecretPassword123!";

    // Enable detailed error capture pipelines
    options.EnableExceptionLogging = true;
    options.CaptureHostLogs = true;
    options.HostLogMinimumLevel = LogLevel.Warning;
});

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

app.UseHttpsRedirection();

// REGISTER ASGUARD MIDDLEWARE
// It is critical to place UseRequestLogging() after routing/exception handlers
// but before endpoints/authorization.
app.UseRequestLogging();

app.UseAuthorization();

// Sample Endpoint to generate normal HTTP logs
app.MapGet("/api/greet", (string? name) => 
    Results.Ok(new { message = $"Hello, {name ?? "Guest"}!" }));

// Sample Endpoint to trigger host ILogger Warnings
app.MapGet("/api/warn", (ILogger<Program> logger) =>
{
    logger.LogWarning("This is a warning event captured by AsGuard.");
    return Results.Ok(new { status = "Warning registered" });
});

// Sample Endpoint to trigger Unhandled Exceptions
app.MapGet("/api/error", () =>
{
    throw new InvalidOperationException("This exception is captured and logged automatically by AsGuard!");
});

app.MapControllers();

app.Run();
```

---

## 3. Middleware Ordering Guide

The placement of `app.UseRequestLogging()` in the middleware pipeline is crucial. 

AsGuard sets up several diagnostic and monitoring services under a single method call:
- Correlation ID propagation.
- Host logging scope mapping.
- Handled and unhandled exception interceptors.
- HTTP Request/Response details and body capture.
- Dashboard/API endpoint endpoints mapping.

To ensure all requests are logged correctly and exceptions are accurately associated with their HTTP contexts, follow this pipeline order:

| Order | Middleware Method | Why it belongs here |
| :---: | :--- | :--- |
| **1** | `app.UseExceptionHandler(...)` | If you use standard exception pages or handlers, register them *first*. This allows AsGuard to catch the exception, write it to the database, and let the outer handler generate the client HTTP response. |
| **2** | `app.UseCors(...)` / `app.UseHsts()` | Security and policy middlewares should execute early before any request logging starts. |
| **3** | `app.UseHttpsRedirection()` | Ensures requests are redirected to secure endpoints before AsGuard logs them (preventing duplicate logging of HTTP/HTTPS). |
| **4** | **`app.UseRequestLogging()`** | **REGISTER ASGUARD HERE**. Places request logging early in the pipeline to capture full duration, body streams, and errors from all route endpoints. |
| **5** | `app.UseAuthentication()` | Placing Authentication *after* `UseRequestLogging` allows AsGuard to log requests that fail with `401 Unauthorized` responses. |
| **6** | `app.UseAuthorization()` | Captures requests that fail with `403 Forbidden` responses. |
| **7** | `app.MapControllers()` / Endpoints | The actual controller actions where your business logic runs. |

### Visual Pipeline Flow:
```text
  [HTTP Request In]
         │
         ▼
  ┌──────────────┐
  │ Exception    │  ◄── Hooked to catch unhandled errors
  │ Handler      │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Https / CORS │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ AsGuard      │  ◄── Correlation ID generated, Scope set up,
  │ Middleware   │      and request/response streaming opened.
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Auth & Authz │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Endpoints /  │  ◄── Your API controllers execute. 
  │ Controllers  │      Warnings/Exceptions are logged here.
  └──────────────┘
```

---

## 4. Verifying Your Setup

Once your application is running locally:

### Step A: visiting the Dashboard
Open your browser and navigate to the dashboard route configured in your settings:
```text
https://localhost:<port>/request-logs-ui
```
*   Your browser will prompt you for Basic Authentication credentials.
*   Enter the `DashboardUsername` (`asguard_admin`) and `DashboardPassword` (`superSecretPassword123!`) defined in your configuration.
*   The empty dashboard should load in either Light or Dark mode (auto-detected based on browser preferences or manually togglable).

### Step B: Generating HTTP Traffic
Open another tab or use an API client (like Postman or curl) to hit your endpoints:
1.  **Normal Request**: Navigate to `https://localhost:<port>/api/greet?name=Mahmood`. 
2.  **Warning Event**: Navigate to `https://localhost:<port>/api/warn`. This writes a `Warning` log to `ILogger`.
3.  **Unhandled Exception**: Navigate to `https://localhost:<port>/api/error`. This will show your standard error page, but AsGuard will capture the full stack trace behind the scenes.

### Step C: Inspecting the Logs
Return to the Dashboard tab.
*   You will see new HTTP requests listed in the **Request Logs** grid immediately.
*   Click **Exceptions / Logs** in the navigation bar. You will see both the `Warning` from `/api/warn` and the unhandled `InvalidOperationException` from `/api/error` listed.
*   Click on any row to expand the details, revealing execution duration, correlation IDs, exception details, and system metadata.

---

## Next Steps

Now that your base installation is working, proceed to configure production-ready storage providers, queues, and retention workers:

*   **[Configuration & Options Reference](configuration.md)**
