---
layout: default
title: 12. FAQ
nav_order: 13
permalink: /docs/faq/
---

# 12. FAQ
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 12.1 Pertanyaan arsitektur

### **Kenapa harus pakai Edge Function sebagai proxy? Tidak bisa langsung dari client ke TRMM?**

Bisa secara teknis, tapi salah secara desain:

- Untuk panggil TRMM API, butuh API key. API key itu privileged — bisa lihat semua agent, semua tenant. Kalau di-embed di binary client, ada di tangan setiap user yang install app. Sekali leak = semua data di TRMM bocor.
- Edge Function = trust boundary. Hanya satu tempat yang pegang key (server-side). Kalau Edge Function compromise, scope kerusakan terbatas pada satu komponen yang bisa di-rotate cepat.

### **Kenapa NIST P-256 untuk ECIES, bukan X25519 (yang lebih modern)?**

X25519 lebih clean dan sedikit lebih cepat, tapi tidak tersedia di .NET stdlib (perlu NuGet `NSec.Cryptography` atau `BouncyCastle`). NIST P-256 setara secara security dan ada di `System.Security.Cryptography.ECDiffieHellman` tanpa dependency tambahan.

Trade-off ini detail, dampaknya minimal. Kalau project nanti pakai NSec untuk hal lain, swap ke X25519 = 5 baris kode.

### **Kenapa harus dihilangkan `ServiceEngine.exe`? Bukankah service custom kasih kontrol lebih?**

Service custom = surface area lebih besar = audit lebih sulit. Yang dilakukan ServiceEngine.exe sekarang:

1. **Spawn process child** (TRMM agent, mesh agent, XDR) — Windows Service Manager bisa lakukan ini langsung lewat `sc.exe`
2. **Forward IPC dari UI ke service** — TRMM API bisa di-call langsung dari UI (atau via Edge proxy)
3. **Query state service** — `ServiceController` (Win) / `launchctl` (Mac) bisa lakukan ini

Tidak ada nilai unik yang ditambahkan ServiceEngine.exe. Menghapusnya = kurang attack surface, kurang code yang harus di-maintain.

### **Apa yang terjadi kalau TRMM backend down?**

Yang terdampak:

- ❌ User tidak bisa lihat status agent terbaru di UI
- ❌ User tidak bisa enroll device baru
- ❌ User tidak bisa run script

Yang TIDAK terdampak:

- ✅ TRMM agent yang sudah jalan tetap jalan (tidak butuh server untuk operasi normal)
- ✅ MeshCentral yang sudah connect tetap jalan
- ✅ Login Supabase masih jalan (Supabase independen dari TRMM)
- ✅ Logging lokal tetap berjalan

UI design harus graceful: tampilkan banner "TRMM service temporarily unavailable" + retry, bukan crash.

### **Bagaimana kalau user tidak punya admin privilege?**

UI **harus prompt elevation** sebelum operasi yang butuh admin (install agent, start/stop service):

```csharp
if (!_supervisor.IsElevated && requiresAdmin)
{
    var result = await ShowDialogAsync(
        "Operasi ini butuh privilege admin. Klik 'OK' untuk lanjutkan.");
    if (result == DialogResult.Ok)
    {
        // Restart app dengan elevation
        RestartElevated();
    }
    return;
}
```

Untuk Windows: re-launch app dengan `Verb = "runas"`. User klik UAC = OK.

Untuk macOS: setiap operasi yang elevate akan minta password admin (osascript dialog). User cukup input sekali per session.

## 12.2 Pertanyaan implementasi

### **Boleh pakai Refit / RestSharp untuk TrmmApiClient?**

Boleh. Refit (interface-driven HTTP client) lebih ringkas:

```csharp
public interface ITrmmApiClient
{
    [Get("/api/v3/agents/{id}/")]
    Task<AgentDto> GetAgentAsync(int id, CancellationToken ct = default);

    [Post("/api/v3/agents/{id}/runscript/")]
    Task<ScriptRunDto> RunScriptAsync(int id, [Body] ScriptRunRequest req, CancellationToken ct = default);
}
```

