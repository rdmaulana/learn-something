# 🚀 WireGuard VPN Setup on Oracle Cloud

Dokumentasi ini menjelaskan cara lengkap membuat server VPN dengan WireGuard di Oracle Cloud VM (Ampere A1 Flex / E2 Micro), termasuk konfigurasi firewall, iptables, dan cara menambahkan client (Android/iOS/macOS).

## 📦 1. Buat VM Oracle Cloud

- Shape: VM.Standard.A1.Flex (1 OCPU, 6 GB RAM, gratis di Always Free)
- Image: Ubuntu 24.04 Minimal (aarch64)
- Region: Singapore (atau region lain yang tersedia)

**Security Rules**

Pastikan UDP 51820 terbuka:
- Ingress: 0.0.0.0/0, Protocol UDP, Port 51820
- Egress: 0.0.0.0/0, Protocol All

## ⚙️ 2. Install WireGuard

Login ke VM via SSH

```
sudo apt update && sudo apt install -y wireguard qrencode iptables-persistent
```

## 🔑 3. Generate Keys

```
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

## 📄 4. Konfigurasi Server (/etc/wireguard/wg0.conf)

Buat file wg0.conf lalu masukan config ini

```
[Interface]
PrivateKey = <server_private.key>
Address = 10.8.0.1/24
ListenPort = 51820

# NAT Rules
PostUp   = iptables -A FORWARD -i %i -j ACCEPT; iptables -I FORWARD -o %i -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -A POSTROUTING -o enp0s6 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -m state --state RELATED,ESTABLISHED; iptables -t nat -D POSTROUTING -o enp0s6 -j MASQUERADE
```

> Catatan: ganti enp0s6 sesuai NIC VM (cek dengan ip route).

## 🔧 5. Enable & Start WireGuard

```
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

## 👤 6. Tambah Client Baru

Generate key untuk client

```
cd /etc/wireguard
wg genkey | tee phone_private.key | wg pubkey > phone_public.key
```

Tambahkan ke **server** config (wg0.conf)

```
[Peer]
PublicKey = <phone_public.key>
AllowedIPs = 10.8.0.2/32
```

Restart WireGuard

```
sudo systemctl restart wg-quick@wg0
```

## 📱 7. Buat File Config Client (phone.conf)

Buat file config, bisa store di local atau server, dengan config seperti ini

```
[Interface]
PrivateKey = <phone_private.key>
Address = 10.8.0.2/32
DNS = 8.8.8.8, 1.1.1.1

[Peer]
PublicKey = <server_public.key>
Endpoint = <public_ip_vm>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

## 📷 8. Import ke Device

- Android/iOS: install app WireGuard → import config dari file 
- macOS/Windows: install WireGuard dari App Store → import file .conf.

Atau generate QR dari server dengan command:

```
sudo apt install qrencode
qrencode -t ansiutf8 < phone.conf
```

## 🛠 9. Fix Firewall (iptables)

Kadang ada rule REJECT yang menolak forward traffic. Perbaiki dengan:

```
sudo iptables -I FORWARD 1 -i wg0 -j ACCEPT
sudo iptables -I FORWARD 2 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -t nat -C POSTROUTING -o enp0s6 -j MASQUERADE 2>/dev/null || \
sudo iptables -t nat -A POSTROUTING -o enp0s6 -j MASQUERADE
sudo netfilter-persistent save
```

## ✅ 10. Verifikasi

Cek status wireguard di server:

```
sudo wg
```

Harus muncul latest handshake & transfer.

Cek di browser Client: buka https://ifconfig.me → harus tampil IP Oracle, bukan IP ISP.


# 🎯 Ringkasan
- Deploy VM → buka UDP 51820.
- Install WireGuard → generate keys.
- Konfigurasi wg0.conf + NAT iptables.
- Enable ip_forward & service.
- Tambah client dengan key pair unik.
- Import client config ke Android/iOS/macOS.
- Pastikan NAT & firewall tidak blokir → cek sudo wg + test IP.