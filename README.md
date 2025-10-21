# Otomasi-Jaringan
Berisi program yang terkait dengan Otomasi Jaringan 

# Disable Unused Network Interfaces Automatically
Program Python ini dibuat untuk mendeteksi dan menonaktifkan interface jaringan yang tidak digunakan secara otomatis di sistem Linux.  
Tujuannya supaya konfigurasi jaringan tetap bersih, efisien, dan tidak menimbulkan konflik antar interface.

# Fitur Utama
- Membaca status semua interface menggunakan `ifconfig -a`
- Mendeteksi interface default secara otomatis lewat `ip route` atau `route -n`
- Menonaktifkan interface yang:
  - Tidak memiliki alamat IPv4 (`inet`)
  - Tidak dalam status UP
  - Tidak memiliki carrier (tidak ada link)
- Mengecualikan:
  - Loopback (`lo`)
  - Interface default
  - Interface yang ditentukan manual melalui `--exclude`
- Mendukung mode simulasi (`--dry-run`) untuk menjalankan tanpa benar-benar menonaktifkan interface

# Tujuan
Program ini membantu mahasiswa yang sedang belajar sistem jaringan untuk menjaga sistem tetap stabil dan teratur dengan cara menonaktifkan interface yang tidak terpakai secara otomatis.

