# Задание 9. Инфраструктура разрешения доменных имён

**Машины:** HQ-SRV, все машины

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

> **Из задания.** Основной DNS-сервер реализован на HQ-SRV. Сервер должен обеспечивать разрешение имён в адреса и обратно согласно таблице. В качестве сервера пересылки используйте `94.232.137.104`.

Таблица записей из задания:

| Устройство | Запись                 | Тип     | Адрес            |
|------------|------------------------|---------|------------------|
| HQ-RTR     | `hq-rtr.au-team.irpo`  | A, PTR  | `192.100.100.1`  |
| BR-RTR     | `br-rtr.au-team.irpo`  | A       | `192.100.20.1`   |
| HQ-SRV     | `hq-srv.au-team.irpo`  | A, PTR  | `192.100.100.2`  |
| HQ-CLI     | `hq-cli.au-team.irpo`  | A, PTR  | `192.100.200.2`  |
| BR-SRV     | `br-srv.au-team.irpo`  | A       | `192.100.20.2`   |
| ISP → HQ   | `docker.au-team.irpo`  | A       | `172.16.1.1`     |
| ISP → BR   | `web.au-team.irpo`     | A       | `172.16.2.1`     |

> Последние две записи указывают на интерфейсы **провайдера**, а не офисов. Это единственное место таблицы, где легко подставить не тот адрес.

**Машина: HQ-SRV (105), Альт Сервер**

### 9.1. Установить BIND

```bash
apt-get update
apt-get install -y bind bind-utils

ls -l /var/lib/bind/etc/
ls -l /var/lib/bind/etc/zone/
```

BIND в Альт работает в chroot внутри `/var/lib/bind`, поэтому пути нестандартные:

| Путь                                | Назначение                                      |
|-------------------------------------|-------------------------------------------------|
| `/var/lib/bind/etc/options.conf`    | параметры сервера                               |
| `/var/lib/bind/etc/rfc1912.conf`    | объявления зон                                  |
| `/var/lib/bind/etc/zone/`           | файлы зон                                       |
| `/var/lib/bind/etc/zone/empty`      | заготовка, с которой копируют новые файлы зон   |

Пакет `bind-utils` обязателен — в нём `dig`, `nslookup`, `named-checkzone`.

### 9.2. Параметры сервера

```bash
cat > /var/lib/bind/etc/options.conf <<'EOF'
options {
    directory "/var/lib/bind/etc/zone";
    listen-on { any; };
    allow-query { any; };
    recursion yes;
    forwarders { 94.232.137.104; };
    forward first;
    dnssec-validation no;
};
EOF
```

| Параметр                        | Назначение                                                                   |
|---------------------------------|------------------------------------------------------------------------------|
| `listen-on { any; }`            | слушать на всех интерфейсах; по умолчанию только `127.0.0.1`                 |
| `allow-query { any; }`          | кому разрешено спрашивать                                                    |
| `recursion yes`                 | сервер сам ищет ответ для клиента                                            |
| `forwarders`                    | адрес из задания: куда отправлять запросы о чужих зонах                      |
| `forward first`                 | сначала спросить forwarder, потом искать самому                              |
| `dnssec-validation no`          | в изолированной сети проверка подписей мешает пересылке                      |

### 9.3. Объявить зоны

```bash
cat > /var/lib/bind/etc/rfc1912.conf <<'EOF'
zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
};

zone "100.100.192.in-addr.arpa" {
    type master;
    file "100.100.192.in-addr.arpa";
};

zone "200.100.192.in-addr.arpa" {
    type master;
    file "200.100.192.in-addr.arpa";
};
EOF
```

Имя обратной зоны собирается так: берём три первых октета сети, переворачиваем и добавляем `in-addr.arpa`. Для `192.100.100.0` получается `100.100.192.in-addr.arpa`.

### 9.4. Файлы зон

