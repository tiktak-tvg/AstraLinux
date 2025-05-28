#### Установка Корпоративного сервера 2024 offline-версии для ОС Astra.

Установка произваодится на VM Ware Workstation, версия ОС Астра 1.7.4

Установка состоит из двух этапов:

1. Предварительная подготовка.
2. Установка Корпоративного сервера 2024 offline-версии.

#### Предварительная подготовка.   

Устанавливаем ОС Астра 1.7.4 образ 1.7.4-24.04.2023_14.23.iso

После установки выполняем команды:

Открываем репозитории `` nano /etc/apt/sources.list``

![image](https://github.com/user-attachments/assets/a9342a3c-bdbd-44fb-8f90-19b6b16f4065)
```bash
apt install open-vm-tools
```
```bash
apt update
apt install astra-update
apt install dnsutils
```
Всё, больше никаких манипуляций с обновлениями.

Проверим версию ``cat /etc/astra/build_version``

Должна быть 1.7.4.7

![image](https://github.com/user-attachments/assets/7050daa1-c528-4897-b120-892af3d87965)

Если версия такая, идём дальше.

Добавим и запустить службу синхронизации времени chrony в автозапуск.
>[!Warning]
>Серверная служба chronyd (пакет chrony) Может обеспечивать работу ОС в режиме как сервера точного времени, так и клиента. Является штатной службой времени для использования с контроллерами домена FreeIPA.

Служба chrony по умолчанию не установлена, установить можно так
```bash
sudo apt install chrony
sudo systemctl start chrony
sudo systemctl enable chrony
```
Делаем статический адрес. Отключаем NetworkManager.

Переписываем свой IP дарес, шлюз, маску, не ошибиться при настройке статического IP-адреса.
```bash
nmcli dev show
```
![image](https://github.com/user-attachments/assets/e4dfa035-87bb-48e1-8a0e-11a11db3717f)

Откючаем и забываем, на серверах не должно быть динамического адреса.
```bash
sudo systemctl status NetworkManager //проверяем статус службы NetworkManager
sudo systemctl stop NetworkManager //останавливает службу
sudo systemctl disable NetworkManager //удаляет её из автозагрузки
sudo systemctl mask NetworkManager //останавливает активность службы
astra-noautonet-control disable //контрольный выстрел
```
![image](https://github.com/user-attachments/assets/27f2a9f5-b55a-44fc-ba41-1345cd231627)

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
address 192.168.25.100
netmask 255.255.255.0
gateway 192.168.25.10
```
![image](https://github.com/user-attachments/assets/b0a9000c-6466-477b-9e61-1952c9560ccb)

Чтобы применить новые настройки, перезагружаем машину:
```bash
reboot
```
Если всё правильно сделали, то ответ на введённую команду ``sudo systemctl status NetworkManager`` после перезагрузки ответ будет таким, пока не перезагружаем:

![image](https://github.com/user-attachments/assets/25a62ca3-5814-47af-96a0-37f04065a4f2)

После загрузки, проверяем
```bash
ping 77.88.8.8 -c 4
```
![image](https://github.com/user-attachments/assets/f3779d53-5032-45f7-bdab-ffa28a56d2ab)

Если пинга нет, смотрите прописаны ли DNS записи в файле ``nano /etc/resolv.conf``. 

Теперь в файл hosts добавим строки с именем сервера ``nano /etc/hosts``.
```bash
127.0.0.1        localhost.localdomain   localhost
# 127.0.1.1      lamba.it.company.lan    lamba   --обязательно закомментировать
x.x.x.x          lamba.it.company.lan    lamba

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
![image](https://github.com/user-attachments/assets/4fdc6c41-d6fe-400b-be70-cc632ba90b8c)

Настраиваем ``FQDN`` имя первого контроллера домена:
```bash
hostnamectl set-hostname lamba.it.company.lan
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
![image](https://github.com/user-attachments/assets/9c6760a3-3d31-404f-b0d8-50536fdac863)

#### Установка Корпоративного сервера 2024 offline-версии. 

Скачиваем с их сайта архив CDinstall_2.0.2024.14752_Astra_1.7.4_offline.zip<br>
или с нашей шары и закидываем в корень папки ``/mnt`` переходим в неё и распаковываем
```bash
cd /mnt/
unzip CDinstall_2.0.2024.14752_Astra_1.7.4_offline.zip
```
![image](https://github.com/user-attachments/assets/29d3d29e-d27f-4732-8851-918518dc749f)

После распаковки заходим в папку
```bash
cd /mnt/CDDiskPack/CDinstall_Astra_1.7.4/sslcert
/mnt/CDDiskPack/CDinstall_Astra_1.7.4/sslcert#
```
##### Генерируем самоподписанные сертификаты.
Генерация самоподписанных SSL-сертификатов включает в себя три простых шага:

- Шаг 1: Создайте закрытый ключ сервера
```bash
openssl genrsa -out it.company.lan.key 2048
```
![image](https://github.com/user-attachments/assets/e85db849-8f10-4fd0-a6ca-439c41394a2d)

- Шаг 2: Создайте запрос подписи сертификата (CSR)
```bash
openssl req -new -key it.company.lan.key -out it.company.lan.csr
```
![image](https://github.com/user-attachments/assets/12c662e8-cbdc-4622-ad39-e9e3ab0b7c4b)

- Шаг 3: Подпишите сертификат с помощью закрытого ключа и CSR
```bash
openssl x509 -req -days 365 -in it.company.lan.csr -signkey it.company.lan.key -out it.company.lan.crt
```
![image](https://github.com/user-attachments/assets/2243e650-0e9a-4da7-8e26-117a42fe054f)

Вы только что сгенерировали SSL-сертификат со сроком действия 365 дней.


##### Усиление безопасности сервера(можно пока не делать).
Инструкциия по усилению безопасности вашего сервера.

Для этого необходимо сгенерировать параметры Диффи-Хеллмана (DHE), обеспечивающие более высокую стойкость.
```bash
openssl dhparam -out dhparam.pem 2048
```
![image](https://github.com/user-attachments/assets/25c29913-9e2a-454a-b6be-e4d42922a724)

Проверяем
```bash
openssl req -in it.company.lan.csr -noout -text
```
![image](https://github.com/user-attachments/assets/3dc3494d-1189-4df5-bdf2-1e88b5b24914)

Далее переходим к скрипту запуска
```bash
mnt/CDDiskPack/CDinstall_Astra_1.7.4/sslcert/#
cd ..
/mnt/CDDiskPack/CDinstall_Astra_1.7.4/#
chmod +x offline_installer.sh
запускаем
./offline_installer.sh
```
![image](https://github.com/user-attachments/assets/caa684ee-d0b8-4618-bad7-ab36eda8d02b)

![image](https://github.com/user-attachments/assets/986d1e7b-d33f-4234-8fd7-aed5e9451eda)

![image](https://github.com/user-attachments/assets/9637b5a5-3114-486a-9dba-18c67441ba18)

![image](https://github.com/user-attachments/assets/878632c3-5ad9-445a-a670-3599e0723062)

![image](https://github.com/user-attachments/assets/42f7d7f0-3b6a-494b-a527-a15b7246c2b6)

![image](https://github.com/user-attachments/assets/2237c1d7-2c86-4491-ae5f-bea349f658cd)

![image](https://github.com/user-attachments/assets/7cce156d-c595-45fe-b81f-47d202d8555b)

![image](https://github.com/user-attachments/assets/0135cee8-32f3-446d-91df-b1ca94db8e9a)

![image](https://github.com/user-attachments/assets/3f5778a3-0834-4aa3-a11d-a182a9bb3aa8)

![image](https://github.com/user-attachments/assets/83f23eaf-2540-4dca-a145-1d2c7066b182)

![image](https://github.com/user-attachments/assets/84e27862-4db4-4d62-a272-eaa53aa900d3)

![image](https://github.com/user-attachments/assets/b8b03d01-4e58-46ef-ac1d-cb19e0719d60)

![image](https://github.com/user-attachments/assets/10eff70d-b8db-421e-ad40-69772b77fa5d)

![image](https://github.com/user-attachments/assets/5afb9ace-7c43-48b1-83ce-22cd7e17e2ed)

![image](https://github.com/user-attachments/assets/2f2e3341-85de-458d-8aac-40ca5e562733)

![image](https://github.com/user-attachments/assets/79e5b956-5dbf-4db6-8240-77bacc7ed8df)

>[!Warning]
>Если Вы выбрали установки без HTTPS, то, после инсталляции, почтовый сервер работать не будет. Он не запустится.

![image](https://github.com/user-attachments/assets/ae23bfe7-4777-4147-ae3a-4776c64e94a2)

![image](https://github.com/user-attachments/assets/6aba3925-df9e-4c4b-ba06-c5dd15954b0d)

![image](https://github.com/user-attachments/assets/64ec3ab8-8845-4765-bdf5-59e5af56220d)

![image](https://github.com/user-attachments/assets/b7bfadab-6d3f-4f86-aa85-b9e1306ff9cf)

![image](https://github.com/user-attachments/assets/8135dcfa-eaae-46f7-bb11-27517b0ea0a3)

![image](https://github.com/user-attachments/assets/02f3207b-6504-493e-a376-f533b36afa0f)

![image](https://github.com/user-attachments/assets/8a648089-4a26-41d2-ba66-47a275ab64cc)

![image](https://github.com/user-attachments/assets/a3bc029d-20b3-4918-860b-77c4556ccbef)

Если всё сделали правильно в конце отработки скрипта получаем это

![image](https://github.com/user-attachments/assets/b838aff3-72b6-4ae4-ae91-b6326baa00f7)

![image](https://github.com/user-attachments/assets/7fa05785-fa54-4e92-b507-a63d709708bc)

Заходим

![image](https://github.com/user-attachments/assets/896a0b8a-2ba6-43a3-844c-69371444be2f)

![image](https://github.com/user-attachments/assets/e05af1a9-e05a-49e0-adf7-af8fa64f2e44)

![image](https://github.com/user-attachments/assets/562ab3b9-f729-40a9-ba95-1bea86127cab)

![image](https://github.com/user-attachments/assets/fd12a237-f8cd-4d04-a1fd-7653eb6b54e8)

![image](https://github.com/user-attachments/assets/254bffd7-6831-4287-a0eb-24214866040f)

![image](https://github.com/user-attachments/assets/f4025aea-f015-4df2-8d7e-9cfaf3ddc3ad)

#### Установка почтового сервера R7.

>[!Warning]
>Рекомендуем, перед продолжением инсталляции, прописать записи в DNS, для работы почтового сервера.

Необходимо добавить запись А (ваш почтовый сервер hostname -f) и обратную запись, а также запись MX и TXT v=spf1 +mx ~all

Пример:

|        |it.company.lan       	|TTL          	|Приоритет
|--------|-----------------------|---------------|--------------|
|MX	 |astra7.it.company.lan  |300            |10		|
|TXT     |it.company.lan         |TTL            |		|
|	 |v=spf1+mx~all          |300            |		|
|A       |astra7.it.company.lan  |TTL            |		|
|	 |192.168.18.141          |300            |		|
   
Теперь по подробней.

**MX-запись** — это тип DNS-записи, который указывает, какой сервер отвечает за прием электронной почты для домена.<br> 
MX-запись указывает на доменное имя почтового сервера (например, в нашем случае - astra7.it.company.lan), которое должно иметь A-запись (IPv4), чтобы связать домен с IP-адресом. Чем меньше число, тем выше приоритет.

>MX-записи не могут указывать напрямую на IP-адрес, только на доменное имя.

Проверить MX-записи можно через команды в терминале:
```bash
nslookup -type=mx it.company.lan
или так
dig mx it.company.lan 
```
Запись ``v=spf1 +mx ~all`` в DNS — это SPF-запись, которая помогает защитить домен от подделки электронной почты (спуфинга). <br>Она указывает, какие серверы имеют право отправлять письма от имени вашего домена.<br>
SPF-запись v=spf1 +mx ~all разрешит отправку писем только с astra7.it.company.lan, а письма с других серверов будут отклонены или помечены.

Если выбрали, то установка запустится автоматически.

![40](https://github.com/user-attachments/assets/3b0169ce-0a52-4fcd-bd48-c21225b4dc85)

Если нет, то надо добавить сертификаты п. выше и запустить в ручную скрипт ``install_mailserver.sh``

![41](https://github.com/user-attachments/assets/30ce2450-33aa-4bfb-8a33-811aa251a2fa)

![42](https://github.com/user-attachments/assets/610075b4-b851-4dd6-b499-b17a9af3de32)

![43](https://github.com/user-attachments/assets/396395fb-010e-4b6d-8c33-a9d1c7243f25)

![44](https://github.com/user-attachments/assets/e0cb49e9-6577-422a-b485-954d502aea6e)

![45](https://github.com/user-attachments/assets/6c0ebd51-4130-436e-92f9-d9af2ddc4be9)

**DKIM** — это технология аутентификации электронной почты, которая позволяет проверить, что письмо действительно отправлено от имени домена и не было изменено в процессе доставки. DKIM использует цифровую подпись, чтобы подтвердить подлинность отправителя и целостность содержимого письма.<br>
DKIM-запись хранится в DNS домена в формате TXT-записи и выглядит примерно так:
```bash
selector._domainkey.it.company.lan.  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC..."
```
После инсталляции в консоли будет предложено сделать TXT запись.


