# Momo Store

Интернет-магазин пельменей: Go API и Vue.js SPA.

## Архитектура

```text
Browser
  |-- :8088 -> frontend (nginx, Vue.js)
  `-- :8081 -> backend (Go API)
```

* `frontend` доступен на порту `8088` хоста и использует порт `80` внутри контейнера.
* `backend` доступен на порту `8081`.
* Frontend обращается к API backend.
* Для backend используется Docker Secret с секретом идентификаторов заказов.
* Backend использует минимальный `scratch`-образ.
* Frontend использует `nginx:alpine`.

## Запуск

Требуются Docker Engine и Docker Compose v2.

Для запуска проекта:

```bash
docker compose up -d --build
```

Проверить состояние контейнеров:

```bash
docker compose ps
```

Оба сервиса должны иметь статус `healthy`.

Адреса:

* фронтенд: http://localhost:8088/momo-store/
* API: http://localhost:8081
* healthcheck API: http://localhost:8081/health

Проверка:

```bash
curl --fail http://localhost:8088/momo-store/
curl --fail http://localhost:8081/health
curl --fail http://localhost:8081/products
docker compose ps
```

Остановка:

```bash
docker compose down
```

Для повторной сборки образов:

```bash
docker compose up -d --build
```

## Docker Compose

Compose содержит два сервиса:

* `frontend`
* `backend`

Для обоих сервисов настроен автоматический перезапуск:

```yaml
restart: unless-stopped
```

Backend имеет healthcheck. Frontend также имеет healthcheck и ожидает готовности backend:

```yaml
depends_on:
  backend:
    condition: service_healthy
```

Для контейнеров установлены ограничения ресурсов.

### Backend

* CPU: `0.5`
* Memory: `256M`

### Frontend

* CPU: `0.5`
* Memory: `128M`

## Конфигурация

Основные параметры приложения задаются через переменные окружения.

### Backend

| Переменная             | Назначение            | Значение по умолчанию          |
| ---------------------- | --------------------- | ------------------------------ |
| `APP_PORT`             | Порт приложения       | `8081`                         |
| `LOG_LEVEL`            | Уровень логирования   | `info`                         |
| `ORDER_DB_PATH`        | Путь к данным заказов | `/data/orders.seq`             |
| `ORDER_ID_SECRET_FILE` | Путь к Docker Secret  | `/run/secrets/order_id_secret` |

### Frontend

Frontend использует переменную:

| Переменная        | Назначение      |
| ----------------- | --------------- |
| `VUE_APP_API_URL` | URL backend API |

Значения frontend-конфигурации передаются во время сборки Vue.js приложения.

## Docker Images

Оба Dockerfile используют multi-stage build.

### Backend

Backend собирается в несколько этапов.

Builder использует:

```text
golang:1.26.8-alpine3.23
```

Production-образ использует:

```text
scratch
```

В итоговый образ переносится только скомпилированный бинарный файл приложения.

В production-образ не попадают:

* Go compiler;
* исходный код;
* package manager;
* shell;
* инструменты сборки;
* build cache.

Backend запускается от непривилегированного пользователя:

```text
UID:GID = 65532:65532
```

Размер итогового образа:

```text
11.8 MB
```

### Frontend

Frontend также использует multi-stage build.

Builder:

```text
node:22-alpine
```

Production:

```text
nginx:alpine
```

На этапе сборки выполняется Vue.js build. В итоговый образ копируются только готовые статические файлы приложения и конфигурация nginx.

Node.js, npm и исходный код не попадают в production-образ.

Размер итогового образа:

```text
64.7 MB
```

Проверить размеры образов:

```bash
docker images momo-backend:1.0
docker images momo-frontend:1.0
```

Фактические размеры после локальной сборки:

| Образ               |  Размер |
| ------------------- | ------: |
| `momo-backend:1.0`  | 11.8 MB |
| `momo-frontend:1.0` | 64.7 MB |

Использование multi-stage build и минимальных базовых образов уменьшает размер production-образов и исключает ненужные инструменты из финального окружения.

## Безопасность

В проекте реализованы следующие меры безопасности:

* backend работает не от `root`;
* backend использует минимальный `scratch`-образ;
* frontend использует Alpine-образ nginx;
* инструменты сборки не попадают в production-образы;
* секреты не встраиваются в Docker images;
* используется Docker Secret;
* файл с секретом исключён из Git;
* для контейнеров установлены ограничения CPU и памяти;
* опубликованы только необходимые для локального запуска порты.

Backend работает от пользователя:

```text
65532:65532
```

Проверить пользователя контейнера:

```bash
docker inspect $(docker compose ps -q backend) \
  --format '{{.Config.User}}'
