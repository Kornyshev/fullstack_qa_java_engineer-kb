# Базовые команды Linux для QA

## Обзор

Linux — основная операционная система для серверов, на которых работают тестируемые приложения.
По статистике, более 90% серверов в мире работают на Linux. Для QA-инженера умение работать
с Linux через командную строку — это не опциональный навык, а ежедневная необходимость.
Анализ логов на сервере, проверка состояния процессов, сетевая диагностика, подготовка тестовых
данных — всё это делается через терминал. В этом разделе — практически ориентированный набор
команд, который реально используется в работе QA.

---

## Навигация по файловой системе

### Основные команды

```bash
# pwd — показать текущую директорию (Print Working Directory)
pwd
# /home/qa-engineer

# ls — список файлов и директорий
ls                   # Простой список
ls -l                # Подробный список (права, размер, дата)
ls -la               # Включая скрытые файлы (начинаются с .)
ls -lh               # Человекочитаемый размер (1K, 2M, 3G)
ls -lt               # Сортировка по времени изменения (новые сверху)
ls -lS               # Сортировка по размеру (большие сверху)

# cd — смена директории (Change Directory)
cd /var/log          # Переход по абсолютному пути
cd logs              # Переход в поддиректорию (относительный путь)
cd ..                # Подняться на уровень вверх
cd ~                 # Домашняя директория текущего пользователя
cd -                 # Вернуться в предыдущую директорию

# tree — дерево директорий (может потребоваться установка)
tree -L 2            # Дерево на 2 уровня вглубь
tree -d              # Только директории
```

### Структура файловой системы Linux

```
/
├── bin/          # Основные команды (ls, cp, mv)
├── etc/          # Конфигурационные файлы
├── home/         # Домашние директории пользователей
├── var/
│   ├── log/      # Системные логи ← QA часто работает здесь
│   └── www/      # Файлы веб-сервера
├── tmp/          # Временные файлы
├── opt/          # Дополнительное ПО
└── usr/          # Пользовательские программы
```

---

## Операции с файлами и директориями

```bash
# === Создание ===
mkdir logs                        # Создать директорию
mkdir -p project/src/test/java    # Создать вложенные директории
touch test-data.txt               # Создать пустой файл

# === Копирование ===
cp file.txt backup.txt            # Копировать файл
cp -r logs/ logs-backup/          # Копировать директорию рекурсивно

# === Перемещение / Переименование ===
mv old-name.txt new-name.txt      # Переименовать файл
mv report.txt /home/qa/reports/   # Переместить файл

# === Удаление ===
rm file.txt                       # Удалить файл
rm -r directory/                  # Удалить директорию рекурсивно
rm -rf directory/                 # Удалить без подтверждения (ОСТОРОЖНО!)
# Никогда не запускайте rm -rf / — удалит ВСЮ файловую систему

# === Просмотр содержимого файлов ===
cat file.txt                      # Вывести весь файл
less file.txt                     # Постраничный просмотр (q для выхода)
head file.txt                     # Первые 10 строк
head -n 50 file.txt               # Первые 50 строк
tail file.txt                     # Последние 10 строк
tail -n 100 file.txt              # Последние 100 строк
tail -f app.log                   # Следить за файлом в реальном времени (follow)

# === Размер файлов и директорий ===
du -sh directory/                 # Размер директории
du -sh *                          # Размер каждого элемента в текущей директории
df -h                             # Свободное место на дисках

# === Поиск файлов ===
find /var/log -name "*.log"                    # Найти файлы по имени
find . -name "*.xml" -mtime -1                 # Файлы XML, изменённые за последний день
find . -type f -size +100M                     # Файлы больше 100 МБ
find /tmp -name "test_*" -delete               # Найти и удалить
```

---

## Обработка текста

### grep — поиск по содержимому

