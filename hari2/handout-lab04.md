# Handout LAB-04 — Multi-homing: Traffic Engineering

Workshop BGP — Hari 2 | 6 Juni 2026

---

## Topologi

```
          ┌─────────────────────────────────────────────────────┐
          │  UMM (AS 65010)                                     │
          │  [border-1]  ←──iBGP──→  [border-2]                │
          │  lo: 100.64.1.1/32       lo: 100.64.1.2/32          │
          │  eth1→IDREN              eth1→ISP-2                  │
          │  eth2→ISP-1              eth2→IIX                    │
          └──────┬──────────┬──────────────────┬──────┬─────────┘
          [exabgp-idren] [exabgp-isp1]  [exabgp-isp2] [exabgp-iix]
          AS 65000     AS 65100         AS 65101        AS 65200
          .1/30        .5/30            .9/30           .13/30
                 (semua pre-configured ExaBGP — trainer)
```

**Target traffic engineering:**

| Upstream | Border | Local Preference | Outbound AS-path | Catatan |
|---|---|---|---|---|
| IDREN | border-1 | 300 (+ 350 univ direct) | no prepend | Step 3+6 |
| IIX | border-2 | 200 | 1× prepend | Step 3+4 |
| ISP-1 | border-1 | 150 | 1× prepend | Step 3+4 |
| ISP-2 | border-2 | 100 (backup) | 2× prepend | Step 3+4 |

**Overlap prefix demo:** ISP-1 dan ISP-2 keduanya advertise `100.121.0.0/16`. Setelah local-pref, UMM prefer ISP-1 (LocPrf 150 > 100).

---

## Informasi Peering

| Upstream | Border | IP Peer | ASN |
|---|---|---|---|
| IDREN | border-1 eth1 | `100.64.0.1` | `65000` |
| ISP-1 | border-1 eth2 | `100.64.0.5` | `65100` |
| ISP-2 | border-2 eth1 | `100.64.0.9` | `65101` |
| IIX | border-2 eth2 | `100.64.0.13` | `65200` |

---

## Step 1 — Deploy + konfigurasi eBGP & iBGP dasar

```bash
mkdir ~/lab04 && cd ~/lab04
# Salin file dari trainer: lab04.clab.yml, border-1/, border-2/,
#   exabgp-idren/, exabgp-isp1/, exabgp-isp2/, exabgp-iix/
docker pull pierky/exabgp   # jika belum ada
containerlab deploy -t lab04.clab.yml
```

**border-1:**

```bash
docker exec -it clab-lab04-multihoming-border-1 bash
vtysh
```

```
configure terminal

ip route 100.72.1.0/24 null0
ip route 100.72.2.0/24 null0

ip prefix-list OWN-PREFIX seq 10 permit 100.72.1.0/24
ip prefix-list OWN-PREFIX seq 20 permit 100.72.2.0/24
ip prefix-list OWN-PREFIX seq 99 deny any

route-map EXPORT-UPSTREAM permit 10
 match ip address prefix-list OWN-PREFIX

router bgp 65010
 no bgp ebgp-requires-policy
 bgp router-id 100.64.1.1
 neighbor 100.64.0.1 remote-as 65000
 neighbor 100.64.0.1 description "IDREN PoP"
 neighbor 100.64.0.5 remote-as 65100
 neighbor 100.64.0.5 description "ISP-1"
 neighbor 100.64.1.2 remote-as 65010
 neighbor 100.64.1.2 description "iBGP ke border-2"
 neighbor 100.64.1.2 update-source lo
 !
 address-family ipv4 unicast
  network 100.72.1.0/24
  network 100.72.2.0/24
  neighbor 100.64.0.1 activate
  neighbor 100.64.0.1 soft-reconfiguration inbound
  neighbor 100.64.0.1 route-map EXPORT-UPSTREAM out
  neighbor 100.64.0.5 activate
  neighbor 100.64.0.5 soft-reconfiguration inbound
  neighbor 100.64.0.5 route-map EXPORT-UPSTREAM out
  neighbor 100.64.1.2 activate
  neighbor 100.64.1.2 next-hop-self
 exit-address-family

end
write memory
```

**border-2:**

```bash
docker exec -it clab-lab04-multihoming-border-2 bash
vtysh
```

```
configure terminal

ip route 100.72.1.0/24 null0
ip route 100.72.2.0/24 null0

ip prefix-list OWN-PREFIX seq 10 permit 100.72.1.0/24
ip prefix-list OWN-PREFIX seq 20 permit 100.72.2.0/24
ip prefix-list OWN-PREFIX seq 99 deny any

route-map EXPORT-UPSTREAM permit 10
 match ip address prefix-list OWN-PREFIX

router bgp 65010
 no bgp ebgp-requires-policy
 bgp router-id 100.64.1.2
 neighbor 100.64.0.9 remote-as 65101
 neighbor 100.64.0.9 description "ISP-2"
 neighbor 100.64.0.13 remote-as 65200
 neighbor 100.64.0.13 description "IIX"
 neighbor 100.64.1.1 remote-as 65010
 neighbor 100.64.1.1 description "iBGP ke border-1"
 neighbor 100.64.1.1 update-source lo
 !
 address-family ipv4 unicast
  network 100.72.1.0/24
  network 100.72.2.0/24
  neighbor 100.64.0.9 activate
  neighbor 100.64.0.9 soft-reconfiguration inbound
  neighbor 100.64.0.9 route-map EXPORT-UPSTREAM out
  neighbor 100.64.0.13 activate
  neighbor 100.64.0.13 soft-reconfiguration inbound
  neighbor 100.64.0.13 route-map EXPORT-UPSTREAM out
  neighbor 100.64.1.1 activate
  neighbor 100.64.1.1 next-hop-self
 exit-address-family

end
write memory
```