```

## Docker Secrets

Секрет для backend передаётся через Docker Secrets.

В `docker-compose.yml` используется:

```yaml
secrets:
  order_id_secret:
    file: ./order_id_secret.txt
```

В контейнере секрет доступен по адресу:

```text
/run/secrets/order_id_secret
```

Переменная окружения:

```text
ORDER_ID_SECRET_FILE=/run/secrets/order_id_secret
```

Файл `order_id_secret.txt` имеет ограниченные права доступа и не должен попадать в Git.

Проверить права:

```bash
ls -l order_id_secret.txt
```

Ожидаемые права:

```text
-rw------- 
```

Проверить, что файл исключён из Git:

```bash
git status --short
```

## Healthcheck

Для backend настроен Docker healthcheck.

Проверка выполняется командой:

```text
/api healthcheck
```

Healthcheck обращается к:

```text
http://127.0.0.1:8081/health
```

Для frontend настроен healthcheck nginx:

```text
http://127.0.0.1/momo-store/
```

Конфигурация frontend healthcheck:

```yaml
healthcheck:
  test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1/momo-store/"]
  interval: 30s
  timeout: 3s
  retries: 3
  start_period: 5s
```

Проверить состояние:

```bash
docker compose ps
```

Рабочее состояние:

```text
backend    Up (healthy)
frontend   Up (healthy)
```

## Restart Policy

Для обоих контейнеров используется:

```yaml
restart: unless-stopped
```

Это позволяет автоматически перезапускать контейнер после сбоя.

Проверить конфигурацию:

```bash
docker compose config
```

## Ограничение ресурсов

Для защиты хост-системы от чрезмерного потребления ресурсов установлены лимиты.

Backend:

```yaml
deploy:
  resources:
    limits:
      cpus: "0.5"
      memory: 256M
```

Frontend:

```yaml
deploy:
  resources:
    limits:
      cpus: "0.5"
      memory: 128M
```

## Проверка безопасности образов

Для проверки Docker images используется Trivy.

Проверка backend:

```bash
trivy image momo-backend:1.0
```

Результат:

```text
Vulnerabilities: 0
```

Backend не содержит обнаруженных Trivy уязвимостей.

Проверка frontend:

```bash
trivy image momo-frontend:1.0
```

Результат также не содержит обнаруженных уязвимостей.

Таким образом, оба production-образа прошли локальную проверку Trivy без обнаруженных уязвимостей.

## Проверка API

### Healthcheck

```bash
curl --fail http://localhost:8081/health
```

### Получение товаров

```bash
curl --fail http://localhost:8081/products
```

### Проверка frontend

```bash
curl --fail http://localhost:8088/momo-store/
```

### Проверка контейнеров

```bash
docker compose ps
```

## Структура проекта

```text
.
├── backend/
│   ├── cmd/
│   │   └── api/
│   ├── internal/
│   ├── Dockerfile
│   ├── go.mod
│   └── go.sum
│
├── frontend/
│   ├── src/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   └── vue.config.js
│
├── docker-compose.yml
├── order_id_secret.txt
├── .gitignore
└── README.md
```

## Использованные технологии

### Backend

* Go
* HTTP API
* Docker
* Alpine Linux
* `scratch`

### Frontend

* Vue.js
* Node.js
* Nginx
* Alpine Linux
* Docker

### Infrastructure

* Docker Engine
* Docker Compose
* Docker Secrets
* Docker Healthcheck
* Trivy

## Итог

В проекте реализована контейнеризация frontend и backend приложений.

Используются:

* multi-stage Docker builds;
* минимальные production-образы;
* `scratch` для backend;
* Alpine для frontend;
* непривилегированный пользователь backend;
* Docker Compose;
* healthchecks;
* `depends_on` с ожиданием готовности backend;
* `restart: unless-stopped`;
* ограничения CPU и памяти;
* Docker Secrets;
* проверка Docker images с помощью Trivy.

Фактический размер образов:

| Образ    |  Размер |
| -------- | ------: |
| Backend  | 11.8 MB |
| Frontend | 64.7 MB |

Оба контейнера успешно запускаются и имеют статус `healthy`.
