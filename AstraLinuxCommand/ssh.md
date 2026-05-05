#### Раскомментировать PermitRootLogin и установить значение yes:
##### Способ 1: Одной командой sed
```bash
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
```
##### Способ 2: Более точный вариант
```bash
sudo sed -i '/^#\?PermitRootLogin[[:space:]]/ s/.*/PermitRootLogin yes/' /etc/ssh/sshd_config
```
##### Способ 3: Раскомментировать и установить значение
```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
```
##### Способ 4: С использованием awk (более безопасно)
```bash
sudo awk '/^#?PermitRootLogin/ {$1="PermitRootLogin"; $2="yes"} 1' /etc/ssh/sshd_config | sudo tee /etc/ssh/sshd_config.tmp && sudo mv /etc/ssh/sshd_config.tmp /etc/ssh/sshd_config
```
##### Способ 5: Если нужно раскомментировать только и установить yes (игнорируя другие значения):
```bash
sudo sed -i '0,/^#*PermitRootLogin/ s/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
```
##### Способ 6: Полный контроль через несколько команд
###### Создать резервную копию
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```
###### Раскомментировать и изменить значение
```bash
sudo sed -i '/PermitRootLogin/ s/^#//' /etc/ssh/sshd_config
sudo sed -i '/PermitRootLogin/ s/no/yes/' /etc/ssh/sshd_config
sudo sed -i '/PermitRootLogin/ s/prohibit-password/yes/' /etc/ssh/sshd_config
sudo sed -i '/PermitRootLogin/ s/without-password/yes/' /etc/ssh/sshd_config
```
****
##### Проверить изменения:
```bash
grep -i "PermitRootLogin" /etc/ssh/sshd_config
```
##### Перезапустить SSH службу для применения изменений:
######  Для системы с systemd (большинство современных дистрибутивов)
```bash
sudo systemctl restart sshd
```
######  или
```bash
sudo systemctl restart ssh
```
######  Для старых систем
```bash
sudo service ssh restart
```
######  или
```bash
sudo /etc/init.d/ssh restart
```
##### Проверить, что изменения применились:
```bash
sudo sshd -T | grep permitrootlogin
```
##### Как правильно выполнить удалённую команду через SSH
1. Прямое выполнение команды (сессия закроется после завершения):

```bash
ssh root@10.1.100.19 "update-ca-certificates -v"
```
или (без кавычек, если команда без пробелов и спецсимволов):
```bash
ssh root@10.1.100.19 update-ca-certificates -v
```
несколько команд
```bash
ssh root@10.1.100.19 "update-ca-certificates -v && update-ca-certificates --fresh"
```
или правильно так, можно просто выполнить --fresh, который сам полностью обновляет сертификаты (без отдельного первого вызова):
```bash
ssh root@10.1.100.19 "update-ca-certificates --fresh -v"
```
> Пояснение:
> - ``update-ca-certificates`` без аргументов просто обновляет сертификаты (добавляет новые).
> - ``--fresh`` удаляет все существующие и пересоздаёт из текущих источников. Обычно достаточно одного --fresh, отдельный предварительный вызов не нужен.
> - Опция ``-v (или --verbose)`` увеличивает детализацию вывода.

2. Интерактивный вход (сначала подключиться, потом вручную ввести команду):
```bash
ssh root@10.1.100.19
```
После входа выполнить:
```bash
update-ca-certificates --fresh -v
```
3. Команды удаления
  - Если файлов нет – rm выдаст ошибку No such file or directory. Чтобы этого избежать и не видеть ошибку, можно использовать rm -f:
```bash
ssh root@10.1.100.19 "rm -f /mnt/cert/r01*"
```
  - Проверьте, что удаляете именно то, что нужно – r01* удалит все файлы, чьи имена начинаются с r01. Можно сначала посмотреть список:
```bash
ssh root@10.1.100.19 "ls -la /mnt/cert/r01*"
```
Безопасность – удаляете от root, поэтому будьте уверены, что путь и паттерн верны.<br>
  - Рекурсивное удаление эта команда не делает (если среди r01* есть директории, rm без -r их не удалит, выдаст ошибку). Если нужна рекурсия – добавьте -r:
```bash
ssh root@10.1.100.19 "rm -rf /mnt/cert/r01*"
```
> Итог: ваша команда корректна, но для повседневного безопасного использования часто добавляют ``-f``.

