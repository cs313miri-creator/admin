# Задание 4. Коммутация в сегменте HQ

**Машины:** HQ-SW, HQ-RTR

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

> **Из задания.** Трафик HQ-SRV — VLAN 150, трафик HQ-CLI — VLAN 250, предусмотреть передачу трафика управления в VLAN 333. Реализовать на HQ-RTR маршрутизацию всех указанных VLAN с использованием одного сетевого адаптера.

Схема называется **router-on-a-stick**: к роутеру идёт один транковый порт, а на нём создаются подынтерфейсы — по одному на VLAN. Половина конструкции уже сделана в задании 1 (подынтерфейсы `gi1/0/3.150/.250/.333`), здесь — вторая половина, на коммутаторе.

**Машина: HQ-SW (103), Альт Сервер**

### 4.1. Установить Open vSwitch

```bash
apt-get update
apt-get install -y openvswitch
systemctl enable --now openvswitch
```

> **У HQ-SW нет сети, пакет скачать неоткуда.** Коммутатору мы адрес не настраивали — и правильно, он работает на втором уровне. Но `apt-get` из-за этого не достучится до репозитория.
>
> Решение — временный выход в сеть. В Proxmox: **HQ-SW → Hardware**, добавить адаптер на мост `vmbr0` (или снять с существующего галочку Disconnect). Затем:
>
> ```bash
> ip -br link                      # найти имя нового интерфейса
> mkdir -p /etc/net/ifaces/ens22   # подставить своё имя
> cat > /etc/net/ifaces/ens22/options <<'EOF'
> BOOTPROTO=dhcp
> TYPE=eth
> CONFIG_WIRELESS=no
> SYSTEMD_BOOTPROTO=dhcp4
> CONFIG_IPV4=yes
> DISABLED=no
> NM_CONTROLLED=no
> SYSTEMD_CONTROLLED=no
> EOF
> systemctl restart network
> apt-get update && apt-get install -y openvswitch
> ```
>
> После установки адаптер лучше отключить, чтобы на проверке коммутатор был без адресов. Запасной вариант без сети — подключить ISO Альт Сервера и выполнить `apt-repo add cdrom`.

### 4.2. Создать коммутатор и настроить порты

```bash
ovs-vsctl add-br SW

# транк к роутеру — несёт три VLAN с тегами
ovs-vsctl add-port SW ens20 trunk=150,250,333

# access к HQ-SRV — всё входящее помечается тегом 150
ovs-vsctl add-port SW ens19 tag=150

# access к HQ-CLI — тег 250
ovs-vsctl add-port SW ens21 tag=250

ovs-vsctl show
```

| Команда                        | Назначение                                                                 |
|--------------------------------|----------------------------------------------------------------------------|
| `add-br SW`                    | создать виртуальный коммутатор с именем `SW`                               |
| `add-port SW ens20 trunk=...`  | транковый порт: кадры уходят **с тегами**, роутер по ним различает сети     |
| `add-port SW ens19 tag=150`    | access-порт: входящий кадр получает тег, исходящему тег снимается           |

Конфигурация OVS хранится в собственной базе и переживает перезагрузку. Имена интерфейсов обязательно сверь с `ip -br link` — распределение портов на твоём стенде может отличаться.

### 4.3. Поднять интерфейсы и закрепить

```bash
ip link set ens19 up
ip link set ens20 up
ip link set ens21 up

for i in ens19 ens20 ens21; do
  mkdir -p /etc/net/ifaces/$i
  cat > /etc/net/ifaces/$i/options <<'EOF'
BOOTPROTO=static
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=static
CONFIG_IPV4=no
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
EOF
done

systemctl restart network
```

`CONFIG_IPV4=no` — адрес на этих интерфейсах не нужен, но подниматься при загрузке они обязаны: через опущенный порт кадры не пойдут.

### Проверка задания 4

```bash
# на HQ-SW
ovs-vsctl show       # у ens20 trunk: [150, 250, 333], у ens19 tag: 150, у ens21 tag: 250
ip -br link          # все три UP
```

```bash
# на HQ-SRV
ping -c3 192.100.100.1      # свой шлюз
ping -c3 192.100.200.2      # HQ-CLI в другом VLAN — через роутер
```

Второй пинг проверяет сразу всё: тегирование на коммутаторе, подынтерфейсы на роутере и маршрутизацию между ними. Если он идёт — задание закрыто.

---

---

[← Задание 3](03-users.md) · [Задание 5 →](05-ssh.md) · [Ошибки](troubleshooting.md)
