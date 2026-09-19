### Установка операционной системы Astra Linux Special Edition по сети 
Это стандартная практика для централизованного развертывания системы на множестве машин. Этот процесс требует предварительной подготовки сервера, который будет выступать в роли установочного узла. Ниже приведено подробное руководство по настройке всех необходимых компонентов.

#### 🖥️ Настройка HTTP-сервера репозитория
Для установки ОС по сети необходимо, чтобы клиентские машины имели доступ к пакетам установки. Для этого на сервере поднимается HTTP-сервер, который будет отдавать файлы репозитория.

Установка Apache2: Если веб-сервер еще не установлен, его нужно установить.

```bash
sudo apt install apache2
```
Создание и монтирование репозитория: Создайте директорию для репозитория и примонтируйте в неё установочный диск (ISO-образ или физический носитель).

```bash
sudo mkdir -p /srv/repo/smolensk/main
sudo mount /dev/sr0 /srv/repo/smolensk/main
```
Настройка доступа через Apache: Создайте символьную ссылку на директорию с репозиторием в корневом каталоге веб-сервера и настройте права доступа.

```bash
sudo ln -s /srv/repo /var/www/html/repo
```
Затем в конфигурационном файле вашего сайта (например, /etc/apache2/sites-enabled/000-default.conf) добавьте директиву для разрешения листинга и доступа к этой директории.

```apache
<Directory /var/www/html/repo>
    Options Indexes MultiViews FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```
Перезапуск службы: После внесения изменений перезапустите Apache.

```bash
sudo systemctl restart apache2
```
#### 📡 Настройка TFTP-сервера
**TFTP-сервер** используется для передачи клиентским машинам загрузочных файлов: ядра (linux) и начального образа оперативной памяти ``(initrd.gz)``.

Установка пакетов: Установите tftpd-hpa, а также пакеты для поддержки PXE-загрузки.

```bash
sudo apt install tftpd-hpa pxelinux syslinux
```
Конфигурация: Основные настройки хранятся в файле ``/etc/default/tftpd-hpa``. Убедитесь, что он указывает на директорию ``/srv/tftp``.

```text
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--ipv4 --secure --create --umask 027 --permissive"
```
Размещение загрузочных файлов: Создайте каталог ``/srv/tftp/`` и скопируйте туда необходимые файлы с установочного диска. Также скопируйте файлы загрузчика ``pxelinux.0`` и его библиотеки.

```bash
sudo mkdir /srv/tftp/smolensk
sudo cp /srv/repo/smolensk/main/netinst/linux /srv/tftp/smolensk/
sudo cp /srv/repo/smolensk/main/netinst/initrd.gz /srv/tftp/smolensk/

sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/
sudo cp /usr/lib/syslinux/modules/bios/{chain.c32,ldlinux.c32,libcom32.c32,libutil.c32,menu.c32} /srv/tftp/
```
Перезапуск службы:

```bash
sudo systemctl restart tftpd-hpa
```
#### 🌐 Настройка DHCP-сервера
DHCP-сервер сообщает клиентским машинам их сетевые параметры (IP-адрес, шлюз, DNS) и указывает, где искать загрузочные файлы.

Установка: Установите isc-dhcp-server.

```bash
sudo apt install isc-dhcp-server
```
Указание интерфейса: В файле ``/etc/default/isc-dhcp-server`` укажите сетевой интерфейс, на котором будет работать DHCP-сервер (например, eth0).

```text
INTERFACESv4="eth0"
```
Настройка dhcpd.conf: Отредактируйте файл ``/etc/dhcp/dhcpd.conf``, добавив параметры для поддержки сетевой загрузки.

```text
authoritative;
option domain-name "my.dom";
default-lease-time 600;
max-lease-time 7200;
log-facility local7;

allow booting;
allow bootp;

# Указываем TFTP-сервер
next-server 192.168.56.1;

# Определяем тип архитектуры для выдачи правильного загрузчика
option architecture code 93 = unsigned integer 16;
if option architecture = 00:07 {
    filename "bootx64.efi"; # Для UEFI
} elsif option architecture = 00:09 {
    filename "bootx64.efi"; # Для UEFI
} else {
    filename "pxelinux.0"; # Для Legacy BIOS
}

subnet 192.168.56.0 netmask 255.255.255.0 {
    range 192.168.56.20 192.168.56.250;
    option routers 192.168.56.1;
    option subnet-mask 255.255.255.0;
}
```
Перезапуск службы:

```bash
sudo systemctl restart isc-dhcp-server
```
#### 📝 Подготовка файла с автоматическими ответами
Для полностью автоматической установки используется файл ответов (preseed). В Astra Linux Special Edition x.8 используется новый установщик astra-installer, который принимает файлы в формате YAML, в то время как более старые версии (1.7 и ранее) использовали формат debian-installer.

Для Astra Linux SE x.8 (и новее): Файл ответов имеет формат YAML и должен быть размещен в HTTP-репозитории. Пример структуры файла и его параметров можно найти в официальной документации.

```yaml
# Пример фрагмента файла astra-installer-preseed.yaml
license:
  accepted: true
language:
  locale: ru_RU.UTF-8
  keymap: ru
# ... и так далее
``
Для Astra Linux SE 1.7 (и старее): Используется файл preseed.cfg в формате deb. В нём указываются параметры, такие как путь к репозиторию.

```text
# Пример для debian-installer
d-i mirror/protocol string http
d-i mirror/http/hostname string 192.168.56.1
d-i mirror/http/directory string /repo/smolensk
```
Для удобства в состав Astra Linux SE x.8 входит утилита astra-installer-converter, которая помогает конвертировать старые preseed-файлы в новый формат YAML. 

#### 📂 Настройка доступа к репозиторию
Чтобы установщик мог получить пакеты, путь к HTTP-репозиторию должен быть передан в загрузочных параметрах ядра. Это делается в конфигурационном файле PXE-меню (``pxelinux.cfg/default`` для BIOS или в файлах GRUB для UEFI) на TFTP-сервере.

В строке загрузки ядра (append) указывается параметр, например:

```text
url=http://192.168.56.1/repo/smolensk/preseed.cfg
```
