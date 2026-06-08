# Workshop Pelatihan BGP — Universitas Muhammadiyah Malang

**Penyelenggara:** INDOSAT
**Tanggal:** 5–7 Juni 2026
**Lokasi:** Gedung TIK UMM
**Platform:** ContainerLab + FRRouting (FRR)

---

## Cara mulai

Clone repo ini:

```bash
git clone https://github.com/indi9o/bgp-workshop-labs.git
cd bgp-workshop-labs
```

Selesai. Semua file lab, handout, dan slide ada di dalam.

---

## Prasyarat

siapkan **VM Ubuntu 22.04** di laptop:

- VirtualBox terinstall di laptop
- VM Ubuntu 22.04 dengan RAM minimal 4 GB, disk 25 GB
- Docker terinstall di dalam VM (tanpa sudo)
- ContainerLab terinstall di dalam VM
- Image FRRouting sudah di-pull: `docker pull frrouting/frr:latest`
- Repo ini sudah di-clone di dalam VM

Panduan lengkap instalasi: [setup/setup-containerlab.md](setup/setup-containerlab.md)

> **Catatan:** Gunakan VM Ubuntu, bukan WSL2. ContainerLab tidak berjalan stabil di WSL2.

---

## Struktur repo

```
bgp-workshop-labs/
├── setup/                        ← panduan instalasi ContainerLab
├── hari1/  (Jumat 5 Juni)        ← BGP Fundamentals & eBGP
├── hari2/  (Sabtu 6 Juni)        ← iBGP, BGP Policy & Prefix Filtering
└── hari3/  (Minggu 7 Juni)       ← Multi-homing TE (LAB-04) & Troubleshooting (LAB-05)
```

Tiap folder hari berisi:
- **Slide presentasi** (`.pptx`)
- **Handout lab** (`.md`) — panduan step-by-step
- **File lab** — topology ContainerLab siap deploy

---

## Agenda Workshop

Lihat [agenda-workshop.md](agenda-workshop.md) untuk detail lengkap: tujuan pembelajaran per modul, topik yang dibahas, dan deskripsi setiap lab.

---

## Agenda singkat

| Hari | Tanggal | Topik | Mulai |
|---|---|---|---|
| Hari 1 | Jumat 5 Juni | BGP Fundamentals, eBGP Config, LAB-01 | 13.00 |
| Hari 2 | Sabtu 6 Juni | LAB-01 lanjutan, iBGP (LAB-02), BGP Policy & Prefix Filtering (LAB-03) | 09.00 |
| Hari 3 | Minggu 7 Juni | Multi-homing TE (LAB-04), Troubleshooting (LAB-05), Post-test | 10.30 |

---

## Kontak

Pertanyaan sebelum/sesudah workshop → group chat peserta.