```bash
# Базовый поиск
grep "ERROR" app.log                           # Найти строки с "ERROR"
grep -i "error" app.log                        # Без учёта регистра
grep -n "ERROR" app.log                        # С номерами строк
grep -c "ERROR" app.log                        # Количество совпадений
grep -r "password" /etc/                       # Рекурсивный поиск в директории

# Контекст вокруг совпадения
grep -A 5 "Exception" app.log                  # 5 строк ПОСЛЕ совпадения (After)
grep -B 3 "Exception" app.log                  # 3 строки ДО совпадения (Before)
grep -C 3 "Exception" app.log                  # 3 строки до и после (Context)

# Регулярные выражения
grep -E "ERROR|WARN" app.log                   # ИЛИ: ERROR или WARN
grep -E "^2025-01-15" app.log                  # Строки, начинающиеся с даты
grep -E "status=(4|5)[0-9]{2}" access.log      # HTTP-ошибки (4xx, 5xx)

# Инвертированный поиск
grep -v "DEBUG" app.log                        # Все строки КРОМЕ DEBUG

# Комбинация: найти ошибки, исключив типовые
grep "ERROR" app.log | grep -v "HealthCheck"
```

### awk — обработка структурированного текста

```bash
# awk работает с разделёнными полями (по умолчанию — пробелы)

# Вывести определённые колонки
awk '{print $1, $4}' access.log                # 1-я и 4-я колонки
awk -F',' '{print $2}' data.csv                # 2-я колонка CSV (разделитель — запятая)

# Фильтрация по условию
awk '$9 >= 500' access.log                     # Строки, где 9-я колонка >= 500 (HTTP 5xx)
awk '$9 == 404' access.log                     # Только 404 ошибки

# Подсчёт
awk '{count[$9]++} END {for (c in count) print c, count[c]}' access.log
# Подсчитывает количество запросов по HTTP status code

# Вычисление среднего времени ответа
awk '{sum += $NF; count++} END {print sum/count}' access.log
```

### sed — потоковый редактор

```bash
# Замена текста
sed 's/old/new/' file.txt                      # Заменить первое вхождение в каждой строке
sed 's/old/new/g' file.txt                     # Заменить все вхождения
sed -i 's/old/new/g' file.txt                  # Заменить прямо в файле (-i = in-place)

# Удаление строк
sed '/DEBUG/d' app.log                         # Удалить строки с DEBUG
sed '1,10d' file.txt                           # Удалить строки 1-10

# Вывод определённых строк
sed -n '100,200p' app.log                      # Строки 100-200
```

### wc — подсчёт

```bash
wc -l file.txt                                 # Количество строк
wc -w file.txt                                 # Количество слов
wc -c file.txt                                 # Количество байт

# Практические примеры
grep "ERROR" app.log | wc -l                   # Количество ошибок
cat access.log | wc -l                         # Всего запросов
```

### Пайплайны (конвейеры)

Мощь Linux — в комбинировании простых команд через pipe (`|`):

```bash
# Топ-10 IP-адресов по количеству запросов
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Уникальные ошибки (дедупликация)
grep "ERROR" app.log | awk '{$1=$2=$3=""; print $0}' | sort -u

# Количество запросов по status code
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Найти самые частые exceptions
grep "Exception" app.log | awk -F': ' '{print $2}' | sort | uniq -c | sort -rn | head -10
```

---

## Управление процессами

```bash
# === Просмотр процессов ===
ps aux                             # Все процессы в системе
ps aux | grep java                 # Процессы Java
ps -ef --forest                    # Дерево процессов (родитель-потомок)

# === Мониторинг в реальном времени ===
top                                # Интерактивный монитор (q для выхода)
htop                               # Улучшенный top (может потребоваться установка)

# Внутри top: полезные клавиши
#   M — сортировка по памяти
#   P — сортировка по CPU
#   1 — показать все ядра CPU

# === Управление процессами ===
kill <PID>                         # Корректное завершение (SIGTERM)
kill -9 <PID>                      # Принудительное завершение (SIGKILL)
killall java                       # Завершить все процессы с именем "java"

# === Фоновые процессы ===
command &                          # Запуск в фоне
nohup command &                    # Запуск в фоне (не завершится при закрытии терминала)
jobs                               # Список фоновых задач
fg %1                              # Вернуть задачу 1 на передний план

# === Потребление ресурсов ===
free -h                            # Свободная память (human-readable)
uptime                             # Время работы системы и средняя нагрузка
```

