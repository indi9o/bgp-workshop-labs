# Handout LAB-05 — Diagnose & Fix Broken BGP Session

Workshop BGP — Hari 3 | 7 Juni 2026

---

## Topologi

```
  ┌──────────────────────────┐         ┌──────────────────────────┐
  │  umm-border (AS 65010)   ├─────────┤  idren-pop (AS 65000)    │
  │  100.64.0.2/30           │         │  100.64.0.1/30           │
  │                          │         │  Prefix: 100.68.0.0/16   │
  └──────────────────────────┘         └──────────────────────────┘
```

Topologi sederhana 2 node. **Tiga bug sudah ditanam** — temukan dan perbaiki dari gejala, bukan dari konfigurasi.

**Kondisi akhir yang benar:**
- Session `Established` antara umm-border ↔ idren-pop
- umm-border advertise `100.72.1.0/24` dan `100.72.2.0/24`
- umm-border terima `100.68.0.0/16` dari idren-pop → masuk routing table

---

## Step 1 — Deploy

```bash
mkdir ~/lab05 && cd ~/lab05
# Salin file dari trainer: lab05.clab.yml, umm-border/, idren-pop/
containerlab deploy -t lab05.clab.yml
```

Tunggu 15–20 detik agar FRR daemon startup selesai.

---

## Step 2 — Mulai diagnosis (jangan lihat konfigurasi dulu)

```bash
docker exec -it clab-lab05-troubleshooting-umm-border bash
```

```bash
# Layer 3: connectivity ada?
ping 100.64.0.1 -c 3

# BGP: status session?
vtysh -c "show bgp summary"
```

Diagnosis bottom-up: Layer 3 → TCP 179 → BGP session → prefix.

---

## Bug #1 — Session tidak Established

**Gejala:** `show bgp summary` → state `Active`

```bash
# Cek log BGP
tail -20 /var/log/frr/bgpd.log
```

atau di vtysh:
```
show bgp ipv4 unicast neighbors 100.64.0.1
```

Perhatikan NOTIFICATION message — apa yang dinegosiasikan saat OPEN?

<details>
<summary>▶ Spoiler Bug #1</summary>

**Root cause:** `remote-as 65099` di umm-border — seharusnya `65000`.

```
vtysh
configure terminal
router bgp 65010
 neighbor 100.64.0.1 remote-as 65000
end
write memory
```

Tunggu 30 detik, cek `show bgp summary` → harus `Established`.

</details>

---

## Bug #2 — Prefix tidak diiklankan

**Gejala:** session Established, tapi `100.72.1.0/24` tidak muncul di advertised-routes.

```
show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes
show bgp ipv4 unicast
show ip route 100.72.1.0/24
```

Petunjuk: BGP `network` statement hanya iklankan prefix yang ada di routing table (exact match).

<details>
<summary>▶ Spoiler Bug #2</summary>

**Root cause:** Tidak ada `ip route 100.72.1.0/24 null0` — prefix tidak ada di routing table.

```
configure terminal
ip route 100.72.1.0/24 null0
ip route 100.72.2.0/24 null0
end
write memory
```

Verifikasi: `show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes` → harus ada `100.72.1.0/24`.

</details>

---

## Bug #3 — Prefix UMM tidak diterima di idren-pop

**Gejala:** `PfxRcd: 0` di idren-pop meski umm-border sudah advertise.

Masuk ke idren-pop:
```bash
docker exec -it clab-lab05-troubleshooting-idren-pop bash
vtysh
```

```
show bgp ipv4 unicast neighbors 100.64.0.2 received-routes
show bgp ipv4 unicast neighbors 100.64.0.2 routes
show route-map
```

Petunjuk: `received-routes` ada isinya tapi `routes` kosong → ada inbound policy yang memblokir.

<details>
<summary>▶ Spoiler Bug #3</summary>

**Root cause:** Route-map `BLOCK-INBOUND` dengan prefix-list `BLOCK-ALL` aktif di idren-pop.

```
configure terminal
router bgp 65000
 address-family ipv4 unicast
  no neighbor 100.64.0.2 route-map BLOCK-INBOUND in
 exit-address-family
end
write memory
clear ip bgp 100.64.0.2 soft in
```

</details>

---

## Verifikasi akhir

| Check | Node | Command | Expected |
|---|---|---|---|
| Session Established | umm-border | `show bgp summary` | State Established |
| Prefix UMM diiklankan | umm-border | `show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes` | `100.72.1.0/24`, `100.72.2.0/24` |
| Prefix IDREN diterima | umm-border | `show ip route bgp` | `100.68.0.0/16` B ada |
| Prefix UMM diterima | idren-pop | `show bgp ipv4 unicast` | `100.72.1.0/24`, `100.72.2.0/24` |

---

## Ringkasan bug

| Bug | Layer | Gejala | Root Cause |
|---|---|---|---|
| #1 | BGP session | State `Active`, NOTIFICATION Bad Peer AS | `remote-as 65099` → seharusnya `65000` |
| #2 | Prefix advertisement | Prefix tidak di advertised-routes | Tidak ada `ip route 100.72.x.0/24 null0` |
| #3 | Inbound policy | `PfxRcd: 0` di idren-pop | Route-map `BLOCK-ALL` inbound di idren-pop |

---

## Cleanup

```bash
containerlab destroy -t lab05.clab.yml
```

---

*Setelah LAB-05: post-test & penutupan.*
