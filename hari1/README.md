# Hari 1 — Jumat, 5 Juni 2026

**Mulai:** 13.00 (setelah sholat Jumat)
**Selesai:** 17.00
**Topik:** BGP Fundamentals & eBGP Configuration

---

## Agenda

| Waktu | Sesi |
|---|---|
| 13.00–13.30 | Registrasi & pembukaan |
| 13.30–14.00 | Pengenalan ContainerLab |
| 14.00–15.00 | MOD-01: BGP Fundamentals & Use Case UMM |
| 15.00–15.15 | Coffee break |
| 15.15–16.15 | MOD-02: eBGP Configuration (FRRouting) |
| 16.15–17.00 | LAB-01: Setup eBGP (mulai) |

---

## Isi folder ini

| File/Folder | Keterangan |
|---|---|
| `BGP-Workshop-Hari1.pptx` | Slide presentasi hari 1 |
| `handout-lab01.md` | Panduan step-by-step LAB-01 |
| `lab01/` | File topology ContainerLab untuk LAB-01 |

---

## LAB-01: Setup eBGP Border Router UMM ↔ IDREN PoP

### Topologi

```
  [umm-border]  ──────────────────  [idren-pop]
  AS 65010                           AS 65000
  IP: 100.64.0.2/30                  IP: 100.64.0.1/30
  Prefix: 100.68.1.0/24              (pre-configured)
```

`idren-pop` sudah dikonfigurasi — kerjakan sisi `umm-border` saja.

### Deploy lab

```bash
cd bgp-workshop-labs/hari1/lab01
containerlab deploy -t lab01.clab.yml
```

Tunggu 15–20 detik, lalu ikuti langkah di `handout-lab01.md`.

### Cleanup (setelah lab selesai)

```bash
containerlab destroy -t lab01.clab.yml
```

---

> **Catatan:** LAB-01 dilanjutkan di awal Hari 2 (60 menit tambahan).
> Jangan destroy lab jika belum selesai — biarkan berjalan sampai besok.
