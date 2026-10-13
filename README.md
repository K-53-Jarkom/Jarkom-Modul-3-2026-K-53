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

### Soal 5 : Short Lease dan Long Lease pada Network 3 (tengen)

Tim lapangan Divisi B (Network 3) punya durasi operasi yang berbeda. Pada soal ini scope sementara Network 3 dari Soal 2 diganti dengan **dua pool** yang dipisahkan memakai **DHCP class berdasarkan MAC address**, dan tiap kelas diberi **lease time** sendiri.

#### Gambaran pembagian

| Kelas | Client | Rentang IP | default-lease-time | max-lease-time |
|---|---|---|---|---|
| short | ino, kusakabe | 10.90.3.10 - 10.90.3.89 | 1800 detik (30 menit) | 1800 detik |
| long | miwa | 10.90.3.100 - 10.90.3.199 | 43200 detik (12 jam) | 43200 detik |

Rentang `10.90.3.90 - 10.90.3.99` tidak dimasukkan ke pool mana pun. Alamat `10.90.3.99` dicadangkan untuk fixed address kugisaki pada Soal 6.

MAC address eth0 tiap client (diambil dengan `cat /sys/class/net/eth0/address`):

| Client | MAC address eth0 | Kelas |
|---|---|---|
| ino | `02:42:c9:06:06:00` | short |
| kusakabe | `02:42:33:cb:b7:00` | short |
| miwa | `02:42:46:ce:21:00` | long |

> Semua langkah konfigurasi dikerjakan di konsol **tengen**. Verifikasi dikerjakan di ketiga client dan tengen.

---

#### 1. Konfigurasi class dan pool (tengen)

Buka konsol tengen, lalu jalankan:
```bash
cat > /root/tengen-net3.sh << 'EOF'
#!/bin/bash
MAC_INO="02:42:c9:06:06:00"
MAC_KUSAKABE="02:42:33:cb:b7:00"
MAC_MIWA="02:42:46:ce:21:00"

cat > /etc/dhcp/net3.conf << EOT
# Kelas client Network 3 berdasarkan MAC address
# Short lease: 30 menit (1800 detik)
class "short" {
  match hardware;
  default-lease-time 1800;
  max-lease-time 1800;
}
subclass "short" 1:$MAC_INO;
subclass "short" 1:$MAC_KUSAKABE;

# Long lease: 12 jam (43200 detik)
class "long" {
  match hardware;
  default-lease-time 43200;
  max-lease-time 43200;
}
subclass "long" 1:$MAC_MIWA;

subnet 10.90.3.0 netmask 255.255.255.0 {
  option subnet-mask 255.255.255.0;
  option routers 10.90.3.1;
  option broadcast-address 10.90.3.255;
  option domain-name-servers 10.90.1.2, 10.90.1.3;
  option domain-search "K53.com";

  # Pool short lease: ino dan kusakabe
  pool {
    allow members of "short";
    range 10.90.3.10 10.90.3.89;
  }

  # Pool long lease: miwa
  pool {
    allow members of "long";
    range 10.90.3.100 10.90.3.199;
  }
  # 10.90.3.90 - 10.90.3.99 tidak masuk pool mana pun (.99 untuk fixed address kugisaki di Soal 6)
}
EOT

dhcpd -t -cf /etc/dhcp/dhcpd.conf && service isc-dhcp-server restart
EOF

bash /root/tengen-net3.sh
```

Penjelasan:

| Bagian | Fungsi |
|---|---|
| `class "short" { match hardware; ... }` | Membuat kelas yang mencocokkan client berdasarkan hardware address dari paket DHCP |
| `default-lease-time` dan `max-lease-time` di dalam class | Lease time khusus untuk anggota kelas tersebut. Nilai global dari Soal 2 (600/7200) tidak diubah dan tetap berlaku untuk client lain |
| `subclass "short" 1:<MAC>` | Memasukkan satu MAC ke kelas `short`. Awalan `1:` adalah tipe hardware Ethernet |
| `pool { allow members of "short"; range ...; }` | Pool short hanya boleh dipakai anggota kelas `short`, dengan rentang `10.90.3.10 - 10.90.3.89` |
| `pool { allow members of "long"; range ...; }` | Pool long hanya boleh dipakai anggota kelas `long`, dengan rentang `10.90.3.100 - 10.90.3.199` |
| `option routers/broadcast/domain-name-servers/domain-search` | Gateway `10.90.3.1`, broadcast `10.90.3.255`, DNS `10.90.1.2` dan `10.90.1.3`, domain-search `K53.com` |
| `dhcpd -t -cf ... && service ... restart` | Validasi sintaks dulu, service baru di-restart bila valid |

> File `net3.conf` sudah di-`include` dari `dhcpd.conf` pada Soal 2, jadi `dhcpd.conf` tidak perlu diubah. Script ini hanya menimpa isi `net3.conf`.

> Client Network 3 yang MAC-nya belum terdaftar di kelas mana pun (misalnya kugisaki sebelum Soal 6) tidak akan memperoleh alamat, karena kedua pool dibatasi dengan `allow members of`.

---

#### 2. Client meminta lease baru

Jalankan di ino, kusakabe, dan miwa:
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
| `ip addr flush dev eth0` | Membuang IP lama (dari scope sementara Soal 3) |
| `udhcpc -i eth0` | Meminta lease baru. Permintaan melewati jichi (relay) menuju tengen |
| `ip -br addr`, `ip route`, `cat /etc/resolv.conf` | Memeriksa IP, gateway, dan DNS yang diterima |

Hasil yang diharapkan:
- ino dan kusakabe memperoleh IP di `10.90.3.10 - 10.90.3.89` dengan lease time 1800.
- miwa memperoleh IP di `10.90.3.100 - 10.90.3.199` dengan lease time 43200.
- Gateway `10.90.3.1`, DNS `10.90.1.2` dan `10.90.1.3`.

##### Bukti :
ino (short lease) : memperoleh `10.90.3.10/24` dari server `10.90.4.10`, lease time 1773, default via `10.90.3.1`, nameserver `10.90.1.2` dan `10.90.1.3`.
![](assets/soal5-ino.png)

kusakabe (short lease) : memperoleh `10.90.3.11/24` dari server `10.90.4.10`, lease time 1800, default via `10.90.3.1`, nameserver `10.90.1.2` dan `10.90.1.3`.
![](assets/soal5-kusakabe.png)

miwa (long lease) : memperoleh `10.90.3.100/24` dari server `10.90.4.10`, lease time 43200, default via `10.90.3.1`, nameserver `10.90.1.2` dan `10.90.1.3`.
![](assets/soal5-miwa.png)

> Pada ino, lease time tertulis 1773, bukan 1800. Ini karena ino masih memegang lease `10.90.3.10` yang dibuat sebelumnya, sehingga udhcpc melaporkan **sisa** waktu lease yang sedang berjalan. Batas lease yang diberikan server tetap 1800 detik, terbukti dari kolom `ends` di `dhcpd.leases` (lihat bagian berikut).

Ringkasan hasil:

| Client | Kelas | IP yang diterima | Lease time | Di dalam rentang? |
|---|---|---|---|---|
| ino | short | 10.90.3.10 | 1773 (sisa dari 1800) | Ya (10 - 89) |
| kusakabe | short | 10.90.3.11 | 1800 | Ya (10 - 89) |
| miwa | long | 10.90.3.100 | 43200 | Ya (100 - 199) |

---

