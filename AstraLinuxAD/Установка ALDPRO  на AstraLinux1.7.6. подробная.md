#### Установка контролеров домена ALD PRO версии (2.4.1-2.5.0) на Astra Linux 1.7.x. будет состоять из двух этапов.

- Предварительная подготовка сервера.
- Установка ALD Pro.

##### Предварительная подготовка сервера.

Что надо знать?
1. Рекомендуемые версии такие с 1.7.4-24.04.2023_14.23 до 1.7.6.15-15.11.24_17.20 Смоленск<br>
2. ALD Pro не поддерживает ядра hardened. Поддерживаются только generic ядра.<br>
3. Минимальное количество мегабайт оперативной памяти **4Гб**, необходимое для развертывания служб контроллера домена ALD Pro. Развертывание контроллера на узле запускается только в том случае, когда на нем установлено количество оперативной памяти большее или равное указанному. Значение по умолчанию – 4096 (4Гб).
4. Для установки ALD PRO необходимо использовать редакцию Astra Linux с максимальным уровнем защищенности – **Смоленск** или усиленным уровнем защищенности – **Воронеж**.<br>**Орел** – вариант лицензирования несертифицированной ОС с базовым уровнем защищенности, не может применяться в системах, где предъявляются требования в части защиты информации.<br>
   > Переключает и отображает уровни защищенности ОС:<br>
   
       0  Базовый;
       1  Усиленный;
       2  Максимальный.
5. Для установки ALD PRO настоятельно рекомендуется использовать статическую адресацию на серверах.<br>
6. Для имени домена рекомендуется выбирать такое имя, которые не будет конфликтовать с другими вашими сервисами.
      - Например, для LAD Pro выделить отдельный домен подуровня(третьего уровня). Например, it.company.lan.


