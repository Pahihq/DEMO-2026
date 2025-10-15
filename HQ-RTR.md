```bash
 # Настойка времени
 timedatectl set-timezone Europe/Moscow
```

```bash
# Настойка hostname
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
```

1. Настройка сети до ISP
```bash
nmtui
```
<img width="782" height="535" alt="image" src="https://github.com/user-attachments/assets/eae9d9da-969e-47c1-b0c9-d6d7ffc08b07" />

2. Создание пользователей
```bash
 useradd sshuser -u 2026 -U
 passwd sshuser #P@ssw0rd
 useradd net_admin -U
 passwd net_admin #P@ssw0rd
 usermod -aG wheel sshuser
 usermod -aG wheel net_admin
 
# Настроить команду sudo можно в файле /etc/sudoers, в нём хранятся все нужные параметры.
visudo
# Добавить строчки
sshuser ALL=(ALL) NOPASSWD: ALL
net_admin ALL=(ALL) NOPASSWD: ALL
# Esc и :wq
```
<img width="797" height="344" alt="image" src="https://github.com/user-attachments/assets/52cc290f-a12c-4dd9-a764-30ca6e554694" />
<img width="873" height="187" alt="image" src="https://github.com/user-attachments/assets/eeae9a64-3d74-4f3f-b369-88d4a0a38464" />


3. Настройка vlan
Для `100`
```bash
# должна вмещать не более 32 адресов
- Производим настройку
- Выбираем «Изменить подключение»
- Выбираем «Добавить»
- Выбираем «VLAN»
- Задаём понятные имена «Имя профиля» и «Устройство»
- «Родительский» указываем интерфейс в сторону HQ-SRV (ens224)
- задаём «Индефикатор VLAN» (указываем согласно заданию 100)
- переходим к «КОНФИГУРАЦИЯ IPv4»
- задаём **адрес IPv4** для VLAN (192.168.100.1/27)
```

Для `200`
```bash
# должна вмещать не менее 16 адресов
- Производим настройку
- Выбираем «Изменить подключение»
- Выбираем «Добавить»
- Выбираем «VLAN»
- Задаём понятные имена «Имя профиля» и «Устройство»
- «Родительский» указываем интерфейс в сторону HQ-CLI (ens224)
- задаём «Индефикатор VLAN» (указываем согласно заданию 200)
- переходим к «КОНФИГУРАЦИЯ IPv4»
- задаём **адрес IPv4** для VLAN (192.168.200.1/28)
```

Для `999`
```bash
# Должна вмещать не более 8 адресов
- Производим настройку
- Выбираем «Изменить подключение»
- Выбираем «Добавить»
- Выбираем «VLAN»
- Задаём понятные имена «Имя профиля» и «Устройство»
- «Родительский» указываем интерфейс в сторону (ens224)
- задаём «Индефикатор VLAN» (указываем согласно заданию 999)
- переходим к «КОНФИГУРАЦИЯ IPv4»
- задаём **адрес IPv4** для VLAN (192.168.0.1/29)
```

4. Настройка ssh
```bash
nano /etc/selinux/config
# Заменив текст SELINUX=enforcing на SELINUX=permissive.
setenforce 0
```

```bash
nano /etc/ssh/sshd_config
# Находим строчки и меняем если нет то добовляем
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh_banner

systemctl restart sshd
```

5. GRE тунель
```bash
**Производим настройку
- Выбираем «Изменить подключение»
- Выбираем «Добавить»
- Выбираем «IP-туннель
- Задаём понятные имена «Имя профиля» и «Устройство»
- «Режим работы» выбираем «GRE»
- «Родительский» указываем интерфейс в сторону ISP (ens192)
- задаём «Локальный IP» (IP на интерфейсе HQ-RTR в сторону IPS 172.16.1.2)
- задаём «Удалённый IP» (IP на интерфейсе BR-RTR в сторону ISP 172.16.2.2)
- переходим к «КОНФИГУРАЦИЯ IPv4»
- задаём **адрес IPv4** для туннеля (10.10.0.1/30)
# Для корректной работы протокола динамической маршрутизации требуется увеличить параметр TTL на интерфейсе туннеля:
nmcli connection modify tun1 ip-tunnel.ttl 64
```

6. Настройка динамической маршрутизации средствами FRR
```bash
dnf install -y frr
# Для настройки ospf необходимо включить соответствующий демон в конфигурации /etc/frr/daemons
nano /etc/frr/daemons
# Меняем на ospfd = yes
systemctl enable --now frr
```

```bash
vtysh
configure terminal
router ospf
passive-interface default
# Указываем локальные сети HQ и GRE 
network 192.168.0.0/29 area 0
network 192.168.200.0/28 area 0
network 192.168.100.0/27 area 0
network 10.10.0.0/30 area 0

area 0 authentication
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
ip ospf authentication
ip ospf authentication-key P@ssw0rd 
exit
exit
write

systemctl restart frr # Возможно нужно будет reboot
```

```bash
vtysh
show running-config
show ip ospf neighbor
show ip route ospf
```

7. Настройка динамической трансляции адресов
```bash
sysctl -w net.ipv4.ip_forward=1
sysctl net.ipv4.ip_forward >> /etc/sysctl.conf
syscctl -p 

nft add table ip nat
nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule ip nat postrouting oifname "ens192" masquerade
nft list ruleset > /etc/nftables.conf
# Включаем использование данного файла в sysconfig
nano /etc/sysconfig/nftables.conf
include "/etc/nftables/nftables.nft"
systemctl enable --now nftables
```

8. Настройте протокол динамической конфигурации хостов для сети 
```bash 
dnf install dhcp-server -y
cp /usr/share/doc/dhcp-server/dhcpd.conf.example /etc/dhcp/dhcpd.conf

nano /etc/dhcp/dhcpd.conf

subnet 192.168.200.0 netmask 255.255.255.240 {  
range 192.168.200.3 192.168.100.14;  
option domain-name-servers 192.168.200.2;  
option domain-name "au-team.irpo";  
option routers 192.168.200.1;  
default-lease-time 600;  
max-lease-time 7200;  
}

systemctl enable --now dhcpd

```