#### 3. Bukti di sisi server (tengen)

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
cat /etc/dhcp/net3.conf
grep -E "^lease|hardware ethernet|ends" /var/lib/dhcp/dhcpd.leases | tail -20
```

Hasil yang diharapkan:
- `dhcpd -t` tidak mengeluarkan error dan status service **dhcpd is running**.
- `net3.conf` berisi dua class, tiga subclass, dan dua pool sesuai tabel di atas.
- `dhcpd.leases` mencatat lease ino dan kusakabe berdurasi 30 menit, serta lease miwa berdurasi 12 jam.

##### Bukti :
tengen (status service dan isi `net3.conf`) :
![](assets/soal5-tengen.png)
![](assets/soal5-tengen2.png)

Pencocokan lease di tengen (waktu `ends` pada `dhcpd.leases` memakai UTC):

| IP | MAC address | Client | `ends` | Keterangan |
|---|---|---|---|---|
| 10.90.3.10 | `02:42:c9:06:06:00` | ino | 2026/10/12 14:13:47 | Lease dibuat 13:43:47, berdurasi 30 menit |
| 10.90.3.11 | `02:42:33:cb:b7:00` | kusakabe | 2026/10/12 14:14:40 | Lease dibuat 13:44:40, berdurasi 30 menit |
| 10.90.3.100 | `02:42:46:ce:21:00` | miwa | 2026/10/13 01:44:56 | Lease dibuat 13:44:56, berdurasi 12 jam |

Dari tabel terlihat selisih waktu lease ino dan kusakabe 30 menit, sedangkan miwa 12 jam, sesuai kebijakan short lease dan long lease.

> Entri lease Network 2 (`10.90.2.x`) yang ikut muncul pada `tail` adalah sisa lease Soal 4 dan tidak berkaitan dengan Soal 5.

> Jika tengen di-stop lalu di-start lagi, konfigurasi IP hilang dan paket yang terpasang bisa ikut hilang. Jalankan ulang `bash /root/tengen.sh` (Soal 1), `bash /root/tengen-dhcp.sh` (Soal 2), `bash /root/tengen-net2.sh` (Soal 4), lalu `bash /root/tengen-net3.sh` (Soal 5) agar `net2.conf` dan `net3.conf` kembali berisi kebijakan pool final.

---

### Soal 6 : Static Lease untuk Kugisaki (tengen)

Kugisaki ditetapkan sebagai personel VIP yang harus mempertahankan alamat yang sama setiap kali terhubung. Pada soal ini dibuat **DHCP static lease** (reservasi) berdasarkan MAC address kugisaki dengan **fixed address `10.90.3.99`**.

#### Gambaran reservasi

| Client | Network | MAC address eth0 | Fixed address |
|---|---|---|---|
| kugisaki | 3 (Divisi B) | `02:42:59:c7:e6:00` | 10.90.3.99 |

Alamat `10.90.3.99` berada **di luar seluruh pool dinamis** Network 3:

| Pool | Rentang | Mencakup 10.90.3.99? |
|---|---|---|
| short (ino, kusakabe) | 10.90.3.10 - 10.90.3.89 | Tidak |
| long (miwa) | 10.90.3.100 - 10.90.3.199 | Tidak |

MAC address diambil dengan `cat /sys/class/net/eth0/address` di kugisaki.

> Semua langkah konfigurasi dikerjakan di konsol **tengen**. Verifikasi dikerjakan di kugisaki dan tengen.

---

#### 1. Konfigurasi static lease (tengen)

Buka konsol tengen, lalu jalankan:
```bash
cat > /root/tengen-kugisaki.sh << 'EOF'
#!/bin/bash
MAC_KUGISAKI="02:42:59:c7:e6:00"

cat > /etc/dhcp/net3-static.conf << EOT
# Static lease (VIP) Network 3: kugisaki selalu memperoleh 10.90.3.99
host kugisaki {
  hardware ethernet $MAC_KUGISAKI;
  fixed-address 10.90.3.99;
}
EOT

# Tambahkan include ke dhcpd.conf hanya jika belum ada
grep -q 'net3-static.conf' /etc/dhcp/dhcpd.conf || \
  echo 'include "/etc/dhcp/net3-static.conf";' >> /etc/dhcp/dhcpd.conf

dhcpd -t -cf /etc/dhcp/dhcpd.conf && service isc-dhcp-server restart
EOF

