### Получаем информацию о железе сервера в Linux

#### Информация об оперативной памяти (RAM) в Linux

Вы можете получить информацию о количестве оперативной памяти на сервере с помощь встроенных средств Astra Linux (данные команды не дают подробной информации, но вполне приемлемы для быстрой оценки).
```bash
# free -m
```
Или
```bash
# free -g
```
Первая покажет количество памяти в мегабайтах, вторая в гигабайтах (информация о количестве оперативной памяти указано в значении Mem: total).<br>
Тут же будет и показан размер swap.<br>
Также вы можете получить информацию о RAM из файла /proc/meminfo:
```bash
# grep MemTotal /proc/meminfo
# grep SwapTotal /proc/meminfo
```
<img width="1238" height="207" alt="image" src="https://github.com/user-attachments/assets/c3ae5715-e0d6-49c8-bb2c-ce79c5f63ad6" />

Первый вариант на мой взгляд удобнее, так как вы сразу видите и используемую память, и свободную.<br>
Так же существует еще несколько вариантов проверки количества ОЗУ на сервере:
```bash
# vmstat -s
```
<img width="1235" height="428" alt="image" src="https://github.com/user-attachments/assets/b52717ac-3a6b-432b-b446-7ead14d42bc9" />

Vmstat показывает не только физическую память сервера, но и всю статистику по виртуальной памяти.<br>
Либо запустите команду top и посмотрите информацию о RAM в самом верхнем блоке:

<img width="1229" height="95" alt="image" src="https://github.com/user-attachments/assets/a95ebdcd-4591-4ff3-97c5-d91ab2634aeb" />

Так же, есть удобная утилита ``atop``, которая покажет вам количество ОЗУ на сервере, а также информацию по занятой, кешированной и свободной памяти.

Для семейства c установщиками ``apt``
```bash
# apt install atop
```
Для семейства c установщиками ``yum`` вы можете установить утилиту atop из EPEL репозитория с помощью yum (dnf):
```bash
# yum install atop -y
```
Вывод ``atop``:

<img width="1238" height="335" alt="image" src="https://github.com/user-attachments/assets/b77e4c3f-15f5-41da-ab4b-3142ebef401c" />

Должна быть в вашем арсенале и не менее удобная утилита nmon. Установите ее на сервер:
```bash
# apt install nmon
или
# yum install nmon -y
```
Выполните команду ``nmon``, и для проверки ОЗУ нажмите ``m``:

<img width="1211" height="359" alt="image" src="https://github.com/user-attachments/assets/cdaffbe9-d3ba-430e-957f-b7492a456448" />

<img width="1234" height="187" alt="image" src="https://github.com/user-attachments/assets/166ba918-b2ff-4bd9-a6fc-82d769fa8f73" />

Но все вышеперечисленные утилиты, показывают лишь объем памяти, а модель скорость и другие характеристики нет. Если нужна более подробная информация о бланках памяти (производитель, тип, частота), можно воспользоваться утилитой dmidecode:
```bash
# dmidecode -t 17
```
<img width="1190" height="463" alt="image" src="https://github.com/user-attachments/assets/70b9ea14-551f-4526-a24f-53eca3277a9c" />

Как видите, dmidecode выводит более подробную информацию о установленных модулях памяти.

#### Как узнать информацию о процессоре (CPU) в Linux?

Информацию о процессоре в Linux можно получить несколькими способами. Начнем с самого простого — получение информации из файла /proc/cpuinfo:
```bash
# cat /proc/cpuinfo | grep model
```
<img width="1256" height="159" alt="image" src="https://github.com/user-attachments/assets/688af255-5697-4793-ab6f-f4b729df43f2" />

Чтобы узнать количество ядер, выполните:
```bash
# cat /proc/cpuinfo | grep processor
```
<img width="1225" height="517" alt="image" src="https://github.com/user-attachments/assets/2febbccb-3edf-4e80-8d18-74a79eb37fa5" />

Более подробную информацию о процессоре, можно узнать командой lscpu:
```bash
# lscpu
```
<img width="1237" height="423" alt="image" src="https://github.com/user-attachments/assets/da26dadf-d2df-4312-95ee-4043a10e6efc" />

Утилита ``lscpu`` покажет вам количество ядер, модель процессора, максимальную частоту, рамеры кэшей CPU, ноды NUMA и многое другое.

Количество ядер, так же можно узнать запустив команду ``atop или nproc --all``:

Для отображения подробной информации, можно дополнительно установить утилиту cpuid:
```bash
# apt install cpuid
или
# yum install cpuid -y
```
После установки запустите командой:
```bash
# cpuid
```
<img width="1207" height="949" alt="image" src="https://github.com/user-attachments/assets/662fd503-e653-4773-a344-a599aeb435e0" />

Вы получите информацию не только о модели процессора, но тип и семейство процессора, конфигурацию кеша, функцию управления питанием и другое.

С помощью утилиты demidecodev вы так же можете узнать всю информацию об установленных на сервере процессорах:
```bash
# dmidecode --type processor
```
<img width="1213" height="918" alt="image" src="https://github.com/user-attachments/assets/aa313af6-9ae8-4845-a6f4-a93d93ff4e3f" />

Ещё одна утилита для проверки процессора inxi. Это скрипт на bash, который покажет вам модель процессора, размер кеша, частоту и дополнительные возможности процессора. Установим его:
```bash
# apt install inxi -y
или
# yum install inxi -y
```
Запустите скрипт:
```bash
# inxi -C
```
<img width="1232" height="165" alt="image" src="https://github.com/user-attachments/assets/8c54fc01-fee8-478e-b99b-ddcf9a924e7d" />

#### Информация о жестких дисках сервера в Linux
Чтобы получить информацию о жестких дисках в системе, я обычно использую утилиту hdparm. Сначала нужно установить ее из репозитория:
```bash
# apt install hdparm -y
# yum install hdparm -y
```
Чтобы получить инфу по жесткому диску, нужно указать название устройства:
```bash
# hdparm -I /dev/sdb
```
<img width="1173" height="379" alt="image" src="https://github.com/user-attachments/assets/ab78a6a7-6394-4fb6-b29e-9b145423b071" />

**hdparm** - просмот информации о типах жестких дисков

Как видите, при проверке отображается модель диска, серийный номер, версия прошивки диска, цилиндрах, rpm, поддерживаемые функции и ряд другой информации.

Вторая не менее популярная утилита это ``smartctl`` (она по умолчанию уже установлена в системе centos). Чтобы вывести информацию о диске, выполните:
```bash
# smartctl -d ata -a -i /dev/sdb
```
Утилита smartctl входит в состав пакета smartmontools. Это стандартный инструмент для Linux, который есть и в репозиториях Astra Linux. Установить его можно одной командой.
```bash
sudo apt update && sudo apt install smartmontools -y
```bash
Информация будет предоставлена так же подробно:
```bash
smartctl 
```
Очередная, очень удобная утилита lshw. Установите ее:
```bash
# apt install lshw -y
# yum install lshw -y
```
Выполните команду:
```bash
# lshw -class disk
```
<img width="1215" height="652" alt="image" src="https://github.com/user-attachments/assets/cc141e42-8cd5-4566-a98a-b64912a01733" />

Утилита ``dmidecode:`` получения информации о материнской плате, BIOS и др.

В данном разделе я приведу примеры более расширенного использования утилиты ``dmidecode``. ``Dmidecode`` позволяет получить информацию об аппаратном обеспечении сервера на основе данных из ``BIOS`` по стандарту ``SMBIOS/DMI``.

С помощью ``dmidecode`` мы можем получить информацию о материнской плате, bios, шасси и слотах сервера. 

> Например:<br>
**dmidecode --type baseboard** – получим информацию о материнской плате.
```bash
dmidecode --type baseboard 
```
**dmidecode --type bios** – информация о BIOS (версия, поддерживаемые функции).
```bash
dmidecode --type bios
```
**dmidecode --type chassis** – сведения о корпусе (шасси) сервера.
```bash
dmidecode --type chassis 
```
**dmidecode --type slot** – сведения о используемых слотах на материнской плате.
```bash
dmidecode --type slot 
```
<img width="1218" height="763" alt="image" src="https://github.com/user-attachments/assets/5fe8c6c1-fd91-4d37-93ba-6cdba626ed7e" />

Чтобы собрать вообще всю информацию о железе вашего сервера Linux, можно воспользоваться ранее указанную утилиту lshw:
```bash
# lshw -html > server_info.html
```
Вся информация будет выгружена в html файл.
```bash
lshw -html 
```
<img width="1637" height="993" alt="image" src="https://github.com/user-attachments/assets/847294bc-92de-4d07-a6c2-65b7b8fdd5d4" />