Trade-off: tambah NuGet dependency. Kalau project sudah pakai Refit di tempat lain, konsisten. Kalau tidak, manual `HttpClient` baik-baik saja.

### **Boleh pakai gRPC untuk Edge Function?**

Bisa, tapi Supabase Edge Functions hanya support HTTP. Untuk gRPC native, perlu deploy ke Cloud Run / Fly.io / dll.

Untuk scope ini, REST sudah cukup. Latency tambahan vs gRPC negligible untuk operasi rare seperti enrollment.

### **Apakah perlu real-time WebSocket untuk update agent status?**

Phase 1: tidak perlu. Polling tiap 30 detik sudah cukup untuk UX baik.

Phase 2 (kalau ada bandwidth): tambah WebSocket consumer di desktop, subscribe ke `wss://api.hermesnetwork.cloud/ws/agents/?agent_id=...`. Update push langsung ke UI.

Trade-off:

- ✅ Realtime
- ❌ State management lebih kompleks
- ❌ Reconnection logic perlu robust

### **Bagaimana kalau perlu support OS lain selain Win + Mac (Linux)?**

`IAgentSupervisor` adalah interface — tinggal tambah `LinuxAgentSupervisor` yang pakai `systemctl`:

```csharp
public sealed class LinuxAgentSupervisor : IAgentSupervisor
{
    public async Task StartAsync(string name, CancellationToken ct = default)
        => await RunAsync("systemctl", $"start {name}", ct);
    // dst.
}
```

Avalonia sudah support Linux. TRMM agent Linux juga ada (`tacticalagent-linux-amd64`). Tidak ada blocker, hanya scope work tambahan.

### **Bagaimana cara test kalau saya tidak punya akses ke TRMM production?**

Setup TRMM staging instance (gratis, open-source, bisa di Docker):

```bash
# Quick start TRMM development
git clone https://github.com/amidaware/tacticalrmm.git
cd tacticalrmm
docker compose -f docker-compose.dev.yml up
```

Akan jalan di `http://localhost`. Generate API key dari local dashboard dan pakai untuk test.

## 12.3 Pertanyaan operasional

### **Berapa cost TRMM untuk 1000 endpoint?**

TRMM open-source, free secara license. Cost adalah:

| Komponen | Estimasi cost untuk 1000 endpoint |
|---|---|
| VPS untuk backend (~4 vCPU, 8 GB RAM, 100 GB SSD) | $40–80/bulan |
| Domain + Let's Encrypt | gratis |
| Bandwidth | included biasanya |
| MeshCentral di server yang sama | 0 |
| Supabase (free tier untuk dev, $25 untuk prod) | $0–25/bulan |
| Apple Developer ID (untuk signing macOS) | $99/tahun = ~$8/bulan |
| **Total** | **~$50–100/bulan** untuk 1000 endpoint |

Bandingkan dengan competitor RMM komersial: $1–2/endpoint/bulan = $1000–2000/bulan untuk 1000 endpoint. TRMM hemat 95%.

### **Bagaimana kalau perlu high availability untuk TRMM?**

