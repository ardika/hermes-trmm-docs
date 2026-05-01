---
layout: default
title: 5. Layer 2 — AgentSupervisor
nav_order: 6
permalink: /docs/agent-supervisor/
---

# 5. Layer 2 — AgentSupervisor
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 5.1 Tujuan

`AgentSupervisor` adalah abstraksi cross-platform untuk **lifecycle service di OS lokal**. Tugasnya:

1. Menjawab "apakah service `tacticalagent` lagi running?"
2. Start / Stop service
3. Install / Uninstall agent
4. Elevate privilege (UAC di Windows, sudo/osascript di Mac) saat butuh

**Yang TIDAK menjadi tugas layer ini:**

- Tahu detail TRMM API (itu Layer 1)
- Mengambil keputusan bisnis (itu ViewModel)
- Logging ke server (itu logger berbeda)

## 5.2 Interface

```csharp
using System.Threading;
using System.Threading.Tasks;

namespace HermesNetwork.Supervisor;

public interface IAgentSupervisor
{
    /// <summary>Service name di OS (mis. "tacticalrmm" di Windows, "com.tacticalrmm.tacticalagent" di Mac).</summary>
    Task<ServiceState> GetStateAsync(string serviceName, CancellationToken ct = default);

    /// <summary>Start service. No-op kalau sudah running.</summary>
    Task StartAsync(string serviceName, CancellationToken ct = default);

    /// <summary>Stop service. No-op kalau sudah stopped.</summary>
    Task StopAsync(string serviceName, CancellationToken ct = default);

    /// <summary>Install agent dari installer file. Mungkin minta elevation.</summary>
    Task InstallAsync(InstallRequest request, CancellationToken ct = default);

    /// <summary>Uninstall agent. Mungkin minta elevation.</summary>
    Task UninstallAsync(string serviceName, CancellationToken ct = default);

    /// <summary>True kalau process berjalan dengan privilege admin/root.</summary>
    bool IsElevated { get; }
}

public enum ServiceState
{
    NotInstalled,
    Stopped,
    Starting,
    Running,
    Stopping,
    Unknown
}

public sealed record InstallRequest(
    string InstallerPath,        // path absolut ke .msi atau .pkg
    string Arguments,            // CLI args yang dikirim ke installer
    bool RequiresElevation = true,
    int TimeoutSeconds = 300
);
```

## 5.3 Pemilihan implementasi runtime

```csharp
namespace HermesNetwork.Supervisor;

public static class AgentSupervisorFactory
{
    public static IAgentSupervisor Create()
    {
        if (OperatingSystem.IsWindows())
            return new Windows.WindowsAgentSupervisor();
        if (OperatingSystem.IsMacOS())
            return new Mac.MacAgentSupervisor();

        throw new PlatformNotSupportedException(
            "AgentSupervisor only supports Windows and macOS.");
    }
}
```

Atau lebih bersih, register via DI:

```csharp
if (OperatingSystem.IsWindows())
    services.AddSingleton<IAgentSupervisor, WindowsAgentSupervisor>();
else if (OperatingSystem.IsMacOS())
    services.AddSingleton<IAgentSupervisor, MacAgentSupervisor>();
```

## 5.4 Implementasi Windows

