## Jarkom-Modul-3-2026-K-53
Sudah saatnya...

**Curtain Protocol: Shibuya Incident**

### Konvensi yang dipakai

| Item | Nilai |
|---|---|
| PREFIX IP | `10.90` |
| `<xxxx>` (nama kelompok) | `K53` |
| Image Docker | `ardhptr21/debinet:latest` (jichi, tengen, utahime, shoko, gojo, geto, worker, database), `ardhptr21/alpinet:latest` (semua client) |
| Lokasi script | `/root` |
| Resolver awal | `nameserver 192.168.122.1` |

---

### Soal 1 : Router Utama (jichi), IP Statis, IPv4 Forwarding, dan NAT

Pada soal ini jichi disiapkan sebagai router utama yang menghubungkan empat network internal sekaligus menjadi gerbang ke internet lewat NAT. Setelah itu seluruh host infrastruktur diberi IP statis dan default gateway ke jichi.

#### Topologi

![](assets/Topologi.png)

Pemetaan interface **jichi** terhadap switch:

| Interface | Terhubung ke | Node | Network | Subnet |
|---|---|---|---|---|
| eth0 | NAT1 | - | WAN | 192.168.122.0/24 |
| eth1 | Switch1 | utahime, shoko, gojo | Network 1 (DMZ) | 10.90.1.0/24 |
| eth2 | Switch2 | itadori, fushiguro, zenin, okkotsu | Network 2 (Divisi A) | 10.90.2.0/24 |
| eth3 | Switch4 | ino, kusakabe, kugisaki, miwa | Network 3 (Divisi B) | 10.90.3.0/24 |
| eth4 | Switch3 | geto, maki, panda, nanami, todo, kirara, tengen, yaga, inumaki | Network 4 (Data Center) | 10.90.4.0/24 |

> Nama switch di GNS3 tidak sama dengan nomor network (Switch4 = Network 3, Switch3 = Network 4). Yang dijadikan patokan adalah isi switch dan nomor interface jichi: eth3 harus ke Network 3 dan eth4 ke Network 4.

#### Alamat host infrastruktur (IP statis)

| Node | Network | IP | Gateway |
|---|---|---|---|
| jichi | 1 - 4 | eth1 `10.90.1.1`, eth2 `10.90.2.1`, eth3 `10.90.3.1`, eth4 `10.90.4.1` | - |
| utahime | 1 | 10.90.1.2/24 | 10.90.1.1 |
| shoko | 1 | 10.90.1.3/24 | 10.90.1.1 |
| gojo | 1 | 10.90.1.4/24 | 10.90.1.1 |
| tengen | 4 | 10.90.4.10/24 | 10.90.4.1 |
| geto | 4 | 10.90.4.20/24 | 10.90.4.1 |
| maki | 4 | 10.90.4.31/24 | 10.90.4.1 |
| panda | 4 | 10.90.4.32/24 | 10.90.4.1 |
| nanami | 4 | 10.90.4.41/24 | 10.90.4.1 |
| todo | 4 | 10.90.4.42/24 | 10.90.4.1 |
| kirara | 4 | 10.90.4.43/24 | 10.90.4.1 |
| yaga | 4 | 10.90.4.51/24 | 10.90.4.1 |
| inumaki | 4 | 10.90.4.52/24 | 10.90.4.1 |

Client Network 2 dan 3 (itadori, fushiguro, zenin, okkotsu, ino, kusakabe, kugisaki, miwa) tidak diberi IP statis karena akan memperoleh alamat dari DHCP pada soal berikutnya.

#### Persiapan

1. Pastikan link jichi sesuai tabel pemetaan di atas.
2. Jalankan semua node dengan tombol **Start** di GNS3.
3. Buka konsol tiap node (telnet ke alamat yang tertera di Topology Summary).

---

#### 1. Router (jichi)

