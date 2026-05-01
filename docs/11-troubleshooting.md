---
layout: default
title: 11. Troubleshooting
nav_order: 12
permalink: /docs/troubleshooting/
---

# 11. Troubleshooting
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 11.1 Diagnostic toolkit

Langkah pertama untuk semua masalah:

```bash
# TRMM API basic health
curl -sS -H "X-API-KEY: $KEY" https://api.hermesnetwork.cloud/api/v3/agents/ | jq 'length'
# Expected: angka jumlah agent. 0 = belum ada agent. Error 401 = key salah.

# Edge function health
curl -X POST https://YOUR_PROJECT.supabase.co/functions/v1/enroll-rmm \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"hostname":"test","platform":"windows"}'
# Expected: 200 dengan download_url, atau 401/404/etc dengan pesan jelas.

# Cek agent status di endpoint
# Windows
sc query tacticalrmm
# macOS
sudo launchctl list com.tacticalrmm.tacticalagent
```

Di mana log:

| Apa | Lokasi (Windows) | Lokasi (macOS) |
|---|---|---|
| Hermes app log | `%LOCALAPPDATA%\HermesNetwork360Guard\app.log` | `~/Library/Logs/HermesNetwork360Guard/app.log` |
| TRMM agent log | `C:\ProgramData\TacticalRMM\agent.log` | `/var/log/tacticalagent/stdout.log` + `stderr.log` |
| Mesh agent log | `C:\Program Files\Mesh Agent\meshagent.log` | `/var/log/meshagent.log` |
| Edge function log | Supabase Dashboard → Functions → Logs | sama |
| nginx upload service | `sudo journalctl -u log-upload -n 100` | sama |

## 11.2 Error: HTTP 401 dari TRMM API

**Gejala:** `TrmmAuthException` di desktop client, atau curl test return `{"detail":"Invalid token"}`.

**Penyebab kemungkinan:**

1. API key salah ketik atau di-truncate
2. API key sudah di-revoke di TRMM dashboard
3. Header name salah (`Authorization` vs `X-API-KEY`)
4. Edge function pakai key staging untuk panggilan production

**Cara verifikasi:**

```bash
# Test langsung dengan curl (bypass edge function)
curl -i -H "X-API-KEY: $KEY" https://api.hermesnetwork.cloud/api/v3/agents/ | head -5
```

**Fix:**

- Login ke `https://rmm.hermesnetwork.cloud` → Settings → API Keys
- Verifikasi key masih ada dan belum expired
- Kalau hilang, generate baru, update via `supabase secrets set TRMM_API_KEY=...`
- Redeploy edge function: `supabase functions deploy enroll-rmm`

## 11.3 Error: Agent tidak muncul di TRMM setelah enroll

**Gejala:** `EnrollmentService.EnrollAsync` timeout setelah 2 menit, agent tidak ada di TRMM dashboard.

**Penyebab kemungkinan:**

| Penyebab | Cara cek | Fix |
|---|---|---|
| Installer crash | Cek event log Windows (`Application` log) | Re-download installer, retry |
| Service tidak start | `sc query tacticalrmm` | Manual `sc start tacticalrmm` |
| Firewall block outbound | Test `curl https://api.hermesnetwork.cloud` dari endpoint | Whitelist domain |
| API URL salah | `Get-Content "C:\ProgramData\TacticalRMM\agent.conf"` | Reinstall |
| TRMM backend down | Curl test dari machine lain | Check status `https://rmm.hermesnetwork.cloud` |
| Hostname conflict | TRMM dashboard → cari hostname yang sama | Hapus duplicate, retry |

**Diagnosa step-by-step:**

```powershell
# Windows — cek apakah service jalan
sc query tacticalrmm
# Expected: STATE: 4 RUNNING

# Cek log agent
Get-Content "C:\ProgramData\TacticalRMM\agent.log" -Tail 50

# Manual checkin paksa
& "C:\Program Files\TacticalAgent\tacticalrmm.exe" -m checkin

# Test koneksi ke TRMM backend
Test-NetConnection api.hermesnetwork.cloud -Port 443
```

```bash
# macOS — sama
sudo launchctl list com.tacticalrmm.tacticalagent
sudo tail -50 /var/log/tacticalagent/stderr.log
sudo /usr/local/tacticalagent/tacticalagent -m checkin
nc -zv api.hermesnetwork.cloud 443
```

## 11.4 Error: macOS app crash saat panggil osascript

**Gejala:** App freeze atau crash saat klik "Enroll device" di Mac, error log mention osascript.

**Penyebab kemungkinan:**

1. App belum di-signed/notarized → Gatekeeper block
2. Karakter aneh di shell command yang merusak escaping
3. Quote tidak match di AppleScript

**Diagnosa:**