Проверяем на рекомендуемое соответствие
```bash
cat /etc/astra/build_version
sudo astra-modeswitch getname
sudo astra-modeswitch list
```
![image](https://github.com/user-attachments/assets/1b721318-470a-40d9-b27f-027ac09a7d19)


Если при установке выбрали режим другой, то переходим на требуемый (выбирайте при установке сразу нужный, чтобы было меньше лишних манипуляций)
```bash
sudo astra-modeswitch set 2
```
>[!Warning]
>При изменении уровня защищенности:
- В сторону снижения уровня защищенности: автоматически отключаются опции, которые для данного уровня защищенности недоступны;
- В сторону увеличения уровня защищенности: при увеличении уровня защищенности состояние опций (МРД и МКЦ) автоматически не изменяется, но опции, доступные в новом режиме, становятся доступными для включения.

Из того, что "состояние опций автоматически не изменяется"  следует, что после выполнения команды повышения уровня, например: 
```bash
sudo astra-modeswitch set 2
нужно будет отдельно включить необходимые опции. Полный набор команд для включения всех опций:
sudo astra-mic-control enable
sudo astra-mac-control enable
sudo astra-digsig-control enable
sudo astra-secdel-control enable
sudo astra-swapwiper-control enable
```
Добавим и запустить службу синхронизации времени chrony в автозапуск.
>[!Warning]
>Серверная служба chronyd (пакет chrony) Может обеспечивать работу ОС в режиме как сервера точного времени, так и клиента. Является штатной службой времени для использования с контроллерами домена FreeIPA.

```bash
sudo systemctl status chrony
если не установлена, установить можно так
sudo apt install chrony
sudo systemctl start chrony
sudo systemctl enable chrony
```
Добавим Российские NTP сервера и можно указать разрешённую сеть для клиентов.
Вводим ``nano /etc/chrony/chrony.conf`` и добавляем эти строки
```bash
server 0.ru.pool.ntp.org iburst
server 1.ru.pool.ntp.org iburst
server 2.ru.pool.ntp.org iburst
server 3.ru.pool.ntp.org iburst
allow 192.168.25.0/24
```
>[!Warning]
>Серверная служба chronyd (пакет chrony) Может обеспечивать работу ОС в режиме как сервера точного времени, так и клиента. Является штатной службой времени для использования с контроллерами домена FreeIPA.<br>
>Если при начальной настройке будущего контроллера домена желаете использовать службу синхронизации времени ntp, то делается это следующим образом (но!!! после установки контроллера домана, будет всё равно использоваться служба chrony).

>[!Warning]
>Более подробно описано в файле УстановкаСлужбыВремени.

#### Настраиваем сеть и установливаем статический IP адрес.
Узнаем название сетевого интерфейса, IP адресс, шлюз
```bash
ip a | grep 'inet '
route
```
![image](https://github.com/user-attachments/assets/977ef3c4-81a2-4ff9-b5da-625ebc090a28)


или так
```bash
nmcli dev sh            //nmcli эта команда работает только при сетевой службе NetworkManager
```
или так
```bash
ip a show dev eth0
```
Для начала сделаем статический адрес на будущем контроллере домена.

>[!Warning]
>На рабочих станциях этого делать не обязательно, да и не нужно.

Мы будем использовать службу ``networking.service`` и соответственно команды пакета ``ifupifdown``<br>
Проверить корректность файла: ``sudo ifquery eth0``<br>
Перезапустить интерфейс. Лучше всегда делать это одной командой, чтобы не потерять машину при работе через удалённое подключение:
``sudo ifdown eth0; sudo ifup eth0``

Настраиваем статический адрес службы ``networking.service`` вводим команду: ``nano /etc/network/interfaces``
```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

auto eth0
allow-hotplug eth0
iface eth0 inet static
address 192.168.25.115
netmask 255.255.255.0
gateway 192.168.25.10
```
![image](https://github.com/user-attachments/assets/ba5eac07-9a21-4ff4-96c5-d396faf19795)


- auto eth0  --поднимать интерфейс автоматически при старте системы
- allow-hotplug eth0 --автоматически выполнять перезапуск интерфейса при его падении
- iface eth0 inet static --к какому интерфейсу мы привязываем статический адрес
- address 192.168.25.115 --статический адрес
- netmask 255.255.255.0  --маска
- gateway 192.168.25.10  --шлюз
>[!Warning]
>Эти записи в файле /etc/network/interfaces

- dns-domain it.company.lan
- dns-nameservers 192.168.25.10

работать не будут, так как

- Имя домена: "it.company.lan" (используется пакетом resolvconf)
- IP-адрес сервера DNS: 192.168.25.10 (используется пакетом resolvconf)

Пакет ``resolvconf`` не утанавливается автоматичеки, поэтому этот пакет ``resolvconf`` использовать не будем<br>
и так как мы не используем пакет ``resolvconf``, соответствующая настройка параметров DNS должна быть выполнена вручную в файле ``/etc/resolv.conf`` 

>[!Warning]
>Если используете ``systemd-resolved``

То нужно связать заглушку, предоставленную systemd-resolved , с /etc/resolv.conf следующим образом:
```bash
# ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```
Номы его тоже использовать не будем.

>Во избежание конфликтов между службами отключить, остановить и заблокировать все остальные службы управления сетевыми интерфейсами

```bash
sudo systemctl status NetworkManager //проверяем статус службы NetworkManager
sudo systemctl stop NetworkManager //останавливает службу
sudo systemctl disable NetworkManager //удаляет её из автозагрузки
sudo systemctl mask NetworkManager //останавливает активность службы

sudo systemctl stop systemd-networkd
sudo systemctl disable systemd-networkd
sudo systemctl mask systemd-networkd

служба systemd-networkd устанавливается автоматически
после установки служба находится в заблокированном состоянии, соответственно, не запускается, и ничем не управляет
служба systemd-resolved представлена отдельным пакетом systemd-resolved и может быть установлена при необходимости

если установлен пакет
sudo systemctl --now mask resolvconf
```
также отключаем элемент управления сетью в трее графического интерфейса
```bash
astra-noautonet-control disable
```
- astra-noautonet-control <enable/disable/status/is-enabled>
- astra-noautonet-control is-enabled  ВКЛЮЧЕНО 
- astra-noautonet-control disable
- astra-noautonet-control is-enabled ВЫКЛЮЧЕНО
- astra-noautonet-control status НЕАКТИВНО

Если всё правильно сделали, то ответ на введённые команды по статусу, например ``sudo systemctl status NetworkManager`` будет таким:

![image](https://github.com/user-attachments/assets/366a31ef-6910-44c5-a4b4-0956f192f03a)


и сеть в трее графического интерфейса будет не активна.

![image](https://github.com/user-attachments/assets/d050b803-c5c7-4819-9f13-bbe39c3b680d)


Если пропал файл ``/etc/resolv.conf`` просто создайте его заново и внесите такие данные
```bash
suod nano /etc/resolv.conf

search it.company.lan
#nameserver 192.168.5.15   //если у вас есть маршурты ретрансляторы DNS Relay (выполняет роль посредника между клиентскими устройствами внутри сети и удаленным DNS-сервером)
их тоже надо прописать, узнать предварительно можно было до отключения ``NetworkManager`` командой ``nmcli dev sh``
#nameserver 192.168.25.20
nameserver 77.88.8.8
```
Чтобы применить новые настройки, достаточно перезапустить службу ``networking`` командой ``systemctl restart networking``. Может потребоваться также очистить старое соединение командой ``ip addr flush dev <имя устройства>``:
```bash
sudo systemctl restart networking
sudo ip addr flush dev eth0
или просто ребут системы

Проверяем
ping 77.88.8.8 -c 4
```

> [!Warning]
> Предложенные зоны ``.lan`` и ``.internal`` не зарегистрированы в глобальном списке ``Top-Level Domains``, но всегда будет оставаться вероятность, что их ведут в эксплуатацию в будущем.

> Соответственно, следует использовать зоны ``.lan``, ``.internal`` и ``.local`` учитывая эти риски.

> Полный список доменов верхнего уровня можно посмотреть здесь https://ru.wikipedia.org/wiki/%D0%A1%D0%BF%D0%B8%D1%81%D0%BE%D0%BA_%D0%B4%D0%BE%D0%BC%D0%B5%D0%BD%D0%BE%D0%B2_%D0%B2%D0%B5%D1%80%D1%85%D0%BD%D0%B5%D0%B3%D0%BE_%D1%83%D1%80%D0%BE%D0%B2%D0%BD%D1%8F 

Теперь после перезагрузки в файл hosts добавим строки с именем сервера ``nano /etc/hosts``.
```bash
127.0.0.1        localhost.localdomain localhost
# 127.0.1.1      dc01.it.company.lan dc01   --обязательно закомментировать
192.168.25.115   dc01.it.company.lan dc01

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
![image](https://github.com/user-attachments/assets/8f1c33fa-f9a2-4c15-85ac-8854b819117e)


Настраиваем ``FQDN`` имя первого контроллера домена:
```bash
hostnamectl set-hostname dc01.it.company.lan
```
Перезапустим сетевой интерфейс для применения настроек
```bash 
systemctl restart networking.service
```
Проверяем
```bash
hostname -s
hostname -f // если не работает проверяем запись в файле etc/hosts
```
![image](https://github.com/user-attachments/assets/97531ef9-fce5-455d-a184-b9e744ec1706)


Ещё раз проверяем все настройки, вводим команду ``ifquery`` результат должен быть такой

![image](https://github.com/user-attachments/assets/87ffbe61-9c0e-4d19-8752-7feb3b2553d9)


Проверяем пинг ``ping -c 4 dl.astralinux.ru``
Проверяем
```bash
hostname -s
hostname -f 
```
Если всё верно идём дальше.
>[!Warning]
>Для проверки работы DNS обязательно установим пакет утилит ``nslookup, dig`` командой ``apt instal dnsutils`` перед повышением сервера до контроллера домена.

Добавляем репозитории.
>[!Warning]
>Репозиторий base включает репозитории main и update, а репозиторий extended содержит большое количество дополнительного программного обеспечения.

Вводим команду ``nano /etc/apt/sources.list``
```bash
# Astra Linux repository description https://wiki.astralinux.ru/x/0oLiC Основной репозиторий
# deb https://dl.astralinux.ru/astra/stable/1.7_x86-64/repository-main/ 1.7_x86-64 main contrib non-free
# Оперативные обновления основного репозитория
deb https://dl.astralinux.ru/astra/stable/1.7_x86-64/repository-update/ 1.7_x86-64 main contrib non-free
# Рекомендуемые репозитории для установки сервера
deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.3/repository-base/ 1.7_x86-64 main contrib non-free
deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.3/repository-extended/ 1.7_x86-64 main contrib non-free
deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.3/repository-update/ 1.7_x86-64 main contrib non-free
deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.3/uu/2/repository-update/ 1.7_x86-64 main contrib non-free
```
![image](https://github.com/user-attachments/assets/4805e888-d9b3-44a9-9ca7-17949c2f0b43)


>[!Warning]
>Версии репозиторий меняйте под версию ОС которую устанавливаете.
>Например, если ``cat /etc/astra/build_version`` 1.7.4.7<br> значит в строке репозитория должна стоять эта версия  <br>``deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/  1.7.4   /repository-base/ 1.7_x86-64 main contrib non-free``

Определения репозиториев также могут быть указаны файлах, расположенных в каталоге /etc/apt/sources.list.d/. Файлы могут иметь произвольное имя c обязательным расширением ".list".
Для ALD PRO в папкe source.list.d добавим файл с записью
```bash
cat > /etc/apt/sources.list.d/aldpro.list
deb https://dl.astralinux.ru/aldpro/frozen/01/2.4.1 1.7_x86-64 main base
#deb https://dl.astralinux.ru/aldpro/frozen/01/2.5.0 1.7_x86-64 main base
```
Для использования сетевых репозиториев, работающих по протоколу HTTPS необходимо, чтобы в системе был установлен пакет ``apt-transport-https`` и пакет ``ca-certificates``.<br> 
Проверить наличие пакетов можно командой: ``apt policy apt-transport-https ca-certificates``

![image](https://github.com/user-attachments/assets/36c6fa53-8dc6-4a43-8cce-e1489592b424)


>[!Warning]
>Установить пакеты, если вдруг утеряны ``apt-transport-https`` и ``ca-certificates`` можно командой: ``sudo apt install apt-transport-https ca-certificates``

Обновляем
```bash
apt update
apt list --upgradable
apt dist-upgrade -y -o Dpkg::Optoins::=--force-confnew

правильно будет лучше если вместо dist-upgrade использовать аналогичную родную команду ОС Астра Линукс astra-update

apt install astra-update
astra-update -A -r -T 
```
![image](https://github.com/user-attachments/assets/9ecf84cd-d59f-45ab-b6ab-1102aaf89971)



Перезагружаем сервер ``reboot``

##### Установка первого контроллера ALD Pro

После предварительной настройки продолжаем установку.
Установка ALD Pro
1. Устанавливаем необходимые пакеты:
```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y -q aldpro-mp aldpro-gc aldpro-syncer
```
- aldpro_enable_syncer – установка модуля синхронизации ``aldpro-syncer``. Этот модуль необходим для использования расширенных функций интеграции с доменом Microsoft Active Directory.
- aldpro_enable_gc – установка модуля глобального каталога ``aldpro-gc``. Этот модуль необходим, если используется топология из контроллера домена и нескольких реплик. Службы, предоставляемые этим модулем, выполняют синхронизацию данных пользователей между контроллером домена и его репликами.

2. Прежде чем продолжить, ознакомьтесь с журналом пакетного менеджера в файле /var/log/apt/term.log на наличие ошибок:
```bash
sudo grep 'error:' /var/log/apt/term.log
```
3. Теперь повысим сервер до контроллера домена. Дополнительно отключим историю выполнения команд, чтобы пароль не был записан в эту историю:
```bash
set + o history
sudo aldpro-server-install -d it.company.lan -n dc01 -p 'QwertyQAZWSX' --ip 192.168.25.115 --no-reboot --setup_syncer --setup_gc
```
4. После завершения установки проверим журнал на наличие ошибок:
```bash
sudo grep error: /var/log/apt/term.log
```
5. Дожидаемся окончания процедуры повышения сервера до контроллера домена и проверяем:   
```bash
sudo aldproctl status
sudo ipactl status
```
![image](https://github.com/user-attachments/assets/ee023fbd-7a25-474e-b0d9-2e60bd14b825)


6. Включаем обратно историю ведения команд:
```bash
set -o history
```
По завершению установки вы увидите сообщение об успешном выполнении операции.
```bash
------------
Succeeded: 5 (changed=5)
Failed:    0
------------
```
7. Проверим настройки разрешения имен:
```bash
sudo cat /etc/resolv.conf
```
![image](https://github.com/user-attachments/assets/21ab2d00-303a-4d67-9efa-c0d827c3b415)


В файле должен быть указан ваш домен и адрес сервера – 127.0.0.1, т.к. этот файл настраивается на службу bind9.

7. Перезагружаем сервер.
   
9. Проверьте в первую очередь доступность клиентской службы SSSD:
```bash
systemctl status sssd
```
Входим в домен 

![image](https://github.com/user-attachments/assets/507db5fd-ef0d-45be-88f8-d86dcd2873d0)

![image](https://github.com/user-attachments/assets/ff855c8f-e326-41d0-9505-0e1f1035ae01)

![image](https://github.com/user-attachments/assets/48991c99-2db3-4703-99a5-f268aa8824dd)


##### Отключение DNSSEC
```bash
sudo sed -i 's/dnssec-validation yes/dnssec-validation no/g' /etc/bind/ipa-options-ext.conf
sudo systemctl restart bind9-pkcs11.service

sudo cat > /etc/bind/ipa-options-ext.conf
allow-recursion { any; };
allow-query-cache { any; };
```
Настройка глобального перенаправления DNS

Роли и службы сайта → Служба разрешения имён → Глобальная конфигурация DNS
Указать IP-адрес внешнего резолвера

![image](https://github.com/user-attachments/assets/b8d65df2-44a8-4ae1-825a-5b8a07d16dce)


Например: 77.88.8.8 или 8.8.8.8 или 1.1.1.1

![image](https://github.com/user-attachments/assets/9a70af4d-4528-4020-916d-9e1d75942e32)


Если всё правильно сделали, то при проверке настроек должно быть так
```bash
klist
```

![image](https://github.com/user-attachments/assets/7bab4ab8-2786-4ce9-812c-a69ff8f7daef)


Если вдруг там нет билета HTTP добляем его вручную и заодно проверим права пользователя admin 
```bash
ipa user-show admin
```

![image](https://github.com/user-attachments/assets/a222c10f-89e5-4dba-922c-1e63ae8fddd2)


![image](https://github.com/user-attachments/assets/af20f53f-aae0-4b9c-806e-16f410079b6b)


![image](https://github.com/user-attachments/assets/63362a71-9f8f-4ecb-8507-7e46da4ab685)


>[!Warning]
>Если не входит по Kerberos попробуйте настроить браузер как описано ниже или удаляем выданные билеты и заводим заново.

Для уничтожения билетов используется команда kdestroy, после выполнения которой мы увидим, что в связке ключей кэш больше не найден:
```bash
kdestroy
klist
```
Для запроса нового TGT-билета на доменного пользователя admin воспользуемся утилитой kinit. Как мы видим, после прохождения успешной Kerberos-аутентификации нам был выдан новый TGT-билет:
```bash
kinit admin
klist
```
Выполните аутентификацию через алиас root, используя пароль пользователя admin:
```bash
kdestroy
sudo kinit
Password for root@ALD.COMPANY.LAN:

klist
```
##### Проверьте портал управления и настройки браузера.

Установите правильный адрес домашней страницы для браузера FireFox следующей командой:
```bash
sudo sed -i 's/dc-1\|dc-1.ald.company.lan/dc-1.ald.company.lan/g' \
     /usr/lib/firefox/distribution/policies.json
```
Содержимое файла после коррекции должно быть следующим:
```bash
cat /usr/lib/firefox/distribution/policies.json
{
         "policies": {
               "BlockAboutAddons": true,
               "BlockAboutConfig": true,
               "Certificates": {
                        "ImportEnterpriseRoots": true,
                        "Install": ["/etc/ipa/ca.crt"]
               },

               "Homepage": {
                        "URL": "https://dc-1.ald.company.lan/",
                        "Locked": true,
                        "StartPage": "homepage-locked"
               }
         }
}
```
Проверьте, что в связке ключей текущего пользователя появится сервисный билет на доступ к службе HTTP/dc-1.ald.company.lan.
```bash
klist
```
![image](https://github.com/user-attachments/assets/89635e55-1cdf-44d3-9fe9-5c22dbcd50f0)


Пробуем зайти по Kerberos.

Проверяем работу DNS
```bash
root@dc01:~# systemctl status bind9
● bind9.service
   Loaded: masked (Reason: Unit bind9.service is masked.) //служба должа быть отключена по маске
   Active: inactive (dead)
root@dc01:~#

systemctl status bind9-pkcs11.service
```
![image](https://github.com/user-attachments/assets/429393a4-a90a-4912-9782-140e0359c5e8)


```bash
ipa dnsconfig-show
```
![image](https://github.com/user-attachments/assets/640ec9f5-e5ae-4a47-979c-3483e610a112)


#### Далее, если всё норм подключаем с такой же предварительной подготовкой, ещё один контроллер домена.

Проверяем, что мы его видим

![image](https://github.com/user-attachments/assets/71fd79db-eae7-4b31-84ee-fb1e656cdca6)


На другом контроллере домена наш первый контроллер домена должен пинговаться, проверяем (по короткому и длинному имени)

![image](https://github.com/user-attachments/assets/3cc33820-3dd3-4423-a770-ec8e80102d84)


Если всё ок, вводим в домен клиентский компьютер.
```bash
/opt/rbta/aldpro/client/bin/aldpro-client-installer --domain it.company.lan --account admin --password 'ваш пароль' --host dc03 --gui --force
```

Далее после перезагрузки входим в домен


