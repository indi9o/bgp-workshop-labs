# Handout LAB-03 — Prefix Filtering

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
          [frr-idren]  [frr-isp1]   [frr-isp2]  [frr-iix]
          AS 65000     AS 65100     AS 65101    AS 65200
          .1/30        .5/30        .9/30       .13/30
               │iBGP        │iBGP       │iBGP       │iBGP
          [exabgp-idren] [exabgp-isp1] [exabgp-isp2] [exabgp-iix]
                 (semua pre-configured trainer — peserta tidak perlu sentuh)
```

**Prefix yang diinject upstream (ExaBGP internet simulation):**

| Node | Contoh Prefix | Jumlah | Catatan |
|---|---|---|---|
| exabgp-idren | `152.118.0.0/16`, `167.205.0.0/16`, `103.13.180.0/22` | ~20 | Prefix akademik Indonesia (UI, ITB, UGM, ITS, dll) |
| exabgp-isp1 | `36.66.0.0/16`, `8.8.8.0/24`, `1.1.1.0/24`, `100.121.0.0/16` | ~36 | Prefix komersial + CDN global |
| exabgp-isp1 | `203.0.113.0/24` | 1 | **Bogon RFC 5737 TEST-NET-3 — harus difilter inbound di border-1** |
| exabgp-isp2 | `100.121.0.0/16`, `100.122.0.0/16`, `1.1.1.0/24` | ~25 | Prefix komersial via ISP-2 |
| exabgp-iix | `157.240.0.0/16`, `103.53.12.0/22`, `8.8.8.0/24` | ~25 | Konten lokal Indonesia |

---

## Informasi Peering

| Upstream | Border | IP Peer | ASN |
|---|---|---|---|
| IDREN | border-1 eth1 | `100.64.0.1` | `65000` |
| ISP-1 | border-1 eth2 | `100.64.0.5` | `65100` |
| ISP-2 | border-2 eth1 | `100.64.0.9` | `65101` |
| IIX | border-2 eth2 | `100.64.0.13` | `65200` |

---

## Step 1 — Deploy topologi

```bash
mkdir ~/lab03 && cd ~/lab03
# Salin file dari trainer: lab03.clab.yml, border-1/, border-2/,
#   frr-idren/, frr-isp1/, frr-isp2/, frr-iix/,
#   exabgp-idren/, exabgp-isp1/, exabgp-isp2/, exabgp-iix/
docker pull pierky/exabgp   # jika belum ada
containerlab deploy -t lab03.clab.yml
```

Tunggu 15–20 detik.

---

## Step 2 — Konfigurasi eBGP + iBGP dasar

**border-1** (IDREN + ISP-1 + iBGP ke border-2):

```bash
docker exec -it clab-lab03-policy-border-1 bash
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
  neighbor 100.64.0.5 activate
  neighbor 100.64.0.5 soft-reconfiguration inbound
  neighbor 100.64.1.2 activate
  neighbor 100.64.1.2 next-hop-self
 exit-address-family

end
write memory
```

**border-2** (ISP-2 + IIX + iBGP ke border-1):

```bash
docker exec -it clab-lab03-policy-border-2 bash
vtysh
```

```
configure terminal

ip route 100.72.1.0/24 null0
ip route 100.72.2.0/24 null0

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
  neighbor 100.64.0.13 activate
  neighbor 100.64.0.13 soft-reconfiguration inbound
  neighbor 100.64.1.1 activate
  neighbor 100.64.1.1 next-hop-self
 exit-address-family

end
write memory
```

Verifikasi — masing-masing border harus punya 3 session Established:
```
show bgp summary
```

---

## Step 3 — Amati kondisi sebelum policy

Di border-1:
```
show bgp ipv4 unicast
show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes
```

Perhatikan: `203.0.113.0/24` dari ISP-1 masuk BGP table, dan border-1 masih iklankan semua prefix ke IDREN.

---

## Step 4 — Outbound filter: hanya prefix UMM

**border-1:**

```
configure terminal

ip prefix-list OWN-PREFIX seq 10 permit 100.72.1.0/24
ip prefix-list OWN-PREFIX seq 20 permit 100.72.2.0/24
ip prefix-list OWN-PREFIX seq 99 deny any

route-map EXPORT-UPSTREAM permit 10
 match ip address prefix-list OWN-PREFIX

router bgp 65010
 address-family ipv4 unicast
  neighbor 100.64.0.1 route-map EXPORT-UPSTREAM out
  neighbor 100.64.0.5 route-map EXPORT-UPSTREAM out
 exit-address-family

end
write memory

clear ip bgp 100.64.0.1 soft out
clear ip bgp 100.64.0.5 soft out
```

**border-2** (konfigurasi sama, terapkan ke ISP-2 dan IIX):

```
configure terminal

