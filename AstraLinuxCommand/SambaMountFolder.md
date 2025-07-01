##### Подключаем сетевой диск по имени пользователя домена:
```bash
sudo gedit /etc/fstab
редактируем строку //192.168.25.253/aldpro /mnt/utils/ cifs username=sol,password=*********,rw,nounix,iocharset=utf8,file_mode=0775,dir_mode=0775

меняем название папки и логин с пароле
```
##### Подключаемся к сетевому ресурсу через терминал:
```bash
     sudo smbclient -U sol //192.168.25.253/aldpro

домен можно не указывать, если ваш комп заведён в домен

```
