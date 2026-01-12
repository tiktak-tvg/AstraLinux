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
##### Способ 5: Полный контроль через несколько команд
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
