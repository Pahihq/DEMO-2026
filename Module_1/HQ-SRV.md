```bash
 # Настойка времени
 timedatectl set-timezone Europe/Moscow
```

```bash
# Настойка hostname
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
```

1. Настройка сети
```bash
# Возможно настройка в интерфейсе proxmox
# должна вмещать не более 32 адресов
- Производим настройку
- Выбираем «Изменить подключение»
- Выбираем «Добавить»
- Выбираем «VLAN»
- Задаём понятные имена «Имя профиля» и «Устройство»
- «Родительский» указываем интерфейс в сторону HQ-RTR (ens224)
- задаём «Индефикатор VLAN» (указываем согласно заданию 100)
- переходим к «КОНФИГУРАЦИЯ IPv4»
- задаём **адрес IPv4** для VLAN (192.168.100.5/27)
```
<img width="1140" height="901" alt="image" src="https://github.com/user-attachments/assets/7a2e42db-ad7c-4352-ad73-627de7d9cd19" />

2. Создание пользователей
```bash
 useradd sshuser -u 2026 -G wheel -U
 passwd sshuser #P@ssw0rd

 
# Настроить команду sudo можно в файле /etc/sudoers, в нём хранятся все нужные параметры.
visudo
# Добавить строчки
sshuser ALL=(ALL) NOPASSWD: ALL
# Esc и :wq
# Альтернатива для удобства mcedit /etc/sudoers

```
<img width="797" height="344" alt="image" src="https://github.com/user-attachments/assets/52cc290f-a12c-4dd9-a764-30ca6e554694" />
<img width="873" height="187" alt="image" src="https://github.com/user-attachments/assets/eeae9a64-3d74-4f3f-b369-88d4a0a38464" />

3. Настройка ssh
```bash
nano /etc/selinux/config
# Заменив текст SELINUX=enforcing на SELINUX=permissive.
setenforce 0
```
<img width="798" height="119" alt="image" src="https://github.com/user-attachments/assets/7d328fae-c075-4680-bb35-5c7e965d865d" />


```bash
nano /etc/ssh/sshd_config
# Находим строчки и меняем если нет то добовляем
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh_banner

systemctl restart sshd
```
<img width="362" height="443" alt="image" src="https://github.com/user-attachments/assets/4d69f30c-c425-4453-be06-20620866bf00" />
<img width="252" height="60" alt="image" src="https://github.com/user-attachments/assets/98fb6f00-77fb-44f5-ab13-63d37c838101" />

4. Настройка bind 
```bash
dnf install bind bind-utils
nano /etc/named.conf
# В данном файле необходимо изменить следующие строки, содержащие
listen-on port 53;  
listen-on-v6 port 53;  
allow-query;  
forwarders ; (_дописать строку)_  
dnssec-validation (заменить на none)
```
На
```bash
listen-on port 53 { any; };
listen-on-v6 port 53 { none;};
allow-query     { any; };
forwarders	{77.88.8.8;};
dnssec-validation no;
```

```bash
# Объявляем файлы зон, дописываем в конец файла
zone "au-team.irpo" {
	type master;
	file "master/au-team.db";
};

zone "100.168.192.in-addr.arpa" {
	type master;
	file "master/au-team_rev1.db";
};

zone "200.168.192.in-addr.arpa" {
	type master;
	file "master/au-team_rev2.db";
};

#Проверяем конфигурации 
named-checkconf
```
<img width="871" height="346" alt="image" src="https://github.com/user-attachments/assets/f6b12fe1-d8f3-4943-9f07-ae6696cfcf96" />
<img width="734" height="494" alt="image" src="https://github.com/user-attachments/assets/843d9636-c446-4e26-82f9-bbcba65f3dc6" />



5. Создание локальных зон DNS
```bash
mkdir /var/named/master
cp /var/named/named.localhost /var/named/master/au-team.db
# Редактируем его 
nano /var/named/master/au-team.db
# Приводим к такому виду
$TTL 1D
@	IN SOA	@ rname.invalid. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum

	IN	NS	au-team.irpo.
	IN	A	192.168.100.5
hq-rtr	IN	A	192.168.200.1
hq-rtr	IN	A	192.168.100.1
hq-rtr	IN	A	192.168.0.1
br-rtr	IN	A	192.168.250.1
hq-srv	IN	A	192.168.200.5
hq-cli	IN	A	192.168.200.2
br-srv	IN	A	192.168.250.2
docker  IN  A   172.16.2.1     
web     IN  A   172.16.1.1 
```

```bash
cp /var/named/named.loopback /var/named/master/au-team_rev1.db
# Редактируем его 
nano /var/named/master/au-team_rev1.db
# Приводим к такому виду
$TTL 1D
@	IN SOA	@ rname.invalid. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
	IN	NS	au-team.irpo.
1	IN	PTR	hq-rtr.au-team.irpo.
2	IN	PTR	hq-srv.au-team.irpo.
```

```bash
cp /var/named/master/au-team_rev1.db /var/named/master/au-team_rev2.db
# Редактируем его 
nano /var/named/master/au-team_rev2.db
# Приводим к такому виду
$TTL 1D
@	IN SOA	@ rname.invalid. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
	IN	NS	au-team.irpo.
1	IN	PTR	hq-rtr.au-team.irpo.
2	IN	PTR	hq-cli.au-team.irpo.
```

```bash
chown -R root:named /var/named/master
chmod 0640 /var/named/master/*
named-checkconf -z
#Такой вывод нормален
zone au-team.irpo/IN: loaded serial 0
zone 100.168.192.in-addr.arpa/IN: loaded serial 0
zone 200.168.192.in-addr.arpa/IN: loaded serial 0
zone localhost.localdomain/IN: loaded serial 0
zone localhost/IN: loaded serial 0
zone 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa/IN: loaded serial 0
zone 1.0.0.127.in-addr.arpa/IN: loaded serial 0
zone 0.in-addr.arpa/IN: loaded serial 0
```

6. Настройка dns 
```bash
# Прописываем собсвенный ip в днс
nmtui
```
7. Запускаем сервис
```bash
systemctl enable --now named
systemctl status named
```
