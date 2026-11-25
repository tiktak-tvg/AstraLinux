# Отказоустойчивый DHCP-кластер на Astra Linux (L3 Relay + Multi-VLAN + DDNS в ALD Pro)

Полное руководство по развёртыванию отказоустойчивого кластера **ISC Kea DHCP** на **Astra Linux Special Edition** (1.7 / 1.8) с подключением через **L3-коммутатор (`ip helper-address`)**, обслуживанием нескольких VLAN и автоматической регистрацией DNS-записей (DDNS) в домене **ALD Pro**.

---

## 1. Архитектура решения и особенности ALD Pro

### Сетевая топология (L3 DHCP Relay)
* **Серверный VLAN 10 (`10.10.10.0/24`):**
  * `astra-dhcp-01.ald.company.lan` (Primary): `10.10.10.11`
  * `astra-dhcp-02.ald.company.lan` (Secondary / Standby): `10.10.10.12`
  * `dc-01.ald.company.lan` (Контроллер домена ALD Pro / DNS BIND 9): `10.10.10.2`
  * `dc-02.ald.company.lan` (Резервный КД ALD Pro / DNS BIND 9): `10.10.10.3`
* **Клиентские VLAN:**
  * **VLAN 20 — Офис (`10.10.20.0/24`):** шлюз `10.10.20.1`, пул `10.10.20.50 – 10.10.20.240`.
  * **VLAN 30 — Техника / Wi-Fi (`10.10.30.0/24`):** шлюз `10.10.30.1`, пул `10.10.30.50 – 10.10.30.240`.
* На L3-коммутаторе (маршрутизаторе) на интерфейсах клиентских VLAN прописаны оба DHCP-сервера (`ip helper-address`). Оба сервера имеют только по одному сетевому интерфейсу в VLAN 10.

### Два подхода к DDNS в домене ALD Pro (Важно!)
В инфраструктуре **ALD Pro** (базирующейся на FreeIPA и BIND 9) динамическое обновление DNS-записей (A и PTR) реализуется одним из двух способов:

1. **Способ 1 (Рекомендуемый разработчиками ALD Pro — через SSSD / GSS-TSIG):**
   * Для доменных рабочих станций на Astra Linux обновление DNS выполняет сама служба `sssd` на клиенте с авторизацией по Kerberos (GSS-TSIG).
   * **Плюс:** каждый клиент может менять только свою DNS-запись; не нужно передавать общий секретный ключ `rndc-key` и синхронизировать его между контроллерами домена.
   * **Минус:** не работает для недоменных устройств (принтеры, IP-телефоны, Linux-серверы вне домена, Wi-Fi-устройства).

2. **Способ 2 (Централизованный DDNS через Kea DHCP + TSIG-ключ BIND 9):**
   * Записи (A и PTR) в DNS контроллера домена создаёт и удаляет сам кластер Kea DHCP (через службу `kea-dhcp-ddns`) при выдаче и истечении аренды IP.
   * **Плюс:** регистрирует в DNS **абсолютно все устройства** (и доменные ПК, и принтеры/IoT).
   * **Нюанс в ALD Pro:** в ALD Pro контроллеры домена используют BIND 9 с бэкендом `bind-dyndb-ldap` (данные хранятся в LDAP). При наличии нескольких КД ключ `/etc/bind/rndc.key` по умолчанию уникален на каждом контроллере, поэтому для отказоустойчивого DDNS лучше создать отдельный общий TSIG-ключ (например, `ddns-kea-key`) и добавить его на оба контроллера домена.

> **Лучшая практика:** Использовать **комбинированный подход** или **Способ 2** с явным перезаписыванием клиентских записей (`ddns-override-client-update: true`), чтобы избежать конфликта прав между `sssd` и DHCP-сервером при обновлении одной и той же записи.

---

## 2. Настройка DNS (BIND 9) на контроллерах домена ALD Pro

Выполните эти действия на основном (и при наличии — на резервном) контроллере домена ALD Pro (`dc-01`, `dc-02`).

### Шаг 1. Генерация общего TSIG-ключа для DHCP-кластера
На первом контроллере домена (`dc-01`) сгенерируйте ключ:

```bash
sudo tsig-keygen -a hmac-sha256 ddns-kea-key
```

Вывод будет выглядеть примерно так (сохраните его — он понадобится и для второго КД, и для обоих серверов Kea):
```text
key "ddns-kea-key" {
    algorithm hmac-sha256;
    secret "xK9vP2mQ8rT5wZ1nB4yC7jL0fD3gH6sA9eR2uI5oP8w=";
};
```

