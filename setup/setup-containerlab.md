# Setup Environment Lab — Panduan Peserta

Status: `final`
Last updated: 2026-06-04

---

## Ringkasan

Semua lab dijalankan di dalam **VM Ubuntu**. ContainerLab dan Docker berjalan lebih stabil di VM Linux dibanding WSL2 karena tidak ada lapisan virtualisasi tambahan dan networking berjalan native.

Yang dibutuhkan di laptop:
1. **VirtualBox** — hypervisor gratis untuk menjalankan VM Ubuntu
2. **VM Ubuntu 22.04** — environment tempat ContainerLab dan Docker berjalan
3. **ContainerLab + Docker** — terinstall di dalam VM
4. **Image FRRouting** — sudah di-pull sebelum workshop

Estimasi waktu setup: 45–60 menit (tergantung kecepatan internet dan laptop)

---

## Persyaratan Sistem (laptop)

| Item | Minimum | Rekomendasi |
|---|---|---|
| RAM | 8 GB | 16 GB |
| Disk kosong | 30 GB | 50 GB |
| CPU | 4 core, support virtualisasi | 4+ core |
| OS | Windows 10/11, macOS, Linux | — |

Pastikan **virtualisasi hardware aktif di BIOS** (Intel VT-x / AMD-V). Cara cek di Windows: buka Task Manager → tab Performance → CPU → lihat "Virtualization: Enabled".

---

## Langkah 1 — Install VirtualBox

Download VirtualBox dari: **https://www.virtualbox.org/wiki/Downloads**

Pilih versi sesuai OS laptop:
- Windows: `Windows hosts`
- macOS: `macOS / Intel hosts` atau `macOS / Arm64 hosts` (M1/M2/M3)

Jalankan installer, ikuti langkah default. Restart laptop jika diminta.

---

## Langkah 2 — Download ISO Ubuntu 22.04

Download Ubuntu 22.04 LTS Server (lebih ringan, tidak perlu GUI):

**https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso**

Ukuran: ~2 GB. Simpan di lokasi yang mudah ditemukan.

> Boleh juga pakai Ubuntu 22.04 Desktop jika lebih familiar — tapi Server lebih ringan.

---

## Langkah 3 — Buat VM di VirtualBox

Buka VirtualBox → klik **New**:

| Setting | Nilai |
|---|---|
| Name | bgp-lab |
| Type | Linux |
| Version | Ubuntu (64-bit) |
| RAM | **4096 MB** (minimum) — lebih besar lebih baik |
| CPU | **2 core** (minimum) |
| Disk | **25 GB** (dynamically allocated) |

**Network adapter:**
- Adapter 1: **NAT** (untuk akses internet dari dalam VM)

Setelah VM dibuat, buka **Settings → Storage → Controller IDE** → klik ikon disk → pilih ISO Ubuntu yang sudah didownload.

---

## Langkah 4 — Install Ubuntu di VM

Nyalakan VM → ikuti proses instalasi Ubuntu Server:

1. Pilih bahasa: **English**
2. Layout keyboard: sesuai keyboard laptop
3. Network: biarkan default (DHCP via NAT)
4. Storage: **Use entire disk** → Done → Continue
5. Profile:
   - Name: bebas (misal: `peserta`)
   - Server name: `bgp-lab`
   - Username: `peserta` (atau nama lain yang mudah diingat)
   - Password: isi dan ingat — dipakai untuk login dan sudo
6. **Install OpenSSH server: YA** (centang)
7. Featured snaps: lewati saja (tidak perlu centang)
8. Tunggu instalasi selesai → **Reboot Now**

Setelah reboot, login dengan username dan password yang dibuat tadi.

---

## Langkah 5 — Install Docker

Login ke VM, lalu jalankan:

