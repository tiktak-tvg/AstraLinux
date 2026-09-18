### Управление конфигурациями хостов с помощью Ansible
#### 1. Архитектура Ansible
**Ansible** — это система управления конфигурациями, написанная на Python и использующая декларативный язык разметки для описания конфигураций. Она применяется для автоматизации настройки и массового развёртывания программного обеспечения.

Базовая концепция Ansible основана на создании сценариев автоматизации, интерпретируемых управляющим узлом (control node) для выполнения требуемых операций на управляемых узлах (managed nodes).

Ключевые компоненты архитектуры:

Компонент                   | Описание             
--------------------------- | --------------------------------- 
Управляющий узел (Control Node)	 | Узел (рабочая станция, ВМ, контейнер), на котором установлен Ansible. С него запускаются команды ansible и ansible-playbook. Узел содержит: программное обеспечение Ansible Core, описание инвентаря (inventory) и наборы сценариев автоматизации (playbooks)
Управляемый узел (Managed Node)	 | Сервер, устройство или иной ресурс, состояние которого Ansible изменяет или контролирует с помощью сценариев автоматизации
Целевой узел (Target Node)	 | Узел, на котором непосредственно выполняется задача из сценария автоматизации. В общем случае может совпадать с управляемым узлом или выступать в роли промежуточного узла (proxy)
Инвентарь (Inventory)	 | Файл, содержащий список управляемых узлов, сгруппированных по определённым признакам
Модули (Modules)	 | Готовые блоки кода, выполняющие конкретные задачи (установка пакетов, управление файлами, настройка сервисов)
Плейбуки (Playbooks)	 | YAML-файлы, описывающие последовательность задач для выполнения на управляемых узлах
Роли (Roles)	 | Модульная структура для организации плейбуков и повторного использования кода

Принцип работы: Ansible не требует установки агентского программного обеспечения на управляемые узлы. Взаимодействие осуществляется по протоколам SSH (для Linux/Unix) или WinRM (для Windows). Модули передаются на управляемый узел, выполняются там и удаляются после завершения задачи.

#### 2. Установка и настройка Ansible
##### 2.1. Где используется Ansible: клиентские или серверные ОС?
Ansible применяется преимущественно для управления серверными операционными системами, однако поддерживает и клиентские ОС.

Серверные ОС:

- Linux-серверы (Astra Linux, Red Hat, Ubuntu, Debian и др.) — управление по SSH
- Windows Server (2008, 2012, 2016, 2019, 2022) — управление по WinRM
- Сетевые устройства (коммутаторы, маршрутизаторы) — через специализированные модули

Клиентские ОС:

- Windows 8.1, Windows 10 — управление по WinRM
- Linux-рабочие станции — управление по SSH

Управляющий узел (Control Node) должен работать под управлением Linux/Unix (в Astra Linux — Astra Linux Special Edition). Установка Ansible на Windows в качестве управляющего узла не поддерживается.

##### 2.2. Установка в Astra Linux
Пакет ansible доступен в репозитории Astra Linux Common Edition и в дистрибутиве Astra Linux Special Edition.

Установка из командной строки:

```bash
sudo apt update
sudo apt install ansible
```
Проверка установки:

```bash
ansible --version
```
##### 2.3. Настройка пакета
При установке пакета создаётся каталог /etc/ansible, содержащий два конфигурационных файла:

- /etc/ansible/hosts — файл инвентаря (список хостов);
- /etc/ansible/ansible.cfg — файл настроек Ansible.

В Astra Linux Special Edition 1.8 конфигурационные файлы при установке не создаются и должны быть созданы вручную. Файл ansible.cfg может располагаться в следующих локациях (в порядке приоритета):

Путь, указанный в переменной окружения ANSIBLE_CONFIG;

- ansible.cfg в текущем рабочем каталоге;
- ~/.ansible.cfg в домашнем каталоге пользователя;
- /etc/ansible/ansible.cfg — общесистемная конфигурация.

Пример минимального файла ansible.cfg:

```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
remote_user = administrator
ask_pass = False
```
##### 2.4. Настройка беспарольного доступа по SSH
Для работы Ansible с управляемыми узлами необходимо настроить аутентификацию по SSH-ключам:

```bash
# Генерация ключа на управляющем узле
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_ansible -C "ansible@control"

# Копирование ключа на управляемый узел
ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub administrator@192.168.32.100
```
Проверка подключения:

```bash
ansible all -m ping
```
#### 3. Использование Ansible из командной строки
Ansible предоставляет набор утилит командной строки для выполнения различных задач.

##### 3.1. Основные команды

