# Chapter 6: Instrumenting .NET 8 with OpenTelemetry

> **Part II — The Demo Application**

This is the most important chapter in the book. Everything in Grafana, Prometheus, and Jaeger exists because of the instrumentation you add here. Get this right and the rest flows naturally.

---

## 6.1 NuGet Packages

```xml
<!-- src/Api/Api.csproj -->
<ItemGroup>
  <!-- OpenTelemetry core -->
  <PackageReference Include="OpenTelemetry"                                 Version="1.8.1" />
  <PackageReference Include="OpenTelemetry.Sdk"                             Version="1.8.1" />

  <!-- Tracing exporters -->
  <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.8.1" />

  <!-- Auto-instrumentation sources (traces + metrics) -->
  <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore"      Version="1.8.1" />
  <PackageReference Include="OpenTelemetry.Instrumentation.Http"            Version="1.8.1" />
  <PackageReference Include="OpenTelemetry.Instrumentation.Runtime"         Version="1.8.1" />
  <PackageReference Include="OpenTelemetry.Instrumentation.Process"         Version="0.5.0-beta.6" />
  <PackageReference Include="Npgsql.OpenTelemetry"                          Version="8.0.3" />

  <!-- Metrics: Prometheus exporter -->
  <PackageReference Include="OpenTelemetry.Exporter.Prometheus.AspNetCore"  Version="1.8.0-rc.1" />
  <PackageReference Include="prometheus-net.AspNetCore"                      Version="8.2.1" />

  <!-- Application packages -->
  <PackageReference Include="Npgsql"                                        Version="8.0.3" />
  <PackageReference Include="Dapper"                                        Version="2.1.35" />
  <PackageReference Include="StackExchange.Redis"                           Version="2.7.33" />
  <PackageReference Include="FluentValidation"                              Version="11.9.0" />
</ItemGroup>
```

---

## 6.2 Program.cs — The Complete Wiring

