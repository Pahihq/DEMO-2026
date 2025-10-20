```bash
 # Настойка времени
 timedatectl set-timezone Europe/Moscow
```

```bash
# Настойка hostname
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
```

1. Настройка сети
```bash
# В сторону клиента не более 16 адресов /28
nmtui
```
<img width="840" height="581" alt="image" src="https://github.com/user-attachments/assets/6d0a327d-ce5c-4664-9924-a87a6b391b7a" />
<img width="831" height="281" alt="image" src="https://github.com/user-attachments/assets/8021c791-d237-4502-b277-282bc00113cf" />

2. GRE тунель
```bash
**Производим настройку
- Выбираем «Изменить подключение»
- Выбираем «Добавить»
- Выбираем «IP-туннель
- Задаём понятные имена «Имя профиля» и «Устройство»
- «Режим работы» выбираем «GRE»
- «Родительский» указываем интерфейс в сторону ISP (ens192)
- задаём «Локальный IP» (IP на интерфейсе BR-RTR в сторону IPS 172.16.2.2)
- задаём «Удалённый IP» (IP на интерфейсе HQ-RTR в сторону ISP 172.16.1.2)
- переходим к «КОНФИГУРАЦИЯ IPv4»
- задаём **адрес IPv4** для туннеля (10.10.0.2/30)
# Для корректной работы протокола динамической маршрутизации требуется увеличить параметр TTL на интерфейсе туннеля:
nmcli connection modify tun1 ip-tunnel.ttl 64
```
<img width="774" height="701" alt="image" src="https://github.com/user-attachments/assets/e024f36e-c133-4a76-b48f-218f06aa2f8e" />

2. Создание пользователей
```bash
 useradd net_admin -U
 passwd net_admin #P@ssw0rd
 usermod -aG wheel net_admin
 
# Настроить команду sudo можно в файле /etc/sudoers, в нём хранятся все нужные параметры.
visudo
# Добавить строчки
net_admin ALL=(ALL) NOPASSWD: ALL
# Esc и :wq
# Альтернатива для удобства mcedit /etc/sudoers

```
<img width="797" height="344" alt="image" src="https://github.com/user-attachments/assets/52cc290f-a12c-4dd9-a764-30ca6e554694" />
<img width="873" height="187" alt="image" src="https://github.com/user-attachments/assets/eeae9a64-3d74-4f3f-b369-88d4a0a38464" />

3. Настройка динамической маршрутизации средствами FRR
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
# Указываем локальные сети BR и GRE 
network 192.168.250.0/28 area 0
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

4. Настройка динамической трансляции адресов
```bash
# Может после перезагрузки отваливаться достаточно заново применить
sysctl -w net.ipv4.ip_forward=1
sysctl net.ipv4.ip_forward >> /etc/sysctl.conf
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