```bash
# Test escaping manual
osascript -e 'do shell script "echo test" with administrator privileges'

# Kalau itu error, masalah di environment, bukan di code
# Kalau itu sukses, jalan, masalah ada di shell command yang generate by code
```

**Fix:**

- Pastikan code sign valid: `codesign --verify --verbose=2 HermesNetwork360Guard.app`
- Pastikan notarized: `spctl -a -v HermesNetwork360Guard.app` → "accepted"
- Untuk shell command: pakai escape ganda `\\\"` di code C#

## 11.5 Error: Edge function "TRMM deployment failed"

**Gejala:** `enroll-rmm` return 502, log Supabase show "TRMM deployment failed: 500/400".

**Penyebab kemungkinan:**

| Status TRMM | Penyebab |
|---|---|
| 400 | `client` atau `site` ID tidak valid |
| 401 | API key di edge function salah / revoked |
| 404 | Endpoint deployment dimatikan di TRMM admin |
| 429 | Rate limit hit |
| 500 | Bug atau crash di TRMM backend |

**Cara cek:**

Lihat log Supabase Functions:

```bash
supabase functions logs enroll-rmm --tail
```

Cari line `TRMM deployment failed: <status> <body>`. Body biasanya berisi pesan error TRMM yang spesifik.

**Fix:**

- Validate `client_id` + `site_id` dengan curl manual:
  ```bash
  curl -H "X-API-KEY: $KEY" https://api.hermesnetwork.cloud/api/v3/clients/ | jq '.[] | {id, name}'
  ```
- Update environment variable: `supabase secrets set TRMM_DEFAULT_CLIENT_ID=<correct>`
- Redeploy: `supabase functions deploy enroll-rmm`

## 11.6 Performa: TRMM API call lambat (>5 detik)

**Gejala:** UI lag saat buka tab "RMM Status", request take 5+ detik.

**Penyebab kemungkinan:**

1. TRMM backend overloaded (banyak agent + tidak ada caching)
2. Network latency dari user ke `api.hermesnetwork.cloud`
3. Polly retry sedang aktif (3x retry × 30 detik timeout = 90 detik max)
4. WebSocket connection drop dan fallback polling agresif

**Diagnosa:**

```bash
# Time direct API call
time curl -sS -H "X-API-KEY: $KEY" \
  https://api.hermesnetwork.cloud/api/v3/agents/?hostname=JADE-PC > /dev/null

# Time edge function call
time curl -X POST https://YOUR_PROJECT.supabase.co/functions/v1/enroll-rmm \
  -H "Authorization: Bearer $JWT" -d '{"hostname":"x","platform":"windows"}' > /dev/null
```

**Fix:**

- Tambah timeout di `TrmmApiClientOptions.Timeout = TimeSpan.FromSeconds(10)`
- Gunakan polling kurang sering (30 detik vs 5 detik)
- Kalau scaleable, switch ke WebSocket events
- Server-side: scale Celery worker di TRMM backend

## 11.7 Error: "Service tidak terinstall" tapi sebenarnya install sukses

**Gejala:** `AgentSupervisor.GetStateAsync()` return `NotInstalled`, tapi agent jalan.

**Penyebab kemungkinan:**

1. Service name salah (case-sensitive di Windows tapi case-insensitive di pemanggilan ServiceController)
2. Service registered dengan display name beda dari service name
3. macOS: launchctl label di plist beda dari yang dicari

**Cara cek:**

```powershell
# Windows — list semua service yang nama mengandung "tactical"
Get-Service | Where-Object Name -Like "*tactical*"
# Output: Name yang resmi
```

```bash
# macOS
sudo launchctl list | grep tactical
# Output: PID  Status  Label
```

**Fix:**

- Gunakan nama yang **persis** sesuai output di atas, simpan di `AgentServiceNames` constants
- Jangan hardcode di multiple tempat — central di satu file

## 11.8 Error: User report "tampilan kosong, tidak ada agent"

**Gejala:** User login, buka tab RMM, tapi "Total agents: 0" walau dia harusnya punya 2 device.

**Penyebab kemungkinan:**

1. Filter `client_id` di TrmmApiClient salah
2. RLS Supabase block read dari user
3. User profile belum punya `trmm_client_id`

**Diagnosa:**

```sql
-- Di Supabase SQL editor
SELECT id, email, trmm_client_id, trmm_site_id, trmm_agent_id
FROM auth.users
JOIN user_profiles ON auth.users.id = user_profiles.id
WHERE email = 'user@example.com';
```

Kalau `trmm_client_id` NULL → user belum di-assign ke tenant.

**Fix:**

- Update profile manual:
  ```sql
  UPDATE user_profiles
  SET trmm_client_id = 34, trmm_site_id = 41
  WHERE id = '<user-uuid>';
  ```
- Atau bikin admin UI untuk manage tenant assignment

## 11.9 Build error: `'AesGcm' is obsolete`