ip prefix-list OWN-PREFIX seq 10 permit 100.72.1.0/24
ip prefix-list OWN-PREFIX seq 20 permit 100.72.2.0/24
ip prefix-list OWN-PREFIX seq 99 deny any

route-map EXPORT-UPSTREAM permit 10
 match ip address prefix-list OWN-PREFIX

router bgp 65010
 address-family ipv4 unicast
  neighbor 100.64.0.9 route-map EXPORT-UPSTREAM out
  neighbor 100.64.0.13 route-map EXPORT-UPSTREAM out
 exit-address-family

end
write memory

clear ip bgp 100.64.0.9 soft out
clear ip bgp 100.64.0.13 soft out
```

---

## Step 5 — Inbound filter: buang bogon

Terapkan di border-1 DAN border-2 (prefix-list BOGON identik):

**border-1:**

```
configure terminal

ip prefix-list BOGON seq 10 deny 10.0.0.0/8 le 32
ip prefix-list BOGON seq 20 deny 172.16.0.0/12 le 32
ip prefix-list BOGON seq 30 deny 192.168.0.0/16 le 32
ip prefix-list BOGON seq 40 deny 127.0.0.0/8 le 32
ip prefix-list BOGON seq 50 deny 169.254.0.0/16 le 32
ip prefix-list BOGON seq 60 deny 192.0.2.0/24
ip prefix-list BOGON seq 70 deny 198.51.100.0/24
ip prefix-list BOGON seq 80 deny 203.0.113.0/24
ip prefix-list BOGON seq 99 permit any

route-map IMPORT-COMMON permit 10
 match ip address prefix-list BOGON

router bgp 65010
 address-family ipv4 unicast
  neighbor 100.64.0.1 route-map IMPORT-COMMON in
  neighbor 100.64.0.5 route-map IMPORT-COMMON in
 exit-address-family

end
write memory

clear ip bgp 100.64.0.5 soft in
```

**border-2:**

```
configure terminal

ip prefix-list BOGON seq 10 deny 10.0.0.0/8 le 32
ip prefix-list BOGON seq 20 deny 172.16.0.0/12 le 32
ip prefix-list BOGON seq 30 deny 192.168.0.0/16 le 32
ip prefix-list BOGON seq 40 deny 127.0.0.0/8 le 32
ip prefix-list BOGON seq 50 deny 169.254.0.0/16 le 32
ip prefix-list BOGON seq 60 deny 192.0.2.0/24
ip prefix-list BOGON seq 70 deny 198.51.100.0/24
ip prefix-list BOGON seq 80 deny 203.0.113.0/24
ip prefix-list BOGON seq 99 permit any

route-map IMPORT-COMMON permit 10
 match ip address prefix-list BOGON

router bgp 65010
 address-family ipv4 unicast
  neighbor 100.64.0.9 route-map IMPORT-COMMON in
  neighbor 100.64.0.13 route-map IMPORT-COMMON in
 exit-address-family

end
write memory

clear ip bgp 100.64.0.9 soft in
clear ip bgp 100.64.0.13 soft in
```

---

## Verifikasi

### Outbound — hanya prefix UMM

Di border-1:
```
show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes
```

Di border-2:
```
show bgp ipv4 unicast neighbors 100.64.0.9 advertised-routes
```

Keduanya hanya boleh tampil `100.72.1.0/24` dan `100.72.2.0/24`.

### Inbound — bogon terfilter

Di border-1:
```
show bgp ipv4 unicast neighbors 100.64.0.5 received-routes
```
`203.0.113.0/24` ada di `received-routes` (datang dari ISP-1).

```
show bgp ipv4 unicast
```
`203.0.113.0/24` **tidak ada** (dibuang inbound policy).

---

## Expected Output

**`show bgp ipv4 unicast neighbors 100.64.0.1 advertised-routes` di border-1:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 100.72.1.0/24   0.0.0.0                  0         32768 i
*> 100.72.2.0/24   0.0.0.0                  0         32768 i

Total number of prefixes 2
```

---

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| `advertised-routes` masih semua prefix | Lupa `clear soft out` | `clear ip bgp X soft out` ke tiap neighbor |
| `received-routes` kosong | `soft-reconfiguration inbound` belum aktif | Tambahkan di konfigurasi, lalu `clear soft in` |
| Bogon masih ada di BGP table | `clear soft in` belum dijalankan | `clear ip bgp 100.64.0.5 soft in` di border-1 |
| Semua prefix dibuang inbound | Route-map implicit deny | Pastikan prefix-list BOGON ada `seq 99 permit any` |
| border-2 tidak lihat IDREN/ISP-1 | iBGP session belum Established | `show bgp summary` di border-2 |

---

## Cleanup (setelah lab selesai dan diverifikasi)

```bash
containerlab destroy -t lab03.clab.yml
```

---

*LAB-04 (Multi-homing TE) dilanjutkan setelah Coffee Break.*