### Шаг 2. Разрешение динамических обновлений в BIND 9 (ALD Pro)
В ALD Pro конфигурация BIND хранится в `/etc/bind/named.conf.options` и `/etc/bind/named.conf.local` (а зоны реплицируются через LDAP).

1. Добавьте описание ключа и политику обновлений в `/etc/bind/named.conf.local` на **обоих контроллерах домена** (`dc-01` и `dc-02`):

```text
key "ddns-kea-key" {
    algorithm hmac-sha256;
    secret "xK9vP2mQ8rT5wZ1nB4yC7jL0fD3gH6sA9eR2uI5oP8w=";
};
```

2. В настройках прямой зоны (`ald.company.lan`) и обратных зон (`20.10.10.in-addr.arpa`, `30.10.10.in-addr.arpa`) в ALD Pro (через веб-портал управления ALD Pro: *Сеть → DNS → Зоны → Настройки зоны* -> разрешить динамическое обновление, либо в политике обновлений BIND) разрешите обновление записей `A`, `AAAA`, `PTR`, `DHCID` по ключу `ddns-kea-key`:

```text
update-policy {
    grant "ddns-kea-key" zonesub A AAAA PTR DHCID;
};
```
*(Примечание: если зоны управляются через `bind-dyndb-ldap` в ALD Pro, добавление ключа в `named.conf.local` и включение «Динамического обновления» в свойствах прямой и обратных зон на портале управления ALD Pro сохранит настройки при перегенерации конфигов).*

3. Проверьте конфигурацию и перезапустите BIND 9:
```bash
sudo named-checkconf
sudo systemctl restart bind9
```

---

## 3. Подготовка серверов Kea DHCP на Astra Linux (оба узла)

Выполните на `astra-dhcp-01` (`10.10.10.11`) и `astra-dhcp-02` (`10.10.10.12`).

### Шаг 1. Установка пакетов (включая службу DDNS)
```bash
sudo apt update
sudo apt install kea-dhcp4-server kea-dhcp-ddns-server kea-ctrl-agent jq -y
```

### Шаг 2. Настройка времени и файрвола (`ufw`)
```bash
sudo timedatectl set-ntp true

# Разрешаем DHCP-трафик (67/udp) и REST API кластера (8000/tcp)
sudo ufw allow 67/udp
sudo ufw allow from 10.10.10.0/24 to any port 8000 proto tcp
sudo ufw reload
```

---

## 4. Настройка Kea Control Agent (`/etc/kea/kea-ctrl-agent.conf`)

Файл управляет связью между узлами кластера по REST API.

**На узле `astra-dhcp-01` (`10.10.10.11`):**
```json
{
  "Control-agent": {
    "http-host": "10.10.10.11",
    "http-port": 8000,
    "control-sockets": {
      "dhcp4": {
        "socket-type": "unix",
        "socket-name": "/run/kea/kea4-ctrl-socket"
      },
      "d2": {
        "socket-type": "unix",
        "socket-name": "/run/kea/kea-ddns-ctrl-socket"
      }
    },
    "loggers": [
      {
        "name": "kea-ctrl-agent",
        "output_options": [ { "output": "/var/log/kea/kea-ctrl-agent.log" } ],
        "severity": "INFO"
      }
    ]
  }
}
```
*(На узле `astra-dhcp-02` замените `"http-host"` на `"10.10.10.12"`).*

---

## 5. Настройка службы Kea DHCP-DDNS (`/etc/kea/kea-dhcp-ddns.conf`)

Эта служба принимает запросы от `kea-dhcp4` по локальному loopback-адресу (`127.0.0.1:53001`) и отправляет подписанные TSIG-ключом обновления на DNS-серверы ALD Pro. **Файл полностью идентичен на обоих серверах (`astra-dhcp-01` и `astra-dhcp-02`).**