---

## Права доступа

### Чтение прав

```
-rwxr-xr-- 1 qa-user qa-group 4096 Jan 15 10:30 script.sh
│└┬─┘└┬─┘└┬─┘
│  │    │    │
│  │    │    └── Другие (other): read (r--)
│  │    └─────── Группа (group): read + execute (r-x)
│  └──────────── Владелец (owner): read + write + execute (rwx)
└─────────────── Тип: - файл, d директория, l ссылка
```

### Изменение прав

```bash
# chmod — изменение прав доступа
chmod +x script.sh                 # Добавить право на выполнение
chmod 755 script.sh                # rwxr-xr-x (владелец: всё, остальные: чтение+выполнение)
chmod 644 config.txt               # rw-r--r-- (владелец: чтение+запись, остальные: чтение)
chmod -R 755 directory/            # Рекурсивно для директории

# Числовые значения:
# 4 = read (r)
# 2 = write (w)
# 1 = execute (x)
# 7 = rwx, 6 = rw-, 5 = r-x, 4 = r--, 0 = ---

# chown — изменение владельца
chown qa-user:qa-group file.txt    # Изменить владельца и группу
chown -R qa-user:qa-group dir/     # Рекурсивно
```

---

## Сетевые утилиты

### curl — HTTP-клиент

```bash
# GET-запрос
curl http://localhost:8080/api/health

# Подробный вывод (заголовки запроса и ответа)
curl -v http://localhost:8080/api/users

# Только заголовки ответа
curl -I http://localhost:8080/api/users

# POST-запрос с JSON
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer token123" \
  -d '{"name": "Тест", "email": "test@example.com"}'

# Загрузка файла
curl -o report.pdf http://example.com/report.pdf

# Таймаут
curl --connect-timeout 5 --max-time 30 http://slow-service:8080/api

# Следовать за редиректами
curl -L http://example.com/redirect

# Отправка формы
curl -X POST http://localhost:8080/upload \
  -F "file=@test-data.csv" \
  -F "description=Тестовые данные"

# Показать время ответа
curl -o /dev/null -s -w "HTTP Code: %{http_code}\nTime: %{time_total}s\n" \
  http://localhost:8080/api/health
```

### wget — загрузка файлов

```bash
# Скачать файл
wget http://example.com/file.zip

# Скачать в определённую директорию
wget -P /tmp/ http://example.com/file.zip

# Скачать с переименованием
wget -O my-file.zip http://example.com/file.zip
```

### Сетевая диагностика

```bash
# ping — проверка доступности хоста
ping -c 4 google.com              # 4 пакета (без -c будет бесконечно)

# nslookup / dig — DNS-запросы
nslookup example.com              # Получить IP по домену
dig example.com                   # Детальный DNS-запрос

# netstat — сетевые соединения и порты
netstat -tlnp                     # TCP-порты в состоянии LISTEN
# -t = TCP, -l = LISTEN, -n = числовые адреса, -p = процесс

# ss — современная замена netstat
ss -tlnp                          # TCP-порты в состоянии LISTEN
ss -tnp | grep 8080               # Кто слушает порт 8080

# traceroute — маршрут пакета до хоста
traceroute example.com

# telnet / nc — проверка доступности порта
nc -zv hostname 8080               # Проверить, открыт ли порт
# -z = только проверка (без отправки данных), -v = verbose

# Проверка, что приложение запустилось
while ! nc -z localhost 8080; do
    echo "Ждём запуск приложения на порту 8080..."
    sleep 2
done
echo "Приложение доступно!"
```

---

## Удалённый доступ

### SSH