Buka konsol jichi, lalu jalankan:
```bash
cat > /root/jichi.sh << 'EOF'
#!/bin/bash
# --- WAN: eth0 ke NAT GNS3 ---
ip addr flush dev eth0
ip addr add 192.168.122.153/24 dev eth0
ip link set eth0 up
ip route replace default via 192.168.122.1

# --- LAN: eth1 - eth4 menjadi gateway Network 1 - 4 ---
for i in 1 2 3 4; do
  ip addr flush dev eth$i
  ip addr add 10.90.$i.1/24 dev eth$i
  ip link set eth$i up
done

# Resolver sementara supaya nama domain bisa dipakai
echo "nameserver 192.168.122.1" > /etc/resolv.conf

# Aktifkan IPv4 forwarding dan source NAT
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o eth0 -s 10.90.0.0/16 -j MASQUERADE
EOF

bash /root/jichi.sh
```

Penjelasan:

| Perintah | Fungsi |
|---|---|
| `ip addr flush dev ethX` | Membersihkan IP lama supaya script aman dijalankan berulang |
| `ip addr add ... dev ethX` | Memasang IP gateway pada tiap network (`10.90.N.1/24`) |
| `ip link set ethX up` | Mengaktifkan interface |
| `ip route replace default via 192.168.122.1` | Default route jichi menuju gateway NAT GNS3 |
| `sysctl -w net.ipv4.ip_forward=1` | Mengizinkan jichi meneruskan paket antar network |
| `iptables -t nat ... MASQUERADE` | Source NAT: paket dari `10.90.0.0/16` yang keluar lewat eth0 disamarkan dengan IP WAN jichi |

> Awalnya eth0 diminta lewat `dhclient eth0`, tetapi pada lingkungan ini eth0 tidak memperoleh IPv4 dari NAT (hanya alamat IPv6 link-local), sehingga ping ke internet gagal dengan `Network is unreachable`. Karena itu eth0 diberi alamat statis `192.168.122.153/24` dengan gateway `192.168.122.1`. Alamat `.153` dipilih karena subnet NAT dipakai bersama; ganti angka lain bila terjadi bentrok alamat.

Verifikasi di jichi:
```bash
ip -br addr
ip route
sysctl net.ipv4.ip_forward
iptables -t nat -L POSTROUTING -n -v
ping -c 3 192.168.122.1
ping -c 3 8.8.8.8
ping -c 3 google.com
```
Hasil yang diharapkan:
- eth1 `10.90.1.1`, eth2 `10.90.2.1`, eth3 `10.90.3.1`, eth4 `10.90.4.1`, eth0 `192.168.122.153`.
- `default via 192.168.122.1 dev eth0` ada di tabel routing.
- `net.ipv4.ip_forward = 1`.
- Ada aturan `MASQUERADE` pada interface keluar `eth0` dengan source `10.90.0.0/16`.
- Ketiga ping berhasil tanpa packet loss.
![](assets/soal1-jichi.png)


---

#### 2. Network 1 (utahime, shoko, gojo) - Gateway: 10.90.1.1

utahime:
```bash
cat > /root/utahime.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.1.2/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.1.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/utahime.sh
```
shoko:
```bash
cat > /root/shoko.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.1.3/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.1.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/shoko.sh
```
gojo:
```bash
cat > /root/gojo.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.1.4/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.1.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/gojo.sh
```

---

#### 3. Network 4 (tengen, geto, maki, panda, nanami, todo, kirara, yaga, inumaki) - Gateway: 10.90.4.1

tengen:
```bash
cat > /root/tengen.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.10/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/tengen.sh
```
geto:
```bash
cat > /root/geto.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.20/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/geto.sh
```
maki:
```bash
cat > /root/maki.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.31/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/maki.sh
```
panda:
```bash
cat > /root/panda.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.32/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/panda.sh
```
nanami:
```bash
cat > /root/nanami.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.41/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/nanami.sh
```
todo:
```bash
cat > /root/todo.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.42/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/todo.sh
```
kirara:
```bash
cat > /root/kirara.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.43/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/kirara.sh
```
yaga:
```bash
cat > /root/yaga.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.51/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/yaga.sh
```
inumaki:
```bash
cat > /root/inumaki.sh << 'EOF'
ip addr flush dev eth0
ip addr add 10.90.4.52/24 dev eth0
ip link set eth0 up
ip route replace default via 10.90.4.1
echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF

bash /root/inumaki.sh
```

