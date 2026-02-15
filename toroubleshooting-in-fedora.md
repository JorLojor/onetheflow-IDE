# Troubleshooting Tauri v2 — Fedora 43 (GNOME Wayland)

Dokumentasi masalah dan solusi yang ditemukan selama setup Tauri v2 di Fedora 43.

---

## Masalah 1: `failed to run linuxdeploy` saat build AppImage

### Gejala

Build `.deb` dan `.rpm` berhasil, tapi AppImage gagal:

```
Bundling my-tauri-setup_0.1.0_amd64.AppImageuname -a
Linux fedora 6.18.7-200.fc43.x86_64 #1 SMP PREEMPT_DYNAMIC Fri Jan 23 16:42:34 UTC 2026 x86_64 GNU/Linux
hostnamectl
  Transient hostname: fedora
     Static hostname: (unset)                              
           Icon name: computer-desktop
             Chassis: desktop 🖥️
          Machine ID: adee640a1d9741c9af5ab94d9cc67d32
             Boot ID: 8356b0d3315b4d898526c84cf210c401
    Operating System: Fedora Linux 43 (Workstation Edition)
         CPE OS Name: cpe:/o:fedoraproject:fedora:43
      OS Support End: Wed 2026-12-02
OS Support Remaining: 9month 2w 1d
              Kernel: Linux 6.18.7-200.fc43.x86_64
        Architecture: x86-64
     Hardware Vendor: Gigabyte Technology Co., Ltd.
      Hardware Model: B850M AORUS ELITE WIFI6E ICE
    Hardware Version: Default string-CF-WCP-ADO
    Firmware Version: F6
       Firmware Date: Thu 2025-07-31
        Firmware Age: 6month 2w 3d
```

Ini terjadi di hampir semua `.so` library: `libwayland-client`, `libxml2`, `libwebp`, `libzstd`, dll.

### Solusiuname -a
Linux fedora 6.18.7-200.fc43.x86_64 #1 SMP PREEMPT_DYNAMIC Fri Jan 23 16:42:34 UTC 2026 x86_64 GNU/Linux
hostnamectl
  Transient hostname: fedora
     Static hostname: (unset)                              
           Icon name: computer-desktop
             Chassis: desktop 🖥️
          Machine ID: adee640a1d9741c9af5ab94d9cc67d32
             Boot ID: 8356b0d3315b4d898526c84cf210c401
    Operating System: Fedora Linux 43 (Workstation Edition)
         CPE OS Name: cpe:/o:fedoraproject:fedora:43
      OS Support End: Wed 2026-12-02
OS Support Remaining: 9month 2w 1d
              Kernel: Linux 6.18.7-200.fc43.x86_64
        Architecture: x86-64
     Hardware Vendor: Gigabyte Technology Co., Ltd.
      Hardware Model: B850M AORUS ELITE WIFI6E ICE
    Hardware Version: Default string-CF-WCP-ADO
    Firmware Version: F6
       Firmware Date: Thu 2025-07-31
        Firmware Age: 6month 2w 3d

Ini bilang ke `linuxdeploy` supaya gak nge-strip debug symbols dari library `.so`. Efeknya cuma ukuran AppImage sedikit lebih besar, fungsionalnya sama.

**Solusi 2: Install FUSE (tambahan)**

```bash
sudo dnf install fuse fuse-libs
```

FUSE dibutuhin untuk menjalankan AppImage. Fedora 43 mungkin cuma punya FUSE 3, tapi `linuxdeploy` butuh FUSE 2.

**Solusi 3: Skip AppImage, build RPM & DEB aja**

Edit `src-tauri/tauri.conf.json`:

```json
{
  "bundle": {
    "targets": ["deb", "rpm"]
  }
}
```

**Solusi 4: Env variable alternatif**

```bash
APPIMAGETOOL_NO_FUSE=1 bun run tauri build
```

> **Catatan:** Untuk Fedora 43, `NO_STRIP=true` adalah solusi yang paling efektif.

---

## Masalah 2: App RPM force close saat dibuka dari GNOME App Launcher

### Gejala

- App **berjalan normal** kalau dijalankan dari terminal: `my-tauri-setup` atau `./my-tauri-setup_0.1.0_amd64.AppImage`
- App **force close** (langsung nutup tanpa muncul window) kalau diklik dari GNOME app launcher
- `coredumpctl list my-tauri-setup` → "No coredumps found" (app gak crash, tapi gak start)
- `gtk-launch my-tauri-setup` dari terminal → **berhasil** (app muncul normal)

