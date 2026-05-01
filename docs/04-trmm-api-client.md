---
layout: default
title: 4. Layer 1 — TrmmApiClient
nav_order: 5
permalink: /docs/trmm-api-client/
---

# 4. Layer 1 — TrmmApiClient
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 4.1 Tujuan layer ini

`TrmmApiClient` adalah satu-satunya pintu gerbang antara aplikasi desktop dan TRMM backend. Semua interaksi (read state, run script, request enrollment) lewat sini.

**Prinsip:**

1. **Strongly typed** — tidak ada `dynamic`, tidak ada `JObject`. Semua response = DTO record.
2. **Tidak ada side effect lokal** — class ini hanya memanggil HTTP, tidak modify state lokal, tidak menulis file.
3. **Async-first** — semua method return `Task<T>` atau `IAsyncEnumerable<T>`.
4. **Cancellation support** — semua method terima `CancellationToken`.
5. **Authenticated** — header `X-API-KEY` di-inject otomatis.

## 4.2 Struktur folder

```
HermesNetwork/
└── Trmm/
    ├── TrmmApiClient.cs               ← class utama
    ├── TrmmApiClientOptions.cs        ← configuration
    ├── ITrmmApiClient.cs              ← interface untuk DI/mock
    ├── Models/
    │   ├── AgentDto.cs
    │   ├── AgentSummaryDto.cs
    │   ├── DeploymentDto.cs
    │   ├── ScriptDto.cs
    │   ├── ScriptRunDto.cs
    │   ├── CheckDto.cs
    │   └── ClientDto.cs
    └── Exceptions/
        ├── TrmmApiException.cs
        └── TrmmAuthException.cs
```

## 4.3 DTO models

### 4.3.1 `AgentDto.cs`

```csharp
using System;
using System.Text.Json.Serialization;

namespace HermesNetwork.Trmm.Models;

public sealed record AgentDto(
    [property: JsonPropertyName("agent_id")]    string AgentId,
    [property: JsonPropertyName("hostname")]    string Hostname,
    [property: JsonPropertyName("client")]      ClientSummaryDto? Client,
    [property: JsonPropertyName("site")]        SiteSummaryDto? Site,
    [property: JsonPropertyName("plat")]        string Platform,         // "windows", "darwin", "linux"
    [property: JsonPropertyName("operating_system")] string? OsName,
    [property: JsonPropertyName("version")]     string AgentVersion,
    [property: JsonPropertyName("last_seen")]   DateTimeOffset? LastSeen,
    [property: JsonPropertyName("status")]      string Status,           // "online" | "offline" | "overdue"
    [property: JsonPropertyName("logged_in_username")] string? LoggedInUser,
    [property: JsonPropertyName("public_ip")]   string? PublicIp,
    [property: JsonPropertyName("local_ip")]    string? LocalIp,
    [property: JsonPropertyName("cpu_model")]   string? CpuModel,
    [property: JsonPropertyName("total_ram")]   double? TotalRamGb,
    [property: JsonPropertyName("disks")]       DiskInfoDto[]? Disks
);

public sealed record ClientSummaryDto(
    [property: JsonPropertyName("id")]   int Id,
    [property: JsonPropertyName("name")] string Name);

public sealed record SiteSummaryDto(
    [property: JsonPropertyName("id")]   int Id,
    [property: JsonPropertyName("name")] string Name);

public sealed record DiskInfoDto(
    [property: JsonPropertyName("device")]  string Device,
    [property: JsonPropertyName("free")]    string Free,
    [property: JsonPropertyName("total")]   string Total,
    [property: JsonPropertyName("percent")] int PercentUsed);
```

### 4.3.2 `DeploymentDto.cs`

```csharp
using System;
using System.Text.Json.Serialization;

namespace HermesNetwork.Trmm.Models;

public sealed record DeploymentDto(
    [property: JsonPropertyName("id")]            int Id,
    [property: JsonPropertyName("uuid")]          string Uuid,
    [property: JsonPropertyName("client")]        int ClientId,
    [property: JsonPropertyName("site")]          int SiteId,
    [property: JsonPropertyName("expiry")]        DateTimeOffset ExpiresAt,
    [property: JsonPropertyName("install_flags")] string? InstallFlags,
    [property: JsonPropertyName("download_url")]  string DownloadUrl
);

public sealed record CreateDeploymentRequest(
    [property: JsonPropertyName("client")]        int ClientId,
    [property: JsonPropertyName("site")]          int SiteId,
    [property: JsonPropertyName("expires")]       DateTimeOffset Expires,
    [property: JsonPropertyName("arch")]          string Arch,        // "amd64", "arm64", "amd64-mac"
    [property: JsonPropertyName("install_flags")] string? InstallFlags = null
);
```

