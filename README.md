# Docker + Django

---

## Что такое Docker

Docker — платформа для запуска приложений в контейнерах. Контейнер — это изолированная среда со своей файловой системой, зависимостями и сетью. Приложение внутри контейнера работает одинаково на любой машине.

**Чем отличается от виртуальной машины:**
- ВМ эмулирует целый компьютер с отдельной ОС — тяжело, медленно
- Контейнер использует ядро хост-системы — запускается за секунды, весит мегабайты

---

## Ключевые понятия

| Понятие | Что это |
|---|---|
| **Image** | Неизменяемый снимок: ОС + зависимости + код. Как класс в Python |
| **Container** | Запущенный экземпляр Image. Как объект класса |
| **Dockerfile** | Инструкция для сборки Image |
| **docker-compose** | Инструмент для запуска нескольких контейнеров вместе |
| **Volume** | Хранилище данных вне контейнера. Данные живут после перезапуска |
| **Network** | Виртуальная сеть между контейнерами |

---

## Установка

Скачай и установи **Docker Desktop**: https://docs.docker.com/get-docker/

Проверь установку:

```bash
docker --version
docker compose version
```

---

## Dockerfile

Dockerfile — это рецепт сборки образа. Каждая инструкция создаёт новый слой.

```dockerfile
# Базовый образ — берём готовый Python
FROM python:3.11-slim

# Рабочая директория внутри контейнера
WORKDIR /app

# Устанавливаем системные зависимости
RUN apt-get update && apt-get install -y libpq-dev gcc && rm -rf /var/lib/apt/lists/*

# Сначала копируем только requirements — это кешируется отдельно
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем остальной код
COPY . .

# Документируем порт (не открывает его, просто пометка)
EXPOSE 8000

# Команда при запуске контейнера
CMD ["gunicorn", "core.wsgi:application", "--bind", "0.0.0.0:8000"]
```

### Почему `requirements.txt` копируется отдельно

Docker кеширует каждый слой. Если скопировать сразу весь код, при любом изменении файла Docker будет переустанавливать все зависимости заново. Разделив копирование, зависимости пересобираются только если изменился `requirements.txt`.

### Основные инструкции

| Инструкция | Назначение |
|---|---|
| `FROM` | Базовый образ |
| `WORKDIR` | Рабочая директория, все следующие команды выполняются в ней |
| `RUN` | Выполнить команду при сборке |
| `COPY` | Скопировать файлы с хоста в образ |
| `ENV` | Установить переменную окружения |
| `EXPOSE` | Задокументировать порт |
| `CMD` | Команда по умолчанию при старте контейнера |

---

## docker-compose

В реальном проекте несколько сервисов: Django, PostgreSQL, Redis, Nginx. docker-compose описывает их все в одном файле и запускает одной командой.

```yaml
services:
  db:
    image: postgres:15              # готовый образ с Docker Hub
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data   # данные живут между перезапусками
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    build: .                        # собрать образ из Dockerfile в текущей папке
    command: >
      sh -c "python manage.py migrate &&
             gunicorn core.wsgi:application --bind 0.0.0.0:8000"
    ports:
      - "8000:8000"                 # host:container
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy  # ждать готовности БД перед стартом

volumes:
  postgres_data:
```

### Как работает `ports`

`"8000:8000"` — левая часть это порт на твоей машине, правая — порт внутри контейнера. Можно написать `"9000:8000"` и открыть приложение на `localhost:9000`.

### Как работает `healthcheck`

`depends_on` без `healthcheck` ждёт только запуска контейнера, но не готовности базы. PostgreSQL стартует быстро, но принимает соединения чуть позже. `healthcheck` запускает `pg_isready` каждые 10 секунд и только после успеха разрешает старт `web`.

### Как работают `volumes`

Данные в контейнере исчезают при его удалении. Volume хранит данные на хост-машине и монтирует их внутрь контейнера. Есть два типа:

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data  # именованный volume — управляется Docker
  - ./app:/app                              # bind mount — папка с хоста прямо в контейнер
```

Bind mount удобен в разработке: меняешь код на хосте — изменения сразу видны в контейнере без пересборки.

---

## Переменные окружения

Никогда не храни секреты в `Dockerfile` или `docker-compose.yml`. Используй `.env` файл:

```env
SECRET_KEY=your-secret-key-here
ALLOWED_HOSTS=localhost,127.0.0.1,0.0.0.0

DB_NAME=django_db
DB_USER=django_user
DB_PASSWORD=123456
DB_HOST=db
DB_PORT=5432
```

`DB_HOST=db` — это имя сервиса из docker-compose, не `localhost`. Контейнеры общаются друг с другом по имени сервиса.

Добавь `.env` в `.gitignore` — он не должен попасть в репозиторий.

---

## settings.py

```python
import os
from dotenv import load_dotenv

load_dotenv()

SECRET_KEY = os.getenv("SECRET_KEY")
DEBUG = True
ALLOWED_HOSTS = os.getenv("ALLOWED_HOSTS").split(",")

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.getenv("DB_NAME"),
        "USER": os.getenv("DB_USER"),
        "PASSWORD": os.getenv("DB_PASSWORD"),
        "HOST": os.getenv("DB_HOST"),
        "PORT": os.getenv("DB_PORT"),
    }
}
```

---

## Команды

### Сборка и запуск

```bash
# Собрать образы и запустить
docker compose up --build

# Запустить без пересборки
docker compose up

# Запустить в фоне
docker compose up -d

# Остановить
docker compose down

# Остановить и удалить volumes (удалит данные БД)
docker compose down -v
```

### Логи

```bash
# Логи всех сервисов
docker compose logs

# Логи в реальном времени
docker compose logs -f

# Логи только одного сервиса
docker compose logs -f web
```

### Выполнение команд внутри контейнера

```bash
# Django команды
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
docker compose exec web python manage.py shell

# Зайти в контейнер
docker compose exec web bash

# Зайти в PostgreSQL
docker compose exec db psql -U django_user -d django_db
```

### Управление образами и контейнерами

```bash
# Посмотреть запущенные контейнеры
docker ps

# Посмотреть все контейнеры
docker ps -a

# Посмотреть образы
docker images

# Удалить контейнер
docker rm <container_id>

# Удалить образ
docker rmi <image_name>

# Удалить всё неиспользуемое
docker system prune
```

---

## Структура проекта

```
DockerDjango/
├── core/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── manage.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── .env
└── .gitignore
```

---

## .dockerignore

Как `.gitignore` — исключает файлы из контекста сборки. Без него в образ попадёт `venv`, `.git` и прочее лишнее.

```
venv/
__pycache__/
*.pyc
.env
.git/
*.sqlite3
```

---

## Что происходит при `docker compose up --build`

1. Docker читает `docker-compose.yml`
2. Собирает образ `web` по инструкциям из `Dockerfile`
3. Скачивает образ `postgres:15` с Docker Hub (если нет локально)
4. Запускает контейнер `db`
5. Каждые 10 секунд проверяет `pg_isready` — ждёт `healthy`
6. Запускает контейнер `web`
7. Внутри `web` выполняется `migrate`, затем стартует `gunicorn`
8. Приложение доступно на `http://localhost:8000`