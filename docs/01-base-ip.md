# Задание 1. Базовая настройка устройств и IPv4-адресация

**Машины:** ISP, HQ-RTR, BR-RTR, HQ-SRV, HQ-CLI, BR-SRV, BR-CLI, HQ-SW

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

> **Из задания.** Настройте имена устройств согласно топологии, используйте полное доменное имя. На всех устройствах сконфигурируйте IPv4 по размерам сетей, указанным в условии.

### 1.1. ISP — имя и адреса

**Машина: ISP (200), Альт Сервер**

```bash
# имя хоста
hostnamectl set-hostname ISP.au-team.irpo; exec bash

# ens19 — в сторону магистрального провайдера, адрес по DHCP
mkdir -p /etc/net/ifaces/ens19
cat > /etc/net/ifaces/ens19/options <<'EOF'
BOOTPROTO=dhcp
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=dhcp4
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF

# ens20 — в сторону HQ-RTR
mkdir -p /etc/net/ifaces/ens20
cat > /etc/net/ifaces/ens20/options <<'EOF'
BOOTPROTO=static
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
echo "172.16.1.1/28" > /etc/net/ifaces/ens20/ipv4address

# ens21 — в сторону BR-RTR
mkdir -p /etc/net/ifaces/ens21
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens21/options
echo "172.16.2.1/28" > /etc/net/ifaces/ens21/ipv4address

systemctl restart network
```

Разбор параметров файла `options`:

| Параметр                    | Значение                                                      |
|-----------------------------|---------------------------------------------------------------|
| `BOOTPROTO=static\|dhcp`    | откуда берётся адрес                                          |
| `TYPE=eth`                  | обычный Ethernet-интерфейс                                    |
| `CONFIG_IPV4=yes`           | включить IPv4                                                 |
| `DISABLED=no`               | поднимать интерфейс при загрузке                              |
| `NM_CONTROLLED=no`          | NetworkManager этот интерфейс не трогает                      |
| `SYSTEMD_CONTROLLED=no`     | systemd-networkd тоже не трогает — управляет только etcnet    |

**Проверка:**

```bash
hostname -f          # ISP.au-team.irpo
ip -br a             # ens19 — адрес от DHCP, ens20 — 172.16.1.1/28, ens21 — 172.16.2.1/28
ip route             # default via ... dev ens19
```

Маршрут по умолчанию ISP получает по DHCP от магистрального провайдера, вручную его задавать не нужно.

### 1.2. HQ-RTR — имя, стык с ISP, подынтерфейсы VLAN

**Машина: HQ-RTR (108), Eltex vESR**

```
configure

hostname HQ-RTR

! стык с провайдером
int gi1/0/2
  ip firewall disable
  ip address 172.16.1.2/28
  exit

! подынтерфейсы VLAN в сторону HQ-SW
interface gi1/0/3.150
  description "Vlan150"
  ip firewall disable
  ip address 192.100.100.1/25
  exit

interface gi1/0/3.250
  description "Vlan250"
  ip firewall disable
  ip address 192.100.200.1/26
  exit

interface gi1/0/3.333
  description "Vlan333"
  ip firewall disable
  ip address 192.100.30.1/29
  exit

! маршрут по умолчанию к провайдеру
ip route 0.0.0.0/0 172.16.1.1

do commit
do confirm
exit
```

| Команда                        | Что делает                                                                            |
|--------------------------------|---------------------------------------------------------------------------------------|
| `int gi1/0/2`                  | физический порт; полная форма — `interface gigabitethernet 1/0/2`                     |
| `ip firewall disable`          | отключить обработку интерфейса межсетевым экраном                                      |
| `ip address 172.16.1.2/28`     | адрес с префиксом одной строкой                                                        |
| `interface gi1/0/3.150`        | подынтерфейс; число после точки задаёт VLAN, отдельная команда `encapsulation` не нужна |
| `ip route 0.0.0.0/0 172.16.1.1`| «всё, чего не знаю, отдавай провайдеру»                                                |

**Проверка:**

```
show ip interfaces
show int status
show ip route
ping 172.16.1.1
```

Ожидаем на `gi1/0/2` и `gi1/0/3` состояние **Up/Up**, пять строк в таблице маршрутов и успешный пинг провайдера.

### 1.3. BR-RTR — имя и адреса

**Машина: BR-RTR (109), Eltex vESR**

```
configure

hostname BR-RTR

int gi1/0/2
  description "ISP"
  ip firewall disable
  ip address 172.16.2.2/28
  exit

int gi1/0/3
  description "LAN"
  ip firewall disable
  ip address 192.100.20.1/28
  exit

ip route 0.0.0.0/0 172.16.2.1

do commit
do confirm
exit
```

В филиале одна сеть, поэтому подынтерфейсы не нужны — адрес вешается прямо на порт.

**Проверка:**

```
show ip interfaces
show ip route
ping 172.16.2.1
```

### 1.4. Серверы и клиенты

**Машины: HQ-SRV (105), HQ-CLI (107), BR-SRV (106), BR-CLI (110)**

На рабочих станциях (HQ-CLI, BR-CLI) сначала убираем NetworkManager:

```bash
systemctl disable --now NetworkManager
systemctl enable --now network
```

Дальше шаблон одинаковый, меняются имя, адрес и шлюз:

```bash
# ===== HQ-SRV =====
hostnamectl set-hostname HQ-SRV.au-team.irpo; exec bash
mkdir -p /etc/net/ifaces/ens19
cat > /etc/net/ifaces/ens19/options <<'EOF'
BOOTPROTO=static
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
echo "192.100.100.2/25" > /etc/net/ifaces/ens19/ipv4address
echo "default via 192.100.100.1" > /etc/net/ifaces/ens19/ipv4route
systemctl restart network
```

Остальные машины — тот же блок с заменой:

| Машина  | Имя хоста                | `ipv4address`       | `ipv4route`                 |
|---------|--------------------------|---------------------|-----------------------------|
| HQ-SRV  | `HQ-SRV.au-team.irpo`    | `192.100.100.2/25`  | `default via 192.100.100.1` |
| HQ-CLI  | `HQ-CLI.au-team.irpo`    | `192.100.200.2/26`  | `default via 192.100.200.1` |
| BR-SRV  | `BR-SRV.au-team.irpo`    | `192.100.20.2/28`   | `default via 192.100.20.1`  |
| BR-CLI  | `BR-CLI.au-team.irpo`    | `192.100.20.3/28`   | `default via 192.100.20.1`  |

### 1.5. HQ-SW — только имя

**Машина: HQ-SW (103), Альт Сервер**

```bash
hostnamectl set-hostname HQ-SW.au-team.irpo; exec bash
```

Коммутатор работает на втором уровне, IP-адрес ему не нужен. Его настройка — в задании 4.

### Проверка задания 1

Работает уже сейчас:

| Откуда  | Куда            | Что проверяет            |
|---------|-----------------|--------------------------|
| HQ-RTR  | `172.16.1.1`    | стык с провайдером       |
| BR-RTR  | `172.16.2.1`    | стык с провайдером       |
| ISP     | `172.16.1.2`    | HQ-RTR виден провайдеру  |
| ISP     | `172.16.2.2`    | BR-RTR виден провайдеру  |
| BR-SRV  | `192.100.20.1`  | шлюз филиала             |

Пока **не должно** работать: HQ-SRV и HQ-CLI не видят свой шлюз — между ними и роутером стоит HQ-SW без настроенных VLAN, это задание 4.

---

---

[← README](../README.md) · [Задание 2 →](02-isp-nat.md) · [Ошибки](troubleshooting.md)
