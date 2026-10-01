# Задание 2. Доступ к сети Интернет на ISP

**Машины:** ISP

> **Контекст стенда** (нужен, чтобы команды ниже были корректны)
>
> Proxmox VE. Маршрутизаторы — **Eltex vESR 1.18**, остальное — **Альт Линукс**.
> Порты роутеров сдвинуты относительно топологии задания: **`gi1/0/2` — к ISP**, **`gi1/0/3` — внутрь офиса**
> (`net0`→`gi1/0/1` не используется, `net1`→`gi1/0/2`, `net2`→`gi1/0/3`).
> На vESR: применение — `do commit`, затем `do confirm`; на каждом интерфейсе и туннеле — `ip firewall disable`;
> удаление — повтор строки целиком с `no`. Просмотр: `show ip interfaces`, `show int status`, `show ip route`.
> В Альт: сеть в `/etc/net/ifaces/<iface>/`, применение `systemctl restart network`, пакеты через `apt-get`.
>
> Адресация: ISP `172.16.1.1/28` и `172.16.2.1/28` · HQ-RTR `172.16.1.2/28`, `192.100.100.1/25`, `192.100.200.1/26`, `192.100.30.1/29`
> · BR-RTR `172.16.2.2/28`, `192.100.20.1/28` · туннель `10.10.10.1/30` ↔ `10.10.10.2/30`
> · HQ-SRV `192.100.100.2` · HQ-CLI `192.100.200.2` · BR-SRV `192.100.20.2` · BR-CLI `192.100.20.3`
>
> Полный контекст — в [README](../README.md).

---

> **Из задания.** Настройте динамическую сетевую трансляцию портов для доступа к сети Интернет HQ-RTR и BR-RTR.

**Машина: ISP (200), Альт Сервер**

### 2.1. Включить пересылку пакетов

```bash
sed -i 's/^net.ipv4.ip_forward.*/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf
grep ip_forward /etc/net/sysctl.conf

systemctl restart network
sysctl net.ipv4.ip_forward
```

По умолчанию Linux ведёт себя как конечный хост и выбрасывает транзитные пакеты. Нужная строка в файле уже есть со значением `0` — её меняют, а не добавляют.

### 2.2. Настроить NAT и разрешить транзит

```bash
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens19 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens19 -j MASQUERADE

iptables -A FORWARD -i ens20 -o ens19 -j ACCEPT
iptables -A FORWARD -i ens21 -o ens19 -j ACCEPT
iptables -A FORWARD -i ens19 -o ens20 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i ens19 -o ens21 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

| Параметр                              | Назначение                                                        |
|---------------------------------------|-------------------------------------------------------------------|
| `-t nat`                              | таблица подмены адресов                                           |
| `-A POSTROUTING`                      | точка «пакет отмаршрутизирован и сейчас уйдёт»                     |
| `-s 172.16.1.0/28`                    | только трафик от этой сети                                        |
| `-o ens19`                            | только уходящий через внешний интерфейс                            |
| `-j MASQUERADE`                       | подменить адрес источника на адрес выходного интерфейса            |
| `-m state --state RELATED,ESTABLISHED`| обратно пропускать только ответы на начатые изнутри соединения      |

Пишется строго `MASQUERADE` — регистр и порядок букв важны.

### 2.3. Сохранить правила

```bash
apt-get install -y iptables-service
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```

Без сохранения правила исчезнут при перезагрузке.

### Проверка задания 2

```bash
# на ISP
iptables -t nat -L -n -v      # две строки MASQUERADE
iptables -L FORWARD -n -v     # четыре строки ACCEPT
ping -c2 8.8.8.8
```

```
# на HQ-RTR и BR-RTR
ping 8.8.8.8
```

С роутеров интернет должен пойти. С серверов — пока нет, для этого нужен NAT из задания 8.

---

---

[← Задание 1](01-base-ip.md) · [Задание 3 →](03-users.md) · [Ошибки](troubleshooting.md)
