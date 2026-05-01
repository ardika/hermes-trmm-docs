---
layout: default
title: 10. Rencana Migrasi
nav_order: 11
permalink: /docs/migrasi/
---

# 10. Rencana Migrasi
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 10.1 Filosofi: inkremental, bukan big-bang

Pendekatan migrasi ini **paralel** — arsitektur baru ditambahkan **berdampingan** dengan kode lama, bukan menggantikan langsung. Setelah arsitektur baru terbukti stabil di produksi (rolled out ke 100% client), kode lama dihapus.

Manfaat:

- **Rollback gampang** — kalau ada masalah, tinggal disable feature flag arsitektur baru
- **Tidak ada downtime** — user tidak merasakan perubahan sampai tab UI baru di-rollout
- **Bisa bertahap per-screen** — refactor satu tab dulu, ambil pelajaran, baru lanjut tab lain
- **Tim bisa kerja paralel** — backend Supabase Edge Function tidak block frontend client work

## 10.2 Phase overview

| Phase | Durasi target | Aktivitas | Hasil terlihat |
|---|---|---|---|
| **0** | 1 minggu | Audit + dokumentasi | Tabel mapping IPC code → TRMM endpoint |
| **1** | 1 minggu | Setup TrmmApiClient + Supabase schema | Tested standalone, tidak terintegrasi UI |
| **2** | 2 minggu | Migrate read operations | UI tab "RMM Status" pakai TRMM API langsung |
| **3** | 1 minggu | AgentSupervisor abstraction | Lifecycle pakai stdlib, IPC untuk install masih existing |
| **4** | 2 minggu | Edge Function enroll-rmm | Enrollment flow baru, IPC untuk enrollment dimatikan |
| **5** | 1 minggu | Migrate command operations (run script, restart agent) | Semua IPC traffic dihilangkan |
| **6** | 1 minggu | Cleanup ServiceEngine.exe | Hapus binary lama, hapus IPC code |
| **Total** | **~9 minggu** | | Production fully migrated |

## 10.3 Phase 0 — Audit

### Tujuan

Memetakan **semua** call IPC yang ada saat ini dan menentukan tujuan migrasinya.

### Tugas konkret

1. Search semua occurrence `IpcComService.SendingMessage` di repo:

   ```powershell
   cd F:\Upwork\CharlesUSA\HermesNetwork360-Avalonia
   dotnet build 2>&1 | grep -v warning  # baseline kompil
   git grep -n "IpcComService" -- '*.cs' > audit-ipc-callsites.txt
   git grep -n '"Code":' -- '*.cs' > audit-ipc-codes.txt
   ```

2. Untuk tiap call site, isi tabel berikut:

   | File:Line | Code | Service | Arg | Operasi | Tujuan baru |
   |---|---|---|---|---|---|
   | `XdrViewModel.cs:455` | `Z1398V` | `XDR` | `Start` | Start XDR service | `AgentSupervisor.StartAsync("XDRService")` |
   | `RmmViewModel.cs:1216` | `DL2KNT` | `LOG` | `Path` | Get log path | Hapus, tidak relevan setelah refactor |
   | `MainViewModel.cs:42` | `K9P3MQ` | `RMM` | `Status` | Check agent status | `TrmmApiClient.GetAgentByHostnameAsync(hostname)` |
   | ... | ... | ... | ... | ... | ... |

3. Klasifikasi tiap row jadi:
   - **Read state** → migrate ke `TrmmApiClient` query
   - **Lifecycle** → migrate ke `AgentSupervisor`
   - **Install/Enroll** → migrate ke `EnrollmentService`
   - **Hapus** (sudah tidak relevan)

### Deliverable

Spreadsheet `audit-ipc-mapping.xlsx` dengan tab "Read", "Lifecycle", "Enrollment", "Deprecated", masing-masing berisi rows dari classification di atas.

### Exit criteria

- [ ] Semua call site IpcComService teridentifikasi
- [ ] Setiap row punya target migration jelas
- [ ] Sign-off dari tech lead

## 10.4 Phase 1 — Setup TrmmApiClient

### Tujuan

`TrmmApiClient` siap dipakai, tertest standalone, **tapi belum dipakai oleh UI**.

### Tugas

1. Buat folder `HermesNetwork/Trmm/` (lihat [Bab 4]({% link docs/04-trmm-api-client.md %}))
2. Implementasikan semua DTO record + `ITrmmApiClient` + `TrmmApiClient`
3. Tambah unit test di project test baru:

   ```
   HermesNetwork.Tests/
   └── Trmm/
       ├── TrmmApiClientTests.cs
       └── TestData.cs
   ```

4. Tambah Supabase schema migration:

   ```bash
   supabase migration new add_trmm_columns
   # edit file SQL yang dihasilkan
   supabase db push
   ```

