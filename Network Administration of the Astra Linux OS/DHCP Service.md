### Служба DHCP в Astra Linux
#### 1. Терминология DHCP
**DHCP (Dynamic Host Configuration Protocol)** — протокол динамической конфигурации хоста, который автоматически назначает IP-адреса и другие сетевые параметры устройствам в сети. 

**DHCP** — это клиент-серверная технология, где взаимодействие происходит по схеме DORA (Discover — Offer — Request — Acknowledge).

Основные термины:

- **DHCP-сервер** — устройство, выдающее IP-адреса и сетевые параметры.
- **DHCP-клиент** — устройство, запрашивающее конфигурацию у сервера.
- **Аренда (lease)** — период времени, на который клиенту выделяется IP-адрес.
- **Область (scope/range)** — диапазон IP-адресов, которые сервер может выдать клиентам.
- **Резервирование (reservation)** — закрепление фиксированного IP-адреса за конкретным MAC-адресом.
- **Пул (pool)** — набор доступных для выдачи адресов внутри подсети.
- **Relay-агент** — устройство, перенаправляющее DHCP-запросы между подсетями (например, BOOTP relay).

Состав дистрибутива Astra Linux:

В состав ОС Astra Linux входит пакет DHCP-сервера isc-dhcp-server и графический инструмент для его быстрой настройки fly-admin-dhcp. Помимо isc-dhcp-server, в состав Astra Linux входит упрощённая служба DHCP dnsmasq.

#### 2. Алгоритм работы DHCP
Процесс получения IP-адреса состоит из четырёх итераций по схеме DORA:
```txt
DHCPDISCOVER — клиент отправляет широковещательное сообщение в свою подсеть.
             В качестве IP-адреса источника указывается 0.0.0.0, в качестве адреса назначения — 255.255.255.255.
             Если DHCP-сервер отсутствует в подсети, сообщение передаётся в другие подсети агентами протокола BOOTP.

DHCPOFFER — получив запрос, DHCP-сервер отвечает сообщением DHCPOFFER, в которое включается предлагаемый IP-адрес (yiaddr)
             и прочие конфигурации для клиента (адреса маршрутизаторов, DNS-серверов и т.д.).
             На этом этапе сервер не обязан резервировать предложенный адрес.

DHCPREQUEST — получив конфигурации от серверов (их может быть несколько, если в подсети более одного DHCP-сервера),
             клиент отправляет широковещательное сообщение DHCPREQUEST.
             В нём содержатся идентификатор выбранного сервера и, возможно, желательные значения запрашиваемых параметров.
             Если клиента не устроит ни один из предложенных адресов, он вновь отправит DHCPDISCOVER.

DHCPACK — получив DHCPREQUEST и убедившись, что в сообщении его идентификатор, сервер проверяет, свободен ли запрошенный адрес.
             Если да — отправляет DHCPACK и вносит запись в базу, иначе отправляет DHCPNACK.
             Получив DHCPACK, клиент должен убедиться в уникальности IP средствами протокола ARP и зафиксировать суммарный срок аренды.

Срок аренды — время, прошедшее между отправкой сообщения DHCPREQUEST и приёмом ответного DHCPACK.
             Плюс срок аренды, указанный в DHCPACK.
             Если адрес уже используется другой станцией, клиент должен отказаться от него и начать процедуру заново.
```
#### 3. Установка и настройка сервера DHCP
##### 3.1. Установка
```bash
sudo apt install isc-dhcp-server
```
Для установки графического инструмента fly-admin-dhcp (который автоматически установит и isc-dhcp-server):

```bash
sudo apt install fly-admin-dhcp
```
После установки инструмент может быть запущен из системного меню («Пуск» → «Панель управления» → «Сеть» → «DHCP сервер») или из командной строки.

##### 3.2. Настройка
Конфигурация сервиса DHCP хранится в двух файлах:

Файл /etc/default/isc-dhcp-server — в параметре INTERFACES нужно указать сетевые интерфейсы, с которыми будет работать сервис. При необходимости указать несколько интерфейсов — они перечисляются через пробел.

