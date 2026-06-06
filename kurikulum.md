# Kurikulum Workshop Pelatihan BGP

**Penyelenggara:** INDOSAT

**Peserta:** Network Engineer Universitas Muhammadiyah Malang

**Tanggal:** 5–7 Juni 2026 | Gedung TIK UMM

**Platform:** ContainerLab + FRRouting (FRR) di VM Ubuntu 22.04

---

## Tujuan Workshop

Setelah mengikuti workshop ini, peserta mampu:

1. Memahami peran BGP dalam jaringan universitas multihoming (IDREN, ISP komersial, IIX)
2. Mengkonfigurasi eBGP dan iBGP menggunakan FRRouting di lingkungan lab
3. Menerapkan BGP policy dasar: prefix filtering, BGP community, traffic engineering
4. Mendiagnosa dan memperbaiki masalah BGP secara sistematis

---

## Target Peserta & Prasyarat

**Target:** Network Engineer Universitas Muhammadiyah Malang

**Prasyarat:**
- Memahami konsep routing dasar (routing table, next-hop, prefix/subnet)
- Familiar dengan CLI Linux atau CLI router (Cisco/MikroTik/Juniper — salah satu cukup)
- Mengerti perbedaan IGP dan EGP secara konseptual

---

## Ringkasan Kurikulum

| Hari | Topik Utama | Modul | Lab |
|---|---|---|---|
| Hari 1 | BGP Fundamentals & eBGP | MOD-01, MOD-02 | LAB-01 |
| Hari 2 | iBGP, Route Reflector & BGP Policy | MOD-03, MOD-04 | LAB-01 (lanjutan), LAB-02, LAB-03 |
| Hari 3 | Multi-homing & Troubleshooting | MOD-05, MOD-06 | LAB-04, LAB-05 |

---

## Hari 1 — Jumat, 5 Juni 2026 | 13.00–17.00

### MOD-01: BGP Fundamentals & Use Case UMM *(60 menit — Teori)*

**Yang dipelajari:**
- Mengapa jaringan UMM membutuhkan BGP — bukan hanya static route
- Perbedaan BGP dari IGP (OSPF, static) dan kapan masing-masing dipakai
- Komponen dasar BGP: AS, ASN, prefix, peer, session, state machine
- Topologi BGP di jaringan UMM: border router, tiga upstream, peran masing-masing

### MOD-02: eBGP Configuration (FRRouting) *(60 menit — Teori + Demo)*

**Yang dipelajari:**
- Mengkonfigurasi eBGP session antara border router UMM dan upstream (IDREN/ISP)
- Mengiklankan IP block UMM ke eBGP peer
- Memverifikasi status BGP session dan tabel routing dengan perintah `show`
- Mendiagnosis penyebab session tidak Established dari output verifikasi

### LAB-01: Setup eBGP — Border Router UMM ↔ IDREN PoP *(45 menit, lanjut Hari 2)*

**Yang dipraktikkan:**
- Deploy topologi ContainerLab: border router UMM dan IDREN PoP
- Konfigurasi eBGP session dari sisi UMM ke IDREN PoP
- Mengiklankan prefix UMM (`100.68.1.0/24`) ke IDREN
- Verifikasi session Established dan prefix terkirim dua arah

---

## Hari 2 — Sabtu, 6 Juni 2026 | 08.00–15.30

### LAB-01 Lanjutan *(60 menit)*

Melanjutkan dan menyelesaikan LAB-01 dari hari sebelumnya. Review bersama: solusi, common mistakes, Q&A.

### MOD-03: iBGP & Route Reflector *(45 menit — Teori + Demo)*

**Yang dipelajari:**
- Mengapa iBGP dibutuhkan ketika UMM punya lebih dari satu BGP-speaking router
- Perbedaan perilaku iBGP vs eBGP: next-hop, TTL, split horizon
- Masalah full-mesh iBGP dan solusinya via Route Reflector
- Konfigurasi Route Reflector di FRRouting

### LAB-02: iBGP Full-mesh & Route Reflector *(60 menit)*

**Yang dipraktikkan:**
- Konfigurasi iBGP session antara dua border router UMM menggunakan loopback
- Verifikasi route dari IDREN di-distribute ke border router kedua via iBGP
- Konfigurasi Route Reflector dan verifikasi efeknya terhadap distribusi route

