#### Установка Корпоративного сервера 2024 offline-версии для ОС Astra.

Установка произваодится на VM Ware Workstation, версия ОС Астра 1.7.4 обновлена до 1.7.7.9

Предварительная подготовка.

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
apt install fly-astra-update // обновление через графический интерфейс
```
![image](https://github.com/user-attachments/assets/eddc9d5e-6f22-4436-9473-5c3d796eda11)

![image](https://github.com/user-attachments/assets/b58a3d64-660f-4010-9e4d-273ee12cb15a)


Команда ``fly-astra-update``  запустит графический интерфейс

Команда ``apt update -A -r -T`` аналогична с командой ``dist-upgrade``

После обновления проверим версию ``cat /etc/astra/build_version``

![image](https://github.com/user-attachments/assets/408fd6bd-2f84-46fa-81da-038e968ff2e4)

Идём дальше

Скачиваем архив CDinstall_2.0.2024.14752_Astra_1.7.4_offline.zip
закидываем в корень папки ``/mnt`` переходим в неё и распаковываем
```bash
cd /mnt/
unzip CDinstall_2.0.2024.14752_Astra_1.7.4_offline.zip
```
После распаковки заходим в папку
```bash
cd /mnt/CDDiskPack/CDinstall_Astra_1.7.4/sslcert
```
Создаём сертификаты
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout it.company.lan.key -out it.company.lan.crt
```
- it.company.lan.* меняем на свои доменные имена

![image](https://github.com/user-attachments/assets/3e87eb36-1a2b-4553-9329-f24a3bdf7337)
  
![image](https://github.com/user-attachments/assets/dc9d72ad-64bc-4e56-a62b-e4b8e31a775b)

Далее переходим к скрипту запуска
```bash
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

![image](https://github.com/user-attachments/assets/fc587c14-bb22-4463-8bdd-08fa4baa9c5f)

>[!Warning]
>Если Вы выбрали установки без HTTPS, то, после инсталляции, почтовый сервер работать не будет. Он не запустится.

Для его работы необходимо положить сертификаты по пути:
```bash
smtpd_tls_cert_file = /etc/nginx/ssl/it.company.lan.crt
smtpd_tls_key_file = /etc/nginx/ssl/it.company.lan.key
```
Продолжим установку.

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
   
Если выбрали, то установка запустится автоматически.

![40](https://github.com/user-attachments/assets/3b0169ce-0a52-4fcd-bd48-c21225b4dc85)

Если нет, то надо добавить сертификаты п. выше и запустить в ручную скрипт ``install_mailserver.sh``

![41](https://github.com/user-attachments/assets/30ce2450-33aa-4bfb-8a33-811aa251a2fa)

![42](https://github.com/user-attachments/assets/610075b4-b851-4dd6-b499-b17a9af3de32)

![43](https://github.com/user-attachments/assets/396395fb-010e-4b6d-8c33-a9d1c7243f25)

![44](https://github.com/user-attachments/assets/e0cb49e9-6577-422a-b485-954d502aea6e)

![45](https://github.com/user-attachments/assets/6c0ebd51-4130-436e-92f9-d9af2ddc4be9)



