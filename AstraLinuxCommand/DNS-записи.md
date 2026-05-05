#### Windows: Команда nslookup
Откройте командную строку: Нажмите ``Win + R``, введите cmd и нажмите Enter.

Выполните последовательные запросы для разных типов записей:

**А-запись** (IPv4-адрес): ``nslookup -type=A example.com``

**MX-запись** (почтовые серверы): ``nslookup -type=MX example.com``

**NS-запись** (серверы имен): ``nslookup -type=NS example.com``

**TXT-запись** (текстовые данные, например, для SPF): ``nslookup -type=TXT example.com``

**SOA-запись** (начало зоны): ``nslookup -type=SOA example.com``

**CNAME-запись** (псевдоним): ``nslookup -type=CNAME example.com``

**PTR-запись** (обратный DNS-поиск по IP): ``nslookup 8.8.8.8``