bash /root/tengen-kugisaki.sh
```

Penjelasan:

| Bagian | Fungsi |
|---|---|
| `host kugisaki { ... }` | Reservasi alamat untuk satu client |
| `hardware ethernet <MAC>` | MAC eth0 kugisaki sebagai kunci reservasi |
| `fixed-address 10.90.3.99` | Alamat tetap yang selalu diberikan ke MAC tersebut. Berada di celah `10.90.3.90 - 10.90.3.99`, di luar pool short dan long |
| `net3-static.conf` terpisah | Reservasi disimpan di file sendiri, sehingga `tengen-net3.sh` (Soal 5) boleh dijalankan ulang tanpa menghapus reservasi |
| `grep -q ... \|\| echo ... >>` | Menambahkan `include` ke `dhcpd.conf` hanya sekali, aman dijalankan berulang |
| `dhcpd -t ... && service ... restart` | Validasi sintaks dulu, service baru di-restart bila valid |

> Kugisaki tidak dimasukkan ke kelas `short` maupun `long`, jadi alamatnya hanya berasal dari reservasi ini. Karena tidak memakai kelas, lease time-nya mengikuti nilai global (`default-lease-time 600`).

Verifikasi konfigurasi di tengen:
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
cat /etc/dhcp/net3-static.conf
grep -n include /etc/dhcp/dhcpd.conf
```
Hasil yang diharapkan:
- `dhcpd -t` tidak mengeluarkan error.
- Status service **dhcpd is running**.
- `net3-static.conf` berisi blok `host kugisaki` dengan MAC dan `fixed-address 10.90.3.99`.
- `dhcpd.conf` meng-include `net2.conf`, `net3.conf`, dan `net3-static.conf`.

##### Bukti :
tengen :
![](assets/soal6-tengen.png)

Hasil di atas sesuai harapan: validasi konfigurasi lolos tanpa error, `dhcpd is running`, blok `host kugisaki` berisi MAC `02:42:59:c7:e6:00` dan `fixed-address 10.90.3.99`, serta ketiga file ter-include di baris 11, 12, dan 13 pada `dhcpd.conf`.

---

#### 2. Renew lease di kugisaki

Jalankan di kugisaki:
```bash
killall udhcpc 2>/dev/null
ip addr flush dev eth0
ip link set eth0 up
udhcpc -i eth0
ip -br addr
ip route
cat /etc/resolv.conf
```

Lalu ulangi permintaan sekali lagi untuk membuktikan alamatnya tetap:
```bash
killall udhcpc 2>/dev/null
ip addr flush dev eth0
udhcpc -i eth0
ip -br addr
```

Penjelasan:

| Perintah | Fungsi |
|---|---|
| `killall udhcpc` | Menghentikan klien DHCP lama supaya tidak bentrok |
| `ip addr flush dev eth0` | Membuang IP lama, sehingga permintaan berikutnya benar-benar dari awal |
| `udhcpc -i eth0` | Meminta lease baru. Permintaan melewati jichi (relay) menuju tengen |
| `ip -br addr`, `ip route`, `cat /etc/resolv.conf` | Memeriksa IP, gateway, dan DNS yang diterima |

Hasil yang diharapkan:
- kugisaki memperoleh `10.90.3.99/24` dari server `10.90.4.10`.
- Gateway `10.90.3.1`, DNS `10.90.1.2` dan `10.90.1.3`.
- Alamat tetap `10.90.3.99` pada permintaan pertama maupun kedua.

##### Bukti :
kugisaki : memperoleh `10.90.3.99/24` dari server `10.90.4.10`, lease time 600, default via `10.90.3.1`, nameserver `10.90.1.2` dan `10.90.1.3`. Setelah IP dibuang dan lease diminta ulang, alamat yang diterima tetap `10.90.3.99`.
![](assets/soal6-kugisaki.png)

Ringkasan hasil:

| Permintaan | IP yang diterima | Server | Sesuai reservasi? |
|---|---|---|---|
| Pertama | 10.90.3.99 | 10.90.4.10 | Ya |
| Kedua (setelah flush) | 10.90.3.99 | 10.90.4.10 | Ya |

Alamat `10.90.3.99` sesuai dengan reservasi `fixed-address` pada `net3-static.conf` dan tidak berada di dalam pool short (10 - 89) maupun long (100 - 199).

> Pada `/var/lib/dhcp/dhcpd.leases`, host dengan `fixed-address` biasanya tidak dicatat sebagai lease dinamis, sehingga pencarian `lease 10.90.3.99` di file tersebut dapat kosong. Bukti reservasi ada pada blok `host kugisaki` di sisi server dan alamat yang diterima di sisi client.

> Jika tengen di-stop lalu di-start lagi, konfigurasi IP hilang dan paket yang terpasang bisa ikut hilang. Jalankan ulang `bash /root/tengen.sh` (Soal 1), `bash /root/tengen-dhcp.sh` (Soal 2), `bash /root/tengen-net2.sh` (Soal 4), `bash /root/tengen-net3.sh` (Soal 5), lalu `bash /root/tengen-kugisaki.sh` (Soal 6).

---

### Soal 7 : Verifikasi Konfigurasi Client Divisi A dan Divisi B

Setelah seluruh aturan alokasi diterapkan (Soal 2 - 6), pada soal ini dibuktikan bahwa **setiap client** menerima konfigurasi jaringan sesuai wilayahnya: subnet mask `/24`, gateway dan broadcast yang benar, DNS `10.90.1.2` dan `10.90.1.3`, serta domain-search `K53.com`. Setelah itu lease dicocokkan dengan `/var/lib/dhcp/dhcpd.leases` di tengen untuk memastikan tidak ada alamat yang bertabrakan atau berada di luar kebijakan pool.

#### Yang diverifikasi

| Aspek | Nilai yang diharapkan |
|---|---|
| Subnet mask | 255.255.255.0 (/24) |
| Gateway Network 2 | 10.90.2.1 |
| Gateway Network 3 | 10.90.3.1 |
| Broadcast Network 2 | 10.90.2.255 |
| Broadcast Network 3 | 10.90.3.255 |
| DNS | 10.90.1.2 dan 10.90.1.3 |
| Domain-search | K53.com |

> Verifikasi dikerjakan di delapan client (Divisi A: itadori, fushiguro, zenin, okkotsu. Divisi B: ino, kusakabe, miwa, kugisaki) dan di tengen.

---

#### 1. Client meminta lease baru dan memeriksa konfigurasi

Jalankan di kedelapan client:
```bash
killall udhcpc 2>/dev/null
ip addr flush dev eth0
ip link set eth0 up
udhcpc -i eth0
ip addr show eth0 | grep -E "inet |brd"
ip route
cat /etc/resolv.conf
```

Penjelasan:

| Perintah | Fungsi |
|---|---|
| `killall udhcpc` | Menghentikan klien DHCP lama supaya tidak bentrok |
| `ip addr flush dev eth0` | Membuang IP lama sehingga permintaan benar-benar dari awal |
| `udhcpc -i eth0` | Meminta lease baru lewat relay jichi menuju tengen |
| `ip addr show eth0 \| grep -E "inet \|brd"` | Menampilkan IP beserta prefix (`/24`) dan broadcast (`brd`) |
| `ip route` | Memeriksa default gateway dan route jaringan lokal |
| `cat /etc/resolv.conf` | Memeriksa DNS yang diterima |

##### Bukti :
**Divisi A (Network 2)**

itadori :
![](assets/soal7-itadori.png)

fushiguro :
![](assets/soal7-fushiguro.png)

zenin :
![](assets/soal7-zenin.png)

okkotsu :
![](assets/soal7-okkotsu.png)

**Divisi B (Network 3)**

ino :
![](assets/soal7-ino.png)

kusakabe :
![](assets/soal7-kusakabe.png)

