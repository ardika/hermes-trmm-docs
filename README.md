# Hermes Network 360 Guard — Panduan Integrasi Tactical RMM

Dokumentasi multi-halaman dalam Bahasa Indonesia untuk arsitektur dan implementasi integrasi Tactical RMM yang bersih di aplikasi desktop Hermes Network 360 Guard.

Site ini dibangun dengan **Jekyll + theme [just-the-docs](https://just-the-docs.com/)** dan dipublish lewat **GitHub Pages**.

## Struktur

```
.
├── _config.yml                # Jekyll config
├── Gemfile                    # Ruby deps untuk preview lokal
├── index.md                   # Landing / TOC
├── docs/
│   ├── 01-pendahuluan.md
│   ├── 02-arsitektur.md
│   ├── 03-prasyarat.md
│   ├── 04-trmm-api-client.md
│   ├── 05-agent-supervisor.md
│   ├── 06-enrollment-flow.md
│   ├── 07-auth-keamanan.md
│   ├── 08-mac-support.md
│   ├── 09-api-reference.md
│   ├── 10-migrasi.md
│   ├── 11-troubleshooting.md
│   └── 12-faq.md
└── README.md                  # File ini
```

## Cara publish ke GitHub Pages

### Opsi A — Repo personal `ardika.github.io`

Kalau ingin publish di `https://ardika.github.io/`:

```bash
# Clone repo personal page
git clone https://github.com/ardika/ardika.github.io.git
cd ardika.github.io

# Salin semua isi folder ini ke root repo
cp -r /path/to/HermesNetwork360-TrmmDocs/* .
cp -r /path/to/HermesNetwork360-TrmmDocs/.gitignore .  # kalau ada

# Edit _config.yml — pastikan:
#   baseurl: ""
#   url: "https://ardika.github.io"

git add .
git commit -m "Add Hermes Network 360 Guard TRMM integration docs"
git push origin main
```

Site akan tersedia di `https://ardika.github.io/` dalam ~1 menit.

### Opsi B — Repo terpisah dengan subfolder URL

Kalau ingin publish di `https://ardika.github.io/hermes-trmm-docs/`:

```bash
# Bikin repo baru di GitHub: ardika/hermes-trmm-docs
gh repo create ardika/hermes-trmm-docs --public

cd /path/to/HermesNetwork360-TrmmDocs

# Edit _config.yml — pastikan:
#   baseurl: "/hermes-trmm-docs"
#   url: "https://ardika.github.io"

git init
git add .
git commit -m "Initial: Hermes Network 360 Guard TRMM integration docs"
git remote add origin https://github.com/ardika/hermes-trmm-docs.git
git branch -M main
git push -u origin main

# Enable Pages
gh repo edit --enable-pages --pages-branch main
```

Setelah ~1 menit, site available di `https://ardika.github.io/hermes-trmm-docs/`.

### Opsi C — Repo di organization `Hermes-Network-Inc`

```bash
gh repo create Hermes-Network-Inc/trmm-integration-docs --private
# Sama dengan Opsi B, ganti owner repo
```

Karena private, GitHub Pages butuh GitHub Pro/Enterprise untuk private Pages site.

## Preview lokal (sebelum push)

Butuh Ruby 3.0+ + Bundler:

```bash
cd HermesNetwork360-TrmmDocs/

# Install dependencies (sekali saja)
bundle install

# Jalankan preview server
bundle exec jekyll serve

# Buka http://localhost:4000 di browser
```

Edit file markdown → save → site auto-rebuild.

## Penyesuaian sebelum publish

### 1. Update URLs di `_config.yml`

```yaml
baseurl: ""             # atau "/repo-name" untuk Opsi B/C
url: "https://ardika.github.io"
```

### 2. Update aux link di sidebar

Di `_config.yml`:

```yaml
aux_links:
  "Hermes Network Inc. di GitHub":
    - "https://github.com/Hermes-Network-Inc"
```

### 3. (Opsional) Custom domain

Kalau ada domain custom (mis. `docs.hermesnetwork.cloud`):

```bash
echo "docs.hermesnetwork.cloud" > CNAME
```

Lalu set DNS A record:

```
docs.hermesnetwork.cloud  A   185.199.108.153
docs.hermesnetwork.cloud  A   185.199.109.153
docs.hermesnetwork.cloud  A   185.199.110.153
docs.hermesnetwork.cloud  A   185.199.111.153
```

GitHub Pages akan auto-issue Let's Encrypt cert.

## Update content

Edit file `.md` di folder `docs/`. Tiap file punya **front matter YAML** di atas:

```yaml
---
layout: default
title: 5. Layer 2 — AgentSupervisor
nav_order: 6
permalink: /docs/agent-supervisor/
---
```

- `nav_order`: posisi di sidebar (1, 2, 3, …)
- `title`: muncul di sidebar dan tab browser
- `permalink`: URL friendly

Push ke `main` branch → GitHub Pages auto-rebuild dalam ~1 menit.

## Tambah halaman baru

```bash
# Buat file baru
cat > docs/13-changelog.md <<'EOF'
---
layout: default
title: 13. Changelog
nav_order: 14
permalink: /docs/changelog/
---

# 13. Changelog

## v1.0 — 2026-04-21

- Initial release
EOF

# Commit + push
git add docs/13-changelog.md
git commit -m "Add changelog page"
git push
```

Halaman baru otomatis muncul di sidebar.

## Theme customization

`just-the-docs` punya banyak option di `_config.yml`. Untuk override CSS:

```bash
mkdir -p _sass/custom
cat > _sass/custom/custom.scss <<'EOF'
.site-title {
  color: #00ff88 !important;
}
EOF
```

Detail di [just-the-docs.com/docs/customization](https://just-the-docs.com/docs/customization/).

## Troubleshooting

### "Page build failed" di GitHub

Cek tab "Actions" di GitHub repo — error message lengkap di sana. Common issue:

- Front matter YAML invalid (tab vs space, missing `---`)
- Link `{% link path/to/file %}` salah path
- Mermaid diagram syntax error

### Mermaid diagrams tidak render

Pastikan di `_config.yml`:

```yaml
mermaid:
  version: "10.9.0"
```

dan tag mermaid pakai backtick triple:

````
```mermaid
graph LR
  A --> B
```
````

### Sidebar urutan salah

Pastikan tiap halaman punya `nav_order` numerik unik di front matter.

## Lisensi

Internal Hermes Network Inc. — bukan untuk distribusi publik tanpa izin.
