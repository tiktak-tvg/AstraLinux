### Веб-сервер на основе Apache в Astra Linux
#### 1. Основы протокола HTTP
HTTP (HyperText Transfer Protocol) — протокол прикладного уровня, лежащий в основе обмена данными в World Wide Web. Он работает по клиент-серверной модели: клиент (браузер) отправляет запрос, сервер возвращает ответ. HTTP — протокол без сохранения состояния (stateless), то есть каждый запрос независим от предыдущих.

Основные методы HTTP:

Метод	          | Назначение
--------------- | ----------------------------------
GET	 | Запрос ресурса (страницы, файла)
POST	 | Отправка данных на сервер (формы, загрузка файлов)
PUT	 | Загрузка ресурса на сервер
DELETE	 | Удаление ресурса
HEAD	 | Запрос заголовков без тела ответа

Структура HTTP-запроса:

```text
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
Accept: text/html
```
Структура HTTP-ответа:

```text
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...</html>
```
Коды состояния HTTP:

Код	            | Значение
--------------- | ----------------------------------
200	 | OK — успешный запрос
301/302	 | Перенаправление
403	 | Доступ запрещён
404	 | Ресурс не найден
500	 | Внутренняя ошибка сервера

HTTP работает поверх TCP (обычно порт 80). Для шифрования используется HTTPS (HTTP over TLS) на порту 443.

#### 2. Установка веб-сервера и утилиты управления Apache
##### 2.1. Установка
В Astra Linux пакет Apache2 по умолчанию не устанавливается, но может быть выбран на этапе установки ОС или установлен отдельно:

```bash
sudo apt update
sudo apt install apache2
```
Для использования аутентификации Kerberos дополнительно устанавливается модуль:

```bash
sudo apt install libapache2-mod-auth-kerb
```
##### 2.2. Управление службой
После установки службу необходимо запустить и включить автозагрузку:

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
```
Проверка состояния:

```bash
sudo systemctl status apache2
```
##### 2.3. Утилиты управления

Утилита	        | Назначение
--------------- | ----------------------------------
a2enmod / a2dismod	 | Включение / отключение модулей
a2ensite / a2dissite	 | Включение / отключение виртуальных хостов
a2enconf / a2disconf	 | Включение / отключение конфигурационных файлов
apache2ctl	 | Управление сервером (start, stop, restart, configtest)

#### 3. Конфигурационные файлы Apache
Конфигурация Apache2 в Astra Linux (как и в Debian/Ubuntu) организована по модульному принципу:

Файл / Каталог	        | Назначение
----------------------- | ----------------------------------
/etc/apache2/apache2.conf	 | Главный конфигурационный файл (глобальные настройки)
/etc/apache2/ports.conf	 | Слушаемые порты (Listen 80)
/etc/apache2/sites-available/	 | Доступные виртуальные хосты
/etc/apache2/sites-enabled/	 | Активированные виртуальные хосты (символические ссылки)
/etc/apache2/mods-available/	 | Доступные модули
/etc/apache2/mods-enabled/	 | Активированные модули
/etc/apache2/conf-available/	 | Дополнительные конфигурации
/var/www/html/	 | Каталог веб-сайта по умолчанию

Главный файл apache2.conf включает в себя остальные конфигурации через директивы Include и IncludeOptional.

#### 4. Базовая настройка веб-сервера
Основные директивы базовой настройки:

Директива     	        | Назначение     | Пример
----------------------- | -------------- | ----------------
Listen	 | Порт прослушивания	 | Listen 80
ServerName	 | Основное доменное имя сервера	 | ServerName server.domain.name
ServerAlias	 | Дополнительные имена (алиасы)	 | ServerAlias www.server.domain.name
ServerAdmin	 | Email администратора	 | ServerAdmin webmaster@localhost
DocumentRoot	 | Корневой каталог сайта	 | DocumentRoot /var/www/html

Минимальная конфигурация виртуального хоста:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName server.domain.name
    ServerAlias www.server.domain.name
    DocumentRoot /var/www/html
    <Directory /var/www/html>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride None
        Require all granted
    </Directory>
    ErrorLog /var/log/apache2/error.log
    LogLevel warn
    CustomLog /var/log/apache2/access.log combined
</VirtualHost>
```
После изменения конфигурации необходимо перезапустить сервер:

```bash
sudo systemctl restart apache2
```
#### 5. Настройка виртуального хостинга
Виртуальный хостинг позволяет размещать несколько сайтов на одном сервере. Apache поддерживает два типа:
```txt
Name-based — на одном IP-адресе, сайты различаются по имени (директива ServerName)
IP-based — на разных IP-адресах
```
Пошаговая настройка:

Создать каталог для нового сайта:

```bash
sudo mkdir -p /var/www/example.com/public_html
sudo chown -R www-data:www-data /var/www/example.com
```
Создать конфигурационный файл виртуального хоста в /etc/apache2/sites-available/example.com.conf:

```apache
<VirtualHost *:80>
    ServerAdmin admin@example.com
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example.com/public_html
    <Directory /var/www/example.com/public_html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/example.com_error.log
    CustomLog ${APACHE_LOG_DIR}/example.com_access.log combined
</VirtualHost>
```
Активировать сайт:

```bash
sudo a2ensite example.com.conf
```
Отключить сайт по умолчанию (при необходимости):

```bash
sudo a2dissite 000-default.conf
```
Проверить конфигурацию и перезапустить:

```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
```
#### 6. Управление модулями Apache
Apache2 использует модульную архитектуру. Модули хранятся в /etc/apache2/mods-available/ и активируются через символические ссылки в /etc/apache2/mods-enabled/.

Управление модулями:

