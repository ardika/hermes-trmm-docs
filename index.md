---
layout: default
title: Beranda
nav_order: 1
description: "Panduan lengkap arsitektur dan implementasi integrasi Tactical RMM untuk Hermes Network 360 Guard desktop client."
permalink: /
---

# Panduan Integrasi Tactical RMM

**Hermes Network 360 Guard — Desktop Client (Windows & macOS)**
{: .fs-6 .fw-300 }

[Mulai dari Pendahuluan]({{ site.baseurl }}{% link docs/01-pendahuluan.md %}){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[Lihat Arsitektur]({{ site.baseurl }}{% link docs/02-arsitektur.md %}){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## Tentang Dokumen Ini

Dokumen ini adalah panduan lengkap untuk **merefactor integrasi Tactical RMM** di aplikasi desktop **Hermes Network 360 Guard** (`HermesNetwork360-Avalonia`) — dari pendekatan ad-hoc berbasis IPC custom yang ada saat ini, menjadi arsitektur tiga-lapis yang bersih, terstandar, dan mudah dipelihara.

### Audiens

Dokumen ini ditulis untuk:

- **Engineer .NET / Avalonia** yang akan mengimplementasikan refactor di sisi desktop client
- **Backend engineer** yang akan menyiapkan Supabase Edge Function sebagai proxy ke TRMM API
- **DevOps / SRE** yang mengelola server TRMM dan agent deployment
- **Tech lead** yang perlu memahami trade-off arsitektur sebelum approve scope kerja

### Prasyarat pengetahuan

Anda diharapkan sudah familiar dengan:

- Pemrograman C# dan ekosistem .NET 8
- Konsep dasar REST API, JSON, HTTP
- Operasi systemctl/launchctl pada level dasar
- Konsep dasar TRMM (Tactical Remote Monitoring & Management)

Tidak perlu paham TRMM secara mendalam — bagian-bagian penting akan dijelaskan kembali di tempat yang relevan.

---

## Daftar Isi

| # | Bab | Ringkasan |
|---|-----|-----------|
| 1 | [Pendahuluan]({{ site.baseurl }}{% link docs/01-pendahuluan.md %}) | Konteks, masalah saat ini, tujuan refactor |
| 2 | [Arsitektur]({{ site.baseurl }}{% link docs/02-arsitektur.md %}) | Diagram before/after, tiga lapis terpisah |
| 3 | [Prasyarat]({{ site.baseurl }}{% link docs/03-prasyarat.md %}) | Setup TRMM server, NuGet, environment |
| 4 | [Layer 1 — TrmmApiClient]({{ site.baseurl }}{% link docs/04-trmm-api-client.md %}) | HTTP client typed untuk TRMM REST API |
| 5 | [Layer 2 — AgentSupervisor]({{ site.baseurl }}{% link docs/05-agent-supervisor.md %}) | Service lifecycle lokal, Win + Mac |
| 6 | [Layer 3 — Enrollment Flow]({{ site.baseurl }}{% link docs/06-enrollment-flow.md %}) | Deployment URL, install, polling |
| 7 | [Authentication & Keamanan]({{ site.baseurl }}{% link docs/07-auth-keamanan.md %}) | JWT exchange, token rotation, threat model |
| 8 | [Dukungan macOS]({{ site.baseurl }}{% link docs/08-mac-support.md %}) | LaunchDaemon, signing, notarization |
| 9 | [TRMM API Reference]({{ site.baseurl }}{% link docs/09-api-reference.md %}) | Endpoint cheat-sheet |
| 10 | [Rencana Migrasi]({{ site.baseurl }}{% link docs/10-migrasi.md %}) | Phase 0–6, rollback |
| 11 | [Troubleshooting]({{ site.baseurl }}{% link docs/11-troubleshooting.md %}) | Error umum + debugging |
| 12 | [FAQ]({{ site.baseurl }}{% link docs/12-faq.md %}) | Pertanyaan & keputusan desain |

---

## Konvensi

Sepanjang dokumen ini:

- Kode contoh dalam C# ditarget untuk **.NET 8** (TFM `net8.0`).
- Kode Python untuk Edge Function / utilitas server menggunakan **Python 3.10+**.
- Path Windows menggunakan format `C:\Path\To\File.exe`.
- Path macOS / Linux menggunakan format `/Library/...` atau `/etc/...`.
- Block dengan `bash` adalah perintah shell yang bisa dijalankan di server.
- Block dengan `csharp` adalah snippet untuk drop-in ke project Avalonia.

> **Catatan**: kalau Anda baru menemukan dokumen ini, mulai dari [Pendahuluan]({{ site.baseurl }}{% link docs/01-pendahuluan.md %}) dan baca berurutan. Kalau Anda hanya butuh referensi cepat, langsung ke [API Reference]({{ site.baseurl }}{% link docs/09-api-reference.md %}) atau [Troubleshooting]({{ site.baseurl }}{% link docs/11-troubleshooting.md %}).

---

## Versi & Pembaruan

| Tanggal | Versi | Catatan |
|---|---|---|
| 2026-04-21 | 1.0 | Versi awal — refactor arsitektur tiga-lapis |

Saran perbaikan dokumen → buka issue di repo GitHub Pages atau hubungi tim engineering Hermes Network Inc.
