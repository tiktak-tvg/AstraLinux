### Как удалить postgresql 11

#### 1. Остановите службы PostgreSQL
```bash
sudo systemctl stop postgresql
```
#### 2. Удалите пакеты PostgreSQL
#####  Удалите основной пакет и зависимости
```bash
sudo apt purge postgresql postgresql-11 postgresql-client-11
```
#####  Или удалите все пакеты, связанные с postgresql
```bash
sudo apt purge postgresql*
```
#### 3. Удалите оставшиеся файлы и каталоги
#####  Удалите каталог с данными
```bash
sudo rm -rf /var/lib/postgresql/
```
#####  Удалите конфигурационные файлы
```bash
sudo rm -rf /etc/postgresql/
```
#####  Удалите файлы логов
```bash
sudo rm -rf /var/log/postgresql/
```
#### 4. Удалите пользователя и группу postgres (опционально)
```bash
sudo userdel postgres
sudo groupdel postgres
```
#### 5. Очистите систему от ненужных зависимостей
```bash
sudo apt autoremove
sudo apt autoclean
```
#### 6. Проверьте, что всё удалено
##### Проверьте установленные пакеты
```bash
dpkg -l | grep postgres
```
#####  Проверьте наличие оставшихся файлов
```bash
sudo find / -name "*postgres*" 2>/dev/null
```
#### Альтернативный способ - одной командой:
```bash
sudo apt purge postgresql\* && sudo rm -rf /var/lib/postgresql/ /etc/postgresql/ /var/log/postgresql/ && sudo apt autoremove
```
После выполнения этих команд PostgreSQL 11 будет полностью удалён из вашей системы.
