```bash
# На всех хостах
 timedatectl set-timezone Europe/Moscow
```

1. Настройка ip на isp
```bash
nmtui
```

2. Настройка форвардинга
```bash
sysctl -w net.ipv4.ip_forward=1
sysctl net.ipv4.ip_forward >> /etc/sysctl.conf
syscctl -p 
```

3. Настройка nftables
```bash
nft add table ip nat
nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule ip nat postrouting oifname "ens192" masquerade
nft list ruleset > /etc/nftables.conf
# Включаем использование данного файла в sysconfig
nano /etc/sysconfig/nftables.conf
include "/etc/nftables/nftables.nft"
systemctl enable --now nftables
```