miwa :
![](assets/soal7-miwa.png)

kugisaki :
![](assets/soal7-kugisaki.png)

Ringkasan hasil:

| Client | Pool / kelas | IP | Mask | Gateway | Broadcast | Lease time | DNS |
|---|---|---|---|---|---|---|---|
| itadori | Pool 1 | 10.90.2.10 | /24 | 10.90.2.1 | 10.90.2.255 | 600 | 10.90.1.2, 10.90.1.3 |
| fushiguro | Pool 1 | 10.90.2.11 | /24 | 10.90.2.1 | 10.90.2.255 | 600 | 10.90.1.2, 10.90.1.3 |
| zenin | Pool 2 | 10.90.2.70 | /24 | 10.90.2.1 | 10.90.2.255 | 600 | 10.90.1.2, 10.90.1.3 |
| okkotsu | Pool 2 | 10.90.2.71 | /24 | 10.90.2.1 | 10.90.2.255 | 600 | 10.90.1.2, 10.90.1.3 |
| ino | short | 10.90.3.10 | /24 | 10.90.3.1 | 10.90.3.255 | 1800 | 10.90.1.2, 10.90.1.3 |
| kusakabe | short | 10.90.3.11 | /24 | 10.90.3.1 | 10.90.3.255 | 1800 | 10.90.1.2, 10.90.1.3 |
| miwa | long | 10.90.3.100 | /24 | 10.90.3.1 | 10.90.3.255 | 43200 | 10.90.1.2, 10.90.1.3 |
| kugisaki | fixed address | 10.90.3.99 | /24 | 10.90.3.1 | 10.90.3.255 | 600 | 10.90.1.2, 10.90.1.3 |

Seluruh client memperoleh server DHCP `10.90.4.10` (tengen) lewat relay di jichi. Route yang terbentuk pada tiap client adalah `default via <gateway> dev eth0` ditambah route jaringan lokal `/24` dengan `src` sesuai IP client.

---

#### 2. Pembuktian domain-search

Pada `resolv.conf` client hanya muncul baris `nameserver`, tanpa baris `search`. Ini karena script bawaan `udhcpc` pada image client hanya menulis DNS ke `resolv.conf` dan tidak memproses opsi domain-search. Agar terbukti bahwa opsi tersebut sampai ke client, nilai opsi yang diterima `udhcpc` ditampilkan dengan script sementara. Jalankan di satu client per network (itadori dan ino):
```bash
cat > /tmp/dump.sh << 'EOF'
#!/bin/sh
env | grep -E "^(ip|subnet|router|broadcast|dns|domain|search)="
EOF
chmod +x /tmp/dump.sh
udhcpc -i eth0 -n -q -O search -s /tmp/dump.sh
```

Penjelasan:

| Bagian | Fungsi |
|---|---|
| `-O search` | Meminta opsi domain-search (opsi 119) dari server |
| `-n -q` | Berhenti setelah satu kali memperoleh lease |
| `-s /tmp/dump.sh` | Menjalankan script sementara yang hanya menampilkan opsi yang diterima. Script ini tidak mengubah IP client |

##### Bukti :
itadori (Network 2) : opsi yang diterima `subnet=255.255.255.0`, `router=10.90.2.1`, `search=K53.com`, `dns=10.90.1.2 10.90.1.3`, `broadcast=10.90.2.255`, `ip=10.90.2.10`.
![](assets/soal7-itadori2.png)

ino (Network 3) : opsi yang diterima `subnet=255.255.255.0`, `router=10.90.3.1`, `search=K53.com`, `dns=10.90.1.2 10.90.1.3`, `broadcast=10.90.3.255`, `ip=10.90.3.10`.
![](assets/soal7-ino2.png)

> Pada ino, lease time yang dilaporkan `udhcpc` pada permintaan ini adalah 1599, bukan 1800. Itu karena ino masih memegang lease yang baru dibuat, sehingga yang dilaporkan adalah **sisa** waktu lease yang berjalan. Batas yang diberikan server tetap 1800 detik (lihat permintaan sebelumnya yang menampilkan lease time 1800).

