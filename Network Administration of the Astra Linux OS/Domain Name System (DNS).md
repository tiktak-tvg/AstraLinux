### Служба доменных имен DNS в Astra Linux
#### 1. Терминология и компоненты DNS
**DNS (Domain Name System)** — распределённая иерархическая система, обеспечивающая преобразование доменных имён в IP-адреса и обратно.

Ключевые компоненты:

**DNS-клиент (резолвер)** — компонент операционной системы, формирующий запросы к DNS-серверам. В Linux настройки клиента хранятся в файле /etc/resolv.conf:
```text
domain my.dom
search my.dom
nameserver 192.168.1.100
```
Здесь ``domain`` задаёт домен по умолчанию, ``search`` — список доменов для поиска коротких имён, ``nameserver`` — адрес DNS-сервера.

- DNS-сервер — приложение, обрабатывающее запросы. В Astra Linux рекомендуется использовать BIND9, входящий в состав дистрибутивов.
- Пространство имён DNS — иерархическое дерево, в котором каждый узел (домен) имеет своё имя. Доменное имя читается справа налево: от домена верхнего уровня к поддоменам.
- Зона DNS — часть пространства имён, обслуживаемая конкретным DNS-сервером. Зона может содержать один или несколько доменов и поддоменов.
- Делегирование — передача ответственности за часть зоны другому DNS-серверу.
- Рекурсия — режим, при котором DNS-сервер самостоятельно опрашивает другие серверы для полного разрешения имени.

#### 2. Домены и зоны
Домены образуют иерархию:
```text
. (корень)
├── ru
│   └── example
│       └── localnet
├── com
│   └── example
└── local
```
**Зоны** — это административные единицы. Например, зона ``localnet.example.ru`` может содержать записи для хостов ``dns.localnet.example.ru``, ``host.localnet.example.ru`` и т.д.
```txt
Прямая зона — преобразует имя → IP (записи A, AAAA).
Обратная (реверсивная) зона — преобразует IP → имя (записи PTR).
                            Имя обратной зоны для подсети 192.168.32.0/24 — 32.168.192.in-addr.arpa.
```
#### 3. Типы и режимы работы DNS-серверов
Тип	                    | Описание
----------------------- | ---------------------------------------------------------------
Master (primary)	      | Основной сервер, хранит эталонные данные зоны. Изменения вносятся только на нём
Slave (secondary)	      | Резервный сервер, получает копию зоны от master через механизм transfer (AXFR/IXFR)
Forwarder	              | Сервер, перенаправляющий запросы другому DNS-серверу
Caching-only	          | Кеширующий сервер без собственных зон
Stealth	                | Скрытый сервер, не указанный в NS-записях

Режимы работы:
```txt
Рекурсивный — сервер самостоятельно разрешает имена, опрашивая другие серверы.
Итеративный — сервер возвращает клиенту ссылку на другой сервер, если не может разрешить имя сам.
```
#### 4. Ресурсные записи
Основные типы ресурсных записей, используемые в зонах ``BIND9``:

Запись                  | Назначение                    | Пример
----------------------- | ----------------------------- | ----------------------------------------
SOA	 | Start of Authority — начальная запись зоны, содержит параметры: Serial, Refresh, Retry, Expire, Negative Cache TTL	 | @ IN SOA my.dom. root.my.dom. ( 2014031301 604800 86400 2419200 604800 )
NS	 | Name Server — имя DNS-сервера, обслуживающего зону	 | @ IN NS server.my.dom.
A	 | Address — связь имени с IPv4-адресом	server  | IN A 192.168.1.100
AAAA	 | Address — связь имени с IPv6-адресом	server  | IN AAAA 2001:db8::1
PTR | Pointer — обратная связь IP-адреса с именем	100  | IN PTR server.my.dom.
MX	 | Mail Exchange — почтовый сервер для домена	 | @ IN MX 1 server.my.dom.
SRV	 | Service — запись о сетевом сервисе (имя хоста и порт) | 	_https._tcp IN SRV 10 10 443 server.my.com.

Пример файла прямой зоны /var/cache/bind/db.my.dom:
```text
$TTL 604800
@ IN SOA my.dom. root.my.dom. (
    2014031301 ; Serial
    604800     ; Refresh
    86400      ; Retry
    2419200    ; Expire
    604800 )   ; Negative Cache TTL
@ IN NS server.my.dom.
@ IN A 192.168.1.100
@ IN MX 1 server.my.dom.
server IN A 192.168.1.100
client1 IN A 192.168.1.101
client2 IN A 192.168.1.102
ns IN CNAME server
```
Пример файла обратной зоны /var/cache/bind/db.192.168.1:
```text
$TTL 86400
@ IN SOA my.dom. root.my.dom. (
    2014031301 ; Serial
    604800     ; Refresh
    86400      ; Retry
    2419200    ; Expire
    86400 )    ; Negative Cache TTL
@ IN NS server.my.dom.
100 IN PTR server.my.dom.
101 IN PTR client1.my.dom.
102 IN PTR client2.my.dom.

. При создании записей важно ставить точку в конце полных имён, чтобы BIND не добавлял суффикс зоны автоматически.
```
#### 5. Установка DNS-сервера
Пакет bind9 входит в стандартные дистрибутивы Astra Linux. Установка:
```bash
sudo apt update
sudo apt install bind9
sudo apt install dnsutils
```
При установке ``bind9`` автоматически устанавливается пакет ``bind9utils``, содержащий утилиты:

**named-checkconf** — проверка синтаксиса конфигурации;

**named-checkzone** — проверка файлов зон;

**rndc** — управление службой DNS.

Пакет ``dnsutils`` содержит ``dig, nslookup, nsupdate`` для диагностики и динамического обновления записей.

Важно: В Astra Linux служба работает от имени пользователя bind:bind, а не named:named, как в устаревших руководствах.

Управление службой:
```bash
sudo systemctl start bind9
sudo systemctl restart bind9
sudo systemctl status bind9
```
#### 6. Настройка ведущего (master) сервера
##### Шаг 1. Настройка глобальных параметров в /etc/bind/named.conf.options:
```text
options {
    directory "/var/cache/bind";
    forwarders { 8.8.8.8; 8.8.4.4; };
    listen-on { 127.0.0.1; 192.168.32.211; };
    allow-query { any; };
    recursion yes;
    dnssec-validation auto;
};
```
##### Шаг 2. Описание зон в /etc/bind/named.conf.local:
```text
zone "localnet.example.ru" {
    type master;
    file "/etc/bind/zones/db.localnet.example.ru";
    allow-transfer { 192.168.32.212; };
};

zone "32.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.32.168.192";
    allow-transfer { 192.168.32.212; };
};
```
##### Шаг 3. Создание каталога для зон и файлов зон:
```bash
sudo mkdir -p /etc/bind/zones
sudo nano /etc/bind/zones/db.localnet.example.ru
sudo nano /etc/bind/zones/db.32.168.192
```
##### Шаг 4. Проверка и перезапуск:
```bash
sudo named-checkconf
sudo named-checkzone localnet.example.ru /etc/bind/zones/db.localnet.example.ru
sudo named-checkzone 32.168.192.in-addr.arpa /etc/bind/zones/db.32.168.192
sudo systemctl restart bind9
```
#### 7. Настройка подчинённого (slave) сервера
На ``master-сервере`` в директиве ``allow-transfer`` указывается IP-адрес slave-сервера (уже сделано на шаге 6).

На slave-сервере в /etc/bind/named.conf.options:
```text
options {
    directory "/var/cache/bind";
    forwarders { 8.8.8.8; 8.8.4.4; };
    listen-on { 127.0.0.1; 192.168.32.212; };
    allow-query { any; };
};
```
В /etc/bind/named.conf.local зоны описываются с типом slave и указанием адреса master:
```text
zone "localnet.example.ru" {
    type slave;
    file "slaves/db.localnet.example.ru";
    masters { 192.168.32.211; };
};

zone "32.168.192.in-addr.arpa" {
    type slave;
    file "slaves/db.32.168.192";
    masters { 192.168.32.211; };
};
```
Файлы зон на slave автоматически загружаются в каталог slaves относительно directory (обычно ``/var/cache/bind/slaves/``). Проверка и перезапуск:
```bash
sudo named-checkconf
sudo systemctl restart bind9
```
После перезапуска ``slave-сервер`` запросит зоны у master и сохранит их локально.

#### 8. Диагностика службы DNS
Проверка синтаксиса конфигурации:
```bash
sudo named-checkconf
sudo named-checkzone localnet.example.ru /etc/bind/zones/db.localnet.example.ru
```
Проверка разрешения имён с помощью **dig**:
```bash
# Прямой запрос A-записи
dig @localhost host.localnet.example.ru A

# Обратный запрос PTR
dig @localhost -x 192.168.32.96

# Краткий вывод
dig +short @localhost dns.localnet.example.ru
```
Проверка с помощью **nslookup**:
```bash
nslookup host.localnet.example.ru 192.168.32.211
nslookup 192.168.32.96 192.168.32.211
```
Проверка с помощью **host**:
```bash
host -t A localhost 127.0.0.1
host -t PTR 127.0.0.1 127.0.0.1
```
Управление службой через **rndc**:
```bash
sudo rndc status
sudo rndc reload
sudo rndc reload localnet.example.ru
```
Просмотр журналов:
```bash
sudo journalctl -u bind9 -f
sudo tail -f /var/log/syslog | grep named
```
Проверка прослушиваемых портов:
```bash
sudo ss -tulnp | grep :53
```
#### 9. Практическое решение: развёртывание отказоустойчивой DNS-инфраструктуры
Сценарий: развернуть master- и slave-DNS-серверы для домена localnet.example.ru (подсеть 192.168.32.0/24) на базе Astra Linux.

