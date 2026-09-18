### Система управления идентификацией (IdM) — FreeIPA в Astra Linux
#### 1. Архитектура и компоненты FreeIPA
FreeIPA (Free Identity, Policy and Audit) — открытый проект централизованной системы управления идентификацией пользователей, политиками доступа и аудита. В контексте Astra Linux FreeIPA представляет собой систему организации единого пространства пользователей (ЕПП).

Основные компоненты:

Компонент	       | Назначение
---------------- | --------------------------
389 Directory Server	 | Служба каталогов (LDAP) для хранения учётных записей, групп, хостов и сервисов
MIT Kerberos	 | Служба аутентификации и единой точки входа (SSO)
BIND и DHCP	 | Управление службой DNS в сети
DogTag	 | Служба управления сертификатами (опционально)
Apache + Python	 | Веб-интерфейс управления
Samba	 | Интеграция с Microsoft Active Directory и поддержка Windows/macOS

Служба FreeIPA состоит из ядра, отвечающего за основной функционал, ряда интерфейсов (LDAP, Kerberos, Config, RPC) и модулей расширения.

Важные требования при установке в Astra Linux:

- Сервис FreeIPA необходимо устанавливать на чистую ОС, на которой ранее не были установлены другие сервисы.
- Работа FreeIPA в Astra Linux Special Edition осуществляется только при отключённом режиме AstraMode web-сервера Apache2.
- Доменное имя не должно быть именем первого уровня (нельзя использовать имена, состоящие из одного слова, например domain, testdomain).
- В имени домена нельзя использовать кириллицу.

#### 2. Обзор основных протоколов FreeIPA
##### 2.1. LDAP
LDAP (Lightweight Directory Access Protocol) — основной протокол службы каталогов 389 Directory Server. FreeIPA использует LDAP-based хранилище для общих объектов: пользователей, групп, хостов, сервисов.

##### 2.2. Kerberos
Kerberos — сетевой протокол аутентификации, обеспечивающий единую точку входа (SSO). FreeIPA использует MIT Kerberos KDC (Key Distribution Center) для выдачи билетов и аутентификации пользователей.

##### 2.3. SMB
SMB (Server Message Block) — протокол, используемый для интеграции с Microsoft Active Directory и поддержки файловых серверов Samba. FreeIPA предоставляет возможность настройки внутреннего Samba-сервера с использованием бэкенда ipasam, который позволяет Samba получать доступ к данным и хранить их в LDAP-каталоге FreeIPA. Эти сервисы доступны через транспорт SMB/SMB2 (порт 445).

#### 3. Установка и начальная настройка сервера FreeIPA
##### 3.1. Подготовка системы
```bash
# Установка FQDN имени
sudo hostnamectl hostname dc1.ipa.lc

# Добавление записи в /etc/hosts
# <IP-адрес> dc1.ipa.lc dc1

# Отключение AstraMode Apache2
sudo sed -i 's/^.*AstraMode\s*.*/AstraMode off/' /etc/apache2/apache2.conf
sudo systemctl restart apache2
```
##### 3.2. Установка пакетов
```bash
sudo apt install fly-admin-freeipa-server astra-freeipa-server
```
##### 3.3. Инициализация контроллера домена
```bash
sudo astra-freeipa-server -o -n dc1 -d ipa.lc
```
Параметры: -o — режим для изолированной сети (без шлюза/DNS), -n — краткое имя узла, -d — доменное имя. Команда запросит подтверждение и пароль администратора FreeIPA.

После успешной инициализации необходимо перезагрузить компьютер.

Требования к ресурсам: не менее 2 ГБ ОЗУ и 3 процессоров.

#### 4. Ввод клиентского хоста в домен FreeIPA
##### 4.1. Установка клиентской части
```bash
sudo apt install astra-freeipa-client
```
Достаточно установить пакет astra-freeipa-client, все остальные пакеты будут установлены автоматически.