### Penyebabuname -a
Linux fedora 6.18.7-200.fc43.x86_64 #1 SMP PREEMPT_DYNAMIC Fri Jan 23 16:42:34 UTC 2026 x86_64 GNU/Linux
hostnamectl
  Transient hostname: fedora
     Static hostname: (unset)                              
           Icon name: computer-desktop
             Chassis: desktop 🖥️
          Machine ID: adee640a1d9741c9af5ab94d9cc67d32
             Boot ID: 8356b0d3315b4d898526c84cf210c401
    Operating System: Fedora Linux 43 (Workstation Edition)
         CPE OS Name: cpe:/o:fedoraproject:fedora:43
      OS Support End: Wed 2026-12-02
OS Support Remaining: 9month 2w 1d
              Kernel: Linux 6.18.7-200.fc43.x86_64
        Architecture: x86-64
     Hardware Vendor: Gigabyte Technology Co., Ltd.
      Hardware Model: B850M AORUS ELITE WIFI6E ICE
    Hardware Version: Default string-CF-WCP-ADO
    Firmware Version: F6
       Firmware Date: Thu 2025-07-31
        Firmware Age: 6month 2w 3d

GNOME Shell di Wayland meng-cache desktop entry database. Setelah install RPM, GNOME belum nge-refresh cache-nya, jadi app launcher gak bisa nge-launch app dengan benar.

### Solusi

**Step 1: Update desktop database**

```bash
sudo update-desktop-database /usr/share/applications/
```

**Step 2: Restart GNOME session**

Karena Fedora 43 pakai Wayland (bukan X11), gak bisa restart GNOME shell dengan `Alt+F2` → `r`. Harus **log out lalu log in** kembali:

1. Klik pojok kanan atas desktop (area power/settings)
2. Klik **Log Out**
3. Masukin password lagi untuk login

> **Catatan:** Gak perlu restart komputer. Cukup log out → log in.

**Step 3: Coba buka dari app launcher**

Setelah log in kembali, app harusnya bisa dibuka dari GNOME app launcher.

### Debugging tambahan (kalau masih bermasalah)

Kalau masih force close setelah log out/log in, coba pendekatan ini untuk diagnosa:

**Cek crash log:**

```bash
journalctl --user -b -g "my-tauri-setup" --no-pager
coredumpctl list my-tauri-setup
```

**Test launch dengan environment minimal:**

```bash
env -i HOME=$HOME DISPLAY=$DISPLAY WAYLAND_DISPLAY=$WAYLAND_DISPLAY XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR /usr/bin/my-tauri-setup 2>&1
```

**Bikin debug wrapper untuk tangkep error:**

```bash
sudo nano /usr/local/bin/my-tauri-setup-debug.sh
```

Isi:

```bash
#!/bin/bash
/usr/bin/my-tauri-setup >> /tmp/tauri-debug.log 2>&1
```

```bash
sudo chmod +x /usr/local/bin/my-tauri-setup-debug.sh
```

Edit desktop entry sementara:

```bash
sudo nano /usr/share/applications/my-tauri-setup.desktop
```

Ganti `Exec=` jadi:

```
Exec=/usr/local/bin/my-tauri-setup-debug.sh
```

Klik dari app launcher, lalu cek log:

```bash
cat /tmp/tauri-debug.log
```

**Workaround WebKitGTK + Wayland (kalau emang crash beneran):**

```
Exec=env WEBKIT_DISABLE_DMABUF_RENDERER=1 my-tauri-setup
```

atau:

```
Exec=env GDK_BACKEND=x11 my-tauri-setup
```

---

## Referensi

- Tauri v2 Prerequisites: https://v2.tauri.app/start/prerequisites/
- Tauri v2 AppImage issue: https://github.com/tauri-apps/tauri/issues/14796
- linuxdeploy `.relr.dyn` incompatibility: terjadi karena Fedora 43 compile library dengan format ELF baru yang belum didukung oleh `strip` bawaan linuxdeploy

---

## Environment

```
OS: Fedora Linux 43 (Workstation Edition)
Kernel: Linux 6.18.7-200.fc43.x86_64
Desktop: GNOME (Wayland)
Hardware: Gigabyte B850M AORUS ELITE WIFI6E ICE
Bun: 1.3.2
Rust: rustc 1.93.0 (254b59607 2026-01-19)
Tauri CLI: latest (via bunx)
```