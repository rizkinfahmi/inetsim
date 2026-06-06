# INetSim DNS Fix untuk Net::DNS 1.44+

## Latar Belakang

INetSim 1.3.2 menggunakan `Net::DNS::Nameserver` dengan method `main_loop` yang sudah deprecated di Net::DNS versi 1.03+. Pada Net::DNS 1.44, method ini melakukan fork subprocess secara internal sehingga:

- DNS service tampak "started" di log tapi port 53 tidak terbind
- Proses `inetsim_dns_53_tcp_udp` tidak muncul di `ps aux`
- Query DNS tidak mendapat response

---

## Environment

- OS: REMnux (Ubuntu-based)
- INetSim: 1.3.2
- Net::DNS: 1.44
- Perl: 5.38

---

## Gejala

```
* dns_53_tcp_udp - started (PID 2515)
Attempt to start Net::DNS::Nameserver in a subprocess at /usr/share/perl5/INetSim/DNS.pm line 70.
```

```bash
sudo ss -lunp | grep :53
# (tidak ada output)

sudo ps aux | grep inetsim_dns
# (tidak ada proses)
```

---

## Root Cause

Di Net::DNS 1.44:

1. `main_loop` → hanya wrapper, memanggil `start_server()` lalu print warning deprecated
2. `start_server()` → fork subprocess untuk TCP dan UDP server, lalu **return** (non-blocking)
3. Karena `start_server()` langsung return, proses parent langsung `exit 0`
4. Child process ikut mati karena parent exit
5. Socket bind ke port 53 terjadi di dalam `loop_once` — **setelah** drop privileges — sehingga muncul "Permission denied" untuk port < 1024

---

## Solusi

Gunakan `loop_once()` dalam while loop, dengan urutan:

1. Buat `$server` object (belum bind socket)
2. Panggil `loop_once(0)` **sebagai root** → bind socket ke port 53
3. Drop privileges ke user `inetsim`
4. Loop `while(1) { loop_once(10) }` → handle request secara blocking tanpa fork

---

## Langkah Fix

### 1. Backup file asli

```bash
sudo cp /usr/share/perl5/INetSim/DNS.pm /usr/share/perl5/INetSim/DNS.pm.bak
```

### 2. Replace DNS.pm

Download file `DNS_fixed.pm` dari repo ini, lalu:

```bash
sudo cp DNS_fixed.pm /usr/share/perl5/INetSim/DNS.pm
```

Atau edit manual — ganti bagian fungsi `dns()` dari:

```perl
# SEBELUM (broken)
$server->main_loop;
# atau
$server->start_server();
```

Menjadi urutan berikut (pastikan loop_once(0) dipanggil SEBELUM drop privileges):

```perl
# Bind socket sebagai root
$server->loop_once(0);

# Drop privileges
POSIX::setgid($gid);
POSIX::setuid($uid);

# Loop utama
while (1) {
    $server->loop_once(10);
}
```

### 3. Verifikasi syntax

```bash
perl -c /usr/share/perl5/INetSim/DNS.pm
```

### 4. Nonaktifkan systemd-resolved (jika aktif)

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

### 5. Jalankan INetSim

```bash
sudo inetsim &
sleep 5
sudo ss -lunp | grep :53
sudo ps aux | grep inetsim_dns
```

Output yang diharapkan:

```
* dns_53_tcp_udp - started (PID XXXX)
...
UNCONN 0  0  0.0.0.0:53  0.0.0.0:*  users:(("inetsim_dns_53_",pid=XXXX,fd=4))
inetsim  XXXX  inetsim_dns_53_tcp_udp
```

---

## Konfigurasi inetsim.conf

Edit `/etc/inetsim/inetsim.conf`:

```
# Aktifkan DNS service
start_service dns

# Bind ke semua interface
service_bind_address 0.0.0.0

# IP yang dikembalikan untuk semua DNS query
# Harus = IP REMnux
dns_default_ip 10.0.0.5
```

---

## Konfigurasi Victim/Client

### Linux

```bash
sudo chattr -i /etc/resolv.conf
sudo rm /etc/resolv.conf
echo "nameserver 10.0.0.5" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```

### Windows (CMD sebagai Administrator)

```cmd
netsh interface ip set dns "Ethernet" static 10.0.0.5
```

Ganti `"Ethernet"` dengan nama adapter (cek via `ipconfig`).

---

## Verifikasi End-to-End

**Di REMnux:**
```bash
dig @10.0.0.5 google.com
```

**Di victim:**
```bash
# Linux
dig google.com
nslookup google.com

# Windows
nslookup google.com
```

Semua domain harus resolve ke `10.0.0.5` (IP REMnux) — berarti DNS redirect berhasil dan traffic HTTP/HTTPS akan terlayani oleh fake inetsim server.

---

## Referensi

- [INetSim Official](https://www.inetsim.org/)
- [Net::DNS::Nameserver CPAN](https://metacpan.org/pod/Net::DNS::Nameserver)
- Net::DNS changelog: `start_server()` menggantikan `main_loop()` sejak versi 1.03
