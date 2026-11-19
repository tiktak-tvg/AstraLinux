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
```bash
ssh root@10.1.100.19 "mkdir -p /mnt/plugins"
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

### Как создать ssh-key в mobaX и скопировать его на виртуальную машину 192.168.1.14
В MobaXterm утилита ssh-copy-id уже встроена в локальный терминал и работает так же, как в Linux. Скопировать открытый SSH-ключ на сервер можно двумя способами: через командную строку или графический интерфейс.

#### Способ 1. Через локальный терминал MobaXterm
Этот вариант самый быстрый и позволяет использовать привычную команду без установки дополнительных пакетов.
- Откройте локальный терминал. Нажмите кнопку Start local terminal на главном экране MobaXterm или откройте новую вкладку через +.
- Создайте пару ключей. Если у вас еще нет ключа, сгенерируйте его командой ssh-keygen -t ed25519 (или ssh-keygen -t rsa -b 4096) и нажмите Enter на все запросы.
- Отправьте ключ на сервер, который имеет доступ до машины 192.168.1.14. Выполните команду, подставив имя пользователя и IP-адрес вашего сервера (у меня 192.168.1.10):

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.1.10
```
- Подтвердите подключение. Введите yes при первом соединении и укажите пароль от учетной записи на сервере.
- Укажите нестандартный порт. Если SSH работает не на 22 порту, добавьте флаг -p:

```bash
ssh-copy-id -p 2222 -i ~/.ssh/id_ed25519.pub root@192.168.1.10
```
Выполните на сервере developers(192.168.1.10) две команды по очереди:

1. Извлеките публичный ключ из приватного:
```bash
ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub
```
3. Отправьте созданный публичный ключ на сервер 10.1.100.14:
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@10.1.100.14
```

<img width="1163" height="280" alt="image" src="https://github.com/user-attachments/assets/4d75549a-b9f4-45db-8f62-12b816fc3b26" />

Что делать дальше:

- Подключитесь в MobaXterm. Откройте настройки сессии для 10.1.100.14 -> Advanced SSH settings -> поставьте галочку Use private key и выберите ваш файл C:\Users\sol\.ssh\id_ed25519.

<img width="1225" height="726" alt="image" src="https://github.com/user-attachments/assets/b95c973b-4e7b-4956-92c8-2fa097601ed1" />

- На своём компьютере тоже положите ключ

<img width="1215" height="553" alt="image" src="https://github.com/user-attachments/assets/ac2b7bba-c0a6-409a-8eeb-830bcabab140" />


#### Способ 2. Если путь к ключу находится в Windows
В локальном терминале MobaXterm используется файловая система формата Cygwin, поэтому диски Windows монтируются в папку /drives/.
- Переведите путь в формат MobaXterm. Файл C:\Users\Name\.ssh\id_rsa.pub в терминале программы будет выглядеть как /drives/c/Users/Имя пользователя/.ssh/id_rsa.pub.
- Запустите копирование с новым путем. Выполните команду:

```bash
ssh-copy-id -i /drives/c/Users/Имя пользователя/.ssh/id_rsa.pub user@192.168.1.10
```

<img width="1187" height="95" alt="image" src="https://github.com/user-attachments/assets/a660ae39-21d7-4add-91a1-02475a1f0f67" />

#### Способ 3. Через графический менеджер ключей (MobaKeyGen)
Если ключ был создан через встроенный генератор в интерфейсе программы, его удобнее перенести вручную.
- Откройте генератор. Перейдите в верхнее меню Tools и выберите MobaKeyGen (SSH key generator).
- Скопируйте публичный ключ. Загрузите ваш закрытый ключ через кнопку Load и скопируйте весь текст из верхнего поля Public key for pasting into OpenSSH authorized_keys file.
- Добавьте ключ на сервер. Подключитесь к серверу по паролю и вставьте скопированную строку в конец файла ~/.ssh/authorized_keys.
- Привяжите ключ к сессии. В настройках сессии MobaXterm перейдите на вкладку Advanced SSH settings, поставьте галочку Use private key и укажите путь к файлу закрытого ключа.

<img width="1391" height="827" alt="image" src="https://github.com/user-attachments/assets/82ff8340-633e-46a7-a5d2-e8e094281da2" />

Полезные советы при ошибках

- Включите постоянную домашнюю директорию. Если после перезапуска MobaXterm ключи пропадают из ~/.ssh/, зайдите в Settings -> Configuration -> General и укажите постоянную папку для Persistent home directory.
- Проверьте права на сервере. Если после копирования сервер все равно запрашивает пароль, выполните на нем команду chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys.