Penjelasan script host (sama untuk semua node, hanya IP dan gateway yang berbeda):

| Perintah | Fungsi |
|---|---|
| `ip addr flush dev eth0` | Membersihkan IP lama |
| `ip addr add <IP>/24 dev eth0` | Memasang IP statis |
| `ip link set eth0 up` | Mengaktifkan interface |
| `ip route replace default via <gateway>` | Default gateway menuju jichi |
| `echo "nameserver 192.168.122.1" > /etc/resolv.conf` | Resolver sementara agar `apt` dan nama domain bisa dipakai |

---

#### Verifikasi Soal 1

**a. Konektivitas ke gateway (semua host infrastruktur)**

Jalankan di setiap host. Gateway-nya `10.90.1.1` untuk utahime, shoko, dan gojo, serta `10.90.4.1` untuk tengen, geto, maki, panda, nanami, todo, kirara, yaga, dan inumaki:
```bash
ip -br addr
ping -c 3 <gateway>
```
**b. Akses eksternal (host yang relevan: jichi, tengen, utahime, shoko - tapi semua host saya juga pakai karena why not?)**

```bash
ping -c 3 8.8.8.8          # jalur NAT lewat jichi
ping -c 3 google.com       # resolver 192.168.122.1
```
##### Bukti :
utahime :
![](assets/soal1-utahime2.png)\
![](assets/soal1-utahime.png)\
shoko :
![](assets/soal1-shoko.png)
gojo :
![](assets/soal1-gojo.png)
tengen :
![](assets/soal1-tengen.png)
geto :
![](assets/soal1-geto.png)
maki :
![](assets/soal1-maki.png)
panda :
![](assets/soal1-panda.png)
nanami :
![](assets/soal1-nanami.png)
todo :
![](assets/soal1-todo.png)
kirara :
![](assets/soal1-kirara.png)
yaga :
![](assets/soal1-yaga.png)
inumaki :
![](assets/soal1-inumaki.png)

**c. Routing antar network lewat jichi**

Dari tengen (Network 4) ke utahime (Network 1):
```bash
ping -c 3 10.90.1.2
```
![](assets/soal1-tengen2.png)

Dari utahime ke tengen:
```bash
ping -c 3 10.90.4.10
```
![](assets/soal1-utahime.png)

TTL yang turun menjadi 63 menandakan paket melewati satu router (jichi).

> Jika node di-stop lalu di-start lagi, konfigurasi IP hilang. Cukup jalankan ulang script di `/root` pada node tersebut (`bash /root/jichi.sh`, `bash /root/tengen.sh`, dan seterusnya).

---

### Soal 2 : DHCP Server (tengen), Subnet Lokal Tanpa Pool, dan Scope Network 2 & 3
 
Pada soal ini tengen (`10.90.4.10`, Network 4 / Data Center) disiapkan sebagai DHCP server untuk seluruh jaringan. Ada empat hal yang dikerjakan:
 
1. Install `isc-dhcp-server` di tengen.
2. Deklarasikan Network 4 sebagai subnet lokal **tanpa pool** (tanpa `range`).
3. Siapkan scope untuk Network 2 dan Network 3.
4. Validasi konfigurasi sampai tidak ada error **"No subnet declaration for eth0"**.
#### Gambaran scope
 
| Network | Subnet | Gateway | Pool | Keterangan |
|---|---|---|---|---|
| Network 4 (Data Center) | 10.90.4.0/24 | 10.90.4.1 | tidak ada | Hanya deklarasi subnet lokal, supaya dhcpd bisa jalan di eth0 |
| Network 2 (Divisi A) | 10.90.2.0/24 | 10.90.2.1 | 10.90.2.10 - 10.90.2.35 | Sementara, kebijakan pool final diganti di Soal 4 |
| Network 3 (Divisi B) | 10.90.3.0/24 | 10.90.3.1 | 10.90.3.10 - 10.90.3.89 | Sementara, kebijakan pool final diganti di Soal 5 |
 