```csharp
// src/Api/Program.cs
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;
using OpenTelemetry.Resources;
using obs_api.Observability;
using obs_api.Endpoints;
using obs_api.Data;

var builder = WebApplication.CreateBuilder(args);

// ── Configuration ──────────────────────────────────────────────────────────
var config = builder.Configuration;
var serviceName    = config["Observability:ServiceName"]    ?? "obs-api";
var jaegerEndpoint = config["Observability:JaegerEndpoint"] ?? "http://localhost:4318";

// ── OpenTelemetry: Resource ─────────────────────────────────────────────────
// The resource identifies this service in all telemetry data.
var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService(
        serviceName:        serviceName,
        serviceVersion:     "1.0.0",
        serviceInstanceId:  Environment.MachineName
    )
    .AddAttributes(new Dictionary<string, object> {
        ["deployment.environment"] = builder.Environment.EnvironmentName,
        ["host.name"]              = Environment.MachineName,
    });

// ── OpenTelemetry: Tracing ──────────────────────────────────────────────────
builder.Services.AddOpenTelemetry()
    .WithTracing(tracerBuilder => tracerBuilder
        .SetResourceBuilder(resourceBuilder)

        // Auto-instrument: every HTTP request gets a span
        .AddAspNetCoreInstrumentation(opts => {
            opts.RecordException       = true;
            opts.EnrichWithHttpRequest = (activity, request) => {
                activity.SetTag("http.client_ip",  request.HttpContext.Connection.RemoteIpAddress?.ToString());
                activity.SetTag("http.user_agent", request.Headers.UserAgent.ToString());
            };
            opts.EnrichWithHttpResponse = (activity, response) => {
                activity.SetTag("http.response_size", response.ContentLength);
            };
            // Filter out health check noise
            opts.Filter = ctx => ctx.Request.Path.Value is not ("/healthz" or "/readyz" or "/metrics");
        })

        // Auto-instrument: every outgoing HttpClient call gets a span
        .AddHttpClientInstrumentation(opts => {
            opts.RecordException = true;
            opts.EnrichWithHttpRequestMessage = (activity, request) => {
                activity.SetTag("http.request.target", request.RequestUri?.PathAndQuery);
            };
        })

        // Auto-instrument: every Npgsql query gets a span
        .AddNpgsql()

        // Custom spans defined in application services
        .AddSource(ObsTelemetry.ActivitySourceName)

        // Export to Jaeger via OTLP HTTP
        .AddOtlpExporter(opts => {
            opts.Endpoint = new Uri(jaegerEndpoint);
            opts.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
        })
    )

    // ── OpenTelemetry: Metrics ──────────────────────────────────────────────
    .WithMetrics(metricsBuilder => metricsBuilder
        .SetResourceBuilder(resourceBuilder)

        // Auto-instrument: HTTP server metrics (request duration, active requests)
        .AddAspNetCoreInstrumentation()

        // Auto-instrument: HTTP client metrics (for external calls)
        .AddHttpClientInstrumentation()

        // Auto-instrument: .NET runtime metrics (GC, thread pool, etc.)
        .AddRuntimeInstrumentation()

        // Auto-instrument: process-level metrics (CPU, memory)
        .AddProcessInstrumentation()

        // Custom application metrics
        .AddMeter(ObsMetrics.MeterName)

        // Views: customize histogram bucket boundaries
        .AddView(
            instrumentName: "http.server.request.duration",
            metricStreamConfiguration: new ExplicitBucketHistogramConfiguration {
                Boundaries = new double[] {
                    0.005, 0.010, 0.025, 0.050, 0.100,
                    0.250, 0.500, 1.000, 2.500, 5.000, 10.000
                }
            }
        )
        .AddView(
            instrumentName: "db.query.duration",
            metricStreamConfiguration: new ExplicitBucketHistogramConfiguration {
                Boundaries = new double[] {
                    0.001, 0.005, 0.010, 0.025, 0.050,
                    0.100, 0.250, 0.500, 1.000, 5.000
                }
            }
        )

        // Export to Prometheus via /metrics endpoint
        .AddPrometheusExporter()
    );

// ── Application Services ───────────────────────────────────────────────────
builder.Services.AddSingleton<ObsMetrics>();
builder.Services.AddSingleton<NpgsqlDataSource>(sp => {
    var connStr = config.GetConnectionString("Postgres")!;
    var ds = new NpgsqlDataSourceBuilder(connStr)
        .EnableParameterLogging(false)
        .Build();
    return ds;
});
builder.Services.AddStackExchangeRedisCache(opts => {
    opts.Configuration = config.GetConnectionString("Redis");
});
builder.Services.AddHttpClient<ExternalApiClient>(c => {
    c.BaseAddress = new Uri(config["ExternalApi:BaseUrl"]!);
    c.Timeout     = TimeSpan.FromSeconds(5);
});
builder.Services.AddScoped<OrderService>();

// ── Build and Configure Pipeline ───────────────────────────────────────────
var app = builder.Build();

// Request pipeline middleware order matters!
app.UseMiddleware<RequestIdMiddleware>();    // Inject X-Request-Id header
app.UseMiddleware<CorrelationMiddleware>();  // Propagate trace context to logs

// Map endpoints
app.MapOrderEndpoints();
app.MapProductEndpoints();
app.MapReportEndpoints();

// Observability endpoints
app.MapPrometheusScrapingEndpoint("/metrics");   // Prometheus scrapes here
app.MapGet("/healthz", () => Results.Ok("healthy"));
app.MapGet("/readyz",  async (NpgsqlDataSource ds) => {
    try {
        await using var conn = await ds.OpenConnectionAsync();
        await conn.ExecuteScalarAsync("SELECT 1");
        return Results.Ok(new { status = "ready", db = "ok" });
    } catch (Exception ex) {
        return Results.ServiceUnavailable(new { status = "not ready", db = ex.Message });
    }
});

await app.RunAsync();
```

---

## 6.3 Custom Metrics Definition

