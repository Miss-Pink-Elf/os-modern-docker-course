# Лабораторная работа: Продакшн-образы Docker
---
## Среда
---
- **ОС:** Windows 11
- **Docker Desktop:** 4.92.0
- **Docker Client/Engine:** 29.8.0
- **Терминал:** Git Bash
- **Docker Hub логин:** `bibimka`
## Задания
---
## Dockerfile (multi-stage)
```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim

ARG LAB_LOGIN=unknown
ARG LAB_TOKEN=unknown

LABEL org.lab.login="$LAB_LOGIN" \
      org.lab.token="$LAB_TOKEN" \
      org.opencontainers.image.title="image-lab"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    ROCKET_SIZE=Small \
    PORT=8000 \
    LAB_LOGIN=$LAB_LOGIN \
    LAB_TOKEN=$LAB_TOKEN

WORKDIR /opt/app
COPY --from=builder /install /usr/local
COPY app/ ./app/

RUN useradd --create-home --shell /bin/bash appuser \
    && chown -R appuser:appuser /opt/app
USER appuser

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health').status==200 else 1)"

EXPOSE 8000

VOLUME ["/tmp"]
ENTRYPOINT ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "app.app:app"]
```

**Пояснения к командам:**
- FROM: Задаёт базовый образ, с которого начинается сборка.
- WORKDIR: Создаёт директорию (если её нет) и делает её текущей рабочей папкой для всех последующих инструкций (`RUN`, `COPY`, `CMD` и т.д.).
- COPY: Копирует файлы или папки с компьютера (из контекста сборки) внутрь образа.
- RUN: Выполняет команду внутри контейнера во время сборки образа (создаёт новый слой).
- ARG: Объявляет переменную, которую можно передать только на этапе сборки (через `--build-arg`). В готовом контейнере этой переменной уже не будет, если её не перенести в `ENV`.
- LABEL: Добавляет метаданные (ключ-значение) к образу.
- ENV: Задаёт переменные окружения, которые будут доступны приложению во время работы контейнера.
- USER: Переключает пользователя, от имени которого будут запускаться все последующие команды (`RUN`, `CMD`, `ENTRYPOINT`) и сам контейнер.
- HEALTHCHECK: Периодически запускает команду внутри контейнера, чтобы проверить, «жив» ли сервис.
- EXPOSE: Это документация для человека. Она сообщает, что приложение внутри слушает порт 8000.
- VOLUME: Объявляет точку монтирования. Данные в этой папке не будут сохраняться в слое контейнера, а будут храниться в специальной области на хосте (или в оперативной памяти).
- ENTRYPOINT: Задаёт главный процесс контейнера (PID 1). Именно он запускается при старте.
## Результаты проверки
- **Размер образа:** < 150 MB (см. вывод `docker images`)![docker_images](img_02/docker_images.png)
  126 MB
- **HEALTHCHECK:** статус `(healthy)` в `docker ps`![docker_ps](img_02/docker_ps.png)
- **Non-root:** `whoami` → `appuser`![whoami](img_02/whoami.png)
- **SIGTERM:** код выхода `0`![ExitCode_0](img_02/ExitCode_0.png)
  Контейнер завершился с кодом 0 (корректно обработал SIGTERM).
- **`--read-only`:** контейнер работает, `/health` отвечает![docker_run4](img_02/docker_run4.png)
  Проверка эндпоинтов через `curl`:![curl](img_02/curl.png)![Hello](img_02/Hello.png)
- **`ROCKET_SIZE`:** ответ `Rocket=Big` при `-e ROCKET_SIZE=Big`
## Вывод `docker history`
![docker_history](img_02/docker_history.png) Видно, что финальный образ состоит из легковесных слоев, а зависимости (6.37MB) скопированы отдельно из builder-стадии.
## Чем образ лучше «наивной» сборки
Образ собран с использованием multi-stage сборки. В первом этапе (builder) устанавливаются зависимости, а во второй (финальный) копируются только готовые библиотеки из папки `/install`. Это позволило избавиться от кэша pip, временных файлов и компиляторов, благодаря чему итоговый размер образа составляет 126 MB (вместо ~1 ГБ у «наивной» сборки на базе python:3.12). Зависимости копируются отдельно от исходного кода — при изменении только `app.py` слой с pip-пакетами берётся из кэша, что ускоряет пересборку. Приложение запускается под специально созданным пользователем `appuser`, а не под root, что повышает безопасность. `ENTRYPOINT` задан в exec-форме (`gunicorn ...`), поэтому SIGTERM передаётся напрямую процессу, обеспечивая корректное завершение. Благодаря `PYTHONDONTWRITEBYTECODE=1` и объявлению `VOLUME ["/tmp"]`, контейнер успешно работает с флагом `--read-only`. Переменная окружения `ROCKET_SIZE` передаётся при запуске через `-e`, позволяя менять поведение приложения без пересборки. Также добавлен `HEALTHCHECK`, который автоматически проверяет доступность эндпоинта `/health`.
## Ссылка на Docker Hub
- Образ: `bibimka/image-lab:97d1d5b4`
- Репозиторий: https://hub.docker.com/r/bibimka/image-lab/tags

---
## Доп. материалы
Dockerfile "наивной" сборки:
```GitBash
FROM python:3.12

WORKDIR /app

COPY requirements.txt .
COPY app/ ./app/

RUN pip install -r requirements.txt

EXPOSE 8000
ENTRYPOINT ["gunicorn", "--bind", "0.0.0.0:8000", "app.app:app"]
```
Использовался полный базовый образ `python:3.12` (без `-slim`), зависимости устанавливались одной командой без флага `--no-cache-dir`, а исходный код копировался до установки пакетов.
Сборка образа:
![dockerfile_naive](img_02/dockerfile_naive.png)
Сравнение образов:
![images_compare](img_02/images_compare.png)