### MOD-04: BGP Policy — Prefix Filtering & Communities *(60 menit — Teori + Demo)*

**Yang dipelajari:**
- Menerapkan prefix-list dan route-map untuk filter prefix inbound dan outbound
- Mencegah UMM menjadi transit antara dua upstream (routing leak)
- Menandai route dengan BGP community untuk kebutuhan policy lanjutan

### LAB-03: Prefix Filtering & BGP Communities *(60 menit)*

**Yang dipraktikkan:**
- Outbound filter: border router UMM hanya mengiklankan prefix milik UMM ke semua upstream
- Inbound filter: membuang prefix bogon yang dikirim upstream
- Tagging route dengan BGP community dan verifikasi propagasinya

---

## Hari 3 — Minggu, 7 Juni 2026 | 11.00–17.15

### MOD-05: BGP Multi-homing — Traffic Engineering *(60 menit — Teori + Demo)*

**Yang dipelajari:**
- Perbedaan outbound vs inbound traffic engineering di konteks multi-homing
- Menggunakan Local Preference untuk mengarahkan traffic keluar UMM lewat upstream yang tepat
- Menggunakan AS-path prepending untuk mempengaruhi traffic masuk ke UMM
- Menggunakan MED untuk komunikasi preferensi ke upstream yang sama

### LAB-04: Multi-homing — 3 Upstream (IDREN + ISP + IIX) *(60 menit)*

**Yang dipraktikkan:**
- Konfigurasi Local Preference: IDREN (200) > IIX (150) > ISP (100)
- Verifikasi BGP best-path selection memilih path dengan local-pref tertinggi
- Konfigurasi AS-path prepending ke ISP
- Verifikasi failover otomatis saat IDREN disimulasikan down

### MOD-06: BGP Troubleshooting — Metodologi & Tools *(45 menit — Teori + Demo)*

**Yang dipelajari:**
- Metodologi sistematis untuk diagnosa masalah BGP (bottom-up)
- Membaca output `show` dan mengidentifikasi root cause dari gejala yang ada
- Mendiagnosa tiga kategori masalah: session tidak Established, prefix tidak muncul, traffic tidak lewat jalur yang diharapkan

### LAB-05: Diagnose & Fix Broken BGP Session *(45 menit)*

**Yang dipraktikkan:**
- Menerapkan metodologi troubleshooting BGP secara sistematis
- Mendiagnosa dan memperbaiki tiga bug yang sengaja ditanam di topologi
- Menggunakan output `show bgp summary`, `show bgp ipv4 unicast neighbors`, dan log untuk identifikasi masalah

---

## Tools & Environment

| Tool | Versi | Fungsi |
|---|---|---|
| VirtualBox | latest | Hypervisor untuk VM Ubuntu |
| Ubuntu | 22.04 LTS | Environment tempat lab berjalan |
| Docker | latest | Container runtime |
| ContainerLab | latest | Deploy topologi jaringan virtual |
| FRRouting (FRR) | 8.4 | Routing daemon — syntax vtysh mirip Cisco IOS |

---

## Topologi Lab

Semua lab menggunakan topologi berbasis jaringan universitas IDREN:

```
  ┌─────────────────────────────────────────────┐
  │  UMM (AS 65010 / prefix: 100.68.x.x)        │
  │  [border router]                            │
  └────────────────┬────────────────────────────┘
              eBGP │  eBGP  │  eBGP
                   │        │
         [IDREN PoP]  [ISP]  [IIX]
          AS 65000   AS 65100  AS 65200
```

IP addressing menggunakan range RFC 6598 (100.64.0.0/10) — khusus untuk shared address space, tidak dipakai di internet publik.

---

## Referensi Utama

| Dokumen | Keterangan |
|---|---|
| RFC 4271 | BGP-4 — standar session, message types, best-path selection |
| RFC 1997 | BGP Communities |
| RFC 4456 | Route Reflection |
| RFC 6996 | ASN private ranges |
| RFC 6598 | Shared Address Space (100.64.0.0/10) — dipakai di semua lab |
| FRRouting docs | https://docs.frrouting.org/en/latest/bgp.html |
