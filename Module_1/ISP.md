```bash
 # Настойка времени
 timedatectl set-timezone Europe/Moscow
```

```bash
# Настойка hostname
hostnamectl set-hostname isp.au-team.irpo; exec bash
```

1. Настройка ip на isp
```bash
# Настраиваем WAN на DHCP
# Настраиваем в сторону HQ 172.16.1.0/28
# Настраиваем в сторону BR 172.16.2.0/28
nmtui
```
<img width="787" height="536" alt="image" src="https://github.com/user-attachments/assets/3b797a92-0487-41cc-877c-18aca2d72097" />
<img width="788" height="526" alt="image" src="https://github.com/user-attachments/assets/2e89fb19-85c1-455c-aeec-0e7ce61aa324" />



2. Настройка форвардинга
```bash
# Может после перезагрузки отваливаться достаточно заново применить
# Дописать строку файл /etc/sysctl.conf
net.ipv4.ip_forward = 1 
sysctl -p 
```
<img width="723" height="299" alt="image" src="https://github.com/user-attachments/assets/2d34f196-c7e3-4883-9f42-0e7bad22d1ce" />


3. Настройка nftables
```bash
nft add table ip nat
nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule ip nat postrouting oifname "ens192" masquerade
nft list ruleset > /etc/nftables.conf
# Включаем использование данного файла в sysconfig
nano /etc/sysconfig/nftables.conf
include "/etc/nftables.conf"
systemctl enable --now nftables
```
<img width="868" height="177" alt="image" src="https://github.com/user-attachments/assets/83e8906a-6601-43e4-91d3-18ad65dbf71a" />

