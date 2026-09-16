### Создание сценариев bash в Astra Linux
**Bash (Bourne Again SHell)** — это основная командная оболочка в Astra Linux и большинстве Linux-дистрибутивов. Сценарии bash позволяют автоматизировать рутинные задачи администрирования, развёртывания и обслуживания системы.

#### 📜 Сценарий bash
Сценарий (скрипт) — это текстовый файл, содержащий последовательность команд, которые интерпретатор bash выполняет построчно. Сценарии позволяют автоматизировать повторяющиеся действия, объединять сложные цепочки команд и создавать собственные утилиты.

##### Структура простого сценария
```bash
#!/bin/bash
# Комментарий: первая строка — это shebang, указывающий интерпретатор

echo "Привет, Astra Linux!"
```
Обязательные элементы:

**Shebang (#!/bin/bash)** — указывает ядру, какой интерпретатор использовать. Без неё сценарий может быть выполнен текущей оболочкой, что не всегда предсказуемо.

**Комментарии (# текст)** — игнорируются интерпретатором, служат для документирования.

##### Запуск сценария
```bash
# Способ 1: сделать файл исполняемым и запустить
chmod +x script.sh
./script.sh

# Способ 2: передать интерпретатору как аргумент
bash script.sh

# Способ 3: выполнить в текущей оболочке (без создания подпроцесса)
source script.sh
```
##### Соглашения об именовании

- Расширение ``.sh`` — не обязательно, но полезно для читаемости.

- Имя файла — осмысленное, отражающее назначение (например, ``backup_home.sh, check_disk.sh``).

- Расположение: пользовательские сценарии — ``/usr/local/bin/`` или ``~/bin/``, системные — ``/usr/local/sbin/``.

#### 🔢 Переменные
Переменные в bash — это именованные области памяти, хранящие значения. Они не требуют объявления типа.

##### Определение и использование
```bash
# Присваивание (без пробелов вокруг знака =)
NAME="Astra Linux"
VERSION=1.7
COUNT=42

# Использование значения
echo "Дистрибутив: $NAME, версия: $VERSION"
echo "Дистрибутив: ${NAME}, версия: ${VERSION}"   # Рекомендуемая форма
```
> ⚠️ Важно: Между именем переменной, знаком = и значением не должно быть пробелов. NAME = "value" — ошибка.

##### Типы переменных

Тип                 | Пример                    | Описание
------------------- | ------------------------- | -------------------------
Пользовательские    | 	MY_VAR="test"	        | Определяются пользователем в сценарии.
Переменные окружение| PATH, HOME, USER	        | Наследуются дочерними процессами.
Специальные	        | $0, $1, $#, $?, $$	    | Управление сценарием.
Только для чтения	| readonly CONST=10	        | Изменить нельзя.

##### Специальные переменные

Переменная          | 	Значение
------------------- | -------------------------
$0	                | Имя сценария.
$1, $2, ...	        | Аргументы командной строки.
$#	                | Количество аргументов.
$@	                | Все аргументы как отдельные строки.
$*	                | Все аргументы одной строкой.
$?	                | Код возврата последней команды.
$$	                | PID текущего сценария.
$!	                | PID последнего фонового процесса.

##### Работа со строками и числами
```bash
NAME="Astra"
SURNAME="Linux"
FULL="$NAME $SURNAME"           # Конкатенация
LENGTH=${#FULL}                 # Длина строки: 11

# Арифметика (несколько способов)
A=5
B=3
SUM=$((A + B))                  # Арифметическое выражение
let "MUL = A * B"               # Через let
DIV=$(expr $A / $B)             # Через expr (устаревший способ)
```
##### Подстановка команд
```bash
# Значение переменной = вывод команды
DATE=$(date +%Y-%m-%d)
KERNEL=$(uname -r)
USERS=$(who | wc -l)
echo "Сегодня $DATE, ядро $KERNEL, пользователей: $USERS"
```
##### Область видимости
```bash
# Локальная переменная (только внутри функции)
my_func() {
    local LOCAL_VAR="local value"
    echo "$LOCAL_VAR"
}

# Глобальная переменная (доступна везде)
GLOBAL_VAR="global value"
```
##### Экспорт переменных
```bash
# Переменная окружения — доступна дочерним процессам
export MY_ENV_VAR="shared"
#### ⌨️ Ввод и вывод данных
```
##### Вывод: echo и printf
```bash
# echo — простой вывод с переносом строки
echo "Привет, мир"
echo -n "Без переноса строки: "
echo -e "С табуляцией:\tтекст"

# printf — форматированный вывод (надёжнее)
printf "%-10s %5d\n" "Всего:" 42
printf "Файл: %s, размер: %d байт\n" "test.txt" 1024
```
##### Ввод: read
```bash
# Простой ввод
read -p "Введите ваше имя: " NAME
echo "Здравствуйте, $NAME!"

# С ограничением по времени (таймаут 5 секунд)
read -t 5 -p "Быстро ответьте: " ANSWER

# Скрытый ввод (для паролей)
read -s -p "Пароль: " PASSWORD
echo    # Перенос строки после скрытого ввода

# С ограничением количества символов
read -n 1 -p "Нажмите любую клавишу: " KEY
```
##### Перенаправление потоков

Оператор            | 	Действие
------------------- | -------------------------------------------
' > '	            | Перенаправить stdout в файл (перезапись).
' >> '	            | Перенаправить stdout в файл (дозапись).
' < '	            | Перенаправить stdin из файла.
' 2> '              | Перенаправить stderr в файл.
' &> '	            | Перенаправить stdout и stderr.
' 2>&1              | Объединить stderr с stdout.
' |	'               | Конвейер: stdout одной команды → stdin другой.

```bash
# Примеры
ls /nonexistent 2> /dev/null                  # Скрыть ошибки
command > output.log 2>&1                     # Всё в лог
echo "data" | tee file.txt                    # Вывод и запись одновременно
```
##### Чтение файла построчно
```bash
while IFS= read -r line; do
    echo "Строка: $line"
done < /etc/hosts
Важно: IFS= и -r предотвращают обрезку пробелов и интерпретацию обратных слэшей.
```
#### 🧩 Алгоритмические конструкции
##### Условные операторы
```
if / elif / else

bash
if [ "$USER" = "root" ]; then
    echo "Вы администратор"
elif [ "$USER" = "guest" ]; then
    echo "Вы гость"
else
    echo "Вы обычный пользователь: $USER"
fi
```
##### Операторы сравнения:

Оператор	| Числа	| Строки	| Файлы
----------- | ----- | --------- | -----------------
Равно	 | -eq	 | =	 | —
Не равно	 | -ne	 | !=	 | —
Больше	 | -gt	 | > (в [[ ]])	 | —
Меньше	 | -lt	 | < (в [[ ]])	 | —
Файл существует | 	—	 | —	 | -e
Это файл	 | —	 | —	 | -f
Это каталог	 | —	 | —	 | -d
Файл пустой	 | —	 | —	 | -s (непустой)
Доступен для чтения	 | —	 | —	 | -r
Доступен для записи	 | —	 | —	 | -w
Исполняемый	 | —	 | —	 | -x

##### Логические операторы:
```bash
# И (AND), ИЛИ (OR), НЕ (NOT)
if [ "$A" -gt 0 ] && [ "$B" -lt 10 ]; then echo "OK"; fi
if [ "$A" -eq 1 ] || [ "$B" -eq 2 ]; then echo "OK"; fi
if [ ! -f "/etc/passwd" ]; then echo "Файл отсутствует"; fi

# В [[ ]] можно использовать && и || напрямую
if [[ "$A" -gt 0 && "$B" -lt 10 ]]; then echo "OK"; fi
```
##### case
```bash
case "$1" in
    start)
        echo "Запуск службы..."
        ;;
    stop)
        echo "Остановка службы..."
        ;;
    restart|reload)
        echo "Перезапуск службы..."
        ;;
    *)
        echo "Использование: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```
##### Циклы
for по списку

```bash
for FILE in /etc/*.conf; do
    echo "Конфигурация: $FILE"
done

# C-подобный синтаксис
for ((i=1; i<=5; i++)); do
    echo "Итерация $i"
done
```
##### while

```bash
COUNT=1
while [ $COUNT -le 5 ]; do
    echo "Счётчик: $COUNT"
    ((COUNT++))
done

# Бесконечный цикл с условием выхода
while true; do
    read -p "Введите 'quit' для выхода: " CMD
    [ "$CMD" = "quit" ] && break
done
```
##### until — цикл, пока условие ложно:

```bash
COUNT=1
until [ $COUNT -gt 5 ]; do
    echo "Счётчик: $COUNT"
    ((COUNT++))
done
```
##### Управление циклом:

```bash
for i in {1..10}; do
    [ $i -eq 5 ] && continue    # Пропустить итерацию
    [ $i -eq 8 ] && break       # Прервать цикл
    echo $i
done
```
#### 🧩 Функции
Функции позволяют инкапсулировать повторяющийся код. Они объявляются двумя способами:

```bash
# Способ 1
function my_func {
    echo "Функция вызвана"
}

# Способ 2 (POSIX-совместимый)
my_func() {
    echo "Функция вызвана"
}
```
##### Параметры функции
Внутри функции $1, $2 и т.д. — это её аргументы, а не аргументы сценария.
```bash
greet() {
    local NAME="$1"
    local GREETING="${2:-Здравствуйте}"   # Значение по умолчанию
    echo "$GREETING, $NAME!"
}

greet "Иван"
greet "Мария" "Привет"
Возврат значений
return N — возвращает код завершения (0–255).
```
##### echo — для возврата произвольного значения через stdout.

```bash
is_even() {
    local NUM=$1
    if (( NUM % 2 == 0 )); then
        return 0        # Успех (чётное)
    else
        return 1        # Неудача (нечётное)
    fi
}

# Использование
if is_even 10; then
    echo "Число чётное"
fi

# Возврат значения через echo
get_date() {
    echo "$(date +%Y-%m-%d)"
}
TODAY=$(get_date)
echo "Сегодня: $TODAY"
```
##### Рекурсия
```bash
factorial() {
    if [ $1 -le 1 ]; then
        echo 1
    else
        local PREV=$(factorial $(($1 - 1)))
        echo $(($1 * PREV))
    fi
}

echo "5! = $(factorial 5)"    # 120
```
> ⚠️ Обработка ошибок и завершение

##### Коды возврата
Каждая команда возвращает код завершения: 0 — успех, 1–255 — ошибка.
```bash
ls /etc/passwd
echo "Код возврата: $?"

ls /nonexistent
echo "Код возврата: $?"    # Будет ненулевым
```
##### Проверка успешности команды
```bash
# Способ 1: через $?
cp file1 file2
if [ $? -ne 0 ]; then
    echo "Ошибка копирования" >&2
    exit 1
fi
# Способ 2: через && и ||
mkdir /tmp/test && echo "Каталог создан" || echo "Ошибка создания"
```
##### Команда set и её опции
```bash
#!/bin/bash
set -e          # Прервать сценарий при первой ошибке
set -u          # Ошибка при использовании неинициализированной переменной
set -o pipefail # Ошибка в конвейере возвращает код ошибки
set -x          # Отладочный режим (печатать команды)

# Комбинированная форма (рекомендуется для серьёзных сценариев)
set -euo pipefail
```
##### Обработка сигналов: trap
```bash
#!/bin/bash
# Обработка выхода и сигналов
cleanup() {
    echo "Очистка временных файлов..."
    rm -f /tmp/mytemp.*
}

trap cleanup EXIT
trap 'echo "Прервано пользователем"; exit 130' INT TERM

# Основной код
touch /tmp/mytemp.$$
echo "Работа завершена"
```
##### Завершение сценария: exit
```bash
# Нормальное завершение
exit 0

# Завершение с ошибкой
echo "Ошибка: файл не найден" >&2
exit 1

# Использование кодов
# 0   — успех
# 1   — общая ошибка
# 2   — неправильное использование команды
# 126 — команда не исполняема
# 127 — команда не найдена
# 130 — прервано Ctrl+C (128 + SIGINT=2)
```
##### Вывод сообщений об ошибках
```bash
# stderr вместо stdout
echo "Ошибка: недостаточно прав" >&2
```
#### 💻 Практическая работа: простые сценарии

##### Сценарий 1: Приветствие пользователя
```bash
#!/bin/bash
# hello.sh — выводит приветствие

set -euo pipefail

read -p "Введите ваше имя: " NAME
read -p "Введите ваш возраст: " AGE

if [ -z "$NAME" ]; then
    echo "Ошибка: имя не может быть пустым" >&2
    exit 1
fi

echo "Здравствуйте, $NAME!"
echo "Вам $AGE лет."
echo "Сегодня $(date '+%d.%m.%Y %H:%M')"
```
##### Сценарий 2: Проверка существования файла
```bash
#!/bin/bash
# check_file.sh — проверяет существование файла

set -euo pipefail

if [ $# -ne 1 ]; then
    echo "Использование: $0 <путь_к_файлу>" >&2
    exit 1
fi

FILE="$1"

if [ ! -e "$FILE" ]; then
    echo "Файл '$FILE' не существует" >&2
    exit 1
elif [ -d "$FILE" ]; then
    echo "'$FILE' — это каталог"
    echo "Содержимое:"
    ls -la "$FILE"
elif [ -f "$FILE" ]; then
    echo "'$FILE' — это файл"
    echo "Размер: $(stat -c%s "$FILE") байт"
    echo "Права: $(stat -c%A "$FILE")"
fi
```
##### Сценарий 3: Счётчик с циклом
```bash
#!/bin/bash
# countdown.sh — обратный отсчёт

set -euo pipefail

read -p "Введите число: " NUM

if ! [[ "$NUM" =~ ^[0-9]+$ ]]; then
    echo "Ошибка: нужно положительное целое число" >&2
    exit 1
fi

while [ "$NUM" -gt 0 ]; do
    echo "$NUM"
    sleep 0.5
    ((NUM--))
done
echo "Поехали!"
```
##### Сценарий 4: Меню с case
```bash
#!/bin/bash
# menu.sh — интерактивное меню

set -euo pipefail

while true; do
    echo "=== Меню ==="
    echo "1. Показать дату"
    echo "2. Показать пользователей"
    echo "3. Показать дисковое пространство"
    echo "0. Выход"
    read -p "Выбор: " CHOICE

    case "$CHOICE" in
        1) date ;;
        2) who ;;
        3) df -h ;;
        0) echo "До свидания!"; exit 0 ;;
        *) echo "Неверный выбор" >&2 ;;
    esac
    echo
done
```
#### 💻 Практическая работа: сложные сценарии

##### Сценарий 1: Резервное копирование каталога
```bash
#!/bin/bash
# backup.sh — резервное копирование каталога с проверками

set -euo pipefail

# --- Конфигурация ---
SOURCE_DIR="${1:-/home}"
BACKUP_DIR="${2:-/var/backups/manual}"
RETENTION_DAYS=7
LOG_FILE="/var/log/backup_$(date +%Y%m%d).log"

# --- Функции ---
log() {
    local LEVEL="$1"; shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$LEVEL] $*" | tee -a "$LOG_FILE"
}

cleanup() {
    log INFO "Завершение работы сценария"
    if [ -n "${TEMP_ARCHIVE:-}" ] && [ -f "$TEMP_ARCHIVE" ]; then
        rm -f "$TEMP_ARCHIVE"
    fi
}

error_exit() {
    log ERROR "$1"
    exit 1
}

# --- Обработчики ---
trap cleanup EXIT
trap 'error_exit "Прервано пользователем"' INT TERM

# --- Проверки ---
[ -d "$SOURCE_DIR" ] || error_exit "Исходный каталог не найден: $SOURCE_DIR"
[ -r "$SOURCE_DIR" ] || error_exit "Нет прав на чтение: $SOURCE_DIR"

mkdir -p "$BACKUP_DIR" || error_exit "Не удалось создать $BACKUP_DIR"

# --- Основная логика ---
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
ARCHIVE_NAME="backup_${TIMESTAMP}.tar.gz"
TEMP_ARCHIVE="${BACKUP_DIR}/${ARCHIVE_NAME}"

log INFO "Начало резервного копирования: $SOURCE_DIR → $TEMP_ARCHIVE"

if tar -czf "$TEMP_ARCHIVE" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")" 2>>"$LOG_FILE"; then
    SIZE=$(du -h "$TEMP_ARCHIVE" | cut -f1)
    log INFO "Архив создан успешно. Размер: $SIZE"
else
    error_exit "Ошибка создания архива"
fi

# --- Удаление старых архивов ---
log INFO "Удаление архивов старше $RETENTION_DAYS дней..."
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete 2>>"$LOG_FILE" || true

log INFO "Резервное копирование завершено успешно"
exit 0
```
##### Сценарий 2: Мониторинг ресурсов
```bash
#!/bin/bash
# monitor.sh — контроль ресурсов системы с уведомлениями

set -uo pipefail

# --- Пороги ---
CPU_THRESHOLD=80
MEM_THRESHOLD=80
DISK_THRESHOLD=85
CHECK_INTERVAL=60
ALERT_EMAIL="admin@example.com"

# --- Функции ---
notify() {
    local SUBJECT="$1"
    local MESSAGE="$2"
    echo "[ALERT] $SUBJECT: $MESSAGE"
    # Отправка письма (требуется настроенный MTA)
    # echo "$MESSAGE" | mail -s "$SUBJECT" "$ALERT_EMAIL"
}

check_cpu() {
    local CPU_USAGE
    CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print int($2 + $4)}')
    if [ "$CPU_USAGE" -gt "$CPU_THRESHOLD" ]; then
        notify "Высокая загрузка CPU" "${CPU_USAGE}% (порог: ${CPU_THRESHOLD}%)"
    fi
    echo "CPU: ${CPU_USAGE}%"
}

check_memory() {
    local MEM_USAGE
    MEM_USAGE=$(free | awk '/Mem:/ {printf "%.0f", $3/$2 * 100}')
    if [ "$MEM_USAGE" -gt "$MEM_THRESHOLD" ]; then
        notify "Высокое потребление RAM" "${MEM_USAGE}% (порог: ${MEM_THRESHOLD}%)"
    fi
    echo "RAM: ${MEM_USAGE}%"
}

check_disk() {
    while read -r USAGE MOUNT; do
        USAGE_NUM="${USAGE%\%}"
        if [ "$USAGE_NUM" -gt "$DISK_THRESHOLD" ]; then
            notify "Мало места на диске" "$MOUNT: $USAGE (порог: ${DISK_THRESHOLD}%)"
        fi
        echo "DISK $MOUNT: $USAGE"
    done < <(df -h --output=pcent,target | tail -n +2 | grep -vE '/run|/sys|/proc')
}

# --- Основной цикл ---
echo "=== Мониторинг запущен (интервал ${CHECK_INTERVAL}s) ==="
while true; do
    echo "--- $(date '+%Y-%m-%d %H:%M:%S') ---"
    check_cpu
    check_memory
    check_disk
    echo
    sleep "$CHECK_INTERVAL"
done
```
##### Сценарий 3: Управление пользователями
```bash
#!/bin/bash
# user_mgmt.sh — пакетное создание пользователей из файла

set -euo pipefail

USERS_FILE="${1:-}"
LOG_FILE="/var/log/user_mgmt.log"
GROUP="users"

usage() {
    echo "Использование: $0 <файл_с_пользователями>" >&2
    echo "Формат файла: имя:полное_имя:пароль" >&2
    exit 1
}

log() {
    echo "[$(date '+%F %T')] $*" | tee -a "$LOG_FILE"
}

[ -n "$USERS_FILE" ] || usage
[ -f "$USERS_FILE" ] || { echo "Файл не найден: $USERS_FILE" >&2; exit 1; }
[ "$EUID" -eq 0 ] || { echo "Запустите сценарий с правами root" >&2; exit 1; }

# Создание группы, если нет
getent group "$GROUP" >/dev/null || groupadd "$GROUP"

CREATED=0
FAILED=0

while IFS=':' read -r USERNAME FULLNAME PASSWORD; do
    # Пропуск пустых строк и комментариев
    [[ -z "$USERNAME" || "$USERNAME" =~ ^# ]] && continue

    if id "$USERNAME" &>/dev/null; then
        log "Пользователь $USERNAME уже существует — пропуск"
        ((FAILED++))
        continue
    fi

    if useradd -m -G "$GROUP" -c "$FULLNAME" -s /bin/bash "$USERNAME"; then
        echo "${USERNAME}:${PASSWORD}" | chpasswd
        log "Создан пользователь: $USERNAME ($FULLNAME)"
        ((CREATED++))
    else
        log "Ошибка создания пользователя: $USERNAME"
        ((FAILED++))
    fi
done < "$USERS_FILE"

log "Итог: создано $CREATED, ошибок $FAILED"
```
Пример файла users.txt:

```text
ivanov:Иван Иванов:Passw0rd1
petrov:Пётр Петров:Passw0rd2
# sidorov:Сидоров (закомментирован)
```
##### Сценарий 4: Проверка и перезапуск службы
```bash
#!/bin/bash
# service_watchdog.sh — следит за состоянием службы и перезапускает её

set -uo pipefail

SERVICE="${1:-ssh}"
CHECK_INTERVAL=30
MAX_RESTARTS=5
LOG_FILE="/var/log/watchdog_${SERVICE}.log"

log() {
    echo "[$(date '+%F %T')] $*" | tee -a "$LOG_FILE"
}

restart_count=0

log "Запуск наблюдения за службой: $SERVICE"

while true; do
    if systemctl is-active --quiet "$SERVICE"; then
        # Служба работает — сброс счётчика
        [ "$restart_count" -gt 0 ] && log "Служба восстановлена"
        restart_count=0
    else
        log "Служба $SERVICE не работает!"
        if [ "$restart_count" -lt "$MAX_RESTARTS" ]; then
            log "Попытка перезапуска ($((restart_count + 1))/$MAX_RESTARTS)"
            systemctl restart "$SERVICE" && sleep 5
            ((restart_count++))
        else
            log "Превышено число перезапусков. Требуется вмешательство администратора."
            exit 2
        fi
    fi
    sleep "$CHECK_INTERVAL"
done
```
##### Сценарий 5: Разбор логов с отчётом
```bash
#!/bin/bash
# log_report.sh — отчёт по ошибкам в системном журнале

set -euo pipefail

SINCE="${1:-24 hours ago}"
OUTPUT="/tmp/log_report_$(date +%Y%m%d_%H%M%S).txt"

{
    echo "======================================"
    echo "  Отчёт по журналу"
    echo "  Период: с $SINCE"
    echo "  Сформирован: $(date '+%F %T')"
    echo "  Хост: $(hostname)"
    echo "======================================"
    echo

    echo "--- Топ-10 ошибок (по сообщениям) ---"
    journalctl --since "$SINCE" -p err --no-pager 2>/dev/null \
        | awk '{$1=$2=$3=""; print}' \
        | sort | uniq -c | sort -rn | head -10
    echo

    echo "--- Сообщения по службам ---"
    journalctl --since "$SINCE" --no-pager 2>/dev/null \
        | awk '{print $5}' | sort | uniq -c | sort -rn | head -10
    echo

    echo "--- Предупреждения ядра ---"
    journalctl -k --since "$SINCE" -p warning --no-pager 2>/dev/null | tail -20
    echo

    echo "--- Критические ошибки ---"
    journalctl --since "$SINCE" -p crit --no-pager 2>/dev/null
} > "$OUTPUT" 2>&1

echo "Отчёт сохранён в: $OUTPUT"
echo "Строк: $(wc -l < "$OUTPUT")"
```
#### 🛡️ Особенности Astra Linux SE
Мандатный контроль целостности (МКЦ): В Special Edition с включённым МКЦ выполнение сценариев, изменяющих систему (управление пользователями, службами, настройками безопасности), требует высокого уровня целостности. Обычный пользовательский сценарий, запущенный с низким уровнем, может не получить доступ к защищённым ресурсам.

Проверка целостности скриптов: В защищённых конфигурациях можно использовать средства контроля целостности (например, подсистему Parsec или утилиты gostcrypt/afick) для проверки, что сценарии администрирования не были изменены.

Специфические пути: В Astra Linux есть собственные утилиты (fly-admin-*, astra-*), которые часто вызываются из сценариев для автоматизации. Например: fly-admin-repo, astra-update, astra-systemsettings.

Логирование действий администратора: В SE рекомендуется записывать действия сценариев в системный журнал (logger или journalctl), чтобы они попадали в аудит.

Шелл по умолчанию: В Astra Linux bash является оболочкой по умолчанию как для интерактивных сессий, так и для скриптов.

#### 💎 Сводка ключевых практик

Практика            | Зачем                  
------------------- | ---------------------- 
#!/bin/bash в первой строке	 | Явное указание интерпретатора.
set -euo pipefail	 | Раннее обнаружение ошибок.
trap для очистки	 | Гарантированное освобождение ресурсов.
local в функциях	 | Избежать побочных эффектов.
"$VAR" в кавычках	 | Защита от пробелов и глоббинга.
Проверка $# и $EUID	 | Корректная обработка аргументов и прав.
Логирование в файл и journalctl	 | Возможность аудита и отладки.
