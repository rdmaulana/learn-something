# 🛡️ WireGuard + AdGuard Home VPN Setup Guide

## 1. Setup Server (Ubuntu 22.04/24.04)

### Install dependencies
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireguard qrencode iptables-persistent curl
```

### Enable IP forwarding
```bash
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 2. Wireguard Setup Config

### Generate server keys
```bash
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

### Buat file /etc/wireguard/wg0.conf
```bash
[Interface]
PrivateKey = <server_private_key>
Address = 10.8.0.1/24
ListenPort = 51820
PostUp   = iptables -I FORWARD 1 -i %i -j ACCEPT; iptables -I FORWARD 2 -o %i -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -A POSTROUTING -o enp0s6 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -D POSTROUTING -o enp0s6 -j MASQUERADE
```

> 🔹 Ganti enp0s6 dengan nama interface internet (ip route | grep default).

### Start service
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

## 3. Firewall Rules

### Allow WireGuard & DNS
```bash
sudo iptables -I INPUT -i wg0 -p udp --dport 51820 -j ACCEPT
sudo iptables -I INPUT -i wg0 -p udp --dport 53 -j ACCEPT
sudo iptables -I INPUT -i wg0 -p tcp --dport 53 -j ACCEPT
sudo netfilter-persistent save
```

## 4. Install AdGuard Home

### Download & Install
```bash
cd /opt
curl -s -L https://static.adguard.com/adguardhome/release/AdGuardHome_linux_amd64.tar.gz | sudo tar xz
cd AdGuardHome
sudo ./AdGuardHome -s install
```

### Web UI Setup

- Akses: http://10.8.0.1:3000/install.html via VPN.
- Pilih Admin UI: 10.8.0.1:8080
- Pilih DNS Server: 0.0.0.0:53
- Set upstream DNS → 1.1.1.1, 8.8.8.8.
- Buat akun admin.

## 5. WireGuard Client Config

### Contoh config
```bash
[Interface]
PrivateKey = <client_private_key>
Address = 10.8.0.2/32
DNS = 10.8.0.1

[Peer]
PublicKey = <server_public_key>
Endpoint = <server_public_ip>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

### Generate untuk mobile
```bash
qrencode -t ansiutf8 < iphone.conf
```

## 6. esting 
From client:

```bash
ping 8.8.8.8
nslookup google.com 10.8.0.1
```

Di AdGuard Home UI → cek Query Log, harus muncul query dari IP client.

## 7. Extra Tips
- Tambahkan blocklist di Filters → DNS blocklists.
- Gunakan Allowlist untuk domain penting.
- Monitoring di dashboard AdGuard (statistik iklan yang diblokir).
- Simpan semua config di folder /root/wg-clients/.