Verifikasi: `show bgp summary` di masing-masing border — harus 3 session Established.

---

## Step 2 — Amati kondisi sebelum Traffic Engineering

Di border-1:
```
show bgp ipv4 unicast 100.121.0.0/16
```
Dua path tersedia: ISP-1 (eBGP langsung, LocPrf 100) dan ISP-2 via iBGP dari border-2 (LocPrf 100). Catat path mana yang terpilih (`>`).

Di border-1 juga cek berapa prefix yang diterima:
```
show bgp summary
```
Expect: IDREN ~20 prefix akademik, ISP-1 ~36 prefix komersial, border-2 iBGP ~25+ prefix.

---

## Step 3 — Konfigurasi Local Preference

**border-1** (IDREN=300, ISP-1=150):

```
configure terminal

route-map IMPORT-FROM-IDREN permit 10
 set local-preference 300

route-map IMPORT-FROM-ISP1 permit 10
 set local-preference 150

router bgp 65010
 address-family ipv4 unicast
  neighbor 100.64.0.1 route-map IMPORT-FROM-IDREN in
  neighbor 100.64.0.5 route-map IMPORT-FROM-ISP1 in
 exit-address-family

end
write memory

clear ip bgp 100.64.0.1 soft in
clear ip bgp 100.64.0.5 soft in
```

**border-2** (ISP-2=100, IIX=200):

```
configure terminal

route-map IMPORT-FROM-ISP2 permit 10
 set local-preference 100

route-map IMPORT-FROM-IIX permit 10
 set local-preference 200

router bgp 65010
 address-family ipv4 unicast
  neighbor 100.64.0.9 route-map IMPORT-FROM-ISP2 in
  neighbor 100.64.0.13 route-map IMPORT-FROM-IIX in
 exit-address-family

end
write memory

clear ip bgp 100.64.0.9 soft in
clear ip bgp 100.64.0.13 soft in
```

Verifikasi di border-1:
```
show bgp ipv4 unicast 100.121.0.0/16
```
Path ISP-1 (`LocPrf 150`) harus terpilih (`>`).

**Expected output:**
```
Paths: (1 available, best #1, table default)
  65100
      Origin IGP, localpref 150, valid, external, best (Local Pref)
```

Cek juga IDREN prefix di border-2 — harus localpref 300 via iBGP:
```
show bgp ipv4 unicast 152.118.0.0/16
```
```
Paths: (2 available, best #1, table default)
  7642
      Origin IGP, localpref 300, valid, internal, best (Local Pref)
  4761 7713 23700 7642
      Origin IGP, localpref 100, valid, external
```

---

## Step 4 — Konfigurasi AS-path Prepending

**border-1** (IDREN: no prepend, ISP-1: 1× prepend):

```
configure terminal

route-map EXPORT-TO-IDREN permit 10
 match ip address prefix-list OWN-PREFIX

route-map EXPORT-TO-ISP1 permit 10
 match ip address prefix-list OWN-PREFIX
 set as-path prepend 65010

router bgp 65010
 address-family ipv4 unicast
  neighbor 100.64.0.1 route-map EXPORT-TO-IDREN out
  neighbor 100.64.0.5 route-map EXPORT-TO-ISP1 out
 exit-address-family

end
write memory

clear ip bgp 100.64.0.1 soft out
clear ip bgp 100.64.0.5 soft out
```

**border-2** (IIX: 1× prepend, ISP-2: 2× prepend):

```
configure terminal

route-map EXPORT-TO-IIX permit 10
 match ip address prefix-list OWN-PREFIX
 set as-path prepend 65010

route-map EXPORT-TO-ISP2 permit 10
 match ip address prefix-list OWN-PREFIX
 set as-path prepend 65010 65010

router bgp 65010
 address-family ipv4 unicast
  neighbor 100.64.0.13 route-map EXPORT-TO-IIX out
  neighbor 100.64.0.9 route-map EXPORT-TO-ISP2 out
 exit-address-family

end
write memory

clear ip bgp 100.64.0.13 soft out
clear ip bgp 100.64.0.9 soft out
```