# Cara Penggunaan
```bash
nano python3 disable_unused_ifaces.py

#!/usr/bin/env python3
"""
disable_unused_ifaces.py

Fungsi:
Baca status interface pakai ifconfig -a
Tentukan interface default (pakai ip route atau fallback ke route -n)
Disable interface yang:
    - tidak punya alamat IPv4 (tidak ada 'inet'), OR
    - flag UP tidak terpasang (mis. DOWN)
  kecuali:
    - loopback (lo)
    - interface default (yang mengarah ke default route)
    - interface yang user-specified lewat --exclude

Usage:
    sudo python3 disable_unused_ifaces.py [--dry-run] [--exclude eth0,eth1]
"""

import subprocess
import re
import argparse
import os
import shlex

def run_cmd(cmd):
    try:
        out = subprocess.check_output(cmd, shell=True, stderr=subprocess.DEVNULL, text=True)
        return out
    except subprocess.CalledProcessError:
        return ""

def detect_default_interface():
    # Prefer 'ip route'
    out = run_cmd("ip route show default")
    if out:
        m = re.search(r"default via [\d\.]+ dev (\S+)", out)
        if m:
            return m.group(1)
    # Fallback to route -n
    out = run_cmd("route -n")
    if out:
        for line in out.splitlines():
            if line.strip().startswith("0.0.0.0") or line.strip().startswith("default"):
                parts = re.split(r"\s+", line.strip())
                if len(parts) >= 8:
                    return parts[-1]
    return None

def parse_ifconfig():
    out = run_cmd("ifconfig -a")
    if not out:
        raise RuntimeError("ifconfig tidak tersedia atau tidak menghasilkan output.")
    blocks = re.split(r"\n(?=\S)", out)
    ifaces = {}
    for blk in blocks:
        lines = blk.strip().splitlines()
        if not lines:
            continue
        header = lines[0]
        m = re.match(r"^([^\s:]+)", header)
        if not m:
            continue
        name = m.group(1)
        text = "\n".join(lines)

        # Detect flags/UP
        flags_up = False
        if re.search(r"<[^>]UP[^>]>", text) or re.search(r"\bUP\b", text):
            flags_up = True

        # Detect IPv4 inet
        inet = None
        m2 = re.search(r"inet (?:addr:)?([\d\.]+)", text)
        if m2:
            inet = m2.group(1)

        # Detect RUNNING or no carrier
        running = bool(re.search(r"RUNNING", text))
        no_carrier = bool(re.search(r"no carrier", text, re.IGNORECASE))

        ifaces[name] = {
            "raw": text,
            "up_flag": flags_up,
            "inet": inet,
            "running": running,
            "no_carrier": no_carrier
        }
    return ifaces

def disable_interface(iface, dry_run=True):
    cmd = f"ifconfig {shlex.quote(iface)} down"
    if dry_run:
        print(f"[DRY-RUN] would run: {cmd}")
        return True
    try:
        subprocess.check_call(cmd, shell=True)
        print(f"[OK] Disabled interface: {iface}")
        return True
    except subprocess.CalledProcessError as e:
        print(f"[ERROR] Failed to disable {iface}: {e}")
        return False

def main():
    parser = argparse.ArgumentParser(description="Disable unused network interfaces (based on ifconfig).")
    parser.add_argument("--dry-run", action="store_true", help="Jangan benar-benar men-disable; hanya tampilkan apa yang akan dilakukan.")
    parser.add_argument("--exclude", type=str, default="", help="Daftar interface yang tidak boleh didisable, dipisah koma, mis: eth0,wlan0")
    args = parser.parse_args()

    if os.geteuid() != 0 and not args.dry_run:
        print("⚠️  Warning: Skrip ini sebaiknya dijalankan sebagai root (sudo). Gunakan --dry-run untuk simulasi.")

    # daftar interface yang tidak akan disentuh
    exclude_list = {"lo"}
    if args.exclude:
        for p in args.exclude.split(","):
            p = p.strip()
            if p:
                exclude_list.add(p)

    default_iface = detect_default_interface()
    if default_iface:
        exclude_list.add(default_iface)
    print(f"Default interface (excluded): {default_iface}")
    print(f"Always excluded: {', '.join(sorted(exclude_list))}")

    try:
        ifaces = parse_ifconfig()
    except RuntimeError as e:
        print("Error:", e)
        return

    print("\nDetected interfaces and status:")
    for name, info in ifaces.items():
        print(f"- {name}: up_flag={info['up_flag']}, running={info['running']}, inet={info['inet']}, no_carrier={info['no_carrier']}")

    # Tentukan interface yang akan didisable
    to_disable = []
    for name, info in ifaces.items():
        if name in exclude_list:
            continue
        if (info['inet'] is None) or (not info['up_flag']) or info['no_carrier']:
            to_disable.append(name)

    if not to_disable:
        print("\n✅ Tidak ada interface yang memenuhi kriteria untuk didisable.")
        return

    print("\nInterfaces yang akan didisable:")
    for i in to_disable:
        print(f" - {i}")

    # Jalankan disable atau simulasi
    for iface in to_disable:
        disable_interface(iface, dry_run=args.dry_run)

if __name__ == "__main__":
    main()

# Paste seluruh kode di atas → tekan Ctrl + O, Enter, lalu Ctrl + X.

# Ubah permission agar bisa dieksekusi
chmod +x disable_unused_ifaces.py

# Jalankan mode simulasi dulu (aman)
sudo python3 disable_unused_ifaces.py --dry-run

# Jalankan benar-benar (disable interface yang tidak aktif)
sudo python3 disable_unused_ifaces.py

# Maka hasilnya kemungkinan akan seperti ini:
Default interface (excluded): enp0s3
Always excluded: enp0s3, lo

Detected interfaces and status:
- dummy0: up_flag=True, running=True, inet=192.0.2.1, no_carrier=False
- enp0s3: up_flag=True, running=True, inet=10.0.2.15, no_carrier=False
- lo: up_flag=True, running=True, inet=127.0.0.1, no_carrier=False

✅ Tidak ada interface yang memenuhi kriteria untuk didisable.


# Kalau kamu mau ngetes apakah fungsi “disable”-nya benar, kamu bisa buat 1 interface dummy tambahan lalu lihat apakah skrip otomatis menonaktifkannya.

Misalnya:

sudo ip link add testdummy type dummy
sudo ifconfig testdummy up
sudo ifconfig testdummy 0.0.0.0


# Lalu jalankan skrip lagi:
sudo python3 disable_unused_ifaces.py


# Maka hasilnya kemungkinan akan seperti ini:
Default interface (excluded): enp0s3
Always excluded: enp0s3, lo

Detected interfaces and status:
- dummy0: up_flag=True, running=True, inet=192.0.2.1, no_carrier=False
- enp0s3: up_flag=True, running=True, inet=10.0.2.15, no_carrier=False
- lo: up_flag=True, running=True, inet=127.0.0.1, no_carrier=False
- testdummy: up_flag=True, running=True, inet=None, no_carrier=False

Interfaces yang akan didisable:
 - testdummy
[OK] Disabled interface: testdummy

