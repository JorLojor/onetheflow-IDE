# Setup GitHub Actions — Tauri v2 Cross-Platform Build

Panduan untuk setup CI/CD otomatis yang build app Tauri lo untuk Linux, macOS, dan Windows sekaligus.

---

## Cara Pakai

**Setiap project Tauri yang mau di-build cross-platform, ikutin step ini:**

### Step 1: Bikin folder `.github/workflows/` di root project

```bash
cd the-note-flow
mkdir -p .github/workflows
```g

### Step 2: Bikin file workflow

```bash
nano .github/workflows/release.yml
```

Paste isi berikut:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

  # Bisa trigger manual dari GitHub UI
  workflow_dispatch:

jobs:
  release:
    permissions:
      contents: write

    strategy:
      fail-fast: false
      matrix:
        include:
          # Linux
          - platform: ubuntu-22.04
            args: ''
          # macOS ARM (Apple Silicon)
          - platform: macos-latest
            args: '--target aarch64-apple-darwin'
          # macOS Intel
          - platform: macos-latest
            args: '--target x86_64-apple-darwin'
          # Windows
          - platform: windows-latest
            args: ''

    runs-on: ${{ matrix.platform }}

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      # Linux dependencies
      - name: Install dependencies (Linux)
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            libwebkit2gtk-4.1-dev \
            build-essential \
            curl \
            wget \
            file \
            libxdo-dev \
            libssl-dev \
            libayatana-appindicator3-dev \
            librsvg2-dev

      # Setup Bun
      - name: Setup Bun
        uses: oven-sh/setup-bun@v2

      # Setup Rust
      - name: Setup Rust stable
        uses: dtolnay/rust-toolchain@stable

      # macOS: tambah target yang dibutuhin
      - name: Add macOS target (aarch64)
        if: matrix.args == '--target aarch64-apple-darwin'
        run: rustup target add aarch64-apple-darwin

      - name: Add macOS target (x86_64)
        if: matrix.args == '--target x86_64-apple-darwin'
        run: rustup target add x86_64-apple-darwin

      # Cache Rust build
      - name: Rust cache
        uses: swatinem/rust-cache@v2
        with:
          workspaces: './src-tauri -> target'

      # Install frontend dependencies
      - name: Install frontend dependencies
        run: bun install

      # Build & release
      - name: Build Tauri app
        uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # Fix linuxdeploy strip error di Ubuntu runner
          NO_STRIP: ${{ matrix.platform == 'ubuntu-22.04' && 'true' || '' }}
        with:
          tagName: ${{ github.ref_name }}
          releaseName: 'Release ${{ github.ref_name }}'
          releaseBody: |
            ## Downloadg

            Pilih installer sesuai OS:
            - **Linux:** `.AppImage` (portable) atau `.deb` / `.rpm`
            - **macOS:** `.dmg`
            - **Windows:** `.msi` atau `.exe`
          releaseDraft: true
          prerelease: false
          args: ${{ matrix.args }}
```

### Step 3: Pastiin `tauri.conf.json` udah bener

Buka `src-tauri/tauri.conf.json`, pastiin field ini ada:

```json
{
  "productName": "the-note-flow",
  "version": "0.1.0",
  "identifier": "com.the-note-flow.app",
  "bundle": {
    "active": true,
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ]
  }
}
```

### Step 4: Init git & push ke GitHub

```bash
cd the-note-flow
git init
git add .
git commit -m "initial commit"
git remote add origin https://github.com/USERNAME/the-note-flow.git
git branch -M main
git push -u origin main
```

### Step 5: Trigger build dengan tag

```bash
git tag v0.1.0
git push origin v0.1.0
```

Setelah push tag, GitHub Actions otomatis jalan dan build untuk semua platform.

---

## Cara Cek Build

1. Buka repo di GitHub
2. Klik tab **Actions** → liat workflow running
3. Setelah selesai, klik tab **Releases** → ada draft release dengan semua file:

```
the-note-flow_0.1.0_amd64.AppImage    # Linux (portable)
the-note-flow_0.1.0_amd64.deb         # Linux (Debian/Ubuntu)
the-note-flow-0.1.0-1.x86_64.rpm      # Linux (Fedora/RHEL)
the-note-flow_0.1.0_aarch64.dmg       # macOS (Apple Silicon)
the-note-flow_0.1.0_x64.dmg           # macOS (Intel)
the-note-flow_0.1.0_x64-setup.exe     # Windows (NSIS installer)
the-note-flow_0.1.0_x64_en-US.msi     # Windows (MSI installer)
```

4. Klik **Edit** → **Publish release** kalau udah siap dirilis

---

## Trigger Manual (tanpa tag)

Karena workflow udah ada `workflow_dispatch`, lo bisa trigger manual:

1. Buka repo → tab **Actions**
2. Klik workflow **Release** di sidebar kiri
3. Klik tombol **Run workflow**
4. Pilih branch → klik **Run workflow**

> ⚠️ Kalau trigger manual, release gak otomatis ke-create. Build artifacts ada di summary workflow run.

---

## Ulangi untuk Project Lain

Buat setiap project Tauri baru, cukup copy folder `.github/` ke root project:

```bash
cp -r the-note-flow/.github my-tauri-setup/
```

Atau bikin ulang dari Step 1.

---

## Tips

- **Draft release** = belum publik, cuma lo yang bisa liat. Publish manual setelah lo test.
- **Rust cache** bikin build kedua dan seterusnya jauh lebih cepet.
- **`NO_STRIP`** otomatis di-set cuma untuk Linux runner, biar gak kena error linuxdeploy.
- **Update versi** sebelum tag baru: edit `version` di `src-tauri/tauri.conf.json` dan `package.json`.
- **`GITHUB_TOKEN`** udah otomatis ada, gak perlu setup secrets manual.