```csharp
// src/Api/Observability/ObsMetrics.cs
using System.Diagnostics.Metrics;

namespace obs_api.Observability;

/// <summary>
/// All custom application metrics in one place.
/// Register as singleton — Meter instances are thread-safe and expensive to create.
/// </summary>
public sealed class ObsMetrics : IDisposable
{
    public const string MeterName = "obs.api";

    private readonly Meter _meter;

    // ── Order Metrics ─────────────────────────────────────────────────────
    public readonly Histogram<double> OrderCreationDuration;
    public readonly Counter<long>     OrdersCreated;
    public readonly Counter<long>     OrderCreationErrors;
    public readonly ObservableGauge<int> PendingOrdersGauge;

    // ── Database Metrics ──────────────────────────────────────────────────
    public readonly Histogram<double> DbQueryDuration;
    public readonly ObservableGauge<int> DbConnectionsActive;
    public readonly Counter<long>     DbQueryErrors;

    // ── Cache Metrics ─────────────────────────────────────────────────────
    public readonly Counter<long> CacheHits;
    public readonly Counter<long> CacheMisses;

    // ── External API Metrics ──────────────────────────────────────────────
    public readonly Histogram<double> ExternalApiDuration;
    public readonly Counter<long>     ExternalApiErrors;
    public readonly Counter<long>     ExternalApiTimeouts;

    public ObsMetrics()
    {
        _meter = new Meter(MeterName, "1.0.0");

        OrderCreationDuration = _meter.CreateHistogram<double>(
            name:        "order.creation.duration",
            unit:        "s",
            description: "End-to-end order creation time including external API call"
        );

        OrdersCreated = _meter.CreateCounter<long>(
            name:        "order.created.total",
            description: "Total number of orders created"
        );

        OrderCreationErrors = _meter.CreateCounter<long>(
            name:        "order.creation.errors.total",
            description: "Total number of order creation failures"
        );

        DbQueryDuration = _meter.CreateHistogram<double>(
            name:        "db.query.duration",
            unit:        "s",
            description: "Duration of database queries by operation type"
        );

        CacheHits = _meter.CreateCounter<long>(
            name:        "cache.hits.total",
            description: "Number of cache hits"
        );

        CacheMisses = _meter.CreateCounter<long>(
            name:        "cache.misses.total",
            description: "Number of cache misses (required a DB query)"
        );

        ExternalApiDuration = _meter.CreateHistogram<double>(
            name:        "external_api.duration",
            unit:        "s",
            description: "External API call latency"
        );

        ExternalApiErrors = _meter.CreateCounter<long>(
            name:        "external_api.errors.total",
            description: "External API call failures"
        );

        ExternalApiTimeouts = _meter.CreateCounter<long>(
            name:        "external_api.timeouts.total",
            description: "External API calls that timed out"
        );
    }

    public void Dispose() => _meter.Dispose();
}
```

---

## 6.4 Custom Tracing

```csharp
// src/Api/Observability/ObsTelemetry.cs
using System.Diagnostics;

namespace obs_api.Observability;

public static class ObsTelemetry
{
    public const string ActivitySourceName = "obs.api";

    private static readonly ActivitySource Source = new(ActivitySourceName, "1.0.0");

    /// <summary>
    /// Start a custom span for an order operation.
    /// The 'using' pattern ensures the span is always ended.
    /// </summary>
    public static Activity? StartOrderActivity(string operation, string orderId)
    {
        var activity = Source.StartActivity(
            name: $"order.{operation}",
            kind: ActivityKind.Internal
        );
        activity?.SetTag("order.id",        orderId);
        activity?.SetTag("order.operation", operation);
        return activity;
    }

    /// <summary>
    /// Add database span attributes — these appear in Jaeger
    /// and allow you to see the exact SQL being executed.
    /// </summary>
    public static void EnrichWithDbStatement(this Activity? activity, string sql, string table)
    {
        activity?.SetTag("db.system",     "postgresql");
        activity?.SetTag("db.statement",  sql.Length > 200 ? sql[..200] + "..." : sql);
        activity?.SetTag("db.table",      table);
    }

    /// <summary>
    /// Mark an activity as failed with structured error information.
    /// </summary>
    public static void RecordException(this Activity? activity, Exception ex, string context = "")
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex, new TagList {
            { "error.context", context },
            { "error.type",    ex.GetType().Name },
        });
    }
}
```

---

## 6.5 The Order Service — Instrumented

