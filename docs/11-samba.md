# Задание 11. Контроллер домена Samba DC

**Машины:** BR-SRV, HQ-CLI, BR-CLI

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

> **Из задания.** Настройте контроллер домена Samba DC на BR-SRV. Имя домена `au-team.irpo`. Введите в домен HQ-CLI и BR-CLI. Создайте 8 пользователей `hq_cli№` и группу `cat_hq`, 6 пользователей `br_cli№` и группу `cat_br`. Убедитесь, что пользователи `cat_hq` имеют право аутентифицироваться на HQ-CLI, а `cat_br` — на BR-CLI. Пользователи `cat_hq` должны иметь возможность повышать привилегии для `nano`, `cat`, `id` и не иметь права запускать другие команды.

Самое долгое задание, выполняется последним. У него три внешние зависимости — проверь их до начала:

| Зависимость           | Проверка                                      | Симптом при нарушении                      |
|-----------------------|-----------------------------------------------|--------------------------------------------|
| Время (задание 10)    | `date` на BR-SRV и клиентах                   | `Clock skew too great`                     |
| DNS (задание 9)       | `nslookup br-srv.au-team.irpo` с HQ-CLI       | контроллер домена не найден                |
| Связь (задания 6–8)   | `ping 192.100.20.2` с HQ-CLI                  | ввод в домен не проходит                   |

### 11.1. Подготовка BR-SRV

**Машина: BR-SRV (106)**

```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
hostname -f

echo "nameserver 127.0.0.1" > /etc/net/ifaces/ens19/resolv.conf
echo "search au-team.irpo" >> /etc/net/ifaces/ens19/resolv.conf
systemctl restart network

grep -q br-srv /etc/hosts || echo "192.100.20.2 br-srv.au-team.irpo br-srv" >> /etc/hosts
```

Контроллер домена поднимает собственный DNS, поэтому спрашивать он должен сам себя. `hostname -f` обязан выдавать полное имя — иначе развёртывание откажется работать.

### 11.2. Установка и очистка

```bash
apt-get update
apt-get install -y task-samba-dc

for s in smb nmb winbind krb5kdc slapd; do
  systemctl disable --now $s 2>/dev/null
done

rm -f /etc/samba/smb.conf
rm -f /etc/krb5.conf
rm -rf /var/lib/samba/private/*
rm -rf /var/lib/samba/sysvol/*
```

`task-samba-dc` — метапакет Альт, который ставит Samba в режиме Active Directory DC вместе с Kerberos. Обычный пакет `samba` собран для файлового сервера, контроллер домена на нём не поднимется.

Очистка нужна только при повторной попытке: развёртывание падает, если `smb.conf` уже существует.

### 11.3. Развернуть домен

```bash
samba-tool domain provision \
  --realm=AU-TEAM.IRPO \
  --domain=AU-TEAM \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --adminpass='P@ssw0rd' \
  --use-rfc2307
```

| Параметр                        | Назначение                                                                           |
|---------------------------------|--------------------------------------------------------------------------------------|
| `--realm=AU-TEAM.IRPO`          | область Kerberos, **заглавными** — это требование стандарта, не косметика             |
| `--domain=AU-TEAM`              | короткое NetBIOS-имя, до 15 символов                                                  |
| `--server-role=dc`              | роль контроллера домена                                                               |
| `--dns-backend=SAMBA_INTERNAL`  | встроенный DNS Samba                                                                  |
| `--adminpass`                   | пароль администратора домена, в одинарных кавычках из-за символа `@`                   |
| `--use-rfc2307`                 | хранить UID и GID в каталоге — без этого доменные пользователи не смогут владеть файлами |

### 11.4. Запуск

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

systemctl enable --now samba
systemctl status samba

ss -tlnp | grep -E ':(53|88|389|445)'
```

Служба `samba` заменяет `smb`, `nmb` и `winbind` — запускать их отдельно не нужно и вредно. Порты: 53 — DNS, 88 — Kerberos, 389 — LDAP, 445 — SMB.

### 11.5. Пользователи и группы

```bash
for i in $(seq 1 8); do
  samba-tool user create hq_cli$i 'P@ssw0rd' --given-name="HQ User $i"
done

for i in $(seq 1 6); do
  samba-tool user create br_cli$i 'P@ssw0rd' --given-name="BR User $i"
done

samba-tool group add cat_hq
samba-tool group add cat_br

for i in $(seq 1 8); do samba-tool group addmembers cat_hq hq_cli$i; done
for i in $(seq 1 6); do samba-tool group addmembers cat_br br_cli$i; done