TRMM v0.20+ support HA setup dengan PostgreSQL + Redis cluster. Detail di [docs.tacticalrmm.com](https://docs.tacticalrmm.com/install_server/).

Untuk Hermes Network scale (estimasi 5000–10000 endpoint), single-instance dengan good backups sudah cukup.

### **Berapa lama agent install di endpoint?**

| OS | Durasi (typical) |
|---|---|
| Windows 10/11 | 30–60 detik |
| macOS Sonoma | 45–90 detik (lebih lama karena codesign verify + privacy approval) |
| Linux (Ubuntu/Debian) | 20–40 detik |

Includes download installer (~30 MB) + service registration + first checkin.

### **Bagaimana update agent setelah deployed?**

TRMM dashboard punya bulk update feature. Admin bisa:

1. Settings → Agent Updates → Set version target (mis. v2.5.1)
2. Pilih agent yang akan di-update (semua, atau filter by client/site)
3. Click "Update Now"

Agent self-update di background, no user interaksi needed.

## 12.4 Pertanyaan keamanan

### **Apakah TRMM agent buka backdoor ke endpoint?**

TRMM agent hanya **outbound connection** ke server. Tidak ada port listening di endpoint (kecuali kalau MeshCentral remote desktop session aktif, dan itu pun tunneled lewat MeshCentral relay).

Kalau perusahaan butuh review code, TRMM open-source di [GitHub](https://github.com/amidaware/tacticalrmm), audit free.

### **Apakah komunikasi agent ↔ server encrypted?**

Ya. Agent pakai HTTPS untuk REST + WSS (WebSocket Secure) untuk push events. Plus NATS dengan TLS untuk message bus internal.

MeshCentral pakai TLS juga. Remote desktop session di-tunnel via WebSocket ter-encrypt.

### **Bisakah user lihat data user lain via TRMM?**

Tidak, jika RBAC di-setup benar. TRMM punya:

- **Roles** (admin, technician, read-only)
- **Permission per client** (user A hanya bisa lihat agent client tertentu)
- **API key scope** (key dengan scope read-only tidak bisa run script)

Edge Function di Hermes juga apply layer tambahan: filter `WHERE user_id = jwt.sub` di setiap query.

### **Apa yang dilakukan kalau endpoint hilang (dicuri)?**

Workflow incident response:

1. Disable user di Supabase Auth (rotate password / suspend user)
2. Di TRMM dashboard, find agent berdasarkan hostname
3. Run remote script untuk wipe sensitive data (kalau corporate device)
4. Delete agent dari TRMM (`DELETE /api/v3/agents/{id}/`)
5. Audit log di `enrollment_log` table untuk forensic

## 12.5 Pertanyaan compliance

### **Apakah arsitektur ini compatible dengan SOC 2?**

Ya, dengan kondisi:

- API key rotation policy documented + executed quarterly
- Access log retained 6+ bulan (TRMM + Supabase audit log)
- Incident response runbook ada
- Backup + DR procedure tested annually

Detail compliance bukan scope dokumen ini — konsultasi dengan auditor.

### **Bagaimana dengan GDPR?**

User punya hak:

- **Akses** — bisa export data via `GET /api/v3/agents/{id}/`
- **Penghapusan** — `DELETE /api/v3/agents/{id}/` + cascade ke `enrollment_log`
- **Portabilitas** — JSON export sudah standard format

Implementasikan endpoint admin "delete user data" yang chain:
1. Delete TRMM agent
2. Delete enrollment_log row
3. Delete user_profile row
4. Delete auth.users row

## 12.6 Roadmap & future work

### **Yang tidak ada di scope ini tapi mungkin di-add nanti:**

| Fitur | Estimasi effort | Value |
|---|---|---|
| WebSocket real-time agent events | 1 sprint | High UX, low ops cost |
| Bulk script execution (run di banyak agent sekaligus) | 1 sprint | Useful for ops, low priority |
| Built-in remote desktop (embed MeshCentral viewer) | 2 sprints | High value buat support team |
| Mobile app (iOS/Android) untuk monitoring | 4+ sprints | Out of scope |
| Custom dashboard widgets per user | 2 sprints | Nice-to-have |
| Webhook integration (Slack alert dari TRMM) | 1 sprint | High ops value |
| AI-powered anomaly detection | 4+ sprints | Research project |

Prioritas based on user feedback dan ops need.

---

## Pertanyaan tidak ada di sini?

Buka issue di repo internal `Hermes-Network-Inc/HermesNetwork360Guard` atau Slack `#engineering`.

---

[← Bab 11 Troubleshooting]({% link docs/11-troubleshooting.md %}){: .btn }
[← Kembali ke Beranda]({% link index.md %}){: .btn .btn-primary }
