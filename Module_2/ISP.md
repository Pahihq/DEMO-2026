1. Настройка chrony
```bash
dnf install chrony
# Редактируем файл nano /etc/chrony.conf
# Коментируем строчки начинающиеся на server
# Дописываем это
server 127.0.0.1 iburst prefer
local stratum 5
allow 0/0

systemctl restart chronyd
systemctl enable --now chronyd

# Проверка
chronyc sources # Проверка что работает
chronyc clients # Проверка клиентов

```

2. Настройка обратного прокси
```bash
dnf install -y nginx
systemctl enable --now nginx

# Редактируем файл nano /etc/nginx/conf.d/proxy.conf
server {
    listen 80;
    server_name web.au-team.irpo;
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
    location / { 
	    proxy_pass http://172.16.1.2:8080; 
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;
    location / { proxy_pass http://172.16.2.2:8080; }
}


nginx -t
systemctl restart nginx
```

3. Настройка аутентификации
```bash
dnf install -y httpd-tools
htpasswd -c /etc/nginx/.htpasswd WEB
# Введи пароль: P@ssw0rd
nginx -t
systemctl restart nginx
```