### Процесс загрузки и выключения системы
1. Исследование порядка и стадий начальной загрузки;
2. Работа с BIOS и EFI;
3. Настройка загрузчика GRUB2;
4. Загрузка ядра ОС, параметры, передаваемые ядру;
5. Загрузка и управление модулями ядра;
6. Управление службами через systemd;
7. Управление целевыми состояниями системы через systemd;
8. Создание собственных юнитов systemd;
9. Запуск служб с мандатными атрибутами.

#### Практическая работа:
Загрузка в режиме single Astra Linux, с использованием командной строки GRUB, смена пароля и таймаута у GRUB. Создание unit (типа service) для включения маршрутизации в ядре.

#### 1. Исследование порядка и стадий начальной загрузки
Процесс загрузки Linux состоит из нескольких последовательных этапов:

*BIOS/UEFI* — выполняет самотестирование (POST), инициализирует оборудование и находит загрузочное устройство.

*Загрузчик (GRUB2)* — загружается из MBR или EFI-раздела, предоставляет меню выбора ОС, загружает ядро и initramfs в память.

*Ядро (vmlinuz)* — распаковывается, инициализирует оборудование, монтирует initramfs.

*initramfs* — временная корневая ФС с модулями для доступа к реальной корневой ФС.

*systemd (PID 1)* — запускает службы и переводит систему в целевое состояние

#### 2. Работа с BIOS и EFI
**BIOS (Legacy)** — устаревший режим, использует MBR-разметку, максимум 4 первичных раздела.
**UEFI** — современный режим, использует GPT-разметку, загрузочный раздел EFI (FAT32) с файлами .efi. В Astra Linux загрузчик устанавливается в ``/boot/efi/EFI/astralinux/``

#### 3. Настройка загрузчика GRUB2
Основные файлы конфигурации:

*/etc/default/grub* — основные параметры (таймаут, параметры ядра).

*/etc/grub.d/* — скрипты генерации конфигурации.

*/boot/grub/grub.cfg* — итоговый конфиг (генерируется автоматически).

После правки обязательно выполняется **sudo update-grub**.

#### 4. Загрузка ядра ОС, параметры, передаваемые ядру
Параметры передаются через GRUB_CMDLINE_LINUX_DEFAULT в ``/etc/default/grub``. 

Примеры:

*quiet splash* — скрыть вывод загрузки;

*vga=788* — режим видеоконсоли;

*rw* — монтирование корня на запись;

*systemd.unit=multi-user.target* — выбор целевого состояния.