```csharp
// src/Api/Services/OrderService.cs
using System.Diagnostics;
using obs_api.Observability;

namespace obs_api.Services;

public class OrderService
{
    private readonly NpgsqlDataSource    _db;
    private readonly IDistributedCache   _cache;
    private readonly ExternalApiClient   _extApi;
    private readonly ObsMetrics          _metrics;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        NpgsqlDataSource db,
        IDistributedCache cache,
        ExternalApiClient extApi,
        ObsMetrics metrics,
        ILogger<OrderService> logger)
    {
        _db      = db;
        _cache   = cache;
        _extApi  = extApi;
        _metrics = metrics;
        _logger  = logger;
    }

    public async Task<Order?> GetOrderAsync(Guid id, CancellationToken ct = default)
    {
        var cacheKey = $"order:{id}";

        // Try cache first — this span appears in Jaeger under the parent HTTP span
        var cached = await _cache.GetStringAsync(cacheKey, ct);
        if (cached is not null) {
            _metrics.CacheHits.Add(1, new("entity", "order"));
            return JsonSerializer.Deserialize<Order>(cached);
        }

        _metrics.CacheMisses.Add(1, new("entity", "order"));

        // Database query — Npgsql instrumentation auto-creates a span for this
        var sw  = Stopwatch.StartNew();
        Order? order;

        await using var conn = await _db.OpenConnectionAsync(ct);
        try {
            order = await conn.QuerySingleOrDefaultAsync<Order>(
                """
                SELECT o.*, json_agg(json_build_object(
                    'productId', oi.product_id,
                    'quantity',  oi.quantity,
                    'unitPrice', oi.unit_price
                )) AS items
                FROM orders o
                LEFT JOIN order_items oi ON oi.order_id = o.id
                WHERE o.id = @id
                GROUP BY o.id
                """,
                new { id }
            );

            _metrics.DbQueryDuration.Record(
                sw.Elapsed.TotalSeconds,
                new("operation", "select_order"),
                new("outcome",   order is null ? "miss" : "hit")
            );
        } catch (Exception ex) {
            _metrics.DbQueryErrors.Add(1, new("operation", "select_order"));
            _logger.LogError(ex, "Failed to load order {OrderId}", id);
            throw;
        }

        if (order is not null) {
            await _cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(order),
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) },
                ct
            );
        }

        return order;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest req, CancellationToken ct = default)
    {
        // Start a custom span for the whole creation workflow.
        // This will contain child spans for: external API, DB insert.
        using var activity = ObsTelemetry.StartOrderActivity("create", "new");
        var sw = Stopwatch.StartNew();

        try {
            // Step 1: Call external API (payment authorization / fraud check)
            // This span is created automatically by HttpClientInstrumentation
            var authResult = await _extApi.AuthorizeOrderAsync(req, ct);
            activity?.SetTag("external.auth.result", authResult.Status);

            if (!authResult.Approved) {
                _metrics.OrderCreationErrors.Add(1, new("reason", "authorization_denied"));
                throw new InvalidOperationException($"Order not authorized: {authResult.Message}");
            }

            // Step 2: Insert into database (Npgsql auto-creates a span)
            await using var conn   = await _db.OpenConnectionAsync(ct);
            await using var tx     = await conn.BeginTransactionAsync(ct);

            var orderId = Guid.NewGuid();
            activity?.SetTag("order.id", orderId.ToString());

            try {
                await conn.ExecuteAsync(
                    "INSERT INTO orders (id, customer_email, status) VALUES (@id, @email, 'pending')",
                    new { id = orderId, email = req.CustomerEmail }, tx
                );

                foreach (var item in req.Items) {
                    await conn.ExecuteAsync(
                        """
                        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
                        SELECT @orderId, p.id, @qty, p.price
                        FROM products p WHERE p.id = @productId
                        """,
                        new { orderId, productId = item.ProductId, qty = item.Quantity }, tx
                    );
                }

                await tx.CommitAsync(ct);
            } catch {
                await tx.RollbackAsync(ct);
                throw;
            }

            var order = await GetOrderAsync(orderId, ct);

            _metrics.OrdersCreated.Add(1,
                new("customer.email.domain",
                    req.CustomerEmail.Split('@').LastOrDefault() ?? "unknown"));

            _metrics.OrderCreationDuration.Record(
                sw.Elapsed.TotalSeconds,
                new("outcome", "success")
            );

            _logger.LogInformation(
                "Order {OrderId} created for {CustomerEmail} in {DurationMs}ms",
                orderId, req.CustomerEmail, sw.ElapsedMilliseconds
            );

            return order!;

        } catch (Exception ex) {
            activity?.RecordException(ex, "create_order");
            _metrics.OrderCreationDuration.Record(sw.Elapsed.TotalSeconds, new("outcome", "error"));
            _logger.LogError(ex, "Order creation failed for {CustomerEmail}", req.CustomerEmail);
            throw;
        }
    }
}
```

