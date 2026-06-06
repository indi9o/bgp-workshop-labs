# Handout LAB-02 — iBGP: 2 Border Router

Workshop BGP — Hari 2 | 6 Juni 2026

---

## Topologi

```
          ┌────────────────────────────────────────────┐
          │  UMM (AS 65010)                            │
          │  [border-1]  ←──iBGP──→  [border-2]       │
          │  lo: 100.64.1.1/32       lo: 100.64.1.2/32 │
          │  eth1: 100.64.0.2/30     (ISP-2 & IIX      │
          │  eth2: 100.64.0.6/30      dikonfigurasi     │
          │  eth3: 100.64.0.17/30     di LAB-03)        │
          └──────────┬────────────────────────────────── ┘
                     │ eBGP (border-1 eth1 — lab ini)
              [exabgp-idren]  ← ExaBGP pre-configured trainer
              AS 65000 | 100.64.0.1/30
              Advertise: ~20 prefix akademik Indonesia
```

**Tugas lab ini:** Konfigurasi eBGP border-1↔IDREN + iBGP border-1↔border-2.
exabgp-isp1 (border-1 eth2), exabgp-isp2 (border-2 eth1), dan exabgp-iix (border-2 eth2) ada di topologi tapi dikonfigurasi mulai LAB-03.

---

## Informasi Peering

| Parameter | Nilai |
|---|---|
| ASN UMM | `65010` |
| border-1 loopback | `100.64.1.1/32` |
| border-2 loopback | `100.64.1.2/32` |
| border-1 eth1 (ke IDREN) | `100.64.0.2/30` |
| border-1 eth3 (ke border-2) | `100.64.0.17/30` |
| border-2 eth3 (ke border-1) | `100.64.0.18/30` |
| ASN IDREN | `65000` |
| IP IDREN PoP | `100.64.0.1` |

---

## Step 1 — Deploy topologi

```bash
mkdir ~/lab02 && cd ~/lab02
# Salin file dari trainer: lab02.clab.yml, border-1/, border-2/,
#   exabgp-idren/, exabgp-isp1/, exabgp-isp2/, exabgp-iix/
docker pull pierky/exabgp   # jika belum ada
containerlab deploy -t lab02.clab.yml
```

Tunggu 15–20 detik. Verifikasi konektivitas dari border-1:

```bash
docker exec -it clab-lab02-ibgp-border-1 bash
ping 100.64.0.1 -c 2    # ke exabgp-idren
ping 100.64.1.2 -c 2    # ke loopback border-2
```

Keduanya harus berhasil sebelum lanjut.

---

## Step 2 — Konfigurasi eBGP border-1 ↔ IDREN

```bash
docker exec -it clab-lab02-ibgp-border-1 bash
vtysh
```

```
configure terminal

ip route 100.72.1.0/24 null0
ip route 100.72.2.0/24 null0

router bgp 65010
 no bgp ebgp-requires-policy
 bgp router-id 100.64.1.1
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

Verifikasi: `show bgp summary` → `State/PfxRcd` harus angka.

---

## Step 3 — Konfigurasi iBGP border-1 ↔ border-2

Tambahkan ke border-1 (masih di vtysh):

```
configure terminal

router bgp 65010
 neighbor 100.64.1.2 remote-as 65010
 neighbor 100.64.1.2 description "iBGP ke border-2"
 neighbor 100.64.1.2 update-source lo
 !
 address-family ipv4 unicast
  neighbor 100.64.1.2 activate
  neighbor 100.64.1.2 next-hop-self
 exit-address-family

end
write memory
```

Konfigurasi border-2:

```bash
docker exec -it clab-lab02-ibgp-border-2 bash
vtysh
```

```
configure terminal

ip route 100.72.1.0/24 null0
ip route 100.72.2.0/24 null0

router bgp 65010
 no bgp ebgp-requires-policy
 bgp router-id 100.64.1.2
 neighbor 100.64.1.1 remote-as 65010
 neighbor 100.64.1.1 description "iBGP ke border-1"
 neighbor 100.64.1.1 update-source lo
 !
 address-family ipv4 unicast
  network 100.72.1.0/24
  network 100.72.2.0/24
  neighbor 100.64.1.1 activate
  neighbor 100.64.1.1 next-hop-self
 exit-address-family

end
write memory
```

Verifikasi route muncul di border-2: `show bgp ipv4 unicast`

---

## Verifikasi

### 1. iBGP session Established — dari border-2
```
show bgp summary
```
Session ke border-1 (iBGP) dengan `State/PfxRcd` = angka.

### 2. Route IDREN muncul di border-2
```
show bgp ipv4 unicast
```
Prefix akademik seperti `152.118.0.0/16` harus ada dengan tanda `i` (iBGP), next-hop `100.64.1.1`.

### 3. Routing table border-2
```
show ip route bgp
```
`152.118.0.0/16` (dan prefix IDREN lain) harus ada dengan tanda `B i`.

---

## Expected Output

**`show bgp summary` di border-2:**
```
Neighbor        V         AS   MsgRcvd   MsgSent   Up/Down State/PfxRcd
100.64.1.1      4      65010       30        10  00:02:10           22
```
(~20 prefix IDREN akademik + 2 prefix UMM — angka bisa sedikit berbeda)

**`show bgp ipv4 unicast` di border-2 (sebagian):**
```
   Network             Next Hop       Metric LocPrf Weight Path
*>i152.118.0.0/16    100.64.1.1                    100      0 7642 i
*>i167.205.0.0/16    100.64.1.1                    100      0 7654 i
*>i103.13.180.0/22   100.64.1.1                    100      0 24855 i
*>i100.72.1.0/24     100.64.1.1               0    100      0 i
*>i100.72.2.0/24     100.64.1.1               0    100      0 i
```
- `i` sebelum network = route dari iBGP
- AS-path: ASN universitas asli (7642=UI, 7654=ITB, 24855=UGM, dll) — ExaBGP static route tidak auto-prepend ASN IDREN (65000), ini normal untuk simulasi
- Next-hop `100.64.1.1` = loopback border-1 (bukan IP exabgp-idren `100.64.0.1`)

**`show ip route bgp` di border-2:**
```
B>  152.118.0.0/16 [200/0] via 100.64.1.1 (recursive), weight 1
  *                          via 100.64.0.17, eth3
B>  100.72.1.0/24 [200/0] via 100.64.1.1 (recursive), weight 1
  *                          via 100.64.0.17, eth3
```

---

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| iBGP session `Active` | Loopback tidak reachable | `ping 100.64.1.2 source 100.64.1.1` — jika gagal, cek static route di topologi |
| Session `Active` meski ping loopback OK | `update-source lo` tidak dikonfigurasi | Tambahkan di kedua border, `clear ip bgp 100.64.1.2` |
| Route tidak muncul di border-2 | `next-hop-self` belum aktif di border-1 | Tambahkan, `clear ip bgp 100.64.1.2 soft` |
| Next-hop masih `100.64.0.1` di border-2 | Lupa `next-hop-self` di border-1 | Tambahkan, `clear ip bgp 100.64.1.2 soft` |

---

## Cleanup (setelah lab selesai dan diverifikasi)

```bash
containerlab destroy -t lab02.clab.yml
```

---

*LAB-03 (BGP Policy) dilanjutkan setelah ISHOMA.*
