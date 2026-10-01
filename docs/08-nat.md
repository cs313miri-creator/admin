# Задание 8. Динамическая трансляция адресов на маршрутизаторах

**Машины:** HQ-RTR, BR-RTR

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

> **Из задания.** Настройте динамическую трансляцию адресов для обоих офисов в сторону ISP, все устройства в офисах должны иметь доступ к сети Интернет.

NAT нужен в двух местах. Правило на ISP срабатывает только для сетей `172.16.x.x`, а пакет с адресом `192.100.100.2` под него не попадает. Поэтому роутер офиса подменяет адрес первым, и к провайдеру пакет приходит уже «подходящим».

### 8.1. HQ-RTR

```
configure

nat source
  pool HQ
    ip address-range 192.100.100.1-192.100.100.126
    ip address-range 192.100.200.1-192.100.200.62
    ip address-range 192.100.30.1-192.100.30.6
    exit
  ruleset HQ_NAT
    to interface gigabitethernet 1/0/2
    rule 1
      action source-nat pool HQ
      enable
      exit
    exit
  exit

do commit
do confirm
```

### 8.2. BR-RTR

```
configure

nat source
  pool BR
    ip address-range 192.100.20.1-192.100.20.14
    exit
  ruleset BR_NAT
    to interface gigabitethernet 1/0/2
    rule 1
      action source-nat pool BR
      enable
      exit
    exit
  exit

do commit
do confirm
```

| Команда                                  | Что делает                                                                        |
|------------------------------------------|-----------------------------------------------------------------------------------|
| `nat source`                             | раздел трансляции адреса источника                                                |
| `pool HQ`                                | именованный набор адресов; диапазоны — от `HostMin` до `HostMax` каждой подсети   |
| `ruleset HQ_NAT`                         | набор правил                                                                      |
| `to interface gigabitethernet 1/0/2`     | применять только к трафику, уходящему к провайдеру                                |
| `action source-nat pool HQ`              | транслировать адрес источника                                                     |
| `enable`                                 | включить правило — без этого оно не работает                                      |

Перечислены **все три** VLAN: интернет нужен и серверам, и клиентам, и сети управления.

Ограничение `to interface` важно принципиально: оно не даёт NAT примениться к трафику в туннель. Иначе соседний офис увидел бы пакеты с транслированным адресом, и связь между офисами сломалась бы.

Если трансляция не заработает, есть вторая форма, не зависящая от трактовки пула:

```
configure

object-group network HQ_LAN
  ip address-range 192.100.100.0/25
  ip address-range 192.100.200.0/26
  ip address-range 192.100.30.0/29
  exit

nat source
  ruleset SNAT_HQ
    to interface gigabitethernet 1/0/2
    rule 1
      match source-address HQ_LAN
      action source-nat interface
      enable
      exit
    exit
  exit

do commit
do confirm
```

### Проверка задания 8

```
show nat source ruleset HQ_NAT
show ip nat translations
```

```bash
# с HQ-SRV и с HQ-CLI
ping -c3 8.8.8.8
# с BR-SRV
ping -c3 8.8.8.8
```

Дополнительно проверь, что связь между офисами не сломалась: с BR-SRV `ping 192.100.100.2` должен идти через туннель, без трансляции.

---

---

[← Задание 7](07-ospf.md) · [Задание 9 →](09-dns.md) · [Ошибки](troubleshooting.md)