Пример настроек (для IPv4 используется интерфейс eth0, работа по IPv6 запрещена):

```text
INTERFACESv4="eth0"
#INTERFACESv6=" "
```
Список имеющихся интерфейсов можно проверить командой ip a или sudo ifconfig.

Файл /etc/dhcp/dhcpd.conf — указывается топология сети и параметры выдаваемой через DHCP информации. В самом файле имеется много примеров задания параметров.

Пример конфигурации:

```text
ddns-update-style none;
option domain-name "my.dom";
default-lease-time 600;
max-lease-time 7200;
log-facility local7;
option domain-name-servers 10.0.10.1;

subnet 10.0.10.0 netmask 255.255.255.0 {
    range 10.0.10.100 10.0.10.200;
    option routers 10.0.10.1;
    max-lease-time 86400;
    filename "pxelinux.0";
}
```
Для резервирования адреса за конкретным MAC-адресом:

```text
host hostname1 {
    hardware ethernet 52:54:00:36:8f:d1;
    fixed-address 10.0.2.10;
}
```
> Важное условие: указанному в ``/etc/default/isc-dhcp-server`` сетевому интерфейсу должен быть присвоен IP-адрес.
Поскольку DHCP-сервис только настраивается, автоматически получить адрес не получится — адрес нужно назначить вручную до запуска сервиса.<br>
Неудачный запуск с ошибкой ``«Not configured to listen on any interfaces!»`` говорит о том, что у сетевого интерфейса нет IP-адреса.

##### 3.3. Запуск и перезапуск
```bash
sudo systemctl restart isc-dhcp-server
```
Или через init-скрипт:

```bash
sudo /etc/init.d/isc-dhcp-server restart
```
#### 4. Настройка клиента DHCP
В Astra Linux настройка DHCP-клиента может выполняться двумя способами:

##### 4.1. Через NetworkManager (рекомендуемый)
NetworkManager по умолчанию использует DHCP-клиент dhclient. Для явного указания использования dhclient создаётся файл /etc/NetworkManager/conf.d/dhcp-client.conf:

```text
[main]
dhcp=dhclient
```
Команда для создания файла:

```bash
echo -e "[main]\ndhcp=dhclient" | sudo tee /etc/NetworkManager/conf.d/dhcp-client.conf
```
##### 4.2. Особенность Astra Linux Special Edition x.7
В Astra Linux Special Edition x.7 DHCP-клиент dhclient по умолчанию использует DUID (DHCP Unique Identifier) для идентификации. При этом NetworkManager может использовать MAC-адрес. Чтобы привести поведение к единообразию, в файл /etc/dhcp/dhclient.conf добавляется:

```text
send dhcp-client-identifier = hardware;
```
##### 4.3. Настройка DNS
Если в системе используется локальный DNS-кеш (dnsmasq), необходимо настроить получение DNS-запросов через него. В файл /etc/dhcp/dhclient.conf добавляется:

```text
prepend domain-name-servers 127.0.0.1;
```
Затем в конфигурации dnsmasq (/etc/dnsmasq.d/dnacache.conf) указываются вышестоящие DNS-серверы:

```text
server=<IP-адрес_1>
server=<IP-адрес_2>
```
После изменений перезапускаются службы:

```bash
sudo systemctl restart NetworkManager dnsmasq
```
В файле /etc/resolv.conf будет указан локальный адрес:

```text
# Generated by NetworkManager
nameserver 127.0.0.1
options edns0 trust-ad
```
#### 5. Диагностика службы DHCP
##### 5.1. Проверка работоспособности сервиса
Проверить, что сервер запущен и прослушивает порт 67:

```bash
sudo ss -tulnp | grep :67
sudo systemctl status isc-dhcp-server
```
##### 5.2. Просмотр выданных аренд
Информация о выданных адресах хранится в файле:

```bash
cat /var/lib/dhcp/dhcpd.leases
```
##### 5.3. Использование nmap для обнаружения DHCP-серверов
```bash
sudo apt install nmap
sudo nmap --script broadcast-dhcp-discover -e eth0
```
Эта команда отправляет широковещательный DHCP-запрос и собирает ответы от всех DHCP-серверов в сети.