### 4.3.3 `ScriptDto.cs` & `ScriptRunDto.cs`

```csharp
using System.Text.Json.Serialization;

namespace HermesNetwork.Trmm.Models;

public sealed record ScriptDto(
    [property: JsonPropertyName("id")]          int Id,
    [property: JsonPropertyName("name")]        string Name,
    [property: JsonPropertyName("shell")]       string Shell,           // "powershell" | "cmd" | "python" | "bash"
    [property: JsonPropertyName("description")] string? Description
);

public sealed record ScriptRunRequest(
    [property: JsonPropertyName("script")]   int ScriptId,
    [property: JsonPropertyName("output")]   string Output = "wait",    // "wait" | "forget" | "collector"
    [property: JsonPropertyName("args")]     string[]? Args = null,
    [property: JsonPropertyName("timeout")]  int TimeoutSeconds = 90
);

public sealed record ScriptRunDto(
    [property: JsonPropertyName("execution_time")] string? ExecutionTime,
    [property: JsonPropertyName("retcode")]        int? ReturnCode,
    [property: JsonPropertyName("stdout")]         string? Stdout,
    [property: JsonPropertyName("stderr")]         string? Stderr
);
```

### 4.3.4 `CheckDto.cs`

```csharp
using System;
using System.Text.Json.Serialization;

namespace HermesNetwork.Trmm.Models;

public sealed record CheckDto(
    [property: JsonPropertyName("id")]          int Id,
    [property: JsonPropertyName("check_type")]  string CheckType,        // "diskspace", "ping", "cpuload", "memory", "winsvc", "script", "eventlog"
    [property: JsonPropertyName("name")]        string Name,
    [property: JsonPropertyName("status")]      string Status,           // "passing" | "failing" | "pending"
    [property: JsonPropertyName("more_info")]   string? MoreInfo,
    [property: JsonPropertyName("last_run")]    DateTimeOffset? LastRun
);
```

## 4.4 Exception classes

```csharp
namespace HermesNetwork.Trmm.Exceptions;

public class TrmmApiException : Exception
{
    public int StatusCode { get; }
    public string? ResponseBody { get; }

    public TrmmApiException(int statusCode, string? body, string message)
        : base(message)
    {
        StatusCode = statusCode;
        ResponseBody = body;
    }
}

public sealed class TrmmAuthException : TrmmApiException
{
    public TrmmAuthException(string? body)
        : base(401, body, "TRMM authentication failed (invalid or expired API key)")
    { }
}
```

## 4.5 Configuration class

```csharp
namespace HermesNetwork.Trmm;

public sealed class TrmmApiClientOptions
{
    public required string BaseUrl { get; init; }
    public required string ApiKey { get; init; }
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
    public int MaxRetries { get; init; } = 3;
}
```

## 4.6 Interface untuk DI / mock

```csharp
using System.Threading;
using System.Threading.Tasks;
using HermesNetwork.Trmm.Models;

namespace HermesNetwork.Trmm;

public interface ITrmmApiClient
{
    // Read
    Task<AgentDto?> GetAgentAsync(int agentId, CancellationToken ct = default);
    Task<AgentDto?> GetAgentByHostnameAsync(string hostname, CancellationToken ct = default);
    Task<AgentDto[]> ListAgentsAsync(int? clientId = null, int? siteId = null, CancellationToken ct = default);
    Task<CheckDto[]> GetChecksAsync(int agentId, CancellationToken ct = default);
    Task<ScriptDto[]> ListScriptsAsync(CancellationToken ct = default);

    // Command
    Task<ScriptRunDto> RunScriptAsync(int agentId, ScriptRunRequest request, CancellationToken ct = default);
    Task<DeploymentDto> CreateDeploymentAsync(CreateDeploymentRequest request, CancellationToken ct = default);
    Task DeleteAgentAsync(int agentId, CancellationToken ct = default);
}
```

## 4.7 Implementasi `TrmmApiClient`