```csharp
using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.Versioning;
using System.Security.Principal;
using System.ServiceProcess;
using System.Threading;
using System.Threading.Tasks;

namespace HermesNetwork.Supervisor.Windows;

[SupportedOSPlatform("windows")]
public sealed class WindowsAgentSupervisor : IAgentSupervisor
{
    public bool IsElevated
    {
        get
        {
            using var identity = WindowsIdentity.GetCurrent();
            var principal = new WindowsPrincipal(identity);
            return principal.IsInRole(WindowsBuiltInRole.Administrator);
        }
    }

    public Task<ServiceState> GetStateAsync(string serviceName, CancellationToken ct = default)
    {
        try
        {
            using var sc = new ServiceController(serviceName);
            return Task.FromResult(sc.Status switch
            {
                ServiceControllerStatus.Running        => ServiceState.Running,
                ServiceControllerStatus.Stopped        => ServiceState.Stopped,
                ServiceControllerStatus.StartPending   => ServiceState.Starting,
                ServiceControllerStatus.StopPending    => ServiceState.Stopping,
                _                                       => ServiceState.Unknown,
            });
        }
        catch (InvalidOperationException)
        {
            return Task.FromResult(ServiceState.NotInstalled);
        }
    }

    public async Task StartAsync(string serviceName, CancellationToken ct = default)
    {
        using var sc = new ServiceController(serviceName);
        if (sc.Status == ServiceControllerStatus.Running) return;

        sc.Start();
        await WaitForStatusAsync(sc, ServiceControllerStatus.Running, ct);
    }

    public async Task StopAsync(string serviceName, CancellationToken ct = default)
    {
        using var sc = new ServiceController(serviceName);
        if (sc.Status == ServiceControllerStatus.Stopped) return;

        sc.Stop();
        await WaitForStatusAsync(sc, ServiceControllerStatus.Stopped, ct);
    }

    public async Task InstallAsync(InstallRequest request, CancellationToken ct = default)
    {
        if (!File.Exists(request.InstallerPath))
            throw new FileNotFoundException("Installer not found", request.InstallerPath);

        var ext = Path.GetExtension(request.InstallerPath).ToLowerInvariant();
        var psi = ext switch
        {
            ".msi" => new ProcessStartInfo("msiexec.exe",
                $"/i \"{request.InstallerPath}\" /qn /norestart {request.Arguments}"),

            ".exe" => new ProcessStartInfo(request.InstallerPath, request.Arguments),

            _ => throw new NotSupportedException(
                $"Installer extension {ext} not supported. Expected .msi or .exe.")
        };

        psi.UseShellExecute = true;
        if (request.RequiresElevation && !IsElevated)
            psi.Verb = "runas";   // memunculkan UAC prompt

        using var process = Process.Start(psi)!;
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(request.TimeoutSeconds));

        await process.WaitForExitAsync(cts.Token);

        if (process.ExitCode != 0)
            throw new InvalidOperationException(
                $"Installer exited with code {process.ExitCode}.");
    }

    public async Task UninstallAsync(string serviceName, CancellationToken ct = default)
    {
        // Stop service dulu sebelum uninstall
        try { await StopAsync(serviceName, ct); } catch { /* ignore */ }

        // Uninstall via WMI / sc.exe / vendor-specific. Untuk TRMM:
        var psi = new ProcessStartInfo("cmd.exe", "/c sc delete " + serviceName)
        {
            UseShellExecute = true, Verb = "runas",
            CreateNoWindow = true, WindowStyle = ProcessWindowStyle.Hidden,
        };
        using var p = Process.Start(psi);
        if (p is not null) await p.WaitForExitAsync(ct);
    }

    private static async Task WaitForStatusAsync(
        ServiceController sc, ServiceControllerStatus target, CancellationToken ct)
    {
        var deadline = DateTime.UtcNow.AddSeconds(60);
        while (DateTime.UtcNow < deadline)
        {
            ct.ThrowIfCancellationRequested();
            sc.Refresh();
            if (sc.Status == target) return;
            await Task.Delay(500, ct);
        }
        throw new TimeoutException(
            $"Service {sc.ServiceName} did not reach {target} within 60s.");
    }
}
```

## 5.5 Implementasi macOS

