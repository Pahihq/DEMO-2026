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
