---
layout: default
title: 3. Prasyarat
nav_order: 4
permalink: /docs/prasyarat/
---

# 3. Prasyarat
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 3.1 Server Tactical RMM

Anda butuh akses ke instance TRMM yang sudah berjalan. Untuk Hermes Network Inc., sudah tersedia di:

| Komponen | URL |
|---|---|
| Backend Django | `https://api.hermesnetwork.cloud` |
| MeshCentral | `https://mesh.hermesnetwork.cloud` |
| Frontend dashboard | `https://rmm.hermesnetwork.cloud` |

Kalau Anda perlu instance baru (mis. staging), ikuti panduan resmi:
[https://docs.tacticalrmm.com/install_server/](https://docs.tacticalrmm.com/install_server/)

### 3.1.1 Membuat API key

1. Login ke `https://rmm.hermesnetwork.cloud` sebagai super-admin
2. Pergi ke **Settings → Global Settings → API Keys**
3. Klik **Add API Key**
4. Pilih user yang akan dipakai (rekomendasi: bikin user dedicated `hermes-app-svc` dengan permission terbatas)
5. Salin key yang dikeluarkan — **hanya ditampilkan satu kali**

> **Penting:** API key ini **hanya boleh disimpan di Supabase Edge Function**, jangan pernah embed di desktop client. Detail di [Bab 7]({% link docs/07-auth-keamanan.md %}).

### 3.1.2 Menentukan client_id & site_id

TRMM mengelompokkan agent ke dalam:

```
Client (top-level, mis. "Hermes Network Inc.")
  └── Site (anak, mis. "Jakarta Office", "Remote Workers")
       └── Agent (endpoint, mis. "JADE-PC")
```

Untuk multi-tenant, Anda butuh mapping `user → client_id → site_id`. Cara dapat ID-nya:

```bash
# Pakai API key dari step sebelumnya
curl -H "X-API-KEY: YOUR_API_KEY" https://api.hermesnetwork.cloud/api/v3/clients/

# Output:
# [
#   { "id": 34, "name": "Hermes Network Inc.", "sites": [...] },
#   ...
# ]
```

Catat ID-nya untuk dipakai di Supabase Edge Function (lihat [Bab 6]({% link docs/06-enrollment-flow.md %})).

## 3.2 Development environment

### 3.2.1 Tooling .NET

| Tool | Versi minimum | Catatan |
|---|---|---|
| .NET SDK | 8.0 | Pakai 8.0.x latest stable |
| Avalonia | 11.x | Sudah ada di project |
| JetBrains Rider | 2024.x | Atau Visual Studio 2022 17.8+ |
| Git | 2.40+ | |

Verifikasi:

```powershell
dotnet --version
# Output: 8.0.x
```

### 3.2.2 NuGet packages baru yang dibutuhkan

Tambahkan ke `HermesNetwork/HermesNetwork.csproj`:

```xml
<ItemGroup>
  <!-- Layer 1: HTTP client utilities -->
  <PackageReference Include="System.Net.Http.Json" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.Http.Polly" Version="8.0.0" />

  <!-- Logging utility (kalau belum ada) -->
  <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="8.0.0" />

  <!-- Untuk WebSocket TRMM real-time events (opsional, fase 2) -->
  <!-- <PackageReference Include="System.Net.WebSockets.Client" Version="4.3.2" /> -->
</ItemGroup>
```

> Tidak butuh NuGet untuk crypto, ECDH, AesGcm, ServiceController — semua sudah di stdlib `System.Security.Cryptography` dan `System.ServiceProcess`.

### 3.2.3 Tooling backend (Supabase Edge Function)

| Tool | Versi minimum | Catatan |
|---|---|---|
| Supabase CLI | 1.150+ | `npm i -g supabase` |
| Deno | 1.40+ | Otomatis terinstall via Supabase CLI |
| Node.js | 20.x | Untuk tooling local |

Verifikasi:

```bash
supabase --version
# Output: 1.x.x
```

Login ke project Supabase Anda:

```bash
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

## 3.3 Akses & kredensial

Pastikan Anda punya akses ke:

- [ ] Repo `Hermes-Network-Inc/HermesNetwork360Guard` di GitHub (write access)
- [ ] Supabase project (admin role untuk deploy edge function)
- [ ] TRMM dashboard (super-admin untuk generate API key + cari client/site ID)
- [ ] SSH access ke server TRMM (kalau butuh debug NATS/Celery)

## 3.4 Environment variables yang akan dipakai

Untuk **desktop client** (akan dipanggil saat startup):

| Variable | Contoh | Asal |
|---|---|---|
| `HNGUARD_SUPABASE_URL` | `https://xxx.supabase.co` | Supabase dashboard |
| `HNGUARD_SUPABASE_ANON_KEY` | `eyJ...` | Supabase dashboard |
| `HNGUARD_LOG_PUB_HEX` | (opsional) | dari `LogReportService/log_pub.key` |

Untuk **Supabase Edge Function** (set via `supabase secrets`):

| Variable | Contoh | Asal |
|---|---|---|
| `TRMM_API_URL` | `https://api.hermesnetwork.cloud` | static |
| `TRMM_API_KEY` | `your-api-key-here` | step 3.1.1 |
| `TRMM_DEFAULT_CLIENT_ID` | `34` | step 3.1.2 |
| `TRMM_DEFAULT_SITE_ID` | `41` | step 3.1.2 |

Set di Supabase:

```bash
supabase secrets set TRMM_API_URL=https://api.hermesnetwork.cloud
supabase secrets set TRMM_API_KEY=your-api-key-here
supabase secrets set TRMM_DEFAULT_CLIENT_ID=34
supabase secrets set TRMM_DEFAULT_SITE_ID=41
```

## 3.5 Verifikasi cepat

Sebelum mulai coding, test bahwa Anda bisa:

### 3.5.1 Akses TRMM API dengan API key

```bash
curl -sS -H "X-API-KEY: YOUR_API_KEY" \
  https://api.hermesnetwork.cloud/api/v3/agents/ | head -c 500
```

Expected: JSON array of agents (atau `[]` kalau belum ada).

### 3.5.2 Build ulang HermesNetwork360-Avalonia

```powershell
cd F:\Upwork\CharlesUSA\HermesNetwork360-Avalonia
dotnet restore
dotnet build -c Debug
```

Expected: Build sukses (dengan beberapa warning OK).

### 3.5.3 Deploy stub edge function

```bash
cd <your-supabase-project>
supabase functions new enroll-rmm
supabase functions deploy enroll-rmm
```

Expected: function muncul di Supabase dashboard di Functions section.

## 3.6 Checklist sebelum lanjut

Pastikan semua centang sebelum lanjut ke [Bab 4]({% link docs/04-trmm-api-client.md %}):

- [ ] TRMM API key sudah dibuat dan disimpan di password manager
- [ ] `client_id` dan `site_id` sudah dicatat
- [ ] .NET SDK 8.0 terinstall, project bisa di-build
- [ ] Supabase CLI bisa login + link ke project
- [ ] Test cURL ke TRMM API berhasil HTTP 200
- [ ] Edge function stub berhasil di-deploy
- [ ] Anda punya akses write ke repo GitHub `HermesNetwork360Guard`

---

[← Bab 2 Arsitektur]({% link docs/02-arsitektur.md %}){: .btn }
[Bab 4 — TrmmApiClient →]({% link docs/04-trmm-api-client.md %}){: .btn .btn-primary }