Команда                     | Назначение               
--------------------------- | --------------------------------- 
ansible	 | Выполнение отдельных модулей (ad-hoc команды)
ansible-playbook	 | Запуск плейбуков
ansible-vault	 | Шифрование и управление секретными данными
ansible-galaxy	 | Управление ролями и коллекциями
ansible-doc	 | Просмотр документации по модулям

##### 3.2. Ad-hoc команды
Ad-hoc команды используются для выполнения разовых задач без создания плейбука.

Синтаксис:

```bash
ansible <группа_хостов> -m <модуль> -a "<аргументы>" [опции]
```
Примеры:

```bash
# Проверка доступности всех хостов
ansible all -m ping

# Получение информации о системе
ansible all -m setup

# Установка пакета на всех хостах
ansible webservers -m apt -a "name=nginx state=present" --become

# Копирование файла на группу хостов
ansible dbservers -m copy -a "src=/etc/my.cnf dest=/etc/my.cnf"

# Перезапуск сервиса
ansible webservers -m service -a "name=nginx state=restarted" --become

# Выполнение команды на всех хостах
ansible all -m command -a "uptime"
```
Опции командной строки:

Опция                       | Описание               
--------------------------- | --------------------------------- 
-i <файл>	 | Указать файл инвентаря
-m <модуль>	 | Указать модуль
-a "<аргументы>"	 | Передать аргументы модулю
-u <пользователь>	 | Указать пользователя для подключения
--become	 | Выполнить с повышением привилегий (sudo)
-b	 | Краткая форма --become
-K	 | Запросить пароль для sudo
-v, -vvv	 | Увеличить уровень детализации вывода

#### 4. Создание файлов инвентаризации и плейбуков
##### 4.1. Файл инвентаризации (Inventory)
Файл /etc/ansible/hosts содержит список хостов, с которыми работает Ansible. Файл снабжён подробными комментариями и поддерживает группировку хостов.

Пример простого инвентаря:

```ini
# Хосты без групп
192.168.32.100
192.168.32.101

[webservers]
web1.localnet.example.ru
web2.localnet.example.ru
192.168.32.110

[dbservers]
db1.localnet.example.ru
db2.localnet.example.ru
192.168.32.120

[all:vars]
ansible_user=administrator
ansible_ssh_private_key_file=/home/user/.ssh/id_ed25519_ansible

[webservers:vars]
http_port=80
```
Применение шаблонов для определения нескольких хостов:

```ini
[webservers]
www[001:006].example.com
```
Это создаст хосты www001.example.com, www002.example.com и т.д..

Расширенный инвентарь с переменными для каждого хоста:

```ini
[webservers]
web1 ansible_host=192.168.32.110 ansible_port=2222 http_port=80
web2 ansible_host=192.168.32.111 ansible_port=22 http_port=8080

[dbservers]
db1 ansible_host=192.168.32.120 ansible_user=postgres
```
##### 4.2. Плейбуки (Playbooks)
Плейбуки — это YAML-файлы, описывающие последовательность задач для выполнения на управляемых узлах.

Структура простого плейбука:

```yaml
---
- name: Установка и настройка веб-сервера
  hosts: webservers
  become: yes
  vars:
    http_port: 80
    max_clients: 200

  tasks:
    - name: Установить nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Запустить и включить nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Скопировать конфигурационный файл
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Перезапустить nginx

  handlers:
    - name: Перезапустить nginx
      service:
        name: nginx
        state: restarted
```
Ключевые элементы плейбука:

Элемент                     | Описание               
--------------------------- | --------------------------------- 
name	 | Описание плейбука или задачи
hosts	 | Группа хостов из инвентаря
become	 | Выполнение с повышением привилегий
vars	 | Переменные плейбука
tasks	 | Список задач
handlers	 | Обработчики, вызываемые через notify
roles	 | Подключаемые роли

Запуск плейбука:

```bash
ansible-playbook -i /etc/ansible/hosts site.yml
```
Полезные опции:

Опция                       | Описание               
--------------------------- | --------------------------------- 
--check	 | Режим проверки (dry-run) — показывает, что будет сделано, но не выполняет
--diff	 | Показывает различия в файлах
--limit <хост>	 | Ограничить выполнение конкретным хостом
-e "var=value"	 | Передать переменную из командной строки
-t <тег>	  | Выполнить только задачи с указанным тегом
-v, -vvv	  | Увеличить детализацию вывода

#### 5. Переменные
Переменные в Ansible позволяют параметризовать плейбуки и роли, делая их гибкими и повторно используемыми.

