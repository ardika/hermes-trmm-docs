---
layout: default
title: 9. TRMM API Reference
nav_order: 10
permalink: /docs/api-reference/
---

# 9. TRMM API Reference
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Cheat-sheet endpoint TRMM yang dipakai oleh implementasi ini. Untuk daftar lengkap lihat [docs.tacticalrmm.com](https://docs.tacticalrmm.com/).

**Base URL:** `https://api.hermesnetwork.cloud`

**Authentication:** Header `X-API-KEY: <your-api-key>` di setiap request.

## 9.1 Agents

### `GET /api/v3/agents/`

List semua agent. Filter via query string.

**Query parameters:**

| Param | Type | Description |
|---|---|---|
| `hostname` | string | Filter exact hostname |
| `client` | int | Filter by client_id |
| `site` | int | Filter by site_id |
| `monitoring_type` | string | "server" / "workstation" |

**Contoh:**

```bash
curl -H "X-API-KEY: $KEY" \
  "https://api.hermesnetwork.cloud/api/v3/agents/?hostname=JADE-PC"
```

**Response (200):**

```json
[
  {
    "agent_id":           "abc-uuid",
    "hostname":           "JADE-PC",
    "client":             { "id": 34, "name": "Hermes Network Inc." },
    "site":               { "id": 41, "name": "Jakarta Office" },
    "plat":               "windows",
    "operating_system":   "Microsoft Windows 11 Pro",
    "version":            "2.5.0",
    "last_seen":          "2026-04-21T10:32:15Z",
    "status":             "online",
    "logged_in_username": "jerem",
    "public_ip":          "203.0.113.42",
    "local_ip":           "192.168.1.50",
    "cpu_model":          "Intel(R) Core(TM) i7-12700H",
    "total_ram":          16.0,
    "disks": [
      { "device": "C:", "free": "120.5 GB", "total": "476.9 GB", "percent": 75 }
    ]
  }
]
```

### `GET /api/v3/agents/{agent_id}/`

Detail satu agent.

```bash
curl -H "X-API-KEY: $KEY" \
  "https://api.hermesnetwork.cloud/api/v3/agents/abc-uuid/"
```

Response sama dengan list, tapi single object dan punya field tambahan (`patches_pending`, `wmi_detail`, dll).

### `DELETE /api/v3/agents/{agent_id}/`

Hapus agent dari TRMM. Tidak menghapus binary di endpoint — hanya record di server.

```bash
curl -X DELETE -H "X-API-KEY: $KEY" \
  "https://api.hermesnetwork.cloud/api/v3/agents/abc-uuid/"
```

## 9.2 Deployments

### `POST /api/v3/agents/deployments/`

Generate deployment URL satu kali untuk install agent baru.

**Body:**

```json
{
  "client":         34,
  "site":           41,
  "expires":        "2026-04-22T10:00:00Z",
  "arch":           "amd64",
  "install_flags":  ""
}
```

| Field | Type | Description |
|---|---|---|
| `client` | int | ID dari `GET /api/v3/clients/` |
| `site` | int | ID site di bawah client tersebut |
| `expires` | datetime ISO 8601 | TTL deployment URL |
| `arch` | string | `amd64`, `arm64`, `amd64-mac`, `arm64-mac` |
| `install_flags` | string | Optional CLI args untuk installer |

**Response (201):**

```json
{
  "id":            123,
  "uuid":          "abc-uuid",
  "client":        34,
  "site":          41,
  "expiry":        "2026-04-22T10:00:00Z",
  "install_flags": "",
  "download_url":  "https://api.hermesnetwork.cloud/api/v3/agents/deployments/abc-uuid/download/"
}
```

**Cara dapat installer:**

```bash
curl -L -O "$DOWNLOAD_URL"
# Output filename biasanya: trmm-agent-<uuid>.exe atau .pkg
```

### `GET /api/v3/agents/deployments/`

List deployment yang aktif.

### `DELETE /api/v3/agents/deployments/{id}/`

Revoke deployment URL sebelum expires.

## 9.3 Clients & Sites

### `GET /api/v3/clients/`

List semua client (tenant) di TRMM.

```json
[
  {
    "id":   34,
    "name": "Hermes Network Inc.",
    "sites": [
      { "id": 41, "name": "Jakarta Office" },
      { "id": 42, "name": "Remote Workers" }
    ]
  }
]
```

### `POST /api/v3/clients/`

Buat client baru (admin only).

### `GET /api/v3/clients/{id}/sites/`

List sites di bawah satu client.

## 9.4 Scripts

### `GET /api/v3/scripts/`

List script yang sudah dibuat di TRMM dashboard.

```json
[
  {
    "id":          1,
    "name":        "Get-DiskInfo",
    "shell":       "powershell",
    "description": "Returns disk usage as JSON"
  },
  {
    "id":          2,
    "name":        "Restart-Service",
    "shell":       "powershell",
    "description": "Restart a Windows service by name"
  }
]
```

### `POST /api/v3/agents/{agent_id}/runscript/`

Eksekusi script di agent.

**Body:**

```json
{
  "script":   1,
  "output":   "wait",
  "args":     ["arg1", "arg2"],
  "timeout":  90
}
```

| Field | Type | Description |
|---|---|---|
| `script` | int | ID dari `GET /api/v3/scripts/` |
| `output` | enum | `"wait"` (sync return), `"forget"` (fire-and-forget), `"collector"` (untuk store) |
| `args` | string[] | CLI args |
| `timeout` | int | Detik, max 300 |

**Response (200) untuk `output: "wait"`:**

```json
{
  "execution_time": "2.45",
  "retcode":        0,
  "stdout":         "Disk C: 75% used\n",
  "stderr":         ""
}
```

## 9.5 Checks

### `GET /api/v3/checks/`

List checks. Bisa filter per agent.

```bash
curl -H "X-API-KEY: $KEY" \
  "https://api.hermesnetwork.cloud/api/v3/checks/?agent=abc-uuid"
```

```json
[
  {
    "id":         101,
    "check_type": "diskspace",
    "name":       "C: Drive",
    "status":     "passing",
    "more_info":  "C: 75% used (120.5 GB free of 476.9 GB)",
    "last_run":   "2026-04-21T10:30:00Z"
  },
  {
    "id":         102,
    "check_type": "ping",
    "name":       "Internet",
    "status":     "passing",
    "more_info":  "8.8.8.8 - avg 12ms",
    "last_run":   "2026-04-21T10:31:00Z"
  }
]
```

**Check types yang tersedia:**

| Type | Yang dimonitor |
|---|---|
| `diskspace` | Free space drive |
| `ping` | ICMP ke target |
| `cpuload` | Average CPU usage |
| `memory` | RAM usage |
| `winsvc` | State Windows service |
| `script` | Custom script result |
| `eventlog` | Windows event log filter |

## 9.6 Tasks (Scheduled Scripts)

### `GET /api/v3/tasks/`

List automated tasks (script yang jalan otomatis sesuai schedule).

### `POST /api/v3/tasks/`

Buat task baru.

```json
{
  "name":     "Weekly disk cleanup",
  "agent":    "abc-uuid",
  "script":   3,
  "schedule": "0 3 * * 0",     // cron: tiap Minggu 03:00
  "enabled":  true
}
```

## 9.7 Alerts

### `GET /api/v3/alerts/`

List alert yang aktif.

```json
[
  {
    "id":         5001,
    "alert_type": "check",
    "agent":      "abc-uuid",
    "severity":   "warning",
    "message":    "C: drive 90% used",
    "created_at": "2026-04-21T08:15:00Z",
    "snoozed":    false,
    "resolved":   false
  }
]
```

### `POST /api/v3/alerts/{id}/acknowledge/`

Ack alert (mark as seen but not resolved).

### `POST /api/v3/alerts/{id}/resolve/`

Mark alert as resolved.

## 9.8 WebSocket events (opsional)

TRMM punya WebSocket di `/trmm/{agent_id}/` untuk real-time events. Belum dipakai di scope dokumen ini, tapi bisa di-tambah untuk:

- Live agent online/offline notification
- Real-time script output streaming
- Alert push tanpa polling

Detail di [TRMM docs](https://docs.tacticalrmm.com/).

## 9.9 Error responses

| Status | Body | Penyebab |
|---|---|---|
| `400` | `{"detail": "validation error"}` | Body JSON invalid |
| `401` | `{"detail": "Invalid token"}` | API key salah / expired |
| `403` | `{"detail": "permission denied"}` | API key tidak punya scope yang diminta |
| `404` | `{"detail": "Not found"}` | Resource tidak ada |
| `429` | `{"detail": "rate limited"}` | Terlalu banyak request |
| `500` | `{"detail": "server error"}` | Bug atau crash di TRMM backend |

## 9.10 Rate limits

TRMM default rate limit:

- **100 requests/menit per API key** (across all endpoints)
- **20 deployments/menit per API key** (specific untuk `POST /api/v3/agents/deployments/`)

Bisa di-tune di TRMM admin settings. Untuk Hermes Network, default cukup karena edge function adalah satu-satunya consumer.

## 9.11 Quick reference table

| Operation | Method | Endpoint |
|---|---|---|
| List agents | GET | `/api/v3/agents/` |
| Get agent detail | GET | `/api/v3/agents/{id}/` |
| Delete agent | DELETE | `/api/v3/agents/{id}/` |
| List clients | GET | `/api/v3/clients/` |
| Get sites for client | GET | `/api/v3/clients/{id}/sites/` |
| Create deployment | POST | `/api/v3/agents/deployments/` |
| List scripts | GET | `/api/v3/scripts/` |
| Run script on agent | POST | `/api/v3/agents/{id}/runscript/` |
| List checks | GET | `/api/v3/checks/` |
| List tasks | GET | `/api/v3/tasks/` |
| List alerts | GET | `/api/v3/alerts/` |
| Acknowledge alert | POST | `/api/v3/alerts/{id}/acknowledge/` |
| Resolve alert | POST | `/api/v3/alerts/{id}/resolve/` |

---

[← Bab 8 Dukungan macOS]({% link docs/08-mac-support.md %}){: .btn }
[Bab 10 — Rencana Migrasi →]({% link docs/10-migrasi.md %}){: .btn .btn-primary }
