# Отказоустойчивый DHCP-кластер на Astra Linux (Multi-VLAN)

Практическое руководство по настройке отказоустойчивого DHCP-кластера на базе **Astra Linux Special Edition** (1.7 / 1.8) с поддержкой нескольких VLAN и возможностью переключения между режимами **балансировки нагрузки 50/50 (Load-Balancing)** и **горячего резерва (Hot-Standby)**.

В Astra Linux (основанной на Debian) используются пакеты **ISC Kea DHCP** из стандартного репозитория (`kea-dhcp4-server`, `kea-ctrl-agent`).

---

## 1. Архитектура для нескольких VLAN

Для обслуживания нескольких подсетей (VLAN) используются **два варианта подключения** (рекомендуется Вариант А):

### Вариант А: Через DHCP Relay (`ip helper-address`) — рекомендуемый
* DHCP-серверы находятся в выделенном серверном сегменте (например, **VLAN 10 — Servers**, `10.10.10.0/24`) и имеют только один сетевой интерфейс.
* На маршрутизаторе или L3-коммутаторе, который является шлюзом для клиентских VLAN (**VLAN 20 — Office**, **VLAN 30 — Wi-Fi**), настраивается ретрансляция DHCP-запросов на **оба** сервера одновременно.
* Kea автоматически определяет нужную подсеть по полю `giaddr` (IP-адресу шлюза, переславшего запрос).

### Вариант Б: Прямое подключение (Trunk / 802.1Q)
* На серверах поднимаются VLAN-подинтерфейсы (`eth0.20`, `eth0.30`), и в конфиге Kea перечисляются все интерфейсы (`"interfaces": [ "eth0.10", "eth0.20", "eth0.30" ]`). Используется только при отсутствии L3-коммутатора с DHCP Relay.

### Схема стенда в примере
* **Серверный VLAN 10 (`10.10.10.0/24`):**
  * `astra-dhcp-01` (Primary): `10.10.10.11`
  * `astra-dhcp-02` (Secondary / Standby): `10.10.10.12`
* **Клиентский VLAN 20 — Офис (`10.10.20.0/24`):**
  * Шлюз: `10.10.20.1`
  * Пул DHCP: `10.10.20.50 – 10.10.20.240`
* **Клиентский VLAN 30 — Гости / Wi-Fi (`10.10.30.0/24`):**
  * Шлюз: `10.10.30.1`
  * Пул DHCP: `10.10.30.50 – 10.10.30.240`

---

## 2. Подготовка узлов Astra Linux (выполняется на обоих серверах)

### Шаг 1. Установка пакетов
Убедитесь, что подключены базовый и расширенный репозитории Astra Linux, после чего установите компоненты Kea и утилиту `jq` для проверки статуса:

```bash
sudo apt update
sudo apt install kea-dhcp4-server kea-ctrl-agent jq -y
```

> **Путь к библиотекам (hooks) в Astra Linux:**
> В Astra Linux 1.7 / 1.8 модули Kea располагаются в каталоге `/usr/lib/x86_64-linux-gnu/kea/hooks/`.

### Шаг 2. Синхронизация времени и настройка файрвола (`ufw`)
Для корректной синхронизации аренд расхождение времени между узлами не должно превышать 1–2 секунд.

```bash
# Проверка службы синхронизации времени
sudo timedatectl set-ntp true
chronyc tracking

# Настройка UFW (если используется)
sudo ufw allow 67/udp
sudo ufw allow from 10.10.10.0/24 to any port 8000 proto tcp
sudo ufw reload
```

> **Примечание по МКЦ (мандатный контроль целостности) в Astra Linux:**
> При стандартной установке из репозитория Astra Linux системные службы `kea-dhcp4-server` и `kea-ctrl-agent` запускаются с нулевым уровнем целостности и не требуют ручной смены меток для `/var/lib/kea` и `/run/kea`. Если файлы конфигурации копируются с другого хоста через `scp`, убедитесь, что владелец каталога `/etc/kea` — `root:_kea` (или `root:root` с правами `644`).

---

## 3. Настройка Kea Control Agent (`/etc/kea/kea-ctrl-agent.conf`)

Этот агент отвечает за обмен данными между узлами кластера по HTTP.

### На первом узле (`astra-dhcp-01`, `10.10.10.11`):
```json
{
  "Control-agent": {
    "http-host": "10.10.10.11",
    "http-port": 8000,
    "control-sockets": {
      "dhcp4": {
        "socket-type": "unix",
        "socket-name": "/run/kea/kea4-ctrl-socket"
      }
    },
    "loggers": [
      {
        "name": "kea-ctrl-agent",
        "output_options": [
          { "output": "/var/log/kea/kea-ctrl-agent.log" }
        ],
        "severity": "INFO"
      }
    ]
  }
}
```

### На втором узле (`astra-dhcp-02`, `10.10.10.12`):
Идентичный файл, но в строке `"http-host"` указывается IP второго сервера:
```json
    "http-host": "10.10.10.12",
```

---

## 4. Настройка Kea DHCPv4 (`/etc/kea/kea-dhcp4.conf`)

Ниже приведена конфигурация для **нескольких VLAN (подсетей)** и выбора нужного режима работы кластера.

### Выбор режима работы (Сравнение)

