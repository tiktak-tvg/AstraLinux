### Как удалить postgresql 11

#### 1. Остановите службы PostgreSQL
```bash
sudo systemctl stop postgresql
```
#### 2. Удалите пакеты PostgreSQL
#####  Удалите основной пакет и зависимости
```bash
sudo apt purge postgresql postgresql-11 postgresql-client-11
```
#####  Или удалите все пакеты, связанные с postgresql
```bash
sudo apt purge postgresql*
```
#### 3. Удалите оставшиеся файлы и каталоги
#####  Удалите каталог с данными
```bash
sudo rm -rf /var/lib/postgresql/
```
#####  Удалите конфигурационные файлы
```bash
sudo rm -rf /etc/postgresql/
```
#####  Удалите файлы логов
```bash
sudo rm -rf /var/log/postgresql/
```
#### 4. Удалите пользователя и группу postgres (опционально)
```bash
sudo userdel postgres
sudo groupdel postgres
```
#### 5. Очистите систему от ненужных зависимостей
```bash
sudo apt autoremove
sudo apt autoclean
```
#### 6. Проверьте, что всё удалено
##### Проверьте установленные пакеты
```bash
dpkg -l | grep postgres
```
#####  Проверьте наличие оставшихся файлов
```bash
sudo find / -name "*postgres*" 2>/dev/null
```
#### Альтернативный способ - одной командой:
```bash
sudo apt purge postgresql\* && sudo rm -rf /var/lib/postgresql/ /etc/postgresql/ /var/log/postgresql/ && sudo apt autoremove
```
После выполнения этих команд PostgreSQL 11 будет полностью удалён из вашей системы.


### Как правильно установить PostgreSQL 14 из исходного кода

#### 1. Установите необходимые зависимости
Для Debian/Ubuntu:
```bash
sudo apt update
sudo apt install gcc make libreadline-dev zlib1g-dev
```
Для CentOS/RHEL:
```bash
sudo yum install gcc make readline-devel zlib-devel
```
#####   или для newer версий:
```bash
sudo dnf install gcc make readline-devel zlib-devel
```
#### 2. Подготовка к установке
#####   Перейдите в директорию с исходным кодом
```bash
cd /mnt/postgresql-14.0
```
#####   Создайте директорию для установки (лучше использовать стандартные пути)
```bash
sudo mkdir -p /usr/local/pgsql
```
#### 3. Конфигурация и компиляция
#####   Настройка с правильным префиксом
```bash
./configure --prefix=/usr/local/pgsql
```
#####   Компиляция
```bash
make
```
#####   Установка
```bash
sudo make install
```
#### 4. Создание пользователя и директорий данных
#####   Создайте пользователя postgres
```bash
sudo useradd -r -s /bin/bash postgres
```
#####   Создайте директорию для данных
```bash
sudo mkdir -p /usr/local/pgsql/data
sudo chown postgres:postgres /usr/local/pgsql/data
```
#### 5. Инициализация базы данных
#####   Переключитесь на пользователя postgres
```bash
sudo su - postgres
```
#####   Инициализируйте кластер баз данных
```bash
/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
```
#####   Запустите PostgreSQL
```bash
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile start
```
#### 6. Настройка переменных окружения
Добавьте в /etc/profile или ~/.bashrc:
```bash
export PATH=/usr/local/pgsql/bin:$PATH
export PGDATA=/usr/local/pgsql/data
```
#### Альтернативный вариант (рекомендуется)
Если вы используете Debian/Ubuntu, проще установить из официальных репозиториев:
#####   Добавление официального репозитория PostgreSQL
```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
```
#####   Установка PostgreSQL 14
```bash
sudo apt install postgresql-14
```
Если проблемы сохраняются

Проверьте наличие всех зависимостей:

#####   Для полной установки могут потребоваться дополнительные пакеты
```bash
sudo apt install build-essential libreadline-dev zlib1g-dev flex bison libxml2-dev libxslt-dev libssl-dev
```
После успешной установки не забудьте настроить аутентификацию и сетевые настройки в файле ``pg_hba.conf`` и ``postgresql.conf``.


