5. Add NuGet packages:

   ```xml
   <PackageReference Include="System.Net.Http.Json" Version="8.0.0" />
   <PackageReference Include="Microsoft.Extensions.Http.Polly" Version="8.0.0" />
   ```

### Deliverable

- PR `feature/trmm-api-client` di repo `HermesNetwork360Guard`
- DB migration applied di Supabase staging

### Exit criteria

- [ ] CI build hijau
- [ ] Unit test coverage >= 80% untuk `TrmmApiClient`
- [ ] Smoke test manual: jalankan `TrmmApiClient` dengan API key staging, dapat list agents
- [ ] Code review approval

## 10.5 Phase 2 — Migrate read operations

### Tujuan

Ganti **semua** call IPC yang sifatnya "read state" dengan call ke `TrmmApiClient`.

### Strategi: feature flag

Tambah flag di `appsettings.json`:

```json
{
  "FeatureFlags": {
    "UseTrmmApiForState": false
  }
}
```

Tiap ViewModel:

```csharp
public async Task<RmmStatusDto> GetRmmStatusAsync()
{
    if (_featureFlags.UseTrmmApiForState)
    {
        // path baru
        return await GetStatusViaTrmmApiAsync();
    }
    else
    {
        // path lama (IPC)
        return await GetStatusViaIpcAsync();
    }
}
```

### Tugas per tab/screen

Mulai dari yang paling simple:

1. **Tab "RMM Status"** — query agent online/offline, last seen
   - Replace `IpcComService.GetRmmStatus()` → `_trmm.GetAgentByHostnameAsync(hostname)`
2. **Tab "Health Checks"** — diskspace, ping, CPU
   - Replace `IpcComService.GetHealthChecks()` → `_trmm.GetChecksAsync(agentId)`
3. **Tab "Inventory"** — CPU, RAM, OS, disks
   - Sudah ada di response `_trmm.GetAgentAsync()`

### Test plan

Untuk tiap tab:

- [ ] Set flag `false` → behavior lama, identik
- [ ] Set flag `true` → behavior baru, hasil sama
- [ ] Toggle flag saat app jalan → tidak crash
- [ ] Network error → fallback ke loading state (tidak crash)

### Rollout

1. Build dengan flag default `false`
2. Internal QA test (1 minggu) dengan flag `true` di staging
3. Beta release ke 5% user dengan flag `true`
4. Monitor error rate via telemetry
5. 100% rollout
6. Hapus flag + path lama di phase 6

### Exit criteria

- [ ] Semua read operation tab sudah pakai TrmmApiClient
- [ ] Beta + 100% rollout sukses
- [ ] No regression di error rate

## 10.6 Phase 3 — AgentSupervisor

### Tujuan

Implementasikan `IAgentSupervisor` (Win + Mac) dan ganti panggilan **lifecycle** (start/stop service).

### Catatan

Phase ini **tidak menyentuh install/enrollment** — itu phase 4. Phase 3 hanya tentang start/stop service yang sudah terinstall.

### Tugas

1. Implementasi sesuai [Bab 5]({% link docs/05-agent-supervisor.md %})
2. Replace `IpcComService.StartService(serviceName)` → `_supervisor.StartAsync(serviceName)`
3. Replace `IpcComService.StopService(serviceName)` → `_supervisor.StopAsync(serviceName)`
4. Replace `IpcComService.IsServiceRunning(serviceName)` → `_supervisor.GetStateAsync(serviceName)`

### Test plan

- [ ] Start TRMM agent via supervisor di Windows
- [ ] Stop TRMM agent via supervisor di Windows
- [ ] Same di macOS
- [ ] Permission denied path (non-admin user) → throw exception, UI prompt elevation

### Rollout

Sama seperti phase 2 (feature flag, beta, 100%).

## 10.7 Phase 4 — Edge Function & enrollment baru

### Tujuan

Mendaftarkan endpoint baru via TRMM deployment URL (bukan custom IPC).

### Tugas

1. Deploy `enroll-rmm` Edge Function (lihat [Bab 6]({% link docs/06-enrollment-flow.md %}))
2. Implementasi `EnrollmentService` di desktop
3. Tambah tab/dialog "Enroll device" di UI yang panggil `EnrollmentService.EnrollAsync()`
4. Audit `enrollment_log` table di Supabase staging

### Test plan

- [ ] Enroll Win 10 VM bersih → agent muncul di TRMM dashboard ≤ 2 menit
- [ ] Enroll Mac M1 → agent muncul di TRMM dashboard
- [ ] Re-enroll same device → tidak duplicate, agent_id tetap
- [ ] Enroll dengan JWT expired → 401, prompt re-login
- [ ] Enroll dengan TRMM down → graceful error message

### Catatan transition

Selama phase 4, ada 2 cara enrollment:

- **Existing devices** — sudah enrolled via cara lama, **jangan di-touch**. Tetap pakai agent_id yang sudah tercatat.
- **New devices** — pakai `EnrollmentService` baru.

