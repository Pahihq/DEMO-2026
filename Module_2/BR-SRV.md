1. Настройка samba
```bash
dnf install samba* krb5* -y
# Добовляем второй днс сервер 
# Первый должен быть HQ-SRV
# Второй должен быть BR-SRV
nmtui
# Редактируем файл nano /etc/systemd/resolved.conf
# Редактируем эту строку
DNSStubListener=no

systemctl restart systemd-resolved.service NetworkManager
rm -f /etc/samba/*

# Приводим его к такому формату nano /etc/krb5.conf
[libdefaults]
    dns_lookup_realm = false
    dns_lookup_kdc = true # Добовляем эту строку
    ticket_lifetime = 24h
#    renew_lifetime = 7d
    forwardable = true
#    rdns = false
#    pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
#    spake_preauth_groups = edwards25519
#    dns_canonicalize_hostname = fallback
#    qualify_shortname = ""
    default_realm = AU-TEAM.IRPO
#    default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 AU-TEAM.IRPO = {
     kdc = br-srv.au-team.irpo
     admin_server = br-srv.au-team.irpo
 }

[domain_realm]
 .au-team.irpo = AU-TEAM.IRPO
 au-team.irpo = AU-TEAM.IRPO

```

```bash
samba-tool domain provision --use-rfc2307 --interactive
Enter
Enter
dc # Вводим
Enter
Enter
# Вводим пароль от домена
systemctl enable --now samba
# Проверка 
samba-tool domain info 127.0.0.1
samba-tool domain info 192.168.250.2
```

2. Добавляем пользователей и группу
```bash
samba-tool group add hq
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd

samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
```

3. Настройка chrony
```bash
dnf install chrony
# Редактируем файл nano /etc/chrony.conf
# Коментируем строчки начинающиеся на server
# Дописываем это
server 172.16.2.1 iburst 

systemctl restart chronyd
systemctl enable --now chronyd

# Проверка
chronyc sources
```

4. Настройка ansible
```bash
dnf install ansible
ssh-keygen
ssh-copy-id -p 2026 sshuser@192.168.100.2 # HQ-SRV
ssh-copy-id net_admin@172.16.1.2
ssh-copy-id net_admin@172.16.2.2
ssh-copy-id user@192.168.200.3 #HQ-CLI



# Дописываем в начало файла nano /etc/ansible/hosts
[hq]
192.168.100.2 ansible_port=2026 ansible_user=sshuser
172.16.1.2 ansible_user=net_admin
192.168.200.3  ansible_user=user
[br]
172.16.2.2 ansible_user=net_admin

# Дописываем в файл nano /etc/ansible/ansible.cfg\
[defaults]
host_key_checking = false
interpreter_python=auto_silent  

ansible all -m ping

```

5. Развертывание веб приложение в docker
```bash
dnf install docker-ce docker-compose
systemctl enable --now docker
mount /dev/sr0 /mnt
cd /mnt/docker

docker load -i site_latest.tar
docker load -i mariadb_latest.tar
# Создайте файл и внесите туда это содержимое
```

```yaml
  testapp:
    container_name: testapp
    image: site:latest
    restart: always
    ports:
      - "8080:8000"
    environment:
      DB_HOST: "192.168.50.2"
      DB_PORT: "3306"
      DB_NAME: mariadb
      DB_USER: maria
      DB_PASS: Passw0rd
      DB_TYPE: maria
    depends_on:
      - db

  db:
    container_name: db
    image: mariadb:latest
    restart: always
    ports:
      - "3306:3306"
    environment:
      DB_USER: maria
      DB_PASS: Passw0rd
      DN_NAME: mariadb
      MARIADB_ROOT_PASSWORD: Passw0rd
```

```bash
docker compose up -d
# Перейдите на сайт 192.168.250.2:8080
# И добовьте 3 записи для проверки и обновите страницу
```