```bash
# Подключение к серверу
ssh qa-user@server.example.com
ssh -p 2222 qa-user@server.example.com         # Нестандартный порт

# Подключение с ключом
ssh -i ~/.ssh/id_rsa qa-user@server.example.com

# Выполнить команду на удалённом сервере (без интерактивной сессии)
ssh qa-user@server.example.com "tail -100 /var/log/app/app.log"

# SSH-туннель (проброс порта)
ssh -L 5432:db-server:5432 qa-user@jump-server.com
# Локальный порт 5432 → через jump-server → к db-server:5432
# Теперь можно подключиться к БД через localhost:5432
```

### SCP — копирование файлов через SSH

```bash
# Скопировать файл на сервер
scp test-data.csv qa-user@server:/tmp/

# Скопировать файл с сервера
scp qa-user@server:/var/log/app/app.log ./app.log

# Скопировать директорию
scp -r test-data/ qa-user@server:/tmp/test-data/
```

---

## Анализ логов: практические сценарии

### Сценарий 1: Поиск ошибок после деплоя

```bash
# Подключаемся к серверу
ssh qa-user@prod-server.example.com

# Смотрим последние 200 строк лога
tail -200 /var/log/app/application.log

# Следим за логом в реальном времени (после деплоя)
tail -f /var/log/app/application.log

# Ищем ошибки за сегодня
grep "2025-01-15" /var/log/app/application.log | grep "ERROR"

# Ищем конкретный stack trace
grep -A 20 "NullPointerException" /var/log/app/application.log

# Количество ошибок по типам
grep "ERROR" /var/log/app/application.log | \
  awk -F'Exception' '{print $1"Exception"}' | \
  sort | uniq -c | sort -rn
```

### Сценарий 2: Анализ access-лога

```bash
# Формат access-лога (типичный для Nginx):
# 192.168.1.1 - - [15/Jan/2025:10:30:00 +0300] "GET /api/users HTTP/1.1" 200 1234

# Топ-10 самых частых URL
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Все запросы с ошибкой 500
awk '$9 == 500' /var/log/nginx/access.log

# Количество запросов по статусам
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Запросы, которые длились более 5 секунд (если время в последней колонке)
awk '$NF > 5.0' /var/log/nginx/access.log

# Количество запросов в минуту (для анализа нагрузки)
awk '{print $4}' /var/log/nginx/access.log | \
  cut -d: -f1-3 | sort | uniq -c | sort -rn | head -20
```

### Сценарий 3: Мониторинг во время нагрузочного тестирования

```bash
# В первом терминале: следим за логами ошибок
tail -f /var/log/app/application.log | grep --line-buffered "ERROR"

# Во втором терминале: мониторим ресурсы
top -b -d 5 | grep java          # Обновлять каждые 5 секунд

# В третьем терминале: смотрим количество подключений
watch -n 5 'ss -tn | grep 8080 | wc -l'

# Проверяем, не заканчивается ли дисковое пространство
watch -n 10 'df -h | grep -E "/$|/var"'
```

### Сценарий 4: Подготовка тестовых данных

```bash
# Генерация файла с тестовыми данными
for i in $(seq 1 1000); do
    echo "user_${i},user_${i}@test.com,password_${i}" >> test-users.csv
done

# Извлечь email-адреса из CSV
awk -F',' '{print $2}' test-users.csv

# Заменить домен в тестовых данных
sed 's/@old-domain.com/@new-domain.com/g' test-users.csv > updated-users.csv

# Подсчитать количество записей
wc -l test-users.csv

# Проверить уникальность email-адресов
awk -F',' '{print $2}' test-users.csv | sort | uniq -d
# Если вывод пустой — все email уникальны
```

---

## Связь с тестированием

Навыки работы с Linux позволяют QA-инженеру:

1. **Анализировать логи** — быстро находить ошибки на серверах
2. **Диагностировать проблемы** — проверять доступность сервисов, порты, DNS
3. **Подготавливать тестовые данные** — скрипты для генерации и обработки данных
4. **Работать с CI/CD** — pipeline'ы исполняются в Linux-средах
5. **Тестировать API** — curl для быстрых проверок без Postman
6. **Мониторить нагрузочные тесты** — ресурсы сервера в реальном времени
7. **Настраивать тестовые среды** — Docker, Kubernetes работают на Linux

---