```csharp
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Net.Http.Json;
using System.Threading;
using System.Threading.Tasks;
using HermesNetwork.Trmm.Exceptions;
using HermesNetwork.Trmm.Models;

namespace HermesNetwork.Trmm;

public sealed class TrmmApiClient : ITrmmApiClient, IDisposable
{
    private readonly HttpClient _http;
    private readonly bool _ownsHttpClient;

    /// <summary>
    /// Constructor untuk DI: HttpClient di-inject dari outside (pakai IHttpClientFactory).
    /// </summary>
    public TrmmApiClient(HttpClient http, TrmmApiClientOptions options)
    {
        _http = http;
        _http.BaseAddress = new Uri(options.BaseUrl.TrimEnd('/') + "/");
        _http.Timeout = options.Timeout;
        _http.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/json"));
        _http.DefaultRequestHeaders.Add("X-API-KEY", options.ApiKey);
        _ownsHttpClient = false;
    }

    /// <summary>
    /// Convenience constructor untuk standalone use (test, scripts).
    /// </summary>
    public TrmmApiClient(TrmmApiClientOptions options)
        : this(new HttpClient(), options)
    {
        _ownsHttpClient = true;
    }

    // ---------------- Read endpoints ----------------

    public async Task<AgentDto?> GetAgentAsync(int agentId, CancellationToken ct = default)
    {
        using var resp = await _http.GetAsync($"api/v3/agents/{agentId}/", ct);
        if (resp.StatusCode == HttpStatusCode.NotFound) return null;
        await EnsureSuccessAsync(resp, ct);
        return await resp.Content.ReadFromJsonAsync<AgentDto>(cancellationToken: ct);
    }

    public async Task<AgentDto?> GetAgentByHostnameAsync(string hostname, CancellationToken ct = default)
    {
        var encoded = Uri.EscapeDataString(hostname);
        using var resp = await _http.GetAsync($"api/v3/agents/?hostname={encoded}", ct);
        await EnsureSuccessAsync(resp, ct);
        var list = await resp.Content.ReadFromJsonAsync<AgentDto[]>(cancellationToken: ct);
        return list is { Length: > 0 } ? list[0] : null;
    }

    public async Task<AgentDto[]> ListAgentsAsync(int? clientId = null, int? siteId = null, CancellationToken ct = default)
    {
        var qs = new List<string>();
        if (clientId.HasValue) qs.Add($"client={clientId.Value}");
        if (siteId.HasValue)   qs.Add($"site={siteId.Value}");
        var query = qs.Count > 0 ? "?" + string.Join("&", qs) : "";

        using var resp = await _http.GetAsync($"api/v3/agents/{query}", ct);
        await EnsureSuccessAsync(resp, ct);
        return await resp.Content.ReadFromJsonAsync<AgentDto[]>(cancellationToken: ct)
               ?? Array.Empty<AgentDto>();
    }

    public async Task<CheckDto[]> GetChecksAsync(int agentId, CancellationToken ct = default)
    {
        using var resp = await _http.GetAsync($"api/v3/checks/?agent={agentId}", ct);
        await EnsureSuccessAsync(resp, ct);
        return await resp.Content.ReadFromJsonAsync<CheckDto[]>(cancellationToken: ct)
               ?? Array.Empty<CheckDto>();
    }

    public async Task<ScriptDto[]> ListScriptsAsync(CancellationToken ct = default)
    {
        using var resp = await _http.GetAsync("api/v3/scripts/", ct);
        await EnsureSuccessAsync(resp, ct);
        return await resp.Content.ReadFromJsonAsync<ScriptDto[]>(cancellationToken: ct)
               ?? Array.Empty<ScriptDto>();
    }

    // ---------------- Command endpoints ----------------

    public async Task<ScriptRunDto> RunScriptAsync(int agentId, ScriptRunRequest request, CancellationToken ct = default)
    {
        using var resp = await _http.PostAsJsonAsync(
            $"api/v3/agents/{agentId}/runscript/", request, ct);
        await EnsureSuccessAsync(resp, ct);
        return (await resp.Content.ReadFromJsonAsync<ScriptRunDto>(cancellationToken: ct))!;
    }

    public async Task<DeploymentDto> CreateDeploymentAsync(CreateDeploymentRequest request, CancellationToken ct = default)
    {
        using var resp = await _http.PostAsJsonAsync(
            "api/v3/agents/deployments/", request, ct);
        await EnsureSuccessAsync(resp, ct);
        return (await resp.Content.ReadFromJsonAsync<DeploymentDto>(cancellationToken: ct))!;
    }

    public async Task DeleteAgentAsync(int agentId, CancellationToken ct = default)
    {
        using var resp = await _http.DeleteAsync($"api/v3/agents/{agentId}/", ct);
        await EnsureSuccessAsync(resp, ct);
    }

    // ---------------- helpers ----------------

    private static async Task EnsureSuccessAsync(HttpResponseMessage resp, CancellationToken ct)
    {
        if (resp.IsSuccessStatusCode) return;

        string? body = null;
        try { body = await resp.Content.ReadAsStringAsync(ct); } catch { }

        if (resp.StatusCode == HttpStatusCode.Unauthorized ||
            resp.StatusCode == HttpStatusCode.Forbidden)
        {
            throw new TrmmAuthException(body);
        }

        throw new TrmmApiException(
            (int)resp.StatusCode, body,
            $"TRMM API call failed: {(int)resp.StatusCode} {resp.ReasonPhrase}");
    }

    public void Dispose()
    {
        if (_ownsHttpClient) _http.Dispose();
    }
}
```

