---
layout: default
title: 8. Dukungan macOS
nav_order: 9
permalink: /docs/mac-support/
---

# 8. Dukungan macOS
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 8.1 Tantangan utama di macOS

Berbeda dari Windows, macOS punya batasan ketat untuk aplikasi yang ingin install service / mengelola privileges:

| Aspek | Windows | macOS |
|---|---|---|
| Service mechanism | Windows Service Manager (`sc.exe`, `ServiceController`) | LaunchDaemon plist + `launchctl` |
| Privilege elevation | UAC (otomatis dari `Verb = "runas"`) | osascript dialog atau sudo terminal |
| Code signing | Optional untuk install | Wajib untuk service jalan tanpa block Gatekeeper |
| Notarization | N/A | Wajib untuk app yang didistribusikan di luar App Store |
| Firewall prompt | Dimute kalau signed | Selalu muncul untuk listener baru |
| Log access | Event Log via API | `Console.app` via `OSLog` |

Implikasi: **TRMM agent dan MeshAgent harus signed dan notarized** sebelum bisa install di Mac end-user tanpa drama.

## 8.2 Bundle signing untuk aplikasi Hermes

### 8.2.1 Apple Developer ID

Anda butuh akun **Apple Developer Program** ($99/tahun) untuk bisa tanda tangan binary. Setelah subscribe:

