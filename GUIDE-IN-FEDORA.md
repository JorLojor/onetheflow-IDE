# Panduan Tauri v2 — Bun + React + TypeScript (Fedora 43)

---

## Spesifikasi Mesin

| Item | Detail |
|------|--------|
| OS | Fedora Linux 43 (Workstation Edition) |
| Kernel | Linux 6.18.7-200.fc43.x86_64 |
| Architecture | x86-64 |
| Desktop | GNOME (Wayland) |
| Hardware | Gigabyte B850M AORUS ELITE WIFI6E ICE |
| Bun | 1.3.2 |
| Rust | rustc 1.93.0 (254b59607 2026-01-19) |

---

## 1. Instalasi

### 1.1 System Dependencies (Fedora 43)

Pastiin Rust dan Bun udah terinstall. Lalu install system dependencies yang dibutuhin Tauri:

```bash
sudo dnf check-update

sudo dnf install webkit2gtk4.1-devel \
  openssl-devel \
  curl \
  wget \
  file \
  libappindicator-gtk3-devel \
  librsvg2-devel \
  libxdo-devel

sudo dnf group install "c-development"
```

**Dependency tambahan untuk AppImage bundling:**

```bash
sudo dnf install fuse fuse-libs
```

**Penjelasan package:**

| Package | Fungsi |
|---------|--------|
| `webkit2gtk4.1-devel` | WebView engine — ini yang nge-render UI web lo di desktop |
| `openssl-devel` | TLS/SSL support |
| `libappindicator-gtk3-devel` | System tray icon |
| `librsvg2-devel` | SVG rendering untuk icon |
| `libxdo-devel` | Keyboard/mouse automation library |
| `c-development` group | gcc, make, pkg-config, dll |
| `fuse` + `fuse-libs` | Dibutuhin untuk AppImage bundling |

### 1.2 Verifikasi Instalasi

Setelah install, pastiin semuanya ready:

```bash
rustc --version       # rustc 1.93.0
cargo --version       # ikut terinstall bareng rustc
bun --version         # 1.3.2
pkg-config --version  # pastiin ada
```

### 1.3 Buat Project Baru

```bash
bun create tauri-app
```

Nanti bakal muncul prompt interaktif, pilih seperti ini:

```
✔ Project name · my-app
✔ Identifier · com.my-app.app
✔ Choose which language to use for your frontend · TypeScript / JavaScript
✔ Choose your package manager · bun
✔ Choose your UI template · React
✔ Choose your UI flavor · TypeScript
```

Atau kalau mau langsung skip prompt:

```bash
bunx create-tauri-app my-app --template react-ts --manager bun
```

### 1.4 Install Dependencies

```bash
cd my-app
bun install
```

---

## 2. Menjalankan (Development Mode)

```bash
bun run tauri dev
```

**Apa yang terjadi:**

1. Bun nge-start Vite dev server untuk frontend React (biasanya di `http://localhost:1420`)
2. Tauri nge-compile Rust backend (`src-tauri/`)
3. Window native terbuka dengan UI React lo di dalamnya
4. Hot reload aktif — edit file React dan langsung ke-update di window

> ⚠️ **Compile pertama kali agak lama** karena Rust harus download dan compile semua crates. Setelah itu incremental build jadi cepet.

### Struktur Project

```
my-app/
├── src/                    # Frontend (React + TypeScript)
│   ├── App.tsx             # Komponen utama
│   ├── main.tsx            # Entry point React
│   ├── App.css
│   └── assets/
├── src-tauri/              # Backend (Rust)
│   ├── src/
│   │   └── lib.rs          # Rust commands & logic
│   ├── Cargo.toml          # Rust dependencies
│   ├── tauri.conf.json     # Konfigurasi Tauri
│   ├── capabilities/       # Permission system
│   └── icons/              # App icons
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
└── bun.lock
```

### Tips Development

**Cek info environment Tauri:**

```bash
bunx tauri info
```

Ini bakal nunjukin versi semua dependency dan status instalasi.

**Panggil Rust dari React:**

```rust
// src-tauri/src/lib.rs
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}! You've been greeted from Rust!", name)
}
```

```typescript
// src/App.tsx
import { invoke } from "@tauri-apps/api/core";

const greeting = await invoke("greet", { name: "World" });
```

---

## 3. Build untuk Distribusi

### 3.1 Build di Linux (Fedora 43)

**PENTING: Di Fedora 43, pakai `NO_STRIP=true` untuk menghindari error linuxdeploy.**