##### 5.1. Приоритет переменных
Ansible использует иерархию приоритетов переменных. Общее правило: переменные, определённые позже, имеют более высокий приоритет.

Основные уровни приоритета (от низшего к высшему):

- Конфигурационные настройки (ansible.cfg) — низший приоритет
- Опции командной строки — переопределяют конфигурацию
- Ключевые слова плейбука — переопределяют опции командной строки
- Переменные — высший приоритет

Детальная иерархия переменных (от низшего к высшему):

- role defaults — переменные по умолчанию роли (самый низкий приоритет)
- Переменные инвентаря (group_vars/all)
- Переменные инвентаря (group_vars/<группа>)
- Переменные инвентаря (host_vars/<хост>)
- vars в плейбуке
- role vars — переменные роли
- set_fact / register — переменные, созданные во время выполнения
- extra vars (-e в командной строке) — высший приоритет

##### 5.2. Способы определения переменных
В плейбуке (блок vars):

```yaml
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  tasks:
    - name: Вывести порт
      debug:
        msg: "Порт: {{ http_port }}"
```
В инвентаре (переменные группы и хоста):

```ini
[webservers:vars]
http_port=80

[webservers]
web1 http_port=8080
```
В отдельных файлах (group_vars и host_vars):

Структура каталогов:

```text
inventory/
├── hosts
├── group_vars/
│   ├── all.yml
│   └── webservers.yml
└── host_vars/
    └── web1.yml
```
Файл group_vars/webservers.yml:

```yaml
http_port: 80
max_clients: 200
```
Файл host_vars/web1.yml:

```yaml
http_port: 8080
```
Из командной строки (-e):

```bash
ansible-playbook site.yml -e "http_port=8080 version=1.2.3"
```
##### 5.3. Факты (Facts)
Ansible автоматически собирает информацию о системе (факты) перед выполнением задач. Факты доступны как переменные с префиксом ansible_.

```yaml
- hosts: all
  tasks:
    - name: Вывести информацию о системе
      debug:
        msg: "ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}"
    - name: Вывести IP-адрес
      debug:
        msg: "IP: {{ ansible_default_ipv4.address }}"
```
#### 6. Роли
Роли (Roles) — это способ организации плейбуков в модульную и повторно используемую структуру. Роли позволяют автоматически загружать связанные переменные, файлы, задачи и обработчики на основе известной файловой структуры.

##### 6.1. Структура роли
Роль имеет определённую структуру каталогов с восемью основными стандартными каталогами:

```text
roles/
└── example_role/
    ├── tasks/
    │   └── main.yml      # Основной список задач
    ├── handlers/
    │   └── main.yml      # Обработчики
    ├── defaults/
    │   └── main.yml      # Переменные по умолчанию (низкий приоритет)
    ├── vars/
    │   └── main.yml      # Переменные роли (высокий приоритет)
    ├── files/
    │   └── ...           # Файлы для развёртывания
    ├── templates/
    │   └── ...           # Шаблоны Jinja2
    ├── meta/
    │   └── main.yml      # Метаданные (зависимости)
    └── tests/
        └── ...           # Тесты
```
Назначение каталогов:

- tasks/main.yml — основной список задач, выполняемых ролью
- handlers/main.yml — обработчики, вызываемые через notify
- defaults/main.yml — переменные по умолчанию (легко переопределяются)
- vars/main.yml — переменные роли (сложнее переопределяются)
- files/ — файлы, копируемые на управляемые узлы
- templates/ — шаблоны Jinja2 для генерации конфигурационных файлов
- meta/main.yml — метаданные, включая зависимости от других ролей
- tests/ — тесты для проверки роли

##### 6.2. Создание роли
Создание структуры вручную:

```bash
mkdir -p roles/nginx/{tasks,handlers,defaults,vars,files,templates,meta}
```
Или с помощью ansible-galaxy:

```bash
ansible-galaxy init roles/nginx
```
##### 6.3. Пример простой роли: установка nginx
roles/nginx/tasks/main.yml:

```yaml
---
- name: Установить nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Скопировать конфигурацию
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Перезапустить nginx

- name: Запустить и включить nginx
  service:
    name: nginx
    state: started
    enabled: yes
```
roles/nginx/handlers/main.yml:

```yaml
---
- name: Перезапустить nginx
  service:
    name: nginx
    state: restarted
```
roles/nginx/defaults/main.yml:

```yaml
---
nginx_port: 80
nginx_worker_processes: auto
roles/nginx/templates/nginx.conf.j2:

jinja2
user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections 768;
}

http {
    server {
        listen {{ nginx_port }};
        server_name localhost;
        location / {
            root /var/www/html;
        }
    }
}
```
Использование роли в плейбуке (site.yml):