Opsi yang dibagikan ke client Network 2 dan 3: DNS `10.90.1.2` dan `10.90.1.3` (utahime dan shoko), serta domain-search `K53.com`.
 
> Semua langkah dikerjakan di konsol **tengen**.
 
---
 
#### 1. Install dan konfigurasi (tengen)
 
Buka konsol tengen, lalu jalankan:
```bash
cat > /root/tengen-dhcp.sh << 'EOF'
#!/bin/bash
export DEBIAN_FRONTEND=noninteractive
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt update && apt install -y isc-dhcp-server
 
# DHCP hanya dilayani lewat eth0 (Network 4)
sed -i 's/^INTERFACESv4=.*/INTERFACESv4="eth0"/' /etc/default/isc-dhcp-server
 
# File utama
cat > /etc/dhcp/dhcpd.conf << 'EOT'
ddns-update-style none;
authoritative;
default-lease-time 600;
max-lease-time 7200;
 
# Network 4 (Data Center): subnet lokal tanpa pool dinamis
subnet 10.90.4.0 netmask 255.255.255.0 {
}
 
# Network 2 dan 3 dipisah ke file masing-masing
include "/etc/dhcp/net2.conf";
include "/etc/dhcp/net3.conf";
EOT
 
# Scope sementara Network 2 (akan diganti di Soal 4)
cat > /etc/dhcp/net2.conf << 'EOT'
subnet 10.90.2.0 netmask 255.255.255.0 {
  range 10.90.2.10 10.90.2.35;
  option routers 10.90.2.1;
  option broadcast-address 10.90.2.255;
  option domain-name-servers 10.90.1.2, 10.90.1.3;
  option domain-search "K53.com";
}
EOT
 
# Scope sementara Network 3 (akan diganti di Soal 5)
cat > /etc/dhcp/net3.conf << 'EOT'
subnet 10.90.3.0 netmask 255.255.255.0 {
  range 10.90.3.10 10.90.3.89;
  option routers 10.90.3.1;
  option broadcast-address 10.90.3.255;
  option domain-name-servers 10.90.1.2, 10.90.1.3;
  option domain-search "K53.com";
}
EOT
 
touch /var/lib/dhcp/dhcpd.leases
dhcpd -t -cf /etc/dhcp/dhcpd.conf && service isc-dhcp-server restart
EOF
 
bash /root/tengen-dhcp.sh
```
 
Penjelasan:
 
| Bagian | Fungsi |
|---|---|
| `apt install isc-dhcp-server` | Memasang DHCP server (butuh internet, tengen sudah terhubung lewat jichi pada Soal 1) |
| `INTERFACESv4="eth0"` | DHCP hanya mendengarkan di eth0, yang berada di Network 4 |
| `ddns-update-style none` | Menonaktifkan dynamic DNS update (tidak dipakai) |
| `authoritative` | tengen menjadi server resmi untuk subnet yang dideklarasikan |
| `default-lease-time` dan `max-lease-time` | Lama lease default (600 detik) dan maksimum (7200 detik) |
| `subnet 10.90.4.0 ... { }` | Network 4 dideklarasikan **tanpa `range`**. Ini yang menghilangkan error "No subnet declaration for eth0", karena dhcpd mewajibkan ada subnet yang cocok dengan IP interface yang dipakai |
| `net2.conf` dan `net3.conf` | Scope Network 2 dan 3 dipisah ke file masing-masing lewat `include`. Rentang di sini **sementara** supaya Soal 3 (relay) bisa dibuktikan. Kebijakan pool sebenarnya diganti di Soal 4 dan 5 |
| `option routers` dan `broadcast-address` | Gateway dan broadcast tiap network |
| `domain-name-servers` dan `domain-search` | DNS `10.90.1.2` dan `10.90.1.3`, domain-search `K53.com` |
| `touch /var/lib/dhcp/dhcpd.leases` | Memastikan file lease ada, supaya dhcpd tidak gagal start |
| `dhcpd -t -cf ...` | Validasi sintaks sebelum service di-restart |
 
---
 
#### Verifikasi Soal 2
 