**Gejala:** Compile warning `AesGcm(byte[])` is obsolete.

**Penyebab:** .NET 8 deprecate constructor lama, harus pakai `new AesGcm(key, tagSizeInBytes: 16)`.

**Fix:**

```csharp
// Ganti
using var gcm = new AesGcm(key);

// Dengan
using var gcm = new AesGcm(key, tagSizeInBytes: 16);
```

Sudah benar di code di [Bab 4]({{ site.baseurl }}{% link docs/04-trmm-api-client.md %}) dan dokumen LogEncryption.

## 11.10 Avalonia: UI freeze saat panggil TRMM API

**Gejala:** UI thread freeze 3+ detik saat klik refresh.

**Penyebab:** Call sync di UI thread → block. Walau API call async, kalau ada `.Result` atau `.Wait()` somewhere, akan deadlock.

**Cara cek:**

Search code `.Result` atau `.Wait()`:

```bash
git grep -nE '\.Result\b|\.Wait\(\)' -- '*.cs'
```

**Fix:**

```csharp
// JANGAN
public void OnRefreshClick()
{
    var result = _trmm.GetAgentAsync(123).Result;   // ❌ deadlock di UI thread
    Status = result.Status;
}

// OK
public async Task OnRefreshClick()
{
    var result = await _trmm.GetAgentAsync(123);
    Status = result.Status;
}
```

Pakai `[RelayCommand]` di MVVM toolkit yang sudah generate command async dengan benar.

## 11.11 Mac: Notarization gagal "hardened runtime not enabled"

**Gejala:** `xcrun notarytool submit` upload OK tapi reject dengan "hardened runtime missing".

**Fix:**

Pastikan `codesign` pakai flag `--options runtime`:

```bash
codesign --force --options runtime --sign "$SIGN_ID" --timestamp "$APP_PATH"
```

Tanpa `--options runtime`, Apple tolak notarization.

## 11.12 Debug logging level

Untuk troubleshoot mendalam, bisa naikkan log level di desktop app:

```csharp
// Program.cs
builder.Services.AddLogging(b =>
{
    b.AddDebug();
    b.AddConsole();
    b.SetMinimumLevel(LogLevel.Trace);   // dari Information ke Trace
});
```

`HttpClient` request/response detail bisa di-log dengan handler:

```csharp
public sealed class LoggingHttpHandler : DelegatingHandler
{
    private readonly ILogger _log;
    public LoggingHttpHandler(ILogger logger) => _log = logger;

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        _log.LogTrace("HTTP {Method} {Url}", request.Method, request.RequestUri);
        var resp = await base.SendAsync(request, ct);
        _log.LogTrace("HTTP {Status} {Url}", (int)resp.StatusCode, request.RequestUri);
        return resp;
    }
}

// register
services.AddTransient<LoggingHttpHandler>();
services.AddHttpClient<ITrmmApiClient, TrmmApiClient>()
    .AddHttpMessageHandler<LoggingHttpHandler>();
```

> **Penting:** jangan log full request/response body di production — bisa bocor sensitive data.

## 11.13 Diagnostic checklist untuk support ticket

Saat user report masalah, kumpulkan info ini:

- [ ] OS + version (Windows 10 22H2, macOS Sonoma 14.4, dll)
- [ ] Hermes Network 360 Guard version (lihat About dialog)
- [ ] Username + email Supabase
- [ ] Hostname endpoint
- [ ] `agent_id` (kalau ada)
- [ ] Screenshot error
- [ ] Tail 100 lines dari `app.log` (decrypt dulu kalau encrypted)
- [ ] Tail 50 lines dari TRMM agent log
- [ ] Output `sc query tacticalrmm` (Win) atau `launchctl list com.tacticalrmm.tacticalagent` (Mac)
- [ ] Steps to reproduce

Template support ticket di Slack/email — bikin shortcut command untuk auto-collect.

## 11.14 Eskalasi

Kalau masalah tidak teratasi di support level 1:

| Skenario | Eskalasi ke |
|---|---|
| TRMM agent crash terus-menerus | TRMM upstream issue di [github.com/amidaware/tacticalrmm](https://github.com/amidaware/tacticalrmm) |
| MeshCentral remote desktop tidak connect | MeshCentral upstream |
| Bug di `TrmmApiClient` | Issue internal repo Hermes |
| Bug di Edge Function | Issue internal repo Hermes |
| Bug di Supabase platform | Supabase support ticket |
| Bug di Avalonia | [github.com/AvaloniaUI/Avalonia](https://github.com/AvaloniaUI/Avalonia) |

---

[← Bab 10 Rencana Migrasi]({{ site.baseurl }}{% link docs/10-migrasi.md %}){: .btn }
[Bab 12 — FAQ →]({{ site.baseurl }}{% link docs/12-faq.md %}){: .btn .btn-primary }