Verifikasi:
```
# Di border-1 — ke IDREN (no prepend)
show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes
```
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 100.72.1.0/24    0.0.0.0                  0         32768 i
*> 100.72.2.0/24    0.0.0.0                  0         32768 i
```
(Path = `65010` saja — local AS ditambah saat kirim)

```
# Di border-1 — ke ISP-1 (1× prepend)
show bgp ipv4 unicast neighbors 100.64.0.5 advertised-routes
```
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 100.72.1.0/24    0.0.0.0                  0         32768 65010 i
*> 100.72.2.0/24    0.0.0.0                  0         32768 65010 i
```
(Path = `65010 65010` — route-map prepend + local AS)

```
# Di border-2 — ke ISP-2 (2× prepend)
show bgp ipv4 unicast neighbors 100.64.0.9 advertised-routes
```
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 100.72.1.0/24    0.0.0.0                  0         32768 65010 65010 i
*> 100.72.2.0/24    0.0.0.0                  0         32768 65010 65010 i
```
(Path = `65010 65010 65010` — 2× prepend + local AS)

---

## Step 5 — Simulasi Failover

Shutdown ISP-1 di border-1:
```
configure terminal
router bgp 65010
 neighbor 100.64.0.5 shutdown
end
```

Tunggu ~10 detik (shutdown kirim BGP NOTIFICATION — konvergen cepat), lalu:
```
show bgp summary
show bgp ipv4 unicast 100.121.0.0/16
```
`100.121.0.0/16` harus beralih ke ISP-2 path (via border-2 iBGP, LocPrf 100):
```
Paths: (1 available, best #1, table default)
  65101
      Origin IGP, localpref 100, valid, internal, best (First path received)
```

Aktifkan kembali:
```
configure terminal
router bgp 65010
 no neighbor 100.64.0.5 shutdown
end
```

---

## Step 6 — AS-path filtering: prioritas route universitas langsung

Ubah `IMPORT-FROM-IDREN` di border-1 — prefix universitas Indonesia yang dioriginate langsung (AS-path satu hop) mendapat LocPrf 350:

```
configure terminal

bgp as-path access-list UNIV-ID-DIRECT permit ^(7642|7654|24855|45166|45974|45958|45997)$

no route-map IMPORT-FROM-IDREN permit 10

route-map IMPORT-FROM-IDREN permit 10
 match as-path UNIV-ID-DIRECT
 set local-preference 350

route-map IMPORT-FROM-IDREN permit 20
 set local-preference 300

end
write memory

clear ip bgp 100.64.0.1 soft in
```

Verifikasi:
```
show bgp ipv4 unicast 152.118.0.0/16
```
**Expected:** path IDREN (AS-path `7642`) → `localpref 350, best`; path ISP-1 (AS-path `7713 23700 7642`) → `localpref 150, not best`.

```
show bgp ipv4 unicast regexp ^7642$
```
Semua prefix dengan AS-path `7642` (hanya UI) harus tampil dengan LocPrf 350.

**Konfigurasi best practice — filter private ASN (di border-1 dan border-2):**

```
! border-1
bgp as-path access-list NO-PRIVATE-AS deny _(6451[2-9]|645[2-9][0-9]|64[6-9][0-9][0-9]|6[5-9][0-9][0-9][0-9])_
bgp as-path access-list NO-PRIVATE-AS permit .*

no route-map IMPORT-FROM-ISP1 permit 10
route-map IMPORT-FROM-ISP1 permit 10
 match as-path NO-PRIVATE-AS
 set local-preference 150

end ; write memory ; clear ip bgp 100.64.0.5 soft in
```

```
! border-2
bgp as-path access-list NO-PRIVATE-AS deny _(6451[2-9]|645[2-9][0-9]|64[6-9][0-9][0-9]|6[5-9][0-9][0-9][0-9])_
bgp as-path access-list NO-PRIVATE-AS permit .*

no route-map IMPORT-FROM-ISP2 permit 10
route-map IMPORT-FROM-ISP2 permit 10
 match as-path NO-PRIVATE-AS
 set local-preference 100

end ; write memory ; clear ip bgp 100.64.0.9 soft in
```

> ExaBGP lab tidak inject private ASN — semua ISP route tetap lolos. Ini konfigurasi best practice untuk produksi.

---

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| Hanya 1 path `100.121.0.0/16` | iBGP belum propagate | Cek `show bgp summary` di kedua border |
| Local-pref tidak berubah | Lupa `clear soft in` | `clear ip bgp X soft in` per neighbor |
| AS-path tidak berubah | Lupa `clear soft out` | `clear ip bgp X soft out` per neighbor |
| Failover tidak terjadi | Konvergen butuh waktu | Tunggu ~10 detik setelah `neighbor shutdown` |
| Session tidak kembali | Reconnect perlu waktu | Tunggu 30–60 detik |
| `152.118.0.0/16` tetap LocPrf 300 | seq 10 lama belum dihapus | `no route-map IMPORT-FROM-IDREN permit 10` lalu konfigurasi ulang |
| Semua IDREN dapat 350 | seq 20 tidak ada / regex terlalu lebar | Cek `show route-map IMPORT-FROM-IDREN` |

---

## Cleanup (setelah lab selesai dan diverifikasi)

```bash
containerlab destroy -t lab04.clab.yml
```

---

*LAB-05 (Troubleshooting) dilanjutkan di Hari 3.*