```bash
cd /var/lib/bind/etc/zone
cp empty au-team.irpo
cp empty 100.100.192.in-addr.arpa
cp empty 200.100.192.in-addr.arpa

cat > au-team.irpo <<'EOF'
$TTL 1D
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2026100201  ; serial
        12H         ; refresh
        1H          ; retry
        1W          ; expire
        1H )        ; ncache

@           IN  NS      hq-srv.au-team.irpo.

hq-rtr      IN  A       192.100.100.1
br-rtr      IN  A       192.100.20.1
hq-srv      IN  A       192.100.100.2
hq-cli      IN  A       192.100.200.2
br-srv      IN  A       192.100.20.2
docker      IN  A       172.16.1.1
web         IN  A       172.16.2.1
EOF

cat > 100.100.192.in-addr.arpa <<'EOF'
$TTL 1D
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2026100201 12H 1H 1W 1H )
@   IN  NS  hq-srv.au-team.irpo.

1   IN  PTR hq-rtr.au-team.irpo.
2   IN  PTR hq-srv.au-team.irpo.
EOF

cat > 200.100.192.in-addr.arpa <<'EOF'
$TTL 1D
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2026100201 12H 1H 1W 1H )
@   IN  NS  hq-srv.au-team.irpo.

2   IN  PTR hq-cli.au-team.irpo.
EOF
```

| Элемент      | Пояснение                                                                                 |
|--------------|-------------------------------------------------------------------------------------------|
| `@`          | сокращение для имени самой зоны                                                           |
| `SOA`        | обязательная первая запись: главный сервер зоны и почта администратора                    |
| `serial`     | версия зоны; при любой правке **увеличивать**, иначе изменения не подхватятся              |
| `NS`         | какой сервер отвечает за зону                                                              |
| короткое имя | без точки на конце — BIND допишет имя зоны, получится `hq-rtr.au-team.irpo`                |
| имя с точкой | абсолютное, берётся как есть; в SOA, NS и PTR точка в конце **обязательна**                |

Самая частая ошибка в DNS — забытая точка. В записи PTR `hq-rtr.au-team.irpo` без точки превратится в `hq-rtr.au-team.irpo.100.100.192.in-addr.arpa`.

### 9.5. Ключ, права и запуск

```bash
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key
cat /var/lib/bind/etc/rndc.key

chgrp -R named /var/lib/bind/etc/zone
chmod 644 /var/lib/bind/etc/zone/*

named-checkconf
named-checkconf -z

systemctl enable --now bind
systemctl status bind
```

| Команда                 | Назначение                                                                     |
|-------------------------|--------------------------------------------------------------------------------|
| `rndc-confgen`          | сгенерировать ключ управления службой                                          |
| `sed -i '6,$d'`         | оставить только блок ключа, удалив закомментированный пример                    |
| `chgrp -R named`        | BIND работает не от root; без прав зона не загрузится с ошибкой permission denied |
| `named-checkconf -z`    | загрузить все зоны и показать `loaded serial` для каждой — лучшая проверка       |

Если `chgrp` не сработал, задай владельца каждому файлу: `chown root:named /var/lib/bind/etc/zone/au-team.irpo`.

### 9.6. Прописать DNS всем машинам

```bash
# на каждой машине Альт
echo "nameserver 192.100.100.2" > /etc/net/ifaces/ens19/resolv.conf
echo "search au-team.irpo" >> /etc/net/ifaces/ens19/resolv.conf
systemctl restart network
```

```
# на маршрутизаторах
configure
ip name-server 192.100.100.2
do commit
do confirm
```

Файл `/etc/resolv.conf` в Альт **генерируется** службой сети — править его руками бесполезно, настройка переживёт только до следующего `systemctl restart network`.

### Проверка задания 9

```bash
nslookup hq-srv.au-team.irpo
nslookup br-rtr.au-team.irpo
nslookup docker.au-team.irpo
nslookup web.au-team.irpo

nslookup 192.100.100.1      # hq-rtr
nslookup 192.100.100.2      # hq-srv
nslookup 192.100.200.2      # hq-cli

nslookup ya.ru              # проверка forwarder
dig @192.100.100.2 hq-cli.au-team.irpo +short
```

---

---

[← Задание 8](08-nat.md) · [Задание 10 →](10-time.md) · [Ошибки](troubleshooting.md)
