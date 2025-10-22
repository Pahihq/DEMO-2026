1. Настройка chrony
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

2. Настройка статическую трансляцию портов
```bash
nft add chain ip nat prerouting { type nat hook prerouting priority dstnat\;}
nft add rule ip nat prerouting tcp dport 8080 dnat to 192.168.250.2:8080
nft add rule ip nat prerouting tcp dport 2026 dnat to 192.168.250.2:2026
nft list ruleset > /etc/nftables/nft