```bash
NO_STRIP=true bun run tauri build
```

Output ada di `src-tauri/target/release/bundle/`:

```
bundle/
├── deb/          # .deb untuk Debian/Ubuntu
├── rpm/          # .rpm untuk Fedora/RHEL
└── appimage/     # .AppImage (portable, jalan di hampir semua distro)
```

**Untuk distribusi ke seluruh distro Linux, pakai `.AppImage`** — ini format portable yang udah bundle semua dependency, jadi user tinggal download, `chmod +x`, dan jalanin.

### 3.2 Build untuk macOS

Lo **harus build di mesin macOS** (atau macOS VM/CI). Tauri gak support cross-compile ke macOS dari Linux.

**Di mesin macOS:**

1. Install Xcode Command Line Tools: `xcode-select --install`
2. Install Rust: `curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh`
3. Install Bun: `curl -fsSL https://bun.sh/install | bash`
4. Clone repo, `bun install`, lalu `bun run tauri build`

Output: `.app` bundle dan `.dmg` installer.

### 3.3 Build untuk Windows

Lo **harus build di mesin Windows** (atau Windows VM/CI). Tauri gak support cross-compile ke Windows dari Linux.

**Di mesin Windows:**

1. Install [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/) — pilih "Desktop development with C++"
2. WebView2 udah pre-installed di Windows 10/11
3. Install Rust: download dari https://www.rust-lang.org/tools/install
4. Install Bun: `powershell -c "irm bun.sh/install.ps1 | iex"`
5. Clone repo, `bun install`, lalu `bun run tauri build`

Output: `.msi` installer dan `.exe` (NSIS installer).

### 3.4 Otomasi Build Cross-Platform dengan GitHub Actions

Ini cara paling praktis buat build di semua platform sekaligus. Buat file `.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    permissions:
      contents: write
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: ubuntu-22.04
            args: ''
          - platform: macos-latest
            args: '--target aarch64-apple-darwin'
          - platform: macos-latest
            args: '--target x86_64-apple-darwin'
          - platform: windows-latest
            args: ''

    runs-on: ${{ matrix.platform }}

    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies (Ubuntu)
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y libwebkit2gtk-4.1-dev \
            build-essential curl wget file \
            libxdo-dev libssl-dev \
            libayatana-appindicator3-dev librsvg2-dev

      - name: Setup Bun
        uses: oven-sh/setup-bun@v2

      - name: Setup Rust
        uses: dtolnay/rust-action/setup@v1

      - name: Rust cache
        uses: swatinem/rust-cache@v2
        with:
          workspaces: './src-tauri -> target'

      - name: Install frontend dependencies
        run: bun install

      - name: Build Tauri
        uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tagName: ${{ github.ref_name }}
          releaseName: 'v__VERSION__'
          releaseBody: 'See the assets to download and install this version.'
          releaseDraft: true
          prerelease: false
          args: ${{ matrix.args }}
```

**Cara pakai:**

```bash
git tag v0.1.0
git push origin v0.1.0
```

GitHub Actions otomatis build untuk Linux, macOS (ARM + Intel), dan Windows, lalu upload hasilnya ke GitHub Releases.

---

## Ringkasan Command

| Aksi | Command |
|------|---------|
| Buat project baru | `bun create tauri-app` |
| Install dependencies | `bun install` |
| Dev mode (hot reload) | `bun run tauri dev` |
| Build production (Fedora 43) | `NO_STRIP=true bun run tauri build` |
| Cek environment | `bunx tauri info` |
| Generate app icons | `bunx tauri icon ./app-icon.png` |

---

## Catatan Penting

- **Cross-compile tidak didukung.** Lo harus build di masing-masing OS target, atau pakai CI/CD seperti GitHub Actions.
- **AppImage** adalah format terbaik untuk distribusi Linux universal — portable dan gak perlu install.
- **Tauri v2 butuh `webkit2gtk-4.1`** (bukan 4.0), Fedora 43 udah support.
- **Ukuran binary Tauri sangat kecil** dibanding Electron (~2-10 MB vs ~150+ MB) karena pakai WebView bawaan OS.
- **Pertama kali build agak lama** karena compile Rust crates, tapi setelah itu incremental.
- **Fedora 43 butuh `NO_STRIP=true`** waktu build karena linuxdeploy bawaan Tauri gak kompatibel dengan format library `.so` baru.
- **GNOME Wayland** kadang perlu log out → log in setelah install RPM biar app launcher bisa nge-launch app dengan bener.