Jalankan di tengen:
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
ss -ulnp | grep :67
grep INTERFACES /etc/default/isc-dhcp-server
```
 
Hasil yang diharapkan:
- `dhcpd -t` tidak mengeluarkan error. Pesan **"No subnet declaration for eth0"** tidak muncul.
- Status service **running**.
- `ss` menampilkan `dhcpd` mendengarkan di port 67 (`0.0.0.0:67`).
- `INTERFACESv4="eth0"`.
##### Bukti :\
tengen :\
![](assets/soal2-tengen.png)
 
Hasil di atas sesuai harapan: validasi konfigurasi lolos tanpa error, `dhcpd is running`, dhcpd mendengarkan di port 67, dan `INTERFACESv4="eth0"`.
 
> Jika tengen di-stop lalu di-start lagi, konfigurasi IP hilang dan paket yang terpasang bisa ikut hilang. Jalankan ulang `bash /root/tengen.sh` (Soal 1), lalu `bash /root/tengen-dhcp.sh`.
 
---

### Soal 3 : DHCP Relay (jichi)

Permintaan alamat dari Divisi A (Network 2) dan Divisi B (Network 3) berupa broadcast, sehingga tidak bisa menyeberangi subnet secara langsung menuju DHCP server di Network 4. Pada soal ini jichi dikonfigurasi sebagai **DHCP Relay** menuju tengen (`10.90.4.10`), menggunakan **eth2** untuk Network 2 dan **eth3** untuk Network 3. Setelah itu dibuktikan bahwa client dari kedua subnet berhasil memperoleh lease melalui relay.

#### Alur kerja

| Langkah | Keterangan |
|---|---|
| 1 | Client di Network 2 atau 3 mengirim DHCPDISCOVER (broadcast) |
| 2 | jichi menerima di eth2 atau eth3, lalu meneruskan ke tengen `10.90.4.10` lewat eth4 dengan alamat relay (giaddr) `10.90.2.1` atau `10.90.3.1` |
| 3 | tengen mencocokkan giaddr dengan subnet yang sesuai (`net2.conf` atau `net3.conf`) dan mengirim penawaran |
| 4 | jichi meneruskan balasan ke client |

> Semua langkah konfigurasi dikerjakan di konsol **jichi**. Verifikasi dikerjakan di client dan tengen.

---

#### 1. Konfigurasi relay (jichi)

Buka konsol jichi, lalu jalankan:
```bash
cat > /root/jichi-relay.sh << 'EOF'
#!/bin/bash
export DEBIAN_FRONTEND=noninteractive
apt update && apt install -y isc-dhcp-relay

cat > /etc/default/isc-dhcp-relay << 'EOT'
SERVERS="10.90.4.10"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
EOT

service isc-dhcp-relay restart
EOF