1. Generate **Developer ID Application** certificate di [developer.apple.com](https://developer.apple.com)
2. Download `.p12` ke Mac development machine
3. Import ke Keychain (Login keychain)
4. Verify:
   ```bash
   security find-identity -v -p codesigning
   ```
   Output harus include baris seperti:
   ```
   1) AB12CD34... "Developer ID Application: Hermes Network Inc. (XXXXXXXXXX)"
   ```

### 8.2.2 Sign aplikasi Avalonia

Setelah `dotnet publish -c Release -r osx-arm64 --self-contained` menghasilkan bundle `.app`:

```bash
APP_PATH="bin/Release/net8.0/osx-arm64/publish/HermesNetwork360Guard.app"
SIGN_ID="Developer ID Application: Hermes Network Inc. (XXXXXXXXXX)"

# Sign all .dylib + .so + executables di dalam bundle
find "$APP_PATH" -type f \( -name "*.dylib" -o -name "*.so" -o -perm +111 \) \
  -exec codesign --force --options runtime --sign "$SIGN_ID" --timestamp {} \;

# Sign top-level bundle
codesign --force --options runtime --sign "$SIGN_ID" --timestamp \
  --entitlements "Resources/HermesNetwork360Guard.entitlements" \
  "$APP_PATH"

# Verify
codesign --verify --deep --strict --verbose=2 "$APP_PATH"
spctl -a -v "$APP_PATH"
```

`HermesNetwork360Guard.entitlements`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.network.server</key>
    <false/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
</dict>
</plist>
```

### 8.2.3 Notarization

```bash
# Bundle .app menjadi .zip
ditto -c -k --keepParent "$APP_PATH" HermesNetwork360Guard.zip

# Submit untuk notarization
xcrun notarytool submit HermesNetwork360Guard.zip \
  --apple-id "your-apple-id@hermesnetwork.com" \
  --password "@keychain:AC_PASSWORD" \
  --team-id "XXXXXXXXXX" \
  --wait

# Setelah lulus (biasanya 5-15 menit), staple notarization ticket
xcrun stapler staple "$APP_PATH"

# Verify
xcrun stapler validate "$APP_PATH"
spctl -a -v "$APP_PATH"
```

Di CI/CD (GitHub Actions), simpan password app-specific di keychain CI:

```bash
xcrun notarytool store-credentials AC_PASSWORD \
  --apple-id "your-apple-id@hermesnetwork.com" \
  --team-id "XXXXXXXXXX" \
  --password "abcd-efgh-ijkl-mnop"      # app-specific password
```

## 8.3 LaunchDaemon untuk TRMM agent

### 8.3.1 Lokasi & ownership

| Path | Ownership | Mode |
|---|---|---|
| `/Library/LaunchDaemons/com.tacticalrmm.tacticalagent.plist` | `root:wheel` | `0644` |
| `/usr/local/tacticalagent/tacticalagent` | `root:wheel` | `0755` |

LaunchDaemon (di `/Library/LaunchDaemons/`) jalan saat boot, sebagai root, before user login. Bedakan dengan **LaunchAgent** (di `~/Library/LaunchAgents/`) yang jalan per-user setelah login.

Untuk TRMM agent, **selalu LaunchDaemon** karena:
- Harus jalan tanpa user login
- Butuh privilege root untuk install software / akses file system

### 8.3.2 Format plist

`com.tacticalrmm.tacticalagent.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.tacticalrmm.tacticalagent</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/tacticalagent/tacticalagent</string>
        <string>-m</string>
        <string>svc</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/var/log/tacticalagent/stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/var/log/tacticalagent/stderr.log</string>

    <key>WorkingDirectory</key>
    <string>/usr/local/tacticalagent</string>

    <key>UserName</key>
    <string>root</string>

    <key>GroupName</key>
    <string>wheel</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    </dict>
</dict>
</plist>
```

> Bagian baik: TRMM `.pkg` installer akan generate plist ini otomatis. Anda jarang harus tulis manual kecuali untuk debugging atau kustomisasi.

### 8.3.3 Manage via launchctl

```bash
# Load (start otomatis di reboot)
sudo launchctl load /Library/LaunchDaemons/com.tacticalrmm.tacticalagent.plist

# Unload (disable + stop)
sudo launchctl unload /Library/LaunchDaemons/com.tacticalrmm.tacticalagent.plist

# Start manually (sekali jalan, tanpa enable di reboot)
sudo launchctl start com.tacticalrmm.tacticalagent

# Stop
sudo launchctl stop com.tacticalrmm.tacticalagent

# Status
sudo launchctl list com.tacticalrmm.tacticalagent

# Output cont:
# {
#     "LimitLoadToSessionType" = "System";
#     "Label" = "com.tacticalrmm.tacticalagent";
#     "OnDemand" = false;
#     "LastExitStatus" = 0;
#     "PID" = 1234;
#     ...
# };
```

### 8.3.4 Modern API: `launchctl bootstrap`

Untuk macOS 10.10+:

```bash
sudo launchctl bootstrap system /Library/LaunchDaemons/com.tacticalrmm.tacticalagent.plist
sudo launchctl bootout system/com.tacticalrmm.tacticalagent
```

Lebih konsisten daripada `load`/`unload` lama. `MacAgentSupervisor` di Bab 5 bisa di-update untuk pakai ini di macOS modern.

## 8.4 Privacy permissions (TCC)

macOS Mojave+ punya **Transparency, Consent, and Control (TCC)** yang membatasi akses ke:

| Resource | Kapan butuh? |
|---|---|
| Full Disk Access | Kalau agent perlu read `/Users/*/Library/...` |
| Screen Recording | MeshCentral remote desktop |
| Accessibility | Otomasi UI / keyboard input |
| Microphone | Kalau ada fitur voice |
| Camera | Kalau ada fitur video |
| Location | Tidak relevan untuk RMM |

User **harus approve manual** di System Settings → Privacy & Security. Cara user approve:

1. App pertama kali request → muncul dialog
2. Kalau user dismiss → tidak ada cara re-trigger via API
3. User harus buka Settings, cari app, centang manual

### 8.4.1 Request Full Disk Access untuk TRMM agent

Kalau pakai MDM (Mobile Device Management) seperti Jamf atau Kandji, profile bisa pre-approve TCC otomatis. Untuk install manual, user harus:

1. Buka **System Settings → Privacy & Security → Full Disk Access**
2. Klik `+`, navigate ke `/usr/local/tacticalagent/tacticalagent`, add
3. Restart agent: `sudo launchctl kickstart -k system/com.tacticalrmm.tacticalagent`

> Untuk distribusi internal Hermes Network, pertimbangkan ship konfigurasi MDM atau dokumentasi step-by-step ke user.

## 8.5 Firewall (Application Firewall)

macOS Application Firewall block listener baru by default. TRMM agent connect outbound ke `api.hermesnetwork.cloud`, jadi tidak perlu listener — tapi MeshAgent perlu open port untuk remote desktop.

Saat MeshAgent pertama jalan, dialog akan muncul: "Do you want the application 'meshagent' to accept incoming network connections?" — user harus klik **Allow**.

Untuk auto-allow di MDM:

```xml
<!-- com.apple.alf.plist payload -->
<key>applications</key>
<array>
    <dict>
        <key>bundleid</key>
        <string>com.meshcentral.agent</string>
        <key>state</key>
        <integer>2</integer>     <!-- 2 = always allow -->
    </dict>
</array>
```

## 8.6 Bundle struktur Avalonia di macOS

```
HermesNetwork360Guard.app/
└── Contents/
    ├── Info.plist                       ← bundle metadata
    ├── MacOS/
    │   └── HermesNetwork360Guard       ← main executable (signed)
    ├── Resources/
    │   ├── HermesNetwork360Guard.icns  ← icon
    │   ├── log_pub.key                 ← public key untuk log encryption
    │   └── *.dylib, *.so, ...
    ├── _CodeSignature/
    └── embedded.provisionprofile        ← (optional, kalau pakai App Store)
```

`Info.plist` minimum:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleIdentifier</key>
    <string>com.hermesnetwork.guard</string>

    <key>CFBundleName</key>
    <string>Hermes Network 360 Guard</string>

    <key>CFBundleVersion</key>
    <string>8.4.0</string>

    <key>CFBundleShortVersionString</key>
    <string>8.4.0</string>

    <key>CFBundleExecutable</key>
    <string>HermesNetwork360Guard</string>

    <key>CFBundleIconFile</key>
    <string>HermesNetwork360Guard</string>

    <key>LSMinimumSystemVersion</key>
    <string>11.0</string>

    <key>NSHighResolutionCapable</key>
    <true/>

    <key>LSApplicationCategoryType</key>
    <string>public.app-category.utilities</string>

    <key>NSHumanReadableCopyright</key>
    <string>© 2026 Hermes Network Inc.</string>

    <!-- Privacy usage descriptions (optional, hanya kalau request akses) -->
    <key>NSAppleEventsUsageDescription</key>
    <string>Hermes Network 360 Guard needs to elevate to install agent components.</string>
</dict>
</plist>
```

## 8.7 osascript untuk privilege elevation

`MacAgentSupervisor` di Bab 5 pakai osascript:

```bash
osascript -e 'do shell script "installer -pkg /tmp/trmm.pkg -target /" with administrator privileges'
```

Yang terjadi:

1. macOS munculkan dialog: "Hermes Network 360 Guard wants to make changes."
2. User input password admin
3. Shell command jalan sebagai root
4. Dialog cuma muncul **sekali per session app** — selama app tetap jalan, request berikutnya tidak prompt lagi

**Kekurangan:**

- Kalau aplikasi butuh elevation di banyak tempat, dialog ini bisa muncul per-session di awal. UX bisa diperbaiki dengan upfront elevation request saat enroll.
- Tidak ada cara untuk pre-approve admin di code — selalu butuh user interaksi.

## 8.8 Distribusi PKG installer

Kalau Anda mau bundle Hermes Network 360 Guard sendiri menjadi `.pkg` (bukan `.app` di-drag-drop ke `/Applications`):

```bash
# Build component pkg
pkgbuild --root "bin/Release/net8.0/osx-arm64/publish" \
         --identifier "com.hermesnetwork.guard" \
         --version "8.4.0" \
         --install-location "/Applications" \
         HermesNetwork360Guard.component.pkg

# Sign
productsign --sign "Developer ID Installer: Hermes Network Inc. (XXXXXXXXXX)" \
            HermesNetwork360Guard.component.pkg \
            HermesNetwork360Guard.pkg

# Notarize
xcrun notarytool submit HermesNetwork360Guard.pkg \
  --keychain-profile AC_PASSWORD \
  --wait

# Staple
xcrun stapler staple HermesNetwork360Guard.pkg
```

Hasil `HermesNetwork360Guard.pkg` siap distribusi. End-user double-click → install via standard installer flow.

## 8.9 Build matrix

| Output | Command | Target |
|---|---|---|
| Windows x64 | `dotnet publish -c Release -r win-x64 --self-contained` | Win 10/11 64-bit |
| macOS ARM64 (M1/M2/M3) | `dotnet publish -c Release -r osx-arm64 --self-contained` | Apple Silicon |
| macOS x64 (Intel) | `dotnet publish -c Release -r osx-x64 --self-contained` | Mac Intel 2019+ |

Untuk universal Mac binary (single .app jalan di Intel + ARM):

```bash
# Build keduanya terpisah
dotnet publish -r osx-arm64 --self-contained -c Release -o publish/arm64
dotnet publish -r osx-x64 --self-contained -c Release -o publish/x64

# Lipo executable
lipo -create publish/arm64/HermesNetwork360Guard \
            publish/x64/HermesNetwork360Guard \
     -output publish/universal/HermesNetwork360Guard

# Repack jadi .app dengan executable universal
```

## 8.10 Testing checklist macOS

- [ ] Build di mesin macOS (atau CI dengan macOS runner)
- [ ] Sign + notarize berhasil, `spctl -a -v` return "accepted"
- [ ] Install di Mac bersih (clean VM atau wipe)
- [ ] Login dengan user non-admin → enrollment minta password admin (osascript dialog)
- [ ] Setelah install, `launchctl list com.tacticalrmm.tacticalagent` return PID > 0
- [ ] Agent muncul di TRMM dashboard dengan platform = "darwin"
- [ ] Run script via TRMM → stdout/stderr terkirim balik
- [ ] Reboot Mac → agent auto-start (RunAtLoad + KeepAlive)
- [ ] Uninstall via app → service unloaded, plist dihapus, binary hilang

---

[← Bab 7 Auth & Keamanan]({{ site.baseurl }}{% link docs/07-auth-keamanan.md %}){: .btn }
[Bab 9 — TRMM API Reference →]({{ site.baseurl }}{% link docs/09-api-reference.md %}){: .btn .btn-primary }