##### 4.2. Ввод в домен
```bash
sudo ipa-client-install --mkhomedir --enable-dns-updates
```
Для установки в пакетном режиме:

```bash
sudo ipa-client-install -U -p admin -w пароль_администратора
```
Параметры:
```txt
--mkhomedir — автоматическое создание домашних каталогов;

-U — автоматический режим (без вопросов);

-p — имя администратора домена;

-w — пароль администратора.
```
##### 4.3. Проверка ввода в домен
```bash
kinit admin
id admin
getent passwd admin
```
#### 5. Установка реплики FreeIPA
Репликация обеспечивает отказоустойчивость и балансировку нагрузки. Реплика — это точная копия оригинального сервера, и изменения, внесённые в любой мастер, автоматически реплицируются на другие.

##### 5.1. Подготовка основного сервера
```bash
# Проверка работоспособности
kinit admin

# Установка службы DogTag (если не была установлена)
ipa-ca-install
ipa-kra-install
```
##### 5.2. Настройка сервера-реплики
```bash
# Установка пакетов
sudo apt install astra-freeipa-server astra-freeipa-client

# Установка реплики
ipa-replica-install
```
Параметры для реплики:

FQDN: replica.ipadomain.ru

IP-адрес: должен быть фиксированным

Ресурсы: не менее 2 ГБ ОЗУ (при DogTag — не менее 4 ГБ)

##### 5.3. Проверка репликации
```bash
ipa-replica-manage list
ipa-replica-manage status
```
#### 6. Управление пользователями и группами
##### 6.1. Управление пользователями
Создание пользователя:

```bash
ipa user-add iivanov --first="Иван" --last="Иванов" --password
```
Просмотр списка пользователей:

```bash
ipa user-find
```
Просмотр информации о пользователе:

```bash
ipa user-show iivanov
```
Изменение пользователя:

```bash
ipa user-mod iivanov --email=iivanov@example.com
```
Удаление пользователя:

```bash
ipa user-del iivanov
```
##### 6.2. Управление группами
Создание группы:

```bash
ipa group-add developers --desc="Группа разработчиков"
```
Добавление пользователя в группу:

```bash
ipa group-add-member developers --users=iivanov
```
Просмотр групп:

```bash
ipa group-find
```
#### 7. Ограничение использования сервисов с помощью HBAC-правил
HBAC (Host-Based Access Control) — набор правил для настройки доступа пользователей или групп пользователей к определённым хостам с использованием определённых сервисов.

Примеры применения:

- ограничение доступа по SSH к контроллеру домена только для группы администраторов;
- разрешение использовать только определённую службу для определённых пользователей на определённых хостах.

> Важно: правила предоставляют только разрешения доступа. Правила запрета доступа настроить невозможно. В домене FreeIPA по умолчанию установлено правило allow_all, которое после настройки своих правил следует отключить.

##### 7.1. Создание правила
В веб-интерфейсе: Policy → Host-Based Access Control → HBAC Rules → Add.

Указываются:

- Пользователи и/или группы;
- Узлы и/или группы узлов;
- Службы и/или группы служб.

##### 7.2. Службы HBAC
В разделе Policy → Host-Based Access Control → HBAC Services перечислены службы: ftp, sshd, su, login. Для разрешения входа в графический интерфейс необходимо создать службы fly-dm и fly-wm.

##### 7.3. Проверка правила
Используется инструмент Policy → Host-Based Access Control → HBAC Test: выбирается пользователь, узел и служба для проверки применения правила.

#### 8. Управление централизованным хранением правил SUDO
FreeIPA позволяет настраивать правила разрешения и запрета использования sudo для пользователей и групп пользователей.

##### 8.1. Добавление команд
Policy → Sudo → Sudo Commands → Add. Указывается полный путь к команде.

##### 8.2. Создание правила
``Policy → Sudo → Sudo Rules → Add``. Задаётся имя правила.

##### 8.3. Настройка правила