bash /root/jichi-relay.sh
```

Penjelasan:

| Bagian | Fungsi |
|---|---|
| `apt install isc-dhcp-relay` | Memasang DHCP relay agent (`dhcrelay`) |
| `SERVERS="10.90.4.10"` | Alamat DHCP server (tengen) tujuan forward |
| `INTERFACES="eth2 eth3 eth4"` | eth2 dan eth3 menerima permintaan dari client Network 2 dan 3. eth4 adalah jalur menuju tengen, dimasukkan supaya balasan server dapat diterima relay |
| `service isc-dhcp-relay restart` | Menjalankan relay dengan konfigurasi baru |

> Saat instalasi muncul pesan `invoke-rc.d: policy-rc.d denied execution of start`. Ini normal di container: service tidak otomatis start setelah install, sehingga `service isc-dhcp-relay restart` pada script yang menjalankannya.

Verifikasi di jichi:
```bash
service isc-dhcp-relay status
ps aux | grep [d]hcrelay
cat /etc/default/isc-dhcp-relay
```
Hasil yang diharapkan:
- Proses `/usr/sbin/dhcrelay -q -i eth2 -i eth3 -i eth4 10.90.4.10` berjalan.
- `SERVERS="10.90.4.10"` dan `INTERFACES="eth2 eth3 eth4"`.

##### Bukti :
jichi :
![](assets/soal3-jichi.png)

---

#### 2. Client memperoleh lease lewat relay

Jalankan di satu client Network 2 (itadori) dan satu client Network 3 (ino):
```bash
ip link set eth0 up
udhcpc -i eth0
ip -br addr
ip route
cat /etc/resolv.conf
ping -c 3 10.90.4.10
```

Hasil yang diharapkan:
- itadori memperoleh IP dari pool Network 2 (`10.90.2.10 - 10.90.2.35`) dengan gateway `10.90.2.1`.
- ino memperoleh IP dari pool Network 3 (`10.90.3.10 - 10.90.3.89`) dengan gateway `10.90.3.1`.
- `resolv.conf` berisi nameserver `10.90.1.2` dan `10.90.1.3`.
- Ping ke tengen berhasil. TTL 63 menandakan paket melewati satu router (jichi).

##### Bukti :
itadori (Network 2) : memperoleh `10.90.2.10/24`, server `10.90.4.10`, lease time 600, default via `10.90.2.1`.
![](assets/soal3-itadori.png)

ino (Network 3) : memperoleh `10.90.3.10/24`, server `10.90.4.10`, lease time 600, default via `10.90.3.1`.
![](assets/soal3-ino.png)

---

#### 3. Bukti di sisi server (tengen)

```bash
grep -E "lease|client-hostname|hardware" /var/lib/dhcp/dhcpd.leases | tail -20
```

Hasil yang diharapkan: tengen mencatat dua lease dari subnet berbeda (`10.90.2.10` dan `10.90.3.10`) dengan MAC address masing-masing client.

##### Bukti :
tengen :
![](assets/soal3-tengen.png)

MAC `02:42:9e:05:11:00` adalah eth0 itadori dan `02:42:c9:06:06:00` adalah eth0 ino, sesuai dengan alamat link-local pada output `ip -br addr` di masing-masing client.

> Jika jichi di-stop lalu di-start lagi, konfigurasi IP hilang dan relay tidak berjalan. Jalankan ulang `bash /root/jichi.sh`, lalu `service isc-dhcp-relay restart`.

---

### Soal 4 : Dua Pool DHCP pada Network 2 dengan DHCP Class (tengen)

Tim Dev dan Support di Divisi A (Network 2) harus menerima jatah alamat yang berbeda, padahal semua client berada pada subnet yang sama (`10.90.2.0/24`). Karena itu pada soal ini scope sementara Network 2 dari Soal 2 diganti dengan **dua pool** yang dipisahkan memakai **DHCP class berdasarkan MAC address**.

#### Gambaran pembagian

| Pool | Client | Rentang IP | Keterangan |
|---|---|---|---|
| Pool 1 | itadori, fushiguro | 10.90.2.10 - 10.90.2.35 | Dev |
| Pool 2 | zenin, okkotsu | 10.90.2.70 - 10.90.2.95 | Support |
| - | - | 10.90.2.36 - 10.90.2.69 | **Tidak boleh dialokasikan** (sengaja tidak dimasukkan ke pool mana pun) |

MAC address eth0 tiap client (diambil dari `ip link show eth0` atau dari alamat link-local pada `ip -br addr`):

| Client | MAC address eth0 | Class |
|---|---|---|
| itadori | `02:42:9e:05:11:00` | pool1 |
| fushiguro | `02:42:d6:c7:fd:00` | pool1 |
| zenin | `02:42:a7:06:c5:00` | pool2 |
| okkotsu | `02:42:1e:db:db:00` | pool2 |

> Semua langkah konfigurasi dikerjakan di konsol **tengen**. Verifikasi dikerjakan di keempat client dan tengen.

---

#### 1. Konfigurasi class dan pool (tengen)

Buka konsol tengen, lalu jalankan:
```bash
cat > /root/tengen-net2.sh << 'EOF'
#!/bin/bash
MAC_ITADORI="02:42:9e:05:11:00"
MAC_FUSHIGURO="02:42:d6:c7:fd:00"
MAC_ZENIN="02:42:a7:06:c5:00"
MAC_OKKOTSU="02:42:1e:db:db:00"

