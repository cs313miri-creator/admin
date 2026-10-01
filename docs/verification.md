# Итоговая проверка

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

| Задание | Команда проверки                                        | Где выполнять        |
|---------|---------------------------------------------------------|----------------------|
| 1       | `ping 172.16.1.1` / `ping 172.16.2.1`                   | HQ-RTR / BR-RTR      |
| 2       | `ping 8.8.8.8`                                          | HQ-RTR, BR-RTR       |
| 3       | `id sshcli` → uid=2525; `sudo -n id` → uid=0            | HQ-SRV, BR-SRV       |
| 4       | `ping 192.100.200.2`                                    | HQ-SRV               |
| 5       | `ssh sshcli@192.100.100.2` → баннер, 2 попытки          | HQ-CLI               |
| 6       | `ping 10.10.10.2`                                       | HQ-RTR               |
| 7       | `show ip ospf neighbors` → Full                         | HQ-RTR, BR-RTR       |
| 8       | `ping 8.8.8.8`                                          | HQ-SRV, HQ-CLI, BR-SRV |
| 9       | `nslookup hq-cli.au-team.irpo`, `nslookup 192.100.100.2`| любая машина         |
| 10      | `timedatectl` / `show clock`                            | все устройства       |
| 11      | `getent passwd hq_cli1`, вход под hq_cli1               | HQ-CLI, BR-CLI       |
| 12      | `df -h /storage` → ~2 ГБ                                | HQ-SRV               |
| 13      | файл с HQ-CLI виден на BR-CLI                           | HQ-CLI, BR-CLI       |

Для отчёта сними скриншоты: `ip -br a` и `ip route` с каждой машины Альт, `show ip interfaces` и `show ip route` с маршрутизаторов, и все успешные пинги.

---

---

[← README](../README.md) · [Ошибки](troubleshooting.md)