#### 5. Загрузка и управление модулями ядра
Автозагрузка: файлы в ``/etc/modules-load.d/*.conf`` или параметр ``modules_load=`` в командной строке ядра.

- Параметры модулей: **/etc/modprobe.d/*.conf**.
- Управление: **modprobe, lsmod, rmmod, insmod**.

##### 5.1. lsmod — список загруженных модулей
Показывает, какие модули сейчас загружены в ядро.

```bash
$ lsmod | head -5
Module                  Size  Used by
nls_utf8               16384  1
isofs                  49152  1
vfat                   24576  1
fat                    86016  1 vfat
```
Пояснение колонок:
```txt
Module — имя модуля;
Size — размер в памяти;
Used by — счётчик использований и имена зависимых модулей.
```
Поиск конкретного модуля:

```bash
$ lsmod | grep dummy
dummy                  16384  0
```
Если вывод пуст — модуль не загружен.

##### 5.2. modprobe — загрузка/выгрузка модуля с учётом зависимостей
Загрузка модуля
```bash
$ sudo modprobe dummy
$ lsmod | grep dummy
dummy                  16384  0
```
``modprobe`` автоматически подтянет зависимости, если они есть.

Загрузка с параметрами
```bash
$ sudo modprobe dummy numdummies=2
$ cat /sys/module/dummy/parameters/numdummies
2
```
Выгрузка модуля
```bash
$ sudo modprobe -r dummy
$ lsmod | grep dummy
(пусто)
```
Ключ -r (reverse) выгружает модуль и его неиспользуемые зависимости.

Проверка без реальной загрузки (dry-run)
```bash
$ sudo modprobe -n -v dummy
insmod /lib/modules/$(uname -r)/kernel/drivers/net/dummy.ko
```
##### 5.3. insmod — загрузка одного модуля по полному пути
``insmod`` не разрешает зависимости и принимает только путь к .ko-файлу.

```bash
$ sudo insmod /lib/modules/$(uname -r)/kernel/drivers/net/dummy.ko
$ lsmod | grep dummy
dummy                  16384  0
```
Если у модуля есть зависимости, insmod завершится ошибкой:

```bash
$ sudo insmod /path/to/module.ko
insmod: ERROR: could not insert module module.ko: Unknown symbol in module
```
Для собственного модуля:

```bash
$ sudo insmod ./hello.ko
$ lsmod | grep hello
hello                  16384  0
```
Выгрузка:

```bash
$ sudo rmmod hello
```
##### 5.4. rmmod — выгрузка одного модуля
Удаляет конкретный модуль. Не удаляет зависимости автоматически.

```bash
$ sudo rmmod dummy
$ lsmod | grep dummy
(пусто)
```
Если модуль используется, будет ошибка:

```bash
$ sudo rmmod vfat
rmmod: ERROR: Module vfat is in use
```
Проверить, кто использует модуль, можно по колонке Used by в lsmod или через lsof.

##### 5.5. Дополнительно: modinfo
Просмотр информации о модуле:

```bash
$ modinfo dummy
filename:       /lib/modules/5.10.0-.../kernel/drivers/net/dummy.ko
license:        GPL
description:    Dummy net driver
author:         ...
depends:
```
##### 5.6. Автозагрузка модуля и параметры
Автозагрузка при старте системы
Создайте файл:

```bash
$ sudo nano /etc/modules-load.d/dummy.conf
```
Содержимое:

```text
dummy
```
Параметры модуля по умолчанию
Создайте файл:

```bash
$ sudo nano /etc/modprobe.d/dummy.conf
```
Содержимое:

```text
options dummy numdummies=2
```
После этого при загрузке модуль dummy будет загружаться с двумя интерфейсами.

Сводная таблица

Команда         | Назначение                    | Требует root
--------------- | ----------------------------- | -------------
lsmod	          | Показать загруженные модули	  | Нет
modprobe <name>	| Загрузить модуль с зависимостями	| Да
modprobe -r <name>	| Выгрузить модуль с зависимостями	| Да
insmod <path>	  | Загрузить один модуль по пути	| Да
rmmod <name>	  | Выгрузить один модуль	| Да
modinfo <name>	| Информация о модуле	  | Нет


#### 6. Управление службами через systemd
Основные команды:

```bash
systemctl start|stop|restart|status <service>
systemctl enable|disable <service>
systemctl list-units --type=service
```
#### 7. Управление целевыми состояниями через systemd
Цели (targets) заменяют runlevels:

Target              | Аналог runlevel               | Назначение
------------------- | ----------------------------- | -------------
multi-user.target	  | 3	                            | Многопользовательский текстовый режим
graphical.target	  | 5	                            | Графический режим
rescue.target       | 1	                            | Режим восстановления

Команды:

```bash
systemctl isolate multi-user.target   # переключение на ходу
systemctl set-default multi-user.target  # по умолчанию
systemctl get-default
```
#### 8. Создание собственных юнитов systemd
Юнит типа service состоит из секций:
```txt
[Unit] — описание, зависимости (After=, Before=, Requires=).
[Service] — тип запуска (Type=), команды (ExecStart=, ExecStop=), перезапуск (Restart=).
[Install] — WantedBy= (к какому target привязать).
```
#### 9. Запуск служб с мандатными атрибутами
В Astra Linux для запуска службы с ненулевой меткой безопасности в секцию ``[Service]`` добавляется параметр ``PDPLabel=<уровень>:<уровень целостности>:<категории>``. Для сокетов используется ``CapabilitiesParsec=``. После правки:

```bash
sudo systemctl daemon-reload
sudo systemctl restart <service>
```
#### Практическая работа
##### Часть 1. Загрузка в режиме single через командную строку GRUB
Цель: получить доступ к системе с правами root без ввода пароля.

Шаги:

Перезагрузите систему и при появлении меню ``GRUB`` нажмите **e** для редактирования выбранного пункта.

Найдите строку, начинающуюся с ``linux (или linuxefi)``. В конце строки допишите:

```text
init=/bin/bash
```
или
```text
single
```
В некоторых сборках Astra Linux используется ``rw init=/bin/bash``.

Нажмите ``Ctrl+X`` или ``F10`` для загрузки с изменёнными параметрами. Система загрузится в однопользовательском режиме с root-оболочкой.

Проверьте права:

```bash
whoami   # должно вывести root
```
> - Важно: В этом режиме корневая ФС может быть смонтирована только для чтения. Для изменения пароля потребуется перемонтировать её:

```bash
mount -o remount,rw /
```
##### Часть 2. Смена пароля пользователя
Смените пароль root:

```bash
passwd root
```
Введите новый пароль дважды.

Смените пароль обычного пользователя (если нужно):

```bash
passwd username
```
Перемонтируйте ФС обратно и перезагрузитесь:

```bash
mount -o remount,ro /
exec /sbin/init
```
или просто reboot -f.

##### Часть 3. Смена таймаута и пароля GRUB
###### 3.1. Изменение таймаута
Откройте файл конфигурации:

```bash
sudo nano /etc/default/grub
```
Найдите параметр GRUB_TIMEOUT и установите нужное значение (например, 10 секунд):

```text
GRUB_TIMEOUT=10
```
Для полного скрытия меню можно установить ``GRUB_TIMEOUT=0``.

Примените изменения:

```bash
sudo update-grub
```
###### 3.2. Установка/смена пароля GRUB
Сгенерируйте хеш пароля:

```bash
grub-mkpasswd-pbkdf2
```
Введите пароль дважды. Скопируйте полученную строку, начинающуюся с grub.pbkdf2.sha512....

Отредактируйте файл пароля:

```bash
sudo nano /etc/grub.d/07_password
```
Приведите содержимое к виду:

```bash
#!/bin/bash
cat << EOF
set superusers="admin"
password_pbkdf2 admin grub.pbkdf2.sha512.10000.ВАШ_ХЕШ
EOF
```
Установите права и обновите GRUB:

```bash
sudo chmod 700 /etc/grub.d/07_password
sudo update-grub
```
Примечание: Для отключения запроса пароля для определённых пунктов (например, Windows) в ``/boot/grub/grub.cfg`` к нужному menuentry добавляется ``--unrestricted``.

##### Часть 4. Создание unit (service) для включения маршрутизации в ядре
Цель: создать службу, которая при загрузке включает IP-форвардинг.

Создайте файл юнита:

```bash
sudo nano /etc/systemd/system/ip-forward.service
```
Содержимое файла:
```
ini
[Unit]
Description=Enable IP Forwarding
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -w net.ipv4.ip_forward=1
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Пояснение:
```txt
Type=oneshot — служба выполняется один раз и завершается.
ExecStart — команда включения форвардинга.
RemainAfterExit=yes — systemd считает службу активной после завершения.
WantedBy=multi-user.target — автозапуск при загрузке в многопользовательском режиме.
```
Активируйте и запустите службу:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ip-forward.service
sudo systemctl start ip-forward.service
```
Проверьте статус и результат:

```bash
systemctl status ip-forward.service
cat /proc/sys/net/ipv4/ip_forward   # должно быть 1
```
##### Часть 5. Запуск службы с мандатными атрибутами (дополнительно)
Если требуется запустить службу с ненулевой меткой безопасности, в секцию ``[Service]`` юнита добавляется параметр ``PDPLabel``. Пример для уровня конфиденциальности 1:
```
ini
[Service]
PDPLabel=1:63:0
```
После правки:

```bash
sudo systemctl daemon-reload
sudo systemctl restart <service>
```
Проверить метку службы можно командой:

```bash
systemctl show <service> -p PDPLabel
```
#### Примечание для Astra Linux
- Если включён Secure Boot, загрузка неподписанных модулей (например, собственных) будет заблокирована. Потребуется подписать модуль или отключить Secure Boot.
