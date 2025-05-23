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


Если всё сделали правильно в конце отработки скрипта получаем это

![image](https://github.com/user-attachments/assets/b838aff3-72b6-4ae4-ae91-b6326baa00f7)

![image](https://github.com/user-attachments/assets/7fa05785-fa54-4e92-b507-a63d709708bc)

Заходим

![image](https://github.com/user-attachments/assets/896a0b8a-2ba6-43a3-844c-69371444be2f)

