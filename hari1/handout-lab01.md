# Handout LAB-01 — Setup eBGP: Border Router UMM ↔ IDREN PoP

Workshop BGP — Hari 1 | 5 Juni 2026

---

## Topologi

```
  [umm-border]  ──────────────────  [idren-pop]
  AS 65010                           AS 65000
  eth1: 100.64.0.2/30                eth1: 100.64.0.1/30
  Prefix: 100.72.1.0/24,             (pre-configured trainer)
          100.72.2.0/24
```

**Tugas:** Konfigurasi BGP di `umm-border`. Router `idren-pop` sudah dikonfigurasi trainer.

---

## Informasi Peering

| Parameter | Nilai |
|---|---|
| IP peer (IDREN) | `100.64.0.1` |
| ASN peer (IDREN) | `65000` |
| IP lokal (UMM di link) | `100.64.0.2` |
| ASN lokal (UMM) | `65010` |
| Prefix UMM yang diiklankan | `100.72.1.0/24`, `100.72.2.0/24` |

---

## Step 1 — Deploy topologi

```bash
mkdir ~/lab01 && cd ~/lab01
# Salin file dari trainer: lab01.clab.yml, daemons, idren-pop/frr.conf, umm-border/frr.conf
containerlab deploy -t lab01.clab.yml
```

Tunggu 15–20 detik agar container dan FRR selesai startup.

---

## Step 2 — Verifikasi konektivitas dasar

Masuk ke `umm-border`:

```bash
docker exec -it clab-lab01-ebgp-umm-border bash
```

Ping ke IDREN PoP:

```bash
ping 100.64.0.1 -c 3
```

Harus berhasil sebelum lanjut. Jika gagal → cek: `ip addr show eth1`.

---

## Step 3 — Konfigurasi BGP di umm-border

```bash
vtysh
```

```
configure terminal

router bgp 65010
 no bgp ebgp-requires-policy
 bgp router-id 100.64.0.2
 neighbor 100.64.0.1 remote-as 65000
 neighbor 100.64.0.1 description "eBGP ke IDREN PoP"
 !
 address-family ipv4 unicast
  network 100.72.1.0/24
  network 100.72.2.0/24
  neighbor 100.64.0.1 activate
 exit-address-family

end
write memory
```

Tunggu 30–60 detik setelah konfigurasi, lalu jalankan verifikasi di bawah.

---

## Verifikasi

### 1. Status session
```
show bgp summary
```
Kolom `State/PfxRcd` harus berupa **angka** (bukan `Active` atau `Idle`).

### 2. Prefix yang diiklankan ke IDREN
```
show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes
```
Harus muncul `100.72.1.0/24` dan `100.72.2.0/24`.

### 3. BGP table
```
show bgp ipv4 unicast
```
Harus ada `100.72.1.0/24` dan `100.72.2.0/24` (prefix UMM, local) dan `100.68.0.0/16` (dari IDREN).

### 4. Routing table OS
```
show ip route bgp
```
`100.68.0.0/16` harus masuk routing table.

### 5. Verifikasi dari sisi IDREN PoP
```bash
docker exec -it clab-lab01-ebgp-idren-pop vtysh -c "show bgp summary"
docker exec -it clab-lab01-ebgp-idren-pop vtysh -c "show bgp ipv4 unicast"
```
Prefix `100.72.1.0/24` dan `100.72.2.0/24` dari UMM harus terlihat di BGP table `idren-pop`.

---

## Expected Output

**`show bgp summary` di umm-border:**
```
Neighbor     V    AS  MsgRcvd  MsgSent  Up/Down  State/PfxRcd  PfxSnt
100.64.0.1   4  65000        6        6 00:01:23             1       2
```

**`show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes`:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 100.72.1.0/24   0.0.0.0                  0         32768 i
*> 100.72.2.0/24   0.0.0.0                  0         32768 i

Total number of prefixes 2
```

**`show bgp ipv4 unicast`:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 100.68.0.0/16   100.64.0.1               0             0 65000 i
*> 100.72.1.0/24   0.0.0.0                  0         32768 i
*> 100.72.2.0/24   0.0.0.0                  0         32768 i
```

---

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| Session stuck `Active` | `idren-pop` belum selesai startup | Tunggu 30 detik, cek lagi |
| Session stuck `Active` | `remote-as` salah | Harus `65000` di konfigurasi `umm-border` |
| Session stuck `Active` | TCP 179 diblokir | `iptables -I INPUT -p tcp --dport 179 -j ACCEPT` |
| `State/PfxRcd: (Policy)` | `ebgp-requires-policy` aktif | Tambahkan `no bgp ebgp-requires-policy` di dalam `router bgp 65010` |
| `PfxRcd: 0` dari IDREN | BGP belum re-advertise | `clear ip bgp * soft` di `umm-border` |
| `PfxSnt: 0` — prefix UMM tidak terkirim | Null route `100.72.1.0/24` atau `100.72.2.0/24` tidak ada di routing table | Cek: `show ip route 100.72.1.0/24` — harus ada null/blackhole route |

---

## Cleanup (setelah lab selesai dan diverifikasi)

```bash
containerlab destroy -t lab01.clab.yml
```

---

*LAB-01 lanjutan dilanjutkan di awal Hari 2 — jika belum selesai hari ini tidak apa-apa.*