##### 5.4. Специализированные утилиты
dhcpdump — прослушивает сетевой интерфейс и отображает DHCP-пакеты в удобном для анализа виде. Позволяет отслеживать весь процесс DORA.

```bash
sudo dhcpdump -i eth0
```
``dhcping`` — позволяет администратору проверить, работает ли удалённый DHCP-сервер. Отправляет запрос и ожидает ответ.

```bash
sudo dhcping -s 192.168.1.1 -c 192.168.1.100
```
##### 5.5. Анализ журналов
```bash
sudo journalctl -u isc-dhcp-server -f
sudo tail -f /var/log/syslog | grep dhcpd
```
##### 5.6. Типичные ошибки и их решение
Ошибка	                            | Причина	                                | Решение
----------------------------------- | ---------------------------------------- | ------------------------------------------------
Not configured to listen on any interfaces!	 | У сетевого интерфейса нет IP-адреса	 | Назначить статический IP-адрес до запуска сервиса
Can't open /var/lib/dhcp/dhcpd.leases	 | Отсутствует файл аренд	 | Создать файл: sudo touch /var/lib/dhcp/dhcpd.leases
Сервер не стартует, сообщая, что уже запущен	 | Остался PID-файл	 | Удалить файл вручную: sudo rm /var/run/dhcpd.pid
IPv6-ошибки при запуске	 | Отключён IPv6	 | Часть, работающая с IPv4, запускается и работает нормально

#### 6. Динамический DNS
Динамический ``DNS (DDNS)`` позволяет DHCP-серверу автоматически сообщать ``DNS-серверу (BIND)`` о назначенных IP-адресах, чтобы имена хостов всегда соответствовали текущим адресам.

##### 6.1. Настройка серверов DNS и DHCP
##### Шаг 1. Создание ключа для DHCP-сервера

Ключ создаётся командой ``rndc-confgen``. После запуска и ввода достаточного количества случайных нажатий ключ автоматически сохраняется в файле ``/etc/bind/rndc.key``.

##### Шаг 2. Настройка DNS-сервера (BIND)

В конфигурацию ``BIND`` необходимо добавить включение ключа и разрешить динамические обновления для нужной зоны.<br> 
Для ``FreeIPA`` настройка выполняется через веб-интерфейс: в свойствах зоны в разделе «Правила обновления BIND» добавляется запись, разрешающая обновления от DHCP-сервера.<br> 
После сохранения изменений службы ``FreeIPA`` перезапускаются командой ``ipactl restart``.

##### Шаг 3. Настройка DHCP-сервера

В файле ``/etc/dhcp/dhcpd.conf`` добавляются параметры для включения динамических обновлений DNS:

```text
ddns-updates on;
ddns-update-style interim;
include "/etc/bind/rndc.key";
ddns-domainname "samdom.example.com.";
update-static-leases on;
option domain-name "samdom.example.com";
option domain-search "samdom.example.com";
option domain-name-servers 10.0.2.102;
option dhcp-server-identifier 10.0.2.102;
default-lease-time 129600;
max-lease-time 1296000;
authoritative;
server-name "ipa.samdom.example.com";
server-identifier 10.0.2.102;

subnet 10.0.2.0 netmask 255.255.255.0 {
    option broadcast-address 10.0.2.255;
    option subnet-mask 255.255.255.0;
    option routers 10.0.2.1;
    pool {
        range 10.0.2.10 10.0.2.100;
        allow known-clients;
        allow unknown-clients;
        max-lease-time 86400;
        default-lease-time 43200;
    }
    zone samdom.example.com. {
        primary 127.0.0.1;
        key "rndc-key";
    }
    zone 2.0.10.in-addr.arpa. {
        primary 127.0.0.1;
        key "rndc-key";
    }
}
```
##### 6.2. Настройка на стороне клиента DHCP
На клиенте необходимо убедиться, что он отправляет своё имя хоста в DHCP-запросе. Для этого в файле ``/etc/dhcp/dhclient.conf`` добавляется:

```text
send host-name = gethostname();
```
Также рекомендуется указать домен для поиска:

```text
supersede domain-name "samdom.example.com";
```
После изменения конфигурации клиент перезапускает сетевой интерфейс:

```bash
sudo dhclient -r eth0 && sudo dhclient eth0
```
##### 6.3. Проверка динамического обновления DNS
Проверить работу динамического обновления можно, запустив отдельный компьютер, настроенный на получение адреса по DHCP. После получения адреса на DNS-сервере должна появиться соответствующая A-запись и PTR-запись. Проверка выполняется командами:

```bash
dig @localhost hostname.samdom.example.com A
dig @localhost -x 10.0.2.10
```
На DHCP-сервере в журналах можно увидеть сообщения об успешном обновлении DNS:

```bash
sudo journalctl -u isc-dhcp-server | grep ddns
```
#### 7. Практическое решение: развёртывание DHCP и DDNS в Astra Linux
Сценарий
Развернуть DHCP-сервер с динамическим обновлением DNS для домена l``ocalnet.example.ru (подсеть 192.168.32.0/24)``. DNS-сервер (BIND9) уже настроен на том же хосте.

##### Этап 1. Установка пакетов
```bash
sudo apt install isc-dhcp-server dnsutils
```
##### Этап 2. Настройка интерфейса
Назначить статический IP-адрес на интерфейс, который будет обслуживать DHCP:

```bash
sudo ip addr add 192.168.32.211/24 dev eth0
```
В файле /etc/default/isc-dhcp-server:

```text
INTERFACESv4="eth0"
```
##### Этап 3. Создание ключа DDNS
```bash
sudo rndc-confgen -a
```
Ключ сохранится в /etc/bind/rndc.key.

##### Этап 4. Настройка BIND для приёма динамических обновлений
В файле ``/etc/bind/named.conf.local`` для зоны ``localnet.example.ru`` добавить:

```text
zone "localnet.example.ru" {
    type master;
    file "/etc/bind/zones/db.localnet.example.ru";
    allow-update { key "rndc-key"; };
};
```
Аналогично для обратной зоны 32.168.192.in-addr.arpa.

Перезапустить BIND:

```bash
sudo systemctl restart bind9
```
##### Этап 5. Настройка DHCP-сервера
Файл ``/etc/dhcp/dhcpd.conf``:

```text
ddns-updates on;
ddns-update-style interim;
include "/etc/bind/rndc.key";
ddns-domainname "localnet.example.ru.";
update-static-leases on;

option domain-name "localnet.example.ru";
option domain-search "localnet.example.ru";
option domain-name-servers 192.168.32.211;
option dhcp-server-identifier 192.168.32.211;

default-lease-time 43200;
max-lease-time 86400;
authoritative;

subnet 192.168.32.0 netmask 255.255.255.0 {
    option broadcast-address 192.168.32.255;
    option subnet-mask 255.255.255.0;
    option routers 192.168.32.1;
    range 192.168.32.100 192.168.32.200;

    zone localnet.example.ru. {
        primary 127.0.0.1;
        key "rndc-key";
    }
    zone 32.168.192.in-addr.arpa. {
        primary 127.0.0.1;
        key "rndc-key";
    }
}
```
Перезапустить DHCP-сервер:

```bash
sudo systemctl restart isc-dhcp-server
```
##### Этап 6. Настройка клиента
На клиентской машине в /etc/dhcp/dhclient.conf:

```text
send host-name = gethostname();
supersede domain-name "localnet.example.ru";
```
##### Этап 7. Проверка
На клиенте получить адрес:

```bash
sudo dhclient -r eth0 && sudo dhclient eth0
```
Проверить полученный адрес:

```bash
ip addr show eth0
```
На DNS-сервере проверить появление записей:

```bash
dig @localhost client-hostname.localnet.example.ru A
dig @localhost -x 192.168.32.100
```
Проверить журналы DHCP:

```bash
sudo journalctl -u isc-dhcp-server -n 50
```