```csharp
using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.Versioning;
using System.Threading;
using System.Threading.Tasks;

namespace HermesNetwork.Supervisor.Mac;

[SupportedOSPlatform("macos")]
public sealed class MacAgentSupervisor : IAgentSupervisor
{
    /// <summary>
    /// Di Mac, "elevated" = process running sebagai root (UID 0).
    /// Tapi UI biasanya jalan sebagai user biasa, dan elevasi terjadi per-action via osascript.
    /// </summary>
    public bool IsElevated => Environment.UserName == "root";

    public async Task<ServiceState> GetStateAsync(string serviceName, CancellationToken ct = default)
    {
        // launchctl list akan return entry kalau service loaded;
        // exit 0 = found, exit 113 = not loaded
        var (code, stdout, _) = await RunAsync("launchctl", $"list {serviceName}", ct);

        if (code != 0) return ServiceState.NotInstalled;

        // Parse "PID = <int>" — kalau bukan integer berarti loaded tapi tidak running
        if (TryExtractPid(stdout, out var pid))
            return pid > 0 ? ServiceState.Running : ServiceState.Stopped;

        return ServiceState.Unknown;
    }

    public Task StartAsync(string serviceName, CancellationToken ct = default) =>
        RunWithElevationAsync($"launchctl start {serviceName}", ct);

    public Task StopAsync(string serviceName, CancellationToken ct = default) =>
        RunWithElevationAsync($"launchctl stop {serviceName}", ct);

    public async Task InstallAsync(InstallRequest request, CancellationToken ct = default)
    {
        if (!File.Exists(request.InstallerPath))
            throw new FileNotFoundException("Installer not found", request.InstallerPath);

        var ext = Path.GetExtension(request.InstallerPath).ToLowerInvariant();
        if (ext != ".pkg")
            throw new NotSupportedException(
                $"Installer extension {ext} not supported on macOS. Expected .pkg.");

        // installer command harus jalan sebagai root
        var cmd = $"installer -pkg \"{request.InstallerPath}\" -target / {request.Arguments}";
        await RunWithElevationAsync(cmd, ct, request.TimeoutSeconds);
    }

    public async Task UninstallAsync(string serviceName, CancellationToken ct = default)
    {
        // 1. Stop dulu
        try { await StopAsync(serviceName, ct); } catch { }

        // 2. Unload LaunchDaemon
        var plistPath = $"/Library/LaunchDaemons/{serviceName}.plist";
        await RunWithElevationAsync($"launchctl unload {plistPath}", ct);

        // 3. Hapus binary + plist (vendor-specific path)
        await RunWithElevationAsync(
            $"rm -f {plistPath} && rm -rf /usr/local/mesh_agent /usr/local/tacticalagent",
            ct);
    }

    // ---------------- helpers ----------------

    /// <summary>
    /// Jalankan shell command dengan administrator privileges via osascript.
    /// Memunculkan dialog "Allow ... to make changes" pada user pertama kali per session.
    /// </summary>
    private async Task RunWithElevationAsync(string shellCmd, CancellationToken ct, int timeoutSeconds = 60)
    {
        // Escape double-quotes for AppleScript
        var escaped = shellCmd.Replace("\"", "\\\"");
        var script = $"do shell script \"{escaped}\" with administrator privileges";

        var (code, _, stderr) = await RunAsync("osascript", $"-e '{script}'", ct, timeoutSeconds);
        if (code != 0)
            throw new InvalidOperationException(
                $"Elevated command failed (exit {code}): {stderr}");
    }

    private static async Task<(int Code, string Stdout, string Stderr)> RunAsync(
        string file, string args, CancellationToken ct, int timeoutSeconds = 30)
    {
        var psi = new ProcessStartInfo(file, args)
        {
            RedirectStandardOutput = true,
            RedirectStandardError  = true,
            UseShellExecute = false,
            CreateNoWindow  = true,
        };

        using var p = Process.Start(psi)!;
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(timeoutSeconds));

        var stdoutTask = p.StandardOutput.ReadToEndAsync();
        var stderrTask = p.StandardError.ReadToEndAsync();
        await p.WaitForExitAsync(cts.Token);

        return (p.ExitCode, await stdoutTask, await stderrTask);
    }

    private static bool TryExtractPid(string launchctlOutput, out int pid)
    {
        // Output contoh:
        //   {
        //     "LimitLoadToSessionType" = "System";
        //     "Label" = "com.tacticalrmm.tacticalagent";
        //     "OnDemand" = false;
        //     "LastExitStatus" = 0;
        //     "PID" = 1234;
        //     "Program" = "/usr/local/tacticalagent/tacticalagent";
        //   };
        pid = 0;
        var idx = launchctlOutput.IndexOf("\"PID\"", StringComparison.Ordinal);
        if (idx < 0) return false;
        var eq = launchctlOutput.IndexOf('=', idx);
        var sc = launchctlOutput.IndexOf(';', eq);
        if (eq < 0 || sc < 0) return false;
        var slice = launchctlOutput.Substring(eq + 1, sc - eq - 1).Trim();
        return int.TryParse(slice, out pid);
    }
}
```

## 5.6 Penggunaan dari ViewModel

```csharp
public sealed class RmmInstallViewModel : ViewModelBase
{
    private readonly IAgentSupervisor _supervisor;
    private readonly ITrmmApiClient _trmm;
    private readonly ILogger<RmmInstallViewModel> _log;

    public async Task EnrollAsync(int clientId, int siteId, CancellationToken ct = default)
    {
        // 1. Request deployment URL dari Supabase Edge Function (lihat Bab 6)
        var deployment = await RequestDeploymentAsync(clientId, siteId, ct);

        // 2. Download installer
        var installerPath = await DownloadInstallerAsync(deployment.DownloadUrl, ct);

        // 3. Install via supervisor (UAC prompt akan muncul di sini)
        await _supervisor.InstallAsync(new InstallRequest(
            InstallerPath: installerPath,
            Arguments:     "",                     // TRMM installer self-configures dari deployment URL
            RequiresElevation: true), ct);

        // 4. Polling sampai agent muncul di TRMM API
        var hostname = Environment.MachineName;
        var deadline = DateTime.UtcNow.AddMinutes(2);
        while (DateTime.UtcNow < deadline)
        {
            ct.ThrowIfCancellationRequested();
            var agent = await _trmm.GetAgentByHostnameAsync(hostname, ct);
            if (agent is not null)
            {
                _log.LogInformation("Agent enrolled with id {AgentId}", agent.AgentId);
                // simpan agent_id ke profile user
                return;
            }
            await Task.Delay(5_000, ct);
        }

        throw new TimeoutException("Agent did not appear in TRMM within 2 minutes.");
    }
}
```