samba-tool user list
samba-tool group listmembers cat_hq
samba-tool group listmembers cat_br
```

### 11.6. Конфликт DNS и как его обойти

В задании 9 HQ-SRV стал первичным сервером зоны `au-team.irpo`. В задании 11 Samba поднимает **свой** DNS для той же зоны. Два авторитетных сервера для одной зоны — конфликт: клиент, спрашивающий HQ-SRV, не узнает о служебных записях контроллера домена.

**Способ 1 — простой.** На время ввода в домен переключить клиента на DNS контроллера, потом вернуть обратно:

```bash
echo "nameserver 192.100.20.2" > /etc/net/ifaces/ens19/resolv.conf
echo "search au-team.irpo" >> /etc/net/ifaces/ens19/resolv.conf
systemctl restart network
```

**Способ 2 — аккуратный.** Добавить служебные записи в зону BIND на HQ-SRV:

```
_ldap._tcp                      IN SRV 0 100 389 br-srv.au-team.irpo.
_kerberos._tcp                  IN SRV 0 100 88  br-srv.au-team.irpo.
_kerberos._udp                  IN SRV 0 100 88  br-srv.au-team.irpo.
_kpasswd._tcp                   IN SRV 0 100 464 br-srv.au-team.irpo.
_kpasswd._udp                   IN SRV 0 100 464 br-srv.au-team.irpo.
_ldap._tcp.dc._msdcs            IN SRV 0 100 389 br-srv.au-team.irpo.
_kerberos._tcp.dc._msdcs        IN SRV 0 100 88  br-srv.au-team.irpo.
```

Не забудь увеличить `serial` и выполнить `systemctl reload bind`.

### 11.7. Ввод клиентов в домен

**Машины: HQ-CLI (107) и BR-CLI (110)**

```bash
apt-get install -y task-auth-ad-sssd

system-auth write ad au-team.irpo br-srv au-team 'administrator' 'P@ssw0rd'

getent passwd hq_cli1
id hq_cli1
```

Универсальный путь, если `system-auth` не сработала:

```bash
apt-get install -y realmd sssd adcli
realm discover au-team.irpo
realm join -U administrator au-team.irpo
realm list
```

`realm discover` полезна сама по себе: если она ничего не находит — проблема в DNS, а не в вводе в домен.

### 11.8. Права на аутентификацию

Задание требует, чтобы `cat_hq` входили на HQ-CLI, а `cat_br` — на BR-CLI. Ограничиваем через SSSD:

```bash
# на HQ-CLI
sed -i '/^\[domain\/au-team.irpo\]/a access_provider = simple\nsimple_allow_groups = cat_hq' /etc/sssd/sssd.conf
systemctl restart sssd

# на BR-CLI
sed -i '/^\[domain\/au-team.irpo\]/a access_provider = simple\nsimple_allow_groups = cat_br' /etc/sssd/sssd.conf
systemctl restart sssd
```

| Параметр                      | Назначение                                                      |
|-------------------------------|-----------------------------------------------------------------|
| `access_provider = simple`    | решение о допуске принимается по спискам, а не «пускать всех»   |
| `simple_allow_groups`         | пускать только членов этой группы                               |

Локальных учёток (`root`) это не затрагивает, но проверь вход под root во второй консоли до того, как закроешь текущую сессию.

### 11.9. Ограниченный sudo для cat_hq

**Машина: HQ-CLI (107)**

```bash
which nano cat id
# если nano не найден — он нужен по условию
apt-get install -y nano

cat > /etc/sudoers.d/cat_hq <<'EOF'
%cat_hq ALL=(ALL) NOPASSWD: /usr/bin/nano, /usr/bin/cat, /usr/bin/id
EOF

chmod 440 /etc/sudoers.d/cat_hq
visudo -c
```

| Элемент          | Пояснение                                                                     |
|------------------|-------------------------------------------------------------------------------|
| `%cat_hq`        | знак процента означает **группу**, а не пользователя                          |
| список команд    | вместо `ALL` — перечисление; всё остальное sudo отклонит                      |
| абсолютные пути  | обязательны: иначе подменой `PATH` можно выполнить что угодно под именем `cat` |

### Проверка задания 11

```bash
# на BR-SRV
host -t SRV _ldap._tcp.au-team.irpo 127.0.0.1
kinit administrator@AU-TEAM.IRPO
klist
samba-tool user list

# на HQ-CLI
getent passwd hq_cli1
# войти как hq_cli1 — должно пустить
# войти как br_cli1 — должно отказать
sudo id        # работает
sudo rm /tmp/x # отказ — доказывает, что доступ ограничен
```

Именно второй тест sudo доказывает выполнение требования: разрешены только три команды.

---

---

[← Задание 10](10-time.md) · [Задание 12 →](12-raid.md) · [Ошибки](troubleshooting.md)