Tidak perlu force re-enroll device existing — TRMM agent yang sudah jalan akan tetap report ke server tanpa perubahan.

### Exit criteria

- [ ] 5 device baru berhasil enroll via flow baru
- [ ] Edge function audit log shows expected behavior
- [ ] Rollback plan tested (kalau ada bug, kembali ke flow lama)

## 10.8 Phase 5 — Migrate command operations

### Tujuan

Operasi **command** (run script, send message, restart agent) yang sebelumnya via IPC, ganti pakai TRMM API.

### Tugas

1. Tab "Run Script" — replace `IpcComService.RunScript(...)` → `_trmm.RunScriptAsync(agentId, request)`
2. Tab "Restart Agent" — call `_supervisor.StopAsync` lalu `StartAsync`
3. Notifikasi result ke UI via WebSocket atau polling

### Test plan

- [ ] Run PowerShell script di Windows agent → stdout muncul di UI
- [ ] Run bash script di Mac agent → stdout muncul di UI
- [ ] Script timeout → UI tampilkan timeout error, tidak hang
- [ ] Run script ke agent offline → meaningful error

### Exit criteria

- [ ] Semua tab command sudah pakai TRMM API atau supervisor
- [ ] No more IPC traffic untuk command operation

## 10.9 Phase 6 — Cleanup

### Tujuan

Hapus kode lama yang sudah tidak terpakai.

### Tugas

1. Verifikasi tidak ada lagi reference ke `IpcComService`:

   ```bash
   git grep "IpcComService"   # harus 0 hits di branch utama
   ```

2. Hapus file:

   ```
   HermesNetwork/Service/IpcComService.cs
   HermesNetwork/Service/IpcMessage.cs
   ServiceEngine/                 (folder lengkap)
   ```

3. Hapus binary `ServiceEngine.exe` dari installer
4. Hapus Windows Service registration di `setup.iss` / installer scripts:

   ```iss
   ; Sebelum
   [Run]
   Filename: "{app}\ServiceEngine.exe"; Parameters: "install"

   ; Hapus seluruh blok itu
   ```

5. Update README, dokumentasi internal
6. Hapus feature flag yang ditambahkan di phase 2 dan 3 (sudah tidak perlu)

### Test plan

- [ ] Build installer baru tanpa ServiceEngine
- [ ] Install di VM bersih → app jalan tanpa ServiceEngine.exe
- [ ] Upgrade dari versi lama (yang punya ServiceEngine) → ServiceEngine ter-uninstall otomatis
- [ ] Existing agent tetap report normal ke TRMM

### Exit criteria

- [ ] CI hijau setelah cleanup
- [ ] Release notes mention "removed legacy ServiceEngine.exe"
- [ ] Smoke test produksi 1 minggu tanpa regression

## 10.10 Rollback strategy

Jika ada masalah serius di salah satu phase:

| Phase | Rollback |
|---|---|
| 1 | Revert PR, drop database column tambahan (kalau memungkinkan) |
| 2 | Toggle feature flag `false` → kembali ke IPC |
| 3 | Toggle feature flag `false` |
| 4 | Disable tombol "Enroll" baru, kembalikan flow lama |
| 5 | Toggle feature flag `false` |
| 6 | Force release ulang versi sebelum cleanup; `IpcComService.cs` di backup branch |

Yang penting: **arsitektur baru tidak menyentuh state lokal** sampai phase 6. Sampai phase 5, IPC + arsitektur baru jalan paralel — rollback = tinggal flip flag.

## 10.11 Risk register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| TRMM API rate limit hit saat 100% rollout | Medium | High | Edge function caching + retry policy |
| User di network restrictif tidak bisa hit TRMM langsung | Low | Medium | Edge function jadi proxy untuk semua call |
| Agent installer download gagal di lingkungan corporate | Medium | High | Retry + manual installer fallback dari URL stabil |
| User tidak comfortable dengan UAC prompt | Low | Low | Educational message di UI sebelum klik |
| Mac code signing expire | Low | Critical | Calendar reminder 30 hari sebelum, auto-renew via Apple |

## 10.12 Definition of Done untuk migrasi

Migrasi dianggap **100% selesai** jika:

- [ ] Phase 0–6 semua exit criteria centang
- [ ] `IpcComService` sudah dihapus dari `main` branch
- [ ] `ServiceEngine.exe` tidak ada di installer
- [ ] Telemetry produksi 2 minggu menunjukkan no increase di error rate
- [ ] Dokumentasi README, internal wiki, dan onboarding doc sudah di-update
- [ ] Tim engineering sudah onboard dengan arsitektur baru (knowledge transfer session)

---

[← Bab 9 API Reference]({% link docs/09-api-reference.md %}){: .btn }
[Bab 11 — Troubleshooting →]({% link docs/11-troubleshooting.md %}){: .btn .btn-primary }
