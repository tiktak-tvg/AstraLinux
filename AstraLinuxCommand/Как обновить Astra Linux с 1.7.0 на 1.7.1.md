#### Как обновить через командную строку.

Для этого необходимо установить apt-transport-https и ca-certificates, для обновления по протоколу https:
```bash
sudo apt install apt-transport-https ca-certificates
```
или прописать (на время обновления) адреса репозиториев через http

Редактируем файл /etc/apt/sources.list:
```bash
sudo nano /etc/apt/sources.list
```
Добавляем следующую строку:
```bash
deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-base/ 1.7_x86-64 main contrib non-free
или
deb http://download.astralinux.ru/astra/stable/1.7_x86-64/repository-base/ 1.7_x86-64 main contrib non-free
```
Обновляем списки пакетов:
```bash
sudo apt update
```
Обновляемся:
```bash
sudo astra-update -A -r
```
или аналогичная команда
```bash
sudo apt dist-upgrade
```
