```bash
 # Настойка времени
 timedatectl set-timezone Europe/Moscow
```

```bash
# Настойка hostname
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
```

1. Настройка сети 
```bash
# Прописываем ip hq-srv в днс
nmtui
```

2. Создание пользователей
```bash
 useradd sshuser -u 2026 -U
 passwd sshuser #P@ssw0rd
 usermod -aG wheel sshuser
 
# Настроить команду sudo можно в файле /etc/sudoers, в нём хранятся все нужные параметры.
visudo
# Добавить строчки
sshuser ALL=(ALL) NOPASSWD: ALL
# Esc и :wq
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