## Типичные ошибки

1. **`rm -rf` без проверки** — удаление не тех файлов (всегда используйте `ls` перед `rm`)
2. **Не используют `grep` с контекстом** — видят строку ошибки, но не видят stack trace (-A/-B)
3. **`cat` для больших файлов** — используйте `less`, `tail`, `head` вместо `cat` для больших логов
4. **Игнорируют `tail -f`** — самый полезный инструмент для отладки в реальном времени
5. **Не используют пайплайны** — пишут длинные команды вместо комбинации простых
6. **Забывают про права доступа** — скрипт не запускается без `chmod +x`
7. **Жёсткие пути в скриптах** — не используют переменные и относительные пути
8. **Не используют `screen`/`tmux`** — длинные задачи прерываются при разрыве SSH

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Как посмотреть последние 100 строк файла?
2. Как найти все строки с текстом "ERROR" в файле?
3. Как проверить, запущен ли процесс Java?
4. Как проверить, доступен ли порт 8080?
5. Как скопировать файл с сервера на локальную машину?

### 🟡 Средний уровень
6. Как следить за логами в реальном времени, фильтруя только ошибки?
7. Как найти топ-10 самых частых ошибок в логе?
8. Как узнать, какой процесс занимает порт 8080?
9. В чём разница между `grep`, `awk` и `sed`?
10. Как создать SSH-туннель для доступа к удалённой базе данных?
11. Как проверить свободное место на диске?

### 🔴 Продвинутый уровень
12. Напишите однострочник для анализа access-лога: подсчёт запросов по HTTP status code.
13. Как организовать мониторинг ресурсов сервера во время нагрузочного тестирования?
14. Как написать скрипт, который проверяет доступность нескольких сервисов и отправляет алерт?
15. Как использовать `awk` для вычисления среднего времени ответа из access-лога?

---

## Практические задания

### Задание 1: Анализ логов
Скачайте или создайте файл с логами приложения. Выполните:
1. Найдите все ERROR-строки
2. Подсчитайте количество ошибок по типам (Exception)
3. Выведите строки за определённый период времени
4. Найдите stack trace конкретной ошибки с контекстом

### Задание 2: Сетевая диагностика
1. Проверьте доступность google.com (ping)
2. Узнайте IP-адрес по доменному имени (nslookup)
3. Проверьте, какие порты открыты на localhost (ss)
4. Отправьте HTTP-запрос с помощью curl и проанализируйте ответ

### Задание 3: Скрипт для проверки здоровья сервисов
Напишите bash-скрипт, который:
1. Проверяет доступность 3 URL (curl)
2. Выводит статус каждого (UP/DOWN)
3. Логирует результат в файл с timestamp

```bash
#!/bin/bash
# health-check.sh
SERVICES=("http://localhost:8080/health" "http://localhost:8081/health" "http://localhost:8082/health")
LOG_FILE="health-check.log"

for url in "${SERVICES[@]}"; do
    status=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$url")
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    if [ "$status" -eq 200 ]; then
        echo "$timestamp | $url | UP ($status)" | tee -a "$LOG_FILE"
    else
        echo "$timestamp | $url | DOWN ($status)" | tee -a "$LOG_FILE"
    fi
done
```

### Задание 4: Анализ access-лога
Создайте тестовый access-лог и выполните:
1. Подсчитайте общее количество запросов
2. Найдите топ-5 IP-адресов по количеству запросов
3. Подсчитайте запросы по HTTP-методам (GET, POST, PUT, DELETE)
4. Найдите все запросы со статусом 500

---

## Дополнительные ресурсы

- [Linux Command Line Basics (Coursera)](https://www.coursera.org/learn/unix)
- [The Linux Command Line (книга, бесплатная)](https://linuxcommand.org/tlcl.php)
- [Bash Scripting Tutorial](https://www.shellscript.sh/)
- [explainshell.com — объяснение команд](https://explainshell.com/)
- [Linux Journey — интерактивное обучение](https://linuxjourney.com/)
- [tldr pages — упрощённые man pages](https://tldr.sh/)
