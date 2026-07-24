# Инструкция: Управление портами и туннелем (Oracle ↔ Pelican)

## Базовый принцип работы и схема сети

Маршрутизация между серверами функционирует на основе подсетевой
маршрутизации по условиям (Policy-Based Routing by Subnet): 1.
**Входящий трафик (Игроки подключаются):** - Сервер Oracle принимает
входящее подключение из интернета на свой внешний IP-адрес. - По
правилам `DNAT` пакет перенаправляется в туннель WireGuard (`wg0`) на
IP-адрес домашнего сервера (`10.0.0.2`). - Пакет поступает в игровой
Docker-контейнер сети `pelican_nw` (`172.18.0.0/16`). 2. **Исходящий
трафик и ответы (Ответы игрокам, Steam Auth, API, Rust+):** - Контейнер
формирует ответ или инициализирует внешнее соединение. - Ядро Linux на
домашнем сервере проверяет источник пакета. Если источник принадлежит
подсети `172.18.0.0/16` (сеть Pelican), правило `ip rule` принудительно
перенаправляет пакет в таблицу маршрутизации `200`. - Таблица `200`
отправляет пакет через интерфейс `wg0` к серверу Oracle, откуда он
выходит в интернет с внешним IP-адресом Oracle. 3. **Локальный трафик
хоста (AdGuard, Панель Pelican, Wings):** - Все процессы и контейнеры
вне подсети `172.18.0.0/16` используют стандартную таблицу маршрутизации
и выходят в интернет напрямую через домашнего провайдера.

    ИНТЕРНЕТ
       ↓                         ↑
    ORACLE VPS (DNAT) ⇄ WireGuard ⇄ Домашний сервер
                                    ↓
                            Docker (pelican_nw)

------------------------------------------------------------------------

## Часть 1. Алгоритм добавления нового порта

### Шаг 1. Oracle Cloud

Добавьте Ingress Rule:

-   Source: `0.0.0.0/0`
-   Protocol: `TCP` или `UDP`
-   Destination Port: `28085`

### Шаг 2. Oracle VPS

``` bash
sudo iptables -t nat -I PREROUTING 1 -p tcp --dport 28085 -j DNAT --to-destination 10.0.0.2:28085
sudo iptables -I FORWARD 1 -p tcp --dport 28085 -j ACCEPT
```

### Шаг 3. Pelican

-   Admin Area → Servers → Build Configuration.
-   Увеличить **Allocated Ports**.
-   Network → привязать `10.0.0.2:28085`.
-   Перезапустить сервер.

> Маркировка через `iptables mangle` больше не требуется.

------------------------------------------------------------------------

## Часть 2. Диагностика

### Oracle

``` bash
sudo iptables -t nat -S PREROUTING | grep DNAT
```

### Домашний сервер

``` bash
ip rule show | grep 172.18.0.0
sudo systemctl status pelican-tunnel-routing.service
```

### Проверка IP контейнера

``` bash
docker run --rm --net pelican_nw curlimages/curl curl -sL ifconfig.me
```

### Проверка открытых портов

``` bash
sudo netstat -tulpn | grep 10.0.0.2
```

------------------------------------------------------------------------

## Часть 3. Сохранение

### Oracle

``` bash
sudo netfilter-persistent save
```

или

``` bash
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### Домашний сервер

Файлы:

-   `/etc/sysctl.d/99-pelican-routing.conf`
-   `/etc/systemd/system/pelican-tunnel-routing.service`

Перезапуск:

``` bash
sudo systemctl restart pelican-tunnel-routing.service
```
