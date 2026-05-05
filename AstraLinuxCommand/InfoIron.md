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

