# PPTP
# Onlinkan Localhost dengan menggunakan VPS dan Mikrotik


# Install Dan Settings PPTP
Yang pertama kalian Update dan install PPTP nya

```bash
sudo apt-get update
sudo apt-get upgrade -y
```
```bash
sudo apt-get install pptpd -y
```

Sekarang edit di bagian /etc/pptpd.conf
```bash
nano /etc/pptpd.conf
```
Dan isikan ini
```bash
localip 10.0.0.1
remoteip 10.0.0.100-200
```
Terus kalian konfigurasi Pw dan Username nya
```bash
sudo nano /etc/ppp/chap-secrets
```
Kalian isikan Pw yang kuat 
(contoh configurasi nya)
```bash
# client    server    secret    IP addresses
mikrotik    pptpd    password_anda    *
```
Terus kalian konfigurasi dns nya
```bash
sudo nano /etc/ppp/pptpd-options
```
Tambahkan ini
```bash
ms-dns 8.8.8.8
ms-dns 8.8.4.4
```
Aktidkan IP Forwading
```bash
sudo nano /etc/sysctl.conf
```
Tambahkan
```bash
net.ipv4.ip_forward=1
```
Dan terapkan
```bash
sudo sysctl -p
```


# Setings Iptables
Sekarang kalian cek dulu ip a nya kalian
```bash
ip a
```
Seperti ini biasa nya
```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether bc:24:11:67:49:54 brd ff:ff:ff:ff:ff:ff
    altname enp0s18
    inet 192.168.1.30/24 metric 100 brd 192.168.1.255 scope global dynamic ens18
       valid_lft 77951sec preferred_lft 77951sec
    inet6 fe80::be24:11ff:fe67:4954/64 scope link
       valid_lft forever preferred_lft forever
3: ppp0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1446 qdisc fq_codel state UNKNOWN group default qlen 3
    link/ppp
    inet 10.0.0.1 peer 10.0.0.100/32 scope global ppp0
       valid_lft forever preferred_lft forever
```


Sekarang sesuaikan coman iptables nya dengan interface kalian
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o ppp0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i ppp0 -o eth0 -j ACCEPT
```
Terus save
```bash
sudo apt-get install iptables-persistent -y
sudo netfilter-persistent save
```

Terakir kalian restart PPTP nya
```bash
sudo systemctl restart pptpd
```

# Dan sekarang kalian bisa seting di Mikrotik nya
1 Buat PPTP Client:

2 Buka menu PPP
  Pilih tab "Interface"
  Klik tombol "+" (add)
  Pilih "PPTP Client"
  Name: pptp-to-vps
  Connect To: [IP Publik VPS]
  User: mikrotik (username yang Anda buat di server PPTP)
  Password: password_anda
  Di tab "Dial Out", pastikan "Add Default Route" tidak dicentang (kecuali jika Anda ingin semua traffic melalui VPN)
  Klik Apply, lalu OK

3 Verifikasi koneksi:
  Setelah beberapa saat, status PPTP Client seharusnya "R" (running)
  Anda bisa memeriksa status di menu PPP → Interface → PPTP Client
  Periksa IP yang didapat dengan masuk ke menu IP → Addresses (seharusnya ada IP dari range 10.0.0.100-200)

# Sekarang kalian seting port forwading di vps nya
Kalian bisa sesuaikan port dan ip yang di dapat dari koneksi vpn kalian
```bash
# Format: iptables -t nat -A PREROUTING -p [protokol] --dport [port_vps] -j DNAT --to-destination [ip_mikrotik_pptp]:[port_mikrotik]

# Contoh: Forward HTTP (port 80)
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.100:80

# Contoh: Forward HTTPS (port 443) 
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination 10.0.0.100:443

# Contoh: Forward port khusus (misalnya 8080)
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.100:8080

iptables -t nat -A POSTROUTING -p tcp -d 10.0.0.100 --dport 80 -j MASQUERADE

# Simpan aturan
sudo netfilter-persistent save
```

# Terus sekarang Port Forwading di Mikrotik
1 Buka menu IP → Fierwall
2 Pilih tab "NAT"
3 Isi konfigurasi berikut:
4 Chain: srcnat
  Out Interface: Nama koneksi PPTP kalian
  Action: masquerade


1 Buka menu IP → Firewall
2 Pilih tab "NAT"
3 Klik tombol "+" (add)
4 Isi konfigurasi berikut:
5 Chain: dstnat
  Protocol: tcp
  Dst. Interface: pptp-to-vps (interface PPTP yang Anda buat)
  Dst. Port: 80 (atau port yang sama dengan konfigurasi di VPS)
  Action: dst-nat
  To Addresses: 192.168.x.x (IP server lokal di jaringan Anda)
  To Ports: 80 (port yang digunakan server lokal)
  Klik OK

Ulangi langkah yang sama untuk port lain yang ingin Anda forward








