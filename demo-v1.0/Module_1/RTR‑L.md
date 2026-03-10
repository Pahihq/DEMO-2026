```bash
 # Настойка времени
 timedatectl set-timezone Europe/Moscow
```

```bash
# Настойка hostname
hostnamectl set-hostname rtr-l.au-team.irpo; exec bash
```

1. Настройка сети до ISP
```bash
nmtui
```
<img width="782" height="535" alt="image" src="https://github.com/user-attachments/assets/eae9d9da-969e-47c1-b0c9-d6d7ffc08b07" />

2. Настройка vlan
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
<img width="800" height="640" alt="image" src="https://github.com/user-attachments/assets/ab679501-b28b-4a96-b3e5-d241c528b04e" />

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
<img width="797" height="626" alt="image" src="https://github.com/user-attachments/assets/1b04912a-4126-4d06-919c-8178415896e0" />

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
<img width="800" height="629" alt="image" src="https://github.com/user-attachments/assets/7dd6bb7e-70a9-47ac-add4-3d693833a448" />

Проверка:

<img width="750" height="185" alt="image" src="https://github.com/user-attachments/assets/377f34ad-dadd-43c6-bafc-bc40244543e1" />

4. GRE тунель
```bash
**Производим настройку
- Выбираем «Изменить подключение»
- Выбираем «Добавить»
- Выбираем «IP-туннель
- Задаём понятные имена «Имя профиля» и «Устройство»
- «Режим работы» выбираем «GRE»
- «Родительский» указываем интерфейс в сторону ISP (ens192)
- задаём «Локальный IP» (IP на интерфейсе HQ-RTR в сторону IPS 172.16.221.2)
- задаём «Удалённый IP» (IP на интерфейсе BR-RTR в сторону ISP 172.16.222.2)
- переходим к «КОНФИГУРАЦИЯ IPv4»
- задаём **адрес IPv4** для туннеля (10.10.0.1/30)
# Для корректной работы протокола динамической маршрутизации требуется увеличить параметр TTL на интерфейсе туннеля:
nmcli connection modify tun1 ip-tunnel.ttl 64
```
<img width="745" height="395" alt="image" src="https://github.com/user-attachments/assets/1bd8fa55-fab1-4a3c-99ee-606f9c1c6d01" />


5. Настройка динамической маршрутизации средствами FRR
```bash
dnf install -y frr
# Для настройки ospf необходимо включить соответствующий демон в конфигурации /etc/frr/daemons
nano /etc/frr/daemons
# Меняем на ospfd = yes
systemctl enable --now frr
```
<img width="197" height="301" alt="image" src="https://github.com/user-attachments/assets/d3608361-d488-4c94-a92c-fff3fe440267" />


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

6. Настройка динамической трансляции адресов
```bash
# Может после перезагрузки отваливаться достаточно заново применить
# Дописать строку файл nano /etc/sysctl.conf
net.ipv4.ip_forward = 1 
syscctl -p 

nft add table ip nat
nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule ip nat postrouting oifname "ens192" masquerade
nft list ruleset > /etc/nftables.conf
# Включаем использование данного файла в sysconfig
nano /etc/sysconfig/nftables.conf
include "/etc/nftables.conf"
systemctl enable --now nftables
```

7. Настройте протокол динамической конфигурации хостов для сети 
```bash 
dnf install dhcp-server -y
cp /usr/share/doc/dhcp-server/dhcpd.conf.example /etc/dhcp/dhcpd.conf

nano /etc/dhcp/dhcpd.conf

subnet 192.168.200.0 netmask 255.255.255.240 {  
range 192.168.200.2 192.168.200.14;  
option domain-name-servers 192.168.100.2;  
option domain-name "au-team.irpo";  
option routers 192.168.200.1;  
default-lease-time 600;  
max-lease-time 7200;  
}

systemctl enable --now dhcpd

```
