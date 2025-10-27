1. Настройка bind
```bash
# Дописываем строчку nano /etc/named.conf
allow-transfer { 192.168.250.2; };
systemctl restart named
```

2. Создание raid
```bash
dnf install mdadm -y
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail --scan --verbose >> /etc/mdadm.conf
fdisk /dev/md0
# Вводим 
g
n
1
g
w

mkfs.ext4 /dev/md0
mkdir /raid

# Добовляем строчку nano /etc/fstab 
/dev/md0 /raid ext4 defaults 0 0

systemctl daemon-reload
mount -a
```

3. Создание nfs хранилища
```bash
dnf install nfs-utils 
mkdir /raid/nfs
chmod -R 777 /raid/nfs
# Добовляем строчку nano /etc/exports
/raid/nfs 192.168.200.0/28(rw)

systemctl enable --now nfs-server
exportfs -ra
```

4. Настройка chrony
```bash
dnf install chrony
# Редактируем файл nano /etc/chrony.conf
# Коментируем строчки начинающиеся на server
# Дописываем это
server 172.16.1.1 iburst 

systemctl restart chronyd
systemctl enable --now chronyd

# Проверка
chronyc sources
```

5. Настройка веб приложения
```bash

mount /dev/sr0 /mnt

dnf install -y httpd mariadb-server php php-mysqlnd
systemctl enable --now httpd mariadb

mariadb
#Создаем бд
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
FLUSH PRIVILEGES;
EXIT;

mariadb webdb < /mnt/web/dump.sql
cp /mnt/web/index.php /var/www/html/
cp /mnt/web/logo.png /var/www/html/
cp -r /mnt/web/images /var/www/html/

chmod -R 755 /var/www/html

# Редактируем файл nano /var/www/html/index.php на нужные данные
$host = "localhost";
$user = "web";
$pass = "P@ssw0rd";
$dbname = "webdb";

systemctl restart httpd

# Открой браузер http://192.168.100.2/


```