| Параметр | Балансировка 50/50 (`load-balancing`) | Горячий резерв (`hot-standby`) |
| :--- | :--- | :--- |
| **Принцип работы** | Оба сервера активно выдают IP (каждый обслуживает ~50% клиентов по хешу MAC-адреса). | Все IP выдаёт только `primary`. Второй (`standby`) молчит, но получает копии всех аренд в реальном времени. |
| **При отказе 1-го узла** | Второй узел автоматически забирает на себя 100% клиентов без прерывания сервиса. | Резервный узел переходит в состояние `partner-down` и начинает выдавать IP вместо упавшего основного. |
| **Значение `"mode"`** | `"load-balancing"` | `"hot-standby"` |
| **Роль 2-го сервера** | `"secondary"` | `"standby"` |
| **Когда выбирать** | Высокая нагрузка (тысячи клиентов), максимальная утилизация обоих серверов. | Простая диагностика (все логи выдачи в одном месте, предсказуемый ответ от одного сервера). |

### Конфигурация для узла `astra-dhcp-01` (Режим: Балансировка 50/50)

Файл `/etc/kea/kea-dhcp4.conf`:

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
          { "name": "domain-name", "data": "office.local" }
        ],
        "reservations": [
          {
            "hw-address": "00:11:22:33:44:55",
            "ip-address": "10.10.20.15",
            "hostname": "printer-office-01"
          }
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
          { "name": "domain-name-servers", "data": "77.88.8.8, 77.88.8.1" }
        ]
      }
    ],

    "loggers": [
      {
        "name": "kea-dhcp4",
        "output_options": [
          { "output": "/var/log/kea/kea-dhcp4.log" }
        ],
        "severity": "INFO"
      }
    ]
  }
}
```

### Как переключить конфиг в режим горячего резерва (Active-Passive)
Если вместо балансировки 50/50 нужен режим **Active-Passive**, измените всего два поля в блоке `high-availability`:
1. `"mode": "hot-standby"`
2. У второго узла (`astra-dhcp-02`) укажите `"role": "standby"` вместо `"secondary"`.

### Настройка второго узла (`astra-dhcp-02`)
Скопируйте готовый `/etc/kea/kea-dhcp4.conf` с первого узла на второй и измените **только имя текущего узла**:
```json
"this-server-name": "astra-dhcp-02",
```
*(Блок `peers` и описание всех `subnet4` на обоих узлах должны быть полностью идентичны).*

---

## 5. Настройка сетевого оборудования (DHCP Relay)

Чтобы клиенты из VLAN 20 и VLAN 30 получали адреса от кластера, на L3-коммутаторе или маршрутизаторе (Cisco, Eltex, MikroTik, Huawei и др.) необходимо прописать **оба** DHCP-сервера на SVI-интерфейсах клиентских VLAN.

**Пример для Cisco / Eltex:**
```text
interface vlan 20
 ip address 10.10.20.1 255.255.255.0
 ip helper-address 10.10.10.11
 ip helper-address 10.10.10.12
!
interface vlan 30
 ip address 10.10.30.1 255.255.255.0
 ip helper-address 10.10.10.11
 ip helper-address 10.10.10.12
```

> **Важно:** Если используется режим `load-balancing`, маршрутизатор пересылает широковещательный пакет одновременно на оба адреса (`10.10.10.11` и `10.10.10.12`), а серверы сами решают по хешу MAC-адреса клиента, кто из них должен ответить.

---

## 6. Запуск и проверка кластера в Astra Linux

### Шаг 1. Валидация конфигов и запуск служб (на обоих серверах)
```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo kea-ctrl-agent -t /etc/kea/kea-ctrl-agent.conf

sudo systemctl enable --now kea-ctrl-agent kea-dhcp4-server
```

### Шаг 2. Проверка состояния HA-кластера
Выполните запрос к REST API на первом сервере:

```bash
curl -s -X POST -H "Content-Type: application/json"   -d '{ "command": "status-get", "service": [ "dhcp4" ] }'   http://10.10.10.11:8000/ | jq '.[0].arguments."high-availability"'
```

Ожидаемый результат при штатной работе:
```json
[
  {
    "ha-servers": {
      "local": {
        "role": "primary",
        "scopes": [ "astra-dhcp-01" ],
        "state": "load-balancing"
      },
      "remote": {
        "age": 2,
        "in-touch": true,
        "last-scopes": [ "astra-dhcp-02" ],
        "last-state": "load-balancing",
        "role": "secondary"
      }
    }
  }
]
```
*(В режиме горячего резерва `"state"` и `"last-state"` будут равны `"hot-standby"`, а у `remote` список `"last-scopes"` будет пустым `[]`, пока активен `primary`).*

### Шаг 3. Проверка отказоустойчивости (Failover Test)
1. Остановите службу на основном сервере (`astra-dhcp-01`):
   ```bash
   sudo systemctl stop kea-dhcp4-server
   ```
2. Через ~15–60 секунд (согласно `max-response-delay`) на втором сервере (`astra-dhcp-02`) в логе `/var/log/kea/kea-dhcp4.log` появится сообщение:
   `HA_STATE_TRANSITION server transitions from LOAD-BALANCING to PARTNER-DOWN state`
3. Второй сервер автоматически заберёт себе область `"scopes": [ "astra-dhcp-01", "astra-dhcp-02" ]` и начнёт обслуживать 100% клиентов во всех VLAN.
4. После запуска первого узла (`sudo systemctl start kea-dhcp4-server`) он автоматически скачает актуальную базу выданных за время простоя аренд и вернётся в штатный режим.