```json
{
  "DhcpDdns": {
    "ip-address": "127.0.0.1",
    "port": 53001,
    "dns-server-timeout": 500,
    "ncr-protocol": "UDP",
    "ncr-format": "JSON",

    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/run/kea/kea-ddns-ctrl-socket"
    },

    "tsig-keys": [
      {
        "name": "ddns-kea-key",
        "algorithm": "HMAC-SHA256",
        "secret": "xK9vP2mQ8rT5wZ1nB4yC7jL0fD3gH6sA9eR2uI5oP8w="
      }
    ],

    "forward-ddns": {
      "ddns-domains": [
        {
          "name": "ald.company.lan.",
          "key-name": "ddns-kea-key",
          "dns-servers": [
            { "ip-address": "10.10.10.2", "port": 53 },
            { "ip-address": "10.10.10.3", "port": 53 }
          ]
        }
      ]
    },

    "reverse-ddns": {
      "ddns-domains": [
        {
          "name": "20.10.10.in-addr.arpa.",
          "key-name": "ddns-kea-key",
          "dns-servers": [
            { "ip-address": "10.10.10.2", "port": 53 },
            { "ip-address": "10.10.10.3", "port": 53 }
          ]
        },
        {
          "name": "30.10.10.in-addr.arpa.",
          "key-name": "ddns-kea-key",
          "dns-servers": [
            { "ip-address": "10.10.10.2", "port": 53 },
            { "ip-address": "10.10.10.3", "port": 53 }
          ]
        }
      ]
    },

    "loggers": [
      {
        "name": "kea-dhcp-ddns",
        "output_options": [ { "output": "/var/log/kea/kea-dhcp-ddns.log" } ],
        "severity": "INFO"
      }
    ]
  }
}
```
> **Важно:** Обратите внимание на точку в конце имён зон (`"ald.company.lan."` и `"20.10.10.in-addr.arpa."`) — это обязательное требование формата FQDN в Kea DDNS.

---

## 6. Настройка Kea DHCPv4 (`/etc/kea/kea-dhcp4.conf`)

Конфигурация объединяет **HA-кластер**, **несколько VLAN** (через L3 Relay) и **интеграцию с DDNS**.

### Выбор режима работы (Балансировка 50/50 vs Горячий резерв)

| Параметр в конфиге | Вариант 1: Балансировка 50/50 (Active-Active) | Вариант 2: Горячий резерв (Active-Passive) |
| :--- | :--- | :--- |
| **`"mode"`** | `"load-balancing"` | `"hot-standby"` |
| **Роль 1-го узла (`astra-dhcp-01`)** | `"primary"` | `"primary"` |
| **Роль 2-го узла (`astra-dhcp-02`)** | `"secondary"` | `"standby"` |

### Конфигурация для узла `astra-dhcp-01` (`/etc/kea/kea-dhcp4.conf`)

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "eth0" ]
    },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/run/kea/kea4-ctrl-socket"
    },
    "lease-database": {
      "type": "memfile",
      "persist": true,
      "name": "/var/lib/kea/kea-leases4.csv",
      "lfc-interval": 3600
    },
    "valid-lifetime": 28800,
    "renew-timer": 14400,
    "rebind-timer": 25200,

    "dhcp-ddns": {
      "enable-updates": true,
      "server-ip": "127.0.0.1",
      "server-port": 53001,
      "sender-ip": "",
      "sender-port": 0,
      "max-queue-size": 1024,
      "ncr-protocol": "UDP",
      "ncr-format": "JSON"
    },
    "ddns-send-updates": true,
    "ddns-override-no-update": true,
    "ddns-override-client-update": true,
    "ddns-replace-client-name": "when-not-present",
    "ddns-generated-prefix": "host",
    "ddns-qualifying-suffix": "ald.company.lan",
    "ddns-update-on-renew": true,

    "hooks-libraries": [
      {
        "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_lease_cmds.so"
      },
      {
        "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_ha.so",
        "parameters": {
          "high-availability": [
            {
              "this-server-name": "astra-dhcp-01",
              "mode": "load-balancing",
              "heartbeat-delay": 10000,
              "max-response-delay": 60000,
              "max-ack-delay": 5000,
              "max-unacked-clients": 5,
              "peers": [
                {
                  "name": "astra-dhcp-01",
                  "url": "http://10.10.10.11:8000/",
                  "role": "primary",
                  "auto-failover": true
                },
                {
                  "name": "astra-dhcp-02",
                  "url": "http://10.10.10.12:8000/",
                  "role": "secondary",
                  "auto-failover": true
                }
              ]
            }
          ]
        }
      }
    ],

    "subnet4": [
      {
        "id": 20,
        "subnet": "10.10.20.0/24",
        "pools": [
          { "pool": "10.10.20.50 - 10.10.20.240" }
        ],
        "option-data": [
          { "name": "routers", "data": "10.10.20.1" },
          { "name": "domain-name-servers", "data": "10.10.10.2, 10.10.10.3" },
          { "name": "domain-name", "data": "ald.company.lan" },
          { "name": "domain-search", "data": "ald.company.lan" }
        ]
      },
      {
        "id": 30,
        "subnet": "10.10.30.0/24",
        "pools": [
          { "pool": "10.10.30.50 - 10.10.30.240" }
        ],
        "option-data": [
          { "name": "routers", "data": "10.10.30.1" },
          { "name": "domain-name-servers", "data": "10.10.10.2, 10.10.10.3" },
          { "name": "domain-name", "data": "ald.company.lan" }
        ]
      }
    ],

    "loggers": [
      {
        "name": "kea-dhcp4",
        "output_options": [ { "output": "/var/log/kea/kea-dhcp4.log" } ],
        "severity": "INFO"
      }
    ]
  }
}
```

### Настройка узла `astra-dhcp-02`
Скопируйте `/etc/kea/kea-dhcp4.conf` на второй сервер и измените в нём только параметр:
```json
"this-server-name": "astra-dhcp-02",
```

> **Рекомендация для доменных клиентов ALD Pro (SSSD):**
> Поскольку в конфигурации выше включены `"ddns-override-client-update": true` и `"ddns-override-no-update": true`, именно DHCP-кластер единолично управляет записями в DNS для выданных пулов. Чтобы избежать гонки обновлений со стороны доменных клиентов Astra Linux, убедитесь, что в `/etc/sssd/sssd.conf` на рабочих станциях не включено принудительное дублирование `dyndns_update = true` (либо выключите `ddns-send-updates` для подсети доменных ПК, оставив DDNS только для не-доменных VLAN).

---

## 7. Настройка L3-коммутатора (DHCP Relay)

На SVI-интерфейсах клиентских VLAN (где находятся шлюзы по умолчанию) пропишите **оба** DHCP-сервера:

**Пример для Cisco IOS / Eltex MES:**
```text
interface vlan 20
 description Office_Clients
 ip address 10.10.20.1 255.255.255.0
 ip helper-address 10.10.10.11
 ip helper-address 10.10.10.12
