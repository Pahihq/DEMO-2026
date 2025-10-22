1. Ввод клиента в домен
```bash
# Добовляем DNS ip BR-SRV
nmtui

sudo join-to-domain.sh
y
1
Enter
hq-cli
Administrator
Enter
# Вводим пароль от админитратора
y

# Перезагружаем и входим под доменным пользователем

# Создаем файл nano /etc/sudoers.d/hq
%hq ALL=(ALL) /usr/bin/cat, /usr/bin/grep, /usr/usr/bin/id
```

2. Автомонтирование nfs
```bash
dnf install nfs-utils 
mkdir /mnt/nfs

# Добовляем строчку nano /etc/fstab 
192.168.100.2:/raid/nfs /mnt/nfs nfs auto 0 0

systemctl daemon-reload
mount -a
```

3. Настройка chrony
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

4. Установка браузер
```bash
dnf install yandex-browser
```