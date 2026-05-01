---
layout: default
title: 1. Pendahuluan
nav_order: 2
permalink: /docs/pendahuluan/
---

# 1. Pendahuluan
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 1.1 Konteks

**Hermes Network 360 Guard** adalah aplikasi desktop cross-platform (Windows + macOS) yang dibangun dengan Avalonia (.NET 8). Aplikasi ini berfungsi sebagai *unified security agent* di endpoint pengguna, mengintegrasikan beberapa layanan:

- **Tactical RMM (TRMM)** — remote monitoring & management
- **MeshCentral** — remote desktop / file transfer
- **XDR** (Extended Detection & Response) dari vendor pihak ketiga
- **SASE / NGFW** (Secure Access Service Edge / Next-Gen Firewall)
- Komunikasi dengan backend Supabase + Firebase + Google Cloud

Dari semua layanan tersebut, **Tactical RMM adalah backbone** untuk monitoring kesehatan endpoint, eksekusi script remote, deployment update, dan inventaris software/hardware.

## 1.2 Apa itu Tactical RMM?

[Tactical RMM](https://github.com/amidaware/tacticalrmm) adalah platform RMM open-source berbasis Django + Vue. Komponennya:

- **Backend Django** dengan REST API di `/api/v3/...` (endpoint utama yang akan kita pakai)
- **Frontend Vue** untuk dashboard ops (di luar scope dokumen ini)
- **Agent Go-based** (`tacticalagent`) yang dipasang di setiap endpoint, jalan sebagai Windows Service / LaunchDaemon
- **MeshCentral** (sub-komponen) untuk remote shell, screen, dan file transfer
- **NATS** sebagai message broker antara agent dan backend
- **Celery + Redis** untuk job queue di backend

Arsitektur dasarnya:

```
┌─────────────────┐      HTTPS/REST      ┌──────────────────┐
│  Tactical RMM   │◄───────────────────► │  TRMM Backend    │
│  Agent (Go)     │      NATS msgs       │  (Django + DRF)  │
│  di endpoint    │                      │                  │
└─────────────────┘                      └──────────────────┘
        │                                         │
        │ MeshCentral channel                     │
        ▼                                         ▼
┌─────────────────┐                      ┌──────────────────┐
│  MeshAgent      │◄────────────────────►│  MeshCentral     │
│  (remote ctrl)  │                      │  Server          │
└─────────────────┘                      └──────────────────┘
```

Hermes Network Inc. men-deploy TRMM di `api.hermesnetwork.cloud` (backend) dan `mesh.hermesnetwork.cloud` (MeshCentral).

## 1.3 Masalah pada implementasi saat ini

Berdasarkan audit kode (`HermesNetwork/Service/IpcComService.cs`, `ServiceEngineUtils.cs`, dan log produksi `app.log`), integrasi TRMM saat ini punya beberapa masalah:

### 1.3.1 Custom IPC protocol tanpa kontrak

UI Avalonia berkomunikasi dengan komponen *ServiceEngine* (Windows service custom) lewat named-pipe / TCP localhost menggunakan JSON ad-hoc:

```json
{ "Code": "DL2KNT", "Service": "LOG", "Arg": "Path", "Config": "...", "OptionalData": "" }
{ "Code": "Z1398V", "Service": "XDR", "Arg": "Start", "Config": "...", "OptionalData": "" }
```

Masalahnya:

- **Magic strings** (`DL2KNT`, `Z1398V`) tanpa katalog/dokumentasi
- **Tidak ada versioning** — penambahan kode baru bisa bentrok diam-diam
- **Tidak ada schema validation** — typo di `Config` baru ketahuan saat agent crash
- **Susah di-test** — perlu running ServiceEngine.exe lokal untuk integration test

### 1.3.2 State drift

State agent (online/offline, install status, last check-in) di-track lokal di UI **dan** di TRMM backend. Karena tidak ada single source of truth, dua-duanya bisa drift:

- UI bilang "RMM not installed", tapi di TRMM dashboard agent terlihat online
- TRMM dashboard bilang agent offline, tapi UI menampilkan "Service engine is running"

User support menjadi mimpi buruk karena tidak jelas mana yang benar.

### 1.3.3 Enrollment manual

Saat ini enrollment dilakukan dengan menyusun command line panjang yang berisi `agent_id`, `client_id`, `org_id`, mesh ID, dan API key — di-encode jadi base64, dipassthrough ke ServiceEngine via IPC, lalu dijalankan oleh ServiceEngine sebagai process child.

Contoh dari log produksi (sudah dikaburkan):

```
{"Code":"Z1398V","Service":"XDR","Arg":"Start","Config":"OTQ3IEphZGUtUEMgYW55IGY2YTY0Nzg5...|user_369|org_34|n1.ndr24.com|C:\\...\\ngfw.log"}
```

Masalahnya:

- **API key embed di payload** → kalau IPC bocor, key bocor
- **Format string tradisi** (`a|b|c|d`) — tidak self-describing, mudah salah parse
- **Re-enroll = cleanup manual** registry + file lokal sebelum bisa enroll ulang
- **Multi-tenant susah**: tidak ada cara bersih memilih client/site di TRMM dari UI

### 1.3.4 Coupling tinggi UI ↔ infra

Setiap kali backend TRMM upgrade atau API berubah:

- ServiceEngine.exe harus di-recompile + redeploy
- UI Avalonia harus tahu detail "kode IPC mana untuk apa"
- Tidak bisa ganti backend (mis. dari TRMM ke ConnectWise) tanpa rewrite besar

### 1.3.5 Reinventing wheel

Beberapa fitur sudah disediakan TRMM tapi diimplementasikan ulang di kode Hermes:

| Fitur | Sudah ada di TRMM | Yang dilakukan kode Hermes saat ini |
|-------|-------------------|-------------------------------------|
| Heartbeat / online detection | ✅ Built-in | Polling `IsServiceEngineRunning` lokal |
| Script execution | ✅ `runscript` API | Belum dipakai |
| Agent deployment URL | ✅ `/agents/deployments/` | Custom installer wrapper |
| Multi-tenant (clients/sites) | ✅ Built-in | Hardcoded `user_369`, `org_34` |
| RBAC + audit log | ✅ Built-in | Tidak ada |
| WebSocket real-time | ✅ Built-in | Tidak dipakai |

## 1.4 Tujuan dokumen

Setelah membaca dan menerapkan dokumen ini, Anda akan:

1. Memahami **arsitektur tiga-lapis** yang menggantikan IPC custom
2. Bisa mengimplementasikan **`TrmmApiClient`** — typed C# client untuk TRMM REST API
3. Bisa mengimplementasikan **`AgentSupervisor`** — abstraksi cross-platform untuk service lifecycle
4. Bisa mengimplementasikan **enrollment flow** baru menggunakan TRMM deployment URL
5. Memahami **strategi migrasi inkremental** dari IPC custom ke arsitektur baru, tanpa big-bang rewrite
6. Tahu cara **mengamankan API key** (tidak embed di client)
7. Tahu cara **dukung macOS** (LaunchDaemon, codesign, notarization)

## 1.5 Apa yang BUKAN cakupan

Dokumen ini **tidak** membahas:

- Setup awal TRMM server itu sendiri (sudah ada di `api.hermesnetwork.cloud`)
- Konfigurasi MeshCentral di luar yang dibutuhkan TRMM
- Konfigurasi XDR / SASE vendor (di luar scope refactor RMM)
- UI/UX redesign Avalonia (kita hanya ganti **layer di bawahnya**)
- Database migration di TRMM backend

Kalau Anda butuh salah satu di atas, lihat dokumen internal terpisah atau dokumentasi resmi TRMM di [docs.tacticalrmm.com](https://docs.tacticalrmm.com/).

---

[← Beranda]({{ site.baseurl }}{% link index.md %}){: .btn }
[Bab 2 — Arsitektur →]({{ site.baseurl }}{% link docs/02-arsitektur.md %}){: .btn .btn-primary }