!
interface vlan 30
 description WiFi_and_Devices
 ip address 10.10.30.1 255.255.255.0
 ip helper-address 10.10.10.11
 ip helper-address 10.10.10.12
```

**Пример для Huawei VRP:**
```text
dhcp enable
!
interface Vlanif20
 ip address 10.10.20.1 255.255.255.0
 dhcp select relay
 dhcp relay server-ip 10.10.10.11
 dhcp relay server-ip 10.10.10.12
!
interface Vlanif30
 ip address 10.10.30.1 255.255.255.0
 dhcp select relay
 dhcp relay server-ip 10.10.10.11
 dhcp relay server-ip 10.10.10.12
```

---

## 8. Запуск, проверка кластера и диагностика DDNS

### Шаг 1. Проверка синтаксиса и запуск всех служб (на обоих узлах)
```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo kea-dhcp-ddns -t /etc/kea/kea-dhcp-ddns.conf
sudo kea-ctrl-agent -t /etc/kea/kea-ctrl-agent.conf

sudo systemctl enable --now kea-ctrl-agent kea-dhcp-ddns-server kea-dhcp4-server
```

### Шаг 2. Проверка состояния HA-кластера
```bash
curl -s -X POST -H "Content-Type: application/json"   -d '{ "command": "status-get", "service": [ "dhcp4" ] }'   http://10.10.10.11:8000/ | jq '.[0].arguments."high-availability"'
```
Убедитесь, что у обоих серверов статус `"state": "load-balancing"` (или `"hot-standby"`), а `"in-touch": true`.

### Шаг 3. Проверка работы DDNS в домене ALD Pro
1. Подключите тестовый клиент в VLAN 20 (например, с именем `pc-test-01`).
2. Проверьте лог службы DDNS на активном DHCP-сервере:
   ```bash
   sudo tail -f /var/log/kea/kea-dhcp-ddns.log
   ```
   Успешная регистрация выглядит так:
   `DHCP_DDNS_ADD_SUCCEEDED DHCP_DDNS Request ID ...: successfully added the DNS mapping addition for this request: Type: 0 (CHG_ADD), Forward Change: yes, Reverse Change: yes, FQDN: [pc-test-01.ald.company.lan.], IP Address: [10.10.20.51]`
3. Проверьте разрешение имени в прямую и обратную сторону через DNS контроллера домена ALD Pro:
   ```bash
   dig @10.10.10.2 pc-test-01.ald.company.lan +short
   dig @10.10.10.2 -x 10.10.20.51 +short
   ```