```bash
# Включить модуль
sudo a2enmod rewrite

# Отключить модуль
sudo a2dismod rewrite

# Просмотр списка доступных модулей
ls /etc/apache2/mods-available/

# Просмотр активных модулей
apache2ctl -M
```
a2enmod создаёт символическую ссылку в mods-enabled, a2dismod — удаляет её. После изменения модулей необходимо перезапустить Apache.

Наиболее используемые модули:

Модуль     	            | Назначение   
----------------------- | -------------- 
rewrite	 | Перезапись URL
ssl	     | Поддержка HTTPS
headers	 | Управление HTTP-заголовками
proxy	   | Проксирование
auth_kerb	  | Аутентификация Kerberos
authz_core	 | Авторизация

#### 7. Интеграция Apache2 и FreeIPA
При работе в составе домена FreeIPA веб-сервер Apache2 может использоваться для аутентификации пользователей с использованием Kerberos, в том числе для сквозной аутентификации (SSO) в приложениях.

##### 7.1. Настройка на контроллере домена FreeIPA
Действуя от имени администратора домена, необходимо добавить службу HTTP и выгрузить keytab:

```bash
# Добавить службу HTTP
ipa service-add HTTP/client.astra.domain@ASTRA.DOMAIN

# Выгрузить keytab
ipa-getkeytab -p HTTP/client.astra.domain@ASTRA.DOMAIN -k /tmp/http.keytab
```
Полученный файл http.keytab копируется на веб-сервер в каталог /etc/apache2/.

##### 7.2. Настройка на веб-сервере
Установить необходимые пакеты:

```bash
sudo apt install apache2 libapache2-mod-auth-kerb
```
Скопировать keytab и выставить права:

```bash
sudo chown www-data /etc/apache2/http.keytab
sudo chmod 644 /etc/apache2/http.keytab
```
Настроить конфигурацию виртуального хоста:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <Directory /var/www>
        AuthType Kerberos
        KrbAuthRealms ASTRA.DOMAIN
        KrbServiceName HTTP/client.astra.domain@ASTRA.DOMAIN
        Krb5Keytab /etc/apache2/http.keytab
        KrbMethodNegotiate on
        KrbMethodK5Passwd off
        require valid-user
        KrbSaveCredentials on
    </Directory>
</VirtualHost>
```
Параметры аутентификации Kerberos:

Параметр     	          | Описание  
----------------------- | -------------- 
AuthType Kerberos	 | Тип аутентификации
KrbAuthRealms	 | Имя реалма Kerberos (домен заглавными буквами)
KrbServiceName	 | Полное доменное имя сервера
Krb5Keytab	 | Путь к keytab-файлу
KrbMethodNegotiate	 | Использовать Negotiate-аутентификацию
KrbMethodK5Passwd	 | Разрешить ввод пароля (обычно off для SSO)
```
Перезапустить Apache:

```bash
sudo systemctl restart apache2
```
#### 8. Поддержка мандатного доступа в Apache2
Apache2 в составе Astra Linux Special Edition поддерживает работу с данными, имеющими различные (ненулевые) классификационные метки. Для управления обязательной авторизацией используется параметр AstraMode.

##### 8.1. Параметр AstraMode
Параметр задаётся в файле /etc/apache2/apache2.conf:

```apache
# Включено (по умолчанию) — обязательная авторизация
AstraMode on

# Выключено — без обязательной авторизации
AstraMode off
```
> Важно: начиная с обновления Astra Linux Special Edition x.7.1, при включённом AstraMode авторизация пользователей требуется даже при выключенном мандатном управлении доступом (МРД).

При значении off сервер Apache2 осуществляет все запросы к своим ресурсам от имени одной системной учётной записи (по умолчанию www-data).

##### 8.2. Разграничение доступа к содержимому
Разграничение доступа к содержимому веб-сайтов осуществляется за счёт мандатного управления доступом при включённом параметре AstraMode. Доступ пользователя к содержимому веб-сайта определяется сопоставлением классификационных меток файлов веб-сервера с классификационной меткой пользователя. Файлы с ненулевыми метками доступны только из сессии пользователя с соответствующей меткой.

Назначение меток каталогам:

```bash
# Назначить каталогу максимальную метку с атрибутом ccnr
sudo pdpl-file -u 2::-1:ccnr /var/www
sudo pdpl-file -u 2::-1:ccnr /var/www/html
```
Атрибут ccnr позволяет файлам и подкаталогам иметь различные метки (в том числе нулевые), но не выше метки каталога. Это даёт возможность создать заглавную страницу с нулевой меткой, видимую всем, и ограничить доступ к остальному содержимому.

Пример назначения меток:

```bash
# Каталогу — второй уровень конфиденциальности, все категории, ccnr
sudo pdpl-file -u 2::-1:ccnr /var/www/html/restricted

# Файлу — первый уровень и конкретная категория
sudo pdpl-file -u 1::Категория_1 /var/www/html/restricted/secret.html
```
##### 8.3. Требования к правам www-data
Для корректной работы авторизации через PAM пользователю www-data необходимо выдать права на чтение информации из БД пользователей и сведений о мандатных метках:

```bash
sudo usermod -a -G shadow www-data
sudo setfacl -d -m u:www-data:r /etc/parsec/macdb
sudo setfacl -R -m u:www-data:r /etc/parsec/macdb
sudo setfacl -m u:www-data:rx /etc/parsec/macdb
```
##### 8.4. Проверка работы мандатных ограничений
- Создать доменного пользователя с ненулевым максимальным уровнем конфиденциальности (например, 1).
- Создать в каталоге веб-сайта два файла: один с нулевой меткой, другой — с ненулевой.
- При попытке доступа пользователем с недостаточной меткой к файлу с более высокой меткой веб-сервер вернёт ошибку 404.
