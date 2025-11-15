### В Linux есть несколько способов монтирования сетевых дисков. Вот основные методы:
#### 1. Монтирование SMB/CIFS (Windows-шары)
Базовая команда:
```bash
mount -t cifs //SERVER_IP/SHARE_NAME /mnt/mount_point -o username=USERNAME,password=PASSWORD
```
Пример:
```bash
mount -t cifs //192.168.1.100/shared /mnt/myshare -o username=user,password=pass
```
Безопасный способ с файлом учетных данных:
#####  Создаем файл с учетными данными
```bash
echo "username=user" > ~/.smbcredentials
echo "password=pass" >> ~/.smbcredentials
chmod 600 ~/.smbcredentials
```
#####  Монтируем
```bash
mount -t cifs //192.168.1.100/shared /mnt/myshare -o credentials=~/.smbcredentials
```
#### 2. Монтирование NFS
```bash
mount -t nfs SERVER_IP:/export/path /mnt/mount_point
```
Пример:
```bash
mount -t nfs 192.168.1.100:/home/share /mnt/nfs_share
```
#### 3. Автоматическое монтирование через /etc/fstab
Для SMB/CIFS добавить в /etc/fstab:
```text
//192.168.1.100/shared /mnt/myshare cifs credentials=/home/user/.smbcredentials,iocharset=utf8,file_mode=0777,dir_mode=0777 0 0
Для NFS добавить в /etc/fstab:
```
```text
192.168.1.100:/home/share /mnt/nfs_share nfs defaults 0 0
```
После редактирования fstab:
```bash
mount -a  # Монтирует все из fstab
```
#### 4. Практические примеры с опциями
SMB с русскими символами:
```bash
mount -t cifs //server/share /mnt/share -o username=user,password=pass,iocharset=utf8,file_mode=0777,dir_mode=0777
```
SMB с указанием версии:
```bash
mount -t cifs //server/share /mnt/share -o username=user,password=pass,vers=3.0
```
NFS с опциями:
```bash
mount -t nfs server:/path /mnt/nfs -o rw,hard,intr,timeo=300,retrans=3
```
#### 5. Проверка и размонтирование
Проверить смонтированные ресурсы:
```bash
mount | grep -E "(cifs|nfs)"
df -h
```
Размонтировать:
```bash
umount /mnt/mount_point
```
#### 6. Установка необходимых пакетов
Для Ubuntu/Debian:
```bash
apt update
apt install cifs-utils nfs-common
```
Для CentOS/RHEL:
```bash
yum install cifs-utils nfs-utils
```
#### 7. Постоянное монтирование (через systemd)
Создать файл /etc/systemd/system/mnt-myshare.mount:
```ini
[Unit]
Description=Mount Network Share
After=network.target

[Mount]
What=//192.168.1.100/shared
Where=/mnt/myshare
Type=cifs
Options=credentials=/home/user/.smbcredentials,iocharset=utf8,file_mode=0777,dir_mode=0777

[Install]
WantedBy=multi-user.target
Активировать:
bash
systemctl daemon-reload
systemctl enable mnt-myshare.mount
systemctl start mnt-myshare.mount
```
Советы:
•	Создайте точку монтирования: mkdir -p /mnt/myshare
•	Проверьте доступность сервера: ping SERVER_IP
•	Для отладки используйте опцию -v (verbose)
•	При проблемах с правами используйте uid и gid в опциях монтирования

Выберите метод в зависимости от типа сетевого ресурса и ваших потребностей!
