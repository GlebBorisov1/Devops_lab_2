# DevOps — запуск микросервисов через Docker Compose

Репозиторий содержит `docker-compose.yml` для сборки и запуска двух
микросервисов v2.0:

- **backend** — FastAPI-сервер, хранит данные в `data.txt`;
- **frontend** — React-приложение, раздаётся nginx'ом.

## Требования

- Docker Engine 20.10+ (или Docker Desktop)
- Docker Compose v2 (`docker compose version`)

## Структура окружения

Все три репозитория должны лежать рядом:
```bash
projects/
├── backend/
├── frontend/
└── DevOps/
    └── docker-compose.yml
```

## Итоговая структура
```bash
projects/
├── backend/                 # репозиторий lab 1 (v2.0)
│   ├── Dockerfile           ← новый
│   ├── .dockerignore        ← новый
│   ├── server.py            # +1 строка (env для DATA_FILE_PATH)
│   └── requirements.txt
├── frontend/                # репозиторий lab 1 (v2.0)
│   ├── Dockerfile           ← новый
│   ├── .dockerignore        ← новый
│   ├── nginx.conf           ← новый
│   ├── src/config.js        # небольшая правка под prod-режим
│   └── ...
└── DevOps/                  # новый репозиторий
    ├── docker-compose.yml
    ├── .gitignore
    ├── README.md
    └── data/                # сюда монтируется volume (создаётся автоматически)
```

### Архитектура
```mermaid
flowchart LR
    User(["👤 Пользователь<br/>браузер"])

    subgraph host["Хост (сервер / ноутбук)"]
        subgraph compose["Docker Compose: data-sender"]
            subgraph net["Сеть app-net (bridge) — изолированная"]

                subgraph fe["Контейнер frontend<br/>data-sender-frontend:2.0.0"]
                    NGX["nginx :80<br/>раздаёт dist/<br/>проксирует /api/*"]
                end

                subgraph be["Контейнер backend<br/>data-sender-backend:2.0.0"]
                    FAPI["FastAPI + uvicorn :8000<br/>GET/POST /api/data"]
                end

                NGX -- "proxy_pass<br/>http://backend:8000" --> FAPI
            end

            VOL["bind-mount<br/>./data:/data"]
        end
    end

    FS[("DevOps/data/data.txt<br/>на хосте")]

    User -- "HTTP :8080" --> NGX
    FAPI -- "DATA_FILE_PATH=/data/data.txt" --> VOL
    VOL == "данные переживают<br/>restart / down" ==> FS

    classDef public fill:#e0e7ff,stroke:#4f46e5,color:#1e1b4b
    classDef private fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef storage fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef user fill:#f1f5f9,stroke:#64748b,color:#0f172a

    class User user
    class NGX public
    class FAPI private
    class VOL,FS storage
```

- Порт `8080` — единственный, опубликованный на хост.
- Бэкенд недоступен снаружи; фронт обращается к нему по имени
  сервиса `http://backend:8000` внутри `app-net`.
- `data.txt` хранится в `./data/` рядом с `docker-compose.yml`.

## Запуск

```bash
cd DevOps
docker compose up -d --build
```

Открыть в браузере: `http://localhost:8080`

## Проверка запуска
```bash
# Статус контейнеров
docker compose ps

# Логи
docker compose logs -f frontend
docker compose logs -f backend

# Файл данных на хосте
ls -l data/
cat data/data.txt
```

Дальше — открыть `http://localhost:8080`, отправить сообщение, нажать
«Прочитать data.txt» — данные должны отобразиться и появиться в
DevOps/data/data.txt.

## Проверка персистентности
```bash
# 1. Отправить сообщение через UI, затем:
docker compose restart backend
# 2. Снова «Прочитать data.txt» — сообщение на месте.

# Или радикальнее:
docker compose down
docker compose up -d
cat data/data.txt   # данные сохранены
```

## Остановка
```bash
docker compose down              # остановить и удалить контейнеры
docker compose down --rmi local  # + удалить собранные образы
```

> `docker compose down` **не удаляет** `./data/` — данные остаются на хосте.