```yaml
---
- hosts: webservers
  become: yes
  roles:
    - nginx
```
Переопределение переменных роли:

```yaml
- hosts: webservers
  become: yes
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
```
#### 7. Примеры
##### 7.1. Простой пример: установка и настройка веб-сервера
Инвентарь (/etc/ansible/hosts):

```ini
[webservers]
web1 ansible_host=192.168.32.110
web2 ansible_host=192.168.32.111

[webservers:vars]
ansible_user=administrator
ansible_ssh_private_key_file=/home/user/.ssh/id_ed25519_ansible
```
Плейбук (webserver.yml):

```yaml
---
- name: Установка и настройка веб-сервера
  hosts: webservers
  become: yes
  vars:
    http_port: 80
    doc_root: /var/www/html

  tasks:
    - name: Обновить кэш пакетов
      apt:
        update_cache: yes

    - name: Установить nginx
      apt:
        name: nginx
        state: present

    - name: Создать корневой каталог
      file:
        path: "{{ doc_root }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Создать index.html
      copy:
        content: "<h1>Добро пожаловать на {{ inventory_hostname }}</h1>"
        dest: "{{ doc_root }}/index.html"
        owner: www-data
        group: www-data
        mode: '0644'

    - name: Запустить nginx
      service:
        name: nginx
        state: started
        enabled: yes
```
Запуск:

```bash
ansible-playbook -i /etc/ansible/hosts webserver.yml
```
Результат: на хостах web1 и web2 будет установлен nginx, создан каталог /var/www/html и файл index.html с именем хоста.

##### 7.2. Сложный пример: развёртывание стека LEMP с ролями
Структура проекта:

```text
lemp-project/
├── ansible.cfg
├── inventory/
│   ├── hosts
│   ├── group_vars/
│   │   ├── all.yml
│   │   ├── webservers.yml
│   │   └── dbservers.yml
│   └── host_vars/
│       └── web1.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── php/
│   └── mysql/
└── site.yml
```
ansible.cfg:

```ini
[defaults]
inventory = ./inventory/hosts
host_key_checking = False
remote_user = administrator
```
inventory/hosts:

```ini
[webservers]
web1 ansible_host=192.168.32.110
web2 ansible_host=192.168.32.111

[dbservers]
db1 ansible_host=192.168.32.120

[lemp:children]
webservers
dbservers
```
inventory/group_vars/all.yml:

```yaml
---
timezone: Europe/Moscow
ntp_servers:
  - ntp1.vniiftri.ru
  - ntp2.vniiftri.ru
```
inventory/group_vars/webservers.yml:

```yaml
---
nginx_port: 80
php_version: "8.1"
document_root: /var/www/lemp
```
inventory/group_vars/dbservers.yml:

```yaml
---
mysql_root_password: "SecurePass123!"
mysql_databases:
  - name: appdb
    encoding: utf8mb4
    collation: utf8mb4_unicode_ci
mysql_users:
  - name: appuser
    host: "%"
    password: "UserPass456!"
    priv: "appdb.*:ALL"
```
roles/common/tasks/main.yml:

```yaml
---
- name: Установить часовой пояс
  timezone:
    name: "{{ timezone }}"

- name: Установить NTP-клиент
  apt:
    name: chrony
    state: present

- name: Настроить NTP-серверы
  template:
    src: chrony.conf.j2
    dest: /etc/chrony/chrony.conf
  notify: Перезапустить chrony

- name: Установить базовые утилиты
  apt:
    name:
      - curl
      - wget
      - vim
      - htop
      - net-tools
    state: present
roles/common/templates/chrony.conf.j2:

jinja2
{% for server in ntp_servers %}
server {{ server }} iburst
{% endfor %}

driftfile /var/lib/chrony/drift
rtcsync
makestep 1.0 3
```
roles/nginx/tasks/main.yml:

```yaml
---
- name: Установить nginx
  apt:
    name: nginx
    state: present

- name: Создать корневой каталог
  file:
    path: "{{ document_root }}"
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'

- name: Настроить виртуальный хост
  template:
    src: vhost.conf.j2
    dest: /etc/nginx/sites-available/lemp
  notify: Перезапустить nginx

- name: Активировать виртуальный хост
  file:
    src: /etc/nginx/sites-available/lemp
    dest: /etc/nginx/sites-enabled/lemp
    state: link
  notify: Перезапустить nginx

- name: Удалить стандартный сайт
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: Перезапустить nginx

- name: Запустить nginx
  service:
    name: nginx
    state: started
    enabled: yes
```
roles/nginx/handlers/main.yml:

```yaml
---
- name: Перезапустить nginx
  service:
    name: nginx
    state: restarted
```
roles/php/tasks/main.yml:

```yaml
---
- name: Установить PHP и модули
  apt:
    name:
      - "php{{ php_version }}-fpm"
      - "php{{ php_version }}-mysql"
      - "php{{ php_version }}-cli"
      - "php{{ php_version }}-curl"
      - "php{{ php_version }}-gd"
    state: present
  notify: Перезапустить php-fpm

- name: Настроить PHP-FPM
  template:
    src: php-fpm.conf.j2
    dest: "/etc/php/{{ php_version }}/fpm/pool.d/www.conf"
  notify: Перезапустить php-fpm

- name: Запустить php-fpm
  service:
    name: "php{{ php_version }}-fpm"
    state: started
    enabled: yes
```
roles/php/handlers/main.yml:

```yaml
---
- name: Перезапустить php-fpm
  service:
    name: "php{{ php_version }}-fpm"
    state: restarted
```
roles/mysql/tasks/main.yml:

```yaml
---
- name: Установить MySQL
  apt:
    name:
      - mysql-server
      - python3-pymysql
    state: present

- name: Запустить MySQL
  service:
    name: mysql
    state: started
    enabled: yes

- name: Установить пароль root
  mysql_user:
    name: root
    password: "{{ mysql_root_password }}"
    login_unix_socket: /var/run/mysqld/mysqld.sock
    state: present

- name: Создать базы данных
  mysql_db:
    name: "{{ item.name }}"
    encoding: "{{ item.encoding }}"
    collation: "{{ item.collation }}"
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"
  loop: "{{ mysql_databases }}"

- name: Создать пользователей
  mysql_user:
    name: "{{ item.name }}"
    host: "{{ item.host }}"
    password: "{{ item.password }}"
    priv: "{{ item.priv }}"
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"
  loop: "{{ mysql_users }}"
```
site.yml:

```yaml
---
- name: Общая настройка всех серверов
  hosts: lemp
  become: yes
  roles:
    - common

- name: Настройка веб-серверов
  hosts: webservers
  become: yes
  roles:
    - nginx
    - php

- name: Настройка серверов баз данных
  hosts: dbservers
  become: yes
  roles:
    - mysql
```
Запуск полного развёртывания:

```bash
ansible-playbook site.yml
```
Запуск только для веб-серверов:

```bash
ansible-playbook site.yml --limit webservers
```
Проверка без выполнения (dry-run):

```bash
ansible-playbook site.yml --check
```
##### 7.3. Пример использования переменных с разными уровнями приоритета
inventory/group_vars/all.yml:

```yaml
---
app_port: 3000
app_env: production
```
inventory/group_vars/webservers.yml:

```yaml
---
app_port: 8080
```
inventory/host_vars/web1.yml:

```yaml
---
app_port: 9090
```
Плейбук (priority.yml):

```yaml
---
- hosts: webservers
  become: yes
  vars:
    app_port: 4000  # Переопределяет group_vars и host_vars

  tasks:
    - name: Вывести порт приложения
      debug:
        msg: "Порт: {{ app_port }}, Среда: {{ app_env }}"
```
Запуск:

```bash
# Без переопределения — используется app_port из vars плейбука (4000)
ansible-playbook priority.yml

# С переопределением через extra vars — используется 5000
ansible-playbook priority.yml -e "app_port=5000"
```
Результат: для web1 будет выведено Порт: 5000 (extra vars имеет высший приоритет), для web2 — также Порт: 5000.

#### 8. Устранение распространённых проблем
Симптом	                    | Вероятная причина	                | Решение
--------------------------- | --------------------------------- | ------------------------------------
UNREACHABLE	 | Нет SSH-доступа к хосту	 | Проверить настройки SSH, ключи, доступность порта
Permission denied	 | Неверные учётные данные	 | Проверить ansible_user и ключи
Missing sudo password	 | Не передан пароль для become	 | Использовать -K или настроить NOPASSWD в sudoers
Module not found	 | Модуль не установлен	 | Установить коллекцию: ansible-galaxy collection install <collection>
Роль не найдена	 | Неверный путь к ролям	 | Указать roles_path в ansible.cfg
Переменные не применяются	 | Неверный приоритет	 | Проверить иерархию переменных
Плейбук не запускается	 | Ошибка синтаксиса YAML	 | Проверить отступы и синтаксис: ansible-playbook --syntax-check site.yml
