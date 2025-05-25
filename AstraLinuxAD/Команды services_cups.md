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
##### отфильтруем только службы
```bash
systemctl list-unit-files --type service
```
##### Запущеные
```bash
systemctl list-units --type service --state running
```
##### Не запущенные
```bash
systemctl list-units --type service --state failed
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
##### чтобы включить ``localhost:631``
```bash
sudo cupsctl WebInterface=Yes
```
##### перезапуск службы cups
```bash
sudo /etc/init.d/cups restart
```
##### проверка на ошибки службы cupsd
```bash
cupsd -t   #cupsd -h
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
sudo systemctl is-enabled cups 
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