cat > /etc/dhcp/net2.conf << EOT
# Kelas client Network 2 berdasarkan MAC address
class "pool1" {
  match hardware;
}
subclass "pool1" 1:$MAC_ITADORI;
subclass "pool1" 1:$MAC_FUSHIGURO;

class "pool2" {
  match hardware;
}
subclass "pool2" 1:$MAC_ZENIN;
subclass "pool2" 1:$MAC_OKKOTSU;

subnet 10.90.2.0 netmask 255.255.255.0 {
  option subnet-mask 255.255.255.0;
  option routers 10.90.2.1;
  option broadcast-address 10.90.2.255;
  option domain-name-servers 10.90.1.2, 10.90.1.3;
  option domain-search "K53.com";

  # Pool 1: itadori dan fushiguro
  pool {
    allow members of "pool1";
    range 10.90.2.10 10.90.2.35;
  }

  # Pool 2: zenin dan okkotsu
  pool {
    allow members of "pool2";
    range 10.90.2.70 10.90.2.95;
  }
  # 10.90.2.36 - 10.90.2.69 sengaja tidak dimasukkan ke pool mana pun
}
EOT

dhcpd -t -cf /etc/dhcp/dhcpd.conf && service isc-dhcp-server restart
EOF

bash /root/tengen-net2.sh
```

Penjelasan:

| Bagian | Fungsi |
|---|---|
| `class "pool1" { match hardware; }` | Membuat kelas yang mencocokkan client berdasarkan hardware address (tipe + MAC) dari paket DHCP |
| `subclass "pool1" 1:<MAC>` | Memasukkan satu MAC ke kelas `pool1`. Awalan `1:` adalah tipe hardware Ethernet |
| `pool { allow members of "pool1"; range ...; }` | Pool hanya boleh dipakai anggota kelas `pool1`, dengan rentang `10.90.2.10 - 10.90.2.35` |
| `pool { allow members of "pool2"; range ...; }` | Pool hanya boleh dipakai anggota kelas `pool2`, dengan rentang `10.90.2.70 - 10.90.2.95` |
| `10.90.2.36 - 10.90.2.69` | Tidak ada `range` yang mencakupnya, sehingga dhcpd tidak akan pernah mengalokasikannya |
| `option routers/broadcast/domain-name-servers/domain-search` | Opsi yang sama seperti Soal 2: gateway `10.90.2.1`, broadcast `10.90.2.255`, DNS `10.90.1.2` dan `10.90.1.3`, domain-search `K53.com` |
| `dhcpd -t -cf ... && service ... restart` | Validasi sintaks dulu, service baru di-restart bila valid |

> File `net2.conf` sudah di-`include` dari `dhcpd.conf` pada Soal 2, jadi `dhcpd.conf` tidak perlu diubah. Script ini hanya menimpa isi `net2.conf`.

> Jika client tidak memperoleh alamat, periksa lagi MAC pada script. Pada font terminal, angka `1` bisa terlihat mirip huruf `l`, padahal hex tidak punya huruf `l`. Contohnya MAC okkotsu adalah `02:42:1e:db:db:00`, sesuai dengan alamat link-local `fe80::42:1eff:fedb:db01` pada `ip -br addr` di okkotsu.

Verifikasi konfigurasi di tengen:
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
cat /etc/dhcp/net2.conf
```
Hasil yang diharapkan:
- `dhcpd -t` tidak mengeluarkan error.
- Status service **dhcpd is running**.
- `net2.conf` berisi dua class, empat subclass, dan dua pool sesuai tabel di atas.

##### Bukti :
tengen :
![](assets/soal4-tengen.png)

---

#### 2. Client meminta lease baru

Jalankan di keempat client (itadori, fushiguro, zenin, okkotsu):
```bash
killall udhcpc 2>/dev/null
ip addr flush dev eth0
ip link set eth0 up
udhcpc -i eth0
ip -br addr
ip route
cat /etc/resolv.conf
```

Penjelasan:

| Perintah | Fungsi |
|---|---|
| `killall udhcpc` | Menghentikan klien DHCP lama supaya tidak bentrok |
| `ip addr flush dev eth0` | Membuang IP lama (misalnya dari pool sementara Soal 2 dan 3) |
| `udhcpc -i eth0` | Meminta lease baru. Permintaan melewati jichi (relay) menuju tengen |
| `ip -br addr`, `ip route`, `cat /etc/resolv.conf` | Memeriksa IP, gateway, dan DNS yang diterima |

Hasil yang diharapkan:
- itadori dan fushiguro memperoleh IP di `10.90.2.10 - 10.90.2.35`.
- zenin dan okkotsu memperoleh IP di `10.90.2.70 - 10.90.2.95`.
- Gateway `10.90.2.1`, DNS `10.90.1.2` dan `10.90.1.3`.

##### Bukti :
itadori (Pool 1) : memperoleh `10.90.2.10/24` dari server `10.90.4.10`, default via `10.90.2.1`, nameserver `10.90.1.2` dan `10.90.1.3`.
![](assets/soal4-itadori.png)

fushiguro (Pool 1) : memperoleh `10.90.2.11/24` dari server `10.90.4.10`, default via `10.90.2.1`, nameserver `10.90.1.2` dan `10.90.1.3`.
![](assets/soal4-fushiguro.png)

zenin (Pool 2) : memperoleh `10.90.2.70/24` dari server `10.90.4.10`, default via `10.90.2.1`, nameserver `10.90.1.2` dan `10.90.1.3`.
![](assets/soal4-zenin.png)

okkotsu (Pool 2) : memperoleh `10.90.2.71/24` dari server `10.90.4.10`, default via `10.90.2.1`, nameserver `10.90.1.2` dan `10.90.1.3`.
![](assets/soal4-okkotsu.png)

Ringkasan hasil:

| Client | Pool | IP yang diterima | Di dalam rentang? |
|---|---|---|---|
| itadori | Pool 1 | 10.90.2.10 | Ya (10 - 35) |
| fushiguro | Pool 1 | 10.90.2.11 | Ya (10 - 35) |
| zenin | Pool 2 | 10.90.2.70 | Ya (70 - 95) |
| okkotsu | Pool 2 | 10.90.2.71 | Ya (70 - 95) |

Tidak ada client yang memperoleh alamat di rentang `10.90.2.36 - 10.90.2.69`.

---

#### 3. Bukti di sisi server (tengen)

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
grep -E "^lease|hardware ethernet" /var/lib/dhcp/dhcpd.leases
```

Hasil yang diharapkan: tengen mencatat lease untuk keempat client Network 2, masing-masing dengan MAC yang sesuai kelasnya.

##### Bukti :
tengen :
![](assets/soal4-tengen2.png)

Pencocokan lease dengan MAC:

| IP | MAC address | Client | Pool |
|---|---|---|---|
| 10.90.2.10 | `02:42:9e:05:11:00` | itadori | Pool 1 |
| 10.90.2.11 | `02:42:d6:c7:fd:00` | fushiguro | Pool 1 |
| 10.90.2.70 | `02:42:a7:06:c5:00` | zenin | Pool 2 |
| 10.90.2.71 | `02:42:1e:db:db:00` | okkotsu | Pool 2 |

> Satu IP dapat muncul beberapa kali di `dhcpd.leases` karena file ini bersifat log yang terus ditambah setiap lease dibuat atau diperbarui. Entri paling bawah untuk suatu IP adalah yang berlaku.

> Pada Soal 3, itadori sudah memperoleh `10.90.2.10` dari scope sementara. Setelah pool dipisah, itadori tetap memperoleh alamat yang sama karena `10.90.2.10` memang awal Pool 1, sehingga hasilnya konsisten.

> Jika tengen di-stop lalu di-start lagi, konfigurasi IP hilang dan paket yang terpasang bisa ikut hilang. Jalankan ulang `bash /root/tengen.sh` (Soal 1), `bash /root/tengen-dhcp.sh` (Soal 2), lalu `bash /root/tengen-net2.sh` (Soal 4) agar `net2.conf` kembali berisi dua pool.

---