## 4.8 Setup retry + circuit breaker dengan Polly

Untuk menangani 5xx / network blip, wrap di Polly. Tambahkan ke startup app:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Http;
using Polly;
using Polly.Extensions.Http;

services.AddHttpClient<ITrmmApiClient, TrmmApiClient>()
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy() =>
    HttpPolicyExtensions
        .HandleTransientHttpError()                    // 5xx, 408
        .OrResult(r => (int)r.StatusCode == 429)
        .WaitAndRetryAsync(3, attempt =>
            TimeSpan.FromMilliseconds(Math.Pow(2, attempt) * 250));   // 500ms, 1s, 2s

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy() =>
    HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30));
```

## 4.9 Contoh penggunaan dari ViewModel

```csharp
public sealed class RmmStatusViewModel : ViewModelBase
{
    private readonly ITrmmApiClient _trmm;
    private readonly ILogger<RmmStatusViewModel> _log;

    [ObservableProperty] private string? _status;
    [ObservableProperty] private string? _hostname;
    [ObservableProperty] private DateTimeOffset? _lastSeen;
    [ObservableProperty] private CheckDto[] _checks = Array.Empty<CheckDto>();

    public RmmStatusViewModel(ITrmmApiClient trmm, ILogger<RmmStatusViewModel> log)
    {
        _trmm = trmm; _log = log;
    }

    public async Task RefreshAsync(int agentId, CancellationToken ct = default)
    {
        try
        {
            var agent = await _trmm.GetAgentAsync(agentId, ct);
            if (agent is null)
            {
                Status = "Agent not registered";
                return;
            }

            Hostname = agent.Hostname;
            LastSeen = agent.LastSeen;
            Status   = agent.Status;

            Checks = await _trmm.GetChecksAsync(agentId, ct);
        }
        catch (TrmmAuthException)
        {
            Status = "Authentication expired — please re-enroll";
        }
        catch (TrmmApiException ex)
        {
            _log.LogError(ex, "TRMM API call failed");
            Status = $"Error: {ex.Message}";
        }
    }
}
```

## 4.10 Unit testing

`TrmmApiClient` mudah di-test karena dia bergantung hanya pada `HttpClient`. Pakai `HttpMessageHandler` mock:

```csharp
public class TrmmApiClientTests
{
    [Fact]
    public async Task GetAgentAsync_ReturnsNull_OnNotFound()
    {
        var handler = new MockHttpMessageHandler();
        handler.When("https://test/api/v3/agents/999/")
               .Respond(HttpStatusCode.NotFound);

        using var http = new HttpClient(handler);
        var client = new TrmmApiClient(http, new TrmmApiClientOptions
        {
            BaseUrl = "https://test", ApiKey = "x"
        });

        var result = await client.GetAgentAsync(999);
        result.Should().BeNull();
    }

    [Fact]
    public async Task GetAgentByHostnameAsync_ReturnsFirstMatch()
    {
        var handler = new MockHttpMessageHandler();
        handler.When("https://test/api/v3/agents/")
               .WithQueryString("hostname=JADE-PC")
               .Respond("application/json", """
                   [{"agent_id":"abc","hostname":"JADE-PC","plat":"windows",
                     "version":"2.5.0","status":"online"}]
                   """);

        using var http = new HttpClient(handler);
        var client = new TrmmApiClient(http, new TrmmApiClientOptions
        {
            BaseUrl = "https://test", ApiKey = "x"
        });

        var agent = await client.GetAgentByHostnameAsync("JADE-PC");
        agent!.Hostname.Should().Be("JADE-PC");
    }
}
```

NuGet untuk mock: `RichardSzalay.MockHttp`.

## 4.11 Best practices

- **Jangan instansi ulang `HttpClient`** untuk tiap call — pakai `IHttpClientFactory` (atau singleton client)
- **Jangan log API key** di error handler — pastikan `ResponseBody` tidak berisi header
- **Selalu pass `CancellationToken`** dari ViewModel — kalau user navigate ke tab lain, batalkan request
- **Gunakan `using` untuk `HttpResponseMessage`** — dispose response, jangan biarkan leak
- **Validasi response code** sebelum deserialize — `EnsureSuccessAsync` di atas sudah handle
- **Test `TrmmAuthException` path** — UI harus tampilkan "re-enroll" prompt saat 401/403

---

[← Bab 3 Prasyarat]({% link docs/03-prasyarat.md %}){: .btn }
[Bab 5 — AgentSupervisor →]({% link docs/05-agent-supervisor.md %}){: .btn .btn-primary }
