##### Установите необходимые пакеты:

Убедитесь, что у вас установлены пакеты `cifs-utils` (или `smbclient` в некоторых дистрибутивах), которые содержат утилиты для работы с SMB.

Например, в Debian/Ubuntu: `sudo apt-get install cifs-utils`

В Fedora/CentOS/RHEL: `sudo dnf install cifs-utils`

##### Подключаем сетевой диск по имени пользователя домена:
```bash
sudo mkdir /mnt/utils
sudo nano /etc/fstab
```
добавляем строку
```bash
//192.168.25.253/aldpro /mnt/utils/ cifs username=sol,password=*********,rw,nounix,iocharset=utf8,file_mode=0775,dir_mode=0775

меняем название папки и логин с паролем на свои

file_mode=0775,dir_mode=0775 - режимы доступа к файлам и папкам

```
##### Подключаемся к сетевому ресурсу через терминал:
```bash
     sudo smbclient -U sol //192.168.25.253/aldpro

домен можно не указывать, если ваш комп заведён в домен

```
##### Подключиться к сетевому ресурсу через проводник:
Открываете проводник, выбераете сеть и вводите такой путь:
```bash
     smb://192.168.25.253/aldpro 
ENTER
	вводите логин, домен, пароль.
```
##### Подключиться к сетевому ресурсу через терминал с правами:
```bash
mount -t cifs //192.168.25.253/aldpro /mnt/utils/ -o username=sol,password=g*********,rw,nounix,iocharset=utf8,file_mode=0775,dir_mode=0775
mount -a
```
##### Отмонтировать сетевой ресурс
```bash
umount -f cifs //192.168.25.253/aldpro /mnt/utils
или
umount -l cifs //192.168.25.253/aldpro /mnt/utils
```