Исходные данные:

Параметр	  | Master	          | Slave
----------- | ----------------- | ----------------
IP-адрес	  | 192.168.32.211	  | 192.168.32.212
Имя хоста	  | dns.localnet.example.ru	  | dns2.localnet.example.ru

##### Этап 1. Подготовка обоих серверов
```bash
sudo apt update
sudo apt install bind9 dnsutils
```
Настройка /etc/resolv.conf на обоих серверах:
```text
domain localnet.example.ru
search localnet.example.ru
nameserver 127.0.0.1
```
##### Этап 2. Настройка master-сервера

/etc/bind/named.conf.options:

```text
options {
    directory "/var/cache/bind";
    forwarders { 8.8.8.8; 8.8.4.4; };
    listen-on { 127.0.0.1; 192.168.32.211; };
    allow-query { any; };
    recursion yes;
};
```
/etc/bind/named.conf.local:
```text
zone "localnet.example.ru" {
    type master;
    file "/etc/bind/zones/db.localnet.example.ru";
    allow-transfer { 192.168.32.212; };
};

zone "32.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.32.168.192";
    allow-transfer { 192.168.32.212; };
};
```
Файл прямой зоны /etc/bind/zones/db.localnet.example.ru:
```text
$TTL 604800
@ IN SOA localnet.example.ru. admin.localnet.example.ru. (
    2026091501 ; Serial
    604800     ; Refresh
    86400      ; Retry
    2419200    ; Expire
    604800 )   ; Negative Cache TTL
;
@ IN NS dns.localnet.example.ru.
@ IN NS dns2.localnet.example.ru.
@ IN MX 10 mail.localnet.example.ru.
;
dns     IN A 192.168.32.211
dns2    IN A 192.168.32.212
host    IN A 192.168.32.96
mail    IN A 192.168.32.100
_ldap._tcp IN SRV 10 10 389 dns.localnet.example.ru.
```
Файл обратной зоны /etc/bind/zones/db.32.168.192:
```text
$TTL 604800
@ IN SOA localnet.example.ru. admin.localnet.example.ru. (
    2026091501 ; Serial
    604800     ; Refresh
    86400      ; Retry
    2419200    ; Expire
    604800 )   ; Negative Cache TTL
;
@ IN NS dns.localnet.example.ru.
@ IN NS dns2.localnet.example.ru.
;
211 IN PTR dns.localnet.example.ru.
212 IN PTR dns2.localnet.example.ru.
96  IN PTR host.localnet.example.ru.
100 IN PTR mail.localnet.example.ru.
```
##### Этап 3. Настройка slave-сервера

/etc/bind/named.conf.options:
```text
options {
    directory "/var/cache/bind";
    forwarders { 8.8.8.8; 8.8.4.4; };
    listen-on { 127.0.0.1; 192.168.32.212; };
    allow-query { any; };
};
```
/etc/bind/named.conf.local:
```text
zone "localnet.example.ru" {
    type slave;
    file "slaves/db.localnet.example.ru";
    masters { 192.168.32.211; };
};

zone "32.168.192.in-addr.arpa" {
    type slave;
    file "slaves/db.32.168.192";
    masters { 192.168.32.211; };
};
```
##### Этап 4. Проверка и запуск

На обоих серверах:
```bash
sudo named-checkconf
sudo systemctl restart bind9
sudo systemctl enable bind9
```
##### Этап 5. Диагностика

С master-сервера:
```bash
dig @localhost host.localnet.example.ru A
dig @localhost -x 192.168.32.96
```
С slave-сервера:
```bash
dig @localhost host.localnet.example.ru A
dig @192.168.32.211 localnet.example.ru AXFR
```
Проверка передачи зоны:
```bash
sudo rndc status
sudo journalctl -u bind9 | grep transfer
```
##### Этап 6. Настройка клиентов

На клиентских машинах в /etc/resolv.conf:
```text
search localnet.example.ru
nameserver 192.168.32.211
nameserver 192.168.32.212
```
При использовании DHCP-сервера необходимо перенастроить его для выдачи клиентам адресов DNS-серверов.

#### 10. Устранение распространённых проблем
Симптом	                | Вероятная причина	             | Решение
----------------------- | ------------------------------ | ----------------------------
SERVFAIL при запросе	 | Ошибка в файле зоны	 | Проверить named-checkzone, исправить синтаксис
Slave не получает зону	 | allow-transfer не настроен или порт 53 закрыт	 | Проверить named.conf.local на master, открыть порт 53
Запросы не обрабатываются	 | BIND слушает не тот интерфейс	 | Проверить listen-on в named.conf.options
Не работают короткие имена	 | Не настроен search в /etc/resolv.conf	 | Добавить домен в search
rndc: connect failed	 | Неверный ключ или служба не запущена	 | Проверить /etc/bind/rndc.key, перезапустить bind9