---

## 6.6 What Gets Emitted Automatically

After wiring OpenTelemetry as shown above, these metrics appear in Prometheus **without any additional code**:

```
# HTTP server (AspNetCore instrumentation)
http_server_request_duration_seconds_bucket{...}
http_server_active_requests{...}

# HTTP client (HttpClient instrumentation)
http_client_request_duration_seconds_bucket{...}

# .NET runtime (Runtime instrumentation)
dotnet_gc_collections_total{generation="gen0|gen1|gen2"}
dotnet_gc_heap_size_bytes{generation="..."}
dotnet_gc_pause_ratio
dotnet_thread_pool_threads_count
dotnet_thread_pool_queue_length
dotnet_jit_compiled_methods_total

# Process (Process instrumentation)
process_cpu_time_seconds_total
process_working_set_bytes
process_private_memory_bytes
process_open_handles

# Npgsql (database)
db_client_connections_pending_requests{pool_name="..."}
db_client_connections_usage{pool_name="...", state="used|idle"}
db_client_operation_duration_seconds_bucket{...}
```

And these traces appear in Jaeger automatically:
- One root span per HTTP request
- Child span for each database query (SQL text, duration, status)
- Child span for each outgoing HTTP call (URL, method, status)

Plus your custom spans from `ObsTelemetry.StartOrderActivity()` appear as additional children, giving you the complete picture of what the code did.

---

## 6.7 Structured Logging with Trace Correlation

```csharp
// Add to Program.cs — logs include trace_id, enabling correlation with Jaeger
builder.Logging.AddJsonConsole(opts => {
    opts.UseUtcTimestamp  = true;
    opts.IncludeScopes    = true;
    opts.JsonWriterOptions = new JsonWriterOptions { Indented = false };
});

// Middleware that injects trace ID into log scope
public class CorrelationMiddleware : IMiddleware
{
    private readonly ILogger<CorrelationMiddleware> _logger;

    public CorrelationMiddleware(ILogger<CorrelationMiddleware> logger)
        => _logger = logger;

    public async Task InvokeAsync(HttpContext ctx, RequestDelegate next)
    {
        var traceId = Activity.Current?.TraceId.ToString() ?? "none";
        var spanId  = Activity.Current?.SpanId.ToString()  ?? "none";

        using (_logger.BeginScope(new Dictionary<string, object> {
            ["trace_id"] = traceId,
            ["span_id"]  = spanId,
            ["request_id"] = ctx.TraceIdentifier,
        })) {
            // Every log statement within this request will include trace_id.
            // In Grafana Loki, you can click trace_id to jump to Jaeger trace.
            await next(ctx);
        }
    }
}
```

Now every log line looks like:
```json
{
  "Timestamp": "2025-06-15T10:23:45.123Z",
  "Level": "Information",
  "Message": "Order ord-abc123 created in 847ms",
  "trace_id": "4f8b9a2c1d3e5f6a7b8c9d0e1f2a3b4c",
  "span_id": "e4f5a6b7c8d9e0f1",
  "request_id": "0HN3JKQB5KFAG:00000001"
}
```

This `trace_id` is the same ID that appears in Jaeger. In Grafana, clicking the trace_id in a log line opens the exact trace in Jaeger. This is the **pillar correlation** that makes observability 10x more powerful than isolated tools.