- Options — параметры sudo (например, не запрашивать пароль);
- Who — пользователи и группы, на которые применяется правило;
- Access this host — узлы и группы узлов;
- Run Commands — разрешённые или запрещённые команды;
- As Whom — от имени какого пользователя выполняется команда.

> Важно: правила sudo не могут применяться к встроенной группе хостов ipaserver. Для немедленного применения правил необходимо очистить кэш SSSD:

```bash
systemctl stop sssd
rm /var/lib/sss/db/*
systemctl start sssd
```
#### 9. Интеграция FreeIPA с файловым сервером SAMBA
##### 9.1. Подготовка контроллера домена
```bash
sudo apt install freeipa-server-trust-ad libwbclient-sssd samba smbclient

# Добавление сервиса CIFS
sudo ipa service-add cifs/fs.astra.loc

# Добавление прав для Samba
ipa permission-add "CIFS server can read user passwords" \
  --attrs={ipaNTHash,ipaNTSecurityIdentifier} --type=user \
  --right={read,search,compare} --bindtype=permission
ipa privilege-add "CIFS server privilege"
ipa privilege-add-permission "CIFS server privilege" \
  --permission="CIFS server can read user passwords"
ipa role-add "CIFS server"
ipa role-add-privilege "CIFS server" --privilege="CIFS server privilege"
```
##### 9.2. Настройка файлового сервера
Файловый сервер должен быть клиентом домена FreeIPA. На нём выполняется:

```bash
sudo apt install astra-freeipa-client
sudo astra-freeipa-client -d ipadomain.ru
```
##### 9.3. Проверка аутентификации
```bash
kinit admin
smbclient --use-kerberos=required //ipa0.ipadomain.ru/admin
```
#### 10. Настройка сервисов для аутентификации через домен FreeIPA
##### 10.1. Apache2 с аутентификацией Kerberos
На контроллере домена:

```bash
# Добавление службы HTTP
ipa service-add HTTP/client.astra.domain@ASTRA.DOMAIN

# Выгрузка keytab
ipa-getkeytab -s dc1.astra.domain -p HTTP/client.astra.domain@ASTRA.DOMAIN -k /etc/apache2/http.keytab
```
Рекомендуемый модуль: libapache2-mod-auth-gssapi.

##### 10.2. SSH с аутентификацией Kerberos
Для включения GSSAPI-аутентификации в SSH-клиенте необходимо раскомментировать и изменить соответствующие строки в /etc/ssh/ssh_config.

#### 11. Реплицирование сервера FreeIPA
##### 11.1. Регистрация реплики на основном сервере
На основном сервере FreeIPA необходимо зарегистрировать реплику. В поле Hostname указывается имя сервера-реплики replica.ipadomain.ru.

##### 11.2. Настройка реплики
```bash
# Установка пакетов
sudo apt install astra-freeipa-server astra-freeipa-client

# Установка реплики
ipa-replica-install
```
##### 11.3. Проверка
```bash
# На основном сервере
ipa-replica-manage list
ipa-replica-manage status

# Проверка синхронизации
kinit admin
ipa user-find
```
#### 12. Интеграция с Microsoft Active Directory
##### 12.1. Предварительные условия
- Имеется домен Active Directory (AD) с контроллером домена;
- Имеется сервер FreeIPA с настроенным доменом;
- Серверы находятся в одной сети, часы синхронизированы;
- Администратор AD имеет имя Administrator (кириллица не поддерживается).

##### 12.2. Установка службы доверительных отношений
Если при установке FreeIPA не была применена опция --setup-adtrust:

```bash
sudo apt install freeipa-server-trust-ad
sudo ipa-adtrust-install --add-sids --add-agents
```
##### 12.3. Создание доверительных отношений
```bash
ipa trust-add --type=ad windomain.ad --admin Administrator --password
```
Важно: область FreeIPA доверяет лесу доменов Active Directory, используя механизм доверительных отношений между деревьями. Дерево доменов AD не доверяет области FreeIPA (одностороннее доверие).

##### 12.4. Проверка
```bash
ipa trust-show windomain.ad
```