## 5.7 Service name reference

Mapping service name yang akan dipakai di `GetStateAsync` / `StartAsync` / dst.:

| Komponen | Windows service name | macOS LaunchDaemon label |
|---|---|---|
| Tactical RMM agent | `tacticalrmm` | `com.tacticalrmm.tacticalagent` |
| MeshCentral agent | `Mesh Agent` | `meshagent` (atau `com.meshcentral.agent`) |
| XDR client | (vendor-specific) | (vendor-specific) |
| SASE / NGFW | (vendor-specific) | (vendor-specific) |

Centralize di constants:

```csharp
namespace HermesNetwork.Supervisor;

public static class AgentServiceNames
{
    public static string TacticalRmm => OperatingSystem.IsWindows()
        ? "tacticalrmm"
        : "com.tacticalrmm.tacticalagent";

    public static string MeshAgent => OperatingSystem.IsWindows()
        ? "Mesh Agent"
        : "meshagent";
}
```

## 5.8 Testing

### 5.8.1 Mock supervisor untuk unit test ViewModel

```csharp
public sealed class FakeAgentSupervisor : IAgentSupervisor
{
    public Dictionary<string, ServiceState> States { get; } = new();
    public List<InstallRequest> InstallRequests { get; } = new();

    public bool IsElevated => true;

    public Task<ServiceState> GetStateAsync(string name, CancellationToken ct = default)
        => Task.FromResult(States.GetValueOrDefault(name, ServiceState.NotInstalled));

    public Task StartAsync(string name, CancellationToken ct = default)
    { States[name] = ServiceState.Running; return Task.CompletedTask; }

    public Task StopAsync(string name, CancellationToken ct = default)
    { States[name] = ServiceState.Stopped; return Task.CompletedTask; }

    public Task InstallAsync(InstallRequest req, CancellationToken ct = default)
    { InstallRequests.Add(req); return Task.CompletedTask; }

    public Task UninstallAsync(string name, CancellationToken ct = default)
    { States.Remove(name); return Task.CompletedTask; }
}
```

### 5.8.2 Manual test di Windows

```powershell
# Install TRMM agent dengan deployment URL
$installer = "C:\temp\trmm-deploy-abc123.exe"
$args = ""

# Test via supervisor (perlu hand-roll test program)
dotnet run --project HermesNetwork.SupervisorTest -- install $installer $args
```

### 5.8.3 Manual test di macOS

```bash
sudo installer -pkg /tmp/trmm-deploy-abc123.pkg -target /
sudo launchctl load /Library/LaunchDaemons/com.tacticalrmm.tacticalagent.plist
sudo launchctl start com.tacticalrmm.tacticalagent
```

Verifikasi:

```bash
sudo launchctl list com.tacticalrmm.tacticalagent
# Output should include "PID" = <some integer>
```

## 5.9 Best practices

- **Selalu cek `IsElevated`** sebelum panggil install/uninstall — kalau false, prompt user untuk relaunch as admin
- **Timeout** semua proses external (`installer`, `msiexec`, `osascript`) dengan `CancellationTokenSource.CancelAfter`
- **Jangan parse output `launchctl list` dengan regex** — gunakan `launchctl print system/<label>` di macOS modern (>= 10.10) yang lebih konsisten
- **Log stderr dari osascript** — error messages dari sana sangat informatif
- **Gunakan `DllImport` minimal** — lebih baik shell out ke `sc.exe`/`launchctl` daripada P/Invoke ke API native, kecuali ada alasan kuat

---

[← Bab 4 TrmmApiClient]({{ site.baseurl }}{% link docs/04-trmm-api-client.md %}){: .btn }
[Bab 6 — Enrollment Flow →]({{ site.baseurl }}{% link docs/06-enrollment-flow.md %}){: .btn .btn-primary }
