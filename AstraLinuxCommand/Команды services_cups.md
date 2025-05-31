##### Посмотреть доступность служб.
```bash
top
service --status-all
systemctl status cups
```
```bash
systemctl list-units --type service
systemctl list-units --type service -all
```
![image](https://github.com/user-attachments/assets/f65afaee-ae8f-4d41-8fdc-cdefe852625b)

##### отфильтруем только службы
```bash
systemctl list-unit-files --type service
```
![image](https://github.com/user-attachments/assets/bbeae0f2-9374-4799-a68f-a4e7a1d0f28a)

##### Запущеные
```bash
systemctl list-units --type service --state running
```
![image](https://github.com/user-attachments/assets/9eecf347-ca85-4538-b370-69f258f99ab6)

##### Не запущенные
```bash
systemctl list-units --type service --state failed
systemctl list-unit-files --type service --state disabled
```
##### Запускаем службу
```bash
sudo systemctl start cups.service
```
##### Остановить службу 
```bash
sudo systemctl stop cups
```
##### Состояние службы
```bash
sudo systemctl status cups
```
##### Службы запускаемые автоматические
```bash
systemctl list-unit-files --state enabled
```
![image](https://github.com/user-attachments/assets/b6c3fb5e-3bc6-4e0e-9b66-3b2977e23c06)

##### чтобы включить ``localhost:631``
```bash
sudo cupsctl WebInterface=Yes
```
![image](https://github.com/user-attachments/assets/26c7f88a-10b8-4ad1-94b9-e008001c7dea)

##### перезапуск службы cups
```bash
sudo /etc/init.d/cups restart
```
##### проверка на ошибки службы cupsd
```bash
#cupsd -h
cupsd -t
```
##### добавить службу в автозагрузку
```bash
sudo systemctl enable cups
```
##### Убрать из автозагрузки службу
```bash
sudo systemctl disable
```
##### Посмотреть разрешена ли сейчас автозагрзука для службы
```bash
sudo systemctl is-enabled chrony 
```
##### Проверка доступности USB принтера
```bash
lsusb
lsmod | grep usb
```
##### Подгружаем модули
```bash
sudo modprobe usbcore
sudo modprobe usbhid
sudo modprobe ehci_hcd
sudo modprobe ohci_hcd
sudo modprobe usb-storage
```
##### Обновляем ядро
```bash
sudo depmod -a
```
