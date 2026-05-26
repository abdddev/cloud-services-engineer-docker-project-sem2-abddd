# Momo Store

Проект содержит Go backend и Vue frontend, подготовленные к запуску в Docker и Docker Compose

Frontend собирается в отдельном build-stage на Node.js, после чего статика раздаётся через nginx. Backend собирается как статический Go-бинарь и запускается в минимальном Alpine runtime-образе. 

Для балансировки нескольких реплик backend используется отдельный nginx-контейнер `lb`.

Проект рассчитан на локальный запуск, масштабирование backend через Docker Compose и базовую проверку безопасности образов через Trivy в CI.

---

## Запуск

```bash
# Собрать и поднять все сервисы
docker compose up -d --build
```

После запуска:

```text
Frontend: http://localhost/
API:      http://localhost:8081/health
```

Остановка:

```bash
docker compose down
```

Полная пересборка без кэша:

```bash
docker compose build --no-cache
docker compose up -d
```

---

## Запуск со скейлингом backend

Несколько реплик сервиса `backend` поднимаются флагом `--scale`.

```bash
docker compose up -d --build --scale backend=3
```

Запросы к API идут через контейнер `lb`, который проксирует трафик на сервис `backend` внутри Docker-сети.

```text
браузер / клиент
  │
  ├─ :80   ──► frontend nginx ──► статика Vue.js
  │
  └─ :8081 ──► lb nginx
                  ├─► backend-1
                  ├─► backend-2
                  └─► backend-N
```

Docker Compose создаёт несколько контейнеров `backend`, а nginx-балансировщик распределяет запросы между ними по имени сервиса `backend` в сети `momo-net`.

---

## Архитектура

```text
браузер
  │
  ├─ http://localhost/
  │      │
  │      └─► frontend
  │             └─ nginx раздаёт собранную Vue.js-статику
  │
  └─ http://localhost:8081/
         │
         └─► lb
                └─ nginx проксирует запросы к backend
                       ├─► backend-1 Go API
                       ├─► backend-2 Go API
                       └─► backend-N Go API
```

Все сервисы находятся в Docker-сети `momo-net`.

Frontend обращается к API по значению переменной `VUE_APP_API_URL`, которая вшивается в JS-бандл на этапе production-сборки.

---

## Переопределение адреса API

По умолчанию frontend использует API URL из `.env` или значение, заданное при запуске.

Пример:

```bash
VUE_APP_API_URL=http://localhost:8081 docker compose up -d --build
```

Или через `.env`:

```bash
cp .env.example .env
```

Пример `.env`:

```env
VUE_APP_API_URL=http://localhost:8081
```

После изменения `VUE_APP_API_URL` frontend нужно пересобрать, потому что значение встраивается в статический JS-бандл:

```bash
docker compose up -d --build
```

---

## Образы и размеры

Все образы используют multi-stage сборку: build-стадия не попадает в финальный образ.

| Сервис | Build-образ | Runtime-образ | Размер |
|--------|-------------|---------------|--------|
| backend | `golang:1.25-alpine` | `alpine:3.22` | **17 MB** |
| frontend | `node:16-alpine` | `nginxinc/nginx-unprivileged:1.27-alpine` | **50 MB** |
| lb | — | `nginxinc/nginx-unprivileged:1.27-alpine` | — |

---

## Ресурсы контейнеров

В `docker-compose.yml` для сервисов заданы ограничения ресурсов через `deploy.resources`.

### Backend

Backend - небольшой Go-сервис, поэтому его типичное потребление памяти невелико. 

Лимит CPU и памяти нужен, чтобы контейнер не мог бесконтрольно использовать ресурсы хоста при ошибках, росте нагрузки или всплесках одновременных запросов.

### Frontend / nginx

Frontend-контейнер только раздаёт статические файлы через nginx, поэтому его нагрузка обычно низкая.

Для nginx достаточно небольшого CPU-лимита и умеренного лимита памяти

### LB / nginx

`lb` выполняет только проксирование запросов к backend. 

Основные расходы - обработка соединений, буферизация ответов и служебные процессы nginx.


Для production-нагрузки эти значения нужно подбирать по метрикам и результатам нагрузочного тестирования.

---

## Переменные окружения

Основные переменные проекта:

| Переменная | Назначение | Значение по умолчанию   |
|-----------|------------|-------------------------|
| `VUE_APP_API_URL` | API base URL для frontend | `http://localhost:8081` |
| `DOCKER_USER` | Логин Docker Hub для публикации образов | -                       |
| `DOCKER_PASSWORD` | Пароль или token Docker Hub для CI | -                       |

Пример локальной настройки:

```bash
cp .env.example .env
```

Файл `.env` добавлен в `.gitignore`, поэтому локальные значения не попадают в репозиторий.

В репозитории должен храниться только `.env.example` с безопасными значениями по умолчанию.

---

## Безопасность контейнеров

В проекте применяются базовые меры hardening:

- backend запускается от непривилегированного пользователя (`app`);
- `nginxinc/nginx-unprivileged` запускает nginx без root — не требует `CHOWN`, `SETUID`, `SETGID`;
- `cap_drop: ALL` на всех сервисах — capabilities не наследуются;
- `security_opt: no-new-privileges:true` — процессы не могут расширить права через setuid/setgid;
- `read_only: true` + `tmpfs` для временных директорий nginx;
- лимиты CPU и памяти через `deploy.resources.limits`;
- изолированная сеть `momo-net` с `internal: true` — контейнеры не имеют прямого доступа в интернет;
- Go-бинарь собирается статически (`CGO_ENABLED=0`), runtime-образы не содержат build-зависимостей.

---

## Trivy

Сканирование Docker-образов настроено в GitHub Actions через `aquasecurity/trivy-action`.

- Проверяются уязвимости уровня `CRITICAL` и `HIGH`

- `ignore-unfixed: true` — pipeline не блокируется уязвимостями, для которых ещё нет патча

- Сканирование запускается после push образов на Docker Hub

---

## Проверка работы

Healthcheck backend:

```bash
curl http://localhost:8081/health
```

Frontend:

```bash
curl http://localhost/
```

Проверка масштабирования backend:

```bash
docker compose up -d --scale backend=3
docker compose ps
```

После этого в списке контейнеров должно быть несколько экземпляров сервиса `backend`.