```bash
# Update package list
sudo apt update && sudo apt upgrade -y

# Install dependensi
sudo apt install -y ca-certificates curl gnupg

# Tambah Docker GPG key dan repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Tambahkan user ke group docker supaya bisa jalankan docker **tanpa sudo**:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verifikasi:

```bash
docker ps
```

Output yang benar (tidak ada error permission):
```
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

---

## Langkah 6 — Install ContainerLab

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
```

Verifikasi:

```bash
containerlab version
```

Output contoh:
```
                           _                   _       _
 ____ ___  ____ | |_  ____ _ ____  ____  ____| | ____| | _
...
    version: 0.75.0
```

ContainerLab juga berjalan **tanpa sudo** karena user sudah masuk group docker.

---

## Langkah 7 — Pull image FRRouting

```bash
docker pull frrouting/frr:latest
```

Proses ini butuh 2–5 menit. Image ukurannya sekitar 200 MB.

Verifikasi:

```bash
docker images | grep frr
```

Output yang benar:
```
frrouting/frr   latest   abc123def456   2 weeks ago   198MB
```

---

## Langkah 8 — Install git dan clone repo lab

```bash
sudo apt install -y git
git clone https://github.com/indi90/bgp-workshop-labs.git
cd bgp-workshop-labs
ls
```

Harus muncul: `README.md  hari1  hari2  hari3  setup`

---

## Langkah 9 — Verifikasi ContainerLab berjalan

```bash
cd ~/bgp-workshop-labs/hari1/lab01
containerlab deploy -t lab01.clab.yml
```

Output yang benar (bagian akhir):
```
╭────────────────────────────┬──────────────────────┬─────────┬───────────────────╮
│            Name            │      Kind/Image      │  State  │   IPv4/6 Address  │
├────────────────────────────┼──────────────────────┼─────────┼───────────────────┤
│ clab-lab01-ebgp-idren-pop  │ frrouting/frr:latest │ running │ 172.20.20.x       │
│ clab-lab01-ebgp-umm-border │ frrouting/frr:latest │ running │ 172.20.20.x       │
╰────────────────────────────┴──────────────────────┴─────────┴───────────────────╘
```

Setelah verifikasi:

```bash
containerlab destroy -t lab01.clab.yml
```

Setup selesai. VM siap dipakai untuk workshop.

---

## Cara akses VM saat workshop

Dari VirtualBox, bisa langsung ketik di terminal VM. Atau, jika lebih nyaman pakai terminal laptop:

**Aktifkan port forwarding di VirtualBox** (opsional tapi nyaman):

Settings VM → Network → Adapter 1 (NAT) → Advanced → Port Forwarding:

| Name | Protocol | Host Port | Guest Port |
|---|---|---|---|
| ssh | TCP | 2222 | 22 |

Lalu dari terminal laptop:
```bash
ssh -p 2222 peserta@127.0.0.1
```

---

## Troubleshooting

### VirtualBox tidak bisa start VM — "VT-x is disabled"

Virtualisasi belum aktif di BIOS. Cara mengaktifkan:
- **Lenovo:** restart → tekan F1/F2 → Security → Virtualization → Enable
- **HP:** restart → tekan F10 → Advanced → System Options → Enable VT-x
- **Dell:** restart → tekan F2 → Virtualization Support → Enable

### `docker ps` gagal — "permission denied"

Belum masuk group docker. Jalankan:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

Jika masih gagal, logout dari VM dan login ulang.

### `containerlab deploy` gagal — "cannot create container"

Pastikan Docker service aktif:
```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### `docker pull` lambat atau gagal

Cek koneksi internet dari dalam VM:
```bash
ping 8.8.8.8 -c 3
```

Jika tidak bisa ping → cek network adapter VM sudah NAT dan dalam keadaan "Connected".

### VM tidak dapat IP (tidak bisa internet)

Di VirtualBox: Settings VM → Network → pastikan Adapter 1 aktif dan tipe **NAT**. Klik OK, restart VM.

---

## Cross-references

- [LAB-01: Setup eBGP](../hari1/handout-lab01.md)
