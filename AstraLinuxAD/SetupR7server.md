#### Установка Корпоративного сервера 2024 offline-версии для ОС Astra.

Установка произваодится на VM Ware Workstation, версия ОС Астра 1.7.4 обновлена до 1.7.7.9

Предварительная подготовка.

Устанавливаем ОС Астра 1.7.4 образ 1.7.4-24.04.2023_14.23.iso

После установки выполняем команды:
```bash
apt install open-vm-tools
```
Открываем репозитории `` nano /etc/apt/sources.list``

![image](https://github.com/user-attachments/assets/a9342a3c-bdbd-44fb-8f90-19b6b16f4065)

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
