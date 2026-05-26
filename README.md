# Momo Store

Go backend + Vue frontend в Docker Compose. 

Frontend собирается на Node и раздаётся через nginx; 

backend - статический Go-бинарь в Alpine. 

Перед backend стоит nginx-балансировщик `lb` для нескольких реплик.

## Запуск

```bash
docker compose up -d --build
```

- Frontend: http://localhost/
- API: http://localhost:8081/health

Остановка - `docker compose down`.

## Production (образы с Docker Hub)

`prod`-override берёт готовые образы вместо локальной сборки (тег - `IMAGE_TAG`, по умолчанию `latest`):

```bash
DOCKER_USER=myuser docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Масштабирование backend

```bash
docker compose up -d --build --scale backend=3
```

Запросы к API идут через `lb`, который через Docker DNS и переменную в `proxy_pass` динамически находит новые реплики:

```text
:80   ──► frontend (статика Vue)
:8081 ──► lb ──► backend-1 … backend-N
```

## Образы и размеры

Все образы - multi-stage (build-стадия в финал не попадает).

| Сервис   | Build                | Runtime                                   | Размер |
|----------|----------------------|-------------------------------------------|--------|
| backend  | `golang:1.25-alpine` | `alpine:3.22`                             | ~17 MB |
| frontend | `node:16-alpine`     | `nginxinc/nginx-unprivileged:1.27-alpine` | ~50 MB |
| lb       | -                    | `nginxinc/nginx-unprivileged:1.27-alpine` | ~20 MB |

- `alpine:3.22` - минимальный runtime (~5 MB), без лишних пакетов.
- `nginx-unprivileged` - nginx без root.
- `node:16-alpine` - webpack 4 / vue-cli 4.5 не работают на node 17+.

## Переменные окружения

| Переменная            | Назначение                              | По умолчанию                          |
|-----------------------|-----------------------------------------|---------------------------------------|
| `VUE_APP_API_URL`     | API URL, вшивается в JS-бандл на сборке  | `http://localhost:8081`               |
| `DOCKER_USER`         | Логин Docker Hub (prod)                  | -                                     |
| `IMAGE_TAG`           | Тег prod-образов                         | `latest`                              |

## Безопасность

- backend - от непривилегированного пользователя `app`; 
- nginx-образы - `nginx-unprivileged`
- `cap_drop: ALL`, `no-new-privileges:true`, `read_only: true` на всех сервисах; 
- `tmpfs` для временных каталогов.
- Лимиты и резервации CPU/RAM через `deploy.resources`.
- backend изолирован в сети `momo-net` (`internal: true`) - без выхода в интернет, наружу только через `lb`; внешний доступ - отдельная сеть `edge`.
- Статический бинарь (`CGO_ENABLED=0`), runtime без build-зависимостей.

## Trivy

Сканирование образов в GitHub Actions (`aquasecurity/trivy-action`) после push на Docker Hub: уровни `CRITICAL`/`HIGH`, `ignore-unfixed: true`.
