# Задание 5. Безопасный удалённый доступ

**Машины:** HQ-SRV, BR-SRV

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

> **Из задания.** На HQ-SRV и BR-SRV ограничьте количество попыток входа до двух и настройте баннер «Only authorized access is allowed».

**Машины: HQ-SRV (105) и BR-SRV (106)**

```bash
apt-get install -y openssh-server
systemctl enable --now sshd

echo "Only authorized access is allowed" > /etc/openssh/banner

CFG=/etc/openssh/sshd_config
sed -i 's/^#\?MaxAuthTries.*/MaxAuthTries 2/' $CFG
sed -i 's|^#\?Banner.*|Banner /etc/openssh/banner|' $CFG
grep -q '^MaxAuthTries' $CFG || echo 'MaxAuthTries 2' >> $CFG
grep -q '^Banner'       $CFG || echo 'Banner /etc/openssh/banner' >> $CFG

sshd -t && systemctl restart sshd
```

| Шаг                          | Пояснение                                                                       |
|------------------------------|---------------------------------------------------------------------------------|
| `/etc/openssh/sshd_config`   | путь в Альт; в Debian и RHEL это `/etc/ssh/sshd_config`                         |
| `MaxAuthTries 2`             | защита от подбора: после двух неудач соединение рвётся                          |
| `Banner`                     | предупреждение показывается **до** аутентификации                               |
| `sshd -t`                    | проверка конфига перед перезапуском — ошибка не даст службе подняться           |
| `&&`                         | перезапуск только если проверка прошла                                          |

Текст баннера копируй дословно, без точки в конце и без перевода.

### Проверка задания 5

```bash
sshd -T | grep -iE 'maxauthtries|banner'    # действующие значения
ssh sshcli@192.100.100.2                    # с HQ-CLI
```

Должен появиться текст баннера, а после двух неверных паролей — разрыв соединения вместо третьей попытки.

---

---

[← Задание 4](04-vlan.md) · [Задание 6 →](06-gre.md) · [Ошибки](troubleshooting.md)
