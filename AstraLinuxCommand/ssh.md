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

