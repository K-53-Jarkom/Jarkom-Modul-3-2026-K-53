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
