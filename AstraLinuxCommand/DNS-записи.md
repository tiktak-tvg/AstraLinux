#### Windows: Команда nslookup
1. Откройте командную строку: Нажмите ``Win + R``, введите cmd и нажмите Enter.<br>
2. Выполните последовательные запросы для разных типов записей:

  - **А-запись** (IPv4-адрес): ``nslookup -type=A example.com``

  - **MX-запись** (почтовые серверы): ``nslookup -type=MX example.com``

  - **NS-запись** (серверы имен): ``nslookup -type=NS example.com``

  - **TXT-запись** (текстовые данные, например, для SPF): ``nslookup -type=TXT example.com``

  - **SOA-запись** (начало зоны): ``nslookup -type=SOA example.com``

  - **CNAME-запись** (псевдоним): ``nslookup -type=CNAME example.com``

  - **PTR-запись** (обратный DNS-поиск по IP): ``nslookup 8.8.8.8``

#### Linux/macOS: Команды dig и host
Эти системы предоставляют более мощные инструменты.

Команда dig
Откройте терминал: (Ctrl + Alt + T в Linux, поиск "Terminal" в macOS).

Выполните запросы:

dig example.com ANY или dig example.com ANY: Запрашивает ВСЕ доступные записи. Важно: Этот метод не рекомендуется, так как многие DNS-серверы его игнорируют или блокируют. dig example.com ANY +noall +answer может дать более чистый вывод.

Запрос конкретных типов записей:

А-запись: dig example.com A

MX-запись: dig example.com MX

NS-запись: dig example.com NS

TXT-запись: dig example.com TXT

SOA-запись: dig example.com SOA

CNAME-запись: dig example.com CNAME

PTR-запись: dig -x 8.8.8.8

Чтобы получить только значение записи (без технических деталей): добавьте +short. Например: dig example.com A +short.

Команда host
Более простая альтернатива, выдающая лаконичный вывод.

Общий запрос: host example.com (покажет A, AAAA, MX и т.д.).

Запрос конкретного типа: host -t MX example.com