---

#### 3. Mencocokkan lease di tengen

Jalankan di tengen:
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
cat /etc/dhcp/net2.conf /etc/dhcp/net3.conf /etc/dhcp/net3-static.conf
awk '/^lease/{ip=$2} /^  binding state/{st[ip]=$3} /hardware ethernet/{mac[ip]=$3} /^  ends/{e[ip]=$3" "$4} END{for(i in mac) print i, mac[i], st[i], e[i]}' /var/lib/dhcp/dhcpd.leases | sort -V
```

Penjelasan:

| Perintah | Fungsi |
|---|---|
| `dhcpd -t -cf ...` | Memvalidasi konfigurasi tanpa error |
| `service isc-dhcp-server status` | Memastikan dhcpd berjalan |
| `cat net2.conf net3.conf net3-static.conf` | Menampilkan seluruh kebijakan pool, kelas, dan reservasi |
| `awk ... dhcpd.leases \| sort -V` | `dhcpd.leases` bersifat log yang terus bertambah (satu IP bisa muncul berkali-kali). `awk` merangkumnya menjadi satu baris per IP dengan MAC, status, dan waktu `ends` terbaru (waktu mengikuti jam server, UTC) |

##### Bukti :
tengen :
![](assets/soal7-tengen.png)

Hasil di atas sesuai harapan: validasi konfigurasi lolos tanpa error, `dhcpd is running`, dan isi tiga file konfigurasi sesuai kebijakan Soal 4, 5, dan 6.

Pencocokan lease dengan client:

| IP | MAC address | Client | Pool / kelas | Di dalam kebijakan? |
|---|---|---|---|---|
| 10.90.2.10 | `02:42:9e:05:11:00` | itadori | Pool 1 (10 - 35) | Ya |
| 10.90.2.11 | `02:42:d6:c7:fd:00` | fushiguro | Pool 1 (10 - 35) | Ya |
| 10.90.2.70 | `02:42:a7:06:c5:00` | zenin | Pool 2 (70 - 95) | Ya |
| 10.90.2.71 | `02:42:1e:db:db:00` | okkotsu | Pool 2 (70 - 95) | Ya |
| 10.90.3.10 | `02:42:c9:06:06:00` | ino | short (10 - 89) | Ya |
| 10.90.3.11 | `02:42:33:cb:b7:00` | kusakabe | short (10 - 89) | Ya |
| 10.90.3.100 | `02:42:46:ce:21:00` | miwa | long (100 - 199) | Ya |
| 10.90.3.99 | `02:42:59:c7:e6:00` | kugisaki | `host kugisaki` (fixed-address) | Ya |

Pemeriksaan kebijakan:

| Pemeriksaan | Hasil |
|---|---|
| Tabrakan alamat | Tidak ada. Tujuh lease dinamis memakai tujuh IP berbeda dan tujuh MAC berbeda |
| Alamat di luar pool | Tidak ada. Tidak satu pun IP berada di `10.90.2.36 - 10.90.2.69` maupun `10.90.3.90 - 10.90.3.98` |
| Kugisaki | `10.90.3.99` berasal dari reservasi `fixed-address` di `net3-static.conf` dan berada di luar pool short dan long. Pada `dhcpd.leases` host dengan `fixed-address` tidak dicatat sebagai lease dinamis, sehingga kugisaki tidak muncul pada ringkasan `awk` |

> Jika node di-stop lalu di-start lagi, konfigurasi IP hilang dan paket yang terpasang bisa ikut hilang. Jalankan ulang script pemulihan sesuai urutan: `bash /root/jichi.sh`, `bash /root/tengen.sh`, `bash /root/tengen-dhcp.sh`, `bash /root/tengen-net2.sh`, `bash /root/tengen-net3.sh`, `bash /root/tengen-kugisaki.sh`, lalu `bash /root/jichi-relay.sh`.

---
